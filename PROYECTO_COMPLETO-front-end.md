# PROJECT: front-end

### `.env.example`
```example
# Backend API (required for receipts, billing, etc.)
# For local development, use the relative path '/api/v1' to route traffic through the Next.js proxy.
# This ensures HttpOnly cookies (Path=/api/v1) are sent correctly by the browser.
# In production, set this to your full backend URL (e.g., https://backend.up.railway.app/api/v1).
NEXT_PUBLIC_BACKEND_URL=/api/v1

# Local development proxy target (points to your local FastAPI instance)
BACKEND_PROXY_TARGET=http://localhost:8000

# Fallback user id when not logged in (optional). If unset, the app uses "anonymous"
# (aligned with backend default) for X-User-Id and receipt query params.
# Tier in the UI still requires GET /receipts/usage?user_id=... to return { tier, ... }.
NEXT_PUBLIC_USER_ID=dev-user-xyz

# Optional (dev only): if /receipts/usage fails or times out, show this tier label in the receipts nav.
# NEXT_PUBLIC_RECEIPTS_DEV_TIER=admin

# Supabase (optional; used by other routes such as /auth/callback with Supabase exchange)
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key

# Receipts “Sign in with Google” uses the BACKEND, not Supabase directly:
# - GET {BACKEND}/auth/providers → { "google": "<full OAuth URL>" | null }
# - After OAuth, GET {BACKEND}/auth/callback/google?code=...
# See docs/receipts-backend-auth.md

# Same URL as on the Google Cloud OAuth consent screen; shown in the pre-login
# disclosure modal for Receipts and PDF–Anki (optional — link hidden if unset).
# NEXT_PUBLIC_PRIVACY_POLICY_URL=https://example.com/privacy

# PDF–Anki wallet: Stripe Payment Link or Checkout URL shown on /pdf-anki/pricing.
# Configure Stripe metadata: module=pdf_anki, payg=true, user_id or
# client_reference_id = Google sub, currency USD.
# NEXT_PUBLIC_PDF_ANKI_WALLET_CHECKOUT_URL=https://buy.stripe.com/...

# Receipts (dev-only): mock backend response for POST /receipts/process
# with HTTP 206 (requires human correction) so the HITL form appears.
# NEXT_PUBLIC_MOCK_RECEIPTS_PROCESS_206=1
```

### `api-client.ts`
```ts
export type HttpMethod = "GET" | "POST" | "PUT" | "DELETE" | "PATCH";

export type ApiClientConfig = {
	baseUrl?: string;
	retries?: number;
	timeoutMs?: number;
};

const DEFAULT_TIMEOUT_MS = 15000;

const defaultConfig: Required<ApiClientConfig> = {
	baseUrl: process.env.NEXT_PUBLIC_BACKEND_URL ?? "http://localhost:8000/api/v1",
	retries: 3,
	timeoutMs: DEFAULT_TIMEOUT_MS,
};

export class ApiError extends Error {
	public readonly status: number;

	public readonly data: any;

	public constructor(message: string, status: number, data?: any) {
		super(message);
		this.name = "ApiError";
		this.status = status;
		this.data = data;
	}
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

export async function apiRequest<TResponse, TBody = unknown>(
	path: string,
	options: {
		method?: HttpMethod;
		body?: TBody | FormData;
		signal?: AbortSignal;
		headers?: Record<string, string>;
		retries?: number;
	} = {}
): Promise<TResponse> {
	const config = defaultConfig;

	const cleanPath = path.startsWith("/") ? path.slice(1) : path;
	const cleanBase = config.baseUrl.endsWith("/") ? config.baseUrl : `${config.baseUrl}/`;
	const url = new URL(cleanPath, cleanBase);

	const isFormData = options.body instanceof FormData;
	const headers = new Headers(options.headers);

	if (!isFormData && options.body !== undefined && !headers.has("Content-Type")) {
		headers.set("Content-Type", "application/json");
	}

	const maxRetries = options.retries ?? config.retries;
	let attempt = 0;

	while (attempt <= maxRetries) {
		const controller = new AbortController();
		const timeoutId = setTimeout(() => controller.abort(), config.timeoutMs);
		const signal = options.signal ?? controller.signal;

		try {
			const response = await fetch(url.toString(), {
				method: options.method ?? "GET",
				headers,
				body: isFormData 
					? (options.body as FormData) 
					: (options.body !== undefined ? JSON.stringify(options.body) : undefined),
				signal,
			});

			if (!response.ok) {
				const isRetryable = response.status === 429 || response.status >= 500;
				if (isRetryable && attempt < maxRetries) {
					attempt++;
					const backoff = Math.min(1000 * 2 ** attempt, 10000);
					await sleep(backoff);
					continue;
				}

				let errorData: any;
				try {
					errorData = await response.json();
				} catch {
					errorData = await response.text();
				}
				throw new ApiError(`API request failed with status ${response.status}`, response.status, errorData);
			}

			if (response.status === 204) {
				return {} as TResponse;
			}

			return (await response.json()) as TResponse;

		} catch (error: any) {
			if (error.name === "AbortError" || error.name === "TimeoutError") {
				if (attempt < maxRetries) {
					attempt++;
					await sleep(1000 * 2 ** attempt);
					continue;
				}
				throw new ApiError("Request timed out", 408);
			}
			if (error instanceof ApiError) {
				throw error;
			}
			if (attempt < maxRetries) {
				attempt++;
				await sleep(1000 * 2 ** attempt);
				continue;
			}
			throw new ApiError(error.message || "Network error", 0);
		} finally {
			clearTimeout(timeoutId);
		}
	}
	throw new ApiError("Max retries exceeded", 0);
}
```

### `comandos.md`
```md
# Reiniciar front-end (puerto 3001, ver `package.json`)

```powershell
# Liberar puerto 3001 y arrancar
Get-NetTCPConnection -LocalPort 3001 -State Listen -ErrorAction SilentlyContinue | ForEach-Object { Stop-Process -Id $_.OwningProcess -Force -ErrorAction SilentlyContinue }
npm run dev
```

# PDF-ANKI
- Comprobar POST /pdf/upload
- Comprobar GET /pdf/status/

Despues de subir PDF, buscar el task_id (202)
http://localhost:8000/api/v1/pdf/status/EL-TASK-ID-UUID-AQUI
```

### `jest.config.cjs`
```cjs
/** @type {import('jest').Config} */
module.exports = {
	testEnvironment: "jsdom",
	roots: ["<rootDir>/src"],
	moduleFileExtensions: ["ts", "tsx", "js", "jsx"],
	transform: {
		"^.+\\.(ts|tsx)$": [
			"ts-jest",
			{
				tsconfig: "<rootDir>/tsconfig.jest.json"
			}
		]
	},
	setupFilesAfterEnv: ["<rootDir>/jest.setup.ts"],
	moduleNameMapper: {
		"^@/(.*)$": "<rootDir>/src/$1"
	}
};
```

### `jest.setup.ts`
```ts
import "@testing-library/jest-dom";

if (!(globalThis as unknown as { Headers?: unknown }).Headers) {
	class PolyfillHeaders {
		private readonly map = new Map<string, string>();

		constructor(init?: HeadersInit) {
			if (!init) return;
			if (init instanceof PolyfillHeaders) {
				init.forEach((value, key) => this.set(key, value));
				return;
			}
			if (Array.isArray(init)) {
				for (const [key, value] of init) {
					this.set(key, value);
				}
				return;
			}
			for (const [key, value] of Object.entries(init)) {
				if (value !== undefined) {
					this.set(key, String(value));
				}
			}
		}

		get(name: string): string | null {
			return this.map.get(name.toLowerCase()) ?? null;
		}

		set(name: string, value: string): void {
			this.map.set(name.toLowerCase(), String(value));
		}

		has(name: string): boolean {
			return this.map.has(name.toLowerCase());
		}

		forEach(
			callback: (value: string, key: string, parent: PolyfillHeaders) => void,
			thisArg?: unknown,
		): void {
			for (const [key, value] of this.map.entries()) {
				callback.call(thisArg, value, key, this);
			}
		}

		entries(): IterableIterator<[string, string]> {
			return this.map.entries();
		}
	}

	(globalThis as unknown as { Headers: typeof Headers }).Headers =
		PolyfillHeaders as unknown as typeof Headers;
}

if (!(globalThis as unknown as { Response?: unknown }).Response) {
	class PolyfillResponse {
		readonly status: number;
		readonly headers: Headers;
		readonly ok: boolean;
		private readonly bodyText: string;

		constructor(body?: BodyInit | null, init: ResponseInit = {}) {
			this.status = init.status ?? 200;
			this.headers = new Headers(init.headers);
			this.ok = this.status >= 200 && this.status < 300;
			if (typeof body === "string") {
				this.bodyText = body;
			} else if (body instanceof Blob) {
				this.bodyText = "";
			} else if (body == null) {
				this.bodyText = "";
			} else {
				this.bodyText = String(body);
			}
		}

		async text(): Promise<string> {
			return this.bodyText;
		}

		async json(): Promise<unknown> {
			return JSON.parse(this.bodyText || "{}");
		}

		async blob(): Promise<Blob> {
			return new Blob([this.bodyText]);
		}
	}

	(globalThis as unknown as { Response: typeof Response }).Response =
		PolyfillResponse as unknown as typeof Response;
}

if (!(globalThis as unknown as { Request?: unknown }).Request) {
	class PolyfillRequest {
		readonly url: string;
		readonly method: string;
		readonly headers: Headers;
		readonly body: BodyInit | null;

		constructor(url: string, init: RequestInit = {}) {
			this.url = url;
			this.method = (init.method ?? "GET").toUpperCase();
			this.headers = new Headers(init.headers);
			this.body = (init.body ?? null) as BodyInit | null;
		}

		async formData(): Promise<FormData> {
			if (this.body instanceof FormData) {
				return this.body;
			}
			return new FormData();
		}
	}

	(globalThis as unknown as { Request: typeof Request }).Request =
		PolyfillRequest as unknown as typeof Request;
}
```

### `next.config.js`
```js
const withPWA = require("@ducanh2912/next-pwa").default({
	dest: "public",
	disable: process.env.NODE_ENV === "development",
	fallbacks: {
		document: "/offline"
	}
});

/**
 * When BACKEND_PROXY_TARGET is set (e.g. http://localhost:8000), same-origin
 * requests to /api/v1/* are forwarded to {target}/api/v1/* so the browser
 * never hits cross-origin CORS during local dev. Set NEXT_PUBLIC_BACKEND_URL to
 * /api/v1 (see .env.local).
 */
const backendProxyTarget = process.env.BACKEND_PROXY_TARGET?.replace(/\/$/, "");

/** @type {import('next').NextConfig} */
const nextConfig = {
	reactStrictMode: true,
	poweredByHeader: false,

	experimental: {
		proxyClientMaxBodySize: "110mb", // backend hard limit is 100 MB; +10 MB for overhead
	},
	turbopack: {},
	async rewrites() {
		if (
			process.env.NODE_ENV !== "development" ||
			!backendProxyTarget ||
			backendProxyTarget.length === 0
		) {
			return [];
		}
		return [
			{

				source: "/api/v1/:path*",
				destination: `${backendProxyTarget}/api/v1/:path*`,
			},
			{

				source: "/support-attachments/:path*",
				destination: `${backendProxyTarget}/support-attachments/:path*`,
			},
		];
	},
};

module.exports = withPWA(nextConfig);
```

### `package.json`
```json
{
	"name": "frontend-platform",
	"version": "1.0.0",
	"description": "Edge-first Intelligent Portfolio frontend (Next.js 14, App Router).",
	"private": true,
	"scripts": {
		"dev": "next dev --webpack -p 3001",
		"build": "next build --webpack",
		"start": "next start",
		"lint": "next lint",
		"test": "jest"
	},
	"keywords": [
		"nextjs",
		"edge-compute",
		"pwa",
		"frugal-ai"
	],
	"author": "",
	"license": "ISC",
	"engines": {
		"node": ">=20.19.0"
	},
	"dependencies": {
		"@ducanh2912/next-pwa": "^10.2.9",
		"@hookform/resolvers": "^5.2.2",
		"@radix-ui/react-slot": "^1.2.4",
		"@supabase/ssr": "^0.9.0",
		"@supabase/supabase-js": "^2.99.3",
		"@tanstack/react-query": "^5.90.21",
		"@types/node": "^25.4.0",
		"@types/react": "^19.2.14",
		"autoprefixer": "^10.4.27",
		"class-variance-authority": "^0.7.1",
		"dexie": "^4.3.0",
		"lucide-react": "^0.577.0",
		"next": "^16.2.0",
		"postcss": "^8.5.8",
		"react": "^18.3.1",
		"react-dom": "^18.3.1",
		"react-hook-form": "^7.66.0",
		"tailwind-merge": "^3.5.0",
		"tailwindcss": "^4.2.1",
		"typescript": "^5.9.3",
		"xlsx": "^0.18.5",
		"zod": "^4.1.11",
		"zustand": "^5.0.11"
	},
	"devDependencies": {
		"@tailwindcss/postcss": "^4.2.1",
		"@testing-library/jest-dom": "^6.9.1",
		"@testing-library/react": "^16.3.2",
		"@testing-library/user-event": "^14.6.1",
		"@types/jest": "^30.0.0",
		"eslint": "^9.39.4",
		"eslint-config-next": "^16.1.6",
		"jest": "^30.3.0",
		"jest-environment-jsdom": "^30.3.0",
		"ts-jest": "^29.4.6"
	},
	"overrides": {
		"serialize-javascript": "^7.0.4"
	}
}
```

### `postcss.config.js`
```js
module.exports = {
	plugins: {
		"@tailwindcss/postcss": {},
		autoprefixer: {}
	}
};
```

### `README.md`
```md
# Frontend Platform — Intelligent Portfolio

Edge-first Next.js 14 frontend for the **Frugal Fortress** Enterprise AI Solution. This repository serves as the web adapter for the FastAPI backend and is designed as a **Gold Standard** template for US Fortune 500–grade reliability, scalability, and security (SOC2/FinOps).

---

## Overview

- **Stack:** Next.js 14 (App Router), TypeScript, Tailwind CSS, Jest + Testing Library.
- **Architecture:** Module-per-route with typed contracts (Zod ↔ backend Pydantic). Each business domain is isolated with its own route, error boundaries, and loading states.
- **Backend:** Single FastAPI monolith; all API calls go through a centralized typed client.

---

## Repository structure

```
frontend-platform/
├── src/
│   ├── app/                    # App Router routes and layouts
│   │   ├── page.tsx             # Home with links to modules
│   │   ├── layout.tsx
│   │   ├── genui/               # GenUI module (generative proposals)
│   │   ├── pdf-anki/            # PDF to Anki (flashcards + pricing)
│   │   └── receipts/            # Receipt Parser (PWA, image → backend VLM)
│   ├── lib/
│   │   └── api-client.ts        # Typed HTTP client for backend
│   ├── modules/                 # Domain logic per feature
│   │   ├── genui/
│   │   ├── pdf-anki/
│   │   ├── receipt-parser/
│   │   └── observability/
│   └── workers/
│       └── ocr.worker.ts        # Edge OCR (receipts)
├── docs/                        # Architecture and per-module docs
│   ├── genui/                   # README.md, how_to_use.md, how_to_test.md
│   ├── pdf-anki/
│   ├── receipts/
│   ├── ARCHITECTURE_FRONTEND.md
│   └── DEPLOYMENT.md
├── package.json
├── next.config.js
├── tailwind.config.js
└── jest.config.cjs
```

---

## Modules

| Module      | Route      | Purpose |
|------------|------------|--------|
| **GenUI**  | `/genui`   | Static portfolio + streaming generative UI for tailored proposals. Fast path: static projects; slow path: LLM-streamed analysis and components. |
| **PDF–Anki** | `/pdf-anki` | Upload PDFs; backend generates flashcards asynchronously. Polling-based progress and deck preview. Uses subscription tiers documented under `/pdf-anki/pricing`. |
| **Receipts** | `/receipts` | Mobile-first PWA: capture receipt images, compress on-device, send Base64 to backend VLMs for parsing. Offline queue (IndexedDB) for sync when back online. |

Detailed docs per module:

- [GenUI](docs/genui/README.md) · [How to use GenUI](docs/genui/how_to_use.md)
- [PDF to Anki](docs/pdf-anki/README.md) · [How to use PDF–Anki](docs/pdf-anki/how_to_use.md)
- [Receipt Parser](docs/receipts/README.md) · [How to use Receipts](docs/receipts/how_to_use.md)

---

## Prerequisites

- **Node.js** 18+ (LTS recommended)
- **npm** (or compatible package manager)
- **Backend** running and reachable at `NEXT_PUBLIC_BACKEND_URL` (default: `http://localhost:8000/api/v1`) 

---

## Quick start

```bash
# Install dependencies
npm install

# Development
npm run dev
```

Open [http://localhost:3001](http://localhost:3001). The home page links to each module.

---

## Scripts

| Command       | Description |
|---------------|-------------|
| `npm run dev` | Start Next.js dev server |
| `npm run build` | Production build |
| `npm run start` | Run production server |
| `npm run lint` | Run ESLint |
| `npm test`     | Run Jest test suite |

To run tests for a single module:

```bash
npm test -- genui
npm test -- pdf-anki
npm test -- receipts
```

---

## Environment variables

| Variable                    | Required | Description |
|----------------------------|----------|-------------|
| `NEXT_PUBLIC_BACKEND_URL`  | No       | Base URL for the FastAPI backend (default: `http://localhost:8000/api/v1`) |
| `NEXT_PUBLIC_CALCOM_URL`   | No       | Cal.com (or similar) URL for “Book Meeting” in GenUI (default: `https://cal.com`) |

All client-side config must use the `NEXT_PUBLIC_` prefix so Next.js exposes it to the browser.

---

## Design principles

- **Edge-first:** Heavy work (e.g. OCR for receipts) runs in the browser when possible to reduce backend load and preserve privacy.
- **Module isolation:** Each domain has its own route folder, Zod schemas, and error/loading boundaries to avoid cascading failures.
- **Typed contracts:** Backend responses are validated with Zod schemas aligned to Pydantic models; no untyped payloads in the app layer.
- **Observability:** Shared attention-tracking and structured logging patterns for production diagnostics.

---

## Testing

- **Location:** Tests live in `__tests__` folders next to the code they cover.
- **Strategy:** Smoke tests for the app and navigation; per-module tests for hooks, streaming, offline queue, and UI with mocks where appropriate.
- **References:** [Core testing](docs/core/how_to_test.md), plus each module’s `how_to_test.md`.

---

## Deployment

See [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) for deployment notes (e.g. Railway + Cloudflare, proxy of `/api/*` to the backend, caching and FinOps).

---

## License

ISC.
```

### `repositories.md`
```md
Inventario Final de Repositorios (Solo 4 Repos)
📦 backend-monolith: (Python) API para AudioNotes, ReceiptParser, PDF2ANKI, GenUI.
💻 frontend-platform: (Next.js) Tu Portafolio interactivo. Contiene las UIs de GenUI, Receipt Parser y PDF2Anki.
⚡ cv-tailor-edge: (HTML/JS/WASM) Proyecto aislado porque usa WebLLM y debe ser una PWA estática pura (sin servidor Node).
🧩 linkedin-viral-ext: (JS) Extensión de Chrome (Manifest V3).
```

### `resumen visual de arquitectura.txt`
```txt
[USUARIO]
		│
		├── Accede a Web (Navegador) ────────────────────────┐
		│                                                    │
		│   [Vercel Cloud] (Sin Docker)                      │
		│   ┌──────────────────────────────────────────┐     │
		│   │ REPO 2: frontend-platform (Next.js)      │     │
		│   │  ├─ /genui                               │     │
		│   │  ├─ /pdf-anki                            │     │
		│   │  └─ /receipts (PWA)                      │     │
		│   └────────────────────┬─────────────────────┘     │
		│                        │ (HTTPS JSON)              │
		│                        ▼                           │
		│   [Railway Cloud] (Con Docker Seguro)              │
		│   ┌──────────────────────────────────────────┐     │
		│   │ REPO 1: backend-monolith (Python)        │◄────┘
		│   │  ├─ FastAPI (Router)                     │
		│   │  ├─ Audio Notes Logic                    │
		│   │  ├─ PDF Logic                            │
		│   │  └─ Receipt Logic                        │
		│   └──────────────────────────────────────────┘
		│
		├── Usa Extensión Chrome ───────────────────────────┐
		│   [Local Browser]                                 │
		│   ┌──────────────────────────────────────────┐    │
		│   │ REPO 4: linkedin-viral-ext               │    │
		│   └──────────────────────────────────────────┘    │
		│                                                   │
		└── Usa CV Tailor ──────────────────────────────────┘
				[Vercel Cloud] (Sin Docker)
				┌──────────────────────────────────────────┐
				│ REPO 3: cv-tailor-edge                   │
				└──────────────────────────────────────────┘
```

### `tailwind.config.js`
```js
/** @type {import('tailwindcss').Config} */
module.exports = {
	content: [
		"./src/app/**/*.{ts,tsx}",
		"./src/components/**/*.{ts,tsx}",
		"./src/lib/**/*.{ts,tsx}",
		"./src/modules/**/*.{ts,tsx}"
	],
	theme: {
		extend: {
			colors: {
				background: "hsl(0 0% 100%)",
				foreground: "hsl(222.2 47.4% 11.2%)",
				muted: "hsl(210 40% 96%)",
			},
			borderRadius: {
				lg: "0.75rem"
			}
		}
	},
	plugins: []
};
```

### `TODO.md`
```md
[DONE]
Vamos a añadir en este panel de tickets comun para los modulos pdf-anki y receipts, la opcion para que el usuario pueda subir imagenes adjuntas (hasta 10 para un solo ticket) al crear un ticket o al enviar una respuesta (en cualquiera de los dos casos).  Como en la captura, cada imagen estará representada con un cuadrado pequeño. El usuario puede hacer clic en el cuadrado para agrandar la imagen, o clic en la esquina superior derecha de este cuadrado (una X) para borrar la imagen. Cuando el usuario está creando un ticket estas imagenes que adjunte aparecerán a la derecha del texto "Tickets". Cuando el usuario está respondiendo a un ticket previamente creado, aparecerán junto al texto "Puedes responder ahora.". Las imagenes adjuntas dentro de un texto del historial deben aparecer después del mensaje al que van adjuntas (en formato más grande, adaptado al tamaño del historial). 

Añade además una confirmación adicional cuando se crea un ticket.

El campo de historial con todos los mensajes de un ticket debe tener una barra para hacer scroll, para que el area de texto de la respuesta (más abajo) se quede estática en lugar de moverse hacia abajo cuando hay muchos mensajes en el historial de un ticket.

Como dije anteriormente... al adjuntar una imagen a un ticket (sea cuando lo creamos o cuando respondemos a uno ya creado), tiene que aparecer un cuadrado similar a este (adjunto) junto al botón de añadir adjunto. Ese cuadrado representa la imagen recien adjuntada (una miniatura). Si se hace clic sobre ella, la imagen se abre a pantalla completa en un modal. Cada uno de estos cuadrados de imagenes adjuntas tendrá una "x" en la esquina superior derecha. Si se pulsa, se borra la imagen de los adjuntos.

Al crear el ticket (o responder) se adjuntan las imagenes tambien al historial del ticket (despues del mensaje de texto). Estas imagenes se deben poder visualizar a escala media (30x30px por ejemplo) DENTRO del campo de historial de un ticket.

Cuando se sube una imagen adjunta o se elimina, hay que actualizar el contador "0/10 imagenes". 

Anima el botón de adjuntar imagenes (los 2) al hacer clic en el, para que el usuario confirme visualmente que se ha hecho clic en el (y no tenga que esperar a que se abra el explorer para confirmar dicho clic).

- Comprueba que las imagenes adjuntas en un historial de un ticket aparecen correctamente en el historial ys e puede clic sobre ellas para abrirlas en un modal grande.

 - Añade un pie de pagina común a todos los módulos. En la esquina izquierda, nombre del módulo y nombre de la compañía (JWAY SALES). En la esquina derecha, los iconos correspondientes para que el usuario pueda administrar tickets o incluir feedback (excepto en modulo receipts, implementaciones de ambos descritas a continuación).

Implementación de Sistema de Tickets
-----------------------------------------
Contexto:
Actúa como un Staff Backend Engineer y Frontend Architect. Vamos a expandir el sistema de tickets de soporte (actualmente exclusivo de audio_notes) para que esté disponible en los módulos web (pdf_anki y receipt_parser) para usuarios de pago (Premium, Pro, PAYG).
Reglas Estrictas:
Arquitectura Hexagonal: No dupliques lógica. Extrae la lógica existente al Core.
Tipado: mypy --strict obligatorio en backend y TypeScript estricto en frontend.
No rompas el flujo actual de Telegram para audio_notes.
Ejecuta esto en FASES. No me des código de la siguiente fase hasta que yo apruebe la actual.

Integración Frontend (Next.js)
Objetivo: Crear la interfaz de usuario para los módulos web.
En frontend-platform/src/lib/, crea un archivo support-api.ts con las funciones fetchTickets, createTicket, replyToTicket y deleteTicket usando el apiRequest existente.
Crea un componente React <SupportTicketModal /> (o una página dedicada) usando Tailwind CSS.
El componente debe:
Mostrar la lista de tickets (máximo 5).
Permitir crear uno nuevo (si hay menos de 3 activos).
Mostrar el historial de un ticket seleccionado.
Deshabilitar el input de respuesta si el último mensaje es del usuario (regla: esperar respuesta de soporte).
Integra este componente en los layouts o páginas principales de /receipts y /pdf-anki, visible solo si el usuario tiene un tier de pago.

-----------
FEEDBACK

 A. Módulo: receipt_parser (Facturas)
Estado actual: Ya tenemos feedback implícito de altísimo valor. Cuando el LLM falla la validación matemática (HTTP 206), el usuario lo corrige en el frontend y hace un POST /receipts/history. 

Feedback Explícito:
Mecanismo: Cuando el usuario corrige el campo Vendedor, Comprador o NIF, y guarda los cambios, se guarda la corrección para ser aplicada en facturas posteriores. Ej: campo Comprador es "Quillet" y lo cambia a "Quillet S.A.". Cuando se procese otra factura con este comprador, se actualizará el campo de texto automáticamente.

Para los campos de Vendedor, Comprador o NIF, cuando se corrigen (al invalidar una factura cuando se sube, o en la pagina de historial cuando se edita una factura), si antes no había valor (Ej: el original tiene NIF=null pero luego al editarlo añado un valor), tambien cuenta como feedback_log. Añade los cambios necesarios y dame el prompt para el frontend, si hace falta.

Impacto AI: Este texto se guarda en FeedbackLog. En el futuro, si sube una factura del mismo proveedor, el sistema recupera este feedback mediante búsqueda vectorial y lo inyecta en el System Prompt: "Contexto previo: El comprador Quillet se escribe como Quillet S.A.".

B. Módulo: pdf_anki (Tarjetas de Estudio)
Estado actual: Cero feedback. El usuario recibe el mazo y lo exporta.
Oportunidad de Feedback Explícito (Calidad de las Flashcards):
Implementación UI: En la vista de revisión (/pdf-anki/deck/[id]), añadir dos acciones por tarjeta:
Editar Tarjeta: Permite al usuario modificar el anverso/reverso de la tarjeta y guardar los cambios. 
Descartar Tarjeta (👎): Si la tarjeta es irrelevante (ej. extrajo texto del índice del PDF).

Mecanismo:
Al editar, enviamos al backend: original_front, original_back, new_front, new_back.
Al descartar, enviamos la tarjeta con un flag de rejected.
Impacto AI: Guardamos esto en FeedbackLog. Modificamos el FlashcardGeneratorAdapter para que, antes de generar tarjetas para un chunk, busque en el vector store. Si encuentra tarjetas rechazadas previamente por ese usuario, las inyecta como Negative Few-Shot Examples ("DO NOT generate cards like this: ..."), siempre cuidando que quepan dentro del contexto del LLM. Si encuentra tarjetas editadas/modificadas, las inyecta como Positive Few-Shot Examples ("Format your answers like this: ..."), igualmente teniendo en cuenta los limites de contexto del modelo. Los ejemplos Positivos tendrán prioridad por encima de los negativos para ser añadidos en el contexto.

- En ambos modulos pdf-anki y receipt_parser, NO mostrar el contenido de la pagina correspondiente HASTA no haber cargado el nombre del tier del usuario en la barra de navegacion superior. Ej: la pagina del historial de receipts no carga su panel de "Emitidos/Recibidos" hasta que el nombre del tier del usuario se visualiza en la barra de navegacion superior.

- Remueve el texto "Intelligent Portafolio" del encabezado de paginas. Donde estaba este texto, añade el nombre del modulo (cuando el usuario navega dentro de las paginas de un modulo especifico).
Nombre del Modulo Receipt Parser: "Receipt Parser AI Copilot"
Nombre del modulo pdf-anki: "PDF to ANKI AI Copilot"

- Elimina el nombre del modulo del pie de pagina (esquina inferior izquierda).

PROMPT TABLA DE TRANSACCIONES Y OBSERVABILIDAD
----------------------------------------------
Act as a Staff Frontend Architect. We need to implement Phase 4: The UI for the PAYG Transaction Ledger in our Next.js application.

Context:
- The backend exposes `GET /api/v1/billing/transactions` and `GET /api/v1/billing/transactions/export?format=xlsx`.
- We need to show this to users ONLY if they have `payg=true` (or `pdf_anki_payg=true` / `receipt_parser_payg=true`).

Task:
1. Update `src/app/AuthHeader.tsx`:
	 - Inside the dropdown menu (where the user's name, email, and "Sign Out" are), add a new link: "Transaction History" (use i18n).
	 - This link MUST ONLY be rendered if `isPdfAnkiWalletUser(me)` is true OR `me.receipt_parser_payg` is true.
	 - The link should route to `/billing/transactions`.
2. Create the page `src/app/billing/transactions/page.tsx`:
	 - Fetch data from the new backend endpoint.
	 - Display a clean, TailwindCSS-styled table: Date, Module, Description, Type (Credit/Debit), Amount, Balance After.
	 - Add an "Export to XLSX" button at the top right.
	 - Handle loading and empty states gracefully.
3. Update `src/lib/i18n-config.ts` (and related dictionaries):
	 - Add translations for the new menu link, table headers, and export button in all supported languages (en, es, de, it, fr, ru).

Strict Rules:
- Use the existing `apiRequest` utility.
- Ensure strict TypeScript typing for the API responses.
- Follow the existing Tailwind design system (slate/emerald/rose colors for credits/debits).

- En la tabla de transacciones (http://localhost:3001/billing/transactions), permite que el usuario pueda filtrar los datos de la tabla por modulo (campo module en la tabla de supabase).

- Esta pagina /billing/transactions tiene que mostrar tambien la cabecera del modulo en cuestion desde el cual hemos hecho clic para abrir esta pagina. Por tanto, los detalles sobre esa cabecera (nombre del modulo, tier del usuario, panel de usuario..) dependerán de la pagina desde donde el usuario dio clic para abrir esta pagina. Además:
- Si el usuario viene del modulo pdf_anki, se seleccionará este filtro automaticamente.
- Si el usuario viene del modulo receipt_parser, se seleccionara este filtro automaticamente tambien al visualizar las transacciones

El usuario seguirá teniendo libertad para quitar o cambiar los filtros.

- en el pdf-anki (si bien no esta implementado aun), como el resultado del proceso no requiere atencion humana en ningun caso (el usuario descarta o edita flashcards desde la pagina del mazo, no desde la pagina principal) si que podriamos añadir procesamiento multiple de PDFs (online solo, la opcion offline no esta en este modulo), donde el usuario sigue añadiendo PDFs para procesamiento, y la vista previa mostrará siempre el ultimo resultado de tarjetas generadas.

Act as a Senior Frontend Engineer expert in React, Next.js, and TypeScript.

Context:
I have a backend module called `pdf-anki` built with FastAPI and Arq (Hexagonal Architecture). 
When a user uploads a PDF to `POST /api/v1/pdf/upload`, the backend instantly returns an HTTP 202 Accepted with a `task_id`. The heavy processing (LlamaParse, Vector Search, LLM generation) happens asynchronously.
The frontend must poll `GET /api/v1/pdf/status/{task_id}` to get the status (`pending`, `processing`, `completed`, `failed`) and the resulting flashcards deck.

Objective:
Refactor the main upload page for the `pdf-anki` module to support MULTIPLE sequential/concurrent uploads without blocking the user.

Strict Requirements:
1. Non-Blocking UI: The file dropzone/upload button MUST remain visible and active immediately after a PDF is submitted. The user can upload PDF #2 while PDF #1 is still processing.
2. Concurrent Polling: The component must be able to poll multiple `task_id`s simultaneously. Use a robust state management approach (e.g., `useRef` or functional state updates) to avoid stale closures and React state race conditions during polling.
3. Latest Preview Focus: The UI should have a "Latest Generated Deck" preview section. Whenever ANY of the polling tasks reaches the `completed` status, this preview section should update to display the flashcards from that specific deck.
4. Task List (Optional but recommended): A small sidebar or toast list showing the status of currently processing PDFs (e.g., "Biology.pdf - Processing...", "History.pdf - Completed").
5. No Offline Mode: Do not implement IndexedDB or offline queuing (unlike our receipt parser). This is strictly online.
6. No Inline Editing: The preview section should be read-only. Do not implement edit/discard logic here, as that belongs to the dedicated `/deck/[id]` route. Just show the cards and a "Go to Deck" button.

Technical Constraints:
- Use TailwindCSS for styling.
- Use standard `fetch` or `axios` for API calls.
- Ensure polling stops cleanly when a task completes, fails, or if the component unmounts (cleanup `setInterval` or `setTimeout`).

Please provide the complete TypeScript React component for this upload page.

En el cuadro de "Cola Visual" quita la vista previa de tarjetas dentro de cada deck (mazo) especifico. Ademas, ahora mismo no se visualiza al pie del deck el numero de tarjetas hasta que el usuario hace clic en ese deck, cambia esto para que el numero de tarjetas se visualice desde que el deck es creado e insertado en esta cola. Además, en la cola visual los mazos deben listarse uno debajo del otro, no uno al lado del otro como ahora (si no, al crear muchos mazos de golpe el bloque de Cola Visual se hace cada vez mas grande hasta que solapa el tamaño del bloque de vista previa de tarjetas). 

Cuando se suben documentos y estos aparecen procesandose en el bloque "Procesos de esta sesión", en este bloque quita el campo "Tarea: 3341c3e7-2589-4698-b65a-37a9c367df00" y la leyenda de abajo "Para el plan FREE, una página típica con texto plano suele tardar ~30s, y una ..."
Además, tienes que captar el momento en el que un documento ha terminado de procesarse (respuesta desde el backend) para quitarlo del bloque "Procesos de esta sesión" y añadirlo en el bloque "Cola Visual" de forma automática (ahora mismo solo se actualiza al recargar la página).

Act as a Senior Frontend Engineer working on a Next.js PWA. I need to fix a bug in our offline-sync queue mechanism for the "Receipt Parser" module.

### Context & Current Behavior
1. **Offline Queue (IndexedDB):** When the user is offline, captured receipt images (Base64 payloads) are stored in IndexedDB. When the connection is restored, a sync function drains this queue and sends them to `POST /api/v1/receipts/process`.
2. **HITL State Recovery:** If the API returns `HTTP 206 Partial Content`, the user enters a Human-in-the-Loop (HITL) correction form. If they accidentally refresh the page, the form state is successfully recovered. *This part is already implemented and working perfectly.*

### The Bug
The raw receipt payload is **never deleted from IndexedDB** after the backend responds. 
Because it remains in IndexedDB, if the user finishes the flow, clicks "New Receipt", or reloads the page later, the offline sync mechanism detects the old payload and re-sends it to the backend, causing duplicate processing.

### Requirements to Implement
Please provide the React/TypeScript code (or the specific IndexedDB utility logic) to fix this:

1. **Targeted Deletion:** Inside the offline sync loop, immediately after the `fetch` to `/api/v1/receipts/process` resolves successfully (specifically, if it returns `HTTP 200 OK` or `HTTP 206 Partial Content`), the exact item MUST be deleted from IndexedDB using its unique ID or key.
2. **Error Handling:** If the request fails due to a network error (e.g., the user drops offline again during the request), the item should REMAIN in IndexedDB to be retried later. If it fails with a `4xx` or `5xx` error (meaning the server rejected it permanently), it should also be deleted from IndexedDB to prevent an infinite loop of bad requests.

### Tech Stack
- Next.js (React)
- TypeScript
- `idb` (or whatever standard IndexedDB wrapper we might be using)

Please provide an example of how the `syncOfflineReceipts` function should be structured to safely iterate over the IndexedDB records, await the fetch, and delete the record upon a definitive response.

- Cuando el idioma es ingles, veo que se siguen harcodeando algunos valores en español.

Correction required

This extraction was recovered automatically. Review fields and charges before saving.

Could not extract invoice number.

Total must be 99.0. It is 98.0.

Validation errors

Field: Nº factura. Nº factura. Expected: identificador visible de factura. Got: N/A.

Obviamente no tiene que decir Nº Factura si el idioma es inglés, sino su traducción al idioma correspondiente (Invoice # en este caso, pero tiene que estar traducido a todos los idiomas contemplados.)

El botón de "Recalcular" tampoco se está traduciendo al idioma correspondiente (ni en la pagina de escaneo ni tampoco en la de edición de una factura). Además, en la pagina del historial de facturas, las columnas Noº Factura	Fecha Expiración	Moneda, no se están traduciendo al idioma apropiado.

[TESTS]

[PENDING]

MODULO GENUI
- Pagina de Portafolio (modulo GenUI):
		- Home (enlace a esta pagina de portafolio)
		- Pestaña "About Us", y añade una plantilla de pagina.
		- GenUI (Generador de Propuestas de Negocio)...

- Seguridad y Cumplimiento (SOC2): Implementar el manejo del SecurityBlockCard en src/app/genui/page.tsx para interceptar y mostrar violaciones de PII/PHI sin romper la UI.

[STAGING]
```

### `TODO_genui.md`
```md
[DONE]
[TEST]
[PENDING]
Act as a Staff Frontend Architect expert in Next.js 14, React, and TailwindCSS. We are implementing the component registry for the `genui` module.

Context:
- The backend streams NDJSON chunks. Some chunks have `type: "component"` and a `component` string name (e.g., "PdfAnkiFlashcard").
- We need to render these components dynamically inside the proposal text stream in `src/app/genui/page.tsx`.

Task:
1. Create a new file `src/modules/genui/GenUIComponentRegistry.tsx`.
2. Implement a mapping function or dictionary that takes a `component` string and `props` and returns the corresponding React node.
3. Implement "Demo/Read-Only" versions of our existing static components to be used in the registry. They must look professional and use TailwindCSS:
	 - `<DemoPdfAnkiFlashcard front={...} back={...} tags={...} />`: A flippable 3D card.
	 - `<DemoReceiptHitlForm demoMode={true} />`: A simplified, read-only version of the Human-in-the-Loop form showing a math error (e.g., Subtotal 10 + Tax 2 = Total 15).
	 - `<DemoAudioNoteTelegramMock transcript={...} summary={...} />`: A visual mock of a Telegram message with a fake audio player and "Useful/Inaccurate" inline buttons.
	 - `<DemoPdfAnkiErrorAlert error={{code: "groq_free_tier_busy", title: "..."}} />`: A Graceful Degradation notice.
	 - `<DemoCapacityBanner usage={{at_80_percent: true}} />`: A banner warning about storage limits.
4. Update `src/app/genui/page.tsx` to use this registry inside the `streamChunks.map` loop.

Strict Rules:
- Ensure strict TypeScript interfaces for the props of each demo component.
- Do not import the actual heavy components from other modules if they require complex Context Providers. Build lightweight, visually identical "Demo" wrappers specifically for the GenUI portfolio.

If you need to consult the backend code, it's synthetized on 'PROYECTO_COMPLETO-back-end.md'

COMPONENTES DINAMICOS

Act as a Staff Frontend Architect and UI/UX Animator. We are building the dynamic visualizer components for the `genui` module.

Context:
- The backend will stream requests to render complex architectural visualizers.
- These components must look highly professional, targeting Enterprise CTOs.
- We also need to handle the `SecurityBlockCard` if the backend DLP proxy intercepts the request.

Task:
1. In `src/modules/genui/GenUIComponentRegistry.tsx`, add support for the following new components.
2. Build `<FinOpsROICalculator competitorCost={...} frugalFortressCost={...} savings={...} />`: A visual bar chart comparing the two costs, highlighting the savings percentage in Emerald green.
3. Build `<AsyncSingleFlightVisualizer concurrentRequests={...} cacheHits={...} />`: A CSS-animated component showing multiple request nodes merging into a single "LLM Execution" node, then fanning back out to the users.
4. Build `<RAGLearningLoopVisualizer originalError={...} correction={...} />`: A 3-step visual showing: 1) User Correction, 2) pgvector embedding, 3) Few-Shot Injection.
5. Build `<SecurityBlockCard reason={...} />`: A prominent, red-themed alert card explaining that the DLP Proxy intercepted the request to prevent a data leak (SOC2 compliance demonstration).
6. Build `<ArchitectureFitScore score={...} reason={...} />`: A speedometer-style gauge or circular progress bar showing the fit score, followed by a "Book Technical Interview" CTA button.

Strict Rules:
- Use TailwindCSS for all styling and animations (e.g., `animate-pulse`, custom keyframes in `globals.css` if necessary). Do not add heavy external animation libraries unless absolutely necessary.
- Ensure responsive design (mobile-friendly).
- Validate all incoming props using TypeScript interfaces.

If you need to consult the backend code, it's synthetized on 'PROYECTO_COMPLETO-back-end.md'

PRUEBAAAAAAAAAAAAAAAAAAAAAAAAAAS:

- We are a healthcare company processing thousands of medical PDFs per month. We need HIPAA-safe extraction, lower LLM costs, graceful fallback when providers are busy, and auditable study-card generation for internal training.

- We process vendor invoices and receipts at scale. Our main risk is inaccurate OCR math, especially subtotal, tax, and total mismatches. Show me how your architecture handles human-in-the-loop validation, cost controls, and auditability.

- We receive many field-team voice notes through Telegram and need reliable transcription, summaries, feedback buttons, and protection against duplicate LLM calls during traffic spikes. Explain the architecture and ROI.

- We need a proposal, but this prompt includes a fake api key for testing the DLP proxy.

Objetivo: Forzar la aparición del <CapacityBanner /> y evaluar cómo el sistema explica la poda de datos (Pruning) y el cumplimiento de la minimización de datos.
Prompt: "We have strict GDPR data retention policies. Our users generate massive amounts of data, but we cannot store it indefinitely, and we need to warn them before their storage caps are reached. How does your architecture handle data lifecycle, storage limits, and automated pruning?"

Objetivo: Forzar la aparición del <AlternativeApproachCard /> y ver cómo el sistema maneja una petición que va en contra de los principios de la Frugal Fortress (ej. pidiendo un stack distinto o procesamiento síncrono pesado).
Prompt: "We are a Java/Spring Boot shop looking to build a synchronous, monolithic batch processor for video files. We don't want to use async workers or queues, we just want the HTTP request to stay open until the 2-hour video is processed. Can your architecture support this?"

Objetivo: Forzar la aparición del <AsyncSingleFlightVisualizer /> de forma aislada. Aunque lo tocaste en el prompt de Telegram, este prompt evalúa puramente el problema de concurrencia extrema sobre un mismo recurso.
Prompt: "We run a social platform where a single piece of media can go viral in seconds. If 10,000 users submit the exact same prompt or file simultaneously, our current LLM API bill skyrockets and the database connection pool crashes. How do you prevent cache stampedes and duplicate inference?"

Objetivo: Forzar la aparición del <RAGLearningLoopVisualizer />. Evalúa cómo el sistema vende la idea de corregir al modelo sin gastar dinero en Fine-Tuning.
Prompt: "Our industry uses highly specific acronyms and internal jargon. Off-the-shelf LLMs constantly hallucinate or misspell our product names. We cannot afford the MLOps overhead of fine-tuning models every week. How can your system learn our vocabulary in real-time?"

Objetivo: Ver cómo el LLM prioriza y renderiza múltiples componentes a la vez (FinOpsROICalculator, SecurityBlockCard, PdfAnkiErrorAlert, etc.) cuando el cliente tiene un problema masivo que abarca todos los dominios.
Prompt: "We are an enterprise logistics company. Field workers send voice notes via Telegram, accountants upload thousands of receipts, and HR processes massive compliance PDFs. We are facing 3 critical issues: 1) LLM costs are out of control, 2) PII like SSNs are leaking into logs, and 3) when OpenAI goes down, our entire app crashes with 500 errors. Propose a comprehensive architecture."

Objetivo: Forzar al LLM a explicar el patrón de Ports & Adapters y renderizar un <ArchitectureFitScore /> alto, justificando cómo el core domain está aislado de la infraestructura.
Prompt:
"We are terrified of vendor lock-in with OpenAI. If they raise prices, deprecate a model, or suffer a massive outage, we need to swap to Anthropic, Google Vertex, or local open-source models immediately without rewriting our core business logic. How does your codebase handle provider agnosticism?"

Objetivo: Mientras que el prompt del API Key prueba el DLP Proxy (datos en tránsito hacia el LLM), este escenario evalúa cómo el sistema defiende los datos en reposo contra amenazas internas (DBAs maliciosos o brechas de base de datos).
Prompt:
"Our DBAs and DevOps engineers have root access to our production PostgreSQL databases. For SOC2 compliance, we must guarantee that even with direct database access, our clients' financial totals and private study notes remain completely unreadable to our internal staff. How is data-at-rest secured in your architecture?"

Objetivo: Poner a prueba si el LLM es capaz de articular la diferencia entre el Nested Learning Loop (RAG en tiempo real) y la compilación de heurísticas offline (DSPy), demostrando madurez en AI Engineering.
Prompt:
"We have complex routing rules and prompt instructions that are currently hardcoded and brittle. We want to optimize our system prompts systematically based on past evaluations, but we don't want to run heavy prompt-optimizers in production during user requests. How do you handle prompt compilation and routing policies without adding latency?"

Prompt:
We have complex routing rules and prompt instructions that are currently hardcoded and brittle. We want to optimize our system prompts systematically based on past evaluations, but we don't want to run heavy prompt-optimizers in production during user requests. How do you handle prompt compilation and routing policies without adding latency?

Prompt: 
We need a proposal, but this prompt includes a fake api key for testing the DLP proxy: sk-1234567890abcdef1234567890abcdef

Observabilidad
Mientras ejecutas estos prompts en el frontend, mantén abierta la pestaña de Network (Red) en las DevTools de tu navegador:
-observar la consola para detectar errores

- en Loki, filtrar por '{job="app"} |= "/api/v1/genui/proposal/generate"'
o: {job="app"} | json | tenant_id =~ "genui-.*"

y extrae el tenant id, y filtra así: {job="app"} | json | trace_id="8af910a2b3e00bfd35e9f2cb7140a998"
```

### `TODO_PDF_ANKI.md`
```md
[DONE]

- Añade las siguientes traducciones para el nuevo botón de procesar nuevo PDF (junto al botón Ver Mazo después de que se generen las flashcards):
// En el diccionario `en`:
"pdfAnki.main.processAnother": "Process another PDF",

// En el diccionario `es`:
"pdfAnki.main.processAnother": "Procesar otro PDF",

// En el diccionario `de`:
"pdfAnki.main.processAnother": "Weiteres PDF verarbeiten",

// En el diccionario `it`:
"pdfAnki.main.processAnother": "Elabora un altro PDF",

// En el diccionario `fr`:
"pdfAnki.main.processAnother": "Traiter un autre PDF",

// En el diccionario `ru`:
"pdfAnki.main.processAnother": "Обработать другой PDF",

- El botón de Mazos de la barra de navegación superior SOLO está visible cuando un usuario se loguea (Google u otros).

- Añade xlsx como formato para exportar tarjetas. 

- En la página de Mazos, donde se visualizan los mazos de un usuario (campos declaracion derechos humanos-7.pdf25/3/2026 completed), despues de la etiqueta "completed/completed_partial", añadir un botón con un icono de papelera rojo para borrar un deck (con mensaje emergente para pedir confirmación antes de confirmar la acción).

- El botón "Quitar filtro de etiquetas" en la pagina del Deck, duplicarlo y ponerlo justo debajo de "Etiquetas", para que en el caso de que haya muuuchas etiquetas, el usuario no necesite desplazarse al final de la lista para quitar los filtros.

- En el mensaje de espera cuando se está procesando un pdf "Para el plan admin, una página típica con texto plano suele tardar unos 2–4 minutos, y una página con gráficos o tablas unos 6–10 minutos.", reduce a la mitad los tiempos para cada tier en particular. Nuevo mensaje sería: "Para el plan X, procesar una página típica con texto plano tarda ~30s, y una página con gráficos o tablas 2-3 minutos." - siendo X el tier correspondiente

- Añadimos la nueva clave de error y actualizamos los textos de la página de precios para reflejar los nuevos límites de páginas.

// src/lib/i18n-config.ts
// (Añade estas líneas en cada diccionario de idioma)

const en: TranslationDict = {
		// ...
		"pdfAnki.main.error.pageLimitExceeded": "Page limit exceeded: {{msg}}",
		// Actualizar los highlights de pricing:
		"pdfAnki.pricing.tier.free.highlight.1": "Up to 10 pages per PDF",
		"pdfAnki.pricing.tier.premium.highlight.1": "Up to 100 pages per PDF",
		"pdfAnki.pricing.tier.pro.highlight.1": "Up to 500 pages per PDF",
		// ...
};

const es: TranslationDict = {
		// ...
		"pdfAnki.main.error.pageLimitExceeded": "Límite de páginas excedido: {{msg}}",
		// Actualizar los highlights de pricing:
		"pdfAnki.pricing.tier.free.highlight.1": "Hasta 10 páginas por PDF",
		"pdfAnki.pricing.tier.premium.highlight.1": "Hasta 100 páginas por PDF",
		"pdfAnki.pricing.tier.pro.highlight.1": "Hasta 500 páginas por PDF",
		// ...
};

const de: TranslationDict = {
		// ...
		"pdfAnki.main.error.pageLimitExceeded": "Seitenlimit überschritten: {{msg}}",
		"pdfAnki.pricing.tier.free.highlight.1": "Bis zu 10 Seiten pro PDF",
		"pdfAnki.pricing.tier.premium.highlight.1": "Bis zu 100 Seiten pro PDF",
		"pdfAnki.pricing.tier.pro.highlight.1": "Bis zu 500 Seiten pro PDF",
		// ...
};

const it: TranslationDict = {
		// ...
		"pdfAnki.main.error.pageLimitExceeded": "Limite di pagine superato: {{msg}}",
		"pdfAnki.pricing.tier.free.highlight.1": "Fino a 10 pagine per PDF",
		"pdfAnki.pricing.tier.premium.highlight.1": "Fino a 100 pagine per PDF",
		"pdfAnki.pricing.tier.pro.highlight.1": "Fino a 500 pagine per PDF",
		// ...
};

const fr: TranslationDict = {
		// ...
		"pdfAnki.main.error.pageLimitExceeded": "Limite de pages dépassée : {{msg}}",
		"pdfAnki.pricing.tier.free.highlight.1": "Jusqu'à 10 pages par PDF",
		"pdfAnki.pricing.tier.premium.highlight.1": "Jusqu'à 100 pages par PDF",
		"pdfAnki.pricing.tier.pro.highlight.1": "Jusqu'à 500 pages par PDF",
		// ...
};

const ru: TranslationDict = {
		// ...
		"pdfAnki.main.error.pageLimitExceeded": "Превышен лимит страниц: {{msg}}",
		"pdfAnki.pricing.tier.free.highlight.1": "До 10 страниц на PDF",
		"pdfAnki.pricing.tier.premium.highlight.1": "До 100 страниц на PDF",
		"pdfAnki.pricing.tier.pro.highlight.1": "До 500 страниц на PDF",
		// ...
};

- Al acercar el raton al icono de usuario, que se abra, y hasta que el usuario decida apartar el ratón. Pero si hace clic en el icono, el panel se queda abierto hasta que se vuelva a dar clic para cerrarlo (como hasta ahora). Solo este modulo.

[TEST]

[PENDING]
```

### `TODO_receipts.md`
```md
[DONE]
- Edge Compression, Dynamic Form & Auth (Frontend - Next.js)

Act as a Staff Frontend Architect. We are building the Next.js (Vercel) frontend for the Receipt Parser module.

Context:
- We must offload compute to the Edge (Browser) to save server costs.

Task:
1. Edge Image Compression: Create a utility function that takes a File input (Camera/Gallery), draws it onto an HTML5 `<canvas>`, resizes the longest edge to a maximum of 1024px, and exports it as a JPEG with 0.8 quality. The resulting Base64 string should be ~100-150KB.
2. Human-in-the-Loop UI: Create a Dynamic Form (using `react-hook-form` and `zod`) that mirrors the `ReceiptSchema`. 
	 - When the backend returns the parsed JSON, populate this form.
	 - Allow the user to manually correct any math or OCR errors.
	 - Add a "Save to History" button that sends the *corrected* JSON back to the backend to be saved in Supabase.
3. History View: Create a dashboard with two tabs: "Issued" (Emitidas) and "Received" (Recibidas), fetching data from the backend.
4. Auth: Implement Supabase Auth UI for Google and Apple ID login.

Ensure strict TypeScript typing and responsive design (TailwindCSS).

- FinOps, Stripe, Tiers & UI Banners (Fullstack)
Act as a Staff FinOps Engineer. We are implementing the billing and tier limits for the Receipt Parser.

Context:
- Tiers: Free, Premium, Pro, PAYG.
- Stripe URLs are in `.env` (e.g., `RP_STRIPE_LINK_PREMIUM`).

Task:
3. Frontend (Navigation): Add a real-time Balance/Tier indicator in the top navigation bar next to the User Profile icon.
4. Frontend (Capacity Warning): Implement a check that calculates `(current_receipts / max_receipts) * 100`. If the result is >= 80%, display a prominent warning banner: "Te acercas a tu límite de almacenamiento. Exporta tu historial en CSV/JSON o actualiza al plan Pro." Include the Stripe upgrade link in the banner.
5. Frontend (Pricing Page): Create a `/pricing` route displaying the tiers based on `tiers.md`, linking to the respective Stripe URLs from the environment variables.

Ensure all database updates are transactional and secure.

En la pagina de historial (modulo receipts), en el formulario inteligente que aparece cuando seleccionamos una factura, el subtotal se debe calcular de forma inteligente según añadimos/quitamos items, aumentamos o reducimos el descuento. Haz que también los impuestos estén calculados automáticamente, pero SOLO si hay creada ya al menos una Tasa para que sirva de referencia. El campo Descuento ahora es "Descuento (Moneda)", y debajo hay otro campo "Descuento %". Estos dos campos se deshabilitan correspondientemente cuando el usuario introduce un valor en uno de los dos (también se deshabilitan si por defecto hay valores en el otro campo).

En FORMULARIOS DE EDICIÓN DE FACTURAS:
- Hacer que el campo "Tipo de Recibo" tenga ambos valores posibles para ser seleccionados ("Recibido" | "Emitido"). Si se guarda un valor distinto, la próxima vez la factura aparecerá en el bloque correspondiente (Emitidos/Recibidos)

-  módulo de receipts ahora puede aceptar png,jpg,jpeg,webp, .HEIC, pdf, tiff como formato del documento al escanear una factura. Si el archivo no tiene ninguna de estas extensiones, hay que mostrar un error de formato de archivo no permitido.

- En el Historial de facturas, en la tabla donde se visualizan las facturas (con las columnas Fecha	Noº Factura	Tipo	Fecha Expiración	Moneda	Vendedor	Comprador	Total) añade un campo adicional para cada factura, justo después de Total, con un icono de papelera rojo, para borrar una factura (con mensaje emergente para confirmar).

- Cambiar "botón" para subir una factura. 
Texto actual es: "Seleccionar archivo", conviertelo en un botón.

- El botón "Nuevo recibo" SOLO se muestra cuando YA se ha parseado una factura y se está mostrando al usuario. Esto le da opción para empezar a parsear una nueva factura (y borra la vista de la factura parseada para comenzar de nuevo el proceso).

- Al exportar un historial, añade también la opción para exportar a XLSX. Cuidado con los acentos al exportar. Ejemplo:
2014-12-07,2014-12-30,0029,0004,Isabel GonzÃ¡lez LÃ³pez,"Quillet, S.A.",,355.2,17.76,74.59,412.03,EUR,RECEIVED,payer

- Al crear un documento con el historial de facturas, crea las columnas utilizando el lenguaje seleccionado en la aplicación. Cualquier campo almacenado en base de datos en inglés (RECEIVED, PAYER,...) DEBE traducirse también al idioma correspondiente, dentro del documento a exportar.

- Implementar una opción para activar la cámara y hacer una foto de una factura directamente para parsearla, en lugar de estar forzado a subir el fichero.

- Cuando un formulario que se está editando en la página de history, se intenta guardar pero la validación falla, en lugar de mostrar el error así:"Line item total does not match quantity * unit_price: line 7 has total 24 but expected 23.95", asegúrate de ponerlo en el idioma del usuario. En el mensaje que también aparece arriba " Regla: line_item_total_mismatch. La regla line_item_total_mismatch no coincide: esperado 23.95, obtenido 24." quitar toda esa parte, dejando solo "Corrección requerida El total de la línea #7 debe ser 23.95. Ahora es 24."

- Cambiar etiqueta "Lineas" por "Artículos"

- El botón "Guardar en el historial" es ahora "Guardar cambios".

[TEST]

- confirma que cuando un usuario tiene +80% de almacenamiento lleno, aparecera un icono de alerta y al pasar el raton o hacer clic en el se podra ver el area de texto "Estás llegando al limite de almacenamiento. Si tu almacenamiento se llena, usaremos la técnica FIFO para almacenar nuevos recibos, borrando los más antiguos. No olvides que puedes exportar tu historial de recibos. Para aumentar tu almacenamiento, ve a la página de planes y Suscribete a un plan superior."

[PENDING]

[FUTURE]
-  cuando un usuario sube una factura en el frontend, puede elegir si es el Pagador o el Emisor de esa factura. ¿Se te ocurre CÓMO podamos usar esta información para mejorar el servicio ofrecido al usuario? (aparte de la separación actual entre facturas emitidas/recibidas al visualizar y exportar).
```

### `tsconfig.jest.json`
```json
{
	"extends": "./tsconfig.json",
	"compilerOptions": {
		"jsx": "react-jsx",
		"module": "commonjs",
		"moduleResolution": "node",
		"noEmit": false
	}
}
```

### `tsconfig.json`
```json
{
	"compilerOptions": {
		"target": "ES2022",
		"lib": [
			"dom",
			"dom.iterable",
			"es2022"
		],
		"allowJs": false,
		"skipLibCheck": true,
		"strict": true,
		"noImplicitAny": true,
		"noFallthroughCasesInSwitch": true,
		"noUncheckedIndexedAccess": true,
		"forceConsistentCasingInFileNames": true,
		"module": "esnext",
		"moduleResolution": "bundler",
		"resolveJsonModule": true,
		"isolatedModules": true,
		"jsx": "react-jsx",
		"incremental": true,
		"paths": {
			"@/*": [
				"./src/*"
			]
		},
		"types": [
			"jest",
			"@testing-library/jest-dom",
			"node"
		],
		"noEmit": true,
		"esModuleInterop": true,
		"plugins": [
			{
				"name": "next"
			}
		]
	},
	"include": [
		"next-env.d.ts",
		"src/**/*.ts",
		"src/**/*.tsx",
		".next/types/**/*.ts",
		".next/dev/types/**/*.ts"
	],
	"exclude": [
		"node_modules",
		".next"
	]
}
```

### `.claude/settings.local.json`
```json
{
	"permissions": {
		"allow": [
			"Bash(xargs -I {} basename {})"
		]
	}
}
```

### `scripts/generate-pwa-icons.ps1`
```ps1
$ErrorActionPreference = "Stop"
$base = Split-Path -Parent $PSScriptRoot
$outDir = Join-Path $base "public\icons"
New-Item -ItemType Directory -Force -Path $outDir | Out-Null
Add-Type -AssemblyName System.Drawing
foreach ($size in @(192, 512)) {
	$bmp = New-Object System.Drawing.Bitmap $size, $size
	$g = [System.Drawing.Graphics]::FromImage($bmp)
	$g.Clear([System.Drawing.Color]::FromArgb(2, 6, 23))
	$g.Dispose()
	$path = Join-Path $outDir "icon-$size.png"
	$bmp.Save($path, [System.Drawing.Imaging.ImageFormat]::Png)
	$bmp.Dispose()
	Write-Host "Wrote $path"
}
```

### `src/proxy.ts`
```ts
import { type NextRequest } from "next/server";

import { updateSession } from "@/lib/supabase/middleware";

export async function proxy(request: NextRequest) {
	return updateSession(request);
}

export const config = {
	matcher: [
		"/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)",
	],
};
```

### `src/app/AppHeader.tsx`
```tsx
"use client";

import { Suspense } from "react";
import { usePathname, useSearchParams } from "next/navigation";

import { LanguageSwitcher } from "@/app/LanguageSwitcher";

function AppHeaderContent() {
	const pathname = usePathname();
	const searchParams = useSearchParams();

	const isReceiptsRoute =
		pathname === "/receipts" || pathname?.startsWith("/receipts/");
	const isPdfAnkiRoute =
		pathname === "/pdf-anki" || pathname?.startsWith("/pdf-anki/");
	const isGenUIRoute =
		pathname === "/genui" || pathname?.startsWith("/genui/");
	const billingSource = searchParams.get("source");
	const isBillingTransactionsRoute =
		pathname === "/billing/transactions" &&
		(billingSource === "pdf_anki" || billingSource === "receipt_parser");

	if (
		isReceiptsRoute ||
		isPdfAnkiRoute ||
		isGenUIRoute ||
		isBillingTransactionsRoute
	) {
		return null;
	}

	return (
		<header className="flex items-center justify-end border-b border-slate-200 bg-white px-4 py-2">
			<LanguageSwitcher />
		</header>
	);
}

export function AppHeader() {
	return (
		<Suspense fallback={null}>
			<AppHeaderContent />
		</Suspense>
	);
}
```

### `src/app/AuthHeader.tsx`
```tsx
"use client";

import { useEffect, useRef, useState } from "react";
import Link from "next/link";
import { CircleUser } from "lucide-react";

import { useI18n } from "@/app/i18n-provider";
import { useSupabaseAuth } from "@/app/AuthProvider";
import { isPdfAnkiWalletUser } from "@/app/auth/backend-auth-api";

type AuthHeaderProps = {
	/**
	 * When true the dropdown opens on hover and closes when the mouse leaves,
	 * UNLESS the user has click-locked it open (click to lock, click again to
	 * unlock). Default: false (click-only, existing behaviour).
	*/
	hoverToOpen?: boolean;
	transactionsHref?: string;
};

export function AuthHeader({
	hoverToOpen = false,
	transactionsHref = "/billing/transactions",
}: AuthHeaderProps = {}) {
	const { t } = useI18n();
	const [loginError, setLoginError] = useState<string | null>(null);
	const [menuOpen, setMenuOpen] = useState(false);
	/** True only when the user clicked to pin the panel open. */
	const [clickLocked, setClickLocked] = useState(false);
	const [avatarError, setAvatarError] = useState(false);
	const menuRef = useRef<HTMLDivElement | null>(null);

	const { me, userId, isLoading, loginWithGoogle, signOut } = useSupabaseAuth();

	/** Fully closes and unlocks the panel (for external triggers). */
	const closeMenu = () => {
		setMenuOpen(false);
		setClickLocked(false);
	};

	useEffect(() => {
		setAvatarError(false);
	}, [me?.user_id]);

	useEffect(() => {
		if (!menuOpen) return;

		const onPointerDown = (e: PointerEvent): void => {
			if (menuRef.current?.contains(e.target as Node)) return;
			closeMenu();
		};

		const onKeyDown = (e: KeyboardEvent): void => {
			if (e.key === "Escape") closeMenu();
		};

		document.addEventListener("pointerdown", onPointerDown);
		document.addEventListener("keydown", onKeyDown);
		return () => {
			document.removeEventListener("pointerdown", onPointerDown);
			document.removeEventListener("keydown", onKeyDown);
		};

	}, [menuOpen]);

	if (isLoading) {
		return (
			<span className="text-xs text-slate-500">{t("auth.loading")}</span>
		);
	}

	const isSignedIn = me != null || userId != null;

	if (isSignedIn) {
		const handleButtonClick = () => {
			if (!hoverToOpen) {

				setMenuOpen((open) => !open);
				return;
			}
			if (clickLocked) {

				closeMenu();
			} else {

				setClickLocked(true);
				setMenuOpen(true);
			}
		};

		const containerHoverProps = hoverToOpen
			? {
					onMouseEnter: () => setMenuOpen(true),
					onMouseLeave: () => {
						if (!clickLocked) setMenuOpen(false);
					},
				}
			: {};

		return (
			<div className="relative" ref={menuRef} {...containerHoverProps}>
				<button
					type="button"
					onClick={handleButtonClick}
					className="flex h-8 w-8 shrink-0 items-center justify-center overflow-hidden rounded-full border border-slate-200 bg-white text-slate-700 shadow-sm hover:bg-slate-50"
					aria-expanded={menuOpen}
					aria-haspopup="true"
					aria-label={t("auth.userMenuAria")}
				>
					{me?.avatar_url && !avatarError ? (

						<img
							src={me.avatar_url}
							alt=""
							aria-hidden
							className="h-full w-full object-cover"
							onError={() => setAvatarError(true)}
							referrerPolicy="no-referrer"
						/>
					) : (
						<CircleUser className="h-5 w-5" strokeWidth={1.75} aria-hidden />
					)}
				</button>
				{menuOpen ? (
					<div
						className="absolute right-0 top-full z-50 mt-1 w-[min(calc(100vw-2rem),280px)] rounded-lg border border-slate-200 bg-white py-2 shadow-lg"
						role="menu"
						aria-label={t("auth.userMenuAria")}
					>
						<div className="space-y-1 px-3 pb-2 text-xs">
							{!me ? (
								<p className="text-slate-500">{t("auth.loading")}</p>
							) : (
								<>
									{me.display_name ? (
										<p className="font-semibold leading-snug text-slate-900">
											{me.display_name}
										</p>
									) : null}
									{me.email ? (
										<p
											className={`truncate text-slate-600 ${
												me.display_name ? "pt-0.5" : ""
											}`}
										>
											{me.email}
										</p>
									) : null}
									{!me.display_name && !me.email ? (
										<p className="text-slate-500">{t("auth.userFallback")}</p>
									) : null}

								</>
							)}
						</div>
						<div className="border-t border-slate-100 px-3 pt-2 space-y-1">
							{(isPdfAnkiWalletUser(me) || me?.receipt_parser_payg === true) ? (
								<Link
									href={transactionsHref}
									role="menuitem"
									target="_blank"
									rel="noopener noreferrer"
									onClick={closeMenu}
									className="block w-full rounded-md px-2 py-1.5 text-left text-xs font-medium text-slate-700 hover:bg-slate-100"
								>
									{t("billing.transactions.menuLink")}
								</Link>
							) : null}
							<button
								type="button"
								role="menuitem"
								onClick={() => {
									closeMenu();
									void signOut();
								}}
								className="w-full rounded-md bg-slate-100 px-2 py-1.5 text-left text-xs font-medium text-slate-800 hover:bg-slate-200"
							>
								{t("auth.signOut")}
							</button>
						</div>
					</div>
				) : null}
			</div>
		);
	}

	return (
		<div className="flex max-w-[min(100%,280px)] flex-col items-end gap-1">
			<button
				type="button"
				onClick={() => {
					setLoginError(null);
					void loginWithGoogle().catch((e: unknown) => {
						setLoginError(
							e instanceof Error ? e.message : t("auth.signInFailed"),
						);
					});
				}}
				className="rounded-full bg-slate-900 px-3 py-1.5 text-xs font-medium text-white hover:bg-slate-800"
			>
				{t("auth.signInGoogle")}
			</button>
			{loginError ? (
				<p className="text-right text-[10px] leading-snug text-rose-600">
					{loginError}
				</p>
			) : null}
		</div>
	);
}
```

### `src/app/AuthProvider.tsx`
```tsx
"use client";

import {
	createContext,
	useContext,
	useEffect,
	useMemo,
	useState,
	type ReactNode,
	useCallback,
} from "react";
import type { AuthMeResponse } from "@/app/auth/backend-auth-api";
import {
	fetchAuthMe,
	resolveGoogleOAuthUrl,
	logoutAuthCookies,
} from "@/app/auth/backend-auth-api";
import {
	parseAuthModule,
	type AuthModule,
} from "@/app/auth/auth-storage";
import {
	readBackendAccessToken,
	clearBackendAccessToken,
	BACKEND_ACCESS_TOKEN_CHANGED_EVENT,
} from "@/app/auth/backend-access-token";

const FALLBACK_USER_ID =
	typeof process !== "undefined"
		? process.env.NEXT_PUBLIC_USER_ID ?? "anonymous"
		: "anonymous";

type AuthContextValue = {
	authModule: AuthModule;
	userId: string | null;
	me: AuthMeResponse | null;
	isLoading: boolean;
	loginWithGoogle: () => Promise<void>;
	signOut: () => Promise<void>;
	refreshMe: () => Promise<void>;
};

const AuthContext = createContext<AuthContextValue | undefined>(undefined);

type AuthProviderProps = {
	children: ReactNode;
	module: AuthModule;
};

export function AuthProvider({ children, module }: AuthProviderProps) {
	const [userId, setUserId] = useState<string | null>(null);
	const [me, setMe] = useState<AuthMeResponse | null>(null);
	const [isLoading, setIsLoading] = useState(true);

	useEffect(() => {
		let cancelled = false;

		const run = async (): Promise<void> => {
			setIsLoading(true);
			try {
				const nextMe = await fetchAuthMe(module, { timeoutMs: 45_000 });
				if (cancelled) return;

				const meModule = parseAuthModule(nextMe.module);
				if (meModule != null && meModule !== module) {
					setUserId(null);
					setMe(null);
					return;
				}

				setMe(nextMe);
				setUserId(nextMe.user_id);
			} catch {
				if (cancelled) return;
				setUserId(null);
				setMe(null);
			} finally {
				if (!cancelled) setIsLoading(false);
			}
		};

		void run();

		return () => {
			cancelled = true;
		};
	}, [module]);

	const refreshMe = useCallback(async () => {
		try {
			const nextMe = await fetchAuthMe(module, { timeoutMs: 45_000 });
			const meModule = parseAuthModule(nextMe.module);
			if (meModule != null && meModule !== module) {
				setUserId(null);
				setMe(null);
				return;
			}

			setMe(nextMe);
			setUserId(nextMe.user_id);
		} catch {
			setUserId(null);
			setMe(null);
		}
	}, [module]);

	const loginWithGoogle = useCallback(async () => {
		const url = await resolveGoogleOAuthUrl(module);
		window.location.href = url;
	}, [module]);

	const signOut = useCallback(async () => {
		await logoutAuthCookies(module);
		clearBackendAccessToken();
		setUserId(null);
		setMe(null);
	}, [module]);

	const value = useMemo<AuthContextValue>(
		() => ({
			authModule: module,
			userId,
			me,
			isLoading,
			loginWithGoogle,
			signOut,
			refreshMe,
		}),
		[
			module,
			userId,
			me,
			isLoading,
			loginWithGoogle,
			signOut,
			refreshMe,
		],
	);

	return (
		<AuthContext.Provider value={value}>{children}</AuthContext.Provider>
	);
}

export function useSupabaseAuth(): AuthContextValue {
	const ctx = useContext(AuthContext);
	if (ctx === undefined) {
		throw new Error("useAuth must be used within AuthProvider");
	}
	return ctx;
}

export function useAuthModule(): AuthModule {
	return useSupabaseAuth().authModule;
}

/**
 * User id for receipt/billing API headers: Supabase user id, or fallback env.
 */
export function useUserId(): string {
	const { userId, isLoading } = useSupabaseAuth();
	if (isLoading) return FALLBACK_USER_ID;
	return userId ?? FALLBACK_USER_ID;
}

/**
 * JWT from backend OAuth JSON, stored in sessionStorage for Bearer headers.
 * HttpOnly cookies may still apply for same-origin `apiRequest` calls.
 */
export function useAccessToken(): string | null {
	const [token, setToken] = useState<string | null>(() =>
		typeof window === "undefined" ? null : readBackendAccessToken(),
	);

	useEffect(() => {
		const sync = (): void => {
			setToken(readBackendAccessToken());
		};
		window.addEventListener(BACKEND_ACCESS_TOKEN_CHANGED_EVENT, sync);
		return () => {
			window.removeEventListener(BACKEND_ACCESS_TOKEN_CHANGED_EVENT, sync);
		};
	}, []);

	return token;
}

export function useAuthMe(): AuthMeResponse | null {
	const { me, isLoading } = useSupabaseAuth();
	if (isLoading) return null;
	return me;
}
```

### `src/app/globals.css`
```css
@import "tailwindcss";

@layer base {
	:root {
		--app-background: theme(colors.slate.100);
		--app-foreground: theme(colors.slate.900);
		--app-surface: theme(colors.white);
		--app-border: theme(colors.slate.200);
	}

	html {
		color: var(--app-foreground);
		background: var(--app-background);
	}

	body {
		@apply bg-slate-100 text-slate-900 antialiased;
		font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
	}
}

@keyframes pdf-anki-thinking {
	0% {
		transform: translateX(-100%);
	}
	100% {
		transform: translateX(400%);
	}
}

@keyframes indeterminate-progress {
	0% {
		transform: translateX(-120%);
	}
	100% {
		transform: translateX(420%);
	}
}

@keyframes fade-in {
	from {
		opacity: 0;
		transform: translateY(4px);
	}
	to {
		opacity: 1;
		transform: translateY(0);
	}
}

@keyframes genui-bar-grow {
	from {
		transform: scaleX(0);
	}
	to {
		transform: scaleX(1);
	}
}

@keyframes genui-single-flight-in {
	0% {
		opacity: 0;
		transform: translate(-120px, -50%) scale(0.75);
	}
	25% {
		opacity: 1;
	}
	70% {
		opacity: 1;
		transform: translate(120px, -50%) scale(1);
	}
	100% {
		opacity: 0;
		transform: translate(150px, -50%) scale(0.65);
	}
}

@keyframes genui-single-flight-out {
	0% {
		opacity: 0;
		transform: translate(-130px, -50%) scale(0.65);
	}
	25% {
		opacity: 1;
	}
	75% {
		opacity: 1;
		transform: translate(95px, -50%) scale(1);
	}
	100% {
		opacity: 0;
		transform: translate(125px, -50%) scale(0.75);
	}
}

@keyframes genui-gauge-reveal {
	from {
		opacity: 0;
		transform: rotate(-18deg) scale(0.92);
	}
	to {
		opacity: 1;
		transform: rotate(0deg) scale(1);
	}
}

.pdf-anki-thinking-fill {
	animation: pdf-anki-thinking 2s ease-in-out infinite;
}

.indeterminate-progress {
	animation: indeterminate-progress 1.6s ease-in-out infinite;
	will-change: transform;
}
```

### `src/app/GoogleOAuthConsentDialog.tsx`
```tsx
"use client";

import { useEffect } from "react";

import { useI18n } from "@/app/i18n-provider";
import type { AuthModule } from "@/app/auth/auth-storage";
import { getPublicPrivacyPolicyUrl } from "@/app/auth/google-oauth-disclosure-storage";

type GoogleOAuthConsentDialogProps = {
	open: boolean;
	module: AuthModule;
	onCancel: () => void;
	onContinue: () => void;
};

export function GoogleOAuthConsentDialog({
	open,
	module,
	onCancel,
	onContinue,
}: GoogleOAuthConsentDialogProps) {
	const { t } = useI18n();
	const privacyUrl = getPublicPrivacyPolicyUrl();

	useEffect(() => {
		if (!open) return;
		const onKeyDown = (e: KeyboardEvent): void => {
			if (e.key === "Escape") onCancel();
		};
		document.addEventListener("keydown", onKeyDown);
		return () => document.removeEventListener("keydown", onKeyDown);
	}, [open, onCancel]);

	if (!open) {
		return null;
	}

	const bodyKey =
		module === "receipt_parser"
			? "auth.googleConsent.bodyReceipt"
			: "auth.googleConsent.bodyPdfAnki";

	return (
		<div
			className="fixed inset-0 z-[200] flex items-center justify-center bg-black/45 p-4"
			role="presentation"
			onPointerDown={(e) => {
				if (e.target === e.currentTarget) onCancel();
			}}
		>
			<div
				role="dialog"
				aria-modal="true"
				aria-labelledby="google-oauth-consent-title"
				className="max-h-[min(90vh,560px)] w-full max-w-md overflow-y-auto rounded-xl border border-slate-200 bg-white p-5 shadow-xl"
				onPointerDown={(e) => e.stopPropagation()}
			>
				<h2
					id="google-oauth-consent-title"
					className="text-base font-semibold text-slate-900"
				>
					{t("auth.googleConsent.title")}
				</h2>
				<p className="mt-3 text-sm leading-relaxed text-slate-600">
					{t(bodyKey)}
				</p>
				{privacyUrl ? (
					<p className="mt-4 text-sm">
						<a
							href={privacyUrl}
							target="_blank"
							rel="noopener noreferrer"
							className="font-medium text-slate-900 underline decoration-slate-300 underline-offset-2 hover:decoration-slate-600"
						>
							{t("auth.googleConsent.privacyLink")}
						</a>
					</p>
				) : null}
				<div className="mt-6 flex flex-col-reverse gap-2 sm:flex-row sm:justify-end sm:gap-3">
					<button
						type="button"
						onClick={onCancel}
						className="rounded-full border border-slate-200 bg-white px-4 py-2 text-sm font-medium text-slate-800 hover:bg-slate-50"
					>
						{t("auth.googleConsent.cancel")}
					</button>
					<button
						type="button"
						onClick={onContinue}
						className="rounded-full bg-slate-900 px-4 py-2 text-sm font-medium text-white hover:bg-slate-800"
					>
						{t("auth.googleConsent.continue")}
					</button>
				</div>
			</div>
		</div>
	);
}
```

### `src/app/i18n-provider.tsx`
```tsx
"use client";

import {
	createContext,
	useCallback,
	useContext,
	useEffect,
	useMemo,
	useState,
} from "react";

import {
	DEFAULT_LOCALE,
	isLocale,
	translate,
	type Locale,
} from "@/lib/i18n-config";

type I18nContextValue = {
	locale: Locale;
	t: (key: string, vars?: Record<string, string>) => string;
	setLocale: (locale: Locale) => void;
};

const I18nContext = createContext<I18nContextValue | undefined>(undefined);

type I18nProviderProps = {
	children: React.ReactNode;
	initialLocale?: Locale;
	forcedLocale?: Locale;
	persistLocale?: boolean;
};

export function I18nProvider({
	children,
	initialLocale = DEFAULT_LOCALE,
	forcedLocale,
	persistLocale = true,
}: I18nProviderProps) {
	const [locale, setLocaleState] = useState<Locale>(
		forcedLocale ?? initialLocale
	);

	useEffect(() => {
		if (forcedLocale) {
			setLocaleState(forcedLocale);
			return;
		}

		if (typeof window === "undefined") {
			return;
		}

		const stored = window.localStorage.getItem("locale") as
			| Locale
			| null
			| undefined;

		if (stored && isLocale(stored)) {
			setLocaleState(stored);
			return;
		}

		const browserLang =
			typeof navigator !== "undefined"
				? navigator.language?.split("-")[0]
				: undefined;

		setLocaleState(browserLang && isLocale(browserLang) ? browserLang : DEFAULT_LOCALE);
	}, [forcedLocale]);

	const setLocale = useCallback((next: Locale) => {
		if (forcedLocale) {
			setLocaleState(forcedLocale);
			return;
		}

		setLocaleState(next);
		if (persistLocale && typeof window !== "undefined") {
			window.localStorage.setItem("locale", next);
		}
	}, [forcedLocale, persistLocale]);

	const effectiveLocale = forcedLocale ?? locale;

	const value = useMemo<I18nContextValue>(
		() => ({
			locale: effectiveLocale,
			t: (key, vars) => translate(effectiveLocale, key, vars),
			setLocale,
		}),
		[effectiveLocale, setLocale],
	);

	return (
		<I18nContext.Provider value={value}>{children}</I18nContext.Provider>
	);
}

export function useI18n(): I18nContextValue {
	const ctx = useContext(I18nContext);
	if (!ctx) {
		throw new Error("useI18n must be used within an I18nProvider.");
	}
	return ctx;
}
```

### `src/app/LanguageSwitcher.tsx`
```tsx
"use client";

import { useI18n } from "@/app/i18n-provider";

export function LanguageSwitcher() {
	const { locale, setLocale, t } = useI18n();

	return (
		<label className="flex items-center gap-1 text-[11px] text-slate-700">
			<span className="hidden text-slate-500 sm:inline">
				{t("lang.switcher.label")}
			</span>
			<select
				className="rounded-full border border-slate-300 bg-white px-2 py-0.5 text-[11px] text-slate-800 focus:outline-none focus:ring-2 focus:ring-slate-400"
				value={locale}
				onChange={(event) =>
					setLocale(
						event.target.value === "es" ||
							event.target.value === "de" ||
							event.target.value === "it" ||
							event.target.value === "fr" ||
							event.target.value === "ru"
							? (event.target.value as "es" | "de" | "it" | "fr" | "ru")
							: "en",
					)
				}
			>
				<option value="en">{t("lang.switcher.en")}</option>
				<option value="es">{t("lang.switcher.es")}</option>
				<option value="de">{t("lang.switcher.de")}</option>
				<option value="it">{t("lang.switcher.it")}</option>
				<option value="fr">{t("lang.switcher.fr")}</option>
				<option value="ru">{t("lang.switcher.ru")}</option>
			</select>
		</label>
	);
}
```

### `src/app/layout.tsx`
```tsx
import type { Metadata } from "next";
import "./globals.css";
import { I18nProvider } from "@/app/i18n-provider";
import { AppHeader } from "@/app/AppHeader";

export const metadata: Metadata = {
	title: "Intelligent Portfolio Frontend",
	description: "Edge-first AI portfolio frontend platform."
};

type RootLayoutProps = {
	children: React.ReactNode;
};

export default function RootLayout({ children }: RootLayoutProps) {
	return (
		<html lang="en">
			<body className="min-h-screen bg-slate-100 text-slate-900">
				<I18nProvider>
					<div className="flex min-h-screen flex-col">
						<AppHeader />
						<main className="flex-1">{children}</main>
					</div>
				</I18nProvider>
			</body>
		</html>
	);
}
```

### `src/app/manifest.ts`
```ts
import type { MetadataRoute } from "next";

export default function manifest(): MetadataRoute.Manifest {
	return {
		name: "Receipt & Portfolio PWA",
		short_name: "AI Portfolio",
		start_url: "/",
		display: "standalone",
		background_color: "#020617",
		theme_color: "#020617",
		description:
			"Edge-first PWA for receipts, PDFs to Anki, and GenUI proposals.",
		icons: [
			{
				src: "/icons/icon-192.png",
				sizes: "192x192",
				type: "image/png"
			},
			{
				src: "/icons/icon-512.png",
				sizes: "512x512",
				type: "image/png"
			}
		]
	};
}
```

### `src/app/not-found.tsx`
```tsx
"use client";

import { MoveLeft } from "lucide-react";
import { useI18n } from "@/app/i18n-provider";

export default function NotFound() {
	const { t } = useI18n();

	return (
		<div className="relative flex min-h-screen flex-col items-center justify-center overflow-hidden bg-slate-100 px-6">
			{/* Decorative grid */}
			<div
				aria-hidden
				className="pointer-events-none absolute inset-0"
				style={{
					backgroundImage:
						"linear-gradient(to right, #cbd5e1 1px, transparent 1px), linear-gradient(to bottom, #cbd5e1 1px, transparent 1px)",
					backgroundSize: "40px 40px",
					maskImage:
						"radial-gradient(ellipse 70% 60% at 50% 50%, black 40%, transparent 100%)",
					WebkitMaskImage:
						"radial-gradient(ellipse 70% 60% at 50% 50%, black 40%, transparent 100%)",
					opacity: 0.45,
				}}
			/>

			{/* Glow accent */}
			<div
				aria-hidden
				className="pointer-events-none absolute left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 h-80 w-80 rounded-full bg-indigo-200/40 blur-3xl"
			/>

			{/* Card */}
			<div className="relative flex flex-col items-center gap-6 rounded-2xl border border-slate-200 bg-white/80 px-10 py-12 shadow-lg backdrop-blur-sm text-center max-w-sm w-full">
				{/* 404 */}
				<div className="flex flex-col items-center gap-1">
					<span className="select-none text-[6rem] font-black leading-none tracking-tighter text-slate-900">
						404
					</span>
					<div className="h-1 w-16 rounded-full bg-indigo-500/70" />
				</div>

				{/* Message */}
				<div className="flex flex-col gap-1">
					<h1 className="text-lg font-semibold text-slate-900">
						{t("notFound.title")}
					</h1>
					<p className="text-sm text-slate-500">
						{t("notFound.description")}
					</p>
				</div>

				{/* Action */}
				<button
					type="button"
					onClick={() => history.back()}
					className="flex items-center justify-center gap-2 rounded-lg border border-slate-200 bg-white px-5 py-2.5 text-sm font-medium text-slate-700 transition hover:bg-slate-50 active:scale-95 w-full"
				>
					<MoveLeft size={15} />
					{t("notFound.goBack")}
				</button>
			</div>
		</div>
	);
}
```

### `src/app/page.tsx`
```tsx
"use client";

import Link from "next/link";

import { useAttentionTracker } from "@/modules/observability/useAttentionTracker";

export default function HomePage() {
	useAttentionTracker({ moduleId: "home" });

	return (
		<main className="mx-auto flex min-h-screen max-w-5xl flex-col gap-8 p-6">
			<header className="flex flex-col gap-4">
				<h1 className="text-3xl font-semibold text-slate-900">
					Intelligent Portfolio Frontend
				</h1>
				<p className="max-w-2xl text-sm text-slate-600">
					Edge-first frontend platform for Receipt Parser, PDF to Anki, and
					GenUI modules, optimized for reliability, scalability, and cost.
				</p>
			</header>
			<section className="grid gap-4 md:grid-cols-3">
				<Link
					href="/receipts"
					className="rounded-lg border border-slate-200 bg-white p-4 shadow-sm transition hover:border-slate-300 hover:shadow-md"
				>
					<h2 className="font-medium text-slate-900">Receipt Parser</h2>
					<p className="mt-2 text-xs text-slate-600">
						Mobile-first PWA for edge OCR and backend-validated receipts.
					</p>
				</Link>
				<Link
					href="/pdf-anki"
					className="rounded-lg border border-slate-200 bg-white p-4 shadow-sm transition hover:border-slate-300 hover:shadow-md"
				>
					<h2 className="font-medium text-slate-900">PDF to Anki</h2>
					<p className="mt-2 text-xs text-slate-600">
						Desktop-first async processing for flashcard generation.
					</p>
				</Link>
				<Link
					href="/genui"
					className="rounded-lg border border-slate-200 bg-white p-4 shadow-sm transition hover:border-slate-300 hover:shadow-md"
				>
					<h2 className="font-medium text-slate-900">GenUI</h2>
					<p className="mt-2 text-xs text-slate-600">
						Dynamic proposals with streaming generative UI components.
					</p>
				</Link>
			</section>
		</main>
	);
}
```

### `src/app/admin/audio-notes/page.tsx`
```tsx
"use client";

import { useEffect, useState } from "react";

import { apiRequest } from "@/lib/api-client";
import {
	type AudioNoteLogEntry,
	audioNotesLogResponseSchema
} from "@/modules/observability/schemas";
import { useAttentionTracker } from "@/modules/observability/useAttentionTracker";

type AudioNotesLogResponse = {
	items: AudioNoteLogEntry[];
	total: number;
};

export default function AudioNotesAdminPage() {
	const [logs, setLogs] = useState<AudioNoteLogEntry[]>([]);
	const [error, setError] = useState<string | null>(null);

	useAttentionTracker({ moduleId: "admin-audio-notes" });

	useEffect(() => {
		void loadLogs();
	}, []);

	const loadLogs = async () => {
		setError(null);
		try {
			const response = await apiRequest<AudioNotesLogResponse>(
				"/audio-notes/logs?limit=50"
			);
			const parsed = audioNotesLogResponseSchema.parse(response);
			setLogs(parsed.items);
		} catch {
			setError("Failed to load Audio Notes logs.");
		}
	};

	return (
		<main className="flex min-h-screen flex-col bg-slate-100 text-slate-900">
			<header className="border-b border-slate-200 bg-white px-6 py-4">
				<h1 className="text-2xl font-semibold">Audio Notes observability</h1>
				<p className="mt-1 max-w-2xl text-sm text-slate-600">
					Internal dashboard for inspecting recent Audio Notes executions.
					Access to this view should be restricted to administrators.
				</p>
			</header>
			<section className="flex-1 px-6 py-4">
				{error && (
					<p className="mb-3 text-xs text-rose-600">
						{error}
					</p>
				)}
				<div className="overflow-auto rounded-xl border border-slate-200 bg-white">
					<table className="min-w-full border-separate border-spacing-y-1 text-xs">
						<thead className="bg-slate-50 text-slate-600">
							<tr>
								<th className="px-3 py-2 text-left font-medium">Time</th>
								<th className="px-3 py-2 text-left font-medium">User</th>
								<th className="px-3 py-2 text-left font-medium">Status</th>
								<th className="px-3 py-2 text-left font-medium">Model</th>
								<th className="px-3 py-2 text-left font-medium">Cost (USD)</th>
								<th className="px-3 py-2 text-left font-medium">Duration</th>
								<th className="px-3 py-2 text-left font-medium">Trace</th>
								<th className="px-3 py-2 text-left font-medium">Summary</th>
							</tr>
						</thead>
						<tbody>
							{logs.map((log) => (
								<tr key={log.id} className="align-top">
									<td className="px-3 py-2 text-slate-700">
										{new Date(log.created_at).toLocaleString()}
									</td>
									<td className="px-3 py-2 font-mono text-[11px] text-slate-500">
										{log.user_id ?? "n/a"}
									</td>
									<td className="px-3 py-2">
										<span
											className={`inline-flex rounded-full px-2 py-0.5 text-[11px] ${
												log.status === "success"
													? "bg-emerald-100 text-emerald-800"
													: "bg-amber-100 text-amber-800"
											}`}
										>
											{log.status}
										</span>
									</td>
									<td className="px-3 py-2 text-slate-700">
										{log.model ?? "n/a"}
									</td>
									<td className="px-3 py-2 text-slate-700">
										{log.cost_usd !== null && log.cost_usd !== undefined
											? log.cost_usd.toFixed(4)
											: "n/a"}
									</td>
									<td className="px-3 py-2 text-slate-700">
										{log.duration_seconds ?? "n/a"}
									</td>
									<td className="px-3 py-2 font-mono text-[11px] text-slate-500">
										{log.trace_id ?? "n/a"}
									</td>
									<td className="px-3 py-2 text-slate-700">
										{log.summary ?? "—"}
									</td>
								</tr>
							))}
							{!logs.length && (
								<tr>
									<td
										colSpan={8}
										className="px-3 py-4 text-center text-slate-500"
									>
										No logs available yet.
									</td>
								</tr>
							)}
						</tbody>
					</table>
				</div>
			</section>
		</main>
	);
}
```

### `src/app/admin/cv-tailor/page.tsx`
```tsx
"use client";

import { useEffect, useState } from "react";

import { apiRequest } from "@/lib/api-client";
import {
	type CvTailorLogEntry,
	cvTailorLogResponseSchema
} from "@/modules/observability/schemas";
import { useAttentionTracker } from "@/modules/observability/useAttentionTracker";

type CvTailorLogResponse = {
	items: CvTailorLogEntry[];
	total: number;
};

export default function CvTailorAdminPage() {
	const [logs, setLogs] = useState<CvTailorLogEntry[]>([]);
	const [selected, setSelected] = useState<CvTailorLogEntry | null>(null);
	const [error, setError] = useState<string | null>(null);

	useAttentionTracker({ moduleId: "admin-cv-tailor" });

	useEffect(() => {
		void loadLogs();
	}, []);

	const loadLogs = async () => {
		setError(null);
		try {
			const response = await apiRequest<CvTailorLogResponse>(
				"/cvtailor/logs?limit=50"
			);
			const parsed = cvTailorLogResponseSchema.parse(response);
			setLogs(parsed.items);
		} catch {
			setError("Failed to load CVTailor logs.");
		}
	};

	return (
		<main className="flex min-h-screen flex-col bg-slate-100 text-slate-900">
			<header className="border-b border-slate-200 bg-white px-6 py-4">
				<h1 className="text-2xl font-semibold">CVTailor observability</h1>
				<p className="mt-1 max-w-2xl text-sm text-slate-600">
					Internal dashboard for inspecting fallback chains and DOM validation
					guards used by the CVTailor Edge project. The backend is expected to
					persist JSON snapshots of these structures per session.
				</p>
			</header>
			<section className="flex flex-1 flex-col gap-4 px-6 py-4 lg:flex-row">
				<div className="w-full rounded-xl border border-slate-200 bg-white lg:w-1/2">
					{error && (
						<p className="px-4 pt-3 text-xs text-rose-600">
							{error}
						</p>
					)}
					<div className="overflow-auto px-4 pb-3 pt-2">
						<table className="min-w-full border-separate border-spacing-y-1 text-xs">
							<thead className="bg-slate-50 text-slate-600">
								<tr>
									<th className="px-3 py-2 text-left font-medium">Time</th>
									<th className="px-3 py-2 text-left font-medium">
										Session
									</th>
									<th className="px-3 py-2 text-left font-medium">Model</th>
									<th className="px-3 py-2 text-left font-medium">Cost</th>
									<th className="px-3 py-2 text-left font-medium">
										Fallback steps
									</th>
									<th className="px-3 py-2 text-left font-medium">
										Guards
									</th>
								</tr>
							</thead>
							<tbody>
								{logs.map((log) => (
									<tr
										key={log.id}
										className="cursor-pointer rounded-lg align-top hover:bg-slate-50"
										onClick={() => setSelected(log)}
									>
										<td className="px-3 py-2 text-slate-700">
											{new Date(log.created_at).toLocaleString()}
										</td>
										<td className="px-3 py-2 font-mono text-[11px] text-slate-500">
											{log.session_id}
										</td>
										<td className="px-3 py-2 text-slate-700">
											{log.model ?? "n/a"}
										</td>
										<td className="px-3 py-2 text-slate-700">
											{log.cost_usd !== null && log.cost_usd !== undefined
												? log.cost_usd.toFixed(4)
												: "n/a"}
										</td>
										<td className="px-3 py-2 text-slate-700">
											{log.fallback_chain.length}
										</td>
										<td className="px-3 py-2 text-slate-700">
											{log.validation_guards.length}
										</td>
									</tr>
								))}
								{!logs.length && (
									<tr>
										<td
											colSpan={6}
											className="px-3 py-4 text-center text-slate-500"
										>
											No CVTailor logs available yet.
										</td>
									</tr>
								)}
							</tbody>
						</table>
					</div>
				</div>

				<div className="w-full rounded-xl border border-slate-200 bg-white p-4 text-xs lg:w-1/2">
					<h2 className="text-sm font-semibold text-slate-900">
						Selected session details
					</h2>
					{!selected && (
						<p className="mt-2 text-slate-500">
							Select a row to inspect its fallback chain and validation guards.
						</p>
					)}
					{selected && (
						<div className="mt-3 space-y-3">
							<div>
								<p className="text-[11px] font-semibold text-slate-700">
									Fallback chain
								</p>
								<pre className="mt-1 max-h-40 overflow-auto rounded-md bg-slate-900 p-2 text-[10px] text-slate-100">
									{JSON.stringify(selected.fallback_chain, null, 2)}
								</pre>
							</div>
							<div>
								<p className="text-[11px] font-semibold text-slate-700">
									Validation guards
								</p>
								<pre className="mt-1 max-h-40 overflow-auto rounded-md bg-slate-900 p-2 text-[10px] text-slate-100">
									{JSON.stringify(selected.validation_guards, null, 2)}
								</pre>
							</div>
						</div>
					)}
				</div>
			</section>
		</main>
	);
}
```

### `src/app/admin/tickets/AdminTicketsClient.tsx`
```tsx
"use client";

import Link from "next/link";
import { useRouter } from "next/navigation";
import { useCallback, useEffect, useMemo, useRef, useState } from "react";

import { useI18n } from "@/app/i18n-provider";
import { ApiError, getApiErrorFields } from "@/lib/api-client";
import {
	getAdminTicket,
	getAdminTickets,
	replyToAdminTicket,
	type ActiveTicket,
	type SupportAttachment,
	type SupportHistoryEntry,
} from "@/lib/support-api";

const DEFAULT_LIMIT = 500;

type TicketMessage = {
	id: string;
	sender: "user" | "support";
	message: string;
	attachments: readonly SupportAttachment[];
};

type AdminTicketsClientProps = {
	selectedTicketId?: number;
};

function formatDateTime(value: string): string {
	const date = new Date(value);
	if (Number.isNaN(date.getTime())) {
		return value;
	}
	return date.toLocaleString();
}

function formatAdminTicketError(error: unknown): string {
	const fields = getApiErrorFields(error);
	if (!fields) {
		return "Unexpected error while loading support tickets.";
	}
	if (fields.status === 404) {
		return "Support ticket not found.";
	}
	if (fields.status === 401) {
		return "Authentication is required to access admin tickets.";
	}
	if (fields.status === 403) {
		return "You do not have permission to access admin tickets.";
	}
	if (fields.detailMessage?.trim()) {
		return fields.detailMessage;
	}
	return "Unexpected error while loading support tickets.";
}

function parseLegacyHistory(history: string | null): TicketMessage[] {
	if (!history) {
		return [];
	}
	try {
		const parsed = JSON.parse(history) as unknown;
		if (!Array.isArray(parsed)) {
			return [];
		}
		return parsed
			.map((entry, index): TicketMessage | null => {
				if (!entry || typeof entry !== "object") {
					return null;
				}
				const record = entry as Record<string, unknown>;
				if (typeof record.message !== "string") {
					return null;
				}
				const rawAttachments = Array.isArray(record.attachments)
					? record.attachments
					: [];
				const attachments = rawAttachments.filter(
					(item): item is SupportAttachment =>
						Boolean(item) &&
						typeof item === "object" &&
						typeof (item as SupportAttachment).url === "string",
				);
				return {
					id: `legacy-${index}`,
					sender: record.sender === "support" ? "support" : "user",
					message: record.message,
					attachments,
				};
			})
			.filter((entry): entry is TicketMessage => entry !== null);
	} catch {
		return history
			.split("\n")
			.map((line) => line.trim())
			.filter(Boolean)
			.map((line, index) => ({
				id: `legacy-line-${index}`,
				sender: /^support(?: team)?:/i.test(line) ? "support" : "user",
				message: line.replace(/^support(?: team)?:\s*/i, "").trim(),
				attachments: [],
			}));
	}
}

function buildConversation(ticket: ActiveTicket | null): TicketMessage[] {
	if (!ticket) {
		return [];
	}
	if (Array.isArray(ticket.history_entries) && ticket.history_entries.length > 0) {
		return ticket.history_entries.map(
			(entry: SupportHistoryEntry, index: number): TicketMessage => ({
				id: `entry-${index}`,
				sender: entry.sender === "support" ? "support" : "user",
				message: entry.message,
				attachments: Array.isArray(entry.attachments) ? entry.attachments : [],
			}),
		);
	}

	const parsedHistory = parseLegacyHistory(ticket.history);
	if (parsedHistory.length > 0) {
		return parsedHistory;
	}

	return [
		{
			id: "initial-message",
			sender: "user",
			message: ticket.message,
			attachments: [],
		},
	];
}

function AttachmentList({ attachments }: { attachments: readonly SupportAttachment[] }) {
	if (attachments.length === 0) {
		return null;
	}
	return (
		<ul className="mt-3 space-y-2">
			{attachments.map((attachment, index) => (
				<li key={`${attachment.url}-${index}`}>
					<a
						href={attachment.url}
						target="_blank"
						rel="noreferrer"
						className="text-xs font-medium text-sky-700 underline underline-offset-2"
					>
						{attachment.filename?.trim() || `Attachment ${index + 1}`}
					</a>
				</li>
			))}
		</ul>
	);
}

export default function AdminTicketsClient({
	selectedTicketId,
}: AdminTicketsClientProps) {
	const { t } = useI18n();
	const router = useRouter();
	const [tickets, setTickets] = useState<ActiveTicket[]>([]);
	const [ticketsLoading, setTicketsLoading] = useState(true);
	const [ticketsError, setTicketsError] = useState<string | null>(null);
	const [ticketDetail, setTicketDetail] = useState<ActiveTicket | null>(null);
	const [detailLoading, setDetailLoading] = useState(false);
	const [detailError, setDetailError] = useState<string | null>(null);
	const [replyMessage, setReplyMessage] = useState("");
	const [closeTicket, setCloseTicket] = useState(false);
	const [replyLoading, setReplyLoading] = useState(false);
	const [replyFeedback, setReplyFeedback] = useState<string | null>(null);
	const [replyError, setReplyError] = useState<string | null>(null);
	const latestDetailRequestRef = useRef(0);

	const loadTickets = useCallback(async (): Promise<ActiveTicket[]> => {
		setTicketsLoading(true);
		setTicketsError(null);
		try {
			const nextTickets = await getAdminTickets(DEFAULT_LIMIT);
			setTickets(nextTickets);
			return nextTickets;
		} catch (error) {
			setTickets([]);
			setTicketsError(formatAdminTicketError(error));
			return [];
		} finally {
			setTicketsLoading(false);
		}
	}, []);

	const loadTicketDetail = useCallback(async (ticketId: number): Promise<void> => {
		const requestId = ++latestDetailRequestRef.current;
		setDetailLoading(true);
		setDetailError(null);
		setReplyFeedback(null);
		try {
			const nextTicket = await getAdminTicket(ticketId);
			if (requestId !== latestDetailRequestRef.current) {
				return;
			}
			setTicketDetail(nextTicket);
		} catch (error) {
			if (requestId !== latestDetailRequestRef.current) {
				return;
			}
			setTicketDetail(null);
			setDetailError(formatAdminTicketError(error));
		} finally {
			if (requestId === latestDetailRequestRef.current) {
				setDetailLoading(false);
			}
		}
	}, []);

	useEffect(() => {
		void loadTickets();
	}, [loadTickets]);

	useEffect(() => {
		if (selectedTicketId === undefined) {
			setTicketDetail(null);
			setDetailError(null);
			setDetailLoading(false);
			return;
		}
		void loadTicketDetail(selectedTicketId);
	}, [loadTicketDetail, selectedTicketId]);

	useEffect(() => {
		setReplyMessage("");
		setCloseTicket(false);
		setReplyFeedback(null);
		setReplyError(null);
	}, [selectedTicketId]);

	const conversation = useMemo(() => buildConversation(ticketDetail), [ticketDetail]);

	const handleReplySubmit = useCallback(async (): Promise<void> => {
		if (selectedTicketId === undefined) {
			return;
		}
		const message = replyMessage.trim();
		if (!message) {
			setReplyError("Reply message is required.");
			return;
		}

		setReplyLoading(true);
		setReplyError(null);
		setReplyFeedback(null);

		try {
			await replyToAdminTicket(selectedTicketId, {
				message,
				close_ticket: closeTicket,
			});

			setReplyMessage("");
			setCloseTicket(false);
			setReplyFeedback(closeTicket ? "Reply sent and ticket closed." : "Reply sent.");

			const [nextTickets] = await Promise.all([
				loadTickets(),
				loadTicketDetail(selectedTicketId),
			]);

			if (
				selectedTicketId !== undefined &&
				nextTickets.length > 0 &&
				!nextTickets.some((ticket) => ticket.ticket_id === selectedTicketId)
			) {
				router.replace("/admin/tickets");
			}
		} catch (error) {
			setReplyError(formatAdminTicketError(error));
		} finally {
			setReplyLoading(false);
		}
	}, [closeTicket, loadTicketDetail, loadTickets, replyMessage, router, selectedTicketId]);

	return (
		<main className="flex min-h-screen flex-col bg-slate-100 text-slate-900">
			<header className="border-b border-slate-200 bg-white px-6 py-4">
				<h1 className="text-2xl font-semibold">Admin support tickets</h1>
				<p className="mt-1 max-w-3xl text-sm text-slate-600">
					Review active support tickets, inspect ticket history, and reply to users from the admin panel.
				</p>
			</header>

			<section className="flex flex-1 flex-col gap-4 px-6 py-4 lg:flex-row">
				<div className="w-full rounded-xl border border-slate-200 bg-white lg:w-[42%]">
					<div className="border-b border-slate-100 px-4 py-3">
						<h2 className="text-sm font-semibold text-slate-900">Recent tickets</h2>
						<p className="mt-1 text-xs text-slate-500">Newest active tickets only.</p>
					</div>

					{ticketsError ? (
						<p className="px-4 py-3 text-sm text-rose-600">{ticketsError}</p>
					) : null}

					{ticketsLoading ? (
						<p className="px-4 py-6 text-sm text-slate-500">Loading tickets...</p>
					) : null}

					{!ticketsLoading && !ticketsError && tickets.length === 0 ? (
						<p className="px-4 py-6 text-sm text-slate-500">No active tickets found.</p>
					) : null}

					{tickets.length > 0 ? (
						<ul className="divide-y divide-slate-100">
							{tickets.map((ticket) => {
								const isSelected = ticket.ticket_id === selectedTicketId;
								return (
									<li key={ticket.ticket_id}>
										<Link
											href={`/admin/tickets/${ticket.ticket_id}`}
											className={[
												"block px-4 py-4 transition hover:bg-slate-50",
												isSelected ? "bg-slate-50" : "",
											].join(" ")}
										>
											<div className="flex items-start justify-between gap-3">
												<div>
													<p className="text-sm font-semibold text-slate-900">
														Ticket #{ticket.ticket_id}
													</p>
													<p className="mt-1 text-xs text-slate-500">
														{formatDateTime(ticket.created_at)}
													</p>
												</div>
												<span
													className={[
														"inline-flex rounded-full px-2 py-1 text-[11px] font-medium",
														ticket.status === "active"
															? "bg-amber-100 text-amber-800"
															: "bg-slate-200 text-slate-700",
													].join(" ")}
												>
													{ticket.status}
												</span>
											</div>
											<div className="mt-3 space-y-1 text-xs text-slate-600">
												<p>
													<span className="font-semibold text-slate-800">Module:</span>{" "}
													{ticket.module}
												</p>
												<p>
													<span className="font-semibold text-slate-800">User:</span>{" "}
													{ticket.user_id}
												</p>
												<p className="line-clamp-3 whitespace-pre-wrap">
													<span className="font-semibold text-slate-800">Message:</span>{" "}
													{ticket.message}
												</p>
											</div>
										</Link>
									</li>
								);
							})}
						</ul>
					) : null}
				</div>

				<div className="w-full rounded-xl border border-slate-200 bg-white lg:w-[58%]">
					<div className="border-b border-slate-100 px-4 py-3">
						<h2 className="text-sm font-semibold text-slate-900">Ticket detail</h2>
					</div>

					{selectedTicketId === undefined ? (
						<p className="px-4 py-6 text-sm text-slate-500">
							Select a ticket to view its full conversation.
						</p>
					) : null}

					{selectedTicketId !== undefined && detailLoading ? (
						<p className="px-4 py-6 text-sm text-slate-500">Loading ticket detail...</p>
					) : null}

					{selectedTicketId !== undefined && !detailLoading && detailError ? (
						<p className="px-4 py-6 text-sm text-rose-600">{detailError}</p>
					) : null}

					{selectedTicketId !== undefined && !detailLoading && !detailError && ticketDetail ? (
						<div className="space-y-5 px-4 py-4">
							<div className="grid gap-3 rounded-xl border border-slate-200 bg-slate-50 p-4 text-sm text-slate-700 sm:grid-cols-2">
								<p>
									<span className="font-semibold text-slate-900">Ticket ID:</span>{" "}
									{ticketDetail.ticket_id}
								</p>
								<p>
									<span className="font-semibold text-slate-900">Created:</span>{" "}
									{formatDateTime(ticketDetail.created_at)}
								</p>
								<p>
									<span className="font-semibold text-slate-900">Module:</span>{" "}
									{ticketDetail.module}
								</p>
								<p>
									<span className="font-semibold text-slate-900">User ID:</span>{" "}
									{ticketDetail.user_id}
								</p>
								<p>
									<span className="font-semibold text-slate-900">Status:</span>{" "}
									{ticketDetail.status}
								</p>
							</div>

							<div>
								<h3 className="text-sm font-semibold text-slate-900">Conversation</h3>
								<ul className="mt-3 space-y-3">
									{conversation.map((entry) => (
										<li
											key={entry.id}
											className={[
												"rounded-xl p-4 text-sm",
												entry.sender === "user"
													? "border border-slate-200 bg-white"
													: "bg-slate-900 text-white",
											].join(" ")}
										>
											<p
												className={[
													"text-xs font-semibold uppercase tracking-wide",
													entry.sender === "user" ? "text-slate-500" : "text-slate-300",
												].join(" ")}
											>
												{entry.sender === "user" ? "User" : "Support"}
											</p>
											<p className="mt-2 whitespace-pre-wrap">{entry.message}</p>
											<AttachmentList attachments={entry.attachments} />
										</li>
									))}
								</ul>
							</div>

							<div className="border-t border-slate-100 pt-4">
								<h3 className="text-sm font-semibold text-slate-900">Reply</h3>
								{replyFeedback ? (
									<p className="mt-2 text-sm text-emerald-700">{replyFeedback}</p>
								) : null}
								{replyError ? (
									<p className="mt-2 text-sm text-rose-600">{replyError}</p>
								) : null}
								<textarea
									value={replyMessage}
									onChange={(event) => setReplyMessage(event.target.value)}
									disabled={replyLoading}
									rows={5}
									placeholder="Write your reply"
									aria-label="Admin reply message"
									className="mt-3 w-full rounded-xl border border-slate-300 px-3 py-2 text-sm text-slate-900 shadow-sm outline-none focus:border-slate-500"
								/>
								<label className="mt-3 flex items-center gap-2 text-sm text-slate-700">
									<input
										type="checkbox"
										checked={closeTicket}
										onChange={(event) => setCloseTicket(event.target.checked)}
										disabled={replyLoading}
									/>
									{t("admin.tickets.reply.closeAfterReply")}
								</label>
								<div className="mt-4 flex items-center gap-3">
									<button
										type="button"
										onClick={() => void handleReplySubmit()}
										disabled={replyLoading}
										className="rounded-full bg-slate-900 px-4 py-2 text-sm font-medium text-white disabled:opacity-60"
									>
										{replyLoading
											? t("admin.tickets.reply.sending")
											: t("admin.tickets.reply.send")}
									</button>
									<Link
										href="/admin/tickets"
										className="text-sm font-medium text-slate-600 underline underline-offset-2"
									>
										{t("admin.tickets.reply.backToList")}
									</Link>
								</div>
							</div>
						</div>
					) : null}
				</div>
			</section>
		</main>
	);
}
```

### `src/app/admin/tickets/page.tsx`
```tsx
import AdminTicketsClient from "@/app/admin/tickets/AdminTicketsClient";

export default function AdminTicketsPage() {
	return <AdminTicketsClient />;
}
```

### `src/app/admin/tickets/[ticketId]/page.tsx`
```tsx
import AdminTicketsClient from "@/app/admin/tickets/AdminTicketsClient";

type AdminTicketDetailPageProps = {
	params: Promise<{
		ticketId: string;
	}>;
};

export default async function AdminTicketDetailPage({
	params,
}: AdminTicketDetailPageProps) {
	const { ticketId } = await params;
	const parsedTicketId = Number(ticketId);

	return Number.isFinite(parsedTicketId) ? (
		<AdminTicketsClient selectedTicketId={parsedTicketId} />
	) : (
		<AdminTicketsClient />
	);
}
```

### `src/app/api/v1/genui/projects/route.ts`
```ts
import { NextResponse, type NextRequest } from "next/server";

import type { ProjectGridResponse, StaticProject } from "@/modules/genui/schemas";

export const dynamic = "force-dynamic";

const BACKEND_TIMEOUT_MS = 900;

const DEMO_PROJECTS: StaticProject[] =[
		{
				id: "pdf-anki",
				title: "KERA (Knowledge Extraction and Retention Architecture)",
				stack: "python",
				description:
						"RAG-style document extraction with flashcard generation, model fallback, and storage controls.",
				youtube_id: "_bYYlRWlrs4",
				tags:[
						"RAG", 
						"Semantic Caching", 
						"DSPy", 
						"Async Workers", 
						"FinOps", 
						"SOC2", 
						"ALE Encryption"
				],
		},
		{
				id: "receipt-parser",
				title: "VERA (Verified Expense and Receipt Architecture)",
				stack: "python",
				description:
						"Deterministic receipt validation with human review for math and OCR edge cases.",
				youtube_id: "V_RVnV5D_24",
				tags:[
						"Edge Compute", 
						"HITL", 
						"Deterministic Math", 
						"PII Scrubbing", 
						"RAG", 
						"SOC2", 
						"ALE Encryption"
				],
		},
		{
				id: "audio-notes",
				title: "AURA (Audio Understanding and Retention Architecture)",
				stack: "python",
				description:
						"Async audio transcription, summarization, feedback capture, and observability.",
				youtube_id: "1wK76ZHs2TM",
				tags:[
						"Idempotency", 
						"Nested Learning Loop", 
						"2PC Wallet", 
						"Redis Debouncing", 
						"RAG", 
						"SOC2", 
						"ALE Encryption"
				],
		},
		{
				id: "sre-qa-pipeline",
				title: "Zero-Compromise QA & SRE",
				stack: "fullstack",
				description:
						"Enterprise CI/CD pipeline featuring Mutation Testing, DeepEval (LLM-as-a-Judge), Chaos Engineering, and Adversarial RAG.",
				youtube_id: null, // Ready for your future generic video
				tags:[
						"Mutation Testing", 
						"LLM-as-a-Judge", 
						"Chaos Engineering", 
						"OpenTelemetry", 
						"Circuit Breakers"
				],
		},
		{
				id: "genui",
				title: "GenUI Proposal Stream",
				stack: "typescript",
				description:
						"Next.js component registry that renders backend-selected UI inside streamed proposals.",
				youtube_id: null,
				tags:[
						"Generative UI", 
						"Vercel Data Stream", 
						"React", 
						"TailwindCSS"
				],
		},
];

function filterProjects(stack: string | null): StaticProject[] {
		if (!stack || stack === "fullstack") {
				return DEMO_PROJECTS;
		}

		const matches = DEMO_PROJECTS.filter(
				(project) => project.stack === stack || project.stack === "fullstack"
		);

		return matches.length > 0 ? matches : DEMO_PROJECTS;
}

function resolveBackendApiBase(): string | null {
	const proxyTarget = process.env.BACKEND_PROXY_TARGET?.replace(/\/$/, "");
	if (proxyTarget) {
		return `${proxyTarget}/api/v1`;
	}

	const publicBackendUrl = process.env.NEXT_PUBLIC_BACKEND_URL?.trim();
	if (
		publicBackendUrl?.startsWith("http://") ||
		publicBackendUrl?.startsWith("https://")
	) {
		return publicBackendUrl.replace(/\/$/, "");
	}

	return null;
}

async function fetchUpstreamProjects(
	request: NextRequest
): Promise<Response | null> {
	const backendApiBase = resolveBackendApiBase();
	if (!backendApiBase) {
		return null;
	}

	const controller = new AbortController();
	const timeoutId = setTimeout(() => {
		controller.abort();
	}, BACKEND_TIMEOUT_MS);

	try {
		const response = await fetch(
			`${backendApiBase}/genui/projects${request.nextUrl.search}`,
			{
				cache: "no-store",
				headers: {
					Accept: "application/json",
					Cookie: request.headers.get("cookie") ?? "",
				},
				signal: controller.signal,
			}
		);

		if (!response.ok) {
			return null;
		}

		return new Response(response.body, {
			status: response.status,
			headers: {
				"Cache-Control": "no-store",
				"Content-Type":
					response.headers.get("content-type") ?? "application/json",
			},
		});
	} catch {
		return null;
	} finally {
		clearTimeout(timeoutId);
	}
}

export async function GET(request: NextRequest) {
	const upstream = await fetchUpstreamProjects(request);
	if (upstream) {
		return upstream;
	}

	const response: ProjectGridResponse = {
		projects: DEMO_PROJECTS,
		total: DEMO_PROJECTS.length,
	};

	return NextResponse.json(response, {
		headers: {
			"Cache-Control": "no-store",
		},
	});
}
```

### `src/app/api/v1/genui/proposal/generate/route.ts`
```ts
export const dynamic = "force-dynamic";

const BACKEND_TIMEOUT_MS = 30000;

function resolveClientIp(request: Request): string {
	const forwardedFor = request.headers.get("x-forwarded-for");
	const forwardedIp = forwardedFor?.split(",").at(0)?.trim();
	if (forwardedIp) {
		return forwardedIp;
	}

	const realIp = request.headers.get("x-real-ip")?.trim();
	return realIp || "anonymous-genui-user";
}

function resolveBackendApiBase(): string | null {
	const proxyTarget = process.env.BACKEND_PROXY_TARGET?.replace(/\/$/, "");
	if (proxyTarget) {
		return `${proxyTarget}/api/v1`;
	}

	const publicBackendUrl = process.env.NEXT_PUBLIC_BACKEND_URL?.trim();
	if (
		publicBackendUrl?.startsWith("http://") ||
		publicBackendUrl?.startsWith("https://")
	) {
		return publicBackendUrl.replace(/\/$/, "");
	}

	return null;
}

export async function POST(request: Request) {
	const backendApiBase = resolveBackendApiBase();
	if (!backendApiBase) {
		return new Response(
			JSON.stringify({
				detail:
					"GenUI backend is not configured. Set BACKEND_PROXY_TARGET to the FastAPI origin.",
			}),
			{
				status: 503,
				headers: {
					"Cache-Control": "no-store",
					"Content-Type": "application/json",
					"X-GenUI-Upstream": "next-proxy",
				},
			}
		);
	}

	const bodyText = await request.text();

	const headers = new Headers();
	headers.set("Content-Type", "application/json");

	const authorization = request.headers.get("authorization");
	const cookie = request.headers.get("cookie");
	const requestId = request.headers.get("x-request-id");
	if (authorization) {
		headers.set("Authorization", authorization);
	}
	if (cookie) {
		headers.set("Cookie", cookie);
	}
	if (requestId) {
		headers.set("X-Request-ID", requestId);
	}

	const clientIp = resolveClientIp(request);
	headers.set("X-Tenant-ID", `genui-${clientIp}`);

	const controller = new AbortController();
	const timeoutId = setTimeout(() => {
		controller.abort();
	}, BACKEND_TIMEOUT_MS);

	let upstream: Response;
	try {
		upstream = await fetch(`${backendApiBase}/genui/proposal/generate`, {
			body: bodyText,
			cache: "no-store",
			headers,
			method: "POST",
			signal: controller.signal,
		});
	} catch (error) {
		clearTimeout(timeoutId);
		const reason =
			error instanceof Error && error.name === "AbortError"
				? "GenUI backend connection timed out."
				: "GenUI backend is unreachable.";
		return new Response(JSON.stringify({ detail: reason }), {
			status: 502,
			headers: {
				"Cache-Control": "no-store",
				"Content-Type": "application/json",
				"X-GenUI-Upstream": "next-proxy",
			},
		});
	}

	clearTimeout(timeoutId);

	const upstreamMarker = upstream.headers.get("x-genui-upstream");
	const responseHeaders = new Headers();
	responseHeaders.set("Cache-Control", "no-store");
	responseHeaders.set(
		"Content-Type",
		upstream.headers.get("content-type") ??
			"application/x-ndjson; charset=utf-8"
	);
	responseHeaders.set("X-GenUI-Upstream", "next-proxy");
	if (upstreamMarker) {
		responseHeaders.set("X-GenUI-Upstream-Origin", upstreamMarker);
	}

	if (!upstream.ok) {
		const errorBody = await upstream.text();
		return new Response(errorBody, {
			status: upstream.status,
			headers: responseHeaders,
		});
	}

	if (!upstream.body) {
		return new Response(
			JSON.stringify({ detail: "GenUI backend returned an empty stream." }),
			{
				status: 502,
				headers: {
					"Cache-Control": "no-store",
					"Content-Type": "application/json",
					"X-GenUI-Upstream": "next-proxy",
				},
			}
		);
	}

	return new Response(upstream.body, {
		status: upstream.status,
		headers: responseHeaders,
	});
}
```

### `src/app/api/v1/pdf/upload/route.ts`
```ts
export const runtime = "nodejs";
export const dynamic = "force-dynamic";
export const maxDuration = 300; // 5 min — large PDFs need time to upload + encrypt + cache

const PDF_UPLOAD_TIMEOUT_MS = 290_000;

const HOP_BY_HOP_HEADERS = new Set([
	"connection",
	"keep-alive",
	"proxy-authenticate",
	"proxy-authorization",
	"te",
	"trailer",
	"transfer-encoding",
	"upgrade",
]);

function resolveBackendPdfBaseUrl(): string {
	const target = process.env.BACKEND_PROXY_TARGET?.replace(/\/$/, "");
	if (target) return `${target}/api/v1`;

	const pub = process.env.NEXT_PUBLIC_BACKEND_URL?.trim();
	if (pub && /^https?:\/\//i.test(pub)) return pub.replace(/\/$/, "");

	return "http://localhost:8000/api/v1";
}

function upstreamTimeoutSignal(ms: number): AbortSignal {
	if (typeof AbortSignal.timeout === "function") return AbortSignal.timeout(ms);
	const ctrl = new AbortController();
	setTimeout(() => ctrl.abort(), ms);
	return ctrl.signal;
}

function isAbortError(err: unknown): boolean {
	if (typeof DOMException !== "undefined" && err instanceof DOMException)
		return err.name === "AbortError";
	return !!err && typeof err === "object" && (err as { name?: unknown }).name === "AbortError";
}

export async function POST(request: Request): Promise<Response> {
	const url = `${resolveBackendPdfBaseUrl()}/pdf/upload`;

	let formData: FormData;
	try {
		formData = await request.formData();
	} catch (err) {
		console.error("[pdf/upload] failed to parse incoming FormData", err);
		return new Response(
			JSON.stringify({ detail: "Invalid multipart form data." }),
			{ status: 400, headers: { "Content-Type": "application/json" } },
		);
	}

	const authHeaders = new Headers();
	for (const key of ["authorization", "cookie", "accept-language", "x-user-id"]) {
		const val = request.headers.get(key);
		if (val) authHeaders.set(key, val);
	}

	try {
		const upstream = await fetch(url, {
			method: "POST",
			headers: authHeaders,
			body: formData, // fetch encodes this as proper multipart/form-data
			signal: upstreamTimeoutSignal(PDF_UPLOAD_TIMEOUT_MS),
		});

		const responseHeaders = new Headers();
		upstream.headers.forEach((value, key) => {
			if (!HOP_BY_HOP_HEADERS.has(key.toLowerCase())) {
				responseHeaders.set(key, value);
			}
		});

		const responseBody = await upstream.text();
		return new Response(responseBody, {
			status: upstream.status,
			headers: responseHeaders,
		});
	} catch (err) {
		if (isAbortError(err)) {
			return new Response(
				JSON.stringify({
					detail: "PDF upload timed out. Try a smaller file or check your connection.",
				}),
				{ status: 504, headers: { "Content-Type": "application/json" } },
			);
		}
		console.error("[pdf/upload] upstream proxy failed", err);
		return new Response(
			JSON.stringify({ detail: "Failed to proxy PDF upload to backend." }),
			{ status: 502, headers: { "Content-Type": "application/json" } },
		);
	}
}
```

### `src/app/api/v1/receipts/process/route.ts`
```ts
function resolveReceiptsProcessTimeoutMs(): number {
	const configured = process.env.RECEIPTS_PROCESS_TIMEOUT_MS;
	const parsed = configured ? Number(configured) : NaN;
	return Number.isFinite(parsed) && parsed > 0 ? parsed : 420_000;
}

const RECEIPTS_PROCESS_TIMEOUT_MS = resolveReceiptsProcessTimeoutMs();
const HOP_BY_HOP_HEADERS = new Set([
	"connection",
	"keep-alive",
	"proxy-authenticate",
	"proxy-authorization",
	"te",
	"trailer",
	"transfer-encoding",
	"upgrade",
]);

export const runtime = "nodejs";
export const dynamic = "force-dynamic";
export const maxDuration = 420;

function resolveBackendReceiptsBaseUrl(): string {
	const backendProxyTarget = process.env.BACKEND_PROXY_TARGET?.replace(/\/$/, "");
	if (backendProxyTarget) {
		return `${backendProxyTarget}/api/v1`;
	}

	const publicBackendUrl = process.env.NEXT_PUBLIC_BACKEND_URL?.trim();
	if (publicBackendUrl && /^https?:\/\//i.test(publicBackendUrl)) {
		return publicBackendUrl.replace(/\/$/, "");
	}

	return "http://localhost:8000/api/v1";
}

function forwardHeaders(headers: Headers): HeadersInit {
	const forwarded = new Headers();
	const keys = ["content-type", "authorization", "x-user-id", "cookie", "accept-language"];
	for (const key of keys) {
		const value = headers.get(key);
		if (value) {
			forwarded.set(key, value);
		}
	}
	return forwarded;
}

function upstreamTimeoutSignal(timeoutMs: number): AbortSignal {
	if (typeof AbortSignal !== "undefined" && typeof AbortSignal.timeout === "function") {
		return AbortSignal.timeout(timeoutMs);
	}
	const controller = new AbortController();
	setTimeout(() => controller.abort(), timeoutMs);
	return controller.signal;
}

function isAbortError(error: unknown): boolean {
	if (typeof DOMException !== "undefined" && error instanceof DOMException) {
		return error.name === "AbortError";
	}
	return !!error && typeof error === "object" && (error as { name?: unknown }).name === "AbortError";
}

export async function POST(request: Request): Promise<Response> {
	const url = `${resolveBackendReceiptsBaseUrl().replace(/\/$/, "")}/receipts/process`;
	const body = await request.text();
	const headers = forwardHeaders(request.headers);

	try {
		const upstream = await fetch(url, {
			method: "POST",
			headers,
			body,
			credentials: "include",
			signal: upstreamTimeoutSignal(RECEIPTS_PROCESS_TIMEOUT_MS),
		});

		const responseHeaders = new Headers();
		upstream.headers.forEach((value, key) => {
			if (!HOP_BY_HOP_HEADERS.has(key.toLowerCase())) {
				responseHeaders.set(key, value);
			}
		});

		const responseBody = await upstream.text();
		return new Response(responseBody, {
			status: upstream.status,
			headers: responseHeaders,
		});
	} catch (error) {
		if (isAbortError(error)) {
			return new Response(
				JSON.stringify({
					detail: "Receipt processing timed out.",
					error_message: "Receipt processing timed out.",
				}),
				{
					status: 504,
					headers: { "Content-Type": "application/json" },
				},
			);
		}

		console.error("[receipts/process] upstream proxy failed", error);
		return new Response(
			JSON.stringify({
				detail: "Failed to proxy receipt processing request.",
			}),
			{
				status: 502,
				headers: { "Content-Type": "application/json" },
			},
		);
	}
}
```

### `src/app/api/v1/telemetry/attention/route.ts`
```ts
export const dynamic = "force-dynamic";

const BACKEND_TIMEOUT_MS = 750;

function resolveBackendApiBase(): string | null {
	const proxyTarget = process.env.BACKEND_PROXY_TARGET?.replace(/\/$/, "");
	if (proxyTarget) {
		return `${proxyTarget}/api/v1`;
	}

	const publicBackendUrl = process.env.NEXT_PUBLIC_BACKEND_URL?.trim();
	if (
		publicBackendUrl?.startsWith("http://") ||
		publicBackendUrl?.startsWith("https://")
	) {
		return publicBackendUrl.replace(/\/$/, "");
	}

	return null;
}

async function forwardAttention(payload: string, request: Request): Promise<void> {
	const backendApiBase = resolveBackendApiBase();
	if (!backendApiBase) {
		return;
	}

	const controller = new AbortController();
	const timeoutId = setTimeout(() => {
		controller.abort();
	}, BACKEND_TIMEOUT_MS);

	try {
		await fetch(`${backendApiBase}/telemetry/attention`, {
			body: payload,
			cache: "no-store",
			headers: {
				"Content-Type": request.headers.get("content-type") ?? "application/json",
				Cookie: request.headers.get("cookie") ?? "",
			},
			method: "POST",
			signal: controller.signal,
		});
	} catch {

	} finally {
		clearTimeout(timeoutId);
	}
}

export async function POST(request: Request) {
	let payload = "";

	try {
		payload = await request.text();
	} catch {

	}

	if (payload) {
		await forwardAttention(payload, request);
	}

	return new Response(null, {
		status: 204,
		headers: {
			"Cache-Control": "no-store",
		},
	});
}
```

### `src/app/auth/auth-storage.ts`
```ts
/** Product scope for OAuth session/cookies (must match backend `module` claim). */
export type AuthModule = "receipt_parser" | "pdf_anki";

export function parseAuthModule(
	raw: string | null | undefined,
): AuthModule | null {
	if (raw === "receipt_parser" || raw === "pdf_anki") {
		return raw;
	}
	return null;
}
```

### `src/app/auth/backend-access-token.ts`
```ts
/**
 * Backend JWT from OAuth callback JSON (Bearer). HttpOnly cookies may also
 * apply; we persist the token for endpoints that require Authorization.
 */
export const BACKEND_ACCESS_TOKEN_STORAGE_KEY = "ff_backendaccess_token";

export const BACKEND_ACCESS_TOKEN_CHANGED_EVENT = "ff-backend-access-token-changed";

function isBrowser(): boolean {
	return typeof window !== "undefined" && typeof sessionStorage !== "undefined";
}

export function readBackendAccessToken(): string | null {
	if (!isBrowser()) return null;
	try {
		const raw = sessionStorage.getItem(BACKEND_ACCESS_TOKEN_STORAGE_KEY);
		if (!raw || raw.trim().length === 0) return null;
		return raw.trim();
	} catch {
		return null;
	}
}

function notifyTokenChanged(): void {
	if (!isBrowser()) return;
	window.dispatchEvent(new Event(BACKEND_ACCESS_TOKEN_CHANGED_EVENT));
}

export function writeBackendAccessToken(token: string): void {
	if (!isBrowser()) return;
	const t = token.trim();
	if (t.length === 0) return;
	try {
		sessionStorage.setItem(BACKEND_ACCESS_TOKEN_STORAGE_KEY, t);
		notifyTokenChanged();
	} catch {

	}
}

export function clearBackendAccessToken(): void {
	if (!isBrowser()) return;
	try {
		sessionStorage.removeItem(BACKEND_ACCESS_TOKEN_STORAGE_KEY);
		notifyTokenChanged();
	} catch {

	}
}
```

### `src/app/auth/backend-auth-api.ts`
```ts
import { apiRequest, apiErrorFromJsonBody } from "@/lib/api-client";
import type { AuthModule } from "@/app/auth/auth-storage";

export type AuthProvidersResponse = {
	/** Legacy alias for receipts (`module=receipt_parser`). */
	google?: string | null;
	google_receipt_parser?: string | null;
	google_pdf_anki?: string | null;
};

const BASE_URL =
	process.env.NEXT_PUBLIC_BACKEND_URL ?? "/api/v1";

export function getBackendUrl(path: string): string {
	const base = BASE_URL.replace(/\/$/, "");
	const normalized = path.startsWith("/") ? path : `/${path}`;
	return `${base}${normalized}`;
}

export type GoogleCallbackResponse = {
	access_token: string;
	token_type: "bearer" | string;
	user_id: string;
	email?: string;
	display_name?: string;
	tier?: string;
	expires_in?: number;
	/** OAuth product scope; required for multi-module backends. */
	module?: AuthModule | string;
};

/** Result of exchanging `?code=` with the backend (may require full-page redirect). */
export type GoogleCodeExchangeResult =
	| { kind: "session"; data: GoogleCallbackResponse }
	| { kind: "follow"; url: string };

export async function fetchAuthProviders(): Promise<AuthProvidersResponse> {
	return apiRequest<AuthProvidersResponse>("/auth/providers", {
		method: "GET",
	});
}

/**
 * Full Google authorize URL for the given product (`module` query on backend).
 */
export async function resolveGoogleOAuthUrl(module: AuthModule): Promise<string> {
	const providers = await fetchAuthProviders();
	if (module === "receipt_parser") {
		const url =
			providers.google_receipt_parser ?? providers.google ?? null;
		if (!url) {
			throw new Error(
				"Google receipts login is not configured (backend returned null).",
			);
		}
		return url;
	}
	const url = providers.google_pdf_anki ?? null;
	if (!url) {
		throw new Error(
			"Google PDF–Anki login is not configured (backend returned null).",
		);
	}
	return url;
}

const GOOGLE_CODE_EXCHANGE_TIMEOUT_MS = 60_000;

/**
 * Exchanges the OAuth `code` for a session JSON, or yields a URL the browser
 * must navigate to (302 consent step to `/auth/login/google?consent_pending=…`).
 * Uses `redirect: "manual"` so fetch does not follow Google (CORS); the tab
 * must perform the second OAuth leg via full navigation when needed.
 */
export async function exchangeGoogleCodeForSession(
	code: string,
	state: string | null,
	opts?: { signal?: AbortSignal; timeoutMs?: number },
): Promise<GoogleCodeExchangeResult> {
	const params = new URLSearchParams({
		code,
		...(state ? { state } : {}),
	});
	const url = getBackendUrl(`/auth/callback/google?${params.toString()}`);

	const controller = new AbortController();
	const timeoutMs = opts?.timeoutMs ?? GOOGLE_CODE_EXCHANGE_TIMEOUT_MS;
	const timeoutId = setTimeout(() => {
		controller.abort();
	}, timeoutMs);

	const external = opts?.signal;
	if (external) {
		if (external.aborted) controller.abort();
		else
			external.addEventListener("abort", () => {
				controller.abort();
			});
	}

	try {
		const response = await fetch(url, {
			method: "GET",
			redirect: "manual",
			signal: controller.signal,
			headers: { Accept: "application/json" },
			credentials: "include",
		});

		if (response.type === "opaqueredirect") {
			return { kind: "follow", url };
		}

		if (response.status >= 300 && response.status < 400) {
			const location = response.headers.get("Location");
			if (location) {
				return { kind: "follow", url: new URL(location, url).href };
			}
			return { kind: "follow", url };
		}

		if (!response.ok) {
			let rawBody: unknown;
			try {
				rawBody = await response.json();
			} catch {
				rawBody = null;
			}
			throw apiErrorFromJsonBody(response.status, rawBody);
		}

		const data = (await response.json()) as GoogleCallbackResponse;
		return { kind: "session", data };
	} finally {
		clearTimeout(timeoutId);
	}
}

export type AuthMeResponse = {
	user_id: string;
	email?: string;
	display_name?: string;
	tier?: string;
	/**
	 * PDF–Anki wallet balance in USD (backend may send a decimal string).
	 * Meaningful when `payg` / wallet mode is active.
	 */
	balance_usd?: string | number;
	/**
	 * When true, the user has PDF–Anki wallet (PAYG) enabled; `balance_usd` and
	 * optional unified model picker apply.
	 */
	payg?: boolean;
	/** Alias some backends may send for PDF–Anki wallet flag. */
	pdf_anki_payg?: boolean;
	auth_provider?: string;
	subscription_status?: string;
	stripe_subscription_id?: string | null; // <-- NUEVO
	stripe_customer_id?: string | null;     // <-- NUEVO
	expires_in?: number;
	/** JWT / profile product scope when returned by the backend. */
	module?: AuthModule | string;
	/**
	 * FIFO count of PDF–Anki flashcards persisted for this user (decrypted rows /
	 * business count). Omitted → frontend shows 0 as current usage.
	 */
	pdf_anki_flashcards_stored?: number;
	/**
	 * Effective storage ceiling for this user (must match worker/FIFO policy).
	 * If both this and `pdf_anki_max_decks` are sent, the frontend uses them for
	 * the chrome `current/max` display instead of hardcoded tier tables.
	 */
	pdf_anki_max_flashcards?: number;
	/** Pair of `pdf_anki_max_flashcards`; both required to override client caps. */
	pdf_anki_max_decks?: number;
	/**
	 * Effective unified LLM id for flashcard generation (server-resolved default).
	 */
	pdf_anki_flashcard_llm_model_id?: string | null;
	/**
	 * Allowlisted model ids for the picker; `null` → tier-fixed model only (no UI
	 * selector).
	 */
	pdf_anki_flashcard_llm_model_options?: string[] | null;
	/**
	 * Tier-included unified model when the user has wallet + premium/pro (for
	 * insufficient-balance hints).
	 */
	pdf_anki_tier_included_unified_model_id?: string | null;
	/** Worker pipeline kind for tier-fixed flows (display only). */
	pdf_anki_parse_pipeline_kind?: string | null;
	/**
	 * Receipt Parser: effective LiteLLM-style model id used to parse receipts
	 * for this tier (`module=receipt_parser` on `/auth/me`).
	 */
	receipt_parser_llm_model_id?: string | null;
	/**
	 * Receipt Parser: allowlisted model ids when the user may choose a model;
	 * `null` or empty → tier-fixed model only (no selector in UI until PATCH
	 * exists).
	 */
	receipt_parser_llm_model_options?: string[] | null;
	/** Receipt Parser: pipeline kind for tier-fixed flows (display only). */
	receipt_parser_parse_pipeline_kind?: string | null;
	/**
	 * Receipt Parser: wallet (PAYG) mode; use with `receipt_parser_balance_usd_display`.
	 * The UI preserves the incoming string as-is and may fall back to `balance_usd`
	 * when the backend exposes higher-precision raw wallet data there.
	 */
	receipt_parser_payg?: boolean | null;
	/**
	 * Receipt Parser: formatted USD balance for PAYG (e.g. two decimals); display as sent.
	 */
	receipt_parser_balance_usd_display?: string | null;
	/**
	 * Receipt Parser: the tier's included base model id (used as first option in PAYG model
	 * picker and as fallback when wallet balance is insufficient).
	 * Frontend infers from `tier` when absent: "premium" → gemini/gemini-2.5-flash-lite,
	 * "pro" → gemini/gemini-2.5-flash.
	 */
	receipt_parser_tier_base_model_id?: string | null;
	/** Google profile picture URL stored in the database as avatar_url. */
	avatar_url?: string | null;
};

/** True when PDF–Anki wallet mode is active (`payg` on `pdf_anki_users`). */
export function isPdfAnkiWalletUser(me: AuthMeResponse | null | undefined): boolean {
	if (me == null) {
		return false;
	}
	return me.payg === true || me.pdf_anki_payg === true;
}

export function formatPdfAnkiBalanceUsd(
	value: string | number | null | undefined,
): string | null {
	if (value === null || value === undefined || value === "") {
		return null;
	}
	if (typeof value === "string") {
		const trimmed = value.trim();
		if (trimmed === "") return null;
		const n = Number.parseFloat(trimmed);
		return Number.isFinite(n) ? String(n) : trimmed;
	}
	if (!Number.isFinite(value)) {
		return null;
	}
	return String(value);
}

export function getPdfAnkiTierBaseModelId(
	me: Pick<
		AuthMeResponse,
		"tier" | "pdf_anki_tier_included_unified_model_id"
	> | null | undefined,
): string | null {
	if (!me) {
		return null;
	}

	const explicit = me.pdf_anki_tier_included_unified_model_id?.trim();
	if (explicit) {
		return explicit;
	}

	const tier = me.tier?.trim().toLowerCase() ?? "";
	if (tier === "premium") {
		return "gemini/gemini-2.5-flash-lite";
	}
	if (tier === "pro") {
		return "gemini/gemini-2.5-flash";
	}
	return null;
}

/**
 * Validates PAYG balance display from `/auth/me?module=receipt_parser`.
 * Returns the incoming value unchanged, preserving every decimal digit that
 * arrives from the backend; otherwise null (defensive, no reformatting).
 */
export function parseReceiptParserBalanceUsdDisplay(
	raw: string | number | null | undefined,
): string | null {
	if (raw === null || raw === undefined) {
		return null;
	}
	if (typeof raw === "number") {
		return Number.isFinite(raw) && raw >= 0 ? String(raw) : null;
	}
	const s = raw.trim();
	if (s === "") {
		return null;
	}
	const n = Number.parseFloat(s);
	if (!Number.isFinite(n) || n < 0) {
		return null;
	}
	return String(n);
}

/**
 * Updates Receipt Parser LLM preference when the backend exposes a non-empty
 * `receipt_parser_llm_model_options` allowlist (PAYG Vertex selector).
 * Path/body align with `PATCH /auth/me/pdf-anki/unified-model`.
 */
export async function patchReceiptParserLlmModel(
	receiptParserLlmModelId: string,
	opts?: { timeoutMs?: number },
): Promise<AuthMeResponse> {
	return apiRequest<AuthMeResponse>("/auth/me/receipt-parser/unified-model", {
		method: "PATCH",
		body: { receipt_parser_llm_model_id: receiptParserLlmModelId },
		...(opts?.timeoutMs != null ? { timeoutMs: opts.timeoutMs } : {}),
	});
}

export async function patchPdfAnkiUnifiedModel(
	unifiedPdfModelId: string,
	opts?: { timeoutMs?: number },
): Promise<AuthMeResponse> {
	return apiRequest<AuthMeResponse>("/auth/me/pdf-anki/unified-model", {
		method: "PATCH",
		body: { unified_pdf_model_id: unifiedPdfModelId },
		...(opts?.timeoutMs != null ? { timeoutMs: opts.timeoutMs } : {}),
	});
}

export async function fetchAuthMe(
	module: AuthModule,
	opts?: { timeoutMs?: number },
): Promise<AuthMeResponse> {
	const qp = new URLSearchParams({ module });
	return apiRequest<AuthMeResponse>(`/auth/me?${qp.toString()}`, {
		method: "GET",
		...(opts?.timeoutMs != null ? { timeoutMs: opts.timeoutMs } : {}),
	});
}

export async function logoutAuthCookies(module: AuthModule): Promise<void> {
	const params = new URLSearchParams({ module });
	const url = getBackendUrl(`/auth/logout?${params.toString()}`);

	const response = await fetch(url, {
		method: "POST",
		credentials: "include",
	});

	if (response.ok) return;

	let rawBody: unknown;
	try {
		rawBody = await response.json();
	} catch {
		rawBody = null;
	}

	throw apiErrorFromJsonBody(response.status, rawBody);
}
```

### `src/app/auth/backend-auth-error.ts`
```ts
import { getApiErrorFields } from "@/lib/api-client";
import { ModuleAuthRequiredError } from "@/lib/module-auth-required-error";

/**
 * User-facing copy for OAuth/JWT errors (maps backend `detail.code` via i18n).
 */
export function resolveAuthApiUserMessage(
	error: unknown,
	t: (key: string) => string,
): string | null {
	if (error instanceof ModuleAuthRequiredError) {
		return t("auth.api.signInRequired");
	}
	const fields = getApiErrorFields(error);
	if (!fields) {
		return null;
	}
	if (fields.status !== 401 && fields.status !== 403) {
		return null;
	}
	const c = fields.detailCode;
	if (c === "AUTHENTICATION_REQUIRED") {
		return t("auth.api.signInRequired");
	}
	if (c === "INVALID_OR_EXPIRED_TOKEN") {
		return t("auth.api.sessionExpired");
	}
	if (c === "WRONG_PRODUCT_MODULE") {
		return t("auth.api.wrongProductModule");
	}
	if (fields.status === 401) {
		return t("auth.api.sessionExpired");
	}
	if (fields.status === 403) {

		return null;
	}
	return null;
}

/** Whether to clear the module session (storage + AuthProvider state). */
export function shouldResetSessionForApiError(error: unknown): boolean {
	const fields = getApiErrorFields(error);
	if (!fields) {
		return false;
	}
	if (fields.status === 401) {
		return true;
	}
	if (
		fields.status === 403 &&
		fields.detailCode === "WRONG_PRODUCT_MODULE"
	) {
		return true;
	}
	return false;
}

export function handleAuthApiErrorSideEffects(
	error: unknown,
	signOut: () => void | Promise<void>,
): void {
	if (shouldResetSessionForApiError(error)) {
		void signOut();
	}
}
```

### `src/app/auth/google-oauth-disclosure-storage.ts`
```ts
import type { AuthModule } from "@/app/auth/auth-storage";

const STORAGE_KEY_PREFIX = "jmmstudio_google_oauth_notice";

export function googleOAuthNoticeStorageKey(module: AuthModule): string {
	return `${STORAGE_KEY_PREFIX}:${module}`;
}

export function hasGoogleOAuthNoticeAck(module: AuthModule): boolean {
	if (typeof sessionStorage === "undefined") {
		return false;
	}
	return sessionStorage.getItem(googleOAuthNoticeStorageKey(module)) === "1";
}

export function setGoogleOAuthNoticeAck(module: AuthModule): void {
	if (typeof sessionStorage === "undefined") {
		return;
	}
	sessionStorage.setItem(googleOAuthNoticeStorageKey(module), "1");
}

export function getPublicPrivacyPolicyUrl(): string {
	const raw =
		typeof process !== "undefined"
			? process.env.NEXT_PUBLIC_PRIVACY_POLICY_URL ?? ""
			: "";
	return raw.trim();
}
```

### `src/app/auth/auth-code-error/page.tsx`
```tsx
"use client";

import Link from "next/link";

import { useI18n } from "@/app/i18n-provider";

export default function AuthCodeErrorPage() {
	const { t } = useI18n();

	return (
		<main className="mx-auto flex min-h-[50vh] max-w-md flex-col justify-center gap-4 px-6 py-12 text-center">
			<h1 className="text-lg font-semibold text-slate-900">
				{t("auth.codeError.title")}
			</h1>
			<p className="text-sm text-slate-600">
				{t("auth.codeError.body")}
			</p>
			<Link
				href="/"
				className="text-sm font-medium text-slate-800 underline"
			>
				{t("auth.codeError.homeLink")}
			</Link>
		</main>
	);
}
```

### `src/app/auth/callback/route.ts`
```ts
import { NextResponse } from "next/server";

import { createClient } from "@/lib/supabase/server";

/**
 * OAuth callback: exchanges ?code= for a session and sets Supabase cookies.
 */
export async function GET(request: Request): Promise<NextResponse> {
	const { searchParams, origin } = new URL(request.url);
	const code = searchParams.get("code");
	const next = searchParams.get("next") ?? "/";

	if (code) {
		const supabase = await createClient();
		const { error } = await supabase.auth.exchangeCodeForSession(code);
		if (!error) {
			return NextResponse.redirect(`${origin}${next}`);
		}
	}

	return NextResponse.redirect(`${origin}/auth/auth-code-error`);
}
```

### `src/app/auth/callback/google/page.tsx`
```tsx
"use client";

import { Suspense, useEffect, useState } from "react";
import { useRouter, useSearchParams } from "next/navigation";

import {
	exchangeGoogleCodeForSession,
	type GoogleCallbackResponse,
} from "@/app/auth/backend-auth-api";
import {
	parseAuthModule,
	type AuthModule,
} from "@/app/auth/auth-storage";
import { writeBackendAccessToken } from "@/app/auth/backend-access-token";

/** Fragment after `#`, e.g. `access_token=...&token_type=bearer&...`. */
function hashFragmentParams(hash: string): URLSearchParams | null {
	if (hash === "" || hash === "#") return null;
	const withoutHash = hash.startsWith("#") ? hash.slice(1) : hash;
	if (withoutHash === "") return null;
	return new URLSearchParams(withoutHash);
}

function postLoginPath(mod: AuthModule): string {
	return mod === "pdf_anki" ? "/pdf-anki" : "/receipts";
}

function GoogleCallbackInner() {
	const router = useRouter();
	const searchParams = useSearchParams();
	const [error, setError] = useState<string | null>(null);

	useEffect(() => {
		const hashParams =
			typeof window !== "undefined"
				? hashFragmentParams(window.location.hash)
				: null;
		const hashAccessToken = hashParams?.get("access_token") ?? null;

		if (hashAccessToken) {
			const moduleRaw = hashParams?.get("module");
			const mod = parseAuthModule(moduleRaw);

			if (!mod) {
				setError("Missing or invalid module in OAuth redirect.");
				return;
			}

			writeBackendAccessToken(hashAccessToken);

			const pathAndQuery =
				window.location.pathname + (window.location.search ?? "");
			window.history.replaceState(null, "", pathAndQuery);

			router.replace(postLoginPath(mod));
			return;
		}

		const code = searchParams.get("code");
		const state = searchParams.get("state");

		if (!code) {
			setError("Missing OAuth code.");
			return;
		}

		const run = async (): Promise<void> => {
			try {
				const result = await exchangeGoogleCodeForSession(code, state);

				if (result.kind === "follow") {
					window.location.assign(result.url);
					return;
				}

				const payload: GoogleCallbackResponse = result.data;
				const mod = parseAuthModule(payload.module);
				if (!mod) {
					setError("Missing module in OAuth response.");
					return;
				}

				if (payload.access_token?.trim()) {
					writeBackendAccessToken(payload.access_token);
				}

				router.replace(postLoginPath(mod));
			} catch (e) {
				console.error(e);
				router.replace("/auth/auth-code-error");
			}
		};

		void run();
	}, [router, searchParams]);

	if (error) {
		return (
			<main className="mx-auto flex min-h-[50vh] max-w-md flex-col justify-center gap-4 px-6 py-12 text-center">
				<h1 className="text-lg font-semibold text-slate-900">
					Sign-in could not be completed
				</h1>
				<p className="text-sm text-rose-600">{error}</p>
			</main>
		);
	}

	return (
		<main className="mx-auto flex min-h-[50vh] max-w-md flex-col justify-center gap-4 px-6 py-12 text-center">
			<p className="text-sm text-slate-600">Signing you in…</p>
		</main>
	);
}

export default function GoogleCallbackPage() {
	return (
		<Suspense
			fallback={
				<main className="mx-auto flex min-h-[50vh] max-w-md flex-col justify-center gap-4 px-6 py-12 text-center">
					<p className="text-sm text-slate-600">Signing you in…</p>
				</main>
			}
		>
			<GoogleCallbackInner />
		</Suspense>
	);
}
```

### `src/app/billing/billing-api.ts`
```ts
import { apiRequest } from "@/lib/api-client";

export type BillingInfo = {
	tier: "FREE" | "PREMIUM" | "PRO" | "PAYG";
	current_receipts: number;
	max_receipts: number | null;
	stripe_upgrade_url?: string | null;
};

export async function fetchBillingInfo(): Promise<BillingInfo> {
	try {
		return await apiRequest<BillingInfo>("/billing/me", {
			method: "GET"
		});
	} catch {

		return {
			tier: "FREE",
			current_receipts: 0,
			max_receipts: null,
			stripe_upgrade_url: null
		};
	}
}
```

### `src/app/billing/NavTierIndicator.tsx`
```tsx
"use client";

import { useEffect, useState } from "react";

import { fetchAuthMe, type AuthMeResponse } from "@/app/auth/backend-auth-api";
import { useI18n } from "@/app/i18n-provider";
import {
	fetchReceiptsUsage,
	type ReceiptsUsageResponse,
} from "@/app/receipts/receipt-usage-api";

function maxReceiptsCapacityForEffectiveTier(tier: string | undefined | null): number {
	const t = (tier ?? "").trim().toLowerCase();
	if (t === "premium") return 2_000;
	if (t === "free" || t === "") return 100;

	return 5_000;
}

function inferTierBaseModelForNav(tier: string | undefined | null): string | null {
	const t = (tier ?? "").trim().toLowerCase();
	if (t === "premium") return "gemini/gemini-2.5-flash-lite";
	if (t === "pro") return "gemini/gemini-2.5-flash";
	return null;
}

type NavTierIndicatorProps = {
	/** Called once when the tier data (or error) has resolved. */
	onReady?: () => void;
};

export function NavTierIndicator({ onReady }: NavTierIndicatorProps = {}) {
	const DEFAULT_MAX_RECEIPTS = 5000;
	const { t } = useI18n();
	const [me, setMe] = useState<AuthMeResponse | null>(null);
	const [usage, setUsage] = useState<ReceiptsUsageResponse | null>(null);
	const [error, setError] = useState<string | null>(null);
	const [open, setOpen] = useState(false);

	useEffect(() => {
		let cancelled = false;

		const load = async (): Promise<void> => {
			const devTier =
				typeof process !== "undefined"
					? process.env.NEXT_PUBLIC_RECEIPTS_DEV_TIER?.trim() ?? ""
					: "";

			try {
				const [meResult, usageResult] = await Promise.allSettled([
					fetchAuthMe("receipt_parser", { timeoutMs: 45_000 }).catch(() => null),
					fetchReceiptsUsage(),
				]);

				if (cancelled) return;

				const authMe =
					meResult.status === "fulfilled" ? meResult.value : null;
				if (meResult.status === "rejected") {
					console.error(meResult.reason);
				}

				if (usageResult.status === "fulfilled") {
					setMe(authMe);
					setUsage(usageResult.value);
					setError(null);
					onReady?.();
					return;
				}

				console.error(usageResult.reason);

				if (devTier) {
					setMe(authMe);
					setUsage({
						receipt_count: 0,
						max_receipts: DEFAULT_MAX_RECEIPTS,
						tier: devTier,
						at_80_percent: false,
					});
					setError(null);
					onReady?.();
					return;
				}

				setError("Tier unavailable");
				onReady?.();
			} catch (err) {
				console.error(err);
				if (!cancelled) {
					if (devTier) {
						setUsage({
							receipt_count: 0,
							max_receipts: DEFAULT_MAX_RECEIPTS,
							tier: devTier,
							at_80_percent: false,
						});
						setError(null);
					} else {
						setError("Tier unavailable");
					}
					onReady?.();
				}
			}
		};

		void load();

		const handleInvalidate = (): void => {
			if (!cancelled) void load();
		};
		window.addEventListener("receipts-usage-invalidated", handleInvalidate);

		return () => {
			cancelled = true;
			window.removeEventListener("receipts-usage-invalidated", handleInvalidate);
		};
	}, []);

	if (error) {
		return (
			<div className="flex items-center gap-2 text-xs">
				<span className="text-rose-500" aria-label="Tier unavailable">
					Tier: N/A
				</span>
				<span className="text-slate-500" aria-label="Usage">
					0/{DEFAULT_MAX_RECEIPTS} (0%)
				</span>
			</div>
		);
	}

	if (!usage) {
		return (
			<div className="flex items-center gap-2 text-xs text-slate-400">
				<span>Tier: ...</span>
				<span aria-label="Usage">0/{DEFAULT_MAX_RECEIPTS} (0%)</span>
			</div>
		);
	}

	const tier = me?.tier ?? usage.tier;
	const { receipt_count: current } = usage;

	const effectiveMax = (() => {
		if (!me || me.receipt_parser_payg !== true) {
			return usage.max_receipts;
		}
		const selectedModel = (me.receipt_parser_llm_model_id ?? "").trim();
		const tierBaseModel =
			me.receipt_parser_tier_base_model_id?.trim() ||
			inferTierBaseModelForNav(me.tier) ||
			"";

		if (selectedModel && tierBaseModel && selectedModel !== tierBaseModel) {
			return 5_000; // PAYG with non-subscription model -> PAYG limit
		}
		return maxReceiptsCapacityForEffectiveTier(me.tier);
	})();

	const max = effectiveMax;
	const showWarning =
		max !== null && max > 0 ? current / max >= 0.8 : usage.at_80_percent;

	let usageLabel = "";
	if (max !== null && max > 0) {
		const pct = Math.round((current / max) * 100);
		usageLabel = `${current}/${max} (${pct}%)`;
	}

	return (
		<div className="relative flex items-center gap-2 text-xs text-slate-800">
			<span className="font-semibold">{tier}</span>
			{usageLabel && (
				<span className="text-slate-500" aria-label="Usage">
					{usageLabel}
				</span>
			)}
			{showWarning ? (
				<>
					<button
						type="button"
						className="inline-flex h-5 w-5 items-center justify-center rounded-full text-amber-600 transition hover:bg-amber-100"
						aria-label={t("receipts.capacity.warningAria")}
						title={t("receipts.capacity.warningAria")}
						onClick={() => setOpen((value) => !value)}
						onMouseEnter={() => setOpen(true)}
						onMouseLeave={() => setOpen(false)}
					>
						<span aria-hidden="true">⚠</span>
					</button>
					{open ? (
						<div className="absolute right-0 top-7 z-50 w-80 rounded-2xl border border-amber-200 bg-amber-50 p-3 text-xs leading-5 text-amber-950 shadow-lg">
							{t("receipts.capacity.warningBody")}
						</div>
					) : null}
				</>
			) : null}
		</div>
	);
}
```

### `src/app/billing/transactions/layout.tsx`
```tsx
import type { Metadata } from "next";
import type { ReactNode } from "react";

export const metadata: Metadata = {
	title: { absolute: "PAYG Transactions" },
};

export default function TransactionsLayout({
	children,
}: {
	children: ReactNode;
}) {
	return children;
}
```

### `src/app/billing/transactions/page.tsx`
```tsx
"use client";

import { Suspense, useEffect, useMemo, useState } from "react";
import { useSearchParams } from "next/navigation";
import { Download } from "lucide-react";
import * as XLSX from "xlsx";

import { useI18n } from "@/app/i18n-provider";
import { apiRequest } from "@/lib/api-client";
import { PdfAnkiModuleChrome } from "@/app/pdf-anki/PdfAnkiModuleChrome";
import { ReceiptsModuleChrome } from "@/app/receipts/ReceiptsModuleChrome";

export type Transaction = {
	id: string;
	created_at: string;
	module: string;
	description: string;
	type: "credit" | "debit";
	amount_usd: string | number;
	balance_after_usd: string | number;
};

type TransactionsResponse = {
	transactions: Transaction[];
};

type TransactionsSource = "pdf_anki" | "receipt_parser";

const FIXED_MODULE_LABELS: Record<TransactionsSource, string> = {
	receipt_parser: "Vera",
	pdf_anki: "Kera",
};

function resolveTransactionsSource(value: string | null): TransactionsSource | null {
	if (value === "pdf_anki" || value === "receipt_parser") return value;
	return null;
}

function resolveInitialModuleFilter(source: TransactionsSource | null): string {
	return source ?? "all";
}

function formatUsd(value: string | number): string {
	const raw = typeof value === "string" ? value.trim() : String(value);
	return `$${raw}`;
}

function formatDate(iso: string): string {
	try {
		return new Date(iso).toLocaleString(undefined, {
			year: "numeric",
			month: "short",
			day: "numeric",
			hour: "2-digit",
			minute: "2-digit",
		});
	} catch {
		return iso;
	}
}

function TransactionsContent({
	source,
}: {
	source: TransactionsSource | null;
}) {
	const { t, locale } = useI18n();
	const [transactions, setTransactions] = useState<Transaction[]>([]);
	const [moduleFilter, setModuleFilter] = useState<string>(() =>
		resolveInitialModuleFilter(source)
	);
	const [loading, setLoading] = useState(true);
	const [error, setError] = useState<string | null>(null);
	const [exporting, setExporting] = useState(false);

	const resolveModuleLabel = (moduleName: string): string => {
		const normalized = moduleName.trim().toLowerCase();
		if (normalized === "pdf_anki") return FIXED_MODULE_LABELS.pdf_anki;
		if (normalized === "receipt_parser") {
			return FIXED_MODULE_LABELS.receipt_parser;
		}
		return moduleName;
	};

	const translateTransactionDescription = (tx: Transaction): string => {
		const raw = tx.description.trim();
		const normalized = raw.toLowerCase();
		const moduleId = tx.module.trim().toLowerCase();
		const moduleLabel = resolveModuleLabel(tx.module);

		if (
			moduleId === "receipt_parser" &&
			normalized.startsWith("payg wallet debit for receipt_parser") &&
			normalized.endsWith("processing")
		) {
			return t(
				"billing.transactions.description.paygWalletDebitForReceiptProcessing"
			);
		}

		if (
			moduleId === "pdf_anki" &&
			normalized.startsWith("payg wallet debit for pdf_anki") &&
			normalized.endsWith("processing")
		) {
			return t(
				"billing.transactions.description.paygWalletDebitForDocumentProcessing",
				{ module: moduleLabel }
			);
		}

		if (/^payg wallet credit from top-?up$/i.test(raw)) {
			return t("billing.transactions.description.paygWalletCreditFromTopUp");
		}

		if (/^payg wallet debit adjustment$/i.test(raw)) {
			return t("billing.transactions.description.paygWalletDebitAdjustment");
		}

		if (/^payg wallet credit adjustment$/i.test(raw)) {
			return t("billing.transactions.description.paygWalletCreditAdjustment");
		}

		if (
			normalized.startsWith("payg wallet debit for ") &&
			normalized.endsWith(" document processing")
		) {
			return t(
				"billing.transactions.description.paygWalletDebitForDocumentProcessing",
				{ module: moduleLabel }
			);
		}

		return raw;
	};

	const moduleOptions = useMemo(() => {
		return Array.from(
			new Set(transactions.map((tx) => tx.module).filter(Boolean))
		).sort((a, b) => a.localeCompare(b));
	}, [transactions]);

	const filteredTransactions = useMemo(() => {
		if (moduleFilter === "all") return transactions;
		return transactions.filter((tx) => tx.module === moduleFilter);
	}, [moduleFilter, transactions]);

	useEffect(() => {
		setModuleFilter(resolveInitialModuleFilter(source));
	}, [source]);

	useEffect(() => {
		let cancelled = false;
		setLoading(true);
		setError(null);

		apiRequest<TransactionsResponse>("/billing/transactions", { method: "GET" })
			.then((data) => {
				if (!cancelled) setTransactions(data.transactions ?? []);
			})
			.catch(() => {
				if (!cancelled) setError(t("billing.transactions.error"));
			})
			.finally(() => {
				if (!cancelled) setLoading(false);
			});

		return () => {
			cancelled = true;
		};
	}, [t]);

	const hasModuleFilter = moduleFilter !== "all";

	const handleExportXlsx = (): void => {
		setExporting(true);
		try {
			const rows = filteredTransactions.map((tx) => {
				const isCredit = tx.type === "credit";

				return {
					[t("billing.transactions.col.date")]: formatDate(tx.created_at),
					[t("billing.transactions.col.module")]: resolveModuleLabel(tx.module),
					[t("billing.transactions.col.description")]:
						translateTransactionDescription(tx),
					[t("billing.transactions.col.type")]: isCredit
						? t("billing.transactions.type.credit")
						: t("billing.transactions.type.debit"),
					[t("billing.transactions.col.amount")]: `${isCredit ? "+" : "-"}${formatUsd(tx.amount_usd)}`,
					[t("billing.transactions.col.balanceAfter")]: formatUsd(tx.balance_after_usd),
				};
			});

			const worksheet = XLSX.utils.json_to_sheet(rows);
			const workbook = XLSX.utils.book_new();
			XLSX.utils.book_append_sheet(workbook, worksheet, "Transactions");

			const suffix = moduleFilter === "all" ? "all" : moduleFilter;
			XLSX.writeFile(workbook, `payg-transactions-${suffix}-${locale}.xlsx`);
		} finally {
			setExporting(false);
		}
	};

	return (
		<main className="mx-auto max-w-5xl px-4 py-8">
			<div className="mb-6 flex items-center justify-between gap-4">
				<h1 className="text-xl font-semibold text-slate-900">
					{t("billing.transactions.title")}
				</h1>
				<button
					type="button"
					onClick={handleExportXlsx}
					disabled={exporting || filteredTransactions.length === 0}
					className="inline-flex items-center gap-1.5 rounded-md bg-emerald-600 px-3 py-1.5 text-xs font-medium text-white hover:bg-emerald-700 disabled:cursor-not-allowed disabled:opacity-60"
				>
					<Download className="h-3.5 w-3.5" aria-hidden />
					{t("billing.transactions.exportXlsx")}
				</button>
			</div>

			{!loading && !error && transactions.length > 0 && moduleOptions.length > 0 && (
				<div className="mb-4 flex flex-wrap items-center gap-3 rounded-lg border border-slate-200 bg-white px-4 py-3 text-sm">
					<label
						htmlFor="transactions-module-filter"
						className="font-medium text-slate-700"
					>
						{t("billing.transactions.filter.label")}
					</label>
					<select
						id="transactions-module-filter"
						value={moduleFilter}
						onChange={(e) => setModuleFilter(e.target.value)}
						className="min-w-0 rounded-md border border-slate-300 bg-white px-3 py-2 text-sm text-slate-800 shadow-sm outline-none transition focus:border-emerald-500 focus:ring-2 focus:ring-emerald-100"
					>
						<option value="all">{t("billing.transactions.filter.allModules")}</option>
						{moduleOptions.map((moduleName) => (
							<option key={moduleName} value={moduleName}>
								{resolveModuleLabel(moduleName)}
							</option>
						))}
					</select>
					<p className="text-xs text-slate-500">
						{hasModuleFilter
							? t("billing.transactions.filter.active")
							: t("billing.transactions.filter.helper")}
					</p>
				</div>
			)}

			{loading && (
				<p className="text-sm text-slate-500">{t("billing.transactions.loading")}</p>
			)}

			{!loading && error && <p className="text-sm text-rose-600">{error}</p>}

			{!loading && !error && transactions.length === 0 && (
				<p className="text-sm text-slate-500">{t("billing.transactions.empty")}</p>
			)}

			{!loading && !error && transactions.length > 0 && filteredTransactions.length === 0 && (
				<p className="text-sm text-slate-500">{t("billing.transactions.filteredEmpty")}</p>
			)}

			{!loading && !error && filteredTransactions.length > 0 && (
				<div className="overflow-x-auto rounded-lg border border-slate-200">
					<table className="w-full text-left text-xs">
						<thead>
							<tr className="border-b border-slate-200 bg-slate-50 text-slate-600">
								<th className="px-4 py-3 font-medium">{t("billing.transactions.col.date")}</th>
								<th className="px-4 py-3 font-medium">{t("billing.transactions.col.module")}</th>
								<th className="px-4 py-3 font-medium">
									{t("billing.transactions.col.description")}
								</th>
								<th className="px-4 py-3 font-medium">{t("billing.transactions.col.type")}</th>
								<th className="px-4 py-3 text-right font-medium">
									{t("billing.transactions.col.amount")}
								</th>
								<th className="px-4 py-3 text-right font-medium">
									{t("billing.transactions.col.balanceAfter")}
								</th>
							</tr>
						</thead>
						<tbody className="divide-y divide-slate-100">
							{filteredTransactions.map((tx) => {
								const isCredit = tx.type === "credit";
								return (
									<tr key={tx.id} className="bg-white hover:bg-slate-50">
										<td className="whitespace-nowrap px-4 py-3 text-slate-600">
											{formatDate(tx.created_at)}
										</td>
										<td className="px-4 py-3 text-slate-700">
											{resolveModuleLabel(tx.module)}
										</td>
										<td className="px-4 py-3 text-slate-700">
											{translateTransactionDescription(tx)}
										</td>
										<td className="px-4 py-3">
											<span
												className={`inline-flex items-center rounded-full px-2 py-0.5 text-[10px] font-medium ${
													isCredit
														? "bg-emerald-50 text-emerald-700"
														: "bg-rose-50 text-rose-700"
												}`}
											>
												{isCredit
													? t("billing.transactions.type.credit")
													: t("billing.transactions.type.debit")}
											</span>
										</td>
										<td
											className={`px-4 py-3 text-right font-mono font-medium ${
												isCredit ? "text-emerald-700" : "text-rose-700"
											}`}
										>
											{isCredit ? "+" : "-"}
											{formatUsd(tx.amount_usd)}
										</td>
										<td className="px-4 py-3 text-right font-mono text-slate-700">
											{formatUsd(tx.balance_after_usd)}
										</td>
									</tr>
								);
							})}
						</tbody>
					</table>
				</div>
			)}
		</main>
	);
}

function TransactionsPageContent() {
	const searchParams = useSearchParams();
	const source = resolveTransactionsSource(searchParams.get("source"));

	if (source === "pdf_anki") {
		return (
			<PdfAnkiModuleChrome>
				<TransactionsContent source={source} />
			</PdfAnkiModuleChrome>
		);
	}

	if (source === "receipt_parser") {
		return (
			<ReceiptsModuleChrome>
				<TransactionsContent source={source} />
			</ReceiptsModuleChrome>
		);
	}

	return <TransactionsContent source={null} />;
}

export default function TransactionsPage() {
	return (
		<Suspense fallback={null}>
			<TransactionsPageContent />
		</Suspense>
	);
}
```

### `src/app/genui/error.tsx`
```tsx
"use client";

type ErrorProps = {
	error: Error & { digest?: string };
	reset: () => void;
};

export default function GenUIError({ reset }: ErrorProps) {
	return (
		<main className="flex min-h-screen flex-col items-center justify-center bg-slate-100 px-6 text-center text-slate-900">
			<h1 className="text-2xl font-semibold">Something went wrong</h1>
			<p className="mt-3 max-w-md text-sm text-slate-600">
				The GenUI module failed while loading projects or streaming components.
				You can safely retry; no client data is persisted in the browser.
			</p>
			<button
				type="button"
				onClick={reset}
				className="mt-4 rounded-full bg-slate-900 px-4 py-2 text-xs font-medium text-slate-50"
			>
				Try again
			</button>
		</main>
	);
}
```

### `src/app/genui/loading.tsx`
```tsx
export default function GenUILoading() {
	return (
		<main className="flex min-h-screen flex-col bg-slate-100 text-slate-900">
			<header className="border-b border-slate-200 bg-white px-6 py-4">
				<div className="h-6 w-40 animate-pulse rounded-full bg-slate-200" />
				<div className="mt-2 h-4 w-80 animate-pulse rounded-full bg-slate-200" />
			</header>
			<section className="flex flex-1 flex-col gap-6 px-6 py-6 lg:flex-row">
				<div className="flex w-full flex-col gap-4 lg:w-1/2">
					<div className="h-4 w-32 animate-pulse rounded-full bg-slate-200" />
					<div className="grid gap-3 md:grid-cols-2">
						<div className="h-28 animate-pulse rounded-xl bg-slate-200" />
						<div className="h-28 animate-pulse rounded-xl bg-slate-200" />
					</div>
				</div>
				<div className="flex w-full flex-col gap-3 rounded-2xl border border-slate-200 bg-white p-4 shadow-sm lg:w-1/2">
					<div className="h-4 w-40 animate-pulse rounded-full bg-slate-200" />
					<div className="h-24 w-full animate-pulse rounded-xl bg-slate-100" />
					<div className="h-8 w-32 animate-pulse rounded-full bg-slate-200" />
				</div>
			</section>
		</main>
	);
}
```

### `src/app/genui/page.tsx`
```tsx
"use client";

import { useEffect, useState, useMemo } from "react";
import { PlayCircle, FilterX } from "lucide-react";

import { I18nProvider, useI18n } from "@/app/i18n-provider";
import { apiRequest } from "@/lib/api-client";
import {
	type ProjectGridResponse,
	type ProposalStreamChunk,
	type StaticProject,
	projectGridResponseSchema,
} from "@/modules/genui/schemas";
import { renderGenUIComponent } from "@/modules/genui/GenUIComponentRegistry";
import { streamProposal } from "@/modules/genui/streaming-client";
import { useAttentionTracker } from "@/modules/observability/useAttentionTracker";
import { VideoModal } from "@/components/VideoModal";
import { useGenUIQuota } from "@/modules/genui/useGenUIQuota";

const FILTER_ORDER = [
	"2PC Wallet",
	"ALE Encryption",
	"Async Workers",
	"Chaos Engineering",
	"Circuit Breakers",
	"DSPy",
	"Deterministic Math",
	"Edge Compute",
	"FinOps",
	"Generative UI",
	"HITL",
	"Idempotency",
	"LLM-as-a-Judge",
	"Mutation Testing",
	"Nested Learning Loop",
	"OpenTelemetry",
	"PII Scrubbing",
	"RAG",
	"React",
	"Redis Debouncing",
	"SOC2",
	"Semantic Caching",
	"TailwindCSS",
	"Vercel Data Stream",
	"fullstack",
	"python",
	"typescript",
] as const;

const FILTER_ORDER_INDEX = new Map<string, number>(
	FILTER_ORDER.map((tag, index) => [tag, index])
);

function GenUIPageContent() {
	const { t } = useI18n();
	const [allProjects, setAllProjects] = useState<StaticProject[] | null>(null);
	const [selectedTags, setSelectedTags] = useState<Set<string>>(new Set());

	const [companyContext, setCompanyContext] = useState<string>("");
	const [streamChunks, setStreamChunks] = useState<ProposalStreamChunk[]>([]);
	const[isStreaming, setIsStreaming] = useState<boolean>(false);
	const [error, setError] = useState<string | null>(null);
	const [mode, setMode] = useState<"guided" | "active">("guided");

	const { count: questionCount, isLocked, increment, lockOut, maxAttempts } = useGenUIQuota();

	const[activeVideoId, setActiveVideoId] = useState<string | null>(null);
	const [activeVideoTitle, setActiveVideoTitle] = useState<string>("");

	useAttentionTracker({ moduleId: "genui" });

	useEffect(() => {
		void loadProjects();
	},[]);

	const loadProjects = async () => {
		try {
			const response = await apiRequest<ProjectGridResponse>(`/genui/projects`);
			const parsed = projectGridResponseSchema.parse(response);
			setAllProjects(parsed.projects);
			setError(null);
		} catch {
			setError("Failed to load projects.");
		}
	};

	const availableTags = useMemo(() => {
		if (!allProjects) return[];
		const tags = new Set<string>();
		allProjects.forEach((p) => {
			tags.add(p.stack);
			p.tags?.forEach((t) => tags.add(t));
		});
		return Array.from(tags).sort((a, b) => {
			const aIndex = FILTER_ORDER_INDEX.get(a);
			const bIndex = FILTER_ORDER_INDEX.get(b);
			if (aIndex !== undefined && bIndex !== undefined) return aIndex - bIndex;
			if (aIndex !== undefined) return -1;
			if (bIndex !== undefined) return 1;
			return a.localeCompare(b);
		});
	}, [allProjects]);

	const filteredProjects = useMemo(() => {
		if (!allProjects) return[];
		if (selectedTags.size === 0) return allProjects;

		return allProjects.filter((p) => {
			const projectTags =[p.stack, ...(p.tags || [])];
			return projectTags.some((tag) => selectedTags.has(tag));
		});
	}, [allProjects, selectedTags]);

	const toggleTag = (tag: string) => {
		setSelectedTags((prev) => {
			const next = new Set(prev);
			if (next.has(tag)) next.delete(tag);
			else next.add(tag);
			return next;
		});
	};

	const clearTags = () => setSelectedTags(new Set());

	const handleGenerate = async () => {
		if (!companyContext.trim() || isLocked) return;

		increment();
		setStreamChunks([]);
		setIsStreaming(true);
		setError(null);

		try {
			for await (const chunk of streamProposal({
				context: companyContext,
				tags: Array.from(selectedTags), 
			})) {
				setStreamChunks((prev) =>[...prev, chunk]);
			}
		} catch (err) {
			if (err instanceof Error && err.message === "RATE_LIMIT_EXCEEDED") {
				lockOut();
				setError("Daily limit reached. Please try again tomorrow or book a meeting.");
			} else {
				setError("Streaming proposal failed.");
			}
		} finally {
			setIsStreaming(false);
		}
	};

	const handleBookMeeting = () => {
		const url = process.env.NEXT_PUBLIC_CALCOM_URL ?? "https://cal.com";
		window.open(url, "_blank", "noopener,noreferrer");
	};

	const handleAskSpecificQuestion = () => {
		if (isLocked) return;
		setMode("active");
	};

	const openVideo = (youtubeId: string, title: string) => {
		setActiveVideoId(youtubeId);
		setActiveVideoTitle(title);
	};

	return (
		<main className="flex min-h-screen flex-col bg-slate-100 text-slate-900">
			<header className="border-b border-slate-800/20 bg-slate-950 px-6 py-5 text-slate-50 shadow-sm">
				<div className="mx-auto flex w-full max-w-7xl flex-col gap-2">
					<p className="text-[11px] font-semibold uppercase tracking-[0.24em] text-indigo-300">
						Enterprise portfolio hub
					</p>
					<h1 className="text-2xl font-semibold">{t("genui.title")}</h1>
					<p className="max-w-2xl text-sm text-slate-300">
						{t("genui.subtitle")}
					</p>
				</div>
			</header>

			<section className="mx-auto flex w-full max-w-7xl flex-1 flex-col gap-6 px-6 py-6 lg:flex-row">
				{/* FAST PATH: Project Grid & Tags */}
				<div className="flex w-full flex-col gap-4 lg:w-[42%]">

					{/* Tags Filter Section */}
					<div className="flex flex-col gap-2">
						<div className="flex items-center justify-between">
							<span className="text-xs font-semibold text-slate-700">
								Filter by Architecture & Stack
							</span>
							{selectedTags.size > 0 && (
								<button
									onClick={clearTags}
									className="flex items-center gap-1 text-[10px] font-medium text-slate-500 hover:text-slate-800 transition"
								>
									<FilterX className="h-3 w-3" />
									Clear filters
								</button>
							)}
						</div>
						<div className="flex flex-wrap gap-1.5">
							{availableTags.map((tag) => {
								const isActive = selectedTags.has(tag);
								return (
									<button
										key={tag}
										onClick={() => toggleTag(tag)}
										className={`rounded-full px-2.5 py-1 text-[10px] font-medium transition-all duration-200 ${
											isActive
												? "bg-indigo-100 text-indigo-700 border border-indigo-200 shadow-sm"
												: "bg-white text-slate-600 border border-slate-200 hover:bg-slate-50 hover:border-slate-300"
										}`}
									>
										{tag}
									</button>
								);
							})}
							{!allProjects && (
								<span className="text-xs text-slate-400 animate-pulse">Loading tags...</span>
							)}
						</div>
					</div>

					{/* Projects Grid */}
					<div className="grid gap-3 md:grid-cols-2 mt-2">
						{filteredProjects.map((project) => (
							<article
								key={project.id}
								className="group relative flex flex-col rounded-2xl border border-slate-200 bg-white p-4 shadow-sm transition duration-200 hover:-translate-y-0.5 hover:border-indigo-200 hover:shadow-md"
							>
								<div className="flex items-start justify-between gap-2">
									<h2 className="text-sm font-semibold text-slate-900">
										{project.title}
									</h2>
									{project.youtube_id && (
										<button
											type="button"
											onClick={() => openVideo(project.youtube_id!, project.title)}
											className="flex shrink-0 items-center gap-1 rounded-full bg-indigo-50 px-2 py-1 text-[10px] font-semibold text-indigo-700 transition hover:bg-indigo-100"
											aria-label={`Watch demo for ${project.title}`}
										>
											<PlayCircle className="h-3 w-3" />
											Demo
										</button>
									)}
								</div>

								<p className="mt-1 text-[10px] uppercase tracking-wide text-slate-400">
									{project.stack}
								</p>

								{project.description && (
									<p className="mt-2 flex-1 text-xs leading-relaxed text-slate-600">
										{project.description}
									</p>
								)}

								{project.tags && project.tags.length > 0 && (
									<div className="mt-3 flex flex-wrap gap-1.5">
										{project.tags.map((tag) => (
											<span
												key={tag}
												className="rounded-md bg-slate-100 px-1.5 py-0.5 text-[9px] font-medium text-slate-600"
											>
												{tag}
											</span>
										))}
									</div>
								)}
							</article>
						))}
						{allProjects && filteredProjects.length === 0 && (
							<div className="col-span-2 rounded-xl border border-dashed border-slate-300 p-6 text-center">
								<p className="text-sm text-slate-500">No projects match the selected filters.</p>
								<button 
									onClick={clearTags}
									className="mt-2 text-xs font-medium text-indigo-600 hover:underline"
								>
									Clear filters
								</button>
							</div>
						)}
					</div>
				</div>

				{/* SLOW PATH: Generative UI */}
				<div className="flex w-full flex-col gap-3 rounded-2xl border border-slate-200 bg-white p-5 shadow-sm lg:w-[58%]">
					<div className="rounded-xl bg-slate-950 px-4 py-3 text-slate-50">
						<h2 className="text-sm font-semibold text-white">
							{t("genui.generator.title")}
						</h2>
						<p className="mt-1 text-xs text-slate-300">
							{t("genui.generator.subtitle")}
						</p>
					</div>

					<div className="flex flex-wrap gap-2 text-[11px]">
						<button
							type="button"
							onClick={handleBookMeeting}
							className="rounded-full bg-indigo-600 px-3 py-2 font-medium text-white shadow-sm transition hover:bg-indigo-500"
						>
							{t("genui.actions.bookMeeting")}
						</button>
						<button
							type="button"
							onClick={handleAskSpecificQuestion}
							disabled={isLocked}
							className="rounded-full border border-slate-300 bg-white px-3 py-2 text-slate-800 shadow-sm transition hover:border-indigo-300 hover:text-indigo-700 disabled:opacity-60"
						>
							{t("genui.actions.askSpecificQuestion")}
						</button>
					</div>

					{mode !== "guided" && (
						<textarea
							value={companyContext}
							onChange={(event) => setCompanyContext(event.target.value)}
							placeholder="Describe the client (industry, constraints, cloud stack, data sensitivity)..."
							className="mt-2 h-28 w-full rounded-xl border border-slate-200 bg-slate-50 p-3 text-xs text-slate-800 shadow-sm outline-none focus:border-indigo-400 disabled:opacity-60"
							disabled={isLocked}
						/>
					)}

					<div className="flex items-center gap-3">
						<button
							type="button"
							onClick={handleGenerate}
							disabled={isStreaming || !companyContext.trim() || mode !== "active" || isLocked}
							className="mt-1 self-start rounded-full bg-indigo-600 px-4 py-2 text-xs font-medium text-white shadow-sm transition hover:bg-indigo-500 disabled:opacity-60"
						>
							{isStreaming ? "Generating..." : "Generate proposal"}
						</button>
						<p className="text-[11px] text-slate-500">
							Questions used: {Math.min(questionCount, maxAttempts)}/{maxAttempts}
						</p>
					</div>

					{isLocked && (
						<p className="text-[11px] text-amber-700">
							Question limit reached. Let us continue this in a call.
						</p>
					)}
					{error && <p className="text-xs text-rose-600">{error}</p>}

					<div className="mt-2 flex-1 space-y-2 overflow-auto border-t border-slate-100 pt-3 text-xs">
						{streamChunks.map((chunk, index) => {
							if (chunk.type === "text") {
								return (
									<p key={index} className="animate-[fade-in_280ms_ease-out] text-slate-700">
										{chunk.content}
									</p>
								);
							}

							if (chunk.type === "component" && chunk.component) {
								const node = renderGenUIComponent(
									chunk.component,
									chunk.props,
									chunk.content
								);

								if (node) {
									return (
										<div
											key={index}
											className="animate-[fade-in_280ms_ease-out]"
										>
											{node}
										</div>
									);
								}
							}

							return (
								<p key={index} className="text-slate-500">
									{chunk.content}
								</p>
							);
						})}
						{!streamChunks.length && (
							<p className="text-xs text-slate-500">
								Generated components and narrative will appear here as the
								backend streams responses.
							</p>
						)}
					</div>
				</div>
			</section>

			{/* Video Modal */}
			<VideoModal
				isOpen={activeVideoId !== null}
				youtubeId={activeVideoId}
				title={activeVideoTitle}
				onClose={() => setActiveVideoId(null)}
			/>
		</main>
	);
}

export default function GenUIPage() {
	return (
		<I18nProvider forcedLocale="en" initialLocale="en" persistLocale={false}>
			<GenUIPageContent />
		</I18nProvider>
	);
}
```

### `src/app/modules/genui/domain/schemas.py`
```py
"""
GenUI domain: static project schemas and UI component schemas.

Fast Path: static project list. Slow Path: streaming UI components.
"""

from pydantic import BaseModel, Field

class StaticProjectSchema(BaseModel):
		"""Single project for Fast Path grid."""

		id: str
		title: str = Field(..., max_length=200)
		stack: str = Field(..., max_length=100)
		description: str | None = None

class ProjectGridResponse(BaseModel):
		"""Response for GET /genui/projects (Fast Path)."""

		projects: list[StaticProjectSchema]
		total: int

class ProposalStreamChunk(BaseModel):
		"""One chunk for Vercel AI SDK / Data Stream Protocol."""

		type: str = "text"  # text | component
		content: str = ""
		component: str | None = None  # e.g. FitBadge, DynamicProjectDescription
		props: dict[str, str | int | float] | None = None
```

### `src/app/modules/pdf_anki/domain/schemas.py`
```py
import { z } from "zod";

export const flashcardItemSchema = z.object({
	/** Plain text from API; do not decode as Base64 on the client. */
	front: z.string().min(1).max(50_000),
	back: z.string().min(1).max(100_000),
	tags: z.array(z.string()).max(50).default([])
});

/** Validates optional client-side upload metadata (`user_tier` must match worker). */
export const uploadRequestSchema = z.object({
	user_tier: z.enum([
		"free",
		"premium",
		"pro",
		"payg",
		"canceled",
		"admin",
	]),
});

export const documentStatusResponseSchema = z.object({
	task_id: z.string(),
	status: z.union([
		z.literal("pending"),
		z.literal("processing"),
		z.literal("completed"),
		z.literal("completed_partial"),
		z.literal("failed")
	]),
	file_hash: z.string().nullable().optional(),
	error_message: z.string().nullable().optional(),
	error_code: z.string().nullable().optional(),
	deck: z.array(flashcardItemSchema).nullable().optional(),
	deck_id: z.string().nullable().optional()
});

export const deckListItemSchema = z.object({
	id: z.string().uuid(),
	title: z.string(),
	status: z.string(),
	created_at: z.string(),
	updated_at: z.string().nullable().optional(),
	document_id: z.string().nullable().optional()
});

export const decksListResponseSchema = z.array(deckListItemSchema);

export const deckDetailResponseSchema = z.object({
	id: z.string().uuid(),
	title: z.string(),
	status: z.string(),
	created_at: z.string(),
	flashcards: z.array(flashcardItemSchema)
});

/** GET /pdf/deck/{id}/tags — returns an object containing the deck_id and tags array */
export const deckTagsResponseSchema = z.object({
	deck_id: z.string(),
	tags: z.array(z.string())
});

export const uploadAcceptedResponseSchema = z.object({
	task_id: z.string(),
	status: z.union([
		z.literal("pending"),
		z.literal("processing"),
		z.literal("completed")
	]),
	message: z.string(),
	already_processed: z.boolean().optional(),
	deck_id: z.string().nullable().optional()
});

export type FlashcardItem = z.infer<typeof flashcardItemSchema>;
export type DocumentStatusResponse = z.infer<
	typeof documentStatusResponseSchema
>;
export type DeckListItem = z.infer<typeof deckListItemSchema>;
export type DeckDetailResponse = z.infer<typeof deckDetailResponseSchema>;
export type UploadAcceptedResponse = z.infer<
	typeof uploadAcceptedResponseSchema
>;
export type UploadRequest = z.infer<typeof uploadRequestSchema>;
```

### `src/app/modules/receipt_parser/domain/schemas.py`
```py
"""
Strict Pydantic schemas for receipt extraction and validation.

Used for Edge OCR text input and LLM JSON output validation.
"""

from decimal import Decimal
from typing import Any

from pydantic import BaseModel, Field, field_validator

class ReceiptItemSchema(BaseModel):
		"""Single line item on a receipt."""

		description: str = Field(..., min_length=1, max_length=500)
		quantity: int = Field(1, ge=0)
		unit_price: Decimal = Field(..., ge=0)
		total: Decimal = Field(..., ge=0)

		@field_validator("unit_price", "total", mode="before")
		@classmethod
		def coerce_decimal(cls, v: Any) -> Decimal:
				if isinstance(v, (int, float)):
						return Decimal(str(v))
				if isinstance(v, str):
						return Decimal(v.strip().replace(",", "."))
				raise ValueError(f"Cannot coerce to Decimal: {type(v)}")

class ReceiptSchema(BaseModel):
		"""
		Full receipt with strict validation for math consistency.

		items total must match sum(item.total) and subtotal + tax must match total.
		"""

		items: list[ReceiptItemSchema] = Field(..., min_length=1)
		subtotal: Decimal = Field(..., ge=0)
		tax: Decimal = Field(Decimal("0"), ge=0)
		total: Decimal = Field(..., ge=0)
		currency: str = Field("USD", max_length=3)

		@field_validator("subtotal", "tax", "total", mode="before")
		@classmethod
		def _coerce_decimal(cls, v: Any) -> Decimal:
				if isinstance(v, (int, float)):
						return Decimal(str(v))
				if isinstance(v, str):
						return Decimal(v.strip().replace(",", "."))
				raise ValueError(f"Cannot coerce to Decimal: {type(v)}")

class ReceiptProcessRequest(BaseModel):
		"""Request body: raw OCR text from Edge."""

		raw_text: str = Field(..., min_length=1, max_length=50000)

class ReceiptProcessResponse(BaseModel):
		"""Response: extracted receipt or validation error."""

		status: str  # "verified" | "unprocessable"
		receipt: ReceiptSchema | None = None
		error_message: str | None = None
```

### `src/app/offline/page.tsx`
```tsx
"use client";

export default function OfflinePage() {
	return (
		<main className="flex min-h-screen flex-col items-center justify-center bg-slate-950 px-6 text-center text-slate-50">
			<h1 className="text-2xl font-semibold">You are offline</h1>
			<p className="mt-3 max-w-sm text-sm text-slate-300">
				This application is available as a Progressive Web App. Some features
				require connectivity, but you can continue capturing receipts and your
				data will sync when you are back online.
			</p>
		</main>
	);
}
```

### `src/app/pdf-anki/error.tsx`
```tsx
"use client";

import { useI18n } from "@/app/i18n-provider";

type ErrorProps = {
	error: Error & { digest?: string };
	reset: () => void;
};

export default function PdfAnkiError({ reset }: ErrorProps) {
	const { t } = useI18n();
	return (
		<main className="flex min-h-screen flex-col items-center justify-center bg-slate-100 px-6 text-center text-slate-900">
			<h1 className="text-2xl font-semibold">
				{t("pdfAnki.error.title")}
			</h1>
			<p className="mt-3 max-w-md text-sm text-slate-600">
				{t("pdfAnki.error.description")}
			</p>
			<button
				type="button"
				onClick={reset}
				className="mt-4 rounded-full bg-slate-900 px-4 py-2 text-xs font-medium text-slate-50"
			>
				{t("pdfAnki.error.retryCta")}
			</button>
		</main>
	);
}
```

### `src/app/pdf-anki/layout.tsx`
```tsx
import type { Metadata } from "next";
import type { ReactNode } from "react";

import { PdfAnkiModuleChrome } from "@/app/pdf-anki/PdfAnkiModuleChrome";

export const metadata: Metadata = {
	title: { absolute: "Kera" },
	icons: {
		icon: [{ url: "/icons/pdf-anki-icon-192.png", sizes: "192x192", type: "image/png" }],
	},
};

export default function PdfAnkiLayout({ children }: { children: ReactNode }) {
	return <PdfAnkiModuleChrome>{children}</PdfAnkiModuleChrome>;
}
```

### `src/app/pdf-anki/loading.tsx`
```tsx
export default function PdfAnkiLoading() {
	return (
		<main className="flex min-h-screen flex-col bg-slate-100 text-slate-900">
			<section className="mx-auto flex w-full max-w-5xl flex-1 flex-col gap-6 px-6 py-6 md:flex-row">
				<div className="flex-1">
					<div className="h-56 w-full animate-pulse rounded-2xl border border-dashed border-slate-200 bg-slate-50" />
				</div>
				<div className="flex-[1.2] rounded-2xl border border-slate-200 bg-white p-4 shadow-sm">
					<div className="h-4 w-40 animate-pulse rounded-full bg-slate-200" />
					<div className="mt-3 grid grid-cols-2 gap-3">
						<div className="h-32 animate-pulse rounded-xl bg-slate-100" />
						<div className="h-32 animate-pulse rounded-xl bg-slate-100" />
					</div>
				</div>
			</section>
		</main>
	);
}
```

### `src/app/pdf-anki/page.tsx`
```tsx
"use client";

import Link from "next/link";
import { useEffect, useMemo, useRef, useState } from "react";
import { AlertTriangle, Clock3, FileText } from "lucide-react";
import { ApiError, apiRequest } from "@/lib/api-client";
import {
	handleAuthApiErrorSideEffects,
	resolveAuthApiUserMessage,
} from "@/app/auth/backend-auth-error";
import { useAuthMe, useSupabaseAuth } from "@/app/AuthProvider";
import { useI18n } from "@/app/i18n-provider";
import {
	type DeckDetailResponse,
	type DocumentStatusResponse,
	type DeckListItem,
	type FlashcardItem,
	type UploadAcceptedResponse,
	deckDetailResponseSchema,
	decksListResponseSchema,
	documentStatusResponseSchema,
	uploadAcceptedResponseSchema,
} from "@/modules/pdf-anki/schemas";
import {
	PDF_ANKI_MAX_UPLOAD_SIZE_BYTES,
	PDF_ANKI_TEST_DEFAULT_USER_TIER,
	type PdfAnkiUserTier,
} from "@/modules/pdf-anki/constants";
import { mapAuthTierToPdfAnkiUploadTier } from "@/modules/pdf-anki/tierMapping";
import { pdfAnkiPageMinuteEstimates } from "@/modules/pdf-anki/processingEstimates";
import { PdfAnkiErrorAlert } from "@/modules/pdf-anki/PdfAnkiErrorAlert";
import { getUserFacingPdfError } from "@/modules/pdf-anki/userFacingPdfError";
import {
	formatPdfPageLimitErrorMessage,
	parsePdfPageLimitError,
} from "@/modules/pdf-anki/pdfPageLimitError";
import { usePdfAnkiResolvedUserId } from "@/modules/pdf-anki/usePdfAnkiAuth";
import { usePdfAnkiModelState } from "@/modules/pdf-anki/PdfAnkiModelStateContext";
import { PdfAnkiFlashcard } from "@/modules/pdf-anki/PdfAnkiFlashcard";
import { isPdfLikeFile } from "@/modules/pdf-anki/isPdfLikeFile";
import { useAttentionTracker } from "@/modules/observability/useAttentionTracker";
import { usePdfAnkiQuotaBump } from "@/modules/pdf-anki/PdfAnkiQuotaBumpContext";
import {
	getPdfAnkiPaygStorageCaps,
	getPdfAnkiStorageCapsFromMe,
	getPdfAnkiVisibleTierKind,
} from "@/modules/pdf-anki/pdfAnkiStorageCaps";
import { buildPdfAnkiInsufficientBalanceDetail } from "@/modules/pdf-anki/pdfAnkiInsufficientBalanceCopy";
import { IndeterminateProgressBar } from "@/components/IndeterminateProgressBar";
import { PdfAnkiDeckQueue } from "@/modules/pdf-anki/PdfAnkiDeckQueue";
import { PdfAnkiTaskSessionList } from "@/modules/pdf-anki/PdfAnkiTaskSessionList";
import {
	MAX_POLLS,
	POLL_INTERVAL_MS,
	buildDayKey,
	getInlineCopy,
	isIsoFromDay,
	isTaskCompletedLike,
	isTaskTerminal,
	localizePdfAnkiTierName,
	mergeQueueDecks,
	stripPdfExtension,
	type PreviewDeck,
	type QueueDeck,
	type UploadTask,
} from "@/modules/pdf-anki/multi-upload-helpers";

export default function PdfAnkiPage() {
	const { t, locale } = useI18n();
	const me = useAuthMe();
	const {
		refreshMe,
		signOut,
		loginWithGoogle,
		isLoading: authLoading,
	} = useSupabaseAuth();
	const { addParseFlashcardCount, bump } = usePdfAnkiQuotaBump();
	const {
		clearResolvedModelId,
		setResolvedModelId,
		setFallbackModelId,
		setPaygBalanceOverride,
		fallbackModelId,
	} = usePdfAnkiModelState();
	const { isAuthReady, userIdForApi } = usePdfAnkiResolvedUserId();
	useAttentionTracker({ moduleId: "pdf-anki" });

	const inlineCopy = useMemo(() => getInlineCopy(locale), [locale]);
	const [uploadTier, setUploadTier] = useState<PdfAnkiUserTier>(
		PDF_ANKI_TEST_DEFAULT_USER_TIER,
	);
	const [uploadError, setUploadError] = useState<string | null>(null);
	const [uploadErrorShowLoginCta, setUploadErrorShowLoginCta] =
		useState(false);
	const [uploadNoticeDeckId, setUploadNoticeDeckId] = useState<string | null>(null);
	const [uploadingCount, setUploadingCount] = useState(0);
	const [tasks, setTasks] = useState<Record<string, UploadTask>>({});
	const [todayDecks, setTodayDecks] = useState<QueueDeck[]>([]);
	const [previewDeck, setPreviewDeck] = useState<PreviewDeck | null>(null);
	const [previewError, setPreviewError] = useState<string | null>(null);
	const [isPreviewLoading, setIsPreviewLoading] = useState(false);
	const [todayDecksLoadError, setTodayDecksLoadError] = useState<string | null>(
		null,
	);
	const [dayKey, setDayKey] = useState(() => buildDayKey(new Date()));

	const fileInputRef = useRef<HTMLInputElement | null>(null);
	const tasksRef = useRef<Record<string, UploadTask>>({});
	const todayDecksRef = useRef<QueueDeck[]>([]);
	const dayKeyRef = useRef(dayKey);
	const isMountedRef = useRef(true);
	const pollTimeoutsRef = useRef<Map<string, ReturnType<typeof setTimeout>>>(
		new Map(),
	);
	const midnightTimeoutRef = useRef<number | null>(null);
	const pollAttemptsRef = useRef<Map<string, number>>(new Map());
	const pollAbortControllersRef = useRef<Map<string, AbortController>>(
		new Map(),
	);
	const handledTaskIdsRef = useRef<Set<string>>(new Set());
	const hydratedDeckIdsRef = useRef<Set<string>>(new Set());
	const hydratingDeckIdsRef = useRef<Set<string>>(new Set());
	const queueRefreshLoopTimeoutRef = useRef<ReturnType<typeof setTimeout> | null>(
		null,
	);
	const queueSyncTimeoutsRef = useRef<Map<string, ReturnType<typeof setTimeout>>>(
		new Map(),
	);
	const queueSyncAttemptsRef = useRef<Map<string, number>>(new Map());

	type TodayDeckSummary = {
		deckId: string;
		title: string;
		status: string;
		createdAt: string;
		updatedAt: string | null;
		documentId: string | null;
	};

	const setTasksWithRef = (
		updater: (
			previous: Record<string, UploadTask>,
		) => Record<string, UploadTask>,
	) => {
		const next = updater(tasksRef.current);
		tasksRef.current = next;
		setTasks(next);
	};

	const upsertTask = (
		taskId: string,
		updater: (previous: UploadTask | null) => UploadTask,
	) => {
		setTasksWithRef((previous) => ({
			...previous,
			[taskId]: updater(previous[taskId] ?? null),
		}));
	};

	const clearScheduledTaskPoll = (taskId: string) => {
		const timeoutId = pollTimeoutsRef.current.get(taskId);
		if (timeoutId) {
			clearTimeout(timeoutId);
			pollTimeoutsRef.current.delete(taskId);
		}
	};

	const abortTaskPollRequest = (taskId: string) => {
		const controller = pollAbortControllersRef.current.get(taskId);
		if (controller) {
			controller.abort();
			pollAbortControllersRef.current.delete(taskId);
		}
	};

	const clearTaskPolling = (taskId: string) => {
		clearScheduledTaskPoll(taskId);
		abortTaskPollRequest(taskId);
		pollAttemptsRef.current.delete(taskId);
	};

	useEffect(() => {
		dayKeyRef.current = dayKey;
		setTodayDecks((previous) =>
			previous.filter((deck) => isIsoFromDay(deck.createdAt, dayKey)),
		);
	}, [dayKey]);

	useEffect(() => {
		todayDecksRef.current = todayDecks;
	}, [todayDecks]);

	useEffect(() => {
		isMountedRef.current = true;
		return () => {
			isMountedRef.current = false;
			for (const timeoutId of pollTimeoutsRef.current.values()) {
				clearTimeout(timeoutId);
			}
			pollTimeoutsRef.current.clear();
			for (const controller of pollAbortControllersRef.current.values()) {
				controller.abort();
			}
			pollAbortControllersRef.current.clear();
			pollAttemptsRef.current.clear();
			if (queueRefreshLoopTimeoutRef.current) {
				clearTimeout(queueRefreshLoopTimeoutRef.current);
			}
			for (const timeoutId of queueSyncTimeoutsRef.current.values()) {
				clearTimeout(timeoutId);
			}
			queueSyncTimeoutsRef.current.clear();
			queueSyncAttemptsRef.current.clear();
			if (midnightTimeoutRef.current) {
				clearTimeout(midnightTimeoutRef.current);
			}
		};
	}, []);

	useEffect(() => {
		const scheduleNextReset = () => {
			const now = new Date();
			const nextMidnight = new Date(now);
			nextMidnight.setHours(24, 0, 0, 0);
			midnightTimeoutRef.current = window.setTimeout(() => {
				setDayKey(buildDayKey(new Date()));
				scheduleNextReset();
			}, nextMidnight.getTime() - now.getTime() + 1000);
		};

		scheduleNextReset();
		return () => {
			if (midnightTimeoutRef.current) {
				clearTimeout(midnightTimeoutRef.current);
			}
		};
	}, []);

	useEffect(() => {
		setUploadTier(
			mapAuthTierToPdfAnkiUploadTier(me?.tier, me?.subscription_status),
		);
	}, [me?.tier, me?.subscription_status]);

	const loadDeckDetail = async (
		deckId: string,
	): Promise<DeckDetailResponse | null> => {
		if (!userIdForApi) {
			return null;
		}
		const raw = await apiRequest<unknown>(`/pdf/deck/${deckId}`);
		return deckDetailResponseSchema.parse(raw);
	};

	const loadTodayDeckSummaries = async (): Promise<TodayDeckSummary[]> => {
		if (!isAuthReady || !userIdForApi) {
			setTodayDecksLoadError(null);
			return [];
		}

		const raw = await apiRequest<unknown>("/pdf/decks");
		const parsed = decksListResponseSchema.parse(raw);
		return parsed
			.filter((deck) => isIsoFromDay(deck.created_at, dayKeyRef.current))
			.map((deck: DeckListItem) => ({
				deckId: deck.id,
				title: deck.title,
				status: deck.status,
				createdAt: deck.created_at,
				updatedAt: deck.updated_at ?? null,
				documentId: deck.document_id ?? null,
			}));
	};

	const hydrateQueueDeck = async (
		deckId: string,
		options: {
			title: string;
			status: string;
			sourceTaskId: string | null;
			sourceFileName: string | null;
			fallbackCreatedAt?: string;
			initialFlashcards?: FlashcardItem[] | null;
		},
	) => {
		const hasInlineFlashcards =
			Array.isArray(options.initialFlashcards) &&
			options.initialFlashcards.length > 0;

		if (hasInlineFlashcards) {
			const flashcards = options.initialFlashcards ?? null;
			hydratedDeckIdsRef.current.add(deckId);
			setTodayDecks((previous) =>
				mergeQueueDecks(
					previous,
					[
						{
							deckId,
							title: options.title,
							status: options.status,
							createdAt: options.fallbackCreatedAt ?? new Date().toISOString(),
							updatedAt: options.fallbackCreatedAt ?? new Date().toISOString(),
							flashcards,
							sourceTaskId: options.sourceTaskId,
							sourceFileName: options.sourceFileName,
						},
					],
					dayKeyRef.current,
				),
			);
			return {
				deckId,
				title: options.title,
				status: options.status,
				createdAt: options.fallbackCreatedAt ?? new Date().toISOString(),
				flashcards,
				sourceTaskId: options.sourceTaskId,
			};
		}

		let detail: DeckDetailResponse | null = null;
		try {
			detail = await loadDeckDetail(deckId);
		} catch {
			detail = null;
		}
		if (detail && isMountedRef.current) {
			hydratedDeckIdsRef.current.add(deckId);
			setTodayDecks((previous) =>
				mergeQueueDecks(
					previous,
					[
						{
							deckId: detail.id,
							title: detail.title || options.title,
							status: detail.status || options.status,
							createdAt: detail.created_at,
							updatedAt: detail.created_at,
							flashcards: detail.flashcards,
							sourceTaskId: options.sourceTaskId,
							sourceFileName: options.sourceFileName,
						},
					],
					dayKeyRef.current,
				),
			);

			return {
				deckId: detail.id,
				title: detail.title || options.title,
				status: detail.status || options.status,
				createdAt: detail.created_at,
				flashcards: detail.flashcards,
				sourceTaskId: options.sourceTaskId,
			};
		}

		setTodayDecks((previous) =>
			mergeQueueDecks(
				previous,
				[
					{
						deckId,
						title: options.title,
						status: options.status,
						createdAt: options.fallbackCreatedAt ?? new Date().toISOString(),
						updatedAt: options.fallbackCreatedAt ?? new Date().toISOString(),
						flashcards: null,
						sourceTaskId: options.sourceTaskId,
						sourceFileName: options.sourceFileName,
					},
				],
				dayKeyRef.current,
			),
		);

		return {
			deckId,
			title: options.title,
			status: options.status,
			createdAt: options.fallbackCreatedAt ?? new Date().toISOString(),
			flashcards: null,
			sourceTaskId: options.sourceTaskId,
		};
	};

	const showDeckPreview = async (
		deckId: string,
		options?: {
			title?: string;
			status?: string;
			createdAt?: string;
			flashcards?: FlashcardItem[] | null;
			sourceTaskId?: string | null;
		},
	) => {
		setPreviewError(null);

		if ((options?.flashcards?.length ?? 0) > 0) {
			const previewFlashcards = options?.flashcards ?? [];
			setPreviewDeck({
				deckId,
				title: options?.title ?? stripPdfExtension(deckId),
				status: options?.status ?? "completed",
				createdAt: options?.createdAt ?? new Date().toISOString(),
				flashcards: previewFlashcards,
				sourceTaskId: options?.sourceTaskId ?? null,
			});
			return;
		}

		setPreviewDeck((current) => ({
			deckId,
			title: options?.title ?? current?.title ?? stripPdfExtension(deckId),
			status: options?.status ?? current?.status ?? "completed",
			createdAt:
				options?.createdAt ?? current?.createdAt ?? new Date().toISOString(),
			flashcards:
				current?.deckId === deckId ? current.flashcards : [],
			sourceTaskId: options?.sourceTaskId ?? current?.sourceTaskId ?? null,
		}));
		setIsPreviewLoading(true);
		try {
			const detail = await loadDeckDetail(deckId);
			if (!detail || !isMountedRef.current) {
				return;
			}
			hydratedDeckIdsRef.current.add(deckId);
			setTodayDecks((previous) =>
				mergeQueueDecks(
					previous,
					[
						{
							deckId: detail.id,
							title: detail.title,
							status: detail.status,
							createdAt: detail.created_at,
							updatedAt: detail.created_at,
							flashcards: detail.flashcards,
							sourceTaskId: options?.sourceTaskId ?? null,
							sourceFileName: null,
						},
					],
					dayKeyRef.current,
				),
			);
			setPreviewDeck({
				deckId: detail.id,
				title: detail.title,
				status: detail.status,
				createdAt: detail.created_at,
				flashcards: detail.flashcards,
				sourceTaskId: options?.sourceTaskId ?? null,
			});
		} catch {
			if (isMountedRef.current) {
				setPreviewError(t("pdfAnki.deck.errorLoadDetail"));
			}
		} finally {
			if (isMountedRef.current) {
				setIsPreviewLoading(false);
			}
		}
	};

	const refreshTodayDecks = async (): Promise<TodayDeckSummary[]> => {
		try {
			const summaries = await loadTodayDeckSummaries();
			const nextDecks: QueueDeck[] = summaries.map((deck) => ({
				deckId: deck.deckId,
				title: deck.title,
				status: deck.status,
				createdAt: deck.createdAt,
				updatedAt: deck.updatedAt,
				flashcards: null,
				sourceTaskId: deck.documentId,
				sourceFileName: null,
			}));

			setTodayDecks((previous) =>
				mergeQueueDecks(previous, nextDecks, dayKeyRef.current),
			);
			setTodayDecksLoadError(null);
			return summaries;
		} catch {
			setTodayDecksLoadError(t("pdfAnki.decks.errorLoad"));
			return [];
		}
	};

	useEffect(() => {
		void refreshTodayDecks();
	}, [isAuthReady, userIdForApi, dayKey]);

	useEffect(() => {
		if (previewDeck || todayDecks.length === 0) {
			return;
		}
		const latestDeck = todayDecks[0];
		if (!latestDeck) {
			return;
		}
		void showDeckPreview(latestDeck.deckId, {
			title: latestDeck.title,
			status: latestDeck.status,
			createdAt: latestDeck.createdAt,
			flashcards: latestDeck.flashcards,
			sourceTaskId: latestDeck.sourceTaskId,
		});
	}, [previewDeck, todayDecks]);

	useEffect(() => {
		const decksToHydrate = todayDecks.filter((deck) => {
			if (hydratedDeckIdsRef.current.has(deck.deckId)) {
				return false;
			}
			if (hydratingDeckIdsRef.current.has(deck.deckId)) {
				return false;
			}
			return deck.flashcards == null || deck.flashcards.length === 0;
		});

		for (const deck of decksToHydrate) {
			hydratingDeckIdsRef.current.add(deck.deckId);
			void (async () => {
				try {
					let detail: DeckDetailResponse | null = null;
					try {
						detail = await loadDeckDetail(deck.deckId);
					} catch {
						detail = null;
					}
					if (!detail || !isMountedRef.current) {
						return;
					}
					hydratedDeckIdsRef.current.add(deck.deckId);
					setTodayDecks((previous) =>
						mergeQueueDecks(
							previous,
							[
								{
									deckId: detail.id,
									title: detail.title || deck.title,
									status: detail.status || deck.status,
									createdAt: detail.created_at || deck.createdAt,
									updatedAt: detail.created_at,
									flashcards: detail.flashcards,
									sourceTaskId: deck.sourceTaskId,
									sourceFileName: deck.sourceFileName,
								},
							],
							dayKeyRef.current,
						),
					);
					setPreviewDeck((current) => {
						if (!current || current.deckId !== detail.id) {
							return current;
						}
						return {
							...current,
							title: detail.title,
							status: detail.status,
							createdAt: detail.created_at,
							flashcards: detail.flashcards,
						};
					});
				} finally {
					hydratingDeckIdsRef.current.delete(deck.deckId);
				}
			})();
		}
	}, [todayDecks, userIdForApi]);

	const scheduleQueueSyncRetry = (
		taskId: string,
		payload: DocumentStatusResponse,
		delayMs: number,
	) => {
		const existingTimeout = queueSyncTimeoutsRef.current.get(taskId);
		if (existingTimeout) {
			clearTimeout(existingTimeout);
		}
		const timeoutId = setTimeout(() => {
			void syncCompletedTaskIntoQueue(taskId, payload);
		}, delayMs);
		queueSyncTimeoutsRef.current.set(taskId, timeoutId);
	};

	const clearQueueSyncRetry = (taskId: string) => {
		const timeoutId = queueSyncTimeoutsRef.current.get(taskId);
		if (timeoutId) {
			clearTimeout(timeoutId);
			queueSyncTimeoutsRef.current.delete(taskId);
		}
		queueSyncAttemptsRef.current.delete(taskId);
	};

	const syncCompletedTaskIntoQueue = async (
		taskId: string,
		payload: DocumentStatusResponse,
	) => {
		if (!isMountedRef.current || !isTaskCompletedLike(payload.status)) {
			return;
		}

		const attempt = (queueSyncAttemptsRef.current.get(taskId) ?? 0) + 1;
		queueSyncAttemptsRef.current.set(taskId, attempt);

		const currentTask = tasksRef.current[taskId];
		const existingDeck = todayDecksRef.current.find((deck) => {
			if (payload.deck_id && deck.deckId === payload.deck_id) {
				return true;
			}
			return deck.sourceTaskId === taskId;
		});

		if (existingDeck) {
			clearQueueSyncRetry(taskId);
			void showDeckPreview(existingDeck.deckId, {
				title: existingDeck.title,
				status: existingDeck.status,
				createdAt: existingDeck.createdAt,
				flashcards: existingDeck.flashcards,
				sourceTaskId: existingDeck.sourceTaskId,
			});
			return;
		}

		let resolvedDeck = null as
			| {
					deckId: string;
					title: string;
					status: string;
					createdAt: string;
					flashcards: FlashcardItem[] | null;
					sourceTaskId: string | null;
			  }
			| null;

		if (payload.deck_id) {
			const title = stripPdfExtension(currentTask?.fileName ?? payload.deck_id);
			resolvedDeck = await hydrateQueueDeck(payload.deck_id, {
				title,
				status: payload.status,
				sourceTaskId: taskId,
				sourceFileName: currentTask?.fileName ?? null,
				fallbackCreatedAt: new Date().toISOString(),
				initialFlashcards: payload.deck ?? null,
			});
		} else {
			const summaries = await refreshTodayDecks();
			const matchedSummary = summaries.find((summary) => {
				if (summary.documentId === taskId) {
					return true;
				}
				return false;
			});

			if (matchedSummary) {
				resolvedDeck = await hydrateQueueDeck(matchedSummary.deckId, {
					title: matchedSummary.title || stripPdfExtension(currentTask?.fileName ?? taskId),
					status: matchedSummary.status || payload.status,
					sourceTaskId: matchedSummary.documentId ?? taskId,
					sourceFileName: currentTask?.fileName ?? null,
					fallbackCreatedAt: matchedSummary.createdAt,
					initialFlashcards: payload.deck ?? null,
				});
			}
		}

		if (resolvedDeck && isMountedRef.current) {
			clearQueueSyncRetry(taskId);
			await showDeckPreview(resolvedDeck.deckId, {
				title: resolvedDeck.title,
				status: resolvedDeck.status,
				createdAt: resolvedDeck.createdAt,
				flashcards: resolvedDeck.flashcards,
				sourceTaskId: resolvedDeck.sourceTaskId,
			});
			return;
		}

		if (attempt < 6) {
			scheduleQueueSyncRetry(taskId, payload, Math.min(500 * attempt, 2500));
			return;
		}

		clearQueueSyncRetry(taskId);
	};

	const reconcileTasksWithDeckSummaries = async (
		summaries: TodayDeckSummary[],
	) => {
		const completedSummaries = summaries.filter(
			(summary) =>
				Boolean(summary.documentId) && isTaskCompletedLike(summary.status),
		);

		for (const summary of completedSummaries) {
			const documentId = summary.documentId;
			if (!documentId) {
				continue;
			}

			const currentTask = tasksRef.current[documentId];
			if (!currentTask) {
				continue;
			}

			if (!isTaskCompletedLike(currentTask.status)) {
				upsertTask(documentId, (current) => ({
					...(current as UploadTask),
					status:
						summary.status === "completed_partial"
							? "completed_partial"
							: "completed",
					deckId: summary.deckId,
					updatedAt:
						summary.updatedAt ?? summary.createdAt ?? new Date().toISOString(),
				}));
				clearTaskPolling(documentId);
			}

			await syncCompletedTaskIntoQueue(documentId, {
				task_id: documentId,
				status:
					summary.status === "completed_partial"
						? "completed_partial"
						: "completed",
				file_hash: null,
				error_message: null,
				error_code: null,
				resolved_model_id: null,
				deck: null,
				deck_id: summary.deckId,
			});
		}
	};

	const scheduleTaskPoll = (taskId: string, delayMs: number = POLL_INTERVAL_MS) => {
		clearScheduledTaskPoll(taskId);
		const timeoutId = setTimeout(() => {
			void pollTask(taskId);
		}, delayMs);
		pollTimeoutsRef.current.set(taskId, timeoutId);
	};

	const applyTaskPayloadSideEffects = (
		taskId: string,
		payload: DocumentStatusResponse,
	) => {
		if (payload.payg_wallet_balance_usd != null) {
			setPaygBalanceOverride(payload.payg_wallet_balance_usd);
			void refreshMe();
		}
		if (payload.payg_fallback_model_id != null) {
			setFallbackModelId(payload.payg_fallback_model_id);
			void refreshMe();
		}

		if (!handledTaskIdsRef.current.has(taskId) && isTaskTerminal(payload.status)) {
			handledTaskIdsRef.current.add(taskId);
			setResolvedModelId(payload.resolved_model_id ?? null);
			if (isTaskCompletedLike(payload.status) && (payload.deck?.length ?? 0) > 0) {
				addParseFlashcardCount(payload.deck?.length ?? 0);
			}
			void refreshMe();
		}

		if (!isTaskCompletedLike(payload.status)) {
			return;
		}

		void syncCompletedTaskIntoQueue(taskId, payload);
	};

	const pollTask = async (taskId: string) => {
		clearScheduledTaskPoll(taskId);
		const currentTask = tasksRef.current[taskId];
		if (!isMountedRef.current || !currentTask) {
			clearTaskPolling(taskId);
			return;
		}
		if (isTaskTerminal(currentTask.status)) {
			clearTaskPolling(taskId);
			return;
		}

		const nextAttempt = (pollAttemptsRef.current.get(taskId) ?? 0) + 1;
		pollAttemptsRef.current.set(taskId, nextAttempt);
		if (nextAttempt > MAX_POLLS) {
			upsertTask(taskId, (current) => ({
				...(current as UploadTask),
				status: "error",
				pollingError: "max_attempts",
				updatedAt: new Date().toISOString(),
			}));
			clearTaskPolling(taskId);
			return;
		}

		const controller = new AbortController();
		pollAbortControllersRef.current.set(taskId, controller);

		try {
			const raw = await apiRequest<unknown>(`/pdf/status/${taskId}`, {
				signal: controller.signal,
			});
			if (!isMountedRef.current) {
				return;
			}

			const parsed = documentStatusResponseSchema.parse(raw);
			upsertTask(taskId, (current) => ({
				...(current as UploadTask),
				status: parsed.status,
				deckId: parsed.deck_id ?? current?.deckId ?? null,
				deck: parsed.deck ?? current?.deck ?? null,
				errorMessage: parsed.error_message ?? null,
				errorCode: parsed.error_code ?? null,
				pollingError: null,
				resolvedModelId: parsed.resolved_model_id ?? null,
				updatedAt: new Date().toISOString(),
			}));
			applyTaskPayloadSideEffects(taskId, parsed);

			if (isTaskTerminal(parsed.status)) {
				clearTaskPolling(taskId);
				return;
			}
			scheduleTaskPoll(taskId);
		} catch {
			if (controller.signal.aborted || !isMountedRef.current) {
				return;
			}
			const latestTask = tasksRef.current[taskId];
			if (!latestTask || isTaskTerminal(latestTask.status)) {
				clearTaskPolling(taskId);
				return;
			}
			upsertTask(taskId, (current) => ({
				...(current as UploadTask),
				status: current?.status === "pending" ? "pending" : "processing",
				pollingError: null,
				updatedAt: new Date().toISOString(),
			}));

			scheduleTaskPoll(taskId);
		} finally {
			if (pollAbortControllersRef.current.get(taskId) === controller) {
				pollAbortControllersRef.current.delete(taskId);
			}
		}
	};

	const handleBrowseClick = () => {
		if (!isAuthReady) {
			return;
		}
		fileInputRef.current?.click();
	};

	const uploadFile = async (file: File) => {
		if (!isAuthReady) {
			setUploadError(t("pdfAnki.main.error.signInRequired"));
			setUploadErrorShowLoginCta(true);
			return;
		}
		if (file.size > PDF_ANKI_MAX_UPLOAD_SIZE_BYTES) {
			const maxMb = Math.round(PDF_ANKI_MAX_UPLOAD_SIZE_BYTES / (1024 * 1024));
			setUploadError(
				t("pdfAnki.main.error.fileTooLarge", { maxMb: String(maxMb) }),
			);
			return;
		}

		setUploadError(null);
		setUploadErrorShowLoginCta(false);
		setUploadNoticeDeckId(null);
		setUploadingCount((current) => current + 1);

		const formData = new FormData();
		formData.append("file", file);
		formData.append("title", file.name || "");

		const modelOptions = me?.pdf_anki_flashcard_llm_model_options;
		const unifiedId = me?.pdf_anki_flashcard_llm_model_id?.trim();
		if (
			Array.isArray(modelOptions) &&
			unifiedId &&
			modelOptions.includes(unifiedId)
		) {
			formData.append("pdf_anki_unified_model_id", unifiedId);
		}

		try {
			const json = await apiRequest<UploadAcceptedResponse>("/pdf/upload", {
				method: "POST",
				body: formData,
				retries: 0,
				timeoutMs: 300_000,
			});
			const parsed = uploadAcceptedResponseSchema.parse(json);

			if (parsed.payg_wallet_balance_usd != null) {
				setPaygBalanceOverride(parsed.payg_wallet_balance_usd);
				void refreshMe();
			}
			if (parsed.payg_fallback_model_id != null) {
				setFallbackModelId(parsed.payg_fallback_model_id);
				void refreshMe();
			}

			if (parsed.already_processed && parsed.deck_id) {
				setUploadNoticeDeckId(parsed.deck_id);
				void showDeckPreview(parsed.deck_id, {
					title: stripPdfExtension(file.name),
				});
				void refreshTodayDecks();
				return;
			}

			clearResolvedModelId();
			const nowIso = new Date().toISOString();
			upsertTask(parsed.task_id, () => ({
				taskId: parsed.task_id,
				fileName: file.name,
				status:
					parsed.status === "completed"
						? "completed"
						: parsed.status === "processing"
							? "processing"
							: "pending",
				deckId: null,
				deck: null,
				errorMessage: null,
				errorCode: null,
				pollingError: null,
				resolvedModelId: null,
				createdAt: nowIso,
				updatedAt: nowIso,
			}));
			scheduleTaskPoll(parsed.task_id, 0);
		} catch (error) {
			handleAuthApiErrorSideEffects(error, signOut);
			const authMessage = resolveAuthApiUserMessage(error, t);
			if (authMessage) {
				setUploadError(authMessage);
				setUploadErrorShowLoginCta(true);
				return;
			}
			setUploadErrorShowLoginCta(false);

			if (error instanceof ApiError) {
				const message = error.detailMessage?.trim()
					? error.detailMessage
					: error.message;
				const pageLimit = parsePdfPageLimitError(message);
				if (pageLimit) {
					setUploadError(formatPdfPageLimitErrorMessage(pageLimit, locale, t));
					return;
				}
				setUploadError(message || t("pdfAnki.main.error.uploadFailed"));
				return;
			}

			setUploadError(t("pdfAnki.main.error.uploadFailed"));
		} finally {
			setUploadingCount((current) => Math.max(0, current - 1));
			if (fileInputRef.current) {
				fileInputRef.current.value = "";
			}
		}
	};

	const handleDrop: React.DragEventHandler<HTMLDivElement> = async (event) => {
		event.preventDefault();
		const file = event.dataTransfer.files[0];
		if (!file) {
			return;
		}
		if (!isPdfLikeFile(file)) {
			setUploadError(t("pdfAnki.main.error.onlyPdf"));
			setUploadErrorShowLoginCta(false);
			return;
		}
		await uploadFile(file);
	};

	const handleFileChange = async (
		event: React.ChangeEvent<HTMLInputElement>,
	) => {
		const file = event.target.files?.[0];
		if (!file) {
			return;
		}
		if (!isPdfLikeFile(file)) {
			setUploadError(t("pdfAnki.main.error.onlyPdf"));
			setUploadErrorShowLoginCta(false);
			if (fileInputRef.current) {
				fileInputRef.current.value = "";
			}
			return;
		}
		await uploadFile(file);
	};

	const taskList = useMemo(() => {
		return Object.values(tasks).sort((left, right) => {
			return new Date(right.updatedAt).getTime() - new Date(left.updatedAt).getTime();
		});
	}, [tasks]);
	const activeSessionTasks = taskList.filter(
		(task) => task.status === "pending" || task.status === "processing",
	);
	const visibleSessionTasks = taskList.filter(
		(task) =>
			task.status === "pending" ||
			task.status === "processing" ||
			task.status === "failed" ||
			task.status === "error",
	);
	const latestActiveTask = activeSessionTasks[0] ?? null;
	const latestActiveTaskError = latestActiveTask
		? getUserFacingPdfError(latestActiveTask.errorMessage)
		: null;
	const rawActiveTaskErrorLength = latestActiveTask?.errorMessage?.length ?? 0;
	const groqFreeTierBusyNotice =
		latestActiveTaskError?.code === "groq_free_tier_busy"
			? latestActiveTaskError
			: null;
	const insufficientBalanceNotice =
		latestActiveTaskError?.code === "pdf_anki_insufficient_balance"
			? latestActiveTaskError
			: null;

	const { plainSeconds, richMinutes } = pdfAnkiPageMinuteEstimates(uploadTier);
	const processingEstimate = t("pdfAnki.main.processingEstimate", {
		tier: localizePdfAnkiTierName(getPdfAnkiVisibleTierKind(me), t),
		plainSeconds,
		richMinutes,
	});
	const needsPdfSignIn = !authLoading && !isAuthReady;

	const caps = useMemo(() => {
		if (!me) {
			return getPdfAnkiStorageCapsFromMe(null);
		}
		const walletActive = me.payg === true || me.pdf_anki_payg === true;
		if (walletActive && !fallbackModelId) {
			return getPdfAnkiPaygStorageCaps();
		}
		return getPdfAnkiStorageCapsFromMe(me);
	}, [fallbackModelId, me]);

	const storedRaw = me?.pdf_anki_flashcards_stored;
	const baseStored =
		typeof storedRaw === "number" && Number.isFinite(storedRaw)
			? Math.max(0, Math.trunc(storedRaw))
			: 0;
	const currentCount = Math.min(
		Math.max(0, baseStored + Math.max(0, bump)),
		caps.maxFlashcards,
	);
	const isAtStorageLimit =
		!authLoading && Boolean(me) && currentCount >= caps.maxFlashcards;
	const dropZoneBusy = authLoading || !isAuthReady || isAtStorageLimit;

	useEffect(() => {
		if (!isAuthReady || !userIdForApi || activeSessionTasks.length === 0) {
			if (queueRefreshLoopTimeoutRef.current) {
				clearTimeout(queueRefreshLoopTimeoutRef.current);
				queueRefreshLoopTimeoutRef.current = null;
			}
			return;
		}

		let cancelled = false;
		let refreshInFlight = false;

		const loop = async () => {
			if (cancelled || refreshInFlight) {
				return;
			}

			refreshInFlight = true;
			try {
				const summaries = await refreshTodayDecks();
				if (!cancelled && summaries.length > 0) {
					await reconcileTasksWithDeckSummaries(summaries);
				}
			} finally {
				refreshInFlight = false;
				if (!cancelled) {
					queueRefreshLoopTimeoutRef.current = setTimeout(
						() => void loop(),
						POLL_INTERVAL_MS,
					);
				}
			}
		};

		void loop();

		return () => {
			cancelled = true;
			if (queueRefreshLoopTimeoutRef.current) {
				clearTimeout(queueRefreshLoopTimeoutRef.current);
				queueRefreshLoopTimeoutRef.current = null;
			}
		};
	}, [isAuthReady, userIdForApi, activeSessionTasks.length, dayKey]);

	return (
		<main className="flex min-h-screen flex-col bg-slate-100 text-slate-900">
			<section className="mx-auto flex w-full max-w-7xl flex-1 flex-col gap-6 px-6 py-8 lg:flex-row">
				<div className="flex-1 space-y-4">
					<div
						onDrop={dropZoneBusy ? (event) => event.preventDefault() : handleDrop}
						onDragOver={(event) => event.preventDefault()}
						role="region"
						aria-busy={uploadingCount > 0}
						aria-label={t("pdfAnki.main.dropTitle")}
						className={`rounded-[28px] border border-dashed px-6 py-8 shadow-sm transition ${
							dropZoneBusy
								? "border-slate-300 bg-slate-50"
								: "cursor-pointer border-slate-300 bg-[linear-gradient(160deg,rgba(255,255,255,0.98),rgba(248,250,252,0.92),rgba(226,232,240,0.88))] hover:border-slate-400"
						}`}
						onClick={dropZoneBusy ? undefined : handleBrowseClick}
					>
						<div className="flex flex-col gap-5">
							<div className="flex flex-wrap items-center justify-between gap-3">
								<div className="inline-flex items-center gap-2 rounded-full border border-slate-200 bg-white/85 px-3 py-1 text-[11px] font-medium text-slate-700 shadow-sm">
									<FileText className="h-3.5 w-3.5" aria-hidden />
									<span>{t("pdfAnki.main.dropTitle")}</span>
								</div>
								{activeSessionTasks.length > 0 ? (
									<div className="inline-flex items-center gap-2 rounded-full border border-amber-200 bg-amber-50 px-3 py-1 text-[11px] font-medium text-amber-800">
										<Clock3 className="h-3.5 w-3.5" aria-hidden />
										<span>{activeSessionTasks.length}</span>
										<span>{inlineCopy.taskHint}</span>
									</div>
								) : null}
							</div>

							<div className="space-y-2 text-center">
								<p className="text-lg font-semibold text-slate-900">
									{t("pdfAnki.main.dropTitle")}
								</p>
								<p className="mx-auto max-w-xl text-sm leading-relaxed text-slate-600">
									{t("pdfAnki.main.dropDescription")}
								</p>
							</div>

							<div className="flex flex-wrap items-center justify-center gap-3">
								<button
									type="button"
									onClick={(event) => {
										event.stopPropagation();
										handleBrowseClick();
									}}
									disabled={dropZoneBusy}
									className="inline-flex items-center rounded-xl bg-slate-900 px-4 py-2 text-sm font-medium text-white shadow-sm transition hover:bg-slate-800 disabled:cursor-not-allowed disabled:opacity-60"
								>
									{t("pdfAnki.main.processAnother")}
								</button>
								<span className="rounded-full border border-slate-200 bg-white/85 px-3 py-1 text-[11px] font-medium uppercase tracking-[0.18em] text-slate-500">
									PDF
								</span>
							</div>

							{uploadingCount > 0 ? (
								<div className="space-y-2">
									<IndeterminateProgressBar
										ariaLabel={t("pdfAnki.main.processingTitle")}
										className="w-full"
									/>
									<p className="text-center text-xs text-slate-500">
										{processingEstimate}
									</p>
								</div>
							) : null}

							{authLoading ? (
								<p className="text-center text-xs text-slate-500">
									{t("auth.loading")}
								</p>
							) : null}
							{needsPdfSignIn ? (
								<div className="flex flex-col items-center gap-2 text-center">
									<p className="text-xs text-slate-600">
										{t("pdfAnki.main.error.signInRequired")}
									</p>
									<button
										type="button"
										className="rounded-lg bg-slate-900 px-4 py-2 text-xs font-medium text-white hover:bg-slate-800"
										onClick={(event) => {
											event.stopPropagation();
											void loginWithGoogle();
										}}
									>
										{t("auth.signInGoogle")}
									</button>
								</div>
							) : null}
							{isAtStorageLimit ? (
								<p className="text-center text-xs text-rose-600">
									{t("pdfAnki.main.storageCapReached")}
								</p>
							) : null}
						</div>
						<input
							ref={fileInputRef}
							type="file"
							accept="application/pdf"
							className="hidden"
							disabled={dropZoneBusy}
							onChange={handleFileChange}
						/>
					</div>

					{uploadError ? (
						<div
							className="space-y-2 rounded-2xl border border-rose-200 bg-rose-50 p-4"
							role="alert"
							aria-live="assertive"
						>
							<p className="text-sm font-medium text-rose-700">{uploadError}</p>
							{uploadErrorShowLoginCta ? (
								<button
									type="button"
									className="rounded-lg bg-slate-900 px-3 py-2 text-xs font-medium text-white hover:bg-slate-800"
									onClick={() => {
										void loginWithGoogle();
									}}
								>
									{t("auth.signInGoogle")}
								</button>
							) : null}
						</div>
					) : null}

					{uploadNoticeDeckId ? (
						<div className="rounded-2xl border border-amber-200 bg-amber-50 p-4">
							<div className="flex flex-wrap items-start justify-between gap-3">
								<div className="flex items-start gap-3">
									<div className="rounded-full bg-amber-100 p-2 text-amber-700">
										<AlertTriangle className="h-4 w-4" aria-hidden />
									</div>
									<div className="space-y-1">
										<p className="text-sm font-semibold text-amber-900">
											{t("pdfAnki.main.alreadyProcessed.title")}
										</p>
										<p className="text-xs leading-relaxed text-amber-800">
											{t("pdfAnki.main.alreadyProcessed.description")}
										</p>
									</div>
								</div>
								<Link
									href={`/pdf-anki/deck/${uploadNoticeDeckId}`}
									className="inline-flex items-center rounded-lg bg-amber-600 px-3 py-2 text-xs font-medium text-white shadow-sm transition hover:bg-amber-500"
								>
									{t("pdfAnki.main.viewDeck")}
								</Link>
							</div>
						</div>
					) : null}

					<PdfAnkiTaskSessionList
						tasks={visibleSessionTasks}
						activeCount={activeSessionTasks.length}
						locale={locale}
						title={inlineCopy.sessionTasksTitle}
						subtitle={inlineCopy.taskHint}
						emptyMessage={inlineCopy.sessionTasksEmpty}
					/>

					<PdfAnkiDeckQueue
						decks={todayDecks}
						locale={locale}
						title={inlineCopy.todayQueueTitle}
						subtitle={inlineCopy.todayQueueSubtitle}
						emptyMessage={inlineCopy.todayQueueEmpty}
						selectedDeckId={previewDeck?.deckId ?? null}
						errorMessage={todayDecksLoadError}
						selectedLabel={inlineCopy.selectedLabel}
						cardsLabel={inlineCopy.cardsLabel}
						onSelectDeck={(deck) => {
							void showDeckPreview(deck.deckId, {
								title: deck.title,
								status: deck.status,
								createdAt: deck.createdAt,
								flashcards: deck.flashcards,
								sourceTaskId: deck.sourceTaskId,
							});
						}}
					/>

					{groqFreeTierBusyNotice ? (
						<PdfAnkiErrorAlert
							error={groqFreeTierBusyNotice}
							variant="notice"
							telemetryRawLength={rawActiveTaskErrorLength}
						/>
					) : null}
					{insufficientBalanceNotice ? (
						<PdfAnkiErrorAlert
							error={{
								...insufficientBalanceNotice,
								detail: buildPdfAnkiInsufficientBalanceDetail(me, t),
							}}
							variant="notice"
							telemetryRawLength={rawActiveTaskErrorLength}
						/>
					) : null}
				</div>

				<div className="flex-[1.2] rounded-2xl border border-slate-200 bg-white p-5 shadow-sm">
					<div className="flex flex-wrap items-start justify-between gap-4">
						<div>
							<h2 className="text-sm font-semibold text-slate-900">
								{inlineCopy.latestDeckTitle}
							</h2>
							<p className="mt-1 text-xs text-slate-500">
								{inlineCopy.previewSelectHint}
							</p>
							{previewDeck ? (
								<div className="mt-3 flex flex-wrap items-center gap-3 text-xs text-slate-600">
									<span className="font-medium text-slate-900">
										{previewDeck.title}
									</span>
									<span>
										{previewDeck.flashcards.length} {inlineCopy.cardsLabel}
									</span>
								</div>
							) : null}
						</div>

						<div className="flex flex-wrap items-center gap-3">
							{previewDeck?.deckId ? (
								<Link
									href={`/pdf-anki/deck/${previewDeck.deckId}`}
									className="inline-flex items-center rounded-lg bg-violet-600 px-3 py-2 text-xs font-medium text-white shadow-sm transition hover:bg-violet-500"
								>
									{t("pdfAnki.main.viewDeck")}
								</Link>
							) : (
								<button
									type="button"
									disabled
									className="inline-flex items-center rounded-lg bg-slate-200 px-3 py-2 text-xs font-medium text-slate-500"
								>
									{t("pdfAnki.main.viewDeck")}
								</button>
							)}
							<button
								type="button"
								onClick={handleBrowseClick}
								disabled={dropZoneBusy}
								className="inline-flex items-center rounded-lg border border-slate-300 bg-white px-3 py-2 text-xs font-medium text-slate-700 shadow-sm transition hover:-translate-y-0.5 hover:bg-slate-50 hover:shadow-md disabled:cursor-not-allowed disabled:opacity-60"
							>
								{t("pdfAnki.main.processAnother")}
							</button>
						</div>
					</div>

					{previewDeck?.status === "completed_partial" ? (
						<p className="mt-3 text-xs text-amber-700" role="status">
							{t("pdfAnki.main.completedPartialNotice")}
						</p>
					) : null}
					{previewError ? (
						<p className="mt-4 text-sm text-rose-600">{previewError}</p>
					) : isPreviewLoading ? (
						<div className="mt-6 flex items-center gap-3 rounded-2xl border border-slate-200 bg-slate-50 p-4 text-sm text-slate-600">
							<IndeterminateProgressBar
								ariaLabel={inlineCopy.previewLoading}
								className="w-full max-w-xs"
							/>
						</div>
					) : !previewDeck || previewDeck.flashcards.length === 0 ? (
						<p className="mt-4 text-sm text-slate-500">
							{t("pdfAnki.main.previewEmpty")}
						</p>
					) : (
						<div className="mt-4 grid max-h-[620px] grid-cols-1 gap-5 overflow-auto md:grid-cols-2">
							{previewDeck.flashcards.map((card, index) => (
								<PdfAnkiFlashcard
									key={`${previewDeck.deckId}-preview-${index}`}
									cardKey={`${previewDeck.deckId}-preview-${index}`}
									card={card}
									deckId={previewDeck.deckId}
									hideActions
								/>
							))}
						</div>
					)}
				</div>
			</section>
		</main>
	);
}
```

### `src/app/pdf-anki/pdf-anki-pricing-api.ts`
```ts
import { apiRequest } from "@/lib/api-client";

export type FetchPdfAnkiPricingOptions = Record<string, never>;

export type PdfAnkiPricingStripeLinks = {
	premium?: string;
	pro?: string;
	/** Backend key for pay-as-you-go top-up link. */
	payg_topup?: string;
	/** Legacy alias – some backend versions return `payg` instead of `payg_topup`. */
	payg?: string;
};

export type PdfAnkiPricingResponse = {
	stripe_links: PdfAnkiPricingStripeLinks;
	/**
	 * Backend-provided path for the Stripe customer portal.
	 * E.g. `/billing/stripe/customer-portal` (relative to backend base).
	 * When present, use this instead of the hardcoded fallback.
	 */
	customer_portal_path?: string;
};

export async function fetchPdfAnkiPricing(): Promise<PdfAnkiPricingResponse> {
	return apiRequest<PdfAnkiPricingResponse>("/pdf/pricing", {
		method: "GET",
	});
}
```

### `src/app/pdf-anki/PdfAnkiModuleChrome.tsx`
```tsx
"use client";

import type { ReactNode } from "react";
import Link from "next/link";
import { usePathname } from "next/navigation";
import { useEffect, useMemo, useState } from "react";

/**
 * Root `layout.tsx` does not mount `AuthProvider`; this shell wraps PDF–Anki
 * routes with `module="pdf_anki"` so session storage and OAuth stay isolated
 * from Receipts (`receipt_parser`).
 */
import { AuthProvider, useAuthMe, useSupabaseAuth } from "@/app/AuthProvider";
import { AuthHeader } from "@/app/AuthHeader";
import { LanguageSwitcher } from "@/app/LanguageSwitcher";
import { useI18n } from "@/app/i18n-provider";
import { PdfAnkiChromeQuota } from "@/modules/pdf-anki/PdfAnkiChromeQuota";
import { PdfAnkiQuotaBumpProvider, usePdfAnkiQuotaBump } from "@/modules/pdf-anki/PdfAnkiQuotaBumpContext";
import {
	getPdfAnkiPaygStorageCaps,
	getPdfAnkiStorageCapsFromMe,
} from "@/modules/pdf-anki/pdfAnkiStorageCaps";
import { PdfAnkiWalletModelRow } from "@/modules/pdf-anki/PdfAnkiWalletModelRow";
import { PdfAnkiModelStateProvider } from "@/modules/pdf-anki/PdfAnkiModelStateContext";

import { ModuleFooter } from "@/components/ModuleFooter";
import { SupportTicketPanel } from "@/components/SupportTicketPanel";

export function PdfAnkiModuleChrome({ children }: { children: ReactNode }) {
	const pathname = usePathname();
	const [localeGateReady, setLocaleGateReady] = useState(
		process.env.NODE_ENV === "test",
	);

	useEffect(() => {
		setLocaleGateReady(true);
	}, []);

	if (!localeGateReady) {
		return null;
	}

	return (
		<AuthProvider module="pdf_anki">
			<PdfAnkiModelStateProvider>
				<PdfAnkiModuleChromeInner pathname={pathname}>{children}</PdfAnkiModuleChromeInner>
			</PdfAnkiModelStateProvider>
		</AuthProvider>
	);
}

function StorageCapBanner({ me }: { me: ReturnType<typeof useAuthMe> }) {
	const { t } = useI18n();
	const { bump } = usePdfAnkiQuotaBump();
	const caps = useMemo(() => {
		if (!me) {
			return getPdfAnkiStorageCapsFromMe(null);
		}
		const walletActive = me.payg === true || me.pdf_anki_payg === true;
		if (walletActive) {
			return getPdfAnkiPaygStorageCaps();
		}
		return getPdfAnkiStorageCapsFromMe(me);
	}, [me]);
	const storedRaw = me?.pdf_anki_flashcards_stored;
	const baseStored =
		typeof storedRaw === "number" && Number.isFinite(storedRaw)
			? Math.max(0, Math.trunc(storedRaw))
			: 0;
	const currentCount = Math.min(
		Math.max(0, baseStored + Math.max(0, bump)),
		caps.maxFlashcards,
	);
	const isAtStorageLimit = me != null && currentCount >= caps.maxFlashcards;

	if (!isAtStorageLimit) return null;

	return (
		<div
			role="alert"
			className="border-b border-amber-200 bg-amber-50 px-4 py-2 text-center text-xs text-amber-800"
		>
			{t("pdfAnki.main.storageCapReached")}{" "}
			<Link
				href="/pdf-anki/pricing"
				className="font-semibold underline underline-offset-2 hover:text-amber-900"
			>
				{t("pdfAnki.chrome.plans")}
			</Link>
		</div>
	);
}

function PdfAnkiModuleChromeInner({
	pathname,
	children,
}: {
	pathname: string;
	children: ReactNode;
}) {
	const { t } = useI18n();
	const me = useAuthMe();
	const { isLoading: authLoading } = useSupabaseAuth();
	const tier = (me?.tier ?? "").trim().toLowerCase();
	const canOpenSupport =
		tier === "premium" || tier === "pro" || tier === "admin";
	const [supportOpen, setSupportOpen] = useState(false);

	const decksActive =
		pathname === "/pdf-anki/decks" ||
		pathname.startsWith("/pdf-anki/deck/");
	const pricingActive = pathname.startsWith("/pdf-anki/pricing");

	const navPillClass = (active: boolean): string =>
		active
			? "inline-flex items-center rounded-full bg-slate-900 px-3 py-1.5 text-xs font-semibold text-white shadow-sm"
			: "inline-flex items-center rounded-full border border-slate-300 bg-white px-3 py-1.5 text-xs font-medium text-slate-700 shadow-sm transition hover:border-slate-400 hover:text-slate-900";

	return (
		<PdfAnkiQuotaBumpProvider>
			<div className="flex min-h-screen flex-col">
				<div className="sticky top-0 z-40 border-b border-slate-200 bg-white">
					<div className="relative mx-auto flex max-w-6xl items-center justify-between gap-3 px-4 py-2.5">
						<div className="flex min-w-0 shrink-0 items-center gap-3">
							<span className="text-2xl font-semibold text-slate-900">
								Kera
							</span>
							{me != null ? (
								<Link
									href="/pdf-anki/decks"
									className={navPillClass(decksActive)}
								>
									{t("pdfAnki.main.myDecks")}
								</Link>
							) : null}
						</div>
						<div className="flex min-w-0 shrink-0 flex-wrap items-center justify-end gap-2 sm:flex-nowrap sm:gap-3">
							{me != null ? <PdfAnkiChromeQuota /> : null}
							<AuthHeader
								hoverToOpen
								transactionsHref="/billing/transactions?source=pdf_anki"
							/>
							<Link href="/pdf-anki/pricing" className={navPillClass(pricingActive)}>
								{t("pdfAnki.chrome.plans")}
							</Link>
							<LanguageSwitcher />
						</div>
					</div>
					<PdfAnkiWalletModelRow />
				</div>

				<StorageCapBanner me={me} />

				<div className="flex-1">{authLoading ? null : children}</div>

				<ModuleFooter
					moduleLabel="pdf_anki"
					canOpenSupport={canOpenSupport}
					onOpenTickets={() => setSupportOpen(true)}
					blockedSupportLabel={t("pdfAnki.chrome.plans")}
				/>

				{/* Same `module` as AuthProvider — required for /support/* query + errors */}
				<SupportTicketPanel
					key="pdf_anki"
					module="pdf_anki"
					isOpen={supportOpen}
					onClose={() => setSupportOpen(false)}
					isAllowedTier={canOpenSupport}
					blockedMessage={t("support.tickets.upgradePlan")}
				/>
			</div>
		</PdfAnkiQuotaBumpProvider>
	);
}
```

### `src/app/pdf-anki/cookies/page.tsx`
```tsx
"use client";

export const dynamic = "force-dynamic";

import { useI18n } from "@/app/i18n-provider";
import { LegalPageTemplate } from "@/components/LegalPageTemplate";
import type { Locale } from "@/lib/i18n-config";

type CookiePolicyContent = {
	title: string;
	description: string;
	lastUpdated: string;
	sections: { title: string; body: string }[];
};

const policies: Record<Locale, CookiePolicyContent> = {
	en: {
		title: "Cookie Policy (PDF to Anki)",
		description: "This policy explains how we use cookies and similar technologies (such as browser local storage) to recognize you when you visit our platform and ensure the security of your session.",
		lastUpdated: "Last updated: April 6, 2026",
		sections: [
			{
				title: "1. What are cookies?",
				body: "Cookies are small text files stored on your device when you visit a website. We use them to remember your preferences, keep your session securely logged in, and understand how you interact with the platform.",
			},
			{
				title: "2. Strictly necessary cookies",
				body: "Our platform relies on essential technical cookies to function. Specifically, we use secure cookies (HttpOnly, Secure, SameSite) to manage your authentication session (e.g., `ff_auth_refresh_*`). These cookies cannot be read by browser scripts, protecting you against XSS attacks. Without these cookies, you would not be able to log in or use the private modules.",
			},
			{
				title: "3. Local Storage",
				body: "In addition to cookies, we use your browser's local storage to improve your experience:\n\n- Language preferences: We save your language selection for future visits.\n- Processing State: We temporarily store the ID of your active PDF processing task so you don't lose your progress if you accidentally refresh the page.",
			},
			{
				title: "4. Telemetry and Performance",
				body: "We do not use third-party advertising tracking cookies. We collect performance and attention metrics anonymously to improve the user interface and detect errors on the platform, always respecting your privacy.",
			},
			{
				title: "5. Cookie Management",
				body: "Since we only use cookies that are strictly necessary for the security and operation of the service, we do not offer a panel to disable them, as doing so would break the application's functionality. If you do not want us to use these cookies, you must log out and configure your browser to block them, although this will prevent the use of the platform.",
			},
		],
	},
	es: {
		title: "Política de Cookies (PDF a Anki)",
		description: "Esta política explica cómo utilizamos las cookies y tecnologías similares (como el almacenamiento local del navegador) para reconocerte cuando visitas nuestra plataforma y garantizar la seguridad de tu sesión.",
		lastUpdated: "Última actualización: 6 de abril de 2026",
		sections: [
			{
				title: "1. ¿Qué son las cookies?",
				body: "Las cookies son pequeños archivos de texto que se almacenan en tu dispositivo cuando visitas un sitio web. Las utilizamos para recordar tus preferencias, mantener tu sesión iniciada de forma segura y entender cómo interactúas con la plataforma.",
			},
			{
				title: "2. Cookies estrictamente necesarias",
				body: "Nuestra plataforma depende de cookies técnicas esenciales para funcionar. Específicamente, utilizamos cookies seguras (HttpOnly, Secure, SameSite) para gestionar tu sesión de autenticación (ej. `ff_auth_refresh_*`). Estas cookies no pueden ser leídas por scripts del navegador, lo que te protege contra ataques XSS. Sin estas cookies, no podrías iniciar sesión ni utilizar los módulos privados.",
			},
			{
				title: "3. Almacenamiento Local (Local Storage)",
				body: "Además de las cookies, utilizamos el almacenamiento local de tu navegador para mejorar tu experiencia:\n\n- Preferencias de idioma: Guardamos tu selección de idioma para futuras visitas.\n- Estado de procesamiento: Guardamos temporalmente el ID de tu tarea activa de procesamiento de PDF para que no pierdas tu progreso si recargas la página accidentalmente.",
			},
			{
				title: "4. Telemetría y Rendimiento",
				body: "No utilizamos cookies de rastreo publicitario de terceros. Recopilamos métricas de rendimiento y atención de forma anónima para mejorar la interfaz de usuario y detectar errores en la plataforma, respetando siempre tu privacidad.",
			},
			{
				title: "5. Gestión de Cookies",
				body: "Dado que solo utilizamos cookies estrictamente necesarias para la seguridad y el funcionamiento del servicio, no ofrecemos un panel para deshabilitarlas, ya que rompería la funcionalidad de la aplicación. Si no deseas que utilicemos estas cookies, debes cerrar sesión y configurar tu navegador para bloquearlas, aunque esto impedirá el uso de la plataforma.",
			},
		],
	},
	de: {
		title: "Cookie-Richtlinie (PDF zu Anki)",
		description: "Diese Richtlinie erklärt, wie wir Cookies und ähnliche Technologien (wie den lokalen Speicher des Browsers) verwenden, um Sie bei Ihrem Besuch auf unserer Plattform zu erkennen und die Sicherheit Ihrer Sitzung zu gewährleisten.",
		lastUpdated: "Zuletzt aktualisiert: 6. April 2026",
		sections: [
			{
				title: "1. Was sind Cookies?",
				body: "Cookies sind kleine Textdateien, die auf Ihrem Gerät gespeichert werden, wenn Sie eine Website besuchen. Wir verwenden sie, um sich an Ihre Einstellungen zu erinnern, Ihre Sitzung sicher angemeldet zu halten und zu verstehen, wie Sie mit der Plattform interagieren.",
			},
			{
				title: "2. Unbedingt erforderliche Cookies",
				body: "Unsere Plattform ist auf wesentliche technische Cookies angewiesen, um zu funktionieren. Insbesondere verwenden wir sichere Cookies (HttpOnly, Secure, SameSite), um Ihre Authentifizierungssitzung zu verwalten (z. B. `ff_auth_refresh_*`). Diese Cookies können nicht von Browser-Skripten gelesen werden, was Sie vor XSS-Angriffen schützt. Ohne diese Cookies könnten Sie sich nicht anmelden oder die privaten Module nutzen.",
			},
			{
				title: "3. Lokale Speicherung (Local Storage)",
				body: "Zusätzlich zu Cookies verwenden wir den lokalen Speicher Ihres Browsers, um Ihre Erfahrung zu verbessern:\n\n- Spracheinstellungen: Wir speichern Ihre Sprachauswahl für zukünftige Besuche.\n- Verarbeitungsstatus: Wir speichern vorübergehend die ID Ihrer aktiven PDF-Verarbeitungsaufgabe, damit Sie Ihren Fortschritt nicht verlieren, wenn Sie die Seite versehentlich aktualisieren.",
			},
			{
				title: "4. Telemetrie und Leistung",
				body: "Wir verwenden keine Werbe-Tracking-Cookies von Drittanbietern. Wir erfassen Leistungs- und Aufmerksamkeitsmetriken anonym, um die Benutzeroberfläche zu verbessern und Fehler auf der Plattform zu erkennen, wobei wir stets Ihre Privatsphäre respektieren.",
			},
			{
				title: "5. Cookie-Verwaltung",
				body: "Da wir nur Cookies verwenden, die für die Sicherheit und den Betrieb des Dienstes unbedingt erforderlich sind, bieten wir kein Panel an, um sie zu deaktivieren, da dies die Funktionalität der Anwendung beeinträchtigen würde. Wenn Sie nicht möchten, dass wir diese Cookies verwenden, müssen Sie sich abmelden und Ihren Browser so konfigurieren, dass er sie blockiert, was jedoch die Nutzung der Plattform verhindert.",
			},
		],
	},
	it: {
		title: "Informativa sui Cookie (PDF ad Anki)",
		description: "Questa politica spiega come utilizziamo i cookie e tecnologie simili (come l'archiviazione locale del browser) per riconoscerti quando visiti la nostra piattaforma e garantire la sicurezza della tua sessione.",
		lastUpdated: "Ultimo aggiornamento: 6 aprile 2026",
		sections: [
			{
				title: "1. Cosa sono i cookie?",
				body: "I cookie sono piccoli file di testo memorizzati sul tuo dispositivo quando visiti un sito web. Li utilizziamo per ricordare le tue preferenze, mantenere la tua sessione di accesso sicura e capire come interagisci con la piattaforma.",
			},
			{
				title: "2. Cookie strettamente necessari",
				body: "La nostra piattaforma si basa su cookie tecnici essenziali per funzionare. Nello specifico, utilizziamo cookie sicuri (HttpOnly, Secure, SameSite) per gestire la tua sessione di autenticazione (es. `ff_auth_refresh_*`). Questi cookie non possono essere letti dagli script del browser, proteggendoti dagli attacchi XSS. Senza questi cookie, non saresti in grado di accedere o utilizzare i moduli privati.",
			},
			{
				title: "3. Archiviazione Locale (Local Storage)",
				body: "Oltre ai cookie, utilizziamo l'archiviazione locale del tuo browser per migliorare la tua esperienza:\n\n- Preferenze di lingua: Salviamo la tua selezione della lingua per le visite future.\n- Stato di elaborazione: Salviamo temporaneamente l'ID della tua attività di elaborazione PDF attiva in modo da non perdere i progressi se aggiorni accidentalmente la pagina.",
			},
			{
				title: "4. Telemetria e Prestazioni",
				body: "Non utilizziamo cookie di tracciamento pubblicitario di terze parti. Raccogliamo metriche di prestazioni e attenzione in modo anonimo per migliorare l'interfaccia utente e rilevare errori sulla piattaforma, rispettando sempre la tua privacy.",
			},
			{
				title: "5. Gestione dei Cookie",
				body: "Poiché utilizziamo solo cookie strettamente necessari per la sicurezza e il funzionamento del servizio, non offriamo un pannello per disabilitarli, poiché ciò comprometterebbe la funzionalità dell'applicazione. Se non desideri che utilizziamo questi cookie, devi disconnetterti e configurare il tuo browser per bloccarli, sebbene ciò impedirà l'uso della piattaforma.",
			},
		],
	},
	fr: {
		title: "Politique relative aux cookies (PDF vers Anki)",
		description: "Cette politique explique comment nous utilisons les cookies et des technologies similaires (telles que le stockage local du navigateur) pour vous reconnaître lorsque vous visitez notre plateforme et garantir la sécurité de votre session.",
		lastUpdated: "Dernière mise à jour : 6 avril 2026",
		sections: [
			{
				title: "1. Que sont les cookies ?",
				body: "Les cookies sont de petits fichiers texte stockés sur votre appareil lorsque vous visitez un site Web. Nous les utilisons pour mémoriser vos préférences, maintenir votre session connectée en toute sécurité et comprendre comment vous interagissez avec la plateforme.",
			},
			{
				title: "2. Cookies strictement nécessaires",
				body: "Notre plateforme s'appuie sur des cookies techniques essentiels pour fonctionner. Plus précisément, nous utilisons des cookies sécurisés (HttpOnly, Secure, SameSite) pour gérer votre session d'authentification (par ex., `ff_auth_refresh_*`). Ces cookies ne peuvent pas être lus par les scripts du navigateur, ce qui vous protège contre les attaques XSS. Sans ces cookies, vous ne pourriez pas vous connecter ni utiliser les modules privés.",
			},
			{
				title: "3. Stockage Local (Local Storage)",
				body: "En plus des cookies, nous utilisons le stockage local de votre navigateur pour améliorer votre expérience :\n\n- Préférences linguistiques : Nous enregistrons votre sélection de langue pour vos futures visites.\n- État de traitement : Nous stockons temporairement l'ID de votre tâche de traitement PDF active afin que vous ne perdiez pas votre progression si vous actualisez accidentellement la page.",
			},
			{
				title: "4. Télémétrie et Performances",
				body: "Nous n'utilisons pas de cookies de suivi publicitaire tiers. Nous collectons des mesures de performance et d'attention de manière anonyme pour améliorer l'interface utilisateur et détecter les erreurs sur la plateforme, en respectant toujours votre vie privée.",
			},
			{
				title: "5. Gestion des cookies",
				body: "Étant donné que nous n'utilisons que des cookies strictement nécessaires à la sécurité et au fonctionnement du service, nous ne proposons pas de panneau pour les désactiver, car cela empêcherait l'application de fonctionner. Si vous ne souhaitez pas que nous utilisions ces cookies, vous devez vous déconnecter et configurer votre navigateur pour les bloquer, bien que cela empêchera l'utilisation de la plateforme.",
			},
		],
	},
	ru: {
		title: "Политика использования файлов cookie (PDF в Anki)",
		description: "Эта политика объясняет, как мы используем файлы cookie и аналогичные технологии (например, локальное хранилище браузера), чтобы узнавать вас при посещении нашей платформы и обеспечивать безопасность вашей сессии.",
		lastUpdated: "Последнее обновление: 6 апреля 2026 г.",
		sections: [
			{
				title: "1. Что такое файлы cookie?",
				body: "Файлы cookie — это небольшие текстовые файлы, которые сохраняются на вашем устройстве при посещении веб-сайту. Мы используем их, чтобы запоминать ваши предпочтения, обеспечивать безопасность вашей сессии и понимать, как вы взаимодействуете с платформой.",
			},
			{
				title: "2. Строго необходимые файлы cookie",
				body: "Наша платформа полагается на основные технические файлы cookie для работы. В частности, мы используем безопасные файлы cookie (HttpOnly, Secure, SameSite) для управления вашей сессией аутентификации (например, `ff_auth_refresh_*`). Эти файлы cookie не могут быть прочитаны скриптами браузера, что защищает вас от XSS-атак. Без этих файлов cookie вы не смогли бы войти в систему или использовать приватные модули.",
			},
			{
				title: "3. Локальное хранилище (Local Storage)",
				body: "Помимо файлов cookie, мы используем локальное хранилище вашего браузера для улучшения вашего опыта:\n\n- Языковые предпочтения: Мы сохраняем ваш выбор языка для будущих посещений.\n- Состояние обработки: Мы временно сохраняем идентификатор вашей активной задачи по обработке PDF, чтобы вы не потеряли свой прогресс, если случайно обновите страницу.",
			},
			{
				title: "4. Телеметрия и производительность",
				body: "Мы не используем сторонние рекламные файлы cookie для отслеживания. Мы анонимно собираем метрики производительности и внимания, чтобы улучшить пользовательский интерфейс и обнаруживать ошибки на платформе, всегда уважая вашу конфиденциальность.",
			},
			{
				title: "5. Управление файлами cookie",
				body: "Поскольку мы используем только те файлы cookie, которые строго необходимы для безопасности и работы сервиса, мы не предлагаем панель для их отключения, так как это нарушит функциональность приложения. Если вы не хотите, чтобы мы использовали эти файлы cookie, вы должны выйти из системы и настроить свой браузер на их блокировку, хотя это сделает невозможным использование платформы.",
			},
		],
	},
};

export default function PdfAnkiCookiePolicyPage() {
	const { locale } = useI18n();
	const content = policies[locale] ?? policies.en!;

	return (
		<LegalPageTemplate
			title={content.title}
			description={content.description}
			lastUpdated={content.lastUpdated}
			sections={content.sections}
		/>
	);
}
```

### `src/app/pdf-anki/deck/[id]/page.tsx`
```tsx
"use client";

import Link from "next/link";
import { useParams } from "next/navigation";
import { Info } from "lucide-react";
import { useCallback, useEffect, useId, useMemo, useRef, useState } from "react";
import { apiRequest, ApiError } from "@/lib/api-client";
import { useI18n } from "@/app/i18n-provider";
import {
	type DeckDetailResponse,
	type FlashcardItem,
	deckDetailResponseSchema,
	deckTagsResponseSchema,
} from "@/modules/pdf-anki/schemas";
import { PdfAnkiFlashcard } from "@/modules/pdf-anki/PdfAnkiFlashcard";
import { usePdfAnkiResolvedUserId } from "@/modules/pdf-anki/usePdfAnkiAuth";
import {
	downloadPdfDeckExport,
	type PdfDeckExportFormat,
} from "@/modules/pdf-anki/pdfAnkiDeckExport";

function cardMatchesTagFilter(
	card: FlashcardItem,
	selected: ReadonlySet<string>,
): boolean {
	if (selected.size === 0) {
		return true;
	}
	const onCard = new Set(card.tags ?? []);
	for (const tag of selected) {
		if (!onCard.has(tag)) {
			return false;
		}
	}
	return true;
}

function pdfAnkiUserScopedPath(path: string, userId: string): string {
	const params = new URLSearchParams({ user_id: userId });
	return `${path}?${params.toString()}`;
}

export default function PdfAnkiDeckDetailPage() {
	const { t, locale } = useI18n();
	const { isAuthReady, userIdForApi } = usePdfAnkiResolvedUserId();
	const params = useParams();
	const deckId = typeof params?.id === "string" ? params.id : null;
	const [deck, setDeck] = useState<DeckDetailResponse | null>(null);
	const [cards, setCards] = useState<FlashcardItem[]>([]);
	const [tags, setTags] = useState<string[]>([]);
	const [selectedTags, setSelectedTags] = useState<Set<string>>(() => new Set());
	const [notFound, setNotFound] = useState(false);
	const [loadError, setLoadError] = useState<string | null>(null);
	const [tagsError, setTagsError] = useState<string | null>(null);
	const [loading, setLoading] = useState(true);
	const [exportFormat, setExportFormat] = useState<PdfDeckExportFormat>("csv");
	const [exporting, setExporting] = useState(false);
	const [exportError, setExportError] = useState<string | null>(null);

	const reviewInfoTooltipId = useId();
	const reviewInfoWrapRef = useRef<HTMLSpanElement | null>(null);
	const [reviewInfoPinned, setReviewInfoPinned] = useState(false);
	const [reviewInfoHover, setReviewInfoHover] = useState(false);
	const reviewInfoOpen = reviewInfoPinned || reviewInfoHover;

	const fetchDeck = useCallback(async (): Promise<DeckDetailResponse> => {
		if (!deckId || !userIdForApi) {
			throw new Error("Missing deck context");
		}
		const raw = await apiRequest<unknown>(
			pdfAnkiUserScopedPath(
				`/pdf/deck/${encodeURIComponent(deckId)}`,
				userIdForApi,
			),
		);
		return deckDetailResponseSchema.parse(raw);
	}, [deckId, userIdForApi]);

	useEffect(() => {
		if (!reviewInfoPinned) return;

		const onMouseDown = (e: MouseEvent) => {
			const wrap = reviewInfoWrapRef.current;
			if (!wrap) return;
			const target = e.target;
			if (target instanceof Node && wrap.contains(target)) return;
			setReviewInfoPinned(false);
		};

		const onKeyDown = (e: KeyboardEvent) => {
			if (e.key === "Escape") {
				setReviewInfoPinned(false);
			}
		};

		document.addEventListener("mousedown", onMouseDown);
		document.addEventListener("keydown", onKeyDown);
		return () => {
			document.removeEventListener("mousedown", onMouseDown);
			document.removeEventListener("keydown", onKeyDown);
		};
	}, [reviewInfoPinned]);

	useEffect(() => {
		if (!deckId) {
			setLoading(false);
			setNotFound(true);
			setCards([]);
			return;
		}
		if (!isAuthReady || !userIdForApi) {
			setLoading(true);
			setNotFound(false);
			setLoadError(null);
			setDeck(null);
			setCards([]);
			return;
		}
		let cancelled = false;
		setLoading(true);
		setNotFound(false);
		setLoadError(null);
		fetchDeck()
			.then((raw) => {
				if (cancelled) return;
				setDeck(raw);
				setCards(raw.flashcards);
			})
			.catch((err) => {
				if (cancelled) return;
				if (err instanceof ApiError && err.status === 404) {
					setNotFound(true);
				} else {
					setLoadError(t("pdfAnki.deck.errorLoadDetail"));
				}
			})
			.finally(() => {
				if (!cancelled) setLoading(false);
			});
		return () => {
			cancelled = true;
		};
	}, [deckId, fetchDeck, isAuthReady, t, userIdForApi]);

	useEffect(() => {
		if (!deckId || !isAuthReady || !userIdForApi) {
			return;
		}
		let cancelled = false;
		setTagsError(null);
		apiRequest<unknown>(`/pdf/deck/${deckId}/tags`)
			.then((raw) => {
				if (cancelled) return;
				const parsed = deckTagsResponseSchema.parse(raw);
				setTags(parsed.tags); // <-- FIX: Extraemos el array 'tags' del objeto
			})
			.catch(() => {
				if (!cancelled) setTagsError(t("pdfAnki.deck.tagsError"));
			});
		return () => {
			cancelled = true;
		};
	}, [deckId, isAuthReady, userIdForApi, t]);

	const filteredCards = useMemo(
		() =>
			cards
				.map((card, index) => ({ card, index }))
				.filter(({ card }) => cardMatchesTagFilter(card, selectedTags)),
		[cards, selectedTags],
	);

	const handleCardDiscarded = useCallback(
		async (cardKey: string): Promise<void> => {
			const match = cardKey.match(/-f-(\d+)$/);
			const indexToken = match?.[1];
			if (indexToken) {
				const index = Number.parseInt(indexToken, 10);
				if (Number.isFinite(index) && index >= 0) {
					setCards((prev) => prev.filter((_, i) => i !== index));
				}
			}

			try {
				const refreshed = await fetchDeck();
				setDeck(refreshed);
				setCards(refreshed.flashcards);
				setNotFound(false);
				setLoadError(null);
			} catch {

			}
		},
		[fetchDeck],
	);

	const toggleTag = useCallback((tag: string) => {
		setSelectedTags((prev) => {
			const next = new Set(prev);
			if (next.has(tag)) {
				next.delete(tag);
			} else {
				next.add(tag);
			}
			return next;
		});
	}, []);

	const clearTags = useCallback(() => {
		setSelectedTags(new Set());
	}, []);

	const onExport = useCallback(async () => {
		if (!deckId || !userIdForApi) {
			return;
		}
		setExportError(null);
		setExporting(true);
		try {
			await downloadPdfDeckExport({
				deckId,
				userId: userIdForApi,
				format: exportFormat,
				lang: locale,
				tags: Array.from(selectedTags),
			});
		} catch {
			setExportError(t("pdfAnki.deck.exportError"));
		} finally {
			setExporting(false);
		}
	}, [
		deckId,
		userIdForApi,
		exportFormat,
		locale,
		selectedTags,
		t,
	]);

	if (loading) {
		return (
			<main className="flex min-h-screen flex-col bg-slate-100 text-slate-900">
				<header className="border-b border-slate-200 bg-white px-6 py-4">
					<Link
						href="/pdf-anki"
						className="text-sm font-medium text-slate-600 hover:text-slate-900"
					>
						← {t("pdfAnki.decks.backToMain")}
					</Link>
				</header>
				<section className="mx-auto w-full max-w-5xl px-6 py-8">
					<p className="text-sm text-slate-500">
						{!isAuthReady ? t("auth.loading") : t("pdfAnki.main.decksLoading")}
					</p>
				</section>
			</main>
		);
	}

	if (loadError) {
		return (
			<main className="flex min-h-screen flex-col bg-slate-100 text-slate-900">
				<header className="border-b border-slate-200 bg-white px-6 py-4">
					<Link
						href="/pdf-anki"
						className="text-sm font-medium text-slate-600 hover:text-slate-900"
					>
						← {t("pdfAnki.decks.backToMain")}
					</Link>
				</header>
				<section className="mx-auto w-full max-w-5xl px-6 py-8">
					<p className="text-sm text-rose-600">{loadError}</p>
				</section>
			</main>
		);
	}

	if (notFound || !deck) {
		return (
			<main className="flex min-h-screen flex-col bg-slate-100 text-slate-900">
				<header className="border-b border-slate-200 bg-white px-6 py-4">
					<Link
						href="/pdf-anki"
						className="text-sm font-medium text-slate-600 hover:text-slate-900"
					>
						← {t("pdfAnki.decks.backToMain")}
					</Link>
				</header>
				<section className="mx-auto w-full max-w-5xl px-6 py-8">
					<p className="text-sm text-rose-600">
						{t("pdfAnki.deck.notFound")}
					</p>
				</section>
			</main>
		);
	}

	return (
		<main className="flex min-h-screen flex-col bg-slate-100 text-slate-900">
			<header className="border-b border-slate-200 bg-white px-6 py-4">
				<div className="flex items-center justify-between gap-4">
					<div>
						<Link
							href="/pdf-anki"
							className="text-sm font-medium text-slate-600 hover:text-slate-900"
						>
							← {t("pdfAnki.decks.backToMain")}
						</Link>
						<h1 className="mt-2 text-2xl font-semibold">{deck.title}</h1>
						<p className="mt-1 text-sm text-slate-600">
							{filteredCards.length} / {cards.length}{" "}
							{t("pdfAnki.deck.cardsCount")}
						</p>
					</div>
				</div>
			</header>

			<section className="mx-auto flex w-full max-w-5xl flex-col gap-4 px-6 py-6 lg:flex-row lg:items-start">
				<aside className="w-full shrink-0 lg:w-48">
					<div className="rounded-2xl border border-slate-200 bg-white p-3 shadow-sm">
						<h2 className="text-xs font-semibold uppercase tracking-wide text-slate-500">
							{t("pdfAnki.deck.tagsTitle")}
						</h2>
						{selectedTags.size > 0 ? (
							<button
								type="button"
								onClick={clearTags}
								className="mt-3 w-full rounded-lg border border-slate-200 py-1.5 text-[11px] font-medium text-slate-600 hover:bg-slate-50"
							>
								{t("pdfAnki.deck.tagsClear")}
							</button>
						) : null}
						{tagsError ? (
							<p className="mt-2 text-xs text-rose-600">{tagsError}</p>
						) : tags.length === 0 ? (
							<p className="mt-2 text-xs text-slate-500">
								{t("pdfAnki.deck.tagsEmpty")}
							</p>
						) : (
							<ul className="mt-2 flex flex-col gap-1">
								{tags.map((tag) => {
									const active = selectedTags.has(tag);
									return (
										<li key={tag}>
											<button
												type="button"
												onClick={() => toggleTag(tag)}
												className={`w-full rounded-lg px-2 py-1.5 text-left text-xs font-medium transition ${
													active
														? "bg-slate-900 text-white"
														: "bg-slate-50 text-slate-700 hover:bg-slate-100"
												}`}
											>
												{tag}
											</button>
										</li>
									);
								})}
							</ul>
						)}
						{selectedTags.size > 0 ? (
							<button
								type="button"
								onClick={clearTags}
								className="mt-3 w-full rounded-lg border border-slate-200 py-1.5 text-[11px] font-medium text-slate-600 hover:bg-slate-50"
							>
								{t("pdfAnki.deck.tagsClear")}
							</button>
						) : null}
					</div>
				</aside>

				<div className="min-w-0 flex-1 space-y-4">
					<div
						id="pdf-anki-deck-review"
						className="rounded-2xl border border-slate-200 bg-white p-4 shadow-sm"
					>
						<h2 className="flex items-center gap-2 text-sm font-semibold text-slate-800">
							<span>{t("pdfAnki.deck.reviewTitle")}</span>
							<span
								ref={reviewInfoWrapRef}
								className="relative inline-flex"
								onMouseEnter={() => setReviewInfoHover(true)}
								onMouseLeave={() => setReviewInfoHover(false)}
							>
								<button
									type="button"
									className="inline-flex h-5 w-5 items-center justify-center rounded-full border border-slate-200 bg-white text-slate-600 hover:bg-slate-50"
									aria-label={t("pdfAnki.deck.reviewInfoAriaLabel")}
									aria-describedby={reviewInfoTooltipId}
									onClick={() => setReviewInfoPinned((v) => !v)}
								>
									<Info className="h-3.5 w-3.5" aria-hidden />
								</button>
								{reviewInfoOpen ? (
									<div
										id={reviewInfoTooltipId}
										role="tooltip"
										className="absolute left-0 top-full z-10 mt-2 w-[22rem] max-w-[80vw] rounded-lg border border-slate-200 bg-white p-3 text-xs text-slate-800 shadow-lg"
									>
										<p className="whitespace-pre-line">
											{t("pdfAnki.deck.reviewInfoTooltip")}
										</p>
									</div>
								) : null}
							</span>
						</h2>
						{filteredCards.length === 0 ? (
							<p className="mt-2 text-xs text-slate-500">
								{t("pdfAnki.deck.filteredEmpty")}
							</p>
						) : (
							<div className="mt-3 grid max-h-[600px] grid-cols-1 gap-4 overflow-auto md:grid-cols-2">
								{filteredCards.map(({ card, index }) => (
									<PdfAnkiFlashcard
										key={`${deck.id}-f-${index}`}
										cardKey={`${deck.id}-f-${index}`}
										card={card}
										deckId={deck.id}
										onDiscarded={handleCardDiscarded}
									/>
								))}
							</div>
						)}
					</div>

					<div className="rounded-2xl border border-slate-200 bg-white p-4 shadow-sm">
						<h2 className="text-sm font-semibold text-slate-800">
							{t("pdfAnki.deck.exportTitle")}
						</h2>
						<p className="mt-1 text-xs text-slate-500">
							{t("pdfAnki.deck.exportHint")}
						</p>
						<div className="mt-3 flex flex-wrap items-center gap-3">
							<label className="flex items-center gap-2 text-xs text-slate-700">
								<span>{t("pdfAnki.deck.exportFormat")}</span>
								<select
									className="rounded-md border border-slate-300 bg-white px-2 py-1 text-xs shadow-sm"
									value={exportFormat}
									onChange={(e) =>
										setExportFormat(e.target.value as PdfDeckExportFormat)
									}
								>
									<option value="apkg">.apkg</option>
									<option value="csv">.csv</option>
									<option value="xlsx">.xlsx</option>
									<option value="txt">.txt</option>
								</select>
							</label>
							<button
								type="button"
								disabled={exporting || filteredCards.length === 0}
								onClick={() => void onExport()}
								className="rounded-lg bg-slate-800 px-3 py-2 text-xs font-medium text-white shadow-sm transition hover:bg-slate-700 disabled:cursor-not-allowed disabled:opacity-50"
							>
								{exporting
									? t("pdfAnki.deck.exportLoading")
									: t("pdfAnki.deck.exportButton")}
							</button>
						</div>
						{exportError ? (
							<p className="mt-2 text-xs text-rose-600">{exportError}</p>
						) : null}
					</div>
				</div>
			</section>
		</main>
	);
}
```

### `src/app/pdf-anki/decks/page.tsx`
```tsx
"use client";

import Link from "next/link";
import { useCallback, useEffect, useState } from "react";
import { apiRequest } from "@/lib/api-client";
import { useI18n } from "@/app/i18n-provider";
import {
	type DeckListItem,
	decksListResponseSchema
} from "@/modules/pdf-anki/schemas";
import { usePdfAnkiResolvedUserId } from "@/modules/pdf-anki/usePdfAnkiAuth";
import { ConfirmDialog } from "@/components/ConfirmDialog";
import { Trash2 } from "lucide-react";

function pdfAnkiUserScopedPath(path: string, userId: string): string {
	const params = new URLSearchParams({ user_id: userId });
	return `${path}?${params.toString()}`;
}

export default function PdfAnkiDecksPage() {
	const { t } = useI18n();
	const { isAuthReady, userIdForApi } = usePdfAnkiResolvedUserId();
	const [decks, setDecks] = useState<DeckListItem[]>([]);
	const [loading, setLoading] = useState(true);
	const [error, setError] = useState<string | null>(null);
	const [successMessage, setSuccessMessage] = useState<string | null>(null);
	const [deletingDeckId, setDeletingDeckId] = useState<string | null>(null);
	const [deleteConfirmDialog, setDeleteConfirmDialog] = useState<{
		isOpen: boolean;
		deckId: string | null;
		title: string;
	}>({ isOpen: false, deckId: null, title: "" });

	useEffect(() => {
		let cancelled = false;
		if (!isAuthReady || !userIdForApi) {
			setLoading(true);
			setError(null);
			return () => {
				cancelled = true;
			};
		}
		setLoading(true);
		setError(null);
		setSuccessMessage(null);
		apiRequest<unknown>(pdfAnkiUserScopedPath("/pdf/decks", userIdForApi))
			.then((raw) => {
				if (cancelled) return;
				const parsed = decksListResponseSchema.parse(raw);
				setDecks(parsed);
			})
			.catch(() => {
				if (!cancelled) setError(t("pdfAnki.decks.errorLoad"));
			})
			.finally(() => {
				if (!cancelled) setLoading(false);
			});
		return () => {
			cancelled = true;
		};
	}, [t, isAuthReady, userIdForApi]);

	const handleDeleteDeck = useCallback(
		(deckId: string, title: string): void => {
			if (!isAuthReady || !userIdForApi) return;
			setDeleteConfirmDialog({ isOpen: true, deckId, title });
		},
		[isAuthReady, userIdForApi],
	);

	const handleConfirmDeleteDeck = useCallback(
		async (): Promise<void> => {
			const deckId = deleteConfirmDialog.deckId;
			if (!deckId || !isAuthReady || !userIdForApi) return;

			setDeleteConfirmDialog({ isOpen: false, deckId: null, title: "" });
			setDeletingDeckId(deckId);
			setError(null);
			setSuccessMessage(null);
			try {
				await apiRequest<unknown>(
					pdfAnkiUserScopedPath(
						`/pdf/deck/${encodeURIComponent(deckId)}`,
						userIdForApi,
					),
					{ method: "DELETE" },
				);
				setDecks((prev) => prev.filter((d) => d.id !== deckId));
				setSuccessMessage(t("pdfAnki.decks.deleteSuccess"));
			} catch (err) {
				console.error(err);
				setError(t("pdfAnki.decks.deleteError"));
			} finally {
				setDeletingDeckId(null);
			}
		},
		[deleteConfirmDialog.deckId, isAuthReady, userIdForApi],
	);

	const handleCancelDeleteDeck = useCallback(() => {
		setDeleteConfirmDialog({ isOpen: false, deckId: null, title: "" });
	}, []);

	const getDeckStatusLabel = useCallback(
		(status: string): string => {
			if (status === "completed_partial" || status === "partial_completed") {
				return t("pdfAnki.decks.status.completedPartial");
			}
			if (status === "completed") {
				return t("pdfAnki.decks.status.completed");
			}
			return status;
		},
		[t],
	);

	const isDeckCompletedLike = useCallback((status: string | undefined): boolean => {
		if (!status) return false;
		return (
			status === "completed" ||
			status === "completed_partial" ||
			status === "partial_completed"
		);
	}, []);

	return (
		<main className="flex min-h-screen flex-col bg-slate-100 text-slate-900">
			<header className="border-b border-slate-200 bg-white px-6 py-4">
				<div className="flex items-center justify-between gap-4">
					<div>
						<h1 className="text-2xl font-semibold">
							{t("pdfAnki.main.myDecks")}
						</h1>
						<p className="mt-1 max-w-2xl text-sm text-slate-600">
							{t("pdfAnki.decks.subtitle")}
						</p>
					</div>
					<Link
						href="/pdf-anki"
						className="inline-flex items-center rounded-full border border-slate-300 bg-white px-3 py-1.5 text-xs font-medium text-slate-700 shadow-sm transition hover:border-slate-400 hover:text-slate-900"
					>
						{t("pdfAnki.decks.backToMain")}
					</Link>
				</div>
			</header>

			<section className="mx-auto w-full max-w-3xl px-6 py-6">
				<div className="rounded-2xl border border-slate-200 bg-white p-4 shadow-sm">
					{successMessage ? (
						<p className="mb-3 text-sm text-emerald-700">{successMessage}</p>
					) : null}
					{loading ? (
						<p className="text-sm text-slate-500">
							{!isAuthReady
								? t("auth.loading")
								: t("pdfAnki.main.decksLoading")}
						</p>
					) : error ? (
						<p className="text-sm text-rose-600">{error}</p>
					) : decks.length === 0 ? (
						<p className="text-sm text-slate-500">
							{t("pdfAnki.main.decksEmpty")}
						</p>
					) : (
						<ul className="space-y-2">
							{decks.map((d) => (
								<li key={d.id} className="flex items-start gap-2">
									<Link
										href={`/pdf-anki/deck/${d.id}`}
										className="flex-1 rounded-lg border border-slate-100 bg-slate-50 px-4 py-3 text-sm text-slate-800 transition hover:border-slate-200 hover:bg-slate-100"
									>
										<span className="font-medium">{d.title}</span>
										<span className="ml-2 text-slate-500">
											{new Date(d.created_at).toLocaleDateString()}
										</span>
										{d.status ? (
											<span className="ml-2 rounded bg-slate-200 px-2 py-0.5 text-xs text-slate-600">
												{getDeckStatusLabel(d.status)}
											</span>
										) : null}
									</Link>

									{isDeckCompletedLike(d.status) ? (
										<button
											type="button"
											aria-label={t("pdfAnki.decks.deleteAriaLabel", {
												title: d.title,
											})}
											disabled={deletingDeckId === d.id}
											onClick={() => {
												void handleDeleteDeck(d.id, d.title);
											}}
											className="mt-1 inline-flex h-9 w-9 items-center justify-center rounded-lg border border-rose-200 bg-white text-rose-500 shadow-sm transition hover:bg-rose-50 disabled:opacity-40"
										>
											<Trash2 size={18} />
										</button>
									) : null}
								</li>
							))}
						</ul>
					)}
				</div>
			</section>

			<ConfirmDialog
				isOpen={deleteConfirmDialog.isOpen}
				title={t("pdfAnki.decks.deleteConfirmTitle")}
				message={t("pdfAnki.decks.deleteConfirm", {
					title: deleteConfirmDialog.title,
				})}
				confirmText={t("common.delete")}
				cancelText={t("common.cancel")}
				isDangerous
				onConfirm={handleConfirmDeleteDeck}
				onCancel={handleCancelDeleteDeck}
			/>
		</main>
	);
}
```

### `src/app/pdf-anki/pricing/page.tsx`
```tsx
"use client";

import { useEffect, useState } from "react";
import Link from "next/link";
import { useI18n } from "@/app/i18n-provider";
import { useAuthMe, useUserId } from "@/app/AuthProvider";
import { personalizeStripeLink } from "@/lib/stripe-utils";
import { getBackendUrl } from "@/app/auth/backend-auth-api";
import {
	fetchPdfAnkiPricing,
	type PdfAnkiPricingResponse,
} from "@/app/pdf-anki/pdf-anki-pricing-api";

const walletCheckoutUrl =
	typeof process !== "undefined"
		? process.env.NEXT_PUBLIC_PDF_ANKI_WALLET_CHECKOUT_URL?.trim() ?? ""
		: "";

const FALLBACK_PORTAL_PATH = "/billing/stripe/customer-portal";

/** Resolves the Stripe customer-portal URL from a backend-provided path or fallback. */
function resolvePortalUrl(customerPortalPath: string | undefined): string {
	const path = customerPortalPath?.trim();
	if (!path) return getBackendUrl(FALLBACK_PORTAL_PATH);
	if (path.startsWith("http://") || path.startsWith("https://")) return path;
	return getBackendUrl(path);
}

const tiers = [
	{
		id: "free",
		nameKey: "pdfAnki.pricing.tier.free.name",
		badgeKey: "pdfAnki.pricing.tier.free.badge",
		priceKey: "pdfAnki.pricing.tier.free.price",
		descriptionKey: "pdfAnki.pricing.tier.free.description",
		highlightKeys: [
			"pdfAnki.pricing.tier.free.highlight.1",
			"pdfAnki.pricing.tier.free.highlight.2",
			"pdfAnki.pricing.tier.free.highlight.3",
			"pdfAnki.pricing.tier.free.highlight.4",
			"pdfAnki.pricing.tier.free.highlight.5",
		],
		accent: "from-emerald-500/20 via-emerald-400/10 to-white",
		badgeColor: "bg-emerald-100 text-emerald-700",
		ctaKey: "",
	},
	{
		id: "premium",
		nameKey: "pdfAnki.pricing.tier.premium.name",
		badgeKey: "pdfAnki.pricing.tier.premium.badge",
		priceKey: "pdfAnki.pricing.tier.premium.price",
		descriptionKey: "pdfAnki.pricing.tier.premium.description",
		highlightKeys: [
			"pdfAnki.pricing.tier.premium.highlight.1",
			"pdfAnki.pricing.tier.premium.highlight.2",
			"pdfAnki.pricing.tier.premium.highlight.3",
			"pdfAnki.pricing.tier.premium.highlight.4",
			"pdfAnki.pricing.tier.premium.highlight.5",
		],
		accent: "from-sky-500/20 via-sky-400/10 to-white",
		badgeColor: "bg-sky-100 text-sky-700",
		ctaKey: "pdfAnki.pricing.subscribe",
	},
	{
		id: "pro",
		nameKey: "pdfAnki.pricing.tier.pro.name",
		badgeKey: "pdfAnki.pricing.tier.pro.badge",
		priceKey: "pdfAnki.pricing.tier.pro.price",
		descriptionKey: "pdfAnki.pricing.tier.pro.description",
		highlightKeys: [
			"pdfAnki.pricing.tier.pro.highlight.1",
			"pdfAnki.pricing.tier.pro.highlight.2",
			"pdfAnki.pricing.tier.pro.highlight.3",
			"pdfAnki.pricing.tier.pro.highlight.4",
			"pdfAnki.pricing.tier.pro.highlight.5",
		],
		accent: "from-amber-500/20 via-amber-400/10 to-white",
		badgeColor: "bg-amber-100 text-amber-700",
		ctaKey: "pdfAnki.pricing.subscribe",
	},
	{
		id: "payg",
		nameKey: "pdfAnki.pricing.tier.payg.name",
		badgeKey: "pdfAnki.pricing.tier.payg.badge",
		priceKey: "pdfAnki.pricing.tier.payg.price",
		descriptionKey: "pdfAnki.pricing.tier.payg.description",
		highlightKeys: [
			"pdfAnki.pricing.tier.payg.highlight.1",
			"pdfAnki.pricing.tier.payg.highlight.2",
			"pdfAnki.pricing.tier.payg.highlight.3",
			"pdfAnki.pricing.tier.payg.highlight.4",
			"pdfAnki.pricing.tier.payg.highlight.5",
		],
		accent: "from-violet-500/20 via-violet-400/10 to-white",
		badgeColor: "bg-violet-100 text-violet-700",
		ctaKey: "pdfAnki.pricing.topUp",
	},
];

const genericStripeUrl = "https://buy.stripe.com/replace-with-real-link-later";

export default function PdfAnkiPricingPage() {
	const { t, locale } = useI18n();
	const userId = useUserId();
	const me = useAuthMe();
	const currentTier = (me?.tier ?? "").trim().toLowerCase();

	const [data, setData] = useState<PdfAnkiPricingResponse | null>(null);

	useEffect(() => {
		let cancelled = false;
		fetchPdfAnkiPricing()
			.then((res) => {
				if (!cancelled) setData(res);
			})
			.catch(() => {
				if (!cancelled) setData(null);
			});
		return () => {
			cancelled = true;
		};
	}, []);

	const { stripe_links } = data ?? { stripe_links: {} };

	const paygTopupLink =
		stripe_links.payg_topup ?? stripe_links.payg ?? (walletCheckoutUrl || undefined);

	const linkByTierId: Record<string, string | undefined> = {
		premium: stripe_links.premium,
		pro: stripe_links.pro,
		payg: paygTopupLink,
	};

	const portalUrl = resolvePortalUrl(data?.customer_portal_path);

	/**
	 * The user is a paid PDF-Anki subscriber when:
	 * - auth/me identifies them as the pdf_anki module
	 * - their tier is premium or pro
	 * - they have a stripe_customer_id (means Stripe subscription exists)
	 * If auth/me hasn't loaded yet (null), treat as unauthenticated.
	 */
	const isPdfPaidSubscriber =
		me?.module === "pdf_anki" &&
		(me?.tier === "premium" || me?.tier === "pro") &&
		!!me?.stripe_customer_id;

	return (
		<main className="flex min-h-screen flex-col bg-slate-100 text-slate-900">
			<header className="border-b border-slate-200 bg-white px-6 py-4">
				<div className="mx-auto flex w-full max-w-5xl items-center justify-between gap-4">
					<div>
						<h1 className="text-2xl font-semibold">
							{t("pdfAnki.pricing.title")}
						</h1>
						<p className="mt-1 max-w-2xl text-sm text-slate-600">
							{t("pdfAnki.pricing.subtitle")}
						</p>
					</div>
					<Link
						href="/pdf-anki"
						className="inline-flex items-center rounded-full border border-slate-300 bg-white px-3 py-1.5 text-xs font-medium text-slate-700 shadow-sm transition hover:border-slate-400 hover:text-slate-900"
					>
						{t("pdfAnki.pricing.backToMain")}
					</Link>
				</div>
			</header>

			<section className="mx-auto flex w-full max-w-5xl flex-1 flex-col gap-6 px-6 py-8">
				<div className="grid gap-5 sm:grid-cols-2 lg:grid-cols-4">
					{tiers.map((tier) => {
						const isCurrent = currentTier === tier.id;

						let button: React.ReactNode = null;

						if (tier.id === "free") {
							if (currentTier === "free") {
								button = (
									<span className="inline-flex items-center justify-center rounded-full border border-slate-300 bg-slate-100 px-4 py-2 text-xs font-medium text-slate-700">
										{t("pdfAnki.pricing.currentPlan")}
									</span>
								);
							}
						} else if (tier.id === "premium" || tier.id === "pro") {
							if (isPdfPaidSubscriber) {

								button = (
									<a
										href={portalUrl}
										target="_blank"
										rel="noopener noreferrer"
										className="inline-flex items-center justify-center rounded-full bg-slate-900 px-4 py-2 text-xs font-medium text-white transition hover:bg-slate-800"
									>
										{t("common.manageInStripe")}
									</a>
								);
							} else {

								const rawUrl = linkByTierId[tier.id] ?? genericStripeUrl;
								const personalizedUrl = personalizeStripeLink(
									rawUrl,
									userId,
									locale,
									tier.id,
									"pdf_anki",
								);
								button = isCurrent ? (
									<a
										href={personalizedUrl}
										target="_blank"
										rel="noopener noreferrer"
										className="inline-flex items-center justify-center rounded-full border border-slate-300 bg-slate-100 px-4 py-2 text-xs font-medium text-slate-700 transition hover:bg-slate-200"
									>
										{t("pdfAnki.pricing.currentPlan")}
									</a>
								) : (
									<a
										href={personalizedUrl}
										target="_blank"
										rel="noopener noreferrer"
										className="inline-flex items-center justify-center rounded-full bg-slate-900 px-4 py-2 text-xs font-medium text-white transition hover:bg-slate-800"
									>
										{t(tier.ctaKey)}
									</a>
								);
							}
						} else if (tier.id === "payg") {

							const rawUrl = paygTopupLink ?? genericStripeUrl;
							const personalizedUrl = personalizeStripeLink(
								rawUrl,
								userId,
								locale,
								tier.id,
								"pdf_anki",
							);
							button = (
								<a
									href={personalizedUrl}
									target="_blank"
									rel="noopener noreferrer"
									className="inline-flex items-center justify-center rounded-full bg-slate-900 px-4 py-2 text-xs font-medium text-white transition hover:bg-slate-800"
								>
									{t(tier.ctaKey)}
								</a>
							);
						}

						return (
							<article
								key={tier.id}
								className="flex flex-col overflow-hidden rounded-2xl border border-slate-200 bg-white shadow-sm"
							>
								<div className={`bg-gradient-to-br ${tier.accent} px-4 py-3`}>
									<div className="flex items-center justify-between gap-2">
										<h2 className="text-sm font-semibold text-slate-900">
											{t(tier.nameKey)}
										</h2>
										<span className={`rounded-full px-2 py-0.5 text-[10px] font-medium uppercase tracking-wide ${tier.badgeColor}`}>
											{t(tier.badgeKey)}
										</span>
									</div>
									<p className="mt-1 text-base font-semibold text-slate-900">
										{t(tier.priceKey)}
									</p>
								</div>
								<div className="flex flex-1 flex-col p-4">
									<p className="text-xs text-slate-600">
										{t(tier.descriptionKey)}
									</p>
									<ul className="mt-3 flex flex-1 flex-col gap-1 text-xs text-slate-700">
										{tier.highlightKeys.map((key) => (
											<li key={key} className="flex items-start gap-1.5">
												<span className="mt-0.5 h-1.5 w-1.5 flex-shrink-0 rounded-full bg-emerald-500" />
												<span>{t(key)}</span>
											</li>
										))}
									</ul>
									{button && (
										<div className="mt-4 flex justify-end">
											{button}
										</div>
									)}
								</div>
							</article>
						);
					})}
				</div>

				<p className="mt-2 text-[11px] text-slate-500">
					{t("pdfAnki.pricing.enterpriseFooter")}
				</p>
			</section>
		</main>
	);
}
```

### `src/app/pdf-anki/privacy/page.tsx`
```tsx
"use client";

import { useI18n } from "@/app/i18n-provider";
import { LegalPageTemplate } from "@/components/LegalPageTemplate";
import type { Locale } from "@/lib/i18n-config";

type PrivacyPolicyContent = {
	title: string;
	description: string;
	lastUpdated: string;
	sections: { title: string; body: string }[];
};

const policies: Record<Locale, PrivacyPolicyContent> = {
	en: {
		title: "Privacy Policy (PDF to Anki)",
		description: "Your privacy and data security are fundamental to us. This policy explains how we collect, protect, and process your information using enterprise-grade security standards (SOC2, GDPR).",
		lastUpdated: "Last updated: April 6, 2026",
		sections: [
			{
				title: "1. Data We Collect",
				body: "We collect your account information (name, email, avatar) via Google OAuth. We also process the PDF documents you voluntarily upload and usage telemetry data to ensure system performance and correct billing (FinOps).",
			},
			{
				title: "2. Data Protection and Encryption (ALE)",
				body: "We implement Application-Level Encryption (ALE). This means sensitive data (such as the content of your generated flashcards and deck titles) is encrypted on our servers before being saved to the database. Not even our database administrators can read your information in clear text.",
			},
			{
				title: "3. AI Processing and Data Loss Prevention (DLP)",
				body: "To provide our services, we use language models (LLMs) hosted on secure infrastructures. To ensure your privacy, we use a Data Loss Prevention (DLP Proxy) system and anonymization (PII Scrubbing). Before any sensitive text from your PDFs leaves our infrastructure to an external AI provider, we detect and hide personally identifiable information (names, emails, card numbers, SSNs).",
			},
			{
				title: "4. Zero Data Retention & Third-Party AI",
				body: "Our data sharing policies depend on your subscription tier:\n\n- Free Tiers: We use providers like Groq and LlamaParse. Please note that data processed through these free tiers MAY be used by these providers to train their models.\n- Paid Tiers (Premium, Pro, PAYG): We use Google Vertex AI. We maintain strict agreements ensuring Zero Data Retention. Data sent through these paid APIs is NOT used to train foundational models and is discarded after processing.",
			},
			{
				title: "5. Data Retention and Deletion",
				body: "Your data belongs to you. You can delete your flashcard decks at any time from the interface.\n\nIf you cancel your paid subscription, we apply a 90-day grace period. After this time, an automated process will permanently delete your old data and search vectors to comply with data minimization principles.",
			},
		],
	},
	es: {
		title: "Política de Privacidad (PDF a Anki)",
		description: "Tu privacidad y la seguridad de tus datos son fundamentales para nosotros. Esta política explica cómo recopilamos, protegemos y procesamos tu información utilizando estándares de seguridad de nivel empresarial (SOC2, GDPR).",
		lastUpdated: "Última actualización: 6 de abril de 2026",
		sections: [
			{
				title: "1. Datos que recopilamos",
				body: "Recopilamos información de tu cuenta (nombre, email, avatar) a través de Google OAuth. También procesamos los documentos PDF que subes voluntariamente y los datos de telemetría de uso para garantizar el rendimiento del sistema y la facturación correcta (FinOps).",
			},
			{
				title: "2. Protección de Datos y Cifrado (ALE)",
				body: "Implementamos Cifrado a Nivel de Aplicación (Application-Level Encryption - ALE). Esto significa que los datos sensibles (como el contenido de tus flashcards y los títulos de tus mazos) se cifran en nuestros servidores antes de guardarse en la base de datos. Ni siquiera nuestros administradores de bases de datos pueden leer tu información en texto claro.",
			},
			{
				title: "3. Procesamiento de IA y Prevención de Pérdida de Datos (DLP)",
				body: "Para proporcionar nuestros servicios, utilizamos modelos de lenguaje (LLMs) alojados en infraestructuras seguras. Para garantizar tu privacidad, utilizamos un sistema de Prevención de Pérdida de Datos (DLP Proxy) y anonimización (PII Scrubbing). Antes de que cualquier texto sensible de tus PDFs salga de nuestra infraestructura hacia un proveedor de IA externo, detectamos y ocultamos información personal identificable (nombres, emails, números de tarjeta, SSN).",
			},
			{
				title: "4. Zero Data Retention y Proveedores de IA",
				body: "Nuestras políticas de compartición de datos dependen de tu plan de suscripción:\n\n- Planes Gratuitos (Free Tier): Utilizamos proveedores como Groq y LlamaParse. Ten en cuenta que los datos procesados a través de estos niveles gratuitos SÍ pueden ser utilizados por estos proveedores para entrenar sus modelos.\n- Planes de Pago (Premium, Pro, PAYG): Utilizamos Google Vertex AI. Mantenemos acuerdos estrictos que garantizan 'Zero Data Retention'. Los datos enviados a través de estas APIs de pago NO se utilizan para entrenar modelos fundacionales y se descartan tras el procesamiento.",
			},
			{
				title: "5. Retención y Eliminación de Datos",
				body: "Tus datos te pertenecen. Puedes eliminar tus mazos de tarjetas en cualquier momento desde la interfaz.\n\nSi cancelas tu suscripción de pago, aplicamos un período de gracia de 90 días. Pasado este tiempo, un proceso automatizado eliminará permanentemente tus datos antiguos y vectores de búsqueda para cumplir con los principios de minimización de datos.",
			},
		],
	},
	de: {
		title: "Datenschutzrichtlinie (PDF zu Anki)",
		description: "Ihre Privatsphäre und Datensicherheit sind für uns von grundlegender Bedeutung. Diese Richtlinie erklärt, wie wir Ihre Informationen unter Verwendung von Sicherheitsstandards auf Unternehmensebene (SOC2, DSGVO) erfassen, schützen und verarbeiten.",
		lastUpdated: "Zuletzt aktualisiert: 6. April 2026",
		sections: [
			{
				title: "1. Daten, die wir erfassen",
				body: "Wir erfassen Ihre Kontoinformationen (Name, E-Mail, Avatar) über Google OAuth. Wir verarbeiten auch die PDF-Dokumente, die Sie freiwillig hochladen, sowie Nutzungs-Telemetriedaten, um die Systemleistung und korrekte Abrechnung (FinOps) sicherzustellen.",
			},
			{
				title: "2. Datenschutz und Verschlüsselung (ALE)",
				body: "Wir implementieren Application-Level Encryption (ALE). Das bedeutet, dass sensible Daten (wie der Inhalt Ihrer Karteikarten und Decktitel) auf unseren Servern verschlüsselt werden, bevor sie in der Datenbank gespeichert werden. Nicht einmal unsere Datenbankadministratoren können Ihre Informationen im Klartext lesen.",
			},
			{
				title: "3. KI-Verarbeitung und Data Loss Prevention (DLP)",
				body: "Um unsere Dienste bereitzustellen, verwenden wir Sprachmodelle (LLMs), die auf sicheren Infrastrukturen gehostet werden. Um Ihre Privatsphäre zu gewährleisten, verwenden wir ein Data Loss Prevention (DLP Proxy) System und Anonymisierung (PII Scrubbing). Bevor sensibler Text aus Ihren PDFs unsere Infrastruktur zu einem externen KI-Anbieter verlässt, erkennen und verbergen wir persönlich identifizierbare Informationen (Namen, E-Mails, Kartennummern, SSNs).",
			},
			{
				title: "4. Zero Data Retention & Drittanbieter-KI",
				body: "Unsere Datenaustauschrichtlinien hängen von Ihrer Abonnementstufe ab:\n\n- Kostenlose Stufen: Wir verwenden Anbieter wie Groq und LlamaParse. Bitte beachten Sie, dass Daten, die über diese kostenlosen Stufen verarbeitet werden, von diesen Anbietern zum Trainieren ihrer Modelle verwendet werden KÖNNEN.\n- Kostenpflichtige Stufen (Premium, Pro, PAYG): Wir verwenden Google Vertex AI. Wir unterhalten strenge Vereinbarungen, die Zero Data Retention gewährleisten. Daten, die über diese kostenpflichtigen APIs gesendet werden, werden NICHT zum Trainieren von Basismodellen verwendet und nach der Verarbeitung verworfen.",
			},
			{
				title: "5. Datenaufbewahrung und -löschung",
				body: "Ihre Daten gehören Ihnen. Sie können Ihre Kartendecks jederzeit über die Benutzeroberfläche löschen.\n\nWenn Sie Ihr kostenpflichtiges Abonnement kündigen, gewähren wir eine Nachfrist von 90 Tagen. Nach dieser Zeit löscht ein automatisierter Prozess Ihre alten Daten und Suchvektoren dauerhaft, um den Grundsätzen der Datenminimierung zu entsprechen.",
			},
		],
	},
	it: {
		title: "Informativa sulla Privacy (PDF ad Anki)",
		description: "La tua privacy e la sicurezza dei tuoi dati sono fondamentali per noi. Questa politica spiega come raccogliamo, proteggiamo e trattiamo le tue informazioni utilizzando standard di sicurezza di livello aziendale (SOC2, GDPR).",
		lastUpdated: "Ultimo aggiornamento: 6 aprile 2026",
		sections: [
			{
				title: "1. Dati che raccogliamo",
				body: "Raccogliamo le informazioni del tuo account (nome, email, avatar) tramite Google OAuth. Trattiamo anche i documenti PDF che carichi volontariamente e i dati di telemetria di utilizzo per garantire le prestazioni del sistema e la corretta fatturazione (FinOps).",
			},
			{
				title: "2. Protezione dei Dati e Crittografia (ALE)",
				body: "Implementiamo la crittografia a livello di applicazione (ALE). Ciò significa che i dati sensibili (come il contenuto delle tue flashcard e i titoli dei mazzi) vengono crittografati sui nostri server prima di essere salvati nel database. Nemmeno i nostri amministratori di database possono leggere le tue informazioni in chiaro.",
			},
			{
				title: "3. Elaborazione AI e Prevenzione della Perdita di Dati (DLP)",
				body: "Per fornire i nostri servizi, utilizziamo modelli linguistici (LLM) ospitati su infrastrutture sicure. Per garantire la tua privacy, utilizziamo un sistema di Data Loss Prevention (DLP Proxy) e anonimizzazione (PII Scrubbing). Prima che qualsiasi testo sensibile dei tuoi PDF lasci la nostra infrastruttura verso un fornitore AI esterno, rileviamo e nascondiamo le informazioni di identificazione personale (nomi, email, numeri di carta, SSN).",
			},
			{
				title: "4. Zero Data Retention e AI di Terze Parti",
				body: "Le nostre politiche di condivisione dei dati dipendono dal tuo livello di abbonamento:\n\n- Livelli Gratuiti: Utilizziamo fornitori come Groq e LlamaParse. Tieni presente che i dati elaborati tramite questi livelli gratuiti POSSONO essere utilizzati da questi fornitori per addestrare i loro modelli.\n- Livelli a Pagamento (Premium, Pro, PAYG): Utilizziamo Google Vertex AI. Manteniamo accordi rigorosi che garantiscono la Zero Data Retention. I dati inviati tramite queste API a pagamento NON vengono utilizzati per addestrare modelli fondamentali e vengono scartati dopo l'elaborazione.",
			},
			{
				title: "5. Conservazione e Cancellazione dei Dati",
				body: "I tuoi dati ti appartengono. Puoi eliminare i tuoi mazzi di carte in qualsiasi momento dall'interfaccia.\n\nSe annulli il tuo abbonamento a pagamento, applichiamo un periodo di grazia di 90 giorni. Trascorso questo tempo, un processo automatizzato eliminerà in modo permanente i tuoi vecchi dati e i vettori di ricerca per conformarsi ai principi di minimizzazione dei dati.",
			},
		],
	},
	fr: {
		title: "Politique de Confidentialité (PDF vers Anki)",
		description: "Votre vie privée et la sécurité de vos données sont fondamentales pour nous. Cette politique explique comment nous collectons, protégeons et traitons vos informations en utilisant des normes de sécurité d'entreprise (SOC2, RGPD).",
		lastUpdated: "Dernière mise à jour : 6 avril 2026",
		sections: [
			{
				title: "1. Données que nous collectons",
				body: "Nous collectons les informations de votre compte (nom, e-mail, avatar) via Google OAuth. Nous traitons également les documents PDF que vous téléchargez volontairement et les données de télémétrie d'utilisation pour garantir les performances du système et une facturation correcte (FinOps).",
			},
			{
				title: "2. Protection des Données et Chiffrement (ALE)",
				body: "Nous mettons en œuvre le chiffrement au niveau de l'application (ALE). Cela signifie que les données sensibles (telles que le contenu de vos flashcards et les titres de vos paquets) sont chiffrées sur nos serveurs avant d'être enregistrées dans la base de données. Même nos administrateurs de base de données ne peuvent pas lire vos informations en clair.",
			},
			{
				title: "3. Traitement IA et Prévention des Pertes de Données (DLP)",
				body: "Pour fournir nos services, nous utilisons des modèles de langage (LLM) hébergés sur des infrastructures sécurisées. Pour garantir votre vie privée, nous utilisons un système de prévention des pertes de données (DLP Proxy) et d'anonymisation (PII Scrubbing). Avant que tout texte sensible de vos PDF ne quitte notre infrastructure vers un fournisseur d'IA externe, nous détectons et masquons les informations personnellement identifiables (noms, e-mails, numéros de carte, SSN).",
			},
			{
				title: "4. Zéro Rétention de Données et IA Tiers",
				body: "Nos politiques de partage de données dépendent de votre niveau d'abonnement :\n\n- Niveaux Gratuits : Nous utilisons des fournisseurs comme Groq et LlamaParse. Veuillez noter que les données traitées via ces niveaux gratuits PEUVENT être utilisées par ces fournisseurs pour entraîner leurs modèles.\n- Niveaux Payants (Premium, Pro, PAYG) : Nous utilisons Google Vertex AI. Nous maintenons des accords stricts garantissant la Zéro Rétention de Données. Les données envoyées via ces API payantes ne sont PAS utilisées pour entraîner des modèles fondamentaux et sont supprimées après traitement.",
			},
			{
				title: "5. Rétention et Suppression des Données",
				body: "Vos données vous appartiennent. Vous pouvez supprimer vos paquets de cartes à tout moment depuis l'interface.\n\nSi vous annulez votre abonnement payant, nous appliquons un délai de grâce de 90 jours. Passé ce délai, un processus automatisé supprimera définitivement vos anciennes données et vecteurs de recherche pour se conformer aux principes de minimisation des données.",
			},
		],
	},
	ru: {
		title: "Политика конфиденциальности (PDF в Anki)",
		description: "Ваша конфиденциальность и безопасность данных имеют для нас фундаментальное значение. Эта политика объясняет, как мы собираем, защищаем и обрабатываем вашу информацию, используя стандарты безопасности корпоративного уровня (SOC2, GDPR).",
		lastUpdated: "Последнее обновление: 6 апреля 2026 г.",
		sections: [
			{
				title: "1. Данные, которые мы собираем",
				body: "Мы собираем информацию о вашей учетной записи (имя, email, аватар) через Google OAuth. Мы также обрабатываем PDF-документы, которые вы добровольно загружаете, и данные телеметрии использования для обеспечения производительности системы и правильного биллинга (FinOps).",
			},
			{
				title: "2. Защита данных и шифрование (ALE)",
				body: "Мы внедряем шифрование на уровне приложения (ALE). Это означает, что конфиденциальные данные (такие как содержимое ваших карточек и названия колод) шифруются на наших серверах перед сохранением в базу данных. Даже наши администраторы баз данных не могут прочитать вашу информацию в открытом виде.",
			},
			{
				title: "3. Обработка ИИ и предотвращение потери данных (DLP)",
				body: "Для предоставления наших услуг мы используем языковые модели (LLM), размещенные на защищенных инфраструктурах. Для обеспечения вашей конфиденциальности мы используем систему предотвращения потери данных (DLP Proxy) и анонимизацию (PII Scrubbing). Прежде чем любой конфиденциальный текст из ваших PDF покинет нашу инфраструктуру и отправится к внешнему поставщику ИИ, мы обнаруживаем и скрываем личную информацию (имена, email, номера карт, SSN).",
			},
			{
				title: "4. Нулевое хранение данных и сторонний ИИ",
				body: "Наши политики обмена данными зависят от уровня вашей подписки:\n\n- Бесплатные уровни: Мы используем таких поставщиков, как Groq и LlamaParse. Обратите внимание, что данные, обрабатываемые через эти бесплатные уровни, МОГУТ использоваться этими поставщиками для обучения их моделей.\n- Платные уровни (Premium, Pro, PAYG): Мы используем Google Vertex AI. Мы поддерживаем строгие соглашения, гарантирующие нулевое хранение данных. Данные, отправляемые через эти платные API, НЕ используются для обучения базовых моделей и удаляются после обработки.",
			},
			{
				title: "5. Хранение и удаление данных",
				body: "Ваши данные принадлежат вам. Вы можете удалить свои колоды карточек в любое время из интерфейса.\n\nЕсли вы отмените платную подписку, мы предоставляем льготный период в 90 дней. По истечении этого времени автоматизированный процесс навсегда удалит ваши старые данные и векторы поиска в соответствии с принципами минимизации данных.",
			},
		],
	},
};

export default function PdfAnkiPrivacyPolicyPage() {
	const { locale } = useI18n();
	const content = policies[locale] ?? policies.en;

	return (
		<LegalPageTemplate
			title={content.title}
			description={content.description}
			lastUpdated={content.lastUpdated}
			sections={content.sections}
		/>
	);
}
```

### `src/app/pdf-anki/terms/page.tsx`
```tsx
"use client";

import { useI18n } from "@/app/i18n-provider";
import { LegalPageTemplate } from "@/components/LegalPageTemplate";
import type { Locale } from "@/lib/i18n-config";

type TermsOfUseContent = {
	title: string;
	description: string;
	lastUpdated: string;
	sections: { title: string; body: string }[];
};

const policies: Record<Locale, TermsOfUseContent> = {
	en: {
		title: "Terms of Use (PDF to Anki)",
		description: "These Terms of Use govern the access and use of our Artificial Intelligence platform (specifically the PDF to Anki module). By using our services, you accept these conditions in their entirety.",
		lastUpdated: "Last updated: April 6, 2026",
		sections: [
			{
				title: "1. Description of Service",
				body: "Our platform offers AI-powered tools for study card generation from PDF documents (PDF to Anki).\n\nThe user acknowledges that AI systems are probabilistic and may generate inaccurate results ('hallucinations'). Although we implement strict logical validations, the user is solely responsible for verifying the accuracy of the generated flashcards before their academic or professional use.",
			},
			{
				title: "2. Accounts, Subscriptions, and PAYG System",
				body: "We offer free and paid plans (Premium, Pro) as well as a Pay-As-You-Go (PAYG) system. Payments are securely processed through Stripe.\n\n- PAYG credits and subscription limits are consumed based on the token usage of the AI models.\n- No refunds are offered for AI credits already consumed.\n- We reserve the right to suspend accounts that attempt to evade usage limits (Rate Limits) or that make abusive use of the API.",
			},
			{
				title: "3. Privacy and Data Security",
				body: "Security is our priority. We implement Application-Level Encryption (ALE) and Data Loss Prevention (DLP) systems to protect your information. By using the service, you agree that we process your data according to our Privacy Policy.\n\nThe user agrees not to use the platform to process illegal material, content that infringes copyright, or highly sensitive unauthorized data, despite our anonymization measures.",
			},
			{
				title: "4. Intellectual Property",
				body: "The user retains all intellectual property rights over the PDF documents they upload to the platform. We retain all rights over the code, algorithms, system prompts, and platform infrastructure.\n\nThe user grants the platform a temporary and strictly limited license to process their files for the sole purpose of providing the requested service.",
			},
			{
				title: "5. Limitation of Liability",
				body: "The service is provided 'as is'. To the maximum extent permitted by law, we are not liable for indirect damages, loss of profits, loss of data, or business interruptions arising from the use or inability to use the platform, including failures in external AI providers (e.g., service outages in Vertex AI or Groq).",
			},
		],
	},
	es: {
		title: "Términos de Uso (PDF a Anki)",
		description: "Estos Términos de Uso regulan el acceso y uso de nuestra plataforma de Inteligencia Artificial (específicamente el módulo PDF to Anki). Al utilizar nuestros servicios, aceptas estas condiciones en su totalidad.",
		lastUpdated: "Última actualización: 6 de abril de 2026",
		sections: [
			{
				title: "1. Descripción del Servicio",
				body: "Nuestra plataforma ofrece herramientas potenciadas por Inteligencia Artificial para la generación de tarjetas de estudio a partir de documentos PDF (PDF to Anki).\n\nEl usuario reconoce que los sistemas de IA son probabilísticos y pueden generar resultados inexactos ('alucinaciones'). Aunque implementamos validaciones lógicas estrictas, el usuario es el único responsable de verificar la exactitud de las flashcards generadas antes de su uso académico o profesional.",
			},
			{
				title: "2. Cuentas, Suscripciones y Sistema PAYG",
				body: "Ofrecemos planes gratuitos y de pago (Premium, Pro) así como un sistema de pago por uso (Pay-As-You-Go o PAYG). Los pagos se procesan de forma segura a través de Stripe.\n\n- Los créditos PAYG y los límites de suscripción se consumen según el uso de tokens de los modelos de IA.\n- No se ofrecen reembolsos por créditos de IA ya consumidos.\n- Nos reservamos el derecho de suspender cuentas que intenten evadir los límites de uso (Rate Limits) o que realicen un uso abusivo de la API.",
			},
			{
				title: "3. Privacidad y Seguridad de los Datos",
				body: "La seguridad es nuestra prioridad. Implementamos cifrado a nivel de aplicación (ALE) y sistemas de prevención de pérdida de datos (DLP) para proteger tu información. Al usar el servicio, aceptas que procesemos tus datos según nuestra Política de Privacidad.\n\nEl usuario se compromete a no utilizar la plataforma para procesar material ilegal, contenido que infrinja derechos de autor, o datos altamente sensibles no autorizados, a pesar de nuestras medidas de anonimización.",
			},
			{
				title: "4. Propiedad Intelectual",
				body: "El usuario retiene todos los derechos de propiedad intelectual sobre los documentos PDF que suba a la plataforma. Nosotros retenemos todos los derechos sobre el código, algoritmos, prompts del sistema y la infraestructura de la plataforma.\n\nEl usuario otorga a la plataforma una licencia temporal y estrictamente limitada para procesar sus archivos con el único fin de prestarle el servicio solicitado.",
			},
			{
				title: "5. Limitación de Responsabilidad",
				body: "El servicio se proporciona 'tal cual'. En la medida máxima permitida por la ley, no nos hacemos responsables de daños indirectos, pérdida de beneficios, pérdida de datos o interrupciones del negocio derivadas del uso o la imposibilidad de uso de la plataforma, incluyendo fallos en los proveedores de IA externos (ej. caídas de servicio en Vertex AI o Groq).",
			},
		],
	},
	de: {
		title: "Nutzungsbedingungen (PDF zu Anki)",
		description: "Diese Nutzungsbedingungen regeln den Zugang und die Nutzung unserer Plattform für Künstliche Intelligenz (insbesondere das Modul PDF to Anki). Durch die Nutzung unserer Dienste akzeptieren Sie diese Bedingungen in vollem Umfang.",
		lastUpdated: "Zuletzt aktualisiert: 6. April 2026",
		sections: [
			{
				title: "1. Beschreibung des Dienstes",
				body: "Unsere Plattform bietet KI-gestützte Tools zur Erstellung von Lernkarten aus PDF-Dokumenten (PDF to Anki).\n\nDer Benutzer erkennt an, dass KI-Systeme probabilistisch sind und ungenaue Ergebnisse ('Halluzinationen') erzeugen können. Obwohl wir strenge logische Validierungen implementieren, ist der Benutzer allein dafür verantwortlich, die Richtigkeit der generierten Lernkarten vor ihrer akademischen oder beruflichen Nutzung zu überprüfen.",
			},
			{
				title: "2. Konten, Abonnements und PAYG-System",
				body: "Wir bieten kostenlose und kostenpflichtige Pläne (Premium, Pro) sowie ein Pay-As-You-Go (PAYG)-System an. Zahlungen werden sicher über Stripe abgewickelt.\n\n- PAYG-Guthaben und Abonnementlimits werden basierend auf der Token-Nutzung der KI-Modelle verbraucht.\n- Für bereits verbrauchte KI-Guthaben werden keine Rückerstattungen gewährt.\n- Wir behalten uns das Recht vor, Konten zu sperren, die versuchen, Nutzungslimits (Rate Limits) zu umgehen oder die API missbräuchlich zu nutzen.",
			},
			{
				title: "3. Datenschutz und Datensicherheit",
				body: "Sicherheit ist unsere Priorität. Wir implementieren Application-Level Encryption (ALE) und Data Loss Prevention (DLP)-Systeme, um Ihre Informationen zu schützen. Durch die Nutzung des Dienstes stimmen Sie zu, dass wir Ihre Daten gemäß unserer Datenschutzrichtlinie verarbeiten.\n\nDer Benutzer verpflichtet sich, die Plattform nicht zur Verarbeitung von illegalem Material, urheberrechtsverletzenden Inhalten oder hochsensiblen, nicht autorisierten Daten zu nutzen, trotz unserer Anonymisierungsmaßnahmen.",
			},
			{
				title: "4. Geistiges Eigentum",
				body: "Der Benutzer behält alle geistigen Eigentumsrechte an den PDF-Dokumenten, die er auf die Plattform hochlädt. Wir behalten alle Rechte an Code, Algorithmen, System-Prompts und der Plattform-Infrastruktur.\n\nDer Benutzer gewährt der Plattform eine temporäre und streng begrenzte Lizenz zur Verarbeitung seiner Dateien ausschließlich zu dem Zweck, den angeforderten Dienst bereitzustellen.",
			},
			{
				title: "5. Haftungsbeschränkung",
				body: "Der Dienst wird 'wie besehen' bereitgestellt. Soweit gesetzlich zulässig, haften wir nicht für indirekte Schäden, entgangenen Gewinn, Datenverlust oder Betriebsunterbrechungen, die sich aus der Nutzung oder der Unmöglichkeit der Nutzung der Plattform ergeben, einschließlich Ausfällen bei externen KI-Anbietern (z. B. Dienstausfälle bei Vertex AI oder Groq).",
			},
		],
	},
	it: {
		title: "Termini di Utilizzo (PDF ad Anki)",
		description: "Questi Termini di Utilizzo regolano l'accesso e l'uso della nostra piattaforma di Intelligenza Artificiale (nello specifico il modulo PDF to Anki). Utilizzando i nostri servizi, accetti queste condizioni per intero.",
		lastUpdated: "Ultimo aggiornamento: 6 aprile 2026",
		sections: [
			{
				title: "1. Descrizione del Servizio",
				body: "La nostra piattaforma offre strumenti basati sull'intelligenza artificiale per la generazione di carte studio da documenti PDF (PDF to Anki).\n\nL'utente riconosce che i sistemi di intelligenza artificiale sono probabilistici e possono generare risultati inaccurati ('allucinazioni'). Sebbene implementiamo rigorose convalide logiche, l'utente è l'unico responsabile della verifica dell'accuratezza delle flashcard generate prima del loro utilizzo accademico o professionale.",
			},
			{
				title: "2. Account, Abbonamenti e Sistema PAYG",
				body: "Offriamo piani gratuiti e a pagamento (Premium, Pro) nonché un sistema Pay-As-You-Go (PAYG). I pagamenti vengono elaborati in modo sicuro tramite Stripe.\n\n- I crediti PAYG e i limiti di abbonamento vengono consumati in base all'utilizzo dei token dei modelli AI.\n- Non sono previsti rimborsi per i crediti AI già consumati.\n- Ci riserviamo il diritto di sospendere gli account che tentano di eludere i limiti di utilizzo (Rate Limits) o che fanno un uso abusivo dell'API.",
			},
			{
				title: "3. Privacy e Sicurezza dei Dati",
				body: "La sicurezza è la nostra priorità. Implementiamo sistemi di crittografia a livello di applicazione (ALE) e prevenzione della perdita di dati (DLP) per proteggere le tue informazioni. Utilizzando il servizio, accetti che trattiamo i tuoi dati secondo la nostra Informativa sulla Privacy.\n\nL'utente si impegna a non utilizzare la piattaforma per elaborare materiale illegale, contenuti che violano il copyright o dati altamente sensibili non autorizzati, nonostante le nostre misure di anonimizzazione.",
			},
			{
				title: "4. Proprietà Intellettuale",
				body: "L'utente mantiene tutti i diritti di proprietà intellettuale sui documenti PDF che carica sulla piattaforma. Noi manteniamo tutti i diritti sul codice, sugli algoritmi, sui prompt di sistema e sull'infrastruttura della piattaforma.\n\nL'utente concede alla piattaforma una licenza temporanea e strettamente limitata per elaborare i propri file al solo scopo di fornire il servizio richiesto.",
			},
			{
				title: "5. Limitazione di Responsabilità",
				body: "Il servizio è fornito 'così com'è'. Nella misura massima consentita dalla legge, non siamo responsabili per danni indiretti, perdita di profitti, perdita di dati o interruzioni dell'attività derivanti dall'uso o dall'impossibilità di utilizzare la piattaforma, inclusi guasti nei fornitori di intelligenza artificiale esterni (es. interruzioni del servizio in Vertex AI o Groq).",
			},
		],
	},
	fr: {
		title: "Conditions d'Utilisation (PDF vers Anki)",
		description: "Ces Conditions d'Utilisation régissent l'accès et l'utilisation de notre plateforme d'Intelligence Artificielle (spécifiquement le module PDF to Anki). En utilisant nos services, vous acceptez ces conditions dans leur intégralité.",
		lastUpdated: "Dernière mise à jour : 6 avril 2026",
		sections: [
			{
				title: "1. Description du Service",
				body: "Notre plateforme propose des outils basés sur l'IA pour la génération de cartes d'étude à partir de documents PDF (PDF to Anki).\n\nL'utilisateur reconnaît que les systèmes d'IA sont probabilistes et peuvent générer des résultats inexacts ('hallucinations'). Bien que nous mettions en œuvre des validations logiques strictes, l'utilisateur est seul responsable de la vérification de l'exactitude des flashcards générées avant leur utilisation académique ou professionnelle.",
			},
			{
				title: "2. Comptes, Abonnements et Système PAYG",
				body: "Nous proposons des forfaits gratuits et payants (Premium, Pro) ainsi qu'un système de paiement à l'utilisation (Pay-As-You-Go ou PAYG). Les paiements sont traités de manière sécurisée via Stripe.\n\n- Les crédits PAYG et les limites d'abonnement sont consommés en fonction de l'utilisation des jetons des modèles d'IA.\n- Aucun remboursement n'est proposé pour les crédits d'IA déjà consommés.\n- Nous nous réservons le droit de suspendre les comptes qui tentent de contourner les limites d'utilisation (Rate Limits) ou qui font un usage abusif de l'API.",
			},
			{
				title: "3. Confidentialité et Sécurité des Données",
				body: "La sécurité est notre priorité. Nous mettons en œuvre des systèmes de chiffrement au niveau de l'application (ALE) et de prévention des pertes de données (DLP) pour protéger vos informations. En utilisant le service, vous acceptez que nous traitions vos données conformément à notre Politique de Confidentialité.\n\nL'utilisateur s'engage à ne pas utiliser la plateforme pour traiter du matériel illégal, du contenu enfreignant les droits d'auteur ou des données hautement sensibles non autorisées, malgré nos mesures d'anonymisation.",
			},
			{
				title: "4. Propriété Intellectuelle",
				body: "L'utilisateur conserve tous les droits de propriété intellectuelle sur les documents PDF qu'il télécharge sur la plateforme. Nous conservons tous les droits sur le code, les algorithmes, les invites système et l'infrastructure de la plateforme.\n\nL'utilisateur accorde à la plateforme une licence temporaire et strictement limitée pour traiter ses fichiers dans le seul but de fournir le service demandé.",
			},
			{
				title: "5. Limitation de Responsabilité",
				body: "Le service est fourni 'tel quel'. Dans toute la mesure permise par la loi, nous ne sommes pas responsables des dommages indirects, de la perte de profits, de la perte de données ou des interruptions d'activité découlant de l'utilisation ou de l'impossibilité d'utiliser la plateforme, y compris les défaillances des fournisseurs d'IA externes (par ex., pannes de service chez Vertex AI ou Groq).",
			},
		],
	},
	ru: {
		title: "Условия использования (PDF в Anki)",
		description: "Эти Условия использования регулируют доступ и использование нашей платформы искусственного интеллекта (в частности, модуля PDF to Anki). Используя наши услуги, вы полностью принимаете эти условия.",
		lastUpdated: "Последнее обновление: 6 апреля 2026 г.",
		sections: [
			{
				title: "1. Описание сервиса",
				body: "Наша платформа предлагает инструменты на базе ИИ для создания учебных карточек из PDF-документов (PDF to Anki).\n\nПользователь признает, что системы ИИ являются вероятностными и могут генерировать неточные результаты ('галлюцинации'). Хотя мы внедряем строгие логические проверки, пользователь несет единоличную ответственность за проверку точности сгенерированных карточек перед их академическим или профессиональным использованием.",
			},
			{
				title: "2. Аккаунты, подписки и система PAYG",
				body: "Мы предлагаем бесплатные и платные планы (Premium, Pro), а также систему оплаты по мере использования (Pay-As-You-Go или PAYG). Платежи безопасно обрабатываются через Stripe.\n\n- Кредиты PAYG и лимиты подписки расходуются на основе использования токенов моделями ИИ.\n- Возврат средств за уже израсходованные кредити ИИ не производится.\n- Мы оставляем за собой право приостанавливать действие учетных записей, которые пытаются обойти ограничения использования (Rate Limits) или злоупотребляют API.",
			},
			{
				title: "3. Конфиденциальность и безопасность данных",
				body: "Безопасность — наш приоритет. Мы внедряем системы шифрования на уровне приложения (ALE) и предотвращения потери данных (DLP) для защиты вашей информации. Используя сервис, вы соглашаетесь с тем, что мы обрабатываем ваши данные в соответствии с нашей Политикой конфиденциальности.\n\nПользователь обязуется не использовать платформу для обработки незаконных материалов, контента, нарушающего авторские права, или строго конфиденциальных несанкционированных данных, несмотря на наши меры по анонимизации.",
			},
			{
				title: "4. Интеллектуальная собственность",
				body: "Пользователь сохраняет все права интеллектуальной собственности на PDF-документы, которые он загружает на платформу. Мы сохраняем все права на код, алгоритмы, системные промпты и инфраструктуру платформы.\n\nПользователь предоставляет платформе временную и строго ограниченную лицензию на обработку своих файлов с единственной целью предоставления запрошенной услуги.",
			},
			{
				title: "5. Ограничение ответственности",
				body: "Сервис предоставляется 'как есть'. В максимальной степени, разрешенной законом, мы не несем ответственности за косвенные убытки, упущенную выгоду, потерю данных или перебои в работе бизнеса, возникающие в результате использования или невозможности использования платформы, включая сбои у внешних поставщиков ИИ (например, перебои в обслуживании Vertex AI или Groq).",
			},
		],
	},
};

export default function PdfAnkiTermsOfUsePage() {
	const { locale } = useI18n();
	const content = policies[locale] ?? policies.en!;

	return (
		<LegalPageTemplate
			title={content.title}
			description={content.description}
			lastUpdated={content.lastUpdated}
			sections={content.sections}
		/>
	);
}
```

### `src/app/receipts/document-hints.ts`
```ts
type DocumentHints = {
	document_mime_type: string;
	document_filename: string;
};

const SUPPORTED_DOCUMENT_MIME_TYPES = new Set<string>([
	"image/png",
	"image/jpeg",
	"image/webp",
	"image/heic",
	"image/heif",
	"application/pdf",
	"image/tiff",
]);

const SUPPORTED_DOCUMENT_EXTENSIONS = [
	".png",
	".jpg",
	".jpeg",
	".webp",
	".heic",
	".heif",
	".pdf",
	".tif",
	".tiff",
] as const;

const EXT_TO_MIME: Record<(typeof SUPPORTED_DOCUMENT_EXTENSIONS)[number], string> =
	{
		".jpg": "image/jpeg",
		".jpeg": "image/jpeg",
		".png": "image/png",
		".webp": "image/webp",
		".heic": "image/heic",
		".heif": "image/heif",
		".pdf": "application/pdf",
		".tif": "image/tiff",
		".tiff": "image/tiff",
	};

function getFileLowerExtension(filename: string): string | null {
	const idx = filename.lastIndexOf(".");
	if (idx < 0) return null;
	return filename.slice(idx).toLowerCase();
}

export function inferDocumentHintsFromFile(file: File): DocumentHints | null {
	const document_filename = file.name;
	if (!document_filename.trim()) return null;

	const ext = getFileLowerExtension(document_filename);
	if (!ext) return null;

	const allowed = SUPPORTED_DOCUMENT_EXTENSIONS.includes(
		ext as (typeof SUPPORTED_DOCUMENT_EXTENSIONS)[number]
	);
	if (!allowed) return null;

	const type = file.type?.trim().toLowerCase() ?? "";
	const usefulType = type.length > 0 && type !== "application/octet-stream";

	if (usefulType && SUPPORTED_DOCUMENT_MIME_TYPES.has(type)) {
		return {
			document_mime_type: type,
			document_filename,
		};
	}

	const mime = EXT_TO_MIME[ext as keyof typeof EXT_TO_MIME] ?? null;
	if (!mime) return null;

	return {
		document_mime_type: mime,
		document_filename,
	};
}
```

### `src/app/receipts/error.tsx`
```tsx
"use client";

type ErrorPageProps = {
	error: Error & { digest?: string };
	reset: () => void;
};

export default function ReceiptsError({ reset }: ErrorPageProps) {
	return (
		<main className="flex min-h-screen flex-col items-center justify-center bg-slate-950 px-6 text-center text-slate-50">
			<h1 className="text-2xl font-semibold">Something went wrong</h1>
			<p className="mt-3 max-w-sm text-sm text-slate-300">
				The receipt parser encountered an unexpected error. You can try again
				and the session will be reset. No images are stored on the server.
			</p>
			<button
				type="button"
				onClick={reset}
				className="mt-4 rounded-full bg-slate-100 px-4 py-2 text-xs font-medium text-slate-900"
			>
				Try again
			</button>
		</main>
	);
}
```

### `src/app/receipts/layout.tsx`
```tsx
import type { Metadata } from "next";
import type { ReactNode } from "react";

import { ReceiptsModuleChrome } from "@/app/receipts/ReceiptsModuleChrome";

export const metadata: Metadata = {
	title: { absolute: "Vera" },
	icons: {
		icon: [{ url: "/icons/receipts-icon-192.png", sizes: "192x192", type: "image/png" }],
	},
};

export default function ReceiptsLayout({ children }: { children: ReactNode }) {
	return <ReceiptsModuleChrome>{children}</ReceiptsModuleChrome>;
}
```

### `src/app/receipts/loading.tsx`
```tsx
export default function ReceiptsLoading() {
	return (
		<main className="flex min-h-screen flex-col bg-slate-950 text-slate-50">
			<header className="flex items-center justify-between px-4 py-3">
				<div className="h-6 w-32 animate-pulse rounded-full bg-slate-800" />
				<div className="h-6 w-20 animate-pulse rounded-full bg-slate-800" />
			</header>
			<section className="flex-1 px-4 pb-28 pt-4">
				<div className="h-64 w-full animate-pulse rounded-2xl border border-slate-800 bg-slate-900/60" />
				<div className="mt-6 space-y-3">
					<div className="h-4 w-40 animate-pulse rounded-full bg-slate-800" />
					<div className="h-4 w-56 animate-pulse rounded-full bg-slate-800" />
				</div>
			</section>
			<nav className="fixed inset-x-0 bottom-0 border-t border-slate-800 bg-slate-950/90 px-4 py-3 backdrop-blur">
				<div className="flex items-center justify-between text-xs text-slate-500">
					<span>Procesando factura…</span>
					<span>Cargando cola offline…</span>
				</div>
			</nav>
		</main>
	);
}
```

### `src/app/receipts/page.tsx`
```tsx
"use client";

import { useCallback, useEffect, useMemo, useRef, useState } from "react";
import { nanoid } from "nanoid";
import Link from "next/link";
import { useForm, type Resolver } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";

import {
	type Receipt,
	receiptSchema,
	RECEIPT_JPEG_BASE64_MAX_CHARS,
} from "@/modules/receipt-parser/schemas";
import {
	receiptFormSchema,
	type ReceiptForm,
} from "@/modules/receipt-parser/receipt-form-schema";
import { receiptQueueDb } from "@/modules/receipt-parser/offline-queue";
import { useAttentionTracker } from "@/modules/observability/useAttentionTracker";
import {
	submitReceiptToBackend,
	RECEIPT_PROCESS_TIMEOUT_MS,
	ReceiptProcessError,
	ReceiptProcessSchemaMismatchError,
	ReceiptProcessValidationError,
	isReceiptProcessTimeoutError,
	ReceiptProcessServerError,
	scheduleReceiptsUsageInvalidation,
} from "./receipt-api";
import {
	useReceiptOfflineSync,
	type ReceiptBackendResult as BackendResult,
	type ReceiptPageStatus as Status,
} from "./use-receipt-offline-sync";
import { fetchReceiptsUsage } from "./receipt-usage-api";
import { CapacityBanner, type ReceiptsUsage } from "./components/CapacityBanner";
import { sha256HexFromBase64 } from "@/lib/image-hash";
import { useI18n } from "@/app/i18n-provider";
import {
	handleAuthApiErrorSideEffects,
	resolveAuthApiUserMessage,
} from "@/app/auth/backend-auth-error";
import { useAuthMe, useSupabaseAuth, useUserId } from "@/app/AuthProvider";
import { isAbortError } from "@/lib/api-client";
import { useReceiptsWallet } from "@/app/receipts/receipts-wallet-context";
import { ApiError } from "@/lib/api-client";
import { ModuleAuthRequiredError } from "@/lib/module-auth-required-error";
import { ReceiptHitlForm } from "@/app/receipts/components/ReceiptHitlForm";
import {
	saveReceiptToHistory,
	SaveReceiptHistoryError,
	type ReceiptMathValidationError,
} from "@/app/receipts/receipt-history-api";
import {
	formatReceiptMathValidationSuggestion,
	getReceiptStaticText,
} from "./receipt-math-validation-i18n";
import {
	clearReceiptParseMemory,
	createReceiptParseMemorySnapshot,
	loadReceiptParseMemory,
	saveReceiptParseMemory,
} from "./receipt-parse-memory";
import {
	serializeTaxRatePercentStringToFractionString,
} from "@/modules/receipt-parser/tax-rate";
import { inferDocumentHintsFromFile } from "./document-hints";
import {
	receiptLimitI18nKey,
	receiptProcessErrorI18nKey,
	RECEIPT_LIMIT_I18N_KEYS,
} from "./receipt-error-utils";
import ReceiptsLoading from "./loading";
import { IndeterminateProgressBar } from "@/components/IndeterminateProgressBar";

export default function ReceiptsPage() {
	useAttentionTracker({ moduleId: "receipts" });
	const userId = useUserId();
	const me = useAuthMe();
	const { signOut, loginWithGoogle, isLoading: authLoading, refreshMe } = useSupabaseAuth();
	const { setPaygBalanceOverride, setFallbackModelId } = useReceiptsWallet();
	const { t, locale } = useI18n();
	const [userRole, setUserRole] = useState<"issuer" | "payer">("payer");
	const [status, setStatus] = useState<Status>("idle");
	const [imagePreview, setImagePreview] = useState<string | null>(null);
	const [backendResult, setBackendResult] = useState<BackendResult | null>(null);
	const [error, setError] = useState<string | null>(null);
	const [receipt, setReceipt] = useState<Receipt | null>(null);
	const [receiptFormInitial, setReceiptFormInitial] = useState<
		ReceiptForm | null
	>(null);
	const [usage, setUsage] = useState<ReceiptsUsage | null>(null);
	const [isSavingReceipt, setIsSavingReceipt] = useState(false);
	const [saveReceiptSuccess, setSaveReceiptSuccess] = useState(false);
	const [saveReceiptError, setSaveReceiptError] = useState(false);
	const [saveReceiptErrorMessage, setSaveReceiptErrorMessage] = useState<
		string | null
	>(null);
	const [validationErrors, setValidationErrors] = useState<
		ReceiptMathValidationError[] | null
	>(null);
	const [lastImageHash, setLastImageHash] = useState<string | null>(null);
	const [jsonActionFeedback, setJsonActionFeedback] = useState<string | null>(
		null
	);
	const [showAuthLoginCta, setShowAuthLoginCta] = useState(false);

	const [cameraOpen, setCameraOpen] = useState(false);
	const [cameraError, setCameraError] = useState<string | null>(null);
	const [cameraBusy, setCameraBusy] = useState(false);
	const cameraStreamRef = useRef<MediaStream | null>(null);
	const cameraVideoRef = useRef<HTMLVideoElement | null>(null);
	const restoredParseMemoryKeyRef = useRef<string | null>(null);

	const receiptForm = useForm<ReceiptForm>({
		resolver: zodResolver(receiptFormSchema as Parameters<typeof zodResolver>[0]) as unknown as Resolver<ReceiptForm>,
		defaultValues: undefined,
	});

	const isOnline = useOnlineStatus();
	const isAuthReady = !authLoading && me != null;
	const restorableParseMemory = useMemo(() => {
		if (!userId || !isAuthReady) {
			return null;
		}
		return loadReceiptParseMemory(userId);
	}, [isAuthReady, userId]);

	useReceiptOfflineSync({
		isOnline,
		userId,
		isAuthReady,
		suspendSync: Boolean(restorableParseMemory),
		userRole,
		signOut,
		translate: t,
		receiptForm,
		setStatus,
		setReceipt,
		setReceiptFormInitial,
		setValidationErrors,
		setLastImageHash,
		setBackendResult,
		setError,
	});

	useEffect(() => {
		if (!cameraOpen) return;

		const videoEl = cameraVideoRef.current;
		if (!videoEl) return;

		const startCamera = async (): Promise<void> => {
			setCameraError(null);
			setCameraBusy(true);
			try {
				if (
					typeof navigator === "undefined" ||
					!navigator.mediaDevices ||
					!navigator.mediaDevices.getUserMedia
				) {
					setCameraError("Camera is not supported in this environment.");
					return;
				}

				const stream = await navigator.mediaDevices.getUserMedia({
					video: { facingMode: "environment" },
				});

				cameraStreamRef.current = stream;
				videoEl.srcObject = stream;

				await videoEl.play();
			} catch {
				setCameraError("Failed to access the camera.");
				setCameraOpen(false);
			} finally {
				setCameraBusy(false);
			}
		};

		void startCamera();

		return () => {
			cameraStreamRef.current?.getTracks().forEach((track) => {
				track.stop();
			});
			cameraStreamRef.current = null;
		};
	}, [cameraOpen]);

	const showJsonFeedback = useCallback((message: string) => {
		setJsonActionFeedback(message);
		window.setTimeout(() => {
			setJsonActionFeedback(null);
		}, 2800);
	}, []);

	const persistReceiptParseMemorySnapshot = useCallback(
		(formValues?: ReceiptForm | null) => {
			if (!userId || !isAuthReady || !receipt || !backendResult) {
				return;
			}
			if (
				status !== "verified" &&
				status !== "requires_human_correction"
			) {
				return;
			}
			if (
				backendResult.status !== "verified" &&
				backendResult.status !== "requires_human_correction"
			) {
				return;
			}

			const receiptFormValues =
				formValues ??
				(status === "requires_human_correction"
					? (receiptForm.getValues() as ReceiptForm)
					: (receipt as ReceiptForm));
			const persistedBackendResult = {
				status: backendResult.status,
				message: backendResult.message,
			} as const;

			saveReceiptParseMemory(
				userId,
				createReceiptParseMemorySnapshot({
					userRole,
					status,
					imagePreview,
					receipt,
					receiptForm: receiptFormValues,
					receiptFormInitial,
					backendResult: persistedBackendResult,
					validationErrors,
					lastImageHash,
				}),
			);
		},
		[
			backendResult,
			imagePreview,
			isAuthReady,
			lastImageHash,
			receipt,
			receiptForm,
			receiptFormInitial,
			status,
			userId,
			userRole,
			validationErrors,
		],
	);

	useEffect(() => {
		if (!userId || !isAuthReady) {
			return;
		}
		if (restoredParseMemoryKeyRef.current === userId) {
			return;
		}
		restoredParseMemoryKeyRef.current = userId;

		const stored = restorableParseMemory;
		if (!stored) {
			return;
		}

		setUserRole(stored.userRole);
		setStatus(stored.status);
		setImagePreview(stored.imagePreview);
		setBackendResult(stored.backendResult);
		setError(null);
		setReceipt(stored.receipt);
		setReceiptFormInitial(stored.receiptFormInitial);
		receiptForm.reset(stored.receiptForm);
		setValidationErrors(stored.validationErrors);
		setLastImageHash(stored.lastImageHash);
		setIsSavingReceipt(false);
		setSaveReceiptSuccess(false);
		setSaveReceiptError(false);
		setSaveReceiptErrorMessage(null);
		setJsonActionFeedback(null);
		setShowAuthLoginCta(false);
		setCameraOpen(false);
		setCameraError(null);
		setCameraBusy(false);
	}, [isAuthReady, receiptForm, restorableParseMemory, userId]);

	useEffect(() => {
		persistReceiptParseMemorySnapshot();
	}, [persistReceiptParseMemorySnapshot]);

	useEffect(() => {
		if (
			status !== "requires_human_correction" ||
			!receipt ||
			!receiptFormInitial
		) {
			return;
		}

		const subscription = receiptForm.watch((values) => {
			persistReceiptParseMemorySnapshot(values as ReceiptForm);
		});

		return () => {
			subscription.unsubscribe();
		};
	}, [
		persistReceiptParseMemorySnapshot,
		receipt,
		receiptForm,
		receiptFormInitial,
		status,
	]);

	const handleResetReceiptFlow = useCallback(() => {
		cameraStreamRef.current?.getTracks().forEach((track) => {
			track.stop();
		});
		cameraStreamRef.current = null;

		setUserRole("payer");
		setStatus("idle");
		setImagePreview(null);
		setBackendResult(null);
		setError(null);
		setReceipt(null);
		setReceiptFormInitial(null);
		setIsSavingReceipt(false);
		setSaveReceiptSuccess(false);
		setSaveReceiptError(false);
		setSaveReceiptErrorMessage(null);
		setValidationErrors(null);
		setLastImageHash(null);
		setJsonActionFeedback(null);
		setShowAuthLoginCta(false);
		setCameraOpen(false);
		setCameraError(null);
		setCameraBusy(false);
		if (userId) {
			clearReceiptParseMemory(userId);
		}
	}, [userId]);

	const handleCopyReceiptJson = useCallback(async () => {
		if (!receipt) return;
		const text = JSON.stringify(receipt, null, 2);
		try {
			await navigator.clipboard.writeText(text);
			showJsonFeedback(t("receipts.main.copyJsonSuccess"));
		} catch {
			showJsonFeedback(t("receipts.main.copyJsonError"));
		}
	}, [receipt, showJsonFeedback, t]);

	const handleDownloadReceiptJson = useCallback(() => {
		if (!receipt) return;
		const text = JSON.stringify(receipt, null, 2);
		const blob = new Blob([text], { type: "application/json;charset=utf-8" });
		const url = URL.createObjectURL(blob);
		const a = document.createElement("a");
		a.href = url;
		a.download = "receipt.json";
		a.click();
		URL.revokeObjectURL(url);
	}, [receipt]);

	const normalizeDecimalString = (
		value: string | null | undefined
	): string | null | undefined => {
		if (value === null) return null;
		if (value === undefined) return undefined;

		const trimmed = value.trim();
		if (!trimmed) return trimmed;

		const hasComma = trimmed.includes(",");
		const hasDot = trimmed.includes(".");
		if (!hasComma) {
			return trimmed;
		}

		const noThousands = hasDot ? trimmed.replace(/\./g, "") : trimmed;
		return noThousands.replace(",", ".");
	};

	const normalizeNullableDecimal = (
		value: string | null | undefined
	): string | null | undefined => {
		const normalized = normalizeDecimalString(value);
		if (normalized === "") return null;
		return normalized;
	};

	const normalizeNullableText = (
		value: string | null | undefined
	): string | null => {
		if (value == null) return null;
		const trimmed = value.trim();
		return trimmed.length > 0 ? trimmed : null;
	};

	const normalizeReceiptDecimals = (input: ReceiptForm): ReceiptForm => {
		return {
			...input,
			issuer_name: input.issuer_name.trim(),
			payer_name: input.payer_name.trim(),
			nif_cif_ssn: normalizeNullableText(input.nif_cif_ssn),
			last_4_card_digits: input.last_4_card_digits.trim(),

			subtotal: normalizeDecimalString(input.subtotal) as string,
			discount: normalizeDecimalString(input.discount) as string,
			tax: normalizeDecimalString(input.tax) as string,
			total: normalizeDecimalString(input.total) as string,
			shipping: normalizeNullableDecimal(input.shipping),
			change_due: normalizeNullableDecimal(input.change_due),

			items: input.items.map((item) => ({
				...item,
				unit_price: normalizeDecimalString(item.unit_price) as string,
				total: normalizeDecimalString(item.total) as string,
				discount_amount: normalizeNullableDecimal(item.discount_amount),
				tax_rate: serializeTaxRatePercentStringToFractionString(item.tax_rate),
			})),

			taxes: input.taxes.map((tax) => ({
				...tax,
				tax_rate:
					serializeTaxRatePercentStringToFractionString(tax.tax_rate) ??
					"0.00",
				base_amount: normalizeDecimalString(tax.base_amount) as string,
				tax_amount: normalizeDecimalString(tax.tax_amount) as string,
			})),
		};
	};

	useEffect(() => {
		let cancelled = false;
		if (!isOnline || !isAuthReady) return;
		fetchReceiptsUsage()
			.then((u) => {
				if (!cancelled) setUsage(u);
			})
			.catch(() => {
				if (!cancelled) setUsage(null);
			});
		return () => {
			cancelled = true;
		};
	}, [isOnline, isAuthReady]);

	const handleSaveReceipt = useCallback(
		async (data: ReceiptForm) => {
			setIsSavingReceipt(true);
			setSaveReceiptSuccess(false);
			setSaveReceiptError(false);
			setShowAuthLoginCta(false);
			try {
				setSaveReceiptErrorMessage(null);

				const normalized = normalizeReceiptDecimals(data);
				if (!lastImageHash) {
					throw new SaveReceiptHistoryError(
						"Missing image_hash for correction save.",
						400,
						"No se pudo calcular el hash de la imagen (image_hash). Intenta de nuevo."
					);
				}

				if (process.env.NODE_ENV === "development") {
					const itemLineErrors =
						validationErrors?.filter((e) => {
							const isItemTotal =
								e.field === "items.total" || e.field.endsWith("items.total");
							return isItemTotal && e.item_line != null;
						}) ?? [];

					console.log("[receipts HITL] Save payload totals", {
						uiTotal: data.total,
						sentTotal: normalized.total,
						itemLineErrors: itemLineErrors
							.slice(0, 10)
							.map((e) => {
								const line = e.item_line ?? null;
								const idx = line != null ? Math.max(0, line - 1) : null;
								const sentItemTotal =
									idx != null && normalized.items[idx]
										? normalized.items[idx].total
										: null;
								return { item_line: line, sentItemTotal };
							}),
					});
				}

				const response = await saveReceiptToHistory(
					{
						receipt: normalized,
						user_role: data.user_role,
						image_hash: lastImageHash,

						original_issuer_name: normalizeNullableText(
							receiptFormInitial?.issuer_name
						),
						original_payer_name: normalizeNullableText(
							receiptFormInitial?.payer_name
						),
						original_nif_cif_ssn: normalizeNullableText(
							receiptFormInitial?.nif_cif_ssn
						),
					},
					{ userId }
				);

				if (response.status === "saved") {
					const normalizedReceipt = receiptSchema.parse(response.receipt);
					setReceipt(normalizedReceipt);
					setReceiptFormInitial(normalizedReceipt);
					receiptForm.reset(normalizedReceipt);
					setValidationErrors(null);
					setSaveReceiptSuccess(true);
					setStatus("verified");
					setBackendResult({
						status: "verified",
						message: t("receipts.main.hitl.saveSuccess"),
					});
					setUsage((prev) => {
						if (!prev) return prev;
						const newCount = prev.receipt_count + 1;
						return {
							...prev,
							receipt_count: newCount,
							at_80_percent: prev.max_receipts > 0 && newCount / prev.max_receipts >= 0.8,
						};
					});
					window.dispatchEvent(new CustomEvent("receipts-usage-invalidated"));
				} else if (response.status === "unprocessable") {
					setSaveReceiptError(true);
					setSaveReceiptSuccess(false);
					setSaveReceiptErrorMessage(
						t("receipts.status.unprocessable")
					);
					setValidationErrors(response.validation_errors);
					setStatus("requires_human_correction");
					setBackendResult({
						status: "requires_human_correction",
						message: t("receipts.status.requiresHumanCorrection"),
					});

				}
			} catch (saveErr) {
				console.error(saveErr);
				handleAuthApiErrorSideEffects(saveErr, signOut);
				const authSaveMsg = resolveAuthApiUserMessage(saveErr, t);
				if (authSaveMsg) {
					setSaveReceiptError(true);
					setSaveReceiptSuccess(false);
					setSaveReceiptErrorMessage(authSaveMsg);
					setShowAuthLoginCta(true);
				} else if (saveErr instanceof SaveReceiptHistoryError) {
					setSaveReceiptError(true);
					if (saveErr.status === 429) {
						setSaveReceiptErrorMessage(t(receiptLimitI18nKey(saveErr.detail)));
					} else {
					const fallbackMessage = (() => {
						switch (saveErr.status) {
							case 422: {
								switch (locale) {
									case "es":
										return "La corrección no pasa la validación matemática.";
									case "de":
										return "Die Korrektur besteht die mathematische Validierung nicht.";
									case "it":
										return "La correzione non supera la validazione matematica.";
									case "fr":
										return "La correction n'a pas passé la validation mathématique.";
									case "ru":
										return "Коррекция не прошла математическую валидацию.";
									case "en":
									default:
										return "Correction failed mathematical validation.";
								}
							}
							case 409: {
								switch (locale) {
									case "es":
										return "Conflicto de versión. Recarga el historial e intenta de nuevo.";
									case "de":
										return "Versionskonflikt. Verlauf neu laden und erneut versuchen.";
									case "it":
										return "Conflitto di versione. Ricarica la cronologia e riprova.";
									case "fr":
										return "Conflit de version. Rechargez l'historique et réessayez.";
									case "ru":
										return "Конфликт версии. Перезагрузите историю и попробуйте снова.";
									case "en":
									default:
										return "Version conflict. Reload history and try again.";
								}
							}
							case 400: {
								switch (locale) {
									case "es":
										return "Solicitud inválida. Revisa el JSON corregido.";
									case "de":
										return "Ungültige Anfrage. Überprüfe das korrigierte JSON.";
									case "it":
										return "Richiesta non valida. Controlla il JSON corretto.";
									case "fr":
										return "Demande invalide. Vérifie le JSON corrigé.";
									case "ru":
										return "Неверный запрос. Проверьте исправленный JSON.";
									case "en":
									default:
										return "Invalid request. Review the corrected JSON.";
								}
							}
							case 401: {
								switch (locale) {
									case "es":
										return "Falta identidad/authorization. Re-inicia sesión.";
									case "de":
										return "Fehlende Identität/Berechtigung. Bitte erneut anmelden.";
									case "it":
										return "Manca identità/autorizzazione. Accedi di nuovo.";
									case "fr":
										return "Identité/autorisation manquante. Veuillez vous reconnecter.";
									case "ru":
										return "Отсутствует идентификация/авторизация. Войдите снова.";
									case "en":
									default:
										return "Missing identity/authorization. Please sign in again.";
								}
							}
							case 403: {
								switch (locale) {
									case "es":
										return "El token no coincide con X-User-Id. Cierra sesión y vuelve a entrar.";
									case "de":
										return "Token stimmt nicht mit X-User-Id überein. Abmelden und erneut anmelden.";
									case "it":
										return "Il token non corrisponde a X-User-Id. Esci e accedi di nuovo.";
									case "fr":
										return "Le jeton ne correspond pas à X-User-Id. Déconnectez-vous et reconnectez-vous.";
									case "ru":
										return "Токен не совпадает с X-User-Id. Выйдите и войдите снова.";
									case "en":
									default:
										return "Token does not match X-User-Id. Sign out and sign in again.";
								}
							}
							default: {
								switch (locale) {
									case "es":
										return "No se pudo guardar la corrección.";
									case "de":
										return "Korrektur konnte nicht gespeichert werden.";
									case "it":
										return "Impossibile salvare la correzione.";
									case "fr":
										return "Impossible d'enregistrer la correction.";
									case "ru":
										return "Не удалось сохранить коррекцию.";
									case "en":
									default:
										return "Could not save the correction.";
								}
							}
						}
					})();
					setSaveReceiptErrorMessage(fallbackMessage);
					}
				} else {
					switch (locale) {
						case "es":
							setSaveReceiptErrorMessage("No se pudo guardar la corrección.");
							break;
						case "de":
							setSaveReceiptErrorMessage("Korrektur konnte nicht gespeichert werden.");
							break;
						case "it":
							setSaveReceiptErrorMessage("Impossibile salvare la correzione.");
							break;
						case "fr":
							setSaveReceiptErrorMessage("Impossible d'enregistrer la correction.");
							break;
						case "ru":
							setSaveReceiptErrorMessage("Не удалось сохранить коррекцию.");
							break;
						case "en":
						default:
							setSaveReceiptErrorMessage("Could not save the correction.");
							break;
					}
				}
			} finally {
				setIsSavingReceipt(false);
			}
		},
		[
			lastImageHash,
			locale,
			normalizeNullableText,
			normalizeReceiptDecimals,
			receiptForm,
			receiptFormInitial,
			signOut,
			t,
			userId,
			userRole,
			validationErrors,
		]
	);

	const statusLabel = useMemo(() => {
		switch (status) {
			case "idle":
				return t("receipts.status.idle");
			case "capturing":
				return t("receipts.status.capturing");
			case "uploading_image":
				return t("receipts.status.uploading_image");
			case "waiting_backend":
				return t("receipts.status.waiting_backend");
			case "verified":
				return t("receipts.status.verified");
			case "unprocessable":
				return t("receipts.status.unprocessable");
			case "requires_human_correction":
				return t("receipts.status.requiresHumanCorrection");
			case "error":
				return t("receipts.status.error");
			default:
				return "";
		}
	}, [status, t]);

	const showProcessingProgress =
		status === "capturing" ||
		status === "uploading_image" ||
		status === "waiting_backend";

	const humanCorrectionHint = useMemo(() => {
		if (status !== "requires_human_correction") return null;

		const errors = validationErrors ?? [];
		if (!errors.length) return null;

		const primary = errors[0] ?? null;
		if (!primary) return null;
		return { correctionLine: formatReceiptMathValidationSuggestion(locale, primary) };
	}, [status, validationErrors, locale]);

	const processReceiptImageBase64 = useCallback(
		async (
			base64: string,
			preview: string | null,
			documentMimeType: string,
			documentFilename: string,
		): Promise<void> => {
			setImagePreview(preview);
			setValidationErrors(null);
			setError(null);
			setBackendResult(null);
			setShowAuthLoginCta(false);

			if (!isOnline) {
				setStatus("idle");
				await enqueueOffline(base64, documentMimeType, documentFilename);
				setBackendResult({
					status: "verified",
					message: t("receipts.backendResult.offlineSynced"),
				});
				return;
			}

			if (!isAuthReady) {
				setStatus("idle");
				setError(t("receipts.main.error.signInRequired"));
				setShowAuthLoginCta(true);
				return;
			}

			setStatus("uploading_image");
			const imageHash = await sha256HexFromBase64(base64).catch(
				() => undefined,
			);
			if (imageHash) {
				setLastImageHash(imageHash);
			}

			setStatus("waiting_backend");

			const parsed = await submitReceiptToBackend(base64, {
				user_role: userRole,
				image_hash: imageHash,
				document_mime_type: documentMimeType,
				document_filename: documentFilename,
				timeoutMs: RECEIPT_PROCESS_TIMEOUT_MS,
			});

			if (parsed.payg_wallet_balance_usd != null) {
				setPaygBalanceOverride(parsed.payg_wallet_balance_usd);
			}
			if (parsed.payg_fallback_model_id != null) {
				setFallbackModelId(parsed.payg_fallback_model_id);
			}
			if (
				parsed.payg_wallet_balance_usd != null ||
				parsed.payg_fallback_model_id != null
			) {
				void refreshMe();
			}

			if (parsed.receipt) {
				setReceipt(parsed.receipt);
				setReceiptFormInitial(parsed.receipt);
				receiptForm.reset(parsed.receipt);
			}

			if (parsed.status === "verified") {
				setStatus("verified");
				setValidationErrors(null);
				setError(null);
				setBackendResult({
					status: "verified",
					message: t("receipts.backendResult.verified"),
				});
				setUsage((prev) => {
					if (!prev) return prev;
					const newCount = prev.receipt_count + 1;
					return {
						...prev,
						receipt_count: newCount,
						at_80_percent: prev.max_receipts > 0 && newCount / prev.max_receipts >= 0.8,
					};
				});
				window.dispatchEvent(new CustomEvent("receipts-usage-invalidated"));
			} else if (parsed.status === "requires_human_correction") {
				setStatus("requires_human_correction");
				setValidationErrors(parsed.validation_errors ?? null);
				setBackendResult({
					status: "requires_human_correction",
					message:
						parsed.error_message?.trim() ||
						t("receipts.status.requiresHumanCorrection"),
				});
				setError(null);
				const bestEffort = parsed.best_effort_receipt ?? parsed.receipt;
				if (bestEffort) {
					const normalizedBestEffort = bestEffort as Receipt;
					setReceipt(normalizedBestEffort);
					setReceiptFormInitial(normalizedBestEffort);
					receiptForm.reset(normalizedBestEffort);
				} else {
					setReceipt(null);
					setReceiptFormInitial(null);
				}
			} else {
				setStatus("unprocessable");
				setValidationErrors(null);
				setError(null);
				setBackendResult({
					status: "unprocessable",
					message:
						parsed.error_message?.trim() ||
						t("receipts.backendResult.unprocessable"),
				});
			}
		},
		[
			enqueueOffline,
			isAuthReady,
			isOnline,
			receiptForm,
			refreshMe,
			setError,
			setFallbackModelId,
			setImagePreview,
			setLastImageHash,
			setPaygBalanceOverride,
			setReceipt,
			setReceiptFormInitial,
			setShowAuthLoginCta,
			setStatus,
			setBackendResult,
			setValidationErrors,
			sha256HexFromBase64,
			submitReceiptToBackend,
			t,
			userRole,
		],
	);

	const handleCaptureFromCamera = useCallback(async (): Promise<void> => {
		const videoEl = cameraVideoRef.current;
		if (!videoEl) return;

		const videoWidth = videoEl.videoWidth ?? 0;
		const videoHeight = videoEl.videoHeight ?? 0;
		if (!videoWidth || !videoHeight) {
			setCameraError("Camera is not ready yet.");
			return;
		}

		const MAX_EDGE = 1024;
		const JPEG_QUALITY = 0.8;

		setCameraBusy(true);
		setCameraError(null);
		setStatus("capturing");
		setError(null);
		setBackendResult(null);
		setShowAuthLoginCta(false);

		try {
			const longestEdge = Math.max(videoWidth, videoHeight);
			const scale = longestEdge > MAX_EDGE ? MAX_EDGE / longestEdge : 1;
			const targetWidth = Math.max(1, Math.round(videoWidth * scale));
			const targetHeight = Math.max(1, Math.round(videoHeight * scale));

			const canvas = document.createElement("canvas");
			canvas.width = targetWidth;
			canvas.height = targetHeight;
			const ctx = canvas.getContext("2d");
			if (!ctx) {
				throw new Error("Failed to obtain 2D canvas context.");
			}

			ctx.drawImage(videoEl, 0, 0, targetWidth, targetHeight);

			const dataUrl = canvas.toDataURL("image/jpeg", JPEG_QUALITY);
			const [, base64] = dataUrl.split(",", 2);
			if (!base64) throw new Error("Invalid camera capture data URL.");

			setCameraOpen(false);
			await processReceiptImageBase64(
				base64,
				dataUrl,
				"image/jpeg",
				"camera_capture.jpg",
			);
		} catch (processingError) {
			if (isReceiptProcessTimeoutError(processingError)) {
				setStatus("error");
				setError(t("receipts.error.timeout"));
				scheduleReceiptsUsageInvalidation();
				return;
			}
			if (processingError instanceof ReceiptProcessServerError) {
				setStatus("error");
				setError(t("receipts.error.serverError"));
				return;
			}
			if (processingError instanceof ReceiptProcessSchemaMismatchError) {
				setStatus("error");
				setError(t("receipts.error.schemaMismatch"));
				return;
			}
			if (processingError instanceof ReceiptProcessValidationError) {
				setStatus("error");
				const detail = processingError.detail ?? "";
				const isTooLarge =
					detail.includes("image_base64") && detail.includes("at most");
				setError(
					isTooLarge
						? t("receipts.error.imageTooLarge")
						: t("receipts.error.imageProcessing"),
				);
				return;
			}
			if (processingError instanceof ReceiptProcessError) {
				setStatus("error");
				if (processingError.status === 409) {
					setError(t("receipts.error.duplicate"));
				} else {
					setError(t(receiptProcessErrorI18nKey(processingError.detail)));
				}
				return;
			}

			handleAuthApiErrorSideEffects(processingError, signOut);
			const authProcMsg = resolveAuthApiUserMessage(processingError, t);
			if (authProcMsg) {
				setStatus("error");
				setError(authProcMsg);
				setShowAuthLoginCta(true);
				return;
			}
			if (processingError instanceof ModuleAuthRequiredError) {
				setStatus("error");
				setError(t("receipts.main.error.signInRequired"));
				setShowAuthLoginCta(true);
				return;
			}
			if (processingError instanceof ApiError) {
				setStatus("error");
				setError(t("receipts.error.imageProcessing"));
				return;
			}

			if (isAbortError(processingError)) {
				setStatus("error");
				setError(t("receipts.error.timeout"));
				scheduleReceiptsUsageInvalidation();
				return;
			}

			if (
				processingError instanceof TypeError &&
				(processingError.message.includes("Failed to fetch") ||
					processingError.message.includes("NetworkError"))
			) {
				setStatus("error");
				setError(t("receipts.error.timeout"));
				scheduleReceiptsUsageInvalidation();
				return;
			}

			console.error(processingError);
			setStatus("error");
			setError(t("receipts.error.imageProcessing"));
		} finally {
			setCameraBusy(false);
		}
	}, [
		ApiError,
		ModuleAuthRequiredError,
		cameraVideoRef,
		processReceiptImageBase64,
		setBackendResult,
		setCameraBusy,
		setCameraError,
		setError,
		setShowAuthLoginCta,
		setStatus,
		signOut,
		t,
	]);

	const handleFileChange = useCallback(
		async (event: React.ChangeEvent<HTMLInputElement>) => {
			const file = event.target.files?.[0];
			if (!file) return;

			const hints = inferDocumentHintsFromFile(file);
			if (!hints) {
				setStatus("error");
				setError(t("receipts.error.unsupportedFileType"));
				setShowAuthLoginCta(false);
				setImagePreview(null);
				setBackendResult(null);
				return;
			}

			setStatus("capturing");
			setError(null);
			setBackendResult(null);
			setShowAuthLoginCta(false);

			try {
				const {
					base64,
					preview,
					effectiveDocumentMimeType,
					effectiveDocumentFilename,
				} = await fileToBase64Original(
					file,
					hints.document_mime_type,
					hints.document_filename,
				);
				await processReceiptImageBase64(
					base64,
					preview,
					effectiveDocumentMimeType,
					effectiveDocumentFilename,
				);
			} catch (processingError) {
				if (isReceiptProcessTimeoutError(processingError)) {
					setStatus("error");
					setError(t("receipts.error.timeout"));
					scheduleReceiptsUsageInvalidation();
					return;
				}
				if (processingError instanceof ReceiptProcessServerError) {
					setStatus("error");
					setError(t("receipts.error.serverError"));
					return;
				}
				if (processingError instanceof ReceiptProcessSchemaMismatchError) {
					setStatus("error");
					setError(t("receipts.error.schemaMismatch"));
					return;
				}
				if (processingError instanceof ReceiptProcessValidationError) {
					setStatus("error");
					const detail = processingError.detail ?? "";
					const isTooLarge =
						detail.includes("image_base64") && detail.includes("at most");
					setError(
						isTooLarge
							? t("receipts.error.imageTooLarge")
							: t("receipts.error.imageProcessing"),
					);
					return;
				}
				if (processingError instanceof ReceiptProcessError) {
					setStatus("error");
					if (processingError.status === 409) {
						setError(t("receipts.error.duplicate"));
					} else {
						setError(t(receiptProcessErrorI18nKey(processingError.detail)));
					}
					return;
				}

				handleAuthApiErrorSideEffects(processingError, signOut);
				const authProcMsg = resolveAuthApiUserMessage(processingError, t);
				if (authProcMsg) {
					setStatus("error");
					setError(authProcMsg);
					setShowAuthLoginCta(true);
					return;
				}
				if (processingError instanceof ModuleAuthRequiredError) {
					setStatus("error");
					setError(t("receipts.main.error.signInRequired"));
					setShowAuthLoginCta(true);
					return;
				}
				if (processingError instanceof ApiError) {
					setStatus("error");
					setError(t("receipts.error.imageProcessing"));
					return;
				}

				if (isAbortError(processingError)) {
					setStatus("error");
					setError(t("receipts.error.timeout"));
					scheduleReceiptsUsageInvalidation();
					return;
				}

				if (
					processingError instanceof TypeError &&
					(processingError.message.includes("Failed to fetch") ||
						processingError.message.includes("NetworkError"))
				) {
					setStatus("error");
					setError(t("receipts.error.timeout"));
					scheduleReceiptsUsageInvalidation();
					return;
				}

				console.error(processingError);
				setStatus("error");
				setError(t("receipts.error.imageProcessing"));
			}
		},
		[processReceiptImageBase64, signOut, t]
	);

	if (authLoading) {
		return <ReceiptsLoading />;
	}

	return (
		<main className="flex min-h-screen flex-col bg-slate-100 text-slate-900">
			<section className="mx-auto flex w-full max-w-4xl flex-1 flex-col gap-6 px-6 py-6">
				<CapacityBanner usage={usage} pricingHref="/receipts/pricing" />
				<div className="flex flex-1 flex-col gap-6 md:flex-row">
				{isAuthReady ? (
					<div className="flex-1 space-y-4">
						<div className="rounded-2xl border border-dashed border-slate-300 bg-slate-50 p-4 shadow-sm">
							<p className="mb-2 text-sm font-medium text-slate-800">
								{t("receipts.main.userRoleLabel")}
							</p>
							<div className="mb-3 flex gap-2">
								<button
									type="button"
									className={`rounded-full px-3 py-1 text-xs font-medium ${
										userRole === "payer"
											? "bg-slate-900 text-white"
											: "bg-slate-200 text-slate-800"
									}`}
									disabled={status === "requires_human_correction"}
									onClick={() => setUserRole("payer")}
								>
									{t("receipts.main.userRole.payer")}
								</button>
								<button
									type="button"
									className={`rounded-full px-3 py-1 text-xs font-medium ${
										userRole === "issuer"
											? "bg-slate-900 text-white"
											: "bg-slate-200 text-slate-800"
									}`}
									disabled={status === "requires_human_correction"}
									onClick={() => setUserRole("issuer")}
								>
									{t("receipts.main.userRole.issuer")}
								</button>
							</div>
							<p className="mb-2 text-sm font-medium text-slate-800">
								{t("receipts.main.selectImageLabel")}
							</p>

							<input
								id="receipt-upload-input"
								type="file"
								accept=".png,.jpg,.jpeg,.webp,.heic,.heif,.pdf,.tif,.tiff"
								onChange={handleFileChange}
								className="hidden"
							/>

							<div className="mt-2 flex flex-wrap items-center gap-2">
								<label
									htmlFor="receipt-upload-input"
									className={`inline-flex items-center rounded-lg border border-slate-300 bg-white px-3 py-1.5 text-xs font-medium text-slate-800 shadow-sm transition hover:-translate-y-0.5 hover:bg-slate-50 ${
										status === "capturing" ||
										status === "uploading_image" ||
										status === "waiting_backend"
											? "pointer-events-none cursor-not-allowed opacity-60"
											: "cursor-pointer"
									}`}
									aria-label={t("receipts.main.uploadButton")}
								>
									{t("receipts.main.uploadButton")}
								</label>
								<button
									type="button"
									className="inline-flex items-center rounded-lg border border-slate-300 bg-white px-3 py-1.5 text-xs font-medium text-slate-800 shadow-sm transition hover:-translate-y-0.5 hover:bg-slate-50 disabled:cursor-not-allowed disabled:opacity-50"
									disabled={
										status === "capturing" ||
										status === "uploading_image" ||
										status === "waiting_backend"
									}
									onClick={() => setCameraOpen(true)}
								>
									{t("receipts.main.cameraButton")}
								</button>
							</div>

							{cameraOpen ? (
								<div className="mt-3 rounded-xl border border-slate-200 bg-slate-950 p-3 text-slate-50 shadow-sm">
									{cameraError ? (
										<p className="text-xs text-rose-600">{cameraError}</p>
									) : null}
									<div className="mt-2">
										<video
											ref={cameraVideoRef}
											className="h-48 w-full rounded-md bg-black object-cover"
											playsInline
											muted
										/>
									</div>
									<div className="mt-3 flex items-center justify-end gap-2">
										<button
											type="button"
											className="rounded-lg border border-slate-200 bg-white px-3 py-1.5 text-xs font-medium text-slate-700 shadow-sm transition hover:-translate-y-0.5 hover:bg-slate-50 disabled:cursor-not-allowed disabled:opacity-50"
											disabled={cameraBusy}
											onClick={() => setCameraOpen(false)}
										>
											{t("receipts.main.cameraCancel")}
										</button>
										<button
											type="button"
											className="rounded-lg bg-emerald-600 px-3 py-1.5 text-xs font-medium text-white shadow-sm transition hover:bg-emerald-500 disabled:cursor-not-allowed disabled:opacity-50"
											disabled={cameraBusy}
											onClick={() => {
												void handleCaptureFromCamera();
											}}
										>
											{t("receipts.main.cameraCapture")}
										</button>
									</div>
								</div>
							) : null}

							{receipt ? (
								<button
									type="button"
									className="mt-3 rounded-lg border border-slate-300 bg-white px-3 py-1.5 text-xs font-medium text-slate-800 shadow-sm transition hover:-translate-y-0.5 hover:bg-slate-50 disabled:cursor-not-allowed disabled:opacity-50"
									disabled={
										status === "capturing" ||
										status === "uploading_image" ||
										status === "waiting_backend" ||
										isSavingReceipt
									}
									onClick={handleResetReceiptFlow}
								>
									{t("receipts.main.newReceipt")}
								</button>
							) : null}
							{showProcessingProgress ? (
								<IndeterminateProgressBar
									ariaLabel={statusLabel}
									className="mt-3"
									trackClassName="bg-slate-200"
									fillClassName="bg-gradient-to-r from-emerald-500 via-emerald-600 to-emerald-500"
								/>
							) : null}
							<p className="mt-3 text-xs text-slate-500">
								{t("receipts.main.statusLabel")}{" "}
								<span className="font-semibold text-slate-800">
									{statusLabel}
								</span>
							</p>
							<p className="mt-1 text-xs text-slate-500">
								{t("receipts.main.connectionLabel")}{" "}
								<span
									className={
										isOnline ? "text-emerald-600 font-semibold" : "text-amber-600"
									}
								>
									{isOnline
										? t("receipts.main.connection.online")
										: t("receipts.main.connection.offline")}
								</span>
							</p>
							{error && (
								<div className="mt-2 space-y-1">
									<p className="text-xs text-rose-600">{error}</p>
									{showAuthLoginCta ? (
										<button
											type="button"
											className="rounded-lg bg-emerald-600 px-3 py-1.5 text-xs font-medium text-white shadow-sm transition hover:bg-emerald-500"
											onClick={() => {
												void loginWithGoogle();
											}}
										>
											{t("auth.signInGoogle")}
										</button>
									) : null}
									{RECEIPT_LIMIT_I18N_KEYS.some((key) => error === t(key)) && (
										<Link
											href="/receipts/pricing"
											className="text-xs font-medium text-slate-700 underline"
										>
											{t("receipts.error.upgradeLink")}
										</Link>
									)}
								</div>
							)}
						</div>

						{backendResult && (
							<div className="rounded-xl border border-slate-200 bg-white p-4 text-xs shadow-sm">
								{backendResult.status ===
								"requires_human_correction" ? (
									<>
										<p className="font-semibold text-slate-800">
											{t("receipts.backendResult.title")}{" "}
											{getReceiptStaticText(
												locale,
												"requiresReviewSuffix",
											)}
										</p>
										<p className="mt-1 text-slate-600">
											{humanCorrectionHint ? (
												<>
													<span className="text-rose-600 font-semibold">
														{getReceiptStaticText(
															locale,
															"correctionRequiredPrefix",
														)}
													</span>{" "}
													{humanCorrectionHint.correctionLine}
												</>
											) : (
												<>
													<span className="text-rose-600 font-semibold">
														{getReceiptStaticText(
															locale,
															"correctionRequiredPrefix",
														)}
													</span>{" "}
													{backendResult.message}
												</>
											)}
										</p>
									</>
								) : (
									<>
										<p className="font-semibold text-slate-800">
											{t("receipts.backendResult.title")}
										</p>
										<p className="mt-1 text-slate-600">
											{backendResult.message}
										</p>
									</>
								)}
							</div>
						)}
					</div>
				) : authLoading ? (
					<div className="flex-1 space-y-4">
						<div className="rounded-2xl border border-dashed border-slate-300 bg-slate-50 p-4 shadow-sm">
							<p className="text-sm font-medium text-slate-800">
								{t("auth.loading")}
							</p>
							<p className="mt-2 max-w-md text-xs text-slate-500">
								{t("receipts.main.sessionHint")}
							</p>
						</div>
					</div>
				) : (
					<div className="flex-1 space-y-4">
						<div className="rounded-2xl border border-dashed border-slate-300 bg-slate-50 p-4 shadow-sm">
							<div className="flex flex-col items-center gap-3 py-2 text-center">
								<p className="max-w-md text-sm font-medium text-slate-800">
									{t("receipts.main.error.signInRequired")}
								</p>
								<button
									type="button"
									className="rounded-lg bg-emerald-600 px-4 py-2 text-xs font-medium text-white shadow-sm transition hover:bg-emerald-500"
									onClick={() => {
										void loginWithGoogle();
									}}
								>
									{t("auth.signInGoogle")}
								</button>
							</div>
						</div>
					</div>
				)}

				<div className="flex-1 rounded-2xl border border-slate-200 bg-white p-5 shadow-sm">
					<h2 className="text-sm font-semibold text-slate-800">
						{t("receipts.main.receiptImageTitle")}
					</h2>
					{!imagePreview && (
						<p className="mt-2 text-xs text-slate-500">
							{t("receipts.main.noPreview")}
						</p>
					)}
					{imagePreview && (
						<div className="mt-2 flex max-h-[420px] items-center justify-center overflow-hidden rounded-xl bg-slate-950 p-3">
							{/* eslint-disable-next-line @next/next/no-img-element */}
							<img
								src={imagePreview}
								alt="Receipt preview"
								className="max-h-[380px] max-w-full object-contain"
							/>
						</div>
					)}

					{receipt && (
						<div className="mt-4">
							<div className="flex flex-wrap items-center justify-between gap-2">
								<h3 className="text-xs font-semibold uppercase text-slate-600">
									{t("receipts.main.parsedReceiptTitle")}
								</h3>
								<div className="flex flex-wrap gap-2">
									<button
										type="button"
										className="rounded-md border border-slate-200 bg-slate-50 px-2 py-1 text-[11px] font-medium text-slate-800 shadow-sm transition hover:-translate-y-0.5 hover:bg-slate-100"
										onClick={() => {
											void handleCopyReceiptJson();
										}}
									>
										{t("receipts.main.copyJson")}
									</button>
									<button
										type="button"
										className="rounded-md border border-slate-200 bg-slate-50 px-2 py-1 text-[11px] font-medium text-slate-800 shadow-sm transition hover:-translate-y-0.5 hover:bg-slate-100"
										onClick={handleDownloadReceiptJson}
									>
										{t("receipts.main.downloadJson")}
									</button>
								</div>
							</div>
							{jsonActionFeedback ? (
								<p className="mt-1 text-[11px] text-slate-600">
									{jsonActionFeedback}
								</p>
							) : null}
							<div className="mt-2 overflow-auto rounded-lg border border-slate-100">
								<table className="min-w-full divide-y divide-slate-100 text-xs">
									<thead className="bg-slate-50">
										<tr>
											<th className="px-3 py-2 text-left font-medium text-slate-700">
												{t("receipts.main.table.header.item")}
											</th>
											<th className="px-3 py-2 text-right font-medium text-slate-700">
												{t("receipts.main.table.header.qty")}
											</th>
											<th className="px-3 py-2 text-right font-medium text-slate-700">
												{t("receipts.main.table.header.unit")}
											</th>
											<th className="px-3 py-2 text-right font-medium text-slate-700">
												{t("receipts.main.table.header.total")}
											</th>
										</tr>
									</thead>
									<tbody className="divide-y divide-slate-100 bg-white">
										{receipt.items.map((item) => (
											<tr key={`${item.description}-${item.total}`}>
												<td className="px-3 py-2 text-slate-800">
													{item.description}
												</td>
												<td className="px-3 py-2 text-right text-slate-700">
													{item.quantity}
												</td>
												<td className="px-3 py-2 text-right text-slate-700">
													{item.unit_price}
												</td>
												<td className="px-3 py-2 text-right text-slate-800">
													{item.total}
												</td>
											</tr>
										))}
									</tbody>
								</table>
							</div>
							<div className="mt-3 flex flex-wrap items-baseline gap-x-4 gap-y-2 text-xs text-slate-700">
								<span>
									{t("receipts.main.summary.subtotal")}{" "}
									<span className="font-semibold">{receipt.subtotal}</span>
								</span>
								<span>
									{t("receipts.main.summary.tax")}{" "}
									<span className="font-semibold">{receipt.tax}</span>
								</span>
								<span>
									{t("receipts.main.summary.total")}{" "}
									<span className="font-semibold">{receipt.total}</span>
								</span>
								<span className="ml-auto text-[11px] uppercase tracking-wide text-slate-500">
									{t("receipts.main.summary.currency")} {receipt.currency}
								</span>
							</div>
						</div>
					)}
					{status === "requires_human_correction" && receiptFormInitial && (
						<ReceiptHitlForm
							form={receiptForm}
							t={t}
							locale={locale}
							showCorrectionHint={status === "requires_human_correction"}
							correctionFallbackMessage={backendResult?.message ?? null}
							onSave={handleSaveReceipt}
							isSaving={isSavingReceipt}
							saveSuccess={saveReceiptSuccess}
							saveError={saveReceiptError}
							saveErrorMessage={saveReceiptErrorMessage}
							saveErrorShowLoginCta={showAuthLoginCta}
							onSaveErrorLogin={() => {
								void loginWithGoogle();
							}}
							validationErrors={validationErrors}
						/>
					)}
				</div>
				</div>
			</section>
		</main>
	);
}

function useOnlineStatus(): boolean {
	const [online, setOnline] = useState<boolean>(
		typeof navigator === "undefined" ? true : navigator.onLine
	);

	useEffect(() => {
		const handleOnline = () => setOnline(true);
		const handleOffline = () => setOnline(false);

		window.addEventListener("online", handleOnline);
		window.addEventListener("offline", handleOffline);

		return () => {
			window.removeEventListener("online", handleOnline);
			window.removeEventListener("offline", handleOffline);
		};
	}, []);

	return online;
}

async function enqueueOffline(
	text: string,
	documentMimeType: string,
	documentFilename: string,
): Promise<void> {
	await receiptQueueDb.receipts.add({
		id: nanoid(),
		imageBase64: text,
		createdAt: new Date(),
		synced: false,
		documentMimeType,
		documentFilename,
	});
}

type FileReadResult = {
	base64: string;
	preview: string | null;
	effectiveDocumentMimeType: string;
	effectiveDocumentFilename: string;
};

async function fileToBase64Original(
	file: File,
	documentMimeType: string,
	documentFilename: string,
): Promise<FileReadResult> {
	const dataUrl = await new Promise<string>((resolve, reject) => {
		const reader = new FileReader();

		reader.onload = () => {
			const result = reader.result;
			if (typeof result !== "string") {
				reject(new Error("Unexpected FileReader result type."));
				return;
			}
			resolve(result);
		};

		reader.onerror = () => {
			reject(reader.error ?? new Error("Failed to read file as data URL."));
		};

		reader.readAsDataURL(file);
	});

	const parts = dataUrl.split(",", 2);
	const base64 = parts.length === 2 ? parts[1] : "";
	if (!base64) {
		throw new Error("Invalid data URL (missing base64 payload).");
	}

	const preview =
		documentMimeType === "application/pdf" || documentMimeType === "image/tiff"
			? null
			: dataUrl;

	let effectiveDocumentMimeType = documentMimeType;
	let effectiveDocumentFilename = documentFilename;
	let effectiveBase64 = base64;
	let effectivePreview = preview;

	const canResizeToJpeg = [
		"image/png",
		"image/jpeg",
		"image/webp",
	].includes(documentMimeType.toLowerCase());

	if (
		canResizeToJpeg &&
		effectiveBase64.length > RECEIPT_JPEG_BASE64_MAX_CHARS &&
		typeof document !== "undefined"
	) {
		const JPEG_QUALITY = 0.8;
		const MAX_EDGE = 1024;

		const resized = await (async (): Promise<{
			base64: string;
			dataUrl: string;
		} | null> => {
			try {
				const img = new Image();
				img.decoding = "async";

				await new Promise<void>((resolve, reject) => {
					img.onload = () => resolve();
					img.onerror = () => reject(new Error("Failed to decode image."));
					img.src = dataUrl;
				});

				const naturalWidth = img.naturalWidth ?? 0;
				const naturalHeight = img.naturalHeight ?? 0;
				const longestEdge = Math.max(naturalWidth, naturalHeight);
				if (!longestEdge) return null;

				const scale = longestEdge > MAX_EDGE ? MAX_EDGE / longestEdge : 1;
				const targetWidth = Math.max(1, Math.round(naturalWidth * scale));
				const targetHeight = Math.max(1, Math.round(naturalHeight * scale));

				const canvas = document.createElement("canvas");
				canvas.width = targetWidth;
				canvas.height = targetHeight;
				const ctx = canvas.getContext("2d");
				if (!ctx) return null;

				ctx.drawImage(img, 0, 0, targetWidth, targetHeight);

				const newDataUrl = canvas.toDataURL("image/jpeg", JPEG_QUALITY);
				const parts = newDataUrl.split(",", 2);
				const newBase64 = parts.length === 2 ? parts[1] : "";
				if (!newBase64) return null;

				return { base64: newBase64, dataUrl: newDataUrl };
			} catch {
				return null;
			}
		})();

		if (resized) {
			effectiveBase64 = resized.base64;
			effectivePreview = resized.dataUrl;
			effectiveDocumentMimeType = "image/jpeg";

			const dotIdx = effectiveDocumentFilename.lastIndexOf(".");
			const name =
				dotIdx > 0
					? effectiveDocumentFilename.slice(0, dotIdx)
					: effectiveDocumentFilename;
			effectiveDocumentFilename = `${name}.jpg`;
		}
	}

	return {
		base64: effectiveBase64,
		preview: effectivePreview,
		effectiveDocumentMimeType,
		effectiveDocumentFilename,
	};
}
```

### `src/app/receipts/receipt-api.ts`
```ts
import {
	type ReceiptProcessRequest,
	type ReceiptProcessResponse,
	type Receipt,
	receiptProcessRequestSchema,
	receiptProcessResponseSchema,
} from "@/modules/receipt-parser/schemas";
import { apiErrorFromJsonBody } from "@/lib/api-client";

const BASE_URL =
	process.env.NEXT_PUBLIC_BACKEND_URL ?? "http://localhost:8000/api/v1";

function resolveReceiptProcessTimeoutMs(): number {
	const configured =
		typeof process !== "undefined"
			? process.env.NEXT_PUBLIC_RECEIPT_PROCESS_TIMEOUT_MS
			: undefined;
	const parsed = configured ? Number(configured) : NaN;
	return Number.isFinite(parsed) && parsed > 0 ? parsed : 420_000;
}

export const RECEIPT_PROCESS_TIMEOUT_MS = resolveReceiptProcessTimeoutMs();

const MOCK_RECEIPTS_PROCESS_206_ENABLED =
	typeof process !== "undefined" &&
	process.env.NEXT_PUBLIC_MOCK_RECEIPTS_PROCESS_206 === "1";

const MOCK_BEST_EFFORT_RECEIPT: Receipt = {
	issuer_name: "ACME Supplies",
	payer_name: "John Doe",
	invoice_number: "INV-1",
	order_number: null,
	nif_cif_ssn: null,
	invoice_date: "2026-03-01",
	due_date: "2026-04-01",
	last_4_card_digits: "4905",
	type: "RECEIVED",
	user_role: "payer",
	items: [
		{
			description: "Service",
			quantity: 18.378,
			unit_price: "10.00",
			total: "10.00",
			discount_amount: null,
			tax_rate: null,
		},
	],
	subtotal: "10.00",
	discount: "0.00",
	tax: "0.00",
	total: "10.00",
	currency: "USD",
	taxes: [],
	id: null,
	created_at: null,
	updated_at: null,
	source: null,
	version: null,
};

const MOCK_RECEIPT_PROCESS_206_PAYLOAD: ReceiptProcessResponse = {
	status: "requires_human_correction",
	receipt: null,
	error_message: "Totals do not match",
	best_effort_receipt: MOCK_BEST_EFFORT_RECEIPT,
	validation_errors: [
		{
			rule: "math_total_mismatch",
			field: "total",
			expected: "10.50",
			actual: "10.00",
			item_line: null,
			message: "Total no coincide con los componentes.",
		},
		{
			rule: "math_item_total_mismatch",
			field: "items.total",
			expected: "10.50",
			actual: "10.00",
			item_line: 1,
			message: "El total de la linea no coincide.",
		},
	],
};

/**
 * Strips `data:image/...;base64,` so the body matches backend expectations.
 */
export function normalizeImageBase64ForApi(input: string): string {
	const trimmed = input.trim();
	const m = /^data:image\/[^;]+;base64,(.*)$/i.exec(trimmed);
	const payload = m?.[1];
	return payload !== undefined && payload !== "" ? payload : trimmed;
}

export function getReceiptHeaders(): Record<string, string> {
	const userId =
		typeof process !== "undefined"
			? process.env.NEXT_PUBLIC_USER_ID?.trim()
			: undefined;
	return userId ? { "X-User-Id": userId } : {};
}

export class ReceiptProcessError extends Error {
	constructor(
		message: string,
		public readonly status: 409 | 429,
		public readonly detail?: string
	) {
		super(message);
		this.name = "ReceiptProcessError";
	}
}

/** 422 from POST /receipts/process (invalid body per backend / client pre-check). */
export class ReceiptProcessValidationError extends Error {
	public readonly status = 422 as const;

	public constructor(
		message: string,
		public readonly detail?: string
	) {
		super(message);
		this.name = "ReceiptProcessValidationError";
	}
}

/** 200/206 body failed `receiptProcessResponseSchema` (contract drift). */
export class ReceiptProcessSchemaMismatchError extends Error {
	public constructor(
		message: string,
		public readonly rawKeys: string[],
		public readonly firstPath: string,
		public readonly firstMessage: string
	) {
		super(message);
		this.name = "ReceiptProcessSchemaMismatchError";
	}
}

/** 5xx from POST /receipts/process (server error). */
export class ReceiptProcessServerError extends Error {
	public readonly status: number;

	public constructor(message: string, status: number) {
		super(message);
		this.name = "ReceiptProcessServerError";
		this.status = status;
	}
}

/** 408/504 from receipt proxy or route timeout. */
export class ReceiptProcessTimeoutError extends Error {
	public readonly status: 408 | 504;

	public constructor(message: string, status: 408 | 504) {
		super(message);
		this.name = "ReceiptProcessTimeoutError";
		this.status = status;
	}
}

export function isReceiptProcessTimeoutError(error: unknown): boolean {
	return (
		(typeof error === "object" && error !== null && (
			(error as { name?: unknown }).name === "ReceiptProcessTimeoutError" ||
			(error as { status?: unknown }).status === 408 ||
			(error as { status?: unknown }).status === 504
		))
	);
}

export function scheduleReceiptsUsageInvalidation(delayMs = 20_000): void {
	if (typeof window === "undefined") {
		return;
	}
	window.setTimeout(() => {
		window.dispatchEvent(new CustomEvent("receipts-usage-invalidated"));
	}, delayMs);
}

function isAbortLikeError(error: unknown): boolean {
	if (typeof DOMException !== "undefined" && error instanceof DOMException) {
		return error.name === "AbortError";
	}
	return (
		!!error &&
		typeof error === "object" &&
		(error as { name?: unknown }).name === "AbortError"
	);
}

type AbortSignalAny = typeof AbortSignal & {
	any?: (signals: AbortSignal[]) => AbortSignal;
};

function composeAbortSignal(
	externalSignal: AbortSignal | undefined,
	timeoutMs: number,
): { signal: AbortSignal; cleanup: () => void } {
	const timeoutController = new AbortController();
	const timeoutId = setTimeout(() => {
		timeoutController.abort();
	}, timeoutMs);

	const anySignal = (AbortSignal as AbortSignalAny | undefined)?.any;
	if (typeof anySignal === "function") {
		const signals = externalSignal
			? [externalSignal, timeoutController.signal]
			: [timeoutController.signal];
		return {
			signal: anySignal(signals),
			cleanup: () => {
				clearTimeout(timeoutId);
			},
		};
	}

	if (!externalSignal) {
		return {
			signal: timeoutController.signal,
			cleanup: () => {
				clearTimeout(timeoutId);
			},
		};
	}

	const controller = new AbortController();
	const abort = () => {
		if (!controller.signal.aborted) {
			controller.abort();
		}
	};
	const onExternalAbort = () => {
		abort();
	};
	const onTimeoutAbort = () => {
		abort();
	};

	if (externalSignal.aborted) {
		abort();
	} else {
		externalSignal.addEventListener("abort", onExternalAbort, { once: true });
	}
	timeoutController.signal.addEventListener("abort", onTimeoutAbort, {
		once: true,
	});

	return {
		signal: controller.signal,
		cleanup: () => {
			clearTimeout(timeoutId);
			externalSignal.removeEventListener("abort", onExternalAbort);
			timeoutController.signal.removeEventListener("abort", onTimeoutAbort);
		},
	};
}

function extractErrorDetail(raw: unknown): string | undefined {
	if (!raw || typeof raw !== "object") return undefined;
	const o = raw as Record<string, unknown>;
	if (typeof o.detail === "string") return o.detail;
	if (typeof o.error_message === "string") return o.error_message;
	return undefined;
}

export type SubmitReceiptOptions = {
	user_role?: "issuer" | "payer";
	image_hash?: string;
	accessToken?: string | null;
	signal?: AbortSignal;
	document_mime_type?: string | null;
	document_filename?: string | null;
	timeoutMs?: number;
};

export async function submitReceiptToBackend(
	imageBase64: string,
	options: SubmitReceiptOptions = {}
): Promise<ReceiptProcessResponse> {
	const {
		user_role = "payer",
		image_hash,
		accessToken,
		signal,
		document_mime_type,
		document_filename,
		timeoutMs = RECEIPT_PROCESS_TIMEOUT_MS,
	} = options;
	const normalizedB64 = normalizeImageBase64ForApi(imageBase64);

	const parsedRequest = receiptProcessRequestSchema.safeParse({
		image_base64: normalizedB64,
		user_role,
		...(image_hash !== undefined ? { image_hash } : {}),
		...(document_mime_type !== undefined
			? { document_mime_type }
			: {}),
		...(document_filename !== undefined
			? { document_filename }
			: {}),
	});

	if (!parsedRequest.success) {
		const first = parsedRequest.error.issues[0];
		const msg = first
			? `${first.path.join(".")}: ${first.message}`
			: "Invalid receipt process request.";
		throw new ReceiptProcessValidationError(msg, msg);
	}

	const body: ReceiptProcessRequest = parsedRequest.data;

	if (MOCK_RECEIPTS_PROCESS_206_ENABLED) {
		const parsedMock = receiptProcessResponseSchema.safeParse(
			MOCK_RECEIPT_PROCESS_206_PAYLOAD
		);
		if (!parsedMock.success) {
			const first = parsedMock.error.issues[0];
			const path =
				first?.path?.length && first.path.length > 0
					? first.path.join(".")
					: "root";
			const rawKeys = Object.keys(MOCK_RECEIPT_PROCESS_206_PAYLOAD);
			const firstMessage = first?.message ?? "schema mismatch";
			if (process.env.NODE_ENV === "development") {
				console.error(
					"[receipt-api] Mocked 206 response validation failed.",
					{
						path,
						message: first?.message,
						issues: parsedMock.error.issues.slice(0, 5),
						rawKeys,
					}
				);
			}
			throw new ReceiptProcessSchemaMismatchError(
				`Invalid mocked 206 response: ${path} — ${firstMessage}.`,
				rawKeys,
				path,
				firstMessage
			);
		}
		return parsedMock.data;
	}

	const { signal: effectiveSignal, cleanup } = composeAbortSignal(
		signal,
		timeoutMs,
	);

	try {
		const url = `${BASE_URL.replace(/\/$/, "")}/receipts/process`;
		const headers: Record<string, string> = {
			"Content-Type": "application/json",
		};

		const token = accessToken?.trim();
		if (token) {
			headers.Authorization = `Bearer ${token}`;
		}

		let response: Response;
		try {
			response = await fetch(url, {
				method: "POST",
				headers,
				body: JSON.stringify(body),
				signal: effectiveSignal,
				credentials: "include",
			});
		} catch (error) {
			if (isAbortLikeError(error)) {
				throw new ReceiptProcessTimeoutError("Receipt process timed out.", 408);
			}
			throw error;
		}

		const rawJson = await response.json().catch(() => ({}));

		if (response.status === 401 || response.status === 403) {
			throw apiErrorFromJsonBody(response.status, rawJson);
		}

		const json =
			typeof rawJson === "object" &&
			rawJson !== null &&
			"data" in rawJson &&
			typeof (rawJson as { data: unknown }).data === "object"
				? (rawJson as { data: Record<string, unknown> }).data
				: rawJson;

		if (response.status === 409) {
			const detail = extractErrorDetail(rawJson);
			throw new ReceiptProcessError(
				detail ?? "A receipt with this image was already uploaded.",
				409,
				detail
			);
		}

		if (response.status === 429) {
			const detail = extractErrorDetail(rawJson);
			throw new ReceiptProcessError(
				detail ?? "Tier or storage limit reached.",
				429,
				detail
			);
		}

		if (response.status === 422) {
			const detail = extractErrorDetail(rawJson);
			throw new ReceiptProcessValidationError(
				detail ?? "Receipt process request rejected by server.",
				detail
			);
		}

		if (response.status === 408 || response.status === 504) {
			const detail = extractErrorDetail(rawJson);
			throw new ReceiptProcessTimeoutError(
				detail ?? "Receipt process timed out.",
				response.status
			);
		}

		if (response.status === 200 || response.status === 206) {
			const parsed = receiptProcessResponseSchema.safeParse(json);
			if (!parsed.success) {
				const issues = parsed.error.issues;
				const first = issues[0];
				const path = first?.path?.length ? first.path.join(".") : "root";
				const rawKeys =
					typeof json === "object" && json !== null ? Object.keys(json) : [];
				const firstMessage = first?.message ?? "schema mismatch";
				if (process.env.NODE_ENV === "development") {
					console.error("[receipt-api] Backend response validation failed.", {
						path,
						message: first?.message,
						firstIssue: first,
						issues: issues.slice(0, 5),
						rawKeys,
					});
				}
				throw new ReceiptProcessSchemaMismatchError(
					`Invalid process response from backend: ${path} — ${firstMessage}.`,
					rawKeys,
					path,
					firstMessage
				);
			}
			return parsed.data;
		}

		throw new ReceiptProcessServerError(
			`Receipt process failed with status ${response.status}.`,
			response.status
		);
	} catch (error) {
		if (isAbortLikeError(error)) {
			throw new ReceiptProcessTimeoutError("Receipt process timed out.", 408);
		}
		throw error;
	} finally {
		cleanup();
	}
}
```

### `src/app/receipts/receipt-calc.ts`
```ts
import {
	formatTaxRateBasisPointsToPercentString,
	parseTaxRateToBasisPoints,
} from "@/modules/receipt-parser/tax-rate";

export type DiscountMode = "amount" | "percent";

function normalizeDecimalInput(value: string): string {
	const trimmed = value.trim();
	if (!trimmed) return "";

	const hasComma = trimmed.includes(",");
	const hasDot = trimmed.includes(".");
	if (!hasComma) return trimmed;

	const noThousands = hasDot ? trimmed.replace(/\./g, "") : trimmed;
	return noThousands.replace(",", ".");
}

function parseNormalizedDecimalToCents(value: string): bigint | null {
	const normalized = normalizeDecimalInput(value);
	if (!normalized) return null;

	const sign = normalized.startsWith("-") ? -1n : 1n;
	const unsigned = sign === -1n ? normalized.slice(1) : normalized;

	const dotIdx = unsigned.indexOf(".");
	const intPartStr = dotIdx >= 0 ? unsigned.slice(0, dotIdx) : unsigned;
	const fracPartStr = dotIdx >= 0 ? unsigned.slice(dotIdx + 1) : "";

	const intPart = intPartStr ? BigInt(intPartStr) : 0n;

	const fracDigits = fracPartStr.replace(/[^\d]/g, "");
	if (fracDigits.length === 0) {
		return sign * intPart * 100n;
	}

	const firstTwo = fracDigits.slice(0, 2);
	const thirdDigit =
		fracDigits.length >= 3 ? fracDigits[2] ?? "0" : "0";

	const baseFraction = BigInt(firstTwo.padEnd(2, "0"));
	const shouldRoundUp = thirdDigit >= "5";
	const roundedFraction = baseFraction + (shouldRoundUp ? 1n : 0n);

	return sign * (intPart * 100n + roundedFraction);
}

export function parseDecimalStringToCents(
	value: string | null | undefined
): bigint | null {
	if (value === null || value === undefined) return null;
	if (typeof value !== "string") return null;
	return parseNormalizedDecimalToCents(value);
}

export function formatCentsToDecimalString(cents: bigint): string {
	const sign = cents < 0n ? "-" : "";
	const abs = cents < 0n ? -cents : cents;
	const intPart = abs / 100n;
	const fracPart = abs % 100n;
	return `${sign}${intPart.toString()}.${fracPart.toString().padStart(2, "0")}`;
}

export function divRoundHalfUp(numerator: bigint, divisor: bigint): bigint {
	if (divisor === 0n) return 0n;
	if (numerator === 0n) return 0n;
	if (numerator < 0n) {
		return -divRoundHalfUp(-numerator, divisor);
	}
	const quotient = numerator / divisor;
	const remainder = numerator % divisor;
	const shouldRoundUp = remainder * 2n >= divisor;
	return quotient + (shouldRoundUp ? 1n : 0n);
}

export function computeSubtotalCents(itemTotals: readonly string[]): bigint {
	let sum = 0n;
	for (const total of itemTotals) {
		const cents = parseDecimalStringToCents(total);
		if (cents === null) continue;
		sum += cents;
	}
	return sum < 0n ? 0n : sum;
}

export function computeDiscountCents(params: {
	discountMode: DiscountMode;
	subtotalCents: bigint;
	discountAmount: string;
	discountPercentInput: string;
}): bigint {
	const { discountMode, subtotalCents, discountAmount, discountPercentInput } = params;

	if (discountMode === "amount") {
		const discountCents = parseDecimalStringToCents(discountAmount) ?? 0n;
		return discountCents < 0n ? 0n : discountCents;
	}

	const percentCents = parseDecimalStringToCents(discountPercentInput) ?? 0n;
	const clampedPercentCents = percentCents < 0n ? 0n : percentCents;

	return divRoundHalfUp(subtotalCents * clampedPercentCents, 10000n);
}

export function computeDiscountPercentInputCents(params: {
	subtotalCents: bigint;
	discountCents: bigint;
}): bigint {
	const { subtotalCents, discountCents } = params;
	if (subtotalCents <= 0n) return 0n;
	const safeDiscount = discountCents < 0n ? 0n : discountCents;

	return divRoundHalfUp(safeDiscount * 10000n, subtotalCents);
}

export function computeTaxesFromRates(params: {
	baseCents: bigint;
	taxRates: readonly string[];
}): {
	taxCents: bigint;
	perLine: Array<{
		baseCents: bigint;
		taxCents: bigint;
		rateBasisPoints: bigint;
	}>;
} {
	const { baseCents, taxRates } = params;
	const safeBase = baseCents < 0n ? 0n : baseCents;

	const perLine = taxRates.map((rateStr) => {
		const rateBasisPoints = parseTaxRateToBasisPoints(rateStr) ?? 0n;
		const safeRateBasisPoints = rateBasisPoints < 0n ? 0n : rateBasisPoints;

		const taxCents = divRoundHalfUp(safeBase * safeRateBasisPoints, 10000n);
		return { baseCents: safeBase, taxCents, rateBasisPoints: safeRateBasisPoints };
	});

	const taxCents = perLine.reduce((acc, x) => acc + x.taxCents, 0n);
	return { taxCents: taxCents < 0n ? 0n : taxCents, perLine };
}

export type DerivedReceiptMathTaxLine = {
	base_amount: string;
	tax_amount: string;
	tax_rate: string;
};

export type DerivedReceiptMathResult = {
	subtotal: string;
	discount: string;
	tax: string;
	total: string;
	taxes: DerivedReceiptMathTaxLine[] | null;
};

export function deriveReceiptMath(params: {
	itemTotals: readonly string[];
	discountMode: DiscountMode;
	discountAmount: string;
	discountPercentInput: string;
	shouldAutoTax: boolean;
	taxRates: readonly string[];
	existingTax: string;
	shippingAmount?: string;
}): DerivedReceiptMathResult {
	const subtotalCents = computeSubtotalCents(params.itemTotals);

	const discountCents = computeDiscountCents({
		discountMode: params.discountMode,
		subtotalCents,
		discountAmount: params.discountAmount,
		discountPercentInput: params.discountPercentInput,
	});
	const shippingCents = parseDecimalStringToCents(params.shippingAmount ?? "0") ?? 0n;

	if (!params.shouldAutoTax) {
		const existingTaxCents = parseDecimalStringToCents(params.existingTax) ?? 0n;
		const safeExistingTax = existingTaxCents < 0n ? 0n : existingTaxCents;
		const totalCents = subtotalCents - discountCents + safeExistingTax + shippingCents;

		const safeTotal = totalCents < 0n ? 0n : totalCents;
		return {
			subtotal: formatCentsToDecimalString(subtotalCents),
			discount: formatCentsToDecimalString(discountCents),
			tax: formatCentsToDecimalString(safeExistingTax),
			total: formatCentsToDecimalString(safeTotal),
			taxes: null,
		};
	}

	const { taxCents, perLine } = computeTaxesFromRates({
		baseCents: subtotalCents,
		taxRates: params.taxRates,
	});

	const totalCents = subtotalCents - discountCents + taxCents + shippingCents;
	const safeTotal = totalCents < 0n ? 0n : totalCents;

	const taxes: DerivedReceiptMathTaxLine[] = perLine.map((line) => {
		return {
			base_amount: formatCentsToDecimalString(line.baseCents),
			tax_amount: formatCentsToDecimalString(line.taxCents),
			tax_rate: formatTaxRateBasisPointsToPercentString(line.rateBasisPoints),
		};
	});

	return {
		subtotal: formatCentsToDecimalString(subtotalCents),
		discount: formatCentsToDecimalString(discountCents),
		tax: formatCentsToDecimalString(taxCents),
		total: formatCentsToDecimalString(safeTotal),
		taxes,
	};
}
```

### `src/app/receipts/receipt-error-utils.ts`
```ts
export const RECEIPT_LIMIT_I18N_KEYS = [
		"receipts.error.limitReached",
		"receipts.error.dailyLimitReached",
		"receipts.error.monthlyLimitReached",
		"receipts.error.storageLimitReached",
		"receipts.error.maxVersionsReached",
		"receipts.error.dailyEditLimitReached",
] as const;

export function receiptLimitI18nKey(detail: string | undefined | null): string {
		if (!detail) return "receipts.error.limitReached";
		if (detail.includes("Daily receipt processing limit")) return "receipts.error.dailyLimitReached";
		if (detail.includes("Monthly receipt processing limit")) return "receipts.error.monthlyLimitReached";
		if (detail.includes("Receipt storage limit reached")) return "receipts.error.storageLimitReached";
		if (detail.includes("Maximum stored versions")) return "receipts.error.maxVersionsReached";
		if (detail.includes("Daily receipt history edit limit")) return "receipts.error.dailyEditLimitReached";
		return "receipts.error.limitReached";
}

const PAYG_INSUFFICIENT_BALANCE_PATTERNS = [
		"receipt_payg_balance_insufficient_fallback",
		"payg balance insufficient",
		"insufficient balance",
		"wallet balance too low",
		"not enough balance",
		"balance insufficient",
		"balance is insufficient",
		"insufficient funds",
		"top up required",
		"top-up required",
		"wallet depleted",
		"estimated charge exceeds balance",
] as const;

export function receiptInsufficientBalanceI18nKey(
		detail: string | undefined | null
): string | null {
		if (!detail) return null;
		const normalized = detail.toLowerCase();
		if (
				PAYG_INSUFFICIENT_BALANCE_PATTERNS.some((pattern) =>
						normalized.includes(pattern)
				)
		) {
				return "receipts.error.insufficientBalance";
		}
		return null;
}

export function receiptProcessErrorI18nKey(
		detail: string | undefined | null
): string {
		return receiptInsufficientBalanceI18nKey(detail) ?? receiptLimitI18nKey(detail);
}
```

### `src/app/receipts/receipt-feedback-api.ts`
```ts
import { apiRequest } from "@/lib/api-client";

export type ReportReceiptExtractionProblemRequest = {
	receipt_id: string;
	feedback_text: string;
};

export type ReportReceiptExtractionProblemResponse = {
	status: "saved";
};

/**
 * IMPORTANT:
 * Endpoint paths are placeholders until backend routes are confirmed.
 * Replace `[TBD]` values with the real API paths once available.
 */
const ENDPOINTS = {
	reportReceiptExtractionProblem: "[TBD]/receipts/history/feedback",
} as const;

function normalizeFeedbackText(input: string): string {
	return input.trim().replaceAll("\u0000", "");
}

export async function reportReceiptExtractionProblem(
	request: ReportReceiptExtractionProblemRequest,
): Promise<ReportReceiptExtractionProblemResponse> {
	const body: ReportReceiptExtractionProblemRequest = {
		receipt_id: request.receipt_id,
		feedback_text: normalizeFeedbackText(request.feedback_text).slice(0, 4000),
	};

	return apiRequest<ReportReceiptExtractionProblemResponse>(
		ENDPOINTS.reportReceiptExtractionProblem,
		{
			method: "POST",
			body,
		},
	);
}
```

### `src/app/receipts/receipt-history-api.ts`
```ts
import { apiErrorFromJsonBody, apiRequest } from "@/lib/api-client";
import { getBackendUrl } from "@/app/auth/backend-auth-api";
import { extractErrorPayloadFromJson } from "@/lib/api-error-detail";
import type { ReceiptForm } from "@/modules/receipt-parser/receipt-form-schema";
export type { ReceiptMathValidationError } from "@/modules/receipt-parser/schemas";
import type { ReceiptMathValidationError } from "@/modules/receipt-parser/schemas";
import type { Locale } from "@/lib/i18n-config";

/** GET list/detail may include this for image-chain versioning on save. */
export type ReceiptHistoryRow = ReceiptForm & {
	image_hash?: string | null;
};

export function persistedReceiptImageHash(
	row: ReceiptHistoryRow | null | undefined
): string | null {
	if (!row) return null;
	const h = row.image_hash;
	return typeof h === "string" && h.trim().length > 0 ? h.trim() : null;
}

/** Structured `detail.code` from DELETE /receipts/history/{id}. */
export const RECEIPT_DELETE_FORBIDDEN_CODE = "RECEIPT_DELETE_FORBIDDEN";

/** Structured `detail.code` from DELETE /receipts/history/{id}. */
export const RECEIPT_DELETE_NOT_FOUND_CODE = "RECEIPT_DELETE_NOT_FOUND";

function extractErrorDetailFromBody(raw: unknown): string | undefined {
	if (!raw || typeof raw !== "object") return undefined;
	const detail = (raw as { detail?: unknown }).detail;
	if (typeof detail === "string") return detail;
	if (Array.isArray(detail)) {
		const parts = detail
			.map((item) => {
				if (!item || typeof item !== "object") return null;
				const msg = (item as { msg?: unknown }).msg;
				return typeof msg === "string" ? msg : null;
			})
			.filter((x): x is string => x != null && x.length > 0);
		if (parts.length) return parts.join("; ");
	}
	return undefined;
}

/**
 * Receipt export/delete/save use `credentials: "include"` (HttpOnly cookie).
 * `Authorization` is added only when `accessToken` is non-empty after trim, so
 * we never send `Bearer` with an empty JWT. Backend (`api_cookie_auth`): absent
 * header, whitespace-only, or `Bearer` without token → cookie session; invalid
 * or expired JWT in Bearer → 401; non-Bearer schemes → 401.
 */
function receiptBearerHeaders(
	locale: Locale | undefined,
	accessToken: string | null | undefined,
): Record<string, string> {
	const headers: Record<string, string> = {};
	if (locale) {
		headers["Accept-Language"] = locale;
	}
	const t = accessToken?.trim();
	if (t) {
		headers.Authorization = `Bearer ${t}`;
	}
	return headers;
}

export type ReceiptHistoryRole = "issuer" | "payer";

export async function listReceiptsHistory(
	userRole?: ReceiptHistoryRole | null,
	limit = 100,
	offset = 0
): Promise<ReceiptHistoryRow[]> {
	const params = new URLSearchParams({
		limit: String(limit),
		offset: String(offset),
	});
	if (userRole) {
		params.set("user_role", userRole);
	}
	const data = await apiRequest<ReceiptHistoryRow[]>(
		`/receipts/history?${params.toString()}`,
		{
			method: "GET",
		}
	);
	return Array.isArray(data) ? data : [];
}

export async function getReceiptHistoryDetail(
	id: string,
): Promise<ReceiptHistoryRow> {
	return apiRequest<ReceiptHistoryRow>(`/receipts/history/${id}`, {
		method: "GET",
	});
}

export type SaveReceiptHistoryOptions = {
	userId?: string | null;
	imageHash?: string | null;
	accessToken?: string | null;
};

export type SaveReceiptHistoryRequest = {
	receipt: ReceiptForm;
	user_role: ReceiptHistoryRole;
	image_hash?: string | null;
	/**
	 * Best-effort/original text values captured before the user edited the
	 * receipt (HITL 206 flow from main page).
	 *
	 * Backend uses these to persist feedback_log mappings even when the math
	 * validation fails (422).
	 */
	original_issuer_name?: string | null;
	original_payer_name?: string | null;
	original_nif_cif_ssn?: string | null;
};

export type SaveReceiptHistoryResponse =
	| {
			status: "saved";
			receipt: ReceiptForm & { id?: string | null; version?: number | null };
		}
	| {
			status: "unprocessable";
			error_message: string | null;
			validation_errors: ReceiptMathValidationError[];
		};

function isReceiptMathValidationError(
	value: unknown
): value is ReceiptMathValidationError {
	if (!value || typeof value !== "object") return false;
	const v = value as {
		rule?: unknown;
		field?: unknown;
		item_line?: unknown;
		expected?: unknown;
		actual?: unknown;
		message?: unknown;
	};

	if (v.rule !== null && v.rule !== undefined && typeof v.rule !== "string") return false;

	if (typeof v.field !== "string") return false;
	if (typeof v.message !== "string") return false;

	const checkNullableNumber = (n: unknown): boolean => {
		if (n === null || n === undefined) return true;
		return typeof n === "number" && Number.isInteger(n);
	};
	if (!checkNullableNumber(v.item_line)) return false;

	const checkNullableString = (s: unknown): boolean => {
		if (s === null || s === undefined) return true;
		return typeof s === "string";
	};
	if (!checkNullableString(v.expected)) return false;
	if (!checkNullableString(v.actual)) return false;

	return true;
}

function normalizeValidationErrors(
	raw: unknown
): ReceiptMathValidationError[] {
	if (!raw || typeof raw !== "object") return [];
	if (!Array.isArray(raw)) return [];
	const arr = raw as unknown[];
	return arr.filter(isReceiptMathValidationError);
}

export class SaveReceiptHistoryError extends Error {
	public readonly status: number;
	public readonly detail?: string;

	public constructor(message: string, status: number, detail?: string) {
		super(message);
		this.name = "SaveReceiptHistoryError";
		this.status = status;
		this.detail = detail;
	}
}

export async function saveReceiptToHistory(
	request: SaveReceiptHistoryRequest,
	options: SaveReceiptHistoryOptions = {}
): Promise<SaveReceiptHistoryResponse> {
	const url = getBackendUrl("/receipts/history");

	const headers: Record<string, string> = {
		"Content-Type": "application/json",
		...receiptBearerHeaders(undefined, options.accessToken),
	};

	const body: Record<string, unknown> = {
		receipt: request.receipt,
		user_role: request.user_role,
		image_hash: request.image_hash ?? options.imageHash ?? null,
	};

	if (request.original_issuer_name !== undefined) {
		body.original_issuer_name = request.original_issuer_name;
	}
	if (request.original_payer_name !== undefined) {
		body.original_payer_name = request.original_payer_name;
	}
	if (request.original_nif_cif_ssn !== undefined) {
		body.original_nif_cif_ssn = request.original_nif_cif_ssn;
	}

	const res = await fetch(url, {
		method: "POST",
		headers,
		body: JSON.stringify(body),
		credentials: "include",
	});

	if (res.status === 422) {
		const raw = await res.json().catch(() => null);
		const errorMessage =
			raw && typeof raw === "object" && "error_message" in raw
				? typeof (raw as { error_message?: unknown }).error_message === "string"
					? ((raw as { error_message: string }).error_message ?? null)
					: null
				: null;
		const validationErrors = normalizeValidationErrors(
			raw && typeof raw === "object" && "validation_errors" in raw
				? (raw as { validation_errors?: unknown }).validation_errors
				: undefined
		);

		return {
			status: "unprocessable",
			error_message: errorMessage,
			validation_errors: validationErrors,
		};
	}

	if (!res.ok) {
		let raw: unknown;
		try {
			raw = await res.json();
		} catch {
			raw = null;
		}
		const extracted = extractErrorPayloadFromJson(raw);

		if (res.status === 401) {
			throw apiErrorFromJsonBody(res.status, raw);
		}
		if (
			res.status === 403 &&
			extracted.structured?.code === "WRONG_PRODUCT_MODULE"
		) {
			throw apiErrorFromJsonBody(res.status, raw);
		}

		const errMsg =
			raw && typeof raw === "object" &&
			typeof (raw as { error_message?: unknown }).error_message === "string"
				? (raw as { error_message: string }).error_message
				: undefined;
		const detail =
			extracted.structured?.message ?? extracted.stringDetail ?? errMsg;

		throw new SaveReceiptHistoryError(
			`Save receipt failed with status ${res.status}.`,
			res.status,
			detail
		);
	}

	return (await res.json()) as SaveReceiptHistoryResponse;
}

/** Triggers CSV download from GET /receipts/export. */
export async function exportReceiptsCsv(
	userRole: ReceiptHistoryRole | undefined,
	locale?: Locale,
	accessToken?: string | null,
): Promise<void> {
	const params = new URLSearchParams();
	if (userRole) {
		params.set("user_role", userRole);
	}
	const qs = params.toString();
	const path = qs ? `/receipts/export?${qs}` : `/receipts/export`;
	const url = getBackendUrl(path);
	const hdrs = receiptBearerHeaders(locale, accessToken);
	const res = await fetch(url, {
		method: "GET",
		credentials: "include",
		headers: Object.keys(hdrs).length ? hdrs : undefined,
	});
	if (!res.ok) {
		let raw: unknown;
		try {
			raw = await res.json();
		} catch {
			raw = null;
		}
		throw apiErrorFromJsonBody(res.status, raw);
	}
	const blob = await res.blob();
	const disposition = res.headers.get("Content-Disposition");
	const filename =
		disposition?.match(/filename[*]?=(?:UTF-8'')?["']?([^"'\s]+)["']?/i)?.[1] ??
		"receipts_export.csv";
	const a = document.createElement("a");
	a.href = URL.createObjectURL(blob);
	a.download = filename;
	a.click();
	URL.revokeObjectURL(a.href);
}

export async function deleteReceiptFromHistory(
	id: string,
	accessToken?: string | null,
): Promise<void> {
	const url = getBackendUrl(
		`/receipts/history/${encodeURIComponent(id)}`,
	);

	const hdrs = receiptBearerHeaders(undefined, accessToken);
	const res = await fetch(url, {
		method: "DELETE",
		credentials: "include",
		headers: Object.keys(hdrs).length ? hdrs : undefined,
	});

	if (res.ok) return;

	let rawBody: unknown;
	try {
		rawBody = await res.json();
	} catch {
		rawBody = null;
	}

	throw apiErrorFromJsonBody(res.status, rawBody);
}

/** Triggers XLSX download from GET /receipts/export (format=xlsx). */
export async function exportReceiptsXlsx(
	userRole: ReceiptHistoryRole | undefined,
	locale: Locale,
	accessToken?: string | null,
): Promise<void> {
	const params = new URLSearchParams();
	params.set("format", "xlsx");
	if (userRole) {
		params.set("user_role", userRole);
	}

	const url = getBackendUrl(`/receipts/export?${params.toString()}`);

	const res = await fetch(url, {
		method: "GET",
		credentials: "include",
		headers: receiptBearerHeaders(locale, accessToken),
	});

	if (!res.ok) {
		let raw: unknown;
		try {
			raw = await res.json();
		} catch {
			raw = null;
		}
		throw apiErrorFromJsonBody(res.status, raw);
	}

	const blob = await res.blob();
	const disposition = res.headers.get("Content-Disposition");
	const filename =
		disposition?.match(/filename[*]?=(?:UTF-8'')?["']?([^"'\s]+)["']?/i)?.[1] ??
		"receipts_export.xlsx";

	const a = document.createElement("a");
	a.href = URL.createObjectURL(blob);
	a.download = filename;
	a.click();
	URL.revokeObjectURL(a.href);
}
```

### `src/app/receipts/receipt-math-validation-i18n.ts`
```ts
import type { Locale } from "@/lib/i18n-config";
import type { ReceiptMathValidationError } from "@/modules/receipt-parser/schemas";

type ReceiptStaticTextKey =
	| "requiresReviewSuffix"
	| "correctionRequiredHeading"
	| "correctionRequiredPrefix"
	| "validationErrorsTitle"
	| "fieldLabel"
	| "expectedLabel"
	| "gotLabel"
	| "dueDateLabel"
	| "invoiceNumberLabel"
	| "orderNumberLabel"
	| "nifCifSsnLabel"
	| "last4CardDigitsLabel"
	| "discountLabel"
	| "discountPercentLabel"
	| "taxesBreakdownLabel"
	| "addTaxLabel"
	| "removeTaxLabel"
	| "rateLabel"
	| "baseLabel"
	| "taxAmountLabel"
	| "noTaxesBreakdownLabel"
	| "billingIdLabel"
	| "shippingLabel"
	| "paymentReceivedLabel"
	| "changeDueLabel"
	| "sellerLabel"
	| "buyerLabel"
	| "issueDateLabel"
	| "issueDatePlaceholder"
	| "dueDatePlaceholder"
	| "itemsLabel"
	| "taxLabel"
	| "anonymizedValueLabel"
	| "recalculateLabel";

type ReceiptStaticTexts = Record<ReceiptStaticTextKey, string>;

const staticTextsByLocale: Record<Locale, ReceiptStaticTexts> = {
	en: {
		requiresReviewSuffix: "Requires review.",
		correctionRequiredHeading: "Correction required",
		correctionRequiredPrefix: "Correction required:",
		validationErrorsTitle: "Validation errors",
		fieldLabel: "Field",
		expectedLabel: "Expected",
		gotLabel: "Got",
		dueDateLabel: "Due date",
		invoiceNumberLabel: "Invoice #",
		orderNumberLabel: "Order #",
		nifCifSsnLabel: "NIF/CIF/SSN",
		last4CardDigitsLabel: "Last 4 card digits",
		discountLabel: "Discount (Amount)",
		discountPercentLabel: "Discount %",
		taxesBreakdownLabel: "Taxes (breakdown)",
		addTaxLabel: "Add",
		removeTaxLabel: "Remove",
		rateLabel: "Rate (%)",
		baseLabel: "Base",
		taxAmountLabel: "Tax",
		noTaxesBreakdownLabel: "No taxes breakdown detected.",
		billingIdLabel: "Billing ID",
		shippingLabel: "Shipping / Delivery",
		paymentReceivedLabel: "Payment Received",
		changeDueLabel: "Change Due",
		sellerLabel: "Seller",
		buyerLabel: "Buyer",
		issueDateLabel: "Issue date",
		issueDatePlaceholder: "YYYY-MM-DD or YYYY-MM-DDTHH:mm:ss",
		dueDatePlaceholder: "YYYY-MM-DD, N/A, N/D, or empty",
		itemsLabel: "Items",
		taxLabel: "Tax",
		anonymizedValueLabel: "Value hidden by anonymization",
		recalculateLabel: "Recalculate",
	},
	es: {
		requiresReviewSuffix: "Requiere revisión.",
		correctionRequiredHeading: "Corrección requerida",
		correctionRequiredPrefix: "Corrección requerida:",
		validationErrorsTitle: "Errores de validación",
		fieldLabel: "Campo",
		expectedLabel: "Esperado",
		gotLabel: "Recibido",
		dueDateLabel: "Fecha de vencimiento",
		invoiceNumberLabel: "Nº factura",
		orderNumberLabel: "Nº pedido",
		nifCifSsnLabel: "NIF/CIF/SSN",
		last4CardDigitsLabel: "Últimos 4 dígitos de tarjeta",
		discountLabel: "Descuento (Moneda)",
		discountPercentLabel: "Descuento %",
		taxesBreakdownLabel: "Impuestos (detalle)",
		addTaxLabel: "Añadir",
		removeTaxLabel: "Quitar",
		rateLabel: "Tasa (%)",
		baseLabel: "Base",
		taxAmountLabel: "Impuesto",
		noTaxesBreakdownLabel: "No se detectó el desglose de impuestos.",
		billingIdLabel: "ID Facturación",
		shippingLabel: "Costes de Envío",
		paymentReceivedLabel: "Recibido",
		changeDueLabel: "Cambio",
		sellerLabel: "Vendedor",
		buyerLabel: "Comprador",
		issueDateLabel: "Fecha de emisión",
		issueDatePlaceholder: "AAAA-MM-DD o AAAA-MM-DDTHH:mm:ss",
		dueDatePlaceholder: "AAAA-MM-DD, N/A, N/D o vacío",
		itemsLabel: "Artículos",
		taxLabel: "Impuestos",
		anonymizedValueLabel: "Valor oculto por anonimización",
		recalculateLabel: "Recalcular",
	},
	de: {
		requiresReviewSuffix: "Erfordert Überprüfung.",
		correctionRequiredHeading: "Korrektur erforderlich",
		correctionRequiredPrefix: "Korrektur erforderlich:",
		validationErrorsTitle: "Validierungsfehler",
		fieldLabel: "Feld",
		expectedLabel: "Erwartet",
		gotLabel: "Erhalten",
		dueDateLabel: "Fälligkeitsdatum",
		invoiceNumberLabel: "Rechnungs-Nr.",
		orderNumberLabel: "Bestell-Nr.",
		nifCifSsnLabel: "NIF/CIF/SSN",
		last4CardDigitsLabel: "Letzte 4 Kartenziffern",
		discountLabel: "Rabatt (Betrag)",
		discountPercentLabel: "Rabatt %",
		taxesBreakdownLabel: "Steuern (Aufschlüsselung)",
		addTaxLabel: "Hinzufügen",
		removeTaxLabel: "Entfernen",
		rateLabel: "Satz (%)",
		baseLabel: "Basis",
		taxAmountLabel: "Steuer",
		noTaxesBreakdownLabel: "Kein Steuerschlüssel erkannt.",
		billingIdLabel: "Rechnungs-ID",
		shippingLabel: "Versandkosten",
		paymentReceivedLabel: "Erhalten",
		changeDueLabel: "Wechselgeld",
		sellerLabel: "Verkäufer",
		buyerLabel: "Käufer",
		issueDateLabel: "Rechnungsdatum",
		issueDatePlaceholder: "JJJJ-MM-TT oder JJJJ-MM-TTTHH:mm:ss",
		dueDatePlaceholder: "JJJJ-MM-TT, N/A, N/D oder leer",
		itemsLabel: "Artikel",
		taxLabel: "Steuer",
		anonymizedValueLabel: "Wert durch Anonymisierung verborgen",
		recalculateLabel: "Neu berechnen",
	},
	it: {
		requiresReviewSuffix: "Richiede revisione.",
		correctionRequiredHeading: "Correzione necessaria",
		correctionRequiredPrefix: "Correzione necessaria:",
		validationErrorsTitle: "Errori di validazione",
		fieldLabel: "Campo",
		expectedLabel: "Atteso",
		gotLabel: "Ottenuto",
		dueDateLabel: "Data di scadenza",
		invoiceNumberLabel: "N. fattura",
		orderNumberLabel: "N. ordine",
		nifCifSsnLabel: "NIF/CIF/SSN",
		last4CardDigitsLabel: "Ultime 4 cifre carta",
		discountLabel: "Sconto (Importo)",
		discountPercentLabel: "Sconto %",
		taxesBreakdownLabel: "Tasse (dettaglio)",
		addTaxLabel: "Aggiungi",
		removeTaxLabel: "Rimuovi",
		rateLabel: "Aliquota (%)",
		baseLabel: "Base",
		taxAmountLabel: "Imposta",
		noTaxesBreakdownLabel: "Nessun dettaglio delle imposte rilevato.",
		billingIdLabel: "ID Fatturazione",
		shippingLabel: "Spese di Spedizione",
		paymentReceivedLabel: "Ricevuto",
		changeDueLabel: "Resto",
		sellerLabel: "Venditore",
		buyerLabel: "Acquirente",
		issueDateLabel: "Data di emissione",
		issueDatePlaceholder: "AAAA-MM-GG o AAAA-MM-GGTHH:mm:ss",
		dueDatePlaceholder: "AAAA-MM-GG, N/A, N/D o vuoto",
		itemsLabel: "Articoli",
		taxLabel: "Imposte",
		anonymizedValueLabel: "Valore nascosto per anonimizzazione",
		recalculateLabel: "Ricalcola",
	},
	fr: {
		requiresReviewSuffix: "Nécessite une vérification.",
		correctionRequiredHeading: "Correction requise",
		correctionRequiredPrefix: "Correction requise :",
		validationErrorsTitle: "Erreurs de validation",
		fieldLabel: "Champ",
		expectedLabel: "Attendu",
		gotLabel: "Reçu",
		dueDateLabel: "Date d'échéance",
		invoiceNumberLabel: "N° facture",
		orderNumberLabel: "N° commande",
		nifCifSsnLabel: "NIF/CIF/SSN",
		last4CardDigitsLabel: "4 derniers chiffres de carte",
		discountLabel: "Remise (Montant)",
		discountPercentLabel: "Remise %",
		taxesBreakdownLabel: "Taxes (détail)",
		addTaxLabel: "Ajouter",
		removeTaxLabel: "Supprimer",
		rateLabel: "Taux (%)",
		baseLabel: "Base",
		taxAmountLabel: "Taxe",
		noTaxesBreakdownLabel: "Aucun détail des taxes détecté.",
		billingIdLabel: "ID Facturation",
		shippingLabel: "Frais de Livraison",
		paymentReceivedLabel: "Reçu",
		changeDueLabel: "Monnaie",
		sellerLabel: "Vendeur",
		buyerLabel: "Acheteur",
		issueDateLabel: "Date d'émission",
		issueDatePlaceholder: "AAAA-MM-JJ ou AAAA-MM-JJTHH:mm:ss",
		dueDatePlaceholder: "AAAA-MM-JJ, N/A, N/D ou vide",
		itemsLabel: "Articles",
		taxLabel: "Taxes",
		anonymizedValueLabel: "Valeur masquée par anonymisation",
		recalculateLabel: "Recalculer",
	},
	ru: {
		requiresReviewSuffix: "Требуется проверка.",
		correctionRequiredHeading: "Требуется исправление",
		correctionRequiredPrefix: "Требуется исправление:",
		validationErrorsTitle: "Ошибки валидации",
		fieldLabel: "Поле",
		expectedLabel: "Ожидалось",
		gotLabel: "Получено",
		dueDateLabel: "Срок оплаты",
		invoiceNumberLabel: "Номер счета",
		orderNumberLabel: "Номер заказа",
		nifCifSsnLabel: "NIF/CIF/SSN",
		last4CardDigitsLabel: "Последние 4 цифры карты",
		discountLabel: "Скидка (Сумма)",
		discountPercentLabel: "Скидка %",
		taxesBreakdownLabel: "Налоги (разбивка)",
		addTaxLabel: "Добавить",
		removeTaxLabel: "Удалить",
		rateLabel: "Ставка (%)",
		baseLabel: "Основа",
		taxAmountLabel: "Налог",
		noTaxesBreakdownLabel: "Разбивка налогов не обнаружена.",
		billingIdLabel: "ID Счёта",
		shippingLabel: "Стоимость доставки",
		paymentReceivedLabel: "Получено",
		changeDueLabel: "Сдача",
		sellerLabel: "Продавец",
		buyerLabel: "Покупатель",
		issueDateLabel: "Дата выставления",
		issueDatePlaceholder: "ГГГГ-ММ-ДД или ГГГГ-ММ-ДДTHH:mm:ss",
		dueDatePlaceholder: "ГГГГ-ММ-ДД, N/A, N/D или пусто",
		itemsLabel: "Позиции",
		taxLabel: "Налог",
		anonymizedValueLabel: "Значение скрыто анонимизацией",
		recalculateLabel: "Пересчитать",
	},
};

const PII_PLACEHOLDER_REGEX = /^(ORGANIZATION|PERSON|PHONE|DATE_TIME)_\d+$/;

const expectedValueMap: Record<string, Record<Locale, string>> = {
	"real value from document": {
		en: "real value from document",
		es: "valor real del documento",
		de: "tatsächlicher Wert aus dem Dokument",
		it: "valore reale del documento",
		fr: "valeur réelle du document",
		ru: "реальное значение из документа",
	},
	"visible invoice identifier": {
		en: "visible invoice identifier",
		es: "identificador visible de factura",
		de: "sichtbare Rechnungskennung",
		it: "identificatore visibile della fattura",
		fr: "identifiant visible de facture",
		ru: "видимый идентификатор счёта",
	},
	"printed date on document": {
		en: "printed date on document",
		es: "fecha impresa en el documento",
		de: "auf dem Dokument gedrucktes Datum",
		it: "data stampata sul documento",
		fr: "date imprimée sur le document",
		ru: "дата, напечатанная в документе",
	},
	"printed numeric amount on document": {
		en: "printed numeric amount on document",
		es: "importe numérico impreso en el documento",
		de: "gedruckter numerischer Betrag im Dokument",
		it: "importo numerico stampato sul documento",
		fr: "montant numérique imprimé sur le document",
		ru: "числовая сумма, напечатанная в документе",
	},
	"at least 1 item": {
		en: "at least 1 item",
		es: "al menos 1 artículo",
		de: "mindestens 1 Artikel",
		it: "almeno 1 articolo",
		fr: "au moins 1 article",
		ru: "как минимум 1 позиция",
	},
	"non-null value": {
		en: "non-null value",
		es: "valor no nulo",
		de: "Wert ungleich null",
		it: "valore non nullo",
		fr: "valeur non nulle",
		ru: "не пустое значение",
	},
};

function getStaticText(locale: Locale, key: ReceiptStaticTextKey): string {
	return staticTextsByLocale[locale]?.[key] ?? staticTextsByLocale.en[key];
}

export function isAnonymizedPlaceholder(value: string | null | undefined): boolean {
	if (typeof value !== "string") return false;
	return PII_PLACEHOLDER_REGEX.test(value.trim());
}

export function formatValidationValue(
	locale: Locale,
	value: string | null | undefined
): string {
	if (value == null) return "N/D";
	const trimmed = value.trim();
	if (!trimmed) return "N/D";
	if (isAnonymizedPlaceholder(trimmed)) {
		return `${getStaticText(locale, "anonymizedValueLabel")} (${trimmed})`;
	}
	return expectedValueMap[trimmed]?.[locale] ?? trimmed;
}

function formatExpectedActual(
	locale: Locale,
	e: ReceiptMathValidationError
): { expectedText: string; actualText: string } {
	const expectedText =
		e.expected === null || e.expected === undefined || e.expected === ""
			? "N/D"
			: formatValidationValue(locale, e.expected);
	const actualText =
		e.actual === null || e.actual === undefined || e.actual === ""
			? "N/D"
			: formatValidationValue(locale, e.actual);
	return { expectedText, actualText };
}

function getRuleDetail(
	locale: Locale,
	rule: string | null | undefined,
	expectedText: string,
	actualText: string
): string {
	const safeRule = rule ?? "unknown_rule";

	if (safeRule === "missing_date_placeholder") {
		switch (locale) {
			case "es":
				return `No se pudo extraer la fecha del campo ${expectedText}. Requiere revisión manual.`;
			case "de":
				return `Das Datum für Feld ${expectedText} konnte nicht extrahiert werden. Manuelle Prüfung erforderlich.`;
			case "it":
				return `Impossibile estrarre la data per il campo ${expectedText}. Revisione manuale richiesta.`;
			case "fr":
				return `Impossible d'extraire la date pour le champ ${expectedText}. Vérification manuelle requise.`;
			case "ru":
				return `Не удалось извлечь дату для поля ${expectedText}. Требуется ручная проверка.`;
			default:
				return `Could not extract date for field ${expectedText}. Requires manual review.`;
		}
	}

	if (safeRule === "presidio_placeholder_leaked") {
		switch (locale) {
			case "es":
				return `El campo ${expectedText} contenía un valor de PII enmascarado. Requiere revisión manual.`;
			case "de":
				return `Das Feld ${expectedText} enthielt einen maskierten PII-Wert. Manuelle Prüfung erforderlich.`;
			case "it":
				return `Il campo ${expectedText} conteneva un valore PII mascherato. Revisione manuale richiesta.`;
			case "fr":
				return `Le champ ${expectedText} contenait une valeur PII masquée. Vérification manuelle requise.`;
			case "ru":
				return `Поле ${expectedText} содержало замаскированное значение PII. Требуется ручная проверка.`;
			default:
				return `Field ${expectedText} contained a masked PII value. Requires manual review.`;
		}
	}

	if (safeRule === "missing_invoice_number") {
		switch (locale) {
			case "es":
				return "No se pudo extraer el número de factura.";
			case "de":
				return "Rechnungsnummer konnte nicht extrahiert werden.";
			case "it":
				return "Impossibile estrarre il numero di fattura.";
			case "fr":
				return "Impossible d'extraire le numéro de facture.";
			case "ru":
				return "Не удалось извлечь номер счёта.";
			default:
				return "Could not extract invoice number.";
		}
	}

	if (safeRule === "missing_numeric_placeholder") {
		switch (locale) {
			case "es":
				return `No se pudo extraer el valor numérico de ${expectedText}.`;
			case "de":
				return `Numerischer Wert für ${expectedText} konnte nicht extrahiert werden.`;
			case "it":
				return `Impossibile estrarre il valore numerico di ${expectedText}.`;
			case "fr":
				return `Impossible d'extraire la valeur numérique de ${expectedText}.`;
			case "ru":
				return `Не удалось извлечь числовое значение для ${expectedText}.`;
			default:
				return `Could not extract numeric value for ${expectedText}.`;
		}
	}

	if (safeRule === "missing_item_quantity_placeholder") {
		switch (locale) {
			case "es":
				return "La cantidad del artículo no se pudo extraer.";
			case "de":
				return "Artikelmenge konnte nicht extrahiert werden.";
			case "it":
				return "Impossibile estrarre la quantità dell'articolo.";
			case "fr":
				return "Impossible d'extraire la quantité de l'article.";
			case "ru":
				return "Не удалось извлечь количество товара.";
			default:
				return "Item quantity could not be extracted.";
		}
	}

	if (safeRule === "missing_item_unit_price_placeholder") {
		switch (locale) {
			case "es":
				return "El precio unitario no se pudo extraer.";
			case "de":
				return "Stückpreis konnte nicht extrahiert werden.";
			case "it":
				return "Impossibile estrarre il prezzo unitario.";
			case "fr":
				return "Impossible d'extraire le prix unitaire.";
			case "ru":
				return "Не удалось извлечь цену за единицу.";
			default:
				return "Item unit price could not be extracted.";
		}
	}

	if (safeRule === "non_item_charge_removed") {
		switch (locale) {
			case "es":
				return "Se excluyó un cargo de envío/servicio de los artículos.";
			case "de":
				return "Eine Versand-/Servicegebühr wurde aus den Artikeln ausgeschlossen.";
			case "it":
				return "Un costo di spedizione/servizio è stato escluso dagli articoli.";
			case "fr":
				return "Des frais de livraison/service ont été exclus des articles.";
			case "ru":
				return "Плата за доставку/обслуживание была исключена из позиций.";
			default:
				return "A shipping/service charge was excluded from items.";
		}
	}

	if (safeRule === "subtotal_discount_tax_total_mismatch") {
		switch (locale) {
			case "es":
				return `Subtotal - descuento + impuestos no coincide con el total: esperado ${expectedText}, obtenido ${actualText}.`;
			case "de":
				return `Zwischensumme - Rabatt + Steuer stimmt nicht mit dem Gesamtbetrag überein: erwartet ${expectedText}, erhalten ${actualText}.`;
			case "it":
				return `Subtotale - sconto + tasse non corrisponde al totale: atteso ${expectedText}, ottenuto ${actualText}.`;
			case "fr":
				return `Le sous-total - remise + taxe ne correspond pas au total : attendu ${expectedText}, obtenu ${actualText}.`;
			case "ru":
				return `Промежуточный итог - скидка + налог не совпадает с итогом: ожидается ${expectedText}, получено ${actualText}.`;
			default:
				return `Subtotal - discount + tax does not match total: expected ${expectedText}, got ${actualText}.`;
		}
	}

	switch (locale) {
		case "es":
			return `La regla ${safeRule} no coincide: esperado ${expectedText}, obtenido ${actualText}.`;
		case "de":
			return `Regel ${safeRule} stimmt nicht: erwartet ${expectedText}, erhalten ${actualText}.`;
		case "it":
			return `La regola ${safeRule} non corrisponde: atteso ${expectedText}, ottenuto ${actualText}.`;
		case "fr":
			return `La règle ${safeRule} ne correspond pas : attendu ${expectedText}, obtenu ${actualText}.`;
		case "ru":
			return `Правило ${safeRule} не совпало: ожидалось ${expectedText}, получено ${actualText}.`;
		default:
			return `Rule ${safeRule} did not match: expected ${expectedText}, got ${actualText}.`;
	}
}

const EXTRACTION_FAILURE_RULES = new Set([
	"missing_date_placeholder",
	"presidio_placeholder_leaked",
	"missing_invoice_number",
	"missing_numeric_placeholder",
	"missing_item_quantity_placeholder",
	"missing_item_unit_price_placeholder",
	"non_item_charge_removed",
]);

export function formatReceiptMathValidationSuggestion(
	locale: Locale,
	e: ReceiptMathValidationError
): string {
	const { expectedText, actualText } = formatExpectedActual(locale, e);

	if (e.rule != null && EXTRACTION_FAILURE_RULES.has(e.rule)) {
		return getRuleDetail(locale, e.rule, expectedText, actualText);
	}

	const baseSentence = (() => {
		const isItemTotal =
			e.field === "items.total" ||
			(typeof e.field === "string" && e.field.endsWith("items.total"));

		if (isItemTotal) {
			if (e.item_line != null) {
				switch (locale) {
					case "es":
						return `El total de la línea #${e.item_line} debe ser ${expectedText}. Ahora es ${actualText}.`;
					case "de":
						return `Zeilensumme #${e.item_line} muss ${expectedText} sein. Aktuell: ${actualText}.`;
					case "it":
						return `Il totale della riga #${e.item_line} deve essere ${expectedText}. Ora è ${actualText}.`;
					case "fr":
						return `Le total de la ligne #${e.item_line} doit être ${expectedText}. Actuel : ${actualText}.`;
					case "ru":
						return `Сумма в строке #${e.item_line} должна быть ${expectedText}. Сейчас: ${actualText}.`;
					default:
						return `Line #${e.item_line} total must be ${expectedText}. It is ${actualText}.`;
				}
			}

			switch (locale) {
				case "es":
					return `El total de artículos debe ser ${expectedText}. Ahora es ${actualText}.`;
				case "de":
					return `Gesamtsumme der Positionen muss ${expectedText} sein. Aktuell: ${actualText}.`;
				case "it":
					return `Il totale degli elementi deve essere ${expectedText}. Ora è ${actualText}.`;
				case "fr":
					return `Le total des éléments doit être ${expectedText}. Actuel : ${actualText}.`;
				case "ru":
					return `Итог по позициям должен быть ${expectedText}. Сейчас: ${actualText}.`;
				default:
					return `Items total must be ${expectedText}. It is ${actualText}.`;
			}
		}

		switch (e.field) {
			case "total":
				switch (locale) {
					case "es":
						return `El total debe ser ${expectedText}. Ahora es ${actualText}.`;
					case "de":
						return `Der Gesamtbetrag muss ${expectedText} sein. Aktuell: ${actualText}.`;
					case "it":
						return `Il totale deve essere ${expectedText}. Ora è ${actualText}.`;
					case "fr":
						return `Le total doit être ${expectedText}. Actuel : ${actualText}.`;
					case "ru":
						return `Итог должен быть ${expectedText}. Сейчас: ${actualText}.`;
					default:
						return `Total must be ${expectedText}. It is ${actualText}.`;
				}
			case "subtotal":
				switch (locale) {
					case "es":
						return `El subtotal debe ser ${expectedText}. Ahora es ${actualText}.`;
					case "de":
						return `Zwischensumme muss ${expectedText} sein. Aktuell: ${actualText}.`;
					case "it":
						return `Il subtotale deve essere ${expectedText}. Ora è ${actualText}.`;
					case "fr":
						return `Le sous-total doit être ${expectedText}. Actuel : ${actualText}.`;
					case "ru":
						return `Промежуточный итог должен быть ${expectedText}. Сейчас: ${actualText}.`;
					default:
						return `Subtotal must be ${expectedText}. It is ${actualText}.`;
				}
			case "discount":
				switch (locale) {
					case "es":
						return `El descuento debe ser ${expectedText}. Ahora es ${actualText}.`;
					case "de":
						return `Rabatt muss ${expectedText} sein. Aktuell: ${actualText}.`;
					case "it":
						return `Lo sconto deve essere ${expectedText}. Ora è ${actualText}.`;
					case "fr":
						return `La remise doit être ${expectedText}. Actuel : ${actualText}.`;
					case "ru":
						return `Скидка должна быть ${expectedText}. Сейчас: ${actualText}.`;
					default:
						return `Discount must be ${expectedText}. It is ${actualText}.`;
				}
			case "invoice_date":
				switch (locale) {
					case "es":
						return `La fecha de emisión debe ser ${expectedText}. Ahora es ${actualText}.`;
					case "de":
						return `Rechnungsdatum muss ${expectedText} sein. Aktuell: ${actualText}.`;
					case "it":
						return `La data di emissione deve essere ${expectedText}. Ora è ${actualText}.`;
					case "fr":
						return `La date d'émission doit être ${expectedText}. Actuel : ${actualText}.`;
					case "ru":
						return `Дата выставления должна быть ${expectedText}. Сейчас: ${actualText}.`;
					default:
						return `Invoice date must be ${expectedText}. It is ${actualText}.`;
				}
			case "due_date":
				switch (locale) {
					case "es":
						return `La fecha de vencimiento debe ser ${expectedText}. Ahora es ${actualText}.`;
					case "de":
						return `Fälligkeitsdatum muss ${expectedText} sein. Aktuell: ${actualText}.`;
					case "it":
						return `La data di scadenza deve essere ${expectedText}. Ora è ${actualText}.`;
					case "fr":
						return `La date d'échéance doit être ${expectedText}. Actuel : ${actualText}.`;
					case "ru":
						return `Срок оплаты должен быть ${expectedText}. Сейчас: ${actualText}.`;
					default:
						return `Due date must be ${expectedText}. It is ${actualText}.`;
				}
			default: {
				const fieldLabel = getFieldDisplayLabel(locale, e.field);
				switch (locale) {
					case "es":
						return `En ${fieldLabel} debe ser ${expectedText}. Ahora es ${actualText}.`;
					case "de":
						return `In ${fieldLabel} muss ${expectedText} sein. Aktuell: ${actualText}.`;
					case "it":
						return `In ${fieldLabel} deve essere ${expectedText}. Ora è ${actualText}.`;
					case "fr":
						return `Dans ${fieldLabel}, doit être ${expectedText}. Actuel : ${actualText}.`;
					case "ru":
						return `В ${fieldLabel} должно быть ${expectedText}. Сейчас: ${actualText}.`;
					default:
						return `In ${fieldLabel} must be ${expectedText}. It is ${actualText}.`;
				}
			}
		}
	})();

	return baseSentence;
}

export function formatReceiptMathValidationInlineError(
	locale: Locale,
	e: ReceiptMathValidationError
): string {
	const { expectedText, actualText } = formatExpectedActual(locale, e);
	const fieldLabel = getFieldDisplayLabel(locale, e.field);

	if (e.expected !== null && e.expected !== undefined && e.actual !== null && e.actual !== undefined) {
		return `${fieldLabel}. ${getStaticText(locale, "expectedLabel")}: ${expectedText}. ${getStaticText(locale, "gotLabel")}: ${actualText}.`;
	}

	return `${fieldLabel}. ${getStaticText(locale, "expectedLabel")}: ${expectedText}. ${getStaticText(locale, "gotLabel")}: ${actualText}.`;
}

export function getReceiptStaticText(
	locale: Locale,
	key: ReceiptStaticTextKey
): string {
	return getStaticText(locale, key);
}

const fieldLabelMap: Record<string, ReceiptStaticTextKey> = {
	billing_id: "billingIdLabel",
	shipping: "shippingLabel",
	payment_received: "paymentReceivedLabel",
	change_due: "changeDueLabel",
	due_date: "dueDateLabel",
	invoice_date: "issueDateLabel",
	invoice_number: "invoiceNumberLabel",
	order_number: "orderNumberLabel",
	nif_cif_ssn: "nifCifSsnLabel",
	last_4_card_digits: "last4CardDigitsLabel",
	issuer_name: "sellerLabel",
	payer_name: "buyerLabel",
	items: "itemsLabel",
	tax: "taxLabel",
};

export function getFieldDisplayLabel(locale: Locale, field: string): string {
	const key = fieldLabelMap[field];
	if (key) return getStaticText(locale, key);
	return field;
}

function normalizedSemanticMessage(locale: Locale, error: ReceiptMathValidationError): string {
	return formatReceiptMathValidationSuggestion(locale, error)
		.toLocaleLowerCase(locale)
		.replace(/\s+/g, " ")
		.trim();
}

export function dedupeReceiptValidationErrors(
	locale: Locale,
	errors: ReceiptMathValidationError[] | null | undefined
): ReceiptMathValidationError[] {
	const seen = new Set<string>();
	const result: ReceiptMathValidationError[] = [];

	for (const error of errors ?? []) {
		const structuralKey = JSON.stringify([
			error.field ?? "",
			error.rule ?? "",
			error.item_line ?? "",
			formatValidationValue(locale, error.expected),
			formatValidationValue(locale, error.actual),
		]);
		const semanticKey = JSON.stringify([
			error.field ?? "",
			error.item_line ?? "",
			normalizedSemanticMessage(locale, error),
		]);

		if (seen.has(structuralKey) || seen.has(semanticKey)) continue;

		seen.add(structuralKey);
		seen.add(semanticKey);
		result.push(error);
	}

	return result;
}
```

### `src/app/receipts/receipt-page-navigation.ts`
```ts
"use client";

export function reloadReceiptPage(): void {
	window.location.reload();
}
```

### `src/app/receipts/receipt-parse-memory.ts`
```ts
import { z } from "zod";

import {
	receiptMathValidationErrorSchema,
	receiptSchema,
} from "@/modules/receipt-parser/schemas";
import {
	receiptFormSchema,
	type ReceiptForm,
} from "@/modules/receipt-parser/receipt-form-schema";
import type { Receipt } from "@/modules/receipt-parser/schemas";

const RECEIPT_PARSE_MEMORY_VERSION = 1;
const RECEIPT_PARSE_MEMORY_KEY_PREFIX = "receipts:last-parse:";

const receiptParseMemorySchema = z.object({
	version: z.literal(RECEIPT_PARSE_MEMORY_VERSION),
	userRole: z.enum(["issuer", "payer"]),
	status: z.union([
		z.literal("verified"),
		z.literal("requires_human_correction"),
	]), 
	imagePreview: z.string().nullable(),
	receipt: receiptSchema,
	receiptForm: receiptFormSchema,
	receiptFormInitial: receiptFormSchema.nullable(),
	backendResult: z.object({
		status: z.union([
			z.literal("verified"),
			z.literal("requires_human_correction"),
		]),
		message: z.string(),
	}),
	validationErrors: z.array(receiptMathValidationErrorSchema).nullable(),
	lastImageHash: z.string().nullable(),
});

export type ReceiptParseMemory = {
	version: 1;
	userRole: "issuer" | "payer";
	status: "verified" | "requires_human_correction";
	imagePreview: string | null;
	receipt: Receipt;
	receiptForm: ReceiptForm;
	receiptFormInitial: ReceiptForm | null;
	backendResult: {
		status: "verified" | "requires_human_correction";
		message: string;
	};
	validationErrors: Array<z.infer<typeof receiptMathValidationErrorSchema>> | null;
	lastImageHash: string | null;
};

export function buildReceiptParseMemoryKey(userId: string): string {
	return `${RECEIPT_PARSE_MEMORY_KEY_PREFIX}${userId}`;
}

export function loadReceiptParseMemory(
	userId: string,
): ReceiptParseMemory | null {
	if (typeof window === "undefined") {
		return null;
	}

	try {
		const raw = window.localStorage.getItem(buildReceiptParseMemoryKey(userId));
		if (!raw) {
			return null;
		}
		const parsed = JSON.parse(raw);
		return receiptParseMemorySchema.parse(parsed);
	} catch {
		return null;
	}
}

export function saveReceiptParseMemory(
	userId: string,
	snapshot: ReceiptParseMemory,
): void {
	if (typeof window === "undefined") {
		return;
	}

	try {
		window.localStorage.setItem(
			buildReceiptParseMemoryKey(userId),
			JSON.stringify(snapshot),
		);
	} catch {

	}
}

export function clearReceiptParseMemory(userId: string): void {
	if (typeof window === "undefined") {
		return;
	}

	try {
		window.localStorage.removeItem(buildReceiptParseMemoryKey(userId));
	} catch {

	}
}

export function createReceiptParseMemorySnapshot(input: {
	userRole: "issuer" | "payer";
	status: "verified" | "requires_human_correction";
	imagePreview: string | null;
	receipt: Receipt;
	receiptForm: ReceiptForm;
	receiptFormInitial: ReceiptForm | null;
	backendResult: {
		status: "verified" | "requires_human_correction";
		message: string;
	};
	validationErrors: Array<z.infer<typeof receiptMathValidationErrorSchema>> | null;
	lastImageHash: string | null;
}): ReceiptParseMemory {
	return {
		version: RECEIPT_PARSE_MEMORY_VERSION,
		...input,
	};
}
```

### `src/app/receipts/receipt-pricing-api.ts`
```ts
import { apiRequest } from "@/lib/api-client";

export type FetchReceiptsPricingOptions = Record<string, never>;

export type ReceiptPricingTier = {
	id: string;
	max_receipts: number;
};

export type ReceiptPricingStripeLinks = {
	premium?: string;
	pro?: string;
	payg_topup?: string;
	pago?: string;
};

export type ReceiptsPricingResponse = {
	tiers: ReceiptPricingTier[];
	stripe_links: ReceiptPricingStripeLinks;
	/**
	 * Backend-provided path for the Stripe customer portal.
	 * E.g. `/billing/stripe/customer-portal` (relative to backend base).
	 * When present, use this instead of the hardcoded fallback.
	 */
	customer_portal_path?: string;
};

export async function fetchReceiptsPricing(): Promise<ReceiptsPricingResponse> {
	return apiRequest<ReceiptsPricingResponse>("/receipts/pricing", {
		method: "GET",
	});
}
```

### `src/app/receipts/receipt-usage-api.ts`
```ts
import { apiRequest } from "@/lib/api-client";

export type ReceiptsUsageResponse = {
	receipt_count: number;
	max_receipts: number;
	tier: string;
	at_80_percent: boolean;
};

export type FetchReceiptsUsageOptions = {
	userId?: string | null;
};

function resolveReceiptsUsageUserId(options: FetchReceiptsUsageOptions): string {
	const explicit = options.userId?.trim();
	if (explicit) {
		return explicit;
	}
	const configured =
		typeof process !== "undefined"
			? process.env.NEXT_PUBLIC_USER_ID?.trim()
			: undefined;
	return configured || "anonymous";
}

export async function fetchReceiptsUsage(
	options: FetchReceiptsUsageOptions = {},
): Promise<ReceiptsUsageResponse> {
	const params = new URLSearchParams({
		user_id: resolveReceiptsUsageUserId(options),
	});

	return apiRequest<ReceiptsUsageResponse>(`/receipts/usage?${params.toString()}`, {
		method: "GET",
		timeoutMs: 60_000,
	});
}
```

### `src/app/receipts/receipts-wallet-context.tsx`
```tsx
"use client";

import {
	createContext,
	useCallback,
	useContext,
	useEffect,
	useState,
} from "react";
import type { ReactNode } from "react";

import { useAuthMe } from "@/app/AuthProvider";
import { parseReceiptParserBalanceUsdDisplay } from "@/app/auth/backend-auth-api";

type ReceiptsWalletContextValue = {
	/** Immediate balance override from the last backend response (bypasses refreshMe delay). */
	paygBalanceOverride: string | null;
	setPaygBalanceOverride: (v: string | null) => void;
	/**
	 * When the backend fell back to the tier base model due to insufficient balance,
	 * this holds that model id so the selector reflects it immediately.
	 */
	fallbackModelId: string | null;
	/** Setting a non-null value also shows the fallback notice banner. */
	setFallbackModelId: (v: string | null) => void;
	showFallbackNotice: boolean;
	dismissFallbackNotice: () => void;
};

const ReceiptsWalletContext = createContext<ReceiptsWalletContextValue | null>(
	null,
);

export function ReceiptsWalletProvider({ children }: { children: ReactNode }) {
	const me = useAuthMe();
	const [paygBalanceOverride, setPaygBalanceOverride] = useState<string | null>(
		null,
	);
	const [fallbackModelId, setFallbackModelIdRaw] = useState<string | null>(
		null,
	);
	const [showFallbackNotice, setShowFallbackNotice] = useState(false);

	const setFallbackModelId = useCallback((id: string | null) => {
		setFallbackModelIdRaw(id);
		setShowFallbackNotice(id != null);
	}, []);

	const dismissFallbackNotice = useCallback(() => {
		setShowFallbackNotice(false);
	}, []);

	useEffect(() => {
		if (me != null) {
			return;
		}
		setPaygBalanceOverride(null);
		setFallbackModelIdRaw(null);
		setShowFallbackNotice(false);
	}, [me]);

	useEffect(() => {
		if (!fallbackModelId || !paygBalanceOverride) {
			return;
		}

		const serverBalance = parseReceiptParserBalanceUsdDisplay(
			me?.balance_usd ?? me?.receipt_parser_balance_usd_display,
		);
		const nextBalance = serverBalance == null ? Number.NaN : Number(serverBalance);
		const currentBalance = Number(paygBalanceOverride);

		if (
			Number.isFinite(nextBalance) &&
			Number.isFinite(currentBalance) &&
			nextBalance > currentBalance
		) {
			setPaygBalanceOverride(null);
			setFallbackModelIdRaw(null);
			setShowFallbackNotice(false);
		}
	}, [
		fallbackModelId,
		me?.balance_usd,
		me?.receipt_parser_balance_usd_display,
		paygBalanceOverride,
	]);

	return (
		<ReceiptsWalletContext.Provider
			value={{
				paygBalanceOverride,
				setPaygBalanceOverride,
				fallbackModelId,
				setFallbackModelId,
				showFallbackNotice,
				dismissFallbackNotice,
			}}
		>
			{children}
		</ReceiptsWalletContext.Provider>
	);
}

export function useReceiptsWallet(): ReceiptsWalletContextValue {
	const ctx = useContext(ReceiptsWalletContext);
	if (!ctx) {
		throw new Error(
			"useReceiptsWallet must be used within ReceiptsWalletProvider",
		);
	}
	return ctx;
}
```

### `src/app/receipts/ReceiptsModuleChrome.tsx`
```tsx
"use client";

import type { ReactNode } from "react";
import { useEffect, useState } from "react";
import Link from "next/link";

import { AuthProvider, useAuthMe, useSupabaseAuth } from "@/app/AuthProvider";
import { AuthHeader } from "@/app/AuthHeader";
import { LanguageSwitcher } from "@/app/LanguageSwitcher";
import { NavTierIndicator } from "@/app/billing/NavTierIndicator";
import { useI18n } from "@/app/i18n-provider";

import { usePathname } from "next/navigation";

import { ModuleFooter } from "@/components/ModuleFooter";
import { SupportTicketPanel } from "@/components/SupportTicketPanel";
import { ReceiptsParseModelRow } from "@/app/receipts/components/ReceiptsParseModelRow";
import { ReceiptsWalletProvider } from "@/app/receipts/receipts-wallet-context";

/**
 * Receipts-only auth context and top bar. Other modules do not share this
 * session or UI (see root layout).
 */
export function ReceiptsModuleChrome({ children }: { children: ReactNode }) {
	const pathname = usePathname();
	const [localeGateReady, setLocaleGateReady] = useState(
		process.env.NODE_ENV === "test",
	);

	useEffect(() => {
		setLocaleGateReady(true);
	}, []);

	if (!localeGateReady) {
		return null;
	}

	return (
		<AuthProvider module="receipt_parser">
			<ReceiptsWalletProvider>
				<ReceiptsModuleChromeInner pathname={pathname}>
					{children}
				</ReceiptsModuleChromeInner>
			</ReceiptsWalletProvider>
		</AuthProvider>
	);
}

function ReceiptsModuleChromeInner({
	pathname,
	children,
}: {
	pathname: string;
	children: ReactNode;
}) {
	const { t } = useI18n();
	const me = useAuthMe();
	const { isLoading: authLoading, me: authMe } = useSupabaseAuth();
	const [navTierReady, setNavTierReady] = useState(false);

	const contentReady = !authLoading && (authMe == null || navTierReady);

	const tier = (me?.tier ?? "").trim().toLowerCase();
	const canOpenSupport =
		tier === "premium" || tier === "pro" || tier === "payg" || tier === "admin";
	const [supportOpen, setSupportOpen] = useState(false);

	const isActive = (href: string): boolean => {
		if (href === "/receipts") return pathname === "/receipts";
		return pathname?.startsWith(href) ?? false;
	};

	const navLinkClass = (href: string): string => {
		const active = isActive(href);
		return active
			? "rounded-full bg-slate-900 px-3 py-1 text-xs font-semibold text-white animate-pulse cursor-default"
			: "rounded-full px-3 py-1 text-xs font-medium text-slate-600 hover:bg-slate-100 hover:text-slate-950 transition-colors";
	};

	const renderNavItem = (href: string, label: string) => {
		if (isActive(href)) {
			return (
				<span aria-current="page" className={navLinkClass(href)}>
					{label}
				</span>
			);
		}

		return (
			<Link href={href} className={navLinkClass(href)}>
				{label}
			</Link>
		);
	};

	return (
		<div className="flex min-h-screen flex-col">
			<div className="border-b border-slate-200 bg-slate-50">
				<div className="mx-auto flex max-w-6xl flex-wrap items-center justify-between gap-2 px-4 py-2">
					<div className="flex flex-wrap items-center gap-x-4 gap-y-1">
						<span className="text-2xl font-semibold text-slate-900">
							Vera
						</span>
						<nav className="flex flex-wrap items-center gap-x-4 gap-y-1 text-xs font-medium">
							{renderNavItem("/receipts", t("receipts.module.navLabel"))}
							{me ? (
								renderNavItem("/receipts/history", t("receipts.module.navHistory"))
							) : null}
						</nav>
					</div>
					<div className="flex flex-wrap items-center justify-end gap-3">
						{me ? <NavTierIndicator onReady={() => setNavTierReady(true)} /> : null}
						<AuthHeader transactionsHref="/billing/transactions?source=receipt_parser" />
						{renderNavItem("/receipts/pricing", t("receipts.module.navPlans"))}
						<LanguageSwitcher />
					</div>
				</div>
				<ReceiptsParseModelRow />
			</div>
			<div className="flex-1">{contentReady ? children : null}</div>
			<ModuleFooter
				moduleLabel="receipt_parser"
				canOpenSupport={canOpenSupport}
				onOpenTickets={() => setSupportOpen(true)}
				blockedSupportLabel={t("receipts.module.navPlans")}
			/>
			<SupportTicketPanel
				key="receipt_parser"
				module="receipt_parser"
				isOpen={supportOpen}
				onClose={() => setSupportOpen(false)}
				isAllowedTier={canOpenSupport}
				blockedMessage={t("support.tickets.upgradePlan")}
			/>
		</div>
	);
}
```

### `src/app/receipts/use-receipt-offline-sync.ts`
```ts
"use client";

import { useEffect, useRef } from "react";
import type { UseFormReturn } from "react-hook-form";

import {
	handleAuthApiErrorSideEffects,
	resolveAuthApiUserMessage,
} from "@/app/auth/backend-auth-error";
import { ApiError } from "@/lib/api-client";
import { ModuleAuthRequiredError } from "@/lib/module-auth-required-error";
import { sha256HexFromBase64 } from "@/lib/image-hash";
import type { ReceiptForm } from "@/modules/receipt-parser/receipt-form-schema";
import { receiptQueueDb } from "@/modules/receipt-parser/offline-queue";
import type { Receipt } from "@/modules/receipt-parser/schemas";
import type { ReceiptMathValidationError } from "@/modules/receipt-parser/schemas";

import {
	ReceiptProcessError,
	ReceiptProcessSchemaMismatchError,
	ReceiptProcessValidationError,
	isReceiptProcessTimeoutError,
	ReceiptProcessServerError,
	RECEIPT_PROCESS_TIMEOUT_MS,
	scheduleReceiptsUsageInvalidation,
	submitReceiptToBackend,
} from "./receipt-api";
import { receiptProcessErrorI18nKey } from "./receipt-error-utils";
import { isAbortError } from "@/lib/api-client";

export type ReceiptPageStatus =
	| "idle"
	| "capturing"
	| "uploading_image"
	| "waiting_backend"
	| "verified"
	| "unprocessable"
	| "requires_human_correction"
	| "error";

export type ReceiptBackendResult = {
	status: "verified" | "unprocessable" | "requires_human_correction";
	message: string;
};

export type UseReceiptOfflineSyncParams = {
	isOnline: boolean;
	userId: string | null;
	isAuthReady: boolean;
	suspendSync?: boolean;
	userRole: "issuer" | "payer";
	signOut: () => void | Promise<void>;
	translate: (key: string) => string;
	receiptForm: UseFormReturn<ReceiptForm>;
	setStatus: React.Dispatch<React.SetStateAction<ReceiptPageStatus>>;
	setReceipt: React.Dispatch<React.SetStateAction<Receipt | null>>;
	setReceiptFormInitial: React.Dispatch<
		React.SetStateAction<ReceiptForm | null>
	>;
	setValidationErrors: React.Dispatch<
		React.SetStateAction<ReceiptMathValidationError[] | null>
	>;
	setLastImageHash: React.Dispatch<React.SetStateAction<string | null>>;
	setBackendResult: React.Dispatch<
		React.SetStateAction<ReceiptBackendResult | null>
	>;
	setError: React.Dispatch<React.SetStateAction<string | null>>;
};

/**
 * Drains the Dexie offline queue when online. Does not touch status when there
 * are no pending rows (avoids overwriting a normal online flow or verified UI).
 */
export function useReceiptOfflineSync(params: UseReceiptOfflineSyncParams): void {
	const {
		isOnline,
		userId,
		isAuthReady,
		suspendSync = false,
		userRole,
		signOut,
		translate,
		receiptForm,
		setStatus,
		setReceipt,
		setReceiptFormInitial,
		setValidationErrors,
		setLastImageHash,
		setBackendResult,
		setError,
	} = params;

	const translateRef = useRef(translate);
	translateRef.current = translate;
	const userRoleRef = useRef(userRole);
	userRoleRef.current = userRole;
	const userIdRef = useRef(userId);
	userIdRef.current = userId;
	const isAuthReadyRef = useRef(isAuthReady);
	isAuthReadyRef.current = isAuthReady;
	const signOutRef = useRef(signOut);
	signOutRef.current = signOut;
	const receiptFormRef = useRef(receiptForm);
	receiptFormRef.current = receiptForm;

	useEffect(() => {
		if (suspendSync) {
			return;
		}
		if (!isOnline) {
			return;
		}

		let cancelled = false;

		const t = (key: string): string => translateRef.current(key);

		const syncOfflineQueue = async (): Promise<void> => {
			if (!isAuthReadyRef.current) {
				return;
			}

			const allRecords = await receiptQueueDb.receipts.toArray();
			const pending = allRecords.filter((record) => !record.synced);

			if (!pending.length) {
				return;
			}

			setStatus("waiting_backend");

			const dropRecord = async (id: string): Promise<void> => {
				try {
					await receiptQueueDb.receipts.delete(id);
				} catch (deleteError) {
					console.error(
						"[offline-sync] failed to delete queue record",
						id,
						deleteError,
					);
				}
			};

			for (const record of pending) {
				if (cancelled) return;

				try {
					const imageHash = await sha256HexFromBase64(record.imageBase64).catch(
						() => undefined
					);
					if (imageHash) {
						setLastImageHash(imageHash);
					}
					const parsed = await submitReceiptToBackend(
						record.imageBase64,
						{
							user_role: userRoleRef.current,
							image_hash: imageHash,
							userId: userIdRef.current,
							document_mime_type: record.documentMimeType ?? "image/jpeg",
							document_filename:
								record.documentFilename ?? "receipt.jpg",
							timeoutMs: RECEIPT_PROCESS_TIMEOUT_MS,
						},
					);

					await dropRecord(record.id);

					if (cancelled) return;

					if (parsed.receipt) setReceipt(parsed.receipt);

					if (parsed.status === "verified") {
						setStatus("verified");
						setError(null);
						setBackendResult({
							status: "verified",
							message: t("receipts.backendResult.offlineSynced"),
						});
						setValidationErrors(null);
						if (parsed.receipt) {
							setReceiptFormInitial(parsed.receipt);
							receiptFormRef.current.reset(parsed.receipt);
						}
					} else if (parsed.status === "requires_human_correction") {
						setStatus("requires_human_correction");
						setBackendResult({
							status: "requires_human_correction",
							message:
								parsed.error_message?.trim() ||
								t("receipts.status.requiresHumanCorrection"),
						});
						setValidationErrors(parsed.validation_errors ?? null);
						setError(null);
						const bestEffort = parsed.best_effort_receipt ?? parsed.receipt;
						if (bestEffort) {
							const normalizedBestEffort = bestEffort as Receipt;
							setReceipt(normalizedBestEffort);
							setReceiptFormInitial(normalizedBestEffort);
							receiptFormRef.current.reset(normalizedBestEffort);
						} else {
							setReceipt(null);
							setReceiptFormInitial(null);
						}
						break;
					} else {
						setStatus("unprocessable");
						setBackendResult({
							status: "unprocessable",
							message:
								parsed.error_message?.trim() ||
								t("receipts.backendResult.offlineUnprocessable"),
						});
						setValidationErrors(null);
						setError(null);
					}
				} catch (syncError) {

					if (isAbortError(syncError)) {
						setStatus("error");
						setError(t("receipts.error.timeout"));
						scheduleReceiptsUsageInvalidation();
						break;
					}

					handleAuthApiErrorSideEffects(syncError, () => {
						void signOutRef.current();
					});
					const authMsg = resolveAuthApiUserMessage(syncError, t);
					if (authMsg) {
						setStatus("error");
						setError(authMsg);
						break;
					}
					if (syncError instanceof ModuleAuthRequiredError) {
						setStatus("error");
						setError(t("receipts.main.error.signInRequired"));
						break;
					}

					if (isReceiptProcessTimeoutError(syncError)) {
						await dropRecord(record.id);
						setStatus("error");
						setError(t("receipts.error.timeout"));
						scheduleReceiptsUsageInvalidation();
						break;
					}
					if (syncError instanceof ReceiptProcessServerError) {
						await dropRecord(record.id);
						setStatus("error");
						setError(t("receipts.error.serverError"));
						break;
					}
					if (syncError instanceof ApiError) {
						await dropRecord(record.id);
						setStatus("error");
						setError(t("receipts.error.imageProcessing"));
						break;
					}
					if (syncError instanceof ReceiptProcessSchemaMismatchError) {
						await dropRecord(record.id);
						setStatus("error");
						setError(t("receipts.error.schemaMismatch"));
						break;
					}
					if (syncError instanceof ReceiptProcessValidationError) {
						await dropRecord(record.id);
						setStatus("error");
						const detail = syncError.detail ?? "";
						const isTooLarge =
							detail.includes("image_base64") && detail.includes("at most");
						setError(
							isTooLarge
								? t("receipts.error.imageTooLarge")
								: t("receipts.error.imageProcessing")
						);
						break;
					}
					if (syncError instanceof ReceiptProcessError) {
						await dropRecord(record.id);
						setStatus("error");
						if (syncError.status === 409) {
							setError(t("receipts.error.duplicate"));
						} else {
							setError(t(receiptProcessErrorI18nKey(syncError.detail)));
						}
						break;
					}

					console.error(syncError);
					setStatus("error");
					setError(t("receipts.error.syncOffline"));
					break;
				}
			}
		};

		void syncOfflineQueue();

		return () => {
			cancelled = true;
		};
	}, [
		isOnline,
		userId,
		isAuthReady,
		suspendSync,
		userRole,
		signOut,
		setStatus,
		setReceipt,
		setReceiptFormInitial,
		setValidationErrors,
		setLastImageHash,
		setBackendResult,
		setError,
	]);
}
```

### `src/app/receipts/components/CapacityBanner.tsx`
```tsx
"use client";

import Link from "next/link";
import { useI18n } from "@/app/i18n-provider";

export type ReceiptsUsage = {
	at_80_percent: boolean;
	receipt_count: number;
	max_receipts: number;
	tier: string;
};

type CapacityBannerProps = {
	usage: ReceiptsUsage | null;
	pricingHref?: string;
	upgradeHref?: string;
};

export function CapacityBanner({
	usage,
	pricingHref = "/receipts/pricing",
	upgradeHref,
}: CapacityBannerProps) {
	const { t } = useI18n();

	if (!usage?.at_80_percent) {
		return null;
	}

	const href = upgradeHref ?? pricingHref;

	return (
		<div className="mb-4 flex flex-wrap items-center justify-between gap-2 rounded-2xl border border-amber-300 bg-amber-50 px-4 py-3 text-xs text-amber-900 shadow-sm">
			<div className="flex flex-col gap-1">
				<span className="font-semibold">{t("receipts.capacity.title")}</span>
				<span>{t("receipts.capacity.subtitle")}</span>
			</div>
			<Link
				href={href}
				className="ml-4 shrink-0 rounded-full bg-amber-700 px-3 py-1.5 font-medium text-white shadow-sm transition hover:bg-amber-800"
			>
				{t("receipts.capacity.upgradeCta")}
			</Link>
		</div>
	);
}
```

### `src/app/receipts/components/ReceiptHitlForm.tsx`
```tsx
"use client";

import { useEffect, useMemo, useState } from "react";
import { Controller, useFieldArray, type UseFormReturn } from "react-hook-form";
import { DatePickerField } from "@/components/DatePickerField";

import type { ReceiptForm } from "@/modules/receipt-parser/receipt-form-schema";
import type { ReceiptMathValidationError } from "@/app/receipts/receipt-history-api";
import type { Locale } from "@/lib/i18n-config";
import {
	formatReceiptMathValidationInlineError,
	formatReceiptMathValidationSuggestion,
	getReceiptStaticText,
	getFieldDisplayLabel,
	dedupeReceiptValidationErrors,
	isAnonymizedPlaceholder,
} from "../receipt-math-validation-i18n";
import {
	parseDecimalStringToCents,
	computeSubtotalCents,
	computeDiscountPercentInputCents,
	formatCentsToDecimalString,
	divRoundHalfUp,
	deriveReceiptMath,
	type DiscountMode,
} from "../receipt-calc";
import { normalizeTaxRatePercentString } from "@/modules/receipt-parser/tax-rate";

type ReceiptHitlFormProps = {
	form: UseFormReturn<ReceiptForm>;
	t: (key: string, vars?: Record<string, string>) => string;
	locale: Locale;
	showCorrectionHint: boolean;
	correctionFallbackMessage?: string | null;
	onSave: (data: ReceiptForm) => Promise<void>;
	isSaving: boolean;
	saveSuccess: boolean;
	saveError: boolean;
	saveErrorMessage?: string | null;
	saveErrorShowLoginCta?: boolean;
	onSaveErrorLogin?: () => void;
	validationErrors?: ReceiptMathValidationError[] | null;
};

export function ReceiptHitlForm({
	form,
	t,
	locale,
	showCorrectionHint,
	correctionFallbackMessage,
	onSave,
	isSaving,
	saveSuccess,
	saveError,
	saveErrorMessage,
	saveErrorShowLoginCta,
	onSaveErrorLogin,
	validationErrors,
}: ReceiptHitlFormProps) {
	const {
		register,
		control,
		handleSubmit,
		watch,
		setValue,
		getValues,
		formState,
	} = form;

	const dueDateSchemaError: string | null = (() => {
		const msg = formState.errors.due_date?.message;
		if (!msg) return null;
		switch (locale) {
			case "en": return "Due date cannot be earlier than the issue date.";
			case "de": return "Das Fälligkeitsdatum kann nicht vor dem Ausstellungsdatum liegen.";
			case "it": return "La data di scadenza non può essere precedente alla data di emissione.";
			case "fr": return "La date d'échéance ne peut pas être antérieure à la date d'émission.";
			case "ru": return "Срок оплаты не может быть раньше даты выставления счёта.";
			case "es":
			default:   return "La fecha de vencimiento no puede ser anterior a la fecha de emisión.";
		}
	})();

	const backendValidationErrors = useMemo(
		() => dedupeReceiptValidationErrors(locale, validationErrors),
		[locale, validationErrors]
	);
	const humanReviewRecoveryNote = (() => {
		switch (locale) {
			case "es":
				return "Extracción recuperada automáticamente. Revisa los campos y cargos antes de guardar.";
			case "de":
				return "Automatisch wiederhergestellte Extraktion. Prüfe Felder und Zuschläge vor dem Speichern.";
			case "it":
				return "Estrazione recuperata automaticamente. Controlla campi e addebiti prima di salvare.";
			case "fr":
				return "Extraction récupérée automatiquement. Vérifie les champs et frais avant d'enregistrer.";
			case "ru":
				return "Извлечение было автоматически восстановлено. Проверьте поля и сборы перед сохранением.";
			case "en":
			default:
				return "This extraction was recovered automatically. Review fields and charges before saving.";
		}
	})();

	const KNOWN_FIELDS = new Set([
		"total",
		"subtotal",
		"discount",
		"invoice_date",
		"due_date",
		"items.total",
	]);

	const formatKnownErrorSuggestion = (
		e: ReceiptMathValidationError
	): string => {
		return formatReceiptMathValidationSuggestion(locale, e);
	};

	const getFocusNameForError = (
		e: ReceiptMathValidationError
	): string | null => {
		switch (e.field) {
			case "total":
				return "total";
			case "subtotal":
				return "subtotal";
			case "discount":
				return "discount";
			case "invoice_date":
				return "invoice_date";
			case "due_date":
				return "due_date";
			default: {
				if (
					e.field === "items.total" ||
					(typeof e.field === "string" && e.field.endsWith("items.total"))
				) {
					if (e.item_line == null) return null;
					const idx0 = Math.max(0, e.item_line - 1);
					return `items.${idx0}.total`;
				}
				return null;
			}
		}
	};

	const firstFocusableError = backendValidationErrors.find((e) => {
		return getFocusNameForError(e) != null;
	});

	const focusName =
		firstFocusableError != null
			? getFocusNameForError(firstFocusableError)
			: null;

	useEffect(() => {
		if (!showCorrectionHint) return;
		if (!focusName) return;

		const focusPath = focusName as Parameters<typeof form.setFocus>[0];
		form.setFocus(focusPath);
	}, [focusName, form, showCorrectionHint]);

	const formatMathError = (e: ReceiptMathValidationError): string => {
		return formatReceiptMathValidationInlineError(locale, e);
	};

	const errorsForField = (field: string): ReceiptMathValidationError[] => {
		return backendValidationErrors
			.filter((x) => x.field === field)
			.slice(0, 3);
	};

	const errorsForItemLineTotal = (
		itemLine1Based: number
	): ReceiptMathValidationError[] => {
		return backendValidationErrors
			.filter(
				(x) =>
					(x.field === "items.total" || x.field.endsWith("items.total")) &&
					x.item_line != null &&
					x.item_line === itemLine1Based
			)
			.slice(0, 3);
	};

	const unknownFieldErrors = backendValidationErrors.filter((e) => {
		const isKnownField = KNOWN_FIELDS.has(e.field);
		if (isKnownField) return false;
		if (
			e.field.endsWith("items.total") &&
			(e.field === "items.total" || e.field.endsWith("items.total"))
		) {
			return false;
		}
		return true;
	});

	const invoiceDateErrors = errorsForField("invoice_date");
	const dueDateErrors = errorsForField("due_date");
	const subtotalErrors = errorsForField("subtotal");
	const discountErrors = errorsForField("discount");
	const totalErrors = errorsForField("total");

	const renderAnonymizedHint = (
		value: string | null | undefined,
		testId?: string
	) => {
		if (!isAnonymizedPlaceholder(value)) return null;
		return (
			<p className="text-[10px] text-amber-700" data-testid={testId}>
				{getReceiptStaticText(locale, "anonymizedValueLabel")}: {value}
			</p>
		);
	};

	const discountRegister = register("discount");

	const { fields, append, remove } = useFieldArray({
		control,
		name: "items",
	});

	const { fields: taxFields, append: appendTax, remove: removeTax } = useFieldArray({
		control,
		name: "taxes",
	});

	const initialDiscountPercentInput = useMemo(() => {
		const initialValues = getValues();
		const subCents = parseDecimalStringToCents(initialValues.subtotal) ?? 0n;
		const discCents = parseDecimalStringToCents(initialValues.discount) ?? 0n;
		const pctCents = computeDiscountPercentInputCents({ subtotalCents: subCents, discountCents: discCents });
		return formatCentsToDecimalString(pctCents);
	}, [getValues]);

	const [discountMode, setDiscountMode] = useState<DiscountMode>("amount");
	const [discountPercentInput, setDiscountPercentInput] = useState<string>(
		initialDiscountPercentInput
	);

	const watchedSubtotal = watch("subtotal");
	const watchedDiscount = watch("discount");

	useEffect(() => {
		if (discountMode === "percent") {
			const subCents = parseDecimalStringToCents(watchedSubtotal) ?? 0n;
			const pctCents = parseDecimalStringToCents(discountPercentInput) ?? 0n;
			const safePct = pctCents < 0n ? 0n : pctCents;
			const discCents = divRoundHalfUp(subCents * safePct, 10000n);
			setValue("discount", formatCentsToDecimalString(discCents), { shouldValidate: true });
		}
	}, [discountMode, watchedSubtotal, discountPercentInput, setValue]);

	useEffect(() => {
		if (discountMode === "amount") {
			const subCents = parseDecimalStringToCents(watchedSubtotal) ?? 0n;
			const discCents = parseDecimalStringToCents(watchedDiscount) ?? 0n;
			const pctCents = computeDiscountPercentInputCents({ subtotalCents: subCents, discountCents: discCents });
			setDiscountPercentInput(formatCentsToDecimalString(pctCents));
		}
	}, [discountMode, watchedDiscount, watchedSubtotal]);

	const handleRecalculate = () => {
		const currentValues = getValues();

		const subtotalCents = computeSubtotalCents(currentValues.items.map(i => i.total));
		const newSubtotal = formatCentsToDecimalString(subtotalCents);
		setValue("subtotal", newSubtotal, { shouldValidate: true });

		const result = deriveReceiptMath({
			itemTotals: currentValues.items.map((item) => item.total),
			discountMode: "amount",
			discountAmount: currentValues.discount,
			discountPercentInput: "0",
			shouldAutoTax: currentValues.taxes.length > 0,
			taxRates: currentValues.taxes.map((tax) => tax.tax_rate ?? "0.00"),
			existingTax: "0",
			shippingAmount: currentValues.shipping ?? "0",
		});
		result.taxes?.forEach((line, idx) => {
			setValue(`taxes.${idx}.base_amount`, line.base_amount, { shouldValidate: true });

			setValue(`taxes.${idx}.tax_amount`, line.tax_amount, {
				shouldValidate: true,
			});
			setValue(`taxes.${idx}.tax_rate`, line.tax_rate, {
				shouldValidate: true,
			});

		});
		setValue("tax", result.tax, { shouldValidate: true });

		setValue("total", result.total, { shouldValidate: true });
	};

	const onSubmitWrapper = (data: ReceiptForm) => {
		if (data.taxes && data.taxes.length > 0) {
			const sumTaxes = data.taxes.reduce((acc, t) => acc + (parseDecimalStringToCents(t.tax_amount) ?? 0n), 0n);
			data.tax = formatCentsToDecimalString(sumTaxes);
		} else {
			data.tax = "0.00";
		}
		return onSave(data);
	};

	return (
		<div className="mt-4">
			<h3 className="text-xs font-semibold uppercase text-slate-600">
				{t("receipts.main.hitl.title")}
			</h3>
			{showCorrectionHint && (
				<div className="mt-1 rounded border border-amber-200 bg-amber-50 p-2">
					<p className="text-[11px] font-semibold text-amber-900">
						{getReceiptStaticText(locale, "correctionRequiredHeading")}
					</p>
					<p className="mt-1 text-[11px] text-amber-800">
						{humanReviewRecoveryNote}
					</p>
					<div className="mt-1 space-y-1">
						{backendValidationErrors.length > 0 ? (
							backendValidationErrors.slice(0, 3).map((e, idx) => (
								<p
									key={`${e.field}-${e.item_line ?? "x"}-${e.rule}-${idx}`}
									className="text-[11px] text-amber-800"
									data-testid="receipt-validation-warning"
								>
									{formatKnownErrorSuggestion(e)}
								</p>
							))
						) : (
							<p className="text-[11px] text-amber-800">
								{correctionFallbackMessage ??
									t("receipts.main.hitl.correctionMessage")}
							</p>
						)}
					</div>
				</div>
			)}
			{unknownFieldErrors.length > 0 && (
				<div className="mt-2 rounded border border-rose-200 bg-rose-50 p-2">
					<p className="text-[11px] font-medium text-rose-700">
						{getReceiptStaticText(locale, "validationErrorsTitle")}
					</p>
					{unknownFieldErrors.slice(0, 3).map((e, idx) => (
						<p key={`${e.field}-${idx}`} className="text-[10px] text-rose-700">
							{getReceiptStaticText(locale, "fieldLabel")}: {getFieldDisplayLabel(locale, e.field)}.{" "}
							{formatMathError(e)}
						</p>
					))}
				</div>
			)}
			<form
				className="mt-2 space-y-3 text-xs"
				onSubmit={handleSubmit(onSubmitWrapper)}
			>
				<input type="hidden" {...register("id")} />
				<input type="hidden" {...register("version")} />
				<input type="hidden" {...register("created_at")} />
				<input type="hidden" {...register("updated_at")} />
				<input type="hidden" {...register("source")} />
				<input type="hidden" {...register("type")} />
				<input type="hidden" {...register("user_role")} />

				<div className="grid grid-cols-2 gap-2">
					<div className="flex flex-col gap-1">
						<span className="text-[11px] text-slate-600">
							{t("receipts.main.hitl.issueDate")}
						</span>
						<Controller
							name="invoice_date"
							control={control}
							render={({ field }) => (
								<DatePickerField
									value={field.value}
									onChange={field.onChange}
									hasError={invoiceDateErrors.length > 0}
									placeholder={getReceiptStaticText(locale, "issueDatePlaceholder")}
									ariaLabel={t("receipts.main.hitl.issueDate")}
									locale={locale}
								/>
							)}
						/>
						{invoiceDateErrors.length > 0 &&
							invoiceDateErrors.map((e, idx) => (
								<p key={`${e.field}-${idx}`} className="text-[10px] text-rose-700">
									{formatMathError(e)}
								</p>
							))}
					</div>
					<div className="flex flex-col gap-1">
						<span className="text-[11px] text-slate-600">
							{getReceiptStaticText(locale, "dueDateLabel")}
						</span>
						<Controller
							name="due_date"
							control={control}
							render={({ field }) => (
								<DatePickerField
									value={field.value}
									onChange={field.onChange}
									hasError={dueDateErrors.length > 0 || dueDateSchemaError != null}
									placeholder={getReceiptStaticText(locale, "dueDatePlaceholder")}
									ariaLabel={getReceiptStaticText(locale, "dueDateLabel")}
									allowClear
									clearLabel={
										locale === "en" ? "Clear date" :
										locale === "de" ? "Datum löschen" :
										locale === "it" ? "Cancella data" :
										locale === "fr" ? "Effacer la date" :
										locale === "ru" ? "Очистить дату" :
										"Borrar fecha"
									}
									locale={locale}
								/>
							)}
						/>
						{dueDateSchemaError != null && (
							<p className="text-[10px] text-rose-700">{dueDateSchemaError}</p>
						)}
						{dueDateErrors.length > 0 &&
							dueDateErrors.map((e, idx) => (
								<p key={`${e.field}-${idx}`} className="text-[10px] text-rose-700">
									{formatMathError(e)}
								</p>
							))}
					</div>
				</div>

				<div className="grid grid-cols-2 gap-2">
					<label className="flex flex-col gap-1">
						<span className="text-[11px] text-slate-600">
							{getReceiptStaticText(locale, "invoiceNumberLabel")}
						</span>
						<input
							{...register("invoice_number")}
							className="rounded border border-slate-300 px-2 py-1"
							type="text"
						/>
					</label>
					<label className="flex flex-col gap-1">
						<span className="text-[11px] text-slate-600">
							{getReceiptStaticText(locale, "billingIdLabel")}
						</span>
						<input
							{...register("billing_id")}
							className="rounded border border-slate-300 px-2 py-1"
							type="text"
						/>
					</label>
				</div>

				<div className="grid grid-cols-2 gap-2">
					<label className="flex flex-col gap-1">
						<span className="text-[11px] text-slate-600">
							{t("receipts.main.hitl.seller")}
						</span>
						<input
							{...register("issuer_name")}
							className="rounded border border-slate-300 px-2 py-1"
							type="text"
						/>
						{renderAnonymizedHint(watch("issuer_name"), "issuer-anonymized-hint")}
					</label>
					<label className="flex flex-col gap-1">
						<span className="text-[11px] text-slate-600">
							{t("receipts.main.hitl.buyer")}
						</span>
						<input
							{...register("payer_name")}
							className="rounded border border-slate-300 px-2 py-1"
							type="text"
						/>
						{renderAnonymizedHint(watch("payer_name"), "payer-anonymized-hint")}
					</label>
				</div>

				<div className="grid grid-cols-2 gap-2">
					<label className="flex flex-col gap-1">
						<span className="text-[11px] text-slate-600">
							{getReceiptStaticText(locale, "orderNumberLabel")}
						</span>
						<input
							{...register("order_number")}
							className="rounded border border-slate-300 px-2 py-1"
							type="text"
						/>
						{renderAnonymizedHint(watch("order_number"))}
					</label>
					<label className="flex flex-col gap-1">
						<span className="text-[11px] text-slate-600">
							{getReceiptStaticText(locale, "nifCifSsnLabel")}
						</span>
						<input
							{...register("nif_cif_ssn")}
							className="rounded border border-slate-300 px-2 py-1"
							type="text"
						/>
						{renderAnonymizedHint(watch("nif_cif_ssn"))}
					</label>
				</div>

				<label className="flex flex-col gap-1">
					<span className="text-[11px] text-slate-600">
						{getReceiptStaticText(locale, "last4CardDigitsLabel")}
					</span>
						<input
							{...register("last_4_card_digits")}
							className="rounded border border-slate-300 px-2 py-1"
							type="text"
						/>
						{renderAnonymizedHint(watch("last_4_card_digits"))}
					</label>

				<div className="grid grid-cols-2 gap-2">
					<label className="flex flex-col gap-1">
						<span className="text-[11px] text-slate-600">
							{t("receipts.main.hitl.currency")}
						</span>
						<input
							{...register("currency")}
							className="rounded border border-slate-300 px-2 py-1 uppercase"
							type="text"
							maxLength={3}
						/>
					</label>

					<label className="flex flex-col gap-1">
						<span className="text-[11px] text-slate-600">
							{t("receipts.main.hitl.type")}
						</span>
						<select
							{...register("type", {
								onChange: (e) => {
									const nextType = e.target.value;
									if (nextType === "ISSUED") {
										setValue("user_role", "issuer", {
											shouldValidate: true,
										});
									} else if (nextType === "RECEIVED") {
										setValue("user_role", "payer", {
											shouldValidate: true,
										});
									}
								},
							})}
							className="rounded border border-slate-300 px-2 py-1"
						>
							<option value="ISSUED">
								{t("receipts.main.hitl.type.issued")}
							</option>
							<option value="RECEIVED">
								{t("receipts.main.hitl.type.received")}
							</option>
						</select>
					</label>
				</div>

				<div>
					<div className="mb-1 flex items-center justify-between">
						<span className="text-[11px] font-medium text-slate-700">
							{t("receipts.main.hitl.line.item")}
						</span>
						<button
							type="button"
							className="rounded-full bg-slate-200 px-2 py-0.5 text-[10px] font-medium text-slate-800"
							onClick={() =>
								append({
									description: "-",
									quantity: 1,
									unit_price: "0",
									total: "0",
									discount_amount: null,
									tax_rate: null,
								})
							}
						>
							{t("receipts.main.hitl.addLine")}
						</button>
					</div>
					<div className="space-y-2 rounded-lg border border-slate-200 bg-slate-50/80 p-2">
						{fields.map((field, index) => {
							const itemLine1Based = index + 1;
							const totalLineErrors = errorsForItemLineTotal(itemLine1Based);
							const hasTotalLineError = totalLineErrors.length > 0;

							return (
							<div
								key={field.id}
								className="grid gap-2 border-b border-slate-200 pb-2 last:border-0 last:pb-0 sm:grid-cols-[1fr_4rem_5rem_5rem_auto]"
							>
								<label className="flex flex-col gap-0.5 sm:col-span-1">
									<span className="text-[10px] text-slate-500">
										{t("receipts.main.hitl.line.description")}
									</span>
									<input
										{...register(`items.${index}.description`)}
										className="rounded border border-slate-300 px-1.5 py-1"
										type="text"
									/>
								</label>
								<label className="flex flex-col gap-0.5">
									<span className="text-[10px] text-slate-500">
										{t("receipts.main.hitl.line.qty")}
									</span>
									<input
										{...register(`items.${index}.quantity`, {
											valueAsNumber: true,
										})}
										className="rounded border border-slate-300 px-1.5 py-1"
										type="number"
										min={0}
										step="any"
									/>
								</label>
								<label className="flex flex-col gap-0.5">
									<span className="text-[10px] text-slate-500">
										{t("receipts.main.hitl.line.unit")}
									</span>
									<input
										{...register(`items.${index}.unit_price`)}
										className="rounded border border-slate-300 px-1.5 py-1"
										type="text"
									/>
								</label>
								<label className="flex flex-col gap-0.5">
									<span className="text-[10px] text-slate-500">
										{t("receipts.main.hitl.line.total")}
									</span>
									<input
										{...register(`items.${index}.total`)}
										className={
											hasTotalLineError
												? "rounded border border-rose-500 bg-rose-50/40 px-1.5 py-1"
												: "rounded border border-slate-300 px-1.5 py-1"
										}
										type="text"
									/>
									{hasTotalLineError &&
										totalLineErrors.map((e, idx) => (
											<p
												key={`${e.field}-${e.item_line ?? "x"}-${idx}`}
												className="text-[10px] text-rose-700"
											>
												{formatMathError(e)}
											</p>
										))}
								</label>
								<input type="hidden" {...register(`items.${index}.discount_amount`)} />
								<input type="hidden" {...register(`items.${index}.tax_rate`)} />
								<div className="flex items-end">
									<button
										type="button"
										className="w-full rounded border border-slate-300 px-1 py-1 text-[10px] text-slate-700 disabled:opacity-40"
										disabled={fields.length <= 1}
										onClick={() => remove(index)}
									>
										{t("receipts.main.hitl.removeLine")}
									</button>
								</div>
							</div>
							);
						})}
					</div>
				</div>

				<div className="grid grid-cols-2 gap-2">
					<label className="flex flex-col gap-1">
						<span className="text-[11px] text-slate-600">
							{t("receipts.main.hitl.subtotal")}
						</span>
						<input
							{...register("subtotal")}
							className={
								subtotalErrors.length > 0
									? "rounded border border-rose-500 bg-rose-50/40 px-2 py-1"
									: "rounded border border-slate-300 px-2 py-1"
							}
							type="text"
						/>
						{subtotalErrors.length > 0 &&
							subtotalErrors.map((e, idx) => (
								<p key={`${e.field}-${idx}`} className="text-[10px] text-rose-700">
									{formatMathError(e)}
								</p>
							))}
					</label>
					<div className="flex flex-col gap-1">
						<label className="flex flex-col gap-1">
							<span className="text-[11px] text-slate-600">
								{getReceiptStaticText(locale, "discountLabel")}
							</span>
							<input
								{...discountRegister}
								className={
									discountErrors.length > 0
										? "rounded border border-rose-500 bg-rose-50/40 px-2 py-1"
										: "rounded border border-slate-300 px-2 py-1"
								}
								readOnly={discountMode === "percent"}
								type="text"
								onChange={(e) => {
									const nextVal = e.target.value;
									setDiscountMode(nextVal.trim() === "" ? "percent" : "amount");
									discountRegister.onChange(e);
								}}
							/>
							{discountErrors.length > 0 &&
								discountErrors.map((e, idx) => (
									<p
										key={`${e.field}-${idx}`}
										className="text-[10px] text-rose-700"
									>
										{formatMathError(e)}
									</p>
								))}
						</label>

						<label className="flex flex-col gap-1">
							<span className="text-[11px] text-slate-600">
								{getReceiptStaticText(locale, "discountPercentLabel")}
							</span>
							<input
								value={discountPercentInput}
								className={
									discountErrors.length > 0
										? "rounded border border-rose-500 bg-rose-50/40 px-2 py-1"
										: "rounded border border-slate-300 px-2 py-1"
								}
								type="text"
								readOnly={discountMode === "amount"}
								onChange={(e) => {
									const nextVal = e.target.value;
									setDiscountMode(nextVal.trim() === "" ? "amount" : "percent");
									setDiscountPercentInput(nextVal);
								}}
							/>
						</label>
					</div>
				</div>

				{/* Campo de impuestos oculto, ya que se calcula automáticamente y se muestra en el desglose */}
				<input type="hidden" {...register("tax")} />

				<label className="flex flex-col gap-1">
					<span className="text-[11px] text-slate-600">
						{getReceiptStaticText(locale, "shippingLabel")}
					</span>
					<input
						{...register("shipping")}
						className="rounded border border-slate-300 px-2 py-1"
						type="text"
					/>
				</label>

				<div className="grid grid-cols-[1fr_auto] gap-2 items-end">
					<label className="flex flex-col gap-1">
						<span className="text-[11px] text-slate-600">
							{t("receipts.main.hitl.total")}
						</span>
						<input
							{...register("total")}
							className={
								totalErrors.length > 0
									? "rounded border border-rose-500 bg-rose-50/40 px-2 py-1"
									: "rounded border border-slate-300 px-2 py-1"
							}
							type="text"
						/>
						{totalErrors.length > 0 &&
							totalErrors.map((e, idx) => (
								<p key={`${e.field}-${idx}`} className="text-[10px] text-rose-700">
									{formatMathError(e)}
								</p>
							))}
					</label>
					<button
						type="button"
						onClick={handleRecalculate}
						className="rounded-lg border border-slate-300 bg-white px-4 py-1.5 text-xs font-medium text-slate-700 hover:bg-slate-50 h-[28px] mb-[1px]"
					>
						{getReceiptStaticText(locale, "recalculateLabel")}
					</button>
				</div>

				<div className="grid grid-cols-2 gap-2">
					<label className="flex flex-col gap-1">
						<span className="text-[11px] text-slate-600">
							{getReceiptStaticText(locale, "paymentReceivedLabel")}
						</span>
						<input
							{...register("payment_received")}
							className="rounded border border-slate-300 px-2 py-1"
							type="text"
						/>
					</label>
					<label className="flex flex-col gap-1">
						<span className="text-[11px] text-slate-600">
							{getReceiptStaticText(locale, "changeDueLabel")}
						</span>
						<input
							{...register("change_due")}
							className="rounded border border-slate-300 px-2 py-1"
							type="text"
						/>
					</label>
				</div>

				<div>
					<div className="mb-1 flex items-center justify-between">
						<span className="text-[11px] font-medium text-slate-700">
							{getReceiptStaticText(locale, "taxesBreakdownLabel")}
						</span>
						<button
							type="button"
							className="rounded-full bg-slate-200 px-2 py-0.5 text-[10px] font-medium text-slate-800"
							disabled={taxFields.length > 0}
							onClick={() =>
							appendTax({
									tax_rate: "0.00",
									base_amount: "0",
									tax_amount: "0",
								})
							}
						>
							{getReceiptStaticText(locale, "addTaxLabel")}
						</button>
					</div>

					<div className="space-y-2 rounded-lg border border-slate-200 bg-slate-50/80 p-2">
						{taxFields.map((field, index) => {
							const storedRate = watch(`taxes.${index}.tax_rate`) as
								| string
								| null
								| undefined;
							const displayRate =
								normalizeTaxRatePercentString(storedRate) ?? "0.00";

							return (
								<div
									key={field.id}
									className="grid gap-2 border-b border-slate-200 pb-2 last:border-0 last:pb-0 sm:grid-cols-[5rem_6rem_6rem_auto]"
								>
								<label className="flex flex-col gap-0.5">
									<span className="text-[10px] text-slate-500">
										{getReceiptStaticText(locale, "rateLabel")}
									</span>
									<input
										value={displayRate}
										onChange={(e) => {
											const nextPercent = e.target.value;
											setValue(
												`taxes.${index}.tax_rate`,
												nextPercent,
												{ shouldValidate: true }
											);
										}}
										className="rounded border border-slate-300 px-1.5 py-1"
										type="number"
										step="0.01"
									/>
								</label>
								<label className="flex flex-col gap-0.5">
									<span className="text-[10px] text-slate-500">
										{getReceiptStaticText(locale, "baseLabel")}
									</span>
									<input
										{...register(`taxes.${index}.base_amount`)}
										className="rounded border border-slate-300 px-1.5 py-1"
										type="text"
									/>
								</label>
								<label className="flex flex-col gap-0.5">
									<span className="text-[10px] text-slate-500">
										{getReceiptStaticText(locale, "taxAmountLabel")}
									</span>
									<input
										{...register(`taxes.${index}.tax_amount`)}
										className="rounded border border-slate-300 px-1.5 py-1"
										type="text"
									/>
								</label>
								<div className="flex items-end">
									<button
										type="button"
										className="w-full rounded border border-slate-300 px-1 py-1 text-[10px] text-slate-700 disabled:opacity-40"
										disabled={taxFields.length <= 0}
										onClick={() => removeTax(index)}
									>
										{getReceiptStaticText(locale, "removeTaxLabel")}
									</button>
								</div>
								</div>
							);
						})}

						{!taxFields.length && (
							<p className="text-[11px] text-slate-500">
								{getReceiptStaticText(locale, "noTaxesBreakdownLabel")}
							</p>
						)}
					</div>
				</div>

				<div className="flex flex-wrap items-center gap-2">
					<button
						type="submit"
						disabled={isSaving}
						className="rounded-full bg-slate-900 px-3 py-1.5 text-[11px] font-semibold text-white disabled:opacity-60"
					>
						{isSaving
							? t("receipts.main.hitl.saving")
							: t("receipts.main.hitl.saveButton")}
					</button>
					{saveSuccess && (
						<span className="text-[11px] font-medium text-emerald-700">
							{t("receipts.main.hitl.saveSuccess")}
						</span>
					)}
					{saveError && (
						<span className="text-[11px] font-medium text-rose-600">
							{t("receipts.main.hitl.saveError")}
						</span>
					)}
					{saveErrorMessage && (
						<span className="text-[11px] text-rose-600">
							{saveErrorMessage}
						</span>
					)}
					{saveError && saveErrorShowLoginCta && onSaveErrorLogin ? (
						<button
							type="button"
							className="rounded-lg bg-slate-900 px-3 py-1.5 text-[11px] font-medium text-white hover:bg-slate-800"
							onClick={() => {
								onSaveErrorLogin();
							}}
						>
							{t("auth.signInGoogle")}
						</button>
					) : null}
				</div>
			</form>
		</div>
	);
}
```

### `src/app/receipts/components/ReceiptsParseModelRow.tsx`
```tsx
"use client";

import { useCallback, useMemo, useState } from "react";

import { useSupabaseAuth } from "@/app/AuthProvider";
import {
	parseReceiptParserBalanceUsdDisplay,
	patchReceiptParserLlmModel,
} from "@/app/auth/backend-auth-api";
import { useI18n } from "@/app/i18n-provider";
import { reloadReceiptPage } from "@/app/receipts/receipt-page-navigation";
import { useReceiptsWallet } from "@/app/receipts/receipts-wallet-context";
import { ApiError } from "@/lib/api-client";

function hasModelPicker(
	options: string[] | null | undefined,
): options is string[] {
	return Array.isArray(options) && options.length > 0;
}

function normalizeModelId(value: string | null | undefined): string | null {
	const trimmed = value?.trim() ?? "";
	return trimmed === "" ? null : trimmed;
}

function sameModelId(a: string | null | undefined, b: string | null | undefined): boolean {
	const left = normalizeModelId(a);
	const right = normalizeModelId(b);
	if (!left || !right) {
		return false;
	}
	return left.toLowerCase() === right.toLowerCase();
}

function inferTierBaseModel(tier: string | undefined): string | null {
	const t = (tier ?? "").trim().toLowerCase();
	if (t === "premium") return "gemini/gemini-2.5-flash-lite";
	if (t === "pro") return "gemini/gemini-2.5-flash";
	return null;
}

export function ReceiptsParseModelRow() {
	const { t } = useI18n();
	const { me, isLoading, refreshMe } = useSupabaseAuth();
	const {
		paygBalanceOverride,
		fallbackModelId,
		showFallbackNotice,
		dismissFallbackNotice,
	} = useReceiptsWallet();
	const [patchError, setPatchError] = useState<string | null>(null);
	const [patching, setPatching] = useState(false);

	const paygBalanceAmount = useMemo(() => {
		if (me?.receipt_parser_payg !== true) {
			return null;
		}
		if (paygBalanceOverride != null) {
			return parseReceiptParserBalanceUsdDisplay(paygBalanceOverride);
		}
		if (me.balance_usd != null) {
			return parseReceiptParserBalanceUsdDisplay(me.balance_usd);
		}
		return parseReceiptParserBalanceUsdDisplay(
			me.receipt_parser_balance_usd_display,
		);
	}, [me, paygBalanceOverride]);

	const options = me?.receipt_parser_llm_model_options;
	const canShowPicker = hasModelPicker(options);
	const effectiveId = fallbackModelId ?? (me?.receipt_parser_llm_model_id ?? "");

	const tierBaseModelId = useMemo(
		() =>
			me?.receipt_parser_tier_base_model_id?.trim() ||
			inferTierBaseModel(me?.tier) ||
			null,
		[me],
	);

	const normalizedFallbackModelId = normalizeModelId(fallbackModelId);
	const fallbackMatchesTierBase = sameModelId(
		normalizedFallbackModelId,
		tierBaseModelId,
	);
	const fallbackLocksPicker = normalizedFallbackModelId != null;
	const showInteractivePicker = canShowPicker && !fallbackLocksPicker;
	const showLockedPicker =
		canShowPicker && fallbackLocksPicker && fallbackMatchesTierBase;

	const selectOptions = useMemo(() => {
		if (!options?.length) {
			return [];
		}
		const id = effectiveId.trim();
		const withEffective = id && !options.includes(id) ? [id, ...options] : [...options];
		return tierBaseModelId
			? withEffective.filter((option) => option !== tierBaseModelId)
			: withEffective;
	}, [effectiveId, options, tierBaseModelId]);

	const pipelineKind = me?.receipt_parser_parse_pipeline_kind;

	const fixedModelLabel = useMemo(() => {
		if (showInteractivePicker || showLockedPicker || !me) {
			return null;
		}
		const id = normalizeModelId(fallbackModelId) ?? me.receipt_parser_llm_model_id?.trim();
		const kind = pipelineKind?.trim();
		if (kind && !fallbackModelId) {
			const key = `receipts.model.pipeline.${kind}`;
			const translated = t(key);
			if (translated !== key) {
				return translated;
			}
		}
		if (id) {
			return t("receipts.model.fixedWithId", { id });
		}
		return t("receipts.model.fixedPlan");
	}, [
		fallbackModelId,
		me,
		pipelineKind,
		showInteractivePicker,
		showLockedPicker,
		t,
	]);

	const onModelChange = useCallback(
		async (nextId: string) => {
			if (!showInteractivePicker) {
				return;
			}
			const nextModelId = nextId.trim();
			if (nextModelId === effectiveId.trim()) {
				return;
			}
			setPatchError(null);
			setPatching(true);
			try {
				await patchReceiptParserLlmModel(nextModelId, { timeoutMs: 45_000 });
				await refreshMe();
				window.dispatchEvent(new CustomEvent("receipts-usage-invalidated"));
				if (me?.receipt_parser_payg === true) {
					reloadReceiptPage();
				}
			} catch (e) {
				if (e instanceof ApiError && e.status === 403) {
					setPatchError(t("receipts.wallet.modelPatch403"));
				} else {
					setPatchError(t("receipts.wallet.modelPatchFailed"));
				}
			} finally {
				setPatching(false);
			}
		},
		[effectiveId, me?.receipt_parser_payg, refreshMe, showInteractivePicker, t],
	);

	if (isLoading || !me) {
		return null;
	}

	const showPaygBalanceLine = paygBalanceAmount != null;
	const hasModelSection =
		showInteractivePicker ||
		showLockedPicker ||
		!!effectiveId.trim() ||
		!!pipelineKind?.trim();

	if (!showPaygBalanceLine && !hasModelSection) {
		return null;
	}

	return (
		<div className="border-t border-slate-100 bg-slate-50/80 px-4 py-2">
			<div className="mx-auto flex max-w-6xl flex-col gap-2 sm:flex-row sm:flex-wrap sm:items-center sm:justify-end sm:gap-4">
				{showPaygBalanceLine ? (
					<p className="text-[11px] font-medium text-slate-700">
						{t("receipts.wallet.balanceLabel", { amount: paygBalanceAmount })}
					</p>
				) : null}

				{showInteractivePicker ? (
					<label className="flex flex-wrap items-center gap-2 text-[11px] text-slate-700">
						<span className="font-medium text-slate-600">
							{t("receipts.wallet.modelLabel")}
						</span>
						<select
							className="max-w-[min(100%,18rem)] rounded-md border border-slate-300 bg-white px-2 py-1 text-[11px] text-slate-900 shadow-sm disabled:opacity-60"
							value={effectiveId}
							disabled={patching}
							onChange={(e) => {
								void onModelChange(e.target.value);
							}}
						>
							{tierBaseModelId ? (
								<option value={tierBaseModelId}>
									{t("receipts.wallet.tierBaseModelLabel", {
										model: tierBaseModelId,
									})}
								</option>
							) : null}
							{selectOptions.map((id) => (
								<option key={id} value={id}>
									{id}
								</option>
							))}
						</select>
					</label>
				) : showLockedPicker ? (
					<label className="flex flex-wrap items-center gap-2 text-[11px] text-slate-700">
						<span className="font-medium text-slate-600">
							{t("receipts.wallet.modelLabel")}
						</span>
						<select
							className="max-w-[min(100%,18rem)] rounded-md border border-slate-300 bg-slate-100 px-2 py-1 text-[11px] text-slate-900 shadow-sm opacity-80"
							value={effectiveId}
							disabled
						>
							<option value={effectiveId}>
								{t("receipts.wallet.tierBaseModelLabel", {
									model: effectiveId,
								})}
							</option>
						</select>
					</label>
				) : hasModelSection && fixedModelLabel ? (
					<p className="text-[11px] text-slate-600">
						<span className="font-medium text-slate-700">
							{t("receipts.wallet.modelLabel")}
						</span>{" "}
						{fixedModelLabel}
					</p>
				) : null}

				{showFallbackNotice ? (
					<p className="flex items-center gap-1 text-[11px] text-amber-700" role="status">
						{fallbackMatchesTierBase
							? t("receipts.wallet.modelFallbackNotice", {
									model: tierBaseModelId ?? effectiveId,
								})
							: t("receipts.wallet.modelFallbackDisabledNotice")}
						<button
							type="button"
							className="ml-1 font-bold leading-none text-amber-700 hover:text-amber-900"
							aria-label={t("receipts.wallet.dismissFallbackNotice")}
							onClick={dismissFallbackNotice}
						>
							x
						</button>
					</p>
				) : null}

				{patchError ? (
					<p className="text-[11px] text-rose-600" role="alert">
						{patchError}
					</p>
				) : null}
			</div>
		</div>
	);
}
```

### `src/app/receipts/cookies/page.tsx`
```tsx
"use client";

import { useI18n } from "@/app/i18n-provider";
import { LegalPageTemplate } from "@/components/LegalPageTemplate";
import type { Locale } from "@/lib/i18n-config";

type CookiePolicyContent = {
	title: string;
	description: string;
	lastUpdated: string;
	sections: { title: string; body: string }[];
};

const policies: Record<Locale, CookiePolicyContent> = {
	en: {
		title: "Cookie Policy (Receipt Parser)",
		description: "This policy explains how we use cookies and similar technologies (such as browser local storage) to recognize you when you visit our platform and ensure the security of your session.",
		lastUpdated: "Last updated: April 6, 2026",
		sections: [
			{
				title: "1. What are cookies?",
				body: "Cookies are small text files stored on your device when you visit a website. We use them to remember your preferences, keep your session securely logged in, and understand how you interact with the platform.",
			},
			{
				title: "2. Strictly necessary cookies",
				body: "Our platform relies on essential technical cookies to function. Specifically, we use secure cookies (HttpOnly, Secure, SameSite) to manage your authentication session (e.g., `ff_auth_refresh_*`). These cookies cannot be read by browser scripts, protecting you against XSS attacks. Without these cookies, you would not be able to log in or use the private modules.",
			},
			{
				title: "3. Local Storage (IndexedDB)",
				body: "In addition to cookies, we use your browser's local storage to improve your experience:\n\n- Language preferences: We save your language selection for future visits.\n- Offline Mode (PWA): We use IndexedDB to temporarily save images of your receipts if you lose your internet connection. This data is automatically synchronized with the server when you regain connection and is then deleted from your device.",
			},
			{
				title: "4. Telemetry and Performance",
				body: "We do not use third-party advertising tracking cookies. We collect performance and attention metrics anonymously to improve the user interface and detect errors on the platform, always respecting your privacy.",
			},
			{
				title: "5. Cookie Management",
				body: "Since we only use cookies that are strictly necessary for the security and operation of the service, we do not offer a panel to disable them, as doing so would break the application's functionality. If you do not want us to use these cookies, you must log out and configure your browser to block them, although this will prevent the use of the platform.",
			},
		],
	},
	es: {
		title: "Política de Cookies (Receipt Parser)",
		description: "Esta política explica cómo utilizamos las cookies y tecnologías similares (como el almacenamiento local del navegador) para reconocerte cuando visitas nuestra plataforma y garantizar la seguridad de tu sesión.",
		lastUpdated: "Última actualización: 6 de abril de 2026",
		sections: [
			{
				title: "1. ¿Qué son las cookies?",
				body: "Las cookies son pequeños archivos de texto que se almacenan en tu dispositivo cuando visitas un sitio web. Las utilizamos para recordar tus preferencias, mantener tu sesión iniciada de forma segura y entender cómo interactúas con la plataforma.",
			},
			{
				title: "2. Cookies estrictamente necesarias",
				body: "Nuestra plataforma depende de cookies técnicas esenciales para funcionar. Específicamente, utilizamos cookies seguras (HttpOnly, Secure, SameSite) para gestionar tu sesión de autenticación (ej. `ff_auth_refresh_*`). Estas cookies no pueden ser leídas por scripts del navegador, lo que te protege contra ataques XSS. Sin estas cookies, no podrías iniciar sesión ni utilizar los módulos privados.",
			},
			{
				title: "3. Almacenamiento Local (IndexedDB)",
				body: "Además de las cookies, utilizamos el almacenamiento local de tu navegador para mejorar tu experiencia:\n\n- Preferencias de idioma: Guardamos tu selección de idioma para futuras visitas.\n- Modo Offline (PWA): Utilizamos IndexedDB para guardar temporalmente las imágenes de tus facturas si pierdes la conexión a internet. Estos datos se sincronizan automáticamente con el servidor cuando recuperas la conexión y luego se eliminan de tu dispositivo.",
			},
			{
				title: "4. Telemetría y Rendimiento",
				body: "No utilizamos cookies de rastreo publicitario de terceros. Recopilamos métricas de rendimiento y atención de forma anónima para mejorar la interfaz de usuario y detectar errores en la plataforma, respetando siempre tu privacidad.",
			},
			{
				title: "5. Gestión de Cookies",
				body: "Dado que solo utilizamos cookies estrictamente necesarias para la seguridad y el funcionamiento del servicio, no ofrecemos un panel para deshabilitarlas, ya que rompería la funcionalidad de la aplicación. Si no deseas que utilicemos estas cookies, debes cerrar sesión y configurar tu navegador para bloquearlas, aunque esto impedirá el uso de la plataforma.",
			},
		],
	},
	de: {
		title: "Cookie-Richtlinie (Beleg-Parser)",
		description: "Diese Richtlinie erklärt, wie wir Cookies und ähnliche Technologien (wie den lokalen Speicher des Browsers) verwenden, um Sie bei Ihrem Besuch auf unserer Plattform zu erkennen und die Sicherheit Ihrer Sitzung zu gewährleisten.",
		lastUpdated: "Zuletzt aktualisiert: 6. April 2026",
		sections: [
			{
				title: "1. Was sind Cookies?",
				body: "Cookies sind kleine Textdateien, die auf Ihrem Gerät gespeichert werden, wenn Sie eine Website besuchen. Wir verwenden sie, um sich an Ihre Einstellungen zu erinnern, Ihre Sitzung sicher angemeldet zu halten und zu verstehen, wie Sie mit der Plattform interagieren.",
			},
			{
				title: "2. Unbedingt erforderliche Cookies",
				body: "Unsere Plattform ist auf wesentliche technische Cookies angewiesen, um zu funktionieren. Insbesondere verwenden wir sichere Cookies (HttpOnly, Secure, SameSite), um Ihre Authentifizierungssitzung zu verwalten (z. B. `ff_auth_refresh_*`). Diese Cookies können nicht von Browser-Skripten gelesen werden, was Sie vor XSS-Angriffen schützt. Ohne diese Cookies könnten Sie sich nicht anmelden oder die privaten Module nutzen.",
			},
			{
				title: "3. Lokale Speicherung (IndexedDB)",
				body: "Zusätzlich zu Cookies verwenden wir den lokalen Speicher Ihres Browsers, um Ihre Erfahrung zu verbessern:\n\n- Spracheinstellungen: Wir speichern Ihre Sprachauswahl für zukünftige Besuche.\n- Offline-Modus (PWA): Wir verwenden IndexedDB, um Bilder Ihrer Rechnungen vorübergehend zu speichern, falls Sie Ihre Internetverbindung verlieren. Diese Daten werden automatisch mit dem Server synchronisiert, wenn Sie wieder eine Verbindung haben, und dann von Ihrem Gerät gelöscht.",
			},
			{
				title: "4. Telemetrie und Leistung",
				body: "Wir verwenden keine Werbe-Tracking-Cookies von Drittanbietern. Wir erfassen Leistungs- und Aufmerksamkeitsmetriken anonym, um die Benutzeroberfläche zu verbessern und Fehler auf der Plattform zu erkennen, wobei wir stets Ihre Privatsphäre respektieren.",
			},
			{
				title: "5. Cookie-Verwaltung",
				body: "Da wir nur Cookies verwenden, die für die Sicherheit und den Betrieb des Dienstes unbedingt erforderlich sind, bieten wir kein Panel an, um sie zu deaktivieren, da dies die Funktionalität der Anwendung beeinträchtigen würde. Wenn Sie nicht möchten, dass wir diese Cookies verwenden, müssen Sie sich abmelden und Ihren Browser so konfigurieren, dass er sie blockiert, was jedoch die Nutzung der Plattform verhindert.",
			},
		],
	},
	it: {
		title: "Informativa sui Cookie (Analizzatore di Ricevute)",
		description: "Questa politica spiega come utilizziamo i cookie e tecnologie simili (come l'archiviazione locale del browser) per riconoscerti quando visiti la nostra piattaforma e garantire la sicurezza della tua sessione.",
		lastUpdated: "Ultimo aggiornamento: 6 aprile 2026",
		sections: [
			{
				title: "1. Cosa sono i cookie?",
				body: "I cookie sono piccoli file di testo memorizzati sul tuo dispositivo quando visiti un sito web. Li utilizziamo per ricordare le tue preferenze, mantenere la tua sessione di accesso sicura e capire come interagisci con la piattaforma.",
			},
			{
				title: "2. Cookie strettamente necessari",
				body: "La nostra piattaforma si basa su cookie tecnici essenziali per funzionare. Nello specifico, utilizziamo cookie sicuri (HttpOnly, Secure, SameSite) per gestire la tua sessione di autenticazione (es. `ff_auth_refresh_*`). Questi cookie non possono essere letti dagli script del browser, proteggendoti dagli attacchi XSS. Senza questi cookie, non saresti in grado di accedere o utilizzare i moduli privati.",
			},
			{
				title: "3. Archiviazione Locale (IndexedDB)",
				body: "Oltre ai cookie, utilizziamo l'archiviazione locale del tuo browser per migliorare la tua esperienza:\n\n- Preferenze di lingua: Salviamo la tua selezione della lingua per le visite future.\n- Modalità Offline (PWA): Utilizziamo IndexedDB per salvare temporaneamente le immagini delle tue ricevute se perdi la connessione a Internet. Questi dati vengono sincronizzati automaticamente con il server quando riacquisti la connessione e vengono quindi eliminati dal tuo dispositivo.",
			},
			{
				title: "4. Telemetria e Prestazioni",
				body: "Non utilizziamo cookie di tracciamento pubblicitario di terze parti. Raccogliamo metriche di prestazioni e attenzione in modo anonimo per migliorare l'interfaccia utente e rilevare errori sulla piattaforma, rispettando sempre la tua privacy.",
			},
			{
				title: "5. Gestione dei Cookie",
				body: "Poiché utilizziamo solo cookie strettamente necessari per la sicurezza e il funzionamento del servizio, non offriamo un pannello per disabilitarli, poiché ciò comprometterebbe la funzionalità dell'applicazione. Se non desideri che utilizziamo questi cookie, devi disconnetterti e configurare il tuo browser per bloccarli, sebbene ciò impedirà l'uso della piattaforma.",
			},
		],
	},
	fr: {
		title: "Politique relative aux cookies (Analyseur de Reçus)",
		description: "Cette politique explique comment nous utilisons les cookies et des technologies similaires (telles que le stockage local du navigateur) pour vous reconnaître lorsque vous visitez notre plateforme et garantir la sécurité de votre session.",
		lastUpdated: "Dernière mise à jour : 6 avril 2026",
		sections: [
			{
				title: "1. Que sont les cookies ?",
				body: "Les cookies sont de petits fichiers texte stockés sur votre appareil lorsque vous visitez un site Web. Nous les utilisons pour mémoriser vos préférences, maintenir votre session connectée en toute sécurité et comprendre comment vous interagissez avec la plateforme.",
			},
			{
				title: "2. Cookies strictement nécessaires",
				body: "Notre plateforme s'appuie sur des cookies techniques essentiels pour fonctionner. Plus précisément, nous utilisons des cookies sécurisés (HttpOnly, Secure, SameSite) pour gérer votre session d'authentification (par ex., `ff_auth_refresh_*`). Ces cookies ne peuvent pas être lus par les scripts du navigateur, ce qui vous protège contre les attaques XSS. Sans ces cookies, vous ne pourriez pas vous connecter ni utiliser les modules privés.",
			},
			{
				title: "3. Stockage Local (IndexedDB)",
				body: "En plus des cookies, nous utilisons le stockage local de votre navigateur pour améliorer votre expérience :\n\n- Préférences linguistiques : Nous enregistrons votre sélection de langue pour vos futures visites.\n- Mode Hors Ligne (PWA) : Nous utilisons IndexedDB pour enregistrer temporairement les images de vos reçus si vous perdez votre connexion Internet. Ces données sont automatiquement synchronisées avec le serveur lorsque vous retrouvez la connexion, puis sont supprimées de votre appareil.",
			},
			{
				title: "4. Télémétrie et Performances",
				body: "Nous n'utilisons pas de cookies de suivi publicitaire tiers. Nous collectons des mesures de performance et d'attention de manière anonyme pour améliorer l'interface utilisateur et détecter les erreurs sur la plateforme, en respectant toujours votre vie privée.",
			},
			{
				title: "5. Gestion des cookies",
				body: "Étant donné que nous n'utilisons que des cookies strictement nécessaires à la sécurité et au fonctionnement du service, nous ne proposons pas de panneau pour les désactiver, car cela empêcherait l'application de fonctionner. Si vous ne souhaitez pas que nous utilisions ces cookies, vous devez vous déconnecter et configurer votre navigateur pour les bloquer, bien que cela empêchera l'utilisation de la plateforme.",
			},
		],
	},
	ru: {
		title: "Политика использования файлов cookie (Парсер чеков)",
		description: "Эта политика объясняет, как мы используем файлы cookie и аналогичные технологии (например, локальное хранилище браузера), чтобы узнавать вас при посещении нашей платформы и обеспечивать безопасность вашей сессии.",
		lastUpdated: "Последнее обновление: 6 апреля 2026 г.",
		sections: [
			{
				title: "1. Что такое файлы cookie?",
				body: "Файлы cookie — это небольшие текстовые файлы, которые сохраняются на вашем устройстве при посещении веб-сайта. Мы используем их, чтобы запоминать ваши предпочтения, обеспечивать безопасность вашей сессии и понимать, как вы взаимодействуете с платформой.",
			},
			{
				title: "2. Строго необходимые файлы cookie",
				body: "Наша платформа полагается на основные технические файлы cookie для работы. В частности, мы используем безопасные файлы cookie (HttpOnly, Secure, SameSite) для управления вашей сессией аутентификации (например, `ff_auth_refresh_*`). Эти файлы cookie не могут быть прочитаны скриптами браузера, что защищает вас от XSS-атак. Без этих файлов cookie вы не смогли бы войти в систему или использовать приватные модули.",
			},
			{
				title: "3. Локальное хранилище (IndexedDB)",
				body: "Помимо файлов cookie, мы используем локальное хранилище вашего браузера для улучшения вашего опыта:\n\n- Языковые предпочтения: Мы сохраняем ваш выбор языка для будущих посещений.\n- Автономный режим (PWA): Мы используем IndexedDB для временного сохранения изображений ваших чеков, если вы потеряете подключение к Интернету. Эти данные автоматически синхронизируются с сервером при восстановлении подключения, а затем удаляются с вашего устройства.",
			},
			{
				title: "4. Телеметрия и производительность",
				body: "Мы не используем сторонние рекламные файлы cookie для отслеживания. Мы анонимно собираем метрики производительности и внимания, чтобы улучшить пользовательский интерфейс и обнаруживать ошибки на платформе, всегда уважая вашу конфиденциальность.",
			},
			{
				title: "5. Управление файлами cookie",
				body: "Поскольку мы используем только те файлы cookie, которые строго необходимы для безопасности и работы сервиса, мы не предлагаем панель для их отключения, так как это нарушит функциональность приложения. Если вы не хотите, чтобы мы использовали эти файлы cookie, вы должны выйти из системы и настроить свой браузер на их блокировку, хотя это сделает невозможным использование платформы.",
			},
		],
	},
};

export default function ReceiptParserCookiePolicyPage() {
	const { locale } = useI18n();
	const content = policies[locale] ?? policies.en!;

	return (
		<LegalPageTemplate
			title={content.title}
			description={content.description}
			lastUpdated={content.lastUpdated}
			sections={content.sections}
		/>
	);
}
```

### `src/app/receipts/history/page.tsx`
```tsx
"use client";

import { useCallback, useEffect, useRef, useState } from "react";

import { zodResolver } from "@hookform/resolvers/zod";
import { useForm, type Resolver } from "react-hook-form";
import { ConfirmDialog } from "@/components/ConfirmDialog";

import {
	receiptFormSchema,
	type ReceiptForm,
} from "@/modules/receipt-parser/receipt-form-schema";
import {
	listReceiptsHistory,
	getReceiptHistoryDetail,
	exportReceiptsCsv,
	exportReceiptsXlsx,
	type ReceiptHistoryRole,
	type ReceiptHistoryRow,
	persistedReceiptImageHash,
	saveReceiptToHistory,
	type ReceiptMathValidationError,
	SaveReceiptHistoryError,
	deleteReceiptFromHistory,
} from "@/app/receipts/receipt-history-api";
import { useI18n } from "@/app/i18n-provider";
import {
	handleAuthApiErrorSideEffects,
	resolveAuthApiUserMessage,
} from "@/app/auth/backend-auth-error";
import {
	useAccessToken,
	useAuthMe,
	useSupabaseAuth,
	useUserId,
} from "@/app/AuthProvider";
import { fetchReceiptsUsage } from "@/app/receipts/receipt-usage-api";
import { ReceiptHitlForm } from "@/app/receipts/components/ReceiptHitlForm";
import {
	type Receipt,
	receiptSchema,
} from "@/modules/receipt-parser/schemas";
import {
	serializeTaxRatePercentStringToFractionString,
} from "@/modules/receipt-parser/tax-rate";
import { ApiError, getApiErrorFields } from "@/lib/api-client";
import { Trash2 } from "lucide-react";
import { receiptLimitI18nKey } from "@/app/receipts/receipt-error-utils";

import { useMemo } from "react";

const PAGE_SIZE = 100;

function normalizeReceiptHistoryRowForUi(
	row: ReceiptHistoryRow | null | undefined
): ReceiptHistoryRow | null {
	if (!row) return null;
	const normalized = receiptSchema.parse(row);
	return { ...row, ...normalized };
}

export default function ReceiptsHistoryPage() {
	const { t, locale } = useI18n();
	const me = useAuthMe();
	const accessToken = useAccessToken();
	const userId = useUserId();
	const { isLoading: authLoading, signOut, loginWithGoogle } = useSupabaseAuth();
	const isAuthReady = !authLoading && me != null;
	const [isTierLoaded, setIsTierLoaded] = useState(false);
	const [activeTab, setActiveTab] = useState<ReceiptHistoryRole>("issuer");
	const [items, setItems] = useState<ReceiptHistoryRow[]>([]);
	const [offset, setOffset] = useState(0);
	const [hasMore, setHasMore] = useState(true);
	const [loading, setLoading] = useState<boolean>(false);
	const [exporting, setExporting] = useState(false);
	const [deletingReceiptId, setDeletingReceiptId] = useState<string | null>(
		null
	);
	const [deleteConfirmDialog, setDeleteConfirmDialog] = useState<{
		isOpen: boolean;
		receiptId: string | null;
		label: string;
	}>({ isOpen: false, receiptId: null, label: "" });
	const [error, setError] = useState<string | null>(null);
	const [showAuthLoginCta, setShowAuthLoginCta] = useState(false);

	const [selectedReceipt, setSelectedReceipt] = useState<ReceiptHistoryRow | null>(
		null
	);
	const [selectedReceiptImageHash, setSelectedReceiptImageHash] = useState<
		string | null
	>(null);
	const [loadingReceiptDetail, setLoadingReceiptDetail] = useState(false);
	const [showCorrectionHint, setShowCorrectionHint] = useState(false);
	const [validationErrors, setValidationErrors] = useState<
		ReceiptMathValidationError[] | null
	>(null);
	const [isSavingReceipt, setIsSavingReceipt] = useState(false);
	const [saveReceiptSuccess, setSaveReceiptSuccess] = useState(false);
	const [saveReceiptError, setSaveReceiptError] = useState(false);
	const [saveReceiptErrorMessage, setSaveReceiptErrorMessage] = useState<
		string | null
	>(null);
	const [correctionFallbackMessage, setCorrectionFallbackMessage] = useState<
		string | null
	>(null);
	const latestListRequestRef = useRef(0);

	const receiptForm = useForm<ReceiptForm>({
		resolver: zodResolver(receiptFormSchema as Parameters<typeof zodResolver>[0]) as unknown as Resolver<ReceiptForm>,
		defaultValues: undefined,
	});

	const selectedReceiptLabel = useMemo(() => {
		if (!selectedReceipt) return null;
		return `${selectedReceipt.invoice_number} (${selectedReceipt.invoice_date ?? "N/D"})`;
	}, [selectedReceipt]);

	const loadPage = useCallback(
		async (
			role: ReceiptHistoryRole,
			offsetVal: number,
			append: boolean
		): Promise<number> => {
			const requestId = ++latestListRequestRef.current;
			if (!isAuthReady) {
				setItems([]);
				setHasMore(false);
				setError(t("receipts.history.error.signInRequired"));
				setShowAuthLoginCta(true);
				return offsetVal;
			}
			setLoading(true);
			setError(null);
			setShowAuthLoginCta(false);
			try {
				const list = await listReceiptsHistory(
					role,
					PAGE_SIZE,
					offsetVal
				);
				if (requestId !== latestListRequestRef.current) {
					return offsetVal;
				}
				const normalizedList = list
					.map(normalizeReceiptHistoryRowForUi)
					.filter((row): row is ReceiptHistoryRow => row != null);
				setItems((prev) => (append ? [...prev, ...normalizedList] : normalizedList));
				setHasMore(list.length === PAGE_SIZE);
				return offsetVal + list.length;
			} catch (err) {
				if (requestId !== latestListRequestRef.current) {
					return offsetVal;
				}
				console.error(err);
				handleAuthApiErrorSideEffects(err, signOut);
				const authMsg = resolveAuthApiUserMessage(err, t);
				if (authMsg) {
					setError(authMsg);
					setShowAuthLoginCta(true);
				} else {
					setError(t("receipts.history.error.load"));
				}
				return offsetVal;
			} finally {
				if (requestId === latestListRequestRef.current) {
					setLoading(false);
				}
			}
		},
		[isAuthReady, signOut, t]
	);

	useEffect(() => {
		if (!isAuthReady) {
			setIsTierLoaded(false);
			return;
		}
		let cancelled = false;
		fetchReceiptsUsage({ userId })
			.then(() => { if (!cancelled) setIsTierLoaded(true); })
			.catch(() => { if (!cancelled) setIsTierLoaded(true); });
		return () => { cancelled = true; };
	}, [isAuthReady, userId]);

	useEffect(() => {
		if (authLoading || !isTierLoaded) {
			return;
		}
		let cancelled = false;
		if (!isAuthReady) {
			setItems([]);
			setHasMore(false);
			setOffset(0);
			setError(t("receipts.history.error.signInRequired"));
			setShowAuthLoginCta(true);
			return () => {
				cancelled = true;
			};
		}
		setError(null);
		setShowAuthLoginCta(false);
		loadPage(activeTab, 0, false).then((nextOffset) => {
			if (!cancelled) setOffset(nextOffset);
		});
		return () => {
			cancelled = true;
		};
	}, [activeTab, isAuthReady, authLoading, isTierLoaded, loadPage, t]);

	const handleLoadMore = (): void => {
		void loadPage(activeTab, offset, true).then(setOffset);
	};

	const handleDeleteReceipt = useCallback(
		async (id: string, label: string): Promise<void> => {
			setDeleteConfirmDialog({ isOpen: true, receiptId: id, label });
		},
		[]
	);

	const handleConfirmDelete = useCallback(
		async (): Promise<void> => {
			const id = deleteConfirmDialog.receiptId;
			if (!id) return;

			setDeleteConfirmDialog({ isOpen: false, receiptId: null, label: "" });
			setDeletingReceiptId(id);
			setError(null);
			try {
				await deleteReceiptFromHistory(id, accessToken);

				setItems((prev) => prev.filter((r) => r.id !== id));
				window.dispatchEvent(new CustomEvent("receipts-usage-invalidated"));
				if (selectedReceipt?.id === id) {
					setSelectedReceipt(null);
					setSelectedReceiptImageHash(null);
					setShowCorrectionHint(false);
					setValidationErrors(null);
				}
			} catch (err) {
				console.error(err);
				handleAuthApiErrorSideEffects(err, signOut);
				const api = getApiErrorFields(err);
				if (api?.status === 401) {
					setError(t("receipts.history.error.signInRequired"));
					setShowAuthLoginCta(true);
				} else if (
					api?.status === 403 &&
					api.detailCode === "RECEIPT_DELETE_FORBIDDEN"
				) {
					setShowAuthLoginCta(false);
					setError(t("receipts.history.deleteReceiptError403"));
				} else if (
					api?.status === 404 &&
					api.detailCode === "RECEIPT_DELETE_NOT_FOUND"
				) {
					setShowAuthLoginCta(false);
					setError(t("receipts.history.deleteReceiptError404"));
				} else {
					const authMsg = resolveAuthApiUserMessage(err, t);
					if (authMsg) {
						setError(authMsg);
						setShowAuthLoginCta(true);
					} else {
						setShowAuthLoginCta(false);
						setError(t("receipts.history.deleteReceiptError"));
					}
				}
			} finally {
				setDeletingReceiptId(null);
			}
		},
		[
			deleteConfirmDialog.receiptId,
			deleteReceiptFromHistory,
			accessToken,
			handleAuthApiErrorSideEffects,
			resolveAuthApiUserMessage,
			selectedReceipt,
			signOut,
			setError,
			t,
		]
	);

	const handleCancelDelete = useCallback(() => {
		setDeleteConfirmDialog({ isOpen: false, receiptId: null, label: "" });
	}, []);

	const normalizeDecimalString = (
		value: string | null | undefined
	): string | null | undefined => {
		if (value === null) return null;
		if (value === undefined) return undefined;

		const trimmed = value.trim();
		if (!trimmed) return trimmed;

		const hasComma = trimmed.includes(",");
		const hasDot = trimmed.includes(".");
		if (!hasComma) {
			return trimmed;
		}

		const noThousands = hasDot ? trimmed.replace(/\./g, "") : trimmed;
		return noThousands.replace(",", ".");
	};

	const normalizeNullableDecimal = (
		value: string | null | undefined
	): string | null | undefined => {
		const normalized = normalizeDecimalString(value);
		if (normalized === "") return null;
		return normalized;
	};

	const normalizeReceiptDecimals = (input: ReceiptForm): ReceiptForm => {
		return {
			...input,
			last_4_card_digits: input.last_4_card_digits.trim(),
			subtotal: normalizeDecimalString(input.subtotal) as string,
			discount: normalizeDecimalString(input.discount) as string,
			tax: normalizeDecimalString(input.tax) as string,
			total: normalizeDecimalString(input.total) as string,
			shipping: normalizeNullableDecimal(input.shipping),
			change_due: normalizeNullableDecimal(input.change_due),
			items: input.items.map((item) => ({
				...item,
				unit_price: normalizeDecimalString(item.unit_price) as string,
				total: normalizeDecimalString(item.total) as string,
				discount_amount: normalizeNullableDecimal(item.discount_amount),
				tax_rate: serializeTaxRatePercentStringToFractionString(item.tax_rate),
			})),
			taxes: input.taxes.map((taxBreakdown) => ({
				...taxBreakdown,
				tax_rate:
					serializeTaxRatePercentStringToFractionString(taxBreakdown.tax_rate) ??
					"0.00",
				base_amount: normalizeDecimalString(taxBreakdown.base_amount) as string,
				tax_amount: normalizeDecimalString(taxBreakdown.tax_amount) as string,
			})),
		};
	};

	const handleSelectReceiptRow = useCallback(
		async (receipt: ReceiptForm): Promise<void> => {
			setError(null);

			setSelectedReceipt(null);
			setLoadingReceiptDetail(true);
			setShowCorrectionHint(false);
			setValidationErrors(null);
			setSaveReceiptSuccess(false);
			setSaveReceiptError(false);
			setSaveReceiptErrorMessage(null);
			setCorrectionFallbackMessage(null);
			setSelectedReceiptImageHash(null);

			try {
				if (!isAuthReady) {
					setError(t("receipts.history.error.signInRequired"));
					setShowAuthLoginCta(true);
					return;
				}
				if (receipt.id) {

					const detail = await getReceiptHistoryDetail(receipt.id);
					const normalizedDetail = normalizeReceiptHistoryRowForUi(detail);
					if (!normalizedDetail) {
						throw new Error("Unable to normalize receipt detail.");
					}
					setSelectedReceipt(normalizedDetail);
					setSelectedReceiptImageHash(persistedReceiptImageHash(detail));
					receiptForm.reset(normalizedDetail);
				} else {

					const normalizedReceipt = normalizeReceiptHistoryRowForUi(receipt);
					if (!normalizedReceipt) {
						throw new Error("Unable to normalize receipt row.");
					}
					setSelectedReceipt(normalizedReceipt);
					setSelectedReceiptImageHash(persistedReceiptImageHash(receipt));
					receiptForm.reset(normalizedReceipt);
				}
			} catch (err) {
				console.error(err);
				if (err instanceof ApiError && err.status === 404) {

					const normalizedReceipt = normalizeReceiptHistoryRowForUi(receipt);
					if (!normalizedReceipt) {
						throw new Error("Unable to normalize receipt row.");
					}
					setSelectedReceipt(normalizedReceipt);
					setSelectedReceiptImageHash(persistedReceiptImageHash(receipt));
					receiptForm.reset(normalizedReceipt);
					return;
				}
				handleAuthApiErrorSideEffects(err, signOut);
				const authMsg = resolveAuthApiUserMessage(err, t);
				if (authMsg) {
					setError(authMsg);
					setShowAuthLoginCta(true);
				} else {
					setError(t("receipts.history.error.load"));
				}
			} finally {
				setLoadingReceiptDetail(false);
			}
		},
		[isAuthReady, receiptForm, signOut, t]
	);

	const handleSaveSelectedReceipt = useCallback(
		async (data: ReceiptForm): Promise<void> => {
			setIsSavingReceipt(true);
			setSaveReceiptSuccess(false);
			setSaveReceiptError(false);
			setShowAuthLoginCta(false);
			try {
				setSaveReceiptErrorMessage(null);
				setCorrectionFallbackMessage(null);

				const normalized = normalizeReceiptDecimals(data);

				const response = await saveReceiptToHistory(
					{
						receipt: normalized,
						user_role: normalized.user_role,
						image_hash: selectedReceiptImageHash,
					},
					{ accessToken }
				);

				if (response.status === "saved") {
					setSaveReceiptSuccess(true);
					setSaveReceiptError(false);
					setShowAuthLoginCta(false);
					setShowCorrectionHint(false);
					setValidationErrors(null);
					setCorrectionFallbackMessage(null);

					const saved = normalizeReceiptHistoryRowForUi(
						response.receipt as ReceiptHistoryRow
					);
					if (!saved) {
						throw new Error("Unable to normalize saved receipt.");
					}
					setSelectedReceipt(saved);
					setSelectedReceiptImageHash(persistedReceiptImageHash(saved));
					receiptForm.reset(saved);

					setItems((prev) =>
						prev.map((r) => {
							if (!r.id || !response.receipt.id) return r;
							if (r.id !== response.receipt.id) return r;
							return saved;
						})
					);
				} else if (response.status === "unprocessable") {
					setSaveReceiptError(true);
					setSaveReceiptSuccess(false);
					const msg = response.error_message ?? t("receipts.status.unprocessable");
					setSaveReceiptErrorMessage(msg);
					setCorrectionFallbackMessage(response.error_message ?? null);
					setShowCorrectionHint(true);
					setValidationErrors(response.validation_errors);
				}
			} catch (saveErr) {
				console.error(saveErr);
				handleAuthApiErrorSideEffects(saveErr, signOut);
				const authSaveMsg = resolveAuthApiUserMessage(saveErr, t);
				if (authSaveMsg) {
					setSaveReceiptError(true);
					setSaveReceiptSuccess(false);
					setSaveReceiptErrorMessage(authSaveMsg);
					setShowAuthLoginCta(true);
				} else if (saveErr instanceof SaveReceiptHistoryError) {
					setSaveReceiptError(true);
					setSaveReceiptSuccess(false);
					setSaveReceiptErrorMessage(
						saveErr.status === 429
							? t(receiptLimitI18nKey(saveErr.detail))
							: saveErr.message ?? t("receipts.status.error")
					);
				} else if (saveErr instanceof Error) {
					setSaveReceiptError(true);
					setSaveReceiptSuccess(false);
					setSaveReceiptErrorMessage(saveErr.message);
				} else {
					setSaveReceiptError(true);
					setSaveReceiptSuccess(false);
					setSaveReceiptErrorMessage(t("receipts.status.error"));
				}
			} finally {
				setIsSavingReceipt(false);
			}
		},
		[accessToken, receiptForm, selectedReceiptImageHash, signOut, t]
	);

	const handleExportCsv = async (): Promise<void> => {
		if (!isAuthReady) {
			setError(t("receipts.history.error.signInRequired"));
			setShowAuthLoginCta(true);
			return;
		}
		setExporting(true);
		setError(null);
		setShowAuthLoginCta(false);
		try {
			await exportReceiptsCsv(activeTab, locale, accessToken);
		} catch (err) {
			console.error(err);
			handleAuthApiErrorSideEffects(err, signOut);
			const authMsg = resolveAuthApiUserMessage(err, t);
			if (authMsg) {
				setError(authMsg);
				setShowAuthLoginCta(true);
			} else {
				setError(t("receipts.history.exportError"));
			}
		} finally {
			setExporting(false);
		}
	};

	const handleExportXlsx = async (): Promise<void> => {
		if (!isAuthReady) {
			setError(t("receipts.history.error.signInRequired"));
			setShowAuthLoginCta(true);
			return;
		}
		setExporting(true);
		setError(null);
		setShowAuthLoginCta(false);
		try {
			await exportReceiptsXlsx(activeTab, locale, accessToken);
		} catch (err) {
			console.error(err);
			handleAuthApiErrorSideEffects(err, signOut);
			const authMsg = resolveAuthApiUserMessage(err, t);
			if (authMsg) {
				setError(authMsg);
				setShowAuthLoginCta(true);
			} else {
				setError(t("receipts.history.exportError"));
			}
		} finally {
			setExporting(false);
		}
	};

	if (authLoading) {
		return (
			<main className="flex min-h-screen flex-col bg-slate-100 text-slate-900">
				<section className="mx-auto flex w-full max-w-4xl flex-1 flex-col px-6 py-6">
					<div className="rounded-xl border border-slate-200 bg-white px-4 py-10 text-center text-sm text-slate-600 shadow-sm">
						{t("auth.loading")}
					</div>
				</section>
			</main>
		);
	}

	return (
		<main className="flex min-h-screen flex-col bg-slate-100 text-slate-900">
			<section className="mx-auto flex w-full max-w-4xl flex-1 flex-col gap-4 px-6 py-6">
				<div className="flex flex-wrap items-center justify-between gap-3">
					<div className="flex flex-wrap gap-2">
						<button
							type="button"
							className={`rounded-full px-3 py-1 text-xs font-medium ${
								activeTab === "issuer"
									? "bg-slate-900 text-white"
									: "bg-slate-200 text-slate-800"
							}`}
							onClick={() => setActiveTab("issuer")}
						>
							{t("receipts.history.tab.issued")}
						</button>
						<button
							type="button"
							className={`rounded-full px-3 py-1 text-xs font-medium ${
								activeTab === "payer"
									? "bg-slate-900 text-white"
									: "bg-slate-200 text-slate-800"
							}`}
							onClick={() => setActiveTab("payer")}
						>
							{t("receipts.history.tab.received")}
						</button>
					</div>
					<div className="flex flex-wrap items-center gap-2">
						<button
							type="button"
							onClick={() => void handleExportCsv()}
							disabled={exporting || authLoading || !isAuthReady}
							className="rounded-full bg-slate-900 px-4 py-1.5 text-xs font-medium text-white disabled:opacity-60"
						>
							{exporting
								? t("receipts.history.exportCsv.loading")
								: t("receipts.history.exportCsv.label")}
						</button>
						<button
							type="button"
							onClick={() => void handleExportXlsx()}
							disabled={exporting || authLoading || !isAuthReady}
							className="rounded-full border border-slate-300 bg-white px-4 py-1.5 text-xs font-medium text-slate-800 disabled:opacity-60"
						>
							{exporting
								? t("receipts.history.exportXlsx.loading")
								: t("receipts.history.exportXlsx.label")}
						</button>
					</div>
				</div>

				{error && (
					<div className="space-y-2">
						<p className="text-xs text-rose-600">{error}</p>
						{showAuthLoginCta ? (
							<button
								type="button"
								className="rounded-lg bg-slate-900 px-3 py-1.5 text-xs font-medium text-white hover:bg-slate-800"
								onClick={() => {
									void loginWithGoogle();
								}}
							>
								{t("auth.signInGoogle")}
							</button>
						) : null}
					</div>
				)}

				<div className="flex-1 overflow-auto rounded-xl border border-slate-200 bg-white">
					<table className="min-w-full divide-y divide-slate-100 text-xs">
						<thead className="bg-slate-50">
							<tr>
								<th className="px-3 py-2 text-left font-medium text-slate-700">
									{t("receipts.history.table.header.date")}
								</th>
								<th className="px-3 py-2 text-left font-medium text-slate-700">
									{t("receipts.history.table.header.invoiceNumber")}
								</th>
								<th className="px-3 py-2 text-left font-medium text-slate-700">
									{t("receipts.history.table.header.dueDate")}
								</th>
								<th className="px-3 py-2 text-left font-medium text-slate-700">
									{t("receipts.history.table.header.currency")}
								</th>
								<th className="px-3 py-2 text-left font-medium text-slate-700">
									{t("receipts.history.table.header.seller")}
								</th>
								<th className="px-3 py-2 text-right font-medium text-slate-700">
									{t("receipts.history.table.header.buyer")}
								</th>
								<th className="px-3 py-2 text-right font-medium text-slate-700">
									{t("receipts.history.table.header.total")}
								</th>
								<th className="px-3 py-2 text-right font-medium text-slate-700">
									{/* Empty header for delete action */}
								</th>
							</tr>
						</thead>
						<tbody className="divide-y divide-slate-100 bg-white">
							{authLoading || (isAuthReady && !isTierLoaded) ? (
								<tr>
									<td
										colSpan={8}
										className="px-3 py-8 text-center text-xs text-slate-500"
									>
										{t("auth.loading")}
									</td>
								</tr>
							) : null}
							{items.map((receipt) => (
								<tr
									key={
										receipt.id ??
										`${receipt.invoice_number}-${receipt.invoice_date}-${receipt.total}`
									}
									role="button"
									tabIndex={0}
									onClick={() => {
										void handleSelectReceiptRow(receipt);
									}}
									className="cursor-pointer hover:bg-slate-50"
								>
									<td className="px-3 py-2 text-slate-800">
										{receipt.invoice_date ?? "N/D"}
									</td>
									<td className="px-3 py-2 font-medium text-slate-800">
										{receipt.invoice_number}
									</td>
									<td className="px-3 py-2 text-slate-700">
										{receipt.due_date ?? "N/D"}
									</td>
									<td className="px-3 py-2 text-slate-700">
										{receipt.currency}
									</td>
									<td className="px-3 py-2 text-slate-700">
										{receipt.issuer_name}
									</td>
									<td className="px-3 py-2 text-right text-slate-700">
										{receipt.payer_name}
									</td>
									<td className="px-3 py-2 text-right font-semibold text-slate-900">
										{receipt.total}
									</td>
									<td className="px-3 py-2 text-right">
										{receipt.id ? (
											<button
												type="button"
												aria-label={t("receipts.history.deleteReceiptAriaLabel")}
												className="rounded p-1 text-rose-500 hover:bg-rose-50 hover:text-rose-600 disabled:opacity-40"
												disabled={deletingReceiptId === receipt.id}
												onClick={(e) => {
													e.stopPropagation();
													if (!receipt.id) return;
													void handleDeleteReceipt(
														receipt.id,
														`${receipt.invoice_number} (${receipt.invoice_date ?? "N/D"})`
													);
												}}
											>
												<Trash2 size={16} />
											</button>
										) : null}
									</td>
								</tr>
							))}
							{!items.length && !loading && isAuthReady && isTierLoaded ? (
								<tr>
									<td colSpan={8} className="px-3 py-8 text-center text-xs text-slate-500">
										{t("receipts.history.empty")}
									</td>
								</tr>
							) : null}
						</tbody>
					</table>
				</div>

				{(selectedReceipt || loadingReceiptDetail) && (
					<div className="mt-4 rounded-xl border border-slate-200 bg-white p-4 shadow-sm">
						{loadingReceiptDetail ? (
							<p className="text-xs text-slate-600">
								{t("receipts.history.loadMore.loading")}
							</p>
						) : (
							<>
								{selectedReceiptLabel && (
									<div className="mb-2 flex items-center justify-between gap-2">
										<p className="text-xs font-semibold text-slate-800">
											Editando: {selectedReceiptLabel}
										</p>
										<button
											type="button"
											className="rounded-full border border-slate-200 bg-slate-50 px-3 py-1 text-[11px] font-medium text-slate-700 hover:bg-slate-100"
											onClick={() => {
												setSelectedReceipt(null);
												setSelectedReceiptImageHash(null);
												setShowCorrectionHint(false);
												setValidationErrors(null);
												setSaveReceiptSuccess(false);
												setSaveReceiptError(false);
												setSaveReceiptErrorMessage(null);
												setCorrectionFallbackMessage(null);
											}}
										>
											Cerrar
										</button>
									</div>
								)}
								<ReceiptHitlForm
									form={receiptForm}
									t={t}
									locale={locale}
									showCorrectionHint={showCorrectionHint}
									correctionFallbackMessage={correctionFallbackMessage}
									onSave={handleSaveSelectedReceipt}
									isSaving={isSavingReceipt}
									saveSuccess={saveReceiptSuccess}
									saveError={saveReceiptError}
									saveErrorMessage={saveReceiptErrorMessage}
									saveErrorShowLoginCta={showAuthLoginCta}
									onSaveErrorLogin={() => {
										void loginWithGoogle();
									}}
									validationErrors={validationErrors}
								/>

							</>
						)}
					</div>
				)}

				{hasMore && items.length > 0 && (
					<button
						type="button"
						onClick={handleLoadMore}
						disabled={loading}
						className="self-center rounded-full bg-slate-900 px-4 py-1.5 text-xs font-medium text-white disabled:opacity-60"
					>
						{loading
							? t("receipts.history.loadMore.loading")
							: t("receipts.history.loadMore.label")}
					</button>
				)}
			</section>

			<ConfirmDialog
				isOpen={deleteConfirmDialog.isOpen}
				title={t("receipts.history.deleteReceiptConfirmTitle")}
				message={t("receipts.history.deleteReceiptConfirm", {
					receipt: deleteConfirmDialog.label,
				})}
				confirmText={t("common.delete")}
				cancelText={t("common.cancel")}
				isDangerous
				onConfirm={handleConfirmDelete}
				onCancel={handleCancelDelete}
			/>
		</main>
	);
}
```

### `src/app/receipts/pricing/page.tsx`
```tsx
"use client";

import { useEffect, useState } from "react";
import Link from "next/link";

import { useI18n } from "@/app/i18n-provider";
import {
	fetchReceiptsPricing,
	type ReceiptsPricingResponse,
} from "@/app/receipts/receipt-pricing-api";
import { useAuthMe, useUserId } from "@/app/AuthProvider";
import { personalizeStripeLink } from "@/lib/stripe-utils";
import { getBackendUrl } from "@/app/auth/backend-auth-api";

const FALLBACK_PORTAL_PATH = "/billing/stripe/customer-portal";

/** Resolves the Stripe customer-portal URL from a backend-provided path or fallback. */
function resolvePortalUrl(customerPortalPath: string | undefined): string {
	const path = customerPortalPath?.trim();
	if (!path) return getBackendUrl(FALLBACK_PORTAL_PATH);
	if (path.startsWith("http://") || path.startsWith("https://")) return path;
	return getBackendUrl(path);
}

const cardConfigs = [
	{ id: "free", fallbackMax: 30, accent: "from-emerald-500/20 via-emerald-400/10 to-white" },
	{ id: "premium", fallbackMax: 500, accent: "from-sky-500/20 via-sky-400/10 to-white" },
	{ id: "pro", fallbackMax: 2_000, accent: "from-amber-500/20 via-amber-400/10 to-white" },
	{ id: "payg_topup", fallbackMax: 5_000, accent: "from-violet-500/20 via-violet-400/10 to-white" },
] as const;

export default function ReceiptsPricingPage() {
	const { t, locale } = useI18n();
	const userId = useUserId();
	const me = useAuthMe();
	const [data, setData] = useState<ReceiptsPricingResponse | null>(null);

	useEffect(() => {
		let cancelled = false;
		fetchReceiptsPricing()
			.then((res) => {
				if (!cancelled) setData(res);
			})
			.catch(() => {
				if (!cancelled) setData(null);
			});
		return () => {
			cancelled = true;
		};
	}, []);

	const { tiers, stripe_links } = data ?? { tiers: [], stripe_links: {} };
	const tierById = new Map(tiers.map((tier) => [tier.id, tier]));
	const linkByTierId: Record<string, string | undefined> = {
		premium: stripe_links.premium,
		pro: stripe_links.pro,
		payg_topup: stripe_links.payg_topup,
	};
	const currentTier = (me?.tier ?? "").trim().toLowerCase();
	const isCurrentTier = (tierId: string): boolean =>
		currentTier === tierId || (currentTier === "payg" && tierId === "payg_topup");
	const genericStripeUrl = "https://buy.stripe.com/replace-with-real-link-later";

	const portalUrl = resolvePortalUrl(data?.customer_portal_path);

	/**
	 * The user is a paid Receipts subscriber when:
	 * - auth/me identifies them as the receipt_parser module
	 * - their tier is premium or pro
	 * - they have a stripe_customer_id (means Stripe subscription exists)
	 * If auth/me hasn't loaded yet (null), treat as unauthenticated.
	 */
	const isReceiptPaidSubscriber =
		me?.module === "receipt_parser" &&
		(me?.tier === "premium" || me?.tier === "pro") &&
		!!me?.stripe_customer_id;

	return (
		<main className="flex min-h-screen flex-col bg-slate-100 text-slate-900">
			<header className="border-b border-slate-200 bg-white px-6 py-4">
				<div className="mx-auto flex w-full max-w-5xl flex-col gap-3 sm:flex-row sm:items-start sm:justify-between">
					<div>
						<h1 className="text-2xl font-semibold">{t("receipts.pricing.title")}</h1>
						<p className="mt-1 max-w-2xl text-sm text-slate-600">
							{t("receipts.pricing.subtitle")}
						</p>
					</div>
					<Link
						href="/receipts"
						className="shrink-0 self-start rounded-full border border-slate-300 bg-white px-3 py-1.5 text-xs font-medium text-slate-700 shadow-sm transition hover:border-slate-400 hover:text-slate-900"
					>
						{t("receipts.pricing.backToScanner")}
					</Link>
				</div>
			</header>

			<section className="mx-auto w-full max-w-5xl flex-1 px-6 py-8">
				<div className="grid gap-5 lg:grid-cols-2">
					{cardConfigs.map((card) => {
						const backendTier = tierById.get(card.id);
						const monthlyMax = backendTier?.max_receipts ?? card.fallbackMax;

						let button: React.ReactNode = null;

						if (card.id === "free") {
							if (currentTier === "free") {
								button = (
									<span className="inline-flex items-center justify-center rounded-full border border-slate-300 bg-slate-100 px-4 py-2 text-xs font-medium text-slate-700">
										{t("receipts.pricing.currentPlan")}
									</span>
								);
							}
						} else if (card.id === "premium" || card.id === "pro") {
							if (isReceiptPaidSubscriber) {

								button = (
									<a
										href={portalUrl}
										target="_blank"
										rel="noopener noreferrer"
										className="inline-flex items-center justify-center rounded-full bg-slate-900 px-4 py-2 text-xs font-medium text-white transition hover:bg-slate-800"
									>
										{t("common.manageInStripe")}
									</a>
								);
							} else {

								const href = personalizeStripeLink(
									linkByTierId[card.id] ?? genericStripeUrl,
									userId,
									locale,
									card.id,
									"receipt_parser",
								);
								button = isCurrentTier(card.id) ? (
									<a
										href={href}
										target="_blank"
										rel="noopener noreferrer"
										className="inline-flex items-center justify-center rounded-full border border-slate-300 bg-slate-100 px-4 py-2 text-xs font-medium text-slate-700 transition hover:bg-slate-200"
									>
										{t("receipts.pricing.currentPlan")}
									</a>
								) : (
									<a
										href={href}
										target="_blank"
										rel="noopener noreferrer"
										className="inline-flex items-center justify-center rounded-full bg-slate-900 px-4 py-2 text-xs font-medium text-white transition hover:bg-slate-800"
									>
										{t("receipts.pricing.subscribe")}
									</a>
								);
							}
						} else if (card.id === "payg_topup") {

							const href = personalizeStripeLink(
								linkByTierId["payg_topup"] ?? genericStripeUrl,
								userId,
								locale,
								card.id,
								"receipt_parser",
							);
							button = (
								<a
									href={href}
									target="_blank"
									rel="noopener noreferrer"
									className="inline-flex items-center justify-center rounded-full bg-slate-900 px-4 py-2 text-xs font-medium text-white transition hover:bg-slate-800"
								>
									{t("receipts.pricing.topUp")}
								</a>
							);
						}

						return (
							<article
								key={card.id}
								className="overflow-hidden rounded-3xl border border-slate-200 bg-white shadow-sm"
							>
								<div className={`bg-gradient-to-br ${card.accent} px-5 py-4`}>
									<p className="text-sm font-semibold uppercase tracking-[0.2em] text-slate-700">
										{t(`receipts.pricing.card.${card.id}.title`)}
									</p>
								</div>

								<div className="p-5">
									<div className="grid gap-3 sm:grid-cols-2">
										<div className="rounded-2xl border border-slate-200 bg-slate-50 px-4 py-3">
											<p className="text-[11px] font-semibold uppercase tracking-wide text-slate-500">
												{t("receipts.pricing.monthly")}
											</p>
											<p className="mt-1 text-sm font-semibold text-slate-900">
												{t("receipts.pricing.monthlyMax", {
													count: String(monthlyMax),
												})}
											</p>
										</div>
										<div className="rounded-2xl border border-slate-200 bg-slate-50 px-4 py-3">
											<p className="text-[11px] font-semibold uppercase tracking-wide text-slate-500">
												{t("receipts.pricing.daily")}
											</p>
											<p className="mt-1 text-sm font-semibold text-slate-900">
												{t(`receipts.pricing.card.${card.id}.daily`)}
											</p>
										</div>
									</div>

									<div className="mt-4 grid gap-3 sm:grid-cols-2">
										<div className="rounded-2xl border border-slate-200 bg-slate-50 px-4 py-3">
											<p className="text-[11px] font-semibold uppercase tracking-wide text-slate-500">
												{t("receipts.pricing.llmModel")}
											</p>
											<p className="mt-1 text-sm font-semibold text-slate-900">
												{t(`receipts.pricing.card.${card.id}.llmModel`)}
											</p>
										</div>
										<div className="rounded-2xl border border-slate-200 bg-slate-50 px-4 py-3">
											<p className="text-[11px] font-semibold uppercase tracking-wide text-slate-500">
												{t("receipts.pricing.storage")}
											</p>
											<p className="mt-1 text-sm font-semibold text-slate-900">
												{t(`receipts.pricing.card.${card.id}.storage`)}
											</p>
										</div>
									</div>

									{button && (
										<div className="mt-5 flex items-center justify-end gap-3">
											{button}
										</div>
									)}
								</div>
							</article>
						);
					})}
				</div>
			</section>
		</main>
	);
}
```

### `src/app/receipts/pricing/ReceiptsPricingDocumentation.tsx`
```tsx
"use client";

export function ReceiptsPricingDocumentation() {
	return (
		<div className="sr-only">
			<h2>Vera pricing</h2>
			<h3>Free</h3>
			<h3>Premium</h3>
			<h3>Pro</h3>
			<h3>PAYG</h3>
		</div>
	);
}
```

### `src/app/receipts/privacy/page.tsx`
```tsx
"use client";

import { useI18n } from "@/app/i18n-provider";
import { LegalPageTemplate } from "@/components/LegalPageTemplate";
import type { Locale } from "@/lib/i18n-config";

type PrivacyPolicyContent = {
	title: string;
	description: string;
	lastUpdated: string;
	sections: { title: string; body: string }[];
};

const policies: Record<Locale, PrivacyPolicyContent> = {
	en: {
		title: "Privacy Policy (Receipt Parser)",
		description: "Your privacy and data security are fundamental to us. This policy explains how we collect, protect, and process your information using enterprise-grade security standards (SOC2, GDPR).",
		lastUpdated: "Last updated: April 6, 2026",
		sections: [
			{
				title: "1. Data We Collect",
				body: "We collect your account information (name, email, avatar) via Google OAuth. We also process the receipt and invoice images you voluntarily upload and usage telemetry data to ensure system performance and correct billing (FinOps).",
			},
			{
				title: "2. Data Protection and Encryption (ALE)",
				body: "We implement Application-Level Encryption (ALE). This means sensitive data (such as the details of your invoices, names, and concepts) is encrypted on our servers before being saved to the database. Not even our database administrators can read your information in clear text.",
			},
			{
				title: "3. AI Processing and Data Loss Prevention (DLP)",
				body: "To provide our services, we use language models (LLMs) hosted on secure infrastructures. To ensure your privacy, we use a Data Loss Prevention (DLP Proxy) system and anonymization (PII Scrubbing). Before any sensitive text from your receipts leaves our infrastructure to an external AI provider, we detect and hide personally identifiable information (names, emails, card numbers, SSNs).",
			},
			{
				title: "4. Zero Data Retention & Third-Party AI",
				body: "Our data sharing policies depend on your subscription tier:\n\n- Free Tiers: We use providers like Groq and LlamaParse. Please note that data processed through these free tiers MAY be used by these providers to train their models.\n- Paid Tiers (Premium, Pro, PAYG): We use Google Vertex AI. We maintain strict agreements ensuring Zero Data Retention. Data sent through these paid APIs is NOT used to train foundational models and is discarded after processing.",
			},
			{
				title: "5. Data Retention and Deletion",
				body: "Your data belongs to you. You can delete your receipts and invoices at any time from your history.\n\nIf you cancel your paid subscription, we apply a 90-day grace period. After this time, an automated process will permanently delete your old data and search vectors to comply with data minimization principles.",
			},
		],
	},
	es: {
		title: "Política de Privacidad (Receipt Parser)",
		description: "Tu privacidad y la seguridad de tus datos son fundamentales para nosotros. Esta política explica cómo recopilamos, protegemos y procesamos tu información utilizando estándares de seguridad de nivel empresarial (SOC2, GDPR).",
		lastUpdated: "Última actualización: 6 de abril de 2026",
		sections: [
			{
				title: "1. Datos que recopilamos",
				body: "Recopilamos información de tu cuenta (nombre, email, avatar) a través de Google OAuth. También procesamos los archivos que subes voluntariamente (imágenes de facturas y recibos) y los datos de telemetría de uso para garantizar el rendimiento del sistema y la facturación correcta (FinOps).",
			},
			{
				title: "2. Protección de Datos y Cifrado (ALE)",
				body: "Implementamos Cifrado a Nivel de Aplicación (Application-Level Encryption - ALE). Esto significa que los datos sensibles (como los detalles de tus facturas, nombres y conceptos) se cifran en nuestros servidores antes de guardarse en la base de datos. Ni siquiera nuestros administradores de bases de datos pueden leer tu información en texto claro.",
			},
			{
				title: "3. Procesamiento de IA y Prevención de Pérdida de Datos (DLP)",
				body: "Para proporcionar nuestros servicios, utilizamos modelos de lenguaje (LLMs) alojados en infraestructuras seguras. Para garantizar tu privacidad, utilizamos un sistema de Prevención de Pérdida de Datos (DLP Proxy) y anonimización (PII Scrubbing). Antes de que cualquier texto sensible de tus facturas salga de nuestra infraestructura hacia un proveedor de IA externo, detectamos y ocultamos información personal identificable (nombres, emails, números de tarjeta, SSN).",
			},
			{
				title: "4. Zero Data Retention y Proveedores de IA",
				body: "Nuestras políticas de compartición de datos dependen de tu plan de suscripción:\n\n- Planes Gratuitos (Free Tier): Utilizamos proveedores como Groq y LlamaParse. Ten en cuenta que los datos procesados a través de estos niveles gratuitos SÍ pueden ser utilizados por estos proveedores para entrenar sus modelos.\n- Planes de Pago (Premium, Pro, PAYG): Utilizamos Google Vertex AI. Mantenemos acuerdos estrictos que garantizan 'Zero Data Retention'. Los datos enviados a través de estas APIs de pago NO se utilizan para entrenar modelos fundacionales y se descartan tras el procesamiento.",
			},
			{
				title: "5. Retención y Eliminación de Datos",
				body: "Tus datos te pertenecen. Puedes eliminar tus recibos y facturas en cualquier momento desde tu historial.\n\nSi cancelas tu suscripción de pago, aplicamos un período de gracia de 90 días. Pasado este tiempo, un proceso automatizado eliminará permanentemente tus datos antiguos y vectores de búsqueda para cumplir con los principios de minimización de datos.",
			},
		],
	},
	de: {
		title: "Datenschutzrichtlinie (Beleg-Parser)",
		description: "Ihre Privatsphäre und Datensicherheit sind für uns von grundlegender Bedeutung. Diese Richtlinie erklärt, wie wir Ihre Informationen unter Verwendung von Sicherheitsstandards auf Unternehmensebene (SOC2, DSGVO) erfassen, schützen und verarbeiten.",
		lastUpdated: "Zuletzt aktualisiert: 6. April 2026",
		sections: [
			{
				title: "1. Daten, die wir erfassen",
				body: "Wir erfassen Ihre Kontoinformationen (Name, E-Mail, Avatar) über Google OAuth. Wir verarbeiten auch die Rechnungs- und Belegbilder, die Sie freiwillig hochladen, sowie Nutzungs-Telemetriedaten, um die Systemleistung und korrekte Abrechnung (FinOps) sicherzustellen.",
			},
			{
				title: "2. Datenschutz und Verschlüsselung (ALE)",
				body: "Wir implementieren Application-Level Encryption (ALE). Das bedeutet, dass sensible Daten (wie die Details Ihrer Rechnungen, Namen und Konzepte) auf unseren Servern verschlüsselt werden, bevor sie in der Datenbank gespeichert werden. Nicht einmal unsere Datenbankadministratoren können Ihre Informationen im Klartext lesen.",
			},
			{
				title: "3. KI-Verarbeitung und Data Loss Prevention (DLP)",
				body: "Um unsere Dienste bereitzustellen, verwenden wir Sprachmodelle (LLMs), die auf sicheren Infrastrukturen gehostet werden. Um Ihre Privatsphäre zu gewährleisten, verwenden wir ein Data Loss Prevention (DLP Proxy) System und Anonymisierung (PII Scrubbing). Bevor sensibler Text aus Ihren Belegen unsere Infrastruktur zu einem externen KI-Anbieter verlässt, erkennen und verbergen wir persönlich identifizierbare Informationen (Namen, E-Mails, Kartennummern, SSNs).",
			},
			{
				title: "4. Zero Data Retention & Drittanbieter-KI",
				body: "Unsere Datenaustauschrichtlinien hängen von Ihrer Abonnementstufe ab:\n\n- Kostenlose Stufen: Wir verwenden Anbieter wie Groq und LlamaParse. Bitte beachten Sie, dass Daten, die über diese kostenlosen Stufen verarbeitet werden, von diesen Anbietern zum Trainieren ihrer Modelle verwendet werden KÖNNEN.\n- Kostenpflichtige Stufen (Premium, Pro, PAYG): Wir verwenden Google Vertex AI. Wir unterhalten strenge Vereinbarungen, die Zero Data Retention gewährleisten. Daten, die über diese kostenpflichtigen APIs gesendet werden, werden NICHT zum Trainieren von Basismodellen verwendet und nach der Verarbeitung verworfen.",
			},
			{
				title: "5. Datenaufbewahrung und -löschung",
				body: "Ihre Daten gehören Ihnen. Sie können Ihre Belege und Rechnungen jederzeit aus Ihrem Verlauf löschen.\n\nWenn Sie Ihr kostenpflichtiges Abonnement kündigen, gewähren wir eine Nachfrist von 90 Tagen. Nach dieser Zeit löscht ein automatisierter Prozess Ihre alten Daten und Suchvektoren dauerhaft, um den Grundsätzen der Datenminimierung zu entsprechen.",
			},
		],
	},
	it: {
		title: "Informativa sulla Privacy (Analizzatore di Ricevute)",
		description: "La tua privacy e la sicurezza dei tuoi dati sono fondamentali per noi. Questa politica spiega come raccogliamo, proteggiamo e trattiamo le tue informazioni utilizzando standard di sicurezza di livello aziendale (SOC2, GDPR).",
		lastUpdated: "Ultimo aggiornamento: 6 aprile 2026",
		sections: [
			{
				title: "1. Dati che raccogliamo",
				body: "Raccogliamo le informazioni del tuo account (nome, email, avatar) tramite Google OAuth. Trattiamo anche le immagini di ricevute e fatture che carichi volontariamente e i dati di telemetria di utilizzo per garantire le prestazioni del sistema e la corretta fatturazione (FinOps).",
			},
			{
				title: "2. Protezione dei Dati e Crittografia (ALE)",
				body: "Implementiamo la crittografia a livello di applicazione (ALE). Ciò significa che i dati sensibili (come i dettagli delle tue fatture, nomi e concetti) vengono crittografati sui nostri server prima di essere salvati nel database. Nemmeno i nostri amministratori di database possono leggere le tue informazioni in chiaro.",
			},
			{
				title: "3. Elaborazione AI e Prevenzione della Perdita di Dati (DLP)",
				body: "Per fornire i nostri servizi, utilizziamo modelli linguistici (LLM) ospitati su infrastrutture sicure. Per garantire la tua privacy, utilizziamo un sistema di Data Loss Prevention (DLP Proxy) e anonimizzazione (PII Scrubbing). Prima che qualsiasi testo sensibile delle tue ricevute lasci la nostra infrastruttura verso un fornitore AI esterno, rileviamo e nascondiamo le informazioni di identificazione personale (nomi, email, numeri di carta, SSN).",
			},
			{
				title: "4. Zero Data Retention e AI di Terze Parti",
				body: "Le nostre politiche di condivisione dei dati dipendono dal tuo livello di abbonamento:\n\n- Livelli Gratuiti: Utilizziamo fornitori come Groq e LlamaParse. Tieni presente che i dati elaborati tramite questi livelli gratuiti POSSONO essere utilizzati da questi fornitori per addestrare i loro modelli.\n- Livelli a Pagamento (Premium, Pro, PAYG): Utilizziamo Google Vertex AI. Manteniamo accordi rigorosi che garantiscono la Zero Data Retention. I dati inviati tramite queste API a pagamento NON vengono utilizzati per addestrare modelli fondamentali e vengono scartati dopo l'elaborazione.",
			},
			{
				title: "5. Conservazione e Cancellazione dei Dati",
				body: "I tuoi dati ti appartengono. Puoi eliminare le tue ricevute e fatture in qualsiasi momento dalla tua cronologia.\n\nSe annulli il tuo abbonamento a pagamento, applichiamo un periodo di grazia di 90 giorni. Trascorso questo tempo, un processo automatizzato eliminerà in modo permanente i tuoi vecchi dati e i vettori di ricerca per conformarsi ai principi di minimizzazione dei dati.",
			},
		],
	},
	fr: {
		title: "Politique de Confidentialité (Analyseur de Reçus)",
		description: "Votre vie privée et la sécurité de vos données sont fondamentales pour nous. Cette politique explique comment nous collectons, protégeons et traitons vos informations en utilisant des normes de sécurité d'entreprise (SOC2, RGPD).",
		lastUpdated: "Dernière mise à jour : 6 avril 2026",
		sections: [
			{
				title: "1. Données que nous collectons",
				body: "Nous collectons les informations de votre compte (nom, e-mail, avatar) via Google OAuth. Nous traitons également les images de reçus et de factures que vous téléchargez volontairement et les données de télémétrie d'utilisation pour garantir les performances du système et une facturation correcte (FinOps).",
			},
			{
				title: "2. Protection des Données et Chiffrement (ALE)",
				body: "Nous mettons en œuvre le chiffrement au niveau de l'application (ALE). Cela signifie que les données sensibles (telles que les détails de vos factures, noms et concepts) sont chiffrées sur nos serveurs avant d'être enregistrées dans la base de données. Même nos administrateurs de base de données ne peuvent pas lire vos informations en clair.",
			},
			{
				title: "3. Traitement IA et Prévention des Pertes de Données (DLP)",
				body: "Pour fournir nos services, nous utilisons des modèles de langage (LLM) hébergés sur des infrastructures sécurisées. Pour garantir votre vie privée, nous utilisons un système de prévention des pertes de données (DLP Proxy) et d'anonymisation (PII Scrubbing). Avant que tout texte sensible de vos reçus ne quitte notre infrastructure vers un fournisseur d'IA externe, nous détectons et masquons les informations personnellement identifiables (noms, e-mails, numéros de carte, SSN).",
			},
			{
				title: "4. Zéro Rétention de Données et IA Tiers",
				body: "Nos politiques de partage de données dépendent de votre niveau d'abonnement :\n\n- Niveaux Gratuits : Nous utilisons des fournisseurs comme Groq et LlamaParse. Veuillez noter que les données traitées via ces niveaux gratuits PEUVENT être utilisées par ces fournisseurs pour entraîner leurs modèles.\n- Niveaux Payants (Premium, Pro, PAYG) : Nous utilisons Google Vertex AI. Nous maintenons des accords stricts garantissant la Zéro Rétention de Données. Les données envoyées via ces API payantes ne sont PAS utilisées pour entraîner des modèles fondamentaux et sont supprimées après traitement.",
			},
			{
				title: "5. Rétention et Suppression des Données",
				body: "Vos données vous appartiennent. Vous pouvez supprimer vos reçus et factures à tout moment depuis votre historique.\n\nSi vous annulez votre abonnement payant, nous appliquons un délai de grâce de 90 jours. Passé ce délai, un processus automatisé supprimera définitivement vos anciennes données et vecteurs de recherche pour se conformer aux principes de minimisation des données.",
			},
		],
	},
	ru: {
		title: "Политика конфиденциальности (Парсер чеков)",
		description: "Ваша конфиденциальность и безопасность данных имеют для нас фундаментальное значение. Эта политика объясняет, как мы собираем, защищаем и обрабатываем вашу информацию, используя стандарты безопасности корпоративного уровня (SOC2, GDPR).",
		lastUpdated: "Последнее обновление: 6 апреля 2026 г.",
		sections: [
			{
				title: "1. Данные, которые мы собираем",
				body: "Мы собираем информацию о вашей учетной записи (имя, email, аватар) через Google OAuth. Мы также обрабатываем изображения чеков и счетов, которые вы добровольно загружаете, и данные телеметрии использования для обеспечения производительности системы и правильного биллинга (FinOps).",
			},
			{
				title: "2. Защита данных и шифрование (ALE)",
				body: "Мы внедряем шифрование на уровне приложения (ALE). Это означает, что конфиденциальные данные (такие как детали ваших счетов, имена и концепции) шифруются на наших серверах перед сохранением в базу данных. Даже наши администраторы баз данных не могут прочитать вашу информацию в открытом виде.",
			},
			{
				title: "3. Обработка ИИ и предотвращение потери данных (DLP)",
				body: "Для предоставления наших услуг мы используем языковые модели (LLM), размещенные на защищенных инфраструктурах. Для обеспечения вашей конфиденциальности мы используем систему предотвращения потери данных (DLP Proxy) и анонимизацию (PII Scrubbing). Прежде чем любой конфиденциальный текст из ваших чеков покинет нашу инфраструктуру и отправится к внешнему поставщику ИИ, мы обнаруживаем и скрываем личную информацию (имена, email, номера карт, SSN).",
			},
			{
				title: "4. Нулевое хранение данных и сторонний ИИ",
				body: "Наши политики обмена данными зависят от уровня вашей подписки:\n\n- Бесплатные уровни: Мы используем таких поставщиков, как Groq и LlamaParse. Обратите внимание, что данные, обрабатываемые через эти бесплатные уровни, МОГУТ использоваться этими поставщиками для обучения их моделей.\n- Платные уровни (Premium, Pro, PAYG): Мы используем Google Vertex AI. Мы поддерживаем строгие соглашения, гарантирующие нулевое хранение данных. Данные, отправляемые через эти платные API, НЕ используются для обучения базовых моделей и удаляются после обработки.",
			},
			{
				title: "5. Хранение и удаление данных",
				body: "Ваши данные принадлежат вам. Вы можете удалить свои чеки и счета в любое время из своей истории.\n\nЕсли вы отмените платную подписку, мы предоставляем льготный период в 90 дней. По истечении этого времени автоматизированный процесс навсегда удалит ваши старые данные и векторы поиска в соответствии с принципами минимизации данных.",
			},
		],
	},
};

export default function ReceiptParserPrivacyPolicyPage() {
	const { locale } = useI18n();
	const content = policies[locale] ?? policies.en;

	return (
		<LegalPageTemplate
			title={content.title}
			description={content.description}
			lastUpdated={content.lastUpdated}
			sections={content.sections}
		/>
	);
}
```

### `src/app/receipts/terms/page.tsx`
```tsx
"use client";

import { useI18n } from "@/app/i18n-provider";
import { LegalPageTemplate } from "@/components/LegalPageTemplate";
import type { Locale } from "@/lib/i18n-config";

type TermsOfUseContent = {
	title: string;
	description: string;
	lastUpdated: string;
	sections: { title: string; body: string }[];
};

const policies: Record<Locale, TermsOfUseContent> = {
	en: {
		title: "Terms of Use (Receipt Parser)",
		description: "These Terms of Use govern the access and use of our Artificial Intelligence platform (specifically the Receipt Parser module). By using our services, you accept these conditions in their entirety.",
		lastUpdated: "Last updated: April 6, 2026",
		sections: [
			{
				title: "1. Description of Service",
				body: "Our platform offers AI-powered tools for invoice and receipt data extraction (Receipt Parser).\n\nThe user acknowledges that AI systems are probabilistic and may generate inaccurate results ('hallucinations'). Although we implement strict mathematical and logical validations, the user is solely responsible for verifying the accuracy of the generated data before its accounting, tax, or commercial use.",
			},
			{
				title: "2. Accounts, Subscriptions, and PAYG System",
				body: "We offer free and paid plans (Premium, Pro) as well as a Pay-As-You-Go (PAYG) system. Payments are securely processed through Stripe.\n\n- PAYG credits and subscription limits are consumed based on the token usage of the AI models.\n- No refunds are offered for AI credits already consumed.\n- We reserve the right to suspend accounts that attempt to evade usage limits (Rate Limits) or that make abusive use of the API.",
			},
			{
				title: "3. Privacy and Data Security",
				body: "Security is our priority. We implement Application-Level Encryption (ALE) and Data Loss Prevention (DLP) systems to protect your information. By using the service, you agree that we process your data according to our Privacy Policy.\n\nThe user agrees not to use the platform to process illegal material, content that infringes copyright, or highly sensitive unauthorized data, despite our anonymization measures.",
			},
			{
				title: "4. Intellectual Property",
				body: "The user retains all intellectual property rights over the receipt and invoice images they upload to the platform. We retain all rights over the code, algorithms, system prompts, and platform infrastructure.\n\nThe user grants the platform a temporary and strictly limited license to process their files for the sole purpose of providing the requested service.",
			},
			{
				title: "5. Limitation of Liability",
				body: "The service is provided 'as is'. To the maximum extent permitted by law, we are not liable for indirect damages, loss of profits, loss of data, or business interruptions arising from the use or inability to use the platform, including failures in external AI providers (e.g., service outages in Vertex AI or Groq).",
			},
		],
	},
	es: {
		title: "Términos de Uso (Receipt Parser)",
		description: "Estos Términos de Uso regulan el acceso y uso de nuestra plataforma de Inteligencia Artificial (específicamente el módulo Receipt Parser). Al utilizar nuestros servicios, aceptas estas condiciones en su totalidad.",
		lastUpdated: "Última actualización: 6 de abril de 2026",
		sections: [
			{
				title: "1. Descripción del Servicio",
				body: "Nuestra plataforma ofrece herramientas potenciadas por Inteligencia Artificial para la extracción de datos de facturas y recibos (Receipt Parser).\n\nEl usuario reconoce que los sistemas de IA son probabilísticos y pueden generar resultados inexactos ('alucinaciones'). Aunque implementamos validaciones matemáticas y lógicas estrictas, el usuario es el único responsable de verificar la exactitud de los datos generados antes de su uso contable, fiscal o comercial.",
			},
			{
				title: "2. Cuentas, Suscripciones y Sistema PAYG",
				body: "Ofrecemos planes gratuitos y de pago (Premium, Pro) así como un sistema de pago por uso (Pay-As-You-Go o PAYG). Los pagos se procesan de forma segura a través de Stripe.\n\n- Los créditos PAYG y los límites de suscripción se consumen según el uso de tokens de los modelos de IA.\n- No se ofrecen reembolsos por créditos de IA ya consumidos.\n- Nos reservamos el derecho de suspender cuentas que intenten evadir los límites de uso (Rate Limits) o que realicen un uso abusivo de la API.",
			},
			{
				title: "3. Privacidad y Seguridad de los Datos",
				body: "La seguridad es nuestra prioridad. Implementamos cifrado a nivel de aplicación (ALE) y sistemas de prevención de pérdida de datos (DLP) para proteger tu información. Al usar el servicio, aceptas que procesemos tus datos según nuestra Política de Privacidad.\n\nEl usuario se compromete a no utilizar la plataforma para procesar material ilegal, contenido que infrinja derechos de autor, o datos altamente sensibles no autorizados, a pesar de nuestras medidas de anonimización.",
			},
			{
				title: "4. Propiedad Intelectual",
				body: "El usuario retiene todos los derechos de propiedad intelectual sobre las imágenes de facturas y recibos que suba a la plataforma. Nosotros retenemos todos los derechos sobre el código, algoritmos, prompts del sistema y la infraestructura de la plataforma.\n\nEl usuario otorga a la plataforma una licencia temporal y estrictamente limitada para procesar sus archivos con el único fin de prestarle el servicio solicitado.",
			},
			{
				title: "5. Limitación de Responsabilidad",
				body: "El servicio se proporciona 'tal cual'. En la medida máxima permitida por la ley, no nos hacemos responsables de daños indirectos, pérdida de beneficios, pérdida de datos o interrupciones del negocio derivadas del uso o la imposibilidad de uso de la plataforma, incluyendo fallos en los proveedores de IA externos (ej. caídas de servicio en Vertex AI o Groq).",
			},
		],
	},
	de: {
		title: "Nutzungsbedingungen (Beleg-Parser)",
		description: "Diese Nutzungsbedingungen regeln den Zugang und die Nutzung unserer Plattform für Künstliche Intelligenz (insbesondere das Modul Receipt Parser). Durch die Nutzung unserer Dienste akzeptieren Sie diese Bedingungen in vollem Umfang.",
		lastUpdated: "Zuletzt aktualisiert: 6. April 2026",
		sections: [
			{
				title: "1. Beschreibung des Dienstes",
				body: "Unsere Plattform bietet KI-gestützte Tools zur Extraktion von Rechnungs- und Belegdaten (Receipt Parser).\n\nDer Benutzer erkennt an, dass KI-Systeme probabilistisch sind und ungenaue Ergebnisse ('Halluzinationen') erzeugen können. Obwohl wir strenge mathematische und logische Validierungen implementieren, ist der Benutzer allein dafür verantwortlich, die Richtigkeit der generierten Daten vor ihrer buchhalterischen, steuerlichen oder kommerziellen Nutzung zu überprüfen.",
			},
			{
				title: "2. Konten, Abonnements und PAYG-System",
				body: "Wir bieten kostenlose und kostenpflichtige Pläne (Premium, Pro) sowie ein Pay-As-You-Go (PAYG)-System an. Zahlungen werden sicher über Stripe abgewickelt.\n\n- PAYG-Guthaben und Abonnementlimits werden basierend auf der Token-Nutzung der KI-Modelle verbraucht.\n- Für bereits verbrauchte KI-Guthaben werden keine Rückerstattungen gewährt.\n- Wir behalten uns das Recht vor, Konten zu sperren, die versuchen, Nutzungslimits (Rate Limits) zu umgehen oder die API missbräuchlich zu nutzen.",
			},
			{
				title: "3. Datenschutz und Datensicherheit",
				body: "Sicherheit ist unsere Priorität. Wir implementieren Application-Level Encryption (ALE) und Data Loss Prevention (DLP)-Systeme, um Ihre Informationen zu schützen. Durch die Nutzung des Dienstes stimmen Sie zu, dass wir Ihre Daten gemäß unserer Datenschutzrichtlinie verarbeiten.\n\nDer Benutzer verpflichtet sich, die Plattform nicht zur Verarbeitung von illegalem Material, urheberrechtsverletzenden Inhalten oder hochsensiblen, nicht autorisierten Daten zu nutzen, trotz unserer Anonymisierungsmaßnahmen.",
			},
			{
				title: "4. Geistiges Eigentum",
				body: "Der Benutzer behält alle geistigen Eigentumsrechte an den Rechnungs- und Belegbildern, die er auf die Plattform hochlädt. Wir behalten alle Rechte an Code, Algorithmen, System-Prompts und der Plattform-Infrastruktur.\n\nDer Benutzer gewährt der Plattform eine temporäre und streng begrenzte Lizenz zur Verarbeitung seiner Dateien ausschließlich zu dem Zweck, den angeforderten Dienst bereitzustellen.",
			},
			{
				title: "5. Haftungsbeschränkung",
				body: "Der Dienst wird 'wie besehen' bereitgestellt. Soweit gesetzlich zulässig, haften wir nicht für indirekte Schäden, entgangenen Gewinn, Datenverlust oder Betriebsunterbrechungen, die sich aus der Nutzung oder der Unmöglichkeit der Nutzung der Plattform ergeben, einschließlich Ausfällen bei externen KI-Anbietern (z. B. Dienstausfälle bei Vertex AI oder Groq).",
			},
		],
	},
	it: {
		title: "Termini di Utilizzo (Analizzatore di Ricevute)",
		description: "Questi Termini di Utilizzo regolano l'accesso e l'uso della nostra piattaforma di Intelligenza Artificiale (nello specifico il modulo Receipt Parser). Utilizzando i nostri servizi, accetti queste condizioni per intero.",
		lastUpdated: "Ultimo aggiornamento: 6 aprile 2026",
		sections: [
			{
				title: "1. Descrizione del Servizio",
				body: "La nostra piattaforma offre strumenti basati sull'intelligenza artificiale per l'estrazione di dati da fatture e ricevute (Receipt Parser).\n\nL'utente riconosce che i sistemi di intelligenza artificiale sono probabilistici e possono generare risultati inaccurati ('allucinazioni'). Sebbene implementiamo rigorose convalide matematiche e logiche, l'utente è l'unico responsabile della verifica dell'accuratezza dei dati generati prima del loro utilizzo contabile, fiscale o commerciale.",
			},
			{
				title: "2. Account, Abbonamenti e Sistema PAYG",
				body: "Offriamo piani gratuiti e a pagamento (Premium, Pro) nonché un sistema Pay-As-You-Go (PAYG). I pagamenti vengono elaborati in modo sicuro tramite Stripe.\n\n- I crediti PAYG e i limiti di abbonamento vengono consumati in base all'utilizzo dei token dei modelli AI.\n- Non sono previsti rimborsi per i crediti AI già consumati.\n- Ci riserviamo il diritto di sospendere gli account che tentano di eludere i limiti di utilizzo (Rate Limits) o che fanno un uso abusivo dell'API.",
			},
			{
				title: "3. Privacy e Sicurezza dei Dati",
				body: "La sicurezza è la nostra priorità. Implementiamo sistemi di crittografia a livello di applicazione (ALE) e prevenzione della perdita di dati (DLP) per proteggere le tue informazioni. Utilizzando il servizio, accetti che trattiamo i tuoi dati secondo la nostra Informativa sulla Privacy.\n\nL'utente si impegna a non utilizzare la piattaforma per elaborare materiale illegale, contenuti che violano il copyright o dati altamente sensibili non autorizzati, nonostante le nostre misure di anonimizzazione.",
			},
			{
				title: "4. Proprietà Intellettuale",
				body: "L'utente mantiene tutti i diritti di proprietà intellettuale sulle immagini di fatture e ricevute che carica sulla piattaforma. Noi manteniamo tutti i diritti sul codice, sugli algoritmi, sui prompt di sistema e sull'infrastruttura della piattaforma.\n\nL'utente concede alla piattaforma una licenza temporanea e strettamente limitata per elaborare i propri file al solo scopo di fornire il servizio richiesto.",
			},
			{
				title: "5. Limitazione di Responsabilità",
				body: "Il servizio è fornito 'così com'è'. Nella misura massima consentita dalla legge, non siamo responsabili per danni indiretti, perdita di profitti, perdita di dati o interruzioni dell'attività derivanti dall'uso o dall'impossibilità di utilizzare la piattaforma, inclusi guasti nei fornitori di intelligenza artificiale esterni (es. interruzioni del servizio in Vertex AI o Groq).",
			},
		],
	},
	fr: {
		title: "Conditions d'Utilisation (Analyseur de Reçus)",
		description: "Ces Conditions d'Utilisation régissent l'accès et l'utilisation de notre plateforme d'Intelligence Artificielle (spécifiquement le module Receipt Parser). En utilisant nos services, vous acceptez ces conditions dans leur intégralité.",
		lastUpdated: "Dernière mise à jour : 6 avril 2026",
		sections: [
			{
				title: "1. Description du Service",
				body: "Notre plateforme propose des outils basés sur l'IA pour l'extraction de données de factures et de reçus (Receipt Parser).\n\nL'utilisateur reconnaît que les systèmes d'IA sont probabilistes et peuvent générer des résultats inexacts ('hallucinations'). Bien que nous mettions en œuvre des validations mathématiques et logiques strictes, l'utilisateur est seul responsable de la vérification de l'exactitude des données générées avant leur utilisation comptable, fiscale ou commerciale.",
			},
			{
				title: "2. Comptes, Abonnements et Système PAYG",
				body: "Nous proposons des forfaits gratuits et payants (Premium, Pro) ainsi qu'un système de paiement à l'utilisation (Pay-As-You-Go ou PAYG). Les paiements sont traités de manière sécurisée via Stripe.\n\n- Les crédits PAYG et les limites d'abonnement sont consommés en fonction de l'utilisation des jetons des modèles d'IA.\n- Aucun remboursement n'est proposé pour les crédits d'IA déjà consommés.\n- Nous nous réservons le droit de suspendre les comptes qui tentent de contourner les limites d'utilisation (Rate Limits) ou qui font un usage abusif de l'API.",
			},
			{
				title: "3. Confidentialité et Sécurité des Données",
				body: "La sécurité est notre priorité. Nous mettons en œuvre des systèmes de chiffrement au niveau de l'application (ALE) et de prévention des pertes de données (DLP) pour protéger vos informations. En utilisant le service, vous acceptez que nous traitions vos données conformément à notre Politique de Confidentialité.\n\nL'utilisateur s'engage à ne pas utiliser la plateforme pour traiter du matériel illégal, du contenu enfreignant les droits d'auteur ou des données hautement sensibles non autorisées, malgré nos mesures d'anonymisation.",
			},
			{
				title: "4. Propriété Intellectuelle",
				body: "L'utilisateur conserve tous les droits de propriété intellectuelle sur les images de factures et de reçus qu'il télécharge sur la plateforme. Nous conservons tous les droits sur le code, les algorithmes, les invites système et l'infrastructure de la plateforme.\n\nL'utilisateur accorde à la plateforme une licence temporaire et strictement limitée pour traiter ses fichiers dans le seul but de fournir le service demandé.",
			},
			{
				title: "5. Limitation de Responsabilité",
				body: "Le service est fourni 'tel quel'. Dans toute la mesure permise par la loi, nous ne sommes pas responsables des dommages indirects, de la perte de profits, de la perte de données ou des interruptions d'activité découlant de l'utilisation ou de l'impossibilité d'utiliser la plateforme, y compris les défaillances des fournisseurs d'IA externes (par ex., pannes de service chez Vertex AI ou Groq).",
			},
		],
	},
	ru: {
		title: "Условия использования (Парсер чеков)",
		description: "Эти Условия использования регулируют доступ и использование нашей платформы искусственного интеллекта (в частности, модуля Receipt Parser). Используя наши услуги, вы полностью принимаете эти условия.",
		lastUpdated: "Последнее обновление: 6 апреля 2026 г.",
		sections: [
			{
				title: "1. Описание сервиса",
				body: "Наша платформа предлагает инструменты на базе ИИ для извлечения данных из счетов и чеков (Receipt Parser).\n\nПользователь признает, что системы ИИ являются вероятностными и могут генерировать неточные результаты ('галлюцинации'). Хотя мы внедряем строгие математические и логические проверки, пользователь несет единоличную ответственность за проверку точности сгенерированных данных перед их бухгалтерским, налоговым или коммерческим использованием.",
			},
			{
				title: "2. Аккаунты, подписки и система PAYG",
				body: "Мы предлагаем бесплатные и платные планы (Premium, Pro), а также систему оплаты по мере использования (Pay-As-You-Go или PAYG). Платежи безопасно обрабатываются через Stripe.\n\n- Кредиты PAYG и лимиты подписки расходуются на основе использования токенов моделями ИИ.\n- Возврат средств за уже израсходованные кредиты ИИ не производится.\n- Мы оставляем за собой право приостанавливать действие учетных записей, которые пытаются обойти ограничения использования (Rate Limits) или злоупотребляют API.",
			},
			{
				title: "3. Конфиденциальность и безопасность данных",
				body: "Безопасность — наш приоритет. Мы внедряем системы шифрования на уровне приложения (ALE) и предотвращения потери данных (DLP) для защиты вашей информации. Используя сервис, вы соглашаетесь с тем, что мы обрабатываем ваши данные в соответствии с нашей Политикой конфиденциальности.\n\nПользователь обязуется не использовать платформу для обработки незаконных материалов, контента, нарушающего авторские права, или строго конфиденциальных несанкционированных данных, несмотря на наши меры по анонимизации.",
			},
			{
				title: "4. Интеллектуальная собственность",
				body: "Пользователь сохраняет все права интеллектуальной собственности на изображения чеков и счетов, которые он загружает на платформу. Мы сохраняем все права на код, алгоритмы, системные промпты и инфраструктуру платформы.\n\nПользователь предоставляет платформе временную и строго ограниченную лицензию на обработку своих файлов с единственной целью предоставления запрошенной услуги.",
			},
			{
				title: "5. Ограничение ответственности",
				body: "Сервис предоставляется 'как есть'. В максимальной степени, разрешенной законом, мы не несем ответственности за косвенные убытки, упущенную выгоду, потерю данных или перебои в работе бизнеса, возникающие в результате использования или невозможности использования платформы, включая сбои у внешних поставщиков ИИ (например, перебои в обслуживании Vertex AI или Groq).",
			},
		],
	},
};

export default function ReceiptParserTermsOfUsePage() {
	const { locale } = useI18n();
	const content = policies[locale] ?? policies.en!;

	return (
		<LegalPageTemplate
			title={content.title}
			description={content.description}
			lastUpdated={content.lastUpdated}
			sections={content.sections}
		/>
	);
}
```

### `src/components/ConfirmDialog.tsx`
```tsx
"use client";

import { useEffect } from "react";

interface ConfirmDialogProps {
	isOpen: boolean;
	title: string;
	message: string;
	confirmText?: string;
	cancelText?: string;
	isDangerous?: boolean;
	onConfirm: () => void;
	onCancel: () => void;
}

export function ConfirmDialog({
	isOpen,
	title,
	message,
	confirmText = "Confirmar",
	cancelText = "Cancelar",
	isDangerous = false,
	onConfirm,
	onCancel,
}: ConfirmDialogProps) {
	useEffect(() => {
		if (!isOpen) return;

		const handleKeyDown = (e: KeyboardEvent) => {
			if (e.key === "Escape") {
				onCancel();
			}
		};

		document.addEventListener("keydown", handleKeyDown);
		return () => document.removeEventListener("keydown", handleKeyDown);
	}, [isOpen, onCancel]);

	if (!isOpen) return null;

	return (
		<>
			<div
				className="fixed inset-0 z-40 bg-black bg-opacity-40"
				onClick={onCancel}
				role="presentation"
			/>
			<div className="fixed left-1/2 top-1/2 z-50 w-full max-w-sm -translate-x-1/2 -translate-y-1/2 rounded-lg border border-slate-200 bg-white shadow-lg">
				<div className="border-b border-slate-200 px-6 py-4">
					<h2 className="text-lg font-semibold text-slate-900">{title}</h2>
				</div>
				<div className="px-6 py-4">
					<p className="text-sm text-slate-600">{message}</p>
				</div>
				<div className="flex gap-2 border-t border-slate-200 px-6 py-3">
					<button
						type="button"
						onClick={onCancel}
						className="flex-1 rounded-lg border border-slate-300 bg-white px-4 py-2 text-sm font-medium text-slate-700 transition hover:bg-slate-50 active:bg-slate-100"
					>
						{cancelText}
					</button>
					<button
						type="button"
						onClick={onConfirm}
						className={`flex-1 rounded-lg px-4 py-2 text-sm font-medium text-white transition ${
							isDangerous
								? "bg-rose-600 hover:bg-rose-700 active:bg-rose-800"
								: "bg-slate-900 hover:bg-slate-800 active:bg-slate-900"
						}`}
					>
						{confirmText}
					</button>
				</div>
			</div>
		</>
	);
}
```

### `src/components/DatePickerField.tsx`
```tsx
"use client";

import { useEffect, useRef, useState } from "react";

type DatePickerFieldProps = {
	value: string | null | undefined;
	onChange: (value: string) => void;
	hasError?: boolean;
	placeholder?: string;
	ariaLabel?: string;
	allowClear?: boolean;
	clearLabel?: string;
	locale?: string;
};

function parseISODateStr(
	value: string | null | undefined
): { year: number; month: number; day: number } | null {
	if (!value) return null;
	const match = value.trim().match(/^(\d{4})-(\d{2})-(\d{2})/);
	if (!match) return null;
	const year = parseInt(match[1]!, 10);
	const month = parseInt(match[2]!, 10) - 1; // 0-indexed
	const day = parseInt(match[3]!, 10);
	if (isNaN(year) || isNaN(month) || isNaN(day)) return null;
	if (month < 0 || month > 11 || day < 1 || day > 31) return null;
	return { year, month, day };
}

function toISO(year: number, month: number, day: number): string {
	return `${year}-${String(month + 1).padStart(2, "0")}-${String(day).padStart(2, "0")}`;
}

function getDaysInMonth(year: number, month: number): number {
	return new Date(year, month + 1, 0).getDate();
}

function getFirstWeekday(year: number, month: number): number {
	return new Date(year, month, 1).getDay(); // 0=Sunday
}

export function DatePickerField({
	value,
	onChange,
	hasError = false,
	placeholder = "—",
	ariaLabel,
	allowClear = false,
	clearLabel = "Borrar",
	locale = "es",
}: DatePickerFieldProps) {
	const parsed = parseISODateStr(value);
	const today = new Date();

	const [isOpen, setIsOpen] = useState(false);
	const [viewYear, setViewYear] = useState(parsed?.year ?? today.getFullYear());
	const [viewMonth, setViewMonth] = useState(parsed?.month ?? today.getMonth());

	const wrapperRef = useRef<HTMLDivElement>(null);

	useEffect(() => {
		const p = parseISODateStr(value);
		if (p) {
			setViewYear(p.year);
			setViewMonth(p.month);
		}
	}, [value]);

	useEffect(() => {
		if (!isOpen) return;
		const onDown = (e: MouseEvent) => {
			if (wrapperRef.current && !wrapperRef.current.contains(e.target as Node)) {
				setIsOpen(false);
			}
		};
		document.addEventListener("mousedown", onDown);
		return () => document.removeEventListener("mousedown", onDown);
	}, [isOpen]);

	const openCalendar = () => {
		const p = parseISODateStr(value);
		setViewYear(p?.year ?? today.getFullYear());
		setViewMonth(p?.month ?? today.getMonth());
		setIsOpen(true);
	};

	const prevMonth = () => {
		if (viewMonth === 0) {
			setViewMonth(11);
			setViewYear((y) => y - 1);
		} else {
			setViewMonth((m) => m - 1);
		}
	};
	const nextMonth = () => {
		if (viewMonth === 11) {
			setViewMonth(0);
			setViewYear((y) => y + 1);
		} else {
			setViewMonth((m) => m + 1);
		}
	};
	const prevYear = () => setViewYear((y) => y - 1);
	const nextYear = () => setViewYear((y) => y + 1);

	const selectDay = (day: number) => {
		onChange(toISO(viewYear, viewMonth, day));
		setIsOpen(false);
	};

	const totalDays = getDaysInMonth(viewYear, viewMonth);
	const startOffset = getFirstWeekday(viewYear, viewMonth);
	const cells: Array<number | null> = [
		...Array<null>(startOffset).fill(null),
		...Array.from({ length: totalDays }, (_, i) => i + 1),
	];
	while (cells.length % 7 !== 0) cells.push(null);

	const displayText = parsed
		? new Intl.DateTimeFormat(locale, {
				day: "2-digit",
				month: "short",
				year: "numeric",
			}).format(new Date(parsed.year, parsed.month, parsed.day))
		: null;

	const monthYearLabel = new Intl.DateTimeFormat(locale, {
		month: "long",
		year: "numeric",
	}).format(new Date(viewYear, viewMonth, 1));

	const dayHeaders = Array.from({ length: 7 }, (_, i) =>
		new Intl.DateTimeFormat(locale, { weekday: "short" }).format(
			new Date(2023, 0, i + 1)
		)
	);

	return (
		<div ref={wrapperRef} className="relative">
			<button
				type="button"
				onClick={openCalendar}
				aria-label={ariaLabel}
				aria-expanded={isOpen}
				aria-haspopup="dialog"
				className={`w-full rounded border text-left px-2 py-1 text-xs leading-normal ${
					hasError
						? "border-rose-500 bg-rose-50/40 text-rose-900"
						: "border-slate-300 bg-white text-slate-800 hover:border-slate-400"
				}`}
			>
				{displayText ?? <span className="text-slate-400">{placeholder}</span>}
			</button>

			{isOpen && (
				<div
					className="absolute z-50 mt-1 rounded-lg border border-slate-200 bg-white p-3 shadow-xl"
					style={{ minWidth: "220px" }}
				>
					{/* Month / Year navigation */}
					<div className="mb-2 flex items-center justify-between gap-1">
						<button
							type="button"
							onClick={prevYear}
							className="rounded px-1.5 py-0.5 text-xs text-slate-500 hover:bg-slate-100"
							title="Año anterior"
						>
							«
						</button>
						<button
							type="button"
							onClick={prevMonth}
							className="rounded px-1.5 py-0.5 text-xs text-slate-500 hover:bg-slate-100"
							title="Mes anterior"
						>
							‹
						</button>
						<span className="flex-1 text-center text-[11px] font-semibold capitalize text-slate-800">
							{monthYearLabel}
						</span>
						<button
							type="button"
							onClick={nextMonth}
							className="rounded px-1.5 py-0.5 text-xs text-slate-500 hover:bg-slate-100"
							title="Mes siguiente"
						>
							›
						</button>
						<button
							type="button"
							onClick={nextYear}
							className="rounded px-1.5 py-0.5 text-xs text-slate-500 hover:bg-slate-100"
							title="Año siguiente"
						>
							»
						</button>
					</div>

					{/* Day name headers */}
					<div className="mb-1 grid grid-cols-7">
						{dayHeaders.map((h, i) => (
							<span
								key={i}
								className="text-center text-[9px] font-medium uppercase text-slate-400"
							>
								{h.slice(0, 2)}
							</span>
						))}
					</div>

					{/* Day cells */}
					<div className="grid grid-cols-7 gap-px">
						{cells.map((day, i) => {
							if (day === null) return <span key={i} />;
							const isSelected =
								parsed?.year === viewYear &&
								parsed?.month === viewMonth &&
								parsed?.day === day;
							const isToday =
								today.getFullYear() === viewYear &&
								today.getMonth() === viewMonth &&
								today.getDate() === day;
							return (
								<button
									key={i}
									type="button"
									onClick={() => selectDay(day)}
									className={`rounded py-1 text-center text-[10px] ${
										isSelected
											? "bg-slate-900 font-semibold text-white"
											: isToday
												? "bg-slate-100 font-medium text-slate-900 ring-1 ring-slate-300"
												: "text-slate-700 hover:bg-slate-100"
									}`}
								>
									{day}
								</button>
							);
						})}
					</div>

					{allowClear && (
						<div className="mt-2 border-t border-slate-100 pt-2 text-center">
							<button
								type="button"
								onClick={() => {
									onChange("");
									setIsOpen(false);
								}}
								className="text-[10px] text-slate-400 hover:text-rose-600"
							>
								{clearLabel}
							</button>
						</div>
					)}
				</div>
			)}
		</div>
	);
}
```

### `src/components/IndeterminateProgressBar.tsx`
```tsx
type IndeterminateProgressBarProps = {
	ariaLabel: string;
	className?: string;
	trackClassName?: string;
	fillClassName?: string;
};

export function IndeterminateProgressBar({
	ariaLabel,
	className = "",
	trackClassName = "bg-slate-200",
	fillClassName = "bg-gradient-to-r from-slate-400 via-slate-600 to-slate-400",
}: IndeterminateProgressBarProps) {
	return (
		<div
			role="status"
			aria-live="polite"
			aria-label={ariaLabel}
			className={className}
		>
			<div className={`h-2 w-full overflow-hidden rounded-full ${trackClassName}`}>
				<div
					className={`indeterminate-progress h-full w-1/3 rounded-full ${fillClassName}`}
				/>
			</div>
		</div>
	);
}
```

### `src/components/LegalPageTemplate.tsx`
```tsx
import React from "react";

type LegalSection = {
	title: string;
	body: string;
};

type LegalPageTemplateProps = {
	title: string;
	description: string;
	lastUpdated: string;
	sections: LegalSection[];
};

export function LegalPageTemplate({
	title,
	description,
	lastUpdated,
	sections,
}: LegalPageTemplateProps) {
	return (
		<main className="min-h-screen bg-slate-100 px-6 py-10 text-slate-900">
			<section className="mx-auto flex w-full max-w-4xl flex-col gap-6">
				<div className="rounded-3xl border border-slate-200 bg-white p-6 shadow-sm">
					<p className="text-xs font-semibold uppercase tracking-[0.2em] text-slate-500">
						Legal
					</p>
					<h1 className="mt-3 text-3xl font-semibold text-slate-950">{title}</h1>
					<p className="mt-3 max-w-3xl text-sm leading-6 text-slate-600">
						{description}
					</p>
					<p className="mt-4 text-xs text-slate-500">
						{lastUpdated}
					</p>
				</div>

				<div className="space-y-4">
					{sections.map((section) => (
						<article
							key={section.title}
							className="rounded-3xl border border-slate-200 bg-white p-6 shadow-sm"
						>
							<h2 className="text-lg font-semibold text-slate-950">
								{section.title}
							</h2>
							<p className="mt-3 whitespace-pre-line text-sm leading-6 text-slate-600">
								{section.body}
							</p>
						</article>
					))}
				</div>
			</section>
		</main>
	);
}
```

### `src/components/ModuleFooter.tsx`
```tsx
"use client";

import Link from "next/link";
import { Ticket } from "lucide-react";
import { useI18n } from "@/app/i18n-provider";

export type ModuleFooterProps = {
	moduleLabel: string;
	canOpenSupport: boolean;
	onOpenTickets: () => void;
	blockedSupportLabel?: string;
};

export function ModuleFooter({
	moduleLabel,
	canOpenSupport,
	onOpenTickets,
	blockedSupportLabel,
}: ModuleFooterProps) {
	const { t } = useI18n();

	const basePath = moduleLabel === "pdf_anki" ? "/pdf-anki" : "/receipts";

	return (
		<footer className="border-t border-slate-200 bg-white">
			<div className="mx-auto flex max-w-6xl flex-col gap-3 px-4 py-3 text-xs sm:flex-row sm:items-center sm:justify-end">
				<div className="flex flex-wrap items-center gap-2">
					<Link
						href={`${basePath}/terms`}
						className="inline-flex h-8 items-center rounded-full border border-slate-200 bg-slate-50 px-3 text-slate-700 transition hover:bg-slate-100 hover:text-slate-900"
					>
						{t("legal.terms")}
					</Link>
					<Link
						href={`${basePath}/privacy`}
						className="inline-flex h-8 items-center rounded-full border border-slate-200 bg-slate-50 px-3 text-slate-700 transition hover:bg-slate-100 hover:text-slate-900"
					>
						{t("legal.privacy")}
					</Link>
					<Link
						href={`${basePath}/cookies`}
						className="inline-flex h-8 items-center rounded-full border border-slate-200 bg-slate-50 px-3 text-slate-700 transition hover:bg-slate-100 hover:text-slate-900"
					>
						{t("legal.cookies")}
					</Link>
					<button
						type="button"
						onClick={onOpenTickets}
						aria-label={t("support.footer.ticketsAria")}
						title={
							canOpenSupport
								? t("support.footer.ticketsAria")
								: blockedSupportLabel ?? t("support.footer.upgradeHint")
						}
						disabled={!canOpenSupport}
						className="inline-flex h-8 w-8 items-center justify-center rounded-full border border-slate-200 bg-slate-50 text-slate-700 hover:bg-slate-100 disabled:cursor-not-allowed disabled:opacity-50"
					>
						<Ticket className="h-4 w-4" strokeWidth={1.75} />
					</button>
				</div>
			</div>
		</footer>
	);
}
```

### `src/components/SupportTicketModal.tsx`
```tsx
export { SupportTicketPanel as SupportTicketModal } from "@/components/SupportTicketPanel";
```

### `src/components/SupportTicketPanel.tsx`
```tsx
"use client";

import { useEffect, useMemo, useRef, useState } from "react";

import { useI18n } from "@/app/i18n-provider";
import { getApiErrorFields } from "@/lib/api-client";
import {
	normalizeSupportAttachmentPath,
	resolveSupportAttachmentSrc,
} from "@/lib/support-attachment-url";
import {
	createTicket,
	deleteTicket,
	fetchTickets,
	MAX_TICKET_ATTACHMENTS,
	replyToTicket,
	type ActiveTicket,
	type SupportModule,
	type SupportHistoryEntry,
} from "@/lib/support-api";

type TicketMessage = {
	sender: "user" | "support";
	body: string;
	attachments: string[];
};

type LocalAttachment = {
	id: string;
	file: File;
	previewUrl: string;
};

const ALLOWED_ATTACHMENT_EXTENSIONS = new Set([
	"jpg",
	"jpeg",
	"png",
	"webp",
	"tiff",
	"avif",
	"heic",
	"heif",
]);

const ALLOWED_ATTACHMENT_MIME_TYPES = new Set([
	"image/jpeg",
	"image/png",
	"image/webp",
	"image/tiff",
	"image/avif",
	"image/heic",
	"image/heif",
]);

const MAX_ATTACHMENT_SIZE_BYTES = 5 * 1024 * 1024;
const MAX_TOTAL_ATTACHMENT_SIZE_BYTES = 25 * 1024 * 1024;

function normalizeSupportMessageBody(body: string): string {
	return body.replace(/^support team:\s*/i, "").replace(/^support:\s*/i, "").trim();
}

function attachmentsFromUnknownRecord(
	record: Record<string, unknown>,
): string[] {
	const raw = record.attachments;
	if (!Array.isArray(raw)) return [];
	return raw
		.map((item): string | null => {
			if (!item || typeof item !== "object") return null;
			const attachment = item as Record<string, unknown>;
			return typeof attachment.url === "string"
				? normalizeSupportAttachmentPath(attachment.url)
				: null;
		})
		.filter((v): v is string => Boolean(v));
}

function createLocalAttachment(file: File): LocalAttachment {
	return {
		id: `${file.name}-${file.size}-${file.lastModified}-${Math.random().toString(36).slice(2)}`,
		file,
		previewUrl: URL.createObjectURL(file),
	};
}

function revokeAttachment(attachment: LocalAttachment): void {
	URL.revokeObjectURL(attachment.previewUrl);
}

function isSupportedTicketImage(file: File): boolean {
	const normalizedType = file.type.toLowerCase();
	if (ALLOWED_ATTACHMENT_MIME_TYPES.has(normalizedType)) {
		return true;
	}
	const dotIndex = file.name.lastIndexOf(".");
	if (dotIndex < 0) return false;
	const extension = file.name.slice(dotIndex + 1).toLowerCase();
	return ALLOWED_ATTACHMENT_EXTENSIONS.has(extension);
}

export type SupportTicketPanelProps = {
	/** Must match the product session: `pdf_anki` or `receipt_parser` (API query). */
	module: SupportModule;
	isOpen: boolean;
	onClose: () => void;
	isAllowedTier: boolean;
	blockedMessage?: string;
};

function parseHistoryMessages(history: string | null): TicketMessage[] {
	if (!history) return [];
	try {
		const parsed = JSON.parse(history) as unknown;
		if (!Array.isArray(parsed)) return [];
		return parsed
			.map((entry): TicketMessage | null => {
				if (!entry || typeof entry !== "object") return null;
				const record = entry as Record<string, unknown>;
				const body = typeof record.message === "string" ? record.message : null;
				if (!body) return null;
				return {
					sender: record.sender === "support" ? "support" : "user",
					body:
						record.sender === "support" ? normalizeSupportMessageBody(body) : body,
					attachments: attachmentsFromUnknownRecord(record),
				};
			})
			.filter((entry): entry is TicketMessage => entry !== null);
	} catch {
		return history
			.split("\n")
			.map((line) => line.trim())
			.filter((line) => line.length > 0)
			.map((line) => {
				if (/^support(?: team)?:\s*/i.test(line)) {
					return {
						sender: "support" as const,
						body: normalizeSupportMessageBody(line),
						attachments: [],
					};
				}
				return { sender: "user" as const, body: line, attachments: [] };
			});
	}
}

function parseHistoryEntriesMessages(
	historyEntries: readonly SupportHistoryEntry[] | null | undefined,
): TicketMessage[] {
	if (!historyEntries) return [];
	return historyEntries
		.map((entry) => {
			if (!entry || typeof entry !== "object") return null;
			const sender: TicketMessage["sender"] =
				entry.sender === "support" ? "support" : "user";
			const body =
				typeof entry.message === "string" ? entry.message : String(entry.message ?? "");
			const normalizedBody =
				sender === "support" ? normalizeSupportMessageBody(body) : body;
			const attachments = Array.isArray(entry.attachments)
				? entry.attachments
						.map((a) =>
							a && typeof a.url === "string"
								? normalizeSupportAttachmentPath(a.url)
								: null,
						)
						.filter((v): v is string => Boolean(v))
				: [];
			return { sender, body: normalizedBody, attachments };
		})
		.filter((v): v is TicketMessage => v !== null);
}

/** History may echo the initial comment; we keep `ticket.message` and drop duplicates. */
function isDuplicateOfInitialUserMessage(
	initial: string,
	entry: TicketMessage,
): boolean {
	if (entry.sender !== "user") return false;
	const a = initial.trim();
	const b = entry.body.trim();
	if (a.length === 0 || b.length === 0) return false;
	if (a === b) return true;
	const withoutYouPrefix = b.replace(/^You:\s*/i, "").trim();
	return withoutYouPrefix === a;
}

function buildLegacyTicketMessages(
	initial: string,
	history: string | null,
): TicketMessage[] {
	const parsed = parseHistoryMessages(history);
	if (parsed.length === 0) {
		return [{ sender: "user", body: initial, attachments: [] }];
	}
	const first = parsed[0];
	if (!first) {
		return [{ sender: "user", body: initial, attachments: [] }];
	}
	if (
		first.sender === "user" &&
		isDuplicateOfInitialUserMessage(initial, first) &&
		first.attachments.length > 0
	) {
		const rest = parsed
			.slice(1)
			.filter((entry) => !isDuplicateOfInitialUserMessage(initial, entry));
		return [
			{ sender: "user", body: initial, attachments: first.attachments },
			...rest,
		];
	}
	const fromHistory = parsed.filter(
		(entry) => !isDuplicateOfInitialUserMessage(initial, entry),
	);
	return [{ sender: "user", body: initial, attachments: [] }, ...fromHistory];
}

/**
 * Dedupes the opening user line against `ticket.message` and prepends the
 * initial comment only when entries start with support (opening user turn
 * missing from `history_entries`).
 */
function mergeTicketMessageIntoParsedHistory(
	initial: string,
	parsed: TicketMessage[],
): TicketMessage[] {
	if (parsed.length === 0) {
		return [{ sender: "user", body: initial, attachments: [] }];
	}
	const first = parsed[0];
	if (!first) {
		return [{ sender: "user", body: initial, attachments: [] }];
	}
	if (
		first.sender === "user" &&
		isDuplicateOfInitialUserMessage(initial, first)
	) {
		return [
			{ sender: "user", body: initial, attachments: first.attachments },
			...parsed.slice(1),
		];
	}
	if (first.sender === "support") {
		return [{ sender: "user", body: initial, attachments: [] }, ...parsed];
	}
	return parsed;
}

function formatSupportTicketError(
	error: unknown,
	module: SupportModule,
	t: (key: string, vars?: Record<string, string>) => string,
): string {
	const fields = getApiErrorFields(error);

	if (fields?.status === 400) {
		const detailMsg = (fields.detailMessage ?? "").trim().toLowerCase();
		const messageMsg =
			(error instanceof Error ? error.message : "").trim().toLowerCase();
		if (
			detailMsg.includes("does not match authenticated session module") ||
			messageMsg.includes("does not match authenticated session module")
		) {
			return t("support.tickets.error.generic");
		}
	}

	if (fields?.detailCode === "AUTHENTICATION_REQUIRED") {
		return module === "pdf_anki"
			? t("support.tickets.error.signInPdfAnki")
			: t("support.tickets.error.signInReceipts");
	}

	const detailMsg = fields?.detailMessage?.trim();
	if (detailMsg) {
		const code = fields?.detailCode?.trim();
		if (code && code.length > 0) {
			return `${code}: ${detailMsg}`;
		}
		return detailMsg;
	}

	switch (fields?.status) {
		case 400:
		case 422:
			return t("support.tickets.error.badRequest");
		case 429:
			return t("support.tickets.error.rateLimited");
		case 401:
			return t("support.tickets.error.sessionExpired");
		case 403:
			return t("support.tickets.error.forbidden");
		case 404:
			return t("support.tickets.error.notFound");
		default:
			return t("support.tickets.error.generic");
	}
}

export function SupportTicketPanel({
	module,
	isOpen,
	onClose,
	isAllowedTier,
	blockedMessage,
}: SupportTicketPanelProps) {
	const closeButtonRef = useRef<HTMLButtonElement | null>(null);
	const createFileInputRef = useRef<HTMLInputElement | null>(null);
	const replyFileInputRef = useRef<HTMLInputElement | null>(null);
	const { t } = useI18n();
	const [loading, setLoading] = useState(false);
	const [error, setError] = useState<string | null>(null);
	const [success, setSuccess] = useState<string | null>(null);
	const [tickets, setTickets] = useState<ActiveTicket[]>([]);
	const [selectedTicketId, setSelectedTicketId] = useState<number | null>(null);
	const [createMessage, setCreateMessage] = useState("");
	const [replyMessage, setReplyMessage] = useState("");
	const [createAttachments, setCreateAttachments] = useState<LocalAttachment[]>([]);
	const [replyAttachments, setReplyAttachments] = useState<LocalAttachment[]>([]);
	const [previewImageUrl, setPreviewImageUrl] = useState<string | null>(null);
	const [createLoading, setCreateLoading] = useState(false);
	const [replyLoading, setReplyLoading] = useState(false);
	const [deleteConfirm, setDeleteConfirm] = useState(false);

	const [createAttachPulse, setCreateAttachPulse] = useState(false);
	const [replyAttachPulse, setReplyAttachPulse] = useState(false);
	const createAttachPulseTimerRef = useRef<number | null>(null);
	const replyAttachPulseTimerRef = useRef<number | null>(null);
	const createAttachmentsRef = useRef<LocalAttachment[]>([]);
	const replyAttachmentsRef = useRef<LocalAttachment[]>([]);

	const triggerCreateAttachPulse = (): void => {
		setCreateAttachPulse(true);
		if (createAttachPulseTimerRef.current) {
			window.clearTimeout(createAttachPulseTimerRef.current);
		}
		createAttachPulseTimerRef.current = window.setTimeout(() => {
			setCreateAttachPulse(false);
		}, 320);
	};

	const triggerReplyAttachPulse = (): void => {
		setReplyAttachPulse(true);
		if (replyAttachPulseTimerRef.current) {
			window.clearTimeout(replyAttachPulseTimerRef.current);
		}
		replyAttachPulseTimerRef.current = window.setTimeout(() => {
			setReplyAttachPulse(false);
		}, 320);
	};

	const appendAttachments = (
		current: LocalAttachment[],
		files: readonly File[],
	): LocalAttachment[] => {
		if (files.length === 0) {
			console.info("[SupportTicketPanel] Empty file selection.");
			return current;
		}
		const available = Math.max(0, MAX_TICKET_ATTACHMENTS - current.length);
		if (available === 0) {
			setError(
				`Solo puedes adjuntar hasta ${MAX_TICKET_ATTACHMENTS} imagenes por mensaje.`,
			);
			console.warn("[SupportTicketPanel] Attachment limit reached", {
				current: current.length,
				max: MAX_TICKET_ATTACHMENTS,
			});
			return current;
		}
		let rejectedByType = 0;
		let rejectedBySize = 0;
		const validFiles = files.filter((file) => {
			if (!isSupportedTicketImage(file)) {
				rejectedByType += 1;
				return false;
			}
			if (file.size > MAX_ATTACHMENT_SIZE_BYTES) {
				rejectedBySize += 1;
				return false;
			}
			return true;
		});
		const incoming = validFiles.slice(0, available).map(createLocalAttachment);
		if (incoming.length === 0) {
			if (rejectedBySize > 0) {
				setError(t("support.tickets.attachmentMaxSizeError"));
			} else if (rejectedByType > 0) {
				setError(
					"Formato no permitido. Usa jpg, jpeg, png, webp, tiff, avif, heic o heif.",
				);
			}
		}
		console.info("[SupportTicketPanel] Attachment selection processed", {
			selected: files.length,
			accepted: incoming.length,
			rejectedByType,
			rejectedBySize,
			currentCount: current.length,
			nextCount: current.length + incoming.length,
		});
		return [...current, ...incoming];
	};

	const removeAttachment = (
		attachments: LocalAttachment[],
		id: string,
	): LocalAttachment[] => {
		const target = attachments.find((item) => item.id === id);
		if (target) {
			revokeAttachment(target);
		}
		return attachments.filter((item) => item.id !== id);
	};

	const selectedTicket = useMemo(() => {
		if (selectedTicketId === null) return null;
		return tickets.find((ticket) => ticket.ticket_id === selectedTicketId) ?? null;
	}, [tickets, selectedTicketId]);

	const activeCount = useMemo(
		() => tickets.filter((ticket) => ticket.status === "active").length,
		[tickets],
	);
	const canCreate = isAllowedTier && activeCount < 3;

	const messages = useMemo(() => {
		if (!selectedTicket) return [];
		if (
			Array.isArray(selectedTicket.history_entries) &&
			selectedTicket.history_entries.length > 0
		) {
			const fromEntries = parseHistoryEntriesMessages(
				selectedTicket.history_entries,
			);
			return mergeTicketMessageIntoParsedHistory(
				selectedTicket.message,
				fromEntries,
			);
		}

		return buildLegacyTicketMessages(
			selectedTicket.message,
			selectedTicket.history,
		);
	}, [selectedTicket]);

	const lastMessage = messages[messages.length - 1] ?? null;
	const canReply =
		selectedTicket?.status === "active" && lastMessage?.sender === "support";

	const loadTickets = async (): Promise<void> => {
		setLoading(true);
		setError(null);
		try {
			const nextTickets = await fetchTickets(module);
			setTickets(nextTickets);
			setSelectedTicketId((prev) => {
				if (prev !== null && nextTickets.some((t) => t.ticket_id === prev)) {
					return prev;
				}
				return nextTickets[0]?.ticket_id ?? null;
			});
		} catch (loadError) {
			setError(formatSupportTicketError(loadError, module, t));
		} finally {
			setLoading(false);
		}
	};

	useEffect(() => {
		if (!isOpen) return;
		void loadTickets();
		setSuccess(null);
		setDeleteConfirm(false);
		setReplyMessage("");
	}, [isOpen, module]);

	useEffect(() => {
		createAttachmentsRef.current = createAttachments;
	}, [createAttachments]);

	useEffect(() => {
		replyAttachmentsRef.current = replyAttachments;
	}, [replyAttachments]);

	useEffect(() => {
		return () => {
			for (const item of createAttachmentsRef.current) revokeAttachment(item);
			for (const item of replyAttachmentsRef.current) revokeAttachment(item);
			if (createAttachPulseTimerRef.current) {
				window.clearTimeout(createAttachPulseTimerRef.current);
			}
			if (replyAttachPulseTimerRef.current) {
				window.clearTimeout(replyAttachPulseTimerRef.current);
			}
		};
	}, []);

	useEffect(() => {
		if (!isOpen) return;
		closeButtonRef.current?.focus();
	}, [isOpen]);

	useEffect(() => {
		if (!isOpen) return;
		const onKeyDown = (event: KeyboardEvent): void => {
			if (event.key === "Escape") onClose();
		};
		window.addEventListener("keydown", onKeyDown);
		return () => window.removeEventListener("keydown", onKeyDown);
	}, [isOpen, onClose]);

	useEffect(() => {
		setReplyAttachments((prev) => {
			for (const item of prev) revokeAttachment(item);
			return [];
		});
	}, [selectedTicketId]);

	if (!isOpen) return null;

	return (
		<div
			className="fixed inset-0 z-50 flex items-end justify-center bg-black/40 p-2 sm:items-center"
			role="dialog"
			aria-modal="true"
			aria-label="Panel de soporte"
		>
			<div className="w-full max-w-4xl overflow-hidden rounded-2xl border border-slate-200 bg-white shadow-xl">
				<div className="flex items-center justify-between gap-3 border-b border-slate-100 px-4 py-3">
					<div className="min-w-0">
						<h2 className="truncate text-sm font-semibold text-slate-800">
							Soporte
						</h2>
						<p className="text-[11px] text-slate-500">
							Hasta 5 tickets listados, maximo 3 activos
						</p>
					</div>
					<button
						ref={closeButtonRef}
						type="button"
						onClick={onClose}
						aria-label="Cerrar panel de soporte"
						className="rounded-full border border-slate-200 bg-slate-50 px-3 py-1 text-xs font-medium text-slate-700 hover:bg-slate-100"
					>
						Cerrar
					</button>
				</div>

				{error ? <p className="px-4 py-2 text-xs text-rose-600">{error}</p> : null}
				{success ? (
					<p className="px-4 py-2 text-xs text-emerald-700">{success}</p>
				) : null}

				<div className="grid h-[70vh] min-h-0 grid-cols-1 overflow-hidden sm:grid-cols-[1fr_1.3fr]">
					<aside className="border-b border-slate-100 sm:border-b-0 sm:border-r">
						<div className="px-4 py-3">
							<div className="flex items-center justify-between gap-3">
								<p className="text-xs font-semibold text-slate-700">Tickets</p>
								<span className="text-[11px] text-slate-500">
									{activeCount} activos
								</span>
							</div>
							<div className="mt-2 flex flex-wrap items-center gap-2">
								<input
									ref={createFileInputRef}
									type="file"
									accept=".jpg,.jpeg,.png,.webp,.tiff,.avif,.heic,.heif,image/jpeg,image/png,image/webp,image/tiff,image/avif,image/heic,image/heif"
									multiple
									className="hidden"
									onChange={(event) => {
										setError(null);
										setSuccess(null);
										const files = Array.from(event.target.files ?? []);
										console.info("[SupportTicketPanel] Create attachment picker", {
											selected: files.length,
											files: files.map((file) => ({
												name: file.name,
												type: file.type,
												size: file.size,
											})),
										});
										setCreateAttachments((prev) =>
											appendAttachments(prev, files),
										);
										event.currentTarget.value = "";
									}}
								/>
								<button
									type="button"
									disabled={!canCreate || createLoading}
								onClick={() => {
									if (!canCreate || createLoading) return;
									triggerCreateAttachPulse();
									createFileInputRef.current?.click();
								}}
								className={[
									"flex h-12 w-12 items-center justify-center rounded-md border border-dashed border-slate-400 text-lg text-slate-600 transition-all disabled:cursor-not-allowed disabled:opacity-50",
									createAttachPulse ? "scale-105 ring-2 ring-slate-900" : "",
								].join(" ")}
									aria-label="Adjuntar imagenes al ticket"
								>
									+
								</button>
								{createAttachments.map((item) => (
									<div
										key={item.id}
												className="relative h-12 w-12 overflow-hidden rounded-md border border-slate-200 bg-slate-100"
									>
										<button
											type="button"
											onClick={() => setPreviewImageUrl(item.previewUrl)}
											className="h-full w-full"
											aria-label="Ver imagen adjunta"
										>
											<img
												src={item.previewUrl}
												alt="Adjunto del ticket"
												className="h-full w-full object-cover"
												onError={() => {
													console.warn(
														"[SupportTicketPanel] Failed to load create attachment preview",
														{ src: item.previewUrl },
													);
												}}
											/>
										</button>
										<button
											type="button"
											onClick={() =>
												setCreateAttachments((prev) =>
													removeAttachment(prev, item.id),
												)
											}
											className="absolute right-0 top-0 flex h-4 w-4 items-center justify-center rounded-bl bg-black/70 text-[10px] text-white"
											aria-label="Quitar imagen"
										>
											x
										</button>
									</div>
								))}
							</div>
							<p className="mt-1 text-[11px] text-slate-500">
								{createAttachments.length}/{MAX_TICKET_ATTACHMENTS} imagenes
							</p>
							<textarea
								value={createMessage}
								onChange={(event) => setCreateMessage(event.target.value)}
								placeholder="Describe tu consulta"
								rows={3}
								disabled={!canCreate || createLoading}
								aria-label="Mensaje inicial del ticket"
								className="mt-2 w-full resize-none rounded-md border border-slate-300 bg-white px-2 py-1 text-xs shadow-sm disabled:opacity-50"
							/>
							<button
								type="button"
								disabled={!canCreate || createLoading}
								onClick={() => {
									const text = createMessage.trim();
									if (!text) return;
									if (
										!window.confirm(
											"Estas seguro de crear este ticket? Luego podras responder en el historial.",
										)
									) {
										return;
									}
									const totalBytes = createAttachments.reduce(
										(sum, item) => sum + item.file.size,
										0,
									);
									if (totalBytes > MAX_TOTAL_ATTACHMENT_SIZE_BYTES) {
										setError(t("support.tickets.attachmentTotalMaxSizeError"));
										return;
									}
									setCreateLoading(true);
									setError(null);
									setSuccess(null);
									void createTicket(
										text,
										module,
										createAttachments.map((item) => item.file),
									)
										.then(async () => {
											setCreateMessage("");
											setCreateAttachments((prev) => {
												for (const item of prev) revokeAttachment(item);
												return [];
											});
											setSuccess("Ticket creado correctamente.");
											await loadTickets();
										})
										.catch((createError: unknown) => {
											setError(
												formatSupportTicketError(createError, module, t),
											);
										})
										.finally(() => setCreateLoading(false));
								}}
								className="mt-2 w-full rounded-lg bg-slate-900 px-3 py-2 text-xs font-medium text-white disabled:cursor-not-allowed disabled:opacity-50"
							>
								{createLoading ? "Creando..." : "Crear ticket"}
							</button>
							<p className="mt-1 text-[11px] text-slate-500">
								{isAllowedTier
									? canCreate
										? "Puedes crear un nuevo ticket."
										: "Alcanzaste el maximo de tickets activos."
									: (blockedMessage ??
										"Tu plan actual no incluye soporte por tickets.")}
							</p>
						</div>
						<div className="max-h-[45vh] overflow-auto px-2 pb-2 sm:max-h-[60vh]">
							{loading ? (
								<p className="px-2 py-3 text-xs text-slate-500">
									Cargando tickets...
								</p>
							) : tickets.length === 0 ? (
								<p className="px-2 py-3 text-xs text-slate-500">
									Aun no tienes tickets.
								</p>
							) : (
								<ul className="space-y-2">
									{tickets.map((ticket) => {
										const selected = ticket.ticket_id === selectedTicketId;
										return (
											<li key={ticket.ticket_id}>
												<button
													type="button"
													onClick={() => {
														setSelectedTicketId(ticket.ticket_id);
														setDeleteConfirm(false);
													}}
													aria-label={`Seleccionar ticket ${ticket.ticket_id}`}
													className={[
														"w-full rounded-xl border px-3 py-2 text-left",
														selected
															? "border-slate-900 bg-slate-50"
															: "border-slate-200 bg-white hover:bg-slate-50",
													].join(" ")}
												>
													<div className="flex items-center justify-between">
														<p className="truncate text-xs font-semibold text-slate-800">
															#{ticket.ticket_id}
														</p>
														<span className="text-[11px] text-slate-500">
															{ticket.status}
														</span>
													</div>
													<p className="mt-1 line-clamp-2 text-[11px] text-slate-600">
														{ticket.message}
													</p>
												</button>
											</li>
										);
									})}
								</ul>
							)}
						</div>
					</aside>
					<section className="flex min-h-0 flex-col overflow-hidden">
						<div className="border-b border-slate-100 px-4 py-3 text-xs font-semibold text-slate-700">
							Historial
						</div>
						<div className="min-h-0 flex-1 overflow-y-auto px-4 py-3">
							{selectedTicket ? (
								<div className="space-y-3">
									<div>
										<p className="text-xs font-semibold text-slate-800">
											Ticket #{selectedTicket.ticket_id}
										</p>
										<p className="mt-1 text-[11px] text-slate-500">
											{selectedTicket.status} - {messages.length} mensajes
										</p>
									</div>
									<ul className="space-y-2">
										{messages.map((message, index) => (
											<li
												key={`${message.sender}-${index}`}
												className={
													message.sender === "user"
														? "rounded-xl bg-slate-900 p-2 text-white"
														: "rounded-xl border border-slate-200 bg-white p-2 text-slate-800"
												}
											>
												<p className="text-[11px] font-semibold">
													{message.sender === "user"
														? "Tu"
														: "Equipo de Soporte Tecnico"}
												</p>
												<p className="mt-1 whitespace-pre-wrap text-xs">
													{message.body}
												</p>
												{message.attachments.length > 0 ? (
													<div className="mt-2 flex flex-wrap gap-2">
														{message.attachments.map((url, attachmentIndex) => (
															<button
																type="button"
																key={`${url}-${attachmentIndex}`}
																onClick={() => setPreviewImageUrl(url)}
																className="h-[30px] w-[30px] overflow-hidden rounded-md border border-slate-300 bg-slate-100"
																aria-label="Ampliar imagen del historial"
															>
																<img
																	src={resolveSupportAttachmentSrc(url)}
																	alt="Imagen adjunta en historial"
																	className="h-full w-full object-cover"
													onError={() => {
														console.warn(
															"[SupportTicketPanel] Failed to load history attachment",
															{ src: url },
														);
													}}
																/>
															</button>
														))}
													</div>
												) : null}
											</li>
										))}
									</ul>
								</div>
							) : (
								<p className="text-xs text-slate-500">
									Selecciona un ticket para ver el historial.
								</p>
							)}
						</div>
						<div className="border-t border-slate-100 px-4 py-3">
							<div className="flex items-start justify-between gap-3">
								<p className="text-[11px] text-slate-500">
									{canReply
										? "Puedes responder ahora."
										: "Solo puedes responder cuando soporte responda primero."}
								</p>
								<div className="flex flex-wrap items-center justify-end gap-2">
									<input
										ref={replyFileInputRef}
										type="file"
										accept=".jpg,.jpeg,.png,.webp,.tiff,.avif,.heic,.heif,image/jpeg,image/png,image/webp,image/tiff,image/avif,image/heic,image/heif"
										multiple
										className="hidden"
										onChange={(event) => {
											setError(null);
											setSuccess(null);
											const files = Array.from(event.target.files ?? []);
											console.info("[SupportTicketPanel] Reply attachment picker", {
												selected: files.length,
												files: files.map((file) => ({
													name: file.name,
													type: file.type,
													size: file.size,
												})),
											});
											setReplyAttachments((prev) =>
												appendAttachments(prev, files),
											);
											event.currentTarget.value = "";
										}}
									/>
									<button
										type="button"
										disabled={!selectedTicket || !canReply || replyLoading}
								onClick={() => {
									if (!selectedTicket || !canReply || replyLoading) return;
									triggerReplyAttachPulse();
									replyFileInputRef.current?.click();
								}}
								className={[
									"flex h-12 w-12 items-center justify-center rounded-md border border-dashed border-slate-400 text-base text-slate-600 transition-all disabled:cursor-not-allowed disabled:opacity-50",
									replyAttachPulse ? "scale-105 ring-2 ring-slate-900" : "",
								].join(" ")}
										aria-label="Adjuntar imagenes a la respuesta"
									>
										+
									</button>
									{replyAttachments.map((item) => (
										<div
											key={item.id}
											className="relative h-12 w-12 overflow-hidden rounded-md border border-slate-200 bg-slate-100"
										>
											<button
												type="button"
												onClick={() => setPreviewImageUrl(item.previewUrl)}
												className="h-full w-full"
												aria-label="Ver imagen adjunta"
											>
												<img
													src={item.previewUrl}
													alt="Adjunto de respuesta"
													className="h-full w-full object-cover"
													onError={() => {
														console.warn(
															"[SupportTicketPanel] Failed to load reply attachment preview",
															{ src: item.previewUrl },
														);
													}}
												/>
											</button>
											<button
												type="button"
												onClick={() =>
													setReplyAttachments((prev) =>
														removeAttachment(prev, item.id),
													)
												}
												className="absolute right-0 top-0 flex h-4 w-4 items-center justify-center rounded-bl bg-black/70 text-[10px] text-white"
												aria-label="Quitar imagen"
											>
												x
											</button>
										</div>
									))}
								</div>
							</div>
							<p className="mt-1 text-[11px] text-slate-500">
								{replyAttachments.length}/{MAX_TICKET_ATTACHMENTS} imagenes
							</p>
							<textarea
								value={replyMessage}
								onChange={(event) => setReplyMessage(event.target.value)}
								placeholder="Escribe tu respuesta"
								rows={3}
								aria-label="Respuesta del ticket"
								disabled={!selectedTicket || !canReply || replyLoading}
								className="mt-2 w-full resize-none rounded-md border border-slate-300 bg-white px-2 py-1 text-xs shadow-sm disabled:opacity-50"
							/>
							<div className="mt-2 flex items-center justify-between gap-3">
								<button
									type="button"
									disabled={!selectedTicket || !canReply || replyLoading}
									onClick={() => {
										if (!selectedTicket) return;
										const text = replyMessage.trim();
										if (!text || !canReply) return;
										const totalBytes = replyAttachments.reduce(
											(sum, item) => sum + item.file.size,
											0,
										);
										if (totalBytes > MAX_TOTAL_ATTACHMENT_SIZE_BYTES) {
											setError(t("support.tickets.attachmentTotalMaxSizeError"));
											return;
										}
										setReplyLoading(true);
										setError(null);
										setSuccess(null);
										void replyToTicket(
											selectedTicket.ticket_id,
											text,
											replyAttachments.map((item) => item.file),
											module,
										)
											.then(async () => {
												setReplyMessage("");
												setReplyAttachments((prev) => {
													for (const item of prev) revokeAttachment(item);
													return [];
												});
												setSuccess("Respuesta enviada.");
												await loadTickets();
											})
											.catch((replyError: unknown) => {
												setError(
													formatSupportTicketError(replyError, module, t),
												);
											})
											.finally(() => setReplyLoading(false));
									}}
									className="rounded-lg bg-slate-900 px-3 py-2 text-xs font-medium text-white disabled:cursor-not-allowed disabled:opacity-50"
								>
									{replyLoading ? "Enviando..." : "Enviar respuesta"}
								</button>

								{deleteConfirm ? (
									<div className="flex items-center gap-2">
										<button
											type="button"
											onClick={() => setDeleteConfirm(false)}
											className="rounded-lg border border-slate-300 px-3 py-2 text-xs font-medium text-slate-700"
										>
											Cancelar
										</button>
										<button
											type="button"
											disabled={!selectedTicket}
											onClick={() => {
												if (!selectedTicket) return;
												setError(null);
												setSuccess(null);
												void deleteTicket(
													selectedTicket.ticket_id,
													module,
												)
													.then(async () => {
														setSuccess("Ticket eliminado.");
														setDeleteConfirm(false);
														await loadTickets();
													})
													.catch((deleteError: unknown) => {
														setError(
															formatSupportTicketError(
																deleteError,
																module,
																t,
															),
														);
													});
											}}
											className="rounded-lg border border-rose-200 bg-rose-50 px-3 py-2 text-xs font-medium text-rose-700 disabled:cursor-not-allowed disabled:opacity-50"
										>
											Confirmar
										</button>
									</div>
								) : (
									<button
										type="button"
										disabled={!selectedTicket}
										onClick={() => setDeleteConfirm(true)}
										className="rounded-lg border border-rose-200 bg-rose-50 px-3 py-2 text-xs font-medium text-rose-700 disabled:cursor-not-allowed disabled:opacity-50"
									>
										Eliminar
									</button>
								)}
							</div>
						</div>
					</section>
				</div>
			</div>
			{previewImageUrl ? (
				<div
					className="fixed inset-0 z-[60] flex items-center justify-center bg-black/70 p-4"
					role="dialog"
					aria-modal="true"
					aria-label="Vista ampliada de imagen"
				>
					<div className="relative max-h-[85vh] max-w-[85vw]">
						<button
							type="button"
							onClick={() => setPreviewImageUrl(null)}
							className="absolute right-2 top-2 z-10 rounded-full bg-black/60 px-2 py-1 text-xs font-medium text-white"
						>
							Cerrar
						</button>
						<img
							src={resolveSupportAttachmentSrc(previewImageUrl)}
							alt="Imagen ampliada"
							className="max-h-[85vh] max-w-[85vw] rounded-lg object-contain"
							onError={() => {
								console.warn(
									"[SupportTicketPanel] Failed to load full-screen attachment",
									{ src: previewImageUrl },
								);
							}}
						/>
					</div>
				</div>
			) : null}
		</div>
	);
}
```

### `src/components/VideoModal.tsx`
```tsx
"use client";

import { useCallback, useEffect, useRef, useState } from "react";
import { Captions, CaptionsOff, X } from "lucide-react";

interface VideoModalProps {
		isOpen: boolean;
		youtubeId: string | null;
		title: string;
		onClose: () => void;
}

const CAPTIONS_STORAGE_KEY = "genui-video-captions-enabled";
const CAPTIONS_LANG = "es";

function readStoredCaptionsPreference(): boolean {
		if (typeof window === "undefined") return true;
		const stored = window.localStorage.getItem(CAPTIONS_STORAGE_KEY);
		if (stored === null) return true;
		return stored === "1";
}

export function VideoModal({ isOpen, youtubeId, title, onClose }: VideoModalProps) {
		const iframeRef = useRef<HTMLIFrameElement | null>(null);
		const [captionsEnabled, setCaptionsEnabled] = useState<boolean>(true);

		useEffect(() => {
				if (!isOpen) return;

				setCaptionsEnabled(readStoredCaptionsPreference());

				const handleKeyDown = (e: KeyboardEvent) => {
						if (e.key === "Escape") {
								onClose();
						}
				};

				document.addEventListener("keydown", handleKeyDown);
				document.body.style.overflow = "hidden";

				return () => {
						document.removeEventListener("keydown", handleKeyDown);
						document.body.style.overflow = "unset";
				};
		},[isOpen, onClose]);

		const sendCaptionCommand = useCallback((enable: boolean) => {
				const iframe = iframeRef.current;
				if (!iframe?.contentWindow) return;
				const message = JSON.stringify({
						event: "command",
						func: enable ? "loadModule" : "unloadModule",
						args: ["captions"],
				});
				iframe.contentWindow.postMessage(message, "*");
		}, []);

		const handleIframeLoad = useCallback(() => {

				if (!captionsEnabled) {
						sendCaptionCommand(false);
				}
		},[captionsEnabled, sendCaptionCommand]);

		const toggleCaptions = useCallback(() => {
				setCaptionsEnabled((prev) => {
						const next = !prev;
						if (typeof window !== "undefined") {
								window.localStorage.setItem(CAPTIONS_STORAGE_KEY, next ? "1" : "0");
						}
						sendCaptionCommand(next);
						return next;
				});
		},[sendCaptionCommand]);

		if (!isOpen || !youtubeId) return null;

		const embedParams = new URLSearchParams({
				autoplay: "1",
				rel: "0",
				modestbranding: "1",
				enablejsapi: "1",
				cc_load_policy: "1",
				cc_lang_pref: CAPTIONS_LANG,
				hl: CAPTIONS_LANG,
		});

		return (
				<div
						className="fixed inset-0 z-[100] flex items-center justify-center bg-slate-950/80 p-4 backdrop-blur-sm transition-opacity"
						role="dialog"
						aria-modal="true"
						aria-labelledby="video-modal-title"
						onClick={onClose}
				>
						<div
								className="relative w-full max-w-5xl overflow-hidden rounded-2xl bg-black shadow-2xl ring-1 ring-white/10"
								onClick={(e) => e.stopPropagation()}
						>
								{/* Header */}
								<div className="absolute left-0 right-0 top-0 z-10 flex items-center justify-between bg-gradient-to-b from-black/80 to-transparent p-4">
										<h2 id="video-modal-title" className="text-sm font-medium text-white drop-shadow-md">
												{title} - Architecture Deep Dive
										</h2>
										<div className="flex items-center gap-2">
												<button
														type="button"
														onClick={toggleCaptions}
														className="rounded-full bg-black/50 p-2 text-white backdrop-blur-md transition hover:bg-white/20"
														aria-label={captionsEnabled ? "Desactivar subtítulos" : "Activar subtítulos"}
														aria-pressed={captionsEnabled}
														title={captionsEnabled ? "Subtítulos: ON" : "Subtítulos: OFF"}
												>
														{captionsEnabled ? (
																<Captions className="h-5 w-5" />
														) : (
																<CaptionsOff className="h-5 w-5" />
														)}
												</button>
												<button
														type="button"
														onClick={onClose}
														className="rounded-full bg-black/50 p-2 text-white backdrop-blur-md transition hover:bg-white/20"
														aria-label="Close video"
												>
														<X className="h-5 w-5" />
												</button>
										</div>
								</div>

								{/* 16:9 Aspect Ratio Container */}
								<div className="relative pt-[56.25%]">
										<iframe
												ref={iframeRef}
												onLoad={handleIframeLoad}
												className="absolute inset-0 h-full w-full border-0"
												src={`https://www.youtube.com/embed/${youtubeId}?${embedParams.toString()}`}
												title={`${title} Demo Video`}
												allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
												allowFullScreen
										/>
								</div>
						</div>
				</div>
		);
}
```

### `src/lib/api-client.ts`
```ts
export type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";

export type ApiClientConfig = {
	baseUrl?: string;
	/** Default request deadline when `options.timeoutMs` is omitted. */
	timeoutMs?: number;
};

const DEFAULT_TIMEOUT_MS = 15000;

/** Single source of truth for `apiRequest` defaults (per-request options override). */
const defaultConfig: Required<Pick<ApiClientConfig, "baseUrl" | "timeoutMs">> = {

	baseUrl:
		process.env.NEXT_PUBLIC_BACKEND_URL ?? "/api/v1",
	timeoutMs: DEFAULT_TIMEOUT_MS,
};

export type ApiErrorOptions = {
	detailCode?: string;
	detailMessage?: string;
	body?: unknown;
};

export class ApiError extends Error {
	public readonly status: number;

	public readonly detailCode?: string;

	public readonly detailMessage?: string;

	public readonly body?: unknown;

	public constructor(message: string, status: number, options?: ApiErrorOptions) {
		super(message);
		this.name = "ApiError";
		this.status = status;
		if (options) {
			this.detailCode = options.detailCode;
			this.detailMessage = options.detailMessage;
			this.body = options.body;
		}
	}
}

type AbortSignalAny = typeof AbortSignal & {
	any?: (signals: AbortSignal[]) => AbortSignal;
};

export function isAbortError(error: unknown): boolean {
	if (typeof DOMException !== "undefined" && error instanceof DOMException) {
		return error.name === "AbortError";
	}
	if (!error || typeof error !== "object") {
		return false;
	}
	const e = error as { name?: unknown };
	return e.name === "AbortError";
}

function composeAbortSignal(
	externalSignal: AbortSignal | undefined,
	timeoutMs: number,
): { signal: AbortSignal; cleanup: () => void } {
	const timeoutController = new AbortController();
	const timeoutId = setTimeout(() => {
		timeoutController.abort();
	}, timeoutMs);

	const anySignal = (AbortSignal as AbortSignalAny | undefined)?.any;
	if (typeof anySignal === "function") {
		const signals = externalSignal
			? [externalSignal, timeoutController.signal]
			: [timeoutController.signal];
		return {
			signal: anySignal(signals),
			cleanup: () => {
				clearTimeout(timeoutId);
			},
		};
	}

	if (!externalSignal) {
		return {
			signal: timeoutController.signal,
			cleanup: () => {
				clearTimeout(timeoutId);
			},
		};
	}

	const controller = new AbortController();
	const abort = () => {
		if (!controller.signal.aborted) {
			controller.abort();
		}
	};
	const onExternalAbort = () => {
		abort();
	};
	const onTimeoutAbort = () => {
		abort();
	};

	if (externalSignal.aborted) {
		abort();
	} else {
		externalSignal.addEventListener("abort", onExternalAbort, { once: true });
	}
	timeoutController.signal.addEventListener("abort", onTimeoutAbort, {
		once: true,
	});

	return {
		signal: controller.signal,
		cleanup: () => {
			clearTimeout(timeoutId);
			externalSignal.removeEventListener("abort", onExternalAbort);
			timeoutController.signal.removeEventListener("abort", onTimeoutAbort);
		},
	};
}

/**
 * Reads status / structured detail from an error. Use when `instanceof ApiError`
 * may fail (e.g. duplicate module instances in Jest).
 */
export type ApiErrorFields = {
	status: number;
	detailCode?: string;
	detailMessage?: string;
};

export function getApiErrorFields(error: unknown): ApiErrorFields | null {
	if (error instanceof ApiError) {
		const detailMessage =
			error.detailMessage ??
			(error.status === 400 || error.status === 422
				? error.message
				: undefined);
		return {
			status: error.status,
			detailCode: error.detailCode,
			detailMessage,
		};
	}
	if (!error || typeof error !== "object") {
		return null;
	}
	const e = error as Record<string, unknown>;
	if (typeof e.status !== "number") {
		return null;
	}
	const detailCode =
		typeof e.detailCode === "string" ? e.detailCode : undefined;
	const detailMessage =
		typeof e.detailMessage === "string" ? e.detailMessage : undefined;
	return { status: e.status, detailCode, detailMessage };
}

/** Builds `ApiError` from a parsed JSON error body (shared by fetch wrappers). */
export function apiErrorFromJsonBody(status: number, rawBody: unknown): ApiError {
	let stringDetail: string | undefined;
	let detailCode: string | undefined;

	if (rawBody && typeof rawBody === "object" && !Array.isArray(rawBody)) {
		const body = rawBody as Record<string, unknown>;
		if (typeof body.detail === "string") {
			stringDetail = body.detail;
		} else if (typeof body.message === "string") {
			stringDetail = body.message;
		} else if (typeof body.detail === "object" && body.detail !== null) {
			const d = body.detail as Record<string, unknown>;
			if (typeof d.message === "string") stringDetail = d.message;
			if (typeof d.code === "string") detailCode = d.code;
		}
	} else if (typeof rawBody === "string" && rawBody.trim()) {
		stringDetail = rawBody;
	}

	const message = stringDetail ?? `API request failed with status ${status}`;
	return new ApiError(message, status, {
		detailCode,
		detailMessage: stringDetail,
		body: rawBody,
	});
}

type ApiRequestOptions<TBody> = {
	method?: HttpMethod;
	body?: TBody;
	signal?: AbortSignal;
	retries?: number;
	headers?: Record<string, string>;
	/** Client abort if the request runs longer than this (default 15000). */
	timeoutMs?: number;
};

export async function apiRequest<TResponse, TBody = unknown>(
	path: string,
	options: ApiRequestOptions<TBody> = {}
): Promise<TResponse> {
	const config = defaultConfig;
	const {
		method,
		body,
		signal,
		retries = 0,
		headers: customHeaders,
	} = options;

	const deadline = options.timeoutMs ?? defaultConfig.timeoutMs;

	const isAbsoluteUrl =
		path.startsWith("http://") || path.startsWith("https://");

	const normalizedBaseUrl = config.baseUrl.endsWith("/")
		? config.baseUrl
		: `${config.baseUrl}/`;

	const normalizedPath = path.replace(/^\/+/, "");

	const urlStr = isAbsoluteUrl
		? path
		: normalizedBaseUrl.startsWith("http")
			? new URL(normalizedPath, normalizedBaseUrl).toString()
			: `${normalizedBaseUrl}${normalizedPath}`;

	const headers: Record<string, string> = {
		...(customHeaders ?? {})
	};

	let requestBody: BodyInit | undefined;
	if (body !== undefined) {
		const isFormData =
			typeof FormData !== "undefined" && body instanceof FormData;

		if (isFormData) {
			requestBody = body as unknown as FormData;
		} else {
			headers["Content-Type"] = "application/json";
			requestBody = JSON.stringify(body) as BodyInit;
		}
	}

	const { signal: effectiveSignal, cleanup } = composeAbortSignal(
		signal,
		deadline,
	);

	try {
		let lastError: unknown;

		for (let attempt = 0; attempt <= retries; attempt += 1) {
			try {

				const response = await fetch(urlStr, {
					method: method ?? "GET",
					headers,
					body: requestBody,
					credentials: "include",
					signal: effectiveSignal
				});

				if (!response.ok) {
					let rawBody: unknown;
					try {
						rawBody = await response.json();
					} catch {
						rawBody = null;
					}
					const apiErr = apiErrorFromJsonBody(response.status, rawBody);
					const retryable =
						response.status === 429 || response.status >= 500;

					if (attempt < retries && retryable) {
						lastError = apiErr;
						await new Promise((resolve) => {
							setTimeout(resolve, Math.min(1000 * 2 ** attempt, 10000));
						});
						continue;
					}

					throw apiErr;
				}

				const json = (await response.json()) as TResponse;
				return json;
			} catch (error) {
				lastError = error;
				if (isAbortError(error)) {
					throw error;
				}
				if (error instanceof ApiError) {
					throw error;
				}
				if (attempt < retries) {
					await new Promise((resolve) => {
						setTimeout(resolve, 200 * (attempt + 1));
					});

					continue;
				}
				throw error;
			}
		}

		throw lastError as Error;
	} finally {
		cleanup();
	}
}
```

### `src/lib/api-error-detail.ts`
```ts
/**
 * Parses FastAPI-style JSON error bodies where `detail` may be a string,
 * a validation array, or `{ code, message }` (OAuth / product-scoped JWT errors).
 */

export type StructuredApiDetail = {
	code: string;
	message: string;
};

function coalesceStructured(
	code: string | undefined,
	message: string | undefined,
): StructuredApiDetail | null {
	const c = code?.trim() ?? "";
	const m = message?.trim() ?? "";
	if (c && m) {
		return { code: c, message: m };
	}
	if (c && !m) {
		return { code: c, message: c };
	}
	if (!c && m) {
		return { code: "UNKNOWN", message: m };
	}
	return null;
}

export function parseStructuredDetailField(detail: unknown): StructuredApiDetail | null {
	if (!detail || typeof detail !== "object" || Array.isArray(detail)) {
		return null;
	}
	const d = detail as Record<string, unknown>;
	const code = typeof d.code === "string" ? d.code : undefined;
	const message = typeof d.message === "string" ? d.message : undefined;
	return coalesceStructured(code, message);
}

function fastApiArrayDetailToString(detail: unknown[]): string | undefined {
	const parts = detail
		.map((item) => {
			if (!item || typeof item !== "object") return null;
			const msg = (item as { msg?: unknown }).msg;
			return typeof msg === "string" ? msg : null;
		})
		.filter((x): x is string => x != null && x.length > 0);
	return parts.length ? parts.join("; ") : undefined;
}

/**
 * Reads `detail` and optional top-level `message` from a typical FastAPI JSON
 * error body.
 */
export function extractErrorPayloadFromJson(raw: unknown): {
	structured: StructuredApiDetail | null;
	stringDetail?: string;
} {
	if (!raw || typeof raw !== "object") {
		return { structured: null };
	}
	const o = raw as Record<string, unknown>;
	const detail = o.detail;

	const structured = parseStructuredDetailField(detail);
	if (structured) {
		return { structured };
	}

	if (typeof detail === "string" && detail.trim()) {
		return { structured: null, stringDetail: detail };
	}

	if (Array.isArray(detail)) {
		const joined = fastApiArrayDetailToString(detail);
		if (joined) {
			return { structured: null, stringDetail: joined };
		}
	}

	if (detail !== null && typeof detail === "object" && !Array.isArray(detail)) {
		try {
			const raw = JSON.stringify(detail);
			if (raw.length > 0 && raw !== "{}") {
				return { structured: null, stringDetail: raw };
			}
		} catch {
			/* ignore */
		}
	}

	if (typeof o.message === "string" && o.message.trim()) {
		return { structured: null, stringDetail: o.message };
	}

	return { structured: null };
}
```

### `src/lib/image-hash.ts`
```ts
/**
 * Computes SHA-256 hash of the image bytes represented by a base64 string.
 * Used for duplicate detection when sending receipts to the backend.
 */
export async function sha256HexFromBase64(base64: string): Promise<string> {
	const binary = atob(base64);
	const bytes = new Uint8Array(binary.length);
	for (let i = 0; i < binary.length; i += 1) {
		bytes[i] = binary.charCodeAt(i);
	}
	const hashBuffer = await crypto.subtle.digest("SHA-256", bytes);
	const hashArray = Array.from(new Uint8Array(hashBuffer));
	return hashArray.map((b) => b.toString(16).padStart(2, "0")).join("");
}
```

### `src/lib/module-auth-required-error.ts`
```ts
/** Thrown when a module-scoped API call is attempted without a Bearer token. */
export class ModuleAuthRequiredError extends Error {
	public constructor(message = "MODULE_AUTH_REQUIRED") {
		super(message);
		this.name = "ModuleAuthRequiredError";
	}
}
```

### `src/lib/stripe-utils.ts`
```ts
export function personalizeStripeLink(
	baseUrl: string | undefined | null,
	userId: string | undefined | null,
	locale: string,
	tier: string,
	module: "pdf_anki" | "receipt_parser"
): string {
	if (!baseUrl || baseUrl.includes("replace-with-real-link-later")) {
		return baseUrl ?? "";
	}
	try {
		const url = new URL(baseUrl);
		const userIdParam =
			module === "pdf_anki" ? "pdf_anki_user_id" : "receipt_parser_user_id";
		const uid = userId || "anonymous";

		url.searchParams.set(userIdParam, uid);
		url.searchParams.set("language_code", locale);
		url.searchParams.set("tier", tier);

		url.searchParams.set("client_reference_id", uid);

		return url.toString();
	} catch {
		return baseUrl;
	}
}
```

### `src/lib/support-api.ts`
```ts
import { apiRequest, getApiErrorFields } from "@/lib/api-client";

/** Use only `pdf_anki` or `receipt_parser` in API queries (underscore), not URL slugs. */
export type SupportModule = "receipt_parser" | "pdf_anki";
export type SupportTicketStatus = "active" | "closed";
export const MAX_TICKET_ATTACHMENTS = 10;

export type SupportAttachment = {
	url: string;
	filename?: string;
	content_type?: string;
	size_bytes?: number;
	created_at?: string;
};

export type SupportHistoryEntry = {
	sender: "user" | "support";
	message: string;
	attachments?: readonly SupportAttachment[];
};

export type ActiveTicket = {
	ticket_id: number;
	user_id: string;
	message: string;
	status: SupportTicketStatus;
	created_at: string;
	history: string | null;
	history_entries?: readonly SupportHistoryEntry[];
	module: "pdf_anki" | "receipt_parser" | "audio_notes";
};

export type TicketReplyRequest = {
	message: string;
	close_ticket: boolean;
};

type FetchTicketsResponse = {
	tickets: ActiveTicket[];
};

function normalizeSupportText(input: string): string {
	return input.trim().replaceAll("\u0000", "");
}

function truncateSupportText(input: string, maxLen: number): string {
	if (input.length <= maxLen) return input;
	return input.slice(0, maxLen);
}

const ENDPOINTS = {
	fetchTickets: "/support/tickets",
	createTicket: "/support/tickets",
	replyToTicket: "/support/tickets/{ticket_id}/reply",
	deleteTicket: "/support/tickets/{ticket_id}",
	getAdminTickets: "/admin/tickets",
	getAdminTicket: "/admin/tickets/{ticket_id}",
	replyToAdminTicket: "/admin/tickets/{ticket_id}/reply",
} as const;

/**
 * `pdf_anki` must never be aliased. For receipts, some backends accept `receipts`
 * as the query value — try that only after `receipt_parser` fails.
 */
function moduleQueryCandidates(module: SupportModule): readonly string[] {
	if (module === "receipt_parser") {
		return ["receipt_parser", "receipts"];
	}
	return ["pdf_anki"];
}

function shouldTryNextReceiptModuleAlias(error: unknown): boolean {
	const fields = getApiErrorFields(error);
	if (!fields) return true;
	if (fields.status === 429) return false;
	return fields.status === 400 || fields.status === 404 || fields.status === 422;
}

async function withSupportModuleQueryRetry<T>(
	module: SupportModule,
	execute: (moduleParam: string) => Promise<T>,
): Promise<T> {
	const candidates = moduleQueryCandidates(module);
	let lastError: unknown = null;
	for (let i = 0; i < candidates.length; i += 1) {
		const param = candidates[i]!;
		try {
			return await execute(param);
		} catch (error) {
			lastError = error;
			const hasNext = i < candidates.length - 1;
			if (!hasNext || !shouldTryNextReceiptModuleAlias(error)) {
				throw error;
			}
		}
	}
	throw lastError;
}

/**
 * Skips empty files and entries without a filename so `FormData` does not send
 * bogus `attachments` parts (Firefox / HEIC can still infer MIME on the server).
 */
export function filterValidSupportAttachmentFiles(
	files: readonly File[],
): File[] {
	const max = MAX_TICKET_ATTACHMENTS;
	const out: File[] = [];
	for (const file of files) {
		if (out.length >= max) break;
		if (file.size <= 0) continue;
		if (!file.name || file.name.trim().length === 0) continue;
		out.push(file);
	}
	return out;
}

export async function fetchTickets(module: SupportModule): Promise<ActiveTicket[]> {
	return withSupportModuleQueryRetry(module, async (moduleParam) => {
		const qp = new URLSearchParams({ module: moduleParam }).toString();
		const response = await apiRequest<FetchTicketsResponse>(
			`${ENDPOINTS.fetchTickets}?${qp}`,
			{
				method: "GET",
			},
		);
		return response.tickets;
	});
}

export async function createTicket(
	message: string,
	module: SupportModule,
	attachments: readonly File[] = [],
): Promise<void> {
	const normalizedMessage = truncateSupportText(normalizeSupportText(message), 4000);
	const files = filterValidSupportAttachmentFiles(attachments);
	await withSupportModuleQueryRetry(module, async (moduleParam) => {
		const qs = new URLSearchParams({ module: moduleParam }).toString();
		if (files.length > 0) {
			const formData = new FormData();
			formData.append("message", normalizedMessage);
			for (const file of files) {
				formData.append("attachments", file);
			}
			await apiRequest<unknown>(`${ENDPOINTS.createTicket}?${qs}`, {
				method: "POST",
				body: formData,
			});
			return;
		}
		await apiRequest<unknown>(`${ENDPOINTS.createTicket}?${qs}`, {
			method: "POST",
			body: {
				message: normalizedMessage,
			},
		});
	});
}

export async function replyToTicket(
	ticketId: number,
	message: string,
	attachments: readonly File[] = [],
	module: SupportModule,
): Promise<void> {
	const base = ENDPOINTS.replyToTicket.replace("{ticket_id}", String(ticketId));
	const qs = new URLSearchParams({ module }).toString();
	const path = `${base}?${qs}`;
	const normalizedMessage = truncateSupportText(normalizeSupportText(message), 4000);
	const files = filterValidSupportAttachmentFiles(attachments);
	if (files.length > 0) {
		const formData = new FormData();
		formData.append("message", normalizedMessage);
		for (const file of files) {
			formData.append("attachments", file);
		}
		await apiRequest<unknown>(path, {
			method: "POST",
			body: formData,
		});
		return;
	}
	await apiRequest<unknown>(path, {
		method: "POST",
		body: {
			message: normalizedMessage,
		},
	});
}

export async function deleteTicket(
	ticketId: number,
	module: SupportModule,
): Promise<void> {
	const base = ENDPOINTS.deleteTicket.replace("{ticket_id}", String(ticketId));
	const qs = new URLSearchParams({ module }).toString();
	await apiRequest<unknown>(`${base}?${qs}`, {
		method: "DELETE",
	});
}

export async function getAdminTickets(limit = 500): Promise<ActiveTicket[]> {
	const qs = new URLSearchParams({ limit: String(limit) }).toString();
	return apiRequest<ActiveTicket[]>(`${ENDPOINTS.getAdminTickets}?${qs}`, {
		method: "GET",
	});
}

export async function getAdminTicket(ticketId: number): Promise<ActiveTicket> {
	const path = ENDPOINTS.getAdminTicket.replace("{ticket_id}", String(ticketId));
	return apiRequest<ActiveTicket>(path, {
		method: "GET",
	});
}

export async function replyToAdminTicket(
	ticketId: number,
	body: TicketReplyRequest,
): Promise<{ status: string }> {
	const path = ENDPOINTS.replyToAdminTicket.replace(
		"{ticket_id}",
		String(ticketId),
	);
	return apiRequest<{ status: string }, TicketReplyRequest>(path, {
		method: "POST",
		body: {
			message: truncateSupportText(normalizeSupportText(body.message), 4000),
			close_ticket: body.close_ticket,
		},
	});
}
```

### `src/lib/support-attachment-url.ts`
```ts
/**
 * Support ticket attachment URLs are often backend-root paths (e.g.
 * `/support-attachments/...`) while API calls use `NEXT_PUBLIC_BACKEND_URL`
 * (often `.../api/v1`). Browsers resolve relative `/...` against the Next.js
 * origin, which 404s without a rewrite. These helpers attach the real backend
 * origin when needed.
 */

function stripTrailingSlashes(input: string): string {
	return input.replace(/\/+$/, "");
}

/**
 * Returns the backend origin (and optional base path) used to load files
 * outside `/api/v1`, without a trailing slash.
 *
 * - `NEXT_PUBLIC_BACKEND_ORIGIN` wins when set (e.g. `http://localhost:8000`).
 * - Otherwise, if `NEXT_PUBLIC_BACKEND_URL` is absolute, strip a trailing
 *   `/api/v1` path segment from its pathname.
 * - For relative `NEXT_PUBLIC_BACKEND_URL` (e.g. `/api/v1`), returns "" so
 *   callers keep same-origin paths (add a dev rewrite or set
 *   `NEXT_PUBLIC_BACKEND_ORIGIN`).
 */
export function getBackendOriginForSupportFiles(): string {
	const explicit = process.env.NEXT_PUBLIC_BACKEND_ORIGIN?.trim();
	if (explicit && explicit.length > 0) {
		return stripTrailingSlashes(explicit);
	}

	const raw = process.env.NEXT_PUBLIC_BACKEND_URL?.trim() ?? "";
	if (!/^https?:\/\//i.test(raw)) {
		return "";
	}

	try {
		const parsed = new URL(raw);
		let pathname = parsed.pathname.replace(/\/+$/, "") || "";
		if (pathname.endsWith("/api/v1")) {
			pathname = pathname.slice(0, -"/api/v1".length);
		}
		const basePath =
			pathname.length > 0 && pathname !== "/" ? pathname : "";
		return stripTrailingSlashes(`${parsed.origin}${basePath}`);
	} catch {
		return "";
	}
}

/** Normalizes stored attachment URL to a path or absolute URL string. */
export function normalizeSupportAttachmentPath(url: string): string {
	let next = url.trim();
	if (next.startsWith("[TBD]")) {
		next = next.replace(/^\[TBD\]/i, "").trim();
	}
	if (next.length === 0) return next;
	if (/^(https?:|blob:|data:)/i.test(next)) return next;
	if (next.startsWith("/")) return next;
	return `/${next}`;
}

/**
 * Builds a value suitable for `<img src>` / preview modal.
 * Absolute `http(s)` URLs are unchanged. Relative paths use
 * `getBackendOriginForSupportFiles()` when set; otherwise stay relative
 * (same origin as the page).
 */
export function resolveSupportAttachmentSrc(url: string): string {
	const pathOrAbsolute = normalizeSupportAttachmentPath(url);
	if (!pathOrAbsolute) return pathOrAbsolute;
	if (/^(https?:|blob:|data:)/i.test(pathOrAbsolute)) return pathOrAbsolute;
	const origin = getBackendOriginForSupportFiles();
	if (!origin) return pathOrAbsolute;
	return `${origin}${pathOrAbsolute}`;
}
```

### `src/lib/supabase/client.ts`
```ts
"use client";

import { createBrowserClient } from "@supabase/ssr";

/**
 * Browser Supabase client for Auth and realtime session updates.
 * Configure NEXT_PUBLIC_SUPABASE_URL and NEXT_PUBLIC_SUPABASE_ANON_KEY.
 */
export function createClient() {
	const url = process.env.NEXT_PUBLIC_SUPABASE_URL ?? "";
	const key = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY ?? "";
	if (!url || !key) {
		if (typeof window !== "undefined" && process.env.NODE_ENV === "development") {

			console.warn(
				"[supabase] Missing NEXT_PUBLIC_SUPABASE_URL or NEXT_PUBLIC_SUPABASE_ANON_KEY."
			);
		}
	}
	return createBrowserClient(url, key);
}
```

### `src/lib/supabase/middleware.ts`
```ts
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

/**
 * Refreshes the Supabase session on each matched request. Required for stable
 * server/client auth state (see Supabase Next.js SSR guide).
 */
export async function updateSession(request: NextRequest): Promise<NextResponse> {
	let supabaseResponse = NextResponse.next({ request });

	const url = process.env.NEXT_PUBLIC_SUPABASE_URL ?? "";
	const key = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY ?? "";
	if (!url || !key) {
		return supabaseResponse;
	}

	const supabase = createServerClient(url, key, {
		cookies: {
			getAll() {
				return request.cookies.getAll();
			},
			setAll(cookiesToSet) {
				cookiesToSet.forEach(({ name, value }) => {
					request.cookies.set(name, value);
				});
				supabaseResponse = NextResponse.next({ request });
				cookiesToSet.forEach(({ name, value, options }) => {
					supabaseResponse.cookies.set(name, value, options);
				});
			},
		},
	});

	await supabase.auth.getUser();

	return supabaseResponse;
}
```

### `src/lib/supabase/server.ts`
```ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

/**
 * Server-side Supabase client (Route Handlers, Server Components).
 */
export async function createClient() {
	const cookieStore = await cookies();

	return createServerClient(
		process.env.NEXT_PUBLIC_SUPABASE_URL ?? "",
		process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY ?? "",
		{
			cookies: {
				getAll() {
					return cookieStore.getAll();
				},
				setAll(cookiesToSet) {
					try {
						cookiesToSet.forEach(({ name, value, options }) => {
							cookieStore.set(name, value, options);
						});
					} catch {

					}
				},
			},
		}
	);
}
```

### `src/modules/genui/GenUIComponentRegistry.tsx`
```tsx
"use client";

import { useState, type ReactNode } from "react";
import {
	AlertTriangle,
	ArrowRight,
	BrainCircuit,
	Check,
	Coins,
	Database,
	DatabaseZap,
	Gauge,
	GitMerge,
	LockKeyhole,
	Play,
	RotateCcw,
	ShieldAlert,
	Sparkles,
	Users,
	X,
	Zap,
} from "lucide-react";

export type GenUIRegistryProps = Record<string, unknown>;

export interface DemoPdfAnkiFlashcardProps {
	front: string;
	back: string;
	tags: string[];
}

export interface DemoReceiptHitlFormProps {
	demoMode: boolean;
}

export interface DemoAudioNoteTelegramMockProps {
	transcript: string;
	summary: string;
}

export interface DemoPdfAnkiErrorAlertProps {
	error: {
		code: string;
		title: string;
		detail?: string;
	};
}

export interface DemoCapacityBannerProps {
	usage: {
		at_80_percent: boolean;
		receipt_count?: number;
		max_receipts?: number;
		tier?: string;
	};
}

export interface FinOpsROICalculatorProps {
	competitorCost: number;
	frugalFortressCost: number;
	savings: number;
}

export interface AsyncSingleFlightVisualizerProps {
	concurrentRequests: number;
	cacheHits: number;
}

export interface RAGLearningLoopVisualizerProps {
	originalError: string;
	correction: string;
}

export interface SecurityBlockCardProps {
	reason: string;
}

export interface ArchitectureFitScoreProps {
	score: number;
	reason: string;
}

interface DemoFitBadgeProps {
	score: number;
}

interface DemoAlternativeApproachCardProps {
	content: string;
}

type RegistryRenderer = (
	props: GenUIRegistryProps,
	content?: string
) => ReactNode;

function asRecord(value: unknown): GenUIRegistryProps | null {
	if (!value || typeof value !== "object" || Array.isArray(value)) {
		return null;
	}
	return value as GenUIRegistryProps;
}

function stringProp(
	props: GenUIRegistryProps,
	key: string,
	fallback: string
): string {
	const value = props[key];
	return typeof value === "string" && value.trim().length > 0
		? value
		: fallback;
}

function numberProp(
	props: GenUIRegistryProps,
	key: string,
	fallback: number
): number {
	const value = props[key];
	if (typeof value === "number" && Number.isFinite(value)) {
		return value;
	}
	if (typeof value === "string" && value.trim().length > 0) {
		const parsed = Number(value);
		return Number.isFinite(parsed) ? parsed : fallback;
	}
	return fallback;
}

function booleanProp(
	props: GenUIRegistryProps,
	key: string,
	fallback: boolean
): boolean {
	const value = props[key];
	return typeof value === "boolean" ? value : fallback;
}

function stringArrayProp(
	props: GenUIRegistryProps,
	key: string,
	fallback: string[]
): string[] {
	const value = props[key];
	if (Array.isArray(value)) {
		const tags = value.filter(
			(item): item is string =>
				typeof item === "string" && item.trim().length > 0
		);
		return tags.length > 0 ? tags : fallback;
	}
	if (typeof value === "string" && value.trim().length > 0) {
		return value
			.split(",")
			.map((tag) => tag.trim())
			.filter(Boolean);
	}
	return fallback;
}

function coercePdfAnkiFlashcardProps(
	props: GenUIRegistryProps
): DemoPdfAnkiFlashcardProps {
	return {
		front: stringProp(props, "front", "What makes this pipeline reliable?"),
		back: stringProp(
			props,
			"back",
			"It combines deterministic validation, graceful model fallback, and human review before data is trusted."
		),
		tags: stringArrayProp(props, "tags", ["Demo", "RAG"]),
	};
}

function coerceReceiptHitlFormProps(
	props: GenUIRegistryProps
): DemoReceiptHitlFormProps {
	return {
		demoMode: booleanProp(props, "demoMode", true),
	};
}

function coerceAudioNoteTelegramMockProps(
	props: GenUIRegistryProps
): DemoAudioNoteTelegramMockProps {
	return {
		transcript: stringProp(
			props,
			"transcript",
			"Deploying to FastAPI after the model queue drains."
		),
		summary: stringProp(
			props,
			"summary",
			"Deployment note captured, summarized, and ready for review."
		),
	};
}

function coercePdfAnkiErrorAlertProps(
	props: GenUIRegistryProps
): DemoPdfAnkiErrorAlertProps {
	const error = asRecord(props.error);
	return {
		error: {
			code: error
				? stringProp(error, "code", "groq_free_tier_busy")
				: "groq_free_tier_busy",
			title: error
				? stringProp(
						error,
						"title",
						"Groq queue full. Falling back to a secondary model."
					)
				: "Groq queue full. Falling back to a secondary model.",
			detail: error ? stringProp(error, "detail", "") || undefined : undefined,
		},
	};
}

function coerceCapacityBannerProps(
	props: GenUIRegistryProps
): DemoCapacityBannerProps {
	const usage = asRecord(props.usage);
	return {
		usage: {
			at_80_percent: usage
				? booleanProp(usage, "at_80_percent", true)
				: true,
			receipt_count: usage
				? numberProp(usage, "receipt_count", 800)
				: 800,
			max_receipts: usage ? numberProp(usage, "max_receipts", 1000) : 1000,
			tier: usage ? stringProp(usage, "tier", "free") : "free",
		},
	};
}

function coerceFitBadgeProps(props: GenUIRegistryProps): DemoFitBadgeProps {
	return {
		score: Math.max(0, Math.min(100, numberProp(props, "score", 92))),
	};
}

function coerceAlternativeApproachProps(
	props: GenUIRegistryProps,
	content?: string
): DemoAlternativeApproachCardProps {
	return {
		content:
			typeof content === "string" && content.trim().length > 0
				? content
				: stringProp(
						props,
						"content",
						"The backend suggested a lower-risk architecture for this constraint."
					),
	};
}

function coerceFinOpsROICalculatorProps(
	props: GenUIRegistryProps
): FinOpsROICalculatorProps {
	const competitorCost = Math.max(1, numberProp(props, "competitorCost", 4200));
	const frugalFortressCost = Math.max(
		1,
		numberProp(props, "frugalFortressCost", 740)
	);
	const computedSavings =
		((competitorCost - frugalFortressCost) / competitorCost) * 100;
	const streamedSavings = numberProp(props, "savings", computedSavings);
	const savingsPercent =
		streamedSavings > 100 ? computedSavings : streamedSavings;

	return {
		competitorCost,
		frugalFortressCost,
		savings: Math.max(0, Math.min(100, savingsPercent)),
	};
}

function coerceAsyncSingleFlightVisualizerProps(
	props: GenUIRegistryProps
): AsyncSingleFlightVisualizerProps {
	return {
		concurrentRequests: Math.max(
			2,
			Math.min(12, Math.round(numberProp(props, "concurrentRequests", 8)))
		),
		cacheHits: Math.max(
			0,
			Math.min(99, Math.round(numberProp(props, "cacheHits", 23)))
		),
	};
}

function coerceRAGLearningLoopVisualizerProps(
	props: GenUIRegistryProps
): RAGLearningLoopVisualizerProps {
	return {
		originalError: stringProp(
			props,
			"originalError",
			"The model confused provider rate limits with an application bug."
		),
		correction: stringProp(
			props,
			"correction",
			"Classify provider saturation as a retryable graceful-degradation event."
		),
	};
}

function coerceSecurityBlockCardProps(
	props: GenUIRegistryProps
): SecurityBlockCardProps {
	return {
		reason: stringProp(
			props,
			"reason",
			"Sensitive customer identifiers were detected in the prompt."
		),
	};
}

function coerceArchitectureFitScoreProps(
	props: GenUIRegistryProps
): ArchitectureFitScoreProps {
	return {
		score: Math.max(0, Math.min(100, Math.round(numberProp(props, "score", 91)))),
		reason: stringProp(
			props,
			"reason",
			"Strong fit for teams that need AI workflows with deterministic validation, cost controls, and operational transparency."
		),
	};
}

function formatUsd(value: number): string {
	return new Intl.NumberFormat("en-US", {
		currency: "USD",
		maximumFractionDigits: 0,
		style: "currency",
	}).format(value);
}

export function DemoPdfAnkiFlashcard({
	front,
	back,
	tags,
}: DemoPdfAnkiFlashcardProps) {
	const [flipped, setFlipped] = useState(false);

	return (
		<article className="my-3 w-full max-w-xl text-left">
			<div className="relative min-h-[178px] [perspective:1100px]">
				<button
					type="button"
					aria-expanded={flipped}
					aria-label="Flip flashcard"
					onClick={() => setFlipped((current) => !current)}
					className="relative min-h-[178px] w-full rounded-lg border border-slate-200 bg-transparent p-0 text-left shadow-sm outline-none transition duration-200 hover:-translate-y-0.5 hover:border-indigo-200 hover:shadow-md focus-visible:ring-2 focus-visible:ring-indigo-400 focus-visible:ring-offset-2"
				>
					<div
						className="relative min-h-[178px] w-full duration-500 [transform-style:preserve-3d] motion-reduce:transition-none"
						style={{ transform: flipped ? "rotateY(180deg)" : "rotateY(0deg)" }}
					>
						<div
							aria-hidden={flipped}
							className="absolute inset-0 flex min-h-[178px] flex-col rounded-lg border border-slate-100 bg-slate-50 p-4 [backface-visibility:hidden]"
						>
							<div className="flex items-center justify-between gap-3">
								<span className="text-[10px] font-semibold uppercase tracking-wide text-slate-500">
									Front
								</span>
								<RotateCcw className="h-3.5 w-3.5 text-slate-400" aria-hidden />
							</div>
							<p className="mt-3 min-h-0 flex-1 overflow-auto whitespace-pre-wrap text-sm leading-relaxed text-slate-800">
								{front}
							</p>
							<span className="mt-3 text-[10px] font-medium text-slate-400">
								Click to reveal the answer
							</span>
						</div>
						<div
							aria-hidden={!flipped}
							className="absolute inset-0 flex min-h-[178px] flex-col rounded-lg border border-indigo-100 bg-indigo-50 p-4 [backface-visibility:hidden] [transform:rotateY(180deg)]"
						>
							<div className="flex items-center justify-between gap-3">
								<span className="text-[10px] font-semibold uppercase tracking-wide text-indigo-700">
									Back
								</span>
								<RotateCcw className="h-3.5 w-3.5 text-indigo-500" aria-hidden />
							</div>
							<p className="mt-3 min-h-0 flex-1 overflow-auto whitespace-pre-wrap text-sm leading-relaxed text-slate-800">
								{back}
							</p>
							<span className="mt-3 text-[10px] font-medium text-indigo-600/80">
								Click to return to the prompt
							</span>
						</div>
					</div>
				</button>
			</div>
			<div className="mt-2 flex flex-wrap gap-1.5">
				{tags.map((tag) => (
					<span
						key={tag}
						className="rounded-full bg-slate-200 px-2.5 py-1 text-[11px] font-medium text-slate-700"
					>
						{tag}
					</span>
				))}
			</div>
		</article>
	);
}

export function DemoReceiptHitlForm({ demoMode }: DemoReceiptHitlFormProps) {
	const expectedTotal = "12.00";

	return (
		<section
			aria-label="Receipt validation demo"
			className="my-3 w-full max-w-2xl rounded-lg border border-slate-200 bg-white p-4 text-left shadow-sm"
		>
			<div className="flex flex-wrap items-start justify-between gap-3">
				<div>
					<p className="text-[11px] font-semibold uppercase tracking-wide text-slate-500">
						Human review queue
					</p>
					<h3 className="mt-1 text-sm font-semibold text-slate-900">
						Receipt math validation
					</h3>
				</div>
				<span className="rounded-full border border-amber-200 bg-amber-50 px-2.5 py-1 text-[11px] font-medium text-amber-800">
					{demoMode ? "Demo mode" : "Read-only"}
				</span>
			</div>

			<div className="mt-3 rounded-md border border-amber-200 bg-amber-50 p-3">
				<p className="text-[11px] font-semibold text-amber-900">
					Correction required
				</p>
				<p className="mt-1 text-[11px] leading-relaxed text-amber-800">
					Subtotal 10.00 + tax 2.00 should equal {expectedTotal}, but the
					extracted total is 15.00.
				</p>
			</div>

			<div className="mt-3 grid gap-3 text-xs sm:grid-cols-[1.2fr_0.45fr_0.7fr_0.7fr]">
				<ReadOnlyField label="Description" value="Implementation sprint" />
				<ReadOnlyField label="Qty" value="1" />
				<ReadOnlyField label="Unit" value="10.00" />
				<ReadOnlyField label="Line total" value="10.00" />
			</div>

			<div className="mt-3 grid gap-3 text-xs sm:grid-cols-3">
				<ReadOnlyField label="Subtotal" value="10.00" />
				<ReadOnlyField label="Tax" value="2.00" />
				<ReadOnlyField
					label="Total"
					value="15.00"
					tone="danger"
					helper={`Expected ${expectedTotal}`}
				/>
			</div>

			<div className="mt-4 flex flex-wrap items-center gap-2">
				<button
					type="button"
					aria-disabled
					className="rounded-full bg-slate-900 px-3 py-1.5 text-[11px] font-semibold text-white shadow-sm"
				>
					Save corrected receipt
				</button>
				<span className="text-[11px] font-medium text-slate-500">
					Actions disabled in portfolio preview
				</span>
			</div>
		</section>
	);
}

function ReadOnlyField({
	label,
	value,
	helper,
	tone = "default",
}: {
	label: string;
	value: string;
	helper?: string;
	tone?: "default" | "danger";
}) {
	const inputClass =
		tone === "danger"
			? "border-rose-400 bg-rose-50 text-rose-900"
			: "border-slate-300 bg-slate-50 text-slate-800";

	return (
		<label className="flex min-w-0 flex-col gap-1">
			<span className="text-[11px] text-slate-600">{label}</span>
			<input
				readOnly
				value={value}
				className={`min-w-0 rounded-md border px-2 py-1.5 text-xs outline-none ${inputClass}`}
			/>
			{helper ? <span className="text-[10px] text-rose-700">{helper}</span> : null}
		</label>
	);
}

export function DemoAudioNoteTelegramMock({
	transcript,
	summary,
}: DemoAudioNoteTelegramMockProps) {
	return (
		<section
			aria-label="Telegram audio note demo"
			className="my-3 w-full max-w-xl rounded-lg border border-slate-200 bg-slate-950 p-3 text-left text-slate-50 shadow-sm"
		>
			<div className="rounded-lg bg-[#2aabee] p-3 shadow-sm">
				<div className="flex items-start gap-3">
					<div className="grid h-9 w-9 shrink-0 place-items-center rounded-full bg-white/20 text-sm font-semibold text-white">
						A
					</div>
					<div className="min-w-0 flex-1 rounded-lg bg-white px-3 py-2 text-slate-900 shadow-sm">
						<div className="flex items-center gap-2">
							<button
								type="button"
								aria-label="Play demo audio"
								aria-disabled
								className="grid h-8 w-8 shrink-0 place-items-center rounded-full bg-[#2aabee] text-white shadow-sm"
							>
								<Play className="h-3.5 w-3.5 fill-current" aria-hidden />
							</button>
							<div className="flex min-w-0 flex-1 items-end gap-0.5">
								{[35, 52, 40, 74, 58, 44, 68, 48, 56, 38, 64, 46].map(
									(height, index) => (
										<span
											key={`${height}-${index}`}
											className="w-1 flex-1 rounded-full bg-sky-200"
											style={{ height: `${height / 5}px` }}
										/>
									)
								)}
							</div>
							<span className="shrink-0 text-[10px] font-medium text-slate-500">
								0:18
							</span>
						</div>

						<div className="mt-3 border-t border-slate-100 pt-2">
							<p className="text-[10px] font-semibold uppercase tracking-wide text-slate-500">
								Transcript
							</p>
							<p className="mt-1 text-xs leading-relaxed text-slate-700">
								{transcript}
							</p>
						</div>

						<div className="mt-2 rounded-md bg-slate-50 p-2">
							<p className="text-[10px] font-semibold uppercase tracking-wide text-slate-500">
								Summary
							</p>
							<p className="mt-1 text-xs leading-relaxed text-slate-800">
								{summary}
							</p>
						</div>

						<div className="mt-3 flex flex-wrap gap-2">
							<button
								type="button"
								aria-disabled
								className="inline-flex items-center gap-1.5 rounded-full bg-emerald-50 px-2.5 py-1 text-[11px] font-medium text-emerald-700"
							>
								<Check className="h-3 w-3" aria-hidden />
								Useful
							</button>
							<button
								type="button"
								aria-disabled
								className="inline-flex items-center gap-1.5 rounded-full bg-rose-50 px-2.5 py-1 text-[11px] font-medium text-rose-700"
							>
								<X className="h-3 w-3" aria-hidden />
								Inaccurate
							</button>
						</div>
					</div>
				</div>
			</div>
		</section>
	);
}

export function DemoPdfAnkiErrorAlert({ error }: DemoPdfAnkiErrorAlertProps) {
	return (
		<section
			role="status"
			aria-live="polite"
			className="my-3 w-full max-w-xl rounded-lg border border-amber-200 bg-amber-50 p-3 text-left text-amber-950 shadow-sm"
		>
			<div className="flex items-start gap-3">
				<div className="grid h-8 w-8 shrink-0 place-items-center rounded-full bg-amber-100 text-amber-700">
					<AlertTriangle className="h-4 w-4" aria-hidden />
				</div>
				<div className="min-w-0 flex-1">
					<div className="flex flex-wrap items-center gap-2">
						<p className="text-xs font-semibold">Graceful degradation active</p>
						<span className="max-w-full rounded-full bg-white px-2 py-0.5 font-mono text-[10px] text-amber-800 shadow-sm">
							{error.code}
						</span>
					</div>
					<p className="mt-1 text-xs leading-relaxed text-amber-900">
						{error.title}
					</p>
					{error.detail ? (
						<p className="mt-1 text-[11px] leading-relaxed text-amber-800">
							{error.detail}
						</p>
					) : null}
				</div>
			</div>
		</section>
	);
}

export function DemoCapacityBanner({ usage }: DemoCapacityBannerProps) {
	if (!usage.at_80_percent) {
		return null;
	}

	const current = usage.receipt_count ?? 800;
	const max = usage.max_receipts ?? 1000;
	const percent = Math.max(0, Math.min(100, Math.round((current / max) * 100)));

	return (
		<section className="my-3 w-full max-w-2xl rounded-lg border border-amber-300 bg-amber-50 p-4 text-left text-amber-950 shadow-sm">
			<div className="flex flex-wrap items-start justify-between gap-3">
				<div className="flex min-w-0 gap-3">
					<div className="grid h-9 w-9 shrink-0 place-items-center rounded-md bg-amber-100 text-amber-700">
						<Database className="h-4 w-4" aria-hidden />
					</div>
					<div className="min-w-0">
						<p className="text-xs font-semibold">Storage limit approaching</p>
						<p className="mt-1 text-[11px] leading-relaxed text-amber-800">
							The {usage.tier ?? "free"} workspace is at {percent}% capacity.
							Older records will be pruned only after warning thresholds.
						</p>
					</div>
				</div>
				<span className="rounded-full bg-white px-2.5 py-1 text-[11px] font-semibold text-amber-800 shadow-sm">
					{current}/{max}
				</span>
			</div>
			<div className="mt-3 h-2 overflow-hidden rounded-full bg-amber-100">
				<div
					className="h-full rounded-full bg-amber-600"
					style={{ width: `${percent}%` }}
				/>
			</div>
		</section>
	);
}

export function FinOpsROICalculator({
	competitorCost,
	frugalFortressCost,
	savings,
}: FinOpsROICalculatorProps) {
	const maxCost = Math.max(competitorCost, frugalFortressCost, 1);
	const competitorWidth = Math.max(8, (competitorCost / maxCost) * 100);
	const frugalWidth = Math.max(8, (frugalFortressCost / maxCost) * 100);

	return (
		<section className="my-3 w-full max-w-2xl rounded-lg border border-slate-200 bg-white p-4 text-left shadow-sm">
			<div className="flex flex-wrap items-start justify-between gap-3">
				<div>
					<p className="text-[11px] font-semibold uppercase tracking-wide text-slate-500">
						FinOps ROI model
					</p>
					<h3 className="mt-1 text-sm font-semibold text-slate-950">
						Frugal Fortress cost profile
					</h3>
				</div>
				<div className="inline-flex items-center gap-2 rounded-full bg-emerald-50 px-3 py-1.5 text-emerald-700 shadow-sm">
					<Coins className="h-3.5 w-3.5" aria-hidden />
					<span className="text-xs font-bold">{Math.round(savings)}% saved</span>
				</div>
			</div>

			<div className="mt-5 space-y-4">
				<RoiBarRow
					label="Typical competitor"
					value={formatUsd(competitorCost)}
					width={competitorWidth}
					tone="competitor"
				/>
				<RoiBarRow
					label="Frugal Fortress"
					value={formatUsd(frugalFortressCost)}
					width={frugalWidth}
					tone="frugal"
				/>
			</div>

			<div className="mt-4 grid gap-2 text-[11px] text-slate-600 sm:grid-cols-3">
				<div className="rounded-md border border-slate-200 bg-slate-50 p-2">
					<p className="font-semibold text-slate-900">Budget guardrails</p>
					<p className="mt-1">Caps, queues, and storage thresholds stay visible.</p>
				</div>
				<div className="rounded-md border border-slate-200 bg-slate-50 p-2">
					<p className="font-semibold text-slate-900">Model routing</p>
					<p className="mt-1">Use premium inference only when value justifies it.</p>
				</div>
				<div className="rounded-md border border-emerald-200 bg-emerald-50 p-2">
					<p className="font-semibold text-emerald-800">Projected delta</p>
					<p className="mt-1 text-emerald-700">
						{formatUsd(Math.max(0, competitorCost - frugalFortressCost))} avoided.
					</p>
				</div>
			</div>
		</section>
	);
}

function RoiBarRow({
	label,
	value,
	width,
	tone,
}: {
	label: string;
	value: string;
	width: number;
	tone: "competitor" | "frugal";
}) {
	const fillClass =
		tone === "frugal"
			? "bg-emerald-500"
			: "bg-slate-700";

	return (
		<div>
			<div className="mb-1 flex items-center justify-between gap-3 text-xs">
				<span className="font-medium text-slate-700">{label}</span>
				<span className="font-semibold text-slate-950">{value}</span>
			</div>
			<div className="h-8 overflow-hidden rounded-md bg-slate-100">
				<div
					className={`flex h-full origin-left animate-[genui-bar-grow_900ms_ease-out_forwards] items-center justify-end rounded-md px-2 text-[11px] font-semibold text-white shadow-sm ${fillClass}`}
					style={{ width: `${width}%` }}
				>
					{Math.round(width)}%
				</div>
			</div>
		</div>
	);
}

export function AsyncSingleFlightVisualizer({
	concurrentRequests,
	cacheHits,
}: AsyncSingleFlightVisualizerProps) {
	const visibleRequests = Array.from(
		{ length: Math.min(concurrentRequests, 6) },
		(_, index) => index + 1
	);
	const visibleResponses = visibleRequests.slice(0, 4);

	return (
		<section className="my-3 w-full max-w-2xl rounded-lg border border-slate-800 bg-slate-950 p-4 text-left text-slate-50 shadow-sm">
			<div className="flex flex-wrap items-start justify-between gap-3">
				<div>
					<p className="text-[11px] font-semibold uppercase tracking-wide text-cyan-300">
						Async single-flight
					</p>
					<h3 className="mt-1 text-sm font-semibold text-white">
						Request collapse before LLM execution
					</h3>
				</div>
				<div className="flex flex-wrap gap-2 text-[11px]">
					<span className="rounded-full bg-cyan-400/10 px-2.5 py-1 font-medium text-cyan-200">
						{concurrentRequests} concurrent
					</span>
					<span className="rounded-full bg-emerald-400/10 px-2.5 py-1 font-medium text-emerald-200">
						{cacheHits} cache hits
					</span>
				</div>
			</div>

			<div className="relative mt-5 overflow-hidden rounded-lg border border-slate-800 bg-slate-900/70 p-3">
				<div className="absolute inset-x-8 top-1/2 hidden h-px bg-cyan-300/20 sm:block" />
				{[0, 1, 2].map((index) => (
					<span
						key={`inbound-packet-${index}`}
						className="absolute left-[18%] top-1/2 hidden h-2 w-2 rounded-full bg-cyan-300 shadow-[0_0_16px_rgba(103,232,249,0.9)] animate-[genui-single-flight-in_2.6s_ease-in-out_infinite] sm:block"
						style={{ animationDelay: `${index * 420}ms` }}
					/>
				))}
				{[0, 1].map((index) => (
					<span
						key={`outbound-packet-${index}`}
						className="absolute right-[20%] top-1/2 hidden h-2 w-2 rounded-full bg-emerald-300 shadow-[0_0_16px_rgba(110,231,183,0.9)] animate-[genui-single-flight-out_2.6s_ease-in-out_infinite] sm:block"
						style={{ animationDelay: `${index * 620 + 500}ms` }}
					/>
				))}

				<div className="relative grid gap-4 sm:grid-cols-[1fr_0.9fr_1fr] sm:items-center">
					<div className="space-y-2">
						{visibleRequests.map((request) => (
							<div
								key={request}
								className="flex items-center gap-2 rounded-md border border-slate-700 bg-slate-950 px-2.5 py-2"
							>
								<Users className="h-3.5 w-3.5 text-cyan-300" aria-hidden />
								<span className="text-[11px] text-slate-200">
									User request {request}
								</span>
							</div>
						))}
					</div>

					<div className="rounded-lg border border-cyan-300/30 bg-cyan-300/10 p-3 text-center shadow-[0_0_30px_rgba(34,211,238,0.12)]">
						<div className="mx-auto grid h-12 w-12 place-items-center rounded-full bg-cyan-300 text-slate-950 shadow-sm">
							<GitMerge className="h-5 w-5" aria-hidden />
						</div>
						<p className="mt-2 text-xs font-semibold text-white">
							1 LLM execution
						</p>
						<p className="mt-1 text-[11px] leading-relaxed text-cyan-100">
							Lock acquisition prevents duplicate inference spend.
						</p>
					</div>

					<div className="space-y-2">
						{visibleResponses.map((response) => (
							<div
								key={response}
								className="flex items-center gap-2 rounded-md border border-emerald-400/30 bg-emerald-400/10 px-2.5 py-2"
							>
								<Zap className="h-3.5 w-3.5 text-emerald-300" aria-hidden />
								<span className="text-[11px] text-emerald-100">
									Shared response {response}
								</span>
							</div>
						))}
					</div>
				</div>
			</div>
		</section>
	);
}

export function RAGLearningLoopVisualizer({
	originalError,
	correction,
}: RAGLearningLoopVisualizerProps) {
	const steps = [
		{
			icon: Sparkles,
			label: "1. User correction",
			text: correction,
			tone: "border-indigo-200 bg-indigo-50 text-indigo-800",
		},
		{
			icon: DatabaseZap,
			label: "2. pgvector embedding",
			text: "Store the corrected pattern as retrievable operational memory.",
			tone: "border-cyan-200 bg-cyan-50 text-cyan-800",
		},
		{
			icon: BrainCircuit,
			label: "3. Few-shot injection",
			text: "Next proposal receives the correction as contextual evidence.",
			tone: "border-emerald-200 bg-emerald-50 text-emerald-800",
		},
	] as const;

	return (
		<section className="my-3 w-full max-w-2xl rounded-lg border border-slate-200 bg-white p-4 text-left shadow-sm">
			<div>
				<p className="text-[11px] font-semibold uppercase tracking-wide text-slate-500">
					RAG learning loop
				</p>
				<h3 className="mt-1 text-sm font-semibold text-slate-950">
					Corrections become retrieval memory
				</h3>
			</div>

			<div className="mt-3 rounded-md border border-rose-200 bg-rose-50 p-3 text-xs text-rose-800">
				<p className="font-semibold">Original error</p>
				<p className="mt-1 leading-relaxed">{originalError}</p>
			</div>

			<div className="mt-4 grid gap-3 md:grid-cols-3">
				{steps.map((step, index) => {
					const Icon = step.icon;
					return (
						<div key={step.label} className="relative">
							<article
								className={`h-full rounded-lg border p-3 shadow-sm ${step.tone}`}
							>
								<Icon className="h-5 w-5" aria-hidden />
								<p className="mt-2 text-xs font-semibold">{step.label}</p>
								<p className="mt-1 text-[11px] leading-relaxed">{step.text}</p>
							</article>
							{index < steps.length - 1 ? (
								<ArrowRight
									className="absolute -right-2 top-1/2 hidden h-4 w-4 -translate-y-1/2 text-slate-400 md:block"
									aria-hidden
								/>
							) : null}
						</div>
					);
				})}
			</div>
		</section>
	);
}

export function SecurityBlockCard({ reason }: SecurityBlockCardProps) {
	return (
		<section
			role="alert"
			className="my-3 w-full max-w-2xl rounded-lg border border-rose-300 bg-rose-50 p-4 text-left text-rose-950 shadow-sm"
		>
			<div className="flex items-start gap-3">
				<div className="grid h-10 w-10 shrink-0 place-items-center rounded-md bg-rose-600 text-white shadow-sm">
					<ShieldAlert className="h-5 w-5" aria-hidden />
				</div>
				<div className="min-w-0 flex-1">
					<div className="flex flex-wrap items-center gap-2">
						<p className="text-sm font-semibold">DLP proxy blocked the request</p>
						<span className="rounded-full bg-white px-2.5 py-1 text-[11px] font-semibold text-rose-700 shadow-sm">
							SOC2 control
						</span>
					</div>
					<p className="mt-2 text-xs leading-relaxed text-rose-900">
						{reason}
					</p>
					<div className="mt-3 grid gap-2 text-[11px] sm:grid-cols-3">
						<span className="rounded-md border border-rose-200 bg-white px-2.5 py-2 font-medium">
							Prompt intercepted
						</span>
						<span className="rounded-md border border-rose-200 bg-white px-2.5 py-2 font-medium">
							Sensitive payload withheld
						</span>
						<span className="rounded-md border border-rose-200 bg-white px-2.5 py-2 font-medium">
							Audit event retained
						</span>
					</div>
				</div>
			</div>
		</section>
	);
}

export function ArchitectureFitScore({
	score,
	reason,
}: ArchitectureFitScoreProps) {
	const clampedScore = Math.max(0, Math.min(100, score));

	return (
		<section className="my-3 w-full max-w-2xl rounded-lg border border-slate-200 bg-white p-4 text-left shadow-sm">
			<div className="grid gap-4 sm:grid-cols-[10rem_1fr] sm:items-center">
				<div className="relative mx-auto h-36 w-36">
					<div
						className="absolute inset-0 rounded-full animate-[genui-gauge-reveal_900ms_ease-out_forwards]"
						style={{
							background: `conic-gradient(#10b981 ${clampedScore * 3.6}deg, #e2e8f0 0deg)`,
						}}
					/>
					<div className="absolute inset-3 grid place-items-center rounded-full bg-white shadow-inner">
						<div className="text-center">
							<p className="text-3xl font-bold text-slate-950">{clampedScore}</p>
							<p className="text-[10px] font-semibold uppercase tracking-wide text-slate-500">
								Fit score
							</p>
						</div>
					</div>
				</div>

				<div className="min-w-0">
					<p className="text-[11px] font-semibold uppercase tracking-wide text-slate-500">
						Architecture fit
					</p>
					<h3 className="mt-1 text-sm font-semibold text-slate-950">
						CTO-ready technical alignment
					</h3>
					<p className="mt-2 text-xs leading-relaxed text-slate-600">{reason}</p>
					<button
						type="button"
						className="mt-4 inline-flex items-center gap-2 rounded-full bg-slate-950 px-4 py-2 text-xs font-semibold text-white shadow-sm transition hover:-translate-y-0.5 hover:bg-slate-800 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-emerald-400 focus-visible:ring-offset-2"
					>
						<LockKeyhole className="h-3.5 w-3.5" aria-hidden />
						Book Technical Interview
					</button>
				</div>
			</div>
		</section>
	);
}

function DemoFitBadge({ score }: DemoFitBadgeProps) {
	return (
		<span className="my-2 inline-flex items-center gap-1.5 rounded-full bg-emerald-100 px-3 py-1.5 text-[11px] font-semibold text-emerald-800 shadow-sm">
			<Gauge className="h-3.5 w-3.5" aria-hidden />
			Fit score: {score}%
		</span>
	);
}

function DemoAlternativeApproachCard({
	content,
}: DemoAlternativeApproachCardProps) {
	return (
		<article className="my-3 w-full max-w-xl rounded-lg border border-amber-200 bg-amber-50 p-3 text-left text-slate-800 shadow-sm">
			<p className="text-[11px] font-semibold uppercase tracking-wide text-amber-700">
				Alternative approach
			</p>
			<p className="mt-1 text-xs leading-relaxed">{content}</p>
		</article>
	);
}

export const GenUIComponentRegistry: Record<string, RegistryRenderer> = {
	PdfAnkiFlashcard: (props) => (
		<DemoPdfAnkiFlashcard {...coercePdfAnkiFlashcardProps(props)} />
	),
	ReceiptHitlForm: (props) => (
		<DemoReceiptHitlForm {...coerceReceiptHitlFormProps(props)} />
	),
	AudioNoteTelegramMock: (props) => (
		<DemoAudioNoteTelegramMock {...coerceAudioNoteTelegramMockProps(props)} />
	),
	PdfAnkiErrorAlert: (props) => (
		<DemoPdfAnkiErrorAlert {...coercePdfAnkiErrorAlertProps(props)} />
	),
	CapacityBanner: (props) => (
		<DemoCapacityBanner {...coerceCapacityBannerProps(props)} />
	),
	FinOpsROICalculator: (props) => (
		<FinOpsROICalculator {...coerceFinOpsROICalculatorProps(props)} />
	),
	AsyncSingleFlightVisualizer: (props) => (
		<AsyncSingleFlightVisualizer
			{...coerceAsyncSingleFlightVisualizerProps(props)}
		/>
	),
	RAGLearningLoopVisualizer: (props) => (
		<RAGLearningLoopVisualizer
			{...coerceRAGLearningLoopVisualizerProps(props)}
		/>
	),
	SecurityBlockCard: (props) => (
		<SecurityBlockCard {...coerceSecurityBlockCardProps(props)} />
	),
	ArchitectureFitScore: (props) => (
		<ArchitectureFitScore {...coerceArchitectureFitScoreProps(props)} />
	),
	FitBadge: (props) => <DemoFitBadge {...coerceFitBadgeProps(props)} />,
	AlternativeApproachCard: (props, content) => (
		<DemoAlternativeApproachCard
			{...coerceAlternativeApproachProps(props, content)}
		/>
	),
};

export function renderGenUIComponent(
	component: string,
	props?: GenUIRegistryProps | null,
	content?: string
): ReactNode {
	const renderer = GenUIComponentRegistry[component];
	if (!renderer) {
		return null;
	}
	return renderer(props ?? {}, content);
}
```

### `src/modules/genui/schemas.ts`
```ts
import { z } from "zod";

export const staticProjectSchema = z.object({
		id: z.string(),
		title: z.string().max(200),
		stack: z.string().max(100),
		description: z.string().nullable().optional(),
		youtube_id: z.string().nullable().optional(),
		tags: z.array(z.string()).default([]),
});

export const projectGridResponseSchema = z.object({
		projects: z.array(staticProjectSchema),
		total: z.number().int().nonnegative()
});

export type StaticProject = z.infer<typeof staticProjectSchema>;
export type ProjectGridResponse = z.infer<typeof projectGridResponseSchema>;
export type ProposalStreamChunk = z.infer<typeof proposalStreamChunkSchema>;

export const proposalStreamChunkSchema = z.object({
		type: z.union([z.literal("text"), z.literal("component")]),
		content: z.string().default(""),
		component: z.string().nullable().optional(),
		props: z.record(z.string(), z.unknown()).nullable().optional()
});
```

### `src/modules/genui/streaming-client.ts`
```ts
import { proposalStreamChunkSchema, type ProposalStreamChunk } from "./schemas";

export async function* streamProposal(
	body: Record<string, unknown>
): AsyncGenerator<ProposalStreamChunk, void, void> {
	const response = await fetch(
		`${process.env.NEXT_PUBLIC_BACKEND_URL ?? "/api/v1"}/genui/proposal/generate`,
		{
			method: "POST",
			headers: {
				"Content-Type": "application/json"
			},
			body: JSON.stringify(body)
		}
	);

	if (!response.ok) {
		if (response.status === 429) {
			throw new Error("RATE_LIMIT_EXCEEDED");
		}
		throw new Error(`HTTP Error: ${response.status}`);
	}

	if (!response.body) {
		return;
	}

	const reader = response.body.getReader();
	const decoder = new TextDecoder();
	let buffer = "";

	while (true) {
		const { done, value } = await reader.read();
		if (done) {
			break;
		}
		buffer += decoder.decode(value, { stream: true });

		let newlineIndex = buffer.indexOf("\n");
		while (newlineIndex !== -1) {
			const line = buffer.slice(0, newlineIndex).trim();
			buffer = buffer.slice(newlineIndex + 1);

			if (line) {
				const parsed = JSON.parse(line) as unknown;
				const chunk = proposalStreamChunkSchema.parse(parsed);
				yield chunk;
			}

			newlineIndex = buffer.indexOf("\n");
		}
	}
}
```

### `src/modules/genui/useGenUIQuota.ts`
```ts
"use client";

import { useState, useEffect, useCallback } from "react";

const QUOTA_KEY = "genui_proposal_quota";
const MAX_ATTEMPTS = 5;

type QuotaState = {
	date: string;
	count: number;
};

function getTodayStr(): string {
	const now = new Date();
	return `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, "0")}-${String(now.getDate()).padStart(2, "0")}`;
}

export function useGenUIQuota() {
	const [count, setCount] = useState<number>(0);
	const[isLocked, setIsLocked] = useState<boolean>(false);

	useEffect(() => {
		if (typeof window === "undefined") return;
		try {
			const raw = localStorage.getItem(QUOTA_KEY);
			const today = getTodayStr();
			if (raw) {
				const parsed = JSON.parse(raw) as QuotaState;
				if (parsed.date === today) {
					setCount(parsed.count);
					setIsLocked(parsed.count >= MAX_ATTEMPTS);
					return;
				}
			}

			localStorage.setItem(QUOTA_KEY, JSON.stringify({ date: today, count: 0 }));
			setCount(0);
			setIsLocked(false);
		} catch (e) {
			console.error("Failed to parse quota", e);
		}
	},[]);

	const increment = useCallback(() => {
		setCount((prev) => {
			const next = prev + 1;
			const today = getTodayStr();
			if (typeof window !== "undefined") {
				localStorage.setItem(QUOTA_KEY, JSON.stringify({ date: today, count: next }));
			}
			if (next >= MAX_ATTEMPTS) {
				setIsLocked(true);
			}
			return next;
		});
	},[]);

	const lockOut = useCallback(() => {
		const today = getTodayStr();
		if (typeof window !== "undefined") {
			localStorage.setItem(QUOTA_KEY, JSON.stringify({ date: today, count: MAX_ATTEMPTS }));
		}
		setCount(MAX_ATTEMPTS);
		setIsLocked(true);
	},[]);

	return { count, isLocked, increment, lockOut, maxAttempts: MAX_ATTEMPTS };
}
```

### `src/modules/observability/schemas.ts`
```ts
import { z } from "zod";

export const audioNoteLogEntrySchema = z.object({
	id: z.string(),
	created_at: z.string(),
	user_id: z.string().nullable().optional(),
	status: z.string(),
	duration_seconds: z.number().nonnegative().nullable().optional(),
	model: z.string().nullable().optional(),
	cost_usd: z.number().nonnegative().nullable().optional(),
	trace_id: z.string().nullable().optional(),
	summary: z.string().nullable().optional()
});

export const audioNotesLogResponseSchema = z.object({
	items: z.array(audioNoteLogEntrySchema),
	total: z.number().int().nonnegative()
});

export type AudioNoteLogEntry = z.infer<typeof audioNoteLogEntrySchema>;

export const cvTailorFallbackStepSchema = z.object({
	name: z.string(),
	triggered: z.boolean(),
	reason: z.string().nullable().optional()
});

export const cvTailorGuardSchema = z.object({
	selector: z.string(),
	rule: z.string(),
	last_result: z.boolean().nullable().optional()
});

export const cvTailorLogEntrySchema = z.object({
	id: z.string(),
	created_at: z.string(),
	session_id: z.string(),
	model: z.string().nullable().optional(),
	cost_usd: z.number().nonnegative().nullable().optional(),
	trace_id: z.string().nullable().optional(),
	fallback_chain: z.array(cvTailorFallbackStepSchema).default([]),
	validation_guards: z.array(cvTailorGuardSchema).default([])
});

export const cvTailorLogResponseSchema = z.object({
	items: z.array(cvTailorLogEntrySchema),
	total: z.number().int().nonnegative()
});

export type CvTailorLogEntry = z.infer<typeof cvTailorLogEntrySchema>;

export const attentionEventSchema = z.object({
	module_id: z.string(),
	duration_ms: z.number().nonnegative(),
	timestamp: z.string()
});
```

### `src/modules/observability/useAttentionTracker.ts`
```ts
"use client";

import { useEffect, useRef } from "react";

type UseAttentionTrackerOptions = {
	moduleId: string;
};

const ATTENTION_FLUSH_INTERVAL_MS = 10000;

export function useAttentionTracker(options: UseAttentionTrackerOptions): void {
	const { moduleId } = options;
	const startTimeRef = useRef<number | null>(null);
	const accumulatedRef = useRef<number>(0);
	const visibilityRef = useRef<boolean>(true);
	const intervalRef = useRef<number | null>(null);

	useEffect(() => {
		startTimeRef.current = performance.now();

		const handleVisibilityChange = () => {
			const now = performance.now();
			if (startTimeRef.current !== null) {
				const delta = now - startTimeRef.current;
				if (visibilityRef.current) {
					accumulatedRef.current += delta;
				}
				startTimeRef.current = now;
			}
			visibilityRef.current = document.visibilityState === "visible";
		};

		const sendAttention = () => {
			if (accumulatedRef.current <= 0) {
				return;
			}
			const durationMs = accumulatedRef.current;
			accumulatedRef.current = 0;

			const payload = JSON.stringify({
				module_id: moduleId,
				duration_ms: durationMs,
				timestamp: new Date().toISOString()
			});

			const backendUrl =
				process.env.NEXT_PUBLIC_BACKEND_URL ?? "/api/v1";
			const url = `${backendUrl}/telemetry/attention`;

			if (typeof navigator !== "undefined" && "sendBeacon" in navigator) {
				const blob = new Blob([payload], { type: "application/json" });
				navigator.sendBeacon(url, blob);
			} else {
				void fetch(url, {
					method: "POST",
					headers: {
						"Content-Type": "application/json"
					},
					body: payload,
					keepalive: true
				}).catch(() => {

				});
			}
		};

		document.addEventListener("visibilitychange", handleVisibilityChange);
		intervalRef.current = window.setInterval(
			sendAttention,
			ATTENTION_FLUSH_INTERVAL_MS
		);

		return () => {
			document.removeEventListener("visibilitychange", handleVisibilityChange);
			if (intervalRef.current !== null) {
				window.clearInterval(intervalRef.current);
			}
			sendAttention();
		};
	}, [moduleId]);
}
```

### `src/modules/pdf-anki/backendTaskErrorI18n.ts`
```ts
/**
 * Legacy prefix constants for tests and worker alignment.
 * Prefer `getUserFacingPdfError` / `classifyPdfAnkiBackendErrorCode` in UI.
 */

import { classifyPdfAnkiBackendErrorCode } from "@/modules/pdf-anki/userFacingPdfError";

export const PDF_ANKI_ERROR_MESSAGE_PREFIX = {
	GROQ_FREE_TIER_BUSY: "GROQ_FREE_TIER_BUSY",
	GROQ_FREE_TIER_EXHAUSTED: "GROQ_FREE_TIER_EXHAUSTED",
} as const;

export type PdfAnkiBackendTaskUiHint =
	| {
			role: "processing_notice";
			i18nKey: "pdfAnki.normError.groq_free_tier_busy.title";
		}
	| {
			role: "failed_user_message";
			i18nKey: "pdfAnki.normError.groq_quota_exceeded.title";
		};

export function resolvePdfAnkiBackendTaskUiHint(
	status: string,
	errorMessage: string | null | undefined,
): PdfAnkiBackendTaskUiHint | null {
	const code = classifyPdfAnkiBackendErrorCode(errorMessage);
	if (!code) {
		return null;
	}
	if (status === "processing" && code === "groq_free_tier_busy") {
		return {
			role: "processing_notice",
			i18nKey: "pdfAnki.normError.groq_free_tier_busy.title",
		};
	}
	if (status === "failed" && code === "groq_quota_exceeded") {
		return {
			role: "failed_user_message",
			i18nKey: "pdfAnki.normError.groq_quota_exceeded.title",
		};
	}
	return null;
}
```

### `src/modules/pdf-anki/constants.ts`
```ts
/**
 * Unauthenticated fallback for PDF–Anki `user_id` query/body params.
 * Keep aligned with `FALLBACK_USER_ID` in `AuthProvider` (NEXT_PUBLIC_USER_ID
 * ?? "anonymous").
 */
export const PDF_ANKI_FALLBACK_USER_ID =
	typeof process !== "undefined"
		? process.env.NEXT_PUBLIC_USER_ID ?? "anonymous"
		: "anonymous";

/**
 * @deprecated Use `PDF_ANKI_FALLBACK_USER_ID` or session `user_id` from auth.
 * Retained name for tests/docs that referenced the old constant.
 */
export const PDF_ANKI_DEFAULT_USER_ID = PDF_ANKI_FALLBACK_USER_ID;

/**
 * Values accepted as `user_tier` in multipart upload (must match backend).
 * `admin`: PAYG-style per-file limits on the worker; LLM pipeline stays on free
 * tier (e.g. Groq free) and LlamaParse `fast`, per product spec.
 */
export type PdfAnkiUserTier =
	| "free"
	| "premium"
	| "pro"
	| "payg"
	| "canceled"
	| "admin";

/**
 * Default upload tier for Storybook/tests only — production uses
 * `mapAuthTierToPdfAnkiUploadTier(me?.tier, me?.subscription_status)`.
 */
export const PDF_ANKI_TEST_DEFAULT_USER_TIER: PdfAnkiUserTier = "free";

/** @deprecated Use `PDF_ANKI_TEST_DEFAULT_USER_TIER` or auth-derived tier. */
export const PDF_ANKI_DEFAULT_USER_TIER = PDF_ANKI_TEST_DEFAULT_USER_TIER;

/**
 * Client-side upload size guard. Files larger than this are rejected before
 * the network request, surfacing a friendlier error than a raw 500.
 * Keep in sync with the backend / proxy body-size limit.
 */
export const PDF_ANKI_MAX_UPLOAD_SIZE_BYTES = 100 * 1024 * 1024; // 100 MB
```

### `src/modules/pdf-anki/isPdfLikeFile.ts`
```ts
/**
 * Some OS/browser combinations leave `File.type` empty or use octet-stream
 * for PDFs; accept a `.pdf` extension (case-insensitive) in those cases.
 */
export function isPdfLikeFile(file: File): boolean {
	if (file.type === "application/pdf") {
		return true;
	}
	const lower = file.name.toLowerCase();
	if (!lower.endsWith(".pdf")) {
		return false;
	}
	return file.type === "" || file.type === "application/octet-stream";
}
```

### `src/modules/pdf-anki/multi-upload-helpers.ts`
```ts
"use client";

import type { FlashcardItem } from "@/modules/pdf-anki/schemas";
import {
	formatPdfPageLimitErrorMessage,
	parsePdfPageLimitError,
} from "@/modules/pdf-anki/pdfPageLimitError";
import { getUserFacingPdfError } from "@/modules/pdf-anki/userFacingPdfError";
import { useI18n } from "@/app/i18n-provider";

export type TaskStatus =
	| "pending"
	| "processing"
	| "completed"
	| "completed_partial"
	| "failed"
	| "error";

export type TaskPollErrorCode =
	| "server_unavailable"
	| "task_not_found"
	| "polling_failed"
	| "max_attempts";

export type UploadTask = {
	taskId: string;
	fileName: string;
	status: TaskStatus;
	deckId: string | null;
	deck: FlashcardItem[] | null;
	errorMessage: string | null;
	errorCode: string | null;
	pollingError: TaskPollErrorCode | null;
	resolvedModelId: string | null;
	createdAt: string;
	updatedAt: string;
};

export type QueueDeck = {
	deckId: string;
	title: string;
	status: string;
	createdAt: string;
	updatedAt: string | null;
	flashcards: FlashcardItem[] | null;
	sourceTaskId: string | null;
	sourceFileName: string | null;
};

export type PreviewDeck = {
	deckId: string;
	title: string;
	status: string;
	createdAt: string;
	flashcards: FlashcardItem[];
	sourceTaskId: string | null;
};

export const POLL_INTERVAL_MS = process.env.NODE_ENV === "test" ? 20 : 3000;
export const MAX_POLLS = 200;

export function localizePdfAnkiTierName(
	tier: string,
	t: ReturnType<typeof useI18n>["t"],
) {
	const normalized = tier.trim().toLowerCase();
	if (normalized === "free") return t("pdfAnki.pricing.tier.free.name");
	if (normalized === "premium") return t("pdfAnki.pricing.tier.premium.name");
	if (normalized === "pro") return t("pdfAnki.pricing.tier.pro.name");
	if (normalized === "canceled") return t("pdfAnki.main.tier.canceled");
	if (normalized === "admin") return t("pdfAnki.main.tier.admin");
	return t("pdfAnki.pricing.tier.free.name");
}

export function buildDayKey(date: Date): string {
	return `${date.getFullYear()}-${date.getMonth() + 1}-${date.getDate()}`;
}

export function isIsoFromDay(iso: string, dayKey: string): boolean {
	if (!iso) {
		return false;
	}
	const parsed = new Date(iso);
	if (Number.isNaN(parsed.getTime())) {
		return false;
	}
	return buildDayKey(parsed) === dayKey;
}

export function stripPdfExtension(fileName: string): string {
	return fileName.replace(/\.pdf$/i, "").trim() || fileName.trim() || "Untitled";
}

export function isTaskCompletedLike(status: string): boolean {
	return status === "completed" || status === "completed_partial";
}

export function isTaskTerminal(status: string): boolean {
	return isTaskCompletedLike(status) || status === "failed" || status === "error";
}

export function formatTaskStatusLabel(status: string, locale: string): string {
	const isSpanish = locale === "es";
	switch (status) {
		case "pending":
			return isSpanish ? "En cola" : "Queued";
		case "processing":
			return isSpanish ? "Procesando" : "Processing";
		case "completed":
			return isSpanish ? "Listo" : "Completed";
		case "completed_partial":
			return isSpanish ? "Parcial" : "Partial";
		case "failed":
			return isSpanish ? "Fallido" : "Failed";
		case "error":
			return isSpanish ? "Error" : "Error";
		default:
			return status;
	}
}

export function formatDeckCompletionLabel(status: string, locale: string): string {
	const isSpanish = locale === "es";
	if (status === "completed_partial") {
		return isSpanish ? "Procesado parcialmente" : "Partially processed";
	}
	if (status === "completed") {
		return isSpanish ? "Procesado correctamente" : "Processed successfully";
	}
	return formatTaskStatusLabel(status, locale);
}

export function formatTaskTime(iso: string, locale: string): string {
	const parsed = new Date(iso);
	if (Number.isNaN(parsed.getTime())) {
		return "";
	}
	return new Intl.DateTimeFormat(locale, {
		hour: "2-digit",
		minute: "2-digit",
	}).format(parsed);
}

export function mergeQueueDecks(
	current: QueueDeck[],
	incoming: QueueDeck[],
	dayKey: string,
): QueueDeck[] {
	const deckMap = new Map<string, QueueDeck>();

	for (const deck of current) {
		if (isIsoFromDay(deck.createdAt, dayKey)) {
			deckMap.set(deck.deckId, deck);
		}
	}

	for (const deck of incoming) {
		if (!isIsoFromDay(deck.createdAt, dayKey)) {
			continue;
		}
		const previous = deckMap.get(deck.deckId);
		deckMap.set(deck.deckId, {
			deckId: deck.deckId,
			title: deck.title || previous?.title || "Untitled",
			status: deck.status || previous?.status || "completed",
			createdAt: deck.createdAt || previous?.createdAt || new Date().toISOString(),
			updatedAt: deck.updatedAt ?? previous?.updatedAt ?? null,
			flashcards: deck.flashcards ?? previous?.flashcards ?? null,
			sourceTaskId: deck.sourceTaskId ?? previous?.sourceTaskId ?? null,
			sourceFileName: deck.sourceFileName ?? previous?.sourceFileName ?? null,
		});
	}

	return Array.from(deckMap.values()).sort((left, right) => {
		return (
			new Date(right.createdAt).getTime() - new Date(left.createdAt).getTime()
		);
	});
}

export function getInlineCopy(locale: string) {
	const isSpanish = locale === "es";
	return {
		latestDeckTitle: isSpanish
			? "Último mazo generado"
			: "Latest generated deck",
		todayQueueTitle: isSpanish
			? "Cola visual de hoy"
			: "Today's visual queue",
		todayQueueSubtitle: isSpanish
			? "Solo se muestran los mazos generados hoy. Pulsa cualquiera para cargar su vista previa."
			: "Only decks generated today are shown here. Select any card to load its preview.",
		todayQueueEmpty: isSpanish
			? "Los mazos que se completen hoy aparecerán aquí."
			: "Decks completed today will appear here.",
		sessionTasksTitle: isSpanish
			? "Procesos de esta sesión"
			: "This session's uploads",
		sessionTasksEmpty: isSpanish
			? "Todavía no has lanzado ningún procesamiento en esta sesión."
			: "No uploads have been started in this session yet.",
		previewLoading: isSpanish ? "Cargando mazo..." : "Loading deck...",
		previewSelectHint: isSpanish
			? "La vista previa cambiará automáticamente cuando termine cualquier PDF nuevo."
			: "The preview updates automatically whenever any new PDF finishes.",
		taskHint: isSpanish
			? "Cada PDF sigue su propio polling sin bloquear el resto."
			: "Each PDF keeps its own polling loop without blocking the others.",
		selectedLabel: isSpanish ? "Seleccionado" : "Selected",
		cardsLabel: isSpanish ? "tarjetas" : "cards",
		taskIdLabel: isSpanish ? "Tarea" : "Task",
	};
}

export function resolvePollingErrorMessage(
	code: TaskPollErrorCode | null,
	t: ReturnType<typeof useI18n>["t"],
): string | null {
	if (code === "server_unavailable") {
		return t("pdfAnki.main.error.serverUnavailable");
	}
	if (code === "task_not_found") {
		return t("pdfAnki.main.error.taskNotFound");
	}
	if (code === "polling_failed") {
		return t("pdfAnki.main.error.pollingFailed");
	}
	if (code === "max_attempts") {
		return t("pdfAnki.main.error.maxAttempts");
	}
	return null;
}

export function resolveTaskFailureMessage(
	task: UploadTask,
	locale: string,
	t: ReturnType<typeof useI18n>["t"],
): string {
	const pollingMessage = resolvePollingErrorMessage(task.pollingError, t);
	if (pollingMessage) {
		return pollingMessage;
	}

	const raw = task.errorMessage?.trim() ?? "";
	if (!raw) {
		return t("pdfAnki.main.error.processingFailed");
	}

	const pageLimit = parsePdfPageLimitError(raw);
	if (pageLimit) {
		return formatPdfPageLimitErrorMessage(pageLimit, locale, t);
	}

	const normalized = getUserFacingPdfError(raw);
	if (normalized) {
		return normalized.title;
	}

	return raw;
}
```

### `src/modules/pdf-anki/pdf-anki-feedback-api.ts`
```ts
import { apiRequest } from "@/lib/api-client";

export type EditFlashcardFeedbackRequest = {
	deck_id: string;
	card_key: string;
	original_front: string;
	original_back: string;
	new_front: string;
	new_back: string;
};

export type EditFlashcardFeedbackResponse = {
	status: "saved";
};

export type DiscardFlashcardFeedbackRequest = {
	deck_id: string;
	card_key: string;
	original_front: string;
	original_back: string;
	rejected: true;
};

export type DiscardFlashcardFeedbackResponse = {
	status: "rejected";
};

/**
 * IMPORTANT:
 * Endpoint paths are relative and will be prefixed by `apiRequest`'s baseUrl
 * (typically `/api/v1`).
 */
const ENDPOINTS = {
	editFlashcardFeedback: "/pdf/deck/flashcards/feedback/edit",
	discardFlashcardFeedback: "/pdf/deck/flashcards/feedback/discard",
} as const;

function normalizeCardText(input: string): string {
	return input.trim().replaceAll("\u0000", "");
}

export async function editFlashcardFeedback(
	request: EditFlashcardFeedbackRequest,
): Promise<EditFlashcardFeedbackResponse> {
	const body: EditFlashcardFeedbackRequest = {
		...request,
		original_front: normalizeCardText(request.original_front).slice(0, 20000),
		original_back: normalizeCardText(request.original_back).slice(0, 20000),
		new_front: normalizeCardText(request.new_front).slice(0, 20000),
		new_back: normalizeCardText(request.new_back).slice(0, 20000),
	};

	return apiRequest<EditFlashcardFeedbackResponse>(
		ENDPOINTS.editFlashcardFeedback,
		{
			method: "POST",
			body,
		},
	);
}

export async function discardFlashcardFeedback(
	request: DiscardFlashcardFeedbackRequest,
): Promise<DiscardFlashcardFeedbackResponse> {
	const body: DiscardFlashcardFeedbackRequest = {
		...request,
		original_front: normalizeCardText(request.original_front).slice(0, 20000),
		original_back: normalizeCardText(request.original_back).slice(0, 20000),
		rejected: true,
	};

	return apiRequest<DiscardFlashcardFeedbackResponse>(
		ENDPOINTS.discardFlashcardFeedback,
		{
			method: "POST",
			body,
		},
	);
}
```

### `src/modules/pdf-anki/PdfAnkiChromeQuota.tsx`
```tsx
"use client";

import { useMemo } from "react";

import { useSupabaseAuth } from "@/app/AuthProvider";
import { usePdfAnkiModelState } from "@/modules/pdf-anki/PdfAnkiModelStateContext";
import { useI18n } from "@/app/i18n-provider";
import {
	getPdfAnkiPaygStorageCaps,
	getPdfAnkiStorageCapsFromMe,
	getPdfAnkiVisibleTierKind,
} from "@/modules/pdf-anki/pdfAnkiStorageCaps";
import type { PdfAnkiChromeTierKind } from "@/modules/pdf-anki/pdfAnkiStorageCaps";
import { usePdfAnkiQuotaBump } from "@/modules/pdf-anki/PdfAnkiQuotaBumpContext";

function tierLabel(kind: PdfAnkiChromeTierKind): string {
	switch (kind) {
		case "guest":
		case "free":
			return "free";
		case "premium":
			return "premium";
		case "pro":
			return "pro";
		case "admin":
			return "admin";
		default:
			return "free";
	}
}

export function PdfAnkiChromeQuota() {
	const { t, locale } = useI18n();
	const { me, isLoading } = useSupabaseAuth();
	const { displayModelId, baseModelId } = usePdfAnkiModelState();
	const { bump } = usePdfAnkiQuotaBump();

	const caps = useMemo(() => {
		if (!me) {
			return getPdfAnkiStorageCapsFromMe(null);
		}
		const walletActive = me.payg === true || me.pdf_anki_payg === true;
		if (walletActive) {
			return getPdfAnkiPaygStorageCaps();
		}
		return getPdfAnkiStorageCapsFromMe(me);
	}, [me]);

	const tierName = tierLabel(getPdfAnkiVisibleTierKind(me));

	const storedRaw = me?.pdf_anki_flashcards_stored;
	const baseStored =
		typeof storedRaw === "number" && Number.isFinite(storedRaw)
			? Math.max(0, Math.trunc(storedRaw))
			: 0;
	const currentCount = Math.min(
		Math.max(0, baseStored + Math.max(0, bump)),
		caps.maxFlashcards,
	);

	const currentFmt = useMemo(
		() =>
			new Intl.NumberFormat(locale, { maximumFractionDigits: 0 }).format(
				currentCount,
			),
		[currentCount, locale],
	);

	const flashFmt = useMemo(
		() =>
			new Intl.NumberFormat(locale, { maximumFractionDigits: 0 }).format(
				caps.maxFlashcards,
			),
		[caps.maxFlashcards, locale],
	);

	const decksFmt = useMemo(
		() =>
			new Intl.NumberFormat(locale, { maximumFractionDigits: 0 }).format(
				caps.maxDecks,
			),
		[caps.maxDecks, locale],
	);

	if (isLoading) {
		return (
			<span className="shrink-0 text-xs text-slate-400" aria-hidden>
				…
			</span>
		);
	}

	const line = t("pdfAnki.chrome.storageLine", {
		tier: tierName,
		current: currentFmt,
		flashcards: flashFmt,
	});

	const tooltip = t("pdfAnki.chrome.storageTooltip", {
		current: currentFmt,
		flashcards: flashFmt,
		decks: decksFmt,
	});

	return (
		<div
			className="min-w-0 max-w-[10rem] shrink text-right sm:max-w-[12rem]"
			title={tooltip}
		>
			<p className="truncate text-[10px] font-medium leading-tight text-slate-800 sm:text-[11px]">
				{line}
			</p>
		</div>
	);
}
```

### `src/modules/pdf-anki/pdfAnkiDeckExport.ts`
```ts
import { ApiError } from "@/lib/api-client";
import { getBackendUrl } from "@/app/auth/backend-auth-api";

export type PdfDeckExportFormat = "csv" | "txt" | "apkg" | "xlsx";

function parseFilenameFromContentDisposition(header: string | null): string | null {
	if (!header) {
		return null;
	}
	const m = /filename\*?=(?:UTF-8''|")?([^";\n]+)"?/i.exec(header);
	if (!m?.[1]) {
		return null;
	}
	try {
		return decodeURIComponent(m[1].trim());
	} catch {
		return m[1].trim();
	}
}

/**
 * Downloads deck export (binary) using the module cookies for auth.
 */
export async function downloadPdfDeckExport(opts: {
	deckId: string;
	userId?: string | null;
	format: PdfDeckExportFormat;
	lang: string;
	tags: string[];
}): Promise<void> {
	const params = new URLSearchParams();
	const userId = opts.userId?.trim();
	if (userId) {
		params.set("user_id", userId);
	}
	params.set("format", opts.format);
	params.set("lang", opts.lang);
	for (const tag of opts.tags) {
		if (tag.trim()) {
			params.append("tag", tag);
		}
	}

	const url = getBackendUrl(`/pdf/deck/${opts.deckId}/export?${params.toString()}`);
	const response = await fetch(url, {
		method: "GET",
		headers: { Accept: "*/*" },
		credentials: "include",
	});
	if (!response.ok) {
		throw new ApiError(
			`Export failed with status ${response.status}`,
			response.status,
		);
	}

	const blob = await response.blob();
	const fromHeader =
		parseFilenameFromContentDisposition(
			response.headers.get("Content-Disposition"),
		) ?? `deck-${opts.deckId}.${opts.format}`;

	const objectUrl = URL.createObjectURL(blob);
	const a = document.createElement("a");
	a.href = objectUrl;
	a.download = fromHeader;
	a.rel = "noopener";
	document.body.appendChild(a);
	a.click();
	a.remove();
	URL.revokeObjectURL(objectUrl);
}
```

### `src/modules/pdf-anki/PdfAnkiDeckQueue.tsx`
```tsx
"use client";

import { Sparkles } from "lucide-react";
import {
	type QueueDeck,
	formatDeckCompletionLabel,
	formatTaskStatusLabel,
	formatTaskTime,
} from "@/modules/pdf-anki/multi-upload-helpers";

type PdfAnkiDeckQueueProps = {
	decks: QueueDeck[];
	locale: string;
	title: string;
	subtitle: string;
	emptyMessage: string;
	selectedDeckId: string | null;
	errorMessage?: string | null;
	selectedLabel: string;
	cardsLabel: string;
	onSelectDeck: (deck: QueueDeck) => void;
};

export function PdfAnkiDeckQueue({
	decks,
	locale,
	title,
	subtitle,
	emptyMessage,
	selectedDeckId,
	errorMessage,
	selectedLabel,
	cardsLabel,
	onSelectDeck,
}: PdfAnkiDeckQueueProps) {
	return (
		<section className="rounded-2xl border border-slate-200 bg-white p-4 shadow-sm">
			<div className="flex flex-wrap items-start justify-between gap-3">
				<div>
					<h2 className="text-sm font-semibold text-slate-900">{title}</h2>
					<p className="mt-1 max-w-xl text-xs leading-relaxed text-slate-500">
						{subtitle}
					</p>
				</div>
				<div className="inline-flex items-center gap-2 rounded-full border border-slate-200 bg-slate-50 px-3 py-1 text-[11px] font-medium text-slate-600">
					<Sparkles className="h-3.5 w-3.5" aria-hidden />
					<span>{decks.length}</span>
				</div>
			</div>

			{decks.length === 0 ? (
				<p className="mt-4 text-sm text-slate-500">{errorMessage ?? emptyMessage}</p>
			) : (
				<div className="mt-4 flex max-h-[28rem] flex-col gap-3 overflow-y-auto pr-1">
					{decks.map((deck) => {
						const isSelected = selectedDeckId === deck.deckId;
						const cardsCount = deck.flashcards?.length ?? 0;
						const secondaryLabel =
							deck.sourceFileName && deck.sourceFileName !== deck.title
								? deck.sourceFileName
								: null;

						return (
							<button
								key={deck.deckId}
								type="button"
								onClick={() => onSelectDeck(deck)}
								className={`w-full rounded-2xl border p-4 text-left shadow-sm transition hover:-translate-y-0.5 hover:shadow-md ${
									isSelected
										? "border-violet-300 bg-[linear-gradient(160deg,rgba(245,243,255,1),rgba(238,242,255,0.95))] ring-2 ring-violet-200"
										: "border-slate-200 bg-[linear-gradient(160deg,rgba(255,255,255,1),rgba(248,250,252,0.92))]"
								}`}
							>
								<div className="flex flex-wrap items-start justify-between gap-3">
									<div className="min-w-0 flex-1">
										<div className="flex flex-wrap items-center gap-2">
											<span className="rounded-full bg-slate-900 px-2.5 py-1 text-[10px] font-semibold uppercase tracking-[0.16em] text-white">
												{formatTaskStatusLabel(deck.status, locale)}
											</span>
											<span className="text-[11px] font-medium text-slate-600">
												{formatDeckCompletionLabel(deck.status, locale)}
											</span>
											{isSelected ? (
												<span className="text-[10px] font-medium text-violet-700">
													{selectedLabel}
												</span>
											) : null}
										</div>
										<p className="mt-3 line-clamp-2 text-sm font-semibold text-slate-900">
											{deck.title}
										</p>
									</div>
									<div className="shrink-0 rounded-2xl border border-slate-200 bg-white/90 px-3 py-2 text-right shadow-inner">
										<p className="text-[10px] font-semibold uppercase tracking-[0.16em] text-slate-500">
											{cardsLabel}
										</p>
										<p className="mt-1 text-lg font-semibold text-slate-900">
											{cardsCount}
										</p>
									</div>
								</div>
								<div className="mt-3 flex items-center justify-between text-[11px] text-slate-500">
									{secondaryLabel ? <span>{secondaryLabel}</span> : <span />}
									<span>{formatTaskTime(deck.createdAt, locale)}</span>
								</div>
							</button>
						);
					})}
				</div>
			)}
		</section>
	);
}
```

### `src/modules/pdf-anki/PdfAnkiErrorAlert.tsx`
```tsx
"use client";

import { useEffect, useRef } from "react";
import { useI18n } from "@/app/i18n-provider";
import {
	type UserFacingPdfError,
	PDF_ERROR_UI_MAX,
} from "@/modules/pdf-anki/userFacingPdfError";
import { reportPdfAnkiNormalizedError } from "@/modules/pdf-anki/pdfAnkiErrorTelemetry";

type PdfAnkiErrorAlertProps = {
	error: UserFacingPdfError;
	variant: "failed" | "notice";
	telemetryRawLength?: number;
};

function clampDisplay(s: string, max: number): string {
	const t = s.trim();
	if (t.length <= max) {
		return t;
	}
	return `${t.slice(0, max - 1)}…`;
}

/**
 * Accessible error / status region for PDF→Anki backend messages.
 */
export function PdfAnkiErrorAlert({
	error,
	variant,
	telemetryRawLength = 0,
}: PdfAnkiErrorAlertProps) {
	const { t } = useI18n();
	const lastTelemetryKey = useRef<string>("");

	useEffect(() => {
		if (!error.code) {
			return;
		}
		const key = `${error.code}:${telemetryRawLength}`;
		if (lastTelemetryKey.current === key) {
			return;
		}
		lastTelemetryKey.current = key;
		reportPdfAnkiNormalizedError({
			code: error.code,
			rawLength: telemetryRawLength,
		});
	}, [error.code, telemetryRawLength]);

	const resolvedTitle =
		error.code === "backend_plain"
			? clampDisplay(error.title, PDF_ERROR_UI_MAX)
			: t(`pdfAnki.normError.${error.code}.title`);

	const resolvedDetail = error.detail
		? clampDisplay(error.detail, Math.max(0, PDF_ERROR_UI_MAX - resolvedTitle.length - 1))
		: undefined;

	const isNotice = variant === "notice";
	const boxClass = isNotice
		? "border-amber-200/90 bg-amber-50 text-amber-950 shadow-sm"
		: "border-rose-200/90 bg-rose-50 text-rose-950 shadow-sm";

	return (
		<div
			role={isNotice ? "status" : "alert"}
			aria-live={isNotice ? "polite" : "assertive"}
			className={`max-w-md rounded-xl border px-3 py-2 text-left text-xs leading-relaxed ${boxClass}`}
		>
			<p className="font-medium">{resolvedTitle}</p>
			{resolvedDetail ? (
				<p className="mt-1 text-[11px] opacity-90">{resolvedDetail}</p>
			) : null}
		</div>
	);
}
```

### `src/modules/pdf-anki/pdfAnkiErrorTelemetry.ts`
```ts
/**
 * Best-effort analytics: code + raw length only (no full error text).
 */

export type PdfAnkiErrorTelemetryPayload = {
	code: string;
	raw_length: number;
	ts: string;
};

export function reportPdfAnkiNormalizedError(payload: {
	code: string;
	rawLength: number;
}): void {
	if (typeof window === "undefined") {
		return;
	}

	const backendUrl =
		process.env.NEXT_PUBLIC_BACKEND_URL ?? "http://localhost:8000/api/v1";
	const base = backendUrl.endsWith("/") ? backendUrl.slice(0, -1) : backendUrl;
	const url = `${base}/telemetry/pdf_anki_error`;

	const body: PdfAnkiErrorTelemetryPayload = {
		code: payload.code,
		raw_length: payload.rawLength,
		ts: new Date().toISOString(),
	};

	const json = JSON.stringify(body);

	if (
		typeof navigator !== "undefined" &&
		typeof navigator.sendBeacon === "function"
	) {
		const blob = new Blob([json], { type: "application/json" });
		navigator.sendBeacon(url, blob);
		return;
	}

	if (typeof fetch === "function") {
		void fetch(url, {
			method: "POST",
			headers: { "Content-Type": "application/json" },
			body: json,
			keepalive: true,
		}).catch(() => {
			/* ignore */
		});
	}
}
```

### `src/modules/pdf-anki/PdfAnkiFlashcard.tsx`
```tsx
"use client";

import { useEffect, useState } from "react";
import { useI18n } from "@/app/i18n-provider";
import type { FlashcardItem } from "@/modules/pdf-anki/schemas";
import {
	discardFlashcardFeedback,
	editFlashcardFeedback,
} from "@/modules/pdf-anki/pdf-anki-feedback-api";

type PdfAnkiFlashcardProps = {
	card: FlashcardItem;
	cardKey: string;
	deckId: string;
	onDiscarded?: (cardKey: string) => void | Promise<void>;
	hideActions?: boolean;
};

/**
 * Renders one flashcard as plain text from the API (no client-side decoding).
 * Back face is revealed with an Anki-style flip on tap/click.
 */
export function PdfAnkiFlashcard({
	card,
	cardKey,
	deckId,
	onDiscarded,
	hideActions = false,
}: PdfAnkiFlashcardProps) {
	const { t } = useI18n();
	const [flipped, setFlipped] = useState(false);

	const [isEditing, setIsEditing] = useState(false);
	const [editFront, setEditFront] = useState(card.front);
	const [editBack, setEditBack] = useState(card.back);
	const [isSaving, setIsSaving] = useState(false);
	const [feedbackError, setFeedbackError] = useState<string | null>(null);
	const [localFront, setLocalFront] = useState(card.front);
	const [localBack, setLocalBack] = useState(card.back);

	useEffect(() => {
		setFlipped(false);
		setIsEditing(false);
		setIsSaving(false);
		setFeedbackError(null);
		setLocalFront(card.front);
		setLocalBack(card.back);
		setEditFront(card.front);
		setEditBack(card.back);
	}, [card.front, card.back, cardKey]);

	const handleSaveEdit = async (): Promise<void> => {
		setFeedbackError(null);
		const trimmedFront = editFront.trim();
		const trimmedBack = editBack.trim();
		if (!trimmedFront || !trimmedBack) return;

		setIsSaving(true);
		try {
			await editFlashcardFeedback({
				deck_id: deckId,
				card_key: cardKey,
				original_front: card.front,
				original_back: card.back,
				new_front: trimmedFront,
				new_back: trimmedBack,
			});
			setLocalFront(trimmedFront);
			setLocalBack(trimmedBack);
			setIsEditing(false);
		} catch {
			setFeedbackError(t("pdfAnki.card.feedback.saveEditError"));
		} finally {
			setIsSaving(false);
		}
	};

	const handleDiscard = async (): Promise<void> => {
		setFeedbackError(null);
		setIsSaving(true);
		try {
			await discardFlashcardFeedback({
				deck_id: deckId,
				card_key: cardKey,
				original_front: card.front,
				original_back: card.back,
				rejected: true,
			});
			await Promise.resolve(onDiscarded?.(cardKey));
			setIsEditing(false);
		} catch {
			setFeedbackError(t("pdfAnki.card.feedback.discardError"));
		} finally {
			setIsSaving(false);
		}
	};

	return (
		<article className="flex flex-col gap-2">
			<div className="relative min-h-[168px] [perspective:1100px]">
				<button
					type="button"
					className="relative min-h-[168px] w-full cursor-pointer rounded-2xl border border-slate-200 bg-transparent p-0 text-left shadow-sm outline-none transition duration-200 hover:-translate-y-0.5 hover:border-slate-300 hover:shadow-md focus-visible:ring-2 focus-visible:ring-violet-400 focus-visible:ring-offset-2"
					aria-expanded={flipped}
					aria-label={t("pdfAnki.card.flipButtonAria")}
					onClick={() => setFlipped((v) => !v)}
				>
					<div
						className="relative min-h-[168px] w-full duration-500 [transform-style:preserve-3d] motion-reduce:transition-none"
						style={{
							transform: flipped ? "rotateY(180deg)" : "rotateY(0deg)"
						}}
					>
						<div
							className="absolute inset-0 flex min-h-[168px] flex-col rounded-2xl border border-slate-100 bg-slate-50 p-4 [backface-visibility:hidden]"
							aria-hidden={flipped}
						>
							<span className="text-[10px] font-semibold uppercase tracking-wide text-slate-500">
								{t("pdfAnki.card.frontHeading")}
							</span>
							<p className="mt-2 min-h-0 flex-1 overflow-auto whitespace-pre-wrap text-sm leading-relaxed text-slate-800">
								{localFront}
							</p>
							<span className="mt-2 text-[10px] text-slate-400">
								{t("pdfAnki.card.flipHint")}
							</span>
						</div>
						<div
							className="absolute inset-0 flex min-h-[168px] flex-col rounded-2xl border border-violet-100 bg-violet-50/90 p-4 [backface-visibility:hidden] [transform:rotateY(180deg)]"
							aria-hidden={!flipped}
						>
							<span className="text-[10px] font-semibold uppercase tracking-wide text-indigo-700">
								{t("pdfAnki.card.backHeading")}
							</span>
							<p className="mt-2 min-h-0 flex-1 overflow-auto whitespace-pre-wrap text-sm leading-relaxed text-slate-800">
								{localBack}
							</p>
							<span className="mt-2 text-[10px] text-indigo-600/80">
								{t("pdfAnki.card.flipBackHint")}
							</span>
						</div>
					</div>
				</button>
			</div>
			{card.tags.length > 0 ? (
				<div className="rounded-xl border border-slate-100 bg-white/80 px-3 py-2 shadow-sm">
					<p className="text-[10px] font-semibold uppercase tracking-wide text-slate-500">
						{t("pdfAnki.card.tagsLabel")}
					</p>
					<ul className="mt-1.5 flex flex-wrap gap-1.5">
						{card.tags.map((tag) => (
							<li key={`${cardKey}-${tag}`}>
								<span className="inline-block rounded-full bg-slate-200 px-2.5 py-0.5 text-[11px] text-slate-800">
									{tag}
								</span>
							</li>
						))}
					</ul>
				</div>
			) : null}

			{!hideActions && (
				<div className="flex flex-wrap gap-2">
					<button
						type="button"
						onClick={() => setIsEditing(true)}
						disabled={isSaving}
						className="rounded-lg border border-slate-200 bg-white px-3 py-1 text-[11px] font-medium text-slate-700 shadow-sm transition hover:-translate-y-0.5 hover:bg-slate-50 disabled:cursor-not-allowed disabled:opacity-50"
					>
						{t("pdfAnki.card.feedback.editButton")}
					</button>

					<button
						type="button"
						onClick={() => void handleDiscard()}
						disabled={isSaving}
						className="rounded-lg bg-rose-50 px-3 py-1 text-[11px] font-medium text-rose-700 shadow-sm transition hover:-translate-y-0.5 hover:bg-rose-100 disabled:cursor-not-allowed disabled:opacity-50"
					>
						{t("pdfAnki.card.feedback.discardButton")}
					</button>
				</div>
			)}

			{feedbackError ? (
				<p className="text-[11px] text-rose-600">{feedbackError}</p>
			) : null}

			{!hideActions && isEditing ? (
				<div className="space-y-2 rounded-xl border border-slate-200 bg-slate-50 p-4 shadow-sm">
					<div className="space-y-1">
						<p className="text-[11px] font-semibold text-slate-700">
							{t("pdfAnki.card.feedback.frontLabel")}
						</p>
						<textarea
							value={editFront}
							onChange={(e) => setEditFront(e.target.value)}
							rows={3}
							className="w-full resize-none rounded-md border border-slate-300 bg-white px-2 py-1 text-xs shadow-sm outline-none focus:border-violet-400"
							disabled={isSaving}
						/>
					</div>
					<div className="space-y-1">
						<p className="text-[11px] font-semibold text-slate-700">
							{t("pdfAnki.card.feedback.backLabel")}
						</p>
						<textarea
							value={editBack}
							onChange={(e) => setEditBack(e.target.value)}
							rows={3}
							className="w-full resize-none rounded-md border border-slate-300 bg-white px-2 py-1 text-xs shadow-sm outline-none focus:border-violet-400"
							disabled={isSaving}
						/>
					</div>
					<div className="flex items-center justify-end gap-2">
						<button
							type="button"
							onClick={() => {
								setIsEditing(false);
								setFeedbackError(null);
								setEditFront(localFront);
								setEditBack(localBack);
							}}
							disabled={isSaving}
							className="rounded-lg border border-slate-200 bg-white px-3 py-1 text-[11px] font-medium text-slate-700 shadow-sm transition hover:-translate-y-0.5 hover:bg-slate-50 disabled:cursor-not-allowed disabled:opacity-50"
						>
							{t("pdfAnki.card.feedback.cancelButton")}
						</button>
						<button
							type="button"
							onClick={() => void handleSaveEdit()}
							disabled={isSaving}
							className="rounded-lg bg-violet-600 px-3 py-1 text-[11px] font-medium text-white shadow-sm transition hover:bg-violet-500 disabled:cursor-not-allowed disabled:opacity-50"
						>
							{isSaving
								? t("pdfAnki.card.feedback.savingButton")
								: t("pdfAnki.card.feedback.saveButton")}
						</button>
					</div>
				</div>
			) : null}
		</article>
	);
}
```

### `src/modules/pdf-anki/pdfAnkiInsufficientBalanceCopy.ts`
```ts
import type { AuthMeResponse } from "@/app/auth/backend-auth-api";

type Translate = (key: string, vars?: Record<string, string>) => string;

/**
 * Tier-included unified model hint applies when PDF–Anki wallet flag is on
 * (`pdf_anki_payg` on `pdf_anki_users`, not Supabase subscription tier) and
 * the user is premium/pro on the PDF–Anki product.
 */
function shouldShowPdfAnkiTierIncludedModelHint(
	me: AuthMeResponse | null,
): boolean {
	if (me?.pdf_anki_payg !== true) {
		return false;
	}
	const tier = me.tier?.toLowerCase().trim() ?? "";
	return tier === "premium" || tier === "pro";
}

/**
 * Extra copy for insufficient-balance failures (tier-included model, recharge).
 */
export function buildPdfAnkiInsufficientBalanceDetail(
	me: AuthMeResponse | null,
	t: Translate,
): string {
	const parts: string[] = [];
	parts.push(t("pdfAnki.error.insufficientBalance.lead"));
	if (shouldShowPdfAnkiTierIncludedModelHint(me)) {
		const included = me?.pdf_anki_tier_included_unified_model_id?.trim();
		if (included) {
			parts.push(
				t("pdfAnki.error.insufficientBalance.includedModel", { id: included }),
			);
		}
	}
	parts.push(t("pdfAnki.error.insufficientBalance.actions"));
	return parts.join(" ");
}
```

### `src/modules/pdf-anki/PdfAnkiModelStateContext.tsx`
```tsx
"use client";

import {
	createContext,
	useContext,
	useEffect,
	useMemo,
	useState,
	type ReactNode,
} from "react";

import { useAuthMe } from "@/app/AuthProvider";
import { getPdfAnkiTierBaseModelId } from "@/app/auth/backend-auth-api";

type PdfAnkiModelStateValue = {
	selectedModelId: string | null;
	resolvedModelId: string | null;
	fallbackModelId: string | null;
	displayModelId: string | null;
	baseModelId: string | null;
	paygBalanceOverride: string | null;
	setSelectedModelId: (value: string | null) => void;
	setResolvedModelId: (value: string | null) => void;
	setFallbackModelId: (value: string | null) => void;
	setPaygBalanceOverride: (value: string | null) => void;
	clearResolvedModelId: () => void;
	dismissFallbackNotice: () => void;
	showFallbackNotice: boolean;
	hasFallbackToTier: boolean;
};

const PdfAnkiModelStateContext =
	createContext<PdfAnkiModelStateValue | null>(null);

function normalizeModelId(value: string | null | undefined): string | null {
	const trimmed = value?.trim() ?? "";
	return trimmed === "" ? null : trimmed;
}

function sameModelId(a: string | null, b: string | null): boolean {
	if (!a || !b) {
		return false;
	}
	return a.trim().toLowerCase() === b.trim().toLowerCase();
}

export function PdfAnkiModelStateProvider({
	children,
}: {
	children: ReactNode;
}) {
	const me = useAuthMe();
	const backendSelectedModelId = normalizeModelId(
		me?.pdf_anki_flashcard_llm_model_id,
	);
	const baseModelId = getPdfAnkiTierBaseModelId(me);
	const [selectedModelId, setSelectedModelId] = useState<string | null>(
		backendSelectedModelId,
	);
	const [resolvedModelId, setResolvedModelIdState] = useState<string | null>(
		null,
	);
	const [fallbackModelId, setFallbackModelIdState] = useState<string | null>(
		null,
	);
	const [paygBalanceOverride, setPaygBalanceOverride] = useState<string | null>(
		null,
	);
	const [showFallbackNotice, setShowFallbackNotice] = useState(false);

	useEffect(() => {
		setSelectedModelId(backendSelectedModelId);
	}, [backendSelectedModelId]);

	useEffect(() => {
		if (me == null) {
			setResolvedModelIdState(null);
			setFallbackModelIdState(null);
			setPaygBalanceOverride(null);
			setShowFallbackNotice(false);
		}
	}, [me]);

	useEffect(() => {
		if (!fallbackModelId || !paygBalanceOverride) {
			return;
		}
		const serverRaw = me?.balance_usd;
		if (serverRaw == null || serverRaw === "") {
			return;
		}
		const nextBalance =
			typeof serverRaw === "string" ? Number.parseFloat(serverRaw) : serverRaw;
		const currentBalance = Number.parseFloat(paygBalanceOverride);
		if (
			Number.isFinite(nextBalance) &&
			Number.isFinite(currentBalance) &&
			nextBalance > currentBalance
		) {
			setFallbackModelIdState(null);
			setPaygBalanceOverride(null);
			setShowFallbackNotice(false);
		}
	}, [fallbackModelId, me?.balance_usd, paygBalanceOverride]);

	const displayModelId = fallbackModelId ?? resolvedModelId ?? selectedModelId;
	const hasFallbackToTier = useMemo(() => {
		if (!me) {
			return false;
		}
		const effectiveResolved = fallbackModelId ?? resolvedModelId;
		if (!baseModelId || !selectedModelId || !effectiveResolved) {
			return false;
		}
		if (!sameModelId(selectedModelId, effectiveResolved)) {
			return sameModelId(effectiveResolved, baseModelId);
		}
		return false;
	}, [baseModelId, fallbackModelId, me, resolvedModelId, selectedModelId]);

	const value = useMemo<PdfAnkiModelStateValue>(
		() => ({
			selectedModelId,
			resolvedModelId,
			fallbackModelId,
			displayModelId,
			baseModelId,
			paygBalanceOverride,
			setSelectedModelId,
			setResolvedModelId: (next: string | null) => {
				setResolvedModelIdState(normalizeModelId(next));
			},
			setFallbackModelId: (next: string | null) => {
				const normalized = normalizeModelId(next);
				setFallbackModelIdState(normalized);
				setShowFallbackNotice(normalized != null);
			},
			setPaygBalanceOverride,
			clearResolvedModelId: () => {
				setResolvedModelIdState(null);
			},
			dismissFallbackNotice: () => {
				setShowFallbackNotice(false);
			},
			showFallbackNotice,
			hasFallbackToTier,
		}),
		[
			baseModelId,
			displayModelId,
			fallbackModelId,
			hasFallbackToTier,
			paygBalanceOverride,
			resolvedModelId,
			selectedModelId,
			showFallbackNotice,
		],
	);

	return (
		<PdfAnkiModelStateContext.Provider value={value}>
			{children}
		</PdfAnkiModelStateContext.Provider>
	);
}

export function usePdfAnkiModelState(): PdfAnkiModelStateValue {
	const ctx = useContext(PdfAnkiModelStateContext);
	if (ctx == null) {
		throw new Error(
			"usePdfAnkiModelState must be used within PdfAnkiModelStateProvider",
		);
	}
	return ctx;
}

export function isPdfAnkiModelPaygSelection(
	modelId: string | null | undefined,
	baseModelId: string | null | undefined,
): boolean {
	const current = normalizeModelId(modelId);
	const base = normalizeModelId(baseModelId);
	if (!current) {
		return false;
	}
	if (!base) {
		return true;
	}
	return !sameModelId(current, base);
}
```

### `src/modules/pdf-anki/PdfAnkiQuotaBumpContext.tsx`
```tsx
"use client";

import {
	createContext,
	useCallback,
	useContext,
	useEffect,
	useRef,
	useState,
	type ReactNode,
} from "react";

import { useAuthMe } from "@/app/AuthProvider";

type PdfAnkiQuotaBumpContextValue = {
	/** Extra flashcards to add on top of `me.pdf_anki_flashcards_stored` (parse UX). */
	bump: number;
	addParseFlashcardCount: (n: number) => void;
};

const PdfAnkiQuotaBumpContext = createContext<
	PdfAnkiQuotaBumpContextValue | undefined
>(undefined);

export function PdfAnkiQuotaBumpProvider({ children }: { children: ReactNode }) {
	const me = useAuthMe();
	const [bump, setBump] = useState(0);
	const prevStoredRef = useRef<number | undefined>(undefined);

	useEffect(() => {
		const cur = me?.pdf_anki_flashcards_stored;
		if (typeof cur !== "number" || !Number.isFinite(cur)) {
			prevStoredRef.current = cur;
			return;
		}
		const prev = prevStoredRef.current;
		if (typeof prev === "number" && cur > prev) {
			const gained = cur - prev;
			setBump((b) => Math.max(0, b - gained));
		}
		prevStoredRef.current = cur;
	}, [me?.pdf_anki_flashcards_stored]);

	const addParseFlashcardCount = useCallback((n: number) => {
		const v = Math.trunc(n);
		if (v > 0) {
			setBump((b) => b + v);
		}
	}, []);

	const value: PdfAnkiQuotaBumpContextValue = {
		bump,
		addParseFlashcardCount,
	};

	return (
		<PdfAnkiQuotaBumpContext.Provider value={value}>
			{children}
		</PdfAnkiQuotaBumpContext.Provider>
	);
}

export function usePdfAnkiQuotaBump(): PdfAnkiQuotaBumpContextValue {
	const ctx = useContext(PdfAnkiQuotaBumpContext);
	if (ctx === undefined) {
		throw new Error("usePdfAnkiQuotaBump requires PdfAnkiQuotaBumpProvider");
	}
	return ctx;
}
```

### `src/modules/pdf-anki/pdfAnkiStorageCaps.ts`
```ts
import type { AuthMeResponse } from "@/app/auth/backend-auth-api";

/**
 * PDF–Anki FIFO storage ceilings for the module chrome.
 *
 * **Precedence:** If `GET /auth/me` includes both `pdf_anki_max_flashcards` and
 * `pdf_anki_max_decks` (positive integers), those values are used for limits.
 * Otherwise the client falls back to tier-based defaults from `me.tier` /
 * `subscription_status` (grace period).
 *
 * Tier labels in the UI still come from `me.tier` (via `labelKind`).
 */
export type PdfAnkiChromeTierKind =
	| "guest"
	| "free"
	| "premium"
	| "pro"
	| "canceled"
	| "admin";

export type PdfAnkiStorageCaps = {
	maxFlashcards: number;
	maxDecks: number;
	/** Drives localized tier label in the chrome. */
	labelKind: PdfAnkiChromeTierKind;
};

function capsPremium(): PdfAnkiStorageCaps {
	return { maxFlashcards: 5_000, maxDecks: 50, labelKind: "premium" };
}

function capsPro(): PdfAnkiStorageCaps {
	return { maxFlashcards: 20_000, maxDecks: 200, labelKind: "pro" };
}

/** Internal tier: Pro-equivalent storage caps; label stays “Admin” in chrome. */
function capsAdmin(): PdfAnkiStorageCaps {
	return { maxFlashcards: 20_000, maxDecks: 200, labelKind: "admin" };
}

function capsFreeLike(kind: PdfAnkiChromeTierKind): PdfAnkiStorageCaps {
	return { maxFlashcards: 100, maxDecks: 5, labelKind: kind };
}

export function getPdfAnkiPaygStorageCaps(): PdfAnkiStorageCaps {
	return { maxFlashcards: 50_000, maxDecks: 500, labelKind: "free" };
}

function parseBackendPdfAnkiStorageCaps(
	me: AuthMeResponse,
): Pick<PdfAnkiStorageCaps, "maxFlashcards" | "maxDecks"> | null {
	const fc = me.pdf_anki_max_flashcards;
	const dk = me.pdf_anki_max_decks;
	if (typeof fc !== "number" || typeof dk !== "number") {
		return null;
	}
	if (!Number.isFinite(fc) || !Number.isFinite(dk)) {
		return null;
	}
	const maxFlashcards = Math.trunc(fc);
	const maxDecks = Math.trunc(dk);
	if (maxFlashcards < 1 || maxDecks < 1) {
		return null;
	}
	return { maxFlashcards, maxDecks };
}

function resolvePdfAnkiTierCaps(me: AuthMeResponse | null): PdfAnkiStorageCaps {
	if (!me) {
		return capsFreeLike("guest");
	}

	const raw = me.tier?.trim().toLowerCase() ?? "";
	const sub = me.subscription_status?.trim().toLowerCase() ?? "";
	const inGracePeriod = sub === "canceled" || sub === "cancelled";

	if (inGracePeriod) {
		if (raw === "pro") {
			return capsPro();
		}
		if (raw === "premium") {
			return capsPremium();
		}
	}

	switch (raw) {
		case "pro":
			return capsPro();
		case "premium":
			return capsPremium();
		case "admin":
			return capsAdmin();
		case "canceled":
			return capsFreeLike("canceled");
		default:
			return capsFreeLike("free");
	}
}

/**
 * Storage limits for the PDF–Anki header: backend caps when present, else
 * tier defaults. `labelKind` always follows `me.tier` (and grace rules).
 */
export function getPdfAnkiStorageCapsFromMe(
	me: AuthMeResponse | null,
): PdfAnkiStorageCaps {
	const tierCaps = resolvePdfAnkiTierCaps(me);
	if (!me) {
		return tierCaps;
	}
	const fromApi = parseBackendPdfAnkiStorageCaps(me);
	if (fromApi) {
		return { ...fromApi, labelKind: tierCaps.labelKind };
	}
	return tierCaps;
}

export function getPdfAnkiVisibleTierKind(
	me: AuthMeResponse | null,
): PdfAnkiChromeTierKind {
	if (!me) {
		return "guest";
	}

	const raw = me.tier?.trim().toLowerCase() ?? "";
	const sub = me.subscription_status?.trim().toLowerCase() ?? "";
	const inGracePeriod = sub === "canceled" || sub === "cancelled";

	if (inGracePeriod) {
		if (raw === "pro") {
			return "pro";
		}
		if (raw === "premium") {
			return "premium";
		}
	}

	switch (raw) {
		case "free":
			return "free";
		case "premium":
			return "premium";
		case "pro":
			return "pro";
		case "admin":
			return "admin";
		case "canceled":
			return "canceled";
		default:
			return "free";
	}
}
```

### `src/modules/pdf-anki/PdfAnkiTaskSessionList.tsx`
```tsx
"use client";

import { LoaderCircle } from "lucide-react";
import {
	type UploadTask,
	formatTaskStatusLabel,
	formatTaskTime,
	resolveTaskFailureMessage,
} from "@/modules/pdf-anki/multi-upload-helpers";
import { useI18n } from "@/app/i18n-provider";

type PdfAnkiTaskSessionListProps = {
	tasks: UploadTask[];
	activeCount: number;
	locale: string;
	title: string;
	subtitle: string;
	emptyMessage: string;
};

export function PdfAnkiTaskSessionList({
	tasks,
	activeCount,
	locale,
	title,
	subtitle,
	emptyMessage,
}: PdfAnkiTaskSessionListProps) {
	const { t } = useI18n();

	return (
		<section className="rounded-2xl border border-slate-200 bg-white p-4 shadow-sm">
			<div className="flex flex-wrap items-start justify-between gap-3">
				<div>
					<h2 className="text-sm font-semibold text-slate-900">{title}</h2>
					<p className="mt-1 text-xs text-slate-500">{subtitle}</p>
				</div>
				{activeCount > 0 ? (
					<div className="inline-flex items-center gap-2 rounded-full border border-amber-200 bg-amber-50 px-3 py-1 text-[11px] font-medium text-amber-800">
						<LoaderCircle className="h-3.5 w-3.5 animate-spin" aria-hidden />
						<span>{activeCount}</span>
					</div>
				) : null}
			</div>

			{tasks.length === 0 ? (
				<p className="mt-4 text-sm text-slate-500">{emptyMessage}</p>
			) : (
				<div className="mt-4 space-y-3">
					{tasks.map((task) => {
						const failedMessage =
							task.status === "failed" || task.status === "error"
								? resolveTaskFailureMessage(task, locale, t)
								: null;

						return (
							<div
								key={task.taskId}
								className="rounded-2xl border border-slate-200 bg-slate-50 p-3"
							>
								<div className="flex flex-wrap items-center justify-between gap-2">
									<p className="text-sm font-medium text-slate-900">
										{task.fileName}
									</p>
									<span
										className={`rounded-full px-2.5 py-1 text-[10px] font-semibold uppercase tracking-[0.16em] ${
											task.status === "completed"
												? "bg-emerald-100 text-emerald-700"
												: task.status === "completed_partial"
													? "bg-amber-100 text-amber-700"
													: task.status === "failed" || task.status === "error"
														? "bg-rose-100 text-rose-700"
														: "bg-slate-200 text-slate-700"
										}`}
									>
										{formatTaskStatusLabel(task.status, locale)}
									</span>
								</div>
								<p className="mt-2 text-[11px] text-slate-500">
									{formatTaskTime(task.updatedAt, locale)}
								</p>
								{failedMessage ? (
									<p className="mt-2 text-xs text-rose-600">{failedMessage}</p>
								) : null}
							</div>
						);
					})}
				</div>
			)}
		</section>
	);
}
```

### `src/modules/pdf-anki/PdfAnkiWalletModelRow.tsx`
```tsx
"use client";

import { useCallback, useMemo, useState } from "react";

import { useSupabaseAuth } from "@/app/AuthProvider";
import {
	formatPdfAnkiBalanceUsd,
	getPdfAnkiTierBaseModelId,
	isPdfAnkiWalletUser,
	patchPdfAnkiUnifiedModel,
} from "@/app/auth/backend-auth-api";
import { useI18n } from "@/app/i18n-provider";
import { ApiError } from "@/lib/api-client";
import { usePdfAnkiModelState } from "@/modules/pdf-anki/PdfAnkiModelStateContext";

function hasModelPicker(
	options: string[] | null | undefined,
): options is string[] {
	return Array.isArray(options) && options.length > 0;
}

function normalizeModelId(value: string | null | undefined): string | null {
	const trimmed = value?.trim() ?? "";
	return trimmed === "" ? null : trimmed;
}

function uniqueModelIds(values: Array<string | null | undefined>): string[] {
	const seen = new Set<string>();
	const output: string[] = [];
	for (const value of values) {
		const normalized = normalizeModelId(value);
		if (!normalized) {
			continue;
		}
		const key = normalized.toLowerCase();
		if (seen.has(key)) {
			continue;
		}
		seen.add(key);
		output.push(normalized);
	}
	return output;
}

export function PdfAnkiWalletModelRow() {
	const { t } = useI18n();
	const { me, isLoading, refreshMe } = useSupabaseAuth();
	const {
		fallbackModelId,
		displayModelId,
		baseModelId,
		paygBalanceOverride,
		setSelectedModelId,
		clearResolvedModelId,
		dismissFallbackNotice,
		showFallbackNotice,
	} = usePdfAnkiModelState();
	const [patchError, setPatchError] = useState<string | null>(null);
	const [patching, setPatching] = useState(false);

	const wallet = isPdfAnkiWalletUser(me);
	const balanceFmt = useMemo(() => {
		if (!wallet) {
			return null;
		}
		return formatPdfAnkiBalanceUsd(paygBalanceOverride ?? me?.balance_usd);
	}, [wallet, me?.balance_usd, paygBalanceOverride]);

	const options = me?.pdf_anki_flashcard_llm_model_options;
	const showPicker = wallet && hasModelPicker(options);
	const backendModelId = normalizeModelId(me?.pdf_anki_flashcard_llm_model_id);
	const effectiveId = displayModelId ?? backendModelId ?? "";
	const tierBaseModelId = baseModelId ?? getPdfAnkiTierBaseModelId(me);
	const normalizedFallbackModelId = normalizeModelId(fallbackModelId);
	const fallbackMatchesTierBase =
		normalizedFallbackModelId != null &&
		tierBaseModelId != null &&
		normalizedFallbackModelId.toLowerCase() === tierBaseModelId.toLowerCase();
	const fallbackLocksPicker = normalizedFallbackModelId != null;
	const showInteractivePicker = showPicker && !fallbackLocksPicker;
	const showLockedPicker = showPicker && fallbackLocksPicker && fallbackMatchesTierBase;

	const selectOptions = useMemo(
		() =>
			uniqueModelIds([
				tierBaseModelId,
				...(options ?? []),
				effectiveId,
			]),
		[effectiveId, options, tierBaseModelId],
	);
	const pipelineKind = me?.pdf_anki_parse_pipeline_kind;

	const fixedModelLabel = useMemo(() => {
		if (showInteractivePicker || showLockedPicker || !me) {
			return null;
		}
		const id = normalizedFallbackModelId ?? backendModelId;
		const kind = pipelineKind?.trim();
		if (kind && !normalizedFallbackModelId) {
			const key = `pdfAnki.model.pipeline.${kind}`;
			const translated = t(key);
			if (translated !== key) {
				return translated;
			}
		}
		if (id) {
			return t("pdfAnki.model.fixedWithId", { id });
		}
		return t("pdfAnki.model.fixedPlan");
	}, [
		showInteractivePicker,
		showLockedPicker,
		me,
		normalizedFallbackModelId,
		backendModelId,
		pipelineKind,
		t,
	]);

	const onModelChange = useCallback(
		async (nextId: string) => {
			if (!showInteractivePicker) {
				return;
			}
			const nextModelId = normalizeModelId(nextId);
			if (!nextModelId || nextModelId === effectiveId.trim()) {
				return;
			}
			setPatchError(null);
			setSelectedModelId(nextModelId);
			clearResolvedModelId();
			setPatching(true);
			try {
				await patchPdfAnkiUnifiedModel(nextModelId, { timeoutMs: 45_000 });
				await refreshMe();
			} catch (e) {
				setSelectedModelId(backendModelId);
				clearResolvedModelId();
				if (e instanceof ApiError && e.status === 403) {
					setPatchError(t("pdfAnki.wallet.modelPatch403"));
				} else {
					setPatchError(t("pdfAnki.wallet.modelPatchFailed"));
				}
			} finally {
				setPatching(false);
			}
		},
		[
			showInteractivePicker,
			effectiveId,
			refreshMe,
			t,
			backendModelId,
			clearResolvedModelId,
			setSelectedModelId,
		],
	);

	if (isLoading) {
		return null;
	}

	if (!me) {
		return null;
	}

	if (!wallet && !showPicker && !effectiveId && !pipelineKind) {
		return null;
	}
	return (
		<div className="border-t border-slate-100 bg-slate-50/80 px-4 py-2">
			<div className="mx-auto flex max-w-6xl flex-col gap-2 sm:flex-row sm:flex-wrap sm:items-center sm:justify-end sm:gap-4">
				{wallet ? (
					<p className="text-[11px] font-medium text-slate-700">
						{balanceFmt != null
							? t("pdfAnki.wallet.balance", { amount: balanceFmt })
							: t("pdfAnki.wallet.balanceUnknown")}
					</p>
				) : null}

				{showInteractivePicker ? (
					<div className="flex flex-col gap-1 text-[11px] text-slate-700">
						<label className="flex flex-wrap items-center gap-2">
							<span className="font-medium text-slate-600">
								{t("pdfAnki.wallet.modelLabel")}
							</span>
							<select
								className="max-w-[min(100%,18rem)] rounded-md border border-slate-300 bg-white px-2 py-1 text-[11px] text-slate-900 shadow-sm disabled:opacity-60"
								value={effectiveId}
								disabled={patching}
								onChange={(e) => {
									void onModelChange(e.target.value);
								}}
							>
								{selectOptions.map((id) => (
									<option key={id} value={id}>
										{tierBaseModelId &&
										id.trim().toLowerCase() === tierBaseModelId.trim().toLowerCase()
											? t("pdfAnki.wallet.planModelLabel", { model: id })
											: id}
									</option>
								))}
							</select>
						</label>
					</div>
				) : showLockedPicker ? (
					<div className="flex flex-col gap-1 text-[11px] text-slate-700">
						<label className="flex flex-wrap items-center gap-2">
							<span className="font-medium text-slate-600">
								{t("pdfAnki.wallet.modelLabel")}
							</span>
							<select
								className="max-w-[min(100%,18rem)] rounded-md border border-slate-300 bg-slate-100 px-2 py-1 text-[11px] text-slate-900 shadow-sm opacity-80"
								value={effectiveId}
								disabled
							>
								<option value={effectiveId}>
									{t("pdfAnki.wallet.planModelLabel", { model: effectiveId })}
								</option>
							</select>
						</label>
					</div>
				) : fixedModelLabel ? (
					<p className="text-[11px] text-slate-600">
						<span className="font-medium text-slate-700">
							{t("pdfAnki.wallet.modelLabel")}
						</span>{" "}
						{fixedModelLabel}
					</p>
				) : null}

				{showFallbackNotice ? (
					<p className="max-w-[min(100%,24rem)] text-[11px] text-amber-700" role="status">
						{fallbackMatchesTierBase
							? t("pdfAnki.wallet.modelFallbackNotice", {
									model: tierBaseModelId ?? effectiveId,
								})
							: t("pdfAnki.wallet.modelFallbackDisabledNotice")}
						{" "}
						<button
							type="button"
							className="font-bold leading-none text-amber-700 hover:text-amber-900"
							aria-label={t("pdfAnki.wallet.dismissFallbackNotice")}
							onClick={dismissFallbackNotice}
						>
							x
						</button>
					</p>
				) : null}

				{patchError ? (
					<p className="text-[11px] text-rose-600" role="alert">
						{patchError}
					</p>
				) : null}
			</div>
		</div>
	);
}
```

### `src/modules/pdf-anki/pdfPageLimitError.ts`
```ts
type Translate = (key: string, vars?: Record<string, string>) => string;

export type PdfPageLimitError =
	| {
			kind: "page";
			tier: string;
			limit: number;
			requested: number;
		}
	| {
			kind: "monthly";
			tier: string;
			limit: number;
			currentUsage: number;
			requested: number;
		};

const PDF_PAGE_LIMIT_RE =
	/^(?:PDF_PAGE_LIMIT_EXCEEDED:\s*)?(?:Your tier \(([^)]+)\) allows up to|PDF page limit exceeded for tier '([^']+)': max)\s+(\d+) pages per document\.\s*(?:This PDF has|This PDF has)\s+(\d+) pages\.?$/i;

const PDF_MONTHLY_PAGE_LIMIT_RE =
	/^(?:PDF_MONTHLY_PAGE_LIMIT_EXCEEDED:\s*)?(?:Your tier \(([^)]+)\) allows up to|PDF monthly page limit exceeded for tier '([^']+)': max)\s+(\d+) pages per month\.\s*(?:Current usage=|Current usage=)(\d+),\s*(?:requested=|requested=)(\d+)\.?$/i;

function pickCapturedTier(match: RegExpMatchArray): string {
	return match[1] ?? match[2] ?? "";
}

export function parsePdfPageLimitError(
	raw: string,
): PdfPageLimitError | null {
	const normalized = raw.trim();
	const monthly = normalized.match(PDF_MONTHLY_PAGE_LIMIT_RE);
	if (monthly) {
		return {
			kind: "monthly",
			tier: pickCapturedTier(monthly),
			limit: Number(monthly[3]!),
			currentUsage: Number(monthly[4]!),
			requested: Number(monthly[5]!),
		};
	}
	const page = normalized.match(PDF_PAGE_LIMIT_RE);
	if (page) {
		return {
			kind: "page",
			tier: pickCapturedTier(page),
			limit: Number(page[3]!),
			requested: Number(page[4]!),
		};
	}
	return null;
}

function localizePdfAnkiTierName(
	tier: string,
	t: Translate,
): string {
	const normalized = tier.trim().toLowerCase();
	if (normalized === "free") return t("pdfAnki.pricing.tier.free.name");
	if (normalized === "premium") return t("pdfAnki.pricing.tier.premium.name");
	if (normalized === "pro") return t("pdfAnki.pricing.tier.pro.name");
	if (normalized === "canceled") return t("pdfAnki.main.tier.canceled");
	if (normalized === "admin") return t("pdfAnki.main.tier.admin");
	return t("pdfAnki.pricing.tier.free.name");
}

export function formatPdfPageLimitErrorMessage(
	pageLimit: PdfPageLimitError,
	locale: string,
	t: Translate,
): string {
	const tierLabel = localizePdfAnkiTierName(pageLimit.tier, t);
	if (pageLimit.kind === "monthly") {
		switch (locale) {
			case "es":
				return `Tu plan (${tierLabel}) permite hasta ${pageLimit.limit} páginas al mes. Uso actual=${pageLimit.currentUsage}, solicitadas=${pageLimit.requested}.`;
			case "de":
				return `Dein Tarif (${tierLabel}) erlaubt bis zu ${pageLimit.limit} Seiten pro Monat. Aktueller Verbrauch=${pageLimit.currentUsage}, angefordert=${pageLimit.requested}.`;
			case "it":
				return `Il tuo piano (${tierLabel}) consente fino a ${pageLimit.limit} pagine al mese. Utilizzo attuale=${pageLimit.currentUsage}, richieste=${pageLimit.requested}.`;
			case "fr":
				return `Ton offre (${tierLabel}) permet jusqu'à ${pageLimit.limit} pages par mois. Utilisation actuelle=${pageLimit.currentUsage}, demandées=${pageLimit.requested}.`;
			case "ru":
				return `Ваш тариф (${tierLabel}) позволяет до ${pageLimit.limit} страниц в месяц. Текущее использование=${pageLimit.currentUsage}, запрошено=${pageLimit.requested}.`;
			default:
				return `Your tier (${tierLabel}) allows up to ${pageLimit.limit} pages per month. Current usage=${pageLimit.currentUsage}, requested=${pageLimit.requested}.`;
		}
	}

	switch (locale) {
		case "es":
			return `Tu plan (${tierLabel}) permite hasta ${pageLimit.limit} páginas por documento. Este PDF tiene ${pageLimit.requested} páginas.`;
		case "de":
			return `Dein Tarif (${tierLabel}) erlaubt bis zu ${pageLimit.limit} Seiten pro Dokument. Dieses PDF hat ${pageLimit.requested} Seiten.`;
		case "it":
			return `Il tuo piano (${tierLabel}) consente fino a ${pageLimit.limit} pagine per documento. Questo PDF ha ${pageLimit.requested} pagine.`;
		case "fr":
			return `Ton offre (${tierLabel}) permet jusqu'à ${pageLimit.limit} pages par document. Ce PDF contient ${pageLimit.requested} pages.`;
		case "ru":
			return `Ваш тариф (${tierLabel}) позволяет до ${pageLimit.limit} страниц на документ. В этом PDF ${pageLimit.requested} страниц.`;
		default:
			return `Your tier (${tierLabel}) allows up to ${pageLimit.limit} pages per document. This PDF has ${pageLimit.requested} pages.`;
	}
}
```

### `src/modules/pdf-anki/processingEstimates.ts`
```ts
/**
 * Rough per-page processing hints for the PDF → Anki drop zone.
 * Free / admin: LlamaParse + Groq (multi-step). Pro / premium / payg: Vertex
 * (single multimodal call). Values are order-of-magnitude guides, not SLAs.
 */
export type PdfAnkiPipelineKind = "llama_groq" | "vertex_direct";

export function pdfAnkiPipelineForTier(tier: string): PdfAnkiPipelineKind {
	const normalized = tier.trim().toLowerCase();
	if (normalized === "free" || normalized === "admin") {
		return "llama_groq";
	}
	return "vertex_direct";
}

export function pdfAnkiPageMinuteEstimates(tier: string): {
	plainSeconds: string;
	richMinutes: string;
} {
	void tier;

	return { plainSeconds: "30", richMinutes: "2–3" };
}
```

### `src/modules/pdf-anki/schemas.ts`
```ts
import { z } from "zod";

export const flashcardItemSchema = z.object({
	/** Plain text from API; do not decode as Base64 on the client. */
	front: z.string().min(1).max(50_000),
	back: z.string().min(1).max(100_000),
	tags: z.array(z.string()).max(50).default([])
});

/** Validates optional client-side upload metadata (`user_tier` must match worker). */
export const uploadRequestSchema = z.object({
	user_tier: z.enum([
		"free",
		"premium",
		"pro",
		"payg",
		"canceled",
		"admin",
	]),
});

export const documentStatusResponseSchema = z.object({
	task_id: z.string(),
	status: z.union([
		z.literal("pending"),
		z.literal("processing"),
		z.literal("completed"),
		z.literal("completed_partial"),
		z.literal("failed"),
		z.literal("error")
	]),
	file_hash: z.string().nullable().optional(),
	error_message: z.string().nullable().optional(),
	error_code: z.string().nullable().optional(),
	resolved_model_id: z.string().nullable().optional(),
	payg_wallet_balance_usd: z.string().nullable().optional(),
	payg_fallback_model_id: z.string().nullable().optional(),
	deck: z.array(flashcardItemSchema).nullable().optional(),
	deck_id: z.string().nullable().optional()
});

export const deckListItemSchema = z.object({
	id: z.string().uuid(),
	title: z.string(),
	status: z.string(),
	created_at: z.string(),
	updated_at: z.string().nullable().optional(),
	document_id: z.string().nullable().optional()
});

export const decksListResponseSchema = z.array(deckListItemSchema);

export const deckDetailResponseSchema = z.object({
	id: z.string().uuid(),
	title: z.string(),
	status: z.string(),
	created_at: z.string(),
	flashcards: z.array(flashcardItemSchema)
});

/** GET /pdf/deck/{id}/tags — returns an object containing the deck_id and tags array */
export const deckTagsResponseSchema = z.object({
	deck_id: z.string(),
	tags: z.array(z.string())
});

export const uploadAcceptedResponseSchema = z.object({
	task_id: z.string(),
	status: z.union([
		z.literal("pending"),
		z.literal("processing"),
		z.literal("completed")
	]),
	message: z.string(),
	payg_wallet_balance_usd: z.string().nullable().optional(),
	payg_fallback_model_id: z.string().nullable().optional(),
	already_processed: z.boolean().optional(),
	deck_id: z.string().nullable().optional()
});

export type FlashcardItem = z.infer<typeof flashcardItemSchema>;
export type DocumentStatusResponse = z.infer<
	typeof documentStatusResponseSchema
>;
export type DeckListItem = z.infer<typeof deckListItemSchema>;
export type DeckDetailResponse = z.infer<typeof deckDetailResponseSchema>;
export type UploadAcceptedResponse = z.infer<
	typeof uploadAcceptedResponseSchema
>;
export type UploadRequest = z.infer<typeof uploadRequestSchema>;
```

### `src/modules/pdf-anki/tierMapping.ts`
```ts
import type { PdfAnkiUserTier } from "@/modules/pdf-anki/constants";

const UPLOAD_TIERS: ReadonlySet<string> = new Set([
	"free",
	"premium",
	"pro",
	"payg",
	"canceled",
	"admin",
]);

/**
 * Maps `/auth/me` `tier` (and related hints) to `user_tier` allowed on upload.
 * `admin` is forwarded as `admin` so the worker applies PAYG-style file limits
 * while keeping the free LLM / LlamaParse `fast` pipeline.
 * [TBD] If the backend documents extra tier strings, extend this map explicitly.
 */
export function mapAuthTierToPdfAnkiUploadTier(
	rawTier: string | undefined,
	subscriptionStatus?: string | undefined,
): PdfAnkiUserTier {
	const normalized = rawTier?.trim().toLowerCase() ?? "";
	if (UPLOAD_TIERS.has(normalized)) {
		return normalized as PdfAnkiUserTier;
	}
	const sub = subscriptionStatus?.trim().toLowerCase() ?? "";
	if (sub === "canceled" || sub === "cancelled") {
		return "canceled";
	}
	return "free";
}
```

### `src/modules/pdf-anki/usePdfAnkiAuth.ts`
```ts
import { useAuthModule, useSupabaseAuth } from "@/app/AuthProvider";

/**
 * Resolves identity for PDF–Anki after the module session has hydrated.
 * Backend auth is cookie-based (HttpOnly) — avoid using Bearer tokens in JS.
 */
export function usePdfAnkiResolvedUserId(): {
	isAuthReady: boolean;
	userIdForApi: string | null;
} {
	const authModule = useAuthModule();
	if (authModule !== "pdf_anki") {
		throw new Error(
			"usePdfAnkiResolvedUserId must be used under AuthProvider module=pdf_anki",
		);
	}

	const { userId, isLoading, me } = useSupabaseAuth();

	if (isLoading) {
		return { isAuthReady: false, userIdForApi: null };
	}

	if (me == null) {
		return { isAuthReady: false, userIdForApi: null };
	}

	return {
		isAuthReady: true,
		userIdForApi: userId ?? me.user_id ?? null,
	};
}
```

### `src/modules/pdf-anki/userFacingPdfError.ts`
```ts
/**
 * Normalizes raw backend PDF→Anki error strings into safe user-facing copy.
 * Never surfaces SQL, tracebacks, or provider dumps in the main title.
 */

export const PDF_ERROR_UI_MAX = 300;
export const PDF_ERROR_SNIPPET_MAX = 500;

export type PdfUserFacingErrorCode =
	| "groq_free_tier_busy"
	| "groq_quota_exceeded"
	| "provider_rate_limit"
	| "pdf_anki_insufficient_balance"
	| "internal_persist"
	| "backend_plain"
	| "unknown";

export type UserFacingPdfError = {
	title: string;
	detail?: string;
	code?: string;
	/** Truncated, whitespace-collapsed raw tail for support (collapsible in UI). */
	supportSnippet?: string;
};

const WORKER_PREFIX_RE = /^Worker error:\s*/i;

const TITLE_ES: Record<Exclude<PdfUserFacingErrorCode, "backend_plain">, string> =
	{
		pdf_anki_insufficient_balance:
			"Saldo insuficiente para el modelo elegido: el coste estimado supera tu " +
			"balance en PDF–Anki. Elige un modelo más barato, el incluido en tu plan " +
			"si aplica, o recarga saldo.",
		groq_free_tier_busy:
			"La cola gratuita de Groq está llena. Vuelve a intentarlo en unos " +
			"minutos; tu PDF puede seguir procesándose en segundo plano.",
		groq_quota_exceeded:
			"Has alcanzado el límite diario de Groq. Prueba mañana o mejora tu plan.",
		provider_rate_limit:
			"El proveedor de IA está saturado (límite de velocidad). Espera unos " +
			"minutos e inténtalo de nuevo.",
		internal_persist:
			"Error interno al guardar el resultado. Si persiste, contacta con soporte.",
		unknown:
			"No se pudo completar el procesamiento. Inténtalo de nuevo más tarde.",
	};

export function stripWorkerErrorPrefix(input: string): string {
	let s = input.trim();
	while (WORKER_PREFIX_RE.test(s)) {
		s = s.replace(WORKER_PREFIX_RE, "").trim();
	}
	return s;
}

function collapseWhitespace(s: string): string {
	return s.replace(/\s+/g, " ").trim();
}

export function makePdfErrorSupportSnippet(
	raw: string,
	maxLen: number = PDF_ERROR_SNIPPET_MAX,
): string {
	const collapsed = collapseWhitespace(raw);
	if (collapsed.length <= maxLen) {
		return collapsed;
	}
	return `${collapsed.slice(0, maxLen)}…`;
}

function matchesInternalPersist(s: string): boolean {
	const u = s.toUpperCase();
	return (
		u.includes("INVALIDTEXTREPRESENTATIONERROR") ||
		u.includes("INVALID_TEXT_REPRESENTATION") ||
		u.includes("DOCUMENTSTATE") ||
		u.includes("DOCUMENT_STATE") ||
		u.includes("COMPLETED_PARTIAL") ||
		u.includes("SQLALCHEMY") ||
		u.includes("PSYCOPG") ||
		u.includes("ASYNCPG") ||
		u.includes("TRACEBACK") ||
		u.includes('FILE "') ||
		u.includes("INSERT INTO") ||
		u.includes("UPDATE ") ||
		u.includes("SELECT ") ||
		u.includes("FROM INFORMATION_SCHEMA") ||
		u.includes("OPERATIONALERROR") ||
		u.includes("PROGRAMMINGERROR") ||
		u.includes("INTEGRITYERROR") ||
		u.includes("PENDINGROLLBACK") ||
		u.includes("DEADLOCK") ||
		u.includes("SERIALIZATIONFAILURE") ||
		u.includes("FOREIGN KEY CONSTRAINT") ||
		u.includes("UNIQUE CONSTRAINT") ||
		u.includes("SYNTAX ERROR AT OR NEAR") ||
		u.includes("POSTGRESQL") ||
		u.includes("RELATION \"") ||
		u.includes("DOES NOT EXIST") ||
		u.includes("STATEMENT HAS BEEN TERMINATED")
	);
}

function matchesHeavyTechnicalDump(s: string): boolean {
	const u = s.toUpperCase();
	if (u.includes("TRACEBACK") || u.includes("FILE \"")) {
		return true;
	}
	const lines = s.split(/\r?\n/).length;
	return lines >= 8 && s.length >= 400;
}

function isSafePlainBackendMessage(s: string): boolean {
	if (s.length > 240) {
		return false;
	}
	const lines = s.split(/\r?\n/).length;
	if (lines > 4) {
		return false;
	}
	const u = s.toUpperCase();
	if (
		u.includes("SQL") ||
		u.includes("TRACE") ||
		u.includes("EXCEPTION") ||
		u.includes("ERROR:") ||
		u.includes("0X") ||
		u.includes("STACK") ||
		u.includes("WORKER") ||
		u.includes("GROQ_")
	) {
		return false;
	}
	if (/[<>{}]/.test(s)) {
		return false;
	}
	return true;
}

function clampTitle(s: string, max: number = PDF_ERROR_UI_MAX): string {
	const t = s.trim();
	if (t.length <= max) {
		return t;
	}
	return `${t.slice(0, max - 1)}…`;
}

/**
 * Classifies a non-empty backend error string (after trim + worker prefix strip).
 */
export function classifyPdfAnkiBackendErrorCode(
	raw: string | null | undefined,
): PdfUserFacingErrorCode | null {
	const cleaned = stripWorkerErrorPrefix(String(raw ?? "").trim());
	if (!cleaned) {
		return null;
	}
	const u = cleaned.toUpperCase();

	if (
		u.startsWith("PDF_ANKI_INSUFFICIENT_BALANCE") ||
		/\bPDF_ANKI_INSUFFICIENT_BALANCE\b/.test(u)
	) {
		return "pdf_anki_insufficient_balance";
	}

	if (u.startsWith("GROQ_FREE_TIER_BUSY") || /\bGROQ_FREE_TIER_BUSY\b/.test(u)) {
		return "groq_free_tier_busy";
	}

	if (
		u.includes("GROQ_FREE_TIER_TPD_ALL_KEYS") ||
		u.startsWith("GROQ_FREE_TIER_EXHAUSTED") ||
		/\bGROQ_FREE_TIER_EXHAUSTED\b/.test(u) ||
		(u.includes("GROQ") && /\bTPD\b/.test(u)) ||
		(u.includes("GROQ") && u.includes("DAILY"))
	) {
		return "groq_quota_exceeded";
	}

	if (
		u.includes("RATE_LIMIT") ||
		u.includes("RATE LIMIT") ||
		/\b429\b/.test(u) ||
		u.includes("TPM") ||
		u.includes("TOKENS PER MINUTE") ||
		u.includes("TOO MANY REQUESTS")
	) {
		return "provider_rate_limit";
	}

	if (matchesInternalPersist(cleaned) || matchesHeavyTechnicalDump(cleaned)) {
		return "internal_persist";
	}

	if (isSafePlainBackendMessage(cleaned)) {
		return "backend_plain";
	}

	if (cleaned.length > 120) {
		return "internal_persist";
	}

	return "unknown";
}

function titleForCode(code: PdfUserFacingErrorCode, cleaned: string): string {
	if (code === "backend_plain") {
		return clampTitle(cleaned);
	}
	return TITLE_ES[code];
}

/**
 * Maps raw `error_message` (or job body errors) to a safe user-facing object.
 * Returns `null` when there is nothing to show.
 */
export function getUserFacingPdfError(
	raw: string | null | undefined,
): UserFacingPdfError | null {
	const cleaned = stripWorkerErrorPrefix(String(raw ?? "").trim());
	if (!cleaned) {
		return null;
	}

	const code = classifyPdfAnkiBackendErrorCode(raw);
	if (!code) {
		return null;
	}

	const snippet = makePdfErrorSupportSnippet(cleaned);
	const includeSnippet =
		code === "internal_persist" ||
		code === "unknown" ||
		code === "provider_rate_limit" ||
		(code === "backend_plain" && snippet.length > 80);

	return {
		code,
		title: titleForCode(code, cleaned),
		supportSnippet: includeSnippet ? snippet : undefined,
	};
}
```

### `src/modules/pdf-anki/useTaskPolling.ts`
```ts
import { useEffect, useRef, useState } from "react";

import { ApiError, apiRequest } from "@/lib/api-client";
import {
	type DocumentStatusResponse,
	documentStatusResponseSchema
} from "./schemas";

export type TaskStatus =
	| "idle"
	| "pending"
	| "processing"
	| "completed"
	| "completed_partial"
	| "failed"
	| "error";

/** i18n keys: pdfAnki.main.error.{code} */
export type TaskPollErrorCode =
	| "server_unavailable"
	| "task_not_found"
	| "polling_failed"
	| "max_attempts";

type UseTaskPollingResult = {
	status: TaskStatus;
	data: DocumentStatusResponse | null;
	error: TaskPollErrorCode | null;
};

const POLL_INTERVAL_MS = process.env.NODE_ENV === "test" ? 10 : 3000;
const MAX_POLLS = 200;

export function useTaskPolling(taskId: string | null): UseTaskPollingResult {
	const [status, setStatus] = useState<TaskStatus>("idle");
	const [data, setData] = useState<DocumentStatusResponse | null>(null);
	const [error, setError] = useState<TaskPollErrorCode | null>(null);

	const pollCountRef = useRef<number>(0);

	useEffect(() => {
		if (!taskId) {
			setStatus("idle");
			setData(null);
			setError(null);
			pollCountRef.current = 0;
			return;
		}

		setStatus("pending");
		setData(null);
		setError(null);
		pollCountRef.current = 0;

		let active = true;
		let timeoutId: ReturnType<typeof setTimeout> | null = null;

		const poll = async () => {
			if (!active) {
				return;
			}

			if (pollCountRef.current >= MAX_POLLS) {
				setStatus("error");
				setError("max_attempts");
				return;
			}

			pollCountRef.current += 1;

			try {
				const response = await apiRequest<DocumentStatusResponse>(
					`/pdf/status/${taskId}`
				);

				const parsed = documentStatusResponseSchema.parse(response);
				setData(parsed);
				setStatus(parsed.status as TaskStatus);

				if (
					parsed.status === "completed" ||
					parsed.status === "completed_partial" ||
					parsed.status === "failed" ||
					parsed.status === "error"
				) {
					return;
				}
			} catch (err) {
				setStatus("error");
				if (err instanceof ApiError && err.status === 404) {
					setError("task_not_found");
				} else {
					setError("polling_failed");
				}
				return;
			}

			timeoutId = setTimeout(poll, POLL_INTERVAL_MS);
		};

		poll();

		return () => {
			active = false;
			if (timeoutId !== null) {
				clearTimeout(timeoutId);
			}
		};
	}, [taskId]);

	return { status, data, error };
}
```

### `src/modules/receipt-parser/mergeExtractorPayload.ts`
```ts
/**
 * Mirrors backend `merge_extractor_payload_with_forced_fields` (receipt processor):
 * deep-copies the LLM JSON, then applies shallow root-level overrides from
 * LlamaParse / forced_fields without mutating the original LLM dict.
 *
 * The API exposes only the merged + validated receipt (`model_dump(mode="json")`);
 * raw LLM output is never sent to the client (see docs/receipts/extractor-llm-audit.md).
 */
export function mergeExtractorPayloadWithForcedFields(
	llmData: Record<string, unknown>,
	forcedFields: Record<string, unknown>,
): Record<string, unknown> {
	const merged = cloneJsonObject(llmData);
	for (const [key, value] of Object.entries(forcedFields)) {
		merged[key] = value;
	}
	return merged;
}

function cloneJsonObject(input: Record<string, unknown>): Record<string, unknown> {
	if (typeof structuredClone === "function") {
		return structuredClone(input) as Record<string, unknown>;
	}
	return JSON.parse(JSON.stringify(input)) as Record<string, unknown>;
}
```

### `src/modules/receipt-parser/offline-queue.ts`
```ts
import Dexie, { type Table } from "dexie";

export type ReceiptQueueRecord = {
	id: string;
	imageBase64: string;
	createdAt: Date;
	synced: boolean;
	documentMimeType?: string | null;
	documentFilename?: string | null;
};

class ReceiptQueueDatabase extends Dexie {
	public receipts!: Table<ReceiptQueueRecord, string>;

	public constructor() {
		super("ReceiptQueueDatabase");
		this.version(1).stores({
			receipts: "id, synced"
		});
	}
}

export const receiptQueueDb = new ReceiptQueueDatabase();
```

### `src/modules/receipt-parser/receipt-form-schema.ts`
```ts
import { z } from "zod";

import { receiptSchema } from "@/modules/receipt-parser/schemas";

const DATE_PLACEHOLDERS = new Set(["", "N/A", "N/D"]);

function normalizeComparableDateValue(value: string | null | undefined): string | null {
	if (value == null) return null;
	const trimmed = value.trim();
	if (DATE_PLACEHOLDERS.has(trimmed)) return null;
	if (/^\d{1,2}\/\d{1,2}\/\d{4}\.\s/.test(trimmed)) {
		return trimmed.replace(/\.\s+/, " ");
	}
	return trimmed;
}

function parseComparableDate(value: string | null | undefined): number | null {
	const normalized = normalizeComparableDateValue(value);
	if (!normalized) return null;
	const timestamp = Date.parse(normalized);
	return Number.isNaN(timestamp) ? null : timestamp;
}

export const receiptFormSchema = receiptSchema.superRefine((receipt, ctx) => {
	const invoiceTimestamp = parseComparableDate(receipt.invoice_date);
	const dueTimestamp = parseComparableDate(receipt.due_date);

	if (invoiceTimestamp == null || dueTimestamp == null) {
		return;
	}

	if (dueTimestamp < invoiceTimestamp) {
		ctx.addIssue({
			code: z.ZodIssueCode.custom,
			path: ["due_date"],
			message: "Due date cannot be earlier than invoice date.",
		});
	}
});

export type ReceiptForm = z.infer<typeof receiptFormSchema>;
```

### `src/modules/receipt-parser/schemas.ts`
```ts
import { z } from "zod";

import { normalizeTaxRatePercentString } from "./tax-rate";

/* Zod mirrors backend ReceiptSchema. Process API fields `receipt` /
 * `best_effort_receipt` are post-merge + validated JSON, not raw LLM output
 * (see docs/receipts/extractor-llm-audit.md, FRONTEND_INTEGRATION.md).
 */

/**
 * Maximum base64 length allowed by the client for `/receipts/process`.
 *
 * Backend validates document types strictly; we relax this limit vs the old
 * "JPEG-only" assumption so larger inputs (e.g. pdf / tiff) can pass client
 * pre-validation and be rejected by the backend if needed.
 *
 * Decoded bytes limit is approx ~1 MiB; Base64 length ≈ ceil(n × 4/3).
 */
export const RECEIPT_JPEG_BASE64_MAX_CHARS = Math.ceil((1024 * 1024 * 4) / 3);

const decimalString = z.union([z.string(), z.number()]).transform(String);
const relaxedString = z.preprocess(
	(v) => {
		if (v === null || v === undefined) return "";
		if (typeof v === "string") return v;
		if (typeof v === "number") return String(v);
		return "";
	},
	z.string()
);

const nullableString = z.preprocess(
	(v) => (v === "" ? null : v),
	z.string().nullable().optional()
);

const dateInputString = z.preprocess(
	(v) => {
		if (v === null || v === undefined) return "";
		if (typeof v === "string") return v;
		if (typeof v === "number") return String(v);
		return "";
	},
	z.string()
);

const finiteNumber = z.preprocess(
	(value) => {
		if (typeof value === "number") return Number.isFinite(value) ? value : undefined;
		if (typeof value === "string") {
			const trimmed = value.trim();
			if (!trimmed) return undefined;
			const normalized = trimmed.replace(",", ".");
			const parsed = Number(normalized);
			return Number.isFinite(parsed) ? parsed : undefined;
		}
		return value;
	},
	z.number().nonnegative()
);

function normalizeReceiptItem(value: unknown): Record<string, unknown> {
	const item = value && typeof value === "object" ? (value as Record<string, unknown>) : {};
	return {
		description:
			typeof item.description === "string" && item.description.length > 0
				? item.description
				: "N/A",
		quantity:
			typeof item.quantity === "number" && Number.isFinite(item.quantity)
				? item.quantity
				: 1,
		unit_price: item.unit_price ?? "0",
		total: item.total ?? "0",
		discount_amount: item.discount_amount ?? null,
		tax_rate: normalizeTaxRatePercentString(item.tax_rate, {
			legacyFraction: true,
		}),
	};
}

function normalizeTaxBreakdown(value: unknown): Record<string, unknown> {
	const tax = value && typeof value === "object" ? (value as Record<string, unknown>) : {};
	return {
		tax_rate:
			normalizeTaxRatePercentString(tax.tax_rate, {
				legacyFraction: true,
			}) ?? "0.00",
		base_amount: tax.base_amount ?? "0",
		tax_amount: tax.tax_amount ?? "0",
	};
}

function normalizeIncomingReceipt(value: unknown): unknown {
	if (!value || typeof value !== "object") return value;

	const raw = value as Record<string, unknown>;
	const items = Array.isArray(raw.items) ? raw.items.map(normalizeReceiptItem) : [];
	const taxes = Array.isArray(raw.taxes)
		? raw.taxes.map(normalizeTaxBreakdown)
		: [];

	return {
		issuer_name: raw.issuer_name ?? "",
		payer_name: raw.payer_name ?? "",
		invoice_number: raw.invoice_number ?? "",
		order_number: raw.order_number ?? null,
		nif_cif_ssn: raw.nif_cif_ssn ?? null,
		invoice_date: raw.invoice_date ?? null,
		due_date: raw.due_date ?? null,
		last_4_card_digits: raw.last_4_card_digits ?? "",
		type: raw.type === "ISSUED" ? "ISSUED" : "RECEIVED",
		user_role: raw.user_role === "issuer" ? "issuer" : "payer",
		items,
		subtotal: raw.subtotal ?? "0",
		discount: raw.discount ?? "0",
		tax: raw.tax ?? "0",
		total: raw.total ?? "0",
		currency: raw.currency ?? "N/A",
		taxes,
		billing_id: raw.billing_id ?? null,
		shipping: raw.shipping ?? null,
		payment_received: raw.payment_received ?? null,
		change_due: raw.change_due ?? null,
		id: raw.id ?? null,
		created_at: raw.created_at ?? null,
		updated_at: raw.updated_at ?? null,
		source: raw.source ?? null,
		version: raw.version ?? null,
	};
}

export const receiptItemSchema = z.preprocess(
	normalizeReceiptItem,
	z.object({
		description: relaxedString,
		quantity: finiteNumber,
		unit_price: decimalString,
		total: decimalString,
		discount_amount: decimalString.nullable().optional(),
		tax_rate: decimalString.nullable().optional(),
	})
);

export const taxBreakdownSchema = z.preprocess(
	normalizeTaxBreakdown,
	z.object({
		tax_rate: decimalString,
		base_amount: decimalString,
		tax_amount: decimalString,
	})
);

export const receiptSchema = z.preprocess(
	normalizeIncomingReceipt,
	z.object({
		issuer_name: relaxedString,
		payer_name: relaxedString,
		invoice_number: relaxedString,
		order_number: nullableString,
		nif_cif_ssn: nullableString,
		invoice_date: dateInputString,
		due_date: dateInputString,
		last_4_card_digits: relaxedString,
		type: z.enum(["ISSUED", "RECEIVED"]),
		user_role: z.enum(["issuer", "payer"]),
		items: z.array(receiptItemSchema),
		subtotal: decimalString,
		discount: decimalString,
		tax: decimalString,
		total: decimalString,
		currency: relaxedString,
		taxes: z.array(taxBreakdownSchema),
		billing_id: nullableString,
		shipping: decimalString.nullable().optional(),
		payment_received: nullableString,
		change_due: decimalString.nullable().optional(),
		id: z.string().nullable().optional(),
		created_at: nullableString,
		updated_at: nullableString,
		source: nullableString,
		version: z.preprocess(
			(v) => (v === "" ? null : v),
			z.number().int().min(1)
		)
			.nullable()
			.optional(),
	})
);

export const receiptProcessRequestSchema = z.object({
	image_base64: z
		.string()
		.min(1)
		.max(RECEIPT_JPEG_BASE64_MAX_CHARS),
	user_role: z.enum(["issuer", "payer"]).default("payer"),
	image_hash: z.union([z.string().max(64), z.null()]).optional(),
	document_mime_type: z
		.enum([
			"image/png",
			"image/jpeg",
			"image/webp",
			"image/heic",
			"image/heif",
			"application/pdf",
			"image/tiff",
		])
		.nullable()
		.optional(),
	document_filename: z
		.string()
		.nullable()
		.optional()
		.refine((value) => {
			if (value == null) return true;
			const lower = value.toLowerCase();
			const allowed = [".png", ".jpg", ".jpeg", ".webp", ".heic", ".pdf", ".tif", ".tiff"];
			return allowed.some((ext) => lower.endsWith(ext));
		}, "Unsupported document filename extension."),
});

export const receiptMathValidationErrorSchema = z.object({
	rule: z.string().nullable().optional(),
	field: z.string(),
	item_line: z.number().int().nullable().optional(),
	expected: z.string().nullable().optional(),
	actual: z.string().nullable().optional(),
	message: z.string(),
});

/** Response body for POST /receipts/process (merged + validated receipt only). */
export const receiptProcessResponseSchema = z.object({
	status: z.union([
		z.literal("verified"),
		z.literal("unprocessable"),
		z.literal("requires_human_correction"),
	]),
	receipt: receiptSchema.nullable().optional(),
	error_message: z.string().nullable().optional(),

	best_effort_receipt: z.record(z.string(), z.unknown()).nullable().optional(),
	validation_errors: z
		.array(receiptMathValidationErrorSchema)
		.nullable()
		.optional(),
	/** PAYG: amount debited from wallet for this request (e.g. "0.03"). */
	payg_wallet_charged_usd: z.string().nullable().optional(),
	/** PAYG: wallet balance remaining after this debit (e.g. "4.97"). */
	payg_wallet_balance_usd: z.string().nullable().optional(),
	/** PAYG: present when balance was insufficient and the tier base model was used instead. */
	payg_fallback_model_id: z.string().nullable().optional(),
});

export type ReceiptItem = z.infer<typeof receiptItemSchema>;
export type Receipt = z.infer<typeof receiptSchema>;
export type ReceiptMathValidationError = z.infer<
	typeof receiptMathValidationErrorSchema
>;

export type ReceiptProcessRequest = z.infer<
	typeof receiptProcessRequestSchema
>;

export type ReceiptProcessResponse = z.infer<
	typeof receiptProcessResponseSchema
>;

export type ReceiptProcessStatus = ReceiptProcessResponse["status"];
```

### `src/modules/receipt-parser/tax-rate.ts`
```ts
type TaxRateInput = unknown;

type TaxRateNormalizationOptions = {
	/**
	 * When true, any sub-1 value is treated as a legacy backend fraction
	 * (for example 0.0825 -> 8.25%). Use this for API/history hydration.
	 */
	legacyFraction?: boolean;
};

function normalizeDecimalInput(value: string): string {
	const trimmed = value.trim();
	if (!trimmed) return "";

	const hasComma = trimmed.includes(",");
	const hasDot = trimmed.includes(".");
	if (!hasComma) return trimmed;

	const noThousands = hasDot ? trimmed.replace(/\./g, "") : trimmed;
	return noThousands.replace(",", ".");
}

function divRoundHalfUp(numerator: bigint, divisor: bigint): bigint {
	if (divisor === 0n) return 0n;
	if (numerator === 0n) return 0n;
	if (numerator < 0n) {
		return -divRoundHalfUp(-numerator, divisor);
	}
	const quotient = numerator / divisor;
	const remainder = numerator % divisor;
	const shouldRoundUp = remainder * 2n >= divisor;
	return quotient + (shouldRoundUp ? 1n : 0n);
}

function parseDecimalStringToScaledInteger(
	value: string,
	scale: bigint
): bigint | null {
	const normalized = normalizeDecimalInput(value);
	if (!normalized) return null;

	const sign = normalized.startsWith("-") ? -1n : 1n;
	const unsigned = sign === -1n ? normalized.slice(1) : normalized;

	const dotIdx = unsigned.indexOf(".");
	const intPartStr = dotIdx >= 0 ? unsigned.slice(0, dotIdx) : unsigned;
	const fracPartStr = dotIdx >= 0 ? unsigned.slice(dotIdx + 1) : "";

	if (intPartStr.length > 0 && !/^\d+$/.test(intPartStr)) return null;
	if (fracPartStr.length > 0 && !/^\d+$/.test(fracPartStr)) return null;

	const intPart = intPartStr ? BigInt(intPartStr) : 0n;
	if (fracPartStr.length === 0) {
		return sign * intPart * scale;
	}

	const numerator = BigInt(`${intPartStr || "0"}${fracPartStr}`);
	const denominator = 10n ** BigInt(fracPartStr.length);
	return sign * divRoundHalfUp(numerator * scale, denominator);
}

function shouldTreatAsLegacyFraction(
	normalized: string,
	options: TaxRateNormalizationOptions
): boolean {
	const unsigned = normalized.startsWith("-") ? normalized.slice(1) : normalized;
	const dotIdx = unsigned.indexOf(".");
	if (dotIdx < 0) return false;

	const intPart = unsigned.slice(0, dotIdx);
	const fracPart = unsigned.slice(dotIdx + 1);

	if (options.legacyFraction) {
		return BigInt(intPart || "0") < 1n;
	}

	return intPart === "0" && fracPart.replace(/0+$/, "").length > 2;
}

export function parseTaxRateToBasisPoints(
	value: TaxRateInput,
	options: TaxRateNormalizationOptions = {}
): bigint | null {
	if (value === null || value === undefined) return null;

	const raw = typeof value === "string" ? value : String(value);
	const normalized = normalizeDecimalInput(raw);
	if (!normalized) return null;

	const useLegacyFraction = shouldTreatAsLegacyFraction(normalized, options);
	const scale = useLegacyFraction ? 10000n : 100n;
	return parseDecimalStringToScaledInteger(normalized, scale);
}

export function formatTaxRateBasisPointsToPercentString(
	basisPoints: bigint
): string {
	const sign = basisPoints < 0n ? "-" : "";
	const abs = basisPoints < 0n ? -basisPoints : basisPoints;
	const intPart = abs / 100n;
	const fracPart = abs % 100n;
	return `${sign}${intPart.toString()}.${fracPart.toString().padStart(2, "0")}`;
}

export function formatTaxRateBasisPointsToFractionString(
	basisPoints: bigint
): string {
	const sign = basisPoints < 0n ? "-" : "";
	const abs = basisPoints < 0n ? -basisPoints : basisPoints;
	const intPart = abs / 10000n;
	const fracPart = abs % 10000n;
	let fracStr = fracPart.toString().padStart(4, "0").replace(/0+$/, "");
	if (fracStr.length < 2) {
		fracStr = fracStr.padEnd(2, "0");
	}
	return `${sign}${intPart.toString()}.${fracStr}`;
}

export function normalizeTaxRatePercentString(
	value: TaxRateInput,
	options: TaxRateNormalizationOptions = {}
): string | null {
	const basisPoints = parseTaxRateToBasisPoints(value, options);
	if (basisPoints === null) return null;
	return formatTaxRateBasisPointsToPercentString(basisPoints);
}

export function serializeTaxRatePercentStringToFractionString(
	value: TaxRateInput
): string | null {
	const basisPoints = parseTaxRateToBasisPoints(value);
	if (basisPoints === null) return null;
	return formatTaxRateBasisPointsToFractionString(basisPoints);
}
```

