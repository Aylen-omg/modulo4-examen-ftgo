# FSD Ligero — FTGO (Food To Go)

| Campo       | Valor                                                              |
|-------------|---------------------------------------------------------------------|
| ID          | PR-FSD-FTGO-001                                                    |
| Versión     | v0.2-mejorado                                                      |
| Artefacto   | Functional Specification Document (ligero)                         |
| Fecha       | 2026-05-24                                                         |
| Autor       | Analista funcional — Equipo de arquitectura FTGO                   |
| Estado      | Aprobado para derivar ADRs y diagramas C4                          |
| Fuente prim.| docs/PRD.md v0.2-mejorado                                          |
| Fuente sec. | Brief Anexo A §A.5 (US semilla) · Richardson "Microservices Patterns" Manning 2019 |

---

## 1. Introducción

Este FSD (Functional Specification Document) formaliza los casos de uso que implementan las capacidades de negocio declaradas en `docs/PRD.md`. Su propósito es traducir los requisitos de negocio del PRD en comportamientos observables y verificables, expresados en formato BDD (Given/When/Then), que sirvan de entrada para los ADRs arquitectónicos y los diagramas C4. El alcance cubre exactamente los 5 UCs derivables de las 3 user stories semilla del brief [Brief §A.5] y de las capacidades del PRD: UC-01 a UC-03 derivan directamente de las US semilla; UC-04 y UC-05 se derivan de capacidades del PRD y del libro de Richardson. Ningún UC ha sido inventado fuera de estas fuentes [Brief §A.6].

---

## 2. Tabla Resumen de Casos de Uso

| ID    | Título                                    | Actor primario          | Capacidad PRD               | Origen                                          |
|-------|-------------------------------------------|-------------------------|-----------------------------|--------------------------------------------------|
| UC-01 | Tomar pedido                              | Consumidor              | CAP-03 — Order Taking       | US-01 [Brief §A.5]                               |
| UC-02 | Aceptar o rechazar ticket de cocina       | Restaurante             | CAP-04 — Order Fulfillment  | US-02 [Brief §A.5]                               |
| UC-03 | Asignar pedido a courier                  | Courier                 | CAP-05 — Delivery           | US-03 [Brief §A.5]                               |
| UC-04 | Procesar pago del pedido                  | Sistema (Billing)       | CAP-06 — Billing & Accounting | Derivado de CAP-06 [PRD §3] + [Richardson Cap. 3] |
| UC-05 | Tracking en tiempo real del consumidor    | Consumidor              | CAP-05 — Delivery           | Derivado de NFR-02 [PRD §4] + CAP-05 [PRD §3]   |

---

## 3. Detalle de Casos de Uso

---

### UC-01: Tomar pedido

| Campo          | Valor                                                                 |
|----------------|-----------------------------------------------------------------------|
| Actor primario | Consumidor                                                            |
| Capacidad PRD  | CAP-03 — Order Taking [PRD §3]                                        |
| Origen         | US-01 [Brief §A.5]                                                    |

**Precondiciones:**
- El Consumidor tiene una sesión autenticada válida (CAP-01 — Consumer Management).
- El restaurante seleccionado está activo y dentro del horario de atención (CAP-02 — Restaurant Management).
- El carrito contiene al menos 1 ítem con precio vigente.
- El Consumidor tiene al menos 1 método de pago registrado o introduce uno nuevo.

**Flujo principal:**
1. El Consumidor navega al menú del restaurante seleccionado.
2. El Consumidor agrega uno o más ítems al carrito.
3. El Consumidor accede al resumen del carrito y confirma la dirección de entrega.
4. El Consumidor selecciona el método de pago y confirma el pedido.
5. El sistema valida la disponibilidad del restaurante y la existencia/precio de cada ítem (CAP-02).
6. El sistema calcula el total del pedido (subtotal + cargo de entrega + impuestos aplicables).
7. El sistema crea el pedido en estado `PENDING_PAYMENT` y le asigna un identificador único.
8. El sistema emite el evento `OrderCreated` para que CAP-06 inicie el procesamiento del pago (ver UC-04).
9. El sistema notifica al Consumidor con el número de pedido y el estado inicial `PENDING_PAYMENT` (CAP-07).

**Flujos alternativos:**
- FA-01: El restaurante no está disponible en el momento de la confirmación → el sistema devuelve error `RESTAURANT_UNAVAILABLE`, el carrito se preserva y se informa al Consumidor para que elija otro restaurante.
- FA-02: Uno o más ítems del carrito ya no están disponibles → el sistema devuelve error `ITEM_UNAVAILABLE` listando los ítems afectados; el Consumidor puede eliminarlos y reintentar.
- FA-03: El método de pago es rechazado por Stripe → el pedido permanece en `PENDING_PAYMENT`; el sistema reintenta hasta 3 veces con backoff exponencial [NFR-04, PRD §4]; si agota reintentos notifica al Consumidor para actualizar el método.

**Postcondiciones:**
- El pedido existe en el sistema con estado `PENDING_PAYMENT` y un identificador único inmutable.
- El evento `OrderCreated` ha sido publicado para su procesamiento por CAP-06 (Billing) y CAP-04 (Kitchen).
- El Consumidor ha recibido confirmación con el número de pedido.
- La acción está trazada con un correlation ID end-to-end [NFR-07, PRD §4].

**Given/When/Then:**
- **Given:** el Consumidor está autenticado, tiene un carrito con al menos 1 ítem de un restaurante activo, y ha seleccionado un método de pago válido.
- **When:** el Consumidor pulsa "Confirmar pedido".
- **Then:** el sistema crea el pedido en estado `PENDING_PAYMENT`, devuelve un número de pedido único (ej. `ORD-20260524-00423`) en ≤ 200 ms p95 [NFR-02, PRD §4], y el Consumidor recibe una notificación de confirmación.

---

### UC-02: Aceptar o rechazar ticket de cocina

| Campo          | Valor                                                                 |
|----------------|-----------------------------------------------------------------------|
| Actor primario | Restaurante                                                           |
| Capacidad PRD  | CAP-04 — Order Fulfillment / Kitchen [PRD §3]                        |
| Origen         | US-02 [Brief §A.5]                                                    |

**Precondiciones:**
- El pedido existe en el sistema en estado `PENDING_PAYMENT` y el pago ha sido autorizado (UC-04 completado con éxito).
- El sistema ha creado un ticket de cocina en estado `AWAITING_ACCEPTANCE` y lo ha publicado en el dashboard del Restaurante.
- El Restaurante tiene una sesión activa en su dashboard.

**Flujo principal:**
1. El Restaurante recibe una notificación push de nuevo ticket en su dashboard (CAP-07).
2. El Restaurante visualiza los detalles del ticket: ítems, cantidades, dirección de entrega estimada y hora de solicitud.
3. El Restaurante ingresa el tiempo estimado de preparación (en minutos).
4. El Restaurante pulsa "Aceptar ticket".
5. El sistema actualiza el ticket al estado `ACCEPTED` con el tiempo estimado registrado.
6. El sistema actualiza el pedido al estado `BEING_PREPARED`.
7. El sistema notifica al Consumidor que su pedido ha sido aceptado con el tiempo estimado (CAP-07).
8. El sistema inicia el proceso de búsqueda de courier cuando el tiempo estimado se aproxima a completarse (CAP-05, trigger de UC-03).

**Flujos alternativos:**
- FA-01: El Restaurante rechaza el ticket → el Restaurante debe ingresar un motivo de rechazo; el sistema cancela el pedido, actualiza su estado a `CANCELLED`, inicia la reversión del cobro en CAP-06, y notifica al Consumidor con el motivo del rechazo (CAP-07).
- FA-02: El Restaurante no responde en el timeout configurado (ej. 15 minutos) → el sistema escala la situación al Empleado FTGO (back office) y notifica al Consumidor del retraso; el back office puede reasignar el ticket a otro restaurante disponible o cancelar el pedido.

**Postcondiciones:**
- El ticket de cocina está en estado `ACCEPTED` con tiempo estimado registrado, o en estado `REJECTED` con motivo.
- El pedido refleja el estado correspondiente (`BEING_PREPARED` o `CANCELLED`).
- El Consumidor ha recibido notificación del resultado.
- El sistema ha disparado el siguiente paso del flujo (búsqueda de courier o reversión de cobro).

**Given/When/Then:**
- **Given:** existe un ticket de cocina en estado `AWAITING_ACCEPTANCE` para el restaurante, con pago previamente autorizado, visible en el dashboard del Restaurante.
- **When:** el Restaurante pulsa "Aceptar ticket" e ingresa un tiempo estimado de preparación de 25 minutos.
- **Then:** el ticket pasa a estado `ACCEPTED`, el pedido pasa a `BEING_PREPARED`, y el Consumidor recibe en ≤ 5 segundos una notificación indicando "Tu pedido ha sido aceptado. Tiempo estimado: 25 minutos."

---

### UC-03: Asignar pedido a courier

| Campo          | Valor                                                                 |
|----------------|-----------------------------------------------------------------------|
| Actor primario | Courier                                                               |
| Capacidad PRD  | CAP-05 — Delivery [PRD §3]                                           |
| Origen         | US-03 [Brief §A.5]                                                    |

**Precondiciones:**
- El ticket de cocina está en estado `ACCEPTED` (UC-02 completado) y el pedido en estado `BEING_PREPARED`.
- Al menos un Courier está disponible (ha marcado disponibilidad en la app) dentro del radio de asignación del restaurante.
- El Courier tiene la app activa y su ubicación GPS actualizada.

**Flujo principal:**
1. El sistema identifica el restaurante de origen y calcula los couriers disponibles más cercanos usando Google Maps (CAP-05).
2. El sistema selecciona al courier más cercano y le envía una oferta de asignación con: restaurante, dirección de entrega estimada y valor de la entrega.
3. El Courier recibe la notificación de oferta en su app con un contador de **30 segundos** para responder [Brief §A.5 — US-03].
4. El Courier acepta la oferta dentro del timeout.
5. El sistema asigna el pedido al Courier y actualiza el estado del pedido a `COURIER_ASSIGNED`.
6. El sistema calcula y presenta al Courier la ruta optimizada: primero al restaurante, luego a la dirección del Consumidor (integración con Google Maps).
7. El sistema notifica al Consumidor que su pedido tiene courier asignado e inicia el tracking en tiempo real (trigger de UC-05).
8. El sistema notifica al Restaurante el nombre y ETA del courier.

**Flujos alternativos:**
- FA-01: El Courier rechaza la oferta o no responde dentro de los 30 segundos → el sistema descarta la oferta para este courier y pasa al siguiente candidato en la lista de cercanos; si no hay couriers disponibles tras 3 intentos, escala al Empleado FTGO (back office).
- FA-02: Google Maps no está disponible → el sistema usa una estimación de ruta estática basada en distancia lineal (degradación aceptable según NFR-04 [PRD §4]); la asignación continúa sin bloqueo.

**Postcondiciones:**
- El pedido está en estado `COURIER_ASSIGNED` con la identidad del Courier registrada.
- El Courier tiene la ruta activa en su app.
- El Consumidor ha recibido notificación de asignación y puede iniciar el tracking (UC-05).
- El estado del Courier es `EN_RUTA`.

**Given/When/Then:**
- **Given:** el ticket de cocina está en estado `ACCEPTED`, hay al menos 1 Courier disponible a ≤ 2 km del restaurante, y el Courier tiene la app activa con GPS habilitado.
- **When:** el sistema envía la oferta al Courier y el Courier pulsa "Aceptar entrega" dentro de los 30 segundos.
- **Then:** el pedido cambia a estado `COURIER_ASSIGNED`, el Courier recibe la ruta optimizada (restaurante → consumidor), y el Consumidor recibe notificación "Tu courier está en camino" con ETA calculado.

---

### UC-04: Procesar pago del pedido

| Campo          | Valor                                                                 |
|----------------|-----------------------------------------------------------------------|
| Actor primario | Sistema (CAP-06 — Billing & Accounting)                              |
| Capacidad PRD  | CAP-06 — Billing & Accounting [PRD §3]                               |
| Origen         | Derivado de CAP-06 [PRD §3] + [Richardson Cap. 3 — Saga pattern]     |

**Precondiciones:**
- El pedido existe en estado `PENDING_PAYMENT` con un identificador único.
- El evento `OrderCreated` ha sido publicado por CAP-03 (UC-01, paso 8).
- Stripe está disponible o la cola de retry está activa (NFR-04 [PRD §4]).

**Flujo principal:**
1. CAP-06 recibe el evento `OrderCreated` con el monto total calculado por CAP-03.
2. El sistema construye la solicitud de cobro: monto, moneda, token de pago del Consumidor.
3. El sistema invoca la API de Stripe para autorizar y capturar el pago.
4. Stripe devuelve confirmación de cobro exitoso con un `charge_id`.
5. El sistema registra el cobro con el `charge_id` y actualiza el estado del pedido a `PAYMENT_CONFIRMED`.
6. El sistema publica el evento `PaymentConfirmed` para que CAP-04 proceda a crear el ticket de cocina (trigger de UC-02).
7. El sistema calcula la comisión de FTGO y registra la cuenta por pagar al restaurante para el próximo ciclo de payout.

**Flujos alternativos:**
- FA-01: Stripe rechaza el pago (tarjeta sin fondos, datos inválidos) → el sistema actualiza el estado del pedido a `PAYMENT_FAILED`, notifica al Consumidor para que actualice su método de pago (CAP-07), y el pedido queda en `PENDING_PAYMENT` aguardando reintento manual o cancelación.
- FA-02: Stripe no responde (timeout) → el sistema encola el intento de cobro y aplica backoff exponencial: reintento 1 a los 30 s, reintento 2 a los 60 s, reintento 3 a los 120 s [NFR-04, PRD §4]; si agota los 3 reintentos, el pedido pasa a `PAYMENT_FAILED` y se notifica al Consumidor.
- FA-03: El pedido es cancelado antes de confirmar el pago (ej. FA-01 de UC-02) → el sistema cancela la solicitud de cobro en Stripe si ya fue iniciada, registra la reversión y actualiza el estado a `CANCELLED`.

**Postcondiciones:**
- El pedido está en estado `PAYMENT_CONFIRMED` con el `charge_id` de Stripe registrado, o en `PAYMENT_FAILED` con motivo registrado.
- El evento `PaymentConfirmed` ha sido publicado para activar el flujo de cocina.
- La comisión de FTGO y la cuenta por pagar al restaurante están registradas en Billing.
- El flujo es trazable via correlation ID end-to-end [NFR-07, PRD §4].

**Given/When/Then:**
- **Given:** el pedido existe en estado `PENDING_PAYMENT` con monto total calculado, y Stripe está disponible con el token de pago del Consumidor válido.
- **When:** CAP-06 recibe el evento `OrderCreated` e invoca la API de Stripe.
- **Then:** Stripe confirma el cobro, el pedido pasa a estado `PAYMENT_CONFIRMED`, se registra el `charge_id`, y el evento `PaymentConfirmed` es publicado en ≤ 3 segundos desde la recepción del evento `OrderCreated` en condiciones normales de red.

---

### UC-05: Tracking en tiempo real del consumidor

| Campo          | Valor                                                                 |
|----------------|-----------------------------------------------------------------------|
| Actor primario | Consumidor                                                            |
| Capacidad PRD  | CAP-05 — Delivery [PRD §3]                                           |
| Origen         | Derivado de NFR-02 Latencia UX [PRD §4] + CAP-05 Delivery [PRD §3]  |

**Precondiciones:**
- El pedido está en estado `COURIER_ASSIGNED` (UC-03 completado).
- El Courier tiene la app activa con GPS habilitado y posición actualizada.
- El Consumidor tiene la app abierta en la pantalla de seguimiento del pedido.

**Flujo principal:**
1. El Consumidor accede a la pantalla de seguimiento usando el número de pedido o desde el historial de pedidos activos.
2. El sistema obtiene la posición GPS actual del Courier desde CAP-05.
3. El sistema renderiza el mapa (integración con Google Maps) con: posición del Courier, restaurante de origen y dirección de entrega del Consumidor.
4. La posición del Courier se actualiza en el mapa del Consumidor con una frecuencia de **≤ 5 segundos** entre actualizaciones.
5. El sistema calcula y muestra el ETA actualizado en tiempo real.
6. Cuando el Courier llega al restaurante, el sistema actualiza el estado del pedido a `COURIER_AT_RESTAURANT` y notifica al Consumidor.
7. Cuando el Courier sale del restaurante con el pedido, el estado cambia a `EN_ROUTE_TO_CONSUMER`.
8. Cuando el Courier completa la entrega, el estado cambia a `DELIVERED` y el tracking finaliza.

**Flujos alternativos:**
- FA-01: El GPS del Courier pierde señal temporalmente (> 30 s sin actualización) → el sistema muestra la última posición conocida con un indicador visual de "señal perdida" y continúa mostrando el ETA con la última posición registrada; no se interrumpe la pantalla de tracking.
- FA-02: Google Maps no está disponible → el sistema muestra el estado textual del pedido y el ETA estimado sin mapa visual; el tracking de estado (no visual) continúa funcionando [NFR-04, PRD §4].
- FA-03: El Consumidor cierra la app → el sistema continúa actualizando el estado en background y envía notificaciones push en los cambios de estado clave: `COURIER_AT_RESTAURANT`, `EN_ROUTE_TO_CONSUMER`, `DELIVERED`.

**Postcondiciones:**
- El Consumidor puede ver la posición del Courier en tiempo real con actualizaciones ≤ 5 segundos.
- El estado del pedido se ha progresado correctamente a `DELIVERED` al completar la entrega.
- El Courier ha confirmado la entrega en la app (foto de confirmación o código de confirmación).
- CAP-07 ha enviado la notificación de entrega completada al Consumidor.

**Given/When/Then:**
- **Given:** el pedido está en estado `COURIER_ASSIGNED`, el Courier tiene GPS activo y se encuentra en ruta al restaurante, y el Consumidor abre la pantalla de tracking del pedido.
- **When:** el Consumidor visualiza la pantalla de seguimiento en la app.
- **Then:** el mapa muestra la posición del Courier actualizada en ≤ 5 segundos, el ETA se calcula y muestra con precisión de ±2 minutos, y los cambios de estado del pedido (`COURIER_AT_RESTAURANT`, `EN_ROUTE_TO_CONSUMER`, `DELIVERED`) se reflejan en tiempo real en la pantalla sin recargar la página.

---

*Trazabilidad: los 5 UCs de este FSD derivan exclusivamente de las US semilla del brief [Brief §A.5], las capacidades del PRD [PRD §3] y los NFRs del PRD [PRD §4], citando Richardson donde aplica. Ningún UC ha sido inventado fuera de estas fuentes [Brief §A.6].*
