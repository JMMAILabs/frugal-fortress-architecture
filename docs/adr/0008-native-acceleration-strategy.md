# ADR 0008: Strategy for Native / Accelerated Code

Date: 2026-03-08

## Status

Accepted

## Context

Performance-critical paths (Ej: embedding batches, receipt math validation) may benefit from code that is not pure Python: C++, Rust, ONNX, CUDA, or Triton. We need a clear strategy so that we do not over-invest in native code when Python is sufficient, and we know when to consider each option.

## Decision

1. **Default:** Implement in **Python** (with NumPy where applicable). Use **asyncio.to_thread()** for CPU-bound work so the event loop is not blocked.
2. **Profile first:** Before introducing native/accelerated code, **profile** (e.g. cProfile, PySpy) to confirm the hotspot and that the gain justifies the maintenance cost.
3. **When to consider alternatives:**
   - **ONNX Runtime:** For **inference** (OCR, classifiers, embeddings). Prefer existing ONNX models (e.g. PaddleOCR, sentence-transformers export); use **quantization (int8)** for lower memory and higher throughput on CPU. Load model **once per process** (singleton at startup).
   - **OpenCV (C++/Python bindings):** For **image preprocessing** (homography, thresholding, morphological ops). Use the existing cv2 Python API unless we hit a proven bottleneck; then consider a C++ extension only if profiling shows it.
   - **Rust (PyO3):** For **new libraries** that are pure computation or I/O-heavy and where Python is too slow. Prefer a small, well-scoped crate with a clear Python API.
   - **CUDA / Triton:** For **GPU-bound** inference or custom kernels (e.g. large embedding batches, training). Only if we have GPU capacity and the workload justifies it.
   - **WASM:** For **client-side** or sandboxed compute (e.g. browser); not for backend server hot paths.
4. **Infrastructure:** Use **Redis Streams** (or Kafka if we scale out) for event-driven pipelines and back-pressure. Use **DB indexing** (e.g. on `file_hash`, `tenant_id`, time columns) for fast lookups. **Load balancers** and horizontal scaling for API and workers; avoid relying on a single process for all CPU-heavy work.

## Consequences

- We avoid premature optimization while keeping the door open for ONNX, OpenCV, and Rust where profiling shows clear benefit.
- New native code must be documented (when, why, how to build/test) and have tests. Prefer pure Python tests that assert behaviour; use integration tests for ONNX/native paths.
- Receipt **math validation** stays in **pure Python** (Pydantic + Decimal); it is fast enough and keeps the domain portable.
