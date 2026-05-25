# Especificación Funcional — FTGO
<!-- Salida de Semilla B.2 · Corrida C1 · Evaluación: 3/7 (43 %) -->

## Resumen de Casos de Uso

| UC | Nombre | Actor |
|---|---|---|
| UC-01 | Tomar pedido | Consumidor |
| UC-02 | Aceptar o rechazar ticket de cocina | Restaurante |
| UC-03 | Asignar pedido a courier | Sistema / Delivery |
| UC-04 | Procesar pago | Sistema |
| UC-05 | Seguimiento en tiempo real | Consumidor |

---

## UC-01 — Tomar Pedido

**Actor:** Consumidor
**Capacidad:** Order Taking

**Precondiciones:**
- El usuario ha iniciado sesión en la aplicación
- El restaurante seleccionado está disponible

**Flujo Principal:**
1. El usuario selecciona un restaurante
2. El usuario agrega items al carrito
3. El usuario confirma el pedido e ingresa su dirección
4. El sistema valida la disponibilidad de los items
5. El sistema crea el pedido

**Flujos Alternativos:**
- FA-01: Si un item no está disponible, se muestra un mensaje de error

**Postcondiciones:**
- El pedido queda registrado en el sistema

**Escenarios GWT:**

```
Given: el usuario está autenticado y selecciona un restaurante disponible
When: el usuario confirma el pedido
Then: el sistema registra el pedido y notifica al restaurante
```

---

## UC-02 — Aceptar o Rechazar Ticket de Cocina

**Actor:** Restaurante
**Capacidad:** Order Fulfillment

**Precondiciones:**
- El restaurante ha recibido un nuevo ticket

**Flujo Principal:**
1. El restaurante recibe la notificación del nuevo pedido
2. El encargado de cocina revisa los items
3. El encargado acepta o rechaza el pedido
4. El sistema actualiza el estado del pedido

**Flujos Alternativos:**
- FA-01: Si el restaurante no responde en 10 minutos, el pedido se cancela automáticamente

**Postcondiciones:**
- El estado del pedido ha sido actualizado

**Escenarios GWT:**

```
Given: el restaurante ha recibido un ticket de pedido
When: el encargado selecciona "Aceptar"
Then: el sistema notifica al consumidor que su pedido está siendo preparado
```

---

## UC-03 — Asignar Pedido a Courier

**Actor:** Sistema (Delivery Service)
**Capacidad:** Delivery Management
**Origen:** US-03 [Brief §A.5]

**Precondiciones:**
- El pedido ha sido aceptado por el restaurante
- Hay couriers disponibles en el área

**Flujo Principal:**
1. El sistema evalúa los couriers disponibles
2. El sistema selecciona el courier más cercano al restaurante
3. El sistema notifica al courier con los detalles del pedido
4. El courier acepta la asignación

**Flujos Alternativos:**
- FA-01: Si no hay couriers disponibles, el sistema reintenta cada 2 minutos

**Postcondiciones:**
- El pedido queda en estado ASSIGNED con un courier asignado

**Escenarios GWT:**

```
Given: el pedido está en estado READY_FOR_PICKUP y hay couriers disponibles
When: el sistema ejecuta el algoritmo de asignación
Then: el pedido pasa a estado ASSIGNED y el courier recibe la notificación
```

---

## UC-04 — Procesar Pago

**Actor:** Sistema (Billing)
**Capacidad:** Billing & Accounting

**Precondiciones:**
- El pedido ha sido confirmado por el consumidor

**Flujo Principal:**
1. El sistema solicita autorización al procesador de pagos
2. El procesador responde con el resultado
3. El sistema actualiza el estado del pedido según el resultado

**Flujos Alternativos:**

**Postcondiciones:**
- El pago ha sido procesado

**Escenarios GWT:**

```
Given: el consumidor ha confirmado el pedido
When: el sistema envía la solicitud de cobro
Then: el sistema registra el resultado del pago
```

---

## UC-05 — Seguimiento en Tiempo Real

**Actor:** Consumidor
**Capacidad:** Delivery Management

**Precondiciones:**
- El pedido está en camino

**Flujo Principal:**
1. El consumidor abre la vista de seguimiento
2. El sistema muestra la posición del courier en el mapa
3. El sistema muestra el tiempo estimado de llegada

**Postcondiciones:**
- El consumidor ha visualizado la ubicación del courier

**Escenarios GWT:**

```
Given: el pedido está asignado a un courier
When: el consumidor abre la pantalla de seguimiento
Then: el sistema muestra la posición en tiempo real y el ETA
```

