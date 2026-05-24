# ADR 0001 — Estilo Arquitectónico para la Migración de FTGO

**Status:** Proposed
**Fecha:** 2026-05-24
**Autores:** Equipo de Arquitectura FTGO
**Fuentes:** Brief §A.1, §A.4 | Richardson Cap. 1, Cap. 2

---

## 1. Contexto

FTGO opera actualmente como un monolito Java empaquetado como WAR que exhibe los síntomas clásicos del *monolithic hell* descritos en [Richardson Cap. 1]: ciclos de build que superan los 20 minutos, imposibilidad de escalar el módulo de Delivery sin escalar también el módulo de Billing, ausencia de aislamiento de fallos (un error en Notifications ha causado degradación del flujo de toma de pedidos en producción), y lock-in tecnológico que impide adoptar soluciones más adecuadas por dominio. El equipo de desarrollo ha crecido hasta el punto en que múltiples squads modifican el mismo código base, generando conflictos de merge frecuentes y reduciendo la velocidad de entrega [Brief §A.1].

La dirección de FTGO ha aprobado la migración a una arquitectura de microservicios, pero el sistema está en producción activa con consumidores reales. Esta condición descarta cualquier estrategia de reemplazo big-bang y obliga a elegir un estilo arquitectónico que pueda coexistir con el monolito existente durante el período de transición [Brief §A.4 — Migración incremental]. La decisión sobre el estilo arquitectónico es el fundamento de todas las decisiones técnicas posteriores: determina cómo se descompone el sistema, qué estrategia de IPC es posible, cómo se gestiona la consistencia de datos y cuál es el riesgo operativo de la transición.

Las restricciones no negociables que acotan las opciones son: el flujo de toma de pedidos debe mantener ≥ 99.9 % de disponibilidad mensual [NFR-02, PRD §4], el sistema debe poder escalar componentes individuales hasta 5× el tráfico baseline en horarios pico [NFR-03, PRD §4], el monolito debe seguir operando durante 18 a 24 meses sin interrupciones [NFR-07, PRD §4], y los servicios del core deben implementarse en Java/Spring Boot para preservar el conocimiento del equipo [Brief §A.4 — Tecnología].

---

## 2. Restricciones Consideradas

| Restricción | Valor | Impacto en la decisión |
|---|---|---|
| NFR-01 Latencia UX | ≤ 200 ms p95 en acciones del consumidor | Penaliza overhead de red innecesario; favorece descomposición quirúrgica sobre descomposición máxima |
| NFR-02 Disponibilidad | ≥ 99.9 % mensual en Order Taking | Exige aislamiento de fallos entre componentes; el monolito actual no puede garantizarlo |
| NFR-03 Escalabilidad | Soportar 5× tráfico pico independientemente | Exige escalado horizontal por componente; imposible si los módulos comparten proceso |
| NFR-04 Tolerancia a fallos externos | Pedidos aunque Stripe esté caído | Favorece desacoplamiento y patrones async; requiere autonomía de los servicios core |
| NFR-07 Migración incremental | Coexistencia con monolito 18-24 meses | Prohíbe reemplazo big-bang; exige Strangler Fig como estrategia de transición |
| NFR-08 Tecnología preferida | Java 17+ / Spring Boot 3.x en servicios core | Restringe stacks exóticos en servicios principales; libertad en servicios satélite |

---

## 3. Opciones Consideradas

---

### Opción 1: Microservicios completos desde el inicio

**Descripción:** se extraen los 7 servicios correspondientes a las 7 capacidades de negocio del PRD [Richardson Cap. 2] de forma simultánea o en un horizonte muy corto (< 3 meses). Desde el día 1 el sistema objetivo opera como un conjunto de microservicios independientes con bases de datos propias, sin el monolito como intermediario.

**Pros:**
- Escalabilidad horizontal independiente por servicio desde el inicio → cubre NFR-03 (5× pico).
- Aislamiento total de fallos: un fallo en Notifications no afecta Order Taking → cubre NFR-02 (99.9 %).
- Libertad tecnológica por servicio desde el principio → alineado con Brief §A.4 — Tecnología.
- Alineación directa con el modelo de descomposición por capacidad de negocio [Richardson Cap. 2].

**Contras:**
- Viola NFR-07: exige un período de reescritura paralela que supera los 18-24 meses de coexistencia permitidos; en la práctica es un big-bang disfrazado de migración.
- Riesgo de *Distributed Monolith*: si las 7 capacidades tienen acoplamiento de datos no identificado, los servicios quedan acoplados vía llamadas síncronas en cadena, degradando la latencia [NFR-01].
- Costo operativo inicial alto: 7 pipelines CI/CD, 7 bases de datos, service discovery, distributed tracing y gestión de secretos desde el día 1, sin el equipo aún capacitado en operaciones distribuidas.
- El equipo de FTGO no tiene experiencia previa en microservicios; operar 7 servicios simultáneamente incrementa el riesgo de error humano en producción.

**Impacto en NFRs:**

| NFR | Impacto |
|---|---|
| NFR-01 Latencia ≤ 200 ms p95 | ⚠ Riesgo alto: llamadas entre servicios en el happy path (Order Taking → Billing → Kitchen) pueden superar el umbral si hay 3+ saltos de red síncronos |
| NFR-02 Disponibilidad ≥ 99.9 % | ✓ Potencialmente cumplible si el aislamiento está bien implementado; pero el riesgo de Distributed Monolith puede reintroducir fallos en cascada |
| NFR-03 Escalabilidad 5× | ✓ Cumplible: cada servicio escala independientemente |
| NFR-07 Migración 18-24 meses | ✗ Incompatible: la extracción simultánea de 7 servicios excede el horizonte de coexistencia y no permite validar cada extracción |

**Compatibilidad Strangler Fig:** Incompatible. Strangler Fig requiere extraer un servicio a la vez y mantener el monolito como fallback; una extracción simultánea de 7 servicios elimina el monolito como red de seguridad antes de que los nuevos servicios estén validados en producción.

---

### Opción 2: Modular Monolith

**Descripción:** en lugar de migrar a microservicios, se refactoriza el monolito existente para que sus módulos internos (correspondientes a las 7 capacidades) estén bien encapsulados con interfaces claras, pero sigan corriendo en el mismo proceso. No hay distribución de red entre módulos.

**Pros:**
- Eliminación inmediata de los conflictos de merge: los módulos bien delimitados reducen el acoplamiento de código [Richardson Cap. 1].
- Ausencia de overhead de red: la latencia entre módulos es una llamada de método en memoria → NFR-01 cumplido sin riesgo.
- Sin complejidad operativa de sistemas distribuidos: un solo deployment, una sola base de datos, sin distributed tracing obligatorio.
- Menor curva de aprendizaje: el equipo trabaja con el stack Java/Spring que ya conoce [Brief §A.4 — Tecnología].

**Contras:**
- No resuelve el escalado conflictivo: el módulo de Delivery y el módulo de Billing siguen corriendo en el mismo proceso; escalar uno escala el otro → NFR-03 no cumplido.
- No resuelve el aislamiento de fallos: un error no controlado en un módulo sigue pudiendo afectar el proceso completo → NFR-02 en riesgo.
- No es la dirección estratégica aprobada por la dirección de FTGO [Brief §A.1]: posterga el problema sin resolverlo.
- Acumula deuda técnica adicional: la refactorización interna no elimina el lock-in tecnológico ni mejora la autonomía de los equipos a largo plazo.
- Bloquea la eventual migración a microservicios: los módulos bien encapsulados ayudan, pero la migración futura seguirá requiriendo la misma inversión de extracción.

**Impacto en NFRs:**

| NFR | Impacto |
|---|---|
| NFR-01 Latencia ≤ 200 ms p95 | ✓ Cumplido: sin overhead de red entre módulos |
| NFR-02 Disponibilidad ≥ 99.9 % | ✗ No cumplido: sin aislamiento de proceso, un fallo en un módulo puede derribar el proceso completo |
| NFR-03 Escalabilidad 5× | ✗ No cumplido: el proceso único no permite escalado diferenciado por módulo |
| NFR-07 Migración 18-24 meses | ⚠ Parcial: no viola el plazo, pero tampoco avanza hacia el objetivo arquitectónico aprobado |

**Compatibilidad Strangler Fig:** Incompatible como destino. El Modular Monolith puede ser un estado intermedio útil durante la refactorización, pero no es el estado objetivo declarado en [Brief §A.1] y no permite aplicar Strangler Fig para extraer servicios independientes en producción.

---

### Opción 3: Migración híbrida progresiva con Strangler Fig

**Descripción:** se mantiene el monolito en producción y se extraen las capacidades de negocio de forma incremental, comenzando por las que ofrecen el mayor beneficio con el menor riesgo (ej. Notifications y Consumer Management primero; Order Taking y Delivery en fases posteriores). El monolito actúa como fallback durante la transición. Un API Gateway o proxy dirige el tráfico entre el monolito y los servicios ya extraídos, conforme a la estrategia Strangler Fig [Richardson Cap. 13].

**Pros:**
- Compatible directamente con NFR-07: la coexistencia de 18-24 meses es el mecanismo de transición, no un problema [Brief §A.4 — Migración incremental].
- Cada extracción se valida en producción antes de proceder con la siguiente → riesgo operativo acotado y gradual.
- El equipo adquiere experiencia en operaciones distribuidas de forma progresiva, reduciendo la curva de aprendizaje.
- Permite priorizar las capacidades con mayor valor de negocio (ej. extraer Delivery primero para habilitar el escalado independiente en picos) → cubre NFR-03 de forma progresiva.
- Compatible con la pila Java/Spring Boot que el equipo ya conoce [Brief §A.4 — Tecnología].
- Permite validar decisiones de IPC y datos (ADR 0002) con un servicio real antes de aplicarlas a todos.

**Contras:**
- Complejidad operativa dual durante 18-24 meses: el equipo debe mantener simultáneamente el monolito y los servicios extraídos, con dos pipelines de deployment distintos.
- El API Gateway / proxy de enrutamiento introduce un punto de fallo adicional que debe gestionarse con cuidado para no degradar NFR-01 o NFR-02.
- El progreso es más lento que una extracción simultánea: los beneficios de escalado y aislamiento se realizan de forma gradual, no inmediata.
- Riesgo de "never-ending migration": sin una hoja de ruta y criterios de éxito claros por fase, la migración puede estancarse indefinidamente con el monolito nunca completamente reemplazado.

**Impacto en NFRs:**

| NFR | Impacto |
|---|---|
| NFR-01 Latencia ≤ 200 ms p95 | ✓ Cumplible: el API Gateway añade ≤ 5-10 ms de overhead; la extracción quirúrgica evita cadenas de llamadas largas |
| NFR-02 Disponibilidad ≥ 99.9 % | ✓ Cumplible progresivamente: cada servicio extraído aporta aislamiento de fallos; el monolito como fallback reduce el riesgo durante la transición |
| NFR-03 Escalabilidad 5× | ✓ Cumplible por fases: las capacidades extraídas escalan independientemente desde el momento de su extracción |
| NFR-07 Migración 18-24 meses | ✓ Diseñado exactamente para este horizonte; el monolito coexiste hasta que el último servicio es extraído y validado |

**Compatibilidad Strangler Fig:** Totalmente compatible. Strangler Fig [Richardson Cap. 13] es el patrón de implementación de esta opción: el proxy enruta gradualmente el tráfico desde el monolito hacia los servicios extraídos, permitiendo un reemplazo sin downtime.

---

## 4. Decisión

**Se elige:** Opción 3 — Migración híbrida progresiva con Strangler Fig.

**Justificación:** es la única opción que cumple simultáneamente todas las restricciones no negociables del brief. NFR-07 [Brief §A.4 — Migración incremental] prohíbe el big-bang implícito de la Opción 1. La Opción 2 no resuelve el escalado diferenciado (NFR-03) ni el aislamiento de fallos (NFR-02), que son los dos síntomas críticos que motivan la migración. La Opción 3 permite coexistencia controlada durante 18-24 meses, extracción validada servicio por servicio, y adquisición gradual de madurez operativa por parte del equipo. El patrón Strangler Fig está explícitamente recomendado en [Richardson Cap. 13] para escenarios de migración incremental de monolitos en producción, y es coherente con las restricciones de tecnología [Brief §A.4] y disponibilidad [NFR-02].

El orden de extracción sugerido, a validar en el plan de migración detallado: (1) Notifications — menor riesgo, desacoplada; (2) Consumer Management — identidad estable; (3) Restaurant Management — catálogo con bajo volumen de escritura; (4) Billing — requiere madurez en patrones async [Richardson Cap. 3]; (5) Order Taking + Order Fulfillment — máxima criticidad, última en extraerse; (6) Delivery — alta demanda de escalado, prioridad si NFR-03 es urgente.

**Referencias:** [Richardson Cap. 1 — Monolithic Hell] + [Richardson Cap. 2 — Decompose by Business Capability] + [Richardson Cap. 13 — Strangler Fig] + [Brief §A.1 — Contexto de negocio] + [Brief §A.4 — Migración incremental].

---

## 5. Consecuencias

### ✅ Positivas

- El monolito sigue atendiendo producción sin interrupciones durante toda la migración; los consumidores no perciben la transición [NFR-07].
- Cada servicio extraído aporta aislamiento de fallos incremental: a medida que Order Taking y Delivery se extraen, la disponibilidad del flujo crítico mejora y se acerca al objetivo 99.9 % [NFR-02].
- El equipo adquiere experiencia en operaciones distribuidas (distributed tracing, service discovery, gestión de secretos) de forma progresiva y con riesgo acotado, en lugar de enfrentar la complejidad completa en el día 1.
- Las decisiones de ADR 0002 (IPC, estrategia de datos) pueden validarse con un servicio piloto antes de aplicarse a las 7 capacidades, reduciendo el riesgo de decisiones arquitectónicas incorrectas a escala.
- La hoja de ruta es revisable: si un servicio extraído presenta problemas, puede reintegrarse al monolito temporalmente sin impacto al resto.

### ⚠ Negativas

- Durante 18-24 meses el equipo debe mantener dos sistemas en paralelo (monolito + servicios extraídos) con distintos ciclos de deployment, frameworks de observabilidad y bases de código; esto aumenta la carga operativa y el riesgo de inconsistencias entre ambos contextos.
- El API Gateway / proxy de Strangler Fig es un componente crítico nuevo: si falla, todo el tráfico se ve afectado. Requiere alta disponibilidad propia y añade un punto de diagnóstico adicional cuando hay incidentes de latencia [NFR-01].
- Sin una gobernanza firme del plan de migración (hoja de ruta con fases, criterios de salida por fase y ownership claro), existe el riesgo real de que la migración se estanque y el monolito nunca sea completamente reemplazado — la deuda técnica se vuelve permanente.
- Los beneficios de escalado independiente (NFR-03) solo se materializan para las capacidades ya extraídas; Order Taking, la capacidad más crítica, puede ser la última en extraerse debido a su complejidad, dejando el cuello de botella de escalado intacto durante la mayor parte del período de transición.

---

## 6. Follow-ups

- **ADR 0002:** decidir la estrategia de IPC (comunicación entre procesos) entre los servicios extraídos y el monolito: síncrona REST/gRPC vs. asíncrona via message broker (Kafka/RabbitMQ). La elección de IPC debe ser coherente con la decisión de Strangler Fig de este ADR y con NFR-04 (tolerancia a fallos externos).
- **POC recomendado:** extraer el servicio de Notifications como piloto de la estrategia Strangler Fig antes de comprometer la extracción de capacidades críticas. El POC debe validar: (a) el overhead de latencia del API Gateway en NFR-01, (b) la estrategia de enrutamiento de tráfico, y (c) la operación de distributed tracing con el primer servicio independiente.
- **Plan de migración detallado:** definir el orden exacto de extracción de las 7 capacidades con criterios de éxito verificables por fase (métricas de disponibilidad, latencia y cobertura de tests) para evitar el riesgo de migración indefinida.
