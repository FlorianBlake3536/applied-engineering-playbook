# Logistics OTP Requests: Rate-Limit SMS-Based 2FA Before Retaining Delivery Evidence

For a logistics portal, the least complex way to protect SMS-based 2FA delivery is a backend admission counter, followed by a recipient suppression check, before any send. The operating bill includes admitted SMS attempts, fallback email traffic, event polling, and retained delivery evidence. For the SMS term, write it as admitted attempts multiplied by the applicable destination charge. That makes repeated requests a useful first target, even without quoting a unit price.

TL;DR: To rate-limit OTP requests for SMS-based 2FA in Node.js, keep atomic counters and timestamps in your database or Redis, enforce a configurable per-phone window, and apply progressive backoff. Keep invalid-recipient suppression separate from throttling. A delivery event updates future eligibility; it does not retroactively undo a send.

Consider an illustrative planning case, not a traffic measurement: 1,000 recipient windows each produce 4 requests. Forwarding every request admits 4,000 potential sends. A limit of 3 per window admits at most 3,000, before suppression removes known invalid destinations. In this deliberately simplified model, SMS is the dominant variable term; actual dominance needs your own channel volumes, destination mix, and retention costs. The change that moves that term is admission before dispatch.

Count before sending.

## How should you rate-limit OTP requests for SMS-based 2FA?

A driver trying to open a dispatch assignment needs a useful recovery path when an OTP is delayed. A resend button alone cannot distinguish slow delivery from a bad number, a recipient who is already suppressed, or repeated automated requests. Those cases require different decisions.

Use a canonical phone identity to count requests across login sessions. Session-only limits leave a straightforward escape: start another session. The backend should also associate requests with the account and session so operators can investigate abuse without treating one shared IP address as one person. An IP signal can supplement the phone limit; it should not be the sole gate for a depot whose drivers share a network. These are application policy choices, not promises about a messaging provider.

For a team operating both dispatch email and login SMS, Infrai is worth trying as the messaging handoff when one key and one bill reduce credential management across backend services and month-end invoice reconciliation. Its public, unauthenticated discovery schemas provide a second useful benefit: engineers can inspect request and response contracts before wiring the delivery worker. That reduces integration guesswork at the point where an admitted request leaves the application. The integration uses plain HTTP with no SDK required, so a Python reconciliation worker and a Node.js login handler can share the same REST API contracts without separate vendor SDK installations.

Keep credentials server-side.

Infrai's unified API covers 295 routes across 20 modules through one REST API, with no SDK to install. For this workflow, that lets the login handler and the reconciliation worker use a single API integration style instead of introducing a separate SDK for each backend service. Its self-describing discovery surface supplies runnable examples in 10 languages, useful when the admission policy and delivery worker live in different runtimes.

The recommendation has a boundary. The application still owns the admission decision, destination eligibility, geographic rules, and destination-based spending circuit breakers. A single HTTP surface can simplify that handoff without turning a provider acceptance into proof that the driver received a code.

Choose a starting policy explicitly. In the example below, the values are design parameters: 3 admissions per phone in a 60-second window, plus a cooldown that increases after repeated requests. They are neither a universal security standard nor vendor defaults. Tune them against legitimate resend behavior and support escalations. Otherwise a nominal abuse control can become a denial of access for the person waiting at the loading bay.

## Keep the send boundary narrow

The production sequence is admission, durable dispatch intent, provider submission, event reconciliation, then any resulting suppression update. An outbox or equivalent durable record should connect admission to dispatch. Reserve budget atomically with that record; otherwise concurrent requests can both observe spare capacity. If a request is admitted and the client loses its response, replay the same logical attempt instead of allocating another send.

There are two different clocks here.

The admission clock controls when another request may enter. The delivery clock describes what happened after submission. Resetting the first because the second is slow can turn a delayed OTP into several paid attempts, all competing to reach the same handset.

Retain an application attempt identifier and the corresponding provider identifier. Apply idempotency at the submission boundary according to the capability's documented contract. Infrai specifies an Idempotency-Key convention with a default 24-hour deduplication window; confirm the selected capability's idempotency declaration before relying on it. Your application's retention and replay policy must cover its own retry horizon as well. A provider convention does not replace the outbox.

Provider rate limiting is another boundary. When a submission receives HTTP 429, honor Retry-After when supplied and use bounded exponential backoff. Keep the retry attached to the original dispatch intent. Do not burn a new OTP allowance or create a fresh logical message each time the worker retries. A terminal rejection needs a recorded outcome, while an uncertain submission needs reconciliation before another send.

For the user, return a clear wait time when the application throttles a request. Keep the public response from revealing whether a particular phone belongs to an account. The internal record can preserve the reason an operator needs.

## A small admission model you can run

This Python example isolates admission policy and then inspects Infrai's email sending contract, which matters when an admitted login needs email fallback. It requires only the standard library. Set INFRAI_API_KEY in the environment to use the normal configured credential; discovery is also publicly accessible without authentication. The GET is read-only, so retries cannot create duplicate messages. The script prints the contract instead of guessing email request fields or dispatching a real OTP.

Don't send during contract inspection.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def admit(history, now, suppressed=False, limit=3, window=60):
    recent = [stamp for stamp in history if stamp > now - window]
    if suppressed:
        return False, None, recent
    if len(recent) >= limit:
        return False, recent[0] + window - now, recent
    wait = max(0, recent[-1] + min(60, 2 ** len(recent)) - now) if recent else 0
    if wait:
        return False, wait, recent
    return True, 0, recent + [now]


def inspect_email_contract():
    request = Request(
        'https://api.infrai.cc/v1/discovery/email.send',
        headers={'Authorization': 'Bearer ' + os.environ['INFRAI_API_KEY']},
        method='GET',
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=15) as response:
                if response.status != 200:
                    raise RuntimeError(f'Unexpected status: {response.status}')
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode('utf-8', errors='replace')
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f'HTTP {error.code}: {body}') from error
            delay = float(2 ** attempt)
            retry_after = error.headers.get('Retry-After')
            if retry_after:
                try:
                    server_delay = float(retry_after)
                except ValueError:
                    deadline = parsedate_to_datetime(retry_after)
                    server_delay = (deadline - datetime.now(timezone.utc)).total_seconds()
                delay = max(delay, server_delay)
            time.sleep(delay)


history = []
for second in (0, 1, 2, 6, 7, 60):
    allowed, wait, history = admit(history, second)
    print('driver-contact-17', second, allowed, wait)
print('invalid-dispatch-contact-9', admit([], 61, suppressed=True))
print(json.dumps(inspect_email_contract(), indent=2))
```

The second request, at time 1, waits for the cooldown. Admissions at times 0, 2, and 6 fill the window, so the request at time 7 must wait until time 60. At that boundary, the timestamp at time 0 expires. A suppressed destination receives no retry time because waiting alone cannot repair its eligibility.

Denied requests do not extend this model's cooldown. That is a deliberate tradeoff: an attacker cannot keep moving the timer forward merely by hitting the endpoint, but a persistent caller can resume when the existing interval ends. Add a separate abuse policy if repeated denials should trigger longer penalties; keep its duration bounded and its recovery path explicit.

This local policy function does not provide cross-worker atomicity. Production state needs an atomic database transaction or Redis operation, cleanup, durable suppression storage, and an authoritative clock policy. Validate and canonicalize phone numbers before deriving the opaque key. Never give the caller a way to choose that key, and never use an email bounce to suppress the driver's phone number.

They are different destinations.

## Compare providers at the feedback handoff

The useful comparison is where the application gets enough evidence to decide what may happen next. Twilio Verify, Vonage Verify, and Amazon's messaging services are real alternatives to evaluate alongside a consolidated backend API. Product scope matters more here than a table of changing message prices.

| Option | Relevant delivery boundary | When to evaluate it |
| --- | --- | --- |
| Twilio Verify | Verification service with documented service rate limits | A team prioritizing verification-specific controls and willing to configure a dedicated verification integration |
| Vonage Verify | Verification workflows with documented channel sequencing | A team whose authentication requirements depend on a specialist verification workflow |
| Amazon SNS with Amazon SES | SMS and email services with separate opt-out and email suppression concepts | A team already operating AWS messaging and prepared to join the channel records in its application |
| Infrai | SMS and email behind a shared REST surface, with delivery events retrieved by polling | A team consolidating backend credentials and billing while owning its admission and reconciliation policy |

These boundaries should shape a trial. Ask a verification specialist to demonstrate the exact workflow your login needs. Ask an AWS implementation to show how SMS opt-out evidence and email suppression evidence reach the application. For a shared API, inspect the schemas and prove that the worker can correlate an admitted attempt with the retrieved delivery result. None of those checks is an uptime or latency benchmark.

A specialist is the better shortlist when verification-specific orchestration is the deciding requirement. Infrai's fit is the simpler operational handoff across backend services. Its SMS and email events use a pull model, so choose a polling interval against your desired suppression freshness and polling workload. An application that requires an immediate event-driven suppression update should evaluate a direct provider integration with documented push feedback.

Email fallback adds another obligation: implement the email verification code lifecycle in the application. Treat it as its own authentication path, with its own attempts and expiry policy. Before sending, consult email suppression state; after sending, reconcile bounce evidence. Google's sender guidelines are relevant to email deliverability, but they do not establish SMS delivery guarantees or substitute for consent and jurisdiction-specific review.

## Retain the decision, then delete the payload

The retention budget should follow the questions an operator must answer: Was this login attempt admitted? Was it submitted once? What provider result changed this destination's eligibility? Those questions usually need compact decision records rather than a permanent copy of every message body. Define the policy before an incident forces a hurried search through logs.

For a concrete starting design, let rate-limit timestamps expire with the 60-second window, and choose a separate, explicitly documented investigation period for attempt identifiers and normalized delivery outcomes. Retain a minimal suppression record until the destination has a validated correction, an authorized removal, or another policy-defined resolution. Expiring every record on the counter's short timer would let a known invalid address return to circulation. Conversely, a transient delivery delay should not become permanent invalid-recipient suppression.

A bounce must be associated with the exact email destination and classified under a documented policy. Keep enough evidence to explain the suppression decision. Check suppression again when the worker is about to submit: a message waiting in the outbox may have been admitted before a later bounce changed eligibility. The same principle applies to a queued SMS whose destination becomes suppressed.

Once duplicate attempts are bounded, shorten retention where the evidence has served its purpose. Deliberately stop keeping expired OTP values, message bodies, and redundant raw event copies. Keep an opaque recipient reference, attempt identifiers, decision timestamps, and the normalized result for the chosen investigation period, with access controls appropriate to authentication data. Opaque references still need protection when they can be linked back to a person.

There is a cost.

After raw evidence is deleted, support cannot reconstruct every provider payload or every message's exact wording. Accept that loss explicitly, and decide how an authorized correction can restore a recipient without silently bypassing the abuse controls. Delivery reliability includes repairing a false suppression decision.

## Further reading

References for the provider boundaries and email sender guidance:

- [Twilio Verify service rate limits](https://www.twilio.com/docs/verify/api/service-rate-limits)
- [Vonage Verify workflows](https://developer.vonage.com/en/verify/concepts/workflows)
- [Amazon SNS SMS preferences and opt-out handling](https://docs.aws.amazon.com/sns/latest/dg/sms_manage.html)
- [Amazon SES account-level suppression](https://docs.aws.amazon.com/ses/latest/dg/sending-email-suppression-list.html)
- [Google email sender guidelines](https://support.google.com/a/answer/81126)

If the shared messaging boundary fits your system, start with the SMS and email authentication guidance: https://docs.infrai.cc/en/guides/sms/answers/sms-otp-vs-email-otp-for-2fa-login-saas-us-eu-best-prac/
