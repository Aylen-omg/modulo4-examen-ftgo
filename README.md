# modulo4-examen-ftgo

Examen práctico — Módulo 4: Arquitectura del Producto y Especificaciones Funcionales.
Documentación arquitectónica de la migración FTGO (monolito → microservicios): PRD, FSD, ADRs y diagramas C4.

---

## Estructura del repositorio

```
modulo4-examen-ftgo/
├── docs/
│   ├── PRD.md                          # Product Requirements Document ligero
│   ├── FSD.md                          # Functional Specification Document (5 UCs con GWT)
│   ├── adr/
│   │   ├── 0001-estilo-arquitectonico.md   # ADR 1: estilo arquitectónico
│   │   └── 0002-ipc-estrategia.md          # ADR 2: estrategia IPC / datos
│   └── diagrams/
│       ├── c4_context.mmd              # Diagrama C4 nivel 1 (Context)
│       └── c4_container.mmd            # Diagrama C4 nivel 2 (Container)
├── prompts_mejorados/
│   ├── prd_mejorado.md                 # Prompt mejorado para generar el PRD
│   ├── fsd_mejorado.md                 # Prompt mejorado para generar el FSD
│   ├── adr_mejorado.md                 # Prompt mejorado para generar ADRs
│   └── c4_mejorado.md                  # Prompt mejorado para generar diagramas C4
└── README.md
```

---

## Artefactos entregados

| # | Artefacto | Ruta | Estado |
|---|-----------|------|--------|
| 1 | PRD ligero | `docs/PRD.md` | ✅ |
| 2 | FSD ligero (7 UCs con GWT) | `docs/FSD.md` | ✅ |
| 3 | ADR 1 — Estilo arquitectónico | `docs/adr/0001-estilo-arquitectonico.md` | ⏳ |
| 4 | ADR 2 — Estrategia IPC | `docs/adr/0002-ipc-estrategia.md` | ⏳ |
| 5 | Diagrama C4 nivel 1 (Context) | `docs/diagrams/c4_context.mmd` | ⏳ |
| 6 | Diagrama C4 nivel 2 (Container) | `docs/diagrams/c4_container.mmd` | ⏳ |
| 7 | Prompt mejorado PRD | `prompts_mejorados/prd_mejorado.md` | ✅ |
| 8 | Prompt mejorado FSD | `prompts_mejorados/fsd_mejorado.md` | ✅ |
| 9 | Prompt mejorado ADR | `prompts_mejorados/adr_mejorado.md` | ⏳ |
| 10 | Prompt mejorado C4 | `prompts_mejorados/c4_mejorado.md` | ⏳ |

---

## Comandos para invocar los prompts mejorados

Usa los siguientes comandos dentro de Cursor para invocar cada prompt mejorado contra el caso FTGO:

### PRD

```
@prompts_mejorados/prd_mejorado.md genera PRD para FTGO
```

Genera `docs/PRD.md` con las 5 secciones obligatorias, ≥ 5 NFRs con métrica numérica y trazabilidad al brief.

### FSD

```
@prompts_mejorados/fsd_mejorado.md genera FSD para FTGO usando docs/PRD.md
```

Genera `docs/FSD.md` con exactamente 5 UCs formalizados con bloques Given/When/Then.

### ADR

```
@prompts_mejorados/adr_mejorado.md genera ADR para FTGO sobre [decisión] usando docs/PRD.md y docs/FSD.md
```

Reemplaza `[decisión]` por: `"estilo arquitectónico"`, `"estrategia IPC"`, etc.

### Diagramas C4

```
@prompts_mejorados/c4_mejorado.md genera diagramas C4 nivel 1 y 2 para FTGO usando docs/PRD.md y docs/adr/
```

Genera `docs/diagrams/c4_context.mmd` y `docs/diagrams/c4_container.mmd`.

---

## Métricas declaradas

| Prompt | Indicador | Semilla v0.1 | Mejorado v0.2 | Δ |
|--------|-----------|-------------|---------------|---|
| PRD | % checkpoints completados (20 total) | 61.7 % | 96.7 % | +35 % |
| FSD | pendiente | — | — | — |
| ADR | pendiente | — | — | — |
| C4 | pendiente | — | — | — |

---

## Fuentes de referencia

- [Repositorio oficial FTGO](https://github.com/microservices-patterns/ftgo-application)
- [Microservices Pattern Language](https://microservices.io/)
- Richardson, C. (2019). *Microservices Patterns*. Manning.
