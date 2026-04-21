# ADR-002: PostgreSQL como base de datos para datos transaccionales

- **Estado:** Aceptado
- **Fecha:** 2025-07
- **Servicios afectados:** auth-service, order-service, support-service

---

## Contexto

El sistema maneja entidades con relaciones estrictas y operaciones que requieren garantías ACID: usuarios con roles, órdenes con ítems, pagos y tickets de soporte. Estas entidades tienen esquemas bien definidos que no cambian frecuentemente y exigen integridad referencial.

## Opciones evaluadas

| Opción | Pros | Contras |
|---|---|---|
| PostgreSQL | ACID completo, relaciones, madurez | Esquema rígido, escala vertical |
| MongoDB para todo | Esquema flexible, un solo motor | Sin joins nativos, sin transacciones multi-documento confiables para finanzas |
| MySQL | Amplio soporte | Menos features que PostgreSQL (JSON, full-text) |

## Decisión

Se utiliza **PostgreSQL** (vía Neon.tech en free tier) para los servicios que manejan datos transaccionales: autenticación, órdenes y soporte.

## Justificación

Las transacciones financieras y el ciclo de vida de órdenes requieren ACID estricto. Una orden no puede crearse si el pago no se confirma — esto es una transacción atómica. PostgreSQL garantiza esto de forma nativa. MongoDB lo haría con transacciones multi-documento, pero agrega complejidad innecesaria cuando el esquema es fijo.

## Consecuencias

- Los tres servicios comparten el motor pero tienen **bases de datos separadas** dentro de la misma instancia Neon (aislamiento lógico por microservicio).
- No se permiten joins entre bases de datos de distintos servicios — la comunicación es por API o eventos.
- En producción real se migraría a Cloud SQL (PostgreSQL) de GCP.
