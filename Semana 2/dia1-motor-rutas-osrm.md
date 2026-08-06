# Día 1 (Lunes) — Semana 2: Motor de Rutas OSRM

## 0. Prerrequisitos (verificar antes de empezar)

- [ ] Semana 1 terminada y commiteada (ya está: `bc777e4` en `feature/modulopipasSemana1`).
- [ ] API de `vehiculos`, `vehiculomarcas` y `solicitudes_pipa` funcional en `/api/v1`.
- [ ] Rama de trabajo creada: `git checkout -b feature/pipas-rutas` (desde `feature/modulopipasSemana1` o `main` según flujo).

**Convenciones del proyecto (respétalas):**
- Migraciones en `db/migrations/` con prefijo numérico **1000xx** (el equipo renumeró 00053-00055 → 100053-100055; la siguiente es **100056**).
- Tablas con nombres en español y columnas con prefijo (`rtsol_` para rutas de solicitud).
- Cliente HTTP siguiendo el patrón de `app/infrastructure/ai/ollama_client.py` (httpx + `httpx.Timeout` + manejo de errores).
- Modelos SQLAlchemy 2.x con `Mapped`/`mapped_column` en `app/infrastructure/database/models.py`.
- La red Docker se llama `dapa-network` (verificar al agregar servicios).

**Datos de coordenadas:** OSRM recibe y devuelve coordenadas en orden **lon, lat**.

---

## 1. Tarea 2.1 — Servicio `osrm` en `docker-compose.yml`

**Definición de Terminado:** `docker compose up` levanta OSRM junto a los demás servicios.

Agrega al final de `services:` (antes de `volumes:`):

```yaml
  # ---------------------------------------------------------------------------
  # OSRM - Motor de rutas
  # ---------------------------------------------------------------------------
  osrm:
    image: ghcr.io/project-osrm/osrm-backend
    container_name: dapa2-osrm
    restart: unless-stopped
    command: osrm-routed --algorithm mld /data/map.osrm
    ports:
      - "5000:5000"
    volumes:
      - osrm-data:/data
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS 'http://localhost:5000/nearest/v1/driving/-103.5,19.2' || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 20s
    networks:
      - dapa-network
```

Y agrega el volumen en la sección `volumes:`:

```yaml
volumes:
  pgdata:
    driver: local
  storage_data:
    driver: local
  osrm-data:
    driver: local
```

Agrega la variable al servicio `backend` (en `environment:`):

```yaml
      # --- OSRM Routing ---
      OSRM_URL: ${OSRM_URL:-http://osrm:5000}
      OSRM_TIMEOUT: ${OSRM_TIMEOUT:-10}
```

> **Nota:** `osrm-routed` necesita los archivos procesados (`map.osrm`) en `/data` (tarea 2.2). Sin ellos el contenedor se reinicia.

---

## 2. Tarea 2.2 — Preparar datos de mapa y cargar `car.lua`

**Definición de Terminado:** `curl http://osrm:5000/route/v1/driving/...` responde.

1. Descarga el extract OSM del área de la delegación (elegir uno pequeño; ej. de Geofabrik el estado/municipio, o `mexico-latest.osm.pbf` si no hay extract menor):

   ```bash
   # ej. con el extract de México (o el de tu estado)
   curl -L -o db/osrm/map.osm.pbf https://download.geofabrik.de/north-america/mexico-latest.osm.pbf
   ```

2. Procesa el mapa con el pipeline de OSRM (se ejecuta como comandos únicos sobre el volumen compartido). Copia el `.pbf` a un folder del host y usa montaje temporal:

   ```bash
   mkdir -p db/osrm
   # 1) extract  -> genera map.osrm + map.osrm.*
   docker run --rm -v "%cd%\db\osrm":/data ghcr.io/project-osrm/osrm-backend \
     osrm-extract -p /opt/car.lua /data/map.osm.pbf
   # 2) partition (requerido para el algoritmo MLD)
   docker run --rm -v "%cd%\db\osrm":/data ghcr.io/project-osrm/osrm-backend \
     osrm-partition /data/map.osrm
   # 3) customize
   docker run --rm -v "%cd%\db\osrm":/data ghcr.io/project-osrm/osrm-backend \
     osrm-customize /data/map.osrm
   ```

   > Si usas volumen `osrm-data`, primero copia los archivos con `docker compose cp map.osrm osrm:/data/` (y sus `.osrm.*`).

3. Levanta el servicio y prueba:

   ```bash
   docker compose up -d osrm
   curl "http://localhost:5000/route/v1/driving/-103.5,19.2;-103.6,19.3?overview=false"
   # -> {"code":"Ok","routes":[...], ...}
   ```

4. Documenta el proceso en `documentacion/` (pasos, URL del extract y fecha) para poder reproducirlo.

---

## 3. Tarea 2.3 — Cliente HTTP `OsrmClient`

**Definición de Terminado:** clase con `get_route()` y `get_nearest()`, timeout configurable.

### 3.1 Configuración — `backend/app/config.py`

Dentro de `class Settings`, agrega (junto a las demás secciones):

```python
    # --- OSRM Routing ---
    OSRM_URL: str = "http://osrm:5000"
    OSRM_TIMEOUT: int = 10
```

> Para correr local fuera de Docker, en `.env` usa `OSRM_URL=http://localhost:5000`.

### 3.2 Cliente — `backend/app/infrastructure/osrm/osrm_client.py` (nuevo)

```python
"""
DAPA2W Enterprise Platform - OSRM HTTP Client.

Cliente HTTP para el motor de rutas OSRM (Open Source Routing Machine).
Provee calculo de rutas y punto mas cercano sobre la red vial.
"""

from typing import Any, Dict, List, Tuple

import httpx

from app.config import settings
from app.shared.logger import get_logger

logger = get_logger(__name__)


class OsrmClient:
    """Cliente HTTP para la API de OSRM."""

    def __init__(self) -> None:
        self._base_url = settings.OSRM_URL.rstrip("/")
        self._timeout = settings.OSRM_TIMEOUT

    def _get_client(self) -> httpx.AsyncClient:
        """Crea un cliente httpx fresco por request."""
        return httpx.AsyncClient(
            base_url=self._base_url,
            timeout=httpx.Timeout(self._timeout),
        )

    async def get_route(
        self,
        coordinates: List[Tuple[float, float]],
        profile: str = "driving",
    ) -> Dict[str, Any]:
        """
        Calcula la ruta entre coordenadas (lon, lat).

        Args:
            coordinates: lista de puntos [(lon, lat), ...].
            profile: perfil de ruteo (driving, cycling, walking).

        Returns:
            Dict con distance_km, duration_min y geometry (GeoJSON).

        Raises:
            httpx.HTTPStatusError: si OSRM no responde 200.
            ValueError: si OSRM no puede calcular la ruta.
        """
        coords = ";".join(f"{lon},{lat}" for lon, lat in coordinates)
        params = {
            "overview": "full",
            "geometries": "geojson",
            "alternatives": "false",
            "steps": "false",
        }
        async with self._get_client() as client:
            response = await client.get(
                f"/route/v1/{profile}/{coords}", params=params
            )
            response.raise_for_status()
            data = response.json()

        if data.get("code") != "Ok" or not data.get("routes"):
            raise ValueError(
                f"OSRM no pudo calcular la ruta: {data.get('code')}"
            )

        route = data["routes"][0]
        return {
            "distance_km": round(route["distance"] / 1000, 3),
            "duration_min": round(route["duration"] / 60, 2),
            "geometry": route.get("geometry"),
        }

    async def get_nearest(
        self, longitude: float, latitude: float, profile: str = "driving"
    ) -> Dict[str, Any]:
        """Encuentra el punto mas cercano de la red vial a (lon, lat)."""
        async with self._get_client() as client:
            response = await client.get(
                f"/nearest/v1/{profile}/{longitude},{latitude}"
            )
            response.raise_for_status()
            data = response.json()

        if data.get("code") != "Ok" or not data.get("waypoints"):
            raise ValueError(
                f"OSRM no encontro punto cercano: {data.get('code')}"
            )

        wp = data["waypoints"][0]
        return {
            "lat": wp["location"][1],
            "lon": wp["location"][0],
            "distance_m": wp["distance"],
        }
```

---

## 4. Tarea 2.4 — Migración `100056_crea_tabla_rutas_solicitud.sql`

**Definición de Terminado:** migración ejecutable y reversible (usa `IF NOT EXISTS`).

```sql
-- 100056_crea_tabla_rutas_solicitud.sql
-- Tabla: rutas_solicitud
-- Descripcion: ruta calculada por solicitud de pipa (origen, destino, km, duracion, geometria GeoJSON)
-- Autor: Ing. <tu nombre>
-- Fecha: 2026-08-06

CREATE TABLE IF NOT EXISTS rutas_solicitud (
    rtsol_id BIGINT NOT NULL GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    rtsol_spp_id BIGINT NOT NULL REFERENCES solicitudpipas(spp_id) ON DELETE CASCADE,
    rtsol_origen_lat DOUBLE PRECISION NOT NULL,
    rtsol_origen_lng DOUBLE PRECISION NOT NULL,
    rtsol_destino_lat DOUBLE PRECISION NOT NULL,
    rtsol_destino_lng DOUBLE PRECISION NOT NULL,
    rtsol_distancia_km NUMERIC(10,3) NOT NULL,
    rtsol_duracion_min NUMERIC(10,2) NOT NULL,
    rtsol_geometria JSONB,
    rtsol_creado TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS ix_rutas_solicitud_spp_id ON rutas_solicitud (rtsol_spp_id);
```

> La geometría GeoJSON de OSRM (`geometry`) se guarda como `JSONB`.

---

## 5. Tarea 2.5 — `RutaSolicitudModel` + `RutaRepository`

**Definición de Terminado:** CRUD funcional (get por solicitud, crear, get por id).

### 5.1 Modelo — `backend/app/infrastructure/database/models.py`

Agrega al final (después de `UbicacionPipaModel`), respetando el estilo:

```python
class RutaSolicitudModel(Base):
    __tablename__ = "rutas_solicitud"

    rtsol_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    rtsol_spp_id: Mapped[int] = mapped_column(
        BigInteger, ForeignKey("solicitudpipas.spp_id", ondelete="CASCADE"),
        nullable=False,
    )
    rtsol_origen_lat: Mapped[float] = mapped_column(Float, nullable=False)
    rtsol_origen_lng: Mapped[float] = mapped_column(Float, nullable=False)
    rtsol_destino_lat: Mapped[float] = mapped_column(Float, nullable=False)
    rtsol_destino_lng: Mapped[float] = mapped_column(Float, nullable=False)
    rtsol_distancia_km: Mapped[Decimal] = mapped_column(Numeric(10, 3), nullable=False)
    rtsol_duracion_min: Mapped[Decimal] = mapped_column(Numeric(10, 2), nullable=False)
    rtsol_geometria: Mapped[Optional[dict]] = mapped_column(JSONB, nullable=True)
    rtsol_creado: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )

    solicitud: Mapped["SolicitudPipaModel"] = relationship(back_populates="rutas")

    __table_args__ = (
        Index("ix_rutas_solicitud_spp_id", "rtsol_spp_id"),
    )
```

**Ojo:** agrega también el `back_populates` en `SolicitudPipaModel` (junto a `historial`, ~línea 508):

```python
    rutas: Mapped[List["RutaSolicitudModel"]] = relationship(
        back_populates="solicitud"
    )
```

### 5.2 Registrar el modelo en Alembic — `backend/app/alembic/env.py`

Agrega `RutaSolicitudModel` a la lista de imports de modelos (aprox. línea 20):

```python
    RutaSolicitudModel,
```

### 5.3 Repositorio — `backend/app/infrastructure/repositories/ruta_solicitud_repository.py` (nuevo)

```python
from datetime import datetime
from typing import List, Optional

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.infrastructure.database.models import RutaSolicitudModel
from app.infrastructure.repositories.base import GenericRepository


class RutaSolicitudRepository(GenericRepository[RutaSolicitudModel]):
    """Repositorio de rutas calculadas por solicitud."""

    def __init__(self, session: AsyncSession) -> None:
        super().__init__(RutaSolicitudModel, session)

    async def get_by_id(self, record_id: int) -> Optional[RutaSolicitudModel]:
        result = await self.session.execute(
            select(RutaSolicitudModel).where(RutaSolicitudModel.rtsol_id == record_id)
        )
        return result.scalar_one_or_none()

    async def get_by_solicitud(self, spp_id: int) -> Optional[RutaSolicitudModel]:
        """Ultima ruta calculada para una solicitud."""
        result = await self.session.execute(
            select(RutaSolicitudModel)
            .where(RutaSolicitudModel.rtsol_spp_id == spp_id)
            .order_by(RutaSolicitudModel.rtsol_creado.desc())
            .limit(1)
        )
        return result.scalar_one_or_none()

    async def create(self, ruta: RutaSolicitudModel) -> RutaSolicitudModel:
        self.session.add(ruta)
        await self.session.flush()
        return ruta
```

> Recuerda: `GenericRepository` usa `self.model.id`, por eso se sobrescribe `get_by_id` (misma razón que los repos de pipas).

---

## Verificación final del día

```bash
# Levanta OSRM (si ya procesaste el mapa)
docker compose up -d --build

# 1. OSRM responde
curl "http://localhost:5000/route/v1/driving/-103.5,19.2;-103.6,19.3?overview=false"

# 2. Migracion aplicada (db-migrate corre las nuevas)
docker compose exec db psql -U postgres -d dapatlqdb -c "\d rutas_solicitud"

# 3. Modelos y cliente importan sin error
docker compose exec backend python -c "
from app.infrastructure.database.models import RutaSolicitudModel
from app.infrastructure.repositories.ruta_solicitud_repository import RutaSolicitudRepository
from app.infrastructure.osrm.osrm_client import OsrmClient
print('IMPORTS OK')
"

# 4. Prueba rapida del cliente contra OSRM (desde el backend)
docker compose exec backend python -c "
import asyncio
from app.infrastructure.osrm.osrm_client import OsrmClient
async def main():
    r = await OsrmClient().get_route([(-103.5, 19.2), (-103.6, 19.3)])
    print(r)
asyncio.run(main())
"

# 5. pytest sin regresiones
docker compose exec backend pytest -q
```

**Definición de Terminado del día:**
- [ ] Servicio `osrm` en `docker-compose.yml` (red `dapa-network`, puerto 5000, volumen `osrm-data`).
- [ ] Mapa procesado (`.osrm`) cargado en el volumen y `curl /route/v1/driving/...` responde `{"code":"Ok",...}`.
- [ ] `OsrmClient` con `get_route()` y `get_nearest()` probado desde el backend.
- [ ] Migración `100056_crea_tabla_rutas_solicitud.sql` aplicada (`\d rutas_solicitud`).
- [ ] `RutaSolicitudModel` + `RutaSolicitudRepository` importando y con CRUD funcional.
- [ ] Commit: `feat: motor de rutas OSRM y tabla rutas_solicitud`.
