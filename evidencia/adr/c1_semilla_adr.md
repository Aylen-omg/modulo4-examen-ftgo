# ADR — Estilo Arquitectónico para FTGO
<!-- Salida de Semilla B.3 · Corrida C1 · Evaluación: 7/16 (44 %) -->

**Status:** Accepted
**Date:** 2026-05-24

## Contexto

FTGO está experimentando los problemas clásicos de un monolito creciente: builds lentos, acoplamiento entre equipos y dificultad para escalar componentes de forma independiente. Se necesita decidir el estilo arquitectónico para la migración.

## Opciones Evaluadas

### Opción A — Microservicios desde el día 1

**Pros:**
- Escalabilidad independiente por servicio
- Equipos autónomos por dominio
- Despliegues independientes

**Contras:**
- Alta complejidad operacional desde el inicio
- Requiere infraestructura de orquestación (Kubernetes, service mesh)
- Riesgo alto de fallo si el equipo no tiene experiencia con microservicios

### Opción B — Monolito Modular

**Pros:**
- Menor complejidad operacional
- Refactoring gradual sin cambiar el modelo de despliegue

**Contras:**
- No resuelve los problemas de escalabilidad
- Sigue siendo un monolito desde el punto de vista de despliegue

### Opción C — Strangler Fig (Migración Híbrida Progresiva)

**Pros:**
- Permite migración incremental sin detener el negocio
- Riesgo controlado: cada servicio extraído es validado en producción antes de continuar
- El monolito sigue funcionando mientras se extrae funcionalidad

**Contras:**
- Coexistencia temporal entre monolito y microservicios genera complejidad de integración

## Decisión

Se elige la **Opción C — Strangler Fig** para migrar de forma progresiva el monolito hacia microservicios. Esta decisión permite continuar operando el negocio sin interrupciones durante la migración.

## Consecuencias

**Positivas:**
- Migración sin downtime del negocio
- Validación iterativa de cada microservicio en producción antes de avanzar
- Equipos pueden especializarse por dominio extraído

