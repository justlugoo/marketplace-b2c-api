# ADR-009: Selección de servicios externos gratuitos como alternativa a servicios GCP de pago

- **Estado:** Aceptado
- **Fecha:** 2025-07
- **Servicios afectados:** todos

---

## Contexto

Varios servicios gestionados de GCP ideales para producción (Cloud SQL, Memorystore, Cloud Load Balancing) tienen costo mensual fijo que no se justifica para un proyecto de portafolio de desarrollo individual. Se requieren alternativas gratuitas y permanentes que sean representativas de la arquitectura real.

## Decisión y justificación por servicio

| Necesidad | Servicio GCP ideal | Alternativa gratuita elegida | Justificación |
|---|---|---|---|
| PostgreSQL gestionado | Cloud SQL | **Neon.tech** | PostgreSQL serverless, free tier permanente, branching de DB para tests |
| Redis gestionado | Memorystore | **Upstash Redis** | Free tier permanente (10k comandos/día), API compatible con Redis estándar |
| MongoDB | — | **MongoDB Atlas M0** | 512MB gratis permanente, mismo motor que producción |
| Envío de emails | — | **Resend.com** | 3,000 emails/mes gratis, API simple, SDK Python disponible |
| Broker de mensajería local | — | **Pub/Sub Emulator (Docker)** | Emulador oficial de GCP, sin costo, idéntico al Pub/Sub real |

## Servicios GCP que sí se usan (dentro del free tier)

| Servicio | Free tier | Uso en el proyecto |
|---|---|---|
| Cloud Run | 2M requests/mes | Despliegue de todos los microservicios |
| Cloud Storage | 5GB/mes | Imágenes de productos |
| Pub/Sub | 10GB/mes | Mensajería entre servicios |
| Artifact Registry | 0.5GB/mes | Imágenes Docker |
| Cloud Logging | 50GB/mes | Logs centralizados |
| Secret Manager | 6 versiones activas | Credenciales y API keys |

## Consecuencias

- La arquitectura local (Docker Compose) usa Pub/Sub Emulator, PostgreSQL en Docker y Redis en Docker para desarrollo sin costo.
- Los servicios externos (Neon, Upstash, Atlas) se usan solo en el entorno de staging/producción en GCP.
- En un entorno de producción real, Neon se reemplazaría por Cloud SQL y Upstash por Memorystore — el cambio es solo de connection string, sin cambio de código.
- Esta decisión está documentada explícitamente para que cualquier revisor del proyecto entienda el razonamiento, no lo interprete como desconocimiento de los servicios gestionados de GCP.
