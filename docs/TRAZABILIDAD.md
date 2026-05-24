# Matriz de Trazabilidad — FTGO

| Campo | Valor |
|-------|-------|
| ID | TR-FTGO-001 |
| Versión | 2.1 |
| Fecha | 2026-05-24 |
| PRD | `docs/PRD.md` (v0.2-mejorado) |
| FSD | `docs/FSD.md` (v0.2-mejorado) |
| ADRs | `0001-estilo-arquitectonico`, `0002-ipc-estrategia`, `0003-estrategia-datos` |
| C4 | `c4_context.mmd`, `c4_container.mmd` |

---

## 1. Brief → PRD

### Stakeholders (8 / 8 🟢)

| Elemento Brief | Sección Brief | Elemento PRD | Sección PRD | Estado |
|----------------|---------------|--------------|-------------|--------|
| Consumidor | §A.2 | Consumidor | PRD §2 | 🟢 |
| Restaurante | §A.2 | Restaurante | PRD §2 | 🟢 |
| Courier | §A.2 | Courier | PRD §2 | 🟢 |
| Empleado FTGO (back office) | §A.2 | Empleado FTGO | PRD §2 | 🟢 |
| Equipo de arquitectura | §A.2 | Equipo de arquitectura | PRD §2 | 🟢 |
| Stripe | §A.2 | Stripe | PRD §2 | 🟢 |
| Google Maps | §A.2 | Google Maps | PRD §2 | 🟢 |
| SendGrid / Twilio | §A.2 | SendGrid / Twilio | PRD §2 | 🟢 |

### Capacidades (7 / 7 🟢)

| Elemento Brief | Sección Brief | Elemento PRD | Sección PRD | Estado |
|----------------|---------------|--------------|-------------|--------|
| Consumer Management | Richardson Cap. 2 | CAP-01 | PRD §3 | 🟢 |
| Restaurant Management | Richardson Cap. 2 | CAP-02 | PRD §3 | 🟢 |
| Order Taking | Richardson Cap. 2 | CAP-03 | PRD §3 | 🟢 |
| Order Fulfillment / Kitchen | Richardson Cap. 2 | CAP-04 | PRD §3 | 🟢 |
| Delivery | Richardson Cap. 2 | CAP-05 | PRD §3 | 🟢 |
| Billing & Accounting | Richardson Cap. 2 | CAP-06 | PRD §3 | 🟢 |
| Notifications | Richardson Cap. 2 | CAP-07 | PRD §3 | 🟢 |

### NFRs (8 / 8 🟢)

| Categoría Brief | Sección Brief | NFR PRD | Sección PRD | Estado |
|-----------------|---------------|---------|-------------|--------|
| Latencia UX | §A.4 | NFR-01 ≤ 200 ms p95 | PRD §4 | 🟢 |
| Disponibilidad | §A.4 | NFR-02 ≥ 99.9 % | PRD §4 | 🟢 |
| Escalabilidad | §A.4 | NFR-03 5× pico | PRD §4 | 🟢 |
| Tolerancia fallos externos | §A.4 | NFR-04 Stripe caído | PRD §4 | 🟢 |
| Consistencia | §A.4 | NFR-05 fuerte/eventual | PRD §4 | 🟢 |
| Trazabilidad | §A.4 | NFR-06 correlation ID | PRD §4 | 🟢 |
| Migración incremental | §A.4 | NFR-07 Strangler Fig | PRD §4 | 🟢 |
| Cumplimiento | §A.4 | NFR-08 PCI/GDPR | PRD §4 | 🟢 |

---

## 2. PRD → FSD

| CAP PRD | UC(s) | GWT (resumen) | Origen | Estado |
|---------|-------|---------------|--------|--------|
| CAP-01 Consumer Management | UC-01 (precondiciones) | Sesión autenticada + tarjeta válida antes de confirmar | FSD §2.1; UC-01 §3 | 🟢 |
| CAP-02 Restaurant Management | UC-07, UC-02 | Given restaurante activo / When agrega ítem / Then `AVAILABLE` | [Brief §A.5] | 🟢 |
| CAP-03 Order Taking | UC-01, UC-06 | Given carrito / When confirmar / Then `PENDING_PAYMENT` | US-01 [Brief §A.5] | 🟢 |
| CAP-04 Order Fulfillment | UC-02 | Given `PENDING_ACCEPTANCE` / When aceptar / Then `PREPARING` | US-02 [Brief §A.5] | 🟢 |
| CAP-05 Delivery | UC-03, UC-05 | Given courier ≤ 2 km / When acepta / Then `ASSIGNED` | US-03 [Brief §A.5] | 🟢 |
| CAP-06 Billing | UC-04 | Given `PENDING_PAYMENT` / When `OrderCreated` → Stripe / Then `APPROVED` | CAP-06 [PRD §3] | 🟢 |
| CAP-07 Notifications | Matriz §2.2 (7 eventos) | Given `APPROVED` / When `PaymentConfirmed` / Then push ≤ 5 s | FSD §2.2 | 🟢 |

---

## 3. PRD + FSD → ADRs

| ADR | Decisión | NFRs | UCs / flujos | Richardson | Estado |
|-----|----------|------|--------------|------------|--------|
| 0001 | Strangler Fig progresivo | 01, 02, 03, 04, 07, 08 | UC-01→UC-05; monolito coexistencia | Cap. 1, 2, 13 | 🟢 |
| 0002 | REST + Kafka híbrido | 01, 02, 04, 05, 07 | UC-01, UC-04, UC-05 | Cap. 3, 4 | 🟢 |
| 0003 | Database-per-service | 03, 05, 06, 07 | UC-04 Saga; persistencia Order | Cap. 5, 11 | 🟢 |

---

## 4. ADRs → C4

| Decisión / CAP | Elemento C4 | Diagrama | Evidencia | Estado |
|----------------|---------------|----------|-----------|--------|
| ADR 0001 Strangler Fig | Monolito Legacy | L1 + L2 | `System_Ext(legacy)`; `Container(monolito)`; `Rel(gateway, monolito)` | 🟢 |
| ADR 0001 CAP-01/02 en monolito | Descripción monolito | L2 | "CAP-01 Consumer… CAP-02 Restaurant parcial" | 🟢 |
| ADR 0001 API Gateway | API Gateway | L2 | `Container(gateway)` | 🟢 |
| ADR 0001 Clientes PRD | Mobile App, Web Admin | L2 | `Container(mobile)`, `Container(webadmin)` | 🟢 |
| ADR 0002 Kafka | Event Broker | L2 | `ContainerQueue(kafka)` + 5 relaciones async | 🟢 |
| ADR 0002 REST | Gateway → servicios | L2 | `Rel(..., "JSON/HTTPS REST")` | 🟢 |
| ADR 0003 Order DB | Order DB | L2 | `ContainerDb(orderDb)` + `Rel(order, orderDb, "JDBC/SQL")` | 🟢 |
| ADR 0003 Kitchen/Billing/Delivery DB | — | L2 | ADR §4 asigna DBs; **solo Order DB dibujada** | 🟡 |
| NFR-06 Observabilidad | Observability Stack | L2 | `Container(observability)` + 6 rel. OpenTelemetry | 🟢 |
| CAP-03 | Order Service | L2 | `Container(order)` | 🟢 |
| CAP-04 | Kitchen Service | L2 | `Container(kitchen)` | 🟢 |
| CAP-05 | Delivery Service | L2 | `Container(delivery)` | 🟢 |
| CAP-06 | Billing Service | L2 | `Container(billing)` + Stripe | 🟢 |
| CAP-07 | Notification Svc | L2 | `Container(notifSvc)` + SendGrid/Twilio | 🟢 |
| Externos brief | Stripe, Maps, SendGrid/Twilio | L1 + L2 | 3× `System_Ext` | 🟢 |

**Inventario C4 Nivel 2:** 12 contenedores · 24 relaciones con tecnología/protocolo.

---

## 5. Brechas

| ID | Capa | Elemento | Descripción | Severidad |
|----|------|----------|-------------|-----------|
| B-01 | ADR → C4 | Kitchen/Billing/Delivery DB | ADR 0003 §4 asigna DB dedicada a kitchen, billing y delivery; C4 solo modela `Order DB` | 🟡 Media |
| B-02 | PRD → C4 | Equipo de arquitectura | Stakeholder PRD §2 sin `Person` en C4 (actor de diseño, no runtime) | 🟢 Baja |
| B-03 | C4 | Monolito L1 vs L2 | `System_Ext` en contexto vs `Container` en contenedores — convención Strangler Fig [ADR 0001] | 🟢 Baja |

**Brechas críticas abiertas: 0**

---

## 6. Resumen semáforo

| Capa | 🟢 / Total | Cobertura | Semáforo | Brechas críticas |
|------|------------|-----------|----------|------------------|
| Brief → PRD | 23 / 23 | 100 % | 🟢 | 0 |
| PRD → FSD | 7 / 7 | 100 % | 🟢 | 0 |
| PRD+FSD → ADR | 3 / 3 | 100 % | 🟢 | 0 |
| ADR → C4 | 14 / 15 | 93 % | 🟡 | 0 |
| **Global** | **47 / 48** | **98 %** | **🟢** | **0** |

---

## 7. Lectura global

La cadena **Brief → PRD → FSD → ADR (×3) → C4** está completa y alineada con el Anexo A del examen. Los 7 UCs del FSD cubren las 7 CAPs vía UCs dedicados (§2.1) y matriz transversal CAP-07 (§2.2). Las tres ADRs cierran estilo, IPC y datos con referencias Richardson.

La única brecha media restante (B-01) es **diagramática**: el C4 muestra el patrón DB-per-service con `Order DB` como ejemplo; las demás DBs de ADR 0003 pueden añadirse al diagrama en una iteración sin cambiar la decisión arquitectónica.

**Flujo feliz trazado:** US-01 → UC-01 → Kafka `OrderCreated` → UC-04 → UC-02 → UC-03 → UC-05, con notificaciones CAP-07 (FSD §2.2) y tracing NFR-06 (`Observability Stack`).

---

*Generado por skill `docs/skills/trazabilidad/SKILL.md` · v2.1 · [Brief §A.6]*
