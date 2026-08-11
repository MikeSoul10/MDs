# HOJA DE INSTRUCCIONES — MODULO DE PIPAS — SPRINT 2 — DIA 4 (JUEVES)

**Autor:** Turno (Lider de turno)
**Fecha:** 2026-08-13
**Repositorio:** dapa2w — ramas `feature/pipas-mapa` y `feature/pipas-pwa`
**Objetivo del dia:** Mapa de supervision en React Leaflet (banderitas de solicitudes +
posicion en vivo de vehiculos via WebSocket) y la PWA del chofer (AppChofer) con
geolocalizacion GPS, envio de posicion por WebSocket y modo offline.

> **IMPORTANTE (convencion del sprint): TODAS las instalaciones, builds y pruebas se hacen
> DENTRO del contenedor Docker.** Nunca `npm install`/`npm run build`/`pytest` en la maquina
> host. Usar `docker compose exec frontend ...` y `docker compose exec backend ...`.

---

## Tareas del dia

| # | Tarea | Archivo(s) principal(es) | DoD |
|---|-------|--------------------------|-----|
| 2.17 | Instalar `leaflet` y `react-leaflet` dentro del contenedor frontend | `frontend/package.json` | `npm install` sin conflictos (React 18 => react-leaflet v4) |
| 2.18 | Crear `MapaSupervision.jsx` (pantalla completa) con capa base OpenStreetMap | `frontend/src/pages/` | Mapa renderiza tiles de OSM en `/mapa` |
| 2.19 | Banderitas de solicitudes pendientes/entregadas desde el servicio de solicitudes | `frontend/src/services/` + `frontend/src/pages/` | Marcadores visibles con tooltip del folio |
| 2.20 | Posicion en vivo de vehiculos conectada al WebSocket | `frontend/src/` | El marcador del vehiculo se mueve con el broadcast |
| 2.21 | Instalar `vite-plugin-pwa` + `manifest.webmanifest` + service worker | `frontend/` | La app es instalable (manifest + SW funcionando) |
| 2.22 | Crear `AppChofer.jsx`: PWA del chofer (login + solicitudes asignadas a su vehiculo) | `frontend/src/pages/` | El chofer ve solo sus entregas asignadas |
| 2.23 | Geolocalizacion GPS con `navigator.geolocation.watchPosition` | `frontend/src/` | La posicion se actualiza al moverse el vehiculo |
| 2.24 | Envio de posicion por WebSocket cada 5-10 s | `frontend/src/` | El backend persiste y retransmite la posicion |
| 2.25 | Modo offline: SW cachea la app y cola de posiciones al caerse la conexion | `frontend/` | La PWA carga offline y recupera posiciones al reconectar |

**DoD del dia:** `/mapa` renderiza banderitas de solicitudes y vehiculos en vivo; `/chofer`
(PWA) instalable, loguea al chofer, envia su GPS por WebSocket cada 5-10 s y funciona
offline; `npm run build` del frontend y `pytest` del backend en verde.

---

## 0. Prerequisitos y convenciones

- Continuar sobre lo MERGEADO del Martes y Miercoles (dominio de pipas, repos `asignacion/pozo/
  ubicacion_pipa`, WebSocket `/ws/ubicacion`). El backend ya expone `GET /api/v1/solicitudes_pipa`
  (PagedResponse), `GET /api/v1/vehiculos` y `GET /api/v1/asignaciones`.
- **Frontend**: React 18 + Vite 5 + Tailwind + react-router-dom v6. `react-leaflet` version 4
  (la v5 requiere React 19 y ROMPE el proyecto). `leaflet` ^1.9.
- **Todas las instalaciones dentro del contenedor:**
  - Instalar paquete nuevo en la imagen: `docker compose up -d --build frontend` (el `Dockerfile`
    ejecuta `npm install` con el `package.json` actualizado). Para probar en caliente sin rebuild:
    `docker compose exec frontend npm install <paquete>`.
  - Verificar: `docker compose exec frontend node -v` y
    `docker compose exec frontend node -e "require('leaflet'); require('react-leaflet'); console.log('ok')"`.
  - Build/verificacion: `docker compose exec frontend npm run build`.
- **Variables de entorno (Vite)**: `api.js` usa `import.meta.env.VITE_API_URL` (prefijo `VITE_`,
  no `REACT_`). Fuera de Docker el navegador llama a `http://localhost:8000/api/v1` (default de
  `api.js`). El token se guarda en `localStorage` bajo la llave `access_token` (ver
  `frontend/src/services/api.js` y `authService.js`). Nota: el compose define
  `REACT_APP_API_URL=http://backend:8000` que Vite ignora — eso se resuelve el VIERNES (tarea 2.27),
  hoy no bloquea.
- **WebSocket**: URL derivada del API base: `ws://localhost:8000/ws/ubicacion?token=<JWT>`
  (reemplazar `http` por `ws` y quitar `/api/v1`). Los middlewares del backend dejan pasar el scope
  websocket. El protocolo de mensaje es el del Miercoles:
  - Cliente -> servidor: `{"vhe_id": 4, "lat": 20.64, "lng": -103.31, "ts": "<ISO opcional>"}`
  - Servidor -> todos: `{"type":"bienvenido"}` / `{"type":"ack", "vhe_id": N}` / `{"type":"ubicacion", "vhe_id": N, "lat":..., "lng":..., "ts":...}`
- **Habilitador backend necesario para 2.19**: el schema `SolicitudPipaResponse`
  (`backend/app/api/schemas/solicitudes_pipa.py`) aun NO expone las coordenadas, aunque la
  migracion `100057` ya agrego `spp_latitud`/`spp_longitud` al modelo. Agregarlas como opcionales
  (snippet en 2.19) para que el mapa pueda pintar banderitas.
- **Iconos de Leaflet**: los iconos default `L.Icon` se rompen con bundlers. Para evitar assets,
  usar `L.divIcon` con emoji/estilo CSS (banderita `📍`, vehiculo `🚚`). No importar PNGs.
- **React StrictMode** esta activo (`main.jsx`): los efectos corren 2 veces en desarrollo;
  usar refs para el WS y limpiar en `useEffect` (return cleanup).
- Pruebas del dia: manuales en navegador (2 sesiones) + `pytest` del backend sin regresiones.
  No hay suite de tests de frontend en el repo; la verificacion es `npm run build` + prueba manual.

---

## TAREA 2.17 — Instalar leaflet + react-leaflet (EN DOCKER)

> **ADVERTENCIA (ya corregido en el repo):** el paquete se llama `leaflet`, NO `leaf`. Con el typo
> `"leaf": "^1.9.4"` npm instala un paquete distinto y `import 'leaflet'` nunca resuelve.
> `frontend/package.json` y `frontend/package-lock.json` YA estan montados en el compose
> (misma mecanica que `vite.config.js`), asi que `docker compose exec frontend npm install <paquete>`
> ahora PERSISTE los cambios al `package.json`/`package-lock.json` del repo. De todas formas, la
> instalacion durable en la imagen se hace con `docker compose up -d --build frontend` (npm install
> dentro del Dockerfile).

1. Editar `frontend/package.json` agregando en `dependencies`:

```json
"leaflet": "^1.9.4",
"react-leaflet": "^4.2.1"
```

2. Rebuild de la imagen (instala dentro del contenedor):

```bash
docker compose up -d --build frontend
```

> Alternativa en caliente (persiste al repo gracias a los montajes, pero la imagen se vuelve a
> instalar en el build final):
> `docker compose exec frontend npm install leaflet@^1.9.4 react-leaflet@^4.2.1`

3. Verificar:

```bash
docker compose exec frontend node -v                       # node 18
docker compose exec frontend node -e "require('leaflet'); require('react-leaflet'); console.log('ok')"
```

**DoD:** no hay conflicto de dependencias y `require('react-leaflet')` responde `ok`.

---

## TAREA 2.18 — `MapaSupervision.jsx` (pantalla completa)

Crear `frontend/src/pages/MapaSupervision.jsx`:

```jsx
import { MapContainer, TileLayer, Marker, Popup } from 'react-leaflet';
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';
import { useEffect, useState, useRef } from 'react';
import { getSolicitudes } from '../services/solicitudesService';

// Centro de Tlaquepaque (misma base que el backend: 20.6400, -103.3110)
const CENTRO = [20.64, -103.311];

// divIcon: evita el problema de los iconos default de Leaflet con bundlers
export function crearIcono(emoji, clase = '') {
  return L.divIcon({
    className: 'bg-transparent',
    html: `<div class="icono-mapa ${clase}" style="font-size:22px;line-height:1;">${emoji}</div>`,
    iconSize: [24, 24],
    iconAnchor: [12, 12],
  });
}

const iconoSolicitud = {
  Pendiente: crearIcono('📍', 'solicitud-pendiente'),
  'En Ruta': crearIcono('🚗', 'solicitud-en-ruta'),
  Entregada: crearIcono('✅', 'solicitud-entregada'),
};

export default function MapaSupervision() {
  const [solicitudes, setSolicitudes] = useState([]);
  const [vehiculos, setVehiculos] = useState({}); // vhe_id -> {lat, lng}
  const wsRef = useRef(null);

  // 2.19 - banderitas de solicitudes
  useEffect(() => {
    getSolicitudes({ page: 1, page_size: 200, estatus: 'Pendiente' })
      .then((data) => setSolicitudes(data.items))
      .catch(() => {});
  }, []);

  // 2.20 - WebSocket de ubicacion en vivo
  useEffect(() => {
    const token = localStorage.getItem('access_token'); // misma llave que services/api.js
    if (!token) return;
    const ws = new WebSocket(`ws://localhost:8000/ws/ubicacion?token=${token}`);
    wsRef.current = ws;
    ws.onmessage = (evt) => {
      const msg = JSON.parse(evt.data);
      if (msg.type === 'ubicacion') {
        setVehiculos((prev) => ({ ...prev, [msg.vhe_id]: { lat: msg.lat, lng: msg.lng } }));
      }
    };
    return () => ws.close();
  }, []);

  return (
    <div className="h-screen w-screen">
      <MapContainer center={CENTRO} zoom={14} style={{ height: '100%', width: '100%' }}>
        <TileLayer
          attribution='&copy; OpenStreetMap contributors'
          url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
        />
        {solicitudes.filter((s) => s.spp_latitud && s.spp_longitud).map((s) => (
          <Marker
            key={s.spp_id}
            position={[s.spp_latitud, s.spp_longitud]}
            icon={iconoSolicitud[s.spp_estatus] || iconoSolicitud.Pendiente}
          >
            <Popup>
              Folio #{s.spp_id} — {s.spp_estatus}
            </Popup>
          </Marker>
        ))}
        {Object.entries(vehiculos).map(([vheId, pos]) => (
          <Marker
            key={`v-${vheId}`}
            position={[pos.lat, pos.lng]}
            icon={crearIcono('🚚')}
          >
            <Popup>Vehiculo {vheId}</Popup>
          </Marker>
        ))}
      </MapContainer>
    </div>
  );
}
```

Nota: para el centro de la pipa usar `MapContainer center={[20.64, -103.311]}`.

Registrar la ruta en `frontend/src/App.jsx` (fuera del `DashboardLayout`, pantalla completa,
detras de `ProtectedRoute`):

```jsx
import MapaSupervision from './pages/MapaSupervision';
// dentro de <Routes>:
<Route path="/mapa" element={<ProtectedRoute><MapaSupervision /></ProtectedRoute>} />
```

**DoD:** `http://localhost:3000/mapa` muestra el mapa con tiles de OSM y el centro en Tlaquepaque.

---

## TAREA 2.19 — Banderitas de solicitudes (servicio + schema backend)

### Crear `frontend/src/services/solicitudesService.js`

```js
import { api } from './api';

export function getSolicitudes(params = {}) {
  const qs = new URLSearchParams(params).toString();
  return api.get(`/solicitudes_pipa${qs ? `?${qs}` : ''}`);
}

export function cambiarEstadoSolicitud(id, body) {
  return api.patch(`/solicitudes_pipa/${id}/estado`, body);
}
```

### Habilitador backend (obligatorio): exponer coordenadas en el response

Editar `backend/app/api/schemas/solicitudes_pipa.py` — agregar a `SolicitudPipaResponse`
(son opcionales; el modelo ya las tiene desde la migracion `100057`):

```python
    spp_latitud: Optional[float] = None
    spp_longitud: Optional[float] = None
```

> El frontend filtra `s.spp_latitud && s.spp_longitud` y usa `s.spp_id` como "folio".
> Las solicitudes del seed no tienen coordenadas: para la prueba manual actualizarlas via SQL:
> `docker compose exec db psql -U postgres -d dapatlqdb -c "UPDATE solicitudpipas SET
> spp_latitud=20.64, spp_longitud=-103.311 WHERE spp_id IN (1,2);"`

**DoD:** en `/mapa` aparecen las banderitas `📍` (Pendiente), `✅` (Entregada) con tooltip del
folio al hacer clic.

---

## TAREA 2.20 — Posicion en vivo de vehiculos

Cubierta por el `useEffect` del WebSocket en `MapaSupervision.jsx` (2.18):
- Conecta a `ws://localhost:8000/ws/ubicacion?token=<access_token>`.
- Al recibir `{"type":"ubicacion", ...}` actualiza el estado `vehiculos[vhe_id]`.
- Los `Marker` de vehiculos se redibujan en la nueva posicion (el marcador "se mueve").

Mejora opcional: `MapContainer` con `useMap()` y `flyTo` cuando cambia la posicion del vehiculo
seleccionado.

**DoD:** abriendo `/mapa` y `/chofer?vehiculo=4` en 2 sesiones, el marcador `🚚` se mueve en vivo
cuando el chofer envia posiciones.

---

## TAREA 2.21 — PWA: vite-plugin-pwa + manifest + service worker

### 1. Instalar (dentro del contenedor)

Agregar a `devDependencies` de `frontend/package.json`: `"vite-plugin-pwa": "^0.20.5"`, y rebuild:

```bash
docker compose up -d --build frontend
```

### 2. Crear `frontend/public/manifest.webmanifest`

```json
{
  "name": "DAPA2W Chofer",
  "short_name": "DAPA2W",
  "description": "PWA del chofer de pipas - envio de ubicacion GPS",
  "start_url": "/chofer",
  "scope": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#0066cc",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

> Los iconos `frontend/public/icons/icon-192.png` y `icon-512.png` deben existir para que la app
> sea instalable (generar un PNG simple de 192/512 px con el logo del proyecto).

### 3. Configurar `frontend/vite.config.js`

```js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    react(),
    VitePWA({
      registerType: 'autoUpdate',
      manifest: false, // el manifest se sirve desde /public/manifest.webmanifest
      workbox: {
        globPatterns: ['**/*.{js,css,html,svg,png,webmanifest}'],
        maximumFileSizeToCacheInBytes: 5 * 1024 * 1024,
      },
      devOptions: { enabled: true },
    }),
  ],
  server: { port: 3000, host: true },
});
```

### 4. Editar `frontend/index.html`

```html
<link rel="manifest" href="/manifest.webmanifest" />
<meta name="theme-color" content="#0066cc" />
<meta name="apple-mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-status-bar-style" content="default" />
```

### 5. Registrar el service worker en `frontend/src/main.jsx`

```jsx
import { registerSW } from 'virtual:pwa-register';
registerSW({ immediate: true });
```

**DoD:** `docker compose exec frontend npm run build` genera `dist/sw.js` y el manifest; en el
navegador aparece el icono "Instalar app" y DevTools > Application muestra el SW activo.

---

## TAREA 2.22 — `AppChofer.jsx` (PWA del chofer)

Crear `frontend/src/pages/AppChofer.jsx`. Es una pagina autonoma (sin `DashboardLayout`):
- Si no hay token en `localStorage`, muestra un login reutilizando `login` de
  `frontend/src/services/authService.js` (mismo POST form-encoded).
- Lee el vehiculo del chofer del query param `?vehiculo=<vhe_id>` (ej. `/chofer?vehiculo=4`).
- Lista SOLO las solicitudes asignadas a ese vehiculo: `getSolicitudes()` y filtrar
  `spp_vhe_id === vheId` (y opcionalmente `GET /asignaciones` para ver la ruta/pozo sugerido).
- Muestra el estado de conexion del WebSocket y del GPS.

```jsx
import { useEffect, useRef, useState } from 'react';
import { useSearchParams } from 'react-router-dom';
import { Droplets, MapPin, Wifi, WifiOff } from 'lucide-react';
import { authStorage, login } from '../services/authService';
import { getSolicitudes } from '../services/solicitudesService';

export default function AppChofer() {
  const [params] = useSearchParams();
  const vheId = Number(params.get('vehiculo')) || 4;
  const [token, setToken] = useState(authStorage.getAccessToken());
  const [user, setUser] = useState(authStorage.getUser());
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [solicitudes, setSolicitudes] = useState([]);
  const [pos, setPos] = useState(null); // {lat, lng}
  const [conectado, setConectado] = useState(false);
  const [offline, setOffline] = useState(!navigator.onLine);
  const wsRef = useRef(null);
  const posRef = useRef(null);

  const handleLogin = async (e) => {
    e.preventDefault();
    try {
      const data = await login({ username, password });
      setToken(data.access_token);
      setUser(data.user);
    } catch (err) {
      setError(err.message);
    }
  };

  // 2.22 - solicitudes asignadas a MI vehiculo
  useEffect(() => {
    if (!token) return;
    getSolicitudes({ page: 1, page_size: 200 })
      .then((data) => setSolicitudes(data.items.filter((s) => s.spp_vhe_id === vheId)))
      .catch(() => {});
  }, [token, vheId]);

  // 2.24 - WebSocket para enviar posicion
  useEffect(() => {
    if (!token) return;
    let ws;
    const conectar = () => {
      ws = new WebSocket(`ws://localhost:8000/ws/ubicacion?token=${token}`);
      ws.onopen = () => setConectado(true);
      ws.onclose = () => { setConectado(false); setTimeout(conectar, 5000); };
      ws.onerror = () => ws.close();
    };
    conectar();
    wsRef.current = { close: () => ws && ws.close() };
    return () => ws && ws.close();
  }, [token]);

  // 2.23 - geolocalizacion GPS
  useEffect(() => {
    if (!navigator.geolocation) return;
    const watchId = navigator.geolocation.watchPosition(
      (p) => setPos({ lat: p.coords.latitude, lng: p.coords.longitude }),
      () => {},
      { enableHighAccuracy: true, timeout: 10000, maximumAge: 5000 },
    );
    return () => navigator.geolocation.clearWatch(watchId);
  }, []);

  // 2.24 - envio cada 10 s (o al mover) + cola offline (2.25)
  useEffect(() => {
    if (!token) return;
    const timer = setInterval(() => {
      if (wsRef.current && wsRef.current.readyState === WebSocket.OPEN) {
        const p = posRef.current || pos;
        if (p) wsRef.current.send(JSON.stringify({ vhe_id: vheId, lat: p.lat, lng: p.lng, ts: new Date().toISOString() }));
      }
    }, 10000);
    return () => clearInterval(timer);
  }, [token, vheId, pos]);

  // 2.25 - modo offline: deteccion + cola en localStorage
  useEffect(() => {
    const onLine = () => setOffline(false);
    const offLine = () => setOffline(true);
    window.addEventListener('online', onLine);
    window.addEventListener('offline', offLine);
    return () => { window.removeEventListener('online', onLine); window.removeEventListener('offline', offLine); };
  }, []);

  if (!token) {
    return (
      <form onSubmit={handleLogin} className="min-h-screen flex flex-col items-center justify-center gap-3 p-6">
        <Droplets size={40} />
        <h1 className="text-xl font-bold">DAPA2W Chofer</h1>
        {error && <p className="text-red-500 text-sm">{error}</p>}
        <input value={username} onChange={(e) => setUsername(e.target.value)} placeholder="usuario" className="border p-2 rounded w-64" />
        <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} placeholder="contraseña" className="border p-2 rounded w-64" />
        <button type="submit" className="bg-blue-600 text-white px-6 py-2 rounded">Entrar</button>
      </form>
    );
  }

  return (
    <div className="min-h-screen p-4 space-y-4" style={{ backgroundColor: 'var(--color-surface)' }}>
      <header className="flex items-center justify-between">
        <div className="flex items-center gap-2">
          <Droplets size={20} />
          <h1 className="font-bold">Chofer — Vehiculo {vheId}</h1>
        </div>
        <div className="flex items-center gap-2 text-sm">
          {offline ? <WifiOff /> : <Wifi />}
          <span>{offline ? 'Sin red' : conectado ? 'Conectado' : 'Reconectando…'}</span>
          <MapPin />
          <span>{pos ? `${pos.lat.toFixed(5)}, ${pos.lng.toFixed(5)}` : 'Sin GPS'}</span>
        </div>
      </header>
      <section>
        <h2 className="font-semibold mb-2">Mis entregas asignadas</h2>
        <ul className="space-y-2">
          {solicitudes.map((s) => (
            <li key={s.spp_id} className="border rounded p-3 flex justify-between">
              <span>Folio #{s.spp_id}</span>
              <span>{s.spp_estatus}</span>
            </li>
          ))}
          {solicitudes.length === 0 && <li className="text-sm text-gray-500">Sin entregas asignadas.</li>}
        </ul>
      </section>
    </div>
  );
}
```

Registrar en `frontend/src/App.jsx` (FUERA del dashboard, sin `ProtectedRoute` propio — la pagina
maneja su propio login):

```jsx
import AppChofer from './pages/AppChofer';
// dentro de <Routes>:
<Route path="/chofer" element={<AppChofer />} />
```

> El `start_url` del manifest apunta a `/chofer`, por lo que la PWA abre directo el AppChofer.

**DoD:** en `/chofer?vehiculo=4` el chofer ve solo las solicitudes con `spp_vhe_id === 4`.

---

## TAREA 2.23 — Geolocalizacion GPS

Cubierta por `navigator.geolocation.watchPosition` en `AppChofer.jsx` (2.22):
- `enableHighAccuracy: true`, `timeout: 10000`, `maximumAge: 5000` (la posicion se actualiza al
  moverse el vehiculo).
- Guardar siempre la ultima posicion en un `ref` (`posRef`) para no perderla entre renders.
- Manejar el error de permisos (`error.code === 1`) mostrando "GPS no permitido".
- En Chrome, `watchPosition` requiere HTTPS o `localhost` — en dev localhost funciona.

**DoD:** al mover el dispositivo/vehiculo, el estado `pos` cambia (visible en el encabezado).

---

## TAREA 2.24 — Envio de posicion por WebSocket cada 5-10 s

Cubierto por el `useEffect` de intervalo en `AppChofer.jsx`:
- Cada 10 s, si el WS esta `OPEN` y hay posicion, enviar:
  `{"vhe_id": <id>, "lat": ..., "lng": ..., "ts": "<ISO>"}` (protocolo del Miercoles).
- Reconexion automatica con `onclose -> setTimeout(conectar, 5000)`.
- El backend persiste UNA fila por mensaje (2.13) y hace broadcast (2.14) — el mapa de supervision
  recibe la posicion en vivo.

**DoD:** verificar en el backend:
```bash
docker compose exec db psql -U postgres -d dapatlqdb -c "SELECT ubp_vhe_id, ubp_latitud, ubp_longitud, ubp_timestamp FROM ubicacionespipa ORDER BY ubp_id DESC LIMIT 5;"
```

---

## TAREA 2.25 — Modo offline (SW + cola de posiciones)

- **Cacheo de la app**: el SW de `vite-plugin-pwa` (Workbox `globPatterns`) precachea JS/CSS/HTML;
  con `registerType: 'autoUpdate'` se actualiza al publicar. La app carga offline.
- **Cola de posiciones**: cuando `navigator.onLine === false` (o WS cerrado), encolar en
  `localStorage` la posicion con su marca de tiempo; al volver (`online` event + WS `OPEN`),
  drenar la cola enviando los mensajes pendientes.

Snippet de cola para `AppChofer.jsx`:

```js
const COLA_KEY = 'dapa_cola_ubicaciones';

function encolarPosicion(msg) {
  const cola = JSON.parse(localStorage.getItem(COLA_KEY) || '[]');
  cola.push(msg);
  localStorage.setItem(COLA_KEY, JSON.stringify(cola));
}

function drenarCola(ws) {
  const cola = JSON.parse(localStorage.getItem(COLA_KEY) || '[]');
  while (cola.length) {
    ws.send(JSON.stringify(cola.shift()));
  }
  localStorage.setItem(COLA_KEY, JSON.stringify(cola));
}
```

Enviar por la cola cuando `!navigator.onLine || ws no abierto`; llamar `drenarCola` en `onopen`
y en el listener de `online`.

**DoD:** con DevTools > Network en "Offline", la PWA recarga; al reconectar, las posiciones
encoladas llegan al backend (aparecen en `ubicacionespipa`).

---

## Verificacion del dia (TODO EN DOCKER)

```bash
# Backend sin regresiones
docker compose exec backend python -m pytest -q

# Frontend build (genera SW + manifest en dist/)
docker compose exec frontend npm run build

# Dependencias instaladas
docker compose exec frontend node -e "console.log('leaflet', require('leaflet/package.json').version, '| react-leaflet', require('react-leaflet/package.json').version)"
```

Chequeos manuales (navegador):
1. `http://localhost:3000/mapa`: mapa OSM centrado en Tlaquepaque, banderitas de solicitudes con
   folio y marcadores de vehiculos.
2. Abrir `/chofer?vehiculo=4` en otra pestana: login (admin/admin123), listado de entregas de
   ese vehiculo, GPS activo y "Conectado".
3. La posicion del chofer aparece en el mapa (2.20) y una fila nueva por mensaje en
   `ubicacionespipa` (2.24).
4. "Instalar app" en el navegador (PWA instalable, 2.21).
5. DevTools > Network Offline: la PWA recarga offline y encola posiciones; al reconectar se
   vacia la cola (2.25).
6. `/docs` del backend sigue OK (sin regresiones por el cambio del schema de solicitudes).

## DoD del dia + commit

- [ ] 2.17 leaflet + react-leaflet v4 instalados en el contenedor (build OK).
- [ ] 2.18 `/mapa` con tiles de OSM a pantalla completa.
- [ ] 2.19 banderitas de solicitudes con tooltip del folio (schema backend expone lat/lng).
- [ ] 2.20 marcador del vehiculo se mueve con el broadcast del WebSocket.
- [ ] 2.21 PWA instalable (manifest + SW) y `npm run build` genera `dist/sw.js`.
- [ ] 2.22 `/chofer` muestra solo las entregas asignadas al vehiculo.
- [ ] 2.23 posicion GPS se actualiza al moverse.
- [ ] 2.24 posicion enviada cada ~10 s y persistida en `ubicacionespipa`.
- [ ] 2.25 carga offline y cola de posiciones recuperada al reconectar.
- [ ] `pytest` del backend en verde (sin regresiones).

Commit sugerido:

```bash
git add -A
git commit -m "feat(pipas): mapa de supervision con leaflet y PWA del chofer con GPS y modo offline (2.17-2.25)"
git push origin feature/pipas-mapa   # o feature/pipas-pwa
```
