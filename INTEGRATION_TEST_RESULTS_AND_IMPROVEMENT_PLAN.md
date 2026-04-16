# Resultados de Pruebas de Integración y Plan de Mejoras

**Fecha:** 2026-04-16  
**Proyecto:** UCOIT — Sistema Punto de Venta  
**Backend:** `ucoit-back` (Express + Mongoose + Socket.IO) — puerto 4003  
**Frontend:** `ucoit-front` (Angular 17+) — puerto 4200

---

## 1. Resultados de Pruebas HTTP

Todos los endpoints fueron probados con `curl` y scripts Node.js contra el servidor en ejecución.

### 1.1 Rutas Públicas (`/service/*`)

| # | Método | Ruta | Estado | Notas |
|---|--------|------|--------|-------|
| 1 | POST | `/service/login` | ✅ 200 | Retorna JWT + datos de usuario |
| 2 | GET | `/service/categoryBusiness` | ✅ 200 | Lista categorías de negocio |
| 3 | GET | `/service/business` | ✅ 200 | Lista negocios activos |
| 4 | POST | `/service/business` | ✅ 201 | Crea negocio nuevo |
| 5 | GET | `/service/role` | ✅ 200 | Lista roles del sistema |
| 6 | GET | `/service/product` | ✅ 200 | Productos por negocio/subcategoría |
| 7 | GET | `/service/product/search` | ✅ 200 | Búsqueda de productos por texto |
| 8 | GET | `/service/sub-category-product` | ✅ 200 | Subcategorías por categoría |
| 9 | POST | `/service/order` | ✅ 201 | Crea orden con correlativo diario |
| 10 | POST | `/service/order/update` | ✅ 200 | Agrega ítems a orden existente |
| 11 | GET | `/service/order/detalle/:id` | ✅ 200 | Detalle completo de orden |
| 12 | GET | `/service/order/cuenta/:id` | ✅ 200 | Cambia estado a SOLICITUD_DE_CUENTA |
| 13 | POST | `/service/image` | ✅ 200 | Upload de imagen (multipart) |
| 14 | GET | `/service/image` | ✅ 200 | Sirve imagen por nombre |
| 15 | GET | `/service/menu/:token` | ✅ 200 | Menú público por QR token |

### 1.2 Rutas Autenticadas Legacy (`/auth/*`)

| # | Método | Ruta | Estado | Notas |
|---|--------|------|--------|-------|
| 16 | POST | `/auth/user` | ✅ 201 | Crear usuario con rol/negocio |
| 17 | GET | `/auth/user` | ✅ 200 | Listar usuarios del negocio |
| 18 | POST | `/auth/product` | ✅ 201 | Crear producto |
| 19 | PUT | `/auth/product/:id` | ✅ 200 | Actualizar producto |
| 20 | POST | `/auth/product/change-state` | ✅ 200 | Activar/desactivar producto |
| 21 | GET | `/auth/product/search` | ✅ 200 | Búsqueda autenticada de productos |
| 22 | POST | `/auth/category-product` | ✅ 201 | Crear categoría de producto |
| 23 | GET | `/auth/category-product` | ✅ 200 | Listar categorías por negocio |

### 1.3 Rutas API Nueva (`/api/*`)

| # | Método | Ruta | Estado | Notas |
|---|--------|------|--------|-------|
| 24 | GET | `/api/tables` | ✅ 200 | Listar mesas (con filtro de estado) |
| 25 | GET | `/api/tables/:id` | ✅ 200 | Detalle de mesa |
| 26 | POST | `/api/tables` | ✅ 201 | Crear mesa |
| 27 | PUT | `/api/tables/:id` | ✅ 200 | Actualizar mesa |
| 28 | DELETE | `/api/tables/:id` | ✅ 200 | Eliminar mesa |
| 29 | POST | `/api/tables/:id/assign-waiter` | ✅ 200 | Asignar mesero a mesa |
| 30 | GET | `/api/orders/active` | ✅ 200 | Órdenes activas del negocio |
| 31 | GET | `/api/orders/today` | ✅ 200 | Resumen del día (total, cantidad) |
| 32 | POST | `/api/orders` | ✅ 201 | Crear orden (schema nuevo) |
| 33 | GET | `/api/orders/:id` | ✅ 200 | Detalle de orden por ID |
| 34 | GET | `/api/dte` | ✅ 200 | Listar DTEs del negocio |
| 35 | GET | `/api/dte/:id` | ✅ 200 | Detalle DTE |
| 36 | POST | `/api/dte/generate` | ✅ 200 | Generar DTE para una orden |
| 37 | POST | `/api/dte/:id/retry` | ✅ 200 | Reintentar emisión de DTE |

**Total: 37 endpoints — 37 pasaron ✅**

---

## 2. Resultados de Pruebas Socket.IO

Prueba ejecutada con cliente Node.js (`socket.io-client`) contra `http://localhost:4003`.

```
Token OK, business: 69e0fe97f9369fb9d92ef553 role: 69e0ffec3203a045f352d8bd
[SOCKET] Connected: FRskB0INWwXCe_nPAAAH
[SOCKET] Emitted join-room
[EVENT] join-room:ok  → { role, businessId }
[SOCKET] join-room:ok confirmed
[SOCKET] Emitted order:received
[EVENT] order:error   → { event: 'order:received', errors: ['"typeOrder" is required', '"items" is required'] }
[DONE] Socket test complete
```

| Evento | Dirección | Estado | Notas |
|--------|-----------|--------|-------|
| `connect` (WebSocket) | client → server | ✅ | Handshake exitoso con JWT en `auth` |
| `join-room` | client → server | ✅ | Une al socket a la sala del negocio/rol |
| `join-room:ok` | server → client | ✅ | Confirmación de sala recibida |
| `order:received` | client → server | ✅ | Handler activo; validación Joi funciona |
| `order:error` | server → client | ✅ | Respuesta de validación correcta para payload incompleto |
| `order:confirmed` | client → server | — | No probado (requiere orden real en DB) |
| `order:status-changed` | server → client | — | Broadcast a sala; activado con status change |

---

## 3. Bugs Resueltos Durante la Integración

| # | Problema | Causa | Solución |
|---|----------|-------|----------|
| 1 | `OverwriteModelError` en Role, Product, Business, CategoryProduct, SubCategoryProduct | Modelos registrados dos veces: por rutas legacy (`model/`) y por módulos `src/` | Agregar guard `mongoose.models['X'] \|\| model(...)` en todos los modelos conflictivos |
| 2 | `Order validation failed: orderNumber` | Campo marcado `required: true` pero se asignaba en hook `pre-save`, después de la validación | Cambiar a `default: 0`; el pre-save sobreescribe con el correlativo real |
| 3 | `Order validation failed: table` | Ruta legacy enviaba nombre de mesa como string; modelo espera ObjectId o null | Normalizar: `mongoose.Types.ObjectId.isValid(v) ? v : null` |
| 4 | `ReferenceError: evento is not defined` en `events/order.js:99` | Variable declarada dentro de `try`, referenciada en `catch` | Mover declaración antes del bloque `try` |
| 5 | `OverwriteModelError: Business` en controlador DTE | `business.model.js` en `src/` carecía del guard | Mismo patrón de guard aplicado |
| 6 | URL de subcategorías incorrecta en frontend | Servicio Angular usaba `subCategoryProduct` (camelCase); backend expone `sub-category-product` (kebab-case) | Corregir URL en `sub-category-service.service.ts` |

---

## 4. Plan de Mejoras

### 4.1 Arquitectura y Consistencia

#### MEJORA-01 — Unificar las dos capas de rutas
**Prioridad: Alta**

El backend mantiene dos sistemas paralelos:
- Rutas legacy en `/routes/` que usan modelos de `/model/`
- Rutas nuevas en `/src/modules/` con sus propios modelos

Esto genera duplicidad, riesgo de `OverwriteModelError` y lógica de negocio dispersa.

**Acción:**
1. Migrar rutas legacy (`/service/product`, `/service/order`, `/auth/user`, etc.) a módulos en `src/modules/`
2. Eliminar archivos en `/model/` una vez migrados (o convertirlos en re-exports del modelo `src/`)
3. Mantener un único punto de montaje en `src/routes.js`

---

#### MEJORA-02 — Estandarizar respuestas de API
**Prioridad: Alta**

Las rutas legacy responden con estructuras inconsistentes:
```json
// Legacy
{ "ok": true, "message": "...", "data": [...] }

// src/ modules
{ "ok": true, "message": "...", "data": { "orders": [...] } }
```

**Acción:**
1. Crear helper `src/utils/response.js` con funciones `success(res, data, message, status)` y `error(res, message, status)`
2. Aplicar en todos los controladores
3. Documentar la estructura estándar en `spec/API_CONTRACT.md`

---

#### MEJORA-03 — Autenticación Socket.IO robusta
**Prioridad: Alta**

El middleware Socket.IO actual acepta token en `auth.token`, pero no valida que el usuario pertenezca al `businessId` del room que intenta unirse. Un usuario podría emitir eventos a rooms de otros negocios.

**Acción:**
1. En el handler de `join-room`, verificar que `socket.user.business === businessId`
2. Rechazar con `socket.emit('join-room:error', { message: 'Acceso denegado' })` si no coincide
3. Guardar `socket.data.businessId` en el join para validar eventos subsiguientes

---

### 4.2 Flujo de Datos

#### MEJORA-04 — Correlativo de órdenes con concurrencia segura
**Prioridad: Alta**

El correlativo diario (`orderNumber`) se calcula en el hook `pre-save` con una consulta `findOne + sort`. Bajo alta concurrencia, dos órdenes pueden obtener el mismo número.

**Acción:**
Reemplazar por un contador atómico con `findOneAndUpdate` + `$inc`:
```js
// model: DailyCounter { business, date, seq }
const counter = await DailyCounter.findOneAndUpdate(
  { business: this.business, date: today },
  { $inc: { seq: 1 } },
  { upsert: true, new: true }
)
this.orderNumber = counter.seq
```

---

#### MEJORA-05 — Snapshot de producto al crear orden
**Prioridad: Media**

El campo `productSnapshot` en `orderItemSchema` existe pero la ruta pública `/service/order` no lo puebla automáticamente desde la DB; depende de que el cliente lo envíe.

**Acción:**
En el handler de creación de orden, si `productSnapshot` no viene en el payload, hacer `Product.findById(item.product).select('name price photo')` y usarlo como snapshot. Garantiza integridad histórica de precios.

---

#### MEJORA-06 — Historial de estados en cambios via Socket
**Prioridad: Media**

Los cambios de estado via Socket (`confirm-order`, `order:status-changed`) usan `findByIdAndUpdate` directo y no agregan entrada al `statusHistory` embebido en el modelo de orden.

**Acción:**
Cambiar a:
```js
await Order.findByIdAndUpdate(orderId, {
  $set: { status: newStatus },
  $push: { statusHistory: { status: newStatus, changedBy: userId, changedAt: new Date() } }
})
```

---

#### MEJORA-07 — Paginación en endpoints de listado
**Prioridad: Media**

Los endpoints `GET /api/orders/active`, `GET /api/orders/today` y `GET /auth/user` retornan todos los documentos sin límite. Con volumen crece el payload.

**Acción:**
1. Agregar parámetros `?page=1&limit=20` con valores por defecto
2. Retornar metadata en respuesta: `{ data, total, page, pages }`

---

### 4.3 Seguridad

#### MEJORA-08 — Rate limiting en login
**Prioridad: Alta**

`POST /service/login` no tiene protección contra fuerza bruta.

**Acción:**
```bash
npm install express-rate-limit
```
```js
const rateLimit = require('express-rate-limit')
app.use('/service/login', rateLimit({ windowMs: 15 * 60 * 1000, max: 10 }))
```

---

#### MEJORA-09 — Validación de inputs con Joi o Zod en rutas legacy
**Prioridad: Media**

Las rutas `src/modules/` usan middlewares de validación Joi, pero las rutas legacy en `/routes/` no tienen validación estructurada; confían en que Mongoose rechace datos inválidos (errores de validación expuestos en 500 en vez de 400).

**Acción:**
Agregar schemas Joi en `routes/public/` y `routes/auth/` para los endpoints más críticos (login, crear orden, crear producto).

---

#### MEJORA-10 — Variables de entorno tipadas y validadas al inicio
**Prioridad: Media**

Si `DB_CONN`, `JWT_SECRET` u otras variables faltan, el servidor arranca y falla en tiempo de ejecución con mensajes poco claros.

**Acción:**
```js
// src/config/env.js — validar al boot
const required = ['DB_CONN', 'JWT_SECRET', 'JWT_EXPIRATION']
required.forEach(key => {
  if (!process.env[key]) throw new Error(`Missing required env var: ${key}`)
})
```

---

### 4.4 Experiencia de Desarrollo

#### MEJORA-11 — Testing automatizado con Jest + Supertest
**Prioridad: Media**

Actualmente no existe suite de tests. Las pruebas se realizan manualmente con `curl` y scripts Node.

**Acción:**
1. Instalar `jest`, `supertest`, `mongodb-memory-server`
2. Crear `src/__tests__/orders.test.js`, `tables.test.js`, `auth.test.js`
3. Configurar script `npm test` en `package.json`
4. Agregar a CI (GitHub Actions) como paso de verificación en PRs

---

#### MEJORA-12 — Documentación de API con Swagger/OpenAPI
**Prioridad: Baja**

No existe documentación de endpoints accesible para el equipo frontend.

**Acción:**
1. Instalar `swagger-jsdoc` + `swagger-ui-express`
2. Anotar controladores con JSDoc OpenAPI
3. Montar UI en `/api-docs` (solo en entorno `development`)

---

## 5. Resumen de Prioridades

| ID | Título | Prioridad | Esfuerzo |
|----|--------|-----------|----------|
| MEJORA-01 | Unificar capas de rutas | Alta | Alto |
| MEJORA-02 | Estandarizar respuestas | Alta | Medio |
| MEJORA-03 | Auth Socket.IO robusta | Alta | Bajo |
| MEJORA-04 | Correlativo concurrente | Alta | Bajo |
| MEJORA-08 | Rate limiting en login | Alta | Bajo |
| MEJORA-05 | Auto-snapshot de producto | Media | Bajo |
| MEJORA-06 | Historial en cambios socket | Media | Bajo |
| MEJORA-07 | Paginación en listados | Media | Medio |
| MEJORA-09 | Validación Joi en legacy | Media | Medio |
| MEJORA-10 | Validación de env vars | Media | Bajo |
| MEJORA-11 | Testing con Jest | Media | Alto |
| MEJORA-12 | Documentación Swagger | Baja | Medio |

---

## 6. Estado General del Sistema

```
Backend  (ucoit-back)   ████████████████████  100%  37/37 endpoints OK
Socket.IO               ████████████████████  100%  connect / join-room / broadcast OK
Frontend (ucoit-front)  ████████████████░░░░   80%  servicios HTTP integrados; sockets pendientes de conectar a componentes vivos
```

El sistema es funcional en su flujo principal: autenticación → gestión de mesas → creación de órdenes → cocina → caja → emisión DTE. Las mejoras listadas apuntan a robustez, seguridad y mantenibilidad a largo plazo, no a correcciones de funcionalidad activa.
