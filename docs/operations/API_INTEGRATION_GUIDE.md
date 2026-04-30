# API Integration Guide

This document provides the standard HTTP contracts for integrating with the Frugal Fortress backend. All endpoints are served under the `/api/v1` prefix.

## 1. Authentication
The API uses JWT Bearer authentication. Tokens are issued via the Google OAuth2 flow (`/api/v1/auth/login/google`).

**Standard header (all modules):**
```http
Authorization: Bearer <YOUR_JWT_TOKEN>
```

**Module-specific header — Receipt Parser only:**
```http
X-User-Id: <USER_UUID>
```
> The `X-User-Id` header is **only consumed by the `/api/v1/receipts/*` endpoints** for legacy compatibility with the PWA. It is ignored by `/pdf`, `/audio`, `/genui`, `/dlp`, and `/auth`. Do not send it on those routes.

## 2. Core Endpoints

### Health & Observability
Check system health (Database, Redis, Queue, AI Models).
```bash
curl -X GET "http://localhost:8000/api/v1/" \
     -H "Accept: application/json"
```

## 3. Module: PDF to Anki

### Upload PDF (Async Processing)
Uploads a PDF and enqueues it for background processing. Returns an HTTP 202 Accepted with a `task_id`.
```bash
curl -X POST "http://localhost:8000/api/v1/pdf/upload" \
     -H "Authorization: Bearer <TOKEN>" \
     -F "user_tier=premium" \
     -F "file=@/path/to/document.pdf"
```

### Poll Task Status
Check the status of the background job.
```bash
curl -X GET "http://localhost:8000/api/v1/pdf/status/<TASK_ID>" \
     -H "Authorization: Bearer <TOKEN>"
```

## 4. Module: Receipt Parser

### Process Receipt (Edge Compressed)
Submit a Base64 encoded image (compressed at the Edge) for extraction.
```bash
curl -X POST "http://localhost:8000/api/v1/receipts/process" \
     -H "Authorization: Bearer <TOKEN>" \
     -H "Content-Type: application/json" \
     -d '{
           "image_base64": "iVBORw0KGgo...",
           "user_role": "payer",
           "document_mime_type": "image/jpeg",
           "document_filename": "ticket.jpg"
         }'
```
*Note: If math validation fails after 3 retries, this endpoint returns `HTTP 206 Partial Content` with the AI's best-effort JSON for Human-in-the-Loop correction.*

## 5. Module: DLP Proxy (Security Guardrail)

### Sanitize Prompt
Scrub PII/PHI from a text string before sending it to an external LLM.
```bash
curl -X POST "http://localhost:8000/api/v1/dlp/process" \
     -H "Authorization: Bearer <TOKEN>" \
     -H "Content-Type: application/json" \
     -d '{"prompt": "My name is John Doe and my SSN is 999-88-7777."}'
```