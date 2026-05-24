# ADR 0002 — Estrategia de Comunicación entre Servicios (IPC) para FTGO

**Status:** Proposed
**Fecha:** 2026-05-24
**Autores:** Equipo de Arquitectura FTGO
**Fuentes:** Brief §A.4 | Richardson Cap. 3, Cap. 4
**Relacionado con:** ADR 0001 — Estilo Arquitectónico (Strangler Fig)

---

## 1. Contexto

En una arquitectura de microservicios, la elección del mecanismo de comunicación entre servicios (IPC — Inter-Process Communication) es una de las decisiones más influyentes sobre la disponibilidad, la latencia y la tolerancia a fallos del sistema. A diferencia del monolito, donde los módulos se comunican mediante llamadas de método en memoria, los servicios distribuidos deben cruzar la red para cada interacción, exponiendo el sistema a fallos de red, latencia variable y cascadas de fallos [Richardson Cap. 3]. La elección incorrecta de IPC puede convertir una arquitectura de microservicios en un *Distributed Monolith*: servicios técnicamente separados pero acoplados operacionalmente.

En el contexto específico de FTGO, el flujo de toma de pedidos involucra una cadena de capacidades que deben interactuar: el Consumidor crea un pedido (CAP-03), se procesa el pago (CAP-06), se notifica al restaurante (CAP-04) y se asigna un courier (CAP-05). Si esta cadena usa exclusivamente comunicación síncrona, un fallo en cualquier eslabón (ej. Stripe caído) puede bloquear todo el flujo, violando NFR-04 [Brief §A.4 — Tolerancia a fallos externos]. Al mismo tiempo, el tracking en tiempo real del consumidor (UC-05) requiere baja latencia de lectura, donde la mensajería asíncrona introduce complejidad innecesaria [NFR-01, PRD §4].

La decisión de ADR 0001 de usar Strangler Fig impone una restricción adicional: el mecanismo de IPC debe ser interoperable con el monolito legacy Java/WAR durante 18-24 meses. Los servicios extraídos deben poder comunicarse tanto con otros microservicios como con el monolito existente sin requerir cambios profundos en el monolito, que sigue en producción [NFR-07, Brief §A.4 — Migración incremental].

---

## 2. Restricciones Consideradas

| Restricción | Valor | Impacto en la decisión |
|---|---|---|
| NFR-01 Latencia UX | ≤ 200 ms p95 en acciones del consumidor | Penaliza cadenas largas de llamadas síncronas (cada salto añade ~10-30 ms); favorece async para eventos de dominio no bloqueantes |
| NFR-02 Disponibilidad | ≥ 99.9 % mensual en Order Taking | Un fallo en cadena síncrona (A llama B llama C) puede derribar el flujo completo; favorece desacoplamiento temporal |
| NFR-04 Tolerancia a fallos externos | Pedidos operativos aunque Stripe esté caído | Exige desacoplamiento temporal entre Order Taking y Billing; incompatible con REST síncrono puro en el flujo de pago |
| NFR-05 Consistencia eventual | Lag ≤ 5 s p99 entre servicios para reporting | Compatible con mensajería async; el broker garantiza entrega eventual |
| NFR-05 Consistencia fuerte | Dentro del aggregate de un pedido | Puede requerir síncrono para validaciones críticas dentro del mismo aggregate (ej. verificar disponibilidad del restaurante antes de confirmar el pedido) |
| NFR-07 Migración incremental | Coexistencia con monolito 18-24 meses | El mecanismo IPC debe soportar interoperabilidad con WAR Java legacy; el monolito debe poder publicar y consumir eventos sin reescritura completa |
| ADR 0001 | Strangler Fig con extracción gradual | Los servicios extraídos deben comunicarse con el monolito durante la transición; el IPC debe ser adoptable de forma incremental |

---

## 3. Opciones Consideradas

---

### Opción 1: REST síncrono (HTTP/JSON) para toda comunicación

**Descripción:** todos los servicios de FTGO se comunican exclusivamente mediante llamadas HTTP/JSON síncronas. Order Taking llama síncronamente a Billing para autorizar el pago, Billing llama síncronamente a Kitchen para crear el ticket, y así sucesivamente. El monolito expone y consume REST endpoints estándar.

**Pros:**
- Simplicidad de implementación: Spring Boot tiene soporte nativo para REST con `RestTemplate` / `WebClient`; el equipo ya conoce el stack → [Brief §A.4 — Tecnología].
- Facilidad de debugging y trazabilidad: las llamadas síncronas tienen un stack trace lineal y son fáciles de observar con herramientas estándar → cubre NFR-07 (trazabilidad).
- Compatible inmediatamente con el monolito legacy: el WAR Java puede exponer y consumir REST endpoints sin modificaciones arquitectónicas profundas → cubre NFR-07 (migración).
- Consistencia fuerte natural: la respuesta síncrona confirma que la operación se completó antes de continuar → cubre NFR-05 (consistencia fuerte dentro del aggregate).

**Contras:**
- Acoplamiento temporal total: si Billing (Stripe) no responde, Order Taking queda bloqueado → viola NFR-04 directamente [Brief §A.4 — Tolerancia a fallos externos].
- Cascadas de fallos: en una cadena Order Taking → Billing → Kitchen → Notifications, un fallo en cualquier servicio se propaga síncronamente hacia arriba → viola NFR-02 (99.9 % disponibilidad).
- Latencia acumulativa: cada salto síncrono añade ~10-30 ms de overhead de red; una cadena de 4 servicios puede superar el umbral de 200 ms p95 en picos de carga → riesgo NFR-01.
- Sin tolerancia a particiones de red: si un servicio está temporalmente inaccesible, la operación falla inmediatamente sin posibilidad de retry transparente.

**Impacto en NFRs:**

| NFR | Impacto |
|---|---|
| NFR-01 Latencia ≤ 200 ms p95 | ⚠ Riesgo en cadenas de 3+ servicios síncronos; cumplible solo si la cadena tiene ≤ 2 saltos de red |
| NFR-02 Disponibilidad ≥ 99.9 % | ✗ No cumplido: una cadena síncrona propaga fallos; Order Taking depende de la disponibilidad de Billing y Stripe |
| NFR-04 Tolerancia a fallos externos | ✗ No cumplido: si Stripe falla, el pedido no puede completarse en el flujo síncrono |
| NFR-05 Consistencia | ✓ Consistencia fuerte garantizada; ✗ consistencia eventual no aplicable en modelo puro síncrono |

**Compatibilidad con monolito legacy:** Compatible. El monolito Java puede exponer y consumir REST endpoints sin cambios arquitectónicos, lo que facilita la fase inicial de Strangler Fig.

---

### Opción 2: Mensajería asíncrona con Apache Kafka para toda comunicación

**Descripción:** todos los servicios de FTGO se comunican exclusivamente mediante eventos publicados en Apache Kafka. Order Taking publica un evento `OrderCreated`, Billing lo consume y publica `PaymentConfirmed`, Kitchen consume `PaymentConfirmed` y publica `TicketCreated`, etc. No hay llamadas REST directas entre servicios.

**Pros:**
- Desacoplamiento temporal total: si Billing está caído, Order Taking puede publicar el evento y Billing lo procesará cuando se recupere → cubre NFR-04 [Brief §A.4 — Tolerancia a fallos externos].
- Aislamiento de fallos: un servicio caído no bloquea síncronamente a los demás → cubre NFR-02 (disponibilidad).
- Escalabilidad natural: Kafka permite múltiples consumidores por tópico; los servicios escalan independientemente sin coordinación → cubre NFR-03 (5× pico).
- Soporte nativo para el patrón Saga [Richardson Cap. 4]: la orquestación de transacciones distribuidas multi-servicio es natural en un modelo de eventos.

**Contras:**
- Complejidad operativa alta desde el día 1: Kafka requiere un clúster propio con alta disponibilidad, gestión de particiones, grupos de consumidores, offsets y retención de mensajes; el equipo de FTGO no tiene experiencia previa en Kafka.
- Consistencia eventual inevitable: el flujo Order Taking no puede confirmar síncronamente al consumidor que el pago fue procesado; el Consumidor recibe `PENDING_PAYMENT` y debe esperar la actualización → puede degradar la UX en el flujo crítico.
- Debugging complejo: una cadena de eventos a través de múltiples tópicos es difícil de trazar sin distributed tracing especializado; un mensaje perdido o en dead-letter queue puede ser difícil de diagnosticar.
- Interoperabilidad con monolito legacy limitada: integrar el WAR Java con Kafka requiere añadir un Kafka producer/consumer al monolito, lo que representa una modificación no trivial en el código legacy [NFR-07].

**Impacto en NFRs:**

| NFR | Impacto |
|---|---|
| NFR-01 Latencia ≤ 200 ms p95 | ⚠ Riesgo en flujos donde el consumidor espera confirmación síncrona: el modelo async puede aumentar el tiempo percibido de respuesta si la UX no gestiona correctamente el estado `PENDING` |
| NFR-02 Disponibilidad ≥ 99.9 % | ✓ Cumplido: el desacoplamiento temporal elimina las cascadas de fallos síncronos |
| NFR-04 Tolerancia a fallos externos | ✓ Cumplido: Order Taking publica el evento y Billing lo procesa cuando Stripe esté disponible |
| NFR-05 Consistencia | ✓ Consistencia eventual natural; ✗ consistencia fuerte dentro del aggregate requiere mecanismos adicionales (ej. Saga orquestada con estado) |

**Compatibilidad con monolito legacy:** Parcial. El monolito debe incorporar un cliente Kafka para producir y consumir eventos, lo que requiere modificaciones al WAR legacy; esto añade riesgo y complejidad durante la fase de coexistencia Strangler Fig.

---

### Opción 3: Híbrido — REST síncrono para queries/lecturas, Kafka async para comandos/eventos de dominio

**Descripción:** se aplica el principio CQRS-light: las **lecturas** (consultar menú, ver estado del pedido, tracking en tiempo real) usan REST/HTTP síncrono; los **comandos y eventos de dominio** (OrderCreated, PaymentConfirmed, TicketAccepted, CourierAssigned) se publican en Apache Kafka. Dentro de un mismo aggregate (ej. validar disponibilidad del restaurante antes de confirmar el pedido), se permite REST síncrono. Las transacciones multi-servicio usan el patrón Saga [Richardson Cap. 4]. Esta opción está alineada directamente con el diseño de FTGO descrito en [Richardson Cap. 3].

**Pros:**
- Balance óptimo entre consistencia y desacoplamiento: REST para donde se necesita respuesta inmediata (lecturas, validaciones dentro del aggregate); Kafka para donde el desacoplamiento temporal es crítico (pago, cocina, courier) → cubre simultáneamente NFR-01 y NFR-04.
- Tolerancia a fallos en el flujo crítico de pago: Order Taking publica `OrderCreated` en Kafka y responde inmediatamente al consumidor con `PENDING_PAYMENT` sin esperar a Billing → cubre NFR-04 [Brief §A.4 — Tolerancia a fallos externos].
- Compatible con Strangler Fig: el monolito puede comenzar consumiendo eventos Kafka sin exponer toda su lógica vía REST; la migración puede hacerse en capas (primero exponer lecturas REST, luego migrar comandos a eventos) → cubre NFR-07.
- Alineación con el libro: Richardson diseña FTGO exactamente con este patrón en Cap. 3 y Cap. 4; las Sagas para Order Taking, Billing y Kitchen son el ejemplo canónico del libro [Richardson Cap. 4].

**Contras:**
- Mayor complejidad conceptual para el equipo: los desarrolladores deben entender cuándo usar REST y cuándo Kafka; sin guías claras, el patrón puede aplicarse de forma inconsistente entre equipos.
- Kafka sigue siendo necesario como infraestructura desde el momento en que se extrae el primer servicio con comandos async; esto introduce la complejidad operativa de Kafka antes de que todos los servicios estén extraídos.
- Las Sagas para transacciones distribuidas (ej. crear pedido + cobrar + crear ticket) son más difíciles de implementar correctamente que una transacción local, y requieren gestión explícita de compensaciones [Richardson Cap. 4].
- El debugging es más complejo que REST puro: una operación puede involucrar tanto llamadas REST trazables como eventos Kafka en tópicos distintos; el equipo necesita distributed tracing desde el inicio [NFR-07 — Trazabilidad].

**Impacto en NFRs:**

| NFR | Impacto |
|---|---|
| NFR-01 Latencia ≤ 200 ms p95 | ✓ Cumplido: las lecturas usan REST síncrono directo; los comandos async no bloquean la respuesta al consumidor |
| NFR-02 Disponibilidad ≥ 99.9 % | ✓ Cumplido: los comandos desacoplados por Kafka eliminan las cascadas de fallos síncronos en el flujo crítico |
| NFR-04 Tolerancia a fallos externos | ✓ Cumplido: Order Taking no espera a Billing; el evento `OrderCreated` se encola y Billing lo procesa cuando Stripe esté disponible |
| NFR-05 Consistencia | ✓ Consistencia fuerte para lecturas síncronas dentro del aggregate; ✓ consistencia eventual para eventos entre servicios con lag ≤ 5 s p99 |

**Compatibilidad con monolito legacy:** Compatible de forma incremental. El monolito puede comenzar exponiendo REST endpoints para lecturas (fase 1 de Strangler Fig) y luego incorporar un cliente Kafka para eventos (fase 2), distribuyendo el esfuerzo de integración a lo largo del período de migración.

---

## 4. Decisión

**Se elige:** Opción 3 — Híbrido REST síncrono para lecturas + Kafka async para comandos y eventos de dominio.

**Justificación:** es la única opción que satisface simultáneamente NFR-01 (latencia en lecturas), NFR-04 (tolerancia a fallos en el flujo de pago) y NFR-07 (coexistencia con el monolito). La Opción 1 (REST puro) viola directamente NFR-04: si Stripe está caído, el pedido no puede completarse, lo cual es inaceptable para el flujo de mayor criticidad de negocio. La Opción 2 (Kafka puro) introduce complejidad operativa total desde el día 1 y complica la integración con el monolito legacy, aumentando el riesgo de la fase inicial de Strangler Fig. La Opción 3 aplica el principio de "usar la herramienta correcta para cada caso": REST donde la respuesta inmediata es esencial (validar disponibilidad del restaurante, confirmar el pedido al consumidor), Kafka donde el desacoplamiento temporal es el valor (procesar el pago, crear el ticket de cocina, asignar el courier). Este diseño es exactamente el que Richardson utiliza para FTGO en los capítulos 3 y 4 de su libro, validando la decisión con el caso de referencia canónico.

**Regla práctica de aplicación:**
- Usar **REST síncrono** cuando: el resultado de la llamada es necesario para continuar el flujo (ej. validar que el restaurante está activo antes de crear el pedido), o cuando el actor espera una respuesta inmediata (ej. consultar el menú).
- Usar **Kafka async** cuando: el procesamiento puede ocurrir después (ej. cobrar, crear ticket, asignar courier), o cuando el receptor puede estar temporalmente caído sin bloquear al emisor.

**Referencias:** [Richardson Cap. 3 — Interprocess Communication] + [Richardson Cap. 4 — Managing Transactions with Sagas] + [Brief §A.4 — Tolerancia a fallos externos] + [Brief §A.4 — Consistencia de datos] + ADR 0001.

---

## 5. Consecuencias

### ✅ Positivas

- El flujo de toma de pedidos puede completarse aunque Stripe esté temporalmente caído: Order Taking publica `OrderCreated` en Kafka y responde al Consumidor con `PENDING_PAYMENT` de inmediato; Billing procesa el pago cuando Stripe se recupere [NFR-04].
- Las lecturas del Consumidor (ver menú, consultar estado del pedido, tracking) tienen latencia predecible vía REST sin overhead de mensajería asíncrona [NFR-01].
- El patrón Saga [Richardson Cap. 4] permite gestionar transacciones multi-servicio (crear pedido + cobrar + crear ticket) con compensaciones explícitas, evitando estados inconsistentes sin necesidad de transacciones distribuidas 2PC.
- La integración con el monolito legacy puede hacerse en fases: primero REST para lecturas, luego Kafka para comandos; esto alinea con el plan de extracción progresiva de ADR 0001.
- Apache Kafka como broker de eventos proporciona un registro duradero de todos los eventos de dominio, que puede usarse para auditoría, replay y construcción de proyecciones de datos para reporting [NFR-07 — Trazabilidad].

### ⚠ Negativas

- El equipo debe adquirir expertise en dos paradigmas de comunicación simultáneamente: REST/HTTP para lecturas y Kafka para comandos. Sin guías de equipo claras (cuándo usar cada uno), el patrón puede aplicarse de forma inconsistente, generando deuda técnica difícil de revertir.
- Apache Kafka introduce un componente de infraestructura crítico adicional: si el clúster de Kafka falla, todos los flujos de comandos async se detienen. Kafka debe operar con alta disponibilidad propia (mínimo 3 brokers en producción), lo que añade costo y complejidad operativa desde el momento de la primera extracción de servicio.
- Las Sagas de compensación son más difíciles de implementar y depurar que las transacciones locales: un error en la lógica de compensación de una Saga (ej. reembolso fallido tras cancelación) puede generar estados inconsistentes entre Order Taking y Billing que son difíciles de detectar y corregir en producción [Richardson Cap. 4].
- El debugging de un flujo que cruza REST y Kafka requiere distributed tracing configurado desde el inicio (correlation ID propagado tanto en headers HTTP como en headers de mensajes Kafka); sin esta instrumentación, diagnosticar un pedido fallido en producción puede tomar horas.

---

## 6. Follow-ups

- **ADR 0003 (recomendado):** decidir la estrategia de gestión de datos entre servicios: *Database-per-Service* vs. esquema compartido. La elección de Kafka como broker de comandos implica que los servicios deben tener bases de datos independientes para evitar acoplamiento; esta decisión debe formalizarse antes de la primera extracción de servicio con datos propios.
- **POC recomendado:** implementar el flujo `OrderCreated → PaymentConfirmed → TicketCreated` usando Kafka con el patrón Saga [Richardson Cap. 4] entre el servicio de Order Taking extraído y el monolito legacy actuando como consumidor de eventos. El POC debe medir: (a) latencia end-to-end del flujo feliz, (b) comportamiento cuando Stripe está caído (mensaje en cola), y (c) capacidad de replay de eventos tras una caída del consumidor.
- **Guía de equipo:** documentar la regla REST vs. Kafka como estándar de arquitectura interno antes de que los equipos comiencen a extraer servicios, para evitar aplicación inconsistente del patrón híbrido.
