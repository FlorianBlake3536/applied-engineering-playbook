# How to Verify Staging DNS Write Boundaries — 3 Zone Checks

A separate staging DNS zone adds a delegation to maintain, but it gives an edtech team a useful place to restrict changes to school mail routing. TL;DR: use a delegated staging zone with credentials restricted to that zone when staging automation must not edit production MX records. A subdomain name alone does not establish that write boundary. Compare approved intent with published records before declaring the mail configuration ready.

The constraint is concrete: a rehearsal for moving company mail to a provider must not let a staging deployment redirect production mail. For a school service that also sends sign-in codes, mail routing and message authentication deserve separate checks. Passing an MX comparison does not prove that an OTP will reach an inbox.

This note uses an illustrative school domain, `academy.example`, and a staging mail domain, `stage.academy.example`. The three checks below are local Python examples with explicit fixtures. Replace those fixtures with reviewed inventory and DNS observations in the deployment pipeline; they are not live DNS probes.

## What actually separates a staging zone from a production subdomain?

A subdomain describes a position in the DNS namespace. A zone describes an administrative portion of that namespace. Under the delegation model in RFC 1034, a parent can delegate a child to its authoritative servers. Thus `stage.academy.example` can be both a production subdomain and a separately administered zone. The practical choice is between keeping its records in the parent zone and delegating it to a child zone.

Delegation creates a DNS authority boundary. Your credential policy must establish the corresponding write boundary: a staging identity should be able to change the child records while lacking permission to change the parent, its delegation, or production mail records. A separate zone still offers little containment if the same deployment credential can edit both zones. That permission behavior must be verified against the actual DNS control interface.

Names aren't permissions.

There is also a mail-specific edge. RFC 7489 defines DMARC policy discovery and alignment; absent a policy at the relevant subdomain, discovery can reach the Organizational Domain. Its `sp` tag provides a subdomain policy, with `p` supplying the policy when `sp` is absent. A DNS zone cut does not by itself create an independent DMARC policy or alignment boundary. Treat staging sender identities and authentication policy as explicit review items, even after zone isolation is complete.

## 1. Check the change boundary before touching routing

Start with an approved change envelope. The following fixture allows only staging MX changes in a delegated child zone. A production owner name must fail this check, regardless of whether its proposed MX target looks reasonable. Run this with Python 3; successful execution prints the one allowed change.

```python
zone = "stage.academy.example"
allowed_owners = {"stage.academy.example"}
changes = [
    {"owner": "stage.academy.example", "type": "MX"},
]

for change in changes:
    owner = change["owner"].rstrip(".").lower()
    inside_zone = owner == zone or owner.endswith("." + zone)
    if not inside_zone or owner not in allowed_owners:
        raise SystemExit("Rejected: owner outside approved mail scope")
    if change["type"] != "MX":
        raise SystemExit("Rejected: record type outside approved mail scope")

print("Approved staging MX changes:", len(changes))
```

This guard catches an incorrectly constructed plan. It cannot restrict a credential. Verify the deployed identity's permissions separately, and make the parent-zone delegation a production-controlled change. Keep unrelated authentication records outside this narrow MX envelope so a routing rehearsal does not quietly become a sender-policy rewrite.

The distinction matters during rollback. If a job can restore a production zone snapshot, limiting its normal change plan to staging records has not contained its effective authority. Review recovery permissions along with deployment permissions.

## 2. Compare mail intent with published evidence

Write down the complete intended MX set, including preference values. RFC 1035 defines the preference and exchange fields; RFC 5321 describes how senders use MX records, preferring lower preference values. An old exchange left alongside the intended records is therefore a routing difference, not harmless historical clutter.

This example compares complete record sets rather than asking whether the desired target appears anywhere. Its published fixture deliberately contains an extra exchange. Execution must report drift and exit unsuccessfully; removing the extra tuple makes it pass. The domain names and preference values are illustrative, not a provider's setup instructions.

```python
import json

intent = [(10, "mx.mail.example.")]
published = [
    (10, "mx.mail.example."),
    (20, "old.mail.example."),
]

def normalize(records):
    return {(preference, host.rstrip(".").lower())
            for preference, host in records}

expected = normalize(intent)
observed = normalize(published)
report = {
    "missing": sorted(expected - observed),
    "unexpected": sorted(observed - expected),
}
print(json.dumps(report, indent=2))
if expected != observed:
    raise SystemExit("MX drift: hold rollout")
```

For a deployment, collect the observed set from the authoritative servers for the affected zone and retain the query time, server, owner, record type, and answer. Compare each observation with the same approved intent. A timeout is missing evidence; do not turn it into an empty record set and issue a corrective delete. That error would convert an observation failure into a routing change.

Then check recursive observations as a separate stage. DNS caching means an authoritative change and a cached answer can differ while earlier data remains cached under its TTL. A useful decision record distinguishes authoritative disagreement from cached older data instead of labeling both simply as propagation. Keep the last known good configuration until the observation explains the difference.

Log DNS evidence.

Keep student addresses and OTP values out of these reports; neither is needed to compare MX sets. For delivery testing, use designated test recipients and record only the evidence needed to establish the result.

Now the zone choice has a concrete comparison axis:

| Arrangement | What the mail review must establish | Operational trade-off |
| --- | --- | --- |
| Staging records inside the parent zone | Actual permissions prevent production edits | Fewer delegations, but narrower access depends on the control interface |
| Delegated staging child zone | Staging credentials cannot edit the parent or its delegation | A distinct authority boundary, plus delegation maintenance |
| Separate staging domain | Credentials and mail identities are independently scoped | Additional domain and authentication configuration to maintain |

A separate domain can change the mail-identity relationship as well as the namespace. Its usefulness depends on what the rehearsal must reproduce. For a routing-only rehearsal, a delegated child with restricted credentials can provide the needed DNS containment; authentication acceptance still requires its own evidence.

## 3. Gate the rollout and preserve a recovery path

Before changing production MX records, rehearse the intended configuration in staging, save the approved production record set, and require an explicit production change authorization. Use observed TTLs to plan cache overlap. A shorter TTL published immediately before a cutover does not retroactively shorten the lifetime of answers already cached.

The final check below models a release decision. Every value starts false so absence of evidence holds the change. Populate these fields from independently collected results, and require a successful execution before applying the production plan.

```python
evidence = {
    "staging_scope_verified": False,
    "authoritative_mx_matches": False,
    "cache_overlap_reviewed": False,
    "mail_authentication_reviewed": False,
    "test_delivery_confirmed": False,
    "previous_records_saved": False,
    "production_change_authorized": False,
}
missing = [name for name, passed in evidence.items() if passed is not True]
if missing:
    raise SystemExit("Hold rollout: " + ", ".join(missing))
print("Evidence complete for the approved production plan")
```

Restoring old records is a new DNS change, and cached answers can outlive that restoration too. Document who can authorize recovery and what observations will trigger it. Avoid a universal waiting period: the relevant TTLs and actual resolver observations determine the evidence you need.

Finally, keep the acceptance claims narrow. Matching MX records establishes agreement between routing intent and observed DNS. Authentication review establishes a different part of the mail setup. A test delivery adds delivery evidence, but does not promise every future OTP will arrive. Choose the staging boundary that constrains writes, then promote only the configuration whose evidence matches the approved plan.

## References

The following standards describe the DNS and mail behavior used in these checks.

## Sources

- RFC 1034, Domain Names: Concepts and Facilities, including zones, delegation, and caching: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 1035, Domain Names: Implementation and Specification, including MX record fields: https://datatracker.ietf.org/doc/html/rfc1035
- RFC 5321, Simple Mail Transfer Protocol, including mail destination resolution: https://datatracker.ietf.org/doc/html/rfc5321
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance, including policy discovery and subdomain policy: https://datatracker.ietf.org/doc/html/rfc7489
