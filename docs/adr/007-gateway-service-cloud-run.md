# ADR-007: API Gateway implementado como microservicio en Cloud Run

- **Estado:** Aceptado
- **Fecha:** 2025-07
- **Servicios afectados:** gateway-service (todos los servicios internos)

---

## Contexto

En una arquitectura de microservicios, los clientes no deben comunicarse directamente con cada servicio. Se requiere un punto de entrada único que centralice el enrutamiento, la autenticación y el logging. Las opciones nativas de GCP para esto (Cloud Load Balancing + Cloud Endpoints) tienen costo. Se requiere una solución funcional, segura y gratuita.

## Opciones evaluadas

| Opción | Pros | Contras |
|---|---|---|
| Cloud Load Balancing + Cloud Endpoints | Solución nativa GCP, escalable | Costo mensual fijo, overkill para MVP |
| Kong Gateway | Potente, open source | Requiere infraestructura adicional (no serverless) |
| gateway-service en Cloud Run (FastAPI + httpx) | Gratuito, serverless, control total, patrón válido en industria | Mantenimiento manual del enrutamiento |
| Exposición directa de cada servicio | Simplicidad | Sin punto de control centralizado, URLs dispersas, no es una API coherente |

## Decisión

Se implementa un **gateway-service** en FastAPI desplegado en Cloud Run como único punto de entrada público. Todos los demás servicios se despliegan como **internal** (sin URL pública).

## Justificación

El patrón de gateway-as-a-service es ampliamente usado en producción por equipos pequeños y startups que no justifican el costo de un API Gateway gestionado. FastAPI con `httpx` permite enrutar requests de forma asincrónica con latencia mínima. Al ser el único servicio con URL pública, los servicios internos quedan protegidos de acceso directo desde internet — equivalente funcional a una red privada.

## Responsabilidades del gateway-service

1. **Enrutamiento:** redirige cada request al servicio interno correspondiente según el path
2. **Validación de JWT:** verifica el token antes de enrutar — los servicios internos confían en el header `X-User-Id` y `X-User-Role` inyectado por el gateway
3. **Rate limiting:** implementado con `slowapi` (librería Python) para prevenir abuso
4. **Logging centralizado:** cada request entrante se loguea con correlation ID antes de enrutar
5. **Health check agregado:** `GET /health` consulta el health de todos los servicios internos

## Tabla de enrutamiento

| Path externo | Servicio interno |
|---|---|
| `/auth/**` | auth-service |
| `/products/**` | catalog-service |
| `/categories/**` | catalog-service |
| `/cart/**` | cart-service |
| `/orders/**` | order-service |
| `/checkout/**` | payment-service |
| `/seller/**` | order-service / catalog-service |
| `/support/**` | support-service |
| `/webhooks/stripe` | payment-service (sin validación JWT) |

## Consecuencias

- El gateway es un **single point of failure** — en producción real se mitigaría con múltiples instancias y Cloud Load Balancing.
- El enrutamiento es manual — si se agregan servicios, el gateway debe actualizarse.
- Esta decisión se revisa si el proyecto escala a producción real con tráfico significativo.
- En producción real se migraría a Cloud Endpoints o Kong sobre GKE.
