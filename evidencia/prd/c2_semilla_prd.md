# PRD — FTGO Platform
<!-- Salida de Semilla B.1 · Corrida C2 · Evaluación: 11/21 (52 %) -->

## 1. Contexto

FTGO es una plataforma de entrega de comida que enfrenta problemas típicos del monolito: despliegues lentos, equipos acoplados y dificultad para escalar componentes de forma independiente. La decisión es migrar a microservicios.

**Objetivo:** Descomponer el monolito en servicios autónomos desplegables de forma independiente.

## 2. Stakeholders

| Rol | Descripción |
|---|---|
| Consumidor | Usa la app para pedir comida |
| Restaurante | Publica menús y gestiona pedidos |
| Repartidor | Acepta y completa entregas |
| CTO | Responsable de la decisión de migración |
| Product Manager | Define roadmap y prioridades |
| Equipo de Ingeniería | Implementa la migración |

## 3. Capacidades de Negocio

**CAP-01 — Consumer Management**
Registro, autenticación y gestión del perfil del consumidor. Incluye historial de pedidos y preferencias.

**CAP-02 — Restaurant Management**
Registro de restaurantes, aprobación, gestión de menús (alta/baja/modificación de items) y configuración de horarios.

**CAP-03 — Order Taking**
Recepción de pedidos: selección de restaurante, items y dirección. Validación de disponibilidad y confirmación.

**CAP-04 — Order Fulfillment**
Coordinación interna del ciclo de vida del pedido entre cocina, entrega y facturación.

**CAP-05 — Delivery Management**
Gestión de repartidores: asignación automática, seguimiento GPS y confirmación de entrega.

**CAP-06 — Billing & Accounting**
Procesamiento del pago al momento de confirmar el pedido y conciliación contable.

**CAP-07 — Notificaciones**
Notificaciones integradas en CAP-06 para confirmar el pago y en CAP-04 para actualizaciones de estado del pedido.

## 4. Requerimientos No Funcionales

**NFR-01 — Latencia de API**
El tiempo de respuesta de los endpoints principales no debe superar los 200ms en el percentil 95. [Brief §A.4 — Rendimiento]
*Justificación:* Estándar de la industria para apps de consumidor; superar 300ms correlaciona con abandono de sesión.

**NFR-02 — Alta Disponibilidad**
La plataforma debe garantizar alta disponibilidad, especialmente en horas de almuerzo y cena.

**NFR-03 — Escalabilidad Horizontal**
Los componentes críticos (pedidos, pagos) deben escalar horizontalmente ante picos de demanda.

**NFR-04 — Seguridad y Cumplimiento**
El procesamiento de pagos debe cumplir PCI-DSS. Los datos de usuario deben manejarse conforme a GDPR.

**NFR-05 — Observabilidad**
El sistema debe contar con logs centralizados y trazabilidad de requests entre servicios.

## 5. Alcance

**Dentro del alcance:**
- Microservicios de pedidos, cocina, entrega, facturación y notificaciones
- API Gateway como punto de entrada unificado
- Integraciones con Stripe y Google Maps
- Panel de administración para el equipo de FTGO

