# HOJA DE INSTRUCCIONES — MODULO DE PIPAS — SPRINT 3 — DIA 1 (LUNES)

**Autor:** Turno (Lider de turno)
**Fecha:** 2026-08-17
**Repositorio:** dapa2w — rama `feature/pipas-dashboard`
**Objetivo del dia:** Tablero logístico (Kanban) de solicitudes de pipa para Sandra:
columnas Pendiente / En Ruta / Entregada / Cancelada, tarjeta reutilizable de solicitud,
acciones (asignar vehículo, cambiar estado, cancelar con motivo), drag & drop que llama al
endpoint de cambio de estado, y filtros por fecha/vehículo/estado con contadores por columna.

> **IMPORTANTE (convencion del sprint): TODAS las instalaciones, builds y pruebas se hacen
> DENTRO del contenedor Docker.** Nunca `npm install`/`npm run build`/`pytest` en la maquina
> host. Usar `docker compose exec frontend ...` y `docker compose exec backend ...`.

---

## Tareas del dia

| # | Tarea | Archivo(s) principal(es) | DoD |
|---|-------|--------------------------|-----|
| 3.1 | Crear `DashboardPipas.jsx`: tablero Kanban de solicitudes (Pendiente / En Ruta / Entregada / Cancelada) | `frontend/src/pages/` | Las solicitudes se agrupan por estado desde la API |
| 3.2 | Componente reutilizable `SolicitudCard.jsx` (folio, vehículo, domicilio, hora de entrega) | `frontend/src/components/common/` | Tarjeta funcional con acciones |
| 3.3 | Acciones en la tarjeta: Asignar vehículo, Cambiar estado, Cancelar (con motivo) | `frontend/src/` | Cada acción llama al endpoint correspondiente |
| 3.4 | Drag & drop entre columnas que llama a `PATCH /solicitudes_pipa/{id}/estado` | `frontend/src/` | El estado cambia en BD y se refleja en el tablero |
| 3.5 | Filtros por fecha, vehículo y estado + contador por columna | `frontend/src/` | Filtros funcionales |

**DoD del dia:** `/pipas` muestra el Kanban con las solicitudes agrupadas por estado, contadores
por columna, acciones que persisten en BD y drag & drop funcional; `npm run build` del frontend y
`pytest` del backend en verde.

---

## 0. Prerequisitos y convenciones

- Continuar sobre lo MERGEADO de las Semanas 1 y 2 (dominio de pipas, repos `asignacion/pozo/
  ruta_solicitud/ubicacion_pipa`, WebSocket `/ws/ubicacion`, mapa `/mapa` y PWA `/chofer`).
- **Endpoints backend ya disponibles** (`http://localhost:8000/api/v1`):
  - `GET /solicitudes_pipa` — `PagedResponse`, filtros `estado`, `fecha_desde`, `fecha_hasta`,
    `page`, `page_size`. **OJO: el filtro se llama `estado`, NO `estatus`.**
  - `POST /solicitudes_pipa` — alta (requiere `spp_vhe_id`).
  - `PATCH /solicitudes_pipa/{id}/estado` — body `{estado_nuevo, motivo}`. Registra bitácora en
    `historial_solicitud`. Reglas de transición: `Pendiente -> En Ruta -> Entregada`, y
    cualquier estado `-> Cancelada`; transición inválida regresa 400.
  - `GET /vehiculos` (paginado) y `GET /vehiculos/disponibles` — `vhe_id`, `vhe_descripcion`, `vhe_modelo`.
  - `POST /asignaciones/automatica` — body `{spp_id}`. Asigna el vehículo más cercano a una
    solicitud `Pendiente` con coordenadas (`spp_latitud`/`spp_longitud`), crea ruta + asignación.
- **Campos del response de solicitud** (`SolicitudPipaResponse`): `spp_id` (folio), `spp_con`,
  `spp_srv`, `spp_vhe_id`, `spp_fechasolicitud`, `spp_horaentrega`, `srv_licencia`, `spp_estatus`,
  `spp_latitud`, `spp_longitud`. **No incluye** la descripción del vehículo ni el domicilio:
  - Vehículo: mapear `vhe_id -> vhe_descripcion` con `GET /vehiculos`.
  - Domicilio: no hay endpoint de servicios/contribuyentes en v1 todavía; para el DoD basta con
    folio + vehículo + hora de entrega + estado (el campo domicilio se puede dejar como "—" o
    agregarse cuando exista el endpoint).
- **Servicios frontend ya existentes**: `frontend/src/services/solicitudesService.js` tiene
  `getSolicitudes(params)` y `cambiarEstadoSolicitud(id, body)`. **Falta** `vehiculosService.js`
  (se crea en la tarea 3.1).
- **Frontend**: React 18 + Vite 5 + Tailwind + react-router-dom v6 + `lucide-react` ^0.400 +
  `recharts`. NO hay librería de drag & drop instalada. Para evitar dependencias nuevas se usa
  **HTML5 drag & drop nativo** (`draggable`, `onDragStart`, `onDrop`). Si el equipo prefiere
  `@dnd-kit/core`, la instalación es DENTRO del contenedor:
  `docker compose exec frontend npm install @dnd-kit/core @dnd-kit/sortable` + rebuild.
- **API base y token**: `api.js` usa `import.meta.env.VITE_API_URL || '/api/v1'`. El token se
  guarda en `localStorage` bajo la llave `access_token`. Usuarios seed: `admin` / `admin123`.
- **React StrictMode** está activo (`main.jsx`): los efectos corren 2 veces en desarrollo; usar
  refs y limpiar en el `return` del `useEffect`.
- Pruebas del día: manuales en navegador + `pytest` del backend sin regresiones. No hay suite de
  tests de frontend; la verificación es `npm run build` + prueba manual.

---

## TAREA 3.1 — `DashboardPipas.jsx` (tablero Kanban)

1. Crear `frontend/src/services/vehiculosService.js`:

```js
import { api } from './api';

export function getVehiculos(params = {}) {
  const qs = new URLSearchParams(params).toString();
  return api.get(`/vehiculos${qs ? `?${qs}` : ''}`);
}

export function getVehiculosDisponibles() {
  return api.get('/vehiculos/disponibles');
}
```

2. Crear `frontend/src/pages/DashboardPipas.jsx`. Estructura mínima:

```jsx
import { useEffect, useState } from 'react';
import { Truck, Droplets } from 'lucide-react';
import { getSolicitudes } from '../services/solicitudesService';
import SolicitudCard from '../components/common/SolicitudCard';

const COLUMNAS = ['Pendiente', 'En Ruta', 'Entregada', 'Cancelada'];

export default function DashboardPipas() {
  const [solicitudes, setSolicitudes] = useState([]);
  const [vehiculos, setVehiculos] = useState({}); // vhe_id -> vhe_descripcion
  const [loading, setLoading] = useState(true);
  const [fechaDesde, setFechaDesde] = useState('');
  const [fechaHasta, setFechaHasta] = useState('');
  const [vehiculoFiltro, setVehiculoFiltro] = useState('');
  const [draggingId, setDraggingId] = useState(null); // 3.4

  const cargar = () => {
    setLoading(true);
    getSolicitudes({ page: 1, page_size: 200 })
      .then((d) => setSolicitudes(d.items))
      .catch(console.error)
      .finally(() => setLoading(false));
  };

  useEffect(() => { cargar(); }, []);

  useEffect(() => {
    getVehiculos({ page: 1, page_size: 100 })
      .then((d) => {
        const mapa = {};
        d.items.forEach((v) => { mapa[v.vhe_id] = v.vhe_descripcion; });
        setVehiculos(mapa);
      })
      .catch(() => {});
  }, []);

  // 3.5 - filtros client-side (fecha sobre spp_horaentrega / spp_fechasolicitud)
  const filtradas = solicitudes.filter((s) => {
    const fecha = (s.spp_horaentrega || s.spp_fechasolicitud || '').slice(0, 10);
    const okFecha =
      (!fechaDesde || fecha >= fechaDesde) && (!fechaHasta || fecha <= fechaHasta);
    const okVehiculo = !vehiculoFiltro || String(s.spp_vhe_id) === vehiculoFiltro;
    return okFecha && okVehiculo;
  });

  const agrupadas = (estado) => filtradas.filter((s) => s.spp_estatus === estado);

  // ... layout con las 4 columnas (ver 3.4 para los handlers de drop)
  return <div>...</div>;
}
```

3. Registrar la ruta en `frontend/src/App.jsx` (DENTRO de `DashboardLayout`, con `ProtectedRoute`):

```jsx
import DashboardPipas from './pages/DashboardPipas';
// dentro del <Route> del DashboardLayout:
<Route path="/pipas" element={<DashboardPipas />} />
```

4. Agregar el enlace en `frontend/src/components/common/Sidebar.jsx` bajo el menú `pipas`:

```jsx
{ id: 'tablero-pipas', label: 'Tablero Pipas', icon: Kanban, path: '/pipas' },
```

(lucide-react exporta `Kanban`.)

**DoD:** `http://localhost:3000/pipas` muestra las 4 columnas con las solicitudes agrupadas por
`spp_estatus` desde la API.

---

## TAREA 3.2 — `SolicitudCard.jsx` (componente reutilizable)

Crear `frontend/src/components/common/SolicitudCard.jsx`. Recibe la solicitud, el mapa de
vehículos y callbacks para las acciones (3.3) y el drag & drop (3.4):

```jsx
import { Droplets, Truck, CalendarClock } from 'lucide-react';

const COLORES = {
  Pendiente: '#eab308',
  'En Ruta': '#2563eb',
  Entregada: '#16a34a',
  Cancelada: '#dc2626',
};

export default function SolicitudCard({ solicitud, vehiculos = {}, onAsignar, onEstado, onCancelar }) {
  const s = solicitud;
  return (
    <div
      draggable
      onDragStart={(e) => e.dataTransfer.setData('text/spp_id', String(s.spp_id))}
      className="rounded border p-3 space-y-1.5 cursor-grab"
      style={{ backgroundColor: 'var(--color-surface)', borderColor: 'var(--color-border)' }}
    >
      <div className="flex items-center justify-between">
        <span className="text-xs font-bold">Folio #{s.spp_id}</span>
        <span className="text-[10px] px-1.5 py-0.5 rounded-full"
          style={{ backgroundColor: COLORES[s.spp_estatus] || '#888', color: '#fff' }}>
          {s.spp_estatus}
        </span>
      </div>
      <div className="flex items-center gap-1.5 text-xs">
        <Truck size={12} /> {vehiculos[s.spp_vhe_id] || `Vehiculo ${s.spp_vhe_id || '—'}`}
      </div>
      <div className="flex items-center gap-1.5 text-xs" style={{ color: 'var(--color-text-muted)' }}>
        <CalendarClock size={12} />
        {s.spp_horaentrega ? new Date(s.spp_horaentrega).toLocaleString() : 'Sin hora'}
      </div>
      {/* 3.3 - acciones */}
      <div className="flex gap-1 pt-1">
        {s.spp_estatus === 'Pendiente' && (
          <button onClick={() => onAsignar(s)} className="text-[10px] px-2 py-1 rounded border">Asignar</button>
        )}
        {s.spp_estatus === 'En Ruta' && (
          <button onClick={() => onEstado(s, 'Entregada')} className="text-[10px] px-2 py-1 rounded border">Marcar entregada</button>
        )}
        {s.spp_estatus !== 'Cancelada' && (
          <button onClick={() => onCancelar(s)} className="text-[10px] px-2 py-1 rounded border">Cancelar</button>
        )}
      </div>
    </div>
  );
}
```

> El `onDragStart` de la tarjeta usa `dataTransfer` con `spp_id`; la columna de destino lo lee en
> su `onDrop` (3.4).

**DoD:** la tarjeta muestra folio, vehículo (descripción mapeada), hora de entrega y estado, y
expone las acciones correspondientes según el estado.

---

## TAREA 3.3 — Acciones de la tarjeta

En `DashboardPipas.jsx`, implementar los tres handlers y pasarlos a `SolicitudCard`:

```jsx
import { asignacionAutomatica } from '../services/asignacionesService';

const recargar = () => cargar();

// Asignar vehículo -> POST /asignaciones/automatica
const onAsignar = async (s) => {
  try {
    await asignacionAutomatica(s.spp_id);   // asigna el vehículo más cercano
    recargar();
  } catch (err) { alert(err.message); }
};

// Cambiar estado -> PATCH /solicitudes_pipa/{id}/estado
const onEstado = async (s, estadoNuevo) => {
  try {
    await cambiarEstadoSolicitud(s.spp_id, { estado_nuevo: estadoNuevo });
    recargar();
  } catch (err) { alert(err.message); }
};

// Cancelar (con motivo) -> PATCH con estado_nuevo='Cancelada' y motivo
const onCancelar = (s) => {
  const motivo = window.prompt('Motivo de cancelacion:');
  if (!motivo) return;
  cambiarEstadoSolicitud(s.spp_id, { estado_nuevo: 'Cancelada', motivo })
    .then(recargar)
    .catch((err) => alert(err.message));
};
```

Notas:
- **Asignar** solo aplica a solicitudes `Pendiente` con coordenadas; si la solicitud no tiene
  `spp_latitud`/`spp_longitud` el backend regresa 400 ("No tiene coordenadas"). Para la prueba
  manual se puede setear directo en BD (ver Verificación).
- **Cancelar** pide motivo antes de llamar al endpoint (se registra en `historial_solicitud`).
- **OJO con el 400**: el backend valida las transiciones; si `estado_nuevo` no es válido desde el
  estado actual, la llamada falla y se muestra el error (no romper el tablero, luego recargar).

**DoD:** cada acción persiste en BD (verificar con `historial_solicitud` y el estado en el tablero).

---

## TAREA 3.4 — Drag & drop entre columnas

Con HTML5 nativo, sin librería:

1. En `DashboardPipas.jsx`, un handler por columna (los mismos para las 4):

```jsx
const onDragOver = (e) => e.preventDefault();

const onDrop = async (e, estadoDestino) => {
  e.preventDefault();
  const sppId = Number(e.dataTransfer.getData('text/spp_id'));
  if (!sppId) return;
  const s = solicitudes.find((x) => x.spp_id === sppId);
  if (!s || s.spp_estatus === estadoDestino) return; // no-op
  try {
    await cambiarEstadoSolicitud(sppId, { estado_nuevo: estadoDestino });
  } catch (err) {
    alert(err.message); // transición inválida -> el backend responde 400
  }
  recargar();
  setDraggingId(null);
};
```

2. Cada columna del Kanban (Pendiente / En Ruta / Entregada / Cancelada):

```jsx
<div
  onDragOver={onDragOver}
  onDrop={(e) => onDrop(e, 'En Ruta')}
  className="flex-1 min-h-[400px] rounded border p-2"
  style={{ backgroundColor: 'var(--color-surface)' }}
>
  <h3 className="text-xs font-bold mb-2">En Ruta <span>({agrupadas('En Ruta').length})</span></h3>
  <div className="space-y-2">
    {agrupadas('En Ruta').map((s) => (
      <SolicitudCard key={s.spp_id} solicitud={s} vehiculos={vehiculos}
        onAsignar={onAsignar} onEstado={onEstado} onCancelar={onCancelar} />
    ))}
  </div>
</div>
```

> Si el estado destino es inválido desde el origen (ej. `Pendiente -> Entregada`), el backend
> regresa 400 y se muestra el error; la recarga del tablero lo deja consistente. Esto evita tener
> que duplicar la lógica de transición en el frontend (ya vive en `SolicitudEstado.es_valida`).

**DoD:** arrastrar una tarjeta de una columna a otra cambia el estado en BD (aparece en
`historial_solicitud`) y el tablero se refresca.

---

## TAREA 3.5 — Filtros por fecha, vehículo y estado + contadores

1. **Filtro de fecha** (sobre `spp_horaentrega`, client-side): dos `<input type="date">` que
   alimentan `fechaDesde`/`fechaHasta` (ya contemplado en el `filter` de 3.1).
2. **Filtro de vehículo**: un `<select>` con los vehículos de `getVehiculos` que filtra por
   `spp_vhe_id`.
3. **Contadores por columna**: en el encabezado de cada columna mostrar el número de tarjetas
   (`agrupadas(estado).length`).
4. Toolbar de filtros sobre el tablero:

```jsx
<div className="flex items-center gap-2 px-5 py-2">
  <input type="date" value={fechaDesde} onChange={(e) => setFechaDesde(e.target.value)} />
  <span className="text-xs">a</span>
  <input type="date" value={fechaHasta} onChange={(e) => setFechaHasta(e.target.value)} />
  <select value={vehiculoFiltro} onChange={(e) => setVehiculoFiltro(e.target.value)}>
    <option value="">Todos los vehículos</option>
    {Object.entries(vehiculos).map(([id, desc]) => (
      <option key={id} value={id}>{desc}</option>
    ))}
  </select>
</div>
```

> El filtro de fecha puede también delegarse al backend (`fecha_desde`/`fecha_hasta` con refetch)
> si los datos crecen; hoy el client-side es suficiente.

**DoD:** los filtros recalculan las columnas y los contadores; el filtro por vehículo usa las
descripciones de `GET /vehiculos`.

---

## Verificación del día (TODO EN DOCKER)

```bash
# Levantar/levantar servicios (si algo cambio en dependencias o backend)
docker compose up -d --build

# Backend sin regresiones
docker compose exec backend python -m pytest -q

# Frontend build (Vite)
docker compose exec frontend npm run build

# Datos de prueba: dar coordenadas y un estado Pendiente a solicitudes para probar "Asignar"
docker compose exec db psql -U postgres -d dapatlqdb -c "UPDATE solicitudpipas SET spp_latitud=20.64, spp_longitud=-103.311 WHERE spp_latitud IS NULL;"
```

Chequeos manuales (navegador):
1. `http://localhost:3000/pipas` (login `admin` / `admin123`): las 4 columnas con tarjetas
   agrupadas por estado y contadores.
2. Columna "Pendiente": botón **Asignar** llama a `POST /asignaciones/automatica` y mueve la
   tarjeta a "En Ruta" (vehículo asignado visible en la tarjeta).
3. Columna "En Ruta": botón **Marcar entregada** pasa la solicitud a "Entregada".
4. Botón **Cancelar** pide motivo y la tarjeta pasa a "Cancelada".
5. Drag & drop: arrastrar una tarjeta Pendiente a "En Ruta" y otra "En Ruta" a "Entregada";
   intentar un salto inválido (`Pendiente -> Entregada`) y verificar que aparece el error y no
   rompe el tablero.
6. Filtros: cambiar fechas y vehículo y verificar columnas + contadores.
7. Verificar la bitácora: `historial_solicitud` tiene una fila por cambio de estado.
8. `/docs` del backend sigue OK (sin regresiones).

## DoD del día + commit

- [ ] 3.1 `/pipas` con Kanban de 4 columnas agrupadas por `spp_estatus` desde la API.
- [ ] 3.2 `SolicitudCard.jsx` reutilizable (folio, vehículo, hora de entrega, estado + acciones).
- [ ] 3.3 Asignar (automatica), Cambiar estado y Cancelar (con motivo) persisten en BD.
- [ ] 3.4 Drag & drop que llama a `PATCH /solicitudes_pipa/{id}/estado` y refleja el cambio.
- [ ] 3.5 Filtros por fecha, vehículo y estado + contadores por columna.
- [ ] `npm run build` del frontend en verde.
- [ ] `pytest` del backend en verde (sin regresiones).

Commit sugerido:

```bash
git add -A
git commit -m "feat(pipas): tablero kanban de solicitudes con tarjetas, acciones, drag & drop y filtros (3.1-3.5)"
git push origin feature/pipas-dashboard
```
