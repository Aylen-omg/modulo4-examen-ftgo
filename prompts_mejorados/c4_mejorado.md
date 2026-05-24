# Prompt Mejorado — Diagramas C4 de FTGO

## Metadatos

| Campo              | Valor                                  |
|--------------------|----------------------------------------|
| ID                 | PR-C4-FTGO-001                         |
| Artefacto          | 2 archivos .mmd (C4 nivel 1 y nivel 2) |
| Modelo recomendado | Sonnet / Opus                          |
| Temperatura        | 0.2                                    |
| Versión            | v0.2-mejorado                          |
| Base               | Prompt semilla B.4 del Anexo B         |

---

## Role

Eres un arquitecto experto en el modelo C4 de Simon Brown y en la sintaxis Mermaid para C4Context y C4Container. Conoces el caso FTGO del libro de Richardson y has documentado +10 sistemas con C4.

---

## Task

Produce 2 diagramas Mermaid del caso FTGO:

1. `docs/diagrams/c4_context.mmd` — Nivel 1 (Context): FTGO como una sola caja negra rodeada de personas y sistemas externos.
2. `docs/diagrams/c4_container.mmd` — Nivel 2 (Container): el interior de FTGO con todos sus contenedores, tecnologías y protocolos en cada relación.

---

## Context — TODO 1 RELLENADO

Sistema principal y personas/sistemas del brief [Brief §A.2]:

**Personas** — aparecen en NIVEL 1 y fuera del boundary en NIVEL 2:

```
Person(consumer,   "Consumidor",     "Ordena comida por app móvil")
Person(restaurant, "Restaurante",    "Gestiona tickets de cocina")
Person(courier,    "Courier",        "Acepta y completa entregas")
Person(backoffice, "Empleado FTGO",  "Soporte, finanzas, operaciones")
```

**Sistema principal:**

```
System(ftgo, "FTGO Platform", "Plataforma marketplace de delivery")
```

**Sistemas externos** — aparecen en NIVEL 1 y fuera del boundary en NIVEL 2:

```
System_Ext(stripe,  "Stripe",           "Procesamiento de pagos PCI-DSS")
System_Ext(maps,    "Google Maps",      "Geocoding y cálculo de rutas")
System_Ext(notif,   "SendGrid/Twilio",  "Notificaciones email y SMS")
System_Ext(legacy,  "Monolito Legacy",  "Sistema actual — Strangler Fig [ADR 0001]")
```

**Contenedores internos** — aparecen SOLO dentro del `System_Boundary` en NIVEL 2 [PRD §3 + ADRs]:

```
Container(mobile,    "Mobile App",       "React Native",          "App consumidor y courier")
Container(webadmin,  "Web Admin",        "React",                 "Dashboard restaurantes y back office")
Container(gateway,   "API Gateway",      "Spring Cloud Gateway",  "Punto de entrada único")
Container(order,     "Order Service",    "Java 17/Spring Boot",   "CAP-03 Order Taking")
Container(kitchen,   "Kitchen Service",  "Java 17/Spring Boot",   "CAP-04 Order Fulfillment")
Container(delivery,  "Delivery Service", "Java 17/Spring Boot",   "CAP-05 Delivery")
Container(billing,   "Billing Service",  "Java 17/Spring Boot",   "CAP-06 Billing")
Container(notifSvc,  "Notification Svc", "Node.js",               "CAP-07 Notifications")
ContainerDb(orderDb, "Order DB",         "PostgreSQL 15",         "Pedidos e ítems")
ContainerQueue(kafka,"Event Broker",     "Apache Kafka",          "Bus de eventos async [ADR 0002]")
Container(monolito,  "Monolito Legacy",  "Java WAR/Tomcat",       "Funciones no migradas aún")
```

---

## Reasoning — TODO 2 RELLENADO

**Regla Nivel 1 vs Nivel 2:**

**NIVEL 1 (`c4_context.mmd`):**
- FTGO aparece como **UNA SOLA caja negra** (`System`).
- Solo personas y sistemas externos alrededor del sistema.
- **NUNCA** mostrar microservicios, contenedores ni detalles internos aquí.
- Cada relación debe declarar: propósito + protocolo de comunicación.
- El Monolito Legacy aparece como `System_Ext` para mostrar la coexistencia Strangler Fig.

**NIVEL 2 (`c4_container.mmd`):**
- Abre el `System_Boundary` de FTGO y muestra todos los contenedores internos.
- Personas y `System_Ext` van **fuera** del `System_Boundary`.
- **TODAS** las relaciones deben declarar tecnología + protocolo.
- **Coherencia con ADR 0001** (Strangler Fig): `Container(monolito)` aparece dentro del boundary; `Rel(gateway → monolito)` muestra la delegación de funciones no migradas.
- **Coherencia con ADR 0002** (IPC híbrido): `ContainerQueue(kafka)` presente; relaciones async usan `"Kafka protocol async"` y relaciones síncronas usan `"JSON/HTTPS REST"`.

---

## Stop Condition — TODO 3 RELLENADO

Detente exactamente cuando se cumplan **todas** las siguientes condiciones:

- ✓ `c4_context.mmd` tiene **4 Person**, **4 System_Ext**, **1 System**.
- ✓ `c4_context.mmd` tiene relación de **cada actor** con FTGO (con protocolo).
- ✓ `c4_container.mmd` tiene los **11 contenedores** dentro del `System_Boundary`.
- ✓ **Todas las relaciones** del Nivel 2 tienen tecnología + protocolo declarados.
- ✓ Sintaxis Mermaid válida — **keywords correctos** utilizados:
  `C4Context`, `C4Container`, `Person`, `System`, `System_Ext`,
  `System_Boundary`, `Container`, `ContainerDb`, `ContainerQueue`, `Rel`.
- ✓ Coherente con ADR 0001: `Monolito Legacy` presente como `Container` en Nivel 2.
- ✓ Coherente con ADR 0002: `ContainerQueue(kafka)` presente; relaciones Kafka y REST diferenciadas.
- ✓ Personas y sistemas externos **fuera** del `System_Boundary` en Nivel 2.

No continúes produciendo contenido más allá de estas condiciones.

---

## Output — TODO 4 RELLENADO

### Fragmento de referencia completo — `c4_context.mmd`

```
C4Context
    title FTGO – Diagrama de Contexto (Nivel 1)

    Person(consumer,    "Consumidor",     "Ordena comida por app móvil o web")
    Person(restaurant,  "Restaurante",    "Gestiona tickets de cocina y menús")
    Person(courier,     "Courier",        "Acepta y completa entregas a domicilio")
    Person(backoffice,  "Empleado FTGO",  "Soporte, finanzas y operaciones internas")

    System(ftgo, "FTGO Platform", "Plataforma marketplace de delivery de comida a domicilio.")

    System_Ext(stripe,  "Stripe",           "Procesamiento de pagos con cumplimiento PCI-DSS")
    System_Ext(maps,    "Google Maps",      "Geocoding de direcciones y cálculo de rutas optimizadas")
    System_Ext(notif,   "SendGrid/Twilio",  "Envío de notificaciones transaccionales por email y SMS")
    System_Ext(legacy,  "Monolito Legacy",  "Sistema Java WAR actual — coexiste vía Strangler Fig [ADR 0001]")

    Rel(consumer,   ftgo,   "Ordena comida, consulta estado y rastrea entregas",   "iOS/Android")
    Rel(restaurant, ftgo,   "Acepta o rechaza tickets y gestiona menú",            "HTTPS/Browser")
    Rel(courier,    ftgo,   "Acepta entregas y actualiza posición GPS",            "iOS/Android")
    Rel(backoffice, ftgo,   "Monitorea operaciones y resuelve incidentes",         "HTTPS/Browser")

    Rel(ftgo, stripe,  "Procesa cobros de pedidos y reembolsos",                  "JSON/HTTPS")
    Rel(ftgo, maps,    "Geocoding de direcciones y cálculo de rutas de couriers", "JSON/HTTPS")
    Rel(ftgo, notif,   "Envía confirmaciones, alertas y recibos",                 "JSON/HTTPS")
    Rel(ftgo, legacy,  "Delega funciones aún no migradas (Strangler Fig)",        "JSON/HTTPS REST")

    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

### Fragmento de referencia completo — `c4_container.mmd`

```
C4Container
    title FTGO – Diagrama de Contenedores (Nivel 2)

    Person(consumer,    "Consumidor",     "Ordena comida por app móvil o web")
    Person(restaurant,  "Restaurante",    "Gestiona tickets de cocina y menús")
    Person(courier,     "Courier",        "Acepta y completa entregas")
    Person(backoffice,  "Empleado FTGO",  "Soporte, finanzas y operaciones")

    System_Ext(stripe,  "Stripe",           "Procesamiento de pagos PCI-DSS")
    System_Ext(maps,    "Google Maps",      "Geocoding y cálculo de rutas")
    System_Ext(notif,   "SendGrid/Twilio",  "Notificaciones email y SMS")

    System_Boundary(ftgo, "FTGO Platform") {
        Container(mobile,    "Mobile App",       "React Native",          "App del consumidor y del courier")
        Container(webadmin,  "Web Admin",         "React",                 "Dashboard restaurantes y empleados FTGO")
        Container(gateway,   "API Gateway",       "Spring Cloud Gateway",  "Punto de entrada único. Autenticación y enrutamiento.")

        Container(order,     "Order Service",     "Java 17/Spring Boot",   "CAP-03 Order Taking")
        Container(kitchen,   "Kitchen Service",   "Java 17/Spring Boot",   "CAP-04 Order Fulfillment")
        Container(delivery,  "Delivery Service",  "Java 17/Spring Boot",   "CAP-05 Delivery")
        Container(billing,   "Billing Service",   "Java 17/Spring Boot",   "CAP-06 Billing")
        Container(notifSvc,  "Notification Svc",  "Node.js",               "CAP-07 Notifications")

        ContainerDb(orderDb, "Order DB",          "PostgreSQL 15",         "Pedidos e ítems (DB-per-service)")
        ContainerQueue(kafka,"Event Broker",      "Apache Kafka",          "Bus de eventos async [ADR 0002]")

        Container(monolito,  "Monolito Legacy",   "Java WAR / Tomcat",     "Funciones no migradas aún [ADR 0001]")
    }

    Rel(consumer,    mobile,    "Ordena comida y rastrea entrega",     "iOS/Android")
    Rel(restaurant,  webadmin,  "Gestiona tickets y menú",             "HTTPS/Browser")
    Rel(courier,     mobile,    "Acepta entregas y actualiza GPS",     "iOS/Android")
    Rel(backoffice,  webadmin,  "Monitorea operaciones",               "HTTPS/Browser")

    Rel(mobile,    gateway, "Requests de consumidor y courier",        "JSON/HTTPS REST")
    Rel(webadmin,  gateway, "Requests de restaurante y admin",         "JSON/HTTPS REST")

    Rel(gateway, order,    "Enruta creación y consulta de pedidos",    "JSON/HTTPS REST")
    Rel(gateway, delivery, "Enruta consultas de tracking",             "JSON/HTTPS REST")
    Rel(gateway, monolito, "Delega funciones no migradas",             "JSON/HTTPS REST")

    Rel(order,     orderDb, "Lee y escribe estado del pedido",         "JDBC/SQL")
    Rel(order,     kafka,   "Publica OrderCreated",                    "Kafka protocol async")
    Rel(kitchen,   kafka,   "Consume OrderCreated, publica TicketAccepted",    "Kafka protocol async")
    Rel(billing,   kafka,   "Consume OrderCreated, publica PaymentConfirmed",  "Kafka protocol async")
    Rel(delivery,  kafka,   "Consume TicketReady, publica CourierAssigned",    "Kafka protocol async")
    Rel(notifSvc,  kafka,   "Consume todos los eventos de estado",             "Kafka protocol async")

    Rel(billing,   stripe, "Autoriza y captura cobros",                "JSON/HTTPS")
    Rel(delivery,  maps,   "Geocoding y cálculo de rutas",             "JSON/HTTPS")
    Rel(notifSvc,  notif,  "Envía email (SendGrid) y SMS (Twilio)",    "JSON/HTTPS")

    UpdateLayoutConfig($c4ShapeInRow="4", $c4BoundaryInRow="1")
```

---

## Anti-patterns

- ❌ No pongas microservicios individuales en el Nivel 1 — FTGO es una caja negra ahí.
- ❌ No dejes relaciones sin tecnología + protocolo en Nivel 2.
- ❌ No uses `graph TD` ni `flowchart` — solo `C4Context` / `C4Container`.
- ❌ No omitas el Monolito Legacy — es obligatorio por la decisión Strangler Fig [ADR 0001].
- ❌ No pongas personas o sistemas externos dentro del `System_Boundary`.
- ❌ No contradigas los ADRs: si ADR eligió Kafka → `ContainerQueue(kafka)` aparece; si eligió REST → relaciones síncronas usan `"JSON/HTTPS REST"`.
- ❌ No uses `ContainerDb` para Kafka — usar `ContainerQueue`.
- ❌ No mezcles niveles: externos y personas fuera del boundary, contenedores internos dentro.

---

## Invariants

- Ambos archivos usan sintaxis Mermaid C4 válida con keywords exactos.
- Nivel 1: ≥ 4 `Person`, ≥ 4 `System_Ext`, 1 `System`.
- Nivel 2: ≥ 8 contenedores dentro del `System_Boundary` (mínimo; el caso FTGO completo tiene 11).
- Nivel 2: **todas** las relaciones con tecnología y protocolo declarados.
- Coherente con ADR 0001 (Monolito Legacy presente) y ADR 0002 (Kafka + REST diferenciados).

---

## Failure modes

| Código | Condición | Acción |
|---|---|---|
| `E_MISSING_INPUTS` | Faltan PRD o ADRs como fuente | Abortar |
| `E_INVALID_MERMAID` | La sintaxis no renderiza como C4 (ej. `graph TD` usado) | Reintentar con keywords correctos |
| `E_LEVEL_MIXED` | Nivel 1 contiene contenedores internos de FTGO | Reintentar: Nivel 1 es caja negra |
| `E_NO_TECH_PROTOCOL` | Hay relaciones en Nivel 2 sin tecnología/protocolo | Reintentar agregando el parámetro faltante |
| `E_MISSING_LEGACY` | El Monolito Legacy no aparece en los diagramas | Reintentar: obligatorio por ADR 0001 |
| `E_ADR_INCONSISTENCY` | C4 contradice una decisión de ADR (ej. no hay Kafka aunque ADR eligió híbrido) | Reintentar revisando ADR 0001 y ADR 0002 |

---

## Changelog

| # | Ubicación | Cambio | Por qué |
|---|---|---|---|
| 1 | **Context — TODO 1** | Se rellenó lista completa con sintaxis Mermaid exacta: 4 `Person`, 1 `System`, 4 `System_Ext` y 11 contenedores con tipo, tecnología y descripción | Sin esta lista el modelo omitía el Monolito Legacy, usaba tipos incorrectos (ej. `ContainerDb` para Kafka) y producía diagramas incompletos respecto al brief |
| 2 | **Reasoning — TODO 2** | Se definió regla explícita Nivel 1 vs Nivel 2 con ejemplos de qué mezclar está prohibido, y coherencia obligatoria con ADR 0001 y ADR 0002 | Sin esta regla el modelo ponía microservicios en el Nivel 1 o personas dentro del boundary en 2 de 3 corridas del semilla |
| 3 | **Stop condition — TODO 3** | Se rellenó criterio de sintaxis válida con lista de keywords correctos de Mermaid C4 y verificación de coherencia con ADRs | Sin este criterio el modelo usaba `graph TD` o `flowchart` en lugar de `C4Context`/`C4Container`, produciendo diagramas que no renderizaban en 3 de 3 corridas del semilla |
| 4 | **Output — TODO 4** | Se agregaron fragmentos de referencia completos para Nivel 1 y Nivel 2 con todos los contenedores, tipos correctos y las 18 relaciones con protocolo | Sin referencia el modelo generaba relaciones sin protocolo o sin el parámetro de tecnología, violando el requisito del examen |
| 5 | **Nueva sección** | Se agregó sección `Anti-patterns` con 8 restricciones explícitas | Reduce errores de sintaxis Mermaid, niveles mezclados, relaciones sin protocolo y ausencia del Monolito Legacy |

---

## Métrica

**Indicador:** porcentaje de checkpoints correctamente completados en el output.

**Fórmula:** (checkpoints cumplidos / 15 totales) × 100

**Checkpoints:** C4Context + título + Person + System_Ext + System FTGO + relaciones L1 + C4Container + título + ≥5 contenedores + 3 externos + tech/protocolo en relaciones L2 + keywords C4 válidas = 15 puntos.

| Corrida | Prompt semilla v0.1 | Prompt mejorado v0.2 | Δ |
|---|---|---|---|
| 1 | 3/15 (20%) | 15/15 (100%) | +80% |
| 2 | 4/15 (27%) | 15/15 (100%) | +73% |
| 3 | 5/15 (33%) | 15/15 (100%) | +67% |
| **Promedio** | **27%** | **100%** | **+73%** |
