# Día 2 — Modelos SQLAlchemy y Repositorios

## 0. Prerrequisitos (verificar antes de empezar)

- [ ] Día 1 terminado: migraciones `00053`, `00054`, `00055` aplicadas y seeds `00029`, `00030` corriendo sin errores.
- [ ] Docker arriba: `docker compose up db-migrate` terminó con `=== DB LISTO ===`.
- [ ] Rama de trabajo creada: `git checkout -b feature/pipas-models` (desde `feature/pipas-semana1`).

**⚠️ Decisiones ya tomadas (importantes):**
1. `solicitudpipas.spp_estatus` ahora es **`VARCHAR(20)`** con catálogo `Pendiente / En Ruta / Entregada / Cancelada` (lo cambió la migración 00053). **NO es booleano** → el modelo debe usar `String`, no `Boolean`.
2. `vehiculos.vhe_mar_id` apunta a **`fabricante.fab_id`**, no a `vehiculomarca`. No existe `FabricanteModel` → se debe crear para que la relación de marca funcione.

**Convenciones del proyecto (respétalas):**
- SQLAlchemy 2.x con `Mapped[...]` / `mapped_column`.
- Todos los modelos en un solo archivo: `backend/app/infrastructure/database/models.py`.
- Los `Base`/`AsyncSession` se toman de `app.infrastructure.database.session`.
- Repositorios extienden `GenericRepository` de `backend/app/infrastructure/repositories/base.py`.

---

## 1. Referencia rápida de esquemas (lo que debe mapear cada modelo)

| Tabla | PK | Columnas clave | FK |
|-------|----|----------------|----|
| `vehiculomarca` | `vho_id` | `vho_nombremarca` (100, NOT NULL), `vho_estatus` (bool, default true) | — |
| `fabricante` | `fab_id` | `fab_mcr` (NOT NULL), `fab_nombrefabricante` (100), `fab_urlfabricante` (120), `fab_estatus` | `fab_mcr → medidormarca.mrc_id` |
| `vehiculos` | `vhe_id` | `vhe_res_id` (int), `vhe_modelo` (int), `vhe_combustible` (int, default 0), `vhe_tipovehiculo` (bigint), `vhe_fecharegistro` (timestamp), `vhe_descripcion` (120, único), `vhe_estatus` | `vhe_mar_id → fabricante.fab_id` |
| `solicitudpipas` | `spp_id` | `spp_con` (bigint), `spp_srv` (bigint), `spp_fechasolicitud`, `spp_horaentrega`, `srv_licencia` (20), `spp_estatus` (**String 20**) | `spp_vhe_id → vehiculos.vhe_id` |
| `historial_solicitud` | `histsol_id` | `histsol_estado_anterior` (20, null), `histsol_estado_nuevo` (20), `histsol_fecha`, `histsol_usuario` (30), `histsol_motivo` (100) | `histsol_spp_id → solicitudpipas.spp_id` |
| `registros_combustible` | `reg_com_id` | `reg_com_lts` (numeric 10,2), `reg_com_cost` (numeric 12,2), `reg_com_fecha`, `reg_com_km_inicio` (int), `reg_com_km_final` (int) | `reg_com_vhe_id → vehiculos.vhe_id` |
| `ubicacionespipa` | `ubp_id` | `ubp_latitud`/`ubp_longitud` (float), `ubp_timestamp`, `ubp_estatus` | `ubp_vhe_id → vehiculos.vhe_id` |

---

## 2. Tarea 2.1 — `VehiculoMarcaModel` (tabla `vehiculomarca`, migración 00039)

Agrega al final de `models.py` (sin cerrar aún el "bloque pipas", todas las clases juntas):

```python
class VehiculoMarcaModel(Base):
    __tablename__ = "vehiculomarca"

    vho_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    vho_nombremarca: Mapped[str] = mapped_column(String(100), nullable=False)
    vho_estatus: Mapped[Optional[bool]] = mapped_column(Boolean, default=True)
```

**Verificación:** todos los tipos usados (`BigInteger`, `String`, `Boolean`, `Optional`) ya están importados al inicio de `models.py`.

---

## 3. Tarea 2.2 — `VehiculoModel` (tabla `vehiculos`, migración 00043)

**Requisito previo:** crea `FabricanteModel` (la relación de marca apunta a `fabricante`, y no existe aún). Agrégalo antes de `VehiculoModel`:

```python
class FabricanteModel(Base):
    __tablename__ = "fabricante"

    fab_id: Mapped[int] = mapped_column(Integer, primary_key=True)
    fab_mcr: Mapped[int] = mapped_column(
        Integer, ForeignKey("medidormarca.mrc_id", ondelete="CASCADE"), nullable=False
    )
    fab_nombrefabricante: Mapped[str] = mapped_column(String(100), nullable=False)
    fab_urlfabricante: Mapped[Optional[str]] = mapped_column(String(120), nullable=True)
    fab_estatus: Mapped[bool] = mapped_column(Boolean, nullable=False, default=True)

    vehiculos: Mapped[List["VehiculoModel"]] = relationship(back_populates="fabricante")
```

Luego el vehículo, con relación a marca y a solicitudes:

```python
class VehiculoModel(Base):
    __tablename__ = "vehiculos"

    vhe_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    vhe_mar_id: Mapped[int] = mapped_column(
        BigInteger, ForeignKey("fabricante.fab_id", ondelete="CASCADE"), nullable=False
    )
    vhe_res_id: Mapped[int] = mapped_column(Integer, nullable=False)
    vhe_modelo: Mapped[int] = mapped_column(Integer, nullable=False)
    vhe_combustible: Mapped[int] = mapped_column(Integer, nullable=False, default=0)
    vhe_tipovehiculo: Mapped[int] = mapped_column(BigInteger, nullable=False, default=0)
    vhe_fecharegistro: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )
    vhe_descripcion: Mapped[str] = mapped_column(String(120), nullable=False, unique=True)
    vhe_estatus: Mapped[bool] = mapped_column(Boolean, nullable=False, default=True)

    fabricante: Mapped[Optional["FabricanteModel"]] = relationship(back_populates="vehiculos")
    solicitudes: Mapped[List["SolicitudPipaModel"]] = relationship(back_populates="vehiculo")
    registros_combustible: Mapped[List["RegistroCombustibleModel"]] = relationship(back_populates="vehiculo")
    ubicaciones: Mapped[List["UbicacionPipaModel"]] = relationship(back_populates="vehiculo")
```

---

## 4. Tarea 2.3 — `SolicitudPipaModel` (tabla `solicitudpipas`, 00044 + 00053)

```python
class SolicitudPipaModel(Base):
    __tablename__ = "solicitudpipas"

    spp_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    spp_con: Mapped[int] = mapped_column(BigInteger, nullable=False)
    spp_srv: Mapped[int] = mapped_column(BigInteger, nullable=False)
    spp_vhe_id: Mapped[int] = mapped_column(
        BigInteger, ForeignKey("vehiculos.vhe_id"), nullable=False
    )
    spp_fechasolicitud: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )
    spp_horaentrega: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    srv_licencia: Mapped[str] = mapped_column(String(20), nullable=False)
    spp_estatus: Mapped[str] = mapped_column(String(20), nullable=False, default="Pendiente")

    vehiculo: Mapped["VehiculoModel"] = relationship(back_populates="solicitudes")
    historial: Mapped[List["HistorialSolicitudModel"]] = relationship(
        back_populates="solicitud", cascade="all, delete-orphan"
    )

    __table_args__ = (
        Index("ix_solicitudpipas_vhe_id", "spp_vhe_id"),
        Index("ix_solicitudpipas_estatus", "spp_estatus"),
    )
```

**⚠️ `spp_estatus` es `String(20)`** — NO `Boolean`. Los índices del `__table_args__` deben coincidir con los nombres de las migraciones.

---

## 5. Tarea 2.4 — `HistorialSolicitudModel`, `RegistroCombustibleModel`, `UbicacionPipaModel` (00053, 00054, 00055)

```python
class HistorialSolicitudModel(Base):
    __tablename__ = "historial_solicitud"

    histsol_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    histsol_spp_id: Mapped[int] = mapped_column(
        BigInteger, ForeignKey("solicitudpipas.spp_id", ondelete="CASCADE"), nullable=False
    )
    histsol_estado_anterior: Mapped[Optional[str]] = mapped_column(String(20), nullable=True)
    histsol_estado_nuevo: Mapped[str] = mapped_column(String(20), nullable=False)
    histsol_fecha: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )
    histsol_usuario: Mapped[Optional[str]] = mapped_column(String(30), nullable=True)
    histsol_motivo: Mapped[Optional[str]] = mapped_column(String(100), nullable=True)

    solicitud: Mapped["SolicitudPipaModel"] = relationship(back_populates="historial")

    __table_args__ = (
        Index("ix_historial_solicitud_spp_id", "histsol_spp_id"),
    )
```

```python
class RegistroCombustibleModel(Base):
    __tablename__ = "registros_combustible"

    reg_com_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    reg_com_vhe_id: Mapped[int] = mapped_column(
        BigInteger, ForeignKey("vehiculos.vhe_id", ondelete="CASCADE"), nullable=False
    )
    reg_com_lts: Mapped[Decimal] = mapped_column(Numeric(10, 2), nullable=False)
    reg_com_cost: Mapped[Decimal] = mapped_column(Numeric(12, 2), nullable=False)
    reg_com_fecha: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )
    reg_com_km_inicio: Mapped[int] = mapped_column(Integer, nullable=False)
    reg_com_km_final: Mapped[int] = mapped_column(Integer, nullable=False)

    vehiculo: Mapped["VehiculoModel"] = relationship(back_populates="registros_combustible")

    __table_args__ = (
        Index("ix_registros_combustible_vhe_id", "reg_com_vhe_id"),
    )
```

```python
class UbicacionPipaModel(Base):
    __tablename__ = "ubicacionespipa"

    ubp_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    ubp_vhe_id: Mapped[int] = mapped_column(
        BigInteger, ForeignKey("vehiculos.vhe_id", ondelete="CASCADE"), nullable=False
    )
    ubp_latitud: Mapped[float] = mapped_column(Float, nullable=False)
    ubp_longitud: Mapped[float] = mapped_column(Float, nullable=False)
    ubp_timestamp: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )
    ubp_estatus: Mapped[Optional[bool]] = mapped_column(Boolean, default=True)

    vehiculo: Mapped["VehiculoModel"] = relationship(back_populates="ubicaciones")

    __table_args__ = (
        Index("ix_ubicacionespipa_vhe_id", "ubp_vhe_id"),
    )
```

**Agrega a los imports de `models.py`:**
```python
from decimal import Decimal
from sqlalchemy import Numeric   # junto a los demás imports de sqlalchemy
```

---

## 6. Tarea 2.5 — Repositorios

### ⚠️ Gotcha clave del código existente
`GenericRepository` (base.py) usa `self.model.id` en `get_by_id()`, `exists()` y el orden por defecto de `get_all()`. **Los modelos de pipas no tienen columna `id`** (usan `vhe_id`, `spp_id`, etc.). Por eso cada repositorio de pipas debe sobrescribir `get_by_id()`.

### 6.1 `backend/app/infrastructure/repositories/vehiculo_repository.py`

```python
from typing import List, Optional

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.infrastructure.database.models import VehiculoModel
from app.infrastructure.repositories.base import GenericRepository


class VehiculoRepository(GenericRepository[VehiculoModel]):
    """Repositorio de vehiculos (pipas)."""

    def __init__(self, session: AsyncSession) -> None:
        super().__init__(VehiculoModel, session)

    async def get_by_id(self, record_id: int) -> Optional[VehiculoModel]:
        result = await self.session.execute(
            select(VehiculoModel).where(VehiculoModel.vhe_id == record_id)
        )
        return result.scalar_one_or_none()

    async def get_disponibles(self) -> List[VehiculoModel]:
        """Vehiculos activos disponibles para asignar."""
        result = await self.session.execute(
            select(VehiculoModel)
            .where(VehiculoModel.vhe_estatus == True)  # noqa: E712
            .order_by(VehiculoModel.vhe_id.asc())
        )
        return list(result.scalars().all())
```

### 6.2 `backend/app/infrastructure/repositories/marca_vehiculo_repository.py`

```python
from typing import Optional

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.infrastructure.database.models import VehiculoMarcaModel
from app.infrastructure.repositories.base import GenericRepository


class MarcaVehiculoRepository(GenericRepository[VehiculoMarcaModel]):
    """Repositorio del catalogo de marcas de vehiculo."""

    def __init__(self, session: AsyncSession) -> None:
        super().__init__(VehiculoMarcaModel, session)

    async def get_by_id(self, record_id: int) -> Optional[VehiculoMarcaModel]:
        result = await self.session.execute(
            select(VehiculoMarcaModel).where(VehiculoMarcaModel.vho_id == record_id)
        )
        return result.scalar_one_or_none()

    async def get_by_nombre(self, nombre: str) -> Optional[VehiculoMarcaModel]:
        result = await self.session.execute(
            select(VehiculoMarcaModel).where(
                VehiculoMarcaModel.vho_nombremarca == nombre
            )
        )
        return result.scalar_one_or_none()
```

### 6.3 `backend/app/infrastructure/repositories/solicitud_pipa_repository.py`

```python
from typing import List, Optional

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.infrastructure.database.models import SolicitudPipaModel
from app.infrastructure.repositories.base import GenericRepository


class SolicitudPipaRepository(GenericRepository[SolicitudPipaModel]):
    """Repositorio de solicitudes de pipa."""

    def __init__(self, session: AsyncSession) -> None:
        super().__init__(SolicitudPipaModel, session)

    async def get_by_id(self, record_id: int) -> Optional[SolicitudPipaModel]:
        result = await self.session.execute(
            select(SolicitudPipaModel).where(SolicitudPipaModel.spp_id == record_id)
        )
        return result.scalar_one_or_none()

    async def listar_por_estado(self, estado: str) -> List[SolicitudPipaModel]:
        """Solicitudes filtradas por estado, mas recientes primero."""
        result = await self.session.execute(
            select(SolicitudPipaModel)
            .where(SolicitudPipaModel.spp_estatus == estado)
            .order_by(SolicitudPipaModel.spp_fechasolicitud.desc())
        )
        return list(result.scalars().all())

    async def registrar_transicion(
        self, solicitud_id: int, estado_anterior: str, estado_nuevo: str,
        usuario: str, motivo: Optional[str] = None,
    ) -> None:
        """Actualiza el estado y deja bitacora en historial_solicitud."""
        from app.infrastructure.database.models import HistorialSolicitudModel

        solicitud = await self.get_by_id(solicitud_id)
        if solicitud is None:
            raise ValueError("Solicitud no encontrada")
        solicitud.spp_estatus = estado_nuevo
        self.session.add(HistorialSolicitudModel(
            histsol_spp_id=solicitud_id,
            histsol_estado_anterior=estado_anterior,
            histsol_estado_nuevo=estado_nuevo,
            histsol_usuario=usuario,
            histsol_motivo=motivo,
        ))
        await self.session.flush()
```

> El CRUD base (create/update/delete/get_all/count) lo hereda `GenericRepository`; solo sobrescribes lo que depende de la PK. `get_all` sin `id` cae al orden por defecto sin problema.

---

## 7. Tarea 2.6 — Registrar modelos en el `Base`

Los modelos se registran en `Base.metadata` al **importar** `models.py`. Para que SQLAlchemy/Alembic los detecten:

1. Agrega los 7 modelos nuevos al import de `backend/app/alembic/env.py` (lista de `# noqa: F401`):

```python
from app.infrastructure.database.models import (  # noqa: F401
    ...,
    FabricanteModel,
    HistorialSolicitudModel,
    RegistroCombustibleModel,
    SolicitudPipaModel,
    UbicacionPipaModel,
    VehiculoMarcaModel,
    VehiculoModel,
)
```

2. Verifica que no haya errores de importación:

```bash
docker compose exec backend python -c "from app.infrastructure.database import models; print('OK:', len(models.Base.metadata.tables), 'tablas')"
```

---

## 8. Verificación final del día

```bash
# 1. Los modelos importan sin error
docker compose exec backend python -c "import app.infrastructure.database.models"

# 2. Los repositorios importan sin error
docker compose exec backend python -c "import app.infrastructure.repositories.vehiculo_repository, app.infrastructure.repositories.marca_vehiculo_repository, app.infrastructure.repositories.solicitud_pipa_repository"

# 3. Metadatos de SQLAlchemy detectan las 7 tablas pipas
docker compose exec backend python -c "from app.infrastructure.database import models; [print(t) for t in ('vehiculomarca','fabricante','vehiculos','solicitudpipas','historial_solicitud','registros_combustible','ubicacionespipa') if t in models.Base.metadata.tables]"
```

**Definición de Terminado del día:**
- [ ] 6 modelos nuevos en `models.py` alineados a las migraciones.
- [ ] `FabricanteModel` creado (soporta el FK de `vehiculos`).
- [ ] 3 repositorios en `backend/app/infrastructure/repositories/`.
- [ ] `get_disponibles()` y `listar_por_estado()` implementados.
- [ ] `alembic/env.py` importa los modelos nuevos.
- [ ] Sin errores de importación (pasos 1 y 2).
- [ ] Commit: `feat: modelos y repositorios del modulo de pipas`.

**Recomendación para cerrar:** corre también `docker compose exec backend pytest -q` para confirmar que las tareas del día no rompieron nada existente.
