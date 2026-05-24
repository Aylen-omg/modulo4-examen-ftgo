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

FTGO (Food To Go) opera desde hace varios años como una aplicación monolítica Java empaquetada como WAR. El monolito ha acumulado los síntomas clásicos del *monolithic hell* descritos en [Richardson Cap. 1]: ciclos de build lentos, escalado conflictivo entre módulos (Delivery y Billing deben escalar juntos aunque tengan demandas independientes), ausencia de aislamiento de fallos (un error en Notifications puede afectar el flujo de toma de pedidos), lock-in tecnológico que impide adoptar tecnologías más adecuadas por dominio, y un equipo creciente que pierde velocidad ante el tamaño de la base de código [Brief §A.1].

La dirección de FTGO ha decidido migrar hacia una arquitectura de microservicios para sostener el crecimiento del negocio. La migración es **incremental**: se aplica el patrón **Strangler Fig** durante un horizonte de **18 a 24 meses** [Brief §A.4 — Migración incremental]. El monolito existente no se reemplaza de golpe; los servicios nuevos se extraen progresivamente mientras el monolito sigue atendiendo las rutas aún no migradas. El objetivo de este PRD es documentar la **arquitectura objetivo** con suficiente claridad para que los equipos puedan iniciar la migración y para que el FSD y los ADRs subsiguientes tengan una base trazable.

---

## 2. Stakeholders

[Brief §A.2]

| Rol | Descripción | Necesidad principal |
|---|---|---|
| **Consumidor** | Usuario final (móvil/web) que ordena comida | UX rápida (≤ 200 ms p95), transparencia del estado del pedido, tracking en tiempo real |
| **Restaurante** | Negocio asociado que prepara la comida | Gestión de tickets de cocina, control de carga, dashboard de pedidos entrantes |
| **Courier** | Repartidor independiente que entrega los pedidos | Asignaciones cercanas, rutas optimizadas, pago confiable |
| **Empleado FTGO (back office)** | Personal interno: customer support, finanzas, operaciones | Visibilidad end-to-end, reportes operativos, resolución de incidentes |
| **Equipo de arquitectura** | Responsable del rediseño hacia microservicios | Calidad arquitectónica, trazabilidad de decisiones, mantenibilidad |
| **Stripe** | Pasarela de pago externa | Integración estable con SLA predecible; delegación de cumplimiento PCI-DSS |
| **Google Maps** | Servicio externo de mapas y geocoding | Geocoding de direcciones y rutas optimizadas para couriers |
| **SendGrid / Twilio** | Servicio externo de notificaciones (email / SMS / push) | Entrega confiable de confirmaciones, alertas y recibos |

---

## 3. Capacidades de Negocio

Las siguientes siete capacidades son las unidades de negocio estables identificadas por Richardson como candidatos naturales a microservicios [Richardson Cap. 2]. **Este PRD no prejuzga el nivel de descomposición**: cuántos microservicios crear y cómo agrupar estas capacidades se decide en los ADRs.

### CAP-01 — Consumer Management
[Richardson Cap. 2]

Gestiona el ciclo de vida completo de los consumidores: registro de cuenta, autenticación, perfiles, direcciones de entrega guardadas y preferencias. Es el punto de verdad sobre la identidad del consumidor dentro de la plataforma. Toda interacción de un consumidor con el sistema parte de una identidad válida en esta capacidad.

### CAP-02 — Restaurant Management
[Richardson Cap. 2]

Administra el catálogo de restaurantes asociados: alta y baja de restaurantes, gestión de menús (ítems, precios, disponibilidad), horarios de apertura y estado operativo en tiempo real. Proporciona al flujo de toma de pedidos la información necesaria para validar que el restaurante está activo y que los ítems seleccionados existen con precio vigente.

### CAP-03 — Order Taking
[Richardson Cap. 2]

Gestiona la toma de pedidos: selección de ítems, validación de disponibilidad del restaurante, cálculo del total (subtotal + cargo de entrega + impuestos), confirmación del pedido y emisión de un número único de pedido. Es la capacidad crítica del negocio; su disponibilidad está sujeta a NFR-02 (99.9 % mensual).

### CAP-04 — Order Fulfillment / Kitchen
[Richardson Cap. 2]

Traduce el pedido confirmado en un ticket de cocina para el restaurante. Gestiona el ciclo de vida del ticket: entregado al restaurante → aceptado (con tiempo estimado de preparación) o rechazado (con motivo) → en preparación → listo para retirar. Comunica los cambios de estado al consumidor vía CAP-07.

### CAP-05 — Delivery
[Richardson Cap. 2]

Orquesta la asignación de un courier disponible al pedido listo para retirar. Incluye selección del courier más cercano, oferta con timeout de 30 segundos, aceptación o rechazo por parte del courier, cálculo y presentación de ruta optimizada (integración con Google Maps), y tracking en tiempo real de la posición del courier hasta la entrega.

### CAP-06 — Billing & Accounting
[Richardson Cap. 2]

Gestiona los flujos de dinero: cobro al consumidor en el checkout (integración con Stripe), cálculo de comisiones de FTGO, y liquidación (payout) a restaurantes y couriers en los ciclos acordados. Debe poder encolar el cobro y reintentarlo si Stripe está temporalmente caído [Brief §A.4 — Tolerancia a fallos externos].

### CAP-07 — Notifications
[Richardson Cap. 2]

Entrega comunicaciones transaccionales a todos los actores: confirmación de pedido al consumidor, ticket entrante al restaurante, asignación al courier, y alertas operativas al back office. Usa SendGrid (email), Twilio (SMS) y push nativo. Es una capacidad satélite; su degradación no debe bloquear el flujo principal de toma de pedidos.

---

## 4. Requisitos No Funcionales

[Brief §A.4]

#### NFR-01: Latencia UX del consumidor
- **Métrica:** tiempo de respuesta percibido **≤ 200 ms p95** en todas las acciones del consumidor en la app (ver menú, agregar ítem, confirmar pedido).
- **Origen:** [Brief §A.4 — Latencia UX]
- **Justificación:** experiencia percibida crítica en horarios pico; latencias > 300 ms p95 correlacionan con abandono de sesión en apps de delivery.

#### NFR-02: Disponibilidad del flujo de pedidos
- **Métrica:** **≥ 99.9 % de uptime mensual** (≤ 43.8 min de downtime/mes) para Order Taking (CAP-03). Tracking en tiempo real (CAP-05) puede degradar a **≥ 99.5 %** mensual.
- **Origen:** [Brief §A.4 — Disponibilidad]
- **Justificación:** cualquier caída del flujo de pedidos impacta ingresos directamente; es la capacidad de mayor criticidad de negocio.

#### NFR-03: Escalabilidad horizontal
- **Métrica:** el sistema debe soportar **5× el tráfico baseline** en horarios de almuerzo (12:00–14:00) y cena (19:00–22:00) sin degradar NFR-01. Cada componente escala de forma independiente (X-axis: replicación; Y-axis: descomposición funcional) [Richardson Cap. 1 — Scale Cube].
- **Origen:** [Brief §A.4 — Carga] + [Brief §A.4 — Escalabilidad horizontal]
- **Justificación:** los picos de demanda son predecibles y concentrados; el monolito actual escala todo el WAR para aliviar un solo módulo, desperdiciando recursos.

#### NFR-04: Tolerancia a fallos externos
- **Métrica:** la toma de pedidos debe estar **operativa aunque Stripe esté caído**, encolando el cobro con retry automático (backoff exponencial, máximo 3 reintentos en 5 minutos). La degradación de Google Maps no debe impedir crear pedidos (routing degrada a estimación estática).
- **Origen:** [Brief §A.4 — Tolerancia a fallos externos]
- **Justificación:** no acoplar la disponibilidad del sistema a terceros; dependencias externas tienen SLAs propios que FTGO no controla.

#### NFR-05: Consistencia de datos
- **Métrica:** **consistencia fuerte** dentro del aggregate de un pedido (estado del pedido siempre coherente en la misma transacción de servicio). **Consistencia eventual** aceptada entre servicios para reporting (lag de propagación **≤ 5 segundos** p99).
- **Origen:** [Brief §A.4 — Consistencia de datos]
- **Justificación:** en una arquitectura de microservicios, los servicios no comparten base de datos [Richardson Cap. 4]; la consistencia eventual reduce contención y habilita alta disponibilidad, mientras la consistencia fuerte dentro del pedido evita estados corruptos.

#### NFR-06: Trazabilidad end-to-end
- **Métrica:** **100 % de las acciones del consumidor** deben emitir un correlation ID propagado a todos los servicios involucrados. El sistema debe soportar distributed tracing con retención de trazas de **≥ 30 días**.
- **Origen:** [Brief §A.4 — Trazabilidad]
- **Justificación:** en sistemas distribuidos, diagnosticar fallos en producción es inviable sin trazabilidad end-to-end; es también requisito de auditoría operativa y de soporte al cliente.

#### NFR-07: Migración incremental (Strangler Fig)
- **Métrica:** el sistema objetivo debe poder **coexistir con el monolito legacy durante 18 a 24 meses** sin interrupciones de servicio. Cada capacidad extraída se despliega con **zero-downtime deployment**.
- **Origen:** [Brief §A.4 — Migración incremental]
- **Justificación:** un reemplazo big-bang tiene riesgo operativo inaceptable; Strangler Fig permite validar cada extracción de forma incremental reduciendo el riesgo acumulado.

#### NFR-08: Cumplimiento regulatorio
- **Métrica:** datos de pago procesados exclusivamente a través de **Stripe** (delegación total de **PCI-DSS**; FTGO nunca almacena números de tarjeta). Datos de consumidores gestionados conforme a **GDPR** y regulaciones locales (derecho de borrado, portabilidad, minimización de datos).
- **Origen:** [Brief §A.4 — Cumplimiento]
- **Justificación:** incumplir PCI-DSS expone a multas y pérdida de licencia de procesamiento de pagos; incumplir GDPR expone a sanciones regulatorias en los mercados de operación.

---

## 5. Alcance

### ✅ Dentro del alcance

- Las **7 capacidades de negocio** del brief [Richardson Cap. 2]: Consumer Management, Restaurant Management, Order Taking, Order Fulfillment/Kitchen, Delivery, Billing & Accounting, Notifications.
- **Microservicios** derivados de las 7 capacidades — la descomposición exacta se decide en los ADRs, no en este PRD.
- **Mobile App** (consumidor) y **Web Admin** (back office / empleados FTGO) como clientes del sistema.
- **API Gateway** como punto de entrada unificado para clientes móviles y web.
- **Message broker** para comunicación asíncrona entre servicios (tecnología a decidir en ADR 0002).
- **Integraciones externas:** Stripe (pagos), Google Maps (rutas y geocoding), SendGrid/Twilio (notificaciones).
- **Observabilidad:** tracing distribuido con correlation ID y logging centralizado [NFR-06].
- **Estrategia de migración incremental** via Strangler Fig con coexistencia del monolito legacy [NFR-07].

### ❌ Fuera del alcance

- **Monolito legacy:** coexiste durante la migración vía Strangler Fig pero no se modifica ni documenta internamente en este PRD [Brief §A.4].
- **Implementación de código:** este PRD documenta la arquitectura objetivo, no el código fuente ni la configuración de infraestructura.
- **Cloud provider y orquestador específico:** la elección de AWS/GCP/Azure y Kubernetes/ECS se decide en ADRs posteriores.
- **Algoritmo interno de rutas:** delegado íntegramente a Google Maps; FTGO no implementa ruteo propio.
- **Módulos de negocio fuera del brief:** programa de fidelización, sistema de reseñas, publicidad de restaurantes — no están en [Brief §A.3] y se excluyen para evitar inventar dominio [Brief §A.6].
- **Módulo de detección de fraude** más allá del cumplimiento PCI-DSS delegado a Stripe y GDPR básico.

---

*Trazabilidad: cada elemento de este PRD puede rastrearse a [Brief §A.1], [Brief §A.2], [Brief §A.3], [Brief §A.4] o [Richardson Cap. 1–2]. Ningún stakeholder, capacidad ni NFR ha sido inventado fuera de estas fuentes [Brief §A.6].*
