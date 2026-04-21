# ADR-008: GCP Cloud Run sobre Kubernetes (GKE) para despliegue de microservicios

- **Estado:** Aceptado
- **Fecha:** 2025-07
- **Servicios afectados:** todos los microservicios

---

## Contexto

Los microservicios deben desplegarse en contenedores Docker en GCP. Las dos opciones principales son GCP Cloud Run (serverless) y Google Kubernetes Engine (GKE). Ambas soportan contenedores y escalan horizontalmente.

## Opciones evaluadas

| Criterio | Cloud Run | GKE |
|---|---|---|
| Costo | Free tier generoso (2M req/mes) | Costo fijo del cluster (~$70/mes mínimo) |
| Gestión de infraestructura | Ninguna (serverless) | Alta (nodos, networking, upgrades) |
| Escalado | Automático a 0 | Manual o HPA configurado |
| Cold start | Presente (mitigable con min-instances) | No aplica |
| Apropiado para | MVPs, microservicios stateless, portafolio | Sistemas con estado, alto tráfico, control fino |
| Curva de aprendizaje | Baja | Alta |

## Decisión

Se utiliza **GCP Cloud Run** para el despliegue de todos los microservicios.

## Justificación

Cloud Run es serverless — no requiere gestionar nodos, upgrades ni networking complejo. Cada microservicio se despliega de forma independiente con su propia configuración de CPU, memoria y concurrencia. El free tier cubre 2 millones de requests por mes y 360,000 GB-segundos de CPU, más que suficiente para un portafolio.

GKE sería la elección correcta para un sistema en producción real con tráfico constante, servicios con estado o necesidad de networking avanzado. Ninguno de esos casos aplica en este MVP.

## Configuración por servicio

| Parámetro | Valor desarrollo | Valor producción sugerido |
|---|---|---|
| `min-instances` | 0 (escala a 0) | 1 (evita cold start) |
| `max-instances` | 3 | 10 |
| `memory` | 256Mi | 512Mi |
| `cpu` | 1 | 1 |
| `concurrency` | 80 | 80 |
| `timeout` | 60s | 60s |

## Consecuencias

- El cold start puede añadir latencia en el primer request tras un período de inactividad. En desarrollo es aceptable; en producción se mitiga con `min-instances: 1`.
- Todos los servicios son **stateless** por diseño — requisito de Cloud Run cumplido (el estado vive en las bases de datos y Redis).
- El despliegue se automatiza vía GitHub Actions: push a `main` → build imagen → push a Artifact Registry → deploy a Cloud Run.
- Si el proyecto evoluciona a producción real con tráfico constante, se evalúa migración a GKE o Cloud Run con configuración más agresiva.
