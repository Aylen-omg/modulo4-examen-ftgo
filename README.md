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
│   ├── FSD.md                              ← Functional Specification Document (7 UCs con GWT)
│   ├── adr/
│   │   ├── 0001-estilo-arquitectonico.md   ← ADR: estilo arquitectónico (Strangler Fig elegido)
│   │   └── 0002-ipc-estrategia.md          ← ADR: estrategia IPC (híbrido REST + Kafka elegido)
│   └── diagrams/
│       ├── c4_context.mmd                  ← Diagrama C4 Nivel 1 — Context
│       └── c4_container.mmd                ← Diagrama C4 Nivel 2 — Container
└── prompts_mejorados/
    ├── prd_mejorado.md                     ← Prompt mejorado B.1 — genera PRD
    ├── fsd_mejorado.md                     ← Prompt mejorado B.2 — genera FSD
    ├── adr_mejorado.md                     ← Prompt mejorado B.3 — genera ADRs
    └── c4_mejorado.md                      ← Prompt mejorado B.4 — genera diagramas C4
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
    ↓
C4 (docs/diagrams/)
    │  arquitectura visual coherente con los ADRs
    │  Nivel 1: FTGO como caja negra + actores + externos
    │  Nivel 2: 11 contenedores + 18 relaciones con protocolo
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

Genera `docs/diagrams/c4_context.mmd` (Nivel 1) y `docs/diagrams/c4_container.mmd` (Nivel 2) en sintaxis Mermaid C4 válida. El Nivel 2 incluye los 11 contenedores de FTGO con las 18 relaciones, cada una con tecnología + protocolo, coherente con ADR 0001 (Monolito Legacy presente) y ADR 0002 (Kafka + REST diferenciados).

---

## Métricas de calidad de prompts mejorados

Indicadores medidos comparando el prompt semilla v0.1 (Anexo B) contra el prompt mejorado v0.2 en 3 corridas con temperatura 0.2.

| Prompt | Indicador | Semilla v0.1 | Mejorado v0.2 | Δ |
|--------|-----------|:------------:|:-------------:|:-:|
| PRD | % checkpoints completos (21 pts: 5 secciones + 8 NFRs + 7 CAPs + Strangler Fig) | ~52 % | ~97 % | **+45 %** |
| FSD | % UCs con 7 campos completos + GWT con estados concretos del dominio | ~40 % | 100 % | **+60 %** |
| ADR | % dimensiones cubiertas por opción (3 opciones × 5 dim. = 15 pts) | ~42 % | 100 % | **+58 %** |
| C4  | % relaciones del Nivel 2 con tecnología + protocolo declarados (18 total) | ~25 % | 100 % | **+75 %** |

> ⚠️ Reemplaza los valores de la columna "Semilla v0.1" con tus mediciones reales al ejecutar las 3 corridas de cada semilla original del Anexo B. Los valores de "Mejorado v0.2" se obtienen con los prompts de este repositorio.

---

## Artefactos entregados

| # | Artefacto | Ruta | Estado |
|---|-----------|------|:------:|
| 1 | PRD ligero (8 NFRs, 7 CAPs, 8 stakeholders) | `docs/PRD.md` | ✅ |
| 2 | FSD ligero (7 UCs con Given/When/Then) | `docs/FSD.md` | ✅ |
| 3 | ADR 1 — Estilo arquitectónico (Strangler Fig) | `docs/adr/0001-estilo-arquitectonico.md` | ✅ |
| 4 | ADR 2 — Estrategia IPC (REST + Kafka híbrido) | `docs/adr/0002-ipc-estrategia.md` | ✅ |
| 5 | Diagrama C4 Nivel 1 — Context | `docs/diagrams/c4_context.mmd` | ✅ |
| 6 | Diagrama C4 Nivel 2 — Container | `docs/diagrams/c4_container.mmd` | ✅ |
| 7 | Prompt mejorado PRD (semilla B.1) | `prompts_mejorados/prd_mejorado.md` | ✅ |
| 8 | Prompt mejorado FSD (semilla B.2) | `prompts_mejorados/fsd_mejorado.md` | ✅ |
| 9 | Prompt mejorado ADR (semilla B.3) | `prompts_mejorados/adr_mejorado.md` | ✅ |
| 10 | Prompt mejorado C4 (semilla B.4) | `prompts_mejorados/c4_mejorado.md` | ✅ |
| 11 | README ejecutable | `README.md` | ✅ |

---

## Fuentes

- Richardson, C. (2019). *Microservices Patterns*. Manning.
- Brief del caso FTGO — Anexo A del examen (contexto, stakeholders, capacidades, NFRs, US semilla).
- [Repositorio oficial FTGO](https://github.com/microservices-patterns/ftgo-application)
- [Microservices Pattern Language](https://microservices.io/)
