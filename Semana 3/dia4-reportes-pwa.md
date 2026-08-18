# HOJA DE INSTRUCCIONES — MODULO DE PIPAS — SPRINT 3 — DIA 4 (JUEVES)

**Autor:** Miguel Espinoza
**Fecha:** 2026-08-20
**Repositorio:** dapa2w — rama `feature/pipas-reportes` (+ toques a `feature/pipas-pwa-pulido`)
**Objetivo del dia:** Reportes (3.13–3.15) y Pulido de la PWA del chofer (3.16). El backend de los
reportes (3.11 y 3.12) **ya se completó el Martes** y se reescribió/verificó el Miércoles por la
tarde; hoy solo se consumen desde el frontend. El pulido consiste en que el chofer pueda marcar
avance de la entrega (Iniciar ruta / Entrega) desde su PWA y que el Supervisor (DashboardPipas)
se entere en tiempo real por el WebSocket de ubicación existente.

> **IMPORTANTE (convencion del sprint): TODAS las instalaciones, builds y pruebas se hacen
> DENTRO del contenedor Docker.** Nunca `npm install`/`npm run build`/`pytest` en la maquina
> host. Usar `docker compose exec frontend ...` y `docker compose exec backend ...`.

> **LEGIBILIDAD:** Esta hoja incluye el **código COMPLETO** de los servicios y de las páginas
> nuevas listos para pegar, y los **fragmentos exactos** para modificar `AppChofer.jsx` y
> `DashboardPipas.jsx`. La base es funcional; ajústala con calma.

---

## Tareas del dia

| # | Tarea | Archivo(s) principal(es) | DoD |
|---|-------|--------------------------|-----|
| 3.11 | Backend: `GET /reportes/combustible` | `backend/app/api/v1/reportes.py` | **Ya existe** — solo verificar en `/docs` |
| 3.12 | Backend: `GET /reportes/tiempos_traslado` | `backend/app/api/v1/reportes.py` | **Ya existe** — solo verificar en `/docs` |
| 3.13 | Frontend: página reporte de combustible con filtros y tabla | `frontend/src/pages/ReporteCombustiblePage.jsx`, `frontend/src/services/reportesService.js` | Muestra datos reales de la API |
| 3.14 | Frontend: página reporte de tiempos de traslado | `frontend/src/pages/ReporteTiemposTrasladosPage.jsx` | Muestra datos reales de la API |
| 3.15 | Exportar CSV + imprimir en ambas páginas | `frontend/src/pages/ReporteCombustiblePage.jsx`, `frontend/src/pages/ReporteTiemposTrasladosPage.jsx` | Exportación funcional |
| 3.16 | Pulido PWA chofer: botones de avance (Iniciar ruta / Entregada) + aviso al Supervisor por WebSocket | `frontend/src/pages/AppChofer.jsx`, `frontend/src/pages/DashboardPipas.jsx`, `backend/app/api/v1/ubicacion_ws.py` | El chofer marca avance y el tablero se entera en vivo |
| 3.17 | Notificaciones Push (opcional) | `frontend/` | Documentado como futuro con VAPID (ver notas al final) |

**DoD del dia:** Los 2 reportes muestran datos reales con filtros, CSV e imprimir; el chofer puede
marcar *Iniciar ruta* y *Marcar entregada* en su PWA y el Supervisor ve el cambio en vivo sin
recargar. `npm run build` del frontend en verde y `pytest` del backend sin regresiones.

---

## 0. Prerequisitos y convenciones

- Continuar sobre lo desarrollado en el LUNES (Kanban `/pipas`), MARTES (backend de pozos/
  combustible/reportes) y MIERCOLES (catálogos frontend 3.6–3.10).
- **El backend de reportes YA está terminado y registrado.**
  - `backend/app/api/v1/reportes.py` — router `prefix="/reportes", tags=["Reportes"]`.
  - `GET /reportes/combustible` (filtros `fecha_desde`, `fecha_hasta`, `vehiculo_id`) → devuelve
    `ReporteCombustibleResponse` con `desde`, `hasta`, `total_lts`, `total_cost` y `filas[]` de
    `ReporteCombustibleRow { fecha, vhe_id, total_lts, total_cost, num_registros }`.
  - `GET /reportes/tiempos_traslado` (filtros `fecha_desde`, `fecha_hasta`) → devuelve
    `ReporteTiemposTrasladosResponse` con `filas[]` de
    `ReporteTiemposTrasladosRow { fecha, entregas, duracion_promedio_min, km_recorridos }`.
  - Registrado en `backend/app/api/v1/router.py` (`reportes_router`, línea 28 y 52). **No tocar.**
  - Schemas en `backend/app/api/schemas/reportes.py` (verificado abajo).
- **Servicio de combustibles / vehículos ya disponibles** para poblar el filtro de vehículo:
  `frontend/src/services/vehiculoService.js` con `getVehiculos({ page, page_size })` que devuelve
  `PagedResponse.items`.
- **PWA del chofer actual** (`frontend/src/pages/AppChofer.jsx`, tareas 2.22–2.25): ya tiene login,
  lista de entregas asignadas (solo las de SU vehículo), mini mapa con ruta del folio activo,
  WebSocket `/ws/ubicacion` para enviar su GPS cada 10 s, cola offline en `localStorage`, y badges
  por estado. **Solo falta** que el chofer pueda CAMBIAR el estado (hoy solo ve el badge) y que el
  Supervisor lo sepa al momento.
- **DashboardPipas** (`frontend/src/pages/DashboardPipas.jsx`, Lunes): Kanban con drag & drop,
  filtros y `cambiarEstadoSolicitud`. Hoy recarga a mano (`cargar()`). **Solo falta** suscribirlo
  al WebSocket para refresco/aviso en vivo.
- **WebSocket existente** (`backend/app/api/v1/ubicacion_ws.py`, ruta `/ws/ubicacion?token=<JWT>`):
  valida `MensajeUbicacion` y reenvía **broadcast** a todos los conectados. Hoy el mensaje es
  estrictamente una ubicación (`vhe_id`, `lat`, `lng`, `ts`). Para notificaciones de **estado** se
  hace un cambio mínimo y localizado (aceptar también un mensaje `tipo="estado"` y broadcastearlo);
  **no** se monta infra nueva (ni VAPID, ni cola de mensajería).
- **Cambio de estado:** `cambiarEstadoSolicitud(spp_id, { estado_nuevo, motivo })` llama a
  `PATCH /solicitudes_pipa/{id}/estado` (ya existe). La regla de transición es **libre**
  (`enums.py::SolicitudEstado.es_valida` = `desde != hacia`), por lo que *Pendiente → En Ruta →
  Entregada* funciona sin problemas.
- **Mapeo de estados decidido:** el sprint menciona *Iniciar ruta / En pozo / Entregada*, pero el
  backend solo conoce `Pendiente`, `En Ruta`, `Entregada`, `Cancelada`. → **"Iniciar ruta" se mapea a
  `En Ruta`** y "En pozo" **no se implementa** (se documenta la decisión al final). Botones del
  chofer: **Iniciar ruta** (Pendiente → En Ruta) y **Marcar entregada** (En Ruta → Entregada).
- **Token para WS:** el supervisor tiene `access_token` en `localStorage`. Reutilizar
  `getWsUrl(path)` de `frontend/src/services/api.js` y `authStorage.getAccessToken()` de
  `authService.js` (o directamente `localStorage.getItem('access_token')`).
- **React StrictMode** activo: los efectos (y por tanto la conexión WS) corren 2 veces en
  desarrollo; el cleanup `() => ws.close()` lo absorbe.
- **Paginación / formato en el DataTable:** el reporte NO es paginado (lo trae `filas`), por lo que
  las páginas de reporte NO pasan `pagination`/`pageSize` a `DataTable` (solo `columns`, `data`,
  `loading`, `fill`).
- Pruebas del día: manuales en navegador + `pytest` del backend. La verificación de build es
  `docker compose exec frontend npm run build`.

---

## Checklist "listo para implementar" (verificado contra el repo, 2026-08)

| Pendiente (crear hoy) | Ya existe en el repo |
|---|---|
| `frontend/src/services/reportesService.js` (2 funciones) | `backend/app/api/v1/reportes.py` + `backend/app/api/schemas/reportes.py` |
| `frontend/src/pages/ReporteCombustiblePage.jsx` (3.13 + 3.15) | `GET /reportes/combustible` en `/docs` |
| `frontend/src/pages/ReporteTiemposTrasladosPage.jsx` (3.14 + 3.15) | `GET /reportes/tiempos_traslado` en `/docs` |
| Botones de estado en `AppChofer.jsx` (3.16) | `PATCH /solicitudes_pipa/{id}/estado` + `cambiarEstadoSolicitud` |
| Aviso en vivo en `DashboardPipas.jsx` (3.16) | `/ws/ubicacion` + `ConnectionManager.broadcast` |
| Mensaje `tipo="estado"` en `ubicacion_ws.py` + `MensajeEstado` (cambio mínimo) | `manager.broadcast` ya definido |

---

## TAREA 3.11 y 3.12 — Verificar el backend de reportes (no implementar)

El backend se creó en el Martes y se verificó/reescribió el Miércoles. No se modifica nada.

**Verificar en `/docs` (navegador, login `admin`/`admin123` o token):**

1. `GET /reportes/combustible` — sin filtros devuelve:
   ```json
   {
     "desde": null,
     "hasta": null,
     "total_lts": 0,
     "total_cost": 0,
     "filas": []
   }
   ```
   Con datos: `filas[0] = { "fecha": "2026-08-19", "vhe_id": 1, "total_lts": 150.0,
   "total_cost": 3600.0, "num_registros": 1 }`.
2. `GET /reportes/tiempos_traslado` — sin filtros devuelve `filas[]` con `fecha`, `entregas`,
   `duracion_promedio_min`, `km_recorridos`. Si no hay rutas/ubicaciones, `filas` puede vaciarse.
3. Filtros: probar `fecha_desde`, `fecha_hasta`, `vehiculo_id` en combustible.

> Si el backend responde 500 al consultar combustible, revisar `total_lts`/`total_cost`: el repo usa
> `func.sum(Numeric)` que puede devolver `Decimal`; el router lo convierte con `float(...)`. Ya está
> así en `reportes.py`. **No tocar.**

---

## TAREA 3.13 + 3.15 — `reportesService.js` + `ReporteCombustiblePage.jsx`

### 3.13.1 Crear `frontend/src/services/reportesService.js`

```js
import { api } from './api';

export function getReporteCombustible(params = {}) {
  const qs = new URLSearchParams(params).toString();
  return api.get(`/reportes/combustible${qs ? `?${qs}` : ''}`);
}

export function getReporteTiemposTraslados(params = {}) {
  const qs = new URLSearchParams(params).toString();
  return api.get(`/reportes/tiempos_traslado${qs ? `?${qs}` : ''}`);
}
```

### 3.13.2 Crear `frontend/src/pages/ReporteCombustiblePage.jsx`

Patrón: toolbar con filtros (fecha desde/hasta + select de vehículo) + botones imprimir/CSV;
`DataTable` sin paginación; tarjeta de totales arriba.

```jsx
import { useEffect, useState } from 'react';
import { Fuel, Search, Printer, FileSpreadsheet, CalendarRange, Truck, Droplets, Banknote, BarChart3 } from 'lucide-react';
import DataTable from '../components/common/DataTable';
import { getReporteCombustible } from '../services/reportesService';
import { getVehiculos } from '../services/vehiculoService';

const INPUT_STYLE = {
  backgroundColor: 'var(--color-surface)',
  borderColor: 'var(--color-border)',
  color: 'var(--color-text-primary)',
};

export default function ReporteCombustiblePage() {
  const [desde, setDesde] = useState('');
  const [hasta, setHasta] = useState('');
  const [vheId, setVheId] = useState('');
  const [vehiculos, setVehiculos] = useState([]);
  const [data, setData] = useState([]);
  const [totales, setTotales] = useState({ total_lts: 0, total_cost: 0 });
  const [loading, setLoading] = useState(true);
  const [ok, setOk] = useState('');

  useEffect(() => {
    getVehiculos({ page: 1, page_size: 100 })
      .then((d) => setVehiculos(Array.isArray(d.items) ? d.items : []))
      .catch(console.error);
  }, []);

  const consultar = (e) => {
    if (e) e.preventDefault();
    setLoading(true);
    setOk('');
    const params = {};
    if (desde) params.fecha_desde = desde;
    if (hasta) params.fecha_hasta = hasta;
    if (vheId) params.vehiculo_id = Number(vheId);
    getReporteCombustible(params)
      .then((r) => {
        setData(Array.isArray(r.filas) ? r.filas : []);
        setTotales({ total_lts: r.total_lts ?? 0, total_cost: r.total_cost ?? 0 });
        setOk(`${(r.filas || []).length} registro(s) de combustible`);
      })
      .catch((err) => setOk(err.message))
      .finally(() => setLoading(false));
  };

  useEffect(() => { consultar(); }, []); // eslint-disable-line react-hooks/exhaustive-deps

  const COLUMNS = [
    { key: 'fecha', label: 'Fecha', render: (v) => (v ? String(v).slice(0, 10) : '') },
    { key: 'vhe_id', label: 'Vehículo', render: (v) => (v ? `#${v}` : '—') },
    { key: 'total_lts', label: 'Litros', render: (v) => `${Number(v).toFixed(2)} L` },
    { key: 'total_cost', label: 'Costo', render: (v) => `$${Number(v).toFixed(2)}` },
    { key: 'num_registros', label: 'Registros' },
  ];

  const handlePrint = () => window.print();

  const handleExportExcel = () => {
    const csvContent = [
      ['Fecha', 'Vehículo', 'Litros', 'Costo', 'Registros'].join(','),
      ...data.map((r) => [
        r.fecha ? String(r.fecha).slice(0, 10) : '',
        r.vhe_id,
        r.total_lts,
        r.total_cost,
        r.num_registros,
      ].join(',')),
      ['', 'TOTAL', totales.total_lts, totales.total_cost, ''].join(','),
    ].join('\n');
    const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.setAttribute('download', 'reporte_combustible.csv');
    link.href = url;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    URL.revokeObjectURL(url);
  };

  return (
    <div className="flex flex-col h-full" style={{ backgroundColor: 'var(--color-surface)' }}>
      <div className="flex items-center justify-between px-5 py-2 shrink-0"
        style={{ borderBottom: '1px solid var(--color-border)' }}>
        <div>
          <span className="text-xs font-medium" style={{ color: 'var(--color-text-primary)' }}>
            Reporte de combustible
          </span>
          <span className="text-[11px] ml-2" style={{ color: 'var(--color-text-muted)' }}>
            Totales de litros y costo por día y vehículo
          </span>
        </div>
        <div className="flex items-center gap-1 text-[11px]" style={{ color: 'var(--color-text-primary)' }}>
          <Fuel size={12} />
          <span>{data.length} filas</span>
        </div>
      </div>

      <div className="flex items-center gap-2 px-5 py-1.5 shrink-0 flex-wrap"
        style={{ borderBottom: '1px solid var(--color-border)' }}>
        <form onSubmit={consultar} className="flex items-center gap-2 flex-wrap">
          <input type="date" value={desde}
            onChange={(e) => setDesde(e.target.value)} className="px-2 py-1 text-xs rounded border outline-none"
            style={INPUT_STYLE} />
          <span className="text-xs" style={{ color: 'var(--color-text-muted)' }}>a</span>
          <input type="date" value={hasta}
            onChange={(e) => setHasta(e.target.value)} className="px-2 py-1 text-xs rounded border outline-none"
            style={INPUT_STYLE} />
          <select value={vheId} onChange={(e) => setVheId(e.target.value)}
            className="px-2 py-1 text-xs rounded border outline-none" style={INPUT_STYLE}>
            <option value="">Todos los vehículos</option>
            {vehiculos.map((v) => (
              <option key={v.vhe_id} value={v.vhe_id}>{v.vhe_descripcion}</option>
            ))}
          </select>
          <button type="submit"
            className="px-3 py-1 text-[11px] font-medium rounded text-white transition-opacity hover:opacity-85"
            style={{ backgroundColor: 'var(--color-accent)' }}>
            <span className="flex items-center gap-1"><Search size={12} /> Consultar</span>
          </button>
        </form>
        <button onClick={handlePrint} className="p-1 rounded transition-colors"
          style={{ color: 'var(--color-text-primary)' }}
          onMouseEnter={(e) => e.currentTarget.style.backgroundColor = 'var(--color-surface-hover)'}
          onMouseLeave={(e) => e.currentTarget.style.backgroundColor = ''} title="Imprimir">
          <Printer size={15} />
        </button>
        <button onClick={handleExportExcel} className="p-1 rounded transition-colors"
          style={{ color: 'var(--color-text-primary)' }}
          onMouseEnter={(e) => e.currentTarget.style.backgroundColor = 'var(--color-surface-hover)'}
          onMouseLeave={(e) => e.currentTarget.style.backgroundColor = ''} title="Exportar a Excel">
          <FileSpreadsheet size={15} />
        </button>
      </div>

      {ok && (
        <div className="px-5 pt-2 text-[11px]" style={{ color: 'var(--color-text-muted)' }}>{ok}</div>
      )}

      <div className="grid grid-cols-2 gap-3 px-5 py-3 shrink-0 max-w-2xl">
        <div className="flex items-center gap-3 rounded-lg border p-3"
          style={{ backgroundColor: 'var(--color-surface-elevated)', borderColor: 'var(--color-border)' }}>
          <Droplets size={20} style={{ color: '#16a34a' }} />
          <div>
            <div className="text-[10px]" style={{ color: 'var(--color-text-muted)' }}>Total litros</div>
            <div className="text-sm font-bold" style={{ color: 'var(--color-text-primary)' }}>
              {Number(totales.total_lts).toFixed(2)} L
            </div>
          </div>
        </div>
        <div className="flex items-center gap-3 rounded-lg border p-3"
          style={{ backgroundColor: 'var(--color-surface-elevated)', borderColor: 'var(--color-border)' }}>
          <Banknote size={20} style={{ color: '#2563eb' }} />
          <div>
            <div className="text-[10px]" style={{ color: 'var(--color-text-muted)' }}>Total costo</div>
            <div className="text-sm font-bold" style={{ color: 'var(--color-text-primary)' }}>
              ${Number(totales.total_cost).toFixed(2)}
            </div>
          </div>
        </div>
      </div>

      <div className="flex-1 min-h-0 mx-5 my-5">
        <DataTable columns={COLUMNS} data={data} loading={loading} fill />
      </div>
    </div>
  );
}
```

**DoD:** consulta por fechas/vehículo; totales y tabla reflejan la respuesta real de la API; el CSV
incluye una fila TOTAL.

---

## TAREA 3.14 + 3.15 — `ReporteTiemposTrasladosPage.jsx`

Mismo patrón; columnas `fecha`, `entregas`, `duracion_promedio_min`, `km_recorridos`.

```jsx
import { useEffect, useState } from 'react';
import { Route as RouteIcon, Search, Printer, FileSpreadsheet, Clock3, MapPin, PackageCheck } from 'lucide-react';
import DataTable from '../components/common/DataTable';
import { getReporteTiemposTraslados } from '../services/reportesService';

const INPUT_STYLE = {
  backgroundColor: 'var(--color-surface)',
  borderColor: 'var(--color-border)',
  color: 'var(--color-text-primary)',
};

export default function ReporteTiemposTrasladosPage() {
  const [desde, setDesde] = useState('');
  const [hasta, setHasta] = useState('');
  const [data, setData] = useState([]);
  const [loading, setLoading] = useState(true);
  const [ok, setOk] = useState('');

  const consultar = (e) => {
    if (e) e.preventDefault();
    setLoading(true);
    setOk('');
    const params = {};
    if (desde) params.fecha_desde = desde;
    if (hasta) params.fecha_hasta = hasta;
    getReporteTiemposTraslados(params)
      .then((r) => {
        setData(Array.isArray(r.filas) ? r.filas : []);
        setOk(`${(r.filas || []).length} día(s) con actividad`);
      })
      .catch((err) => setOk(err.message))
      .finally(() => setLoading(false));
  };

  useEffect(() => { consultar(); }, []); // eslint-disable-line react-hooks/exhaustive-deps

  const COLUMNS = [
    { key: 'fecha', label: 'Fecha', render: (v) => (v ? String(v).slice(0, 10) : '') },
    { key: 'entregas', label: 'Entregas', render: (v) => Number(v) || 0 },
    {
      key: 'duracion_promedio_min', label: 'Duración prom.',
      render: (v) => (v != null ? `~${Number(v).toFixed(1)} min` : '—'),
    },
    { key: 'km_recorridos', label: 'KM recorridos', render: (v) => `${Number(v).toFixed(2)} km` },
  ];

  const handlePrint = () => window.print();

  const handleExportExcel = () => {
    const csvContent = [
      ['Fecha', 'Entregas', 'Duración prom. (min)', 'KM recorridos'].join(','),
      ...data.map((r) => [
        r.fecha ? String(r.fecha).slice(0, 10) : '',
        r.entregas ?? 0,
        r.duracion_promedio_min ?? '',
        r.km_recorridos ?? 0,
      ].join(',')),
    ].join('\n');
    const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.setAttribute('download', 'reporte_tiempos_traslado.csv');
    link.href = url;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    URL.revokeObjectURL(url);
  };

  const totalEntregas = data.reduce((a, r) => a + (Number(r.entregas) || 0), 0);
  const totalKm = data.reduce((a, r) => a + (Number(r.km_recorridos) || 0), 0);

  return (
    <div className="flex flex-col h-full" style={{ backgroundColor: 'var(--color-surface)' }}>
      <div className="flex items-center justify-between px-5 py-2 shrink-0"
        style={{ borderBottom: '1px solid var(--color-border)' }}>
        <div>
          <span className="text-xs font-medium" style={{ color: 'var(--color-text-primary)' }}>
            Reporte de tiempos de traslado
          </span>
          <span className="text-[11px] ml-2" style={{ color: 'var(--color-text-muted)' }}>
            Entregas, duración promedio y km por día
          </span>
        </div>
        <div className="flex items-center gap-1 text-[11px]" style={{ color: 'var(--color-text-primary)' }}>
          <RouteIcon size={12} />
          <span>{data.length} filas</span>
        </div>
      </div>

      <div className="flex items-center gap-2 px-5 py-1.5 shrink-0 flex-wrap"
        style={{ borderBottom: '1px solid var(--color-border)' }}>
        <form onSubmit={consultar} className="flex items-center gap-2 flex-wrap">
          <input type="date" value={desde}
            onChange={(e) => setDesde(e.target.value)} className="px-2 py-1 text-xs rounded border outline-none"
            style={INPUT_STYLE} />
          <span className="text-xs" style={{ color: 'var(--color-text-muted)' }}>a</span>
          <input type="date" value={hasta}
            onChange={(e) => setHasta(e.target.value)} className="px-2 py-1 text-xs rounded border outline-none"
            style={INPUT_STYLE} />
          <button type="submit"
            className="px-3 py-1 text-[11px] font-medium rounded text-white transition-opacity hover:opacity-85"
            style={{ backgroundColor: 'var(--color-accent)' }}>
            <span className="flex items-center gap-1"><Search size={12} /> Consultar</span>
          </button>
        </form>
        <button onClick={handlePrint} className="p-1 rounded transition-colors"
          style={{ color: 'var(--color-text-primary)' }} title="Imprimir">
          <Printer size={15} />
        </button>
        <button onClick={handleExportExcel} className="p-1 rounded transition-colors"
          style={{ color: 'var(--color-text-primary)' }} title="Exportar a Excel">
          <FileSpreadsheet size={15} />
        </button>
      </div>

      {ok && (
        <div className="px-5 pt-2 text-[11px]" style={{ color: 'var(--color-text-muted)' }}>{ok}</div>
      )}

      <div className="grid grid-cols-2 gap-3 px-5 py-3 shrink-0 max-w-2xl">
        <div className="flex items-center gap-3 rounded-lg border p-3"
          style={{ backgroundColor: 'var(--color-surface-elevated)', borderColor: 'var(--color-border)' }}>
          <PackageCheck size={20} style={{ color: '#16a34a' }} />
          <div>
            <div className="text-[10px]" style={{ color: 'var(--color-text-muted)' }}>Total entregas</div>
            <div className="text-sm font-bold" style={{ color: 'var(--color-text-primary)' }}>{totalEntregas}</div>
          </div>
        </div>
        <div className="flex items-center gap-3 rounded-lg border p-3"
          style={{ backgroundColor: 'var(--color-surface-elevated)', borderColor: 'var(--color-border)' }}>
          <MapPin size={20} style={{ color: '#2563eb' }} />
          <div>
            <div className="text-[10px]" style={{ color: 'var(--color-text-muted)' }}>Total km</div>
            <div className="text-sm font-bold" style={{ color: 'var(--color-text-primary)' }}>
              {totalKm.toFixed(2)} km
            </div>
          </div>
        </div>
      </div>

      <div className="flex-1 min-h-0 mx-5 my-5">
        <DataTable columns={COLUMNS} data={data} loading={loading} fill />
      </div>
    </div>
  );
}
```

**DoD:** consulta por fechas; tabla con duración promedio (o "—" si no hay rutas); CSV funcional.

---

## TAREA 3.16 — Pulido PWA del chofer: botones de avance + aviso en vivo al Supervisor

Se divide en 3 piezas:

- **3.16.1** Cambio MÍNIMO de backend: `ubicacion_ws.py` acepta y hace broadcast de un mensaje
  `tipo="estado"` (además del de ubicación). No agrega infra; solo amplía el validador.
- **3.16.2** `AppChofer.jsx`: botones **Iniciar ruta** y **Marcar entregada** por tarjeta, que
  llaman a `PATCH` y luego envían el evento `estado` por el WS ya conectado.
- **3.16.3** `DashboardPipas.jsx`: se suscribe al mismo WS y al recibir `tipo="estado"` muestra un
  aviso y refresca el Kanban automáticamente.

### 3.16.1 Backend — `backend/app/api/schemas/ubicacion.py` (agregar `MensajeEstado`)

```python
class MensajeEstado(BaseModel):
    """Mensaje opcional del chofer para avisar cambio de estado de una entrega (3.16)."""
    tipo: str = "estado"
    vhe_id: int = Field(..., ge=1, description="ID del vehiculo")
    spp_id: int = Field(..., ge=1, description="ID de la solicitud / entrega")
    spp_estado: str = Field(..., min_length=1, max_length=20)
    ts: Optional[datetime] = Field(default=None, description="Marca de tiempo (opcional)")
```

### 3.16.1 Backend — `backend/app/api/v1/ubicacion_ws.py` (aceptar ambos tipos)

Cambiar SOLO el bloque de validación del `while True` para distinguir el tipo y hacer broadcast
sin persistir ubicación cuando es un evento de estado:

```python
    try:
        while True:
            data: Dict[str, Any] = await websocket.receive_json()

            # 3.16 - El chofer puede enviar un evento de ESTADO (no lleva lat/lng)
            if data.get("tipo") == "estado":
                try:
                    evento = MensajeEstado.model_validate(data)
                except Exception:
                    await websocket.send_json({"type": "error", "detail": "mensaje_estado_invalido"})
                    continue
                await manager.broadcast(
                    {
                        "type": "estado",
                        "vhe_id": evento.vhe_id,
                        "spp_id": evento.spp_id,
                        "spp_estado": evento.spp_estado,
                        "ts": (evento.ts or datetime.now(timezone.utc)).isoformat(),
                    }
                )
                continue

            try:
                mensaje = MensajeUbicacion.model_validate(data)
            except Exception:
                await websocket.send_json({"type": "error", "detail": "mensaje_invalido"})
                continue
            # ... el resto (persistir ubicación + broadcast) se mantiene igual
```

Y actualizar el import en `ubicacion_ws.py`:

```python
from app.api.schemas.ubicacion import MensajeEstado, MensajeUbicacion
```

### 3.16.2 Frontend — `frontend/src/pages/AppChofer.jsx` (botones de avance)

Cambios puntuales:

1. **Importar** `cambiarEstadoSolicitud` además de `getSolicitudes`:

```jsx
import { getSolicitudes, cambiarEstadoSolicitud } from '../services/solicitudesService';
```

2. **Agregar helper de recarga + cambio de estado** (dentro del componente, junto al efecto de
   solicitudes):

```jsx
const recargar = () => {
  getSolicitudes({ page: 1, page_size: 100 })
    .then((data) => {
      const asignadas = data.items.filter((s) => s.spp_vhe_id === vheId);
      setSolicitudes(asignadas);
    })
    .catch(() => {});
};

const avisarSupervisor = (sppId, sppEstado) => {
  const ws = wsRef.current;
  if (ws && ws.readyState === WebSocket.OPEN) {
    ws.send(JSON.stringify({
      tipo: 'estado',
      vhe_id: vheId,
      spp_id: sppId,
      spp_estado: sppEstado,
    }));
  }
};

const cambiarEstado = async (s, estadoNuevo) => {
  if (!window.confirm(`¿Marcar el folio #${s.spp_id} como "${estadoNuevo}"?`)) return;
  try {
    await cambiarEstadoSolicitud(s.spp_id, { estado_nuevo: estadoNuevo });
    avisarSupervisor(s.spp_id, estadoNuevo);
    recargar();
  } catch (err) {
    alert(err.message);
  }
};
```

> `wsRef` ya existe en el componente (la conexión `/ws/ubicacion` es la misma). No hace falta
> abrir otra conexión.

3. **Botones por tarjeta**: dentro del `<li key={s.spp_id}>`, justo después del `</p>` de `{info &&
   (...)}` y ANTES del bloque `{s.spp_latitud && s.spp_longitud ? (...)}`, agregar la fila de
   acciones de estado:

```jsx
                  {s.spp_estatus === 'Pendiente' && (
                    <div className="flex gap-2">
                      <button onClick={() => cambiarEstado(s, 'En Ruta')}
                        className="flex-1 text-xs bg-blue-600 text-white px-3 py-1.5 rounded">
                        <span className="flex items-center justify-center gap-1"><Navigation size={13} /> Iniciar ruta</span>
                      </button>
                    </div>
                  )}
                  {s.spp_estatus === 'En Ruta' && (
                    <div className="flex gap-2">
                      <button onClick={() => cambiarEstado(s, 'Entregada')}
                        className="flex-1 text-xs bg-green-600 text-white px-3 py-1.5 rounded">
                        <span className="flex items-center justify-center gap-1"><MapPin size={13} /> Marcar entregada</span>
                      </button>
                    </div>
                  )}
                  {s.spp_estatus !== 'Cancelada' && s.spp_estatus !== 'Entregada' && (
                    <div className="flex gap-2">
                      <button onClick={() => cambiarEstado(s, 'Cancelada')}
                        className="flex-1 text-xs border border-red-500 text-red-500 px-3 py-1.5 rounded">
                        Cancelar
                      </button>
                    </div>
                  )}
```

> **Nota sobre mapeo:** "Iniciar ruta" → `En Ruta` (ver decisión en prerequisitos). "En pozo" no se
> implementa porque el backend no tiene ese estado.

### 3.16.3 Frontend — `frontend/src/pages/DashboardPipas.jsx` (aviso en vivo al Supervisor)

1. **Importes** (agregar al top):

```jsx
import { getWsUrl } from '../services/api';
import { authStorage } from '../services/authService';
```

2. **Estado del aviso** (junto a los demás `useState`):

```jsx
  const [aviso, setAviso] = useState(null); // {spp_id, spp_estado, ts}
```

3. **Suscribirse al WebSocket** (efecto nuevo; el Supervisor ya tiene token en `localStorage`):

```jsx
  // 3.16 - Recibir en vivo los cambios de estado del chofer y refrescar el Kanban
  useEffect(() => {
    const token = authStorage.getAccessToken() || localStorage.getItem('access_token');
    if (!token) return;
    let ws;
    const conectar = () => {
      ws = new WebSocket(`${getWsUrl('/ws/ubicacion')}?token=${token}`);
      ws.onmessage = (ev) => {
        try {
          const msg = JSON.parse(ev.data);
          if (msg.type === 'estado' && msg.spp_id) {
            setAviso({ spp_id: msg.spp_id, spp_estado: msg.spp_estado, ts: Date.now() });
            cargar(); // refresca el Kanban al instante
          }
        } catch { /* ignorar */ }
      };
      ws.onclose = () => setTimeout(conectar, 5000);
    };
    conectar();
    return () => ws && ws.close();
  }, []); // eslint-disable-line react-hooks/exhaustive-deps
```

> `cargar` se define en el componente (ya existe); el efecto lo captura del primer render, que es
> suficiente porque `cargar` solo reejecuta `getSolicitudes`. Si quieres siempre la versión nueva,
> escribe el efecto con `cargar` en las dependencias y envuélvelo con `useCallback` — NO es
> necesario para el DoD.

4. **Mostrar el aviso** (dentro del header, junto al contador de solicitudes):

```jsx
        {aviso && (
          <div className="fixed bottom-4 right-4 z-50 rounded-lg shadow-lg border px-4 py-3 max-w-xs"
            style={{ backgroundColor: 'var(--color-surface-elevated)', borderColor: 'var(--color-border)' }}>
            <div className="text-[11px] font-semibold" style={{ color: 'var(--color-text-primary)' }}>
              Cambio de estado
            </div>
            <div className="text-[11px]" style={{ color: 'var(--color-text-muted)' }}>
              Folio #{aviso.spp_id} → {aviso.spp_estado}
            </div>
            <button onClick={() => setAviso(null)}
              className="mt-2 text-[10px] underline" style={{ color: 'var(--color-accent)' }}>
              Cerrar
            </button>
          </div>
        )}
```

**DoD 3.16:** desde `/chofer?vehiculo=<id>` el chofer marca *Iniciar ruta* y *Marcar entregada*; en
el Kanban `/pipas` (otra pestaña con sesión de Supervisor) la tarjeta cambia de columna **sin
recargar** y aparece el aviso flotante.

---

## TAREA 3.17 — Notificaciones Push (OPCIONAL, documentada)

Por la decisión tomada, la notificación al Supervisor se resuelve con el **WebSocket existente**
(3.16.3). Las **Push Notifications reales** (que lleguen con la app en segundo plano / cerrada)
requieren:

1. Service worker con `push` handler (Vite PWA: `vite-plugin-pwa`; hoy no hay `push` listener).
2. Claves **VAPID** y un endpoint backend que dispare `web-push` al recibir el evento `estado`
   (o al asignar una solicitud).
3. Manejo de permisos/suscripción en `AppChofer.jsx`.

**No se implementa hoy** para mantener el Jueves 100% frontend + el único cambio mínimo de WS.
Queda como mejora documentada para el Viernes/Demo si sobra tiempo.

---

## Verificación del día (TODO EN DOCKER)

```bash
# Levantar servicios (backend del Martes + frontend del Miércoles ya están)
docker compose up -d --build

# Backend sin regresiones (verifica que el cambio de WS no rompió nada)
docker compose exec backend python -m pytest -q

# Frontend build (Vite + PWA)
docker compose exec frontend npm run build
```

Chequeos manuales (navegador, login `admin` / `admin123`):
1. `/reportes-combustible`: consultar sin filtros y con fecha/vehículo; revisar totales y CSV.
2. `/reportes-tiempos`: consultar por fechas; revisar duración promedio (o "—") y CSV.
3. `/chofer?vehiculo=<id>` (pestaña A): marcar *Iniciar ruta* en una solicitud Pendiente.
4. `/pipas` (pestaña B, misma sesión): el folio pasa a la columna "En Ruta" en vivo + aviso flotante.
5. En el chofer: *Marcar entregada* → el Kanban la mueve a "Entregada" en vivo.
6. Servidor: confirmar en consola de backend que el WS acepta el mensaje `tipo:"estado"` sin errores.

> Si abres `/chofer` sin `?vehiculo=`, se toma `vheId = 4` por defecto (código actual). Úsalo para
> probar con el vehículo que tenga solicitudes asignadas.

## DoD del día + commit

- [ ] 3.13 `ReporteCombustiblePage.jsx` con `GET /reportes/combustible` real y filtros.
- [ ] 3.14 `ReporteTiemposTrasladosPage.jsx` con `GET /reportes/tiempos_traslado` real.
- [ ] 3.15 CSV + imprimir en ambas páginas.
- [ ] 3.16 Chofer marca *Iniciar ruta* / *Marcar entregada*; Supervisor recibe aviso en vivo.
- [ ] 3.16 Mensaje `tipo="estado"` aceptado y broadcast por `/ws/ubicacion`.
- [ ] Rutas registradas en `App.jsx` y enlaces en `Sidebar.jsx` (reportes + rutas existentes intactas).
- [ ] `npm run build` del frontend en verde y `pytest` del backend en verde.

Agregar rutas en `frontend/src/App.jsx`:

```jsx
import ReporteCombustiblePage from './pages/ReporteCombustiblePage';
import ReporteTiemposTrasladosPage from './pages/ReporteTiemposTrasladosPage';
// ...
<Route path="/reportes-combustible" element={<ReporteCombustiblePage />} />
<Route path="/reportes-tiempos" element={<ReporteTiemposTrasladosPage />} />
```

Y enlaces en `frontend/src/components/common/Sidebar.jsx` (dentro del grupo `pipas`):

```jsx
    { id: 'rep-combustible', label: 'Reporte combustible', icon: Fuel, path: '/reportes-combustible' },
    { id: 'rep-tiempos', label: 'Reporte tiempos', icon: Clock3, path: '/reportes-tiempos' },
```

> `Fuel` y `Clock3` ya se importan en `Sidebar.jsx` (verificar la línea de imports; `Clock3` existe
> en lucide-react como `Clock3` — si no la tienes, usa `Clock`).

Commit sugerido:

```bash
git add -A
git commit -m "feat(pipas): reportes de combustible y tiempos de traslado + avance del chofer en vivo (3.13-3.16)"
git push origin feature/pipas-reportes
```

## Decisiones y notas técnicas

- **Estado "En pozo":** el sprint 3.16 lo menciona, pero el backend (solicitudes_pipa + enum) solo
  soporta `Pendiente / En Ruta / Entregada / Cancelada`. Se mapea *Iniciar ruta → En Ruta* y se
  omite "En pozo". Si Sandra lo exige, sería una historia aparte: agregar `EN_POZO = "En Pozo"` al
  enum, validarlo en `SolicitudPipaCambioEstado` y reflejarlo en el Kanban + Chofer.
- **`reg_com_fecha` / `spp_horaentrega` en UTC:** el backend guarda con `server_default=func.now()`
  y compara con `datetime` naive; el frontend formatea con `new Date(...)`. Si notas desfase de
  horas, es del manejo de zona horaria del navegador, no del reporte.
- **CSV sin escape de comas:** si una descripción contiene comas (p. ej. vehículo "Pipa 01, tanque
  2"), la fila CSG se partiría. Para el DoD del día no aplica (el reporte solo emite números/fechas).