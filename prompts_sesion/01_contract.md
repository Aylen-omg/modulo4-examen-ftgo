# Contract de la Sesión

> Usado después de la confirmación del PDF para establecer las 9 reglas
> absolutas que aplican a cada artefacto generado en la sesión.

---

Perfecto. Ahora establezco el CONTRACT de esta sesión.
Estas reglas son ABSOLUTAS — aplican a cada artefacto sin excepción.

════════════════════════════════════════════════
CONTRACT — 9 REGLAS DE LA SESIÓN
════════════════════════════════════════════════

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
R-01 | TRAZABILIDAD OBLIGATORIA EN CADA ELEMENTO
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Todo stakeholder, capacidad, NFR, UC, decisión o componente
DEBE citar su origen con este formato:

  Viene del brief        →  [Brief §A.X]
  Viene del libro        →  [Richardson Cap. X]
  Viene de una US        →  [US-01], [US-02] o [US-03]
  Es inferencia propia   →  [Inferido de Brief §A.X — razón]

Si no puedes trazar algo a estas fuentes → NO lo incluyas.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
R-02 | CERO INVENCIÓN DE DOMINIO
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PROHIBIDO inventar stakeholders, capacidades, NFRs, UCs o
componentes fuera del Anexo A o de Richardson.
Inventar dominio penaliza el examen.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
R-03 | CADENA DE COHERENCIA ENTRE ARTEFACTOS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  PRD → FSD → ADRs → C4 (cada uno alimenta al siguiente)

  • FSD: cada UC deriva de una capacidad del PRD
  • ADRs: cada decisión cita NFRs del PRD
  • C4 nivel 2: cada contenedor está justificado en un ADR
  • C4: si ADR eligió Kafka → Kafka aparece en el diagrama
  • C4: si ADR eligió Strangler Fig → Monolito Legacy aparece

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
R-04 | ADRs HONESTOS — POSITIVAS Y NEGATIVAS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Cada ADR debe mostrar consecuencias POSITIVAS y NEGATIVAS.
Mínimo 3 opciones reales con pros, contras e impacto en NFRs.
Sin contras → reintentar. Sin negativas → reintentar.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
R-05 | FORMATO ESTRICTO POR ARTEFACTO
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  PRD   → Markdown, 5 secciones, ≥8 NFRs con métrica numérica
  FSD   → Markdown, tabla UCs + 7 campos por UC + GWT concreto
  ADR   → Markdown, 5 secciones, ≥3 opciones, consecuencias +/-
  C4    → Mermaid válido C4Context + C4Container
          TODAS las relaciones del Nivel 2 con tecnología+protocolo

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
R-06 | UN ARTEFACTO A LA VEZ
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Genera UN artefacto cuando te lo pida.
Espera mi confirmación antes de pasar al siguiente.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
R-07 | AVISA ANTES DE ASUMIR
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Si algo del brief es ambiguo → dímelo antes de asumir.
No asumas en silencio.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
R-08 | MÉTRICAS SIEMPRE CON NÚMERO CONCRETO
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Correcto   →  "≤ 200 ms p95"  /  "99.9% mensual"  /  "5x baseline"
  Incorrecto →  "rápido"  /  "confiable"  /  "bueno rendimiento"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
R-09 | STRANGLER FIG SIEMPRE PRESENTE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
El monolito legacy NO desaparece de golpe [Brief §A.4].
Strangler Fig debe aparecer en:
  • Alcance del PRD
  • Restricciones de los ADRs
  • Diagrama C4 como componente "Monolito Legacy"

════════════════════════════════════════════════
CONFIRMACIÓN REQUERIDA
════════════════════════════════════════════════
Responde ÚNICAMENTE con:

"Contract aceptado ✓
9 reglas registradas.
Listo para generar: docs/PRD.md"
