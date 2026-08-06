# Día 5 — Pruebas Unitarias y Cierre (Sprint 1: módulo de pipas)

## 0. Prerrequisitos (verificar antes de empezar)

- [ ] Día 4 terminado: los 3 routers (`vehiculos`, `vehiculomarcas`, `solicitudes_pipa`) cargan y funcionan con token JWT.
- [ ] Rama de trabajo creada: `git checkout -b feature/pipas-tests` (desde `feature/pipas-semana1`).

**Convenciones del proyecto (respétalas):**
- Tests con `@pytest.mark.asyncio` y fixture `client` de `backend/tests/conftest.py`.
- Autenticación: `auth_headers` (token JWT real de user `id=1`, admin).
- Los tests corren contra **BD real** (convención actual del repo) → usar datos únicos y limpiar al final.
- No se llama `db.commit()` manualmente: `get_db_session` ya commitea al final de la request.

**Datos sembrados disponibles (para los asserts):**
- Vehículos: `vhe_id` 1-5, descripciones tipo `Pipa Nissan - Placa JCG-1201`.
- Marcas: `vho_id` 1-10 (`Nissan`, `Chevrolet`, ...).
- Solicitudes: `spp_id` 1-10 (`spp_id=1` en `Pendiente`).

---

## 1. Git — sincronizar y crear ramas

```bash
git checkout main
git pull origin main
git checkout -b feature/pipas-semana1
# (los cambios sin commitear se quedan en esta rama)
git add -A
git commit -m "feat: modulos de pipas (migraciones, modelos, schemas, endpoints)"
git checkout -b feature/pipas-tests
```

Al final del día: commit `test: ...`, push y PR de `feature/pipas-tests` → `feature/pipas-semana1`.

---

## 2. Tarea 5.1 — `backend/tests/test_vehiculos.py`

**Definición de Terminado:** CRUD de vehículos probado (listado paginado, disponibles, crear/duplicar, get, put, patch, delete) + 401 sin token.

```python
"""
DAPA2W Enterprise Platform - Vehiculos API Tests.
"""

import time

import pytest
from httpx import AsyncClient

PREFIX = "/api/v1/vehiculos"


@pytest.mark.asyncio
async def test_list_vehiculos_requires_auth(client: AsyncClient):
    assert (await client.get(PREFIX)).status_code == 401


@pytest.mark.asyncio
async def test_list_vehiculos(client: AsyncClient, auth_headers):
    r = await client.get(PREFIX, headers=auth_headers)
    assert r.status_code == 200
    assert r.json()["total"] >= 5


@pytest.mark.asyncio
async def test_get_disponibles(client: AsyncClient, auth_headers):
    r = await client.get(f"{PREFIX}/disponibles", headers=auth_headers)
    assert r.status_code == 200
    assert len(r.json()) >= 1


@pytest.mark.asyncio
async def test_get_vehiculo(client: AsyncClient, auth_headers):
    assert (await client.get(f"{PREFIX}/1", headers=auth_headers)).status_code == 200


@pytest.mark.asyncio
async def test_get_vehiculo_not_found(client: AsyncClient, auth_headers):
    assert (await client.get(f"{PREFIX}/999999", headers=auth_headers)).status_code == 404


@pytest.mark.asyncio
async def test_create_vehiculo_duplicate_descripcion(client: AsyncClient, auth_headers):
    r = await client.post(
        PREFIX,
        headers=auth_headers,
        json={
            "vhe_mar_id": 1,
            "vhe_res_id": 1,
            "vhe_modelo": 2018,
            "vhe_combustible": 75,
            "vhe_tipovehiculo": 1,
            "vhe_descripcion": "Pipa Nissan - Placa JCG-1201",  # ya existe (seed)
        },
    )
    assert r.status_code == 409


@pytest.mark.asyncio
async def test_create_update_delete_vehiculo(client: AsyncClient, auth_headers):
    body = {
        "vhe_mar_id": 1,
        "vhe_res_id": 1,
        "vhe_modelo": 2020,
        "vhe_combustible": 50,
        "vhe_tipovehiculo": 1,
        "vhe_descripcion": f"TEST-PIPA-{int(time.time())}",
    }
    r = await client.post(PREFIX, headers=auth_headers, json=body)
    assert r.status_code == 201
    vhe_id = r.json()["vhe_id"]

    r = await client.put(
        f"{PREFIX}/{vhe_id}", headers=auth_headers,
        json={"vhe_combustible": 80},
    )
    assert r.status_code == 200
    assert r.json()["vhe_combustible"] == 80

    r = await client.patch(
        f"{PREFIX}/{vhe_id}", headers=auth_headers,
        json={"vhe_estatus": False},
    )
    assert r.status_code == 200

    r = await client.delete(f"{PREFIX}/{vhe_id}", headers=auth_headers)
    assert r.status_code == 200
```

---

## 3. Tarea 5.2 — `backend/tests/test_vehiculomarcas.py`

**Definición de Terminado:** CRUD del catálogo de marcas probado + 401 sin token.

Mismo patrón con `PREFIX = "/api/v1/vehiculomarcas"`:

- `test_list_marcas_requires_auth` → 401.
- `test_list_marcas` → 200, `len(json) >= 10`.
- `test_get_marca` (`/1`) → 200; `test_get_marca_not_found` (`/999999`) → 404.
- `test_create_marca_duplicate` → `{"vho_nombremarca": "Nissan"}` → 409.
- `test_create_update_delete_marca` → nombre único `f"TEST-MARCA-{int(time.time())}"`, POST 201, PUT 200 (cambiar estatus), DELETE 200.

---

## 4. Tarea 5.3 — `backend/tests/test_solicitudes_pipa.py`

**Definición de Terminado:** alta ligada a vehículo, listado con filtros, transiciones válidas e inválidas + 401 sin token.

```python
"""
DAPA2W Enterprise Platform - Solicitudes Pipa API Tests.
"""

from datetime import datetime, timedelta

import pytest
from httpx import AsyncClient

PREFIX = "/api/v1/solicitudes_pipa"


def _body(licencia: str) -> dict:
    return {
        "spp_con": 1,
        "spp_srv": 1,
        "spp_vhe_id": 1,
        "spp_horaentrega": (datetime.now() + timedelta(hours=2)).isoformat(),
        "srv_licencia": licencia,
    }


@pytest.mark.asyncio
async def test_list_solicitudes_requires_auth(client: AsyncClient):
    assert (await client.get(PREFIX)).status_code == 401


@pytest.mark.asyncio
async def test_list_solicitudes(client: AsyncClient, auth_headers):
    r = await client.get(PREFIX, headers=auth_headers)
    assert r.status_code == 200
    assert r.json()["total"] >= 10


@pytest.mark.asyncio
async def test_list_solicitudes_filtro_estado(client: AsyncClient, auth_headers):
    r = await client.get(f"{PREFIX}?estado=Pendiente", headers=auth_headers)
    assert r.status_code == 200
    assert all(i["spp_estatus"] == "Pendiente" for i in r.json()["items"])


@pytest.mark.asyncio
async def test_create_solicitud_vehiculo_inexistente(client: AsyncClient, auth_headers):
    r = await client.post(
        PREFIX, headers=auth_headers, json={**_body("LIC-TST-0000"), "spp_vhe_id": 999999}
    )
    assert r.status_code == 400


@pytest.mark.asyncio
async def test_ciclo_transiciones(client: AsyncClient, auth_headers):
    r = await client.post(PREFIX, headers=auth_headers, json=_body("LIC-TST-0001"))
    assert r.status_code == 201
    sid = r.json()["spp_id"]

    r = await client.patch(
        f"{PREFIX}/{sid}/estado", headers=auth_headers, json={"estado_nuevo": "En Ruta"}
    )
    assert r.status_code == 200
    assert r.json()["spp_estatus"] == "En Ruta"

    r = await client.patch(
        f"{PREFIX}/{sid}/estado", headers=auth_headers, json={"estado_nuevo": "Entregada"}
    )
    assert r.status_code == 200
    assert r.json()["spp_estatus"] == "Entregada"


@pytest.mark.asyncio
async def test_transicion_invalida(client: AsyncClient, auth_headers):
    r = await client.post(PREFIX, headers=auth_headers, json=_body("LIC-TST-0002"))
    sid = r.json()["spp_id"]

    # Entregada -> Pendiente NO es valida
    r = await client.patch(
        f"{PREFIX}/{sid}/estado", headers=auth_headers, json={"estado_nuevo": "En Ruta"}
    )
    assert r.status_code == 200
    r = await client.patch(
        f"{PREFIX}/{sid}/estado", headers=auth_headers, json={"estado_nuevo": "Entregada"}
    )
    assert r.status_code == 200
    r = await client.patch(
        f"{PREFIX}/{sid}/estado", headers=auth_headers, json={"estado_nuevo": "Pendiente"}
    )
    assert r.status_code == 400


@pytest.mark.asyncio
async def test_mismo_estado_rechazado(client: AsyncClient, auth_headers):
    r = await client.post(PREFIX, headers=auth_headers, json=_body("LIC-TST-0003"))
    sid = r.json()["spp_id"]
    r = await client.patch(
        f"{PREFIX}/{sid}/estado", headers=auth_headers, json={"estado_nuevo": "Pendiente"}
    )
    assert r.status_code == 400


@pytest.mark.asyncio
async def test_estado_invalido(client: AsyncClient, auth_headers):
    r = await client.post(PREFIX, headers=auth_headers, json=_body("LIC-TST-0004"))
    sid = r.json()["spp_id"]
    r = await client.patch(
        f"{PREFIX}/{sid}/estado", headers=auth_headers, json={"estado_nuevo": "Fake"}
    )
    assert r.status_code == 422
```

**Nota:** no hay `DELETE` de solicitudes → las creadas en los tests quedan en la BD (igual que el `newuser` de la suite base). Limpieza al final:

```bash
docker compose exec db psql -U postgres -d dapatlqdb -c "DELETE FROM historial_solicitud WHERE histsol_spp_id IN (SELECT spp_id FROM solicitudpipas WHERE srv_licencia LIKE 'LIC-TST-%'); DELETE FROM solicitudpipas WHERE srv_licencia LIKE 'LIC-TST-%';"
```

---

## 5. Tarea 5.4 — Tests de autenticación (401)

Cubiertos dentro de cada archivo con los tests `*_requires_auth`:
- `GET /vehiculos`, `GET /vehiculomarcas`, `GET /solicitudes_pipa` sin token → 401.

---

## 6. Tarea 5.5 — `pytest` completo + pre-commit

```bash
docker compose exec backend pytest -q
```

- Esperado: tests nuevos en verde. Quedan **3 fallos pre-existentes de auth** (deuda de la app base): `test_roles.py::test_list_roles_requires_admin`, `test_users.py::test_list_users_requires_admin`, `test_users.py::test_create_user_requires_admin`. El DoD pide 100% verde → decidir: arreglarlos (mocking de `get_current_superuser`) o dejarlos documentados.
- `.pre-commit-config.yaml` está **vacío**. Opcional: agregar hooks mínimos:

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.0
    hooks:
      - id: ruff
      - id: ruff-format
```

---

## 7. Tarea 5.6 — Cierre y PR

```bash
git add backend/tests/
git commit -m "test: pruebas unitarias del modulo de pipas"
git push -u origin feature/pipas-tests
# PR feature/pipas-tests -> feature/pipas-semana1, y luego feature/pipas-semana1 -> main
```

**Verificación manual extra** (bitácora de transiciones):

```bash
docker compose exec db psql -U postgres -d dapatlqdb -c "SELECT * FROM historial_solicitud ORDER BY 1 DESC LIMIT 5;"
```

---

## Verificación final del día

- [ ] `test_vehiculos.py` en verde (5.1).
- [ ] `test_vehiculomarcas.py` en verde (5.2).
- [ ] `test_solicitudes_pipa.py` en verde (5.3, transiciones válidas e inválidas).
- [ ] 401 sin token en los 3 módulos (5.4).
- [ ] `pytest` completo sin regresiones (5.5).
- [ ] Pre-commit revisado/configurado (5.5).
- [ ] PR mergeado a `feature/pipas-semana1` con commits `feat:`/`test:` (5.6).

**Definición de Terminado del día:**
- [ ] Tests del módulo pipas en `backend/tests/` corriendo en verde.
- [ ] `pytest` sin regresiones en la suite existente.
- [ ] Commit: `test: pruebas unitarias del modulo de pipas`.
