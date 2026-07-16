# Resenas — Guía de redacción (Spanish client-facing copy)

Applies to **every** string a prospect or client can see: marketing site, analyzer flow,
web report, PDF, onboarding flow, error states, AI prompt templates for post/review
drafts (Phase B). Internal admin UI may be looser but should follow the vocabulary rules.

Enforced by `npm run lint:copy` (PR-03): banned terms below fail CI when found in
`src/copy/` or `src/app/`.

---

## 1. Register and voice

- **Audience:** dueños de PyMEs paraguayas — restaurantes, clínicas, talleres, tiendas.
  Non-technical, on a budget Android, reached via WhatsApp.
- **Register: voseo-neutral.** Prefer constructions that read naturally whether the
  reader "vosea" or "tutea":
  - Shared possessive/objective forms are always safe: **"tu negocio"**, "tus clientes",
    "te llamen", "te escriban".
  - Prefer noun-led or impersonal phrasing over conjugated second person:
    "Informe gratis de tu negocio" ✓ over "Descubrí/Descubre tu informe".
  - Where a direct verb is unavoidable (buttons), use the infinitive or a shared form:
    "Ver mi informe", "Descargar PDF", "Hablar por WhatsApp".
  - Never mix tú and vos conjugations in one surface. Never use **vosotros** or usted-
    formal distance ("su negocio" only in legal pages).
- **Tone:** directo, concreto, de aliado — "esto te está costando clientes y así se
  arregla". Never alarmist, never condescending, no exclamation-mark salesiness, at most
  one "!" per screen.
- **Sentences short.** One idea per sentence. Numbers over adjectives ("3 fotos en el
  último año" beats "pocas fotos").
- **LATAM/Paraguay vocabulary, never Spain-Spanish.** See §3.
- **Outcome framing, never SEO jargon.** Every finding answers: ¿qué gana el negocio?
  (más llamadas, más mensajes de WhatsApp, más gente que llega al local, aparecer cuando
  te buscan). See §2.

## 2. Banned jargon → approved outcome-framed replacements

These are bans on *client-facing* usage; internal code/docs may use the technical terms.

| Prohibido | Usar en su lugar |
|---|---|
| SEO, "SEO local" | "aparecer en Google cuando te buscan", "visibilidad en Google" |
| domain authority / autoridad de dominio | "la confianza que Google le da a tu sitio" (evitar el concepto; preferir resultados: "aparecer más arriba") |
| citations / citaciones | "menciones de tu negocio en otros sitios" |
| SERP / ranking / posicionamiento | "en qué lugar aparecés en Google" → mejor: "que te encuentren antes que a la competencia" |
| keywords / palabras clave | "las palabras que tus clientes buscan" |
| backlinks | "otros sitios que recomiendan el tuyo" (evitar) |
| meta description / etiquetas / tags | "el texto que Google muestra de tu negocio" |
| schema / datos estructurados | "información que ayuda a Google a entender tu negocio" |
| CTR, conversión, funnel, leads | "cuánta gente te contacta", "clientes potenciales", "consultas" |
| indexación / indexar | "que Google pueda mostrar tu página" |
| NAP | "nombre, dirección y teléfono" |
| GBP / Google Business Profile (sigla) | "tu perfil de Google" / "tu ficha de Google" (primera mención: "tu perfil de Google (el que aparece en el mapa)") |
| Core Web Vitals / PageSpeed | "qué tan rápido carga tu página en el celular" |
| engagement | "respuesta de tus clientes" / "interacción" |

**Métricas permitidas en reportes (las que venden):** llamadas, mensajes de WhatsApp,
pedidos de indicaciones ("cómo llegar"), visitas al perfil, reseñas y respuesta a
reseñas, fotos, aparecer en el mapa.

## 3. Vocabulario LATAM/Paraguay (nunca términos de España)

| Prohibido (España) | Correcto |
|---|---|
| ordenador | computadora |
| móvil | celular |
| vale, guay, genial (como muletilla) | listo, perfecto, buenísimo |
| coger (cualquier uso) | tomar, agarrar |
| aquí | acá (o neutro "en este paso") |
| vosotros / os / vuestro | ustedes / les / su — o reformular a "tu" |
| ahora mismo | ahora / ya mismo |
| pinchar / hacer clic en el enlace | tocar el botón / entrar al link |
| gratis → "de gratis" ✗ | "gratis" solo |
| ordenador portátil | notebook |
| escaparate | vidriera |
| conducir tráfico | atraer clientes/visitas |

Moneda: guaraníes (Gs.) si aparece. Teléfonos ejemplo: formato paraguayo (+595 9xx …).
Ciudades ejemplo: Asunción, Ciudad del Este, Encarnación, Luque, San Lorenzo — nunca
Madrid/México DF en ejemplos.

## 4. Reglas estructurales para el reporte

- Cada hallazgo (finding) sigue la plantilla: **[Qué vimos, con número] + [Qué te cuesta,
  en resultado] + [Qué se hace al respecto]**. Ejemplo:
  "Tu perfil tiene 3 fotos y la última es de hace 8 meses. Los negocios con fotos
  recientes reciben más pedidos de 'cómo llegar'. Esto se arregla con un plan simple de
  fotos mensuales."
- Nunca culpar al dueño; el problema es una oportunidad ("te está faltando", no "hiciste
  mal").
- Lo no evaluado se dice sin excusas técnicas: "Aún no evaluamos tu Instagram — podemos
  verlo juntos por WhatsApp."
- CTA de WhatsApp siempre en primera persona del lector: "Quiero mejorar mi puntaje"
  (texto prellenado del wa.me incluye nombre del negocio y puntaje).
- El puntaje se nombra "**Puntaje de presencia local**" (0–100) en todas las superficies.

## 5. Implementation rules (for executing models)

- All copy lives in `src/copy/*.ts` as typed, parameterized blocks (PLAN D2). No literal
  Spanish in components; no LLM-generated text in any prospect-facing surface.
- Parameters are data (counts, names, dates) — never sentence fragments, so blocks stay
  reviewable as complete sentences.
- Pluralization handled explicitly per block (es rules: 1 reseña / 2 reseñas); dates in
  Spanish long form ("8 de julio de 2026"); numbers with Paraguayan formatting (punto de
  miles: 1.500).
- Phase-B AI prompt templates (`src/ai/prompts.ts`) must embed §1–§3 rules verbatim as
  instructions, and generated drafts are linted against §2/§3 banned lists before being
  shown to the operator.

## 6. One-time native-review QA (launch gate — see RISKS.md R10)

1. **When:** after PR-16 merges (all Phase-A copy exists), before launch (PR-17 checklist
   item). Phase-B copy (onboarding + prompt templates) gets a second, smaller pass after
   PR-19/21/23.
2. **Who:** one native Paraguayan Spanish speaker (owner arranges; brief them with this
   guide, especially §2's outcome-framing intent and R8's "true under partial data" rule).
3. **How:** reviewer reads `src/copy/*.ts` rendered via a QA page (`/admin/copy-qa`,
   trivial addition in PR-15) that displays every block with sample params — including
   every finding at each severity, plus the golden-fixture reports end-to-end (web + PDF).
4. **Output:** corrections applied as a single copy-only PR; reviewer signs off in
   `docs/LAUNCH-CHECKLIST.md` (name + date). Copy blocks then get a `copyVersion` bump
   (PDF cache key includes it — PR-14).
5. Any later copy change > typo-level re-triggers a spot review of the changed blocks.
