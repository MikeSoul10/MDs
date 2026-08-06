# Día 4 — Endpoints V1 (API)

## 0. Prerrequisitos (verificar antes de empezar)

- [ ] Día 3 terminado: los schemas de `vehiculos`, `vehiculomarcas`, `solicitudes_pipa` importan sin error.
- [ ] Rama de trabajo creada: `git checkout -b feature/pipas-endpoints` (desde `feature/pipas-semana1`).

**Convenciones del proyecto (respétalas):**
- Router por módulo en `backend/app/api/v1/`, con `router = APIRouter(prefix=..., tags=[...])`.
- Protección JWT: `current_user=Depends(get_current_user)` de `app/api/deps.py`.
- Paginación: patrón de `roles.py` usando `PagedResponse[...]` y `Query(page, page_size)`.
- La sesión `get_db_session` ya hace `commit` al final de la request → **no se llama `db.commit()` manualmente** en endpoints con repositorios.
- Respuestas de error: `HTTPException(status_code=404, detail=...)` (estilo `colonias.py`).
- Permisos en BD con códigos `recurso:accion` (convención existente `auth:login`, `user:read`).

**⚠️ Gotcha del código existente:**
`GenericRepository.exists()` usa `self.model.id` y los modelos de pipas **no tienen columna `id`**. No uses `exists()` para pipas; agrega métodos dedicados al repositorio:
- `VehiculoRepository.get_by_descripcion(desc)` (para validar unicidad de `vhe_descripcion`).
- `MarcaVehiculoRepository.get_by_nombre(nombre)` ya existe (día 2).

---

## 1. Wiring — agregar dependencias de repositorios en `app/api/deps.py`

Agrega los imports:

```python
from app.infrastructure.repositories.marca_vehiculo_repository import MarcaVehiculoRepository
from app.infrastructure.repositories.solicitud_pipa_repository import SolicitudPipaRepository
from app.infrastructure.repositories.vehiculo_repository import VehiculoRepository
```

Y las funciones (junto a las demás de "Repository Dependencies"):

```python
def get_vehiculo_repository(
    session: AsyncSession = Depends(get_session),
) -> VehiculoRepository:
    """Provide a VehiculoRepository instance."""
    return VehiculoRepository(session)


def get_marca_vehiculo_repository(
    session: AsyncSession = Depends(get_session),
) -> MarcaVehiculoRepository:
    """Provide a MarcaVehiculoRepository instance."""
    return MarcaVehiculoRepository(session)


def get_solicitud_pipa_repository(
    session: AsyncSession = Depends(get_session),
) -> SolicitudPipaRepository:
    """Provide a SolicitudPipaRepository instance."""
    return SolicitudPipaRepository(session)
```

**Agrega a `VehiculoRepository`** (`vehiculo_repository.py`) el método para la unicidad:

```python
    async def get_by_descripcion(self, descripcion: str) -> Optional[VehiculoModel]:
        result = await self.session.execute(
            select(VehiculoModel).where(VehiculoModel.vhe_descripcion == descripcion)
        )
        return result.scalar_one_or_none()
```

---

## 2. Tarea 4.1 — `backend/app/api/v1/vehiculos.py`

**Definición de Terminado:** CRUD (`GET/POST/PUT/PATCH/DELETE`) + `GET /vehiculos/disponibles`, protegido por JWT, listado paginado.

```python
"""
DAPA2W Enterprise Platform - Vehiculos Router.

CRUD de vehiculos (pipas) protegido por JWT.
"""

from typing import List, Optional

from fastapi import APIRouter, Depends, HTTPException, Query, status

from app.api.deps import get_current_user, get_vehiculo_repository
from app.api.schemas.common import MessageResponse, PagedResponse
from app.api.schemas.vehiculos import (
    VehiculoCreate,
    VehiculoResponse,
    VehiculoUpdate,
)
from app.infrastructure.database.models import VehiculoModel
from app.infrastructure.repositories.vehiculo_repository import VehiculoRepository

router = APIRouter(prefix="/vehiculos", tags=["Vehiculos"])


def _to_response(m: VehiculoModel) -> VehiculoResponse:
    return VehiculoResponse.model_validate(m)


def _paginar(
    repo: VehiculoRepository, page: int, page_size: int, rows: List[VehiculoModel]
) -> PagedResponse[VehiculoResponse]:
    total = len(rows)  # ajustar si se filtra por BD
    return PagedResponse(
        items=[_to_response(r) for r in rows],
        total=total,
        page=page,
        page_size=page_size,
        pages=(total + page_size - 1) // page_size,
    )


@router.get("", response_model=PagedResponse[VehiculoResponse])
async def list_vehiculos(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    current_user=Depends(get_current_user),
    repo: VehiculoRepository = Depends(get_vehiculo_repository),
):
    """Listado paginado de vehiculos."""
    skip = (page - 1) * page_size
    total = await repo.count()
    rows = await repo.get_all(skip=skip, limit=page_size)
    return PagedResponse(
        items=[_to_response(r) for r in rows],
        total=total,
        page=page,
        page_size=page_size,
        pages=(total + page_size - 1) // page_size,
    )


@router.get("/disponibles", response_model=List[VehiculoResponse])
async def get_disponibles(
    current_user=Depends(get_current_user),
    repo: VehiculoRepository = Depends(get_vehiculo_repository),
):
    """Vehiculos activos disponibles para asignar."""
    return [_to_response(v) for v in await repo.get_disponibles()]


@router.get("/{vehiculo_id}", response_model=VehiculoResponse)
async def get_vehiculo(
    vehiculo_id: int,
    current_user=Depends(get_current_user),
    repo: VehiculoRepository = Depends(get_vehiculo_repository),
):
    vehiculo = await repo.get_by_id(vehiculo_id)
    if vehiculo is None:
        raise HTTPException(status_code=404, detail="Vehiculo no encontrado")
    return _to_response(vehiculo)


@router.post("", response_model=VehiculoResponse, status_code=status.HTTP_201_CREATED)
async def create_vehiculo(
    body: VehiculoCreate,
    current_user=Depends(get_current_user),
    repo: VehiculoRepository = Depends(get_vehiculo_repository),
):
    """Alta de vehiculo. La descripcion debe ser unica."""
    if await repo.get_by_descripcion(body.vhe_descripcion):
        raise HTTPException(status_code=409, detail="La descripcion ya existe")
    creado = await repo.create(VehiculoModel(**body.model_dump()))
    return _to_response(creado)


@router.put("/{vehiculo_id}", response_model=VehiculoResponse)
async def update_vehiculo(
    vehiculo_id: int,
    body: VehiculoUpdate,
    current_user=Depends(get_current_user),
    repo: VehiculoRepository = Depends(get_vehiculo_repository),
):
    vehiculo = await repo.get_by_id(vehiculo_id)
    if vehiculo is None:
        raise HTTPException(status_code=404, detail="Vehiculo no encontrado")
    data = body.model_dump(exclude_unset=True)
    if "vhe_descripcion" in data and await repo.get_by_descripcion(data["vhe_descripcion"]):
        if vehiculo.vhe_descripcion != data["vhe_descripcion"]:
            raise HTTPException(status_code=409, detail="La descripcion ya existe")
    for key, value in data.items():
        setattr(vehiculo, key, value)
    await repo.update(vehiculo)
    return _to_response(vehiculo)


@router.patch("/{vehiculo_id}", response_model=VehiculoResponse)
async def patch_vehiculo(
    vehiculo_id: int,
    body: VehiculoUpdate,
    current_user=Depends(get_current_user),
    repo: VehiculoRepository = Depends(get_vehiculo_repository),
):
    """Actualizacion parcial (mismos datos que PUT)."""
    return await update_vehiculo(vehiculo_id, body, current_user, repo)


@router.delete("/{vehiculo_id}", response_model=MessageResponse)
async def delete_vehiculo(
    vehiculo_id: int,
    current_user=Depends(get_current_user),
    repo: VehiculoRepository = Depends(get_vehiculo_repository),
):
    ok = await repo.delete(vehiculo_id)
    if not ok:
        raise HTTPException(status_code=404, detail="Vehiculo no encontrado")
    return MessageResponse(message="Vehiculo eliminado")
```

**Nota:** si el listado debe filtrarse, agrega los parámetros `Query` y filtra por repositorio (mismo enfoque que en 4.3). El `_paginar` de ejemplo es ilustrativo; usa el `PagedResponse` directo como en `list_vehiculos`.

---

## 3. Tarea 4.2 — `backend/app/api/v1/vehiculomarcas.py`

**Definición de Terminado:** CRUD del catálogo de marcas, protegido por JWT.

```python
"""
DAPA2W Enterprise Platform - Vehiculo Marcas Router.

CRUD del catalogo de marcas de vehiculo.
"""

from fastapi import APIRouter, Depends, HTTPException, status

from app.api.deps import get_current_user, get_marca_vehiculo_repository
from app.api.schemas.common import MessageResponse
from app.api.schemas.vehiculomarcas import (
    MarcaVehiculoCreate,
    MarcaVehiculoResponse,
    MarcaVehiculoUpdate,
)
from app.infrastructure.database.models import VehiculoMarcaModel
from app.infrastructure.repositories.marca_vehiculo_repository import MarcaVehiculoRepository

router = APIRouter(prefix="/vehiculomarcas", tags=["Vehiculo Marcas"])


def _to_response(m: VehiculoMarcaModel) -> MarcaVehiculoResponse:
    return MarcaVehiculoResponse.model_validate(m)


@router.get("", response_model=list[MarcaVehiculoResponse])
async def list_marcas(
    current_user=Depends(get_current_user),
    repo: MarcaVehiculoRepository = Depends(get_marca_vehiculo_repository),
):
    return [_to_response(m) for m in await repo.get_all()]


@router.post("", response_model=MarcaVehiculoResponse, status_code=status.HTTP_201_CREATED)
async def create_marca(
    body: MarcaVehiculoCreate,
    current_user=Depends(get_current_user),
    repo: MarcaVehiculoRepository = Depends(get_marca_vehiculo_repository),
):
    """El nombre de la marca debe ser unico."""
    if await repo.get_by_nombre(body.vho_nombremarca):
        raise HTTPException(status_code=409, detail="La marca ya existe")
    creada = await repo.create(VehiculoMarcaModel(**body.model_dump()))
    return _to_response(creada)


@router.get("/{marca_id}", response_model=MarcaVehiculoResponse)
async def get_marca(
    marca_id: int,
    current_user=Depends(get_current_user),
    repo: MarcaVehiculoRepository = Depends(get_marca_vehiculo_repository),
):
    marca = await repo.get_by_id(marca_id)
    if marca is None:
        raise HTTPException(status_code=404, detail="Marca no encontrada")
    return _to_response(marca)


@router.put("/{marca_id}", response_model=MarcaVehiculoResponse)
async def update_marca(
    marca_id: int,
    body: MarcaVehiculoUpdate,
    current_user=Depends(get_current_user),
    repo: MarcaVehiculoRepository = Depends(get_marca_vehiculo_repository),
):
    marca = await repo.get_by_id(marca_id)
    if marca is None:
        raise HTTPException(status_code=404, detail="Marca no encontrada")
    data = body.model_dump(exclude_unset=True)
    if "vho_nombremarca" in data and await repo.get_by_nombre(data["vho_nombremarca"]):
        if marca.vho_nombremarca != data["vho_nombremarca"]:
            raise HTTPException(status_code=409, detail="La marca ya existe")
    for key, value in data.items():
        setattr(marca, key, value)
    await repo.update(marca)
    return _to_response(marca)


@router.delete("/{marca_id}", response_model=MessageResponse)
async def delete_marca(
    marca_id: int,
    current_user=Depends(get_current_user),
    repo: MarcaVehiculoRepository = Depends(get_marca_vehiculo_repository),
):
    ok = await repo.delete(marca_id)
    if not ok:
        raise HTTPException(status_code=404, detail="Marca no encontrada")
    return MessageResponse(message="Marca eliminada")
```

---

## 4. Tarea 4.3 — `backend/app/api/v1/solicitudes_pipa.py`

**Definición de Terminado:** `POST` alta (liga vehículo), `GET` listado (filtros estado/fecha), `GET /{id}`, `PATCH /{id}/estado` (registra en `historial_solicitud`).

### 4.3.0 Métodos de filtrado en `SolicitudPipaRepository`

Agrega a `solicitud_pipa_repository.py`:

```python
    async def listar_filtrado(
        self,
        estado: Optional[str] = None,
        fecha_desde: Optional[datetime] = None,
        fecha_hasta: Optional[datetime] = None,
        skip: int = 0,
        limit: int = 100,
    ) -> List[SolicitudPipaModel]:
        query = select(SolicitudPipaModel)
        if estado:
            query = query.where(SolicitudPipaModel.spp_estatus == estado)
        if fecha_desde:
            query = query.where(SolicitudPipaModel.spp_fechasolicitud >= fecha_desde)
        if fecha_hasta:
            query = query.where(SolicitudPipaModel.spp_fechasolicitud <= fecha_hasta)
        query = (
            query.order_by(SolicitudPipaModel.spp_fechasolicitud.desc())
            .offset(skip)
            .limit(limit)
        )
        result = await self.session.execute(query)
        return list(result.scalars().all())

    async def count_filtrado(
        self,
        estado: Optional[str] = None,
        fecha_desde: Optional[datetime] = None,
        fecha_hasta: Optional[datetime] = None,
    ) -> int:
        from sqlalchemy import func

        query = select(func.count()).select_from(SolicitudPipaModel)
        if estado:
            query = query.where(SolicitudPipaModel.spp_estatus == estado)
        if fecha_desde:
            query = query.where(SolicitudPipaModel.spp_fechasolicitud >= fecha_desde)
        if fecha_hasta:
            query = query.where(SolicitudPipaModel.spp_fechasolicitud <= fecha_hasta)
        result = await self.session.execute(query)
        return result.scalar_one()
```

*(Agrega `from datetime import datetime` al inicio del archivo.)*

### 4.3.1 Router

```python
"""
DAPA2W Enterprise Platform - Solicitudes Pipa Router.

Alta de solicitudes ligadas a vehiculo, listado con filtros y
cambio de estado con bitacora en historial_solicitud.
"""

from datetime import datetime
from typing import Optional

from fastapi import APIRouter, Depends, HTTPException, Query, status

from app.api.deps import (
    get_current_user,
    get_solicitud_pipa_repository,
    get_vehiculo_repository,
)
from app.api.schemas.common import PagedResponse
from app.api.schemas.solicitudes_pipa import (
    SolicitudPipaCambioEstado,
    SolicitudPipaCreate,
    SolicitudPipaResponse,
    SolicitudPipaUpdate,
)
from app.infrastructure.database.models import SolicitudPipaModel
from app.infrastructure.repositories.solicitud_pipa_repository import SolicitudPipaRepository
from app.infrastructure.repositories.vehiculo_repository import VehiculoRepository
from app.shared.enums import SolicitudEstado

router = APIRouter(prefix="/solicitudes_pipa", tags=["Solicitudes Pipa"])


def _to_response(m: SolicitudPipaModel) -> SolicitudPipaResponse:
    return SolicitudPipaResponse.model_validate(m)


@router.get("", response_model=PagedResponse[SolicitudPipaResponse])
async def list_solicitudes(
    estado: Optional[str] = Query(None, description="Filtro por estado"),
    fecha_desde: Optional[datetime] = Query(None),
    fecha_hasta: Optional[datetime] = Query(None),
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    current_user=Depends(get_current_user),
    repo: SolicitudPipaRepository = Depends(get_solicitud_pipa_repository),
):
    """Listado paginado con filtros por estado y fecha."""
    skip = (page - 1) * page_size
    total = await repo.count_filtrado(estado, fecha_desde, fecha_hasta)
    rows = await repo.listar_filtrado(estado, fecha_desde, fecha_hasta, skip, page_size)
    return PagedResponse(
        items=[_to_response(r) for r in rows],
        total=total,
        page=page,
        page_size=page_size,
        pages=(total + page_size - 1) // page_size,
    )


@router.post("", response_model=SolicitudPipaResponse, status_code=status.HTTP_201_CREATED)
async def create_solicitud(
    body: SolicitudPipaCreate,
    current_user=Depends(get_current_user),
    repo: SolicitudPipaRepository = Depends(get_solicitud_pipa_repository),
    vehiculo_repo: VehiculoRepository = Depends(get_vehiculo_repository),
):
    """Alta de solicitud ligada a un vehiculo (spp_vhe_id requerido)."""
    vehiculo = await vehiculo_repo.get_by_id(body.spp_vhe_id)
    if vehiculo is None:
        raise HTTPException(status_code=400, detail="El vehiculo no existe")
    creada = await repo.create(SolicitudPipaModel(**body.model_dump()))
    return _to_response(creada)


@router.get("/{solicitud_id}", response_model=SolicitudPipaResponse)
async def get_solicitud(
    solicitud_id: int,
    current_user=Depends(get_current_user),
    repo: SolicitudPipaRepository = Depends(get_solicitud_pipa_repository),
):
    solicitud = await repo.get_by_id(solicitud_id)
    if solicitud is None:
        raise HTTPException(status_code=404, detail="Solicitud no encontrada")
    return _to_response(solicitud)


@router.patch("/{solicitud_id}/estado", response_model=SolicitudPipaResponse)
async def cambiar_estado(
    solicitud_id: int,
    body: SolicitudPipaCambioEstado,
    current_user=Depends(get_current_user),
    repo: SolicitudPipaRepository = Depends(get_solicitud_pipa_repository),
):
    """Cambio de estado registrado en historial_solicitud."""
    solicitud = await repo.get_by_id(solicitud_id)
    if solicitud is None:
        raise HTTPException(status_code=404, detail="Solicitud no encontrada")

    estado_anterior = solicitud.spp_estatus
    if estado_anterior == body.estado_nuevo:
        raise HTTPException(status_code=400, detail="La solicitud ya esta en ese estado")
    if not SolicitudEstado.es_valida(estado_anterior, body.estado_nuevo):
        raise HTTPException(
            status_code=400,
            detail=f"Transicion invalida: {estado_anterior} -> {body.estado_nuevo}",
        )

    await repo.registrar_transicion(
        solicitud_id=solicitud_id,
        estado_anterior=estado_anterior,
        estado_nuevo=body.estado_nuevo,
        usuario=current_user.username or "sistema",
        motivo=body.motivo,
    )
    return _to_response(solicitud)
```

*(Si se requiere `PATCH /{id}` para editar la solicitud, agregar usando `SolicitudPipaUpdate` con el mismo patrón `exclude_unset` de 4.1.)*

---

## 5. Tarea 4.4 — Registrar routers en `app/api/v1/router.py`

Agrega los imports:

```python
from app.api.v1.solicitudes_pipa import router as solicitudes_pipa_router
from app.api.v1.vehiculomarcas import router as vehiculomarcas_router
from app.api.v1.vehiculos import router as vehiculos_router
```

Y los `include_router` (al final, junto a los demás):

```python
api_v1_router.include_router(vehiculos_router)
api_v1_router.include_router(vehiculomarcas_router)
api_v1_router.include_router(solicitudes_pipa_router)
```

Las rutas quedan bajo `/api/v1/vehiculos`, `/api/v1/vehiculomarcas`, `/api/v1/solicitudes_pipa` en `/docs`.

---

## 6. Tarea 4.5 — Seed de permisos del módulo

Crear `db/seeds/00031_seed_permisos_pipas.sql` (siguiente número tras 00030). Usa la convención `recurso:accion` existente (`pipas.leer` pasa a ser `pipas:leer`):

```sql
-- 00031_seed_permisos_pipas.sql
-- Permisos del modulo de pipas asignables a roles desde el panel
-- Autor: Ing. Abel Alejandro Martinez Cervantes
-- Fecha: 2026-08-04

INSERT INTO permisos (permiso_nombre, permiso_codigo) VALUES
('Leer pipas', 'pipas:leer'),
('Escribir pipas', 'pipas:escribir'),
('Leer solicitudes', 'solicitudes:leer'),
('Gestionar solicitudes', 'solicitudes:gestionar');

-- Asignar los nuevos permisos al rol admin
INSERT INTO roles_permisos (rol_id, permiso_id)
SELECT r.rol_id, p.permiso_id
FROM roles r
CROSS JOIN permisos p
WHERE r.rol_nombre = 'admin'
  AND p.permiso_codigo IN ('pipas:leer', 'pipas:escribir', 'solicitudes:leer', 'solicitudes:gestionar')
  AND NOT EXISTS (
      SELECT 1 FROM roles_permisos rp
      WHERE rp.rol_id = r.rol_id AND rp.permiso_id = p.permiso_id
  );
```

> En BD nueva, el seed 00014 ya asigna todos los permisos al admin; el `NOT EXISTS` evita duplicados si se re-ejecuta. Los permisos quedan listos para asignarse a otros roles desde el panel.

---

## 7. Verificación final del día

```bash
# Levanta todo (aplica migraciones + seeds, incluye el 00031)
docker compose up -d --build

# 1. Las rutas cargan sin errores de importacion
docker compose exec backend python -c "import app.api.v1.vehiculos, app.api.v1.vehiculomarcas, app.api.v1.solicitudes_pipa"

# 2. Router registrado
docker compose exec backend python -c "from app.api.v1.router import api_v1_router; print([r.path for r in api_v1_router.routes if 'vehiculo' in r.path or 'solicitudes' in r.path])"

# 3. Tests existentes siguen en verde
docker compose exec backend pytest -q

# 4. En /docs (Authorize con token JWT): probar
#    GET /api/v1/vehiculos                  -> paginado (seed 00029)
#    GET /api/v1/vehiculos/disponibles      -> 5 vehiculos activos
#    POST /api/v1/solicitudes_pipa          -> alta ligada a vehiculo
#    PATCH /api/v1/solicitudes_pipa/1/estado -> Pendiente -> En Ruta, revisar historial_solicitud

# 5. Permisos en BD
docker compose exec db psql -U postgres -d dapatlqdb -c "SELECT permiso_codigo FROM permisos WHERE permiso_codigo LIKE 'pipas:%' OR permiso_codigo LIKE 'solicitudes:%';"
```

**Definición de Terminado del día:**
- [ ] `vehiculos.py`: `GET` paginado, `GET /disponibles`, `GET/POST/PUT/PATCH/DELETE /{id}` — todos con `get_current_user`.
- [ ] `vehiculomarcas.py`: CRUD completo protegido por JWT.
- [ ] `solicitudes_pipa.py`: `POST` (liga vehículo), `GET` con filtros estado/fecha, `GET /{id}`, `PATCH /{id}/estado`.
- [ ] Cambio de estado validado y registrado en `historial_solicitud`.
- [ ] `deps.py` con las 3 dependencias de repositorio.
- [ ] `router.py` con los 3 routers → rutas visibles en `/docs`.
- [ ] Seed `00031_seed_permisos_pipas.sql` aplicado y permisos asignables.
- [ ] `pytest` en verde.
- [ ] Commit: `feat: endpoints v1 del modulo de pipas`.

**Recomendación:** en `/docs` prueba primero `PATCH /{id}/estado` con una transición inválida (p. ej. `Pendiente -> Cancelada` es válida; `Entregada -> Pendiente` debe dar 400) para validar la lógica del día 3.
