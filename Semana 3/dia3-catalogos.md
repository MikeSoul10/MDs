# HOJA DE INSTRUCCIONES — MODULO DE PIPAS — SPRINT 3 — DIA 3 (MIERCOLES)

**Autor:** Miguel Espinoza
**Fecha:** 2026-08-19
**Repositorio:** dapa2w — rama `feature/pipas-catalogos`
**Objetivo del dia:** Pantallas de catálogos del módulo de pipas (**solo frontend**): alta/edición de
vehículos y marcas (reutilizando el patrón de `EstadosPage` + `DataTable`), alta de solicitud de pipa,
CRUD de pozos con su disponibilidad, y registro de gasto de combustible por vehículo. El backend
necesario (pozos, combustible) se completó el Martes.

> **IMPORTANTE (convencion del sprint): TODAS las instalaciones, builds y pruebas se hacen
> DENTRO del contenedor Docker.** Nunca `npm install`/`npm run build`/`pytest` en la maquina
> host. Usar `docker compose exec frontend ...` y `docker compose exec backend ...`.

> **LEGIBILIDAD:** Esta hoja incluye el **código COMPLETO** de cada servicio y página listos para
> pegar; no solo la descripción. Cámbialos o ajústalos con calma, pero la base es funcional.

---

## Tareas del dia

| # | Tarea | Archivo(s) principal(es) | DoD |
|---|-------|--------------------------|-----|
| 3.6 | Alta/edición de vehículos (patrón `EstadosPage` + `DataTable`) | `frontend/src/pages/VehiculosPage.jsx`, `frontend/src/services/vehiculoService.js` | CRUD de vehículos contra la API |
| 3.7 | Alta/edición de marcas de vehículo | `frontend/src/pages/VehiculoMarcasPage.jsx`, `frontend/src/services/vehiculoMarcaService.js` | CRUD de marcas contra la API |
| 3.8 | Alta de solicitud de pipa (formulario) | `frontend/src/pages/SolicitudPipaPage.jsx` | `POST /solicitudes_pipa` funciona |
| 3.9 | Alta de pozos y gestión de disponibilidad (tabla `pozos`) | `frontend/src/pages/PozosPage.jsx`, `frontend/src/services/pozoService.js` | CRUD de pozos contra la API |
| 3.10 | Registro de gasto de combustible por vehículo | `frontend/src/pages/CombustiblePage.jsx`, `frontend/src/services/combustibleService.js` | `POST /combustible` funciona |

**DoD del dia:** Las 5 pantallas (vehículos, marcas, solicitud, pozos, combustible) operan contra la
API con alta/edición y listado; `npm run build` del frontend en verde.

---

## 0. Prerequisitos y convenciones

- Continuar sobre lo desarrollado en el LUNES (Kanban `/pipas`, `DashboardPipas.jsx`, `SolicitudCard.jsx`)
  y sobre el **MARTES (backend complementario, rama `feature/pipas-backend`)**: en ese día ya se
  crearon los endpoints de `pozos`, `combustible` y `reportes`. Hoy el Miércoles es SOLO frontend.
- **Endpoints backend YA disponibles** (`http://localhost:8000/api/v1`):
  - `GET/POST /vehiculos` y `PUT/PATCH/DELETE /vehiculos/{id}` — CRUD completo (paginado `PagedResponse`).
  - `GET /vehiculos/disponibles` — vehículos activos sin solicitud en `Pendiente`/`En Ruta`.
  - `GET/POST /vehiculomarcas` y `PUT/DELETE /vehiculomarcas/{id}` — CRUD de marcas.
  - `GET /solicitudes_pipa` (paginado, filtros `estado`/`fecha_desde`/`fecha_hasta`), `POST /solicitudes_pipa`
    (alta), `GET /solicitudes_pipa/{id}`, `PATCH /solicitudes_pipa/{id}/estado`.
  - `GET/POST /pozos`, `GET/PUT/DELETE /pozos/{id}` — CRUD de pozos (creado en el Martes).
  - `POST /combustible`, `GET /combustible` — registro e historial de combustible (Martes).
  - `GET /reportes/combustible`, `GET /reportes/tiempos_traslado` — reportes (Martes, para el Jueves).
- **Campos de los modelos (BK):**
  - `Vehiculo` (`vehiculos`): `vhe_id`, `vhe_mar_id` (FK a `fabricante.fab_id`),
    `vhe_res_id`, `vhe_modelo`, `vhe_combustible` (0-100), `vhe_tipovehiculo`, `vhe_fecharegistro`,
    `vhe_descripcion` (única, 409 si se duplica), `vhe_estatus`.
  - `Marca` (`vehiculomarcas`): `vho_id`, `vho_nombremarca` (única, 409 si duplica), `vho_estatus`.
  - `Pozo` (`pozos`): `poz_id`, `poz_dom_id`, `poz_nombrepozo`, `poz_responsable`, `poz_telefono`,
    `poz_fechaalta`, `poz_estatus` (bool = disponibilidad).
  - `RegistroCombustible` (`registros_combustible`): `reg_com_id`, `reg_com_vhe_id` (FK vehículos),
    `reg_com_lts`, `reg_com_cost`, `reg_com_fecha`, `reg_com_km_inicio`, `reg_com_km_final`.
- **⚠️ Nota sobre `GenericRepository`:** los métodos base `get_by_id`, `delete` y `exists` usan
  `self.model.id`, PERO estos modelos usan PKs con prefijo (`vhe_id`, `poz_id`, `reg_com_id`). Un repo
  que no sobreescriba esos métodos **rompe** en `get_by_id`/`delete`. Revisar `vehiculo_repository.py`
  como referencia (sobrescribe `get_by_id`).
  - **✔ Verificado en el repo:** `vehiculo_repository.py`, `marca_vehiculo_repository.py`,
    `pozo_repository.py` y `registro_combustible_repository.py` **sí** sobrescriben `get_by_id` con su
    PK real (`vhe_id`, `vho_id`, `poz_id`, `reg_com_id`). El `delete` de `pozo_repository` y el
    `delete` vía `GenericRepository` (marcas, vehiculos, combustible) delegan en ese `get_by_id`.
    No hay que tocar el backend.
- **Servicios frontend YA existentes:**
  - `vehiculoService.js`: `getVehiculos(params)`, `getVehiculosDisponibles()`. **Faltan**
    `createVehiculo`, `updateVehiculo`, `deleteVehiculo`, `getVehiculo` (se agregan abajo).
  - `solicitudesService.js`: `getSolicitudes`, `cambiarEstadoSolicitud`, `crearSolicitud` (ya existe).
  - Falta crear `vehiculoMarcaService.js`, `pozoService.js`, `combustibleService.js` (código abajo).
- **Frontend:** React 18 + Vite 5 + Tailwind + react-router-dom v6 + `lucide-react` + `recharts`.
  Patrón de catálogo a copiar: `EstadosPage.jsx` (header, buscador, exportar CSV, imprimir) +
  `DataTable.jsx`. Para alta/edición se añade un formulario/modal custom (no hay librería de modales).
  - El modal usa `position: fixed; inset: 0`, backdrop `rgba(0,0,0,0.4)` y
    `onClick={(e) => e.stopPropagation()}` para no cerrarse al tocar dentro.
- **API base y token:** `api.js` usa `import.meta.env.VITE_API_URL || '/api/v1'`; token en
  `localStorage` bajo `access_token`. `api` ya expone `.get/.post/.put/.patch/.delete`.
  - Cuando el backend responde 409, `api.js` lanza `new Error(err.detail)` → mostrar `err.message`.
- **React StrictMode** activo: los efectos corren 2 veces en desarrollo.
- Los catálogos se registran en `frontend/src/App.jsx` (dentro de `DashboardLayout` con
  `ProtectedRoute`) y se enlazan desde `frontend/src/components/common/Sidebar.jsx`.
- Pruebas del día: manuales en navegador + `pytest` del backend. No hay suite de tests de frontend;
  la verificación es `npm run build` + prueba manual.
- **Paginación client-side:** los índices que devuelven `PagedResponse` se consumen con
  `page_size: 100` y se filtran/paginan en el cliente (mismo patrón de `EstadosPage`), para no
  encadenar llamadas al servidor por página.

---

## Checklist "listo para implementar" (verificado contra el repo, 2026-08)

| Pendiente (crear hoy) | Ya existe en el repo |
|---|---|
| `frontend/src/pages/VehiculosPage.jsx` (3.6) | `vehiculos.py`, schemas, `VehiculoRepository` (CRUD ok) |
| `frontend/src/pages/VehiculoMarcasPage.jsx` (3.7) | `vehiculomarcas.py`, schemas, `MarcaVehiculoRepository` |
| `frontend/src/pages/SolicitudPipaPage.jsx` (3.8) | `solicitudes_pipa.py`, `crearSolicitud` en `solicitudesService.js` |
| `frontend/src/pages/PozosPage.jsx` (3.9) | `pozos.py`, schemas, `PozoRepository` (get_by_id/delete sobrescritos) |
| `frontend/src/pages/CombustiblePage.jsx` (3.10) | `combustible.py`, schemas, `RegistroCombustibleRepository` |
| `frontend/src/services/vehiculoMarcaService.js` | — |
| `frontend/src/services/pozoService.js` | — |
| `frontend/src/services/combustibleService.js` | — |
| Funciones CRUD en `vehiculoService.js` | `getVehiculos`, `getVehiculosDisponibles` |
| Rutas en `App.jsx` (5 nuevas) | Rutas `/pipas`, `/mapa` |
| Enlaces en `Sidebar.jsx` (menú `pipas`) | `sol-pipa` (path `#`), `tablero-pipas` `/pipas` |

**Conclusión:** el backend está listo al 100%. Lo único pendiente es el frontend (5 páginas + 3
servicios nuevos + ampliar `vehiculoService` + rutas/sidebar). Docker estaba apagado durante la
revisión; el `npm run build` y los `pytest` se validan al levantar de nuevo.

---

## PASO COMUN: registro de rutas y enlaces (hacer después de cada página)

En `frontend/src/App.jsx`, importar las 5 páginas y añadir sus `<Route>` **dentro** de
`DashboardLayout` (junto a `/pipas`), antes del `path="*"`:

```jsx
import VehiculosPage from './pages/VehiculosPage';
import VehiculoMarcasPage from './pages/VehiculoMarcasPage';
import SolicitudPipaPage from './pages/SolicitudPipaPage';
import PozosPage from './pages/PozosPage';
import CombustiblePage from './pages/CombustiblePage';

// ...dentro del <Route element={<ProtectedRoute><DashboardLayout /></ProtectedRoute>}>
<Route path="/vehiculos" element={<VehiculosPage />} />
<Route path="/vehiculomarcas" element={<VehiculoMarcasPage />} />
<Route path="/solicitud-pipa" element={<SolicitudPipaPage />} />
<Route path="/pozos" element={<PozosPage />} />
<Route path="/combustible" element={<CombustiblePage />} />
```

En `frontend/src/components/common/Sidebar.jsx`, dentro del menú `pipas`, quedar así (se agregan
vehículos, marcas, pozos y combustible; `sol-pipa` cambia de `'#'` a `/solicitud-pipa`):

```jsx
{
  id: 'pipas', label: 'Pipas', icon: Truck,
  children: [
    { id: 'sol-pipa', label: 'Solicitud de pipa', icon: Truck, path: '/solicitud-pipa' },
    { id: 'vehiculos', label: 'Vehículos', icon: Truck, path: '/vehiculos' },
    { id: 'marcas', label: 'Marcas de vehículo', icon: BookOpen, path: '/vehiculomarcas' },
    { id: 'pozos-pipa', label: 'Pozos', icon: Droplet, path: '/pozos' },
    { id: 'combustible', label: 'Combustible', icon: Fuel, path: '/combustible' },
    { id: 'mapa-supervision', label: 'Mapa', icon: Map, path: '/mapa' },
    { id: 'chofer', label: 'Chofer', icon: Navigation, path: '/chofer' },
    { id: 'tablero-pipas', label: 'Tablero Pipas', icon: Kanban, path: '/pipas' },
  ],
},
```

> `Droplet`, `BookOpen`, `Fuel` ya están importados en `Sidebar.jsx` desde `lucide-react`
> (verificar el import al inicio del archivo; si faltara `Fuel`, agregarlo).

---

## TAREA 3.6 — `VehiculosPage.jsx` (alta/edición de vehículos)

### 3.6.1 Ampliar `frontend/src/services/vehiculoService.js`

Contenido COMPLETO del archivo (agrega `getVehiculo`, `createVehiculo`, `updateVehiculo`, `deleteVehiculo`):

```js
import { api } from './api';

export function getVehiculos(params = {}) {
  const qs = new URLSearchParams(params).toString();
  return api.get(`/vehiculos${qs ? `?${qs}` : ''}`);
}

export function getVehiculosDisponibles() {
  return api.get('/vehiculos/disponibles');
}

export function getVehiculo(id) {
  return api.get(`/vehiculos/${id}`);
}

export function createVehiculo(body) {
  return api.post('/vehiculos', body);
}

export function updateVehiculo(id, body) {
  return api.put(`/vehiculos/${id}`, body);
}

export function deleteVehiculo(id) {
  return api.delete(`/vehiculos/${id}`);
}
```

### 3.6.2 Crear `frontend/src/pages/VehiculosPage.jsx`

**OJO:** `GET /vehiculos` devuelve `PagedResponse` → en el `.then()` usar `setData(d.items)`.
El schema `VehiculoCreate` exige los campos obligatorios `vhe_mar_id`, `vhe_res_id` y `vhe_modelo`
(año 1900-2100); sin ellos el backend responde **422**.

```jsx
import { useEffect, useState } from 'react';
import { Truck, Search, Printer, FileSpreadsheet, Plus, X, Pencil, Trash2 } from 'lucide-react';
import DataTable from '../components/common/DataTable';
import {
  getVehiculos, getVehiculo, createVehiculo, updateVehiculo, deleteVehiculo,
} from '../services/vehiculoService';

const INPUT_STYLE = {
  backgroundColor: 'var(--color-surface)',
  borderColor: 'var(--color-border)',
  color: 'var(--color-text-primary)',
};

const initialForm = {
  vhe_descripcion: '',
  vhe_mar_id: '',
  vhe_res_id: '',
  vhe_modelo: '',
  vhe_combustible: 0,
  vhe_estatus: true,
};

export default function VehiculosPage() {
  const [pageSize, setPageSize] = useState(20);
  const [data, setData] = useState([]);
  const [loading, setLoading] = useState(true);
  const [page, setPage] = useState(1);
  const [search, setSearch] = useState('');
  const [showModal, setShowModal] = useState(false);
  const [editId, setEditId] = useState(null);
  const [form, setForm] = useState(initialForm);
  const [saving, setSaving] = useState(false);
  const [error, setError] = useState('');

  const cargar = () => {
    setLoading(true);
    getVehiculos({ page: 1, page_size: 100 })
      .then((d) => setData(Array.isArray(d.items) ? d.items : []))
      .catch(console.error)
      .finally(() => setLoading(false));
  };

  useEffect(() => { cargar(); }, []);

  const filtered = search
    ? data.filter((v) =>
        (v.vhe_descripcion || '').toLowerCase().includes(search.toLowerCase()) ||
        String(v.vhe_modelo || '').includes(search)
      )
    : data;

  const total = filtered.length;
  const totalPages = Math.ceil(total / pageSize) || 1;
  const paginated = filtered.slice((page - 1) * pageSize, page * pageSize);

  useEffect(() => { setPage(1); }, [search, pageSize]);

  const COLUMNS = [
    { key: 'vhe_id', label: 'ID' },
    { key: 'vhe_descripcion', label: 'Descripción' },
    { key: 'vhe_modelo', label: 'Modelo' },
    { key: 'vhe_combustible', label: 'Combustible', render: (v) => `${v}%` },
    {
      key: 'vhe_estatus', label: 'Estatus',
      render: (v) => (
        <span className="text-[10px] px-2 py-0.5 rounded-full font-semibold"
          style={{ backgroundColor: v ? 'rgba(22,163,74,0.12)' : 'rgba(220,38,38,0.10)', color: v ? '#16a34a' : '#dc2626' }}>
          {v ? 'Activo' : 'Inactivo'}
        </span>
      ),
    },
    {
      key: 'acciones', label: '', render: (_, row) => (
        <div className="flex items-center gap-1">
          <button type="button" onClick={() => openEdit(row)}
            className="p-1 rounded transition-colors"
            style={{ color: 'var(--color-text-primary)' }}
            onMouseEnter={(e) => e.currentTarget.style.color = 'var(--color-accent)'}
            onMouseLeave={(e) => e.currentTarget.style.color = ''}
            title="Editar"><Pencil size={13} /></button>
          <button type="button" onClick={() => openDelete(row)}
            className="p-1 rounded transition-colors"
            style={{ color: 'var(--color-text-primary)' }}
            onMouseEnter={(e) => e.currentTarget.style.color = '#dc2626'}
            onMouseLeave={(e) => e.currentTarget.style.color = ''}
            title="Eliminar"><Trash2 size={13} /></button>
        </div>
      ),
    },
  ];

  const openNew = () => {
    setEditId(null);
    setForm(initialForm);
    setError('');
    setShowModal(true);
  };

  const openEdit = async (row) => {
    try {
      const detail = await getVehiculo(row.vhe_id);
      setEditId(detail.vhe_id);
      setForm({
        vhe_descripcion: detail.vhe_descripcion || '',
        vhe_mar_id: detail.vhe_mar_id ?? '',
        vhe_res_id: detail.vhe_res_id ?? '',
        vhe_modelo: detail.vhe_modelo ?? '',
        vhe_combustible: detail.vhe_combustible ?? 0,
        vhe_estatus: detail.vhe_estatus,
      });
      setError('');
      setShowModal(true);
    } catch (err) { console.error(err); }
  };

  const openDelete = async (row) => {
    if (!window.confirm(`¿Eliminar el vehículo "${row.vhe_descripcion}"?`)) return;
    try {
      await deleteVehiculo(row.vhe_id);
      cargar();
    } catch (err) { alert(err.message); }
  };

  const closeModal = () => {
    setShowModal(false);
    setEditId(null);
    setError('');
  };

  const handleChange = (e) => {
    const { name, value, type, checked } = e.target;
    setForm((prev) => ({ ...prev, [name]: type === 'checkbox' ? checked : value }));
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!form.vhe_descripcion.trim() || !form.vhe_mar_id || !form.vhe_res_id || !form.vhe_modelo) {
      setError('Descripción, marca (ID), responsable (ID) y modelo son obligatorios.');
      return;
    }
    setSaving(true);
    setError('');
    const payload = {
      vhe_descripcion: form.vhe_descripcion.trim(),
      vhe_mar_id: Number(form.vhe_mar_id),
      vhe_res_id: Number(form.vhe_res_id),
      vhe_modelo: Number(form.vhe_modelo),
      vhe_combustible: Number(form.vhe_combustible) || 0,
      vhe_estatus: form.vhe_estatus,
    };
    try {
      if (editId) await updateVehiculo(editId, payload);
      else await createVehiculo(payload);
      setShowModal(false);
      setEditId(null);
      setForm(initialForm);
      cargar();
    } catch (err) {
      setError(err.message || 'Error al guardar');
    } finally {
      setSaving(false);
    }
  };

  const handlePrint = () => window.print();

  const handleExportExcel = () => {
    const headers = ['ID', 'Descripción', 'Modelo', 'Combustible', 'Estatus'];
    const csvContent = [
      headers.join(','),
      ...filtered.map((v) => [v.vhe_id, v.vhe_descripcion, v.vhe_modelo, v.vhe_combustible, v.vhe_estatus].join(',')),
    ].join('\n');
    const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.setAttribute('download', 'vehiculos.csv');
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
          <span className="text-xs font-medium" style={{ color: 'var(--color-text-primary)' }}>Vehículos</span>
          <span className="text-[11px] ml-2" style={{ color: 'var(--color-text-muted)' }}>
            Catálogo de pipas y vehículos del sistema
          </span>
        </div>
        <div className="flex items-center gap-1 text-[11px]" style={{ color: 'var(--color-text-primary)' }}>
          <Truck size={12} />
          <span>{total} vehículos</span>
        </div>
      </div>

      <div className="flex items-center justify-end gap-2 px-5 py-1.5 shrink-0"
        style={{ borderBottom: '1px solid var(--color-border)' }}>
        <div className="relative w-96">
          <Search size={13} className="absolute left-2 top-1/2 -translate-y-1/2"
            style={{ color: 'var(--color-text-primary)' }} />
          <input type="text" placeholder="Buscar..." value={search}
            onChange={(e) => setSearch(e.target.value)}
            className="w-full pl-7 pr-2 py-1 text-xs rounded border outline-none placeholder:text-[var(--color-text-muted)]"
            style={{ ...INPUT_STYLE, borderColor: 'var(--color-accent)' }} />
        </div>
        <button onClick={openNew} className="flex items-center gap-1 px-2 py-1 text-[11px] font-medium rounded transition-colors"
          style={{ color: '#fff', backgroundColor: 'var(--color-accent)' }}
          onMouseEnter={(e) => e.currentTarget.style.opacity = '0.85'}
          onMouseLeave={(e) => e.currentTarget.style.opacity = '1'}>
          <Plus size={13} /> Nuevo vehículo
        </button>
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

      <div className="flex-1 min-h-0 mx-5 my-5">
        <DataTable columns={COLUMNS} data={paginated} loading={loading}
          pagination={{ page, totalPages, total, onPageChange: setPage }}
          pageSize={pageSize} onPageSizeChange={setPageSize} fill />
      </div>

      {showModal && (
        <div className="fixed inset-0 z-50 flex items-center justify-center"
          style={{ backgroundColor: 'rgba(0,0,0,0.4)' }} onClick={closeModal}>
          <div className="rounded-lg shadow-lg w-full max-w-md"
            style={{ backgroundColor: 'var(--color-surface-elevated)', border: '1px solid var(--color-border)' }}
            onClick={(e) => e.stopPropagation()}>
            <div className="flex items-center justify-between px-4 py-2.5"
              style={{ borderBottom: '1px solid var(--color-border)' }}>
              <span className="text-xs font-semibold" style={{ color: 'var(--color-text-primary)' }}>
                {editId ? 'Editar vehículo' : 'Nuevo vehículo'}
              </span>
              <button onClick={closeModal} className="p-0.5 rounded transition-colors"
                style={{ color: 'var(--color-text-muted)' }}>
                <X size={14} />
              </button>
            </div>
            <form onSubmit={handleSubmit} className="flex flex-col gap-3 px-4 py-3">
              {error && (
                <div className="text-[11px] px-2 py-1.5 rounded"
                  style={{ backgroundColor: '#fef2f2', color: '#991b1b', border: '1px solid #fecaca' }}>
                  {error}
                </div>
              )}
              <div>
                <label className="block text-[11px] font-medium mb-1" style={{ color: 'var(--color-text-primary)' }}>Descripción *</label>
                <input name="vhe_descripcion" value={form.vhe_descripcion} onChange={handleChange} required
                  className="w-full px-2 py-1 text-xs rounded border outline-none"
                  style={INPUT_STYLE} placeholder="Ej. Pipa 01" />
              </div>
              <div className="grid grid-cols-3 gap-3">
                <div>
                  <label className="block text-[11px] font-medium mb-1" style={{ color: 'var(--color-text-primary)' }}>Marca (ID) *</label>
                  <input name="vhe_mar_id" type="number" value={form.vhe_mar_id} onChange={handleChange} required min="1"
                    className="w-full px-2 py-1 text-xs rounded border outline-none"
                    style={INPUT_STYLE} placeholder="Ej. 1" />
                </div>
                <div>
                  <label className="block text-[11px] font-medium mb-1" style={{ color: 'var(--color-text-primary)' }}>Responsable (ID) *</label>
                  <input name="vhe_res_id" type="number" value={form.vhe_res_id} onChange={handleChange} required min="1"
                    className="w-full px-2 py-1 text-xs rounded border outline-none"
                    style={INPUT_STYLE} placeholder="Ej. 1" />
                </div>
                <div>
                  <label className="block text-[11px] font-medium mb-1" style={{ color: 'var(--color-text-primary)' }}>Modelo (año) *</label>
                  <input name="vhe_modelo" type="number" value={form.vhe_modelo} onChange={handleChange} required min="1900" max="2100"
                    className="w-full px-2 py-1 text-xs rounded border outline-none"
                    style={INPUT_STYLE} placeholder="Ej. 2020" />
                </div>
              </div>
              <div className="grid grid-cols-2 gap-3">
                <div>
                  <label className="block text-[11px] font-medium mb-1" style={{ color: 'var(--color-text-primary)' }}>Combustible (%)</label>
                  <input name="vhe_combustible" type="number" value={form.vhe_combustible} onChange={handleChange}
                    min="0" max="100" className="w-full px-2 py-1 text-xs rounded border outline-none"
                    style={INPUT_STYLE} placeholder="0-100" />
                </div>
                <div>
                  <label className="block text-[11px] font-medium mb-1" style={{ color: 'var(--color-text-primary)' }}>Estatus</label>
                  <select name="vhe_estatus" value={form.vhe_estatus} onChange={handleChange}
                    className="w-full px-2 py-1 text-xs rounded border outline-none" style={INPUT_STYLE}>
                    <option value={true}>Activo</option>
                    <option value={false}>Inactivo</option>
                  </select>
                </div>
              </div>
              <div className="flex justify-end gap-2 pt-2" style={{ borderTop: '1px solid var(--color-border)' }}>
                <button type="button" onClick={closeModal}
                  className="px-3 py-1 text-[11px] rounded border transition-colors"
                  style={{ borderColor: 'var(--color-border)', color: 'var(--color-text-primary)' }}>
                  Cancelar
                </button>
                <button type="submit" disabled={saving}
                  className="px-3 py-1 text-[11px] font-medium rounded transition-colors disabled:opacity-50"
                  style={{ color: 'var(--color-surface)', backgroundColor: 'var(--color-accent)' }}>
                  {saving ? 'Guardando...' : 'Guardar'}
                </button>
              </div>
            </form>
          </div>
        </div>
      )}
    </div>
  );
}
```

**DoD:** se pueden crear, editar y eliminar vehículos; el listado refresca desde la API y los
errores de unicidad (409 "La descripcion ya existe") se muestran en el modal.

---

## TAREA 3.7 — `VehiculoMarcasPage.jsx` (marcas de vehículo)

### 3.7.1 Crear `frontend/src/services/vehiculoMarcaService.js`

**OJO:** `GET /vehiculomarcas` devuelve una **lista plana** (no `PagedResponse`); `getMarcas()`
resuelve directamente al array (`setData(d)`).

```js
import { api } from './api';

export function getMarcas() {
  return api.get('/vehiculomarcas');
}

export function createMarca(body) {
  return api.post('/vehiculomarcas', body);
}

export function getMarca(id) {
  return api.get(`/vehiculomarcas/${id}`);
}

export function updateMarca(id, body) {
  return api.put(`/vehiculomarcas/${id}`, body);
}

export function deleteMarca(id) {
  return api.delete(`/vehiculomarcas/${id}`);
}
```

### 3.7.2 Crear `frontend/src/pages/VehiculoMarcasPage.jsx`

Mismo patrón de 3.6 (pero datos planos). El schema exige `vho_nombremarca` (única) y `vho_estatus`.

```jsx
import { useEffect, useState } from 'react';
import { BookOpen, Search, Printer, FileSpreadsheet, Plus, X, Pencil, Trash2 } from 'lucide-react';
import DataTable from '../components/common/DataTable';
import { getMarcas, createMarca, updateMarca, deleteMarca } from '../services/vehiculoMarcaService';

const INPUT_STYLE = {
  backgroundColor: 'var(--color-surface)',
  borderColor: 'var(--color-border)',
  color: 'var(--color-text-primary)',
};

const initialForm = { vho_nombremarca: '', vho_estatus: true };

export default function VehiculoMarcasPage() {
  const [pageSize, setPageSize] = useState(20);
  const [data, setData] = useState([]);
  const [loading, setLoading] = useState(true);
  const [page, setPage] = useState(1);
  const [search, setSearch] = useState('');
  const [showModal, setShowModal] = useState(false);
  const [editId, setEditId] = useState(null);
  const [form, setForm] = useState(initialForm);
  const [saving, setSaving] = useState(false);
  const [error, setError] = useState('');

  const cargar = () => {
    setLoading(true);
    getMarcas()
      .then((d) => setData(Array.isArray(d) ? d : []))
      .catch(console.error)
      .finally(() => setLoading(false));
  };

  useEffect(() => { cargar(); }, []);

  const filtered = search
    ? data.filter((m) => (m.vho_nombremarca || '').toLowerCase().includes(search.toLowerCase()))
    : data;

  const total = filtered.length;
  const totalPages = Math.ceil(total / pageSize) || 1;
  const paginated = filtered.slice((page - 1) * pageSize, page * pageSize);

  useEffect(() => { setPage(1); }, [search, pageSize]);

  const COLUMNS = [
    { key: 'vho_id', label: 'ID' },
    { key: 'vho_nombremarca', label: 'Marca' },
    {
      key: 'vho_estatus', label: 'Estatus',
      render: (v) => (
        <span className="text-[10px] px-2 py-0.5 rounded-full font-semibold"
          style={{ backgroundColor: v ? 'rgba(22,163,74,0.12)' : 'rgba(220,38,38,0.10)', color: v ? '#16a34a' : '#dc2626' }}>
          {v ? 'Activo' : 'Inactivo'}
        </span>
      ),
    },
    {
      key: 'acciones', label: '', render: (_, row) => (
        <div className="flex items-center gap-1">
          <button type="button" onClick={() => openEdit(row)}
            className="p-1 rounded transition-colors"
            style={{ color: 'var(--color-text-primary)' }}
            onMouseEnter={(e) => e.currentTarget.style.color = 'var(--color-accent)'}
            onMouseLeave={(e) => e.currentTarget.style.color = ''}
            title="Editar"><Pencil size={13} /></button>
          <button type="button" onClick={() => openDelete(row)}
            className="p-1 rounded transition-colors"
            style={{ color: 'var(--color-text-primary)' }}
            onMouseEnter={(e) => e.currentTarget.style.color = '#dc2626'}
            onMouseLeave={(e) => e.currentTarget.style.color = ''}
            title="Eliminar"><Trash2 size={13} /></button>
        </div>
      ),
    },
  ];

  const openNew = () => {
    setEditId(null);
    setForm(initialForm);
    setError('');
    setShowModal(true);
  };

  const openEdit = (row) => {
    setEditId(row.vho_id);
    setForm({
      vho_nombremarca: row.vho_nombremarca || '',
      vho_estatus: row.vho_estatus,
    });
    setError('');
    setShowModal(true);
  };

  const openDelete = async (row) => {
    if (!window.confirm(`¿Eliminar la marca "${row.vho_nombremarca}"?`)) return;
    try {
      await deleteMarca(row.vho_id);
      cargar();
    } catch (err) { alert(err.message); }
  };

  const closeModal = () => {
    setShowModal(false);
    setEditId(null);
    setError('');
  };

  const handleChange = (e) => {
    const { name, value, type, checked } = e.target;
    setForm((prev) => ({ ...prev, [name]: type === 'checkbox' ? checked : value }));
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!form.vho_nombremarca.trim()) {
      setError('El nombre de la marca es obligatorio.');
      return;
    }
    setSaving(true);
    setError('');
    const payload = { vho_nombremarca: form.vho_nombremarca.trim(), vho_estatus: form.vho_estatus };
    try {
      if (editId) await updateMarca(editId, payload);
      else await createMarca(payload);
      setShowModal(false);
      setEditId(null);
      setForm(initialForm);
      cargar();
    } catch (err) {
      setError(err.message || 'Error al guardar');
    } finally {
      setSaving(false);
    }
  };

  const handlePrint = () => window.print();

  const handleExportExcel = () => {
    const csvContent = [
      ['ID', 'Marca', 'Estatus'].join(','),
      ...filtered.map((m) => [m.vho_id, m.vho_nombremarca, m.vho_estatus].join(',')),
    ].join('\n');
    const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.setAttribute('download', 'marcas.csv');
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
          <span className="text-xs font-medium" style={{ color: 'var(--color-text-primary)' }}>Marcas de vehículo</span>
          <span className="text-[11px] ml-2" style={{ color: 'var(--color-text-muted)' }}>
            Catálogo de marcas de pipas
          </span>
        </div>
        <div className="flex items-center gap-1 text-[11px]" style={{ color: 'var(--color-text-primary)' }}>
          <BookOpen size={12} />
          <span>{total} marcas</span>
        </div>
      </div>

      <div className="flex items-center justify-end gap-2 px-5 py-1.5 shrink-0"
        style={{ borderBottom: '1px solid var(--color-border)' }}>
        <div className="relative w-96">
          <Search size={13} className="absolute left-2 top-1/2 -translate-y-1/2"
            style={{ color: 'var(--color-text-primary)' }} />
          <input type="text" placeholder="Buscar..." value={search}
            onChange={(e) => setSearch(e.target.value)}
            className="w-full pl-7 pr-2 py-1 text-xs rounded border outline-none placeholder:text-[var(--color-text-muted)]"
            style={{ ...INPUT_STYLE, borderColor: 'var(--color-accent)' }} />
        </div>
        <button onClick={openNew} className="flex items-center gap-1 px-2 py-1 text-[11px] font-medium rounded transition-colors"
          style={{ color: '#fff', backgroundColor: 'var(--color-accent)' }}
          onMouseEnter={(e) => e.currentTarget.style.opacity = '0.85'}
          onMouseLeave={(e) => e.currentTarget.style.opacity = '1'}>
          <Plus size={13} /> Nueva marca
        </button>
        <button onClick={handlePrint} className="p-1 rounded transition-colors"
          style={{ color: 'var(--color-text-primary)' }} title="Imprimir">
          <Printer size={15} />
        </button>
        <button onClick={handleExportExcel} className="p-1 rounded transition-colors"
          style={{ color: 'var(--color-text-primary)' }} title="Exportar a Excel">
          <FileSpreadsheet size={15} />
        </button>
      </div>

      <div className="flex-1 min-h-0 mx-5 my-5">
        <DataTable columns={COLUMNS} data={paginated} loading={loading}
          pagination={{ page, totalPages, total, onPageChange: setPage }}
          pageSize={pageSize} onPageSizeChange={setPageSize} fill />
      </div>

      {showModal && (
        <div className="fixed inset-0 z-50 flex items-center justify-center"
          style={{ backgroundColor: 'rgba(0,0,0,0.4)' }} onClick={closeModal}>
          <div className="rounded-lg shadow-lg w-full max-w-sm"
            style={{ backgroundColor: 'var(--color-surface-elevated)', border: '1px solid var(--color-border)' }}
            onClick={(e) => e.stopPropagation()}>
            <div className="flex items-center justify-between px-4 py-2.5"
              style={{ borderBottom: '1px solid var(--color-border)' }}>
              <span className="text-xs font-semibold" style={{ color: 'var(--color-text-primary)' }}>
                {editId ? 'Editar marca' : 'Nueva marca'}
              </span>
              <button onClick={closeModal} className="p-0.5 rounded transition-colors"
                style={{ color: 'var(--color-text-muted)' }}>
                <X size={14} />
              </button>
            </div>
            <form onSubmit={handleSubmit} className="flex flex-col gap-3 px-4 py-3">
              {error && (
                <div className="text-[11px] px-2 py-1.5 rounded"
                  style={{ backgroundColor: '#fef2f2', color: '#991b1b', border: '1px solid #fecaca' }}>
                  {error}
                </div>
              )}
              <div>
                <label className="block text-[11px] font-medium mb-1" style={{ color: 'var(--color-text-primary)' }}>Nombre de marca *</label>
                <input name="vho_nombremarca" value={form.vho_nombremarca} onChange={handleChange} required
                  className="w-full px-2 py-1 text-xs rounded border outline-none"
                  style={INPUT_STYLE} placeholder="Ej. International" />
              </div>
              <div>
                <label className="block text-[11px] font-medium mb-1" style={{ color: 'var(--color-text-primary)' }}>Estatus</label>
                <select name="vho_estatus" value={form.vho_estatus} onChange={handleChange}
                  className="w-full px-2 py-1 text-xs rounded border outline-none" style={INPUT_STYLE}>
                  <option value={true}>Activo</option>
                  <option value={false}>Inactivo</option>
                </select>
              </div>
              <div className="flex justify-end gap-2 pt-2" style={{ borderTop: '1px solid var(--color-border)' }}>
                <button type="button" onClick={closeModal}
                  className="px-3 py-1 text-[11px] rounded border transition-colors"
                  style={{ borderColor: 'var(--color-border)', color: 'var(--color-text-primary)' }}>
                  Cancelar
                </button>
                <button type="submit" disabled={saving}
                  className="px-3 py-1 text-[11px] font-medium rounded transition-colors disabled:opacity-50"
                  style={{ color: 'var(--color-surface)', backgroundColor: 'var(--color-accent)' }}>
                  {saving ? 'Guardando...' : 'Guardar'}
                </button>
              </div>
            </form>
          </div>
        </div>
      )}
    </div>
  );
}
```

**DoD:** CRUD de marcas contra `GET/POST/PUT/DELETE /vehiculomarcas`; el 409 de nombre duplicado
("La marca ya existe") se muestra en el modal.

---

## TAREA 3.8 — `SolicitudPipaPage.jsx` (alta de solicitud de pipa)

`POST /solicitudes_pipa` ya existe y `crearSolicitud` ya está en `solicitudesService.js`; solo se
crea la página con el formulario.

**OJO zona horaria:** `new Date("2026-08-19T15:00").toISOString()` interpreta la hora como LOCAL y la
convierte a UTC (desplaza según la zona del navegador). Si solo te importa la fecha/hora sin correr,
envía el string plano del input: `spp_horaentrega: hora`. El código de abajo usa esa opción simple.
El schema `SolicitudPipaCreate` NO incluye "domicilio" ni "cantidad" → **no enviarlos** (el backend
rechazaría campos extra).

### 3.8.1 Crear `frontend/src/pages/SolicitudPipaPage.jsx`

```jsx
import { useEffect, useState } from 'react';
import { Truck, CalendarClock, User, Hash, Gauge } from 'lucide-react';
import { crearSolicitud } from '../services/solicitudesService';
import { getVehiculos } from '../services/vehiculoService';

const INPUT_STYLE = {
  backgroundColor: 'var(--color-surface)',
  borderColor: 'var(--color-border)',
  color: 'var(--color-text-primary)',
};

const initialForm = {
  vhe_id: '',
  hora: '',
  con: '',
  srv: '',
  licencia: '',
};

export default function SolicitudPipaPage() {
  const [vehiculos, setVehiculos] = useState([]);
  const [form, setForm] = useState(initialForm);
  const [saving, setSaving] = useState(false);
  const [error, setError] = useState('');
  const [ok, setOk] = useState('');

  useEffect(() => {
    getVehiculos({ page: 1, page_size: 100 })
      .then((d) => setVehiculos(Array.isArray(d.items) ? d.items : []))
      .catch(console.error);
  }, []);

  const handleChange = (e) => {
    const { name, value } = e.target;
    setForm((prev) => ({ ...prev, [name]: value }));
  };

  const crear = async (e) => {
    e.preventDefault();
    if (!form.vhe_id || !form.hora || !form.con || !form.srv || !form.licencia) {
      setError('Todos los campos son obligatorios.');
      return;
    }
    setSaving(true);
    setError('');
    setOk('');
    try {
      await crearSolicitud({
        spp_con: Number(form.con),
        spp_srv: Number(form.srv),
        spp_vhe_id: Number(form.vhe_id),
        spp_horaentrega: form.hora, // string plano del input datetime-local
        srv_licencia: form.licencia.trim(),
      });
      setForm(initialForm);
      setOk('Solicitud creada correctamente. Aparecerá en el Kanban /pipas.');
    } catch (err) {
      setError(err.message || 'Error al crear la solicitud');
    } finally {
      setSaving(false);
    }
  };

  return (
    <div className="flex flex-col h-full" style={{ backgroundColor: 'var(--color-surface)' }}>
      <div className="flex items-center justify-between px-5 py-2 shrink-0"
        style={{ borderBottom: '1px solid var(--color-border)' }}>
        <div>
          <span className="text-xs font-medium" style={{ color: 'var(--color-text-primary)' }}>Solicitud de pipa</span>
          <span className="text-[11px] ml-2" style={{ color: 'var(--color-text-muted)' }}>
            Alta de solicitud de abasto con pipa
          </span>
        </div>
      </div>

      <div className="flex-1 min-h-0 overflow-y-auto px-5 py-5">
        <div className="max-w-md rounded-lg border p-4"
          style={{ backgroundColor: 'var(--color-surface-elevated)', borderColor: 'var(--color-border)' }}>
          {error && (
            <div className="mb-3 text-[11px] px-2 py-1.5 rounded"
              style={{ backgroundColor: '#fef2f2', color: '#991b1b', border: '1px solid #fecaca' }}>
              {error}
            </div>
          )}
          {ok && (
            <div className="mb-3 text-[11px] px-2 py-1.5 rounded"
              style={{ backgroundColor: '#f0fdf4', color: '#166534', border: '1px solid #bbf7d0' }}>
              {ok}
            </div>
          )}
          <form onSubmit={crear} className="flex flex-col gap-3">
            <div>
              <label className="flex items-center gap-1 text-[11px] font-medium mb-1"
                style={{ color: 'var(--color-text-primary)' }}>
                <Truck size={11} /> Vehículo asignado *
              </label>
              <select name="vhe_id" value={form.vhe_id} onChange={handleChange} required
                className="w-full px-2 py-1 text-xs rounded border outline-none"
                style={INPUT_STYLE}>
                <option value="">Seleccionar vehículo...</option>
                {vehiculos.map((v) => (
                  <option key={v.vhe_id} value={v.vhe_id}>{v.vhe_descripcion}</option>
                ))}
              </select>
            </div>

            <div>
              <label className="flex items-center gap-1 text-[11px] font-medium mb-1"
                style={{ color: 'var(--color-text-primary)' }}>
                <CalendarClock size={11} /> Hora de entrega *
              </label>
              <input type="datetime-local" name="hora" value={form.hora} onChange={handleChange} required
                className="w-full px-2 py-1 text-xs rounded border outline-none"
                style={INPUT_STYLE} />
            </div>

            <div className="grid grid-cols-2 gap-3">
              <div>
                <label className="flex items-center gap-1 text-[11px] font-medium mb-1"
                  style={{ color: 'var(--color-text-primary)' }}>
                  <Hash size={11} /> Contribuyente (ID) *
                </label>
                <input type="number" name="con" value={form.con} onChange={handleChange} required min="1"
                  className="w-full px-2 py-1 text-xs rounded border outline-none"
                  style={INPUT_STYLE} placeholder="Ej. 1" />
              </div>
              <div>
                <label className="flex items-center gap-1 text-[11px] font-medium mb-1"
                  style={{ color: 'var(--color-text-primary)' }}>
                  <Gauge size={11} /> Servicio (ID) *
                </label>
                <input type="number" name="srv" value={form.srv} onChange={handleChange} required min="1"
                  className="w-full px-2 py-1 text-xs rounded border outline-none"
                  style={INPUT_STYLE} placeholder="Ej. 1" />
              </div>
            </div>

            <div>
              <label className="flex items-center gap-1 text-[11px] font-medium mb-1"
                style={{ color: 'var(--color-text-primary)' }}>
                <User size={11} /> Licencia del operador *
              </label>
              <input name="licencia" value={form.licencia} onChange={handleChange} required
                className="w-full px-2 py-1 text-xs rounded border outline-none"
                style={INPUT_STYLE} placeholder="Ej. LIC-ADMIN-01" />
            </div>

            <div className="flex justify-end gap-2 pt-2" style={{ borderTop: '1px solid var(--color-border)' }}>
              <button type="submit" disabled={saving}
                className="px-3 py-1 text-[11px] font-medium rounded transition-colors disabled:opacity-50"
                style={{ color: 'var(--color-surface)', backgroundColor: 'var(--color-accent)' }}>
                {saving ? 'Creando...' : 'Crear solicitud'}
              </button>
            </div>
          </form>
        </div>
      </div>
    </div>
  );
}
```

**DoD:** al dar de alta, aparece una fila en `GET /solicitudes_pipa` y en el Kanban `/pipas`.

---

## TAREA 3.9 — `PozosPage.jsx` (CRUD de pozos y disponibilidad)

### 3.9.1 Crear `frontend/src/services/pozoService.js`

```js
import { api } from './api';

export function getPozos(params = {}) {
  const qs = new URLSearchParams(params).toString();
  return api.get(`/pozos${qs ? `?${qs}` : ''}`);
}

export function getPozo(id) {
  return api.get(`/pozos/${id}`);
}

export function createPozo(body) {
  return api.post('/pozos', body);
}

export function updatePozo(id, body) {
  return api.put(`/pozos/${id}`, body);
}

export function deletePozo(id) {
  return api.delete(`/pozos/${id}`);
}
```

### 3.9.2 Crear `frontend/src/pages/PozosPage.jsx`

**OJO:** `GET /pozos` devuelve `PagedResponse` → en el `.then()` usar `setData(d.items)`.
El schema `PozoCreate` exige `poz_dom_id` (ID del domicilio), `poz_nombrepozo`, `poz_responsable`,
`poz_telefono`; `poz_estatus` es bool (disponibilidad, default `true`).

```jsx
import { useEffect, useState } from 'react';
import { Droplet, Search, Printer, FileSpreadsheet, Plus, X, Pencil, Trash2 } from 'lucide-react';
import DataTable from '../components/common/DataTable';
import { getPozos, getPozo, createPozo, updatePozo, deletePozo } from '../services/pozoService';

const INPUT_STYLE = {
  backgroundColor: 'var(--color-surface)',
  borderColor: 'var(--color-border)',
  color: 'var(--color-text-primary)',
};

const initialForm = {
  poz_dom_id: '',
  poz_nombrepozo: '',
  poz_responsable: '',
  poz_telefono: '',
  poz_estatus: true,
};

export default function PozosPage() {
  const [pageSize, setPageSize] = useState(20);
  const [data, setData] = useState([]);
  const [loading, setLoading] = useState(true);
  const [page, setPage] = useState(1);
  const [search, setSearch] = useState('');
  const [soloDisponibles, setSoloDisponibles] = useState(false);
  const [showModal, setShowModal] = useState(false);
  const [editId, setEditId] = useState(null);
  const [form, setForm] = useState(initialForm);
  const [saving, setSaving] = useState(false);
  const [error, setError] = useState('');

  const cargar = () => {
    setLoading(true);
    const params = { page: 1, page_size: 100 };
    if (soloDisponibles) params.estatus = true;
    getPozos(params)
      .then((d) => setData(Array.isArray(d.items) ? d.items : []))
      .catch(console.error)
      .finally(() => setLoading(false));
  };

  useEffect(() => { cargar(); }, [soloDisponibles]);

  const filtered = search
    ? data.filter((p) =>
        (p.poz_nombrepozo || '').toLowerCase().includes(search.toLowerCase()) ||
        (p.poz_responsable || '').toLowerCase().includes(search.toLowerCase())
      )
    : data;

  const total = filtered.length;
  const totalPages = Math.ceil(total / pageSize) || 1;
  const paginated = filtered.slice((page - 1) * pageSize, page * pageSize);

  useEffect(() => { setPage(1); }, [search, pageSize, soloDisponibles]);

  const COLUMNS = [
    { key: 'poz_id', label: 'ID' },
    { key: 'poz_nombrepozo', label: 'Pozo' },
    { key: 'poz_responsable', label: 'Responsable' },
    { key: 'poz_telefono', label: 'Teléfono' },
    {
      key: 'poz_estatus', label: 'Disponibilidad',
      render: (v) => (
        <span className="text-[10px] px-2 py-0.5 rounded-full font-semibold"
          style={{ backgroundColor: v ? 'rgba(22,163,74,0.12)' : 'rgba(220,38,38,0.10)', color: v ? '#16a34a' : '#dc2626' }}>
          {v ? 'Disponible' : 'No disponible'}
        </span>
      ),
    },
    {
      key: 'acciones', label: '', render: (_, row) => (
        <div className="flex items-center gap-1">
          <button type="button" onClick={() => openEdit(row)}
            className="p-1 rounded transition-colors"
            style={{ color: 'var(--color-text-primary)' }}
            onMouseEnter={(e) => e.currentTarget.style.color = 'var(--color-accent)'}
            onMouseLeave={(e) => e.currentTarget.style.color = ''}
            title="Editar"><Pencil size={13} /></button>
          <button type="button" onClick={() => openDelete(row)}
            className="p-1 rounded transition-colors"
            style={{ color: 'var(--color-text-primary)' }}
            onMouseEnter={(e) => e.currentTarget.style.color = '#dc2626'}
            onMouseLeave={(e) => e.currentTarget.style.color = ''}
            title="Eliminar"><Trash2 size={13} /></button>
        </div>
      ),
    },
  ];

  const openNew = () => {
    setEditId(null);
    setForm(initialForm);
    setError('');
    setShowModal(true);
  };

  const openEdit = async (row) => {
    try {
      const detail = await getPozo(row.poz_id);
      setEditId(detail.poz_id);
      setForm({
        poz_dom_id: detail.poz_dom_id ?? '',
        poz_nombrepozo: detail.poz_nombrepozo || '',
        poz_responsable: detail.poz_responsable || '',
        poz_telefono: detail.poz_telefono || '',
        poz_estatus: detail.poz_estatus,
      });
      setError('');
      setShowModal(true);
    } catch (err) { console.error(err); }
  };

  const openDelete = async (row) => {
    if (!window.confirm(`¿Eliminar el pozo "${row.poz_nombrepozo}"?`)) return;
    try {
      await deletePozo(row.poz_id);
      cargar();
    } catch (err) { alert(err.message); }
  };

  const closeModal = () => {
    setShowModal(false);
    setEditId(null);
    setError('');
  };

  const handleChange = (e) => {
    const { name, value, type, checked } = e.target;
    setForm((prev) => ({ ...prev, [name]: type === 'checkbox' ? checked : value }));
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!form.poz_dom_id || !form.poz_nombrepozo.trim() || !form.poz_responsable.trim() || !form.poz_telefono.trim()) {
      setError('Domicilio (ID), nombre, responsable y teléfono son obligatorios.');
      return;
    }
    setSaving(true);
    setError('');
    const payload = {
      poz_dom_id: Number(form.poz_dom_id),
      poz_nombrepozo: form.poz_nombrepozo.trim(),
      poz_responsable: form.poz_responsable.trim(),
      poz_telefono: form.poz_telefono.trim(),
      poz_estatus: form.poz_estatus,
    };
    try {
      if (editId) await updatePozo(editId, payload);
      else await createPozo(payload);
      setShowModal(false);
      setEditId(null);
      setForm(initialForm);
      cargar();
    } catch (err) {
      setError(err.message || 'Error al guardar');
    } finally {
      setSaving(false);
    }
  };

  const handlePrint = () => window.print();

  const handleExportExcel = () => {
    const headers = ['ID', 'Pozo', 'Responsable', 'Teléfono', 'Disponible'];
    const csvContent = [
      headers.join(','),
      ...filtered.map((p) => [p.poz_id, p.poz_nombrepozo, p.poz_responsable, p.poz_telefono, p.poz_estatus].join(',')),
    ].join('\n');
    const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.setAttribute('download', 'pozos.csv');
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
          <span className="text-xs font-medium" style={{ color: 'var(--color-text-primary)' }}>Pozos</span>
          <span className="text-[11px] ml-2" style={{ color: 'var(--color-text-muted)' }}>
            Puntos de abastecimiento y su disponibilidad
          </span>
        </div>
        <div className="flex items-center gap-1 text-[11px]" style={{ color: 'var(--color-text-primary)' }}>
          <Droplet size={12} />
          <span>{total} pozos</span>
        </div>
      </div>

      <div className="flex items-center justify-end gap-2 px-5 py-1.5 shrink-0"
        style={{ borderBottom: '1px solid var(--color-border)' }}>
        <label className="flex items-center gap-1.5 text-[11px] cursor-pointer select-none"
          style={{ color: 'var(--color-text-primary)' }}>
          <input type="checkbox" checked={soloDisponibles}
            onChange={(e) => setSoloDisponibles(e.target.checked)}
            className="rounded accent-[var(--color-accent)]" />
          Solo disponibles
        </label>
        <div className="relative w-96">
          <Search size={13} className="absolute left-2 top-1/2 -translate-y-1/2"
            style={{ color: 'var(--color-text-primary)' }} />
          <input type="text" placeholder="Buscar..." value={search}
            onChange={(e) => setSearch(e.target.value)}
            className="w-full pl-7 pr-2 py-1 text-xs rounded border outline-none placeholder:text-[var(--color-text-muted)]"
            style={{ ...INPUT_STYLE, borderColor: 'var(--color-accent)' }} />
        </div>
        <button onClick={openNew} className="flex items-center gap-1 px-2 py-1 text-[11px] font-medium rounded transition-colors"
          style={{ color: '#fff', backgroundColor: 'var(--color-accent)' }}
          onMouseEnter={(e) => e.currentTarget.style.opacity = '0.85'}
          onMouseLeave={(e) => e.currentTarget.style.opacity = '1'}>
          <Plus size={13} /> Nuevo pozo
        </button>
        <button onClick={handlePrint} className="p-1 rounded transition-colors"
          style={{ color: 'var(--color-text-primary)' }} title="Imprimir">
          <Printer size={15} />
        </button>
        <button onClick={handleExportExcel} className="p-1 rounded transition-colors"
          style={{ color: 'var(--color-text-primary)' }} title="Exportar a Excel">
          <FileSpreadsheet size={15} />
        </button>
      </div>

      <div className="flex-1 min-h-0 mx-5 my-5">
        <DataTable columns={COLUMNS} data={paginated} loading={loading}
          pagination={{ page, totalPages, total, onPageChange: setPage }}
          pageSize={pageSize} onPageSizeChange={setPageSize} fill />
      </div>

      {showModal && (
        <div className="fixed inset-0 z-50 flex items-center justify-center"
          style={{ backgroundColor: 'rgba(0,0,0,0.4)' }} onClick={closeModal}>
          <div className="rounded-lg shadow-lg w-full max-w-md"
            style={{ backgroundColor: 'var(--color-surface-elevated)', border: '1px solid var(--color-border)' }}
            onClick={(e) => e.stopPropagation()}>
            <div className="flex items-center justify-between px-4 py-2.5"
              style={{ borderBottom: '1px solid var(--color-border)' }}>
              <span className="text-xs font-semibold" style={{ color: 'var(--color-text-primary)' }}>
                {editId ? 'Editar pozo' : 'Nuevo pozo'}
              </span>
              <button onClick={closeModal} className="p-0.5 rounded transition-colors"
                style={{ color: 'var(--color-text-muted)' }}>
                <X size={14} />
              </button>
            </div>
            <form onSubmit={handleSubmit} className="flex flex-col gap-3 px-4 py-3">
              {error && (
                <div className="text-[11px] px-2 py-1.5 rounded"
                  style={{ backgroundColor: '#fef2f2', color: '#991b1b', border: '1px solid #fecaca' }}>
                  {error}
                </div>
              )}
              <div className="grid grid-cols-2 gap-3">
                <div>
                  <label className="block text-[11px] font-medium mb-1" style={{ color: 'var(--color-text-primary)' }}>Domicilio (ID) *</label>
                  <input name="poz_dom_id" type="number" value={form.poz_dom_id} onChange={handleChange} required min="1"
                    className="w-full px-2 py-1 text-xs rounded border outline-none"
                    style={INPUT_STYLE} placeholder="Ej. 1" />
                </div>
                <div>
                  <label className="block text-[11px] font-medium mb-1" style={{ color: 'var(--color-text-primary)' }}>Teléfono *</label>
                  <input name="poz_telefono" value={form.poz_telefono} onChange={handleChange} required
                    className="w-full px-2 py-1 text-xs rounded border outline-none"
                    style={INPUT_STYLE} placeholder="Ej. 33 1234 5678" />
                </div>
              </div>
              <div>
                <label className="block text-[11px] font-medium mb-1" style={{ color: 'var(--color-text-primary)' }}>Nombre del pozo *</label>
                <input name="poz_nombrepozo" value={form.poz_nombrepozo} onChange={handleChange} required
                  className="w-full px-2 py-1 text-xs rounded border outline-none"
                  style={INPUT_STYLE} placeholder="Ej. Pozo Norte" />
              </div>
              <div>
                <label className="block text-[11px] font-medium mb-1" style={{ color: 'var(--color-text-primary)' }}>Responsable *</label>
                <input name="poz_responsable" value={form.poz_responsable} onChange={handleChange} required
                  className="w-full px-2 py-1 text-xs rounded border outline-none"
                  style={INPUT_STYLE} placeholder="Ej. Ing. Juan Pérez" />
              </div>
              <div>
                <label className="block text-[11px] font-medium mb-1" style={{ color: 'var(--color-text-primary)' }}>Disponibilidad</label>
                <select name="poz_estatus" value={form.poz_estatus} onChange={handleChange}
                  className="w-full px-2 py-1 text-xs rounded border outline-none" style={INPUT_STYLE}>
                  <option value={true}>Disponible</option>
                  <option value={false}>No disponible</option>
                </select>
              </div>
              <div className="flex justify-end gap-2 pt-2" style={{ borderTop: '1px solid var(--color-border)' }}>
                <button type="button" onClick={closeModal}
                  className="px-3 py-1 text-[11px] rounded border transition-colors"
                  style={{ borderColor: 'var(--color-border)', color: 'var(--color-text-primary)' }}>
                  Cancelar
                </button>
                <button type="submit" disabled={saving}
                  className="px-3 py-1 text-[11px] font-medium rounded transition-colors disabled:opacity-50"
                  style={{ color: 'var(--color-surface)', backgroundColor: 'var(--color-accent)' }}>
                  {saving ? 'Guardando...' : 'Guardar'}
                </button>
              </div>
            </form>
          </div>
        </div>
      )}
    </div>
  );
}
```

**DoD:** CRUD de pozos contra `GET/POST/PUT/DELETE /pozos`; la disponibilidad se ve en el listado
y se alterna con el campo `poz_estatus`.

---

## TAREA 3.10 — `CombustiblePage.jsx` (registro de gasto por vehículo)

### 3.10.1 Crear `frontend/src/services/combustibleService.js`

```js
import { api } from './api';

export function registrarCombustible(body) {
  return api.post('/combustible', body);
}

export function getCombustible(params = {}) {
  const qs = new URLSearchParams(params).toString();
  return api.get(`/combustible${qs ? `?${qs}` : ''}`);
}
```

### 3.10.2 Crear `frontend/src/pages/CombustiblePage.jsx`

**OJO:** `GET /combustible` devuelve `PagedResponse` → en el `.then()` usar `setData(d.items)`.
El schema `CombustibleCreate` exige `reg_com_vhe_id`, `reg_com_lts`, `reg_com_cost`,
`reg_com_km_inicio`, `reg_com_km_final` (todos >= 0). El historial se filtra por
`getCombustible({ vehiculo_id })` (nombre del query param del backend).

```jsx
import { useEffect, useState } from 'react';
import { Fuel, Truck, Droplet, Banknote, Milestone } from 'lucide-react';
import DataTable from '../components/common/DataTable';
import { registrarCombustible, getCombustible } from '../services/combustibleService';
import { getVehiculos } from '../services/vehiculoService';

const INPUT_STYLE = {
  backgroundColor: 'var(--color-surface)',
  borderColor: 'var(--color-border)',
  color: 'var(--color-text-primary)',
};

const initialForm = {
  vhe_id: '',
  lts: '',
  cost: '',
  km_inicio: '',
  km_final: '',
};

export default function CombustiblePage() {
  const [vehiculos, setVehiculos] = useState([]);
  const [form, setForm] = useState(initialForm);
  const [saving, setSaving] = useState(false);
  const [error, setError] = useState('');
  const [ok, setOk] = useState('');
  const [data, setData] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    getVehiculos({ page: 1, page_size: 100 })
      .then((d) => setVehiculos(Array.isArray(d.items) ? d.items : []))
      .catch(console.error);
  }, []);

  const cargarHistorial = (vheId) => {
    setLoading(true);
    const params = vheId ? { vehiculo_id: vheId } : {};
    getCombustible({ ...params, page: 1, page_size: 100 })
      .then((d) => setData(Array.isArray(d.items) ? d.items : []))
      .catch(console.error)
      .finally(() => setLoading(false));
  };

  useEffect(() => { cargarHistorial(); }, []);

  const handleChange = (e) => {
    const { name, value } = e.target;
    setForm((prev) => ({ ...prev, [name]: value }));
  };

  const registrar = async (e) => {
    e.preventDefault();
    if (!form.vhe_id || form.lts === '' || form.cost === '' || form.km_inicio === '' || form.km_final === '') {
      setError('Todos los campos son obligatorios.');
      return;
    }
    setSaving(true);
    setError('');
    setOk('');
    try {
      await registrarCombustible({
        reg_com_vhe_id: Number(form.vhe_id),
        reg_com_lts: Number(form.lts),
        reg_com_cost: Number(form.cost),
        reg_com_km_inicio: Number(form.km_inicio),
        reg_com_km_final: Number(form.km_final),
      });
      setForm(initialForm);
      setOk('Gasto de combustible registrado.');
      cargarHistorial(form.vhe_id);
    } catch (err) {
      setError(err.message || 'Error al registrar');
    } finally {
      setSaving(false);
    }
  };

  const COLUMNS = [
    { key: 'reg_com_id', label: 'ID' },
    { key: 'reg_com_vhe_id', label: 'Vehículo ID' },
    { key: 'reg_com_lts', label: 'Litros', render: (v) => `${v} L` },
    { key: 'reg_com_cost', label: 'Costo', render: (v) => `$${v}` },
    { key: 'reg_com_km_inicio', label: 'KM inicio' },
    { key: 'reg_com_km_final', label: 'KM final' },
    { key: 'reg_com_fecha', label: 'Fecha', render: (v) => (v ? new Date(v).toLocaleString() : '') },
  ];

  return (
    <div className="flex flex-col h-full" style={{ backgroundColor: 'var(--color-surface)' }}>
      <div className="flex items-center justify-between px-5 py-2 shrink-0"
        style={{ borderBottom: '1px solid var(--color-border)' }}>
        <div>
          <span className="text-xs font-medium" style={{ color: 'var(--color-text-primary)' }}>Combustible</span>
          <span className="text-[11px] ml-2" style={{ color: 'var(--color-text-muted)' }}>
            Registro de gasto de combustible por vehículo
          </span>
        </div>
      </div>

      <div className="flex-1 min-h-0 overflow-y-auto px-5 py-5">
        <div className="max-w-md rounded-lg border p-4 mb-5"
          style={{ backgroundColor: 'var(--color-surface-elevated)', borderColor: 'var(--color-border)' }}>
          {error && (
            <div className="mb-3 text-[11px] px-2 py-1.5 rounded"
              style={{ backgroundColor: '#fef2f2', color: '#991b1b', border: '1px solid #fecaca' }}>
              {error}
            </div>
          )}
          {ok && (
            <div className="mb-3 text-[11px] px-2 py-1.5 rounded"
              style={{ backgroundColor: '#f0fdf4', color: '#166534', border: '1px solid #bbf7d0' }}>
              {ok}
            </div>
          )}
          <form onSubmit={registrar} className="flex flex-col gap-3">
            <div>
              <label className="flex items-center gap-1 text-[11px] font-medium mb-1"
                style={{ color: 'var(--color-text-primary)' }}>
                <Truck size={11} /> Vehículo *
              </label>
              <select name="vhe_id" value={form.vhe_id} onChange={handleChange} required
                className="w-full px-2 py-1 text-xs rounded border outline-none"
                style={INPUT_STYLE}>
                <option value="">Seleccionar vehículo...</option>
                {vehiculos.map((v) => (
                  <option key={v.vhe_id} value={v.vhe_id}>{v.vhe_descripcion}</option>
                ))}
              </select>
            </div>
            <div className="grid grid-cols-2 gap-3">
              <div>
                <label className="flex items-center gap-1 text-[11px] font-medium mb-1"
                  style={{ color: 'var(--color-text-primary)' }}>
                  <Droplet size={11} /> Litros *
                </label>
                <input type="number" step="0.01" name="lts" value={form.lts} onChange={handleChange} required min="0"
                  className="w-full px-2 py-1 text-xs rounded border outline-none"
                  style={INPUT_STYLE} placeholder="Ej. 150.00" />
              </div>
              <div>
                <label className="flex items-center gap-1 text-[11px] font-medium mb-1"
                  style={{ color: 'var(--color-text-primary)' }}>
                  <Banknote size={11} /> Costo ($) *
                </label>
                <input type="number" step="0.01" name="cost" value={form.cost} onChange={handleChange} required min="0"
                  className="w-full px-2 py-1 text-xs rounded border outline-none"
                  style={INPUT_STYLE} placeholder="Ej. 3600.00" />
              </div>
            </div>
            <div className="grid grid-cols-2 gap-3">
              <div>
                <label className="flex items-center gap-1 text-[11px] font-medium mb-1"
                  style={{ color: 'var(--color-text-primary)' }}>
                  <Milestone size={11} /> KM inicio *
                </label>
                <input type="number" name="km_inicio" value={form.km_inicio} onChange={handleChange} required min="0"
                  className="w-full px-2 py-1 text-xs rounded border outline-none"
                  style={INPUT_STYLE} placeholder="Ej. 12000" />
              </div>
              <div>
                <label className="flex items-center gap-1 text-[11px] font-medium mb-1"
                  style={{ color: 'var(--color-text-primary)' }}>
                  <Milestone size={11} /> KM final *
                </label>
                <input type="number" name="km_final" value={form.km_final} onChange={handleChange} required min="0"
                  className="w-full px-2 py-1 text-xs rounded border outline-none"
                  style={INPUT_STYLE} placeholder="Ej. 12250" />
              </div>
            </div>
            <div className="flex justify-end gap-2 pt-2" style={{ borderTop: '1px solid var(--color-border)' }}>
              <button type="submit" disabled={saving}
                className="px-3 py-1 text-[11px] font-medium rounded transition-colors disabled:opacity-50"
                style={{ color: 'var(--color-surface)', backgroundColor: 'var(--color-accent)' }}>
                {saving ? 'Registrando...' : 'Registrar gasto'}
              </button>
            </div>
          </form>
        </div>

        <div className="flex items-center gap-2 mb-2">
          <Fuel size={13} style={{ color: 'var(--color-text-muted)' }} />
          <span className="text-xs font-semibold" style={{ color: 'var(--color-text-primary)' }}>Historial de combustible</span>
        </div>
        <DataTable columns={COLUMNS} data={data} loading={loading} />
      </div>
    </div>
  );
}
```

**DoD:** `POST /combustible` registra el gasto y aparece en el historial del vehículo.

---

# Verificación del día (TODO EN DOCKER)

```bash
# Levantar servicios (el backend de pozos/combustible ya se levantó y probó el Martes)
docker compose up -d --build

# Frontend build (Vite)
docker compose exec frontend npm run build

# Backend sin regresiones (opcional: confirma que el backend del Martes sigue OK)
docker compose exec backend python -m pytest -q
```

Chequeos manuales (navegador, login `admin` / `admin123`):
1. `/vehiculos`: crear, editar y eliminar un vehículo; duplicar descripción → aparece el 409.
2. `/vehiculomarcas`: crear y editar una marca; duplicar nombre → 409.
3. `/solicitud-pipa`: dar de alta una solicitud; verificar que aparece en `GET /solicitudes_pipa`
   y en el Kanban `/pipas`.
4. `/pozos`: crear/editar un pozo y alternar disponibilidad; verificar que aparece en el listado.
5. `/combustible`: registrar gasto para un vehículo y verlo en el historial.
6. Confirmar que `solicitudes_pipa` con campos extra NO rompe (el backend rechaza campos no
   esperados por defecto; la pantalla de 3.8 no debe enviarlos).

## DoD del día + commit

- [ ] 3.6 `VehiculosPage.jsx` con CRUD contra `GET/POST/PUT/DELETE /vehiculos`.
- [ ] 3.7 `VehiculoMarcasPage.jsx` con CRUD contra `/vehiculomarcas`.
- [ ] 3.8 `SolicitudPipaPage.jsx` con `POST /solicitudes_pipa` funcional.
- [ ] 3.9 `PozosPage.jsx` con CRUD y disponibilidad contra `GET/POST/PUT/DELETE /pozos`.
- [ ] 3.10 `CombustiblePage.jsx` que registra y consulta contra `POST/GET /combustible`.
- [ ] Rutas registradas en `App.jsx` y enlaces en `Sidebar.jsx`.
- [ ] `npm run build` del frontend en verde.

Commit sugerido:

```bash
git add -A
git commit -m "feat(pipas): catalogos de vehiculos, marcas, solicitud, pozos y combustible (3.6-3.10)"
git push origin feature/pipas-catalogos
```




