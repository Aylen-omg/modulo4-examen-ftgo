# Documento de Requerimientos de Producto — FTGO
<!-- Salida de Semilla B.1 · Corrida C3 · Evaluación: 12/21 (57 %) -->

## 1. Contexto y Objetivos

FTGO opera actualmente como un monolito Java WAR. Los síntomas del "monolithic hell" (Richardson, Cap. 1) incluyen: builds de más de 30 minutos, imposibilidad de escalar componentes de forma independiente y acoplamiento entre equipos que trabajan sobre la misma base de código.

El objetivo estratégico es migrar a microservicios para habilitar despliegues independientes, escalado granular y resiliencia por dominio.

## 2. Stakeholders

| Rol | Interés principal |
|---|---|
| Consumidor | Experiencia rápida y confiable al pedir comida |
| Restaurante | Gestión eficiente de menús y pedidos |
| Repartidor (Courier) | Asignación clara y pago correcto |
| CTO | Reducción de deuda técnica y autonomía de equipos |
| Product Manager | Time-to-market de nuevas funcionalidades |
| Equipo de Ingeniería | Modularidad y despliegues sin regresiones |
| Equipo de Finanzas | Cumplimiento PCI-DSS en pagos |

## 3. Capacidades de Negocio

**CAP-01 — Consumer Management**
Gestión del ciclo de vida del consumidor: registro, autenticación, perfil, historial de pedidos y métodos de pago almacenados.

**CAP-02 — Restaurant Management**
Alta, configuración y mantenimiento de restaurantes en la plataforma. Incluye gestión de menús, disponibilidad de items y horarios.

**CAP-03 — Order Taking**
Recepción, validación y confirmación de pedidos. Incluye selección de restaurante, items del menú y dirección de entrega.

**CAP-04 — Order Fulfillment / Kitchen**
Coordinación del ticket de cocina: aceptación/rechazo del restaurante, preparación y notificación al equipo de entrega.

**CAP-05 — Delivery Management**
Asignación de pedidos a couriers disponibles, seguimiento en tiempo real y confirmación de entrega con firma digital.

**CAP-06 — Billing & Accounting**
Autorización y cobro del pago al confirmar el pedido. Soporte para reembolsos y conciliación con el procesador de pagos.

**CAP-07 — Notifications**
Notificaciones push, email y SMS a consumidores y restaurantes en cada transición de estado del pedido.

## 4. Requerimientos No Funcionales

**NFR-01 — Latencia**
La API debe responder en ≤ 200ms p95 para operaciones de usuario. [Brief §A.4 — Rendimiento]
*Justificación:* Superar 300ms correlaciona con abandono en apps de e-commerce; 200ms p95 es estándar de industria.

**NFR-02 — Disponibilidad**
99.9% de uptime mensual medido en ventanas de 30 días. [Brief §A.4 — Disponibilidad]
*Justificación:* La plataforma opera en picos de almuerzo/cena; la indisponibilidad en esos períodos impacta directamente el revenue.

**NFR-03 — Escalabilidad**
El sistema debe soportar 5× el tráfico base en picos de demanda mediante escalado horizontal automático. [Brief §A.4 — Escalabilidad]
*Justificación:* Las plataformas de delivery experimentan picos predecibles y eventos especiales; el escalado manual no es viable.

**NFR-04 — Tolerancia a Fallos**
Ante falla del procesador de pagos (Stripe), el sistema debe implementar circuit breaker y reintentos automáticos.

**NFR-05 — Observabilidad**
El sistema debe implementar logs centralizados y correlación de requests con un correlation ID por transacción.

## 5. Alcance

**Dentro del alcance:**
- Microservicios de Orders, Kitchen, Delivery, Billing y Notifications
- API Gateway como punto de entrada único
- Integraciones externas: Stripe (pagos), Google Maps (geolocalización)
- Dashboard administrativo para el equipo de FTGO

**Fuera del alcance:**
- Rediseño de la app móvil (UI solamente, no lógica de negocio)
- Funcionalidades de fidelización / puntos
- Soporte multi-idioma en esta versión

