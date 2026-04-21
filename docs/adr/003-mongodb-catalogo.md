# ADR-003: MongoDB como base de datos para el catálogo de productos

- **Estado:** Aceptado
- **Fecha:** 2025-07
- **Servicios afectados:** catalog-service

---

## Contexto

El catálogo de un marketplace B2C maneja productos de categorías heterogéneas. Un producto de electrónica tiene atributos como voltaje, conectividad y garantía. Un producto de ropa tiene talla, material y color. Forzar estos atributos en un esquema relacional rígido requeriría tablas de atributos dinámicos (EAV) que son difíciles de consultar y mantener.

## Opciones evaluadas

| Opción | Pros | Contras |
|---|---|---|
| MongoDB | Esquema flexible por documento, búsqueda de texto nativa | Sin joins, consistencia eventual |
| PostgreSQL con JSONB | ACID + flexibilidad | Queries complejos sobre JSONB, menos natural |
| Elasticsearch | Búsqueda avanzada | Overkill para MVP, complejidad operativa alta |

## Decisión

Se utiliza **MongoDB Atlas** (tier M0 gratuito, 512MB) para el catalog-service.

## Justificación

El esquema de un producto varía significativamente por categoría. MongoDB permite que cada documento tenga los campos que corresponden a su categoría sin afectar a los demás. El índice `$text` de MongoDB cubre la búsqueda por nombre y descripción definida en RF-03.1 sin necesidad de Elasticsearch en el MVP.

## Consecuencias

- Los atributos específicos por categoría se almacenan como subdocumento `attributes` de esquema libre.
- La búsqueda full-text usa índice `$text` de MongoDB — suficiente para MVP.
- Si el volumen de búsquedas crece, se migra a MongoDB Atlas Search o Elasticsearch sin cambiar el modelo de datos base.
- El tier M0 de Atlas tiene 512MB de límite — suficiente para un portafolio con datos de prueba.
