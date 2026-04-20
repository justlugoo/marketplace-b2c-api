## 1. Descripción del Proyecto
El objetivo central de este proyecto es el desarrollo de una plataforma de Marketplace B2C diseñada para centralizar la interacción comercial entre proveedores y consumidores finales dentro de un ecosistema digital de alta disponibilidad. La solución busca resolver la fricción en el ciclo de venta digital mediante una arquitectura que garantice la integridad transaccional y una experiencia de usuario fluida, incluso bajo condiciones de alta concurrencia. 

A diferencia de un e-commerce tradicional, este sistema actúa como un orquestador de múltiples catálogos, donde la prioridad técnica es la resiliencia y la escalabilidad granular de sus componentes. El proyecto se fundamenta en la creación de un entorno seguro que automatice el flujo desde el descubrimiento del producto hasta la confirmación del pago y la entrega, permitiendo a los vendedores escalar su oferta sin preocuparse por la infraestructura subyacente.

## 2. Actores y Roles
* **Comprador:** Es el consumidor final del ecosistema. Su rol principal es el descubrimiento de productos a través de búsqueda avanzada y navegación por categorías. Tiene la capacidad de gestionar un carrito de compras dinámico, ejecutar transacciones financieras seguras y realizar el seguimiento en tiempo real del ciclo de vida de sus pedidos.

* **Vendedor:** Actúa como el proveedor de bienes dentro de la plataforma. Su responsabilidad es la gestión del inventario y la publicación de productos con atributos heterogéneos. Cuenta con facultades para administrar el cumplimiento de las órdenes recibidas, actualizando los estados de envío para informar al comprador y al sistema central.

* **Administrador:** Es el operador interno encargado de la gobernanza de la plataforma. Su función es garantizar la consistencia de la taxonomía del catálogo (categorías), la moderación de los actores del sistema y la supervisión global de las métricas operativas y transaccionales para asegurar el correcto funcionamiento del marketplace.

## 3. Alcance del MVP
El Producto Mínimo Viable (MVP) tiene como meta validar el flujo crítico de intercambio comercial de extremo a extremo ("End-to-End"). El alcance técnico se limita a las funcionalidades esenciales para garantizar una transacción exitosa y segura:

Se prioriza la implementación de una identidad digital robusta con control de acceso, la gestión de un catálogo con esquemas de datos flexibles para soportar diversos tipos de productos y una capa de persistencia efímera para la gestión del carrito de compras. La validación financiera se delegará a una pasarela de pagos externa (Stripe) para asegurar el cumplimiento de normativas de seguridad, mientras que la comunicación post-venta se gestionará de forma asíncrona mediante un bus de eventos distribuido. Se excluyen de esta fase inicial los sistemas de analítica compleja, motores de recomendación y lógicas de logística de terceros.

## 4. Requerimientos funcionales por modulo

### RF-01 · Auth & Usuarios

- RF-01.1 Registro de comprador con email y contraseña  
- RF-01.2 Registro de vendedor con datos adicionales (nombre de tienda, RUT/NIT)  
- RF-01.3 Login con JWT + Refresh Token  
- RF-01.4 Perfil editable por cada actor  
- RF-01.5 Roles diferenciados con control de acceso por endpoint  

---

### RF-02 · Catálogo de Productos

- RF-02.1 Vendedor puede crear, editar y eliminar productos  
- RF-02.2 Producto contiene: nombre, descripción, precio, stock, categoría, imágenes  
- RF-02.3 Imágenes almacenadas en GCP Cloud Storage  
- RF-02.4 Categorías administradas por el Admin  
- RF-02.5 Producto tiene estados: borrador, activo, pausado  

---

### RF-03 · Búsqueda y Navegación

- RF-03.1 Búsqueda por texto sobre nombre y descripción del producto  
- RF-03.2 Filtro por categoría, precio mínimo y máximo, relevancia  
- RF-03.3 Resultados paginados  

---

### RF-04 · Carrito y Checkout

- RF-04.1 Comprador puede agregar/eliminar productos del carrito  
- RF-04.2 Carrito persiste en sesión (Redis)  
- RF-04.3 Resumen de orden antes de confirmar  
- RF-04.4 Integración con Stripe para procesamiento de pago  
- RF-04.5 Orden se crea solo si el pago es exitoso  

---

### RF-05 · Gestión de Órdenes

- RF-05.1 Orden tiene estados: pendiente → pagado → en_preparacion → enviado → entregado  
- RF-05.2 Comprador puede ver historial de órdenes  
- RF-05.3 Vendedor puede ver órdenes de sus productos y actualizar estado  
- RF-05.4 Admin puede ver todas las órdenes  

---

### RF-06 · Panel de Vendedor

- RF-06.1 Vista de productos propios con stock actual  
- RF-06.2 Vista de órdenes recibidas con detalle  
- RF-06.3 Actualización de inventario  

---

### RF-07 · Notificaciones

- RF-07.1 Email automático al comprador: confirmación de pago, cambio de estado de orden  
- RF-07.2 Email automático al vendedor: nueva orden recibida  
- RF-07.3 Notificaciones disparadas por eventos asincrónicos (Kafka)  

---

### RF-08 · Soporte Básico

- RF-08.1 Formulario de contacto que genera ticket simple  
- RF-08.2 Notificación por email al Admin al recibir ticket  

---
## Requerimientos no funcionales

| ID      | Requerimiento                                               | Métrica objetivo                                      |
|---------|-------------------------------------------------------------|--------------------------------------------------------|
| RNF-01  | Tiempo de respuesta en endpoints críticos                   | < 300ms p95                                            |
| RNF-02  | Disponibilidad del sistema                                  | 99.5% uptime                                           |
| RNF-03  | Autenticación segura                                        | JWT con expiración + Refresh Token rotativo            |
| RNF-04  | Escalabilidad horizontal                                    | Microservicios desplegables en Cloud Run               |
| RNF-05  | Trazabilidad                                                | Logs estructurados con correlation ID por request      |
| RNF-06  | Seguridad en pagos                                          | Stripe maneja datos de tarjeta                         |
| RNF-07  | Contenerización                                             | Servicios corren en Docker                             |
