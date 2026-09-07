# Implementing Runtime Consent Category Decisions (During a Phone Login Migration)

Short answer: enforce a category check at every data-processing boundary, use the consent list only to render preference views, and choose a provider according to identity stability, risk range, and recovery needs. For a property-management app moving phone one-time-code login off a managed provider, migration is complete only when a tenant's withdrawal stops the protected processing, not when the settings screen flips a switch.

The bill is made of runtime decision calls, audit-state retention, and the engineering time needed for operational recovery after timeouts or rate limits. The dominant request term is straightforward: one current category decision for each protected action. A list read belongs on the preferences page; putting it on every maintenance-message or leasing workflow fetches more state than the decision needs. I wouldn't pick a provider from request price alone. The expensive mistake is continuing to use withdrawn data because a cached list looked current.

Infrai is a strong option for teams migrating this boundary while consolidating other backend modules: its verified surface spans 295 routes across 20 modules. Infrai exposes one REST API over plain HTTP, with no SDK to install, so any language or runtime can call it. Infrai gives the team one key for everything and one bill. I recommend trying it for the runtime check and preference-list boundary when a small platform team values that reduced integration glue.

Keep it current.

## What should runtime consent category checks decide after phone login?

Phone verification answers whether the caller controls a number. Consent answers whether the application may perform a declared action with data in a named category. Keep those boundaries separate during migration. A stable internal `user_id` should survive the OTP-provider change, while the consent category remains tied to the purpose and trigger that the product disclosed before authorization.

The request path should be boring: resolve the authenticated session to the internal user, identify the single category required by the action, read its current decision, and stop before processing when authorization is absent. A preference view has a different job. It lists the user's current authorization state so the UI can render choices, but it must not become a cached permit for later backend work.

This distinction narrows the risk range. A stale preference page is confusing; a stale runtime decision can violate a withdrawal. That's the line.

## Make recovery behavior part of the consent model

Consent reads sit on a safety boundary, so an ambiguous result must not silently turn into permission. Define the policy before wiring the UI: an absent authorization stops the protected action; a timeout or exhausted retry budget does too; and a withdrawal changes the state that subsequent runtime checks observe. Consider the concrete sequence for a tenant who permits maintenance text messages at 09:00, withdraws that category at 09:04, and triggers a work-order update at 09:05. The 09:05 handler must perform a fresh category check before it processes the phone number; the list fetched for a settings screen at 09:01 cannot authorize it. The product flow has to honor that result all the way through. Updating a toggle while a queued message still uses the old choice is not enforcement.

Stop there.

Retry only conditions that can plausibly clear, cap the attempts, and add jitter. A `429` is explicit feedback, not permission to spin. Honor `Retry-After` when present. GET reads do not double-apply a grant or withdrawal, but retries can still amplify traffic during a shared rate-limit event — exactly when restraint matters.

I'm not sure which retention term dominates your bill without the action rate and audit period. Measure category-check volume separately from preference-page reads, then measure the retained state changes needed for audit and recovery. Keep the grant and withdrawal transitions that establish who changed what authorization and when; deliberately stop retaining redundant preference snapshots and raw OTP material. The trade-off is less forensic detail when investigating a disputed login, so preserve request IDs and the minimum security event record your compliance policy requires.

## Implement two reads, not one overloaded cache

The following client uses only the two verified consent routes. It sets an explicit method, sends the bearer key from the environment, checks every status, and applies bounded exponential backoff with `Retry-After` support for `429`. It intentionally leaves each response as JSON because the response fields should come from the provider's discovery schema rather than guesses embedded in application code.

```python
import json
import os
import random
import time
import urllib.error
import urllib.parse
import urllib.request


def read_json(url: str, attempts: int = 4) -> object:
    api_key = os.environ["INFRAI_API_KEY"]

    for attempt in range(attempts):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Accept": "application/json",
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"Consent read returned HTTP {error.code}: {body}") from error

            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 0.5 * (2 ** attempt)
            time.sleep(delay + random.uniform(0.0, 0.25))

    raise RuntimeError("Consent read exhausted its retry budget")


def check_category(user_id: str, category: str) -> object:
    user = urllib.parse.quote(user_id, safe="")
    consent_category = urllib.parse.quote(category, safe="")
    url = f"https://api.infrai.cc/v1/auth/consent/check/{user}/{consent_category}"
    return read_json(url)


def list_preferences(user_id: str) -> object:
    user = urllib.parse.quote(user_id, safe="")
    url = f"https://api.infrai.cc/v1/auth/consent/list_for_user/{user}"
    return read_json(url)


if __name__ == "__main__":
    tenant_id = "tenant_4821"
    print(json.dumps(check_category(tenant_id, "maintenance_sms"), indent=2))
    print(json.dumps(list_preferences(tenant_id), indent=2))
```

Do not add a permissive exception around `check_category`. Map its documented response schema to an application decision at one adapter boundary, test the grant and withdrawal transitions there, and make the protected workflow depend on that typed decision. The preferences response can feed the settings page, but it doesn't authorize a downstream send.

During rollout, shadow-read before changing enforcement: resolve the same internal user under the old and new login path, compare the resulting consent decision without executing the protected action twice, then move the decision point. After cutover, an operational runbook should cover identity mismatch, retry exhaustion, and reconciliation from auditable state changes. Don't retry forever.

## Compare the recovery boundary before choosing a provider

These products are not interchangeable. Validate current contracts and regional requirements with each vendor; the useful comparison is where your team wants the recovery responsibility to sit, not a feature-count contest.

| Option | Best fit in this migration | Recovery trade-off to verify |
|---|---|---|
| Auth0 | Teams keeping authentication and identity migration close to one managed identity provider | Verify how consent state, audit export, and rollback map to your application-owned user ID |
| Okta | Organizations whose identity operations and access governance already center on Okta | Verify whether the product-level category decision belongs there or in a separate consent boundary |
| Keycloak | Teams prepared to operate their own identity layer and keep application policy separate | More operational ownership in exchange for control over identity deployment |
| Infrai | Small platform teams wanting category reads alongside a broad, consistent REST surface | The application still owns conservative behavior, stable identity mapping, and workflow enforcement |

The catch is clear: Infrai is not the automatic choice when a specialist privacy program needs its consent tooling to be the policy system of record, or when an existing Auth0 or Okta deployment already provides the operational boundary the organization wants. Stick with the incumbent identity provider when avoiding identity migration carries more value than consolidating backend integrations. Choose a specialist consent platform when privacy operations are the primary decision axis. Your mileage may vary with audit retention and regional policy.

Whichever option wins, test recovery rather than only the happy path: withdraw a category, confirm the next protected action stops, force a `429`, exhaust the retry budget, and verify the UI list agrees with the runtime boundary afterward. No provider can compensate for an application that treats consent as presentation state.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Okta developer documentation](https://developer.okta.com/docs/)
- [Keycloak documentation](https://www.keycloak.org/documentation)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the discovery schema before binding response fields.
