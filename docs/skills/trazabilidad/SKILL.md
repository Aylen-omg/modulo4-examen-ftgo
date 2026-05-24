---
name: trazabilidad-ftgo
description: >-
  Genera el documento docs/TRAZABILIDAD.md cruzando PRD, FSD, ADRs y diagramas
  C4 del caso FTGO. Usar cuando el usuario invoque @docs/skills/trazabilidad/SKILL.md,
  pida trazabilidad end-to-end, matriz Brief→PRD→FSD→ADR→C4, auditoría de brechas
  arquitectónicas o semáforo de cobertura por capa.
---

# Skill — Trazabilidad FTGO

Genera o actualiza `docs/TRAZABILIDAD.md` cruzando la cadena completa de artefactos del repositorio FTGO.

---

## Cuándo usar esta skill

Invocar cuando:

- El usuario menciona `@docs/skills/trazabilidad/SKILL.md` o pide **trazabilidad**, **matriz de trazabilidad** o **auditoría de brechas**.
- Se modificó algún artefacto (`PRD.md`, `FSD.md`, ADRs o diagramas C4) y hay que verificar coherencia.
- Se necesita evidencia de cobertura Brief → PRD → FSD → ADR → C4 antes de entregar el examen o un PR.

**No usar** para generar artefactos nuevos (PRD, FSD, ADR, C4); usar los prompts de `prompts_mejorados/`.

---

## Inputs requeridos

Leer **todos** estos archivos antes de generar el output. Si falta alguno, abortar con el failure mode correspondiente.

| Archivo | Rol en la trazabilidad |
|---------|------------------------|
| `docs/PRD.md` | Stakeholders, CAP-01…CAP-07, NFR-01…NFR-08, alcance |
| `docs/FSD.md` | UC-01…UC-07, GWT, mapeo CAP ↔ UC |
| `docs/adr/0001-estilo-arquitectonico.md` | Decisión Strangler Fig, NFRs, Richardson Cap. 1/2/13 |
| `docs/adr/0002-ipc-estrategia.md` | Decisión REST + Kafka, NFRs, Richardson Cap. 3/4 |
| `docs/adr/0003-estrategia-datos.md` | Decisión DB-per-service, NFR-05/06, Richardson Cap. 5 |
| `docs/diagrams/c4_context.mmd` | Nivel 1: Person, System, System_Ext, Rel |
| `docs/diagrams/c4_container.mmd` | Nivel 2: contenedores, relaciones tech/protocolo |

**Fuente implícita:** Brief Anexo A (referenciado en PRD/FSD como `[Brief §A.X]`). No inventar elementos que no aparezcan en los inputs.

---

## Proceso paso a paso

### Paso 0 — Validar inputs

1. Confirmar que los **7 archivos** existen y son legibles.
2. Extraer metadatos de versión/fecha de PRD y FSD para el encabezado de `TRAZABILIDAD.md`.

### Paso 1 — Extraer inventario de cada capa

**Desde PRD:**
- 8 stakeholders (§2) con origen `[Brief §A.2]`.
- 7 capacidades CAP-01…CAP-07 (§3) con origen `[Richardson Cap. 2]`.
- 8 NFRs NFR-01…NFR-08 (§4) con origen `[Brief §A.4]`.

**Desde FSD:**
- Tabla resumen UC-01…UC-07 (§2): actor, capacidad PRD, origen.
- Bloque Given/When/Then de cada UC (§3).

**Desde ADRs:**
- ADR 0001: decisión elegida, NFRs citados, capítulos Richardson, UCs implícitos.
- ADR 0002: decisión IPC, NFRs citados, capítulos Richardson, flujos UC referenciados.

**Desde C4:**
- Nivel 1: personas, `System(ftgo)`, `System_Ext`, relaciones con propósito.
- Nivel 2: contenedores dentro de `System_Boundary`, `System_Ext`, cada `Rel(...)` con tecnología/protocolo.

### Paso 2 — Construir las 4 tablas de trazabilidad

#### Tabla 1 — Brief → PRD

Columnas: `| Elemento Brief | Sección Brief | Elemento PRD | Sección PRD | Estado |`

Reglas:
- Stakeholders: mapear cada rol del brief a fila en PRD §2.
- CAPs: mapear las 7 capacidades del Cap. 2 de Richardson a CAP-01…CAP-07 en PRD §3.
- NFRs: mapear categorías `[Brief §A.4 — …]` a NFR-01…NFR-08 en PRD §4.
- Estado: `🟢 Trazado` si hay referencia explícita; `🟡 Parcial` si cubierto pero sin cita; `🔴 Ausente` si no aparece.

#### Tabla 2 — PRD → FSD

Columnas: `| CAP PRD | UC(s) | GWT (resumen) | Origen UC | Estado |`

Reglas:
- Una fila por CAP; si varios UCs cubren la misma CAP, listarlos en la misma celda (ej. CAP-03 → UC-01, UC-06).
- GWT: una línea `Given / When / Then` del UC principal (o del primero si hay varios).
- CAP sin UC dedicado → fila con `—` en UC y estado `🔴 Brecha`.
- CAP-07 (Notifications) y CAP-01 (Consumer Management) suelen aparecer como dependencias en precondiciones/flujos, no como UC propio → marcar `🟡 Parcial` si aplica.

#### Tabla 3 — PRD + FSD → ADRs

Columnas: `| ADR | Decisión | NFRs fundamentan | UCs relacionados | Richardson | Estado |`

Reglas:
- ADR 0001: vincular a NFR-01, NFR-02, NFR-03, NFR-07; UCs del flujo completo; Cap. 1, 2, 13.
- ADR 0002: vincular a NFR-01, NFR-04, NFR-05, NFR-07; UC-01, UC-04, UC-05; Cap. 3, 4.
- Estado `🟢` si la decisión cita explícitamente NFR + capítulo; `🟡` si la relación es inferida; `🔴` si no hay vínculo.

#### Tabla 4 — ADRs → C4

Columnas: `| Decisión ADR | Elemento C4 | Diagrama | Evidencia (Rel/Container) | Estado |`

Reglas de mapeo esperado:

| Decisión | Evidencia en C4 |
|----------|-----------------|
| ADR 0001 — Strangler Fig | `Monolito Legacy` en contexto y contenedor; `Rel(gateway, monolito, …)` |
| ADR 0001 — API Gateway | `Container(gateway, "API Gateway", …)` |
| ADR 0002 — Kafka async | `ContainerQueue(kafka, …)`; relaciones `Kafka protocol async` |
| ADR 0002 — REST lecturas | `Rel(..., "JSON/HTTPS REST")` en gateway → servicios |
| CAP-03 Order Taking | `Container(order, "Order Service", …)` |
| CAP-04 Kitchen | `Container(kitchen, …)` |
| CAP-05 Delivery | `Container(delivery, …)` |
| CAP-06 Billing | `Container(billing, …)` + `Rel(billing, stripe, …)` |
| CAP-07 Notifications | `Container(notifSvc, …)` + `Rel(notifSvc, notif, …)` |
| Externos brief | `System_Ext(stripe)`, `System_Ext(maps)`, `System_Ext(notif)` |

### Paso 3 — Sección Brechas

Listar explícitamente:

1. **CAPs sin UC dedicado** (ej. CAP-01, CAP-07 si no tienen UC propio).
2. **NFRs sin reflejo en ADR o C4** (ej. NFR-06 Trazabilidad si no hay contenedor de observabilidad).
3. **Decisiones ADR sin elemento C4** (ej. ADR 0003 pendiente).
4. **Elementos C4 sin origen en PRD/ADR** (contenedores o relaciones huérfanas).
5. **Stakeholders del PRD sin Person en C4** (ej. Equipo de arquitectura — actor interno, puede quedar fuera del C4).

Formato por brecha:

```markdown
| ID | Capa | Elemento | Descripción | Severidad |
|----|------|----------|-------------|-----------|
| B-01 | PRD→FSD | CAP-01 | Sin UC dedicado; solo precondiciones en UC-01 | 🟡 Media |
```

### Paso 4 — Resumen semáforo por capa

Calcular porcentaje de filas con estado 🟢 en cada tabla y asignar:

| Cobertura | Semáforo |
|-----------|----------|
| ≥ 90 % filas 🟢 | 🟢 Verde |
| 70–89 % o brechas 🟡 sin 🔴 críticas | 🟡 Amarillo |
| < 70 % o brecha 🔴 en elemento crítico (CAP-03, NFR-02, Strangler Fig) | 🔴 Rojo |

Capas a reportar:

```markdown
| Capa | Cobertura | Semáforo | Brechas críticas |
|------|-----------|----------|------------------|
| Brief → PRD | XX % | 🟢/🟡/🔴 | N |
| PRD → FSD | XX % | 🟢/🟡/🔴 | N |
| PRD+FSD → ADR | XX % | 🟢/🟡/🔴 | N |
| ADR → C4 | XX % | 🟢/🟡/🔴 | N |
```

### Paso 5 — Escribir `docs/TRAZABILIDAD.md`

Usar la plantilla de la sección **Output** abajo. Sobrescribir el archivo completo en cada ejecución.

### Paso 6 — Verificación final

Antes de terminar, confirmar:

- [ ] Las 4 tablas tienen al menos una fila por elemento obligatorio (8 stakeholders, 7 CAPs, 8 NFRs, 7 UCs, 2 ADRs).
- [ ] La sección Brechas no está vacía (siempre hay al menos CAP-01/CAP-07 parciales o NFR-06 sin C4).
- [ ] Ningún elemento inventado fuera de los 6 inputs.
- [ ] Fecha de generación actualizada.

---

## Output — plantilla `docs/TRAZABILIDAD.md`

```markdown
# Matriz de Trazabilidad — FTGO

| Campo | Valor |
|-------|-------|
| ID | TR-FTGO-001 |
| Fecha | YYYY-MM-DD |
| PRD | docs/PRD.md (versión del metadato) |
| FSD | docs/FSD.md (versión del metadato) |
| ADRs | 0001-estilo-arquitectonico, 0002-ipc-estrategia |
| C4 | c4_context.mmd, c4_container.mmd |

---

## 1. Brief → PRD

[Tabla stakeholders + CAPs + NFRs]

---

## 2. PRD → FSD

[Tabla CAP → UC → GWT]

---

## 3. PRD + FSD → ADRs

[Tabla ADR → NFR → UC → Richardson]

---

## 4. ADRs → C4

[Tabla decisión → elemento diagrama → evidencia]

---

## 5. Brechas

[Tabla de brechas con ID, capa, elemento, descripción, severidad]

---

## 6. Resumen semáforo

[Tabla por capa con cobertura % y semáforo]

---

*Generado por skill `docs/skills/trazabilidad/SKILL.md`. Solo elementos trazables a los artefactos fuente; ningún dominio inventado [Brief §A.6].*
```

---

## Anti-patterns

- ❌ **Inventar elementos** no presentes en PRD/FSD/ADR/C4 (ej. microservicio "Consumer Service" si no está en el diagrama).
- ❌ **Marcar 🟢 Trazado** sin cita de sección concreta (`PRD §3`, `UC-04`, `Rel(gateway, monolito)`).
- ❌ **Omitir brechas conocidas** — CAP-01 y CAP-07 sin UC propio y NFR-06 sin contenedor de observabilidad son brechas esperadas, no errores a ocultar.
- ❌ **Confundir niveles C4** — stakeholders internos (Equipo de arquitectura) no son `Person` en C4 si no aparecen en el diagrama.
- ❌ **Generar TRAZABILIDAD.md sin leer los 6 inputs** — nunca usar memoria del modelo en lugar de los archivos actuales.
- ❌ **Duplicar UCs como CAPs** — un UC mapea a una CAP; CAP-07 no se convierte en UC-08 inventado.
- ❌ **Ignorar ADR 0002 en C4** — si Kafka no aparece en `c4_container.mmd`, es brecha 🔴, no asumir que existe.

---

## Failure modes

| Código | Condición | Acción |
|--------|-----------|--------|
| `E_MISSING_PRD` | No existe `docs/PRD.md` | Abortar; informar al usuario |
| `E_MISSING_FSD` | No existe `docs/FSD.md` | Abortar; informar al usuario |
| `E_MISSING_ADR` | Falta ADR 0001 o 0002 | Abortar; listar ADR faltante |
| `E_MISSING_C4` | Falta `c4_context.mmd` o `c4_container.mmd` | Abortar; listar archivo faltante |
| `E_EMPTY_INVENTORY` | PRD sin CAPs o FSD sin UCs | Abortar; artefacto fuente corrupto/incompleto |
| `E_INVENTED_TRACE` | Output incluye filas no derivables de inputs | Rechazar y regenerar solo con evidencia |
| `E_NO_GAPS_SECTION` | Brechas vacías cuando hay CAPs parciales | Reintentar Paso 3 |
| `E_STALE_OUTPUT` | TRAZABILIDAD.md no refleja versión actual del PRD | Releer inputs y sobrescribir |

---

## Invocación en Cursor

```
@docs/skills/trazabilidad/SKILL.md
@docs/PRD.md
@docs/FSD.md
@docs/adr/0001-estilo-arquitectonico.md
@docs/adr/0002-ipc-estrategia.md
@docs/adr/0003-estrategia-datos.md
@docs/diagrams/c4_context.mmd
@docs/diagrams/c4_container.mmd

Genera docs/TRAZABILIDAD.md
```

El agente debe: leer esta skill → leer los 6 artefactos → ejecutar Pasos 0–6 → escribir `docs/TRAZABILIDAD.md`.
