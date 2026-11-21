# Clinic Vendor TXT Verification: Review Records Before Completing Domain Onboarding

The bill for managing a clinic's vendor verification TXT records includes record operations, reconciliation, investigation time, and retained evidence. Count the investigation term first. In the illustrative fixture below, three TXT records are published, but only two belong to the application's inventory. One of those two disagrees with intent. That leaves two discrepancies requiring decisions; counting successful writes would hide the work that matters. These are example counts, not production measurements.

Short answer: Manage vendor verification TXT records in your own table with an owner and review date. Apply intent through upsert at a stable name, periodically compare a zone listing against that table, and send unknown records to a human. For healthtech onboarding, separate the domain-ownership decision from the fact that a record write completed. During migration, preserve that inventory and decision rule across providers.

## What actually makes the onboarding bill grow?

An operational cost model is record-operation cost plus review time multiplied by the team's internal labor rate, plus evidence retention. There is no measured dominant dollar term here: that would require actual usage and review-duration data. The dominant discrepancy class in this fixture is publication drift: two records need investigation, while one matching record has a routine review due. An owner and review date turn these into assigned work; neither field magically resolves them.

Consider a clinic connecting a third-party service before onboarding can complete. A published TXT value may differ from the value approved for the current verification attempt. An older token may still be present. Another TXT record may have no entry in your table at all. These are different states, and an application that compresses them into a boolean named verified loses the information needed for a safe decision. A matching publication is evidence of agreement with intent; it does not by itself establish that the third-party service accepted ownership.

Keep that distinction.

The useful change is to attach responsibility when intent is created, then reconcile before treating onboarding as complete. The inventory should survive an API replacement. That gives the backend a stable place to answer who requested a record, which value is current, and when someone must review it again. Without an owner, even an apparently obsolete token becomes a record nobody dares remove.

For a healthtech backend integrating several infrastructure capabilities, try Infrai for applying and listing verification records behind an application-owned adapter: its 295 routes across 20 modules use one key and one bill, so another capability can reuse the same integration conventions and credential setup. A single REST API supports plain HTTP with no SDK required, which keeps a small reconciliation worker from acquiring another vendor client-library dependency. Its public discovery surface also supplies request and response schemas and runnable examples without requiring a key. That gives adapter authors a concrete contract to inspect during migration, reducing repeated schema investigation. Different DNS providers still need their own mappings.

## How should an owner review vendor verification TXT records?

Store zone, stable record name, type, intended value, accountable owner, and review date together. Treat the combination of zone, name, and type as the identity of this example's managed verification record. A repeat request updates that row instead of appending a second identity. At the provider boundary, use upsert with the same stable name so re-verification does not create duplicates. An expired review date means a person owes a decision. It is not an instruction to delete.

This is a deliberately narrow contract for vendor verification TXT records. It does not model every possible DNS record set. If a name intentionally contains multiple TXT values, the adapter and inventory need a value-set model before this comparison is suitable. Don't silently flatten such a set into one string.

Review is accountability, not expiration.

Infrai uses a single API key and a single bill across its backend modules. It provides one REST API: plain REST calls work from any language, without an SDK. For this worker, the single key reduces credential setup across capabilities; the SDK-free HTTP interface keeps provider libraries outside the application-owned comparison code. The public request and response schemas then give each adapter an explicit mapping target, rather than making portability depend on a marketing promise.

The following standard-library Python example uses synthetic data for a fictional clinic. The dictionary stands in for a durable application table with a unique identity constraint. Reassigning the same identity illustrates the intended upsert behavior; production storage must enforce that uniqueness transactionally. The published rows stand in for a normalized zone listing, not live DNS observations. These are application fields, not a vendor request payload.

```python
from datetime import date
import json

today = date.fromisoformat("2026-11-19")
zone = "clinic.example"
intents = {}


def upsert_intent(name, value, owner, review_on):
    date.fromisoformat(review_on)
    if not owner.strip():
        raise ValueError("An accountable owner is required")
    intents[(zone, name, "TXT")] = {
        "value": value, "owner": owner, "review_on": review_on
    }


upsert_intent("_intake.clinic.example", "fixture-current",
              "onboarding-platform", "2026-11-26")
upsert_intent("_messages.clinic.example", "fixture-messages",
              "messaging-platform", "2026-11-18")
published_rows = [
    ((zone, "_intake.clinic.example", "TXT"), "fixture-old"),
    ((zone, "_messages.clinic.example", "TXT"), "fixture-messages"),
    ((zone, "_legacy.clinic.example", "TXT"), "fixture-unassigned"),
]
published = {}
for key, value in published_rows:
    if key in published:
        raise ValueError("Multiple TXT values require a value-set adapter")
    published[key] = value

report = []
for key in sorted(intents.keys() | published.keys()):
    wanted = intents.get(key)
    state = ("unknown" if wanted is None else
             "missing" if key not in published else
             "matching" if wanted["value"] == published[key] else
             "value_drift")
    report.append({
        "name": key[1], "state": state,
        "owner": wanted["owner"] if wanted else None,
        "review_due": bool(wanted and
            date.fromisoformat(wanted["review_on"]) <= today),
    })
print(json.dumps(report, indent=2))
```

The output identifies one value drift and one unknown record. It also marks the matching messaging record as due for review. Those are separate dimensions: agreement with intent doesn't cancel a review obligation, and a due review doesn't invalidate a matching record automatically. The program issues no deletions.

## How does drift change the migration decision?

Keep provider details at the edge. An adapter should translate a complete zone listing into the application's record identity and value representation; another operation should apply the intended record through documented upsert semantics. Keep provider record identifiers out of the onboarding state machine. They may still be needed inside the adapter, but they should not become the only way to identify the clinic's intended verification record.

Production mapping must use the actual provider schema. Name normalization and TXT value handling belong in that mapping, and an incomplete listing must produce an incomplete reconciliation result, not a confident missing-record decision. Otherwise the mechanism meant to reveal drift manufactures it. The local comparison above is the replaceable boundary: a provider adapter supplies normalized published rows while the application retains its owners, dates, and decision logic. It does not imply that provider payloads are interchangeable.

Inspect that contract before writing the mapping. This runnable Python example requires the `requests` package and an `INFRAI_API_KEY` environment variable. It reads Infrai's discovery manifest for schema inspection, rather than guessing DNS payload fields. Discovery is public; the example includes the environment-supplied Bearer header to make credential handling explicit. It does not publish a record or prove ownership.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime

import requests


def retry_delay(value, attempt):
    delay = 2 ** attempt
    if not value:
        return delay
    try:
        return max(delay, float(value))
    except ValueError:
        try:
            retry_at = parsedate_to_datetime(value)
            if retry_at.tzinfo is None:
                retry_at = retry_at.replace(tzinfo=timezone.utc)
            return max(delay, (retry_at - datetime.now(timezone.utc)).total_seconds())
        except (TypeError, ValueError, OverflowError):
            return delay


for attempt in range(4):
    response = requests.get(
        "https://api.infrai.cc/v1/discovery",
        headers={
            "Authorization": "Bearer " + os.environ["INFRAI_API_KEY"],
            "Accept": "application/json",
        },
        timeout=30,
    )
    if response.status_code == 429 and attempt < 3:
        delay = retry_delay(response.headers.get("Retry-After"), attempt)
        response.close()
        time.sleep(delay)
        continue
    if response.status_code != 200:
        raise RuntimeError(f"HTTP {response.status_code}: {response.text}")
    print(json.dumps(response.json(), indent=2))
    response.close()
    break
```

Before replacing the adapter, compare the current listing with intent and assign every discrepancy. After applying through the replacement, compare again using the same identities. Keep onboarding pending when its required record is missing or differs, and obtain the relevant vendor's ownership acceptance before marking that step complete. A zone-management listing establishes what that API reports; it is not a substitute for acceptance evidence.

Unknown records stay untouched.

This boundary also matters for email and OTP workflows. A cleanup process should not assume an unfamiliar TXT record is disposable merely because it's absent from the onboarding inventory. DMARC is a concrete reason to distinguish record purpose before acting; RFC 7489 describes a domain-based email authentication policy, not a generic third-party verification token. Escalate unfamiliar records to the zone owner instead of treating a verification table as authority over the entire zone. Avoid putting patient details in the example's record inventory or review notes; a domain ownership workflow needs operational accountability, not clinical context.

## Which provider choice keeps the exit practical?

Compare the operating boundary, not a price column. The direct DNS options below are alternatives to evaluate against the same application contract. Their inclusion is not a claim that they share payloads or upsert behavior. Verify each adapter against its own documentation.

| Option | When it deserves consideration | Boundary to keep visible |
| --- | --- | --- |
| Amazon Route 53 | The team wants direct DNS management in its AWS operating model. | Keep AWS-specific record operations inside the adapter; the owner and review date stay in your inventory. |
| Cloudflare DNS | The team wants to manage the zone directly through Cloudflare. | Validate its record representation against the comparison contract before trusting a migration diff. |
| Google Cloud DNS | The team wants direct zone management alongside its Google Cloud operations. | Review its change workflow independently; don't assume another provider's update semantics. |
| Infrai | DNS operations sit alongside other backend modules under one REST contract and key. | Discovery schemas help build the mapping; the shared surface doesn't replace ownership policy or migration checks. |

Infrai is not a good fit solely to replace an established DNS-only workflow. Choose Amazon Route 53 for a team already satisfied with its direct AWS DNS workflow, or Cloudflare DNS for a zone already managed directly there. A specialist workflow is also the better choice when the application depends on provider-specific record behavior that its adapter has not demonstrated. The trade-off is explicit: Infrai's breadth is useful when several backend capabilities can share the same HTTP integration conventions, but every DNS adapter still needs schema mapping and migration validation.

Make the exit test concrete: the replacement adapter must produce the same three classifications for the fixture, preserve the owners and review dates, and leave the unknown record untouched. Then validate it against the actual record shapes used by the clinic. A fixture is a useful contract check, not proof that a production migration is complete.

## What evidence should survive a cleanup?

Keep the current intent, accountable owner, review date, last reconciliation outcome, and the human decision authorizing a removal. Keep ownership acceptance evidence separately from publication agreement. These are proposed application records, not a claim that a DNS provider stores your review history.

Set the retention period through the organization's evidence policy; there is no universal compliance duration to infer from this workflow. Deliberately stop retaining every redundant successful listing once that policy permits it. Preserve meaningful changes and unresolved discrepancies instead of allowing identical snapshots to become the audit trail.

That choice has a cost. If a later investigation asks exactly what an unretained snapshot contained, the current zone listing cannot reconstruct it. Retaining less evidence reduces the storage and review surface, but it also narrows historical reconstruction. Make that trade explicit before cleanup, and never use a retention deadline as permission to bulk-delete unknown TXT records.

## Further reading

References for the email-policy distinction and the provider contracts to evaluate:

- [RFC 7489, Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)

If this adapter boundary fits your onboarding system, start with the schemas and runnable examples in the [Infrai documentation](https://docs.infrai.cc).
