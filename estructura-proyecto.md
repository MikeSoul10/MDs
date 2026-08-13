# DAPA2W — Estructura del Proyecto

Guía de la arquitectura del proyecto DAPA2W: base de datos, backend, frontend y servicios.

---

## 1. Base de datos: Migrations y Seeds

### Migrations (`db/migrations/`)

Archivos SQL que **definen la estructura de la base de datos** (tablas, índices, vistas, procedimientos almacenados). Se ejecutan una sola vez y en orden (por el número de prefijo `00000`, `00002`, ...).

- `00000_crea_tabla_medidormarca.sql` → crea la tabla `medidormarca`.
- `00013_crea_tablas_auth.sql` → crea `roles`, `permisos`, `usuarios`, `tokens_negros`, `bitacora`.

**Objetivo:** que todos los ambientes (dev, staging, prod) tengan exactamente la misma estructura de BD de forma reproducible y controlada.

### Seeds (`db/seeds/`)

Archivos SQL que **insertan datos iniciales de prueba o catálogos**.

- `00001_seed_medidormarca.sql` → inserta 25 marcas de medidores.
- `00014_seed_auth.sql` → crea usuarios/roles base.

**Objetivo:** llenar la BD con datos para poder desarrollar y probar sin capturarlos a mano.

### Cómo se ejecutan

- `db/startup.py` (contenedor `db-migrate` en `docker-compose.yml`) las corre automáticamente al levantar los servicios.
- Lleva control en la tabla `schema_migrations` (`db/init/0000_schema_migrations.sql`): guarda qué archivos ya se aplicaron y **no los repite**.
- Las seeds se registran con prefijo `seed:` en esa misma tabla.
- **Regla clave:** si agregas una migración nueva, debe tener número mayor a las existentes (p. ej. `100059_...`), y nunca modifiques una ya aplicada.

---

## 2. Backend — Clean Architecture / DDD

El backend de DAPA2W usa **Clean Architecture / DDD (Domain-Driven Design)**. La idea central: la lógica de negocio (`domain`) no depende de frameworks, bases de datos ni HTTP. Cada capa solo apunta hacia adentro.

### Estructura de capas

**`app/api/` — Capa de presentación (HTTP)**
- `v1/*.py`: los endpoints FastAPI. Solo reciben la petición, llaman al repositorio y devuelven respuestas (ver CRUD de ejemplo en `v1/vehiculomarcas.py`).
- `schemas/*.py`: modelos Pydantic de entrada/salida (validación).
- `deps.py`: **inyección de dependencias**. Cablea repositorios, sesión de BD, servicios y autenticación (`get_current_user`, `get_*_repository`).
- `v1/router.py`: agrupa todos los routers en `/api/v1`.

**`app/domain/` — Capa de negocio (el corazón)**
Contiene entidades y contratos **sin dependencias de FastAPI ni SQLAlchemy**. Por módulo (`auth`, `ai`, `pipas`, `prompts`, `rag`, `storage`):
- `entities.py`: dataclasses puras (p. ej. `User`, `Role`).
- `interfaces.py`: contratos abstractos (`IUserRepository`, ...) que la infraestructura debe implementar.
- `value_objects.py`: objetos de valor.

**`app/infrastructure/` — Capa técnica (implementaciones concretas)**
- `database/`: `models.py` (SQLAlchemy), `session.py` (AsyncSession), `unit_of_work.py`.
- `repositories/`: implementan las interfaces del dominio con SQLAlchemy. Ej: `auth_repository.py` traduce `UserModel` (tabla) → `User` (entidad del dominio).
- `ai/`, `osrm/`, `storage/`: clientes externos (Ollama, OSRM, archivos).

**`app/core/` — Configuración y transversal**
`config.py` (settings), `security.py` (JWT/hash), `middleware.py` (CORS, rate limiting), `exceptions.py`, `websocket_manager.py`, `events.py` (startup/shutdown).

**`app/shared/` — Utilidades comunes**
`enums.py`, `constants.py`, `logger.py`, `utils.py`.

### Flujo de una petición

```
HTTP → api/v1/vehiculomarcas.py (endpoint)
     → api/deps.py (inyecta repo autenticado)
     → infrastructure/repositories/... (SQLAlchemy)
     → infrastructure/database/models.py (tablas)
     ← back: entidad de dominio → Pydantic schema → JSON
```

**Regla de oro:** el `domain` define las reglas y contratos; la `infrastructure` los implementa; la `api` solo orquesta y expone. Esto permite cambiar de BD, framework o cliente externo sin tocar la lógica de negocio.

---

## 3. Frontend — React + Vite + Tailwind

SPA **React 18 + Vite + Tailwind CSS + React Router**.

### Estructura (`frontend/src/`)

- **`main.jsx` / `App.jsx`**: punto de entrada y rutas. `App.jsx` define `/login`, `/chofer` (sin layout) y rutas protegidas con `ProtectedRoute` + `DashboardLayout`.
- **`pages/`**: vistas por pantalla (`LoginPage`, `DashboardPage`, `EstadosPage`, `CiudadesPage`, `ColoniasPage`, `OrganizacionPage`, `MapaSupervision`, `AppChofer`).
- **`components/`**: componentes reutilizables.
  - `common/`: `Sidebar`, `TopBar`, `DataTable`, `BottomPanel`, `RightPanel`, `ProtectedRoute`.
  - `dashboard/`: `MetricsCard`.
- **`services/`**: capa de API (equivalente a infraestructura).
  - `api.js`: cliente HTTP centralizado — adjunta el token JWT, maneja el 401 (redirige a `/login`), expone `api.get/post/put/delete/patch`.
  - `authService.js`, `ciudadesService.js`, `solicitudesService.js`, etc.: servicios por dominio.
- **`contexts/`**: estado global con React Context (`AuthContext` — sesión del usuario, `ThemeContext` — tema).
- **`layouts/`**: `DashboardLayout.jsx` (Sidebar + TopBar + contenido).
- **`constants/`**: `palettes.js` (colores para gráficas/mapas).
- **`public/`**: estáticos (favicon, `icons/`).

### Stack

Vite 5, React Router 6, Tailwind 3, Recharts (gráficas), Leaflet + react-leaflet (mapas), lucide-react (iconos), vite-plugin-pwa.

### Flujo típico

```
Page (EstadoPage) → services/estadosService.js → services/api.js (fetch + token + 401)
                                                        ↓
                                              /api/v1/estados (FastAPI)
```

Separación: `pages` (vistas) → `services` (HTTP) → `contexts` (estado compartido), con `components` reutilizables.

---

## 4. Lo que falta por explorar

### 4.1 Orquestación (`docker-compose.yml` y `dkcom.md`)

Cómo se conectan los servicios: `db` → `db-migrate` (corre migraciones/seeds) → `backend` (espera a que migre) → `frontend`. Más OSRM (rutas), pgAdmin y volúmenes compartidos.

### 4.2 El dominio de negocio: módulo `pipas`

Es el corazón del sistema. En `backend/app/domain/pipas/` está la lógica de **solicitudes de pipas, asignación, disponibilidad, agrupación, rutas y geo**. Repositorios relacionados: `solicitud_pipa_repository`, `asignacion_repository`, `ruta_solicitud_repository`. Endpoints: `solicitudes_pipa.py`, `asignaciones.py`. Es lo más específico de DAPA2W y donde el DDD se ve en acción.

### 4.3 Autenticación end-to-end

Flujo completo: `api/v1/auth.py` (login/refresh), `core/security.py` (JWT), `TokenBlacklist` (logout/revocación) y consumo en frontend (`authService.js`, `AuthContext`, `ProtectedRoute`).

### 4.4 Comunicación en tiempo real

- `api/v1/ubicacion_ws.py` + `core/websocket_manager.py`: WebSockets para ubicación de pipas en `MapaSupervision`.
- OSRM como motor de rutas (`infrastructure/osrm/`).

### 4.5 Infraestructura técnica del backend

- **Alembic** (`app/alembic/`): herramienta de migraciones del backend (convive con `startup.py`).
- `app/config.py` + `.env`: configuración vía variables de entorno.
- `tests/`: pruebas del backend.

### 4.6 Documentación del proyecto

- `documentacion/`: `API_DAPA.md`, `db.md`, `osrm-despliegue.md`, diagrama `INTRAESTRUCTURA DAPA2.drawio.pdf`, issues con especificaciones.
- `servicios.md` / `dkcom.md`: comandos de despliegue y credenciales.
- `README.md`: vista general.

**Orden sugerido de lectura:** `README.md` → `docker-compose.yml` → módulo `pipas` (solicitud → asignación → ruta → supervisión) → autenticación → WebSockets.

---

## Servicios (referencia rápida)

| Servicio    | URL                            |
|-------------|--------------------------------|
| Frontend    | http://localhost:3000          |
| Backend     | http://localhost:8000          |
| API Docs    | http://localhost:8000/docs     |
| pgAdmin     | http://localhost:5050          |
| PostgreSQL  | localhost:5432                 |

Ver `servicios.md` para credenciales y `dkcom.md` para comandos de despliegue.
