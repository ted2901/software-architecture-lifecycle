# Esquema GraphQL - API del Sistema

## Introducción

Este documento define el esquema GraphQL completo para la API del sistema. GraphQL permite consultas flexibles y optimizadas, reduciendo el over-fetching y under-fetching de datos.

## URL del Endpoint
```
POST https://api.example.com/graphql
```

## Autenticación
```graphql
Authorization: Bearer {access_token}
```

---

## Tipos de Datos

### 1. Tipo Usuario

```graphql
type User {
  """Identificador único del usuario"""
  id: ID!

  """Correo electrónico único"""
  email: String!

  """Nombre completo"""
  fullName: String!

  """Rol del usuario: usuario, admin, moderador"""
  role: String!

  """URL del avatar del usuario"""
  avatar: String

  """Órdenes del usuario"""
  orders(limit: Int, offset: Int): [Order!]!

  """Reseñas escritas por el usuario"""
  reviews(limit: Int, offset: Int): [Review!]!

  """Fecha de creación"""
  createdAt: DateTime!

  """Fecha de última actualización"""
  updatedAt: DateTime!
}

input UserInput {
  email: String!
  password: String!
  fullName: String!
  role: String
}

input UpdateUserInput {
  fullName: String
  avatar: String
}
```

### 2. Tipo Producto

```graphql
type Product {
  """Identificador único del producto"""
  id: ID!

  """Nombre del producto"""
  name: String!

  """Descripción detallada"""
  description: String

  """Precio del producto"""
  price: Float!

  """Moneda del precio"""
  currency: String!

  """Categoría del producto"""
  category: String!

  """Stock disponible"""
  stock: Int!

  """Etiquetas asociadas"""
  tags: [String!]!

  """Reseñas del producto"""
  reviews(limit: Int, offset: Int): [Review!]!

  """Promedio de calificación"""
  averageRating: Float

  """Número total de reseñas"""
  reviewCount: Int!

  """Fecha de creación"""
  createdAt: DateTime!

  """Fecha de última actualización"""
  updatedAt: DateTime!
}

input ProductInput {
  name: String!
  description: String
  price: Float!
  currency: String!
  category: String!
  stock: Int!
  tags: [String!]
}

input ProductFilterInput {
  category: String
  minPrice: Float
  maxPrice: Float
  tags: [String!]
  search: String
}
```

### 3. Tipo Orden

```graphql
type Order {
  """Identificador único de la orden"""
  id: ID!

  """Usuario propietario de la orden"""
  user: User!

  """Items de la orden"""
  items: [OrderItem!]!

  """Estado de la orden: pending, processing, shipped, delivered, cancelled"""
  status: String!

  """Monto total de la orden"""
  total: Float!

  """Moneda del total"""
  currency: String!

  """Dirección de envío"""
  shippingAddress: Address!

  """Información de seguimiento"""
  tracking: Tracking

  """Método de pago utilizado"""
  paymentMethod: String!

  """Notas adicionales"""
  notes: String

  """Fecha de creación"""
  createdAt: DateTime!

  """Fecha de última actualización"""
  updatedAt: DateTime!
}

type OrderItem {
  """Producto del item"""
  product: Product!

  """Cantidad solicitada"""
  quantity: Int!

  """Precio unitario al momento de compra"""
  price: Float!

  """Subtotal del item"""
  subtotal: Float!
}

input OrderInput {
  items: [OrderItemInput!]!
  shippingAddress: AddressInput!
  paymentMethod: String!
  notes: String
}

input OrderItemInput {
  productId: ID!
  quantity: Int!
}

type Tracking {
  """Número de seguimiento"""
  number: String!

  """Empresa transportista"""
  carrier: String!

  """URL de seguimiento"""
  trackingUrl: String

  """Fecha estimada de entrega"""
  estimatedDelivery: DateTime

  """Eventos de seguimiento"""
  events: [TrackingEvent!]!
}

type TrackingEvent {
  """Descripción del evento"""
  description: String!

  """Ubicación del evento"""
  location: String

  """Fecha del evento"""
  timestamp: DateTime!
}
```

### 4. Tipo Dirección

```graphql
type Address {
  street: String!
  city: String!
  state: String
  postalCode: String!
  country: String!
  isDefault: Boolean!
}

input AddressInput {
  street: String!
  city: String!
  state: String
  postalCode: String!
  country: String!
  isDefault: Boolean
}
```

### 5. Tipo Reseña

```graphql
type Review {
  """Identificador único de la reseña"""
  id: ID!

  """Usuario que escribió la reseña"""
  user: User!

  """Producto reseñado"""
  product: Product!

  """Calificación de 1 a 5"""
  rating: Int!

  """Comentario de la reseña"""
  comment: String!

  """Reacciones a la reseña (útil, no útil)"""
  helpful: Int!

  """Fecha de creación"""
  createdAt: DateTime!
}

input ReviewInput {
  productId: ID!
  rating: Int!
  comment: String!
}
```

### 6. Tipos de Respuesta

```graphql
type UserConnection {
  """Datos de paginación"""
  pageInfo: PageInfo!

  """Lista de usuarios"""
  edges: [UserEdge!]!
}

type UserEdge {
  cursor: String!
  node: User!
}

type ProductConnection {
  pageInfo: PageInfo!
  edges: [ProductEdge!]!
}

type ProductEdge {
  cursor: String!
  node: Product!
}

type OrderConnection {
  pageInfo: PageInfo!
  edges: [OrderEdge!]!
}

type OrderEdge {
  cursor: String!
  node: Order!
}

type PageInfo {
  """Token para la siguiente página"""
  endCursor: String

  """Hay más resultados disponibles"""
  hasNextPage: Boolean!

  """Token para la página anterior"""
  startCursor: String

  """Hay resultados anteriores disponibles"""
  hasPreviousPage: Boolean!

  """Total de items"""
  totalCount: Int!
}

type Error {
  """Código de error"""
  code: String!

  """Mensaje de error"""
  message: String!

  """Detalles adicionales"""
  details: [ErrorDetail!]
}

type ErrorDetail {
  field: String!
  message: String!
}
```

---

## Queries

```graphql
type Query {
  """Obtener usuario autenticado actual"""
  me: User

  """Obtener usuario por ID"""
  user(id: ID!): User

  """Listar usuarios (solo admin)"""
  users(
    first: Int
    after: String
    filter: UserFilterInput
  ): UserConnection!

  """Obtener producto por ID"""
  product(id: ID!): Product

  """Listar productos"""
  products(
    first: Int
    after: String
    filter: ProductFilterInput
  ): ProductConnection!

  """Buscar productos"""
  searchProducts(
    query: String!
    first: Int
    after: String
  ): ProductConnection!

  """Obtener orden por ID"""
  order(id: ID!): Order

  """Listar órdenes del usuario autenticado"""
  myOrders(
    first: Int
    after: String
    status: String
  ): OrderConnection!

  """Listar todas las órdenes (solo admin)"""
  orders(
    first: Int
    after: String
    status: String
    userId: ID
  ): OrderConnection!

  """Obtener reseña por ID"""
  review(id: ID!): Review

  """Listar reseñas de un producto"""
  productReviews(
    productId: ID!
    first: Int
    after: String
  ): [Review!]!
}

input UserFilterInput {
  role: String
  email: String
  search: String
}
```

---

## Mutations

```graphql
type Mutation {
  """Registrar nuevo usuario"""
  register(input: UserInput!): RegisterPayload!

  """Autenticar usuario"""
  login(email: String!, password: String!): LoginPayload!

  """Cerrar sesión"""
  logout: Boolean!

  """Actualizar perfil del usuario"""
  updateUser(input: UpdateUserInput!): User!

  """Crear nuevo producto (solo admin)"""
  createProduct(input: ProductInput!): Product!

  """Actualizar producto existente (solo admin)"""
  updateProduct(id: ID!, input: ProductInput!): Product!

  """Eliminar producto (solo admin)"""
  deleteProduct(id: ID!): Boolean!

  """Crear nueva orden"""
  createOrder(input: OrderInput!): Order!

  """Cancelar orden"""
  cancelOrder(id: ID!): Order!

  """Actualizar estado de orden (solo admin)"""
  updateOrderStatus(id: ID!, status: String!): Order!

  """Crear reseña de producto"""
  createReview(input: ReviewInput!): Review!

  """Actualizar reseña"""
  updateReview(id: ID!, input: ReviewInput!): Review!

  """Eliminar reseña"""
  deleteReview(id: ID!): Boolean!
}

type RegisterPayload {
  user: User
  errors: [Error!]
}

type LoginPayload {
  user: User
  accessToken: String
  refreshToken: String
  expiresIn: Int
  errors: [Error!]
}
```

---

## Subscriptions

```graphql
type Subscription {
  """Suscribirse a cambios de una orden"""
  orderUpdated(orderId: ID!): Order!

  """Suscribirse a nuevas reseñas de un producto"""
  productReviewCreated(productId: ID!): Review!

  """Notificaciones en tiempo real"""
  notificationReceived: Notification!
}

type Notification {
  id: ID!
  type: String!
  title: String!
  message: String!
  data: String
  read: Boolean!
  createdAt: DateTime!
}
```

---

## Escalares Personalizados

```graphql
scalar DateTime
```

---

## Ejemplos de Queries

### Obtener perfil del usuario actual con órdenes

```graphql
query GetMyProfile {
  me {
    id
    email
    fullName
    avatar
    orders(limit: 5) {
      id
      status
      total
      createdAt
    }
  }
}
```

### Buscar productos con filtros

```graphql
query SearchProducts {
  products(
    first: 10
    filter: {
      category: "electronics"
      minPrice: 50
      maxPrice: 200
      tags: ["nuevo"]
    }
  ) {
    pageInfo {
      totalCount
      hasNextPage
    }
    edges {
      node {
        id
        name
        price
        currency
        averageRating
        reviewCount
      }
    }
  }
}
```

### Crear orden

```graphql
mutation CreateOrder {
  createOrder(input: {
    items: [
      { productId: "prod_123", quantity: 2 }
    ]
    shippingAddress: {
      street: "Calle Principal 123"
      city: "Madrid"
      postalCode: "28001"
      country: "España"
    }
    paymentMethod: "credit_card"
  }) {
    id
    status
    total
    items {
      product {
        name
      }
      quantity
      price
    }
  }
}
```

---

**Última actualización:** 2025-02-18
**Autor:** Arquitecto de Software
**Estado:** Vigente
