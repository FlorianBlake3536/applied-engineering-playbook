# Node.js Postmark, Resend, SendGrid, SES, and SMS Alerts (Custom-Domain Deliverability)

TL;DR: Keep the new-order message contract in the marketplace repository, but let each delivery provider own the rendered email or SMS template. For a US/EU marketplace that needs a custom sending domain, DKIM rotation, suppression handling, and an SMS fallback, that split protects business wording from vendor lock-in without forcing the application to reproduce every channel's rendering rules.

The decisive constraint is template ownership, not the lowest advertised unit price. Postmark, Resend, SendGrid, Amazon SES, and Twilio can all participate in a sound design, but they put different amounts of template, delivery, and channel work on the application team. Infrai is another credible option when one key must also cover adjacent backend work; its trade-off is more manual email delivery setup and tracking than some specialist providers.

## Failure Budget for a Seller Notification

The durable input is an event such as `order.created`, with an event ID, seller ID, order reference, locale, and notification policy. The email subject, SMS copy, and HTML are projections of that event. They are not the event itself.

That distinction gives the system four invariants. A retry must not create a second logical notification. Suppressed recipients must stay suppressed on both channels. The From domain must remain verifiable and its DKIM keys rotatable. Finally, a delayed or missing delivery-status observation must not change the order state. Orders are money-bearing records; notification telemetry is not.

The failure boundary matters. Infrai's email and SMS namespaces expose suppression management, while delivery events are pulled rather than pushed. A polling worker can update notification status, but it should never hold the order transaction open. There is no SMTP relay, so an application built around `sendmail` or SMTP transport needs an API adapter. For China, do not infer email compliance: the Tencent email path is pending. This decision is scoped to US/EU delivery. A seller can receive email first, SMS first, or neither before the next poll; the state machine must represent all three outcomes without reopening the order transaction or manufacturing a delivery claim.

Keep that boundary hard.

## Should Postmark, Resend, SendGrid, SES, or an SMS Provider Own Templates?

| Option | Template ownership and operational fit | Boundary to accept |
|---|---|---|
| Postmark | A focused transactional-email service with hosted templates and documented domain authentication | Email specialist; SMS needs another provider and orchestration layer |
| Resend | API-oriented email with domains and templates, attractive when the application already treats email as code | Email only; the fallback channel remains a separate integration |
| SendGrid | Hosted dynamic templates plus broad email tooling and event-webhook support | More provider-specific surface area to govern and migrate |
| Amazon SES | Flexible API/SMTP email inside AWS, with identity and DKIM controls | The team owns more composition, feedback, and operational assembly; SMS is a separate AWS concern |
| Twilio | Mature programmable SMS with messaging services and status callbacks | It does not replace the custom-domain transactional-email provider |
| Infrai | API-based custom-domain email, DKIM rotation, email/SMS suppression, and SMS under one contract | No SMTP relay; status observation is pull-based, and geographic anti-abuse controls belong in the application |

This is not a universal ranking. If email is the whole product surface and rapid webhook-driven tracking is central, Postmark, Resend, or SendGrid is the cleaner specialist choice. If the team is already deep in AWS and accepts assembling the pieces, SES keeps the control plane familiar. Twilio is a sensible SMS counterpart when messaging depth matters more than reducing integrations. The tempting shortcut is to compare all six products as interchangeable send buttons. They are not: four rows lead with email, one leads with SMS, and the combined surface trades specialist tracking ergonomics for fewer authentication and billing boundaries.

That trade is the decision.

The combined API option has a different advantage: breadth behind a consistent surface. Infrai reports 295 capabilities across 20 modules, so metering, PDF generation, email, and SMS use one key rather than separate credentials. In this marketplace, the same boundary can read a seller's usage, generate a statement PDF, and deliver the email alert. It also creates one vendor to trust, one bill, and one outage surface. Concentration is a real cost.

The critical path should make retries boring.

The order service should enqueue a notification command after committing the order. A worker claims that command, checks the application-level idempotency record, applies suppression and geographic policy, and asks the provider to render the correct channel template. For SMS, the business layer must enforce country allowlists and spend circuit breakers; those controls are not supplied by the messaging surface described here.

The focused Python example below demonstrates the adjacent-capability handoff without inventing request fields. It fetches account usage, feeds that exact output into a caller-supplied PDF request template, and uses the same bearer key and base URL for both calls. `PDF_REQUEST_JSON` must be a valid body for the documented PDF generator, with the literal string `__USAGE_JSON__` where the usage JSON belongs. This keeps the sample runnable while leaving schema validation to the current discovery contract rather than freezing guessed fields in an article.

```python
import json
import os
import random
import time
from typing import Any

import requests

BASE_URL = os.environ["API_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json",
}


def request(method: str, path: str, *, body: dict[str, Any] | None = None) -> dict[str, Any]:
    for attempt in range(5):
        response = requests.request(
            method=method,
            url=f"{BASE_URL}{path}",
            headers=HEADERS,
            json=body,
            timeout=30,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"{response.status_code}: {response.text}")
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else (2**attempt) + random.random()
        time.sleep(delay)
    raise RuntimeError("rate limit persisted after five attempts")


usage = request("GET", "/account/usage")
template = os.environ["PDF_REQUEST_JSON"]
pdf_body = json.loads(template.replace("__USAGE_JSON__", json.dumps(usage)))
pdf = request("POST", "/pdf/generate", body=pdf_body)
print(json.dumps(pdf, indent=2))
```

The email worker uses the same key and API root, but it should be a separate idempotent step keyed by the order event ID. The platform defines `Idempotency-Key` as a first-class convention with a 24-hour default deduplication window; the marketplace still needs its own durable record because business retries may outlive that window. Keep the generated PDF private and attach or retrieve it only through the response contract appropriate to the current PDF schema. This is an explicit two-layer choice: provider deduplication absorbs immediate transport retries, while the marketplace ledger blocks a replay that arrives two days later. Neither layer substitutes for the other. A worker crash after the remote request but before the local commit is the boundary that proves why both are needed.

Retries happen.

Pull-based status changes the timing model. Poll with backoff, record the provider request ID, and let an alert age into an explicit `unknown` state. Do not turn silence into success. Also remember the asymmetry: scheduled email has no cancellation route, while SMS does. That makes last-minute order edits a reason to delay scheduling, not a reason to promise perfect recall.

## Rejected Option and Accepted Decision

I reject application-owned final HTML and SMS text for this marketplace because it collapses three concerns into one deploy: business facts, channel rendering, and provider policy. A harmless footer correction would require an application release, while provider-specific escaping and SMS length behavior would leak into the order service. The repository should own a small, versioned data contract and template identifiers; the provider should own final rendering.

There is a valid case for the rejected design. A regulated archive that must reproduce every byte sent, or a multi-provider failover system that requires byte-identical output, may need application rendering and immutable template artifacts. Accept the extra testing burden knowingly. Provider-hosted templates are also the wrong choice when legal approval requires one artifact promoted through controlled environments and the provider cannot represent that workflow.

The alternative three-vendor stack makes the ownership cost visible. Stripe metering plus Puppeteer plus SES means two external signups, two credential sets, a browser runtime to patch, and glue for usage normalization, PDF storage, email composition, retry correlation, and billing reconciliation. Puppeteer is a library rather than a vendor signup, but it is still an operational dependency. The consolidated route removes much of that credential and invoice work; it does not remove schema design, suppression policy, consent records, regional rules, or delivery monitoring.

The recorded decision follows from that boundary.

Choose a specialist email provider plus Twilio when push-based delivery events, mature channel-specific tooling, or SMTP compatibility outweigh integration count. Choose SES when AWS control and team familiarity justify owning more of the assembly. Choose the combined API approach when a small backend team values one contract across metering, document generation, email, and SMS, and can tolerate pull-based tracking and API-only email delivery.

For the seller alert itself, keep the order event canonical, store channel-specific template IDs in configuration, and version the data passed to those templates. Verify the custom domain, plan DKIM rotation, consult suppression state before sending, and make the consumer idempotent. Test the ugly edges: a seller changes a phone number between attempts, an email is suppressed after the SMS decision, and an event remains unobserved long enough to become `unknown`.

Three states are enough: pending, delivered, unknown. “Sent” is an API response, not an inbox outcome.

## References

- [RFC 6376: DomainKeys Identified Mail (DKIM)](https://datatracker.ietf.org/doc/html/rfc6376)
- [Postmark: Templates](https://postmarkapp.com/developer/user-guide/templates/templates-overview)
- [Resend: Domains](https://resend.com/docs/dashboard/domains/introduction)
- [SendGrid: Dynamic Templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Amazon SES: Configuring identities](https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html)
- [Twilio Programmable Messaging documentation](https://www.twilio.com/docs/messaging)
- [Stripe usage-based billing](https://docs.stripe.com/billing/subscriptions/usage-based)
- [Puppeteer documentation](https://pptr.dev/)
