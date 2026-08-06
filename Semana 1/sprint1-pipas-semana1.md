# Sprint 1 — Semana 1: Fundaciones del Módulo de Pipas (Backend + Base de Datos)

- **Duración:** 5 días hábiles
- **Objetivo:** Tener la API funcional del módulo de pipas: catálogo de marcas y vehículos, y las solicitudes de pipa (ligadas al vehículo) con autenticación JWT, estados y pruebas unitarias.
- **Equipo recomendado:** 1 desarrollador backend + 1 de apoyo (BD) si es posible.
- **Branches:** `feature/pipas-semana1` como rama base; ramas por tarea (`feature/pipas-models`, `feature/pipas-api`, etc.) según el flujo del README.

> ⚠️ **Paso 0 obligatorio — Sincronizar el repositorio:**
> El git local está atrasado (rama `pipas` local = commit viejo). Las tablas ya existen en `origin/main`:
> ```bash
> git checkout main
> git pull origin main
> git checkout -b feature/pipas-semana1
> ```
> Si tenías cambios sin commitear en `docker-compose.yml`, `backend/Dockerfile` y `.env.example`, revísalos y aplícalos en la rama nueva (no se pierden).

---

## Tablas que YA existen (no se vuelven a crear)

| Tabla | Migración | Notas |
|-------|-----------|-------|
| `vehiculomarca` | `00039_crea_tabla_vehiculomarca.sql` | `vho_id`, `vho_nombremarca`, `vho_estatus` |
| `vehiculos` | `00043_crea_tabla_vehiculos.sql` | `vhe_id`, `vhe_mar_id → fabricante(fab_id)`, `vhe_modelo`, `vhe_combustible`, `vhe_tipovehiculo`, `vhe_descripcion`, `vhe_estatus` |
| `solicitudpipas` | `00044_crea_tabla_solicitudpipa.sql` | `spp_id`, `spp_con`, `spp_srv`, `spp_vhe_id → vehiculos(vhe_id)`, `spp_fechasolicitud`, `spp_horaentrega`, `srv_licencia`, `spp_estatus` |
| `fabricante` | `00034` y `00042` | Referenciada por `vehiculos.vhe_mar_id`. **Revisar duplicado de migraciones.** |
| `pozos` | `00011_crea_tabla_pozos.sql` | Disponible para fases posteriores |
| `servicios` | `00050_crea_tabla_servicios.sql` | Relacionada con `solicitudpipas.spp_srv` |
| `incidenciaAgua` | `00051_crear_tabla_incidenciaAgua.sql` | Otra fuente de solicitudes |

**La solicitud pipa se liga SOLO al vehículo** (`spp_vhe_id`), según decisión del equipo (sin FK al pozo).

---

## Lunes — Revisión de Esquema y Migraciones Complementarias

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 1.1 | Hacer pull de `origin/main` y revisar las tablas `vehiculomarca`, `vehiculos`, `solicitudpipas`, `fabricante`, `servicios` y su relación | terminal + `db/migrations/` | Mapa documentado de FK y columnas del módulo |
| 1.2 | **Investigar el duplicado** `00034_create_fabricante.sql` vs `00042_crea_tabla_fabricante.sql` y resolver (¿mismo esquema? ¿debe quitarse una?) | `db/migrations/` | Una sola definición de `fabricante` en el repo |
| 1.3 | Crear migración `00053_crea_tabla_historial_solicitud.sql`: bitácora de transiciones de estado de `solicitudpipas` (estado anterior, nuevo, fecha, usuario, motivo) | `db/migrations/` | Migración ejecutable y reversible |
| 1.4 | Crear migración `00054_crea_tabla_registros_combustible.sql`: gasto por vehículo (litros, costo, fecha, km inicial/final) | `db/migrations/` | Migración ejecutable y reversible |
| 1.5 | Crear migración `00055_crea_tabla_ubicaciones_pipa.sql`: lat/lng en tiempo real por vehículo (para el mapa en Semana 2) | `db/migrations/` | Migración ejecutable y reversible |
| 1.6 | Revisar seed existente `00024_seed_vehiculomarca.sql` y crear seeds de `vehiculos` y `solicitudpipas` (5 vehículos, 10 solicitudes con estados variados) | `db/seeds/` | `docker-compose up` puebla datos automáticamente |

### Índices y restricciones a incluir
- `IX` sobre `solicitudpipas.spp_vhe_id` y `solicitudpipas.spp_estatus`.
- `CHECK` sobre estados de solicitud (según el catálogo que definan: Pendiente / En Ruta / Entregada / Cancelada) o bien manejar el estatus booleano existente — decidir y documentar.
- FK correctos: `solicitudpipas.spp_vhe_id → vehiculos(vhe_id)`.

---

## Martes — Modelos SQLAlchemy y Repositorios

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 2.1 | Agregar `VehiculoMarcaModel` (tabla `vehiculomarca`) | `backend/app/infrastructure/database/models.py` | Clase alineada a la migración 00039 |
| 2.2 | Agregar `VehiculoModel` (tabla `vehiculos`) con relación a marca y solicitudes | `backend/app/infrastructure/database/models.py` | FK e índices alineados a la migración 00043 |
| 2.3 | Agregar `SolicitudPipaModel` (tabla `solicitudpipas`) con relación al vehículo | `backend/app/infrastructure/database/models.py` | FK a `vehiculos.vhe_id` |
| 2.4 | Agregar `HistorialSolicitudModel`, `RegistroCombustibleModel`, `UbicacionPipaModel` | `backend/app/infrastructure/database/models.py` | Idem a migraciones 00053-00055 |
| 2.5 | Crear repositorios: `VehiculoRepository`, `MarcaVehiculoRepository`, `SolicitudPipaRepository` | `backend/app/infrastructure/repositories/` | CRUD + `get_disponibles()`, `listar_por_estado()` |
| 2.6 | Registrar modelos en el `Base` de la sesión | `backend/app/infrastructure/database/session.py` | Sin errores de importación |

---

## Miércoles — Schemas Pydantic

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 3.1 | Crear `vehiculos.py`: `VehiculoCreate`, `VehiculoUpdate`, `VehiculoResponse` | `backend/app/api/schemas/` | Validación de modelo/año, descripción no vacía, combustible >= 0 |
| 3.2 | Crear `vehiculomarcas.py`: `MarcaVehiculoCreate`, `MarcaVehiculoResponse` | `backend/app/api/schemas/` | Nombre de marca requerido y único |
| 3.3 | Crear `solicitudes_pipa.py`: `SolicitudPipaCreate`, `SolicitudPipaUpdate`, `SolicitudPipaResponse`, `SolicitudPipaCambioEstado` | `backend/app/api/schemas/` | `spp_vhe_id` requerido; transición de estado validada |
| 3.4 | Crear `common.py` para paginación/ordenamiento reutilizable | `backend/app/api/schemas/common.py` | `PaginadoResponse[T]` genérico |

---

## Jueves — Endpoints V1 (API)

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 4.1 | Crear `vehiculos.py`: `GET/POST/PUT/PATCH/DELETE` + `GET /vehiculos/disponibles` | `backend/app/api/v1/` | CRUD protegido por JWT, listado paginado |
| 4.2 | Crear `vehiculomarcas.py`: CRUD del catálogo de marcas | `backend/app/api/v1/` | CRUD protegido por JWT |
| 4.3 | Crear `solicitudes_pipa.py`: `POST` alta (liga el vehículo), `GET` listado (filtros por estado/fecha), `GET /{id}`, `PATCH /{id}/estado` | `backend/app/api/v1/` | Cambio de estado registrado en `historial_solicitud` |
| 4.4 | Registrar routers en `router.py` | `backend/app/api/v1/router.py` | Rutas visibles en `/docs` bajo `/api/v1` |
| 4.5 | Registrar permisos del módulo en BD (seed: `pipas.leer`, `pipas.escribir`, `solicitudes.leer`, `solicitudes.gestionar`) | `db/seeds/` | Asignables a roles desde el panel |

---

## Viernes — Pruebas Unitarias y Cierre

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 5.1 | Tests de CRUD de vehículos | `backend/tests/` | Pytest verde |
| 5.2 | Tests de CRUD de marcas de vehículo | `backend/tests/` | Pytest verde |
| 5.3 | Tests de alta de solicitudes y transiciones de estado válidas e inválidas | `backend/tests/` | Pytest verde |
| 5.4 | Tests de autenticación: endpoints rechazan peticiones sin token (401) | `backend/tests/` | Pytest verde |
| 5.5 | Correr `pytest` completo + revisar `pre-commit` | terminal | 100% verde, sin warnings pendientes |
| 5.6 | Pull Request de todos los branches a `feature/pipas-semana1` y revisión | GitHub | PR revisado y mergeado con commits `feat:`/`test:` |

---

## Criterios de Aceptación del Sprint

1. Repositorio sincronizado con `origin/main` y las tablas `vehiculomarca`, `vehiculos` y `solicitudpipas` visibles en PostgreSQL.
2. Las migraciones 00053-00055 (historial, combustible, ubicaciones) se aplican sin errores.
3. `/api/v1/vehiculos`, `/api/v1/vehiculomarcas` y `/api/v1/solicitudes_pipa` funcionan en `/docs` con token JWT.
4. Una solicitud se crea ligada a un vehículo (`spp_vhe_id`) y su cambio de estado queda registrado en el historial.
5. `pytest` pasa en verde.

## Comandos útiles

```bash
git checkout main && git pull origin main      # sincronizar
docker-compose up -d --build                   # levantar todo
docker-compose exec backend pytest             # correr tests
docker-compose exec db psql -U postgres -d dapatlqdb  # inspeccionar BD
git flow: checkout -b feature/pipas-tablas      # ramas por tarea
```
