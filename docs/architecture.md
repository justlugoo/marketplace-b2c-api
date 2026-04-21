# Arquitectura de Microservicios: Bounded Contexts

Este documento define los límites de dominio, las entidades bajo propiedad de cada servicio y los protocolos de integración para el ecosistema del proyecto.

## Matriz de Servicios y Dominios

| Servicio | Responsabilidad | Modelos que maneja | Se comunica con |
| :--- | :--- | :--- | :--- |
| **auth-service** | Gestión de identidad, autenticación y control de acceso (RBAC). | `User`, `Role`, `RefreshToken` | Ninguno (Servicio core consultado por terceros). |
| **catalog-service** | Administración del catálogo de productos y categorización. | `Product`, `Category` | `auth-service` (validación de tokens). |
| **cart-service** | Gestión de persistencia temporal del carrito de compras. | `Cart` (Persistencia en Redis) | `catalog-service` (verificación de stock). |
| **order-service** | Orquestación del ciclo de vida y estado de las órdenes. | `Order`, `OrderItem` | `auth-service`, publica eventos en `Pub/Sub`. |
| **payment-service** | Procesamiento de transacciones financieras mediante terceros. | `Payment`, `PaymentIntent` | `Stripe API`, publica eventos en `Pub/Sub`. |
| **notification-service**| Despacho de comunicaciones transaccionales vía email. | — (Sin modelos de negocio propios) | Consume eventos de `Pub/Sub`. |
| **support-service** | Gestión de tickets y asistencia técnica al usuario. | `Ticket` | Publica eventos en `Pub/Sub`. |

## Protocolos y Reglas de Integración

El diseño arquitectónico impone una restricción técnica crítica para garantizar la escalabilidad y el mantenimiento: la **Autonomía de Datos**.
Regla clave: si el servicio A necesita un dato del servicio B, lo pide por API o lo recibe por evento — nunca accede directamente a la base de datos de B.

## Arquitectura 
```
INTERNET
   │
   ▼
PUBLIC-GATEWAY-SERVICE (Cloud Run, público)
   │
   ├──► auth-service (interno)
   │         └──► Neontech PostgreSQL
   │
   ├──► catalog-service (interno)
   │         ├──► MongoDB Atlas
   │         └──► Cloud Storage (imágenes)
   │
   ├──► cart-service (interno)
   │         └──► Upstash Redis
   │
   ├──► order-service (interno)
   │         ├──► Neontech PostgreSQL
   │         ├──publica──► Pub/Sub [order-status-events]
   │         └──consume◄── Pub/Sub [order-events]
   │
   ├──► payment-service (interno)
   │         ├──► Stripe API
   │         └──publica──► Pub/Sub [order-events]
   │
   └──► support-service (interno)
             ├──► Neontech PostgreSQL
             └──publica──► Pub/Sub [support-events]

notification-service (interno, SIN flecha del gateway)
   ├──consume◄── Pub/Sub [order-events]
   ├──consume◄── Pub/Sub [order-status-events]
   ├──consume◄── Pub/Sub [support-events]
   └──► Resend API

── Infraestructura transversal (nota en el diagrama) ──
Todos los servicios → Cloud Logging
Todos los servicios → Secret Manager (al iniciar)
GitHub Actions → Artifact Registry → Cloud Run (CI/CD)
```
