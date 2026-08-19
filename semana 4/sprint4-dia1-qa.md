# Sprint 4 — Día 1 (Lunes): QA Integral y Backlog de Bugs

**Objetivo:** Ejecutar QA funcional de todo el módulo de pipas, documentar bugs encontrados y priorizar el backlog.

---

## Prerrequisitos

```bash
# Levantar todo desde cero
docker-compose down -v
docker-compose up -d --build

# Verificar que todos los servicios están up
docker-compose ps

# Verificar que el backend responde
curl http://localhost:8000/api/v1/version

# Verificar que el frontend responde
curl -o /dev/null -w "%{http_code}" http://localhost:3000
```

---

## 4.1 — QA Funcional de Catálogos

### Vehículos (`/vehiculos`)

Abrir `http://localhost:3000/vehiculos` en el navegador.

**Checklist de prueba:**

| # | Acción | Resultado esperado | Bug si falla |
|---|--------|-------------------|--------------|
| 1 | Cargar la página | Tabla muestra vehículos del seed | |
| 2 | Click "Nuevo vehículo" | Modal se abre con campos vacíos | |
| 3 | Llenar formulario completo y guardar | Se cierra modal, vehículo aparece en tabla | |
| 4 | Intentar guardar con descripción duplicada | Error "La descripcion ya existe" | |
| 5 | Click editar en un vehículo | Modal se abre con datos precargados | |
| 6 | Cambiar combustible y guardar | Valor se actualiza en tabla | |
| 7 | Click en badge de estatus | Cambia entre Activo/Inactivo | |
| 8 | Click eliminar | Confirmación aparece, luego se elimina | |
| 9 | Buscar por texto | Filtra la tabla correctamente | |
| 10 | Verificar endpoint directamente: | `curl -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/vehiculos` retorna listado | |

**Archivos a revisar:**
- `frontend/src/pages/VehiculosPage.jsx`
- `backend/app/api/v1/vehiculos.py`
- `backend/app/infrastructure/repositories/vehiculo_repository.py`

### Marcas (`/vehiculomarcas`)

**Checklist de prueba:**

| # | Acción | Resultado esperado |
|---|--------|-------------------|
| 1 | Listar marcas | Muestra al menos 10 (seeds) |
| 2 | Crear marca duplicada | Error 409 |
| 3 | Editar marca | Cambio persiste |
| 4 | Eliminar marca | Se elimina correctamente |

### Pozos (`/pozos`)

**Checklist de prueba:**

| # | Acción | Resultado esperado |
|---|--------|-------------------|
| 1 | Listar pozos | Tabla se carga |
| 2 | Crear pozo nuevo | Aparece en tabla |
| 3 | Toggle "Solo disponibles" | Filtra por poz_estatus=true |
| 4 | Editar pozo | Cambios persisten |
| 5 | Eliminar pozo | Se elimina con confirmación |

**Archivos a revisar:**
- `frontend/src/pages/PozoPage.jsx`
- `backend/app/api/v1/pozos.py`
- `backend/app/api/schemas/pozos.py`

### Combustible (`/combustible`)

**Checklist de prueba:**

| # | Acción | Resultado esperado |
|---|--------|-------------------|
| 1 | Registrar gasto de combustible | POST exitoso, aparece en historial |
| 2 | Seleccionar vehículo en filtro | Historial se filtra |
| 3 | Intentar guardar sin campos | Validación del formulario |
| 4 | Verificar que km_final > km_inicio | **BUG si no valida** |

**Archivos a revisar:**
- `frontend/src/pages/CombustiblePage.jsx`
- `backend/app/api/v1/combustible.py`
- `backend/app/api/schemas/combustible.py`

---

## 4.2 — QA del Ciclo de Vida de Solicitudes

### Transiciones de estado

Abrir `http://localhost:3000/pipas` (Kanban).

**Prueba 1: Flujo normal**
1. Crear solicitud en `/solicitud-pipa` con coordenadas
2. Verificar que aparece en columna "Pendiente"
3. Click "Asignar" en la tarjeta → vehículo se asigna
4. Drag & drop a "En Ruta" → estado cambia
5. Drag & drop a "Entregada" → estado cambia
6. Verificar que desaparece del Kanban o va a columna "Entregada"

**Prueba 2: Cancelación**
1. Crear solicitud nueva
2. Click "Cancelar" → pide motivo
3. Ingresar motivo → estado cambia a "Cancelada"

**Prueba 3: Transiciones inválidas (BUG si no bloquea)**
```bash
# Crear solicitud y tratar de saltar de Pendiente a Entregada
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login \
  -d "username=admin&password=admin123" | python -c "import sys,json;print(json.load(sys.stdin)['access_token'])")

# Crear solicitud
SPP_ID=$(curl -s -X POST http://localhost:8000/api/v1/solicitudes_pipa \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"spp_con":1,"spp_srv":1,"spp_vhe_id":1,"spp_horaentrega":"2026-08-20T10:00:00","srv_licencia":"TEST"}' \
  | python -c "import sys,json;print(json.load(sys.stdin)['spp_id'])")

# Intentar salto inválido: Pendiente -> Entregada (debería fallar)
curl -s -X PATCH http://localhost:8000/api/v1/solicitudes_pipa/$SPP_ID/estado \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"estado_nuevo":"Entregada"}'
# Esperado: 400 "Transicion invalida"
```

**Prueba 4: Mismo estado (debería fallar)**
```bash
curl -s -X PATCH http://localhost:8000/api/v1/solicitudes_pipa/$SPP_ID/estado \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"estado_nuevo":"Pendiente"}'
# Esperado: 400 "La solicitud ya esta en ese estado"
```

**Archivos a revisar:**
- `backend/app/shared/enums.py` → `SolicitudEstado.es_valida()`
- `backend/app/api/v1/solicitudes_pipa.py` → `cambiar_estado()`
- `frontend/src/pages/DashboardPipas.jsx` → drag & drop
- `frontend/src/components/common/SolicitudCard.jsx` → botones de acción

---

## 4.3 — QA del Mapa en Vivo y PWA del Chofer

### Mapa de Supervisión (`/mapa`)

**Checklist de prueba:**

| # | Acción | Resultado esperado |
|---|--------|-------------------|
| 1 | Abrir `/mapa` | Mapa carga con solicitudes como marcadores |
| 2 | Abrir `/mapa` en segunda pestaña | Ambas muestran los mismos datos |
| 3 | Abrir consola del navegador | WebSocket se conecta sin errores |
| 4 | Desconectar y reconectar WebSocket | **BUG si no reconecta automáticamente** |
| 5 | Verificar marcadores de solicitudes | Colores según estado (📍 Pendiente, 🚗 En Ruta, ✅ Entregada) |

**Archivo:** `frontend/src/pages/MapaSupervision.jsx`

### PWA del Chofer (`/chofer`)

**Checklist de prueba:**

| # | Acción | Resultado esperado |
|---|--------|-------------------|
| 1 | Abrir `/chofer` | Formulario de login aparece |
| 2 | Login con credenciales válidas | Entra a la vista del chofer |
| 3 | Verificar GPS | Posición aparece en el mapa |
| 4 | Verificar WebSocket | Badge "Conectado" aparece |
| 5 | Simular pérdida de red (DevTools → Offline) | Badge cambia a "Sin red", posiciones se encolan |
| 6 | Restaurar red | Cola se envía, badge "Conectado" |
| 7 | Marcar "Iniciar ruta" | Estado cambia a "En Ruta" |
| 8 | Marcar "Marcar entregada" | Estado cambia a "Entregada" |
| 9 | Verificar que el Kanban de Sandra se actualiza | **BUG si no se actualiza en vivo** |

**Archivos a revisar:**
- `frontend/src/pages/AppChofer.jsx`
- `frontend/src/services/api.js` → `getWsUrl()`
- `backend/app/core/websocket_manager.py`
- `backend/app/api/v1/ubicacion_ws.py`

### Pruebas de WebSocket en Docker

```bash
# Verificar que el WebSocket funciona
docker-compose exec backend python -c "
from app.core.websocket_manager import manager
print(f'Conexiones activas: {manager.count}')
"

# Verificar logs del WebSocket
docker-compose logs backend --tail=50 | grep -i "ws_"
```

---

## 4.4 — QA de la PWA (Instalación, Offline, Batería)

### Instalación en Android

1. Abrir `http://localhost:3000/chofer` en Chrome Android
2. Click en "Agregar a pantalla de inicio"
3. Verificar que la app se instala con ícono correcto
4. Verificar que abre en modo standalone (sin barra de URL)

### Instalación en iOS

1. Abrir en Safari iOS
2. Click en compartir → "Agregar a pantalla de inicio"
3. Verificar que abre sin barra de Safari

### Modo Offline

1. Abrir la PWA
2. Activar modo offline en DevTools
3. Verificar que:
   - El mapa sigue mostrando la última posición conocida
   - Las posiciones GPS se encolan en localStorage
   - Al reconectar se envían las posiciones pendientes

### Batería con GPS activo

1. Activar GPS en el teléfono
2. Dejar la PWA abierta 10 minutos
3. Verificar que el GPS no consume batería excesiva
4. **BUG si el intervalo de envío es muy corto (< 10s)**

**Archivo:** `frontend/public/manifest.webmanifest`, `frontend/vite.config.js`

---

## 4.5 — QA de Reportes con Volúmenes de Datos Reales

### Reporte de Combustible (`/reportes-combustible`)

```bash
# Insertar registros de prueba de combustible
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login \
  -d "username=admin&password=admin123" | python -c "import sys,json;print(json.load(sys.stdin)['access_token'])")

for i in $(seq 1 20); do
  curl -s -X POST http://localhost:8000/api/v1/combustible \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"reg_com_vhe_id\":1,\"reg_com_lts\":$(( RANDOM % 200 + 50 )).50,\"reg_com_cost\":$(( RANDOM % 5000 + 1000 )).00,\"reg_com_km_inicio\":$(( 10000 + i * 100 )),\"reg_com_km_final\":$(( 10000 + i * 100 + 200 ))}" > /dev/null
done
echo "Registros insertados"
```

**Checklist de prueba:**

| # | Acción | Resultado esperado |
|---|--------|-------------------|
| 1 | Abrir `/reportes-combustible` | Tabla muestra datos |
| 2 | Filtrar por fecha | Resultados se reducen |
| 3 | Filtrar por vehículo | Solo muestra ese vehículo |
| 4 | Verificar totales | Litros y costo coinciden con filas |
| 5 | Exportar a CSV | Archivo se descarga |
| 6 | Imprimir | Vista de impresión aparece |

### Reporte de Tiempos (`/reportes-tiempos`)

**Checklist de prueba:**

| # | Acción | Resultado esperado |
|---|--------|-------------------|
| 1 | Abrir `/reportes-tiempos` | Tabla muestra días con actividad |
| 2 | Filtrar por rango de fechas | Resultados se filtran |
| 3 | Verificar coherencia | Entregas > 0 solo en días con solicitudes "Entregada" |
| 4 | Exportar CSV | Archivo correcto |

**Archivos a revisar:**
- `frontend/src/pages/ReportecombustiblePage.jsx`
- `frontend/src/pages/ReporteTiemposTrasladosPage.jsx`
- `backend/app/api/v1/reportes.py`
- `backend/app/api/schemas/reportes.py`

---

## 4.6 — Priorizar Bugs y Registrar

### Formato de registro de bugs

Crear archivo `documentacion/issues/sprint4-bugs.md` con el siguiente formato:

```markdown
# Sprint 4 — Backlog de Bugs

## Críticos (bloquean uso)
| # | Bug | Archivo | Pasos para reproducir | Esperado | Actual |
|---|-----|---------|----------------------|----------|--------|
| C1 | ... | ... | ... | ... | ... |

## Mayores (afectan experiencia)
| # | Bug | Archivo | Pasos para reproducir | Esperado | Actual |
|---|-----|---------|----------------------|----------|--------|
| M1 | ... | ... | ... | ... | ... |

## Menores (cosméticos)
| # | Bug | Archivo | Pasos para reproducir | Esperado | Actual |
|---|-----|---------|----------------------|----------|--------|
| m1 | ... | ... | ... | ... | ... |
```

### Bugs conocidos del análisis de código

Al revisar el código, estos bugs potenciales fueron identificados:

| Severidad | Bug | Archivo | Descripción |
|-----------|-----|---------|-------------|
| **Crítico** | `cambiar_estado` no retorna solicitud actualizada | `solicitudes_pipa.py:107` | Retorna `solicitud` antes de `registrar_transicion()`, el estado en la respuesta es el anterior |
| **Mayor** | Sin check de permisos `solicitudes.gestionar` | `solicitudes_pipa.py:82` | Cualquier usuario autenticado puede cambiar estados |
| **Mayor** | Asignación no verifica si vehículo ya tiene solicitud activa | `asignaciones.py:70` | Doble asignación posible |
| **Mayor** | WebSocket reconexión sin backoff exponencial | `DashboardPipas.jsx:100` | Fixed 5s timeout, no escalado |
| **Mayor** | MapaSupervision usa `localStorage` directo | `MapaSupervision.jsx:47` | No usa `authStorage`, inconsistente |
| **Mayor** | Sin validación km_final > km_inicio | `combustible.py` + `CombustiblePage.jsx` | Permite registros ilógicos |
| **Menor** | `PozoPage` requiere `poz_dom_id` sin FK validation | `pozos.py:40` | Acepta domicilios inexistentes |
| **Menor** | Archivo duplicado `ReposrteTiemposTranslados.jsx` | `frontend/src/pages/` | Typo en nombre, contenido duplicado |
| **Menor** | `ProtectedRoute` no valida roles | `ProtectedRoute.jsx` | Solo check `isAuthenticated` |

### Comandos para verificar bugs específicos

```bash
# Bug: cambiar_estado no retorna estado actualizado
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login \
  -d "username=admin&password=admin123" | python -c "import sys,json;print(json.load(sys.stdin)['access_token'])")

# Crear solicitud
SPP=$(curl -s -X POST http://localhost:8000/api/v1/solicitudes_pipa \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"spp_con":1,"spp_srv":1,"spp_vhe_id":1,"spp_horaentrega":"2026-08-20T10:00:00","srv_licencia":"BUG-TEST"}')
echo "Creada: $SPP"
SPP_ID=$(echo $SPP | python -c "import sys,json;print(json.load(sys.stdin)['spp_id'])")

# Cambiar estado y verificar qué retorna
RESULT=$(curl -s -X PATCH http://localhost:8000/api/v1/solicitudes_pipa/$SPP_ID/estado \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"estado_nuevo":"En Ruta"}')
echo "Respuesta del cambio: $RESULT"
# BUG: spp_estatus en la respuesta debería ser "En Ruta" pero puede ser "Pendiente"

# Bug: doble asignación
curl -s -X POST http://localhost:8000/api/v1/asignaciones/automatica \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"spp_id\":$SPP_ID}"
# Asignar el mismo vehículo a otra solicitud
# Si no hay control, el vehículo queda en dos solicitudes activas
```

---

## Comandos Útiles del Día 1

```bash
# Ver logs en tiempo real
docker-compose logs -f backend
docker-compose logs -f frontend

# Reset completo de la BD
docker-compose down -v
docker-compose up -d --build

# Ejecutar tests existentes para verificar que no hay regresiones
docker-compose exec backend pytest tests/ -v --tb=short

# Verificar estado de Docker
docker-compose ps

# Acceder al backend directamente
docker-compose exec backend bash
```

---

## Entregable del Día 1

- [ ] Archivo `documentacion/issues/sprint4-bugs.md` con todos los bugs documentados
- [ ] Bugs priorizados: críticos / mayores / menores
- [ ] Cada bug tiene pasos claros de reproducción
- [ ] Backlog listo para el día 2 (corrección de bugs críticos)

---

**Tiempo estimado:** 6 horas (8:00 — 14:00)
