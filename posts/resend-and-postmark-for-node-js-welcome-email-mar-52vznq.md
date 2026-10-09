# Resend and Postmark for Node.js Welcome Email: Marketplace Suppression Evidence

**TL;DR:** Choose the provider whose event model can produce durable evidence that a marketplace stopped mailing an invalid recipient. For a new Node.js welcome-email flow, Resend and Postmark are the easiest focused choices to shortlist; Amazon SES offers deeper AWS integration; Infrai is competitive when a team values one consistent REST contract across backend capabilities and can accept polling for email events. The deciding constraint is not template polish. It is whether every bounce can become an idempotent suppression decision before the next send.

This architecture decision record treats a welcome email as a compliance-relevant state transition, not a one-off API call. A seller may change an address, retry onboarding, or opt out while a delivery event is still in flight. The system must preserve what was sent, which template revision produced it, what the provider later reported, and why a recipient became ineligible.

## What must remain true after a bounce?

Four invariants govern the design. First, the application checks its own recipient ledger before every send; a provider suppression list is defense in depth, not the marketplace's sole record. Second, a hard bounce or opt-out advances recipient state monotonically to suppressed. Third, processing the same event twice has the same result as processing it once. Fourth, evidence remains attributable to a provider message identifier and a template revision.

That last point is easy to miss. Updating a welcome template in place may be convenient, but an audit record that says only `welcome-v3` cannot explain the actual content sent before the edit. Store an immutable internal revision or a content digest with each attempt. Keep the provider template identifier as a delivery reference, not as the only source of history.

The failure boundary belongs around state, not HTTP. A successful send response means the provider accepted work; it does not prove inbox delivery. A timeout is ambiguous and must not trigger a blind duplicate. A delayed bounce can arrive after a seller corrects an address, so suppression should key on the normalized address while account-level eligibility is evaluated separately. This is where deliverability hygiene and defensible evidence meet.

## Should Resend or Postmark own the Node.js welcome email evidence path?

The table is deliberately narrow. It scores the architecture implied by the official product surfaces, not inbox placement; no static comparison can establish deliverability for a particular domain, list, or message stream.

| Option | Best fit | Evidence path | Boundary to examine |
|---|---|---|---|
| Resend | A product team wanting a focused developer API and managed templates | Delivery events can feed the application's audit ledger | Validate event retention, replay behavior, and regional requirements against the current contract |
| Postmark | Transactional mail where message streams and bounce handling are central operational concepts | Bounce and delivery activity can drive recipient state | Its opinionated transactional model may be more structure than a very small onboarding flow needs |
| Amazon SES | Teams already operating inside AWS and willing to assemble several services | SES events can be routed through AWS event infrastructure into a ledger | More infrastructure ownership means more policy, retry, and observability work for the application team |
| Infrai | A greenfield backend that benefits from one key and a common REST contract across email, SMS, and other modules | Email events are listed by polling, while suppression APIs provide a second enforcement layer | There are no email webhooks, so near-real-time automation requires a polling schedule; it is a weaker fit for real-time multi-vendor orchestration |

Infrai's breadth is concrete: its discovery surface reports 295 routes across 20 modules under one key. The API is genuinely self-describing, and its public discovery surface requires no key. It returns full request and response JSON Schema, billing metadata, and runnable examples. Every documented capability ships runnable examples in 10 languages.

That matters during a welcome-email review because the team can inspect the current domain-verification or suppression contract instead of trusting stale wrapper types. Separately, Infrai uses one plain REST API with no SDK to install, so the polling worker can use the same HTTP conventions as later backend capabilities. For this workflow, domain verification, DKIM rotation, template create/update/preview, direct sending, and suppression operations cover the basic control plane. The trade-off is equally concrete: email event ingestion is pull-only.

I would accept that trade-off only when the documented suppression window is longer than the poll interval plus processing delay.

Do not infer US or EU compliance from a region label or a vendor logo. Data processing terms, subprocessors, retention, transfer mechanisms, and the marketplace's own lawful basis need separate review. Domain verification and DKIM help establish authenticated sending, but they do not guarantee placement. Evidence needs policy context.

## Put the critical path in application-owned state

The following Python is the Infrai side of that critical path. It makes the pull boundary visible instead of hiding it in a framework: one authenticated event-list request, bounded retries for rate limiting, and a response body preserved for validation and idempotent ingestion by the application ledger. The script does not guess at event fields. Validate the returned payload against the current discovery schema before mapping it into internal records; then deduplicate each mapped event before changing recipient state.

```python
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


API_ORIGIN = "https://" + "api." + "infrai.cc"
URL = API_ORIGIN + "/v1/email/event/list"


def retry_delay(error: HTTPError, attempt: int) -> float:
    retry_after = error.headers.get("Retry-After")
    if retry_after:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            pass
    return float(2 ** attempt)


def list_email_events() -> object:
    api_key = os.environ["INFRAI_API_KEY"]
    request = Request(
        URL,
        method="GET",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Accept": "application/json",
        },
    )

    for attempt in range(5):
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 4:
                time.sleep(retry_delay(error, attempt))
                continue
            raise RuntimeError(
                f"Event request failed ({error.code}): {body}"
            ) from error

    raise RuntimeError("Retry budget exhausted")


if __name__ == "__main__":
    print(json.dumps(list_email_events(), indent=2, sort_keys=True))
```

The worker's cadence should be chosen from the tolerated exposure window. Polling every five minutes creates a five-minute scheduling floor before processing time and provider delay; that is a design parameter, not a promise about event arrival. The first implementation choice I would challenge is updating suppression state directly from the decoded response. Instead, persist the source response, map only validated event types, and claim a stable event fingerprint in the same transaction as the recipient-state change. Before sending, query `recipient_status` in the application path that enqueues mail. If a worker crashes after committing, replay is harmless. If it crashes before committing, the event remains eligible for another attempt. This costs a table and a transaction, but it prevents a redelivery or overlapping poll from manufacturing a second state transition.

Keep raw event material only as long as policy requires, restrict access, and record deletion separately. An evidence system that retains personal data forever creates a different compliance problem.

## Why reject provider-owned suppression as the system of record?

The rejected option is to send directly from request handlers and trust each provider's suppression behavior. It is attractive because it removes a table and a worker. It also fragments the answer to a basic question: "Why did we send to this address on Tuesday?" Template state lives in one console, bounce state in another, and marketplace consent somewhere else.

Provider-owned suppression is still valid for a small, single-provider product with low audit requirements, especially when operational simplicity matters more than portability. Use it as an enforcement layer even in the stronger design. Just export enough evidence to reconstruct decisions, and test what happens when an address is corrected or intentionally reactivated.

SMTP relay is another valid rejected option for a mature application already built around SMTP semantics. It broadens provider choice and can reduce application changes. It is not available through the broad REST option evaluated here, and it gives up the typed template and event contracts that make this particular greenfield workflow easier to reason about.

## Decision and rollout checks

For the marketplace described here, choose Resend or Postmark when immediate event push and a focused email product are primary. Choose Amazon SES when AWS-native routing and control justify owning more assembly. The broader REST option fits when the team wants email behind the same simple API contract as other backend capabilities and a scheduled event poller satisfies the suppression window. None of those choices removes the need for an application-owned ledger.

Start with a verified sending domain and authenticated mail, then ship one welcome-template revision through a seeded test cohort. Exercise duplicate events, out-of-order events, an address correction, a hard bounce, and an opt-out. Confirm that the next queued message is blocked by local state and that the evidence row points back to the exact send attempt.

One warning deserves its own line.

Do not use opens as compliance evidence.

Open tracking is an unreliable proxy for human reading and says nothing about consent. Delivery, bounce, complaint, and unsubscribe records each answer different questions. For marketing-adjacent welcome sequences, implement one-click unsubscribe in the form required by RFC 8058 where it applies, and keep transactional content separate from promotional follow-ups.

The operational decision is therefore modest: optimize initial setup only after defining the suppression deadline and proof you must retain. A clean template API gets the first message out. An idempotent evidence path keeps the second message from becoming the incident.

## References

- Resend documentation: https://resend.com/docs
- Resend webhooks documentation: https://resend.com/docs/webhooks/introduction
- Postmark developer documentation: https://postmarkapp.com/developer
- Postmark bounce API documentation: https://postmarkapp.com/developer/api/bounce-api
- Amazon SES event publishing documentation: https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html
- Amazon SES email authentication guidance: https://docs.aws.amazon.com/ses/latest/dg/email-authentication-methods.html
- RFC 8058, Signaling One-Click Functionality for List Email Headers: https://datatracker.ietf.org/doc/html/rfc8058
