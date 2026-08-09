# HOJA DE INSTRUCCIONES — MODULO DE PIPAS — SPRINT 2 — DIA 3 (MIERCOLES)

**Autor:** Turno (Lider de turno)
**Fecha:** 2026-08-12
**Repositorio:** dapa2w — rama `feature/pipas-websocket`
**Objetivo del dia:** WebSocket `/ws/ubicacion` en FastAPI: conexion autenticada con JWT,
persistencia de lat/lng por mensaje, broadcast a supervisores y limpieza de conexiones.
Tests del WebSocket en verde.

---

## Tareas del dia

| # | Tarea | Archivo(s) principal(es) | DoD |
|---|-------|--------------------------|-----|
| 2.11 | Endpoint WebSocket `/ws/ubicacion` (hook `on_startup` de `app/core/events.py` ya existe) | `app/core/websocket_manager.py`, `app/api/v1/ubicacion_ws.py`, `app/main.py` | Cliente puede conectar y recibir mensajes |
| 2.12 | Autenticar la conexion del chofer con token JWT | `app/api/v1/ubicacion_ws.py` | Conexion sin token valido es rechazada |
| 2.13 | Persistir lat/lng en `ubicaciones_pipa` al recibir mensaje | `ubicacion_pipa_repository.py` | Se inserta un registro por update |
| 2.14 | Broadcast de ubicaciones a todos los conectados | `app/core/websocket_manager.py` | Multiples clientes reciben la misma posicion |
| 2.15 | Manejo de desconexion y limpieza de conexiones huerfanas | `app/core/websocket_manager.py` | Sin fugas de conexiones en logs |
| 2.16 | Tests del WebSocket (conectar con/sin token, recibir, desconectar) | `backend/tests/test_websocket_ubicacion.py` | Pytest verde |

**DoD del dia:** endpoint `/ws/ubicacion` operativo, conexion sin JWT rechazada, un registro
en `ubicaciones_pipa` por cada mensaje, broadcast a multiples clientes, conexiones limpiadas
al desconectarse y 6 pruebas en verde.

---

## 0. Prerequisitos y convenciones

- Continuar en la rama del Sprint 2. Lo del Martes (dominio de pipas, `OsrmProveedorRutas`,
  repositorios `asignacion/pozo/ubicacion_pipa`) debe estar MERGEADO. De aqui se reutiliza:
  `UbicacionPipaRepository` (con `get_ultima_ubicacion`) y `UbicacionPipaModel`.
- **La tabla `ubicaciones_pipa` YA existe**: `db/migrations/100055_crea_tabla_ubicaciones_pipa.sql`.
  El sprint la llama "migracion 00055"; en este repo la numeracion es `1000XX` y ya esta aplicada.
  **No crear una migracion nueva** para 2.13; solo persistir via repositorio.
- Autenticacion JWT: `decode_token(token, expected_type="access")` de `app/core/security.py`
  (ya levanta `TokenExpiredError`/`InvalidTokenError`). Para WS no se usa `Depends(get_current_user)`
  porque los endpoints WebSocket de FastAPI **no soportan dependencias**; el token viaja en
  **query param** `?token=...` (el navegador no puede poner headers en WebSocket).
- Ruta del endpoint: `/ws/ubicacion` (SIN prefijo `/api/v1`). Se registra directamente en `app/main.py`.
- Los middlewares `BaseHTTPMiddleware` (Audit, SecurityHeaders, PathSanitization) **dejan pasar**
  los scopes `websocket` (Starlette los reenvia sin procesar), asi que no bloquean el handshake.
  Verificar en la verificacion del dia.
- Logs estructurados con `get_logger(__name__)` (patron del proyecto: `logger.info("evento", clave=valor)`).
- Pruebas: las HTTP usan `httpx.AsyncClient`/`ASGITransport` (no soporta WS). Para WebSocket se usa
  `starlette.testclient.TestClient` en **funciones sincronas** (`def test_*`).
- Protocolo de mensaje (lo consumira el `AppChofer` del JUEVES, tarea 2.24):

  | Direccion | Mensaje |
  |---|---|
  | Cliente → servidor | `{"vhe_id": 4, "lat": 20.6400, "lng": -103.3110, "ts": "ISO-8601 opcional"}` |
  | Servidor → cliente | `{"type": "bienvenido"}` al conectar |
  | Servidor → emisor | `{"type": "ack", "vhe_id": 4}` tras persistir |
  | Servidor → todos | `{"type": "ubicacion", "vhe_id": 4, "lat": ..., "lng": ..., "ts": ...}` |

---

## TAREA 2.11 — Endpoint WebSocket `/ws/ubicacion`

### Crear `app/core/websocket_manager.py`

```python
"""
DAPAW2 Enterprise Platform - WebSocket Connection Manager.

Administra las conexiones activas de /ws/ubicacion:
conexion, desconexion y broadcast a todos los clientes (2.14, 2.15).
"""
import asyncio
from typing import Any, Dict, List, Set

from starlette.websockets import WebSocket

from app.shared.logger import get_logger

logger = get_logger(__name__)


class ConnectionManager:
    """Sigue las conexiones vivas y las cierra/rechaza de forma segura."""

    def __init__(self) -> None:
        self._conexiones: Set[WebSocket] = set()
        self._lock = asyncio.Lock()

    @property
    def count(self) -> int:
        return len(self._conexiones)

    async def connect(self, websocket: WebSocket) -> None:
        await websocket.accept()
        async with self._lock:
            self._conexiones.add(websocket)
        logger.info("ws_conectado", total=self.count)

    async def disconnect(self, websocket: WebSocket) -> None:
        async with self._lock:
            self._conexiones.discard(websocket)
        logger.info("ws_desconectado", total=self.count)

    async def broadcast(self, mensaje: Dict[str, Any]) -> None:
        """Reenvia el mensaje a todas las conexiones vivas (2.14).

        Si una conexion esta muerta se marca para limpiar (2.15) en vez
        de abortar el broadcast para el resto.
        """
        desconectados: List[WebSocket] = []
        for websocket in self._conexiones:
            try:
                await websocket.send_json(mensaje)
            except Exception:
                desconectados.append(websocket)
        for websocket in desconectados:
            await self.disconnect(websocket)


# Instancia unica compartida por todos los endpoints (singleton de aplicacion)
manager = ConnectionManager()
```

### Crear `app/api/v1/ubicacion_ws.py`

```python
"""
DAPAW2 Enterprise Platform - WebSocket de Ubicacion en Tiempo Real.

Ruta: /ws/ubicacion?token=<JWT>
- 2.11 endpoint WebSocket (el hook on_startup ya existe en app/core/events.py)
- 2.12 autenticacion JWT: rechaza conexiones sin token valido
- 2.13 persiste lat/lng en ubicaciones_pipa por mensaje
- 2.14 broadcast de la posicion a todos los conectados
- 2.15 limpieza de la conexion al desconectarse
"""
from datetime import datetime, timezone
from typing import Any, Dict

from fastapi import APIRouter, WebSocket, WebSocketDisconnect

from app.api.schemas.ubicacion import MensajeUbicacion
from app.core.security import decode_token
from app.core.websocket_manager import manager
from app.infrastructure.database.session import async_session_factory
from app.infrastructure.repositories.ubicacion_pipa_repository import (
    UbicacionPipaRepository,
)
from app.shared.logger import get_logger

logger = get_logger(__name__)

router = APIRouter(tags=["WebSocket Ubicacion"])


@router.websocket("/ws/ubicacion")
async def ws_ubicacion(websocket: WebSocket) -> None:
    # ---------------------------------------------------------------
    # 2.12 - Autenticacion JWT via query param (?token=...)
    # ---------------------------------------------------------------
    token = websocket.query_params.get("token", "")
    try:
        decode_token(token, expected_type="access")
    except Exception:
        logger.warning("ws_rechazado", motivo="token_invalido")
        await websocket.close(code=1008)  # 403 en el handshake
        return

    # ---------------------------------------------------------------
    # 2.11 - Aceptar conexion
    # ---------------------------------------------------------------
    await manager.connect(websocket)
    await websocket.send_json({"type": "bienvenido"})

    try:
        while True:
            data: Dict[str, Any] = await websocket.receive_json()
            try:
                mensaje = MensajeUbicacion.model_validate(data)
            except Exception:
                await websocket.send_json({"type": "error", "detail": "mensaje_invalido"})
                continue

            timestamp = mensaje.ts or datetime.now(timezone.utc)

            # -------------------------------------------------------
            # 2.13 - Persistir un registro por update
            # -------------------------------------------------------
            async with async_session_factory() as session:
                repo = UbicacionPipaRepository(session)
                await repo.crear_ubicacion(
                    vhe_id=mensaje.vhe_id,
                    latitud=mensaje.lat,
                    longitud=mensaje.lng,
                    timestamp=timestamp,
                )
                await session.commit()

            # Confirmacion al emisor
            await websocket.send_json({"type": "ack", "vhe_id": mensaje.vhe_id})

            # -------------------------------------------------------
            # 2.14 - Broadcast a todos los conectados (choferes y supervisores)
            # -------------------------------------------------------
            await manager.broadcast(
                {
                    "type": "ubicacion",
                    "vhe_id": mensaje.vhe_id,
                    "lat": mensaje.lat,
                    "lng": mensaje.lng,
                    "ts": timestamp.isoformat(),
                }
            )

    except WebSocketDisconnect:
        # -----------------------------------------------------------
        # 2.15 - Limpieza de conexiones huerfanas
        # -----------------------------------------------------------
        await manager.disconnect(websocket)
    except Exception as exc:
        logger.warning("ws_error", error=str(exc))
        await manager.disconnect(websocket)
```

### Crear `app/api/schemas/ubicacion.py`

```python
"""
DAPAW2 Enterprise Platform - Schema del mensaje de ubicacion (WebSocket).
"""
from datetime import datetime
from typing import Optional

from pydantic import BaseModel, Field


class MensajeUbicacion(BaseModel):
    """Mensaje que envia el chofer (AppChofer, tarea 2.24) cada 5-10 segundos."""
    vhe_id: int = Field(..., ge=1, description="ID del vehiculo")
    lat: float = Field(..., ge=-90, le=90, description="Latitud")
    lng: float = Field(..., ge=-180, le=180, description="Longitud")
    ts: Optional[datetime] = Field(default=None, description="Marca de tiempo (opcional)")
```

### Registrar en `app/main.py`

Agregar el import y montar el router SIN prefijo (para que la ruta sea `/ws/ubicacion`):

```python
from app.api.v1.ubicacion_ws import router as ubicacion_ws_router

# dentro de create_app(), junto a los demas include_router
app.include_router(ubicacion_ws_router)
```

### Tocar el hook `app/core/events.py` (2.11 menciona que ya existe)

Agregar al final de `on_startup()` una linea de log (reusa el manager):

```python
from app.core.websocket_manager import manager

    logger.info(
        "websocket_ready",
        ruta="/ws/ubicacion",
        conexiones=manager.count,
    )
```

### Agregar metodo al repositorio `ubicacion_pipa_repository.py`

Agregar a la clase `UbicacionPipaRepository` (archivo creado/continuado del Martes):

```python
from datetime import datetime, timezone

    async def crear_ubicacion(
        self,
        vhe_id: int,
        latitud: float,
        longitud: float,
        timestamp: Optional[datetime] = None,
    ) -> UbicacionPipaModel:
        """Persiste una posicion GPS (2.13). Devuelve el registro con su ubp_id."""
        ubicacion = UbicacionPipaModel(
            ubp_vhe_id=vhe_id,
            ubp_latitud=latitud,
            ubp_longitud=longitud,
            ubp_timestamp=timestamp or datetime.now(timezone.utc),
        )
        self.session.add(ubicacion)
        await self.session.flush()
        return ubicacion
```

---

## TAREAS 2.12 / 2.13 / 2.14 / 2.15 — Notas de implementacion

Estas cuatro quedan cubiertas por el codigo de 2.11 (archivo `ubicacion_ws.py` + `websocket_manager.py`):

- **2.12:** el `try/except` alrededor de `decode_token()` y `websocket.close(code=1008)`
  rechaza el handshake (403) sin token o con token invalido.
- **2.13:** cada `MensajeUbicacion` valido crea UNA fila en `ubicaciones_pipa` con su `ubp_timestamp`.
- **2.14:** `manager.broadcast()` reenvia `{"type": "ubicacion", ...}` a todos los conectados.
- **2.15:** `WebSocketDisconnect` dispara `manager.disconnect()`, y `broadcast()` limpia conexiones
  muertas sin abortar el ciclo.

> Opcional (recomendado si sobra tiempo): el token trae el `role` del chofer; si se quiere que el
> broadcast solo llegue a supervisores, guardar en `connect()` un mapeo `websocket -> rol` y filtrar
> en `broadcast()`. Para el DoD de hoy el broadcast a todos es suficiente (2.14 dice "a todos los
> supervisores conectados" y no hay restriccion de recibir del propio emisor).

---

## TAREA 2.16 — Tests del WebSocket

Crear `backend/tests/test_websocket_ubicacion.py`. Los tests de WebSocket son **funciones
sincronas** con `starlette.testclient.TestClient` (el `client` fixture de `conftest.py` usa
httpx, que no soporta WS).

```python
"""
DAPAW2 Enterprise Platform - WebSocket /ws/ubicacion Tests (2.16).

Cobertura: conectar con/sin token, recibir bienvenido, broadcast a
varios clientes, desconexion/limpieza y persistencia en ubicaciones_pipa.
"""
import time

import pytest
from starlette.testclient import TestClient, WebSocketDenialResponse

from app.main import app


def _token() -> str:
    from app.core.security import create_access_token

    return create_access_token(
        subject="chofer",
        user_id=2,
        email="chofer@test.com",
        role="chofer",
        permissions=[],
    )


def test_ws_rechaza_conexion_sin_token():
    with TestClient(app) as client:
        with pytest.raises(WebSocketDenialResponse):
            with client.websocket_connect("/ws/ubicacion"):
                pass


def test_ws_rechaza_token_invalido():
    with TestClient(app) as client:
        with pytest.raises(WebSocketDenialResponse):
            with client.websocket_connect("/ws/ubicacion?token=token-falso"):
                pass


def test_ws_acepta_token_valido_y_envia_bienvenido():
    with TestClient(app) as client:
        with client.websocket_connect(f"/ws/ubicacion?token={_token()}") as ws:
            data = ws.receive_json()
            assert data["type"] == "bienvenido"


def test_ws_broadcast_a_varios_clientes():
    with TestClient(app) as client:
        with client.websocket_connect(f"/ws/ubicacion?token={_token()}") as chofer:
            assert chofer.receive_json()["type"] == "bienvenido"
            with client.websocket_connect(f"/ws/ubicacion?token={_token()}") as supervisor:
                assert supervisor.receive_json()["type"] == "bienvenido"

                chofer.send_json({"vhe_id": 1, "lat": 20.64, "lng": -103.31})
                # el emisor primero recibe el ack (el servidor lo envia antes del broadcast)
                ack = chofer.receive_json()
                assert ack["type"] == "ack"
                assert ack["vhe_id"] == 1

                # el supervisor recibe la posicion
                posicion = supervisor.receive_json()
                assert posicion["type"] == "ubicacion"
                assert posicion["vhe_id"] == 1
                assert posicion["lat"] == 20.64
                assert posicion["lng"] == -103.31


def test_ws_desconexion_limpia_conexiones():
    from app.core.websocket_manager import manager

    with TestClient(app) as client:
        with client.websocket_connect(f"/ws/ubicacion?token={_token()}") as ws1:
            with client.websocket_connect(f"/ws/ubicacion?token={_token()}") as ws2:
                assert manager.count >= 2

    # al salir de los context managers las conexiones se limpian
    for _ in range(50):
        if manager.count == 0:
            break
        time.sleep(0.02)
    assert manager.count == 0


@pytest.mark.asyncio
async def test_ws_persiste_ubicacion():
    """2.13 - cada update inserta una fila en ubicaciones_pipa."""
    from sqlalchemy import select

    from app.infrastructure.database.models import UbicacionPipaModel, VehiculoModel
    from app.infrastructure.database.session import async_session_factory
    from app.infrastructure.repositories.ubicacion_pipa_repository import (
        UbicacionPipaRepository,
    )

    async with async_session_factory() as session:
        vhe_id = (
            await session.execute(select(VehiculoModel.vhe_id).limit(1))
        ).scalar_one()

        repo = UbicacionPipaRepository(session)
        creada = await repo.crear_ubicacion(vhe_id, 20.6412, -103.3105)
        await session.commit()

        assert creada.ubp_id is not None
        assert creada.ubp_vhe_id == vhe_id
        assert float(creada.ubp_latitud) == 20.6412
        assert float(creada.ubp_longitud) == -103.3105

        # limpieza para no ensuciar la base de pruebas
        await session.delete(creada)
        await session.commit()
```

> Nota: si `manager.count` no vuelve a 0 por el timing de la limpieza, revisar que
> `manager.disconnect()` se ejecute siempre en el `except WebSocketDisconnect` y que el
> broadcast tambien limpie conexiones muertas (ya implementado en 2.15).

---

## Verificacion del dia

```bash
cd backend
python -m pytest tests/test_websocket_ubicacion.py -q
# (todos en verde)
```

Prueba manual (cliente WebSocket, requiere `pip install websockets`):

```python
import asyncio
import json

import websockets

TOKEN = "..."  # token de un usuario chofer


async def main() -> None:
    async with websockets.connect(
        f"ws://localhost:8000/ws/ubicacion?token={TOKEN}"
    ) as ws:
        print("recibido:", await ws.recv())  # {"type": "bienvenido"}
        await ws.send(json.dumps({"vhe_id": 1, "lat": 20.6400, "lng": -103.3110}))
        print("ack:", await ws.recv())       # {"type": "ack", ...}
        print("broadcast:", await ws.recv()) # {"type": "ubicacion", ...}


asyncio.run(main())
```

Chequeos extra:
1. Abrir dos consolas con el script anterior: al enviar una posicion en la primera,
   la segunda recibe el `broadcast`.
2. `/ws/ubicacion` sin `?token=` o con token invalido responde 403 (rechazo de handshake).
3. Revisar `ubicaciones_pipa` en la BD: hay una fila por cada mensaje recibido.
4. Al cerrar el cliente, el log registra `ws_desconectado` y el conteo del manager baja
   (sin fugas: `total` vuelve a 0 con un solo cliente cerrado).
5. `/docs` sigue funcionando y el resto de endpoints HTTP intactos (los middlewares
   `BaseHTTPMiddleware` dejan pasar el scope websocket).

## DoD del dia + commit

- [ ] 2.11 `/ws/ubicacion` conecta y envia `bienvenido`.
- [ ] 2.12 conexion sin token valido rechazada (403).
- [ ] 2.13 una fila en `ubicaciones_pipa` por mensaje (repo `crear_ubicacion`).
- [ ] 2.14 broadcast a multiples clientes verificado.
- [ ] 2.15 desconexion limpia el manager (sin fugas en logs).
- [ ] 2.16 6 pruebas en verde (`test_websocket_ubicacion.py`).

Commit sugerido:

```bash
git add -A
git commit -m "feat(pipas): websocket /ws/ubicacion con auth JWT, persistencia, broadcast y limpieza (2.11-2.16)"
git push origin feature/pipas-websocket
```
