# ADR 0006. Pinning Runtime to Python 3.12 (LTS)

**Date:** 2026-01-31
**Status:** Accepted

## Context
We attempted to upgrade to Python 3.13. However, critical dependencies (`thinc`, required by `spacy`) do not yet provide pre-compiled binary wheels for Python 3.13 on Linux. This caused build failures requiring a C++ compiler (`g++`), which violates our security and performance standards for production images.

## Decision
1.  **Strict Pinning:** The entire stack (Dockerfile, pyproject.toml, CI/CD) is pinned to **Python 3.12**.
2.  **Update Strategy:** We perform library updates (`poetry update`) ONLY while pinned to Python 3.12 to ensure we download compatible binary wheels.
3.  **Future Upgrade:** We will migrate to Python 3.13 only when `scripts/check_py313_readiness.py` reports all dependencies as "READY".

## Consequences
*   **Stability:** Builds are deterministic and fast.
*   **Maintenance:** We can safely update libraries (like `litellm` and `spacy`) without breaking the build, as long as we stay on Python 3.12.
