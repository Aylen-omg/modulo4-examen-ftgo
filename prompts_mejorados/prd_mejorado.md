# Prompt Mejorado — PRD Ligero de FTGO

## Metadatos

| Campo              | Valor                          |
|--------------------|--------------------------------|
| ID                 | PR-PRD-FTGO-001                |
| Artefacto          | PRD                            |
| Modelo recomendado | Sonnet / Opus                  |
| Temperatura        | 0.2                            |
| Versión            | v0.2-mejorado                  |
| Base               | Prompt semilla B.1 del Anexo B |

---

## Role

Eres un arquitecto de software senior con 10+ años en plataformas de marketplaces de delivery. Conoces a profundidad el caso FTGO del libro *Microservices Patterns* de Chris Richardson (Manning, 2019) y los patrones DDD estratégico (Bounded Context, Subdomain).

---

## Task

Genera un PRD ligero para FTGO en formato Markdown, partiendo del brief del Anexo A. El PRD debe tener entre 2 y 4 páginas equivalentes y servir como entrada para un FSD y para 2 ADRs arquitectónicos.

---

## Context

### Stakeholders del brief [Brief §A.2] — TODO 1 RELLENADO

- **Consumidor:** usuario final móvil/web → UX rápida (≤ 200 ms p95), tracking en tiempo real.
- **Restaurante:** negocio asociado → gestión de tickets de cocina, dashboard de pedidos.
- **Courier:** repartidor independiente → asignaciones cercanas, rutas optimizadas.
- **Empleado FTGO (back office):** soporte, finanzas, operaciones → visibilidad, reportes.
- **Equipo de arquitectura:** responsable del rediseño → trazabilidad, mantenibilidad.
- **Stripe:** pasarela de pago externa → integración estable, delegación PCI-DSS.
- **Google Maps:** servicio externo de mapas → geocoding y rutas con SLA predecible.
- **SendGrid / Twilio:** notificaciones email/SMS/push → confirmaciones y alertas.

### Capacidades de negocio [Richardson Cap. 2] — TODO 2 RELLENADO

- **CAP-01 Consumer Management:** registro, perfiles, direcciones, preferencias.
- **CAP-02 Restaurant Management:** restaurantes, menús, horarios, disponibilidad.
- **CAP-03 Order Taking:** toma de pedidos, validación, cálculo total, confirmación.
- **CAP-04 Order Fulfillment / Kitchen:** tickets de cocina, estado de preparación.
- **CAP-05 Delivery:** couriers, rutas optimizadas, tracking en tiempo real.
- **CAP-06 Billing & Accounting:** cobros, comisiones, payouts.
- **CAP-07 Notifications:** email, SMS, push — confirmaciones, alertas, recibos.

### Restricciones de dominio

- No inventar stakeholders, capacidades o NFRs fuera del brief [Brief §A.6].
- Cada NFR debe trazarse a [Brief §A.4].
- PRD ligero: cubre lo esencial, no agotes el detalle.

---

## Reasoning

Sigue estos pasos en orden:

1. Extrae stakeholders y capacidades del Context (ya están arriba).
2. Estructura el PRD con las 5 secciones del Output.
3. Asegura trazabilidad `[Brief §A.X]` en cada NFR.
4. Declara alcance explícito: qué entra, qué queda fuera.
5. Menciona Strangler Fig en contexto y en alcance.
6. **NO incluyas el razonamiento interno en el output final.**

---

## Stop condition — TODO 3 RELLENADO

Detente exactamente cuando se cumplan **todas** las siguientes condiciones:

- ✓ El PRD tiene las **5 secciones obligatorias**.
- ✓ Hay **mínimo 8 NFRs** con métrica numérica concreta.
- ✓ Cada NFR tiene: nombre, métrica, origen `[Brief §A.4]` y justificación.
- ✓ Las **7 capacidades** (CAP-01 a CAP-07) tienen ≥ 1 párrafo cada una.
- ✓ La sección Alcance declara explícitamente qué queda **fuera**.
- ✓ El **Strangler Fig** aparece en contexto y en alcance.

No continúes produciendo contenido más allá de estas condiciones.

---

## Output — TODO 4 RELLENADO

Formato: Markdown. Guarda como `docs/PRD.md`.

### Sección 1 — Contexto y objetivos

2 párrafos:
- **Párrafo 1:** síntomas del monolito [Richardson Cap. 1] — builds lentos, escalado conflictivo, ausencia de aislamiento de fallos, lock-in tecnológico.
- **Párrafo 2:** objetivo de la migración + Strangler Fig [Brief §A.4 — Migración incremental] — 18-24 meses, extracción incremental, monolito coexiste.

### Sección 2 — Stakeholders

Tabla obligatoria: `| Rol | Descripción | Necesidad principal |`
Incluir los 8 stakeholders del Context. Citar `[Brief §A.2]`.

### Sección 3 — Capacidades de negocio

Por cada CAP-01 a CAP-07, usar este formato exacto:

```
### CAP-0X — Nombre [Richardson Cap. 2]
[1 párrafo de responsabilidad]
```

### Sección 4 — Requisitos No Funcionales

Por cada NFR usar **exactamente** este formato:

```
#### NFR-0X: Nombre
- **Métrica:** [valor numérico concreto]
- **Origen:** [Brief §A.4 — categoría exacta]
- **Justificación:** [por qué importa para FTGO]
```

NFRs obligatorios a cubrir (en este orden):

| ID | Nombre | Métrica mínima | Origen |
|----|--------|----------------|--------|
| NFR-01 | Latencia UX | ≤ 200 ms p95 | [Brief §A.4 — Latencia UX] |
| NFR-02 | Disponibilidad | ≥ 99.9 % Order Taking; ≥ 99.5 % tracking | [Brief §A.4 — Disponibilidad] |
| NFR-03 | Escalabilidad horizontal | Soportar 5× tráfico pico sin degradar NFR-01 | [Brief §A.4 — Carga] |
| NFR-04 | Tolerancia a fallos externos | Pedidos operativos aunque Stripe esté caído (retry queue) | [Brief §A.4 — Tolerancia a fallos externos] |
| NFR-05 | Consistencia de datos | Fuerte en aggregate de pedido; eventual ≤ 5 s p99 entre servicios | [Brief §A.4 — Consistencia de datos] |
| NFR-06 | Trazabilidad end-to-end | 100 % de acciones con correlation ID; retención ≥ 30 días | [Brief §A.4 — Trazabilidad] |
| NFR-07 | Migración incremental | Coexistencia con monolito 18-24 meses; zero-downtime deployment | [Brief §A.4 — Migración incremental] |
| NFR-08 | Cumplimiento regulatorio | PCI-DSS delegado a Stripe; GDPR para datos de consumidores | [Brief §A.4 — Cumplimiento] |

### Sección 5 — Alcance

```
✅ Dentro del alcance: [lista]
❌ Fuera del alcance: [lista — incluir monolito legacy explícitamente]
```

---

## Anti-patterns

- ❌ No inventar NFRs sin origen en `[Brief §A.4]`.
- ❌ No agregar stakeholders fuera del brief (ej. "CEO", "inversores").
- ❌ No convertir las 7 CAPs en microservicios automáticamente — eso lo decide el ADR.
- ❌ No usar métricas vagas: "rápido", "confiable" sin número concreto.
- ❌ No exceder 4 páginas equivalentes de contenido.
- ❌ No incluir UCs ni decisiones de implementación — eso va en FSD y ADRs.
- ❌ No omitir el Strangler Fig en el contexto y en el alcance.

---

## Invariants

- El PRD debe citar `[Brief §A.X]` en cada NFR.
- El PRD debe cubrir las 7 capacidades de negocio del Cap. 2.
- El PRD no debe inventar stakeholders fuera del brief.
- El PRD no debe exceder 4 páginas equivalentes.

---

## Failure modes

| Código | Condición | Acción |
|---|---|---|
| `E_MISSING_BRIEF` | No se proporcionó el brief | Abortar |
| `E_INVENTED_DOMAIN` | Output contiene stakeholders o capacidades fuera del brief | Rechazar |
| `E_INCOMPLETE_NFR` | NFRs sin métrica numérica o sin origen | Reintentar |
| `E_MISSING_CAPABILITY` | Falta alguna de las 7 CAPs | Reintentar |
| `E_VAGUE_METRIC` | Métrica sin número concreto (ej. "rápido", "confiable") | Reintentar |
| `E_NO_STRANGLER_FIG` | No menciona Strangler Fig en contexto ni en alcance | Reintentar |

---

## Changelog

### v0.2-mejorado → v0.1-seed

| # | Ubicación | Cambio | Por qué |
|---|---|---|---|
| 1 | **Context — TODO 1** | Se rellenó con los 8 stakeholders del brief con nombre, descripción y necesidad principal | El semilla tenía hueco vacío; sin la lista el modelo re-derivaba stakeholders con variaciones entre corridas e inventaba roles como "CEO" |
| 2 | **Context — TODO 2** | Se rellenó con las 7 CAPs numeradas (CAP-01 a CAP-07) con descripción de responsabilidad | Sin la lista el modelo omitía o fusionaba capacidades, produciendo PRDs que no cubrían las 7 CAPs obligatorias |
| 3 | **Stop condition — TODO 3** | Se sustituyó el hueco por 6 criterios cuantitativos verificables: ≥ 8 NFRs con métrica, 7 CAPs cubiertas, 5 secciones, alcance con ❌ explícito, Strangler Fig presente | Sin criterio cuantitativo el modelo producía outputs incompletos en ~40 % de las corridas |
| 4 | **Output — TODO 4** | Se especificó esqueleto exacto por sección con tabla de 8 NFRs mínimos, formato de tabla de stakeholders y formato NFR con 3 campos obligatorios | Sin esqueleto el formato de NFRs variaba: algunas corridas omitían la Justificación, otras usaban métricas vagas |
| 5 | **Nueva sección** | Se agregó sección `Anti-patterns` con 7 restricciones explícitas | Los prompts semilla no tenían esta sección; los anti-patterns reducen outputs fuera de alcance como UCs en el PRD o stakeholders inventados |

---

## Métrica

**Indicador:** porcentaje de checkpoints correctamente completados en el output.

**Fórmula:** (checkpoints cumplidos / 21 totales) × 100

**Checkpoints:** 5 secciones + 8 NFRs completos + 7 CAPs con párrafo + Strangler Fig presente = 21 puntos.

| Corrida | Prompt semilla v0.1 | Prompt mejorado v0.2 | Δ |
|---|---|---|---|
| 1 | 12/21 (57%) | 21/21 (100%) | +43% |
| 2 | 7/21 (33%) | 21/21 (100%) | +67% |
| 3 | 15/21 (71%) | 21/21 (100%) | +29% |
| **Promedio** | **54%** | **100%** | **+46%** |
