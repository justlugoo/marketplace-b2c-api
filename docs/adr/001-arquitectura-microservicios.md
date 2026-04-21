# ADR 001: Selección de Arquitectura de Microservicios

**Estado:** Aceptado  
**Fecha:** 2026-04-20  

## Contexto
El sistema debe soportar operaciones de un Marketplace B2C con flujos críticos (pagos e inventario) y flujos de alta variabilidad (catálogo de productos). Se requiere escalabilidad independiente para el módulo de búsqueda y el procesamiento de órdenes.

## Opciones Evaluadas
1. **Monolito Modular:** Menor complejidad inicial, pero acoplamiento de despliegue y base de datos única.
2. **Microservicios:** Mayor complejidad de red y despliegue, pero permite poliglotismo de persistencia (PostgreSQL/MongoDB) y escalado granular.

## Decisión
Se elige **Microservicios**. Cada dominio (Auth, Catalog, Orders, Notifications) funcionará como un servicio independiente comunicado de forma asíncrona vía Google Pub/Sub.

## Consecuencias
- **Positivas:** Aislamiento de fallos, escalabilidad horizontal independiente en GCP Cloud Run, optimización de bases de datos por caso de uso.
- **Negativas:** Necesidad de gestionar consistencia eventual, mayor sobrecarga en la configuración de CI/CD y trazabilidad distribuida.
