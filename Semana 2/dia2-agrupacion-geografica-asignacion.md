# HOJA DE INSTRUCCIONES — MODULO DE PIPAS — SPRINT 2 — DIA 2 (MARTES)

**Autor:** Turno (Lider de turno)
**Fecha:** 2026-08-11
**Repositorio:** dapa2w — rama `feature/s2-semana2-pipas`
**Objetivo del dia:** Reglas de agrupacion geografica, disponibilidad de vehiculos,
asignacion automatica por ruta mas corta (OSRM), sugerencia de pozo de abastecimiento
y endpoints de asignaciones.

---

## Tareas del dia

| # | Tarea | Archivo(s) principal(es) | DoD |
|---|-------|--------------------------|-----|
| 2.6 | Reglas de agrupacion geografica (radio configurable) | `app/domain/pipas/agrupacion.py` | Funcion pura probada con casos de agrupacion |
| 2.7 | Regla de disponibilidad de vehiculos (activo + sin solicitud activa) | `vehiculo_repository.py`, `app/domain/pipas/disponibilidad.py` | Query optimizada (EXISTS) y prueba unitaria |
| 2.8 | Asignacion automatica: vehiculo disponible + ruta mas corta (OSRM) | `app/domain/pipas/asignacion.py`, `osrm/proveedor_rutas.py` | Pruebas con 1 vehiculo y N vehiculos |
| 2.9 | Sugerencia de pozo de abastecimiento mas cercano | `app/domain/pipas/pozos.py`, `pozo_repository.py` | Funcion pura probada |
| 2.10 | Endpoints de asignaciones | `schemas/asignaciones.py`, `routers/asignaciones.py` | POST + GET en `/docs` con JWT |

**DoD del dia:** 4 funciones puras probadas (2.6, 2.8, 2.9 + disponibilidad 2.7), query de
disponibilidad optimizada con EXISTS, y 2 endpoints operativos (POST/GET) autenticados con JWT.

---

## 0. Prerequisitos y convenciones

- Continuar en la rama del Sprint 2. El codigo del Lunes (endpoints de solicitudes, historial,
  rutas por OSRM, ubicaciones) ya debe estar MERGEADO.
- Ciertos dias previos ya dejaron listo: `OsrmClient` (`app/infrastructure/osrm/osrm_client.py`),
  `RutaSolicitudRepository` y el modelo `RutaSolicitudModel`, `UbicacionPipaModel` y
  `VehiculoRepository.get_disponibles()` (que HOY se ajusta a la regla 2.7).
- Estados activos de una solicitud (no debe cambiarlos): `Pendiente` y `En Ruta`
  (ver `app/shared/enums.py` / `SolicitudEstado`).
- Regla de oro de este Sprint: **funcion pura en `app/domain/pipas/`, sin SQL ni I/O**.
  Todo acceso a datos vive en `app/infrastructure/repositories/`; el endpoint solo orquesta.
- Respuestas paginadas con `PagedResponse` de `app/api/schemas/common.py` (patron del Lunes).
- Autenticacion: `Depends(get_current_user)` (igual que `routers/solicitudes_pipa.py`).
- Migraciones de `db/migrations/` numeradas `1000XX`. Todas las sentencias con
  `IF NOT EXISTS` para que sean idempotentes.
- No olvides registrar cada router nuevo en `app/api/router.py`.

> **HABILITADOR TECNICO (revisar en equipo):** para alimentar 2.6/2.8/2.9 con datos reales y
> para los marcadores del mapa del JUEVES (2.19), las solicitudes necesitan coordenadas.
> Hoy agregamos `spp_latitud` / `spp_longitud` a `solicitudpipas` (migracion 100057) y una tabla
> `asignaciones` (migracion 100058). Ambas se senalan al inicio del dia para que el equipo las
> valide; sin ellas las DoD de 2.6 y 2.10 no son demostrables de punta a punta.

---

## A. HABILITADORES TECNICOS (antes de las tareas)

### A.1 Migracion `db/migrations/100057_alter_solicitudpipas_coordenadas.sql`

```sql
-- 100057_alter_solicitudpipas_coordenadas.sql
-- Coordenadas (lat/lng) del domicilio capturadas al alta de la solicitud.
-- Autor: Turno - Fecha: 2026-08-11
ALTER TABLE solicitudpipas
    ADD COLUMN IF NOT EXISTS spp_latitud  NUMERIC(10, 7),
    ADD COLUMN IF NOT EXISTS spp_longitud NUMERIC(10, 7);

COMMENT ON COLUMN solicitudpipas.spp_latitud  IS 'Latitud del domicilio al momento del alta.';
COMMENT ON COLUMN solicitudpipas.spp_longitud IS 'Longitud del domicilio al momento del alta.';
```

### A.2 Migracion `db/migrations/100058_crea_tabla_asignaciones.sql`

```sql
-- 100058_crea_tabla_asignaciones.sql
-- Registro de asignaciones automaticas (vehiculo + ruta + pozo sugerido).
-- Autor: Turno - Fecha: 2026-08-11
CREATE TABLE IF NOT EXISTS asignaciones (
    asg_id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    asg_spp_id        BIGINT NOT NULL,
    asg_vhe_id        BIGINT NOT NULL,
    asg_rtsol_id      BIGINT,
    asg_pzo_id        BIGINT,
    asg_distance_km   NUMERIC(10, 3),
    asg_duration_min  NUMERIC(10, 2),
    asg_geometry      JSONB,
    asg_estatus       VARCHAR(20) NOT NULL DEFAULT 'Asignada',
    asg_fecha         TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

ALTER TABLE asignaciones
    ADD CONSTRAINT fk_asg_spp FOREIGN KEY (asg_spp_id)
        REFERENCES solicitudpipas(spp_id) ON DELETE CASCADE,
    ADD CONSTRAINT fk_asg_vhe FOREIGN KEY (asg_vhe_id)
        REFERENCES vehiculos(vhe_id),
    ADD CONSTRAINT fk_asg_pzo FOREIGN KEY (asg_pzo_id)
        REFERENCES pozos(poz_id);

CREATE INDEX IF NOT EXISTS ix_asignaciones_spp_id ON asignaciones (asg_spp_id);
CREATE INDEX IF NOT EXISTS ix_asignaciones_vhe_id ON asignaciones (asg_vhe_id);
CREATE INDEX IF NOT EXISTS ix_asignaciones_fecha  ON asignaciones (asg_fecha);
```

### A.3 Modelos nuevos en `app/infrastructure/database/models.py`

Agregar al final del archivo (verificar que `Numeric`, `JSONB` y `DomicilioModel`
ya esten importados; `RutaSolicitudModel` ya usa `JSONB`):

```python
class PozoModel(Base):
    __tablename__ = "pozos"

    poz_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    poz_dom_id: Mapped[int] = mapped_column(BigInteger, nullable=False)
    poz_nombrepozo: Mapped[str] = mapped_column(String(100), nullable=False)
    poz_responsable: Mapped[str] = mapped_column(String(100), nullable=False)
    poz_telefono: Mapped[str] = mapped_column(String(30), nullable=False)
    poz_fechaalta: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )
    poz_estatus: Mapped[bool] = mapped_column(Boolean, default=True, nullable=False)


class DomicilioModel(Base):
    __tablename__ = "domicilio"

    dom_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    dom_calle: Mapped[str] = mapped_column(String(150), nullable=False)
    dom_numero: Mapped[Optional[str]] = mapped_column(String(30), nullable=True)
    dom_colonia: Mapped[Optional[str]] = mapped_column(String(100), nullable=True)
    dom_codigopostal: Mapped[Optional[str]] = mapped_column(String(5), nullable=True)
    dom_municipio: Mapped[Optional[str]] = mapped_column(String(100), nullable=True)
    dom_estado: Mapped[Optional[str]] = mapped_column(String(100), nullable=True)
    dom_pais: Mapped[str] = mapped_column(String(50), default="MEXICO", nullable=False)
    dom_latitud: Mapped[Optional[Decimal]] = mapped_column(Numeric(10, 7), nullable=True)
    dom_longitud: Mapped[Optional[Decimal]] = mapped_column(Numeric(10, 7), nullable=True)
    dom_estatus: Mapped[bool] = mapped_column(Boolean, default=True, nullable=False)


class AsignacionModel(Base):
    __tablename__ = "asignaciones"

    asg_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    asg_spp_id: Mapped[int] = mapped_column(
        BigInteger, ForeignKey("solicitudpipas.spp_id", ondelete="CASCADE"), nullable=False
    )
    asg_vhe_id: Mapped[int] = mapped_column(
        BigInteger, ForeignKey("vehiculos.vhe_id"), nullable=False
    )
    asg_rtsol_id: Mapped[Optional[int]] = mapped_column(BigInteger, nullable=True)
    asg_pzo_id: Mapped[Optional[int]] = mapped_column(BigInteger, nullable=True)
    asg_distance_km: Mapped[Optional[Decimal]] = mapped_column(Numeric(10, 3), nullable=True)
    asg_duration_min: Mapped[Optional[Decimal]] = mapped_column(Numeric(10, 2), nullable=True)
    asg_geometry: Mapped[Optional[dict]] = mapped_column(JSONB, nullable=True)
    asg_estatus: Mapped[str] = mapped_column(
        String(20), nullable=False, default="Asignada"
    )
    asg_fecha: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )

    __table_args__ = (
        Index("ix_asignaciones_spp_id", "asg_spp_id"),
        Index("ix_asignaciones_vhe_id", "asg_vhe_id"),
    )
```

Tambien agregar a `SolicitudPipaModel`:

```python
spp_latitud: Mapped[Optional[Decimal]] = mapped_column(Numeric(10, 7), nullable=True)
spp_longitud: Mapped[Optional[Decimal]] = mapped_column(Numeric(10, 7), nullable=True)
```

> Registra los modelos nuevos en `env.py` de Alembic (lista `target_metadata`) si ahi
> se importa el metadata de `models`.

### A.4 Nuevos repositorios

`app/infrastructure/repositories/pozo_repository.py`:

```python
from typing import List

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.domain.pipas.value_objects import PozoConCoordenadas
from app.infrastructure.database.models import DomicilioModel, PozoModel
from app.infrastructure.repositories.base import GenericRepository


class PozoRepository(GenericRepository[PozoModel]):
    """Repositorio de pozos de abastecimiento."""

    def __init__(self, session: AsyncSession) -> None:
        super().__init__(PozoModel, session)

    async def listar_activos_con_coordenadas(self) -> List[PozoConCoordenadas]:
        """Pozos activos con lat/lng de su domicilio (para 2.9)."""
        result = await self.session.execute(
            select(
                PozoModel.poz_id,
                PozoModel.poz_nombrepozo,
                DomicilioModel.dom_latitud,
                DomicilioModel.dom_longitud,
            )
            .join(DomicilioModel, DomicilioModel.dom_id == PozoModel.poz_dom_id)
            .where(PozoModel.poz_estatus == True)  # noqa: E712
            .where(
                DomicilioModel.dom_latitud.is_not(None),
                DomicilioModel.dom_longitud.is_not(None),
            )
        )
        return [
            PozoConCoordenadas(
                poz_id=row[0],
                poz_nombrepozo=row[1],
                latitud=float(row[2]),
                longitud=float(row[3]),
            )
            for row in result.all()
        ]
```

`app/infrastructure/repositories/ubicacion_pipa_repository.py`:

```python
from typing import Optional

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.infrastructure.database.models import UbicacionPipaModel
from app.infrastructure.repositories.base import GenericRepository


class UbicacionPipaRepository(GenericRepository[UbicacionPipaModel]):
    """Repositorio de ubicaciones GPS de pipas."""

    def __init__(self, session: AsyncSession) -> None:
        super().__init__(UbicacionPipaModel, session)

    async def get_ultima_ubicacion(
        self, vhe_id: int
    ) -> Optional[UbicacionPipaModel]:
        """Ultima ubicacion registrada de un vehiculo (ubp_timestamp mas reciente)."""
        result = await self.session.execute(
            select(UbicacionPipaModel)
            .where(UbicacionPipaModel.ubp_vhe_id == vhe_id)
            .where(UbicacionPipaModel.ubp_estatus == True)  # noqa: E712
            .order_by(UbicacionPipaModel.ubp_timestamp.desc())
            .limit(1)
        )
        return result.scalar_one_or_none()
```

`app/infrastructure/repositories/asignacion_repository.py`:

```python
from datetime import datetime
from typing import List, Optional

from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession

from app.infrastructure.database.models import AsignacionModel
from app.infrastructure.repositories.base import GenericRepository


class AsignacionRepository(GenericRepository[AsignacionModel]):
    """Repositorio de asignaciones."""

    def __init__(self, session: AsyncSession) -> None:
        super().__init__(AsignacionModel, session)

    async def get_by_id(self, record_id: int) -> Optional[AsignacionModel]:
        result = await self.session.execute(
            select(AsignacionModel).where(AsignacionModel.asg_id == record_id)
        )
        return result.scalar_one_or_none()

    async def get_by_solicitud(self, spp_id: int) -> Optional[AsignacionModel]:
        result = await self.session.execute(
            select(AsignacionModel)
            .where(AsignacionModel.asg_spp_id == spp_id)
            .order_by(AsignacionModel.asg_fecha.desc())
            .limit(1)
        )
        return result.scalar_one_or_none()

    async def listar_paginado(
        self,
        skip: int = 0,
        limit: int = 100,
        fecha_desde: Optional[datetime] = None,
        fecha_hasta: Optional[datetime] = None,
    ) -> List[AsignacionModel]:
        query = select(AsignacionModel)
        if fecha_desde:
            query = query.where(AsignacionModel.asg_fecha >= fecha_desde)
        if fecha_hasta:
            query = query.where(AsignacionModel.asg_fecha <= fecha_hasta)
        query = (
            query.order_by(AsignacionModel.asg_fecha.desc())
            .offset(skip)
            .limit(limit)
        )
        result = await self.session.execute(query)
        return list(result.scalars().all())

    async def count(self, **filtros) -> int:
        query = select(func.count()).select_from(AsignacionModel)
        result = await self.session.execute(query)
        return result.scalar_one()
```

Actualizar `solicitud_pipa_repository.py` (agregar al final de la clase):

```python
    async def actualizar_coordenadas(
        self, spp_id: int, latitud: float, longitud: float
    ) -> Optional[SolicitudPipaModel]:
        """Persiste lat/lng de la solicitud (viene del domicilio al alta)."""
        solicitud = await self.get_by_id(spp_id)
        if solicitud is None:
            return None
        solicitud.spp_latitud = latitud
        solicitud.spp_longitud = longitud
        await self.session.flush()
        return solicitud

    async def listar_pendientes_con_coordenadas(self) -> List[SolicitudPipaModel]:
        """Solicitudes 'Pendiente' que ya tienen coordenadas (entrada de 2.6)."""
        result = await self.session.execute(
            select(SolicitudPipaModel)
            .where(SolicitudPipaModel.spp_estatus == "Pendiente")
            .where(
                SolicitudPipaModel.spp_latitud.is_not(None),
                SolicitudPipaModel.spp_longitud.is_not(None),
            )
            .order_by(SolicitudPipaModel.spp_fechasolicitud.desc())
        )
        return list(result.scalars().all())
```

### A.5 Dependencias en `app/api/deps.py`

Agregar (mismo patron que los existentes):

```python
def get_asignacion_repository(
    session: AsyncSession = Depends(get_session),
) -> AsignacionRepository:
    return AsignacionRepository(session)


def get_pozo_repository(
    session: AsyncSession = Depends(get_session),
) -> PozoRepository:
    return PozoRepository(session)


def get_ubicacion_pipa_repository(
    session: AsyncSession = Depends(get_session),
) -> UbicacionPipaRepository:
    return UbicacionPipaRepository(session)
```

---

## TAREA 2.6 — Reglas de agrupacion geografica

**Regla de negocio:** agrupar solicitudes pendientes por cercania usando el **radio configurable**
en metros; el centroide del grupo no debe quedar a mas de `radio_m` de cada miembro. Es una
**funcion pura** (no SQL, no I/O).

### Crear `app/domain/pipas/__init__.py`

```python
"""Dominio del modulo de pipas (funciones puras y value objects)."""
```

### Crear `app/domain/pipas/geo.py`

```python
"""Utilidades geograficas puras (haversine)."""
from math import asin, cos, radians, sin, sqrt

RADIO_TIERRA_M = 6_371_000.0


def haversine_m(lat1: float, lng1: float, lat2: float, lng2: float) -> float:
    """Distancia en metros entre dos coordenadas (formula de haversine)."""
    phi1, phi2 = radians(lat1), radians(lat2)
    dphi = radians(lat2 - lat1)
    dlmb = radians(lng2 - lng1)
    a = sin(dphi / 2) ** 2 + cos(phi1) * cos(phi2) * sin(dlmb / 2) ** 2
    return 2 * RADIO_TIERRA_M * asin(sqrt(a))
```

### Crear `app/domain/pipas/value_objects.py`

```python
"""Value objects del dominio de pipas (inmutables, sin SQL)."""
from dataclasses import dataclass, field
from typing import Dict, List, Optional


@dataclass(frozen=True)
class Punto:
    latitud: float
    longitud: float


@dataclass(frozen=True)
class SolicitudPunto:
    """Solicitud pendiente con coordenadas (entrada de la agrupacion)."""
    spp_id: int
    latitud: float
    longitud: float


@dataclass(frozen=True)
class GrupoSolicitudes:
    """Grupo geografico con su centroide y los ids de sus solicitudes."""
    centroide: Punto
    spp_ids: List[int]


@dataclass(frozen=True)
class VehiculoDisponible:
    vhe_id: int
    latitud: float
    longitud: float


@dataclass(frozen=True)
class OpcionRuta:
    vhe_id: int
    distance_km: float
    duration_min: float
    geometry: Optional[dict] = None


@dataclass(frozen=True)
class PozoConCoordenadas:
    poz_id: int
    poz_nombrepozo: str
    latitud: float
    longitud: float
```

### Crear `app/domain/pipas/agrupacion.py`

```python
"""Agrupacion geografica de solicitudes pendientes (tarea 2.6)."""
from typing import List, Optional

from app.domain.pipas.geo import haversine_m
from app.domain.pipas.value_objects import GrupoSolicitudes, Punto, SolicitudPunto


def agrupar_por_proximidad(
    solicitudes: List[SolicitudPunto],
    radio_m: float = 1000.0,
) -> List[GrupoSolicitudes]:
    """Agrupa solicitudes pendientes por cercania (radio configurable en metros).

    - El radio se interpreta entre el centroide del grupo y cada miembro.
    - Si una solicitud cae fuera de todos los grupos existentes, inicia uno nuevo.
    """
    grupos: List[List[SolicitudPunto]] = []
    for solicitud in solicitudes:
        if not grupos:
            grupos.append([solicitud])
            continue
        mejor_grupo: Optional[int] = None
        mejor_dist = float("inf")
        for i, grupo in enumerate(grupos):
            centroide = _centroide(grupo)
            dist = haversine_m(
                centroide.latitud,
                centroide.longitud,
                solicitud.latitud,
                solicitud.longitud,
            )
            if dist <= radio_m and dist < mejor_dist:
                mejor_grupo, mejor_dist = i, dist
        if mejor_grupo is None:
            grupos.append([solicitud])
        else:
            grupos[mejor_grupo].append(solicitud)
    return [
        GrupoSolicitudes(
            centroide=_centroide(grupo),
            spp_ids=[s.spp_id for s in grupo],
        )
        for grupo in grupos
    ]


def _centroide(grupo: List[SolicitudPunto]) -> Punto:
    n = len(grupo)
    return Punto(
        latitud=sum(s.latitud for s in grupo) / n,
        longitud=sum(s.longitud for s in grupo) / n,
    )
```

### DoD 2.6 — Pruebas `tests/test_agrupacion_geografica.py`

```python
from app.domain.pipas.agrupacion import agrupar_por_proximidad
from app.domain.pipas.value_objects import SolicitudPunto

# Tlaquepaque, Jalisco (20.64, -103.31). ~111 m por 0.001 grados de latitud.
TLAQUEPAQUE = (20.64, -103.31)


def _sol(spp_id: int, lat: float, lng: float) -> SolicitudPunto:
    return SolicitudPunto(spp_id=spp_id, latitud=lat, longitud=lng)


def test_solicitudes_cercanas_forman_un_solo_grupo():
    lat, lng = TLAQUEPAQUE
    solicitudes = [
        _sol(1, lat, lng),
        _sol(2, lat + 0.002, lng),        # ~222 m al norte
        _sol(3, lat, lng + 0.002),        # ~180 m al este
    ]
    grupos = agrupar_por_proximidad(solicitudes, radio_m=500.0)
    assert len(grupos) == 1
    assert sorted(grupos[0].spp_ids) == [1, 2, 3]


def test_solicitudes_lejanas_generan_grupos_separados():
    lat, lng = TLAQUEPAQUE
    solicitudes = [
        _sol(1, lat, lng),
        _sol(2, lat + 0.05, lng),  # ~5.5 km al norte
    ]
    grupos = agrupar_por_proximidad(solicitudes, radio_m=1000.0)
    assert len(grupos) == 2


def test_radio_configurable_agrupa_o_separa():
    lat, lng = TLAQUEPAQUE
    solicitudes = [
        _sol(1, lat, lng),
        _sol(2, lat + 0.008, lng),  # ~0.9 km al norte
    ]
    assert len(agrupar_por_proximidad(solicitudes, radio_m=1000.0)) == 1
    assert len(agrupar_por_proximidad(solicitudes, radio_m=500.0)) == 2


def test_entrada_vacia_devuelve_sin_grupos():
    assert agrupar_por_proximidad([]) == []
```

---

## TAREA 2.7 — Disponibilidad de vehiculos

**Regla de negocio:** un vehiculo esta **disponible** si y solo si:
`vhe_estatus = TRUE` **y** NO tiene ninguna solicitud con estado `Pendiente` o `En Ruta`.
Debe ser una **query optimizada** (subconsulta `NOT EXISTS`), no un loop en Python.

### Crear `app/domain/pipas/disponibilidad.py`

```python
"""Regla de disponibilidad de vehiculos (tarea 2.7)."""
ESTADOS_ACTIVOS = ("Pendiente", "En Ruta")


def tiene_solicitud_activa(solicitudes_activas: int) -> bool:
    """Un vehiculo esta ocupado si tiene al menos una solicitud activa."""
    return solicitudes_activas > 0


def es_disponible(vhe_estatus: bool, solicitudes_activas: int) -> bool:
    """Disponible = activo (TRUE) y sin solicitud Pendiente/En Ruta asignada."""
    return vhe_estatus and not tiene_solicitud_activa(solicitudes_activas)
```

### Ajustar `vehiculo_repository.py` (regla 2.7)

Reemplazar `get_disponibles()` por:

```python
from sqlalchemy import exists, select

    async def get_disponibles(self) -> List[VehiculoModel]:
        """Vehiculos activos SIN solicitud activa asignada (regla 2.7)."""
        result = await self.session.execute(
            select(VehiculoModel)
            .where(VehiculoModel.vhe_estatus == True)  # noqa: E712
            .where(
                ~exists().where(
                    SolicitudPipaModel.spp_vhe_id == VehiculoModel.vhe_id,
                    SolicitudPipaModel.spp_estatus.in_(("Pendiente", "En Ruta")),
                )
            )
            .order_by(VehiculoModel.vhe_id.asc())
        )
        return list(result.scalars().all())
```

Agregar el import `SolicitudPipaModel` en el mismo archivo. Reutiliza los indices
ya existentes `ix_solicitudpipas_vhe_id` / `ix_solicitudpipas_estatus`.

### DoD 2.7 — Pruebas `tests/test_disponibilidad.py`

```python
from app.domain.pipas.disponibilidad import es_disponible


def test_activo_sin_solicitudes_es_disponible():
    assert es_disponible(vhe_estatus=True, solicitudes_activas=0) is True


def test_inactivo_nunca_es_disponible():
    assert es_disponible(vhe_estatus=False, solicitudes_activas=0) is False


def test_con_solicitud_activa_no_es_disponible():
    assert es_disponible(vhe_estatus=True, solicitudes_activas=1) is False
```

---

## TAREA 2.8 — Asignacion automatica (ruta mas corta via OSRM)

**Regla de negocio:** de los vehiculos disponibles, elegir el que tenga la **ruta mas corta**
(menor `distance_km` de OSRM) hacia las coordenadas de la solicitud. Se abstrae la llamada a
OSRM detras de un `Protocol` para poder probar sin red.

### Crear `app/domain/pipas/asignacion.py`

```python
"""Asignacion automatica de vehiculo por ruta mas corta (tarea 2.8)."""
from typing import List, Optional, Protocol, Tuple

from app.domain.pipas.value_objects import OpcionRuta, VehiculoDisponible


class ProveedorRutas(Protocol):
    """Abstraccion del motor de rutas (OSRM) para poder testear sin red."""

    async def calcular_ruta(
        self,
        origen_lng: float,
        origen_lat: float,
        destino_lng: float,
        destino_lat: float,
    ) -> Tuple[float, float, Optional[dict]]:
        """Retorna (distance_km, duration_min, geometry_geojson)."""
        ...


class AsignadorAutomatico:
    """Selecciona el vehiculo disponible con la ruta mas corta a la solicitud."""

    def __init__(self, proveedor_rutas: ProveedorRutas) -> None:
        self._rutas = proveedor_rutas

    async def asignar(
        self,
        latitud: float,
        longitud: float,
        vehiculos: List[VehiculoDisponible],
    ) -> Optional[OpcionRuta]:
        mejor: Optional[OpcionRuta] = None
        for v in vehiculos:
            distance_km, duration_min, geometry = await self._rutas.calcular_ruta(
                v.longitud, v.latitud, longitud, latitud
            )
            opcion = OpcionRuta(
                vhe_id=v.vhe_id,
                distance_km=distance_km,
                duration_min=duration_min,
                geometry=geometry,
            )
            if mejor is None or opcion.distance_km < mejor.distance_km:
                mejor = opcion
        return mejor
```

### Crear `app/infrastructure/osrm/proveedor_rutas.py`

```python
"""Adaptador de OsrmClient al Protocol ProveedorRutas del dominio."""
from typing import Optional, Tuple

from app.domain.pipas.asignacion import ProveedorRutas
from app.infrastructure.osrm.osrm_client import OsrmClient


class OsrmProveedorRutas:
    """Implementa ProveedorRutas usando el OsrmClient existente."""

    def __init__(self, cliente: Optional[OsrmClient] = None) -> None:
        self._cliente = cliente or OsrmClient()

    async def calcular_ruta(
        self,
        origen_lng: float,
        origen_lat: float,
        destino_lng: float,
        destino_lat: float,
    ) -> Tuple[float, float, Optional[dict]]:
        data = await self._cliente.get_route(
            [(origen_lng, origen_lat), (destino_lng, destino_lat)]
        )
        return (
            float(data["distance_km"]),
            float(data["duration_min"]),
            data.get("geometry"),
        )
```

### DoD 2.8 — Pruebas `tests/test_asignacion.py`

```python
from typing import Optional, Tuple

import pytest

from app.domain.pipas.asignacion import AsignadorAutomatico, ProveedorRutas
from app.domain.pipas.value_objects import VehiculoDisponible


class RutaFija:
    """Fake de ProveedorRutas: responde un (distance_km, duration_min, geometry) por vehiculo."""

    def __init__(self, respuestas: dict) -> None:
        self._respuestas = respuestas
        self.llamadas: list = []

    async def calcular_ruta(
        self,
        origen_lng: float,
        origen_lat: float,
        destino_lng: float,
        destino_lat: float,
    ) -> Tuple[float, float, Optional[dict]]:
        self.llamadas.append((origen_lng, origen_lat, destino_lng, destino_lat))
        return self._respuestas[len(self.llamadas) - 1]


@pytest.mark.asyncio
async def test_un_vehiculo_disponible_es_asignado():
    ruta = RutaFija({0: (12.5, 18.0, {"type": "LineString"})})
    asignador = AsignadorAutomatico(ruta)
    vehiculos = [VehiculoDisponible(vhe_id=7, latitud=20.64, longitud=-103.31)]
    mejor = await asignador.asignar(20.65, -103.31, vehiculos)
    assert mejor is not None
    assert mejor.vhe_id == 7
    assert mejor.distance_km == 12.5


@pytest.mark.asyncio
async def test_se_elige_la_ruta_mas_corta_entre_varios_vehiculos():
    # vehiculo 7 -> 30 km ; vehiculo 9 -> 8 km ; vehiculo 3 -> 15 km
    ruta = RutaFija({0: (30.0, 40.0, None), 1: (8.0, 12.0, None), 2: (15.0, 20.0, None)})
    asignador = AsignadorAutomatico(ruta)
    vehiculos = [
        VehiculoDisponible(vhe_id=7, latitud=20.60, longitud=-103.30),
        VehiculoDisponible(vhe_id=9, latitud=20.63, longitud=-103.32),
        VehiculoDisponible(vhe_id=3, latitud=20.70, longitud=-103.20),
    ]
    mejor = await asignador.asignar(20.64, -103.31, vehiculos)
    assert mejor is not None
    assert mejor.vhe_id == 9
    assert mejor.distance_km == 8.0


@pytest.mark.asyncio
async def test_sin_vehiculos_no_hay_asignacion():
    asignador = AsignadorAutomatico(RutaFija({}))
    assert await asignador.asignar(20.64, -103.31, []) is None
```

---

## TAREA 2.9 — Sugerencia de pozo de abastecimiento

**Regla de negocio:** de los pozos activos, sugerir el **mas cercano** a las coordenadas de la
solicitud (o de la ruta). Funcion pura.

### Crear `app/domain/pipas/pozos.py`

```python
"""Sugerencia de pozo de abastecimiento mas cercano (tarea 2.9)."""
from typing import List, Optional

from app.domain.pipas.geo import haversine_m
from app.domain.pipas.value_objects import PozoConCoordenadas


def sugerir_pozo_mas_cercano(
    pozos: List[PozoConCoordenadas],
    latitud: float,
    longitud: float,
) -> Optional[PozoConCoordenadas]:
    """Pozo activo mas cercano al punto dado (None si no hay pozos)."""
    if not pozos:
        return None
    return min(
        pozos,
        key=lambda p: haversine_m(p.latitud, p.longitud, latitud, longitud),
    )
```

### DoD 2.9 — Pruebas (agregar a `tests/test_asignacion.py`)

```python
from app.domain.pipas.pozos import sugerir_pozo_mas_cercano
from app.domain.pipas.value_objects import PozoConCoordenadas


def test_sugiere_pozo_mas_cercano():
    pozos = [
        PozoConCoordenadas(poz_id=1, poz_nombrepozo="Pozo Norte", latitud=20.66, longitud=-103.31),
        PozoConCoordenadas(poz_id=2, poz_nombrepozo="Pozo Centro", latitud=20.641, longitud=-103.311),
    ]
    sugerido = sugerir_pozo_mas_cercano(pozos, latitud=20.64, longitud=-103.31)
    assert sugerido is not None
    assert sugerido.poz_id == 2


def test_sin_pozos_devuelve_none():
    assert sugerir_pozo_mas_cercano([], latitud=20.64, longitud=-103.31) is None
```

---

## TAREA 2.10 — Endpoints de asignaciones

### Crear `app/api/schemas/asignaciones.py`

```python
"""
DAPAW2 Enterprise Platform - Asignaciones API Schemas
"""
from datetime import datetime
from typing import Optional

from pydantic import BaseModel, Field


class AsignacionAutomaticaRequest(BaseModel):
    """Solicitud de asignacion automatica (por id de solicitud)."""
    spp_id: int = Field(..., ge=1, description="ID de la solicitud de pipa")


class PozoSugerido(BaseModel):
    poz_id: int
    poz_nombrepozo: str


class AsignacionResponse(BaseModel):
    """Respuesta de una asignacion (vehiculo + ruta + pozo sugerido)."""
    asg_id: int
    spp_id: int
    vhe_id: int
    rtsol_id: Optional[int] = None
    pozo_sugerido: Optional[PozoSugerido] = None
    distance_km: Optional[float] = None
    duration_min: Optional[float] = None
    geometry: Optional[dict] = None
    asg_estatus: str
    asg_fecha: datetime

    model_config = {"from_attributes": True}
```

> Si prefieres paginar el listado con `PagedResponse`, importalo de
> `app/api/schemas/common.py` (patron del endpoint de solicitudes del Lunes).

### Crear `app/api/routers/asignaciones.py`

```python
"""
DAPAW2 Enterprise Platform - Asignaciones de pipas (2.10)
"""
from typing import Optional

from fastapi import APIRouter, Depends, HTTPException, Query, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.api.deps import (
    get_asignacion_repository,
    get_db,
    get_current_user,
    get_pozo_repository,
    get_solicitud_pipa_repository,
    get_ubicacion_pipa_repository,
    get_vehiculo_repository,
)
from app.api.schemas.asignaciones import (
    AsignacionAutomaticaRequest,
    AsignacionResponse,
    PozoSugerido,
)
from app.domain.pipas.asignacion import AsignadorAutomatico
from app.domain.pipas.pozos import sugerir_pozo_mas_cercano
from app.domain.pipas.value_objects import PozoConCoordenadas, VehiculoDisponible
from app.infrastructure.database.models import (
    AsignacionModel,
    RutaSolicitudModel,
    UsuarioModel,
)
from app.infrastructure.osrm.proveedor_rutas import OsrmProveedorRutas
from app.infrastructure.repositories.asignacion_repository import AsignacionRepository
from app.infrastructure.repositories.pozo_repository import PozoRepository
from app.infrastructure.repositories.ruta_solicitud_repository import RutaSolicitudRepository
from app.infrastructure.repositories.solicitud_pipa_repository import SolicitudPipaRepository
from app.infrastructure.repositories.ubicacion_pipa_repository import UbicacionPipaRepository
from app.infrastructure.repositories.vehiculo_repository import VehiculoRepository

router = APIRouter(prefix="/asignaciones", tags=["asignaciones"])

# Centro de Tlaquepaque (fallback cuando el vehiculo no tiene ubicacion GPS registrada).
BASE_LAT, BASE_LNG = 20.6400, -103.3110


def _serializar(asignacion: AsignacionModel) -> AsignacionResponse:
    return AsignacionResponse(
        asg_id=asignacion.asg_id,
        spp_id=asignacion.asg_spp_id,
        vhe_id=asignacion.asg_vhe_id,
        rtsol_id=asignacion.asg_rtsol_id,
        pozo_sugerido=(
            PozoSugerido(poz_id=asignacion.asg_pzo_id, poz_nombrepozo="")
            if asignacion.asg_pzo_id
            else None
        ),
        distance_km=float(asignacion.asg_distance_km) if asignacion.asg_distance_km else None,
        duration_min=float(asignacion.asg_duration_min) if asignacion.asg_duration_min else None,
        geometry=asignacion.asg_geometry,
        asg_estatus=asignacion.asg_estatus,
        asg_fecha=asignacion.asg_fecha,
    )


@router.post(
    "/automatica",
    response_model=AsignacionResponse,
    status_code=status.HTTP_201_CREATED,
    summary="Asignacion automatica: vehiculo disponible con ruta mas corta",
)
async def asignacion_automatica(
    payload: AsignacionAutomaticaRequest,
    db: AsyncSession = Depends(get_db),
    current_user: UsuarioModel = Depends(get_current_user),
) -> AsignacionResponse:
    solicitudes = SolicitudPipaRepository(db)
    solicitud = await solicitudes.get_by_id(payload.spp_id)
    if solicitud is None:
        raise HTTPException(status_code=404, detail="Solicitud no encontrada.")
    if solicitud.spp_estatus != "Pendiente":
        raise HTTPException(
            status_code=409,
            detail="Solo se asignan solicitudes en estado 'Pendiente'.",
        )
    if solicitud.spp_latitud is None or solicitud.spp_longitud is None:
        raise HTTPException(
            status_code=400,
            detail="La solicitud no tiene coordenadas. Actualicelas antes de asignar.",
        )

    vehiculos = await VehiculoRepository(db).get_disponibles()
    if not vehiculos:
        raise HTTPException(status_code=409, detail="No hay vehiculos disponibles.")

    ubicaciones = UbicacionPipaRepository(db)
    disponibles: list[VehiculoDisponible] = []
    for v in vehiculos:
        ultima = await ubicaciones.get_ultima_ubicacion(v.vhe_id)
        if ultima is not None:
            disponibles.append(
                VehiculoDisponible(
                    vhe_id=v.vhe_id,
                    latitud=ultima.ubp_latitud,
                    longitud=ultima.ubp_longitud,
                )
            )
        else:
            disponibles.append(
                VehiculoDisponible(vhe_id=v.vhe_id, latitud=BASE_LAT, longitud=BASE_LNG)
            )

    lat, lng = float(solicitud.spp_latitud), float(solicitud.spp_longitud)
    asignador = AsignadorAutomatico(OsrmProveedorRutas())
    mejor = await asignador.asignar(lat, lng, disponibles)
    if mejor is None:
        raise HTTPException(status_code=409, detail="No se pudo calcular ninguna ruta.")

    pozo = sugerir_pozo_mas_cercano(
        await PozoRepository(db).listar_activos_con_coordenadas(), lat, lng
    )

    solicitud.spp_vhe_id = mejor.vhe_id
    ruta = RutaSolicitudModel(
        rtsol_spp_id=payload.spp_id,
        rtsol_vhe_id=mejor.vhe_id,
        rtsol_origen_lat=lat,
        rtsol_origen_lng=lng,
        rtsol_destino_lat=lat,
        rtsol_destino_lng=lng,
        rtsol_distancia_km=mejor.distance_km,
        rtsol_duracion_min=mejor.duration_min,
        rtsol_geometria=mejor.geometry,
    )
    ruta_creada = await RutaSolicitudRepository(db).create(ruta)

    asignacion = AsignacionModel(
        asg_spp_id=payload.spp_id,
        asg_vhe_id=mejor.vhe_id,
        asg_rtsol_id=ruta_creada.rtsol_id,
        asg_pzo_id=pozo.poz_id if pozo else None,
        asg_distance_km=mejor.distance_km,
        asg_duration_min=mejor.duration_min,
        asg_geometry=mejor.geometry,
    )
    db.add(asignacion)
    await db.commit()
    await db.refresh(asignacion)

    return _serializar(asignacion)


@router.get(
    "",
    response_model=list[AsignacionResponse],
    summary="Listado de asignaciones",
)
async def listar_asignaciones(
    limit: int = Query(default=100, ge=1, le=500),
    offset: int = Query(default=0, ge=0),
    db: AsyncSession = Depends(get_db),
    current_user: UsuarioModel = Depends(get_current_user),
) -> list[AsignacionResponse]:
    repositorio = AsignacionRepository(db)
    registros = await repositorio.listar_paginado(skip=offset, limit=limit)
    return [_serializar(a) for a in registros]
```

### Registrar en `app/api/router.py`

```python
from app.api.routers import asignaciones

api_router.include_router(asignaciones.router)
```

### DoD 2.10 — Verificacion manual (con JWT)

```bash
# 1) obtener token
TOKEN=$(curl -s -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"..."}' | python -c "import sys,json;print(json.load(sys.stdin)['access_token'])")

# 2) asignacion automatica
curl -s -X POST http://localhost:8000/asignaciones/automatica \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"spp_id": 1}'

# 3) listado de asignaciones
curl -s http://localhost:8000/asignaciones?limit=10 \
  -H "Authorization: Bearer $TOKEN"
```

Respuesta esperada del POST (ejemplo):

```json
{
  "asg_id": 1,
  "spp_id": 1,
  "vhe_id": 4,
  "rtsol_id": 2,
  "pozo_sugerido": { "poz_id": 2, "poz_nombrepozo": "Pozo Centro" },
  "distance_km": 1.25,
  "duration_min": 4.1,
  "geometry": { "type": "LineString", "coordinates": [...] },
  "asg_estatus": "Asignada",
  "asg_fecha": "2026-08-11T09:00:00Z"
}
```

---

## Verificacion final del dia

```bash
cd backend
python -m pytest tests/test_agrupacion_geografica.py tests/test_disponibilidad.py tests/test_asignacion.py -q
# (todos en verde)

# aplica migraciones y confirma que las tablas existen
cd ..
# (ejecutar el script habitual del proyecto para aplicar db/migrations)
```

Chequeos extra:
1. `/docs` muestra `POST /asignaciones/automatica` y `GET /asignaciones`.
2. El POST con una solicitud sin coordenadas responde 400; con una ya entregada/cancelada, 409.
3. Con un vehiculo con solicitud `Pendiente` o `En Ruta`, `get_disponibles()` ya NO lo regresa.
4. La asignacion persiste en `asignaciones` con su `rtsol_id` y, si el pozo tiene coordenadas,
   `asg_pzo_id` poblado.

## DoD del dia + commit

- [ ] 2.6 funcion pura `agrupar_por_proximidad` + 4 pruebas en verde.
- [ ] 2.7 regla `es_disponible` + `get_disponibles()` con NOT EXISTS + pruebas.
- [ ] 2.8 `AsignadorAutomatico` con `ProveedorRutas` (Protocol) + pruebas 1/N vehiculos.
- [ ] 2.9 `sugerir_pozo_mas_cercano` + pruebas.
- [ ] 2.10 POST/GET de asignaciones autenticados y verificados en `/docs`.
- [ ] Migraciones 100057 y 100058 aplicadas y modelos registrados.

Commit sugerido:

```bash
git add -A
git commit -m "feat(pipas): agrupacion geografica, disponibilidad, asignacion automatica y endpoints de asignaciones (2.6-2.10)"
git push origin feature/s2-semana2-pipas
```
