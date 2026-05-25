# Documento de Requerimientos de Producto (PRD) — FTGO
<!-- Salida de Semilla B.1 · Corrida C1 · Evaluación: 11/21 (52 %) -->

## 1. Contexto y Objetivos

FTGO (Food To Go) es una plataforma de entrega de comida que conecta consumidores, restaurantes y repartidores. Actualmente opera sobre un monolito Java/Spring Boot desplegado como WAR en un servidor de aplicaciones. El crecimiento de la plataforma ha generado problemas de escalabilidad, tiempo de despliegue y mantenibilidad.

El objetivo del proyecto es migrar hacia una arquitectura de microservicios para mejorar la autonomía de equipos, la escalabilidad independiente de componentes y la velocidad de despliegue.

## 2. Stakeholders

| Rol | Interés |
|---|---|
| Consumidor | Pedir comida de forma rápida y confiable |
| Restaurante | Gestionar menú y pedidos eficientemente |
| Repartidor | Recibir asignaciones de entrega |
| Product Manager | Velocidad de entrega de features |
| Equipo de Desarrollo | Calidad de código y autonomía de equipos |

## 3. Capacidades de Negocio

Las siguientes capacidades de negocio han sido identificadas para FTGO:

**CAP-01 — Consumer Management**
Permite registrar y gestionar el perfil de los consumidores, incluyendo historial de pedidos, direcciones guardadas y métodos de pago.

**CAP-02 — Restaurant Management**
Gestión de restaurantes en la plataforma: registro, aprobación, gestión de menús y horarios de operación.

**CAP-03 — Order Taking**
Recepción y validación de pedidos de los consumidores, incluyendo selección de items y confirmación.

**CAP-04 — Order Fulfillment / Kitchen**
Procesamiento interno del pedido en cocina: ticket de preparación, actualización de estado y notificación al equipo de entrega.

**CAP-05 — Delivery Management**
Asignación de pedidos a repartidores disponibles y seguimiento de la entrega en tiempo real.

**CAP-06 — Billing**
Procesamiento de pagos.

**CAP-07 — Notifications**
Envío de notificaciones a consumidores y restaurantes sobre el estado de los pedidos.

## 4. Requerimientos No Funcionales

**NFR-01 — Latencia**
La API debe responder en menos de 200ms en el percentil 95 para las operaciones críticas de usuario. [Brief §A.4 — Rendimiento]
*Justificación:* Los usuarios abandonan flujos de compra cuando la latencia supera 300ms; 200ms p95 deja margen para picos de tráfico.

**NFR-02 — Disponibilidad**
La plataforma debe mantener una disponibilidad del 99.9% mensual. [Brief §A.4 — Disponibilidad]
*Justificación:* Una plataforma de entrega opera en horas pico (almuerzo/cena); la caída en esas ventanas impacta directamente el revenue.

**NFR-03 — Escalabilidad**
El sistema debe soportar picos de tráfico elevados durante horas de alta demanda. El escalado debe ser independiente por componente.

**NFR-04 — Tolerancia a Fallos**
El sistema debe manejar fallos en servicios externos como el procesador de pagos mediante reintentos automáticos.

## 5. Alcance

**Dentro del alcance:**
- Migración de la lógica de pedidos, cocina, entrega, facturación y notificaciones
- Implementación de API Gateway
- Integración con pasarela de pagos (Stripe)
- Seguimiento en tiempo real de entregas

