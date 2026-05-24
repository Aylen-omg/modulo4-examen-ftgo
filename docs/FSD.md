# FSD Ligero — FTGO (Food To Go)

| Campo        | Valor                                                                   |
|--------------|-------------------------------------------------------------------------|
| ID           | PR-FSD-FTGO-001                                                         |
| Versión      | v0.2-mejorado                                                           |
| Artefacto    | Functional Specification Document (ligero)                              |
| Fecha        | 2026-05-24                                                              |
| Autor        | Analista funcional — Equipo de arquitectura FTGO                        |
| Estado       | Aprobado para derivar ADRs y diagramas C4                               |
| Fuente prim. | docs/PRD.md v0.2-mejorado                                               |
| Fuente sec.  | Brief Anexo A §A.5 · Richardson "Microservices Patterns" Manning 2019   |

---

## 1. Introducción

Este FSD (Functional Specification Document) formaliza **exactamente 7 casos de uso** que implementan las capacidades de negocio declaradas en `docs/PRD.md`, conforme al alcance del FSD ligero [Brief §A.6]. Los UC-01 a UC-03 derivan de las 3 user stories semilla [Brief §A.5]; los UC-04 a UC-07 derivan de capacidades del PRD y de los UCs adicionales señalados como derivables en [Brief §A.5]. Las siete capacidades del PRD (CAP-01 a CAP-07) quedan cubiertas mediante UCs dedicados, precondiciones compartidas o la matriz transversal de notificaciones (§2.2). Ningún UC ha sido inventado fuera de estas fuentes [Brief §A.6].

---

## 2. Tabla Resumen de Casos de Uso

| ID    | Título                                  | Actor primario   | Capacidad PRD                   | Origen                                                      |
|-------|-----------------------------------------|------------------|---------------------------------|-------------------------------------------------------------|
| UC-01 | Tomar pedido                            | Consumidor       | CAP-03 — Order Taking           | US-01 [Brief §A.5]                                          |
| UC-02 | Aceptar o rechazar ticket de cocina     | Restaurante      | CAP-04 — Order Fulfillment/Kitchen | US-02 [Brief §A.5]                                       |
| UC-03 | Asignar pedido a courier                | Courier          | CAP-05 — Delivery               | US-03 [Brief §A.5]                                          |
| UC-04 | Procesar pago del pedido                | Sistema          | CAP-06 — Billing & Accounting   | Derivado de CAP-06 [PRD §3] + [Richardson Cap. 3]           |
| UC-05 | Tracking en tiempo real del consumidor  | Consumidor       | CAP-05 — Delivery               | Derivado de NFR-02 [PRD §4] + CAP-05 [PRD §3]              |
| UC-06 | Cancelar pedido por el consumidor       | Consumidor       | CAP-03 — Order Taking           | Derivado de UCs adicionales derivables [Brief §A.5]         |
| UC-07 | Gestionar menú del restaurante          | Restaurante      | CAP-02 — Restaurant Management  | Derivado de UCs adicionales derivables [Brief §A.5] + CAP-02 [PRD §3] |

### 2.1 Matriz de cobertura CAP ↔ UC

Trazabilidad explícita de las 7 capacidades del PRD [PRD §3] frente a los 7 UCs de este FSD ligero [Brief §A.6]:

| CAP PRD | UC(s) | Modo de cobertura | Evidencia |
|---------|-------|-------------------|-----------|
| CAP-01 Consumer Management | UC-01 (precondiciones) | Infraestructura compartida: sesión autenticada, direcciones y métodos de pago registrados antes de tomar pedido | UC-01 precondiciones §3; rutas auth en monolito vía API Gateway [C4 L2 `Rel(gateway, monolito)`] |
| CAP-02 Restaurant Management | UC-07 | UC dedicado — gestión de menú (alta, edición, desactivación) | UC-07 §3; tickets de cocina vía UC-02 |
| CAP-03 Order Taking | UC-01, UC-06 | UC dedicados — tomar y cancelar pedido | US-01 [Brief §A.5]; UC-06 [Brief §A.5] |
| CAP-04 Order Fulfillment / Kitchen | UC-02 | UC dedicado — aceptar/rechazar ticket | US-02 [Brief §A.5] |
| CAP-05 Delivery | UC-03, UC-05 | UC dedicados — asignación courier y tracking | US-03 [Brief §A.5]; NFR-02 [PRD §4] |
| CAP-06 Billing & Accounting | UC-04 | UC dedicado — cobro async vía Stripe | CAP-06 [PRD §3]; Richardson Cap. 3 |
| CAP-07 Notifications | UC-01…UC-06 (transversal) | Side-effects documentados — ver matriz §2.2 | CAP-07 [PRD §3]; `Notification Svc` [C4 L2] |

### 2.2 Matriz transversal — CAP-07 Notifications

CAP-07 es capacidad **satélite** [PRD §3]: su degradación no bloquea el flujo principal. Los eventos que disparan notificaciones (SendGrid/Twilio/push) quedan trazados por UC:

| Evento / estado | UC origen | Destinatario | Canal | Paso FSD |
|-----------------|-----------|--------------|-------|----------|
| Pedido creado (`PENDING_PAYMENT`) | UC-01 | Consumidor | Push / email | Flujo principal paso 9 |
| Ticket pendiente | UC-02 | Restaurante | Push dashboard | Flujo principal paso 1 |
| Pedido aceptado / rechazado | UC-02 | Consumidor | Push / SMS | Flujo principal paso 7; FA-01 |
| Courier asignado | UC-03 | Consumidor | Push | Flujo principal paso 8 → trigger UC-05 |
| Pago rechazado | UC-04 | Consumidor | Push / email | FA-01 |
| Cancelación confirmada | UC-06 | Consumidor | Push / email | Flujo principal paso 8 |
| Cambio estado entrega (`PICKED_UP`, `DELIVERED`) | UC-05 | Consumidor | Push | FA-03 |

**Given/When/Then representativo (CAP-07):**

- **Given:** el pedido acaba de pasar a estado `APPROVED` tras UC-04 y el consumidor tiene notificaciones push activas.
- **When:** el sistema publica el evento `PaymentConfirmed` consumido por CAP-07.
- **Then:** el consumidor recibe en ≤ 5 s la notificación "Pago confirmado — tu pedido está en preparación" vía SendGrid/Twilio [NFR-01 UX percibida].

---

## 3. Detalle de Casos de Uso

---

### UC-01: Tomar pedido

| Campo          | Valor                                      |
|----------------|--------------------------------------------|
| Actor primario | Consumidor                                 |
| Capacidad PRD  | CAP-03 — Order Taking [PRD §3]             |
| Origen         | US-01 [Brief §A.5]                         |

**Precondiciones:**
- El Consumidor tiene una sesión autenticada válida (CAP-01).
- El restaurante seleccionado está activo y dentro de su horario de atención (CAP-02).
- El carrito contiene al menos 1 ítem con precio vigente.
- El Consumidor tiene al menos un método de pago registrado o introduce uno nuevo válido.

**Flujo principal:**
1. El Consumidor navega al menú del restaurante seleccionado.
2. El Consumidor agrega uno o más ítems al carrito y confirma cantidades.
3. El Consumidor accede al resumen del carrito y verifica la dirección de entrega.
4. El Consumidor selecciona el método de pago y pulsa "Confirmar pedido".
5. El sistema valida la disponibilidad del restaurante y la existencia y precio de cada ítem (CAP-02).
6. El sistema calcula el total: subtotal + cargo de entrega + impuestos aplicables.
7. El sistema crea el pedido en estado `PENDING_PAYMENT` con un identificador único.
8. El sistema publica el evento `OrderCreated` para que CAP-06 inicie el cobro (trigger UC-04).
9. El sistema notifica al Consumidor con el número de pedido y el estado `PENDING_PAYMENT` (CAP-07).

**Flujos alternativos:**
- FA-01: El restaurante no está disponible al confirmar → el sistema devuelve error `RESTAURANT_UNAVAILABLE`, el carrito se preserva y se informa al Consumidor para elegir otro restaurante.
- FA-02: Uno o más ítems no están disponibles → el sistema devuelve error `ITEM_UNAVAILABLE` listando los ítems afectados; el Consumidor puede eliminarlos y reintentar.
- FA-03: El método de pago es rechazado → el pedido permanece en `PENDING_PAYMENT`; el sistema notifica al Consumidor para actualizar su método de pago (ver UC-04 FA-01).

**Postcondiciones:**
- El pedido existe en el sistema con estado `PENDING_PAYMENT` e identificador único inmutable.
- El evento `OrderCreated` ha sido publicado para activar UC-04 y UC-02.
- El Consumidor ha recibido confirmación con el número de pedido.

**Given/When/Then:**
- **Given:** el Consumidor autenticado tiene al menos 1 ítem en el carrito de un restaurante activo y una tarjeta de crédito válida registrada.
- **When:** el Consumidor presiona "Confirmar pedido".
- **Then:** el sistema crea el pedido con estado `PENDING_PAYMENT`, asigna un número único (ej. `ORD-20260524-0042`) y devuelve confirmación en ≤ 200 ms p95 [NFR-01, PRD §4].

---

### UC-02: Aceptar o rechazar ticket de cocina

| Campo          | Valor                                               |
|----------------|-----------------------------------------------------|
| Actor primario | Restaurante                                         |
| Capacidad PRD  | CAP-04 — Order Fulfillment / Kitchen [PRD §3]       |
| Origen         | US-02 [Brief §A.5]                                  |

**Precondiciones:**
- El pedido existe en estado `APPROVED` (pago confirmado, UC-04 completado).
- El sistema ha creado un ticket de cocina en estado `PENDING_ACCEPTANCE` visible en el dashboard del Restaurante.
- El Restaurante tiene una sesión activa en su dashboard.

**Flujo principal:**
1. El Restaurante recibe notificación push de nuevo ticket en su dashboard (CAP-07).
2. El Restaurante visualiza los detalles del ticket: ítems, cantidades y hora de solicitud.
3. El Restaurante ingresa el tiempo estimado de preparación en minutos.
4. El Restaurante pulsa "Aceptar ticket".
5. El sistema actualiza el ticket al estado `ACCEPTED` con el tiempo estimado registrado.
6. El sistema actualiza el pedido al estado `PREPARING`.
7. El sistema notifica al Consumidor que el pedido ha sido aceptado con el tiempo estimado (CAP-07).

**Flujos alternativos:**
- FA-01: El Restaurante rechaza el ticket con motivo → el sistema actualiza el pedido a `CANCELLED`, inicia la reversión del cobro en CAP-06 y notifica al Consumidor con el motivo del rechazo (CAP-07).
- FA-02: El Restaurante no responde en el timeout configurado (15 minutos) → el sistema escala al Empleado FTGO (back office) y notifica al Consumidor del retraso.

**Postcondiciones:**
- El ticket de cocina está en estado `ACCEPTED` con tiempo estimado registrado, o en `REJECTED` con motivo.
- El pedido refleja el estado correspondiente: `PREPARING` o `CANCELLED`.
- El Consumidor ha recibido notificación del resultado vía CAP-07.

**Given/When/Then:**
- **Given:** existe un ticket de cocina en estado `PENDING_ACCEPTANCE` para el restaurante, con pago previamente confirmado (`APPROVED`), visible en el dashboard del Restaurante.
- **When:** el Restaurante pulsa "Aceptar ticket" e ingresa un tiempo estimado de preparación de 25 minutos.
- **Then:** el ticket pasa a estado `ACCEPTED`, el pedido pasa a `PREPARING`, y el Consumidor recibe en ≤ 5 segundos la notificación "Tu pedido fue aceptado. Tiempo estimado: 25 minutos."

---

### UC-03: Asignar pedido a courier

| Campo          | Valor                                  |
|----------------|----------------------------------------|
| Actor primario | Courier                                |
| Capacidad PRD  | CAP-05 — Delivery [PRD §3]             |
| Origen         | US-03 [Brief §A.5]                     |

**Precondiciones:**
- El ticket de cocina está en estado `ACCEPTED` (UC-02 completado) y el pedido en `PREPARING`.
- Al menos un Courier está disponible (ha marcado disponibilidad en la app) dentro del radio de asignación del restaurante.
- El Courier tiene la app activa con posición GPS actualizada.

**Flujo principal:**
1. El sistema detecta que el ticket de cocina está en `ACCEPTED` e inicia la búsqueda de courier (CAP-05).
2. El sistema consulta la posición de los couriers disponibles y selecciona el más cercano al restaurante (integración Google Maps).
3. El sistema envía al Courier una oferta de asignación con: restaurante, dirección de entrega y valor de la entrega.
4. El Courier recibe la oferta con un contador de **30 segundos** para responder [Brief §A.5 — US-03].
5. El Courier pulsa "Aceptar entrega" dentro del timeout.
6. El sistema asigna el pedido al Courier y actualiza el pedido a estado `ASSIGNED`.
7. El sistema calcula y presenta la ruta optimizada al Courier: primero al restaurante, luego al Consumidor.
8. El sistema notifica al Consumidor que tiene courier asignado e inicia el tracking (trigger UC-05).

**Flujos alternativos:**
- FA-01: El Courier rechaza la oferta o no responde en 30 s → el sistema descarta la oferta y pasa al siguiente courier disponible; tras 3 intentos fallidos, escala al Empleado FTGO (back office).
- FA-02: Google Maps no está disponible → el sistema usa estimación estática de ruta por distancia lineal; la asignación continúa sin bloqueo [NFR-04, PRD §4].

**Postcondiciones:**
- El pedido está en estado `ASSIGNED` con la identidad del Courier registrada.
- El Courier tiene la ruta activa en su app.
- El Consumidor ha recibido notificación de asignación y el tracking está activo (UC-05).

**Given/When/Then:**
- **Given:** el ticket está en estado `ACCEPTED`, hay al menos 1 Courier disponible a ≤ 2 km del restaurante con GPS activo, y el pedido está en `PREPARING`.
- **When:** el sistema envía la oferta y el Courier pulsa "Aceptar entrega" dentro de los 30 segundos.
- **Then:** el pedido cambia a estado `ASSIGNED`, el Courier recibe la ruta optimizada (restaurante → consumidor) y el Consumidor recibe notificación "Tu courier está en camino" con ETA calculado.

---

### UC-04: Procesar pago del pedido

| Campo          | Valor                                                              |
|----------------|--------------------------------------------------------------------|
| Actor primario | Sistema (CAP-06 — Billing & Accounting, proceso automático)        |
| Capacidad PRD  | CAP-06 — Billing & Accounting [PRD §3]                             |
| Origen         | Derivado de CAP-06 [PRD §3] + [Richardson Cap. 3 — Saga pattern]  |

**Precondiciones:**
- El pedido existe en estado `PENDING_PAYMENT` con monto total calculado por CAP-03.
- El evento `OrderCreated` ha sido publicado por UC-01 (paso 8).
- Stripe está disponible o la cola de retry está activa [NFR-04, PRD §4].

**Flujo principal:**
1. CAP-06 recibe el evento `OrderCreated` con el monto total del pedido.
2. El sistema construye la solicitud de cobro: monto, moneda y token de pago del Consumidor.
3. El sistema invoca la API de Stripe para autorizar y capturar el pago.
4. Stripe devuelve confirmación con un `charge_id`.
5. El sistema registra el cobro con el `charge_id` y actualiza el pedido a estado `APPROVED`.
6. El sistema publica el evento `PaymentConfirmed` para que CAP-04 cree el ticket de cocina (trigger UC-02).
7. El sistema calcula la comisión de FTGO y registra la cuenta por pagar al restaurante.

**Flujos alternativos:**
- FA-01: Stripe rechaza el pago (fondos insuficientes, datos inválidos) → el sistema mantiene el pedido en `PENDING_PAYMENT` y notifica al Consumidor para actualizar su método de pago (CAP-07).
- FA-02: Stripe no responde (timeout) → el sistema encola el cobro con backoff exponencial: reintento 1 a los 30 s, reintento 2 a los 60 s, reintento 3 a los 120 s [NFR-04, PRD §4]; tras 3 fallos el pedido pasa a `CANCELLED`.
- FA-03: El pedido fue cancelado antes de confirmar el pago → el sistema cancela o revierte la solicitud en Stripe y registra el estado `CANCELLED`.

**Postcondiciones:**
- El pedido está en estado `APPROVED` con `charge_id` de Stripe registrado, o en `CANCELLED` con motivo de fallo.
- El evento `PaymentConfirmed` ha sido publicado para activar el flujo de cocina (UC-02).
- La comisión de FTGO y la cuenta por pagar al restaurante están registradas en Billing.

**Given/When/Then:**
- **Given:** el pedido está en estado `PENDING_PAYMENT` con monto calculado, y Stripe está disponible con el token de pago del Consumidor válido.
- **When:** CAP-06 recibe el evento `OrderCreated` e invoca la API de Stripe.
- **Then:** Stripe confirma el cobro, el pedido pasa a estado `APPROVED` con el `charge_id` registrado, y el evento `PaymentConfirmed` es publicado en ≤ 3 segundos desde la recepción de `OrderCreated`.

---

### UC-05: Tracking en tiempo real del consumidor

| Campo          | Valor                                                          |
|----------------|----------------------------------------------------------------|
| Actor primario | Consumidor                                                     |
| Capacidad PRD  | CAP-05 — Delivery [PRD §3]                                     |
| Origen         | Derivado de NFR-02 Latencia UX [PRD §4] + CAP-05 [PRD §3]     |

**Precondiciones:**
- El pedido está en estado `ASSIGNED` (UC-03 completado).
- El Courier tiene la app activa con GPS habilitado y posición actualizada.
- El Consumidor tiene la app abierta o en background con notificaciones activas.

**Flujo principal:**
1. El Consumidor abre la pantalla de seguimiento desde su historial de pedidos activos.
2. El sistema obtiene la posición GPS actual del Courier desde CAP-05.
3. El sistema renderiza el mapa (integración Google Maps) con: posición del Courier, restaurante y dirección del Consumidor.
4. La posición del Courier se actualiza en el mapa del Consumidor cada ≤ 5 segundos.
5. El sistema muestra el ETA actualizado en tiempo real.
6. El Courier llega al restaurante → el sistema actualiza el pedido a `PICKED_UP` y notifica al Consumidor.
7. El Courier completa la entrega → el sistema actualiza el pedido a `DELIVERED` y el tracking finaliza.

**Flujos alternativos:**
- FA-01: El GPS del Courier pierde señal (> 30 s sin actualización) → el sistema muestra la última posición conocida con indicador visual "señal perdida" y mantiene el ETA con la última posición registrada; el tracking no se interrumpe.
- FA-02: Google Maps no está disponible → el sistema muestra el estado textual del pedido (`ASSIGNED`, `PICKED_UP`, `DELIVERED`) y el ETA estimado sin mapa visual [NFR-04, PRD §4].
- FA-03: El Consumidor cierra la app → el sistema envía notificaciones push en cada cambio de estado clave: `PICKED_UP` y `DELIVERED`.

**Postcondiciones:**
- El Consumidor pudo ver la posición del Courier actualizada con frecuencia ≤ 5 segundos durante la entrega.
- El pedido ha progresado a estado `DELIVERED` al completar la entrega.
- CAP-07 ha enviado la notificación de entrega completada al Consumidor.

**Given/When/Then:**
- **Given:** el pedido está en estado `ASSIGNED`, el Courier tiene GPS activo y se encuentra en ruta al restaurante, y el Consumidor abre la pantalla de tracking.
- **When:** el Consumidor visualiza la pantalla de seguimiento en la app.
- **Then:** el mapa muestra la posición del Courier actualizada en ≤ 5 segundos, el ETA se calcula con precisión ±2 minutos, y los cambios de estado (`PICKED_UP`, `DELIVERED`) se reflejan en tiempo real sin recargar la pantalla.

---

### UC-06: Cancelar pedido por el consumidor

| Campo          | Valor                                                               |
|----------------|---------------------------------------------------------------------|
| Actor primario | Consumidor                                                          |
| Capacidad PRD  | CAP-03 — Order Taking [PRD §3]                                      |
| Origen         | Derivado de UCs adicionales derivables [Brief §A.5]                 |

**Precondiciones:**
- El Consumidor tiene una sesión autenticada válida.
- El pedido existe en un estado cancelable: `PENDING_PAYMENT` o `APPROVED` (antes de que el restaurante lo acepte).
- El Consumidor accede a la pantalla de detalle del pedido activo.

**Flujo principal:**
1. El Consumidor accede al detalle del pedido activo en la app.
2. El Consumidor selecciona la opción "Cancelar pedido".
3. El sistema verifica que el pedido está en un estado cancelable (`PENDING_PAYMENT` o `APPROVED`).
4. El sistema solicita confirmación al Consumidor con el motivo opcional de cancelación.
5. El Consumidor confirma la cancelación.
6. El sistema actualiza el pedido a estado `CANCELLED`.
7. Si el pago ya fue capturado (`APPROVED`), el sistema inicia la reversión del cobro en CAP-06 (reembolso vía Stripe).
8. El sistema notifica al Consumidor la confirmación de cancelación y el estado del reembolso (CAP-07).

**Flujos alternativos:**
- FA-01: El pedido ya está en estado `PREPARING` o posterior → el sistema informa al Consumidor que la cancelación ya no es posible de forma automática y sugiere contactar al soporte (Empleado FTGO back office).
- FA-02: La reversión del cobro en Stripe falla → el sistema registra el fallo y escala al Empleado FTGO para procesamiento manual del reembolso; el pedido queda en `CANCELLED` pendiente de reembolso.

**Postcondiciones:**
- El pedido está en estado `CANCELLED`.
- Si aplica, el reembolso ha sido iniciado en Stripe o escalado al back office.
- El Consumidor ha recibido confirmación de cancelación vía CAP-07.

**Given/When/Then:**
- **Given:** el Consumidor autenticado tiene un pedido en estado `APPROVED` con pago confirmado, y el restaurante aún no ha aceptado el ticket.
- **When:** el Consumidor selecciona "Cancelar pedido" y confirma la acción.
- **Then:** el pedido pasa a estado `CANCELLED`, se inicia el reembolso vía Stripe, y el Consumidor recibe notificación "Tu pedido ha sido cancelado. El reembolso se procesará en 3-5 días hábiles."

---

### UC-07: Gestionar menú del restaurante

| Campo          | Valor                                                                             |
|----------------|-----------------------------------------------------------------------------------|
| Actor primario | Restaurante                                                                       |
| Capacidad PRD  | CAP-02 — Restaurant Management [PRD §3]                                           |
| Origen         | Derivado de UCs adicionales derivables [Brief §A.5] + CAP-02 [PRD §3]            |

**Precondiciones:**
- El Restaurante tiene una sesión autenticada activa en su dashboard.
- El perfil del restaurante está registrado y activo en el sistema (CAP-02).

**Flujo principal:**
1. El Restaurante accede a la sección "Gestión de menú" en su dashboard.
2. El Restaurante selecciona la acción a realizar: agregar ítem, editar ítem o desactivar ítem.
3. Para **agregar ítem**: el Restaurante ingresa nombre, descripción, precio y categoría del nuevo ítem.
4. El sistema valida que el precio es mayor a cero y que el nombre no está duplicado en el menú del restaurante.
5. El sistema registra el nuevo ítem con estado `AVAILABLE` y lo publica en el catálogo (CAP-02).
6. El cambio es inmediatamente visible para los consumidores que naveguen el menú del restaurante.

**Flujos alternativos:**
- FA-01: El Restaurante edita el precio de un ítem existente → el sistema actualiza el precio y lo refleja en tiempo real en el catálogo; los pedidos ya creados con el precio anterior no se ven afectados.
- FA-02: El Restaurante desactiva un ítem (ej. ítem agotado) → el sistema cambia el estado del ítem a `UNAVAILABLE`; el ítem deja de mostrarse a consumidores en nuevas búsquedas pero no afecta pedidos ya confirmados.
- FA-03: El sistema detecta que el precio ingresado es ≤ 0 → devuelve error de validación `INVALID_PRICE` y el ítem no se registra; el Restaurante debe corregir el valor.

**Postcondiciones:**
- El catálogo del restaurante en CAP-02 refleja los cambios realizados (ítem agregado, editado o desactivado).
- Los cambios son visibles de inmediato para los consumidores que naveguen el menú.
- El historial de pedidos previos no se ve afectado por los cambios de precio o disponibilidad.

**Given/When/Then:**
- **Given:** el Restaurante autenticado accede a la sección de gestión de menú y el restaurante está en estado activo en el sistema.
- **When:** el Restaurante agrega un nuevo ítem "Hamburguesa Clásica" con precio $8.50 y categoría "Principal", y pulsa "Guardar".
- **Then:** el ítem queda registrado con estado `AVAILABLE` en el catálogo del restaurante, es visible de inmediato para los consumidores que accedan al menú, y el Restaurante recibe confirmación "Ítem agregado exitosamente."

---

*Trazabilidad: los 7 UCs cubren las 7 CAPs del PRD mediante UCs dedicados (§2), precondiciones compartidas (CAP-01) o matriz transversal (CAP-07, §2.2). Fuentes: US semilla [Brief §A.5], capacidades [PRD §3], NFRs [PRD §4], Richardson donde aplica. Ningún UC inventado fuera del brief [Brief §A.6].*
