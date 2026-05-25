# Especificación Funcional — FTGO Platform
<!-- Salida de Semilla B.2 · Corrida C3 · Evaluación: 4/7 (57 %) -->

## Tabla de Casos de Uso

| UC | Nombre | Actor | Capacidad PRD |
|---|---|---|---|
| UC-01 | Tomar pedido | Consumidor | CAP-03 Order Taking |
| UC-02 | Aceptar/rechazar ticket de cocina | Restaurante | CAP-04 Order Fulfillment |
| UC-03 | Asignar pedido a courier | Sistema | CAP-05 Delivery |
| UC-04 | Procesar pago | Sistema | CAP-06 Billing |
| UC-05 | Seguimiento en tiempo real | Consumidor | CAP-05 Delivery |

---

## UC-01 — Tomar Pedido

**Actor:** Consumidor
**Capacidad:** CAP-03 Order Taking
**Origen:** US-01 [Brief §A.5]

**Precondiciones:**
- El consumidor está autenticado en la aplicación
- El restaurante seleccionado está operativo en el horario actual

**Flujo Principal:**
1. El consumidor selecciona un restaurante disponible
2. El consumidor navega el menú y agrega items al carrito
3. El consumidor ingresa la dirección de entrega (o usa la guardada)
4. El consumidor selecciona el método de pago
5. El consumidor confirma el pedido
6. El sistema valida la disponibilidad de los items con el restaurante
7. El sistema crea el pedido y retorna el identificador ORD-{uuid}
8. El sistema dispara el flujo de facturación

**Flujos Alternativos:**
- FA-01: Si un item no está disponible, el sistema informa al consumidor y sugiere alternativas
- FA-02: Si el restaurante cierra mientras se arma el carrito, el sistema cancela la operación

**Postcondiciones:**
- El pedido queda registrado en estado PENDING_PAYMENT

**Escenarios GWT:**

```
Scenario 1 — Happy path
  Given: un consumidor autenticado con dirección "Av. Principal 123" y tarjeta Visa registrada
  When: selecciona "Hamburguesa Clásica" (item activo) y confirma el pedido
  Then: el sistema crea el pedido ORD-0042 en estado PENDING_PAYMENT
   And: el consumidor recibe confirmación con el número de pedido

Scenario 2 — Item no disponible
  Given: el consumidor tiene "Pizza Especial" en su carrito
  When: el restaurante desactiva "Pizza Especial" antes de la confirmación
  Then: el sistema muestra el error "Este item ya no está disponible"
   And: el carrito se actualiza eliminando el item
```

---

## UC-02 — Aceptar o Rechazar Ticket de Cocina

**Actor:** Restaurante
**Capacidad:** CAP-04 Order Fulfillment
**Origen:** US-02 [Brief §A.5]

**Precondiciones:**
- El restaurante ha recibido un ticket de pedido

**Flujo Principal:**
1. El restaurante recibe notificación del nuevo ticket
2. El encargado revisa los items del pedido
3. El encargado selecciona "Aceptar" o "Rechazar"
4. El sistema actualiza el estado del pedido a ACCEPTED o REJECTED

**Flujos Alternativos:**
- FA-01: Si el restaurante no responde en 10 minutos, el sistema cancela el pedido automáticamente

**Postcondiciones:**
- El pedido queda en estado ACCEPTED o REJECTED

**Escenarios GWT:**

```
Scenario 1 — Aceptación
  Given: el restaurante tiene un ticket ORD-0042 en estado PENDING_RESTAURANT_APPROVAL
  When: el encargado selecciona "Aceptar"
  Then: el pedido pasa a estado ACCEPTED
   And: el sistema notifica al consumidor y al Delivery Service

Scenario 2 — Rechazo
  Given: el restaurante no puede preparar el pedido por falta de ingredientes
  When: el encargado selecciona "Rechazar" con motivo "Sin stock"
  Then: el pedido pasa a estado REJECTED
   And: el sistema inicia el reembolso automático
```

---

## UC-03 — Asignar Pedido a Courier

**Actor:** Sistema (Delivery Service)
**Capacidad:** CAP-05 Delivery Management
**Origen:** US-03 [Brief §A.5]

**Precondiciones:**
- El pedido está en estado ACCEPTED
- Hay couriers disponibles en la zona

**Flujo Principal:**
1. El sistema evalúa couriers disponibles ordenados por proximidad al restaurante
2. El sistema envía la solicitud de asignación al courier más cercano
3. El courier acepta la asignación
4. El sistema actualiza el pedido a ASSIGNED

**Flujos Alternativos:**
- FA-01: Si el courier rechaza, el sistema ofrece al siguiente disponible
- FA-02: Si no hay couriers en 5 minutos, el sistema escala la alerta al equipo de operaciones

**Postcondiciones:**
- El pedido queda en estado ASSIGNED con courier asignado

**Escenarios GWT:**

```
Scenario 1 — Asignación exitosa
  Given: el pedido ORD-0042 está en estado ACCEPTED y el courier COU-007 está a 500m
  When: el sistema ejecuta el algoritmo de asignación
  Then: el pedido pasa a estado ASSIGNED con COU-007 asignado
   And: el courier recibe la notificación con dirección del restaurante

Scenario 2 — Sin couriers disponibles
  Given: no hay couriers disponibles en un radio de 3km
  When: el sistema intenta asignar por 5 minutos sin éxito
  Then: el sistema notifica al equipo de operaciones
   And: el estado del pedido permanece en ACCEPTED con alerta activa
```

---

## UC-04 — Procesar Pago del Pedido

**Actor:** Sistema (Billing)
**Capacidad:** CAP-06 Billing

**Precondiciones:**
- El pedido está en estado PENDING_PAYMENT
- El consumidor tiene un método de pago registrado

**Flujo Principal:**
1. El Billing Service recibe el evento de nuevo pedido
2. El servicio solicita la autorización de pago a Stripe
3. Stripe responde con APPROVED o DECLINED
4. El Billing Service publica el evento `PaymentProcessed` o `PaymentFailed`

**Flujos Alternativos:**
- FA-01: Si Stripe devuelve DECLINED, el sistema notifica al consumidor y cancela el pedido

**Postcondiciones:**
- El pago queda en estado APPROVED o DECLINED

**Escenarios GWT:**

```
Scenario 1 — Pago exitoso
  Given: el pedido ORD-0042 está en estado PENDING_PAYMENT con tarjeta Visa válida
  When: Stripe responde con APPROVED para el cargo de $25.50
  Then: el Billing Service registra el pago y publica PaymentProcessed
   And: el pedido avanza en el flujo de confirmación
```

---

## UC-05 — Seguimiento en Tiempo Real

**Actor:** Consumidor

**Flujo:**
1. El consumidor abre la app y accede al pedido activo
2. La app consulta la posición del courier
3. La app muestra la ubicación en el mapa y el ETA

**GWT:**
```
Given: el pedido tiene courier asignado
When: el consumidor abre la pantalla de seguimiento
Then: el sistema muestra la posición actual del courier y el tiempo estimado de entrega
```

