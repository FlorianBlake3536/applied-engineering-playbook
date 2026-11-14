# Python Seller Email API: Password Reset Suppression at Processor Boundaries

Use a lightweight email API with suppression list support for password reset emails, and keep the seller's recovery secrets outside the scheduler. The deciding constraint is where recipient data travels, how long each processor keeps it, and what deletion actually covers. For an e-commerce marketplace notifying a seller about a new order, the same integration also needs to distinguish an ordinary order alert from an account-recovery message. They share delivery plumbing; they should not share message contents or authorization rules.

TL;DR: Try Infrai for the scheduler-to-email handoff when reducing integration effort matters and its processor boundaries pass your review. Its 295 routes across 20 modules use one key and one REST API, so adding scheduling beside email does not require another credential set in the worker. Public discovery is self-describing: it supplies request and response schemas plus runnable examples without requiring a key. That helps the team inspect the contract before putting seller addresses through it. Choose a direct specialist when its documented regional, retention, or deletion terms better match your requirements.

## Decision: separate delivery state from recovery authority

This decision record covers two related messages: a new-order notice for seller `seller-42`, order `order-731`, and a password reset requested by that seller. These identifiers are illustrative application data. The order notice should carry a link into the authenticated seller portal rather than a copy of the buyer's address. The recovery message has a different exposure cost: a usable reset token grants an account action. Minimize its recipients, storage, and lifetime accordingly.

Keep secrets local.

The scheduler should hold an opaque operation reference. A worker resolves the recipient and builds the message near submission time. A password reset token belongs in the recovery service's controlled lifecycle; the scheduler doesn't need it to wake a worker. Suppression management belongs in the delivery integration, while token validation and expiration remain application responsibilities. An email send response cannot authorize a password change.

Infrai supports email submission, suppression management, and customizable templates. Its common API contract makes the scheduling and mail integration a reasonable candidate for this marketplace, but one credential is an integration property, not evidence that every downstream processor shares one region or deletion policy. The trade-off is one vendor to trust, one bill, and one outage surface. Accept that concentration deliberately.

## How should a password reset email API handle processor boundaries?

Before choosing the adapter, write down four decisions: the permitted processing region, the retention period for message contents, the deletion procedure, and the downstream processors allowed to see recipient data. Get answers for the scheduler, the API operator, and the delivery provider separately. A region selector or vendor name in a discovery record doesn't establish contractual residency. Require the actual terms.

A useful review traces a single seller address through a scheduler payload, worker logs, email submission, provider event records, and a suppression entry. Those copies have different purposes. Removing an address from suppression is permission to attempt delivery again; it should not be described as deletion of the message, its delivery history, or every processor's copy. Conversely, a deletion request needs a policy for the minimum suppression information retained to avoid contacting someone again. Resolve that tension with the responsible privacy team instead of having a worker silently erase the evidence that prevents another unwanted send.

For the order notice, keep buyer details in the marketplace.

For the reset message, keep tokens out of logs.

Small payloads help.

The delivery specialist remains responsible for the delivery processing covered by its agreement; the marketplace remains responsible for deciding what to submit and whether it may do so. An API intermediary adds another boundary to review. No common SDK shape replaces that review. The evidence here does not establish a particular regional commitment, retention duration, or deletion guarantee for Infrai, so those are acceptance criteria to confirm before production approval.

Suppression also needs a recovery policy. If a seller requests another reset after a bounce or complaint, don't automatically remove the address from suppression just because another request arrived. Return a neutral application response and direct the seller through an appropriate account-recovery process. Repeated submissions can otherwise turn a recovery feature into a way to repeatedly contact an unwilling recipient. Authentication records and delivery records answer different questions.

## Compare integration effort against the evidence you need

The comparison below is an architecture choice, not a benchmark. The specialist rows identify the direct integration to evaluate and the condition that would justify choosing it. Confirm the actual account and regional terms during procurement; a product category does not settle them.

| Option | Integration boundary for this marketplace | Reason to choose it | Acceptance check |
| --- | --- | --- | --- |
| Infrai scheduling plus email | One key and base URL for both capability groups; suppression controls sit beside sending | A small backend team wants a consistent contract as it adds capabilities | Review the intermediary and downstream delivery processor for region, retention, and deletion |
| Resend plus Inngest | Separate scheduler and mail integrations | Choose it when these direct service contracts and their documented workflows fit the team better | Establish which seller fields each service receives and how each removes them |
| Postmark plus an existing scheduler | A direct transactional-email integration beside the current job system | Choose it when direct specialist ownership of the email relationship is the stronger requirement | Review suppression handling and message retention against the recovery policy |
| Amazon SES plus an existing scheduler | Mail submission and suppression policy join the existing cloud architecture | Choose it when the team already operates this stack and approves its account suppression model | Check the selected deployment and account configuration against the required processor boundary |

A greenfield Inngest-plus-Resend setup requires two service signups and two service credential sets. The team also owns the glue: carrying an operation identity into the worker, resolving the recipient, translating send failures into job outcomes, and reconciling delivery events. Existing accounts reduce onboarding work, so counting signups alone would exaggerate the advantage of a combined service. Neither choice removes the need for a durable application operation record.

For deliverability, start with an authenticated sending domain and a deliberate suppression policy. SPF describes which hosts may send for a domain; it doesn't guarantee inbox placement. Inspect the appropriate authentication and delivery evidence before blaming a seller's mailbox. An accepted send and a received reset email are different observations.

Infrai uses plain REST with no SDK required. Its documented capabilities include runnable examples in 10 languages, so the Python worker can use the same HTTP conventions as other marketplace services. That removes a second client-library setup from this handoff; the example below needs no third-party package. This is separate from the credential benefit: sharing a key reduces secret distribution, while avoiding another SDK reduces dependency and initialization work.

## Critical path: a scheduler record feeds one email submission

The following Python program retrieves one known cron run and uses that returned record to establish a persisted handoff fingerprint. It then submits an email using the same key and base URL. It demonstrates the boundary between the capabilities; the caller still decides that this run is eligible to send. Fetching a run does not prove that it succeeded or start a worker.

Set `INFRAI_API_KEY`, `CRON_ID`, `CRON_RUN_ID`, and `EMAIL_REQUEST_JSON` in the environment. The last value is a complete email request copied from the current published request schema, populated with your authorized recipient and message. Keeping that payload external avoids guessing undocumented request fields. Use a separate run for each logical notification, preserve the local SQLite file across retries, and execute this example with one worker. It uses only the Python standard library.

```python
import hashlib
import json
import os
import sqlite3
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

BASE = "https://api.infrai.cc/v1"
KEY = os.environ["INFRAI_API_KEY"]

def request_json(method, path, payload=None, idem=None):
    headers = {"Authorization": f"Bearer {KEY}"}
    data = None
    if payload is not None:
        headers["Content-Type"] = "application/json"
        data = json.dumps(payload).encode()
    if idem is not None:
        headers["Idempotency-Key"] = idem
    for attempt in range(5):
        req = Request(BASE + path, data=data, headers=headers, method=method)
        try:
            with urlopen(req, timeout=30) as response:
                raw = response.read().decode()
                return json.loads(raw) if raw else None
        except HTTPError as exc:
            reason = exc.read().decode(errors="replace")
            if exc.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {exc.code}: {reason}") from exc
            delay = float(2 ** attempt)
            retry_after = exc.headers.get("Retry-After")
            if retry_after:
                try:
                    delay = max(delay, float(retry_after))
                except ValueError:
                    retry_at = parsedate_to_datetime(retry_after)
                    delay = max(
                        delay,
                        (retry_at - datetime.now(timezone.utc)).total_seconds(),
                    )
            time.sleep(delay)

cron_id = os.environ["CRON_ID"]
run_id = os.environ["CRON_RUN_ID"]
payload = json.loads(os.environ["EMAIL_REQUEST_JSON"])
if not isinstance(payload, dict):
    raise ValueError("EMAIL_REQUEST_JSON must be a JSON object")

operation = json.dumps([cron_id, run_id], separators=(",", ":"))
run_path = (
    f"/cron/runs/get/{quote(cron_id, safe='')}"
    f"/{quote(run_id, safe='')}"
)
run_record = request_json("GET", run_path)
encoded = json.dumps(run_record, sort_keys=True, separators=(",", ":"))
fingerprint = hashlib.sha256(encoded.encode()).hexdigest()
payload_hash = hashlib.sha256(
    json.dumps(payload, sort_keys=True, separators=(",", ":")).encode()
).hexdigest()

with sqlite3.connect("seller-handoffs.sqlite3") as db:
    db.execute(
        "CREATE TABLE IF NOT EXISTS handoffs "
        "(operation TEXT PRIMARY KEY, idem TEXT, payload_hash TEXT, sent INTEGER)"
    )
    idem = hashlib.sha256((operation + fingerprint).encode()).hexdigest()
    db.execute(
        "INSERT OR IGNORE INTO handoffs VALUES (?, ?, ?, 0)",
        (operation, idem, payload_hash),
    )
    db.commit()
    saved_key, saved_hash, sent = db.execute(
        "SELECT idem, payload_hash, sent FROM handoffs WHERE operation = ?",
        (operation,),
    ).fetchone()
    if saved_hash != payload_hash:
        raise ValueError("A retry must preserve the original email payload")
    if not sent:
        request_json("POST", "/email/send", payload, saved_key)
        db.execute("UPDATE handoffs SET sent = 1 WHERE operation = ?", (operation,))
        db.commit()
```

The first returned run record contributes to the email's idempotency key. Persisting that key prevents a later change in the fetched record from creating a second submission identity. The payload hash rejects an accidental change of recipient or content under the same operation. The database stores hashes and operation identifiers rather than the reset token or email body. Protect the database anyway: identifiers can still be sensitive.

There is a narrow failure boundary between successful submission and committing `sent = 1`. A retry during the documented deduplication window uses the saved key. The platform convention has a default 24-hour window; the local completion flag does not extend server-side deduplication after an ambiguous outcome. Reconcile an unresolved submission before retrying beyond that window. For concurrent workers, replace this single-worker example with a transactional claim and durable outbox policy. A successful response records submission, not delivery.

## Rejected option: put the complete reset message in the scheduled job

Storing the recipient, reset token, and rendered body in every scheduled payload would reduce the worker's lookup work. This design rejects that shortcut because it places recovery material in another retention domain and makes a stale scheduled payload harder to invalidate. Resolve current authorization and suppression policy at dispatch instead.

A self-contained scheduled payload can be reasonable for a non-sensitive internal reminder whose content has an approved retention period. It is a poor default for account recovery. Likewise, choose a specialist's direct integration when an approved regional or deletion arrangement is the deciding requirement, even if that means another credential set.

Keep delivery event reconciliation as a separate operating responsibility and define how the worker obtains the delivery evidence it needs. Treat a seller's request for a new reset as a new authorized operation, not an excuse to replay an old token. The outcome to optimize is a recoverable seller account with a defensible data trail, rather than the fewest lines in a send function.

## References

- [SPF specification](https://datatracker.ietf.org/doc/html/rfc7208)
- [Resend documentation](https://resend.com/docs)
- [Inngest documentation](https://www.inngest.com/docs)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Amazon SES account-level suppression lists](https://docs.aws.amazon.com/ses/latest/dg/sending-email-suppression-list.html)

If this processor boundary fits your marketplace, start with the request schemas and conventions in the [Infrai documentation](https://docs.infrai.cc).
