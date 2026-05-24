# PRD Ligero — FTGO (Food To Go)

| Campo       | Valor                                                              |
|-------------|---------------------------------------------------------------------|
| ID          | PR-PRD-FTGO-001                                                    |
| Versión     | v0.2-mejorado                                                      |
| Artefacto   | Product Requirements Document (ligero)                             |
| Fecha       | 2026-05-24                                                         |
| Autor       | Arquitecto de soluciones — Equipo de arquitectura FTGO             |
| Estado      | Aprobado para derivar FSD y ADRs                                   |
| Fuentes     | Brief Anexo A · Richardson "Microservices Patterns" Manning 2019   |

---

## 1. Contexto y Objetivos

FTGO (Food To Go) opera desde hace varios años como una aplicación monolítica Java empaquetada como WAR. El monolito ha acumulado los síntomas clásicos del *monolithic hell* descritos en [Richardson Cap. 1]: ciclos de build lentos, escalado conflictivo entre módulos (el módulo de Delivery requiere más instancias que el módulo de Billing, pero ambos escalan juntos), ausencia de aislamiento de fallos (un error en Notifications puede tumbar el flujo de toma de pedidos), lock-in tecnológico que impide adoptar tecnologías más adecuadas por dominio, y un equipo creciente que pierde velocidad a medida que la base de código crece [Brief §A.1].

La dirección de FTGO ha decidido migrar hacia una arquitectura de microservicios para sostener el crecimiento del negocio. Esta migración es **incremental**, siguiendo el patrón **Strangler Fig** durante un horizonte de **18 a 24 meses** [Brief §A.4 — Migración incremental]. El monolito existente no se reemplaza de golpe: los servicios nuevos se extraen progresivamente mientras el monolito sigue atendiendo las rutas no migradas. El objetivo de este PRD es documentar la **arquitectura objetivo** con suficiente claridad para que los equipos de desarrollo puedan iniciar la migración y para que los ADRs y el FSD subsiguientes tengan una base trazable.

---

## 2. Stakeholders

[Brief §A.2]

| Rol | Descripción | Necesidad principal |
|---|---|---|
| **Consumidor** | Usuario final (móvil/web) que ordena comida | UX rápida (< 200 ms p95), transparencia del estado del pedido, tracking en tiempo real |
| **Restaurante** | Negocio asociado que prepara la comida | Gestión de tickets de cocina, control de carga, dashboard de pedidos entrantes |
| **Courier** | Repartidor independiente que entrega los pedidos | Asignaciones cercanas a su posición, rutas optimizadas, pago confiable |
| **Empleado FTGO (back office)** | Personal interno: customer support, finanzas, operaciones | Visibilidad end-to-end, reportes operativos, resolución de incidentes |
| **Equipo de arquitectura** | Responsable del rediseño hacia microservicios | Calidad arquitectónica, trazabilidad de decisiones, mantenibilidad a largo plazo |
| **Stripe** | Pasarela de pago externa | Integración estable con SLA predecible; delegación de cumplimiento PCI-DSS |
| **Google Maps** | Servicio externo de mapas y geocoding | Rutas optimizadas para couriers; puede degradar temporalmente sin bloquear pedidos |
| **SendGrid / Twilio** | Servicio externo de notificaciones (email / SMS / push) | Entrega confiable de confirmaciones, alertas y recibos a consumidores y restaurantes |

---

## 3. Capacidades de Negocio

Las siguientes siete capacidades son las unidades de negocio estables identificadas por Richardson como candidatos naturales a microservicios [Richardson Cap. 2]. **Este PRD no prejuzga el nivel de descomposición**: la decisión de cuántos microservicios crear y cómo agrupar estas capacidades se toma en los ADRs.

### CAP-01 — Consumer Management
[Richardson Cap. 2]

Gestiona el ciclo de vida completo de los consumidores: registro de cuenta, autenticación, perfiles, direcciones de entrega guardadas y preferencias. Es el punto de verdad sobre la identidad del consumidor dentro de la plataforma. Toda interacción de un consumidor con el sistema (crear pedido, ver historial, configurar notificaciones) parte de una identidad válida en esta capacidad.

### CAP-02 — Restaurant Management
[Richardson Cap. 2]

Administra el catálogo de restaurantes asociados: alta y baja de restaurantes, gestión de menús (ítems, precios, fotos), horarios de apertura y disponibilidad en tiempo real. Proporciona al flujo de toma de pedidos la información necesaria para validar que el restaurante está activo y que los ítems seleccionados existen y tienen precio vigente.

### CAP-03 — Order Taking
[Richardson Cap. 2]

Responsable de recibir una solicitud de pedido de un consumidor, validar la disponibilidad del restaurante y los ítems, calcular el total (incluyendo cargos de entrega), confirmar el pedido y emitir un identificador único de pedido. Es la capacidad crítica del negocio: su disponibilidad está sujeta al NFR-03 (99.9 % mensual).

### CAP-04 — Order Fulfillment / Kitchen
[Richardson Cap. 2]

Traduce el pedido confirmado en un ticket de cocina para el restaurante. Gestiona el ciclo de vida del ticket: entregado al restaurante → aceptado (con tiempo estimado) o rechazado (con motivo) → en preparación → listo para retirar. Comunica los cambios de estado al consumidor a través de la capacidad de Notifications.

### CAP-05 — Delivery
[Richardson Cap. 2]

Orquesta la asignación de un courier disponible al pedido listo para retirar. Incluye: selección del courier más cercano, oferta con timeout de 30 segundos, aceptación o rechazo por parte del courier, cálculo y presentación de ruta optimizada (integración con Google Maps), y tracking en tiempo real de la posición del courier hasta la entrega al consumidor.

### CAP-06 — Billing & Accounting
[Richardson Cap. 2]

Gestiona los flujos de dinero de la plataforma: cobro al consumidor al momento del checkout (integración con Stripe), cálculo de comisiones de FTGO, y liquidación (payout) a restaurantes y couriers en los ciclos acordados. Debe poder encolar el cobro si Stripe está temporalmente caído [Brief §A.4 — Tolerancia a fallos externos].

### CAP-07 — Notifications
[Richardson Cap. 2]

Entrega comunicaciones transaccionales a los actores del sistema: confirmación de pedido al consumidor, ticket entrante al restaurante, asignación de entrega al courier, y alertas operativas al back office. Usa SendGrid (email), Twilio (SMS) y push nativo. Es una capacidad satélite: su degradación no debe bloquear el flujo principal de toma de pedidos.

---

## 4. Requisitos No Funcionales

[Brief §A.4]

#### NFR-01: Carga en horarios pico
- **Métrica:** el sistema debe soportar **5× el tráfico baseline** durante los picos de almuerzo (12:00–14:00) y cena (19:00–22:00) hora local, sin degradación del tiempo de respuesta por encima de los umbrales de NFR-02.
- **Origen:** [Brief §A.4 — Carga]
- **Justificación:** los horarios de comida concentran la demanda de delivery; sin escalado diferenciado, el monolito actual colapsa en pico, bloqueando ingresos críticos del negocio.

#### NFR-02: Latencia UX del consumidor
- **Métrica:** tiempo de respuesta percibido **≤ 200 ms p95** para todas las acciones del consumidor en la app (ver menú, agregar ítem, confirmar pedido).
- **Origen:** [Brief §A.4 — Latencia UX]
- **Justificación:** en aplicaciones móviles de delivery, latencias > 300 ms p95 correlacionan con abandono de sesión; 200 ms p95 es el umbral de experiencia "instantánea".

#### NFR-03: Disponibilidad del flujo de toma de pedidos
- **Métrica:** **≥ 99.9 % de uptime mensual** (≤ 43.8 minutos de downtime/mes) para el flujo Order Taking (CAP-03). El tracking en tiempo real (CAP-05) puede degradar a **≥ 99.5 %** mensual.
- **Origen:** [Brief §A.4 — Disponibilidad]
- **Justificación:** Order Taking es la capacidad que genera ingresos directamente; cualquier caída impacta revenue y confianza del consumidor.

#### NFR-04: Tolerancia a fallos de sistemas externos
- **Métrica:** el sistema debe poder **completar la toma de pedidos aunque Stripe esté caído**, encolando el cobro con reintentos automáticos (retry con backoff exponencial, máximo **3 reintentos en 5 minutos**). La degradación de Google Maps no debe impedir crear pedidos (routing se degrada a estimación estática).
- **Origen:** [Brief §A.4 — Tolerancia a fallos externos]
- **Justificación:** dependencias externas tienen SLAs propios que no controla FTGO; acoplar el flujo principal a su disponibilidad introduce puntos de fallo únicos inaceptables.

#### NFR-05: Escalabilidad horizontal independiente
- **Métrica:** cada componente del sistema debe poder escalarse de forma independiente (**X-axis: replicación horizontal**; **Y-axis: descomposición funcional**) sin requerir escalado de componentes no relacionados. Tiempo de aprovisionamiento de una instancia adicional: **≤ 3 minutos**.
- **Origen:** [Brief §A.4 — Escalabilidad horizontal]
- **Justificación:** el monolito actual requiere escalar todo el WAR para aliviar carga en un solo módulo (ej. Delivery en pico); la arquitectura objetivo elimina este desperdicio.

#### NFR-06: Consistencia de datos
- **Métrica:** **consistencia eventual** aceptada para reporting y agregados entre servicios (retraso de propagación ≤ 2 segundos p99). **Consistencia fuerte** requerida dentro del aggregate de un pedido (estado del pedido siempre coherente en la misma transacción de servicio).
- **Origen:** [Brief §A.4 — Consistencia de datos]
- **Justificación:** la consistencia eventual reduce contención y permite disponibilidad alta entre servicios; la consistencia fuerte dentro del pedido evita estados corruptos (ej. pedido cobrado pero no confirmado).

#### NFR-07: Trazabilidad end-to-end
- **Métrica:** **100 % de las acciones del consumidor** deben emitir un correlation ID propagado a todos los servicios involucrados. El sistema debe soportar distributed tracing con retención de trazas de **≥ 7 días**.
- **Origen:** [Brief §A.4 — Trazabilidad]
- **Justificación:** en una arquitectura de microservicios, la depuración de fallos en producción es inviable sin trazabilidad distribuida; es también requisito de auditoría operativa.

#### NFR-08: Migración incremental (Strangler Fig)
- **Métrica:** el sistema debe poder coexistir con el monolito legacy durante **18 a 24 meses** sin interrupciones de servicio. Cada capacidad extraída debe poder desplegarse independientemente con **zero-downtime deployment**.
- **Origen:** [Brief §A.4 — Migración incremental]
- **Justificación:** un reemplazo big-bang tiene riesgo operativo inaceptable para un sistema en producción con usuarios activos; Strangler Fig permite validar cada extracción de forma incremental.

#### NFR-09: Pila tecnológica preferida
- **Métrica:** los servicios del core (Order Taking, Order Fulfillment, Delivery) deben implementarse en **Java 17+ / Spring Boot 3.x** para reutilizar conocimiento del equipo y reducir curva de aprendizaje. Servicios satélite (Notifications, integraciones) tienen libertad tecnológica.
- **Origen:** [Brief §A.4 — Tecnología]
- **Justificación:** el equipo existente tiene expertise en Java/Spring; cambiar el stack core simultáneamente con la migración arquitectónica multiplica el riesgo de la transición.

#### NFR-10: Cumplimiento regulatorio
- **Métrica:** datos de pago deben procesarse exclusivamente a través de **Stripe** (delegación total de cumplimiento **PCI-DSS**; FTGO nunca almacena números de tarjeta). Datos de consumidores deben gestionarse conforme a **GDPR** y regulaciones locales vigentes (derecho de borrado, portabilidad, minimización).
- **Origen:** [Brief §A.4 — Cumplimiento]
- **Justificación:** incumplir PCI-DSS expone a FTGO a multas y pérdida de licencia de procesamiento de pagos; incumplir GDPR expone a sanciones regulatorias en los mercados donde opera.

---

## 5. Alcance

### ✅ Dentro del alcance (arquitectura objetivo documentada en este PRD)

- Las 7 capacidades de negocio identificadas en [Richardson Cap. 2]: Consumer Management, Restaurant Management, Order Taking, Order Fulfillment/Kitchen, Delivery, Billing & Accounting, Notifications.
- Los 4 actores humanos del brief: Consumidor, Restaurante, Courier, Empleado FTGO (back office). [Brief §A.2]
- Los 3 sistemas externos de integración: Stripe (pagos), Google Maps (mapas/rutas), SendGrid/Twilio (notificaciones). [Brief §A.2]
- Los 10 NFRs listados en la sección 4, cada uno trazable a [Brief §A.4].
- La estrategia de migración incremental mediante **Strangler Fig** durante 18–24 meses, con coexistencia del monolito legacy. [Brief §A.4]
- La documentación de la arquitectura **objetivo** (no el estado actual del monolito).

### ❌ Fuera del alcance (no se documenta en este PRD)

- **Diseño detallado de APIs**: contratos de interfaz, esquemas de datos, versioning — corresponde al FSD y a los ADRs técnicos.
- **Descomposición exacta en microservicios**: cuántos servicios, cómo agrupar capacidades, estrategia de datos — corresponde a ADR 0001 y ADR 0002.
- **Infraestructura y DevOps**: orquestación de contenedores (Kubernetes), CI/CD, observabilidad operativa — fuera del alcance del laboratorio [Brief §A.6].
- **Funcionalidades del monolito legacy no migradas**: el monolito sigue vivo y atendiendo rutas no extraídas; su documentación interna no es parte de este ejercicio.
- **Módulos de negocio no cubiertos por el brief**: programa de fidelización, sistema de reseñas, publicidad de restaurantes — no están en [Brief §A.3] ni en [Richardson Cap. 2] y se excluyen para evitar inventar dominio [Brief §A.6].
- **Implementación física**: código, configuración de servidores, pipelines de datos.

---

*Trazabilidad: cada elemento de este PRD puede rastrearse a [Brief §A.1], [Brief §A.2], [Brief §A.3], [Brief §A.4] o [Richardson Cap. 1–2]. Ningún stakeholder, capacidad ni NFR ha sido inventado fuera de estas fuentes.*
