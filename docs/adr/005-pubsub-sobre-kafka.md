# ADR-005: GCP Pub/Sub sobre Apache Kafka para mensajería asincrónica

- **Estado:** Aceptado
- **Fecha:** 2025-07
- **Servicios afectados:** payment-service, order-service, support-service, notification-service

---

## Contexto

El sistema requiere comunicación asincrónica entre microservicios para desacoplar el procesamiento de pagos, la creación de órdenes y el envío de notificaciones. Se evaluó entre Apache Kafka y GCP Pub/Sub como broker de mensajería.

## Opciones evaluadas

| Opción | Pros | Contras |
|---|---|---|
| Apache Kafka | Alta throughput, replay de mensajes, ecosistema maduro | Requiere gestión de brokers (Zookeeper/KRaft), carga operativa alta para desarrollo individual, costo en cloud |
| GCP Pub/Sub | Serverless, integrado con GCP, sin gestión de infraestructura, free tier generoso (10GB/mes) | Sin replay nativo de mensajes (requiere Pub/Sub Lite), menos control sobre particiones |
| RabbitMQ | Conocido, fácil de configurar localmente | Requiere gestión, no es nativo de GCP |

## Decisión

Se utiliza **GCP Pub/Sub** como broker de mensajería para todos los eventos asincrónicos del sistema.

## Justificación

Para un proyecto de desarrollo individual enfocado en portafolio, la carga operativa de gestionar un cluster Kafka (incluso en Docker) es desproporcionada al beneficio. GCP Pub/Sub es serverless, se integra nativamente con Cloud Run y Cloud Logging, y su free tier (10GB/mes) es suficiente para el volumen del proyecto. La funcionalidad requerida — publicar y consumir eventos — está cubierta completamente por Pub/Sub.

Kafka sería la elección correcta si el sistema procesara millones de eventos por hora o requiriera replay de streams históricos. Ninguno de los dos casos aplica en este MVP.

## Tópicos definidos

| Tópico | Publicador | Suscriptores |
|---|---|---|
| `order-events` | payment-service | order-service, notification-service |
| `order-status-events` | order-service | notification-service |
| `support-events` | support-service | notification-service |

## Consecuencias

- Para desarrollo local se usa el **Pub/Sub Emulator** de GCP (Docker), sin costo ni conexión a GCP real.
- Los mensajes tienen un esquema JSON definido por tópico — documentado en `event-flows.md`.
- Si el sistema escala a producción real con alto volumen, se migraría a Kafka (Confluent Cloud o GKE).
