# Express.js 2FA Login with SMS OTP and Email Fallback for a Passwordless Support Form

Short answer: make the SMS OTP and email fallback two states of one passwordless login challenge, then route the verified contact form through the support queue chosen for that account. The Express.js part is small; the hard work is preserving challenge ownership, expiry, and audit evidence when delivery is late.

For an edtech product, a contact form is rarely just “send this message.” A student may need billing help, an instructor may need a course correction, and a school administrator may need a contract answer. The login challenge should establish who is allowed to submit the request, while the queue decision should remain a separate, inspectable application step.

## What should an Express.js passwordless login challenge own?

Create a login attempt before sending an SMS. Its durable record needs a random challenge ID, an account or contact ID, the normalized phone number, a hash of the OTP, an expiry time, a verification-attempt count, a channel, and a consumed flag. Add a link to the eventual support submission, but do not let the contact-form body become part of the authentication token.

The client should receive the challenge ID and a masked destination. It should not receive the code, a database key, or an instruction that implies the message definitely failed. A five-minute expiry is a reasonable policy to start testing, not a universal law. Carrier behavior, school networks, and mailbox rules vary.

The verification transaction has one important ordering constraint: check the hash, expiry, attempt limit, and consumed state, then mark the record consumed before creating the session. Two simultaneous requests with the same valid code must not produce two authenticated sessions. This is a database invariant, not a UI feature. It also gives the support workflow a clean boundary: the route can issue a session only after the challenge transition commits, and the form endpoint can reject a submission that arrives with a stale or unrelated challenge instead of trying to repair the state in application memory.

Keep it boring.

Here is the application-owned part of the state machine. The transport adapters are deliberately absent; they should accept a challenge ID and return a delivery reference, while the database remains the source of truth for verification.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
import hashlib
import hmac
import secrets


@dataclass
class LoginChallenge:
    challenge_id: str
    contact_id: str
    channel: str
    code_hash: str
    expires_at: datetime
    attempts: int = 0
    consumed: bool = False


def issue_challenge(contact_id: str, channel: str, now: datetime) -> tuple[LoginChallenge, str]:
    code = f"{secrets.randbelow(1_000_000):06d}"
    digest = hashlib.sha256(code.encode("ascii")).hexdigest()
    challenge = LoginChallenge(
        challenge_id=secrets.token_urlsafe(18),
        contact_id=contact_id,
        channel=channel,
        code_hash=digest,
        expires_at=now.replace(microsecond=0),
    )
    return challenge, code


def verify_challenge(challenge: LoginChallenge, code: str, now: datetime) -> bool:
    if challenge.consumed or now >= challenge.expires_at or challenge.attempts >= 5:
        return False
    challenge.attempts += 1
    supplied = hashlib.sha256(code.encode("ascii")).hexdigest()
    if not hmac.compare_digest(supplied, challenge.code_hash):
        return False
    challenge.consumed = True
    return True
```

The sample intentionally leaves persistence visible. In a real Express.js service, `attempts += 1` and `consumed = true` must be protected by a transaction or an equivalent compare-and-set update. Also, the expiry value above is a record boundary, not a complete policy: set it to the intended deadline when inserting the row, and never extend that deadline merely because a user pressed resend.

That last detail matters for the contact form. A verified account can be allowed to submit a message to its permitted support queues, but an unverified browser must not select an arbitrary queue by changing a request field. Derive the queue from server-side account data and validated form classification. Authentication proves the actor; authorization decides the queue.

## How should SMS OTP and email fallback behave when delivery is uncertain?

SMS is a delivery attempt, not a delivery proof. An API response saying that a request was accepted does not mean a handset displayed it. Email has the same gap: handing a message to an SMTP or email API does not prove inbox delivery or mailbox control.

Offer the email fallback after a bounded wait, but phrase the action honestly: “Still waiting? Send a code by email.” Do not claim that SMS failed unless the provider gives a trustworthy failure state and your worker has recorded it. A browser timeout is only a timeout.

Keep one parent login attempt and create a linked email challenge when the user switches channel. The email code gets its own hash, expiry, attempt budget, and consumed state. A code from the SMS child must never verify the email child, even if both belong to the same contact. When one child succeeds, expire the siblings and attach the authenticated session to the parent attempt.

I once lost more time than expected to the quiet race: the email code was accepted, then an earlier SMS arrived, and a retrying browser submitted both values. The fix was not a longer timeout. It was an atomic consumed transition plus an audit event for each channel switch. That record made the support queue decision explainable too: who authenticated, which destination was used, and when the form entered the queue.

SMS content adds a less obvious edge case. GSM-7 and UCS-2 have different character budgets, and a message can be segmented when its text or character set crosses a limit. Keep OTP templates short, avoid decorative Unicode, and test the exact rendered message rather than the source template. A six-digit code is not useful if the user receives only part of the instruction.

## Which integration boundaries keep the Express.js flow maintainable?

Separate four responsibilities: challenge storage, message transport, delivery observation, and queue authorization. The route handler can coordinate them, but it should not contain four competing policies. This is where integration effort is won or lost.

The transport interface needs an idempotency key derived from the parent attempt and a resend sequence. It should return a provider reference without logging the code or full destination. A worker can poll or consume delivery events where the chosen transport supports them, updating an internal status such as `accepted`, `delivered`, or `unknown`. None of those statuses replaces OTP verification.

For email, the application owns the security semantics even if a third party handles sending. Generate the code with a cryptographically secure source, store only a digest, enforce a maximum number of attempts, suppress repeated sends, and prevent an old email from extending the session window. Use a verified sending domain and keep authentication, consent, bounce, and suppression records separate from the login challenge.

For an edtech contact form, the event trail should include the authenticated contact ID, challenge ID, selected channel, queue chosen, message ID, and reason for any rejection. Avoid putting the OTP or full phone number into logs. Retain only what the team needs to investigate delivery, abuse, and routing disputes, with a documented deletion period. A useful record of one submission might show that an instructor authenticated by email after an SMS wait, that the server mapped the account to the curriculum-support queue, and that the message was accepted once. It should not expose the code, let an operator replay the challenge, or rely on a dashboard screenshot as the only evidence. That distinction becomes important when a parent says the form disappeared, an administrator asks why billing received a course question, or a privacy review asks which destination data was retained.

The boundary can be made explicit in a small decision table:

| Integration shape | Who owns the code | Main integration cost | Poor fit when |
| --- | --- | --- | --- |
| Application code plus SMS transport | The application | State, rate limits, and audit events | The team cannot operate abuse controls |
| Managed SMS verification plus application email | The SMS service for one channel; the application for the other | Two policy surfaces and linked challenge records | One uniform policy must cover every channel |
| Shared transport adapter for both channels | The application | Adapter contracts and delivery reconciliation | Provider-specific controls are required |
| Self-hosted email with a carrier adapter | The application and operations team | Sender reputation, carrier rules, and maintenance | The team needs a short path to production |

## What should you test before calling passwordless sign-in ready?

Happy-path tests are cheap and misleading. Test a delayed SMS followed by a successful email code, a late SMS after the email path has succeeded, two resends while the first message is in flight, a correct code after expiry, five wrong entries, and two verification requests racing on the same row. Test an account that is authenticated but not allowed to send to the chosen school-admin queue.

The integration test should assert both security and workflow outcomes. A successful challenge creates one session and one contact-form handoff. A rejected challenge creates neither. A queue change after verification must be rejected unless it passes the same authorization check as the initial submission.

Track metrics that expose policy failures: completion rate by channel, time from issue to verification, resend count, invalid-code count, mailbox or carrier region, and queue-routing rejection rate. Watch abuse separately from delivery. A spike in SMS requests can be an attack even when delivery metrics look healthy.

I'm not sure one global SMS wait window will suit every country or school network; your mileage may vary. Make the window configurable, record the policy version with each challenge, and change it from observed evidence rather than from a guess hidden in an Express route.

The catch is that this design is not suitable when you need a fully managed email OTP product, hardware-backed authentication, or a channel with regulatory controls your transport layer cannot provide. Keep a specialized authentication service in those cases, or add that capability before moving the fallback into production. Integration effort is the decision axis here: owning the state machine gives control, but it also gives the team the long tail of abuse, compliance, and delivery operations.

## A compact rollout for the support queue

Start with one feature flag for SMS-first login and one database-backed challenge model. Route a small, known cohort of edtech contacts through it. Keep the existing authenticated form path available for rollback, but do not silently bypass verification after a delivery timeout.

During rollout, compare the new flow with the old one using the same queue taxonomy. Review duplicate submissions, abandoned challenges, channel switches, and misrouted requests. Then expand by account type or region only when the audit trail and rate limits are doing what the policy says.

The final implementation is not a clever Express.js endpoint. It is a short-lived, single-use credential joined to a separately authorized support action, with enough evidence to explain every branch.

## References

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.twilio.com/docs/glossary/what-sms-character-limit
- https://expressjs.com/en/guide/routing.html
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
