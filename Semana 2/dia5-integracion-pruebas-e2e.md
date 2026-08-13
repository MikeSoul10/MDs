# HOJA DE INSTRUCCIONES — MODULO DE PIPAS — SPRINT 2 — DIA 5 (VIERNES)

**Autor:** Turno (Lider de turno)
**Fecha:** 2026-08-14
**Repositorio:** dapa2w — rama de integracion `feature/pipas-semana2`
**Objetivo del dia:** Integracion y pruebas E2E. Servicios de frontend para solicitudes y
asignaciones (`asignacionesService.js`), resolver CORS/red con un proxy de Vite (2.27), el flujo
completo en 2 sesiones de navegador (solicitud -> asignacion -> PWA del chofer -> GPS en el mapa),
probar la instalacion/offline de la PWA, correr la suite completa en Docker y abrir el PR.

> **IMPORTANTE (convencion del sprint): TODAS las instalaciones, builds y pruebas se hacen
> DENTRO del contenedor Docker.** Nunca `npm install`/`npm run build`/`pytest` en la maquina
> host. Usar `docker compose exec frontend ...` y `docker compose exec backend ...`.

---

## Tareas del dia

| # | Tarea | Archivo(s) principal(es) | DoD |
|---|-------|--------------------------|-----|
| 2.26 | Crear `asignacionesService.js` y completar `solicitudesService.js` en el frontend (consumen `localhost:8000/api/v1` / `VITE_API_URL`) | `frontend/src/services/` | Las pantallas usan los servicios, no datos quemados |
| 2.27 | Resolver CORS/red: dentro de Docker el frontend llama a `http://backend:8000`, fuera a `localhost:8000` (proxy de Vite) | `frontend/vite.config.js` + `frontend/src/services/api.js` + `docker-compose.yml` | Sin errores de CORS en consola; el WS tambien pasa por el proxy |
| 2.28 | Prueba E2E: crear solicitud -> asignar vehiculo -> chofer abre PWA -> su GPS aparece en el mapa de supervision -> simular movimiento | navegador (2 sesiones) | El marcador del vehiculo se mueve en vivo |
| 2.29 | Probar instalacion de la PWA (Add to Home Screen) y carga offline | navegador | PWA instalable y funcional sin red |
| 2.30 | Correr `pytest` completo en backend y `npm run build` del frontend | terminal (Docker) | Suite verde (sin regresiones) y build exitoso |
| 2.31 | PR a `feature/pipas-semana2` y revision | GitHub | PR abierto con commits `feat:`/`fix:` |

**DoD del dia:** flujo E2E validado en 2 sesiones con el marcador moviendose en vivo; PWA
instalable y offline; `pytest` + `npm run build` en verde; PR listo para revision.

---

## 0. Prerequisitos y convenciones

- Todo lo de los dias anteriores esta MERGEADO y funcionando en Docker:
  - OSRM responde rutas: `curl "http://localhost:5000/route/v1/driving/-103.35,20.64;-103.31,20.67"`
    -> `{"code":"Ok",...}`. Los datos de mapa ya estan construidos en `db/osrm/`.
  - Backend: `GET /api/v1/solicitudes_pipa`, `POST /api/v1/solicitudes_pipa`, `PATCH .../estado`,
    `POST /api/v1/asignaciones/automatica`, `GET /api/v1/asignaciones`, WS `/ws/ubicacion`.
  - Frontend: `solicitudesService.js`, `MapaSupervision.jsx`, `AppChofer.jsx` (PWA con GPS, cola
    offline y WS cada 10 s), rutas `/mapa` y `/chofer` dentro del `DashboardLayout`.
- **Variables de entorno (Vite)**: Vite solo lee variables con prefijo `VITE_`. El compose define
  `REACT_APP_API_URL=http://backend:8000` (linea 209) que Vite **ignora**; `api.js` cae al default
  `http://localhost:8000/api/v1` y el navegador llama directo al backend (depende de CORS). Esto se
  resuelve HOY en 2.27 con un proxy de Vite: el navegador siempre habla con el MISMO origen
  (`localhost:3000`) y Vite reenvia `/api` y `/ws` a `http://backend:8000` desde el contenedor.
- **WebSocket hardcodeado**: `MapaSupervision.jsx` (linea 43) y `AppChofer.jsx` (linea 63) usan
  `ws://localhost:8000/ws/ubicacion?token=...`. Tras 2.27 deben usar el helper `getWsUrl()` para
  apuntar al mismo origen (`ws://localhost:3000/ws/ubicacion`), que el proxy reenvia a OSRM-backend.
  > Ojo: es el proxy de Vite el que resuelve `http://backend:8000` DENTRO del contenedor; el
  > navegador nunca ve el hostname `backend`.
- **Estado del `docker compose ps`**: `dapa2-osrm` aparece `(unhealthy)` porque el healthcheck usa
  `curl` y la imagen `osrm-backend` no lo trae. El servicio responde OK (verificar con el curl de
  arriba). Fix opcional del healthcheck en 2.28.
- **Nota de red para la prueba E2E**: el usuario seed es `admin` / `admin123` (login form-encoded).
  El AppChofer identifica el vehiculo por query param: `/chofer?vehiculo=<vhe_id>` (default 4).
- Pruebas del dia: manuales en navegador (2 sesiones) + `pytest` del backend + `npm run build`.
  No hay suite de tests de frontend en el repo; la verificacion es build + prueba manual.

---

## TAREA 2.26 — Servicios de frontend (solicitudes + asignaciones)

`solicitudesService.js` ya existe con `getSolicitudes` y `cambiarEstadoSolicitud`. Agregar
`crearSolicitud` (se usa en el E2E 2.28) y crear `asignacionesService.js`.

### `frontend/src/services/solicitudesService.js` (agregar al final)

```js
export function crearSolicitud(body) {
  return api.post('/solicitudes_pipa', body);
}
```

### Crear `frontend/src/services/asignacionesService.js`

```js
import { api } from './api';

export function asignacionAutomatica(sppId) {
  return api.post('/asignaciones/automatica', { spp_id: sppId });
}

export function getAsignaciones(params = {}) {
  const qs = new URLSearchParams(params).toString();
  return api.get(`/asignaciones${qs ? `?${qs}` : ''}`);
}
```

> Forma del response de `POST /asignaciones/automatica` (schema `AsignacionResponse` del backend):
> `{ asg_id, spp_id, vhe_id, rtsol_id, pozo_sugerido: {poz_id, poz_nombrepozo} | null,
> distance_km, duration_min, geometry, asg_estatus, asg_fecha }`. Si no hay vehiculos disponibles
> o no se calcula ruta, responde 409 con `detail`.

### `frontend/src/services/api.js` (ampliar)

El helper `api` ya expone `get/post/put/delete`. Agregar `patch` (para `cambiarEstadoSolicitud`):

```js
patch: (url, data) => request(url, { method: 'PATCH', body: JSON.stringify(data) }),
```

**DoD:** los servicios apuntan al API sin datos quemados; `crearSolicitud`, `asignacionAutomatica`
y `getAsignaciones` responden contra el backend (se ejercitan en 2.28).

---

## TAREA 2.27 — CORS/red con proxy de Vite (TODO EN DOCKER)

Objetivo: que dentro de Docker el frontend (contenedor) llame a `http://backend:8000` y fuera de
Docker a `localhost:8000`, sin errores de CORS. Se logra con el proxy del dev/preview server de
Vite (mismo origen para el navegador; Vite reenvia al backend dentro de la red de Docker).

### 1. `frontend/vite.config.js` — agregar proxy para `/api` y `/ws`

```js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    react(),
    VitePWA({
      registerType: 'autoUpdate',
      manifest: false,
      workbox: {
        globPatterns: ['**/*.{js,css,html,svg,png,webmanifest}'],
        maximumFileSizeToCacheInBytes: 5 * 1024 * 1024,
      },
      devOptions: { enabled: true },
    }),
  ],
  server: {
    port: 3000,
    host: true,
    allowedHosts: true,
    proxy: {
      '/api': { target: 'http://backend:8000', changeOrigin: true },
      '/ws': { target: 'ws://backend:8000', ws: true, changeOrigin: true },
    },
  },
  preview: {
    port: 4173,
    host: true,
    allowedHosts: true,
    proxy: {
      '/api': { target: 'http://backend:8000', changeOrigin: true },
      '/ws': { target: 'ws://backend:8000', ws: true, changeOrigin: true },
    },
  },
});
```

> `preview` se usa en 2.29 para probar la PWA contra el build de produccion (`dist/`) sin romper
> el API ni el WebSocket. Cambiar `vite.config.js` requiere reiniciar el server de Vite:
> `docker compose restart frontend`.
>
> **IMPORTANTE — `allowedHosts` (Vite 5)**: el dev server bloquea con `403 "Blocked request. This
> host (...)"` las peticiones cuyo `Host` no este en `server.allowedHosts`. Sin `allowedHosts: true`
> (o `['localhost','frontend']`), cualquier acceso al frontend por un hostname distinto de
> `localhost` (p. ej. `http://frontend:3000` desde otro contenedor, o el IP de la maquina) falla.
> Se incluye `allowedHosts: true` en `server` y `preview`.

### 2. `frontend/src/services/api.js` — base relativa

```js
const API_BASE = import.meta.env.VITE_API_URL || '/api/v1';
```

Y agregar un helper de URL del WebSocket (mismo origen, reenviado por el proxy):

```js
export function getWsUrl(path) {
  const proto = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
  return `${proto}//${window.location.host}${path}`;
}
```

### 3. `frontend/src/services/authService.js` — mismo cambio en el login

El `login` hardcodea `http://localhost:8000/api/v1` (linea 39). Cambiarlo a la base relativa:

```js
const res = await fetch(
  (import.meta.env.VITE_API_URL || '/api/v1') + '/auth/login',
  { method: 'POST', body }
);
```

### 4. WebSocket en `MapaSupervision.jsx` y `AppChofer.jsx`

Reemplazar `ws://localhost:8000/ws/ubicacion?token=${token}` por:

```js
import { getWsUrl } from '../services/api';
// ...
const ws = new WebSocket(`${getWsUrl('/ws/ubicacion')}?token=${token}`);
```

### 5. `docker-compose.yml` — quitar la env inutil

Eliminar la linea 209: `REACT_APP_API_URL=http://backend:8000` (Vite la ignora y ya no se necesita;
con el proxy el navegador usa el mismo origen). Recargar: `docker compose up -d frontend`.

> Con esto la API y el WS pasan por el proxy y el CORS queda fuera del camino (mismo origen). El
> `CORS_ORIGINS` del backend (`http://localhost:3000`) se deja como esta (fallback inofensivo).

**Verificacion (Docker):**

```bash
docker compose restart frontend
# API a traves del proxy:
curl -s -o NUL -w "%{http_code}\n" http://localhost:3000/api/v1/vehiculos   # 401 (pide JWT) = proxy OK
# Login a traves del proxy (sin errores de CORS en la pestana del navegador):
curl -s -X POST http://localhost:3000/api/v1/auth/login -d "username=admin&password=admin123" | head -c 40
```

**DoD:** sin errores de CORS en la consola del navegador; `/api/*` y `/ws/*` responden por
`localhost:3000` tanto en dev como en preview.

---

## TAREA 2.28 — Prueba E2E (2 sesiones de navegador)

### 0. Pre-check: OSRM respondiendo

```bash
curl "http://localhost:5000/route/v1/driving/-103.35,20.64;-103.31,20.67"
# -> {"code":"Ok", ...}
```

> `dapa2-osrm` aparece `(unhealthy)` solo por el healthcheck con `curl`. Fix opcional: en
> `docker-compose.yml`, reemplazar el healthcheck por uno sin `curl` (la imagen trae bash):
>
> ```yaml
> healthcheck:
>   test: ["CMD-SHELL", "exec 3<>/dev/tcp/127.0.0.1/5000 || exit 1"]
>   interval: 30s
>   timeout: 10s
>   retries: 5
> ```
>
> Aplicar con `docker compose up -d osrm`.

### 1. Crear una solicitud pendiente (API)

Tomar un `spp_con`/`spp_srv`/`srv_licencia` validos de una solicitud existente:

```bash
docker compose exec db psql -U postgres -d dapatlqdb -t -c "SELECT spp_con, spp_srv, srv_licencia FROM solicitudpipas LIMIT 1;"
```

Crear la solicitud (login para token -> POST):

```bash
TOKEN=$(curl -s -X POST http://localhost:3000/api/v1/auth/login -d "username=admin&password=admin123" | python -c "import sys,json;print(json.load(sys.stdin)['access_token'])")
curl -s -X POST http://localhost:3000/api/v1/solicitudes_pipa \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"spp_con": <CON>, "spp_srv": <SRV>, "spp_vhe_id": 4, "spp_horaentrega": "2026-08-14T10:00:00", "srv_licencia": "<LIC>"}'
# -> devuelve spp_id (nuevo folio)
```

### 2. Poner coordenadas (el create no las acepta; se actualizan via SQL)

```bash
docker compose exec db psql -U postgres -d dapatlqdb -c "UPDATE solicitudpipas SET spp_latitud=20.64, spp_longitud=-103.311 WHERE spp_id=<NUEVO_ID>;"
```

### 3. Asignar vehiculo automaticamente

```bash
curl -s -X POST http://localhost:3000/api/v1/asignaciones/automatica \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"spp_id": <NUEVO_ID>}'
# -> 201 con vhe_id, distance_km, duration_min, pozo_sugerido
```

Verificar que quedo asignada y con ruta:

```bash
docker compose exec db psql -U postgres -d dapatlqdb -c "SELECT spp_id, spp_vhe_id, spp_estatus FROM solicitudpipas WHERE spp_id=<NUEVO_ID>;"
docker compose exec db psql -U postgres -d dapatlqdb -c "SELECT rtsol_id, rtsol_vhe_id, rtsol_distancia_km, rtsol_duracion_min FROM rutas_solicitud ORDER BY rtsol_id DESC LIMIT 1;"
```

### 4. Sesion 1 (supervisor) + Sesion 2 (chofer)

1. **Sesion 1** (normal): login `admin/admin123` -> `/mapa`. Ver la banderita `📍` de la solicitud
   nueva y `Conectado` en el WebSocket.
2. **Sesion 2** (ventana de incognito): abrir `/chofer?vehiculo=<vhe_id asignado>` -> login
   `admin/admin123`. Ver el listado "Mis entregas asignadas" con el folio nuevo, GPS activo y
   estado "Conectado".

### 5. Simular movimiento (el GPS real de un laptop no se mueve)

En la Sesion 2 (chofer): `DevTools (F12)` -> pestaña `Sensors` (o `More tools > Sensors`) ->
marcar **Override** y elegir una ubicacion, p. ej. `Lat 20.64, Lng -103.31`; despues cambiar a
`20.645, -103.305`. Cada cambio dispara `watchPosition`, el AppChofer envia por WS (cada 10 s) y
en Sesion 1 el marcador `🚚` se mueve a la nueva posicion.

> Si no funciona `watchPosition` con override (es mas estable en Chrome), cerrar y reabrir la
> Sesion 2 tras el override, o simular el chofer con el script de `websockets` del Miercoles
> enviando `{"vhe_id": <id>, "lat": ..., "lng": ..., "ts": "..."}`.

### 6. Evidencia en el backend

```bash
docker compose exec db psql -U postgres -d dapatlqdb -c "SELECT ubp_vhe_id, ubp_latitud, ubp_longitud, ubp_timestamp FROM ubicacionespipa ORDER BY ubp_id DESC LIMIT 5;"
```

**DoD:** la solicitud nueva aparece como banderita; tras la asignacion, el chofer la ve en su PWA;
y al simular movimiento el marcador `🚚` se mueve en vivo en `/mapa` (con filas nuevas en
`ubicacionespipa`).

---

## TAREA 2.29 — PWA instalable + offline

La instalacion/offline se prueban contra el **build de produccion** (el SW de Workbox precachea
los assets de `dist/`). En dev (`devOptions.enabled`) el SW existe pero el precache es parcial.

1. Build dentro del contenedor:

```bash
docker compose exec frontend npm run build
```

2. Servir `dist/` con `vite preview` (proxy de 2.27 activo, puerto 4173):

```bash
docker compose exec -d frontend npx vite preview --port 4173
# o: docker compose exec frontend npm run preview -- --port 4173
```

3. Abrir `http://localhost:4173`, login `admin/admin123`.

4. **Instalar**: DevTools > Application > Manifest (debe salir "Installability: installable") y
   Service Workers (SW activo). Usar el icono de instalacion de la barra de direcciones o
   `Three-dot menu > Install app...`. La PWA abre en pantalla completa (display standalone) con
   `start_url: /chofer`.

5. **Offline**: DevTools > Network > `Offline` (o en Service Workers marcar "Offline"). Recargar:
   la app carga desde el SW (shell precacheado). Con el chofer: en la PWA offline, si hay GPS,
   las posiciones se encolan en `localStorage` (`dapa_cola_ubicaciones`); al quitar `Offline`, el
   WS se reconecta (`onopen`) y drena la cola hacia el backend.

6. Verificar el drenado:

```bash
docker compose exec db psql -U postgres -d dapatlqdb -c "SELECT ubp_vhe_id, ubp_latitud, ubp_longitud, ubp_timestamp FROM ubicacionespipa ORDER BY ubp_id DESC LIMIT 5;"
```

**DoD:** PWA instalable desde el navegador, carga sin red y las posiciones encoladas llegan al
backend al reconectar.

---

## TAREA 2.30 — Suite completa en Docker

```bash
# Backend: pytest completo
docker compose exec backend python -m pytest -q

# Frontend: build de produccion (SW + manifest)
docker compose exec frontend npm run build
```

> **Estado conocido:** la suite del backend tiene 3 fallos PREEXISTENTES y ajenos a pipas en
> `tests/test_roles.py::test_list_roles_requires_admin`,
> `tests/test_users.py::test_list_users_requires_admin` y
> `tests/test_users.py::test_create_user_requires_admin` (permisos de rol admin). El resto queda en
> verde (64 passed). Si queda tiempo, corregirlos en su propio commit `fix:`, pero NO debe bloquear
> el E2E ni el PR de pipas.

**DoD:** pytest sin regresiones (mismo resultado o mejor que antes) y `npm run build` exitoso.

---

## TAREA 2.31 — PR a `feature/pipas-semana2`

> Estado actual del repo: las ramas de trabajo `feature/pipas-rutas|websocket|mapa|pwa` no existen
> como branches; los commits se fueron a `feature/modulopipasSemana1`. Se crea la rama de
> integracion del sprint para el PR.

1. Confirmar que todos los cambios del dia estan commiteados (solo cuando el usuario lo pida):

```bash
git status
git log --oneline -8
```

2. Crear y subir la rama de integracion del sprint:

```bash
git checkout -b feature/pipas-semana2
git push -u origin feature/pipas-semana2
```

3. Commits con prefijo `feat:`/`fix:` (uno por unidad logica, NO todo en un solo commit):

```bash
git add frontend/src/services/asignacionesService.js frontend/src/services/solicitudesService.js frontend/src/services/api.js frontend/src/services/authService.js
git commit -m "feat(pipas): servicios de frontend para solicitudes y asignaciones (2.26)"

git add frontend/vite.config.js docker-compose.yml frontend/src/pages/MapaSupervision.jsx frontend/src/pages/AppChofer.jsx
git commit -m "fix(pipas): proxy de Vite para /api y /ws, sin errores de CORS (2.27)"

git add .
git commit -m "feat(pipas): validacion E2E y PWA instalable/offline (2.28-2.29)"
```

4. Abrir el PR hacia la rama base del proyecto (`develop` o la que indique el proyecto):

```bash
gh pr create --base develop --head feature/pipas-semana2 --title "feat(pipas): integracion semana 2 - rutas, asignacion, mapa y PWA del chofer" --body "Sprint 2 Semana 2 completa (2.1-2.31). Incluye OSRM, asignacion automatica, WebSocket de ubicacion, mapa de supervision y PWA del chofer con GPS y modo offline. E2E validado en 2 sesiones."
```

5. Checklist de revision en el PR:
   - [ ] OSRM responde `code: Ok` y `POST /asignaciones/automatica` devuelve ruta + pozo sugerido.
   - [ ] WS autentica, persiste y hace broadcast (pruebas de la semana en verde).
   - [ ] `/mapa` y `/chofer?vehiculo=N` dentro del dashboard, sin errores de CORS.
   - [ ] E2E documentado en 2.28 y PWA instalable/offline (2.29).
   - [ ] `pytest` sin regresiones y `npm run build` OK (2.30).

**DoD:** PR abierto contra la rama base con commits `feat:`/`fix:` y checklist de revision llenado.

---

## Verificacion del dia (TODO EN DOCKER)

```bash
# 1. Servicios arriba
docker compose ps

# 2. OSRM responde rutas
curl "http://localhost:5000/route/v1/driving/-103.35,20.64;-103.31,20.67"

# 3. Proxy de Vite (2.27): /api y login sin CORS
curl -s -o NUL -w "%{http_code}\n" http://localhost:3000/api/v1/vehiculos   # 401 = OK

# 4. Backend sin regresiones (2.30)
docker compose exec backend python -m pytest -q

# 5. Build del frontend (2.30) - genera dist/sw.js
docker compose exec frontend npm run build

# 6. Posiciones persistidas del E2E (2.28/2.29)
docker compose exec db psql -U postgres -d dapatlqdb -c "SELECT ubp_vhe_id, ubp_latitud, ubp_longitud FROM ubicacionespipa ORDER BY ubp_id DESC LIMIT 5;"
```

Chequeos manuales (navegador):
1. Sesion 1 `/mapa`: banderita de la solicitud nueva + marcador `🚚` que se mueve al simular
   movimiento en la Sesion 2 (`/chofer?vehiculo=<id>`).
2. PWA instalable y carga offline desde `http://localhost:4173` (build de produccion).
3. Consola del navegador sin errores de CORS.
4. `/docs` del backend sigue OK.

## DoD del dia + commit

- [ ] 2.26 `asignacionesService.js` + `solicitudesService.js` (crearSolicitud) listos.
- [ ] 2.27 proxy de Vite (`/api` + `/ws`) en dev y preview; sin errores de CORS; env `REACT_APP_API_URL` eliminada.
- [ ] 2.28 E2E: solicitud -> asignacion -> PWA del chofer -> marcador moviendose en `/mapa`.
- [ ] 2.29 PWA instalable y funcional sin red (cola de posiciones recuperada).
- [ ] 2.30 `pytest` sin regresiones y `npm run build` exitoso.
- [ ] 2.31 PR a `feature/pipas-semana2` abierto con commits `feat:`/`fix:`.

Commit sugerido:

```bash
git add -A
git commit -m "feat(pipas): integracion y pruebas E2E del modulo de pipas (2.26-2.31)"
git push origin feature/pipas-semana2
```
