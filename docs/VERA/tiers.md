Estrategia de tiers

## Campos Stripe
receipt_parser_user_id
language_code
tier


## Free

Flujo:
- Imagen/PDF -> LlamaParse -> markdown -> tokenización PII -> Groq `llama-3.3-70b-versatile` -> JSON -> cifrado -> Supabase.

Límites:
- 30 recibos al mes.
- 5 al día.
- Retención de 3 meses.
- Capacidad máxima almacenada: 100 recibos.
- Máximo 3 versiones por `image_hash`.

Disclaimer:
- el tier gratuito usa OCR/LLM de terceros.
- no debe usarse para documentos especialmente sensibles.

## Tiers de pago: solo Vertex

Premium:
- modelo incluido: `gemini/gemini-2.5-flash-lite`

Pro:
- modelo incluido: `gemini/gemini-2.5-flash`


PAYG:
- solo modelos Vertex seleccionables:
  - `gemini/gemini-2.5-flash-lite`
  - `gemini/gemini-2.5-flash`
  - `gemini/gemini-2.5-pro`

No se usa ningún modelo de Anthropic ni OpenAI en ningún tier de pago.

## Límites por tier

Premium:
- 500 recibos al mes.
- 50 al día.
- Retención de 1 año.
- Capacidad máxima almacenada: 2.000 recibos.
- Máximo 10 versiones por `image_hash`.

Pro:
- 2.000 recibos al mes.
- 200 al día.
- Retención de 5 años.
- Capacidad máxima almacenada: 5.000 recibos.
- Máximo 50 versiones por `image_hash`.

Ultra:
- 200 al día.
- capacidad máxima almacenada: 10.000 recibos.
- máximo 100 versiones por `image_hash`.

PAYG:
- 1.000 recibos al día.
- sin límite mensual fijo.
- la retención depende del tier base.
- máximo 40 versiones por `image_hash`.

## Versionado y correcciones

Cada persistencia en `receipts` cuenta como una fila:
- el parseo inicial.
- cada corrección humana guardada en `POST /receipts/history`.

El bucle interno de auto-corrección del modelo no crea filas nuevas.

## FinOps y precio mínimo sin pérdidas

Fuente oficial de pricing Vertex:
- https://cloud.google.com/vertex-ai/generative-ai/pricing

Precios usados (por millón de tokens):
- Gemini 2.5 Flash Lite: input `$0.10 / 1M`, output `$0.40 / 1M`.
- Gemini 2.5 Flash: input `$0.30 / 1M`, output `$2.50 / 1M`.

Supuesto conservador para un recibo de una sola imagen:
- 1 imagen de entrada equivalente a `1.120` tokens (proxy publicado para Gemini Flash Image).
- `500` tokens de salida JSON.
- hasta `3` intentos pagados por recibo (Self-Correction Loop con `MAX_RETRIES = 3`).

### Premium

Coste máximo por recibo:
- un intento: `(1.120 * 0.10 / 1.000.000) + (500 * 0.40 / 1.000.000) = $0.000312`
- peor caso con 3 intentos: `$0.000936`

Coste máximo mensual:
- `500 * $0.000936 = $0.468`

Precio mínimo recomendado sin pérdidas:
- **$0.99 / mes**

PRECIO FINAL:
- **$2.99 / mes**

Justificación:
- `$0.99` cubre el techo de inferencia incluso contando tres intentos y deja margen operativo.
- Flash-Lite es el modelo más barato del catálogo Vertex, lo que hace Premium el tier de mayor margen relativo.

### Pro

#### Modelo anterior vs modelo nuevo

El tier Pro cambió de `anthropic/claude-haiku-4.5` a `gemini/gemini-2.5-flash`.
Este es el tier que más se ve afectado en el cambio de precio.

Coste máximo por recibo con Claude Haiku 4.5 (modelo anterior):
- Precios Claude Haiku 4.5: input `$1.00 / 1M`, output `$5.00 / 1M`.
- un intento: `(1.120 * 1.00 / 1.000.000) + (500 * 5.00 / 1.000.000) = $0.003620`
- peor caso con 3 intentos: `$0.010860`
- coste máximo mensual (2.000 recibos): `2.000 * $0.010860 = $21.72`
- **Precio mínimo anterior: ~$24.99 / mes**

Coste máximo por recibo con Gemini 2.5 Flash (modelo actual):
- un intento: `(1.120 * 0.30 / 1.000.000) + (500 * 2.50 / 1.000.000) = $0.001586`
- peor caso con 3 intentos: `$0.004758`

Coste máximo mensual:
- `2.000 * $0.004758 = $9.516`

Precio mínimo recomendado sin pérdidas:
- **$9.99 / mes**

Justificación:
- El cambio de Claude Haiku a Gemini 2.5 Flash reduce el coste por recibo de `$0.010860` a `$0.004758` (mejora del 56%).
- El coste máximo mensual cae de `$21.72` a `$9.516`, permitiendo bajar el precio de `$24.99` a `$9.99`.
- Aun así, el precio no puede bajar agresivamente porque el límite mensual es alto (2.000 recibos) y se permiten hasta 3 intentos por recibo.
- `$9.99` es el primer precio comercial simple que cubre el techo `$9.516` sin dejar el plan en pérdidas.
- Si se quisiera bajar Pro por debajo de `$9.99`, habría que reducir el límite mensual de recibos o el número máximo de intentos del flujo de corrección.

Conclusión sobre el cambio de modelo en Pro:
- Gemini 2.5 Flash tiene un coste por token mucho más bajo que Claude Haiku 4.5.
- El gran número de recibos permitidos en Pro (2.000/mes) y los 3 reintentos son los factores dominantes en el precio final.
- La reducción de precio de `$24.99` a `$9.99` es directamente atribuible al cambio de modelo.

## Data lifecycle

Tras cancelar:
- se mantiene la ventana de gracia de 90 días.
- durante ese periodo el usuario baja a límites de cómputo Free, pero conserva acceso y exportación de su histórico.
- al día 91 se aplica poda FIFO hasta el límite del tier Free.
