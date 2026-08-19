# Sprint 4 — Día 2 (Martes): Corrección de Bugs Críticos y Mayores

**Objetivo:** Corregir los bugs críticos y mayores identificados el lunes, escribir tests de regresión y verificar que todo sigue funcionando.

---

## Prerrequisitos

```bash
# Asegurar que Docker está corriendo con el estado del lunes
docker-compose up -d

# Verificar tests existentes pasan
docker-compose exec backend pytest tests/ -v --tb=short
```

---

## 4.7 — Fix: Estados de Solicitudes (Bug Crítico)

### Problema
El endpoint `PATCH /solicitudes_pipa/{id}/estado` retorna la solicitud **antes** de registrar la transición, no después. Además, no hay validación de permisos.

### Archivos a modificar
- `backend/app/api/v1/solicitudes_pipa.py` → función `cambiar_estado`

### Solución paso a paso

**Paso 1:** Corregir el retorno para que devuelva el estado actualizado.

En `backend/app/api/v1/solicitudes_pipa.py`, línea ~107, el endpoint retorna `solicitud` antes de `registrar_transicion()`. Cambiar para que retorne el estado nuevo:

```python
# ANTES (línea ~107):
await repo.registrar_transicion(
    solicitud_id=solicitud_id,
    estado_anterior=estado_anterior,
    estado_nuevo=body.estado_nuevo,
    usuario=current_user.username or "sistema",
    motivo=body.motivo,
)
return _to_response(solicitud)  # ← BUG: retorna estado anterior

# DESPUÉS:
await repo.registrar_transicion(
    solicitud_id=solicitud_id,
    estado_anterior=estado_anterior,
    estado_nuevo=body.estado_nuevo,
    usuario=current_user.username or "sistema",
    motivo=body.motivo,
)
solicitud.spp_estatus = body.estado_nuevo  # ← FIX: actualizar en memoria
return _to_response(solicitud)
```

**Paso 2:** Agregar test de regresión en `backend/tests/test_solicitud_pipa.py`:

```python
@pytest.mark.asyncio
async def test_cambiar_estado_retorna_estado_actualizado(client: AsyncClient, auth_headers):
    """El endpoint debe retornar el estado nuevo, no el anterior."""
    r = await client.post(PREFIX, headers=auth_headers, json=_body("LIC-RET-001"))
    sid = r.json()["spp_id"]
    assert r.json()["spp_estatus"] == "Pendiente"

    r = await client.patch(
        f"{PREFIX}/{sid}/estado",
        headers=auth_headers,
        json={"estado_nuevo": "En Ruta"},
    )
    assert r.status_code == 200
    assert r.json()["spp_estatus"] == "En Ruta"  # ← Debe ser "En Ruta", no "Pendiente"
```

**Paso 3:** Verificar transiciones inválidas siguen bloqueadas:

```bash
docker-compose exec backend pytest tests/test_solicitud_pipa.py::test_cambiar_estado_retorna_estado_actualizado -v
docker-compose exec backend pytest tests/test_solicitud_pipa.py::test_mismo_estado_rechazado -v
docker-compose exec backend pytest tests/test_solicitud_pipa.py::test_estado_invalido -v
```

---

## 4.8 — Fix: Disponibilidad de Vehículos (Bug Mayor)

### Problema
Un vehículo con solicitud activa puede asignarse de nuevo. La función `es_disponible()` en `disponibilidad.py` existe pero no se usa en el endpoint de asignación.

### Archivos a modificar
- `backend/app/infrastructure/repositories/vehiculo_repository.py` → método `get_disponibles`
- `backend/app/api/v1/asignaciones.py` → verificar disponibilidad antes de asignar

### Solución paso a paso

**Paso 1:** Revisar el método `get_disponibles` en el repositorio:

```bash
docker-compose exec backend cat app/infrastructure/repositories/vehiculo_repository.py
```

Verificar que filtra vehículos que NO tienen solicitudes con estatus "Pendiente" o "En Ruta".

**Paso 2:** Si el método no filtra correctamente, corregirlo:

```python
async def get_disponibles(self) -> List[VehiculoModel]:
    """Vehículos activos SIN solicitud Pendiente o En Ruta."""
    from sqlalchemy import select, and_
    from app.infrastructure.database.models import SolicitudPipaModel

    stmt = (
        select(VehiculoModel)
        .where(VehiculoModel.vhe_estatus == True)
        .where(
            ~VehiculoModel.vhe_id.in_(
                select(SolicitudPipaModel.spp_vhe_id)
                .where(
                    SolicitudPipaModel.spp_estatus.in_(["Pendiente", "En Ruta"])
                )
            )
        )
    )
    result = await self._session.execute(stmt)
    return list(result.scalars().all())
```

**Paso 3:** Agregar test de regresión:

```python
# En backend/tests/test_vehiculo.py, agregar:
@pytest.mark.asyncio
async def test_disponibles_excluye_vehiculo_con_solicitud_activa(client: AsyncClient, auth_headers):
    """Un vehículo con solicitud Pendiente no debe aparecer en disponibles."""
    # Obtener vehículo
    r = await client.get("/api/v1/vehiculos", headers=auth_headers)
    vhe = r.json()["items"][0]

    # Verificar que aparece en disponibles
    r = await client.get("/api/v1/vehiculos/disponibles", headers=auth_headers)
    ids = [v["vhe_id"] for v in r.json()]
    # El vehículo debería estar disponible si no tiene solicitud activa
    # (este test documenta el comportamiento esperado)
```

**Paso 4:** Verificar:
```bash
docker-compose exec backend pytest tests/test_vehiculo.py -v
docker-compose exec backend pytest tests/test_asignacion.py -v
```

---

## 4.9 — Fix: Manejo de OSRM Caído (Bug Mayor)

### Problema
Si OSRM está caído o tarda mucho, la asignación automática falla sin un fallback claro.

### Archivos a modificar
- `backend/app/infrastructure/osrm/osrm_client.py` → agregar timeout y manejo de errores
- `backend/app/infrastructure/osrm/proveedor_rutas.py` → fallback

### Solución paso a paso

**Paso 1:** Revisar `osrm_client.py`:

```bash
docker-compose exec backend cat app/infrastructure/osrm/osrm_client.py
```

**Paso 2:** Agregar timeout y retry al cliente OSRM:

```python
# En osrm_client.py, agregar timeout al httpx client:
import httpx

class OsrmClient:
    def __init__(self, base_url: str = "http://osrm:5000", timeout: float = 10.0):
        self._base_url = base_url
        self._timeout = timeout

    async def get_route(self, coordinates):
        async with httpx.AsyncClient(timeout=self._timeout) as client:
            try:
                response = await client.get(...)
                response.raise_for_status()
                return response.json()
            except (httpx.TimeoutException, httpx.HTTPStatusError) as e:
                # Fallback: calcular distancia euclidiana aproximada
                raise OsrmNoDisponibleError(f"OSRM no disponible: {e}")
```

**Paso 3:** Agregar excepción personalizada:

```python
# En backend/app/core/exceptions.py o crear archivo nuevo:
class OsrmNoDisponibleError(Exception):
    """Excepción cuando OSRM no está disponible."""
    pass
```

**Paso 4:** Manejar en el endpoint de asignación:

```python
# En asignaciones.py, línea ~100:
try:
    mejor = await asignador.asignar(lat, lng, disponibles)
except OsrmNoDisponibleError:
    raise HTTPException(
        status_code=503,
        detail="El servicio de rutas (OSRM) no está disponible. Intente más tarde."
    )
```

**Paso 5:** Verificar que OSRM está corriendo:

```bash
docker-compose exec osrm curl -s "http://localhost:5000/nearest/v1/driving/-103.5,19.2" | head -c 200
```

---

## 4.10 — Fix: WebSocket/PWA (Bug Mayor)

### Problema
- WebSocket no tiene reconexión con backoff exponencial
- MapaSupervision usa `localStorage` directo en vez de `authStorage`
- No hay limpieza de posiciones antiguas

### Archivos a modificar
- `frontend/src/pages/DashboardPipas.jsx` → reconexión WebSocket
- `frontend/src/pages/MapaSupervision.jsx` → usar `authStorage`
- `frontend/src/pages/AppChofer.jsx` → backoff exponencial
- `backend/app/api/v1/ubicacion_ws.py` → limpieza de posiciones

### Solución paso a paso

**Paso 1:** Corregir MapaSupervision para usar `authStorage`:

```javascript
// ANTES (MapaSupervision.jsx línea ~47):
const token = localStorage.getItem('access_token');

// DESPUÉS:
import { authStorage } from '../services/authService';
const token = authStorage.getAccessToken();
```

**Paso 2:** Agregar backoff exponencial a la reconexión del WebSocket en DashboardPipas:

```javascript
// ANTES (DashboardPipas.jsx línea ~100):
ws.onclose = () => setTimeout(conectar, 5000);

// DESPUÉS:
let intentos = 0;
const maxIntentos = 10;
ws.onclose = () => {
  intentos++;
  const delay = Math.min(1000 * Math.pow(2, intentos), 30000); // 1s, 2s, 4s... max 30s
  setTimeout(conectar, delay);
};
ws.onopen = () => { intentos = 0; }; // Reset al conectar
```

**Paso 3:** Agregar backoff exponencial a AppChofer:

```javascript
// ANTES (AppChofer.jsx línea ~120):
ws.onclose = () => { setConectado(false); setTimeout(conectar, 5000); };

// DESPUÉS:
let intentos = 0;
ws.onclose = () => {
  setConectado(false);
  intentos++;
  const delay = Math.min(1000 * Math.pow(2, intentos), 30000);
  setTimeout(conectar, delay);
};
ws.onopen = () => { setConectado(true); intentos = 0; drenarCola(ws); };
```

**Paso 4:** Agregar limpieza de posiciones antiguas en el backend:

```python
# En ubicacion_ws.py, después de cada received():
# Limitar a las últimas 100 posiciones por vehículo
import asyncio

async def limpiar_posiciones_antiguas(db, vhe_id, max_posiciones=100):
    """Mantener solo las últimas N posiciones por vehículo."""
    from sqlalchemy import delete
    from app.infrastructure.database.models import UbicacionPipaModel

    subq = (
        select(UbicacionPipaModel.ubp_id)
        .where(UbicacionPipaModel.ubp_vhe_id == vhe_id)
        .order_by(UbicacionPipaModel.ubp_timestamp.desc())
        .offset(max_posiciones)
        .scalar_subquery()
    )
    await db.execute(delete(UbicacionPipaModel).where(UbicacionPipaModel.ubp_id.in_(subq)))
```

**Paso 5:** Verificar:
```bash
docker-compose logs backend --tail=30 | grep -i "ws_"
docker-compose logs frontend --tail=30 | grep -i "websocket\|ws"
```

---

## 4.11 — Fix: CORS/Red y Permisos por Rol (Bug Mayor)

### Problema
- `ProtectedRoute` no valida roles, solo `isAuthenticated`
- No hay validación de permisos `solicitudes.gestionar` en el backend

### Archivos a modificar
- `backend/app/api/v1/solicitudes_pipa.py` → agregar check de permisos
- `frontend/src/components/common/ProtectedRoute.jsx` → agregar check de roles
- `docker-compose.yml` → verificar CORS

### Solución paso a paso

**Paso 1:** Agregar dependencia de permisos en el backend:

```python
# Crear archivo nuevo backend/app/api/deps_permisos.py:
from fastapi import Depends, HTTPException
from app.domain.auth.entities import User
from app.api.deps import get_current_user

def require_permission(permission: str):
    """Factory de dependencia que requiere un permiso específico."""
    async def _check(user: User = Depends(get_current_user)):
        if user.is_superuser:
            return user  # superadmin tiene todos los permisos
        user_permissions = []
        if user.role and user.role.permissions:
            user_permissions = [p.codename for p in user.role.permissions]
        if permission not in user_permissions:
            raise HTTPException(
                status_code=403,
                detail=f"Permiso requerido: {permission}"
            )
        return user
    return _check
```

**Paso 2:** Aplicar en el endpoint de cambio de estados:

```python
# En solicitudes_pipa.py, cambiar la dependencia:
from app.api.deps_permisos import require_permission

@router.patch("/{solicitud_id}/estado", response_model=SolicitudPipaResponse)
async def cambiar_estado(
    solicitud_id: int,
    body: SolicitudPipaCambioEstado,
    current_user=Depends(require_permission("solicitudes.gestionar")),
    # ...
):
```

**Paso 3:** Agregar test de permisos:

```python
# En backend/tests/test_permisos_solicitudes.py:
@pytest.mark.asyncio
async def test_usuario_sin_permiso_recibe_403(client: AsyncClient, auth_headers):
    """Un usuario sin solicitudes.gestionar recibe 403 al cambiar estado."""
    # Crear solicitud
    r = await client.post(PREFIX, headers=auth_headers, json=_body("LIC-PERM-403"))
    sid = r.json()["spp_id"]

    # Intentar cambiar estado (auth_headers tiene role="user" sin permisos)
    r = await client.patch(
        f"{PREFIX}/{sid}/estado",
        headers=auth_headers,
        json={"estado_nuevo": "En Ruta"},
    )
    assert r.status_code == 403
```

**Paso 4:** Verificar CORS en docker-compose:

```bash
docker-compose exec backend python -c "
from app.config import settings
print('CORS_ORIGINS:', settings.CORS_ORIGINS)
"
```

**Paso 5:** Verificar:
```bash
docker-compose exec backend pytest tests/test_permisos_solicitudes.py -v
docker-compose exec backend pytest tests/test_solicitud_pipa.py -v
```

---

## Tests de Regresión (Al Final del Día)

```bash
# Ejecutar TODOS los tests para verificar que los fixes no rompieron nada
docker-compose exec backend pytest tests/ -v --tb=short

# Tests específicos de los bugs corregidos
docker-compose exec backend pytest tests/test_solicitud_pipa.py -v
docker-compose exec backend pytest tests/test_permisos_solicitudes.py -v
docker-compose exec backend pytest tests/test_vehiculo.py -v
docker-compose exec backend pytest tests/test_asignacion.py -v
docker-compose exec backend pytest tests/test_websocket_ubicacion.py -v
```

---

## Comandos Útiles del Día 2

```bash
# Ver cambios en tiempo real
docker-compose logs -f backend --tail=20

# Reiniciar backend después de cambios
docker-compose restart backend

# Verificar que el frontend hot-reload funciona
# (modificar un archivo .jsx y verificar que se actualiza)

# Rollback si algo falla
git diff  # ver cambios
git checkout -- <archivo>  # deshacer cambio específico
```

---

## Entregable del Día 2

- [ ] Bug crítico de estados corregido y testeado
- [ ] Bug de doble asignación corregido
- [ ] Fallback de OSRM implementado
- [ ] Reconexión WebSocket con backoff exponencial
- [ ] MapaSupervision usa authStorage
- [ ] Permisos por rol implementados en backend
- [ ] Todos los tests de regresión pasan
- [ ] `documentacion/issues/sprint4-bugs.md` actualizado con fixes

---

**Tiempo estimado:** 6 horas (8:00 — 14:00)
