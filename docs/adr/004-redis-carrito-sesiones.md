# ADR-004: Redis para carrito de compras y gestión de sesiones

- **Estado:** Aceptado
- **Fecha:** 2025-07
- **Servicios afectados:** cart-service, auth-service

---

## Contexto

El carrito de compras es una estructura temporal que se lee y escribe con alta frecuencia durante la sesión de un usuario. Los Refresh Tokens de autenticación también necesitan almacenamiento con expiración automática (TTL) y capacidad de invalidación inmediata. Ambos casos requieren un almacén en memoria con TTL nativo, no una base de datos relacional.

## Opciones evaluadas

| Opción | Pros | Contras |
|---|---|---|
| Redis | In-memory, TTL nativo, O(1) en operaciones clave, estructura Hash ideal para carrito | Datos volátiles (aceptable para carrito y tokens) |
| PostgreSQL | Persistencia, ACID | Overkill para datos temporales, latencia innecesaria |
| Sessiones en JWT | Sin estado del servidor | No permite invalidación de tokens sin blacklist |

## Decisión

Se utiliza **Upstash Redis** (free tier: 10,000 comandos/día) para cart-service y para almacenamiento de Refresh Tokens en auth-service.

## Justificación

El carrito no es un dato crítico de negocio — si se pierde, el usuario lo reconstruye. Lo que sí importa es la velocidad de lectura/escritura en cada interacción. Redis resuelve esto con latencia sub-milisegundo. Para los Refresh Tokens, Redis permite invalidación inmediata con `DEL` y expiración automática con `EXPIRE`, sin queries adicionales.

## Consecuencias

**Esquema de claves:**
- Carrito: `cart:{user_id}` → Hash con `{product_id: quantity}`
- Refresh Token: `refresh:{user_id}` → String con el token, TTL 7 días

- El free tier de Upstash (10,000 comandos/día) es suficiente para desarrollo y portafolio.
- En producción real se migraría a GCP Memorystore (Redis).
- El cart-service es stateless — el estado vive en Redis, no en el servicio.
