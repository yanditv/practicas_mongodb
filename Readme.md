# Laboratorio: Sistema de Análisis de Ventas Internacionales en MongoDB

---

## Objetivo

Construir un sistema completo de gestión y análisis de ventas internacionales usando MongoDB, aplicando desde modelado de datos hasta operaciones avanzadas como transacciones, agregaciones, índices, replicación y sharding.

---

## Laboratorio

### 1. Configurar el entorno

- Instala MongoDB en tu máquina local.
- Crea una base de datos llamada `salesDB`:

```javascript
use salesDB
```

### 2. Modelar y Crear las Colecciones

- Crea las siguientes colecciones:
  - `customers`
  - `products`
  - `orders`
  - `shipments`

No es necesario crear las colecciones manualmente; MongoDB las creará al insertar el primer documento.

### 3. Insertar Datos de Prueba

- Genera e inserta:
  - N documentos en `customers`
  - N documentos en `products`
  - N documentos en `orders`
  - N documentos en `shipments`

**Ejemplo de inserción:**

```javascript
db.customers.insertMany([
  {
    customer_id: "CUST12345",
    name: "John Doe",
    email: "johndoe@example.com",
    address: {
      street: "123 Main St",
      city: "New York",
      country: "USA",
    },
    registered_at: ISODate("2023-01-15T00:00:00Z"),
  },
]);
```

### 4. Crear Consultas Básicas y Agregaciones

- Top 5 clientes que más gastaron:

```javascript
db.orders.aggregate([
  { $group: { _id: "$customer_id", totalSpent: { $sum: "$total_amount" } } },
  { $sort: { totalSpent: -1 } },
  { $limit: 5 },
]);
```

- Productos más vendidos por categoría:

```javascript
db.orders.aggregate([
  { $unwind: "$items" },
  {
    $lookup: {
      from: "products",
      localField: "items.product_id",
      foreignField: "product_id",
      as: "product",
    },
  },
  { $unwind: "$product" },
  {
    $group: {
      _id: "$product.category",
      totalSold: { $sum: "$items.quantity" },
    },
  },
  { $sort: { totalSold: -1 } },
]);
```

- Ventas por mes:

```javascript
db.orders.aggregate([
  { $match: { order_date: { $gte: ISODate("2023-01-01T00:00:00Z") } } },
  {
    $group: {
      _id: { $dateToString: { format: "%Y-%m", date: "$order_date" } },
      totalSales: { $sum: "$total_amount" },
    },
  },
  { $sort: { _id: 1 } },
]);
```

### 5. Crear Índices para Mejorar el Rendimiento

- Crear un índice compuesto en `orders`:

```javascript
db.orders.createIndex({ customer_id: 1, order_date: -1 });
```

- Crear un índice de texto en `products`:

```javascript
db.products.createIndex({ name: "text" });
```

### 6. Realizar una Transacción Simulada

Simula una compra:

```javascript
const session = db.getMongo().startSession();
session.startTransaction();

try {
  db.products.updateOne(
    { product_id: "PROD001" },
    { $inc: { stock: -2 } },
    { session }
  );

  db.orders.insertOne(
    {
      order_id: "ORD999",
      customer_id: "CUST12345",
      order_date: new Date(),
      items: [{ product_id: "PROD001", quantity: 2 }],
      total_amount: 1799.98,
    },
    { session }
  );

  db.shipments.insertOne(
    {
      shipment_id: "SHIP999",
      order_id: "ORD999",
      status: "Pending",
      shipment_date: null,
    },
    { session }
  );

  session.commitTransaction();
} catch (e) {
  session.abortTransaction();
  throw e;
} finally {
  session.endSession();
}
```

### 7. Configurar Replica Set

- Inicia 3 instancias `mongod`:

```bash
mongod --replSet rs0 --port 27017 --dbpath /data/rs0/ --bind_ip localhost
mongod --replSet rs0 --port 27018 --dbpath /data/rs1/ --bind_ip localhost
mongod --replSet rs0 --port 27019 --dbpath /data/rs2/ --bind_ip localhost
```

- Inicializa el Replica Set:

```javascript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "localhost:27017" },
    { _id: 1, host: "localhost:27018" },
    { _id: 2, host: "localhost:27019" },
  ],
});
```

### 8. Configurar Sharding

- Inicia 2 instancias para shards, 1 config server, y 1 mongos:

```bash
mongod --shardsvr --port 27020 --dbpath /data/shard0/ --bind_ip localhost
mongod --shardsvr --port 27021 --dbpath /data/shard1/ --bind_ip localhost
mongod --configsvr --port 27019 --dbpath /data/config/ --bind_ip localhost
mongos --configdb localhost:27019 --bind_ip localhost --port 27017
```

- Habilita el sharding en la base:

```javascript
sh.enableSharding("salesDB");
sh.shardCollection("salesDB.orders", { order_id: "hashed" });
```

---
