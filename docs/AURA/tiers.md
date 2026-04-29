Los planes Premium/Ultra cambian la logica: en lugar de transcribir con whisper y resumir con llama, ahora se hace todo de una vez con modelos más punteros

📊 Reglas de Negocio (Tiering & UX)

- **Plan FREE (Popularidad)**
  - **Modelo**: Whisper-v3 + Llama 3.3 70B
  - **Duración máxima por audio**: 5 min
  - **Minutos máximos al mes**: 150 min
  - **Historial**: últimas 20 notas (FIFO)
  - **Glosario activo**: hasta 200 reglas, y 3 descargas de glosario/dia.
  - **Correcciones/día**: 10 (audio o texto). Este presupuesto diario se
    comparte entre: (a) añadir nuevas reglas de glosario y (b) activar o
    desactivar reglas existentes desde `/r`
  - **Papelera**: 30 días de retención
  - **Precio**: GRATIS

- **Plan Premium**
  - **Modelo**: Gemini 2.5 Flash Lite
  - **Duración máxima por audio**: 20 min
  - **Minutos máximos al mes**: 3,000 min (50 h)
  - **Historial**: hasta 2,000 notas (FIFO)
  - **Glosario activo**: hasta 300 reglas, y 10 descargas de glosario/dia.
  - **Correcciones/día**: hasta 100. Este presupuesto diario se comparte
    entre: (a) añadir nuevas reglas y (b) activar/desactivar reglas
    existentes (cada toggle consume 1 corrección).
  - **Papelera**: 60 días + soporte manual
  - **Precio**: 4.99 USD / mes

- **Plan Pro**
  - **Modelo**: Gemini 2.5 Flash Lite
  - **Duración máxima por audio**: 45 min
  - **Minutos máximos al mes**: 9,000 min (150 h)
  - **Historial**: hasta 2,000 notas (FIFO)
  - **Glosario activo**: hasta 300 reglas, y 20 descargas de glosario/dia.
  - **Correcciones/día**: hasta 100. Este presupuesto diario se comparte
    entre: (a) añadir nuevas reglas y (b) activar/desactivar reglas
    existentes (cada toggle consume 1 corrección).
  - **Papelera**: 180 días + soporte prioritario
  - **Precio**: 9.99 USD / mes


Plan PAYG (Pay as You Go):
--------------------------
- **Recargas**: mínimo 5 USD por recarga (top‑up).
- **Minutos máximos/mes**: 50,000 min (solo como circuit breaker contra abusos o bugs del cliente).
- **Duración máxima por audio**: límite del context window del modelo, calculado dinámicamente según:
  - modelo seleccionado
  - saldo PAYG actual en USD
  - precios LLM declarados en el `PricingRegistry` (FinOps-first)
- **Historial, correcciones y glosario**: según el tipo de usuario base (free, premium o pro).

Fórmula PAYG: (Coste Total LLM) * 1.40 (40% Markup).

Detalles de Implementación:
Balance disponible actualizado y añadido al pie de pagina de cada respuesta/correccion, para que el usuario sepa su Balance en todo momento cuando interactua con el bot, sin tener que salir de Telegram.

**Caché de resúmenes por hash (Redis):** Al recibir un audio, se calcula un hash SHA-256 del contenido. Si ese mismo audio ya fue procesado, se devuelve el resumen desde Redis (clave `audio_processed:{hash}`) sin volver a transcribir ni llamar al LLM. El TTL del caché depende ahora del plan del usuario: **FREE: 24 horas**, **Premium: 3 días**, **Pro: 1 semana**, **PAYG: 1 semana**.


** Solicitudes de Paginacion (listado de reglas y paginacion) **
Por minuto: 30 solicitudes para todos los tiers.
Por día:
free: 50
premium: 200
pro: 500
payg: 1000


Retención de datos tras cancelar la suscripción (Data Lifecycle)
Retener los datos durante 3 meses (90 días) es el estándar de oro en la industria SaaS (Win-back strategy).
Desde el punto de vista de arquitectura de datos, el ciclo de vida debe ser el siguiente:
Día 0 (Cancelación): Ej: El usuario PREMIUM cancela. Su plan pasa a estado canceled, pero mantiene sus datos intactos.
Día 1 a 90 (Grace Period): El usuario es degradado a los límites de cómputo del Free Tier (solo puede procesar 150 minutos de audio al mes, hasta 5 minutos por audio), pero aún se mantienen en base de datos sus 2000 notas y sus 300 reglas. Esto le da 3 meses para arrepentirse y volver a suscribirse sin perder nada. Mensaje permanente en su pantalla de perfil que le avisa de que debe exportar su glosario antes del dia 90 (calculado y mostrado en fecha real) o actualizar su suscripcion, o perderá las reglas almacenadas (su plan pasara a Free Tier definitivo y únicamente permanecerán las últimas 200 reglas activas añadidas)
Día 91 (Hard Downgrade / Pruning): Un cron job en tu Worker (Arq) detecta que el periodo de gracia ha expirado. En ese momento, aplica un truncado FIFO masivo, reduciendo sus 300 reglas a las 200 reglas activas más recientes (el límite del Free Tier).






PLAN ENTERPRISE (A FUTURO)
-------------------------
USUARIOS ENTERPRISE
B2B Shared Quota
A. Base de Datos: Relación "Self-Referential"

En lugar de crear tablas complejas de "Equipos", simplemente añadiremos una columna parent_account_id a la tabla User.

    Si eres el jefe/pagador (Admin), tu parent_account_id es NULL.

    Si eres un empleado, tu parent_account_id apunta al telegram_id de tu jefe.

    Ventaja: Consultas SQL ultrarrápidas y esquema minimalista.

B. Resolución de Límites (El truco FinOps)

Actualmente, nuestro BudgetManager y RateLimitMiddleware usan el tenant_id para llevar la cuenta en Redis de forma atómica (con scripts Lua).

    El cambio clave: Cuando un usuario interactúa con el bot, el sistema determinará su effective_tenant_id. Si tiene un parent_account_id, el effective_tenant_id será el ID del jefe. Si no, será el suyo propio.

    Resultado: Todos los empleados consumen automáticamente del mismo "cubo" (bucket) de minutos y presupuesto en Redis. No hay race conditions, y no hay que reescribir la lógica de límites.

C. Gestión del Equipo (Team Management)

Necesitaremos casos de uso para que el Admin pueda añadir o eliminar miembros de su cuenta Enterprise. En Telegram, esto se suele hacer generando un enlace de invitación único (Deep Linking, ej: t.me/TuBot?start=invite_xyz) o mediante comandos. El backend solo necesita exponer la lógica para vincular/desvincular IDs.


Evolución del Dominio (Parent Account)(prompt)
Act as a Staff Software Engineer. We are adding a B2B "Shared Quota" feature to our Telegram Audio Notes bot. Enterprise admins can share their subscription limits with their employees.

Context:
- Architecture: Strict Hexagonal.
- Target Module: `audio_notes`
- Existing Entity: `User` in `src/app/modules/audio_notes/domain/entities.py`

Task:
1. Update the `User` SQLAlchemy entity to include a self-referential foreign key:
   - `parent_account_id` (String, nullable, index, ForeignKey to `users.telegram_id`).
2. Add a relationship `team_members` to easily access the employees from an admin user (one-to-many).
3. Add domain logic methods to the `User` class:
   - `get_effective_tier(parent_user: Optional['User'] = None) -> str`: Returns the parent's tier if `parent_account_id` is set, else its own tier.
   - `get_effective_tenant_id() -> str`: Returns `parent_account_id` if set, else `telegram_id`. This is critical for our Redis FinOps tracking.
4. Generate the Alembic migration command instructions in a comment block.

Ensure strict typing (`mypy --strict`). Do not modify infrastructure files yet.


Casos de Uso de Gestión de Equipo (Team Management)

Act as a Staff Software Engineer. We need Application Use Cases to manage the Enterprise Team members.

Context:
- Architecture: Strict Hexagonal.
- Module: `audio_notes`

Task:
1. Create a new file `src/app/modules/audio_notes/application/team_management.py`.
2. Implement `add_team_member_use_case(admin_id: str, employee_id: str, repo: AudioNotesRepositoryPort)`:
   - Validates that `admin_id` has `tier == 'enterprise'`.
   - Validates that `employee_id` does not already have a different `parent_account_id`.
   - Updates the employee's `parent_account_id` to `admin_id`.
3. Implement `remove_team_member_use_case(admin_id: str, employee_id: str, repo: AudioNotesRepositoryPort)`:
   - Sets the employee's `parent_account_id` to None.
4. Update `AudioNotesRepositoryPort` and its implementation (`AudioNotesRepository`) to support fetching a user with their parent account data if needed (e.g., `get_user_with_parent`).

Ensure all database operations use the existing transactional patterns. Handle edge cases (e.g., Admin trying to add another Admin) by raising appropriate DomainErrors.





Enrutamiento de Consumo (FinOps & Audio Processing)
Act as a Staff Software Engineer. We need to update the core audio processing flow so that employees consume their Admin's quota and use the Admin's custom AI models.

Context:
- Architecture: Strict Hexagonal.
- Target File: `src/app/modules/audio_notes/application/process_audio.py`

Task:
1. In `process_audio_use_case`, when fetching the user via `repo.get_or_create_user(user_id)`, also fetch the parent user if `parent_account_id` is set.
2. Determine the `effective_tier` and `effective_model` based on the parent account (if it exists).
3. CRITICAL: When interacting with the `BudgetManager` or `TokenManager` (FinOps), you MUST use the `effective_tenant_id` (the Admin's ID) so that all employees draw from the same shared Redis token bucket.
4. Ensure the `glossary_context` also merges or uses the Admin's glossary rules, so employees benefit from the company's shared dictionary. Update `get_glossary_context` to fetch rules for the `effective_tenant_id`.

Maintain strict async/await patterns and do not block the event loop.





Evolucion del Dominio y Base de Datos

Act as a Staff Software Engineer. We are upgrading our SaaS billing model to support dynamic Enterprise tiers and automated subscription management via Stripe.

Context:
- Architecture: Strict Hexagonal (Ports & Adapters).
- Target Module: `audio_notes`
- Existing Entity: `User` in `src/app/modules/audio_notes/domain/entities.py`

Task:
1. Update the `User` SQLAlchemy entity to include the following new columns:
   - `stripe_customer_id` (String, nullable, index)
   - `stripe_subscription_id` (String, nullable, unique)
   - `subscription_status` (String, default="active") # active, past_due, canceled
   - `current_period_start` (DateTime, nullable)
   - `current_period_end` (DateTime, nullable)
   - `custom_model` (String, nullable)
   - `custom_max_duration_min` (Integer, nullable)
   - `custom_monthly_minutes` (Integer, nullable)
2. Add domain logic methods to the `User` class:
   - `get_audio_model()`: Returns `custom_model` if tier is enterprise, else returns the default model for their tier.
   - `get_monthly_minutes_limit()`: Returns `custom_monthly_minutes` if enterprise, else standard tier limits.
   - `is_subscription_active()`: Returns True if tier is free, OR if subscription_status is 'active' and current_period_end is in the future.
3. Generate the Alembic migration command instructions in a comment block.

Ensure strict typing (`mypy --strict`) and do not leak infrastructure details (like Stripe SDK) into this domain file.



 Adaptador de Stripe y Cotización Dinámica
Act as a Staff Software Engineer. We need to implement the Payment Gateway Port and the Stripe Adapter to generate dynamic checkout sessions for our Enterprise tier.

Context:
- Architecture: Strict Hexagonal.
- We need to calculate prices dynamically based on user selections and create a Stripe Checkout Session using `price_data` (no pre-created Stripe products).

Task:
1. Create `src/app/modules/audio_notes/domain/ports.py` (if not exists) and define a `PaymentGatewayPort` protocol with a method: `create_dynamic_checkout(user_id: str, model: str, max_duration: int, monthly_minutes: int, calculated_price_usd: float) -> str` (returns checkout URL).
2. Create `src/app/modules/audio_notes/infrastructure/stripe_adapter.py` implementing this port.
3. In the implementation, use the official `stripe` python package.
4. CRITICAL: When creating the `stripe.checkout.Session.create`, you MUST inject a `metadata` dictionary containing: `{"user_id": user_id, "tier": "enterprise", "custom_model": model, "custom_max_duration": max_duration, "custom_monthly_minutes": monthly_minutes}`. This is required for stateless provisioning later.
5. Create an Application Use Case `GenerateEnterpriseQuoteUseCase` that takes the user's requested parameters, uses our existing `LiteLLMCostCalculator` to estimate the raw cost, adds a 30% profit margin (plus a $2 base fee), and calls the `PaymentGatewayPort` to return the Stripe URL.

Ensure all I/O is async (use `asyncio.to_thread` for the synchronous Stripe SDK calls).



Webhooks y Dunning (Reintentos)

Act as a Staff Software Engineer. We need to implement the Stripe Webhook handler to automate provisioning and handle dunning (payment failures/retries).

Context:
- Architecture: Strict Hexagonal.
- We rely on Stripe's Smart Retries. We only need to react to state changes.

Task:
1. Create a new FastAPI router endpoint in `src/app/modules/audio_notes/router.py`: `POST /stripe/webhook`.
2. Implement the signature verification using `stripe.Webhook.construct_event`.
3. Create an Application Use Case `ProcessStripeWebhookUseCase` that handles the following event types:
   - `checkout.session.completed`: Extract the `metadata` we injected during checkout. Find the user in the DB, update their `tier` to 'enterprise', set their `stripe_customer_id` and `stripe_subscription_id`, and apply the `custom_*` limits from the metadata. Set status to 'active'.
   - `invoice.payment_succeeded`: Find user by `stripe_subscription_id`. Update `current_period_start` and `current_period_end`. Reset their monthly usage counters. Set status to 'active'.
   - `invoice.payment_failed`: Find user by subscription ID. Update `subscription_status` to 'past_due'. (Do not downgrade yet, allow grace period).
   - `customer.subscription.deleted`: Find user. Downgrade `tier` to 'free', set `subscription_status` to 'canceled', and clear all `custom_*` limits.
4. Ensure database updates are done transactionally using the `AudioNotesRepositoryPort`.

Write clean, modular code with strict typing and robust error handling for missing metadata or unknown users.


Impacto Arquitectónico: Transición a Límites Dinámicos (Enterprise)
Leer el archivo README.md del modulo audio_notes para entender los distintos tiers.

Para soportar el plan Enterprise sin ensuciar la lógica de negocio con múltiples sentencias if/else, debemos aplicar principios de Domain-Driven Design (DDD) en nuestra capa Hexagonal.
A. Evolución de la Entidad User (Domain Layer)

Actualmente, los límites están probablemente hardcodeados en constantes o evaluados en los casos de uso basados en el string tier.
Para soportar Enterprise, la entidad User debe evolucionar para almacenar sus propios límites (overrides):

    Nuevos campos en BD: custom_model, custom_max_duration_min, custom_monthly_minutes.

    Lógica encapsulada: La entidad User tendrá métodos como get_audio_model() o get_monthly_limit(). Si el tier es enterprise, estos métodos devuelven los valores de los campos custom_*. Si es pro o free, devuelven las constantes del sistema.

    Beneficio: El caso de uso (process_audio_use_case) no necesita saber qué tier tiene el usuario; simplemente le pregunta a la entidad User cuál es su límite y su modelo, y se lo pasa al LLMProvider.

B. Herramienta de Presupuestación (Admin / Sales)

Para que los desarrolladores o el equipo de ventas puedan crear el Quote (Presupuesto) Enterprise, no necesitamos construir una UI compleja de inmediato.

    Fase 1 (MVP): Un script en Python en la carpeta scripts/ (ej. scripts/calc_enterprise_quote.py) que acepte por CLI el modelo, los minutos y la duración, consulte los precios en el PriceRegistry (nuestro calculador de FinOps existente) y escupa el coste base + el margen deseado.

    Fase 2: Un endpoint en el admin_adapter.py protegido por RBAC (admin_required) que devuelva esta cotización para integrarlo en un futuro panel de administración interno




Evolución de Puertos: Deberemos crear un nuevo puerto MultimodalSummarizerPort que acepte directamente los bytes del audio y el contexto del glosario.

Enrutamiento Dinámico (Application Layer): El caso de uso process_audio_use_case deberá evaluar el user_tier. Si es free, orquesta WhisperPort -> SummarizerPort. Si es premium o ultra, inyecta el audio directamente en el MultimodalSummarizerPort.

Evolucionar el Puerto (LLMProvider): Modificaremos la firma del método complete (o añadiremos un complete_multimodal) en la capa de Dominio para que acepte un parámetro opcional audio_bytes: bytes | None.

Capa de Aplicación: El caso de uso decidirá: si es Free, llama a Whisper y luego a llm.complete(text). Si es Pro/Ultra/Enterprise, llama directamente a llm.complete(audio_bytes=audio_bytes). El adaptador de LiteLLM se encargará de codificar en Base64 y armar el payload JSON.


Stripe solo le cobre al jefe (Admin).

Redis descuente los minutos y tokens de la "cuenta" del jefe, sin importar qué empleado envíe el audio (gracias al effective_tenant_id).

El Glosario se comparta a nivel de empresa (si el jefe corrige una palabra técnica, la IA la aprenderá para todos los empleados).