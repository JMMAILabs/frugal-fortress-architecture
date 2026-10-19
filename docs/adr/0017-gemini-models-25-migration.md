# ADR 0017: Migrate Gemini 2.5 Models to Gemini 3
**Date:** 2026-10-19
**Status:** Proposed

## Context

Google Vertex AI's published migration guidance recommends moving the Gemini 2.5 lineup before its scheduled retirement. The migration targets from the supplied Vertex AI announcement are:

| Current model | Replacement |
| --- | --- |
| Gemini 2.5 Flash-Lite | Gemini 3.1 Flash-Lite |
| Gemini 2.5 Flash | Gemini 3.5 Flash-Lite |
| Gemini 2.5 Pro | Gemini 3.8 Flash |

The backend uses these model classes in default fast/smart routing and in VERA, KERA, and AURA. An exploratory paired benchmark compared Gemini 2.5 Pro and Gemini 3.8 Flash:

* KERA: three PDFs (History of Rome ESP-1, Introduction to Mitochondria and Cellular Respiration, and US Constitution).
* VERA: one synthetic receipt PDF.
* AURA: one synthetic Spanish audio clip.

All tested calls returned usable structured output. Gemini 3.8 Flash was faster in the VERA and AURA samples and on two of three KERA documents; it was slower on the US Constitution PDF while producing more cards. Results are one execution per case, not a statistical or human quality evaluation. KERA costs were recalculated with one shared reference token-rate source, not application prices or invoiced Vertex usage.

The official model pages list 1,048,576-token input context for the new models and mark 3.1 Flash-Lite and 3.5 Flash-Lite as GA. The 3.5 Flash-Lite page identifies it as a suitable replacement for Gemini 2.5 Flash. The benchmark project reached Gemini 3.8 Flash at Vertex location `global`; its `us-central1` endpoint returned not found. Google documents that custom temperature/top-k/top-p values are ignored for Gemini 3.5 Flash-Lite and that custom frequency/presence penalties are unsupported.

## Decision

1. Replace Gemini 2.5 model IDs throughout runtime inference paths:
   * `gemini/gemini-2.5-flash-lite` → `gemini/gemini-3.1-flash-lite`.
   * `gemini/gemini-2.5-flash` → `gemini/gemini-3.5-flash-lite`.
   * `gemini/gemini-2.5-pro` → `gemini/gemini-3.8-flash`.
2. Apply this mapping to general fast/smart defaults, classifier/query expansion, VERA Premium/Pro/Ultra and PAYG, KERA Premium/Pro and PAYG, and AURA's included model and PAYG selectable list.
3. Route the three replacement model IDs to Vertex location `global`; leave unrelated models on `VERTEX_LOCATION`.
4. Set each replacement context limit to 1,048,576. Give 3.5 Flash-Lite and 3.8 Flash the extended KERA timeout allowance previously reserved for Flash and Pro workloads.
5. Preserve stored/in-flight old model IDs by mapping them to their replacement at VERA, KERA upload, and AURA model-selection boundaries. Do not rewrite persisted user data during migration.
6. Add non-zero FinOps reference rates, in USD per million tokens:
   * Gemini 3.1 Flash-Lite: `$0.25` input / `$1.50` output / `$0.50` audio-token rate.
   * Gemini 3.5 Flash-Lite: `$0.30` input / `$2.50` output / `$0.30` audio-token estimate.
   * Gemini 3.8 Flash: `$0.75` input / `$3.75` output / `$0.75` audio-token estimate.

   The text rates are aligned with the installed LiteLLM model catalog; the audio rates are catalog references/estimates. These are not confirmed project-specific Vertex invoice rates. Verify current prices per modality before production release.
7. Keep the change behind branch and pull-request review. This ADR does not authorize production deployment.

## Alternatives considered

* Migrate only Gemini 2.5 Pro: rejected because the 2.5 Flash-Lite and Flash models are also retiring and need replacements.
* Replace every old model with Gemini 3.8 Flash: rejected because it discards the intended cost/latency tiers and the 3.1/3.5 model-specific migration recommendations.
* Change the Vertex location globally: rejected because model availability differs; a per-model location override avoids changing unrelated traffic.
* Set missing audio prices to zero: rejected because AURA could treat paid audio inference as free or compute incorrect wallet limits.

## Consequences

### Positive

* No runtime tier or default model intentionally selects the retiring Gemini 2.5 family.
* The existing fast / smart / high-capability cost tiers are retained through the requested mapping.
* Existing persisted choices and stale KERA forms continue through a compatible alias to the replacement model.
* Model-specific global routing avoids the known location failure for the tested project.

### Risks and follow-up

* The 3.1/3.5 models were not included in the initial live benchmark. Their route, response format, quality, latency, and audio billing must be verified in staging with repeated module-specific cases.
* The initial benchmark has a very small sample and no human quality scoring or adversarial/large-document evaluation. Do not claim parity from these results alone.
* Verify all reference rates against current Vertex billing for text, image, and audio; adjust the runtime snapshots and customer pricing before production.
* Confirm the production LiteLLM version supports all three model IDs with Vertex `global` for text and multimodal calls.
* `temperature` is ignored by Gemini 3 models; avoid unsupported penalty parameters and validate provider-specific generation kwargs.
* Pricing pages and model labels in the separate frontend repository and model references in the separate documentation project must be updated before those projects are released.

## References

* [Gemini 3.1 Flash-Lite model page](https://cloud.google.com/vertex-ai/generative-ai/docs/models/gemini/3-1-flash-lite)
* [Gemini 3.5 Flash-Lite model page](https://cloud.google.com/vertex-ai/generative-ai/docs/models/gemini/3-5-flash-lite)
* [Gemini 3.8 Flash model page](https://cloud.google.com/vertex-ai/generative-ai/docs/models/gemini/3-8-flash)
* [Gemini 3.5 Flash-Lite migration guidance](https://cloud.google.com/vertex-ai/generative-ai/docs/models/gemini/3-5-flash-lite)
* [Gemini 3.8 Flash guide](https://cloud.google.com/vertex-ai/generative-ai/docs/models/guides/gemini-3-8-flash)
* [Vertex AI generative AI pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing)
