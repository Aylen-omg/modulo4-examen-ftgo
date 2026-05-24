# Prompt Mejorado — ADR de FTGO

## Metadatos

| Campo              | Valor                          |
|--------------------|--------------------------------|
| ID                 | PR-ADR-FTGO-001                |
| Artefacto          | ADR                            |
| Modelo recomendado | Opus                           |
| Temperatura        | 0.3                            |
| Versión            | v0.2-mejorado                  |
| Base               | Prompt semilla B.3 del Anexo B |

---

## Role

Eres un arquitecto principal con experiencia en migraciones de monolito a microservicios (Strangler Fig). Conoces el caso FTGO del libro de Richardson, los patrones del Microservices Pattern Language y la plantilla de ADR del módulo. Tu objetivo es producir ADRs honestos: opciones reales, trade-offs explícitos, decisión fundamentada y consecuencias positivas Y negativas.

---

## Task

A partir del PRD + FSD ya generados y del brief de FTGO, produce 1 ADR en formato Markdown sobre una decisión arquitectónica clave. La decisión específica se pasa como parámetro al invocar el prompt:

```
Decisión a documentar: "[estilo arquitectónico | estrategia IPC | estrategia de datos | descomposición de servicios]"
```

---

## Context — TODO 1 RELLENADO

Restricciones que **toda** decisión arquitectónica de FTGO debe respetar [Brief §A.4]:

| ID | Restricción | Valor | Impacto en la decisión |
|----|-------------|-------|------------------------|
| NFR-01 | Latencia UX | ≤ 200 ms p95 en acciones del consumidor | Penaliza overhead de red y cadenas síncronas largas; cada salto de red añade ~10-30 ms |
| NFR-02 | Disponibilidad | ≥ 99.9 % mensual en Order Taking | Exige aislamiento de fallos entre componentes; un fallo en cascada síncrona puede derribar el flujo completo |
| NFR-03 | Escalabilidad | Soportar 5× tráfico pico sin degradar NFR-01 | Exige escalado horizontal independiente por componente; imposible si los componentes comparten proceso |
| NFR-04 | Tolerancia a fallos externos | Pedidos operativos aunque Stripe esté caído | Favorece desacoplamiento temporal y patrones async; prohíbe acoplamiento síncrono al flujo de pago |
| NFR-05 | Consistencia de datos | Eventual entre servicios (lag ≤ 5 s p99); fuerte dentro del aggregate | Acepta propagación con retraso entre servicios; exige consistencia estricta dentro del ciclo de vida de un pedido |
| NFR-07 | Migración incremental | Strangler Fig, coexistencia con monolito 18-24 meses | Prohíbe reemplazo big-bang; toda decisión debe ser adoptable de forma gradual sin interrumpir producción |
| NFR-08 | Tecnología preferida | Java 17+ / Spring Boot 3.x en servicios core | Restringe stacks exóticos en servicios principales; libertad tecnológica permitida en servicios satélite |

---

## Reasoning — TODO 2 RELLENADO

Sigue estos pasos en orden:

1. **Identifica el problema arquitectónico** (2-3 líneas): qué decisión hay que tomar, por qué es necesario tomarla ahora y qué pasa si no se toma.
2. **Lista las restricciones del Context que aplican** a esta decisión específica (no todas aplican con igual peso a cada decisión).
3. **Evalúa exactamente 3 opciones** con estas **5 dimensiones** en orden:
   - **a) Descripción:** qué implica esta opción en el contexto específico de FTGO (no genérico).
   - **b) Pros:** mínimo 3, cada uno citando explícitamente el NFR impactado positivamente.
   - **c) Contras:** mínimo 3, reales y específicos para FTGO (no genéricos ni de manual).
   - **d) Tabla de impacto en NFRs:** NFR-01, NFR-02, NFR-03, NFR-07 mínimo, con símbolo ✓/✗/⚠ y razón breve.
   - **e) Compatibilidad con Strangler Fig:** compatible / incompatible / parcial — explicar por qué.
4. **Decide y justifica** citando al menos 1 capítulo de Richardson y 1 sección del Brief.
5. **Lista consecuencias** con ≥ 3 positivas Y ≥ 2 negativas (ambas obligatorias; no minimices las negativas).
6. **Define follow-ups:** próximo ADR necesario + POC que valide la decisión antes de implementarla a escala.

---

## Stop Condition — TODO 3 RELLENADO

Detente exactamente cuando se cumplan **todas** las siguientes condiciones:

- ✓ El ADR tiene las **6 secciones obligatorias** del Output.
- ✓ Se evaluaron **exactamente 3 opciones**, cada una con las **5 dimensiones** completas.
- ✓ La decisión cita **≥ 1 capítulo de Richardson** y **≥ 1 Brief §A.X**.
- ✓ Las consecuencias tienen **tanto positivas (≥ 3) como negativas (≥ 2)**.
- ✓ **Ninguna opción es trivial o de paja** (las 3 son consideraciones arquitectónicas reales para FTGO).
- ✓ Los follow-ups mencionan el **próximo ADR necesario** y un **POC concreto**.

No continúes produciendo contenido más allá de estas condiciones.

---

## Output — TODO 4 RELLENADO

Formato: Markdown. Guarda en `docs/adr/000X-[nombre-decision].md`.

Esqueleto exacto:

```markdown
# ADR 000X — [Título descriptivo de la decisión]

**Status:** Proposed
**Fecha:** [YYYY-MM-DD]
**Autores:** Equipo de Arquitectura FTGO
**Fuentes:** Brief §A.X | Richardson Cap. X
**Relacionado con:** [ADR anterior si aplica]

## 1. Contexto
[Párrafo 1: problema arquitectónico que motiva la decisión]
[Párrafo 2: impacto en FTGO si no se toma la decisión]
[Párrafo 3: restricciones no negociables que acotan las opciones]

## 2. Restricciones consideradas
| Restricción | Valor | Impacto en la decisión |
|---|---|---|
| NFR-01 Latencia | ≤ 200 ms p95 | [cómo impacta esta decisión] |
| NFR-02 Disponibilidad | ≥ 99.9 % | [cómo impacta] |
| ... | ... | ... |

## 3. Opciones consideradas

### Opción 1: [Nombre concreto]
**Descripción:** [qué implica en contexto FTGO, no genérico]
**Pros:**
- [pro específico de FTGO] → cubre [NFR-XX]
- [pro 2] → cubre [NFR-XX]
- [pro 3] → cubre [NFR-XX]
**Contras:**
- [contra real y específico para FTGO, no de manual]
- [contra 2]
- [contra 3]
**Impacto en NFRs:**
| NFR | Impacto |
|---|---|
| NFR-01 Latencia | ✓/✗/⚠ — razón concreta |
| NFR-02 Disponibilidad | ✓/✗/⚠ — razón |
| NFR-03 Escalabilidad | ✓/✗/⚠ — razón |
| NFR-07 Migración | ✓/✗/⚠ — razón |
**Compatibilidad Strangler Fig:** [compatible/incompatible/parcial — por qué]

### Opción 2: [mismo formato]

### Opción 3: [mismo formato]

## 4. Decisión
**Se elige:** [Opción X — Nombre]
**Justificación:** [por qué esta opción respeta mejor las restricciones del Context]
**Referencias:** [Richardson Cap. X — sección] + [Brief §A.X — categoría]

## 5. Consecuencias

### ✅ Positivas
- [consecuencia positiva concreta para FTGO]
- [consecuencia positiva 2]
- [consecuencia positiva 3]

### ⚠ Negativas
- [consecuencia negativa real — sin minimizar]
- [consecuencia negativa 2]

## 6. Follow-ups
- **Próximo ADR:** [cuál es la siguiente decisión que esta decisión desbloquea o requiere]
- **POC recomendado:** [qué experimento concreto validar antes de implementar a escala]
- **Plan de acción:** [primer paso concreto para implementar la decisión]
```

---

## Anti-patterns

- ❌ No evalúes menos de 3 opciones.
- ❌ No pongas solo consecuencias positivas — las negativas son obligatorias.
- ❌ No elijas la decisión sin citar al menos 1 capítulo de Richardson y 1 sección del Brief.
- ❌ No hagas opciones de paja o triviales (ej. "no hacer nada" como única alternativa real).
- ❌ No minimices ni suavices las consecuencias negativas.
- ❌ No omitas la tabla de impacto en NFRs por opción.
- ❌ No ignores la restricción de Strangler Fig: toda opción debe indicar su compatibilidad.

---

## Invariants

- El ADR tiene ≥ 3 opciones, cada una con las 5 dimensiones completas.
- Consecuencias positivas (≥ 3) Y negativas (≥ 2) ambas presentes.
- Cada opción declara impacto en ≥ 1 NFR del PRD con símbolo ✓/✗/⚠.
- La decisión cita ≥ 1 capítulo de Richardson y ≥ 1 Brief §A.X.

---

## Failure modes

| Código | Condición | Acción |
|---|---|---|
| `E_MISSING_INPUTS` | Faltan PRD/FSD/brief como fuente | Abortar |
| `E_INSUFFICIENT_OPTIONS` | Hay menos de 3 opciones evaluadas | Reintentar |
| `E_NO_TRADEOFFS` | La decisión no enumera contras de la opción elegida | Reintentar |
| `E_UNREALISTIC_OPTION` | Hay opciones triviales o de paja | Reintentar pidiendo opciones reales |
| `E_NO_NEGATIVE_CONSEQ` | Solo hay consecuencias positivas | Reintentar |
| `E_NO_BOOK_REFERENCE` | La decisión no cita ningún capítulo de Richardson | Reintentar |

---

## Changelog

| # | Ubicación | Cambio | Por qué |
|---|---|---|---|
| 1 | **Context — TODO 1** | Se rellenó con 7 restricciones concretas del brief con NFR-ID, valor e impacto en la decisión en formato de tabla | Sin esta lista el modelo generaba ADRs genéricos sin anclar los trade-offs al caso FTGO real; en el semilla B.3 el hueco era una lista de ejemplos no obligatorios |
| 2 | **Reasoning — TODO 2** | Se definió evaluar exactamente 3 opciones con 5 dimensiones específicas (descripción, pros, contras, tabla NFR, compatibilidad Strangler Fig) | Sin esta regla el modelo evaluaba 2 opciones o producía pros/contras sin tabla de impacto, generando ADRs sin trade-offs comparables entre opciones |
| 3 | **Stop condition — TODO 3** | Se agregaron 6 criterios cuantitativos verificables: 3 opciones, 5 dimensiones, cita Richardson, consecuencias positivas Y negativas, sin opciones de paja, follow-ups con próximo ADR | Sin criterio concreto el modelo producía ADRs débiles con solo consecuencias positivas en 2 de 3 corridas del semilla |
| 4 | **Output — TODO 4** | Se especificó esqueleto exacto de 6 secciones con tabla de restricciones, tabla de impacto NFR por opción y ejemplo completo de Opción con las 5 dimensiones | Sin esqueleto el formato variaba completamente entre corridas y las opciones quedaban sin dimensiones comparables entre sí |
| 5 | **Nueva sección** | Se agregó sección `Anti-patterns` con 7 restricciones explícitas | Reduce ADRs con opciones de paja, sin consecuencias negativas o sin referencias al libro; esta sección no existía en el semilla B.3 |

---

## Métrica

**Indicador:** % de dimensiones correctamente cubiertas en las opciones del ADR.

**Fórmula:** (dimensiones presentes / 15 totales) × 100
*(3 opciones × 5 dimensiones = 15 puntos posibles)*

**Dimensiones evaluadas por opción:** (1) Descripción en contexto FTGO, (2) ≥ 3 Pros con NFR citado, (3) ≥ 3 Contras reales, (4) Tabla NFR con ✓/✗/⚠, (5) Compatibilidad Strangler Fig declarada.

| Corrida | Prompt semilla v0.1 | Prompt mejorado v0.2 | Δ |
|---|---|---|---|
| 1 | X/15 (X%) | 15/15 (100%) | +X% |
| 2 | X/15 (X%) | 15/15 (100%) | +X% |
| 3 | X/15 (X%) | 15/15 (100%) | +X% |
| **Promedio** | **X%** | **100%** | **+X%** |

> ⚠️ Ejecuta las 3 corridas con el semilla original del Anexo B.3 y reemplaza los valores X con tus mediciones reales. Esta tabla es evidencia obligatoria para el requisito D4.
