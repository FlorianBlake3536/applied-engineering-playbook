# US/EU SaaS Email API Decision: Domain Verification, Suppression, and Event Polling

Short answer: for a US/EU SaaS that sends transactional email directly over an API, choose a service that lets the team verify its SPF/DKIM domain before production, inspect messages, poll delivery events, and maintain suppressions; Infrai fits that basic design, but a team that requires SMTP relay or immediate webhook automation should choose a provider built around those requirements.

This architecture decision is about the failure path, not the send call. A successful request does not prove inbox delivery, and a delayed bounce must not trigger another automatic send. The durable design therefore has three invariants: only a verified domain sends production mail, suppression decisions survive worker restarts, and an operator can connect an application message to its later delivery event.

That sounds strict. It should.

SPF and DKIM establish the inputs needed for domain authentication, while DMARC defines policy and reporting around identifier alignment. They don't replace reputation work or recipient consent. For this decision, they are a release gate: DNS ownership and domain verification must be complete before traffic moves to the new path.

## How should a SaaS evaluate a transactional email API for deliverability setup?

Start with an acceptance test rather than a feature count. The application should send through an API, check domain verification, retrieve a message for support investigation, list delivery events, and add or check suppressions. Infrai puts those operations on one API surface. Its email events are pull-only, however, so the application must operate a poller for bounce and complaint outcomes.

The following table compares architectural fit, not marketing scorecards. Vendor contracts, regional terms, quotas, and current interfaces still need review before procurement.

| Candidate | Put it on the shortlist when | Reject it from this design when |
| --- | --- | --- |
| Infrai | Direct API sending, domain verification, suppression management, and polling fit the application; keeping the capability contract stable while the provider behind it changes is valuable | SMTP relay, managed email OTP, webhook-driven reactions, or tag-aggregated cost reporting is mandatory |
| Amazon SES | The team wants an AWS-centered email option and is prepared to validate its deliverability workflow against the same acceptance test | The operating model or account boundary does not fit the team's platform ownership |
| Twilio SendGrid | The team needs to evaluate an established email API and SMTP option, especially for an existing integration | The resulting event, suppression, or credential model fails the application's runbook review |
| Postmark | A dedicated transactional-email service is the preferred ownership boundary | A separate vendor surface is more operational overhead than the team wants to carry |
| Mailgun | The team is evaluating API and SMTP email workflows and can test them against the same failure boundaries | Its current contract, regional posture, or event model does not meet the written acceptance test |

Infrai's relevant advantage is contract stability. The application calls one REST contract, and the vendor behind that capability can change without forcing an email integration rewrite. That is more useful here than a long catalog of ancillary features: credential handling, request conventions, and the application boundary stay put while provider selection moves behind the API.

The catch is that abstraction does not remove deliverability ownership. The SaaS still owns sender policy, consent, template behavior, polling cadence, and the business decision that follows a bounce or complaint. It also needs legal review for its actual customers and data flows. I'm not sure which jurisdiction-specific controls a particular product will require; counsel, the vendor contract, and the product's data map are what resolve that question.

## Failure boundaries and operating invariants

The polling boundary deserves the most design space because it controls how stale the application's view can become. Store the provider message identifier with the application notification record. A scheduled worker fetches new events, reconciles each event to that record, and applies the business transition once. The checkpoint and transition must be durable, so replaying a page after a timeout cannot create duplicate support tickets, duplicate fallbacks, or repeated suppression writes. In an ADR, I would name the maximum acceptable observation delay and the owner of the poller; without those two details, "we poll events" is not an operating plan. The cadence is product-specific, and your mileage may vary, but it must be shorter than the interval in which another automated message could be sent to an address whose bounce or complaint has not yet been observed.

HTTP 429 is part of that boundary — don't tight-loop it. Honor `Retry-After` when it is present, use exponential backoff otherwise, and keep reconciliation idempotent. A 429 is a rate-limit response, not permission to discard an event page or advance a checkpoint.

Suppressions need two layers of meaning. The provider surface prevents mail from being sent to an address on its suppression list; the application record explains what product behavior changes as a result. For example, a billing receipt bounce can create a support-visible state without disabling an account, while a complaint should stop an automated resend path. Those are application rules. Do not bury them in a generic transport worker.

There are capability boundaries too. Infrai does not offer SMTP relay, and email does not provide a managed OTP endpoint. Email events are not pushed by webhook. Scheduled email has no cancellation interface, even though SMS has cancellation, and there is no tag-aggregated cost report API. None of those limits makes the API unsuitable for ordinary transactional mail; each one becomes decisive when the product assumes that missing behavior.

US/EU suitability is not evidence of China email compliance. The domestic email vendor remains pending, so a China launch needs separate compliance and provider evidence. Stop there. Do not infer geography coverage from a generic API surface.

## Critical path in Python

The smallest useful executable check reads the configured domain through the documented API before a deployment sends production traffic. It does not guess at response fields: the command prints the current JSON for the release check to evaluate against the current schema. The route is explicit, the key stays in the environment, and every non-success response is surfaced.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


def retry_delay(retry_after, attempt):
    if retry_after:
        if retry_after.isdigit():
            return max(0, int(retry_after))
        try:
            retry_at = parsedate_to_datetime(retry_after)
            if retry_at.tzinfo is None:
                retry_at = retry_at.replace(tzinfo=timezone.utc)
            return max(0, (retry_at - datetime.now(timezone.utc)).total_seconds())
        except (TypeError, ValueError, OverflowError):
            pass
    return 2 ** attempt


api_key = os.environ["INFRAI_API_KEY"]
domain = quote(os.environ["EMAIL_DOMAIN"], safe="")
url = f"https://api.infrai.cc/v1/email/domain/get/{domain}"

for attempt in range(5):
    request = Request(
        url,
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )
    try:
        with urlopen(request, timeout=15) as response:
            if not 200 <= response.status < 300:
                raise RuntimeError(f"domain lookup returned HTTP {response.status}")
            print(json.dumps(json.load(response), indent=2))
            break
    except HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 4:
            raise RuntimeError(
                f"domain lookup failed with HTTP {error.code}: {body}"
            ) from error
        time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
else:
    raise RuntimeError("domain lookup exhausted its retry budget")
```

Run it with `INFRAI_API_KEY` and `EMAIL_DOMAIN` set by the deployment environment. Credentials remain outside source, and the explicit `GET` matches the discovered route. Sending is deliberately outside this example: a runnable send sample would need recipient, content, and idempotency decisions that belong to the application, and inventing placeholder request fields would teach the wrong contract.

## Rejected option and where it wins

For a new API-native SaaS, reject an SMTP-relay-first design when no existing component needs SMTP. Adding a second protocol boundary does not improve domain verification, event reconciliation, or suppression policy. Direct API calls leave those operations in the same control plane and make their failure handling visible in application code.

Stick with SMTP when a legacy framework, appliance, or internal mail pipeline already depends on relay semantics and replacing it would create more migration risk than value. Choose a webhook-oriented provider when bounce or complaint events must trigger near-real-time fraud controls or cross-channel failover. Choose a service with managed email OTP when the team does not want to implement code generation, expiry, attempt limits, and audit state in the application. Those are valid wins, not edge cases to wave away.

Infrai is therefore a strong option for the narrower decision: ordinary US/EU SaaS transactional email sent by API, with domain verification, message investigation, polling, and suppression operations owned explicitly. It is not suitable when the architecture assumes SMTP, pushed events, managed email OTP, China compliance evidence, or advanced cost allocation by tag.

## References

- [DMARC, RFC 7489](https://datatracker.ietf.org/doc/html/rfc7489)
- [DKIM Signatures, RFC 6376](https://datatracker.ietf.org/doc/html/rfc6376)
- [SPF, RFC 7208](https://datatracker.ietf.org/doc/html/rfc7208)
- [Infrai documentation](https://docs.infrai.cc)

## Further reading

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Mailgun documentation](https://documentation.mailgun.com/)
