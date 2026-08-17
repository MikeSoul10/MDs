# HOJA DE INSTRUCCIONES — MODULO DE PIPAS — SPRINT 3 — DIA 2 (MARTES)

**Autor:** Turno (Lider de turno)
**Fecha:** 2026-08-18
**Repositorio:** dapa2w — rama `feature/pipas-backend`
**Objetivo del dia:** El Martes **no está en el sprint** (el sprint salta de Lunes a Miércoles).
Este día se usa para **completar el backend que el resto de la semana necesita y que hoy NO existe**:
CRUD de pozos (para 3.9), registro + listado de combustible (para 3.10) y los dos reportes del
jueves (3.11 y 3.12). Así, Miércoles y Jueves serán días 100% frontend.

> **IMPORTANTE (convencion del sprint): TODAS las instalaciones, builds y pruebas se hacen
> DENTRO del contenedor Docker.** Nunca `npm install`/`npm run build`/`pytest` en la maquina
> host. Usar `docker compose exec frontend ...` y `docker compose exec backend ...`.

---

## Tareas del dia

| # | Tarea | Archivo(s) principal(es) | DoD |
|---|-------|--------------------------|-----|
| B2.1 | Backend: CRUD de pozos (router + schemas + repo) | `backend/app/api/v1/pozos.py`, `backend/app/api/schemas/pozos.py`, `backend/app/infrastructure/repositories/pozo_repository.py` | `GET/POST/PUT/DELETE /pozos` en `/docs` |
| B2.2 | Backend: registro e historial de combustible (repo + schemas + router) | `backend/app/api/v1/combustible.py`, `backend/app/api/schemas/combustible.py`, `backend/app/infrastructure/repositories/registro_combustible_repository.py` | `POST /combustible` y `GET /combustible` funcionan |
| B2.3 | Backend: `GET /reportes/combustible` (filtros fecha/vehículo, totales por día) | `backend/app/api/v1/reportes.py`, `backend/app/api/schemas/reportes.py` | Totales agregados correctos |
| B2.4 | Backend: `GET /reportes/tiempos_traslado` (duración promedio, entregas por día, km recorridos) | `backend/app/api/v1/reportes.py`, `backend/app/api/schemas/reportes.py` | Calcula desde `rutas_solicitud` y `ubicaciones_pipa` |

**DoD del dia:** `/docs` expone los routers `pozos`, `combustible` y `reportes`; CRUD de pozos y
combustible operan, los reportes devuelven agregados correctos y `pytest` del backend en verde.

---

## 0. Prerequisitos y convenciones

- **Modelos YA existen en `app/infrastructure/database/models.py`** (no crear tablas):
  - `PozoModel` (tabla `pozos`): `poz_id` (PK), `poz_dom_id`, `poz_nombrepozo`,
    `poz_responsable`, `poz_telefono`, `poz_fechaalta` (server_default), `poz_estatus` (bool =
    disponibilidad).
  - `RegistroCombustibleModel` (tabla `registros_combustible`): `reg_com_id` (PK), `reg_com_vhe_id`
    (FK vehículos), `reg_com_lts` (Numeric), `reg_com_cost` (Numeric), `reg_com_fecha`
    (server_default), `reg_com_km_inicio`, `reg_com_km_final`.
  - `RutaSolicitudModel` (tabla `rutas_solicitud`): `rtsol_id`, `rtsol_spp_id`,
    `rtsol_distancia_km`, `rtsol_duracion_min`, `rtsol_creado`.
  - `UbicacionPipaModel` (tabla `ubicacionespipa`): `ubp_id`, `ubp_vhe_id`, `ubp_timestamp`.
  - `SolicitudPipaModel` (tabla `solicitudpipas`): `spp_id`, `spp_vhe_id`, `spp_horaentrega`,
    `spp_estatus`.
- **⚠️ `GenericRepository` (`repositories/base.py`):** sus métodos `get_by_id`, `delete` y
  `exists` usan `self.model.id`, PERO los modelos de pipas usan PKs con prefijo (`poz_id`,
  `reg_com_id`). Todo repositorio nuevo (o el existente de pozos) DEBE sobreescribir `get_by_id`
  (y `delete` si se usa) con el PK real. Ver `vehiculo_repository.py` como referencia.
- **Dependencies ya disponibles en `app/api/deps.py`:** `get_session`, `get_current_user`,
  `get_vehiculo_repository`, `get_pozo_repository`, `get_solicitud_pipa_repository`.
  Falta registrar `get_registro_combustible_repository`.
- **Routers a registrar en `app/api/v1/router.py`** al final del día (B2.1–B2.4): `pozos`,
  `combustible` y `reportes`.
- **Schemas base ya existentes:** `PagedResponse` y `MessageResponse` en `api/schemas/common.py`.
- Los usuarios se autentican con token (`admin`/`admin123`); todos los endpoints usan
  `Depends(get_current_user)`.

---

## TAREA B2.1 — CRUD de pozos (necesario para la tarea 3.9 del Miércoles)

Paso 1 — **Slug/Codificación de schemas** (`backend/app/api/schemas/pozos.py`):

```python
from datetime import datetime
from typing import Optional
from pydantic import BaseModel, Field

class PozoCreate(BaseModel):
    poz_dom_id: int = Field(..., ge=1)
    poz_nombrepozo: str = Field(..., min_length=1, max_length=100)
    poz_responsable: str = Field(..., min_length=1, max_length=100)
    poz_telefono: str = Field(..., min_length=1, max_length=30)
    poz_estatus: bool = Field(default=True)

class PozoUpdate(BaseModel):
    poz_dom_id: Optional[int] = Field(default=None, ge=1)
    poz_nombrepozo: Optional[str] = Field(default=None, min_length=1, max_length=100)
    poz_responsable: Optional[str] = Field(default=None, min_length=1, max_length=100)
    poz_telefono: Optional[str] = Field(default=None, min_length=1, max_length=30)
    poz_estatus: Optional[bool] = None

class PozoResponse(BaseModel):
    poz_id: int
    poz_dom_id: int
    poz_nombrepozo: str
    poz_responsable: str
    poz_telefono: str
    poz_fechaalta: datetime
    poz_estatus: bool
    model_config = {"from_attributes": True}
```

Paso 2 — **Repositorio:** `PozoRepository` YA existe pero la base rompe con `poz_id`. Agregar a
`backend/app/infrastructure/repositories/pozo_repository.py`:

```python
from sqlalchemy import select
from app.infrastructure.database.models import PozoModel

# dentro de PozoRepository:
async def get_by_id(self, record_id: int):
    result = await self.session.execute(
        select(PozoModel).where(PozoModel.poz_id == record_id)
    )
    return result.scalar_one_or_none()

async def delete(self, record_id: int) -> bool:
    inst = await self.get_by_id(record_id)
    if inst is None:
        return False
    await self.session.delete(inst)
    await self.session.flush()
    return True
```

Paso 3 — **Router** (`backend/app/api/v1/pozos.py`) siguiendo el patrón de `vehiculos.py`
(listado paginado con `PagedResponse`, `POST`, `GET/PUT/DELETE /{id}`, `_to_response`, manejo 404):

```python
"""
DAPAW2 Enterprise Platform - Pozos Router.
CRUD de pozos de abastecimiento (tabla 'pozos').
"""
from typing import List
from fastapi import APIRouter, Depends, HTTPException, Query, status

from app.api.deps import get_current_user, get_pozo_repository
from app.api.schemas.common import MessageResponse, PagedResponse
from app.api.schemas.pozos import PozoCreate, PozoResponse, PozoUpdate
from app.infrastructure.database.models import PozoModel
from app.infrastructure.repositories.pozo_repository import PozoRepository

router = APIRouter(prefix="/pozos", tags=["Pozos"])

def _to_response(m: PozoModel) -> PozoResponse:
    return PozoResponse.model_validate(m)

@router.get("", response_model=PagedResponse[PozoResponse])
async def list_pozos(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    estatus: bool | None = Query(None, description="Filtrar solo disponibles (poz_estatus=true)"),
    current_user=Depends(get_current_user),
    repo: PozoRepository = Depends(get_pozo_repository),
):
    filtros = {}
    if estatus is not None:
        filtros["poz_estatus"] = estatus
    skip = (page - 1) * page_size
    total = await repo.count(filtros)
    rows = await repo.get_all(skip=skip, limit=page_size, filters=filtros)
    return PagedResponse(
        items=[_to_response(r) for r in rows],
        total=total,
        page=page,
        page_size=page_size,
        pages=(total + page_size - 1) // page_size,
    )

@router.post("", response_model=PozoResponse, status_code=status.HTTP_201_CREATED)
async def create_pozo(
    body: PozoCreate,
    current_user=Depends(get_current_user),
    repo: PozoRepository = Depends(get_pozo_repository),
):
    creado = await repo.create(PozoModel(**body.model_dump()))
    return _to_response(creado)

@router.get("/{pozo_id}", response_model=PozoResponse)
async def get_pozo(
    pozo_id: int,
    current_user=Depends(get_current_user),
    repo: PozoRepository = Depends(get_pozo_repository),
):
    pozo = await repo.get_by_id(pozo_id)
    if pozo is None:
        raise HTTPException(status_code=404, detail="Pozo no encontrado")
    return _to_response(pozo)

@router.put("/{pozo_id}", response_model=PozoResponse)
async def update_pozo(
    pozo_id: int,
    body: PozoUpdate,
    current_user=Depends(get_current_user),
    repo: PozoRepository = Depends(get_pozo_repository),
):
    pozo = await repo.get_by_id(pozo_id)
    if pozo is None:
        raise HTTPException(status_code=404, detail="Pozo no encontrado")
    for key, value in body.model_dump(exclude_unset=True).items():
        setattr(pozo, key, value)
    await repo.update(pozo)
    return _to_response(pozo)

@router.delete("/{pozo_id}", response_model=MessageResponse)
async def delete_pozo(
    pozo_id: int,
    current_user=Depends(get_current_user),
    repo: PozoRepository = Depends(get_pozo_repository),
):
    ok = await repo.delete(pozo_id)
    if not ok:
        raise HTTPException(status_code=404, detail="Pozo no encontrado")
    return MessageResponse(message="Pozo eliminado")
```

Paso 4 — Registrar en `backend/app/api/v1/router.py` (ver tarea B2.4 que agrupa los tres).

**DoD B2.1:** `POST /pozos` crea un pozo, `GET /pozos?estatus=true` filtra disponibles (esto
conecta con `listar_activos_con_coordenadas` que usa la asignación automática).

---

## TAREA B2.2 — Registro e historial de combustible (necesario para 3.10)

Paso 1 — **Repositorio** `backend/app/infrastructure/repositories/registro_combustible_repository.py`:

```python
from typing import List, Optional
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from app.infrastructure.database.models import RegistroCombustibleModel
from app.infrastructure.repositories.base import GenericRepository

class RegistroCombustibleRepository(GenericRepository[RegistroCombustibleModel]):
    """Repositorio de registros de gasto de combustible por vehiculo."""

    def __init__(self, session: AsyncSession) -> None:
        super().__init__(RegistroCombustibleModel, session)

    async def get_by_id(self, record_id: int) -> Optional[RegistroCombustibleModel]:
        result = await self.session.execute(
            select(RegistroCombustibleModel).where(
                RegistroCombustibleModel.reg_com_id == record_id
            )
        )
        return result.scalar_one_or_none()

    async def listar(self, vhe_id: Optional[int] = None,
                     skip: int = 0, limit: int = 100) -> List[RegistroCombustibleModel]:
        query = select(RegistroCombustibleModel)
        if vhe_id is not None:
            query = query.where(RegistroCombustibleModel.reg_com_vhe_id == vhe_id)
        query = query.order_by(
            RegistroCombustibleModel.reg_com_fecha.desc()
        ).offset(skip).limit(limit)
        result = await self.session.execute(query)
        return list(result.scalars().all())

    async def count(self, vhe_id: Optional[int] = None) -> int:
        from sqlalchemy import func
        query = select(func.count()).select_from(RegistroCombustibleModel)
        if vhe_id is not None:
            query = query.where(RegistroCombustibleModel.reg_com_vhe_id == vhe_id)
        result = await self.session.execute(query)
        return result.scalar_one()
```

Paso 2 — **Schemas** `backend/app/api/schemas/combustible.py`:

```python
from datetime import datetime
from pydantic import BaseModel, Field

class CombustibleCreate(BaseModel):
    reg_com_vhe_id: int = Field(..., ge=1)
    reg_com_lts: float = Field(..., ge=0)
    reg_com_cost: float = Field(..., ge=0)
    reg_com_km_inicio: int = Field(..., ge=0)
    reg_com_km_final: int = Field(..., ge=0)

class CombustibleCreateResponse(BaseModel):
    reg_com_id: int
    reg_com_vhe_id: int
    reg_com_lts: float
    reg_com_cost: float
    reg_com_fecha: datetime
    reg_com_km_inicio: int
    reg_com_km_final: int
    model_config = {"from_attributes": True}
```

Paso 3 — **Dependency** en `app/api/deps.py` (patrón de `get_vehiculo_repository`):

```python
def get_registro_combustible_repository(
    session: AsyncSession = Depends(get_session),
) -> RegistroCombustibleRepository:
    return RegistroCombustibleRepository(session)
```

Paso 4 — **Router** `backend/app/api/v1/combustible.py`:

```python
"""
DAPAW2 Enterprise Platform - Combustible Router.
Registro de gasto de combustible por vehiculo (tabla 'registros_combustible').
"""
from fastapi import APIRouter, Depends, HTTPException, Query, status

from app.api.deps import (
    get_current_user,
    get_registro_combustible_repository,
    get_vehiculo_repository,
)
from app.api.schemas.combustible import CombustibleCreate, CombustibleCreateResponse
from app.api.schemas.common import PagedResponse
from app.infrastructure.database.models import RegistroCombustibleModel
from app.infrastructure.repositories.registro_combustible_repository import (
    RegistroCombustibleRepository,
)
from app.infrastructure.repositories.vehiculo_repository import VehiculoRepository

router = APIRouter(prefix="/combustible", tags=["Combustible"])

def _to_response(m: RegistroCombustibleModel) -> CombustibleCreateResponse:
    return CombustibleCreateResponse.model_validate(m)

@router.post("", response_model=CombustibleCreateResponse, status_code=status.HTTP_201_CREATED)
async def registrar_combustible(
    body: CombustibleCreate,
    current_user=Depends(get_current_user),
    repo: RegistroCombustibleRepository = Depends(get_registro_combustible_repository),
    vehiculo_repo: VehiculoRepository = Depends(get_vehiculo_repository),
):
    vehiculo = await vehiculo_repo.get_by_id(body.reg_com_vhe_id)
    if vehiculo is None:
        raise HTTPException(status_code=400, detail="El vehículo no existe")
    registro = await repo.create(RegistroCombustibleModel(**body.model_dump()))
    return _to_response(registro)

@router.get("", response_model=PagedResponse[CombustibleCreateResponse])
async def listar_combustible(
    vehiculo_id: int | None = Query(None),
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    current_user=Depends(get_current_user),
    repo: RegistroCombustibleRepository = Depends(get_registro_combustible_repository),
):
    skip = (page - 1) * page_size
    total = await repo.count(vehiculo_id)
    rows = await repo.listar(vehiculo_id, skip=skip, limit=page_size)
    return PagedResponse(
        items=[_to_response(r) for r in rows],
        total=total,
        page=page,
        page_size=page_size,
        pages=(total + page_size - 1) // page_size,
    )
```

**DoD B2.2:** `POST /combustible` valida el vehículo (400 si no existe) y guarda; `GET /combustible`
devuelve historial paginado filtrable por `vehiculo_id`, más reciente primero.

---

## TAREA B2.3 — Reporte de combustible (tarea 3.11 del Jueves, adelantada al backend)

`GET /reportes/combustible` con filtros `fecha_desde`, `fecha_hasta` y `vehiculo_id` devuelve
**totales por día** (litros, costo y registros). Se agrega con SQL sobre `registros_combustible`.

Paso 1 — **Schemas** (agregar a `backend/app/api/schemas/reportes.py`):

```python
from datetime import date, datetime
from typing import Optional
from pydantic import BaseModel

class ReporteCombustibleRow(BaseModel):
    fecha: date
    vhe_id: Optional[int] = None
    total_lts: float = 0
    total_cost: float = 0
    num_registros: int = 0

class ReporteCombustibleResponse(BaseModel):
    desde: Optional[date] = None
    hasta: Optional[date] = None
    total_lts: float = 0
    total_cost: float = 0
    filas: list[ReporteCombustibleRow] = []

class ReporteTiemposTrasladoRow(BaseModel):
    fecha: date
    entregas: int = 0
    duracion_promedio_min: Optional[float] = None
    km_recorridos: float = 0
    ubicaciones_reportadas: int = 0

class ReporteTiemposTrasladoResponse(BaseModel):
    desde: Optional[date] = None
    hasta: Optional[date] = None
    filas: list[ReporteTiemposTrasladoRow] = []
```

Paso 2 — **Query** (solo lectura, se ejecuta directo sobre la sesión). En
`backend/app/api/v1/reportes.py`:

```python
from datetime import date, timedelta
from fastapi import APIRouter, Depends, Query
from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession

from app.api.deps import get_current_user, get_session
from app.api.schemas.reportes import (
    ReporteCombustibleResponse,
    ReporteCombustibleRow,
    ReporteTiemposTrasladoResponse,
    ReporteTiemposTrasladoRow,
)
from app.infrastructure.database.models import (
    RegistroCombustibleModel,
    RutaSolicitudModel,
    SolicitudPipaModel,
    UbicacionPipaModel,
)

router = APIRouter(prefix="/reportes", tags=["Reportes"])

@router.get("/combustible", response_model=ReporteCombustibleResponse)
async def reporte_combustible(
    fecha_desde: date | None = Query(None),
    fecha_hasta: date | None = Query(None),
    vehiculo_id: int | None = Query(None),
    db: AsyncSession = Depends(get_session),
    current_user=Depends(get_current_user),
):
    """Totales de combustible por día (litros, costo y # de registros)."""
    condiciones = []
    if fecha_desde:
        condiciones.append(RegistroCombustibleModel.reg_com_fecha >= fecha_desde)
    if fecha_hasta:
        # comparar contra el día siguiente para incluir todo el día (Postgres):
        condiciones.append(RegistroCombustibleModel.reg_com_fecha < fecha_hasta + timedelta(days=1))
    if vehiculo_id:
        condiciones.append(RegistroCombustibleModel.reg_com_vhe_id == vehiculo_id)

    fecha_col = func.date(RegistroCombustibleModel.reg_com_fecha)  # sin fuse de string labels
    query = (
        select(
            fecha_col.label("fecha"),
            RegistroCombustibleModel.reg_com_vhe_id.label("vhe_id"),
            func.coalesce(func.sum(RegistroCombustibleModel.reg_com_lts), 0).label("lts"),
            func.coalesce(func.sum(RegistroCombustibleModel.reg_com_cost), 0).label("cost"),
            func.count().label("n"),
        )
        .where(*condiciones)
        .group_by(fecha_col, RegistroCombustibleModel.reg_com_vhe_id)
        .order_by(fecha_col)
    )
    result = await db.execute(query)
    filas = [
        ReporteCombustibleRow(
            fecha=row.fecha,
            vhe_id=row.vhe_id,
            total_lts=float(row.lts),
            total_cost=float(row.cost),
            num_registros=row.n,
        )
        for row in result.all()
    ]
    return ReporteCombustibleResponse(
        desde=fecha_desde,
        hasta=fecha_hasta,
        total_lts=round(sum(f.total_lts for f in filas), 2),
        total_cost=round(sum(f.total_cost for f in filas), 2),
        filas=filas,
    )
```

> **Ojo (evita estos bugs):** comparar `reg_com_fecha < fecha_hasta` sin sumar un día EXCLUYE los
> registros del mismo día `fecha_hasta`. También evita `group_by("fecha")`/`order_by("fecha")` con
> string labels si tu driver los rechaza; usa la columna `func.date(...)` (como arriba).

**DoD B2.3:** el reporte agrupa **por día** los litros/costo, respeta `fecha_desde`/`fecha_hasta`/
`vehiculo_id` y suma totales.

---

## TAREA B2.4 — Reporte de tiempos de traslado (tarea 3.12 del Jueves, adelantada)

`GET /reportes/tiempos_traslado` calcula por día: **entregas** (de `solicitudpipas` en estado
`Entregada`), **duración promedio** y **km recorridos** (de `rutas_solicitud`). `ubicaciones_pipa`
se usa para totalizar ubicaciones GPS reportadas por día (la sprint lo menciona como fuente).

Paso 1 — En `backend/app/api/v1/reportes.py`, agregar (código completo y funcional):

```python
@router.get("/tiempos_traslado", response_model=ReporteTiemposTrasladoResponse)
async def reporte_tiempos_traslado(
    fecha_desde: date | None = Query(None),
    fecha_hasta: date | None = Query(None),
    db: AsyncSession = Depends(get_session),
    current_user=Depends(get_current_user),
):
    """Por día: entregas (solicitudpipas), duración/km (rutas_solicitud) y GPS (ubicacionespipa)."""
    desde = fecha_desde or date(1970, 1, 1)
    hasta = (fecha_hasta or date(2100, 1, 1)) + timedelta(days=1)  # incluye todo el día final

    # 1) Entregas por día
    fecha_entregas = func.date(SolicitudPipaModel.spp_horaentrega)
    q_entregas = (
        select(
            fecha_entregas.label("fecha"),
            func.count().label("entregas"),
        )
        .where(SolicitudPipaModel.spp_estatus == "Entregada")
        .where(SolicitudPipaModel.spp_horaentrega >= desde)
        .where(SolicitudPipaModel.spp_horaentrega < hasta)
        .group_by(fecha_entregas)
        .order_by(fecha_entregas)
    )

    # 2) Duración promedio y km por día (rutas_solicitud)
    fecha_rutas = func.date(RutaSolicitudModel.rtsol_creado)
    q_rutas = (
        select(
            fecha_rutas.label("fecha"),
            func.avg(RutaSolicitudModel.rtsol_duracion_min).label("dur"),
            func.coalesce(func.sum(RutaSolicitudModel.rtsol_distancia_km), 0).label("km"),
        )
        .where(RutaSolicitudModel.rtsol_creado >= desde)
        .where(RutaSolicitudModel.rtsol_creado < hasta)
        .group_by(fecha_rutas)
        .order_by(fecha_rutas)
    )

    # 3) Ubicaciones GPS reportadas por día (ubicacionespipa)
    fecha_ubp = func.date(UbicacionPipaModel.ubp_timestamp)
    q_ubicacion = (
        select(
            fecha_ubp.label("fecha"),
            func.count().label("n"),
        )
        .where(UbicacionPipaModel.ubp_timestamp >= desde)
        .where(UbicacionPipaModel.ubp_timestamp < hasta)
        .group_by(fecha_ubp)
    )

    # Fusionar todo por fecha en un solo dict
    por_dia: dict[date, ReporteTiemposTrasladoRow] = {}
    for r in (await db.execute(q_rutas)).all():
        fila = por_dia.setdefault(r.fecha, ReporteTiemposTrasladoRow(fecha=r.fecha))
        fila.duracion_promedio_min = float(r.dur) if r.dur is not None else None
        fila.km_recorridos = float(r.km) or 0
    for r in (await db.execute(q_entregas)).all():
        por_dia.setdefault(r.fecha, ReporteTiemposTrasladoRow(fecha=r.fecha)).entregas = r.entregas
    for r in (await db.execute(q_ubicacion)).all():
        por_dia.setdefault(r.fecha, ReporteTiemposTrasladoRow(fecha=r.fecha)).ubicaciones_reportadas = r.n

    return ReporteTiemposTrasladoResponse(
        desde=fecha_desde,
        hasta=fecha_hasta,
        filas=[por_dia[f] for f in sorted(por_dia)],
    )
```

> **Ojo (evita estos bugs):** el `spp_estatus == "Entregada"` usa el string exacto del enum.
> Las comparaciones de fecha usan `>= desde` y `< hasta+1día`; si no sumas el día, excluyes las
> entregas del último día filtrado. Las 3 queries se fusionan por fecha con `setdefault` (un día
> puede tener entregas SIN ruta y viceversa).

Paso 2 — Registrar TODOS los routers nuevos (pozos, combustible, reportes) en `router.py`:

```python
from app.api.v1.pozos import router as pozos_router
from app.api.v1.combustible import router as combustible_router
from app.api.v1.reportes import router as reportes_router
# ... al final, junto a los demás include_router:
api_v1_router.include_router(pozos_router)
api_v1_router.include_router(combustible_router)
api_v1_router.include_router(reportes_router)
```

**DoD B2.4:** el reporte entrega por día: `entregas`, `duracion_promedio_min`, `km_recorridos` y
`ubicaciones_reportadas`.

---

## Verificación del día (TODO EN DOCKER)

```bash
# Levantar/levantar servicios (backend cambió)
docker compose up -d --build

# Backend sin regresiones + nuevos tests gruesos
docker compose exec backend python -m pytest -q

# Crear datos de prueba (pozos, vehículo con ubicación, solicitud Entregada)
docker compose exec db psql -U postgres -d dapatlqdb -c \
  "INSERT INTO pozos (poz_dom_id, poz_nombrepozo, poz_responsable, poz_telefono, poz_estatus) VALUES (1, 'Pozo Norte', 'Operador', '3312345678', true);"
docker compose exec db psql -U postgres -d dapatlqdb -c \
  "INSERT INTO registros_combustible (reg_com_vhe_id, reg_com_lts, reg_com_cost, reg_com_km_inicio, reg_com_km_final) VALUES (1, 200, 5200, 1000, 1360);"
```

Chequeos con token (login `admin`/`admin123` para obtener `access_token`):
1. `POST /pozos` → 201; `GET /pozos?estatus=true` → aparece el pozo creado; `PUT/DELETE` funcionan.
2. `POST /combustible` con vehículo inexistente → 400; con vehículo válido → 201;
   `GET /combustible?vehiculo_id=1` → historial ordenado.
3. `GET /reportes/combustible` → filas por día con `total_lts`/`total_cost`/`num_registros`;
   probar con `fecha_desde`/`fecha_hasta` y `vehiculo_id`.
4. `GET /reportes/tiempos_traslado` → filas por día con `entregas`, `duracion_promedio_min`,
   `km_recorridos` y `ubicaciones_reportadas` (requiere solicitudes `Entregada` y filas en
   `rutas_solicitud`).
5. `/docs` muestra los tags `Pozos`, `Combustible` y `Reportes` sin regresiones a `/docs` anterior.

## DoD del día + commit

- [ ] B2.1 CRUD de pozos (`GET/POST/PUT/DELETE /pozos`) listo en `/docs`.
- [ ] B2.2 `POST /combustible` y `GET /combustible` operando con historial paginado.
- [ ] B2.3 `GET /reportes/combustible` con totales por día y filtros.
- [ ] B2.4 `GET /reportes/tiempos_traslado` calculado desde `rutas_solicitud` y `ubicaciones_pipa`.
- [ ] Dynamic `deps.py` con `get_registro_combustible_repository`.
- [ ] `router.py` registra `pozos` + `combustible` + `reportes`.
- [ ] `pytest` del backend en verde (sin regresiones).

Commit sugerido:

```bash
git add -A
git commit -m "feat(pipas): backend complementario - CRUD pozos, combustible y reportes (B2.1-B2.4)"
git push origin feature/pipas-backend
```

---

## Nota de coordinación con Miércoles y Jueves

- Las tareas 3.9 y 3.10 del **Miércoles** son ahora **100% frontend** (backend listo hoy).
- Las tareas 3.11 y 3.12 del **Jueves** son ahora **100% frontend** de reportes contra estos
  endpoints (solo falta consumo + exportación, que son las tareas 3.13–3.15).
- Si al terminar el día sobra tiempo: adelantar un test de BACKEND de reportes/pozos/combustible
  en `backend/tests/` (fixtures `client`, `auth_headers`, `admin_headers` ya existen en
  `conftest.py`).