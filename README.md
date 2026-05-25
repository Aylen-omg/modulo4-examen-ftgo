# modulo4-examen-ftgo

> Examen Práctico — Módulo 4: Arquitectura del Producto y Especificaciones Funcionales
> Caso FTGO — *Microservices Patterns* (Richardson, Manning 2019)

---

## Descripción

Este repositorio documenta la **arquitectura objetivo** de FTGO (Food To Go), una plataforma de delivery de comida que migra de un monolito Java (WAR) hacia una arquitectura de microservicios. La migración no parte de cero: sigue el patrón **Strangler Fig** durante 18-24 meses, extrayendo capacidades de negocio de forma incremental mientras el monolito sigue operando en producción. El repositorio contiene la cadena completa de artefactos arquitectónicos — PRD → FSD → ADRs → diagramas C4 — con trazabilidad explícita al brief del caso y al libro de Richardson, además de los 4 prompts mejorados utilizados para generarlos.

---

## Estructura del repositorio

```
modulo4-examen-ftgo/
├── README.md                               ← este archivo
├── docs/
│   ├── PRD.md                              ← Product Requirements Document ligero
│   ├── FSD.md                              ← Functional Specification Document (7 UCs + matrices CAP)
│   ├── TRAZABILIDAD.md                     ← Matriz Brief→PRD→FSD→ADR→C4 (skill trazabilidad)
│   ├── adr/
│   │   ├── 0001-estilo-arquitectonico.md   ← ADR: estilo arquitectónico (Strangler Fig elegido)
│   │   ├── 0002-ipc-estrategia.md          ← ADR: estrategia IPC (híbrido REST + Kafka elegido)
│   │   └── 0003-estrategia-datos.md        ← ADR: estrategia de datos (DB-per-service)
│   ├── diagrams/
│   │   ├── c4_context.mmd                  ← Diagrama C4 Nivel 1 — Context
│   │   └── c4_container.mmd                ← Diagrama C4 Nivel 2 — Container
│   └── skills/
│       └── trazabilidad/
│           └── SKILL.md                    ← Skill Cursor: genera TRAZABILIDAD.md
├── prompts_mejorados/
│   ├── prd_mejorado.md                     ← Prompt mejorado B.1 — genera PRD
│   ├── fsd_mejorado.md                     ← Prompt mejorado B.2 — genera FSD
│   ├── adr_mejorado.md                     ← Prompt mejorado B.3 — genera ADRs
│   └── c4_mejorado.md                      ← Prompt mejorado B.4 — genera diagramas C4
└── prompts_sesion/
    ├── 00_prompt_inicial.md                ← Prompt de apertura de sesión (rol + caso + plan)
    └── 01_contract.md                      ← Contract de sesión (9 reglas absolutas)
```

---

## Cadena de trazabilidad

Cada artefacto alimenta al siguiente. Ningún elemento puede inventarse fuera del Brief o de Richardson [Brief §A.6].

```
Brief del caso FTGO (Anexo A)
    │  stakeholders, capacidades, NFRs, US semilla
    ↓
PRD (docs/PRD.md)
    │  qué necesita el negocio: 8 stakeholders, 7 CAPs, 8 NFRs
    ↓
FSD (docs/FSD.md)
    │  cómo funciona paso a paso: 7 UCs con Given/When/Then
    │  derivados de las US semilla + capacidades del PRD
    ↓
ADRs (docs/adr/)
    │  decisiones arquitectónicas fundamentadas en NFRs del PRD
    │  ADR 0001: estilo → Strangler Fig
    │  ADR 0002: IPC → REST síncrono + Kafka async (híbrido)
    │  ADR 0003: datos → Database-per-service
    ↓
C4 (docs/diagrams/)
    │  arquitectura visual coherente con los ADRs
    │  Nivel 1: FTGO como caja negra + actores + externos
    │  Nivel 2: 12 contenedores + 24 relaciones con protocolo
    ↓
TRAZABILIDAD (docs/TRAZABILIDAD.md)
    │  auditoría end-to-end generada por docs/skills/trazabilidad/SKILL.md
```

---

## Cómo invocar los prompts mejorados

Usa los siguientes comandos dentro de Cursor apuntando al archivo de prompt mejorado correspondiente.

### PRD

```
@prompts_mejorados/prd_mejorado.md
```

Genera `docs/PRD.md` con 8 stakeholders, 7 capacidades de negocio (CAP-01 a CAP-07) y 8 NFRs con métrica numérica trazables a [Brief §A.4]. Trazabilidad explícita y Strangler Fig presente en contexto y alcance.

### FSD

```
@prompts_mejorados/fsd_mejorado.md
```

Genera `docs/FSD.md` con exactamente 7 UCs formalizados en formato Given/When/Then. UC-01 a UC-03 derivan de las US semilla; UC-04 a UC-07 derivan de capacidades del PRD y de [Richardson Cap. 3]. Todos los UCs tienen los 7 campos completos y estados concretos del dominio FTGO (`PENDING_PAYMENT`, `APPROVED`, `PREPARING`, `ASSIGNED`, `DELIVERED`, `CANCELLED`).

### ADR

```
@prompts_mejorados/adr_mejorado.md
```

Genera un ADR para FTGO. Pasar como parámetro la decisión a documentar:

- `"estilo arquitectónico"` → genera `docs/adr/0001-estilo-arquitectonico.md`
- `"estrategia IPC"` → genera `docs/adr/0002-ipc-estrategia.md`
- `"estrategia de datos"` → genera `docs/adr/0003-estrategia-datos.md`

Cada ADR evalúa exactamente 3 opciones con 5 dimensiones (descripción, pros, contras, tabla NFR, compatibilidad Strangler Fig) y documenta consecuencias positivas Y negativas.

### C4

```
@prompts_mejorados/c4_mejorado.md
```

Genera `docs/diagrams/c4_context.mmd` (Nivel 1) y `docs/diagrams/c4_container.mmd` (Nivel 2) en sintaxis Mermaid C4 válida. El Nivel 2 incluye 12 contenedores (incl. Observability Stack [NFR-06]) con relaciones tech + protocolo, coherente con ADR 0001 (Monolito Legacy), ADR 0002 (Kafka + REST) y ADR 0003 (DB-per-service).

### Trazabilidad

```
@docs/skills/trazabilidad/SKILL.md
@docs/PRD.md @docs/FSD.md
@docs/adr/0001-estilo-arquitectonico.md @docs/adr/0002-ipc-estrategia.md
@docs/diagrams/c4_context.mmd @docs/diagrams/c4_container.mmd
```

Genera o actualiza `docs/TRAZABILIDAD.md` con matrices Brief→PRD→FSD→ADR→C4, brechas y semáforo por capa.

---

## Métricas de calidad de prompts mejorados

Indicadores medidos comparando el prompt semilla v0.1 (Anexo B) contra el prompt mejorado v0.2 en 3 corridas con temperatura 0.2.

| Prompt | Indicador | Semilla v0.1 | Mejorado v0.2 | Δ |
|--------|-----------|:------------:|:-------------:|:-:|
| PRD | % checkpoints completos (21 pts) | 54 % | 100 % | **+46 %** |
| FSD | % checkpoints completos (18 pts) | 44 % | 100 % | **+56 %** |
| ADR | % checkpoints completos (16 pts) | 46 % | 100 % | **+54 %** |
| C4  | % checkpoints completos (15 pts) | 27 % | 100 % | **+73 %** |

---

## Artefactos entregados

| # | Artefacto | Ruta | Estado |
|---|-----------|------|:------:|
| 1 | PRD ligero (8 NFRs, 7 CAPs, 8 stakeholders) | `docs/PRD.md` | ✅ |
| 2 | FSD ligero (7 UCs con Given/When/Then) | `docs/FSD.md` | ✅ |
| 3 | ADR 1 — Estilo arquitectónico (Strangler Fig) | `docs/adr/0001-estilo-arquitectonico.md` | ✅ |
| 4 | ADR 2 — Estrategia IPC (REST + Kafka híbrido) | `docs/adr/0002-ipc-estrategia.md` | ✅ |
| 5 | ADR 3 — Estrategia de datos (DB-per-service) | `docs/adr/0003-estrategia-datos.md` | ✅ |
| 6 | Diagrama C4 Nivel 1 — Context | `docs/diagrams/c4_context.mmd` | ✅ |
| 7 | Diagrama C4 Nivel 2 — Container | `docs/diagrams/c4_container.mmd` | ✅ |
| 8 | Matriz de trazabilidad | `docs/TRAZABILIDAD.md` | ✅ |
| 9 | Skill trazabilidad Cursor | `docs/skills/trazabilidad/SKILL.md` | ✅ |
| 10 | Prompt mejorado PRD (semilla B.1) | `prompts_mejorados/prd_mejorado.md` | ✅ |
| 11 | Prompt mejorado FSD (semilla B.2) | `prompts_mejorados/fsd_mejorado.md` | ✅ |
| 12 | Prompt mejorado ADR (semilla B.3) | `prompts_mejorados/adr_mejorado.md` | ✅ |
| 13 | Prompt mejorado C4 (semilla B.4) | `prompts_mejorados/c4_mejorado.md` | ✅ |
| 14 | README ejecutable | `README.md` | ✅ |
| 15 | Prompt inicial de sesión | `prompts_sesion/00_prompt_inicial.md` | ✅ |
| 16 | Contract de sesión (9 reglas) | `prompts_sesion/01_contract.md` | ✅ |

---

## Fuentes

- Richardson, C. (2019). *Microservices Patterns*. Manning.
- Brief del caso FTGO — Anexo A del examen (contexto, stakeholders, capacidades, NFRs, US semilla).
- [Repositorio oficial FTGO](https://github.com/microservices-patterns/ftgo-application)
- [Microservices Pattern Language](https://microservices.io/)
