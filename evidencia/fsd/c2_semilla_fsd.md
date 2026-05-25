# Especificación Funcional de Software — FTGO
<!-- Salida de Semilla B.2 · Corrida C2 · Evaluación: 2/7 (29 %) -->

## Tabla de Casos de Uso

| UC | Descripción | Actor |
|---|---|---|
| UC-01 | Realizar pedido | Consumidor |
| UC-02 | Gestionar ticket de cocina | Restaurante |
| UC-03 | Gestionar entrega | Courier |
| UC-04 | Cobrar pedido | Sistema |
| UC-05 | Ver estado del pedido | Consumidor |

---

## UC-01 — Realizar Pedido

**Actor principal:** Consumidor
**Precondiciones:** Usuario autenticado, restaurante disponible

**Flujo:**
1. El consumidor selecciona restaurante y productos
2. Confirma dirección de entrega
3. Selecciona método de pago
4. Confirma el pedido
5. El sistema genera el pedido y procesa el pago
6. Se notifica al consumidor con el número de pedido

**GWT:**
```
Given: el usuario está logueado
When: confirma el pedido con todos los datos completos
Then: el sistema genera un número de pedido y redirige al seguimiento
```

**Postcondiciones:** Pedido generado, pago procesado, restaurante notificado

---

## UC-02 — Gestionar Ticket de Cocina

**Actor principal:** Restaurante

**Flujo:**
1. El restaurante recibe el ticket en su pantalla de cocina
2. Acepta o rechaza el pedido
3. Marca el pedido como listo para recolección

**GWT:**
```
Given: el restaurante tiene un ticket pendiente
When: el chef selecciona "Aceptar pedido"
Then: se notifica al sistema que el pedido está en preparación
```

**Postcondiciones:** Estado del pedido actualizado

---

## UC-03 — Gestionar Entrega

**Actor principal:** Courier
**Origen:** US-03 [Brief §A.5]

**Precondiciones:** Pedido listo para recolección, courier disponible

**Flujo:**
1. El courier recibe la notificación de asignación
2. Se dirige al restaurante para recolectar el pedido
3. Confirma la recolección
4. Entrega el pedido en la dirección indicada
5. Confirma la entrega

**GWT:**
```
Given: hay un pedido en estado READY_FOR_PICKUP y un courier disponible
When: el sistema asigna el pedido y el courier acepta
Then: el pedido pasa a estado ASSIGNED y el courier inicia el trayecto al restaurante
```

**Postcondiciones:** Pedido en estado ASSIGNED, courier en camino

---

## UC-04 — Cobrar Pedido

**Actor:** Sistema

**Flujo:**
1. Se recibe solicitud de cobro con datos del método de pago
2. El sistema envía la transacción al procesador de pagos
3. El procesador responde con éxito o error
4. El sistema actualiza el estado

**GWT:**
```
Given: el consumidor ha proporcionado datos de pago válidos
When: el sistema solicita el cobro al procesador
Then: el procesador responde y el sistema registra el resultado
```

---

## UC-05 — Ver Estado del Pedido

**Actor:** Consumidor

**Flujo:**
1. El consumidor entra a la sección de "Mis Pedidos"
2. Selecciona el pedido activo
3. El sistema muestra el estado actual y la posición del courier si aplica

**GWT:**
```
Given: el consumidor tiene un pedido activo
When: navega a la pantalla de seguimiento
Then: el sistema muestra el estado y la ubicación del courier
```

