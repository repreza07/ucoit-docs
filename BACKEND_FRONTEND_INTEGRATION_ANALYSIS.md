# Análisis de Integración Backend ↔ Frontend
**Fecha:** 2026-04-16  
**Proyectos:** `ucoit-back` / `ucoit-front`

---

## 1. Inventario completo de endpoints del Backend

El backend tiene **dos capas de rutas**:

### 1.1 Rutas Legacy (`/service` y `/auth`)

| Método | Ruta | Protección | Descripción |
|--------|------|-----------|-------------|
| POST | `/service/login` | Pública | Login (legacy) |
| GET | `/service/categoryBusiness` | Pública | Listar categorías de negocio |
| POST | `/service/categoryBusiness` | Pública | Crear categoría de negocio |
| GET | `/service/business` | Pública | Listar negocios |
| POST | `/service/business` | Pública | Registrar negocio |
| GET | `/service/role` | Pública | Listar roles |
| POST | `/service/role` | Pública | Crear rol |
| GET | `/auth/category-product` | JWT | Listar categorías de producto |
| POST | `/auth/category-product` | JWT | Crear categoría de producto |
| GET | `/auth/sub-category-product` | JWT | Listar sub-categorías |
| POST | `/auth/sub-category-product` | JWT | Crear sub-categoría |
| GET | `/auth/product` | JWT | Listar productos |
| POST | `/auth/product` | JWT | Crear producto |
| GET | `/auth/user` | JWT | Listar usuarios |
| POST | `/auth/user` | JWT | Crear usuario |
| POST | `/auth/user/validate` | JWT | Validar email de usuario |
| GET | `/auth/business-private` | JWT | Datos privados del negocio |

### 1.2 Rutas Nuevas — Sistema `src/` (`/api` y `/service/qr`)

| Método | Ruta | Protección | Descripción |
|--------|------|-----------|-------------|
| POST | `/api/auth` | Pública | Login (nuevo sistema) |
| GET | `/service/qr/:token` | Pública | Lookup de mesa por QR |
| GET | `/api/tables` | JWT + Tenant | Listar mesas (filtro por status) |
| GET | `/api/tables/:id` | JWT + Tenant | Detalle de una mesa |
| POST | `/api/tables` | JWT + Tenant | Crear mesa |
| PATCH | `/api/tables/:id` | JWT + Tenant | Actualizar mesa |
| DELETE | `/api/tables/:id` | JWT + Tenant | Eliminar mesa |
| PATCH | `/api/tables/:id/assign-waiter` | JWT + Tenant | Asignar mesero a mesa |
| GET | `/api/orders` | JWT + Tenant | Listar órdenes con filtros |
| GET | `/api/orders/active` | JWT + Tenant | Órdenes activas |
| GET | `/api/orders/today` | JWT + Tenant | Resumen del día |
| GET | `/api/orders/:id` | JWT + Tenant | Detalle de una orden |
| POST | `/api/dte/generate` | JWT + Tenant | Generar DTE (FCF/CCF/NC/ND) |
| POST | `/api/dte/:id/retry` | JWT + Tenant | Reintentar emisión de DTE |
| GET | `/api/dte` | JWT + Tenant | Listar DTEs con filtros |
| GET | `/api/dte/:id` | JWT + Tenant | Detalle de un DTE |

### 1.3 Endpoints referenciados en servicios frontend que NO aparecen en rutas definidas

| Endpoint | Servicio Frontend | Estado |
|----------|------------------|--------|
| `GET /service/product` | `ProductsService.getProductsPublic()` | ⚠️ No encontrado en rutas legacy ni en `src/routes.js` |
| `POST /service/product/search` | `ProductsService.searchProductsPublic()` | ⚠️ No encontrado en rutas |
| `POST /auth/product/change-state` | `ProductsService.changeStateToProduct()` | ⚠️ No encontrado en rutas |
| `GET /auth/product/search` | `ProductsService.searchProducts()` | ⚠️ No encontrado en rutas |
| `POST /service/order` | `OrderService.saveOrder()` | ⚠️ No en rutas (puede ser legacy en events/socket) |
| `POST /service/order/update` | `OrderService.AddOrderExtra()` | ⚠️ No encontrado en rutas |
| `GET /service/order/detalle/:id` | `OrderService.getOrder()` | ⚠️ No encontrado en rutas |
| `GET /service/order/cuenta/:id` | `OrderService.solicitarCuenta()` | ⚠️ No encontrado en rutas |
| `POST /service/image` | `ImageServiceService.uploadImage()` | ⚠️ No encontrado en rutas |
| `GET /service/image` | `ImageServiceService.getImage()` | ⚠️ No encontrado en rutas |
| `GET /service/business/data` | `BusinessService.getDataPublic()` | ⚠️ No encontrado en rutas |
| `GET /service/subCategoryProduct` | `SubCategoryServiceService.getSubCategories()` | ⚠️ URL inconsistente (back usa `/service/sub-category-product` con guión) |
| `GET /service/menu/:token` | `CustomerMenuComponent` (inline HTTP) | ⚠️ No encontrado en rutas |

---

## 2. Inventario de Servicios Frontend

### 2.1 Servicios existentes y su mapeo al backend

| Servicio | Archivo | Endpoints que consume |
|----------|---------|----------------------|
| `AuthService` | `services/auth.service.ts` | `POST /service/login`, `POST /service/business` |
| `CategoryBusinessService` | `services/category-business.service.ts` | `GET /service/categoryBusiness` |
| `BusinessService` | `pages/admin/services/business.service.ts` | `GET /auth/business-private`, `GET /service/business/data` (⚠️ no existe) |
| `CategoryProductService` | `pages/admin/services/category-product.service.ts` | `GET/POST /auth/category-product`, `GET/POST /auth/sub-category-product` |
| `ProductsService` | `pages/admin/services/products.service.ts` | `GET /auth/product`, `POST /auth/product`, + 4 rutas no mapeadas |
| `UserService` | `pages/admin/services/user.service.ts` | `GET /auth/user`, `POST /auth/user` |
| `RoleserviceService` | `pages/admin/services/roleservice.service.ts` | `GET /service/role` |
| `SubCategoryServiceService` | `pages/admin/services/sub-category-service.service.ts` | `GET /service/subCategoryProduct` (⚠️ URL incorrecta) |
| `OrderService` | `pages/admin/services/order.service.ts` | 4 rutas legacy no mapeadas en routes |
| `ImageServiceService` | `pages/admin/services/image-service.service.ts` | 2 rutas no mapeadas |
| `SocketService` | `core/services/socket.service.ts` | Todos los eventos WS nuevos — ✅ correctamente mapeados |
| `ActionsroleServiceService` | `pages/admin/services/actionsrole-service.service.ts` | Sin HTTP — lógica local de roles |
| `MenuService` | `pages/admin/services/menu.service.ts` | Sin HTTP — configuración estática de menú |
| `WebSocketService` | `pages/admin/services/web.socket.service.ts` | Socket.IO legacy — ⚠️ duplica funcionalidad de `SocketService` |

### 2.2 Módulos de rol (nuevos — `src/app/modules/`)

| Módulo | Ruta Angular | Estado de integración con backend |
|--------|-------------|-----------------------------------|
| `AdminModule` | `/backoffice` | ⚠️ Solo usa SocketService; **no llama `/api/tables` ni `/api/orders`** |
| `CashierModule` | `/cajero` | ⚠️ Solo usa SocketService; **no llama `/api/dte` ni `/api/orders`** |
| `KitchenModule` | `/cocina` | ⚠️ Solo usa SocketService; sin llamadas HTTP |
| `WaiterModule` | `/mesero` | ⚠️ Solo usa SocketService; **no llama `/api/tables`** |
| `CustomerModule` | `/cliente` | ⚠️ Llama `GET /service/menu/:token` — **endpoint no existe en backend** |

---

## 3. Análisis de brechas

### 3.1 Endpoints backend sin ningún consumidor en el frontend

| Endpoint | Módulo Backend | Impacto |
|----------|--------------|---------|
| `GET /api/tables` | `tables` | Alto — WaiterModule y AdminModule necesitan cargar mesas al inicio |
| `GET /api/tables/:id` | `tables` | Medio |
| `POST /api/tables` | `tables` | Alto — Admin debe poder crear mesas |
| `PATCH /api/tables/:id` | `tables` | Medio |
| `DELETE /api/tables/:id` | `tables` | Medio |
| `PATCH /api/tables/:id/assign-waiter` | `tables` | Alto — flujo de asignación de mesero |
| `GET /api/orders` | `orders` | Alto — AdminModule y CashierModule necesitan histórico |
| `GET /api/orders/active` | `orders` | Alto — carga inicial de WaiterModule y KitchenModule |
| `GET /api/orders/today` | `orders` | Medio — dashboard resumen |
| `GET /api/orders/:id` | `orders` | Medio |
| `POST /api/dte/generate` | `dte` | Alto — CashierModule tiene el formulario pero no llama al endpoint |
| `POST /api/dte/:id/retry` | `dte` | Medio |
| `GET /api/dte` | `dte` | Medio |
| `GET /api/dte/:id` | `dte` | Medio |
| `POST /api/auth` (nuevo login) | `auth src/` | Bajo — el login legacy sigue funcionando |
| `GET /service/qr/:token` | `tables public` | Alto — CustomerModule llama a `/service/menu/:token` en vez de `/service/qr/:token` |
| `POST /service/role` | `role legacy` | Bajo — solo necesario en configuración inicial |
| `POST /service/categoryBusiness` | `category_business legacy` | Bajo |
| `POST /auth/user/validate` | `user legacy` | Bajo |

### 3.2 Llamadas frontend a endpoints inexistentes o con URL incorrecta

| Llamada Frontend | Problema | Prioridad |
|-----------------|---------|----------|
| `GET /service/menu/:token` | Endpoint no existe; el back tiene `/service/qr/:token` que devuelve mesa, no menú completo | 🔴 Crítico |
| `GET /service/product` (público) | Ruta no registrada en ningún router | 🔴 Crítico |
| `GET /service/subCategoryProduct` | URL inconsistente; back usa kebab-case `/service/sub-category-product` (y además no está en el router público) | 🔴 Crítico |
| `POST /service/order` | Puede estar en controller legacy no conectado al router | 🔴 Crítico |
| `GET /service/order/detalle/:id` | No encontrado en ninguna ruta | 🔴 Crítico |
| `GET /service/order/cuenta/:id` | No encontrado en ninguna ruta | 🔴 Crítico |
| `POST /service/order/update` | No encontrado en ninguna ruta | 🟠 Alto |
| `POST /service/image` | No encontrado en ninguna ruta | 🟠 Alto |
| `GET /service/image` | No encontrado en ninguna ruta | 🟠 Alto |
| `GET /service/business/data` | No encontrado en rutas públicas | 🟠 Alto |
| `POST /auth/product/change-state` | No encontrado en rutas protegidas | 🟠 Alto |
| `GET /auth/product/search` | No encontrado en rutas protegidas | 🟡 Medio |
| `POST /service/product/search` (público) | No encontrado en rutas públicas | 🟡 Medio |

### 3.3 Componentes sin servicios de datos HTTP

Los módulos de rol nuevos (`/backoffice`, `/cajero`, `/cocina`, `/mesero`) usan **exclusivamente Socket.IO** para recibir actualizaciones en tiempo real, pero **no realizan carga inicial de datos** (HTTP GET) al montar los componentes. Esto significa que si el usuario entra directamente a `/mesero/tables`, verá la pantalla vacía hasta que llegue un evento de socket.

---

## 4. Plan de Acción

### Fase 1 — Correcciones críticas (⚠️ bloquean funcionalidad actual) — Prioridad Alta

#### 4.1 Crear servicio `TablesService` en el frontend

**Archivo sugerido:** `ucoit-front/src/app/core/services/tables.service.ts`

Debe consumir:
- `GET /api/tables?status=<estado>` — carga inicial
- `GET /api/tables/:id` — detalle
- `POST /api/tables` — crear
- `PATCH /api/tables/:id` — actualizar
- `DELETE /api/tables/:id` — eliminar
- `PATCH /api/tables/:id/assign-waiter` — asignar mesero

**Componentes que deben inyectarlo:**
- `WaiterTablesComponent` — llamar `getTables()` en `ngOnInit`
- `AdminDashboardComponent` — llamar `getTables()` en `ngOnInit`

#### 4.2 Crear servicio `OrdersApiService` en el frontend

**Archivo sugerido:** `ucoit-front/src/app/core/services/orders-api.service.ts`

Debe consumir:
- `GET /api/orders/active` — carga inicial de comandas y órdenes activas
- `GET /api/orders/today` — resumen del dashboard
- `GET /api/orders` — histórico con filtros
- `GET /api/orders/:id` — detalle de orden

**Componentes que deben inyectarlo:**
- `CommandsComponent` (Kitchen) — llamar `getActiveOrders()` en `ngOnInit`
- `CashierOrdersComponent` — llamar `getOrders({ status: 'SOLICITUD_DE_CUENTA' })` en `ngOnInit`
- `AdminDashboardComponent` — llamar `getTodaySummary()` en `ngOnInit`

#### 4.3 Conectar `DteComponent` (Cajero) con el endpoint real

El componente tiene el formulario pero no realiza llamada HTTP. Debe:
- Crear o inyectar un `DteService` que consuma:
  - `POST /api/dte/generate` — al enviar el formulario
  - `GET /api/dte` — para listar DTEs emitidos
  - `POST /api/dte/:id/retry` — para reintentar

#### 4.4 Corregir `CustomerModule` — endpoint de menú por QR

El componente `CustomerMenuComponent` llama `GET /service/menu/:token`, pero el backend expone `GET /service/qr/:token` que devuelve los datos de la mesa (no el menú).

**Opciones:**
- **Opción A (recomendada):** Crear endpoint `GET /service/menu/:token` en el backend que devuelva `{ categories, products }` usando el token de la mesa.
- **Opción B:** Hacer que el frontend combine `GET /service/qr/:token` + `GET /service/product?q=<business>` para construir el menú.

#### 4.5 Registrar rutas legacy faltantes o verificar su existencia

Los siguientes endpoints son consumidos por el frontend pero no aparecen en los archivos de rutas revisados. Verificar si existen en archivos no mapeados o crearlos:

| Endpoint | Acción |
|----------|--------|
| `GET /service/product` | Verificar si existe en otro archivo de routes; si no, crear ruta pública en `routes/public/product.js` |
| `POST /service/order` | Verificar controlador `controller/order.js`; conectar al router público o migrar a socket handler |
| `GET /service/order/detalle/:id` | Idem |
| `GET /service/order/cuenta/:id` | Idem |
| `POST /service/order/update` | Idem |
| `POST /service/image` | Crear módulo `image` con multer o equivalente |
| `GET /service/image` | Idem |
| `GET /service/business/data` | Crear GET con query `?q=<businessSlug>` en router público |
| `POST /auth/product/change-state` | Agregar al router `routes/auth/product.js` |
| `GET /auth/product/search` | Agregar al router `routes/auth/product.js` |

#### 4.6 Corregir URL inconsistente de `SubCategoryServiceService`

```typescript
// Actual (incorrecto):
`${PATH}/service/subCategoryProduct`

// Correcto (según convención del backend):
`${PATH}/service/sub-category-product`
```
Además, verificar que esta ruta esté registrada en el router público (`routes/routes.js`).

---

### Fase 2 — Integración de carga inicial HTTP en módulos de rol — Prioridad Media

Una vez creados `TablesService` y `OrdersApiService` (Fase 1), actualizar cada componente de rol:

| Componente | Acción |
|-----------|--------|
| `WaiterTablesComponent` | `ngOnInit` → `tablesService.getTables()` para carga inicial antes de eventos socket |
| `CommandsComponent` | `ngOnInit` → `ordersApi.getActiveOrders()` para mostrar comandas existentes |
| `CashierOrdersComponent` | `ngOnInit` → `ordersApi.getOrders({ status: 'SOLICITUD_DE_CUENTA,LISTO_PARA_LLEVAR' })` |
| `AdminDashboardComponent` | `ngOnInit` → `tablesService.getTables()` + `ordersApi.getTodaySummary()` |

---

### Fase 3 — Limpieza y consolidación — Prioridad Baja

| Tarea | Detalle |
|-------|---------|
| Deprecar `WebSocketService` legacy | `pages/admin/services/web.socket.service.ts` duplica funcionalidad de `core/services/socket.service.ts`. Migrar los componentes del módulo legacy `/admin` al nuevo `SocketService`. |
| Vaciar `CategoryServiceService` | El servicio existe pero no tiene lógica. Eliminar o implementar. |
| Consolidar doble login | Hay dos endpoints de login: `POST /service/login` (legacy) y `POST /api/auth` (nuevo). El `AuthService` usa el legacy. Evaluar migración al nuevo. |
| Módulo `users` en `src/` no registrado | `src/modules/users/` tiene rutas pero no están registradas en `src/routes.js`. Revisar si deben exponerse bajo `/api/users`. |

---

## 5. Resumen ejecutivo

| Categoría | Cantidad |
|-----------|---------|
| Endpoints backend totales | 33 |
| Endpoints consumidos correctamente por el frontend | 10 |
| Endpoints backend sin consumidor frontend | 16 |
| Llamadas frontend a rutas inexistentes/incorrectas | 13 |
| Servicios frontend sin lógica HTTP implementada | 2 (CategoryServiceService, WebSocketService-nuevo) |
| Módulos de rol sin carga HTTP inicial | 4 (Admin, Cajero, Cocina, Mesero) |

**Conclusión:** La integración de Socket.IO está correctamente implementada en ambos lados. La mayor brecha está en las llamadas HTTP REST: los módulos de rol nuevos no realizan carga inicial de datos, y existen numerosas llamadas a endpoints que no están registrados en ningún router del backend. La prioridad debe ser crear `TablesService` y `OrdersApiService` en el frontend, y registrar/crear los endpoints de productos, órdenes, imágenes y menú público en el backend.
