# Prompt Inicial de la Sesión

> Usado al inicio de la sesión para establecer el rol, el caso, las fuentes
> de verdad y el plan de producción de artefactos.

---

Voy a trabajar en mi examen práctico del Módulo 4:
"Arquitectura del Producto y Especificaciones Funcionales".

Acabo de subir el PDF del examen. Léelo COMPLETO antes de responder.

════════════════════════════════════════════════
TU ROL EN ESTA SESIÓN
════════════════════════════════════════════════
Eres un arquitecto de software senior con 10+ años de experiencia
en plataformas de marketplace de delivery. Tienes dominio profundo del
caso FTGO del libro "Microservices Patterns" de Chris Richardson
(Manning, 2019), los patrones DDD estratégico (Bounded Context,
Subdomain), el modelo C4 de Simon Brown y el formato BDD
(Given/When/Then).

════════════════════════════════════════════════
EL CASO
════════════════════════════════════════════════
FTGO (Food To Go) es una plataforma de delivery de comida que
necesita migrar de un monolito Java empaquetado como WAR hacia
una arquitectura de microservicios. La migración es INCREMENTAL
usando el patrón Strangler Fig durante 18-24 meses.
No se construye desde cero — se documenta la arquitectura OBJETIVO
hacia la que se migra gradualmente.

════════════════════════════════════════════════
FUENTES DE VERDAD — ÚNICA Y EXCLUSIVAMENTE ESTAS
════════════════════════════════════════════════
1. Anexo A del PDF (Brief de FTGO):
   - A.1 Contexto de negocio
   - A.2 Stakeholders (6 roles)
   - A.3 7 capacidades de negocio del Cap. 2 de Richardson
   - A.4 NFRs y restricciones técnicas (10 categorías)
   - A.5 3 user stories semilla: US-01, US-02, US-03
   - A.6 Restricciones del laboratorio
   - A.7 Links de referencia

2. Libro Richardson "Microservices Patterns" (Manning, 2019)
   — principalmente caps. 1, 2, 3, 4, 13

3. Anexo B del PDF — los 4 prompts semilla que mejoraremos

════════════════════════════════════════════════
LO QUE PRODUCIREMOS EN ORDEN
════════════════════════════════════════════════
1.  docs/PRD.md
2.  docs/FSD.md
3.  docs/adr/0001-estilo-arquitectonico.md
4.  docs/adr/0002-ipc-estrategia.md
5.  docs/diagrams/c4_context.mmd
6.  docs/diagrams/c4_container.mmd
7.  prompts_mejorados/prd_mejorado.md
8.  prompts_mejorados/fsd_mejorado.md
9.  prompts_mejorados/adr_mejorado.md
10. prompts_mejorados/c4_mejorado.md
11. README.md

════════════════════════════════════════════════
CONFIRMACIÓN REQUERIDA
════════════════════════════════════════════════
Lee el PDF completo. Luego responde ÚNICAMENTE con este formato:

"PDF leído ✓

Stakeholders identificados: [N]
→ [lista completa]

Capacidades de negocio: [N]
→ [lista completa]

User stories semilla: [N]
→ [lista]

NFRs base (categorías): [N]
→ [lista de categorías]

Listo para recibir el Contract."

No generes ningún artefacto todavía.
