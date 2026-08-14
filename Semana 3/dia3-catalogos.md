# HOJA DE INSTRUCCIONES — MODULO DE PIPAS — SPRINT 3 — DIA 3 (MIERCOLES)

**Autor:** Turno (Lider de turno)
**Fecha:** 2026-08-19
**Repositorio:** dapa2w — rama `feature/pipas-catalogos`
**Objetivo del dia:** Pantallas de catálogos del módulo de pipas (**solo frontend**): alta/edición de
vehículos y marcas (reutilizando el patrón de `EstadosPage` + `DataTable`), alta de solicitud de pipa,
CRUD de pozos con su disponibilidad, y registro de gasto de combustible por vehículo. El backend
necesario (pozos, combustible) se completó el Martes.

> **IMPORTANTE (convencion del sprint): TODAS las instalaciones, builds y pruebas se hacen
> DENTRO del contenedor Docker.** Nunca `npm install`/`npm run build`/`pytest` en la maquina
> host. Usar `docker compose exec frontend ...` y `docker compose exec backend ...`.

---

## Tareas del dia

| # | Tarea | Archivo(s) principal(es) | DoD |
|---|-------|--------------------------|-----|
| 3.6 | Alta/edición de vehículos (patrón `EstadosPage` + `DataTable`) | `frontend/src/pages/`, `frontend/src/services/` | CRUD de vehículos contra la API |
| 3.7 | Alta/edición de marcas de vehículo | `frontend/src/pages/`, `frontend/src/services/` | CRUD de marcas contra la API |
| 3.8 | Alta de solicitud de pipa (formulario) | `frontend/src/pages/` | `POST /solicitudes_pipa` funciona |
| 3.9 | Alta de pozos y gestión de disponibilidad (tabla `pozos`) | `frontend/src/pages/` | CRUD de pozos contra la API |
| 3.10 | Registro de gasto de combustible por vehículo | `frontend/src/pages/` | `POST /combustible` funciona |

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
  - `Vehiculo` (`vehiculos`): `vhe_id`, `vhe_mar_id` (FK a `fabricante.fab_id`, ver nota abajo),
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
- **Servicios frontend YA existentes:**
  - `vehiculoService.js`: `getVehiculos(params)`, `getVehiculosDisponibles()`. **Faltan**
    `createVehiculo`, `updateVehiculo`, `deleteVehiculo`.
  - `solicitudesService.js`: `getSolicitudes`, `cambiarEstadoSolicitud`, `crearSolicitud` (ya existe).
  - Falta crear `vehiculoMarcaService.js`, `pozoService.js`, `combustibleService.js`.
- **Frontend:** React 18 + Vite 5 + Tailwind + react-router-dom v6 + `lucide-react` + `recharts`.
  Patrón de catálogo a copiar: `EstadosPage.jsx` (header, buscador, exportar CSV, imprimir) +
  `DataTable.jsx`. Para alta/edición se añade un formulario/modal custom (no hay librería de modales).
- **API base y token:** `api.js` usa `import.meta.env.VITE_API_URL || '/api/v1'`; token en
  `localStorage` bajo `access_token`. `api` ya expone `.get/.post/.put/.patch/.delete`.
- **React StrictMode** activo: los efectos corren 2 veces en desarrollo.
- Los catálogos se registran en `frontend/src/App.jsx` (dentro de `DashboardLayout` con
  `ProtectedRoute`) y se enlazan desde `frontend/src/components/common/Sidebar.jsx`.
- Pruebas del día: manuales en navegador + `pytest` del backend. No hay suite de tests de frontend;
  la verificación es `npm run build` + prueba manual.

---

## TAREA 3.6 — `VehiculosPage.jsx` (alta/edición de vehículos)

1. Ampliar `frontend/src/services/vehiculoService.js` (agregar las funciones que faltan):

```js
export function createVehiculo(body)  { return api.post('/vehiculos', body); }
export function getVehiculo(id)       { return api.get(`/vehiculos/${id}`); }
export function updateVehiculo(id, body) { return api.put(`/vehiculos/${id}`, body); }
export function deleteVehiculo(id)    { return api.delete(`/vehiculos/${id}`); }
```

2. Crear `frontend/src/pages/VehiculosPage.jsx` con el patrón de `EstadosPage`:
   - Header con contador (`N vehículos`) + buscador + botones Imprimir/Exportar CSV.
   - `DataTable` con columnas: `vhe_descripcion`, `vhe_modelo`, `vhe_combustible`, `vhe_estatus`.
   - Botón **Nuevo vehículo** y acción por fila **Editar**/**Eliminar**.
   - Formulario (modal simple) con campos: `vhe_descripcion`, `vhe_modelo`, `vhe_combustible`
     (0-100), `vhe_estatus`. Guarda con `createVehiculo`/`updateVehiculo`.
   - Al crear/editar, validar error 409 ("La descripcion ya existe") y mostrarlo; luego `cargar()`.

3. Registrar ruta en `App.jsx` y enlace en `Sidebar.jsx` (menú `pipas`):

```jsx
{ id: 'vehiculos', label: 'Vehículos', icon: Truck, path: '/vehiculos' }
```

**DoD:** se pueden crear, editar y eliminar vehículos; el listado refresca desde la API y los
errores de unicidad se muestran al usuario.

---

## TAREA 3.7 — `VehiculoMarcasPage.jsx` (marcas de vehículo)

1. Crear `frontend/src/services/vehiculoMarcaService.js`:

```js
export function getMarcas()            { return api.get('/vehiculomarcas'); }
export function createMarca(body)      { return api.post('/vehiculomarcas', body); }
export function updateMarca(id, body)  { return api.put(`/vehiculomarcas/${id}`, body); }
export function deleteMarca(id)        { return api.delete(`/vehiculomarcas/${id}`); }
```

2. Crear `frontend/src/pages/VehiculoMarcasPage.jsx` (mismo patrón de 3.6):
   - `DataTable` con columnas `vho_nombremarca`, `vho_estatus`.
   - Alta/edición por modal con campo `vho_nombremarca` (única) y `vho_estatus`. Manejar 409.

3. Registrar ruta `/vehiculomarcas` en `App.jsx` y enlace en `Sidebar.jsx`:

```jsx
{ id: 'marcas', label: 'Marcas de vehículo', icon: BookOpen, path: '/vehiculomarcas' }
```

**DoD:** CRUD de marcas contra `GET/POST/PUT/DELETE /vehiculomarcas`.

---

## TAREA 3.8 — `SolicitudPipaPage.jsx` (alta de solicitud de pipa)

`POST /solicitudes_pipa` ya existe y `crearSolicitud` ya está en `solicitudesService.js`.

1. Crear `frontend/src/pages/SolicitudPipaPage.jsx` con formulario:
   - **Vehículo asignado** → `<select>` alimentado por `getVehiculos` (envía `spp_vhe_id`).
   - **Hora de entrega** → `<input type="datetime-local">` (se envía ISO; el backend espera `datetime`).
   - **Los demás campos son enteros numéricos** porque la API no expone catálogos de
     contribuyentes/servicios en v1 todavía: `spp_con` (contribuyente), `spp_srv` (servicio),
     `srv_licencia` (licencia del operador). Para el DoD se llenan con valores de prueba.
   - El sprint menciona también "domicilio" y "cantidad", PERO el schema `SolicitudPipaCreate` NO
     incluye esos campos → **no enviarlos** (el backend rechazaría campos extra). Dejarlos ocultos o
     como comentario de camino futuro.
   - Submit: `crearSolicitud({ spp_con, spp_srv, spp_vhe_id, spp_horaentrega, srv_licencia })`,
     luego limpiar el formulario y mostrar éxito.

```js
const crear = async (e) => {
  e.preventDefault();
  try {
    await crearSolicitud({
      spp_con: Number(con),
      spp_srv: Number(srv),
      spp_vhe_id: Number(vheId),
      spp_horaentrega: new Date(hora).toISOString(),
      srv_licencia: licencia,
    });
    alert('Solicitud creada');
  } catch (err) { alert(err.message); }
};
```

2. Registrar ruta `/solicitud-pipa` y cambiar en `Sidebar.jsx` el enlace existente
   `{ id: 'sol-pipa', label: 'Solicitud de pipa', icon: Truck, path: '#' }` de `'#'` a `'/solicitud-pipa'`.

**DoD:** al dar de alta, aparece una fila en `GET /solicitudes_pipa` y en el Kanban `/pipas`.

---

## TAREA 3.9 — `PozosPage.jsx` (CRUD de pozos y disponibilidad)

El backend de pozos ya quedó en el Martes (`GET/POST/PUT/DELETE /pozos`). Hoy solo se consume.

1. Verificar el servicio `frontend/src/services/pozoService.js` (creado en el Martes junto al
   backend, o crear/ajustar si hace falta):

```js
export function getPozos(params = {}) {
  const qs = new URLSearchParams(params).toString();
  return api.get(`/pozos${qs ? `?${qs}` : ''}`);
}
export function createPozo(body)    { return api.post('/pozos', body); }
export function updatePozo(id, body){ return api.put(`/pozos/${id}`, body); }
export function deletePozo(id)      { return api.delete(`/pozos/${id}`); }
```

2. Crear `frontend/src/pages/PozosPage.jsx` (patrón de 3.6):
   - `DataTable` con columnas `poz_id`, `poz_nombrepozo`, `poz_responsable`, `poz_telefono`,
     `poz_estatus` (mostrar como "Disponible"/"No disponible").
   - Alta/edición por modal; **la disponibilidad (alta/baja) se gestiona con el campo `poz_estatus`**.
   - Filtro de disponibilidad extra opcional: `getPozos({ estatus: true })`.

3. Registrar ruta `/pozos` en `App.jsx` y enlace en `Sidebar.jsx` (menú `pipas` o `calidad-agua`):
   `{ id: 'pozos-pipa', label: 'Pozos', icon: Droplet, path: '/pozos' }`.

**DoD:** CRUD de pozos contra `GET/POST/PUT/DELETE /pozos`; la disponibilidad se ve en el listado.

---

## TAREA 3.10 — `CombustiblePage.jsx` (registro de gasto por vehículo)

El backend de combustible ya quedó en el Martes (`POST /combustible`, `GET /combustible`). Hoy solo
se consume.

1. Verificar el servicio `frontend/src/services/combustibleService.js` (creado en el Martes junto al
   backend, o crear/ajustar si hace falta):

```js
export function registrarCombustible(body) { return api.post('/combustible', body); }
export function getCombustible(params = {}) {
  const qs = new URLSearchParams(params).toString();
  return api.get(`/combustible${qs ? `?${qs}` : ''}`);
}
```

2. Crear `frontend/src/pages/CombustiblePage.jsx`:
   - Formulario: `<select>` de vehículos (`getVehiculos`), `reg_com_lts`, `reg_com_cost`,
     `reg_com_km_inicio`, `reg_com_km_final`. Guarda con `registrarCombustible`.
   - `DataTable` del historial del vehículo seleccionado (`getCombustible({ vehiculo_id })`).
3. Registrar ruta `/combustible` en `App.jsx` y enlace en `Sidebar.jsx` (menú `pipas`):
   `{ id: 'combustible', label: 'Combustible', icon: Fuel, path: '/combustible' }`.

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