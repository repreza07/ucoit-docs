# 01 — Fix Crítico: Middleware de Autenticación JWT

**Deuda técnica:** #1 — `next()` fuera del callback de `jwt.verify`  
**Severidad:** CRÍTICA  
**Área:** Backend + Frontend  
**Fecha:** 2026-04-22

---

## Diagnóstico

### Causa raíz

`jwt.verify` con callback es asíncrono. Node.js no espera el callback — avanza inmediatamente a `next()` en la línea 44, antes de que la verificación termine.

```
1. jwt.verify(token, secret, callback)  → encola el callback, retorna de inmediato
2. next()                               → pasa el request al controlador  ← BUG
3. [event loop] callback ejecuta        → intenta res.status(403)...
                                           pero la respuesta ya está en camino
```

| Escenario | Comportamiento actual |
|-----------|----------------------|
| Token ausente | Retorna 403, pero el controlador también ejecuta |
| Token expirado | Retorna 403 Y el controlador ejecuta → `Cannot set headers after they are sent` |
| Token con firma inválida | El callback no hace nada (no coincide ningún `if`), el controlador ejecuta sin bloqueo |
| Token fabricado | El controlador ejecuta normalmente — la protección no existe |

**Problema adicional:** `expiresIn: 28800000` se interpreta como segundos → tokens de 333 días.  
**Problema adicional:** no existe mecanismo de refresh — token único de larga duración, sin detección de inactividad.

### Rutas afectadas

Todas bajo `app.use('/auth', verifyToken, authService)`:

| Endpoint | Controlador |
|----------|-------------|
| `GET/POST /auth/category-product` | `category_product.*` |
| `GET/POST /auth/sub-category-product` | `sub_category_product.*` |
| `GET/POST /auth/product` | `product.*` |
| `GET/POST /auth/user` | `user.*` |
| `POST /auth/user/validate` | `user.validateUser` |
| `GET /auth/business-private` | `business.getDataBusiness` |

---

## Decisiones de Diseño

### Almacenamiento de tokens

| Token | Dónde | Justificación |
|-------|-------|--------------|
| **Access token** (15 min) | Variable privada en `AuthService` (memoria) | No persiste en disco, invisible a XSS, desaparece al cerrar tab |
| **Refresh token** (8h) | `localStorage` con key `refresh_token` | Sobrevive recarga de página sin forzar re-login; usuarios internos en red local, riesgo XSS aceptable |

> **Nota:** Si en el futuro se requiere mayor seguridad, el refresh token puede migrarse a `httpOnly cookie` — requeriría agregar `cookie-parser` al backend y configurar CORS con `credentials: true`. Para la Fase 1 del MVP, `localStorage` es suficiente.

### Tiempos de sesión

- **Access token:** `15m` — ventana de ataque mínima si es interceptado
- **Refresh token:** `8h` — cubre un turno completo de restaurante
- **Inactividad:** `45 min` — balance entre cajero (activo) y cocinero (puede no tocar pantalla por 20 min)

### Sliding session

El access token se renueva automáticamente vía `POST /service/auth/refresh` cuando el interceptor detecta un 401, siempre que el refresh token no haya expirado. Si el usuario lleva 45 min inactivo, el servicio de inactividad cierra la sesión localmente antes de que expire el access token.

---

## Plan de Implementación

### BACKEND — 4 archivos

---

#### B1 · `ucoit-back/middleware/jwt.js` — reescritura completa

```javascript
const jwt = require('jsonwebtoken');

const { SECRET_KEY_JWT, SECRET_KEY_REFRESH } = process.env;

const generateAccessToken = (usuario) =>
    jwt.sign({ usuario }, SECRET_KEY_JWT, { expiresIn: '15m' });

const generateRefreshToken = (usuario) =>
    jwt.sign({ usuario }, SECRET_KEY_REFRESH, { expiresIn: '8h' });

const verifyRefreshToken = (token) => {
    try {
        return jwt.verify(token, SECRET_KEY_REFRESH);
    } catch {
        return null;
    }
};

const verifyToken = (req, res, next) => {
    const header = req.header('Authorization');

    if (!header) {
        return res.status(401).json({ ok: false, message: 'Token de acceso requerido' });
    }

    const token = header.replace('Bearer ', '');

    jwt.verify(token, SECRET_KEY_JWT, (error, decoded) => {
        if (error?.message === 'jwt malformed') {
            return res.status(401).json({ ok: false, message: 'Token inválido' });
        }
        if (error?.message === 'jwt expired') {
            return res.status(401).json({ ok: false, message: 'Token expirado' });
        }
        if (error) {
            return res.status(401).json({ ok: false, message: 'Token no autorizado' });
        }

        req.usuario = decoded.usuario;
        next();
    });
};

module.exports = { generateAccessToken, generateRefreshToken, verifyToken, verifyRefreshToken };
```

Cambios respecto al archivo actual:
- `next()` movido dentro del callback — solo ejecuta cuando `error` es `null`
- `refreshToken` → `generateAccessToken` con `expiresIn: '15m'`
- Agrega `generateRefreshToken` con secret separado `SECRET_KEY_REFRESH`
- Agrega `verifyRefreshToken` síncrono (retorna `null` en lugar de lanzar)
- `req.usuario = decoded.usuario` — propaga payload a controladores
- HTTP `403` → `401` en todos los casos

---

#### B2 · `ucoit-back/controller/auth.js` — actualizar login

Localizar el bloque donde se genera el token y donde se envía la respuesta. Reemplazar:

```javascript
// ANTES — buscar la línea que usa refreshToken o jwt.sign
const token = await refreshToken(usuario)
return res.json({ ok: true, token, usuario })

// DESPUÉS
const { generateAccessToken, generateRefreshToken } = require('../middleware/jwt')

const accessToken  = generateAccessToken(usuario)
const refreshToken = generateRefreshToken(usuario)

return res.json({
    ok: true,
    token: accessToken,
    refreshToken,
    usuario
})
```

---

#### B3 · `ucoit-back/routes/public/auth.refresh.js` — crear archivo nuevo

```javascript
const { Router } = require('express');
const { verifyRefreshToken, generateAccessToken } = require('../../middleware/jwt');

const router = Router();

router.post('/refresh', (req, res) => {
    const { refreshToken } = req.body;

    if (!refreshToken) {
        return res.status(401).json({ ok: false, message: 'Refresh token requerido' });
    }

    const payload = verifyRefreshToken(refreshToken);

    if (!payload) {
        return res.status(401).json({ ok: false, message: 'Refresh token inválido o expirado' });
    }

    const accessToken = generateAccessToken(payload.usuario);
    res.json({ ok: true, token: accessToken });
});

module.exports = router;
```

---

#### B4 · `ucoit-back/routes/routes.js` — registrar el endpoint de refresh

Agregar al final del archivo, antes del `module.exports`:

```javascript
const authRefresh = require('./public/auth.refresh');
router.use('/auth', authRefresh);
// Resultado: POST /service/auth/refresh
```

**Variable de entorno a agregar en `.env`:**
```
SECRET_KEY_REFRESH=<64-caracteres-distintos-a-SECRET_KEY_JWT>
```
Generar con: `node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"`

---

### FRONTEND — 5 archivos

---

#### F1 · `ucoit-front/src/app/services/auth.service.ts` — refactorizar

Reemplazar el contenido completo:

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, Observable, throwError } from 'rxjs';
import { map, tap, catchError } from 'rxjs/operators';
import { Env } from '../environments/env';

const { PATH } = Env;
const REFRESH_KEY = 'refresh_token';

@Injectable({ providedIn: 'root' })
export class AuthService {
  private accessToken: string | null = null;
  currentUser$ = new BehaviorSubject<any>(null);

  constructor(private http: HttpClient) {}

  getAccessToken(): string | null {
    return this.accessToken;
  }

  setTokens(accessToken: string, refreshToken: string, user: any): void {
    this.accessToken = accessToken;
    localStorage.setItem(REFRESH_KEY, refreshToken);
    this.currentUser$.next(user);
  }

  refreshAccessToken(): Observable<string> {
    const refreshToken = localStorage.getItem(REFRESH_KEY);
    if (!refreshToken) return throwError(() => new Error('Sin refresh token'));

    return this.http
      .post<{ ok: boolean; token: string }>(`${PATH}/service/auth/refresh`, { refreshToken })
      .pipe(
        map(res => {
          this.accessToken = res.token;
          return res.token;
        })
      );
  }

  login(body: { email: string; password: string }): Observable<any> {
    return this.http
      .post<any>(`${PATH}/service/login`, body)
      .pipe(
        tap(res => {
          if (res.ok) {
            this.setTokens(res.data.token, res.data.refreshToken, res.data.user);
          }
        })
      );
  }

  logout(): void {
    this.accessToken = null;
    localStorage.removeItem(REFRESH_KEY);
    this.currentUser$.next(null);
  }

  tryRestoreSession(): Observable<string | null> {
    const refreshToken = localStorage.getItem(REFRESH_KEY);
    if (!refreshToken) return throwError(() => null);
    return this.refreshAccessToken().pipe(catchError(() => throwError(() => null)));
  }
}
```

---

#### F2 · `ucoit-front/src/app/header.interceptor.ts` — reescritura completa

```typescript
import { Injectable } from '@angular/core';
import {
  HttpRequest, HttpHandler, HttpEvent,
  HttpInterceptor, HttpErrorResponse
} from '@angular/common/http';
import { BehaviorSubject, Observable, throwError } from 'rxjs';
import { catchError, filter, switchMap, take } from 'rxjs/operators';
import { Router } from '@angular/router';
import { AuthService } from './services/auth.service';

@Injectable()
export class HeaderInterceptor implements HttpInterceptor {
  private isRefreshing = false;
  private refreshToken$ = new BehaviorSubject<string | null>(null);

  constructor(private authService: AuthService, private router: Router) {}

  intercept(request: HttpRequest<unknown>, next: HttpHandler): Observable<HttpEvent<unknown>> {
    const token = this.authService.getAccessToken();
    const req = token ? this.addToken(request, token) : request;

    return next.handle(req).pipe(
      catchError((error: HttpErrorResponse) => {
        if (error.status === 401 && !request.url.includes('/auth/refresh')) {
          return this.handle401(request, next);
        }
        return throwError(() => error);
      })
    );
  }

  private handle401(request: HttpRequest<unknown>, next: HttpHandler): Observable<HttpEvent<unknown>> {
    if (!this.isRefreshing) {
      this.isRefreshing = true;
      this.refreshToken$.next(null);

      return this.authService.refreshAccessToken().pipe(
        switchMap(newToken => {
          this.isRefreshing = false;
          this.refreshToken$.next(newToken);
          return next.handle(this.addToken(request, newToken));
        }),
        catchError(err => {
          this.isRefreshing = false;
          this.authService.logout();
          this.router.navigate(['/login']);
          return throwError(() => err);
        })
      );
    }

    // Requests concurrentes esperan el nuevo token
    return this.refreshToken$.pipe(
      filter((t): t is string => t !== null),
      take(1),
      switchMap(token => next.handle(this.addToken(request, token)))
    );
  }

  private addToken(request: HttpRequest<unknown>, token: string): HttpRequest<unknown> {
    return request.clone({ headers: request.headers.set('Authorization', `Bearer ${token}`) });
  }
}
```

---

#### F3 · `ucoit-front/src/app/pages/admin/guards/session.guard.ts` — reescritura completa

```typescript
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from '../../../services/auth.service';

export const sessionGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  const hasSession =
    authService.getAccessToken() !== null ||
    localStorage.getItem('refresh_token') !== null;

  if (hasSession) return true;

  router.navigate(['/login']);
  return false;
};
```

---

#### F4 · `ucoit-front/src/app/core/services/inactivity.service.ts` — crear archivo nuevo

```typescript
import { Injectable, NgZone, OnDestroy } from '@angular/core';
import { Router } from '@angular/router';
import { fromEvent, merge, Subscription } from 'rxjs';
import { AuthService } from '../../services/auth.service';

@Injectable({ providedIn: 'root' })
export class InactivityService implements OnDestroy {
  private readonly TIMEOUT_MS = 45 * 60 * 1000; // 45 minutos
  private timer: ReturnType<typeof setTimeout> | null = null;
  private events$: Subscription | null = null;

  constructor(
    private authService: AuthService,
    private router: Router,
    private ngZone: NgZone
  ) {}

  startWatching(): void {
    const activity$ = merge(
      fromEvent(document, 'mousemove'),
      fromEvent(document, 'keydown'),
      fromEvent(document, 'click'),
      fromEvent(document, 'touchstart')
    );

    this.ngZone.runOutsideAngular(() => {
      this.events$ = activity$.subscribe(() => this.reset());
      this.reset();
    });
  }

  stopWatching(): void {
    if (this.timer) { clearTimeout(this.timer); this.timer = null; }
    this.events$?.unsubscribe();
    this.events$ = null;
  }

  private reset(): void {
    if (this.timer) clearTimeout(this.timer);
    this.timer = setTimeout(() => {
      this.ngZone.run(() => {
        this.authService.logout();
        this.router.navigate(['/login']);
      });
    }, this.TIMEOUT_MS);
  }

  ngOnDestroy(): void { this.stopWatching(); }
}
```

---

#### F5 · `ucoit-front/src/app/app.module.ts` — agregar `APP_INITIALIZER` y conectar `InactivityService`

Agregar los imports necesarios y el provider:

```typescript
import { APP_INITIALIZER, NgModule } from '@angular/core';
import { of } from 'rxjs';
import { catchError } from 'rxjs/operators';
import { AuthService } from './services/auth.service';

function initAuth(authService: AuthService) {
  return () => authService.tryRestoreSession().pipe(catchError(() => of(null)));
}

// En @NgModule providers, agregar:
{
  provide: APP_INITIALIZER,
  useFactory: initAuth,
  deps: [AuthService],
  multi: true
}
```

En `AppComponent`, activar el watcher de inactividad al detectar sesión:

```typescript
// app.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';
import { AuthService } from './services/auth.service';
import { InactivityService } from './core/services/inactivity.service';

export class AppComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  constructor(
    private authService: AuthService,
    private inactivity: InactivityService
  ) {}

  ngOnInit(): void {
    this.authService.currentUser$
      .pipe(takeUntil(this.destroy$))
      .subscribe(user => {
        user ? this.inactivity.startWatching() : this.inactivity.stopWatching();
      });
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

---

## Checklist de Implementación

### Backend
- [ ] Reescribir `ucoit-back/middleware/jwt.js` (B1)
- [ ] Actualizar login en `ucoit-back/controller/auth.js` para emitir `accessToken` + `refreshToken` (B2)
- [ ] Crear `ucoit-back/routes/public/auth.refresh.js` (B3)
- [ ] Registrar ruta `/service/auth/refresh` en `ucoit-back/routes/routes.js` (B4)
- [ ] Agregar `SECRET_KEY_REFRESH` al `.env`

### Frontend
- [ ] Refactorizar `ucoit-front/src/app/services/auth.service.ts` (F1)
- [ ] Reescribir `ucoit-front/src/app/header.interceptor.ts` (F2)
- [ ] Reescribir `ucoit-front/src/app/pages/admin/guards/session.guard.ts` (F3)
- [ ] Crear `ucoit-front/src/app/core/services/inactivity.service.ts` (F4)
- [ ] Agregar `APP_INITIALIZER` en `app.module.ts` y conectar `InactivityService` en `app.component.ts` (F5)

### Verificación
- [ ] `curl` sin token a `/auth/product/` → `401` (antes: `200`)
- [ ] `curl` con token expirado → `401 "Token expirado"` (antes: `200`)
- [ ] `curl` con token fabricado → `401` (antes: `200`)
- [ ] `POST /service/auth/refresh` con refresh válido → nuevo access token
- [ ] `POST /service/auth/refresh` con refresh inválido → `401`
- [ ] En frontend: request 401 → interceptor llama a `/refresh` → reintenta y responde sin error visible
- [ ] En frontend: refresh también falla → redirect a `/login` + localStorage limpio
- [ ] En frontend: 45 min sin actividad → logout automático → redirect a `/login`
- [ ] Recarga de página con sesión activa → `APP_INITIALIZER` rehidrata el access token sin ir a login
- [ ] Sin `Cannot set headers after they are sent` en los logs del servidor

---

## Deuda Fuera de Scope

- **`logout()` en backend** — `POST /auth/logout` que invalide el refresh token. Requiere lista negra en Redis o MongoDB. Planificar en iteración siguiente.
- **`req.usuario` en controladores** — una vez disponible, el `businessId` puede leerse desde `req.usuario.business` en lugar de `req.query.q`, eliminando el vector de acceso cruzado entre negocios.
- **httpOnly cookie para refresh token** — opción más segura si la app se expone públicamente. Requiere `cookie-parser` en backend y `withCredentials: true` en el interceptor.
