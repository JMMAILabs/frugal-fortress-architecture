Tiers

## Stripe Customized Fields on Payment Links

pdf_anki_user_id
language_code
tier

## Modelos por tier

Free:
- Flujo legacy: LlamaParse -> chunking -> LanceDB -> Groq `llama-3.3-70b-versatile`.

Admin:
- Mismos límites operativos que Pro.
- Misma infraestructura que Free.

Premium:
- Modelo incluido: `gemini/gemini-2.5-flash-lite` vía Vertex AI.

Pro:
- Modelo incluido: `gemini/gemini-2.5-flash` vía Vertex AI.

PAYG:
- Solo modelos Vertex seleccionables:
  - `gemini/gemini-2.5-flash-lite`
  - `gemini/gemini-2.5-flash`
  - `gemini/gemini-2.5-pro`

## Límites operativos

Free:
- 10 páginas por documento.
- 50 páginas al mes.
- 100 flashcards.

Premium:
- 100 páginas por PDF.
- 1.000 páginas al mes.
- 5.000 flashcards.

Pro:
- 500 páginas por PDF.
- 2.000 páginas al mes.
- 20.000 flashcards.

PAYG:
- 2.000 paginas al día (fair-use)
- 50.000 paginas al mes (fair-use)
- 50.000 flashcards de almacenamiento.

## FinOps y precio mínimo sin pérdidas

Fuente oficial de pricing Vertex:
- https://cloud.google.com/vertex-ai/generative-ai/pricing

Precios usados (por millón de tokens):
- Gemini 2.5 Flash Lite: input `$0.10 / 1M`, output `$0.40 / 1M`.
- Gemini 2.5 Flash: input `$0.30 / 1M`, output `$2.50 / 1M`.

Para el cálculo uso el mismo techo conservador que ya aplica el backend en `estimate_unified_pdf_token_budget()`:
- input máximo por petición: `602.000` tokens.
- output Premium: `500 + 100 * 250 = 25.500` tokens.
- output Pro: `500 + 300 * 250 = 75.500` tokens.

### Premium

Coste máximo por PDF:
- input: `602.000 * 0.10 / 1.000.000 = $0.0602`
- output: `25.500 * 0.40 / 1.000.000 = $0.0102`
- total por PDF: `$0.0704`

Coste máximo mensual:
- el límite mensual es 1.000 páginas y el límite por archivo es 100 páginas.
- el peor caso facturable son 10 PDFs al mes.
- `10 * $0.0704 = $0.704`

Precio mínimo recomendado sin pérdidas:
- **$0.99 / mes**

Precio FINAL:
- **$2.99 / mes**

Justificación:
- `$0.99` cubre el techo de coste LLM calculado con margen de seguridad de redondeo.
- bajar de `$0.70` dejaría el plan expuesto a pérdidas en el peor caso.
- Flash-Lite es significativamente más barato que Flash, lo que hace Premium el tier de mayor margen relativo.



### Pro

#### Modelo anterior vs modelo nuevo

El tier Pro cambió de `anthropic/claude-haiku-4.5` a `gemini/gemini-2.5-flash`.

Coste máximo por PDF con Claude Haiku 4.5 (modelo anterior):
- Precios Claude Haiku 4.5: input `$1.00 / 1M`, output `$5.00 / 1M`.
- input: `602.000 * 1.00 / 1.000.000 = $0.602`
- output: `75.500 * 5.00 / 1.000.000 = $0.3775`
- total por PDF: `$0.9795`
- peor caso mensual (4 PDFs): `4 * $0.9795 = $3.918`
- **Precio mínimo anterior: ~$4.99 / mes**

Coste máximo por PDF con Gemini 2.5 Flash (modelo actual):
- input: `602.000 * 0.30 / 1.000.000 = $0.1806`
- output: `75.500 * 2.50 / 1.000.000 = $0.18875`
- total por PDF: `$0.36935`

Coste máximo mensual:
- el límite mensual es 2.000 páginas y el límite por archivo es 500 páginas.
- el peor caso facturable son 4 PDFs al mes.
- `4 * $0.36935 = $1.4774`

Precio mínimo recomendado sin pérdidas:
- **$4.99 / mes**

PRECIO FINAL ELEGIDO:
- **$4.99 / mes**

Justificación:
- El cambio de Claude Haiku a Gemini 2.5 Flash reduce el coste máximo mensual de `$3.918` a `$1.4774`.
- Esto permite bajar el precio de Pro de `$4.99` a `$1.99` sin incurrir en pérdidas.
- `$1.99` es el primer escalón comercial simple que cubre el techo `$1.4774`.
- Gemini 2.5 Flash tiene output 2× más caro que Flash-Lite pero mantiene las capacidades multimodales a un coste por token muy inferior a Claude Haiku.
- Si se quisiera añadir margen para soporte, almacenamiento o Stripe, ese recargo debe sumarse encima de `$1.99`; no hace falta para cubrir solo inferencia.

## PAYG

Fórmula PAYG:
- `(coste proveedor LLM) * 1.40`

Notas:
- el catálogo PAYG queda restringido a Vertex (solo modelos Gemini).
- Premium y Pro no tiran de wallet si usan su modelo incluido; el wallet solo se usa cuando eligen otro modelo Vertex permitido.

## Privacidad y ciclo de vida

Paid tiers:
- flujo unificado multimodal sobre Vertex.
- no se usan modelos de Anthropic ni OpenAI.

Tras cancelación:
- se mantiene la política actual de retención de 90 días.
- durante la gracia, el usuario conserva sus decks y flashcards almacenadas.
