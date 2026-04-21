# Event Flows — GCP Pub/Sub

El sistema usa tres tópicos de GCP Pub/Sub para comunicación asincrónica entre microservicios.
Ningún servicio llama directamente a otro para operaciones que no requieren respuesta inmediata.

---

## Tópicos y suscripciones

| Tópico | Publicador | Suscriptores |
|---|---|---|
| `order-events` | payment-service | order-service, notification-service |
| `order-status-events` | order-service | notification-service |
| `support-events` | support-service | notification-service |

---

## Tópico: `order-events`

### Evento: `payment.succeeded`

Publicado por `payment-service` cuando Stripe confirma el pago vía webhook.

**Payload:**
```json
{
  "event_type": "payment.succeeded",
  "event_id": "uuid-unico-del-evento",
  "published_at": "2025-07-15T14:30:00Z",
  "data": {
    "stripe_payment_intent_id": "pi_3Nk...",
    "buyer_id": "uuid-del-comprador",
    "amount": 3000000.00,
    "currency": "COP",
    "cart_snapshot": [
      {
        "product_id": "64f1a2b3...",
        "product_name": "Laptop Lenovo IdeaPad 3",
        "seller_id": "uuid-vendedor",
        "quantity": 2,
        "unit_price": 1500000.00
      }
    ],
    "shipping_address": {
      "street": "Calle 50 #30-20",
      "city": "Medellín",
      "department": "Antioquia",
      "postal_code": "050001"
    }
  }
}
```

**Flujo:**
```
Stripe Webhook
      │
      ▼
payment-service valida firma Stripe-Signature
      │
      ▼
payment-service publica → Pub/Sub [order-events]
      │
      ├──► order-service consume
      │         └── Crea orden en PostgreSQL con status 'paid'
      │         └── Vacía carrito del buyer en Redis
      │
      └──► notification-service consume
                └── Envía email de confirmación al comprador
                └── Envía email de nueva orden a cada vendedor involucrado
```

### Evento: `payment.failed`

Publicado por `payment-service` cuando Stripe reporta fallo de pago.

**Payload:**
```json
{
  "event_type": "payment.failed",
  "event_id": "uuid-unico-del-evento",
  "published_at": "2025-07-15T14:31:00Z",
  "data": {
    "stripe_payment_intent_id": "pi_3Nk...",
    "buyer_id": "uuid-del-comprador",
    "failure_reason": "insufficient_funds"
  }
}
```

**Flujo:**
```
payment-service publica → Pub/Sub [order-events]
      │
      └──► notification-service consume
                └── Envía email al comprador notificando el fallo
                    (el carrito no se modifica)
```

---

## Tópico: `order-status-events`

### Evento: `order.status_updated`

Publicado por `order-service` cada vez que cambia el estado de una orden.

**Payload:**
```json
{
  "event_type": "order.status_updated",
  "event_id": "uuid-unico-del-evento",
  "published_at": "2025-07-15T16:00:00Z",
  "data": {
    "order_id": "uuid-de-la-orden",
    "buyer_id": "uuid-del-comprador",
    "previous_status": "paid",
    "new_status": "in_preparation",
    "changed_by": "uuid-del-vendedor",
    "order_items": [
      {
        "product_name": "Laptop Lenovo IdeaPad 3",
        "quantity": 2,
        "unit_price": 1500000.00
      }
    ]
  }
}
```

**Flujo:**
```
Vendedor llama PATCH /seller/orders/{id}/status
      │
      ▼
order-service actualiza estado en PostgreSQL
      │
      ▼
order-service registra en order_status_history
      │
      ▼
order-service publica → Pub/Sub [order-status-events]
      │
      └──► notification-service consume
                └── Envía email al comprador con el nuevo estado
```

---

## Tópico: `support-events`

### Evento: `support.ticket_created`

Publicado por `support-service` cuando se crea un nuevo ticket.

**Payload:**
```json
{
  "event_type": "support.ticket_created",
  "event_id": "uuid-unico-del-evento",
  "published_at": "2025-07-15T17:00:00Z",
  "data": {
    "ticket_id": "uuid-del-ticket",
    "user_email": "comprador@email.com",
    "subject": "No recibí mi pedido",
    "description": "Descripción del problema...",
    "created_at": "2025-07-15T17:00:00Z"
  }
}
```

**Flujo:**
```
Usuario llama POST /support/tickets
      │
      ▼
support-service crea ticket en PostgreSQL
      │
      ▼
support-service publica → Pub/Sub [support-events]
      │
      └──► notification-service consume
                └── Envía email al Admin con los datos del ticket
                └── Envía email de confirmación al usuario (número de ticket)
```

---

## Garantías y manejo de errores

### Idempotencia
Cada evento tiene un `event_id` único (UUID). Los consumidores deben verificar si el
evento ya fue procesado antes de actuar — especialmente `order-service` al crear órdenes,
para evitar órdenes duplicadas ante reentregas de Pub/Sub.

```python
# Ejemplo de verificación de idempotencia en order-service
async def handle_payment_succeeded(event: dict):
    event_id = event["event_id"]
    payment_intent_id = event["data"]["stripe_payment_intent_id"]

    # Verificar si ya existe una orden para este PaymentIntent
    existing = await db.orders.find_one_by_payment_intent(payment_intent_id)
    if existing:
        logger.info(f"Orden ya existe para PaymentIntent {payment_intent_id}, ignorando")
        return  # Acknowledge sin crear duplicado
    
    await create_order(event["data"])
```

### Reintento ante fallos
Si un consumidor falla al procesar un mensaje (excepción no controlada), **no debe hacer
acknowledge**. Pub/Sub reentregará el mensaje según la política de reintento configurada
(por defecto: backoff exponencial hasta 7 días, máximo 5 reintentos).

### Dead Letter Topic
En producción se configura un Dead Letter Topic por suscripción para mensajes que
fallan repetidamente, permitiendo inspección manual.

---

## Configuración local (desarrollo)

Para desarrollo local se usa el **Pub/Sub Emulator** oficial de GCP:

```yaml
# docker-compose.yml (fragmento)
pubsub-emulator:
  image: gcr.io/google.com/cloudsdktool/cloud-sdk:latest
  command: gcloud beta emulators pubsub start --host-port=0.0.0.0:8085
  ports:
    - "8085:8085"
  environment:
    - PUBSUB_PROJECT_ID=marketplace-local
```

Variable de entorno en cada servicio:
```
PUBSUB_EMULATOR_HOST=localhost:8085
```
