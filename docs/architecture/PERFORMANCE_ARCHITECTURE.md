# Performance Architecture — Performance Architect Level

This document describes the **performance strategies** implemented and planned for the three flagship modules: **PDF to Anki**, **Receipt Parser**, and **Audio Notes**. It aligns with the "Performance Architect" narrative: system and compute optimization, FinOps, and resilience.

---
 
## 1. Project 3: PDF to Anki — The Async Knowledge Beast

**Evolution:** From "Upload a PDF" to **Resilient Mass Ingest System**.

### 1.1 Strategy: Async & Queues + Semantic / Exact Hash Caching

| Pain | Solution |
|------|----------|
| "If I upload the BOE (500 pages), your API blows up." | Decoupled architecture: **Arq + Redis**. Upload returns 202; processing runs in a background worker. No long HTTP timeout. |
| Same PDF uploaded twice (e.g. same textbook) | **Exact hash cache**: `file_hash = SHA-256(pdf_bytes)`. If `pdf:deck:{file_hash}` exists in Redis, the worker returns the cached deck in ~50 ms and does not call LlamaParse/LLM. |

**Stack (actual):** FastAPI + **Arq** (not Celery) + Redis. Router: LiteLLM. Vector store: LanceDB (S3/R2).

### 1.2 Implemented Behaviour

- **Async flow:** Upload → store **encrypted** raw bytes in Redis (TTL 1h) → enqueue `process_pdf_task` → worker: load PDF from Redis → decrypt → LlamaParse → chunk → embed → LanceDB → flashcard generation → store **encrypted** deck in Redis (`pdf:deck:{file_hash}`, TTL 7 days).
- **Cache hit L1 (exact hash):** Before parsing, worker calls `cache_get(file_hash)`. If a deck exists for that hash, state is set to COMPLETED and the cached deck is returned; no LlamaParse nor LLM.
- **Semantic dedup L2:** Each new chunk embedding is searched in LanceDB; if a highly similar concept already exists (≥95% similarity), LLM generation for that chunk is skipped (FinOps optimisation).
- **Idempotency:** Duplicate uploads (same hash) are detected in the router via `get_by_file_hash`; client gets the same `task_id` or "already queued".

### 1.3 Token Optimization

- **Goal:** Reduce cost and latency by not sending headers/footers (page numbers, "Page X of Y", repeated titles) to the LLM.
- **Implementation:** Pure helper `strip_headers_footers(text)` in `app.modules.pdf_anki.application.token_optimizer`. Applied to chunk text **before** sending to the flashcard LLM (see `FlashcardGeneratorAdapter`). Embeddings can still use full chunk for better retrieval; only the flashcard prompt sees trimmed text.

### 1.4 Semantic Caching (Phase 1 Implemented)

- **Current:** Exact hash cache (L1) + per-chunk **semantic deduplication** (L2) in `process_pdf_use_case`. This avoids re-paying for repeated explanations across documents.
- **Next:** Deck-level semantic cache (full-document reuse) and tenant-aware keys; may be implemented as a follow-up ADR.

### 1.5 Nested Learning — The Adaptive Teacher (Roadmap)

- **Idea:** User corrects a flashcard → system stores the correction as a positive example (e.g. in LanceDB or a feedback table). Next time we generate cards for that user/topic, inject few-shot examples (e.g. via DSPy BootstrapFewShot or a simple retrieval step) so the model adapts to the user’s style.
- **Status:** Design only; not implemented. Value: "The system learns the user’s pedagogical style; the more they use it, the fewer corrections they need."

### 1.6 Tests

- Unit: `process_pdf_use_case` with `cache_get` returning a deck → status completed, no parser/vector_store calls.
- Unit: `strip_headers_footers` removes common patterns.
- Integration: `test_pdf_resilience`: timeout/error → state FAILED.
- Worker: `process_pdf_task` with mocked Redis cache returning deck → document COMPLETED, deck from cache.

---

## 2. Project 5: Receipt Parser — The Cost-Efficient Fortress

**Architecture:** Hybrid — Edge (OCR) + Cloud (orchestration).

### 2.1 Value Proposition

"This system is not just a receipt reader. It is a **hybrid AI architecture** for maximum privacy and minimum cost: we process vision locally (CPU) to remove image transfer and cloud vision cost; we use **intelligent LLM orchestration** to structure data, achieving high accuracy at a fraction of traditional cost."

### 2.2 Current Implementation (Cloud Orchestration)

- **Input:** Raw OCR text (from Edge or external OCR). We do **not** process images in the backend today; the "compute" layer is at the edge.
- **FinOps — LLM router:**
  - **Level 1 (cheap):** Groq Llama-3.1-8B. Prompt: extract JSON (items, subtotal, tax, total, currency). Cost ~$0.0001 per receipt.
  - **Validation (pure Python):** `validate_receipt_math`: items sum = subtotal, subtotal + tax = total. No framework.
  - **Escalation:** If validation fails after retries (e.g. 2× Llama, then 1× GPT-4o), we call **GPT-4o** with an error-correcting prompt. Only ~5–10% of receipts escalate; cost remains low overall.

### 2.3 Edge / Compute Performance (Roadmap)

- **Goal:** Remove dependency on GPT-4 Vision; keep images on-prem or on Edge.
- **Techniques (when we add image input):**
  - **Preprocessing (OpenCV/NumPy):** Perspective correction (homography), adaptive thresholding (Otsu), noise reduction (morphological ops). Pure math on CPU; improves OCR accuracy.
  - **Local OCR:** PaddleOCR or Tesseract exported to **ONNX**, optionally **quantized (int8)** for 3–4× speed and lower RAM.
  - **Singleton model:** Load ONNX model once at FastAPI startup; reuse for every request.

### 2.4 Railway / Low-Memory Rules (when doing image processing)

- **Downsampling:** Resize image to max 1024 px width before heavy ops (e.g. 100 MB → 10 MB RAM).
- **Singleton:** One ONNX model instance per process.
- **Optional GC:** After processing a large image, `del` reference and `gc.collect()` to free RAM sooner.

### 2.5 Dynamic Few-Shot Retrieval (Roadmap)

- **Idea:** Before calling the LLM, search a vector store for **validated receipts from the same vendor** (e.g. "Ferretería Pepe"). If found, inject 1–2 examples into the prompt. The model then "learns" that vendor’s format without retraining.
- **Status:** Design only; requires vendor extraction + vector index of validated receipts.

### 2.6 Tests

- Unit: `validate_receipt_math` (valid / invalid totals).
- Unit: `run_self_correction_loop`: invalid JSON → error; valid JSON but wrong math → retries then smart model; valid JSON + correct math → success.
- Evaluation: `test_receipt_extraction.py` (DeepEval-style; mock and optional live LLM).

---

## 3. Project 2: Audio Notes — The Smart Listener

**Evolution:** From "Transcribe audio" to **Corporate Memory**.

### 3.1 Strategy: Hash-Based Cache (System Performance)

| Pain | Solution |
|------|----------|
| Five employees upload the same Zoom recording; you pay 5× for transcription and summary. | **Content hash (SHA-256)** of the audio. Key: `audio:{hash}`. If cache hit → return stored summary in ~50 ms; cost $0. |

### 3.2 Implemented Behaviour

- **Flow:** Upload → compute content hash → `idempotency.get_cached(hash)`.
  - **Hit:** Return `status: "cached"` and stored summary; no Whisper, no LLM.
  - **Miss:** Transcribe (Whisper) → summarize (LLM) → store in cache → return.
- **Idempotency:** `RedisIdempotencyGuard` with TTL; same file always yields same result and one processing run.

### 3.3 Tests

- Unit: Idempotency hash stable; get/set cached.
- Integration: Upload same file twice → second request cached.

---

## 4. Native and Accelerated Code (Strategy)

When **pure Python** is the bottleneck (e.g. heavy math, image ops, embedding loops), consider, in order:

1. **Profile first:** Identify hot paths (e.g. with `cProfile` or PySpy).
2. **Python-level:** `asyncio.to_thread()` for CPU-bound work; batch APIs to reduce round-trips.
3. **NumPy/SciPy:** Vectorise; avoid Python loops where possible.
4. **Native / compiled:**
   - **OpenCV (C++):** Already used via Python bindings; for new pipelines, C++ extensions or cv2 is sufficient unless you need custom kernels.
   - **ONNX Runtime:** For inference (OCR, classifiers); int8 quantization for speed and memory.
   - **Rust/PyO3:** For new pure-math or I/O-heavy libraries; call from Python via PyO3.
   - **CUDA/Triton:** For GPU-bound inference or custom kernels (e.g. embedding batches).
   - **WASM:** For client-side or sandboxed compute (e.g. browser); not typically for backend.
5. **Infrastructure:** **Kafka** (or Redis Streams, as we use) for event-driven, back-pressure-safe pipelines; **DB indexing** (e.g. on `file_hash`, `tenant_id`, time) for fast lookups; **load balancers** and horizontal scaling for API and workers.

**Rule:** Prefer Python + NumPy + ONNX for most cases; introduce C++/Rust/CUDA only when profiling shows a clear win and maintenance cost is acceptable. Document the decision in an ADR.

---

## 5. References

- [SYSTEM_TOPOLOGY.md](SYSTEM_TOPOLOGY.md) — High-level architecture.
-[docs/HEXAGONAL_DECOUPLING.md](HEXAGONAL_DECOUPLING.md) — Layer boundaries.
- [docs/adr/0008-native-acceleration-strategy.md](../adr/0008-native-acceleration-strategy.md) — When to use ONNX, Rust, CUDA, etc.
