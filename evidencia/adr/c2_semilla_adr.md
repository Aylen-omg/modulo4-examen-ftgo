# ADR — Decisión de Arquitectura: Estilo para FTGO
<!-- Salida de Semilla B.3 · Corrida C2 · Evaluación: 6/16 (38 %) -->

**Status:** Accepted
**Date:** 2026-05-24

## Contexto

FTGO necesita migrar su monolito Java WAR a una arquitectura más escalable. Se evalúan tres opciones de estilo arquitectónico.

## Opciones

### Opción 1 — Microservicios completos desde cero

Reescritura completa de la plataforma usando microservicios. Cada dominio de negocio se convierte en un servicio independiente con su propia base de datos.

Se requiere parar el desarrollo de nuevas funcionalidades por varios meses para completar la migración.

**Pros:** Mayor agilidad a largo plazo, equipos completamente autónomos.
**Contras:** Alto riesgo, tiempo de entrega largo, sin valor al negocio durante el período de transición.

### Opción 2 — Monolito Modular

Refactoring del monolito actual para separar en módulos con interfaces bien definidas, sin cambiar el modelo de despliegue.

**Pros:** Menor riesgo, reutilización del código existente.
**Contras:** No escala más allá del servidor monolítico, no habilita despliegues independientes.

### Opción 3 — Migración Incremental

Extraer gradualmente funcionalidades del monolito hacia microservicios, usando el monolito como punto de partida y fuente de verdad durante la transición.

**Pros:** Riesgo controlado, valor continuo al negocio, curva de aprendizaje gradual del equipo.
**Contras:** Período de coexistencia entre monolito y microservicios requiere coordinación.

## Decisión

Se selecciona la **Opción 3 — Migración Incremental** por ser la estrategia que mejor balancea riesgo y velocidad de entrega de valor.

## Consecuencias Positivas

- La plataforma continúa operando y generando revenue durante la migración
- El equipo adquiere experiencia con microservicios de forma progresiva
- Los fallos quedan contenidos en el servicio extraído, no afectan al monolito restante

