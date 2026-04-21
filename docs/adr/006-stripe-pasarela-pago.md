# ADR-006: Stripe como pasarela de pago

- **Estado:** Aceptado
- **Fecha:** 2025-07
- **Servicios afectados:** payment-service

---

## Contexto

El sistema requiere procesamiento de pagos con tarjeta de crédito/débito de forma segura. Se debe cumplir con el estándar PCI-DSS, que exige que los datos de tarjeta nunca pasen por la infraestructura propia.

## Opciones evaluadas

| Opción | Pros | Contras |
|---|---|---|
| Stripe | SDK Python maduro, modo test gratuito, webhooks robustos, documentación excelente, ampliamente usado en industria | Comisión por transacción en producción (2.9% + $0.30) |
| PayPal | Reconocido por usuarios | API más compleja, SDK menos moderno |
| MercadoPago | Común en Latinoamérica | Documentación menos completa, SDK menos maduro |
| Wompi | Nativo Colombia | Ecosistema pequeño, poco reconocido internacionalmente |

## Decisión

Se utiliza **Stripe** con modo test (sandbox) para el MVP.

## Justificación

Stripe es el estándar de la industria para proyectos backend modernos. Su modelo de PaymentIntents desacopla la creación del intento de pago de la confirmación — el frontend maneja los datos de tarjeta directamente con Stripe (mediante Stripe Elements o Stripe.js), y el backend solo recibe la confirmación vía webhook. Esto garantiza cumplimiento PCI-DSS sin esfuerzo adicional.

El modo test de Stripe es completamente gratuito e ilimitado, lo que lo hace ideal para desarrollo y portafolio.

## Flujo de pago implementado

```
1. Frontend solicita → payment-service: POST /checkout/payment-intent
2. payment-service crea PaymentIntent en Stripe → retorna client_secret al frontend
3. Frontend confirma pago directamente con Stripe (datos de tarjeta nunca tocan el backend)
4. Stripe envía webhook → payment-service: POST /webhooks/stripe
5. payment-service valida firma Stripe-Signature
6. Si payment_intent.succeeded → publica evento en Pub/Sub [order-events]
```

## Consecuencias

- Los datos de tarjeta nunca pasan por los servicios propios — cumplimiento PCI-DSS garantizado.
- Se requiere configurar el webhook de Stripe apuntando a la URL del payment-service en Cloud Run.
- La clave del webhook (`STRIPE_WEBHOOK_SECRET`) se almacena en GCP Secret Manager.
- En producción real se activaría la cuenta de Stripe y se aplicarían comisiones por transacción.
