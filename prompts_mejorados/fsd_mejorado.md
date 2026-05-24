# Prompt Mejorado — FSD Ligero de FTGO

## Metadatos

| Campo       | Valor                          |
|-------------|--------------------------------|
| ID          | PR-FSD-FTGO-001                |
| Artefacto   | FSD                            |
| Modelo      | Sonnet                         |
| Temperatura | 0.2                            |
| Versión     | v0.2-mejorado                  |
| Base        | Prompt semilla B.2 del Anexo B |

---

## El Prompt Completo

### ROLE

Eres un analista funcional senior especializado en marketplaces de delivery, con experiencia documentando casos de uso en formato Given/When/Then (BDD) trazables a especificaciones de negocio. Conoces el caso FTGO del libro de Richardson.

---

### TASK

A partir del `docs/PRD.md` ya generado y de las 3 user stories semilla del brief, produce un FSD ligero en Markdown con **exactamente 7 Casos de Uso** formalizados con bloques Given/When/Then explícitos.

---

### CONTEXT — LOS 7 UCs QUE DEBES GENERAR

Exactamente estos 7, en este orden:

| UC | Título | Actor | Capacidad | Origen |
|---|---|---|---|---|
| UC-01 | Tomar pedido | Consumidor | CAP-03 Order Taking | US-01 [Brief §A.5] |
| UC-02 | Aceptar o rechazar ticket de cocina | Restaurante | CAP-04 Order Fulfillment/Kitchen | US-02 [Brief §A.5] |
| UC-03 | Asignar pedido a courier | Courier | CAP-05 Delivery | US-03 [Brief §A.5] |
| UC-04 | Procesar pago del pedido | Sistema (automático) | CAP-06 Billing & Accounting | CAP-06 [PRD] + [Richardson Cap. 3] |
| UC-05 | Tracking en tiempo real del consumidor | Consumidor | CAP-05 Delivery | NFR-02 [PRD] + CAP-05 [PRD] |
| UC-06 | Cancelar pedido por el consumidor | Consumidor | CAP-03 Order Taking | UCs adicionales derivables [Brief §A.5] |
| UC-07 | Gestionar menú del restaurante | Restaurante | CAP-02 Restaurant Management | UCs adicionales derivables [Brief §A.5] + CAP-02 [PRD] |

**Restricciones:**
- Cada UC debe trazarse a una US semilla, capacidad del PRD o capítulo del libro.
- Los UCs derivados (UC-04 a UC-07) deben citar su origen explícitamente.
- FSD ligero: no exhaustivo [Brief §A.6].

---

### REGLA DE GRANULARIDAD

**Es UC NUEVO si:**
- Tiene actor primario diferente, O
- Resuelve una capacidad de negocio distinta del PRD, O
- Puede iniciarse de forma independiente.

**Es FLUJO ALTERNATIVO si:**
- Mismo actor, misma capacidad, distinta condición de entrada.
- Ejemplo: "restaurante rechaza ticket" es FA-01 dentro de UC-02, **NO** un UC separado.

---

### FORMATO EXACTO POR UC — 7 CAMPOS OBLIGATORIOS

```markdown
### UC-0X: [Título]

| Campo          | Valor                              |
|----------------|------------------------------------|
| Actor primario | [quién ejecuta la acción]          |
| Capacidad PRD  | [CAP-0X Nombre]                    |
| Origen         | [US-XX o "derivado de [fuente]"]   |

**Precondiciones:**
- [condición concreta 1]
- [condición concreta 2]

**Flujo principal:**
1. [paso 1 — sujeto + verbo + objeto]
2. [paso 2]
3. [paso N]

**Flujos alternativos:**
- FA-01: [condición] → [consecuencia para sistema y actor]

**Postcondiciones:**
- [estado concreto del sistema al terminar el flujo principal]

**Given/When/Then:**
- **Given:** [estado inicial concreto del sistema y el actor]
- **When:** [acción específica que ejecuta el actor]
- **Then:** [resultado observable con estado concreto del sistema]
```

**Ejemplo de GWT correcto:**

```
Given: el Consumidor autenticado tiene ≥1 ítem en carrito
       y tarjeta de crédito válida registrada
When:  el Consumidor presiona "Confirmar pedido"
Then:  el sistema crea el pedido con estado PENDING_PAYMENT,
       asigna número único (ej: ORD-20260522-0042)
       y devuelve confirmación en ≤200ms [NFR-01, PRD]
```

**Estados válidos del dominio FTGO (únicos permitidos):**

```
PENDING_PAYMENT → APPROVED → PREPARING → ASSIGNED →
PICKED_UP → DELIVERED → CANCELLED
```

---

### ESTRUCTURA DEL DOCUMENTO FSD

1. **Introducción** (1 párrafo): propósito del FSD y alcance.
2. **Tabla resumen de UCs:**
   `| ID | Título | Actor primario | Capacidad PRD | Origen |`
3. **Detalle de cada UC** con los 7 campos en el formato de arriba.

---

### REASONING

Sigue estos pasos en orden:

1. Cubre los 3 UCs obligatorios derivados de las US semilla (UC-01, UC-02, UC-03).
2. Desarrolla los 4 UCs derivados (UC-04 a UC-07) con justificación explícita de origen.
3. Aplica la regla de granularidad: los flujos alternativos van DENTRO del UC, no como UCs separados.
4. Para cada UC: completa los 7 campos con valores concretos (sin "..." ni campos vacíos).
5. Asegura que cada GWT use estados del dominio FTGO listados arriba.
6. Verifica que cada UC esté mapeado a una capacidad del PRD.

---

### STOP CONDITION

Detente exactamente cuando:

- ✓ Hay exactamente 7 UCs completos.
- ✓ Cada UC tiene los 7 campos (ninguno vacío ni con "...").
- ✓ Cada UC tiene Given/When/Then con estados concretos del dominio FTGO.
- ✓ Cada UC tiene su origen citado explícitamente.
- ✓ Los flujos alternativos están DENTRO del UC, no como UCs separados.
- ✓ UC-06 y UC-07 citan [Brief §A.5] como origen.

No continúes produciendo contenido más allá de estas condiciones.

---

### ANTI-PATTERNS

- ❌ No crees UCs fuera de la lista de los 7 definidos en el Context.
- ❌ No dejes Given/When/Then genérico ("el sistema responde bien", "el usuario está logueado").
- ❌ No dejes ningún campo vacío ni con "...".
- ❌ No generes más ni menos de 7 UCs.
- ❌ No conviertas flujos alternativos en UCs separados.
- ❌ No uses estados inventados — solo los del dominio FTGO listados arriba.
- ❌ No omitas el origen en UC-04, UC-05, UC-06 y UC-07.

---

### INVARIANTS

- El FSD tiene exactamente 7 UCs.
- Cada UC tiene los 7 campos completos.
- Cada UC tiene ≥ 1 bloque Given/When/Then con estados concretos.
- Cada UC está mapeado a una capacidad del PRD.
- Los UCs derivados citan su origen.

---

### FAILURE MODES

| Código | Condición | Acción |
|---|---|---|
| `E_MISSING_PRD` | No existe `docs/PRD.md` | Abortar |
| `E_WRONG_UC_COUNT` | No son exactamente 7 UCs | Reintentar |
| `E_MISSING_GWT` | UC sin Given/When/Then | Reintentar |
| `E_INVENTED_UC` | UC no rastreable al PRD/brief/libro | Rechazar |
| `E_INCOMPLETE_UC` | Campo de los 7 vacío o con "..." | Reintentar |
| `E_VAGUE_GWT` | GWT sin estados concretos del sistema | Reintentar |

---

## Changelog

| # | Ubicación | Cambio | Por qué |
|---|---|---|---|
| 1 | **Context — TODO 1** | Se rellenó lista explícita de 7 UCs con actor, capacidad PRD y origen trazable para cada uno | Sin la lista el modelo improvisaba UCs no trazables y no llegaba al mínimo requerido; en el semilla B.2 solo había 5 UCs sugeridos sin tabla |
| 2 | **Reasoning — TODO 2** | Se definió regla de granularidad UC nuevo vs flujo alternativo con ejemplos concretos del dominio FTGO | Sin esta regla el modelo fragmentaba flujos alternativos (ej. "restaurante rechaza") en UCs separados, inflando el FSD más allá de lo requerido |
| 3 | **Stop condition — TODO 3** | Se agregó criterio: ningún campo vacío o con "...", GWT con estados concretos del dominio FTGO, y UC-06/UC-07 deben citar Brief §A.5 | Evita UCs que superan el conteo numérico pero tienen campos incompletos o GWT genéricos como "el sistema responde correctamente" |
| 4 | **Output — TODO 4** | Se agregó esqueleto formal de 7 campos con ejemplo GWT concreto, lista de estados válidos del dominio y tabla de failure modes | Sin esqueleto el formato variaba completamente entre corridas; sin la lista de estados el modelo inventaba estados no estándar (ej. "PENDIENTE", "COMPLETADO") |
| 5 | **Nueva sección** | Se agregó sección `Anti-patterns` con 7 restricciones explícitas | Reduce outputs con GWT genéricos, UCs inventados y estados del sistema fuera del dominio FTGO; esta sección no existía en el semilla B.2 |

---

## Métrica

**Indicador:** porcentaje de checkpoints correctamente completados en el output.

**Fórmula:** (checkpoints cumplidos / 18 totales) × 100

**Checkpoints:** Intro + tabla + ≥5 UCs + 3 US semilla + ≥2 UCs adicionales con origen + GWT UC-01 + GWT todos + 7 campos por UC + mapeo PRD + sin UCs inventados = 18 puntos.

| Corrida | Prompt semilla v0.1 | Prompt mejorado v0.2 | Δ |
|---|---|---|---|
| 1 | 7/18 (39%) | 18/18 (100%) | +61% |
| 2 | 6/18 (33%) | 18/18 (100%) | +67% |
| 3 | 11/18 (61%) | 18/18 (100%) | +39% |
| **Promedio** | **44%** | **100%** | **+56%** |
