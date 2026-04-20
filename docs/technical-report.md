# Technical Report — Marketplace B2C API

## 1. Descripción del Proyecto
El objetivo central de este proyecto es el desarrollo de una plataforma de Marketplace B2C diseñada para centralizar la interacción comercial entre proveedores y consumidores finales dentro de un ecosistema digital de alta disponibilidad. La solución busca resolver la fricción en el ciclo de venta digital mediante una arquitectura que garantice la integridad transaccional y una experiencia de usuario fluida, incluso bajo condiciones de alta concurrencia. 

A diferencia de un e-commerce tradicional, este sistema actúa como un orquestador de múltiples catálogos, donde la prioridad técnica es la resiliencia y la escalabilidad granular de sus componentes. El proyecto se fundamenta en la creación de un entorno seguro que automatice el flujo desde el descubrimiento del producto hasta la confirmación del pago y la entrega, permitiendo a los vendedores escalar su oferta sin preocuparse por la infraestructura subyacente.

## 2. Actores y Roles

| Rol          | Descripción |
|--------------|------------|
| Comprador    | Consumidor final del ecosistema. Se encarga del descubrimiento de productos mediante búsqueda avanzada y navegación por categorías. Puede gestionar un carrito de compras dinámico, ejecutar transacciones seguras y hacer seguimiento en tiempo real de sus pedidos. |
| Vendedor     | Proveedor de bienes dentro de la plataforma. Gestiona el inventario y publica productos con atributos diversos. Administra el cumplimiento de órdenes y actualiza estados de envío para informar al comprador y al sistema. |
| Administrador| Operador interno responsable de la gobernanza de la plataforma. Garantiza la consistencia de la taxonomía del catálogo, modera a los actores del sistema y supervisa métricas operativas y transaccionales. |

## 3. Alcance del MVP
El Producto Mínimo Viable (MVP) tiene como meta validar el flujo crítico de intercambio comercial de extremo a extremo ("End-to-End"). El alcance técnico se limita a las funcionalidades esenciales para garantizar una transacción exitosa y segura:

Se prioriza la implementación de una identidad digital robusta con control de acceso, la gestión de un catálogo con esquemas de datos flexibles para soportar diversos tipos de productos y una capa de persistencia efímera para la gestión del carrito de compras. La validación financiera se delegará a una pasarela de pagos externa (Stripe) para asegurar el cumplimiento de normativas de seguridad, mientras que la comunicación post-venta se gestionará de forma asíncrona mediante un bus de eventos distribuido. Se excluyen de esta fase inicial los sistemas de analítica compleja, motores de recomendación y lógicas de logística de terceros.

## 4. Requerimientos
→ Ver gestión completa en GitHub Projects (Issues etiquetados como RF y RNF)
→ Resumen ejecutivo: 8 módulos funcionales, 7 requerimientos no funcionales. Ver [tabla de módulos](./modules.md)

## 5. Historias de Usuario
→ Ver GitHub Projects, vista "Por Tipo", filtro HU
→ Épicas principales: Auth, Catálogo, Checkout, Órdenes, Notificaciones

## 6. Arquitectura de la Solución
→ Ver [architecture.md](./architecture.md) + diagrama en /docs/diagrams/

## 7. Modelo de Datos
→ Ver [data-model.md](./data-model.md)

## 8. Contratos de API
→ Ver [/docs/api/openapi.yaml](./api/openapi.yaml)

## 9. Flujos de Eventos
→ Ver [event-flows.md](./event-flows.md)

## 10. Decisiones Técnicas
→ Ver [/docs/adr/](./adr/)

## 11. Plan de Sprints
→ Ver GitHub Projects, vista "Roadmap"

## 12. Registro de Cambios
→ Ver [CHANGELOG.md](../CHANGELOG.md)
