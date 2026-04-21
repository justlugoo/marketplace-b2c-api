# Data Model — Marketplace B2C API

Cada microservicio owna su propia base de datos. Ningún servicio accede directamente
a la base de datos de otro — la comunicación es por API REST o eventos Pub/Sub.

---

## PostgreSQL — auth-service (Neon.tech)

### Tabla: `users`

```sql
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name        VARCHAR(255) NOT NULL,
    phone       VARCHAR(50),
    avatar_url  TEXT,
    role        VARCHAR(20) NOT NULL CHECK (role IN ('buyer', 'seller', 'admin')),
    status      VARCHAR(20) NOT NULL DEFAULT 'active'
                CHECK (status IN ('active', 'inactive', 'pending_verification')),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role  ON users(role);
```

### Tabla: `seller_profiles`

```sql
CREATE TABLE seller_profiles (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id          UUID UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    store_name       VARCHAR(255) UNIQUE NOT NULL,
    nit              VARCHAR(50) UNIQUE NOT NULL,
    store_description TEXT,
    verification_status VARCHAR(20) NOT NULL DEFAULT 'pending'
                        CHECK (verification_status IN ('pending', 'approved', 'rejected')),
    rejection_reason TEXT,
    verified_at      TIMESTAMPTZ,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Tabla: `refresh_tokens`

```sql
CREATE TABLE refresh_tokens (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) UNIQUE NOT NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    revoked    BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens(user_id);
```

> **Nota:** El Refresh Token activo también se almacena en Redis (`refresh:{user_id}`)
> para invalidación O(1). PostgreSQL es el registro persistente de auditoría.

---

## PostgreSQL — order-service (Neon.tech, base de datos separada)

### Tabla: `orders`

```sql
CREATE TABLE orders (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    buyer_id                UUID NOT NULL,
    status                  VARCHAR(30) NOT NULL DEFAULT 'pending'
                            CHECK (status IN (
                                'pending', 'paid', 'in_preparation',
                                'shipped', 'delivered', 'cancelled'
                            )),
    total_amount            NUMERIC(12, 2) NOT NULL,
    stripe_payment_intent_id VARCHAR(255) UNIQUE NOT NULL,
    shipping_address        JSONB NOT NULL,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_orders_buyer_id ON orders(buyer_id);
CREATE INDEX idx_orders_status   ON orders(status);
```

### Tabla: `order_items`

```sql
CREATE TABLE order_items (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id     UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id   VARCHAR(255) NOT NULL,  -- ID de MongoDB (catalog-service)
    product_name VARCHAR(255) NOT NULL,  -- Snapshot al momento de la compra
    seller_id    UUID NOT NULL,
    quantity     INTEGER NOT NULL CHECK (quantity > 0),
    unit_price   NUMERIC(12, 2) NOT NULL CHECK (unit_price > 0),
    subtotal     NUMERIC(12, 2) GENERATED ALWAYS AS (quantity * unit_price) STORED
);

CREATE INDEX idx_order_items_order_id  ON order_items(order_id);
CREATE INDEX idx_order_items_seller_id ON order_items(seller_id);
```

> **Decisión de diseño:** `product_name` y `unit_price` se guardan como snapshot.
> Si el vendedor cambia el precio después, el historial de la orden no se altera.

### Tabla: `order_status_history`

```sql
CREATE TABLE order_status_history (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id   UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    status     VARCHAR(30) NOT NULL,
    changed_by UUID NOT NULL,
    notes      TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_order_history_order_id ON order_status_history(order_id);
```

---

## PostgreSQL — support-service (Neon.tech, base de datos separada)

### Tabla: `tickets`

```sql
CREATE TABLE tickets (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID,  -- NULL si el usuario no está autenticado
    email       VARCHAR(255) NOT NULL,
    subject     VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    status      VARCHAR(20) NOT NULL DEFAULT 'open'
                CHECK (status IN ('open', 'in_progress', 'resolved', 'closed')),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tickets_status ON tickets(status);
```

---

## MongoDB — catalog-service (Atlas M0)

### Colección: `categories`

```json
{
  "_id": "ObjectId",
  "name": "Electrónica",
  "slug": "electronica",
  "description": "Dispositivos electrónicos y accesorios",
  "parent_id": null,
  "is_active": true,
  "created_at": "ISODate",
  "updated_at": "ISODate"
}
```

Índices:
```javascript
db.categories.createIndex({ slug: 1 }, { unique: true })
db.categories.createIndex({ is_active: 1 })
```

### Colección: `products`

```json
{
  "_id": "ObjectId",
  "seller_id": "uuid-del-vendedor",
  "category_id": "ObjectId",
  "name": "Laptop Lenovo IdeaPad 3",
  "slug": "laptop-lenovo-ideapad-3",
  "description": "Descripción detallada del producto...",
  "price": 1500000.00,
  "stock": 25,
  "status": "active",
  "images": [
    {
      "url": "https://storage.googleapis.com/marketplace-images/prod-1/img1.jpg",
      "is_primary": true,
      "order": 1
    }
  ],
  "attributes": {
    "ram": "8GB",
    "storage": "512GB SSD",
    "processor": "Intel Core i5",
    "screen_size": "15.6 pulgadas"
  },
  "created_at": "ISODate",
  "updated_at": "ISODate"
}
```

> **Nota:** El campo `attributes` es libre por categoría — electrónica tendrá campos
> distintos a ropa o libros. Este es el motivo central para usar MongoDB aquí.

Índices:
```javascript
db.products.createIndex({ seller_id: 1 })
db.products.createIndex({ category_id: 1, status: 1 })
db.products.createIndex({ price: 1 })
db.products.createIndex({ status: 1 })
db.products.createIndex(
  { name: "text", description: "text" },
  { weights: { name: 10, description: 5 }, name: "text_search_index" }
)
```

---

## Redis — cart-service y auth-service (Upstash)

### Carrito de compras

```
Clave:  cart:{user_id}
Tipo:   Hash
TTL:    7 días (se renueva en cada modificación)

Campos:
  {product_id} → JSON string con:
  {
    "product_id": "64f1a2b3...",
    "name": "Laptop Lenovo IdeaPad 3",
    "unit_price": 1500000.00,
    "quantity": 2,
    "seller_id": "uuid-vendedor",
    "added_at": "2025-07-15T10:30:00Z"
  }

Ejemplo:
  HSET cart:usr-123 prod-abc '{"product_id":"prod-abc","name":"Laptop","unit_price":1500000,"quantity":1,"seller_id":"sel-xyz","added_at":"2025-07-15T10:30:00Z"}'
  EXPIRE cart:usr-123 604800
```

### Refresh Token activo

```
Clave:  refresh:{user_id}
Tipo:   String
TTL:    7 días

Valor:  token_hash (SHA-256 del token)

Ejemplo:
  SET refresh:usr-123 "abc123hashvalue" EX 604800
```

### Rate limiting (gateway-service)

```
Clave:  rate:{ip_address}
Tipo:   String (contador)
TTL:    60 segundos

Ejemplo:
  INCR rate:192.168.1.1
  EXPIRE rate:192.168.1.1 60
```
