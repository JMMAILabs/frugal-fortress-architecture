# Troubleshooting: Receipt processing (POST /receipts/process)

## Error: `receipt_process_unexpected_error` with "Name or service not known"

**Symptom:** The API returns 200 with `status: "unprocessable"` and `error_message: "[STUB] Receipt processing failed due to unexpected error..."`. In `logs/api.json` you see:

```text
ConnectError: [Errno -2] Name or service not known
```

and in the traceback: `host = 'api.llamaindex.ai'`, `url = 'https://api.llamaindex.ai/v1/upload'`.

**Cause:** The host where the backend runs **cannot resolve the DNS name `api.llamaindex.ai`**. The API keys (e.g. `LLAMAPARSE_RECEIPT_PARSER_API_KEY`, `GROQ_RECEIPT_PARSER_API_KEY`) are fine; the failure is **network/DNS** before any HTTPS request completes.

### Fixes

1. **Run with Docker Compose (recommended)**  
   The `api` service in `docker-compose.yml` is configured with DNS servers `8.8.8.8` and `8.8.4.4` so the container can resolve external hostnames. Restart the stack:

   ```bash
   docker compose down && docker compose up -d api
   ```

   Then call `POST /api/v1/receipts/process` again.

2. **Run uvicorn directly (e.g. on WSL or host)**  
   Ensure the machine has outbound DNS and HTTPS (no firewall blocking `api.llamaindex.ai`). From the same machine, test:

   ```bash
   curl -s -o /dev/null -w "%{http_code}" https://api.llamaindex.ai/v1/health
   ```

   If this fails or times out, fix DNS/network on that machine (e.g. WSL: check `/etc/resolv.conf`; corporate: proxy or VPN).

3. **Optional: override LlamaParse API URL**  
   If you must use a proxy or a different endpoint, set in `.env`:

   ```env
   LLAMAPARSE_BASE_URL=https://your-proxy.example.com/llamaparse/v1
   ```

   Then restart the backend. The receipt flow will use this base URL for upload and job status.

4. **Slow LlamaParse jobs / frontend timeout**  
   Receipt OCR can legitimately take more than 100 seconds before Groq receives
   markdown. The backend exposes receipt-specific LlamaParse timeouts:

   ```env
   RECEIPT_PARSER_LLAMAPARSE_REQUEST_TIMEOUT_SECONDS=180
   RECEIPT_PARSER_LLAMAPARSE_POLL_TIMEOUT_SECONDS=300
   ```

   Keep the Next.js `/api/v1/receipts/process` proxy and client deadline above
   the expected backend window, otherwise the browser can show a timeout while
   FastAPI continues processing the same receipt.

### Flow summary

- **Free tier:** Image → LlamaParse (LlamaIndex Cloud) → markdown → Groq (LLM) → JSON → math validation → response.
- **Keys used:** `LLAMAPARSE_RECEIPT_PARSER_API_KEY` or `LLAMAPARSE_API_KEY` for LlamaParse; Groq is used via the app’s existing LiteLLM/GROQ configuration for the text step.

After DNS/network is fixed, the same request (with valid `image_base64`) should return either `status: "verified"` with a `receipt` or `status: "unprocessable"` with a clear validation/LLM error message (no more ConnectError).
