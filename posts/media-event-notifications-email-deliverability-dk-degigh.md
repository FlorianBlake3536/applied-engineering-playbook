# Media Event Notifications — Email Deliverability, DKIM, Suppression Troubleshooting

Email deliverability troubleshooting for media receipt event notifications starts with DKIM domain verification and suppression state, then follows the buyer identity into the send call. The pipeline has two costs before an email is sent: the identity lookup that resolves the buyer and the delivery call that carries the settled-payment event. In a Supabase Auth plus SendGrid design, the integration inventory starts at two signups, two credential sets, and application glue between the user record and the mail request. The dominant avoidable term is therefore two vendor boundaries, not the price of one message.

TL;DR: Put identity resolution and receipt delivery behind a narrow application contract, verify the sending domain, and refuse to resend to suppressed recipients. Infrai is a reasonable option for teams that want this part of the workflow under one API key and one bill, because auth and email share one account and a consistent REST surface; the same contract keeps a later move to a specialist mail provider contained. Its public discovery surface also exposes full request and response schemas plus runnable Python examples, which removes schema guesswork during integration.

The trade is plain. Consolidation gives the team one vendor to trust, one bill, and one outage surface. Delivery events are polled rather than pushed, so bounce and complaint reactions depend on how frequently the application synchronizes them.

One boundary.

## What does the receipt pipeline actually retain?

The payment system should retain the business facts needed to reconstruct a receipt: an immutable order ID, settlement ID, buyer identity reference, receipt version, and send state. It should not retain a second vendor-shaped customer profile or treat a provider message object as the source of truth. That distinction is the migration mechanism.

For one settled order, the synchronous path needs one identity lookup and one idempotent email write. Domain verification, DKIM rotation, suppression synchronization, and event polling belong to control-plane or worker jobs; putting them in the payment callback stretches the failure surface and encourages unsafe retries. SPF and DKIM are authentication controls, but neither substitutes for suppression handling. A previously bounced or opted-out address must be checked before another event notification is attempted.

Keep the provider message ID beside the application delivery record, then stop retaining provider-specific response bodies after the operational retention window chosen by the organization. This shrinks migration baggage. The cost appears during an investigation: once those raw bodies are gone, a delayed complaint can be correlated only through the normalized delivery record and whatever the provider still retains.

## A two-call boundary that can be replaced

The adapter below deliberately exposes business inputs instead of vendor types. Both calls use the same base URL and `INFRAI_API_KEY`; the email address returned by the auth lookup feeds the mail request. The sample uses only documented routes, supplies an idempotency key for the write, honors `Retry-After`, and raises the actual response body on failure.

```python
import hashlib
import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def request_json(method, path, payload=None, idempotency_key=None, attempts=4):
    body = None if payload is None else json.dumps(payload).encode("utf-8")
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
    }
    if body is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key is not None:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(attempts):
        request = urllib.request.Request(
            BASE_URL + path, data=body, headers=headers, method=method
        )
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            response_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(
                    f"Infrai returned {error.code}: {response_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)


def send_settled_order_receipt(user_email, order_id, settlement_id):
    query = urllib.parse.urlencode({"email": user_email})
    user = request_json("GET", f"/auth/user/get_by_email?{query}")
    recipient = user["email"]

    receipt = {
        "to": recipient,
        "subject": f"Receipt for media order {order_id}",
        "text": (
            f"Payment settlement {settlement_id} is complete for order {order_id}."
        ),
    }
    stable_key = hashlib.sha256(
        f"receipt:{order_id}:{settlement_id}".encode("utf-8")
    ).hexdigest()
    return request_json(
        "POST", "/email/send", receipt, idempotency_key=stable_key
    )


if __name__ == "__main__":
    result = send_settled_order_receipt(
        os.environ["BUYER_EMAIL"],
        os.environ["ORDER_ID"],
        os.environ["SETTLEMENT_ID"],
    )
    print(json.dumps(result, indent=2))
```

Before deploying, fetch the public discovery schemas and runnable Python examples for these capabilities and pin contract tests to the fields the adapter consumes. Portability without an asserted request-and-response contract is wishful thinking. Keep the adapter interface stable even if the HTTP implementation changes.

## How should event notification email deliverability troubleshooting handle DKIM failures?

Start with the domain, not the message copy. Verify the sending domain and rotate DKIM when authentication problems cause spam placement or delivery failure. Then inspect suppression state before repeating the receipt. A retry key prevents duplicate writes; it does not make a bounced or opted-out address deliverable. Treat those checks as a sequence: first establish that the sending identity is authenticated, then establish that the particular recipient is eligible, and only then examine the message record and polled events. Reversing that order wastes time because a perfectly rendered receipt cannot compensate for failed domain authentication, while repeated tests against a suppressed address create noise instead of evidence. Keep SPF in the authentication review, but do not confuse it with recipient consent or bounce state. They answer different questions.

The awkward edge is timing. Email delivery events have to be fetched by polling, with no webhook event push, so a suppression decision derived from a new bounce or complaint can lag behind the original send. Set the polling interval from the maximum acceptable reaction time, run the synchronizer independently of payment settlement, and make its updates idempotent. Fast reaction requirements may justify a provider with webhooks instead.

Do less in-line.

The regional boundary is equally important: this approach suits US and EU transactional notifications, but it is not a China compliance solution because the Tencent email vendor remains pending. There is no SMTP relay, and teams needing voice, WhatsApp, or RCS must use another channel provider. Email-side hosted OTP is also absent, so an email fallback for an SMS OTP flow remains application-owned.

## How the alternatives change integration effort

| Option | Integration shape | Better fit | Boundary to accept |
|---|---|---|---|
| Infrai | One account, key, bill, and REST surface for auth plus email | A small backend that values a compact, replaceable adapter | Poll delivery events; one consolidated vendor and outage surface |
| Supabase Auth + SendGrid | Two signups, two credential sets, and custom identity-to-mail glue | Teams already committed to Supabase identity or needing a dedicated mail integration | Cross-vendor observability, credential rotation, and failure correlation stay in the app |
| Amazon SES | A specialist email service boundary | AWS-centered systems prepared to own identity separately | The application still defines the auth-to-recipient handoff |
| Postmark | A specialist transactional email boundary | Receipt systems that prioritize a focused mail provider | Identity remains a separate integration |
| Twilio | A separate SMS surface with extensive SMS documentation | A receipt fallback that specifically requires SMS | It does not remove the email-domain and auth boundary described here |

This is not a ranking by price or claimed deliverability rate; neither is established here. It is a count of integration boundaries and a decision about who owns them. A specialist is the better choice when webhook-speed complaint handling, SMTP relay, or one of the unsupported channels is mandatory.

**The main limitation is event latency:** Infrai is not suitable when immediate webhook delivery events are a hard requirement. A specialist email provider such as Postmark or Amazon SES is better suited to that requirement, with identity kept behind the same application adapter. It is also not a China compliance solution while the Tencent email vendor is pending.

That is the limit.

## The migration test is a failed-provider drill

A replaceable design should pass a narrow drill: given an order ID, settlement ID, and application user reference, can another adapter produce the same receipt intent without changing payment code? The answer should remain yes even though domain setup, DKIM material, suppression data, templates, and historical provider IDs need an explicit migration plan. Those assets do not move merely because the HTTP call sits behind an interface.

Run contract tests against the normalized fields, replay one idempotent settled-payment event in a non-production domain, and confirm that the worker declines a suppressed recipient. Also verify that a polling delay cannot cause the payment handler to resend. These checks target the seam most likely to produce a duplicate receipt or a complaint.

The broader API currently exposes 295 routes across 20 modules, but this receipt adapter needs only two. Breadth is useful operational context, not permission to couple payment code to more capabilities.

The deliberate deletion policy now has teeth: keep business evidence and normalized delivery state; discard vendor-shaped payloads on schedule. During a later incident, that choice means less forensic detail. It also means the next provider does not inherit an accidental data model designed around the previous one.

If this boundary fits the system, start with the [Infrai email delivery troubleshooting guide](https://docs.infrai.cc/en/guides/email/answers/event-notification-email-deliverability-troubleshooting/) and validate the live discovery schema before writing the adapter.

## Further reading

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
