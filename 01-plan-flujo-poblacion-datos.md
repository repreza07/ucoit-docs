# Plan de Acción — Flujo de Población de Datos (Negocios, Categorías y Productos)

**Fecha:** 2026-04-16  
**Alcance:** Módulo Admin del frontend Angular (`ucoit-front`) comparado contra los controllers y modelos del backend (`ucoit-back`).

---

## 1. Diagnóstico General

El flujo de población de datos tiene **tres etapas** que deben funcionar en orden:

```
Registro de Negocio → Categorías de Producto → Sub-Categorías → Productos
```

Actualmente existen **inconsistencias entre front y back** en las tres etapas, y campos inoperativos que confunden el flujo.

---

## 2. Análisis por Módulo

---

### 2.1 Registro de Negocio (`/business-confirm-save`)

#### Estado actual
- El componente `business-confirm-save` es solo una pantalla de confirmación post-registro; no existe formulario de creación de negocio dentro de `/admin`.
- El backend (`controller/business.js → saveBusiness`) espera el siguiente payload:
  ```json
  {
    "business": { "name": "...", "email": "...", "phone": "...", ... },
    "user":     { "email": "...", "password": "...", ... }
  }
  ```
- Al guardar, el backend **crea automáticamente** los 4 roles del sistema (Admin, Cajero, Mesero, Cocinero) y una categoría de negocio por defecto ("Restaurante") si no existe ninguna. El admin no necesita crearlos.

#### Problemas detectados
| # | Problema | Impacto |
|---|----------|---------|
| 1 | El email hardcodeado `mailmail.com` en el HTML de confirmación no usa el dato real del usuario registrado | Visual / confianza |
| 2 | No existe un formulario de registro de negocio accesible desde el área admin para crear negocios adicionales | Funcional |
| 3 | El modelo `Business` incluye `address` (objeto con `street`, `city`, `state`, `postalCode`, `country`), `nit`, `nrc`, `plan` y `category`, pero no están presentes en ningún formulario del front | Funcional |

#### Campos requeridos por el backend (crear negocio + usuario admin)
| Campo | Objeto | Obligatorio |
|-------|--------|-------------|
| `name` | `business` | Sí |
| `email` | `business` | No |
| `phone` | `business` | No |
| `address` | `business` | No |
| `nit` | `business` | No |
| `nrc` | `business` | No |
| `plan` | `business` | No |
| `category` | `business` | No |
| `email` | `user` | Sí |
| `password` | `user` | Sí |

---

### 2.2 Categorías de Producto (`/admin/categoryProducts`)

#### Estado actual
- Formulario: `name`, `description`, `photo` (campo file oculto), `business` (oculto, del localStorage).
- El backend (`controller/category_product.js → saveCategoryProduct`) requiere: `name`, `description`, `business`.
- `photo` es **opcional** en el modelo del backend.

#### Problemas detectados
| # | Problema | Impacto |
|---|----------|---------|
| 1 | El campo `photo` tiene un `<input type="file" class="d-none">` sin ningún listener ni lógica de carga — el usuario no puede interactuar con él | UX / funcional |
| 2 | `[formGroup]="formularioCategoria"` está aplicado sobre un `<div class="card">` en vez de un elemento `<form>`, lo que viola el estándar HTML y puede causar comportamientos inesperados en validaciones | Técnico |
| 3 | No se muestran mensajes de error de validación para los campos `name` ni `description` dentro del formulario | UX |
| 4 | El modelo del backend tiene `unique: true` en el campo `name` de forma **global** (no por negocio). Dos negocios distintos no pueden compartir el mismo nombre de categoría aunque el `findOne({ name, business })` lo permita lógicamente | Bug backend (corregido parcialmente) |
| 5 | Al guardar exitosamente no hay feedback visual más allá de ocultar el form | UX |

#### Campos a conservar en el formulario
| Campo | Mostrar | Validación |
|-------|---------|-----------|
| `name` | Sí | Requerido |
| `description` | Sí | Requerido |
| `photo` | Sí (con lógica) o eliminar | Opcional |
| `business` | No (oculto, automático) | Automático desde localStorage |

---

### 2.3 Sub-Categorías de Producto (dentro de `/admin/categoryProducts`)

#### Estado actual
- El formulario es el mismo componente de categorías (`CategoryProductComponent`), reutilizado condicionalmente según si hay una `categorySelected`.
- El backend (`controller/sub_category_product.js → saveSubCategoryProduct`) requiere: `name`, `description`, `categoryProduct`.
- El frontend envía: `name`, `description`, `categoryProduct: this.categorySelected._id` — correcto.

#### Problemas detectados
| # | Problema | Impacto |
|---|----------|---------|
| 1 | El modelo del backend tiene `unique: true` en `name` de forma global — mismo bug que en categorías | Bug backend |
| 2 | El mensaje de error `"Subcategoría ya está registrada"` solo se muestra para el caso `status 400` al guardar sub-categoría, pero no se limpia automáticamente al cerrar el formulario | UX |
| 3 | La tabla de sub-categorías muestra columna `Foto` con un ícono placeholder pero nunca carga imagen real | UX |
| 4 | No hay validación mínima de longitud en `name` ni `description` | Calidad |

#### Campos a conservar
| Campo | Mostrar | Validación |
|-------|---------|-----------|
| `name` | Sí | Requerido |
| `description` | Sí | Requerido |
| `photo` | Eliminar del form o implementar | Opcional |
| `categoryProduct` | No (oculto, automático desde `categorySelected`) | Automático |

---

### 2.4 Productos (`/admin/products`)

#### Estado actual
- Formulario: `name`, `file` (imagen), `price`, `description`, `category` (select), `subCategoryProduct` (select).
- El backend (`controller/product.js → saveProduct`) requiere: `name`, `description`, `subCategoryProduct` (ObjectId válido, **obligatorio**), `price`, `business`.
- `photo` en el modelo es opcional — la imagen se sube por separado vía `ImageService`.

#### Problemas detectados
| # | Problema | Impacto |
|---|----------|---------|
| 1 | `subCategoryProduct` es **requerido** en el modelo del backend, pero el select del front tiene una opción vacía (`value=""`) y no tiene `Validators.required` — se puede enviar string vacío y el backend retorna 400 | Bug crítico |
| 2 | `category` está marcado como `Validators.required` en el formulario pero **no se envía al backend** — solo sirve para filtrar sub-categorías. Si la categoría seleccionada no tiene sub-categorías, el formulario queda inválido por siempre | Bug UX crítico |
| 3 | `saveProduct` en el componente **no incluye `category`** en el payload enviado al API (correcto técnicamente), pero la validación required en el form bloquea el guardado aunque se hayan llenado todos los datos reales | Funcional |
| 4 | La función `editProduct` rellena el formulario y deshabilita campos, pero **no existe endpoint `PUT`/`PATCH`** en `ProductsService` ni en el backend para editar un producto — la edición está visualmente presente pero es inoperativa | Bug funcional |
| 5 | El campo `file` es `Validators.required`, pero en edición la imagen ya existe — al editar se exige subir imagen nuevamente sin necesidad | UX |
| 6 | El modelo del backend tiene `unique: true` en `name` de forma global — dos negocios no pueden tener producto con el mismo nombre | Bug backend |
| 7 | `photo` se envía vacío en `saveProduct` — la imagen se carga en llamada separada, pero si falla esa segunda llamada el producto queda sin imagen sin notificación al usuario | UX |

#### Campos a conservar / modificar
| Campo del form | Enviar al backend | Validación correcta |
|----------------|-------------------|---------------------|
| `name` | Sí | Requerido |
| `description` | Sí | Requerido |
| `price` | Sí | Requerido, mínimo > 0 |
| `file` | No (upload separado) | Requerido solo en creación |
| `category` | No (solo para filtrar subcategorías) | Quitar `Validators.required`, poner opcional |
| `subCategoryProduct` | Sí | Agregar `Validators.required` |
| `business` | Sí (automático) | Automático desde localStorage |

---

## 3. Bugs de Backend Adicionales Identificados

| Archivo | Problema | Acción |
|---------|----------|--------|
| `model/product.js` | `name: unique: true` global — bloquea mismos nombres entre negocios | Quitar `unique`, igual que se hizo en `category_product` |
| `model/sub_category_product.js` | `name: unique: true` global — mismo problema | Quitar `unique`, agregar índice compuesto `{ name, categoryProduct }` |
| `controller/product.js:34` | `Product({ ... })` sin `new` — mismo bug que se corrigió en `category_product` | Agregar `new Product(...)` |
| `controller/sub_category_product.js:47` | `SubCategoryProduct({ ... })` sin `new` | Agregar `new SubCategoryProduct(...)` |

---

## 4. Plan de Acción Priorizado

### FASE 1 — Correcciones de Backend (prerequisito)

| Ítem | Archivo | Cambio |
|------|---------|--------|
| 1.1 | `model/sub_category_product.js` | Quitar `unique: true` de `name` |
| 1.2 | `model/product.js` | Quitar `unique: true` de `name` |
| 1.3 | `controller/product.js` | Agregar `new` al instanciar `Product(...)` |
| 1.4 | `controller/sub_category_product.js` | Agregar `new` al instanciar `SubCategoryProduct(...)` |
| 1.5 | MongoDB | Ejecutar `db.Product.dropIndex("name_1")` y `db.SubCategoryProduct.dropIndex("name_1")` |

---

### FASE 2 — Correcciones Críticas del Frontend

| Ítem | Componente | Cambio |
|------|-----------|--------|
| 2.1 | `products.component.ts` | Quitar `Validators.required` de `category`; agregar `Validators.required` a `subCategoryProduct` |
| 2.2 | `products.component.ts` | En `editProduct`, quitar `Validators.required` de `file` durante edición |
| 2.3 | `products.component.html` | Agregar mensaje de error bajo el select de sub-categoría |
| 2.4 | `products.component.html` | Mostrar mensaje si la categoría seleccionada no tiene sub-categorías (bloqueo de flujo) |
| 2.5 | `products.component.ts → saveProduct` | Agregar validación: si `subCategoryProduct` está vacío, no permitir envío |

---

### FASE 3 — Mejoras de UX en Categorías

| Ítem | Componente | Cambio |
|------|-----------|--------|
| 3.1 | `category-product.component.html` | Cambiar `<div [formGroup]>` por `<form [formGroup]>` |
| 3.2 | `category-product.component.html` | Agregar `*ngIf` con mensajes de error para `name` y `description` |
| 3.3 | `category-product.component.html` | Eliminar el campo `<input type="file" class="d-none">` del form o implementar lógica de preview/upload real |
| 3.4 | `category-product.component.ts` | En `showForm()`, limpiar `errorSaveSubCategory` al cerrar el formulario |
| 3.5 | `category-product.component.html` | Eliminar columna "Foto" de la tabla de sub-categorías hasta implementar carga de imagen |

---

### FASE 4 — Edición de Productos (funcionalidad faltante)

| Ítem | Archivo | Cambio |
|------|---------|--------|
| 4.1 | `ucoit-back/controller/product.js` | Crear función `updateProduct` con `PUT /auth/product/:id` |
| 4.2 | `ucoit-back/routes/auth/product.js` | Registrar ruta `PUT /:id` |
| 4.3 | `products.service.ts` | Agregar método `updateProduct(id, data)` |
| 4.4 | `products.component.ts` | Conectar `editProduct` para llamar `updateProduct` al guardar si `isEditing === true` |

---

### FASE 5 — Registro de Negocio con datos completos

| Ítem | Archivo | Cambio |
|------|---------|--------|
| 5.1 | `business-confirm-save.component.html` | Reemplazar `mailmail.com` por el email real del usuario (pasar via state o servicio) |
| 5.2 | Crear nuevo componente `register-business` | Formulario con campos: `name` (req), `email`, `phone`, `address`, `user.email` (req), `user.password` (req) |
| 5.3 | App routing | Registrar ruta pública `/register` → `register-business` |
| 5.4 | Crear `RegisterBusinessService` | Método `POST /service/business` |

---

## 5. Orden de Ejecución Recomendado

```
FASE 1 (Backend bugs)
    ↓
FASE 2 (Producto - validaciones críticas)
    ↓
FASE 3 (Categorías - UX)
    ↓
FASE 4 (Edición producto)
    ↓
FASE 5 (Registro negocio completo)
```

---

## 6. Resumen de Campos a Eliminar del Frontend

| Componente | Campo | Motivo |
|-----------|-------|--------|
| `category-product` form | `<input type="file" class="d-none">` (photo) | No tiene lógica de upload, confunde el formulario |
| `products` form | Validación `required` en `category` | El campo no se envía al backend; bloquea el flujo |
| `sub-categories` table | Columna "Foto" | Placeholder sin datos reales |
| `business-confirm-save` | Email `mailmail.com` | Dato hardcodeado incorrecto |

---

## 7. Resumen de Campos Faltantes en el Frontend

| Flujo | Campo Faltante | Requerido en Backend |
|-------|---------------|---------------------|
| Registro negocio | `address.street`, `address.city`, etc. | No |
| Registro negocio | `nit`, `nrc` | No |
| Registro negocio | `plan` | No (default) |
| Producto - guardar | `subCategoryProduct` con validación `required` | Sí |
| Producto - editar | Endpoint y lógica PUT | N/A |
