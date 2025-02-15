# Diseño de Endpoints REST - API del Sistema

## Resumen Ejecutivo
Este documento define los endpoints REST para la API principal del sistema de arquitectura de software. La API utiliza principios RESTful, soporta paginación, filtrado y fuerza versioning mediante URL.

## Base URL
```
https://api.example.com/v1
```

## Autenticación
Todos los endpoints (excepto `/auth/login` y `/auth/register`) requieren Bearer Token en el header `Authorization`:
```
Authorization: Bearer {access_token}
```

---

## 1. Autenticación

### 1.1 Registro de Usuario
**Endpoint:** `POST /auth/register`

**Request:**
```json
{
  "email": "usuario@example.com",
  "password": "securePassword123",
  "fullName": "Juan Pérez",
  "role": "usuario"
}
```

**Response (201 Created):**
```json
{
  "id": "usr_123456",
  "email": "usuario@example.com",
  "fullName": "Juan Pérez",
  "role": "usuario",
  "createdAt": "2025-02-18T10:30:00Z"
}
```

### 1.2 Login
**Endpoint:** `POST /auth/login`

**Request:**
```json
{
  "email": "usuario@example.com",
  "password": "securePassword123"
}
```

**Response (200 OK):**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "rt_abcd1234",
  "expiresIn": 3600,
  "user": {
    "id": "usr_123456",
    "email": "usuario@example.com",
    "fullName": "Juan Pérez",
    "role": "usuario"
  }
}
```

**Códigos de Error:**
- `401 Unauthorized`: Credenciales inválidas
- `400 Bad Request`: Campos requeridos faltantes

---

## 2. Gestión de Usuarios

### 2.1 Obtener Perfil del Usuario
**Endpoint:** `GET /users/me`

**Response (200 OK):**
```json
{
  "id": "usr_123456",
  "email": "usuario@example.com",
  "fullName": "Juan Pérez",
  "role": "usuario",
  "avatar": "https://cdn.example.com/avatars/usr_123456.jpg",
  "createdAt": "2025-02-18T10:30:00Z",
  "updatedAt": "2025-02-18T14:45:00Z"
}
```

### 2.2 Actualizar Perfil
**Endpoint:** `PUT /users/me`

**Request:**
```json
{
  "fullName": "Juan Carlos Pérez",
  "avatar": "data:image/jpeg;base64,..."
}
```

**Response (200 OK):**
- Retorna objeto usuario actualizado

**Códigos de Error:**
- `400 Bad Request`: Datos inválidos
- `401 Unauthorized`: No autenticado

### 2.3 Listar Usuarios (Admin)
**Endpoint:** `GET /users?page=1&limit=20&role=usuario&search=juan`

**Parámetros de Query:**
- `page` (opcional): Número de página (default: 1)
- `limit` (opcional): Resultados por página (default: 20, máx: 100)
- `role` (opcional): Filtrar por rol
- `search` (opcional): Búsqueda por nombre o email

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": "usr_123456",
      "email": "usuario@example.com",
      "fullName": "Juan Pérez",
      "role": "usuario",
      "createdAt": "2025-02-18T10:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8
  }
}
```

---

## 3. Gestión de Productos

### 3.1 Crear Producto
**Endpoint:** `POST /products`

**Headers Requeridos:**
- `Authorization: Bearer {token}` (solo para rol admin)

**Request:**
```json
{
  "name": "Producto A",
  "description": "Descripción del producto",
  "price": 99.99,
  "currency": "USD",
  "category": "electronics",
  "stock": 50,
  "tags": ["nuevo", "oferta"]
}
```

**Response (201 Created):**
```json
{
  "id": "prod_789abc",
  "name": "Producto A",
  "description": "Descripción del producto",
  "price": 99.99,
  "currency": "USD",
  "category": "electronics",
  "stock": 50,
  "tags": ["nuevo", "oferta"],
  "createdAt": "2025-02-18T10:30:00Z"
}
```

### 3.2 Obtener Producto
**Endpoint:** `GET /products/{productId}`

**Response (200 OK):**
```json
{
  "id": "prod_789abc",
  "name": "Producto A",
  "description": "Descripción del producto",
  "price": 99.99,
  "currency": "USD",
  "category": "electronics",
  "stock": 50,
  "tags": ["nuevo", "oferta"],
  "reviews": [
    {
      "userId": "usr_123456",
      "rating": 5,
      "comment": "Excelente producto",
      "createdAt": "2025-02-17T15:20:00Z"
    }
  ],
  "createdAt": "2025-02-18T10:30:00Z",
  "updatedAt": "2025-02-18T14:45:00Z"
}
```

### 3.3 Listar Productos
**Endpoint:** `GET /products?category=electronics&minPrice=50&maxPrice=200&page=1&limit=20`

**Parámetros de Query:**
- `category` (opcional): Filtrar por categoría
- `minPrice` (opcional): Precio mínimo
- `maxPrice` (opcional): Precio máximo
- `tags` (opcional): Filtrar por tags
- `page` (opcional): Número de página
- `limit` (opcional): Resultados por página
- `sort` (opcional): Campo para ordenar (name, price, createdAt)

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": "prod_789abc",
      "name": "Producto A",
      "price": 99.99,
      "category": "electronics",
      "stock": 50
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 500,
    "totalPages": 25
  }
}
```

---

## 4. Gestión de Órdenes

### 4.1 Crear Orden
**Endpoint:** `POST /orders`

**Request:**
```json
{
  "items": [
    {
      "productId": "prod_789abc",
      "quantity": 2,
      "price": 99.99
    }
  ],
  "shippingAddress": {
    "street": "Calle Principal 123",
    "city": "Madrid",
    "postalCode": "28001",
    "country": "España"
  },
  "paymentMethod": "credit_card"
}
```

**Response (201 Created):**
```json
{
  "id": "ord_456def",
  "userId": "usr_123456",
  "status": "pending",
  "items": [
    {
      "productId": "prod_789abc",
      "quantity": 2,
      "price": 99.99
    }
  ],
  "total": 199.98,
  "currency": "USD",
  "shippingAddress": {...},
  "createdAt": "2025-02-18T10:30:00Z"
}
```

### 4.2 Obtener Orden
**Endpoint:** `GET /orders/{orderId}`

**Response (200 OK):**
```json
{
  "id": "ord_456def",
  "userId": "usr_123456",
  "status": "shipped",
  "items": [...],
  "total": 199.98,
  "currency": "USD",
  "tracking": {
    "number": "TRACK123456",
    "carrier": "DHL",
    "estimatedDelivery": "2025-02-25T23:59:59Z"
  },
  "createdAt": "2025-02-18T10:30:00Z"
}
```

### 4.3 Listar Órdenes del Usuario
**Endpoint:** `GET /users/me/orders?status=shipped&page=1&limit=20`

**Parámetros de Query:**
- `status` (opcional): pending, processing, shipped, delivered, cancelled
- `page` (opcional): Número de página
- `limit` (opcional): Resultados por página

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": "ord_456def",
      "status": "shipped",
      "total": 199.98,
      "currency": "USD",
      "createdAt": "2025-02-18T10:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 45,
    "totalPages": 3
  }
}
```

---

## 5. Códigos de Estado HTTP

| Código | Descripción |
|--------|-------------|
| 200 | OK - Solicitud exitosa |
| 201 | Created - Recurso creado exitosamente |
| 204 | No Content - Solicitud exitosa sin contenido |
| 400 | Bad Request - Solicitud inválida |
| 401 | Unauthorized - No autenticado |
| 403 | Forbidden - No autorizado |
| 404 | Not Found - Recurso no encontrado |
| 409 | Conflict - Conflicto con estado actual |
| 422 | Unprocessable Entity - Validación fallida |
| 429 | Too Many Requests - Límite de rate exceeded |
| 500 | Internal Server Error - Error del servidor |

---

## 6. Formatos de Error

**Formato estándar de error (400, 401, 403, 404, 422):**
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validación fallida en los campos",
    "details": [
      {
        "field": "email",
        "message": "Email inválido"
      },
      {
        "field": "password",
        "message": "Contraseña muy corta"
      }
    ]
  }
}
```

---

## 7. Rate Limiting

- Límite: 1000 solicitudes por hora por token
- Headers de respuesta:
  - `X-RateLimit-Limit: 1000`
  - `X-RateLimit-Remaining: 999`
  - `X-RateLimit-Reset: 1645194000`

---

## 8. CORS

Orígenes permitidos:
- `https://example.com`
- `https://app.example.com`

---

## 9. Versionamiento

La API utiliza versionamiento por URL:
- v1: `https://api.example.com/v1`
- v2: `https://api.example.com/v2` (futuro)

---

**Última actualización:** 2025-02-18
**Autor:** Arquitecto de Software
**Estado:** Vigente
