# ADR-0001 — Estilo Arquitectónico para FTGO
<!-- Salida de Semilla B.3 · Corrida C3 · Evaluación: 9/16 (56 %) -->

**Status:** Accepted
**Deciders:** CTO, Arquitecto Principal, Tech Lead Backend
**Date:** 2026-05-24

## Contexto

FTGO opera un monolito Java/Spring Boot WAR que presenta síntomas de "monolithic hell": builds de 30+ minutos, imposibilidad de escalar componentes independientemente y acoplamiento entre equipos. Se requiere una decisión sobre el estilo arquitectónico de destino y la estrategia de transición. La migración debe completarse en 18-24 meses sin interrumpir las operaciones del negocio.

## Opciones Evaluadas

### Opción A — Microservicios desde Cero (Big Bang)

Reescribir la plataforma completa en microservicios, congelando el desarrollo de nuevas funcionalidades durante la migración.

**Pros:**
- Arquitectura limpia sin deuda técnica del monolito
- Diseño óptimo de cada servicio sin restricciones del código legacy

**Contras:**
- Período de 6-12 meses sin entregas de valor al negocio
- Alto riesgo: si algo falla, no hay fallback
- El equipo debe aprender microservicios mientras migra toda la plataforma simultáneamente

### Opción B — Monolito Modular

Refactorizar el monolito existente separando módulos con interfaces bien definidas. Sin cambio en el modelo de despliegue.

**Pros:**
- Menor riesgo operacional
- Reutilización del código existente
- Curva de aprendizaje baja

**Contras:**
- No habilita escalabilidad independiente por componente
- No resuelve el problema de despliegues acoplados
- Es una solución temporal que pospone el problema

### Opción C — Strangler Fig (Migración Híbrida Progresiva)

Extraer funcionalidades del monolito gradualmente hacia microservicios, manteniendo el monolito operativo como fuente de verdad durante la transición. Cada servicio extraído es validado en producción antes de continuar.

**Pros:**
- Migración sin downtime del negocio durante las 18-24 meses de transición
- Riesgo controlado: rollback a monolito si el servicio extraído falla
- El equipo aprende microservicios de forma incremental con cada extracción

**Contras:**
- Complejidad operacional durante el período de coexistencia
- Requiere sincronización de datos entre monolito y servicios extraídos

## Decisión

Se elige la **Opción C — Strangler Fig** por ser la única que cumple el requisito de 18-24 meses de migración sin interrumpir operaciones del negocio.

## Consecuencias

**Positivas:**
- La plataforma continúa operando y generando revenue durante toda la migración
- Cada microservicio extraído puede escalar independientemente
- Validación iterativa reduce el riesgo de fallos catastróficos

**Negativas:**
- Durante el período de transición existe complejidad adicional de integración entre monolito y microservicios

## Follow-ups

- Definir estrategia de gestión de datos durante la coexistencia (datos duplicados vs. sincronización)
- Documentar el orden de extracción de servicios (prioridad por impacto de negocio)

