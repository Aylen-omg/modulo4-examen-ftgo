# ADR 0003 — Estrategia de Datos para FTGO

**Status:** Proposed
**Fecha:** 2026-05-24
**Autores:** Equipo de Arquitectura FTGO
**Fuentes:** Brief §A.4 | Richardson Cap. 5, Cap. 11
**Relacionado con:** ADR 0001 — Estilo Arquitectónico · ADR 0002 — Estrategia IPC

---

## 1. Contexto

En una arquitectura de microservicios, la forma en que los servicios acceden y persisten datos determina el acoplamiento entre equipos, la capacidad de escalar de forma independiente y la viabilidad de la migración Strangler Fig [Richardson Cap. 5]. El monolito FTGO actual usa una **única base de datos relacional** compartida por todos los módulos: cualquier cambio de esquema requiere coordinación entre squads y un deployment conjunto, lo que contradice NFR-03 (escalado independiente) y NFR-07 (extracción incremental de servicios).

ADR 0002 eligió comunicación híbrida REST + Kafka entre servicios. Esa decisión **exige** que cada servicio extraído sea dueño de sus datos: si dos servicios comparten tablas, la comunicación async por eventos pierde sentido y reaparece el acoplamiento del monolito [Richardson Cap. 4]. El diagrama C4 ya modela `Order DB` (PostgreSQL) como base dedicada al Order Service [C4 L2]; esta ADR formaliza la regla para el resto de capacidades.

Las restricciones que acotan la decisión son: consistencia **fuerte dentro del aggregate de pedido** y **eventual entre servicios** con lag ≤ 5 s p99 [NFR-05, PRD §4]; coexistencia con el monolito legacy 18–24 meses [NFR-07]; y stack preferido Java/Spring Boot + PostgreSQL en servicios core [Brief §A.4 — Tecnología].

---

## 2. Restricciones Consideradas

| Restricción | Valor | Impacto en la decisión |
|---|---|---|
| NFR-05 Consistencia | Fuerte en aggregate pedido; eventual ≤ 5 s entre servicios | Prohíbe transacciones 2PC entre servicios; favorece DB-per-service + Sagas |
| NFR-03 Escalabilidad | Escalado horizontal independiente por componente | Cada servicio debe poder escalar su persistencia sin afectar a otros |
| NFR-07 Migración incremental | Coexistencia con monolito 18–24 meses | La estrategia debe permitir dual-write o lectura desde monolito durante transición |
| NFR-06 Trazabilidad | Correlation ID en 100 % de acciones | Los eventos de dominio deben ser la fuente de verdad para sincronización entre DBs |
| ADR 0001 | Strangler Fig | Servicios extraídos migran datos progresivamente; monolito conserva esquema legacy hasta deprecación |
| ADR 0002 | Kafka + REST híbrido | DB compartida reintroduciría acoplamiento síncrono vía JOINs cross-schema |

---

## 3. Opciones Consideradas

---

### Opción 1: Base de datos compartida (status quo del monolito)

**Descripción:** todos los microservicios extraídos siguen leyendo y escribiendo en la misma instancia PostgreSQL del monolito, diferenciándose solo por convenciones de prefijo de tabla (ej. `order_*`, `kitchen_*`). No hay bases de datos físicamente separadas.

**Pros:**
- Migración de datos mínima al extraer un servicio: el código se mueve, la DB permanece → cubre NFR-07 en fase inicial.
- Consultas cross-capacidad con JOINs SQL simples → reporting operativo inmediato sin proyecciones.
- El equipo conoce el esquema legacy; curva de aprendizaje baja [Brief §A.4 — Tecnología].
- Transacciones locales ACID entre módulos aún extraídos si comparten conexión.

**Contras:**
- Acoplamiento de esquema: un cambio en tablas de Billing puede romper Order Taking → viola el principio de autonomía de microservicios [Richardson Cap. 5].
- Imposible escalar la carga de escritura de Delivery independientemente de Order → NFR-03 no cumplido.
- Riesgo de *Shared Database anti-pattern*: los servicios se convierten en fachadas del monolito de datos.
- Incompatible con el modelo de eventos Kafka de ADR 0002: los consumidores necesitan datos duplicados o acceso directo al esquema compartido.

**Impacto en NFRs:**

| NFR | Impacto |
|---|---|
| NFR-05 Consistencia | ✓ Consistencia fuerte fácil vía transacciones SQL locales |
| NFR-03 Escalabilidad | ✗ Un solo cuello de botella de I/O en la instancia compartida |
| NFR-07 Migración | ✓ Fase inicial simple; ✗ bloquea extracción real a largo plazo |
| NFR-06 Trazabilidad | ⚠ Posible, pero acoplamiento dificulta ownership claro por servicio |

**Compatibilidad Strangler Fig:** Parcial. Permite extraer código sin mover datos, pero prolonga la dependencia al esquema monolítico y retrasa los beneficios de la migración.

---

### Opción 2: Database-per-Service (una DB por microservicio)

**Descripción:** cada servicio extraído posee su propia base de datos (instancia o esquema lógicamente aislado con conexión exclusiva). Order Service → `Order DB`; Kitchen Service → `Kitchen DB`; Billing → `Billing DB`, etc. La sincronización entre servicios ocurre **solo** vía eventos Kafka [ADR 0002] o APIs REST; nunca vía JOIN cross-database [Richardson Cap. 5 — Database per Service].

**Pros:**
- Autonomía total de equipos: cada squad evoluciona su esquema sin coordinación global → cubre NFR-03.
- Alineación directa con ADR 0002: los eventos (`OrderCreated`, `PaymentConfirmed`) propagan datos entre bounded contexts → cubre NFR-05 (eventual entre servicios).
- Escalado independiente de lectura/escritura por servicio en horarios pico (Order vs. Delivery) → NFR-03.
- Patrón canónico de Richardson para FTGO [Richardson Cap. 5]; coherente con `Order DB` ya modelado en C4.

**Contras:**
- Consultas cross-servicio requieren API composition o CQRS/read models; reporting no puede usar JOINs ad hoc.
- Migración de datos del monolito es compleja: requiere dual-write o batch ETL por fase de Strangler Fig.
- Consistencia fuerte multi-servicio requiere Sagas [Richardson Cap. 4]; más complejidad que transacción local.
- Mayor costo operativo: múltiples instancias PostgreSQL (o clusters) que gestionar.

**Impacto en NFRs:**

| NFR | Impacto |
|---|---|
| NFR-05 Consistencia | ✓ Fuerte dentro de cada servicio; ✓ eventual vía Kafka ≤ 5 s p99 |
| NFR-03 Escalabilidad | ✓ Cada DB escala con su servicio |
| NFR-07 Migración | ✓ Compatible con extracción fase a fase; requiere plan de migración de datos por CAP |
| NFR-06 Trazabilidad | ✓ Correlation ID en eventos une operaciones cross-DB |

**Compatibilidad Strangler Fig:** Totalmente compatible. Cada extracción incluye migración de las tablas de esa CAP a su DB dedicada; el monolito conserva el resto hasta la siguiente fase.

---

### Opción 3: Esquemas separados en una sola instancia PostgreSQL

**Descripción:** una única instancia PostgreSQL con **esquemas lógicos** por servicio (`order_schema`, `kitchen_schema`, etc.). Cada servicio tiene credenciales que solo acceden a su esquema. Comparten hardware pero no tablas.

**Pros:**
- Menor costo operativo que múltiples instancias: un cluster PostgreSQL para todo FTGO.
- Aislamiento lógico de esquema evita JOINs accidentales si se enforced con permisos de DB.
- Migración desde monolito más simple que instancias separadas: `CREATE SCHEMA` + migración de tablas in-place.
- Backup y alta disponibilidad centralizados en una sola instancia.

**Contras:**
- Escalado independiente limitado: todos los esquemas comparten CPU, I/O y conexiones del mismo nodo → NFR-03 solo parcialmente cumplido.
- Un incidente en la instancia (corrupción, failover lento) afecta a **todos** los servicios → riesgo NFR-02.
- La tentación de hacer JOINs cross-schema es alta; requiere disciplina estricta y revisión de código.
- A medida que FTGO crece, la instancia única se convierte en cuello de botella difícil de particionar.

**Impacto en NFRs:**

| NFR | Impacto |
|---|---|
| NFR-05 Consistencia | ✓ Similar a Opción 2 dentro de cada esquema |
| NFR-03 Escalabilidad | ⚠ Mejor que shared tables; peor que instancias dedicadas |
| NFR-07 Migración | ✓ Transición gradual viable con esquemas en la misma instancia legacy |
| NFR-06 Trazabilidad | ✓ Igual que Opción 2 si la comunicación es por eventos |

**Compatibilidad Strangler Fig:** Compatible. Es un paso intermedio razonable hacia Database-per-Service completo.

---

## 4. Decisión

**Se elige:** Opción 2 — **Database-per-Service** (una base de datos dedicada por microservicio extraído).

**Justificación:** es la única opción que cumple NFR-03 y NFR-05 simultáneamente en el estado objetivo, y es coherente con ADR 0002 (Kafka como bus de sincronización) y ADR 0001 (extracción por CAP). La Opción 1 perpetúa el anti-pattern Shared Database [Richardson Cap. 5]. La Opción 3 es aceptable como **paso intermedio transitorio** durante las primeras extracciones (ej. Notifications), pero el objetivo final de cada CAP extraída es instancia dedicada o cluster propio antes de deprecar el monolito.

**Reglas de implementación:**
- Cada servicio es el **único escritor** de su base de datos.
- Ningún servicio accede directamente a la DB de otro servicio.
- Los datos necesarios cross-context se replican vía eventos Kafka o se consultan vía API REST (CQRS/read model cuando aplique).
- Durante Strangler Fig: fase transitoria permitida con esquema dedicado en instancia legacy compartida; criterio de salida = instancia propia o cluster aislado.

**Asignación inicial (coherente con C4):**

| Servicio | Base de datos | CAP |
|----------|---------------|-----|
| Order Service | `Order DB` (PostgreSQL 15) | CAP-03 |
| Kitchen Service | `Kitchen DB` (PostgreSQL 15) | CAP-04 |
| Delivery Service | `Delivery DB` (PostgreSQL 15) | CAP-05 |
| Billing Service | `Billing DB` (PostgreSQL 15) | CAP-06 |
| Notification Svc | Sin DB relacional (estado efímero; idempotencia en Redis) | CAP-07 |
| Monolito Legacy | DB monolítica existente | CAP-01, CAP-02 (parcial) |

**Referencias:** [Richardson Cap. 5 — Database per Service] + [Richardson Cap. 11 — CQRS] + [Brief §A.4 — Consistencia de datos] + ADR 0001 + ADR 0002.

---

## 5. Consecuencias

### ✅ Positivas

- Cada equipo puede desplegar cambios de esquema sin ventanas de coordinación global, acelerando la extracción Strangler Fig [NFR-07].
- Order Taking y Delivery pueden escalar sus bases de datos independientemente en horarios pico [NFR-03].
- El modelo de eventos de ADR 0002 tiene una base de datos clara: cada evento transporta el payload necesario para que el consumidor actualice su propia DB [NFR-05].
- Se elimina el riesgo de cambios en esquema de Billing que derriben Order Taking en producción.
- Alineación con el patrón FTGO del libro y con el `Order DB` ya documentado en el diagrama C4.

### ⚠ Negativas

- Las consultas de reporting cross-dominio (ej. "ingresos por restaurante en tiempo real") requieren un read model o data warehouse alimentado por eventos; no hay JOINs SQL directos.
- La migración de datos desde el monolito por cada CAP es un proyecto en sí mismo (dual-write, backfill, validación) que añade 2–4 semanas por servicio extraído.
- Las Sagas de compensación [Richardson Cap. 4] deben diseñarse explícitamente; un fallo en Billing tras crear pedido requiere lógica de reversión documentada en UC-04 FA-02/FA-03.
- Notification Service sin DB relacional necesita store de idempotencia (Redis) para evitar notificaciones duplicadas en reintentos Kafka — componente adicional a operar.

---

## 6. Follow-ups

- **Plan de migración de datos por CAP:** documentar por fase Strangler Fig qué tablas del monolito migran a cada DB dedicada, con criterios de validación (conteo de registros, checksum, pruebas de regresión).
- **POC recomendado:** extraer Order Service con `Order DB` dedicada y validar dual-write monolito ↔ Order DB durante 2 semanas en staging antes de cortar tráfico.
- **Read model / reporting:** evaluar proyección CQRS alimentada por tópicos Kafka para dashboards de back office (Empleado FTGO) sin acoplar servicios operativos.
