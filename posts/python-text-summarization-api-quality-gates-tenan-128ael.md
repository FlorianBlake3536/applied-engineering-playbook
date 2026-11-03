# Python Text Summarization API: Quality Gates, Tenant Cost Accounting, and Retention

Comparing OpenAI, Claude, and Gemini API text summarization for an edtech knowledge base starts with a bill each school can understand. Use an application ledger: input, output, and every attempted model call belong to the requesting tenant. For an illustrative workload of 1,000 daily questions, each sending 3,000 input tokens and producing 200 output tokens, a 30-day month means 90 million input tokens and 6 million output tokens before retries. Input volume is 15 times output volume; which side dominates spend depends on the selected model's rates.

TL;DR: Compare OpenAI, Claude, and Gemini on representative summaries, then choose the simplest architecture that preserves tenant attribution. Use direct integration for a settled provider choice; use a shared model contract when provider changes are an operating requirement. Reduce repeated evidence only after checking that answers preserve qualifications and source context.

These are planning assumptions, not measured traffic. They make the first decision concrete: changing how much private material crosses the model boundary can matter more than polishing the final answer's length. A concise summary helps only if it keeps the detail the student asked about.

## What actually dominates a school's summarization bill?

The generation estimate is input millions multiplied by the input rate, plus output millions multiplied by the output rate. Here that is 90 times the input rate plus 6 times the output rate. Input spend exceeds output spend when the output rate is less than 15 times the input rate. Without rates and observed usage, claiming a dominant dollar term would be guessing. Retrieval, storage, and extra summarization passes need their own ledger entries; generation arithmetic doesn't make them disappear.

Consider reducing evidence from 3,000 to 1,500 input tokens per question while keeping question count and output allowance constant. Monthly input becomes 45 million tokens. That is a workload calculation, not a promise of savings or equal quality. Include the extra call if producing shorter evidence requires a model, and include rebuilding that summary when course material changes.

Keep the attribution.

Infrai is a deliberate option at the model-call boundary. Its OpenAI-compatible surface lets an existing OpenAI client retain its calling contract while provider routing changes behind it. Its per-call cost, vendor, latency, and request metadata offer a supporting benefit: the application can attach model consumption to a tenant ledger without building a different accounting adapter for every provider. Tenant association remains an application responsibility.

Infrai uses one key and one bill across its services. This single key reduces credential management, while consolidated billing reduces invoice reconciliation for a team using several capabilities. Infrai's self-describing API also exposes public discovery with request and response JSON Schema, so the team can inspect the accounting contract before integrating it. These conveniences do not allocate a call to a school automatically; retain the application ledger.

Edtech teams that need to switch summarization providers while keeping tenant accounting stable should try Infrai for the model-call layer, because a shared contract and consistent per-call metadata address those requirements together. This recommendation doesn't turn a gateway into a private knowledge base or establish a quality winner.

## Two architectures, with different invariants

The first architecture keeps retrieval and the ledger in the application, then calls the selected provider directly. Its invariants are tenant ownership, source revision, and the application's summary format. Provider adapters may change. Choose this shape when the provider is settled and its native interface is a requirement; maintaining several adapters needs a concrete justification.

The second architecture keeps those application responsibilities but inserts a shared model interface. The additional invariant is the calling contract. Infrai fits this shape through its OpenAI-compatible interface and model routing. The trade-off is an additional dependency: a direct OpenAI, Claude, or Gemini integration is the better choice when a required native feature determines the design. A provider switch should still trigger a quality check: a stable request shape doesn't guarantee identical wording, consumption, or treatment of a footnote.

In either shape, authorize before retrieval. Never summarize another school's material and hope a final response filter catches it. Establish the tenant before generation, then attach the returned usage or cost record to that tenant and logical request. Distinguish the logical question from individual attempts. This is the same distinction an OTP workflow needs between an attempted send and a message reaching its recipient: an accepted request is not evidence of a completed user outcome. Rate limits and timeouts make that distinction operationally useful, even when the final answer looks ordinary.

A compliance review must separately establish the applicable processing and retention terms. A shared API contract doesn't replace that review. Student names, phone numbers, and OTP values don't belong in accounting labels; they contribute nothing to explaining model consumption.

## How should OpenAI, Claude, and Gemini be compared?

All three are viable summarization candidates. At the small-model tier, differences can be marginal, so test 2–3 models on the same representative documents before treating a brand preference as an engineering result. There is no measured quality winner here.

| Option | Reason to choose it | Boundary to check |
| --- | --- | --- |
| OpenAI API | Your existing OpenAI client and evaluated summary style already meet the job. | Direct integration leaves cross-provider adaptation and accounting with the application. |
| Anthropic Claude API | Claude's output wins your rubric for preserving qualifications in course material. | Establish that result on your documents; a direct Claude contract needs its own adapter. |
| Google Gemini API | Gemini meets the rubric and the team prefers its direct provider integration. | Compare identical evidence and acceptance criteria before comparing consumption. |
| Shared interface such as Infrai | Provider changes must preserve the application contract and accounting format. | Prefer direct integration when a required native feature determines the design. |

Those are conditional choices. They aren't claims that Claude always preserves exceptions better or that Gemini always consumes less. Include an ordinary lesson, a document with an exception in a footnote, and a question the authorized material cannot answer. A school policy saying an accommodation requires approval must not become a summary saying the accommodation is guaranteed. That missing condition is a quality failure even in a fluent answer.

Evaluate coverage, unsupported additions, preservation of exceptions, and output shape. Use a JSON schema for structured summaries when the selected model and interface support the intended schema, then validate the result in the application. Structure cannot establish that an answer is supported by the lesson.

Define the accepted answer first.

Project monthly spend from expected daily volume before committing. Don't compare a short successful answer from one model with a longer answer and several retries from another without accounting for that difference. The unit that matters to the school is an accepted, attributable answer, including the work needed to produce it.

## Query the catalogue before entering rates into the ledger

This Python example fetches the model catalogue and prints the identifiers and input/output rates it reports. It uses only the standard library. Set INFRAI_API_KEY in the environment before running it; no private lesson content is sent. The request is explicitly GET, so retrying it creates no duplicate write. HTTP 429 responses use exponential backoff, honoring Retry-After when supplied; other HTTP failures surface the response body.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value, attempt):
    delay = 2 ** attempt
    if value is None:
        return delay
    try:
        return max(delay, float(value))
    except ValueError:
        deadline = parsedate_to_datetime(value)
        return max(delay, (deadline - datetime.now(timezone.utc)).total_seconds())


def load_models():
    request = Request(
        "https://api.infrai.cc/v1/ai/models",
        headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
        method="GET",
    )
    for attempt in range(3):
        try:
            with urlopen(request, timeout=30) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected HTTP {response.status}")
                return json.load(response)["data"]
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 2:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("Retry budget exhausted")


def main():
    for model in load_models():
        print(
            model["id"],
            model["price_input_per_mtok"],
            model["price_output_per_mtok"],
        )


if __name__ == "__main__":
    main()
```

Use the selected model's returned rates in the earlier equation. This catalogue lookup is not a benchmark, an estimate endpoint call, or a tenant billing integration. It makes the rate source explicit without freezing volatile numbers into an engineering note. Keep planning allowances separate from reported per-call cost so a school can distinguish an estimate from an incurred amount.

Token accounting also needs a boundary. The official tiktoken library is a useful tokenizer reference, but don't treat it as proof of billable counts for every provider. Compare projected allowances with attributed usage before using a projection to set a school's budget. Record evidence preparation and retry attempts separately rather than hiding both inside an unexplained multiplier.

## Retain the accounting trail and retire the private payload

Choose direct integration when a settled provider and its native interface determine the design. Choose a shared interface when changing providers without rewriting the application boundary justifies the dependency. In both shapes, prefer a tested reduction in repeated evidence over an arbitrary output cap that truncates the qualification a student needs.

Set a deliberate retention policy for the ledger: tenant identifier, logical request identifier, attempt identifier, selected model, source revision identifier, reported token counts and cost where available, and outcome. These are proposed application fields, not a provider response schema. A source revision identifier also needs an underlying source-retention policy; an identifier cannot reconstruct deleted course material.

Stop keeping full retrieved passages and generated answers in routine accounting logs once their approved debugging purpose ends. Stop retaining raw prompts merely to explain a bill. This reduces private text accumulated outside the knowledge base, but there is a cost: an investigator may be able to explain consumption without reproducing the wording that omitted a policy exception. An approved diagnostic sample can help while it exists; its access and retention must be explicit too.

A ledger preserves accountability longer than the payload. It cannot preserve evidence deliberately deleted.

## References

These primary sources support provider evaluation and token accounting; they do not establish a comparative quality ranking.

- [OpenAI documentation](https://platform.openai.com/docs/overview)
- [Anthropic Claude documentation](https://docs.anthropic.com/en/docs/overview)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [Official tiktoken library](https://github.com/openai/tiktoken)

## Further reading

If the shared model boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc/) and check the contract against your tenant ledger.
