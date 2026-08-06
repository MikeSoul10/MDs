# Día 3 — Schemas Pydantic

## 0. Prerrequisitos (verificar antes de empezar)

- [ ] Día 2 terminado: los 7 modelos y los 3 repositorios importan sin errores.
- [ ] Rama de trabajo creada: `git checkout -b feature/pipas-schemas` (desde `feature/pipas-semana1`).

**Convenciones del proyecto (respétalas):**
- Pydantic V2 con `Field(...)` para requeridos y `description` en cada campo.
- Schemas de respuesta usan `model_config = {"from_attributes": True}` para poder construir desde los modelos SQLAlchemy.
- Los schemas de respuesta usan los **mismos nombres de atributo de los modelos** (`vhe_id`, `vho_nombremarca`, `spp_estatus`, etc.), NO nombres traducidos.
- Un archivo por módulo en `backend/app/api/schemas/`.

**⚠️ Recordatorios del esquema ya definido:**
- `solicitudpipas.spp_estatus` es `String(20)` con catálogo: `Pendiente / En Ruta / Entregada / Cancelada`.
- `vehiculos.vhe_mar_id` apunta a `fabricante.fab_id`.

---

## 1. Referencia de atributos de los modelos (para los campos de respuesta)

| Modelo | Atributos |
|--------|-----------|
| `VehiculoMarcaModel` | `vho_id`, `vho_nombremarca`, `vho_estatus` |
| `VehiculoModel` | `vhe_id`, `vhe_mar_id`, `vhe_res_id`, `vhe_modelo`, `vhe_combustible`, `vhe_tipovehiculo`, `vhe_fecharegistro`, `vhe_descripcion`, `vhe_estatus` |
| `SolicitudPipaModel` | `spp_id`, `spp_con`, `spp_srv`, `spp_vhe_id`, `spp_fechasolicitud`, `spp_horaentrega`, `srv_licencia`, `spp_estatus` |

**Recomendación:** define el catálogo de estados en `backend/app/shared/enums.py` (como los demás enums del proyecto) y úsalo en los schemas:

```python
class SolicitudEstado(str, Enum):
    """Estados del ciclo de vida de una solicitud de pipa."""
    PENDIENTE = "Pendiente"
    EN_RUTA = "En Ruta"
    ENTREGADA = "Entregada"
    CANCELADA = "Cancelada"

    @classmethod
    def es_valida(cls, desde: str, hacia: str) -> bool:
        """Reglas de transición: Pendiente->En Ruta->Entregada, cualquier estado -> Cancelada."""
        if desde == hacia:
            return False
        if hacia == cls.CANCELADA.value:
            return True
        return {
            cls.PENDIENTE.value: cls.EN_RUTA.value,
            cls.EN_RUTA.value: cls.ENTREGADA.value,
        }.get(desde) == hacia
```

---

## 2. Tarea 3.1 — `backend/app/api/schemas/vehiculos.py`

**Definición de Terminado:** validación de modelo/año, descripción no vacía, combustible >= 0.

```python
"""
DAPA2W Enterprise Platform - Vehiculos API Schemas.

Schemas Pydantic para el CRUD de vehiculos (pipas).
"""

from datetime import datetime
from typing import Optional

from pydantic import BaseModel, Field


class VehiculoCreate(BaseModel):
    """Alta de vehiculo."""
    vhe_mar_id: int = Field(..., ge=1, description="ID del fabricante (fab_id)")
    vhe_res_id: int = Field(..., ge=1, description="ID del responsable (personafisica)")
    vhe_modelo: int = Field(..., ge=1900, le=2100, description="Año del modelo")
    vhe_combustible: int = Field(default=0, ge=0, le=100, description="Porcentaje de combustible")
    vhe_tipovehiculo: int = Field(default=0, ge=0, description="Tipo de vehiculo")
    vhe_descripcion: str = Field(
        ..., min_length=1, max_length=120, description="Descripcion (no vacia)"
    )
    vhe_estatus: bool = Field(default=True)


class VehiculoUpdate(BaseModel):
    """Actualizacion parcial de vehiculo (todos los campos opcionales)."""
    vhe_mar_id: Optional[int] = Field(default=None, ge=1)
    vhe_res_id: Optional[int] = Field(default=None, ge=1)
    vhe_modelo: Optional[int] = Field(default=None, ge=1900, le=2100)
    vhe_combustible: Optional[int] = Field(default=None, ge=0, le=100)
    vhe_tipovehiculo: Optional[int] = Field(default=None, ge=0)
    vhe_descripcion: Optional[str] = Field(default=None, min_length=1, max_length=120)
    vhe_estatus: Optional[bool] = None


class VehiculoResponse(BaseModel):
    """Respuesta de vehiculo."""
    vhe_id: int
    vhe_mar_id: int
    vhe_res_id: int
    vhe_modelo: int
    vhe_combustible: int
    vhe_tipovehiculo: int
    vhe_fecharegistro: datetime
    vhe_descripcion: str
    vhe_estatus: bool

    model_config = {"from_attributes": True}
```

**Nota:** `min_length=1` en `vhe_descripcion` evita cadenas vacías; la unicidad de `vhe_descripcion` y la existencia de `vhe_mar_id` se validan en el repositorio/servicio (día 4).

---

## 3. Tarea 3.2 — `backend/app/api/schemas/vehiculomarcas.py`

**Definición de Terminado:** nombre de marca requerido y único.

```python
"""
DAPA2W Enterprise Platform - VehiculoMarcas API Schemas.

Schemas Pydantic para el catalogo de marcas de vehiculo.
"""

from typing import Optional

from pydantic import BaseModel, Field


class MarcaVehiculoCreate(BaseModel):
    """Alta de marca de vehiculo."""
    vho_nombremarca: str = Field(
        ..., min_length=1, max_length=100,
        description="Nombre de la marca (requerido y unico)"
    )
    vho_estatus: bool = Field(default=True)


class MarcaVehiculoResponse(BaseModel):
    """Respuesta de marca de vehiculo."""
    vho_id: int
    vho_nombremarca: str
    vho_estatus: bool

    model_config = {"from_attributes": True}


class MarcaVehiculoUpdate(BaseModel):
    """Actualizacion de marca (opcional)."""
    vho_nombremarca: Optional[str] = Field(default=None, min_length=1, max_length=100)
    vho_estatus: Optional[bool] = None
```

**Nota:** la unicidad de `vho_nombremarca` se comprueba con `MarcaVehiculoRepository.get_by_nombre()` en el endpoint (día 4).

---

## 4. Tarea 3.3 — `backend/app/api/schemas/solicitudes_pipa.py`

**Definición de Terminado:** `spp_vhe_id` requerido; transición de estado validada.

```python
"""
DAPA2W Enterprise Platform - Solicitudes Pipa API Schemas.

Schemas Pydantic para el CRUD de solicitudes de pipa y su cambio de estado.
"""

from datetime import datetime
from typing import Optional

from pydantic import BaseModel, Field, model_validator

from app.shared.enums import SolicitudEstado


class SolicitudPipaCreate(BaseModel):
    """Alta de solicitud de pipa (ligada al vehiculo)."""
    spp_con: int = Field(..., ge=1, description="ID del contribuyente")
    spp_srv: int = Field(..., ge=1, description="ID del servicio")
    spp_vhe_id: int = Field(..., ge=1, description="ID del vehiculo (obligatorio)")
    spp_horaentrega: datetime = Field(..., description="Fecha/hora de entrega")
    srv_licencia: str = Field(..., min_length=1, max_length=20, description="Licencia del operador")
    spp_estatus: str = Field(default=SolicitudEstado.PENDIENTE.value)


class SolicitudPipaUpdate(BaseModel):
    """Actualizacion parcial de solicitud."""
    spp_con: Optional[int] = Field(default=None, ge=1)
    spp_srv: Optional[int] = Field(default=None, ge=1)
    spp_vhe_id: Optional[int] = Field(default=None, ge=1)
    spp_horaentrega: Optional[datetime] = None
    srv_licencia: Optional[str] = Field(default=None, min_length=1, max_length=20)


class SolicitudPipaResponse(BaseModel):
    """Respuesta de solicitud de pipa."""
    spp_id: int
    spp_con: int
    spp_srv: int
    spp_vhe_id: int
    spp_fechasolicitud: datetime
    spp_horaentrega: datetime
    srv_licencia: str
    spp_estatus: str

    model_config = {"from_attributes": True}


class SolicitudPipaCambioEstado(BaseModel):
    """Cambio de estado con bitacora."""
    estado_anterior: Optional[str] = Field(default=None, description="Estado previo (auto)")
    estado_nuevo: str = Field(..., description="Estado destino")
    motivo: Optional[str] = Field(default=None, max_length=100)

    @model_validator(mode="after")
    def validar_transicion(self) -> "SolicitudPipaCambioEstado":
        if self.estado_nuevo not in SolicitudEstado.__members__.values():
            raise ValueError(
                "Estado invalido. Use: Pendiente, En Ruta, Entregada, Cancelada"
            )
        if self.estado_anterior and not SolicitudEstado.es_valida(
            self.estado_anterior, self.estado_nuevo
        ):
            raise ValueError(
                f"Transicion invalida: {self.estado_anterior} -> {self.estado_nuevo}"
            )
        return self
```

**Nota:** el endpoint `PATCH /solicitudes_pipa/{id}/estado` (día 4) recibe `SolicitudPipaCambioEstado` sin `estado_anterior` (se lee del registro) y usa el repositorio `registrar_transicion()` del día 2. La validación de catálogo ocurre aquí; la regla "no repetir estado" se valida en el endpoint.

---

## 5. Tarea 3.4 — `common.py` (paginación/ordenamiento)

**Ya está hecho ✅** — `backend/app/api/schemas/common.py` contiene `PagedResponse[T]` genérico (Pydantic `Generic[T]`) con `items`, `total`, `page`, `page_size`, `pages`.

**Uso en los endpoints del día 4:**

```python
from typing import List
from app.api.schemas.common import PagedResponse

def _paginar(registros: List[object], total: int, page: int, page_size: int) -> PagedResponse:
    pages = (total + page_size - 1) // page_size if page_size else 0
    return PagedResponse(items=registros, total=total, page=page, page_size=page_size, pages=pages)
```

No se requiere crear nada nuevo; solo reutilizarlo.

---

## 6. Verificación final del día

```bash
# Los schemas importan sin error
docker compose exec backend python -c "import app.api.schemas.vehiculos, app.api.schemas.vehiculomarcas, app.api.schemas.solicitudes_pipa, app.api.schemas.common"

# Prueba rapida de validacion (estado invalido debe fallar)
docker compose exec backend python -c "
from app.api.schemas.solicitudes_pipa import SolicitudPipaCambioEstado
try:
    SolicitudPipaCambioEstado(estado_nuevo='Fake')
except ValueError as e:
    print('OK - rechazado:', e)
"
```

**Definición de Terminado del día:**
- [ ] `vehiculos.py` con `VehiculoCreate`, `VehiculoUpdate`, `VehiculoResponse` (validaciones: año, descripción no vacía, combustible >= 0).
- [ ] `vehiculomarcas.py` con `MarcaVehiculoCreate`, `MarcaVehiculoResponse` (nombre requerido).
- [ ] `solicitudes_pipa.py` con `SolicitudPipaCreate`, `SolicitudPipaUpdate`, `SolicitudPipaResponse`, `SolicitudPipaCambioEstado` (transición validada).
- [ ] `SolicitudEstado` (enum) en `shared/enums.py` con reglas de transición.
- [ ] `common.py` reutilizado (no duplicado).
- [ ] Importan sin error (paso 1) y la validación rechaza estados inválidos (paso 2).
- [ ] Commit: `feat: schemas pydantic del modulo de pipas`.

**Recomendación para cerrar:** corre `docker compose exec backend pytest -q` para confirmar que no rompiste nada existente.
