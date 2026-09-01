# ADR 0015: Migrating Free Tiers to Groq GPT-OSS
**Date:** 2026-08-31
**Status:** Accepted

## Context
Groq decommissioned `llama-3.3-70b-versatile` on **2026-08-16** (announced 2026-06-17). It was the free-tier model for every module: AURA summaries, VERA receipt extraction, KERA flashcard generation and GenUI proposals. The vendor's notice named only that model; `llama-3.1-8b-instant`, our smaller fallback, was retired the same day and would have failed silently behind the primary.

This was not a migration we chose. After the shutdown date every free-tier generation returns `400 model_decommissioned`, so the only decision available was which replacement to adopt.

Groq recommended two: `openai/gpt-oss-120b` and `qwen/qwen3.6-27b`.

## Decision
**`groq/openai/gpt-oss-120b` as the primary across all four modules, with `groq/openai/gpt-oss-20b` as the fallback** (Groq's own recommended replacement for the retired 8B).

`qwen/qwen3.6-27b` was rejected on three grounds:
1. It is a **preview** model, removable without notice — precisely the risk that had just materialised, and unacceptable for a path that four modules depend on.
2. It caps completions at 16K versus 65K.
3. Its output costs $3.00/1M versus $0.60 — five times more.

### Free-tier envelope

| | llama-3.3-70b (retired) | openai/gpt-oss-120b |
|---|---|---|
| RPM / RPD | 30 / 1,000 | 30 / 1,000 |
| TPM / TPD | 12,000 / 100,000 | **8,000 / 200,000** |
| Context / max output | 131,072 / 32,768 | 131,072 / **65,536** |
| Price in/out per 1M | $0.59 / $0.79 | **$0.15 / $0.60** |

Daily throughput doubles; per-minute burst drops by a third, absorbed by the key rotation and TPD backoff already in place.

## Reasoning Parameters
gpt-oss reasons by default, which the retired model did not. `LiteLLMAdapter` injects two parameters for the family on all three completion paths:

* **`reasoning_effort`** (new setting `GROQ_REASONING_EFFORT`, default `low`). Reasoning tokens bill as output tokens and count against the free-tier TPD, so the depth is capped rather than left to the provider default.
* **`reasoning_format: hidden`**. Groq rejects `raw` alongside JSON mode — which AURA summaries and VERA extraction both use — and `raw` would leak `<think>` blocks into GenUI's `|||COMPONENT:|||` marker stream, breaking the parser.

Callers may override either value explicitly; the injection uses `setdefault`.

## Consequences
* **Positive:** Unit cost falls in all three metered modules. KERA drops from $5.90 to $2.50 per 10k pages, VERA from $4.00 to $3.00 per 10k receipts, AURA from $22.00 to $21.00 per 10k minutes.
* **Positive:** Daily free-tier capacity doubles, halving the likelihood of hitting the TPD wall.
* **Negative:** Per-minute burst capacity drops 33%. The KERA chunk loop goes from roughly five to three requests per minute.
* **Verified:** The free-tier chunked pipeline was exercised end to end — 7 flashcards from a single-chunk document in 34s, 618 prompt and 458 completion tokens. With `reasoning_effort=low` the reasoning overhead stayed bounded and JSON mode was unaffected. This mattered because KERA's free path was the only one of the four with no prior evidence under a reasoning model.
* **Operational:** The pricing override key derives from the model id, so it becomes `PRICE_GROQ_OPENAI_GPT_OSS_120B_*`. Any `PRICE_GROQ_LLAMA_3_3_70B_VERSATILE_*` set in a deploy environment must be renamed or it stops applying **silently**.
* **Unaffected:** `whisper-large-v3` was not part of the deprecation, so AURA transcription is unchanged.

## Lesson
A specific hosted model is a dependency with an expiry date, and the vendor sets it. What kept this migration bounded was that model ids live in constants and in `runtime_tunable*.json` rather than scattered through use cases — the blast radius was 12 source files, not a search across the codebase. The same discipline should apply to any future provider-pinned identifier.

The related queue failure uncovered while verifying this migration is recorded separately in [ADR 0016](0016-arq-queue-head-of-line-blocking.md).
