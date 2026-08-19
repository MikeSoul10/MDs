# Sprint 4 — Día 3 (Miércoles): Optimización de Rendimiento

**Objetivo:** Optimizar consultas SQL, broadcast del WebSocket, consumo de batería de la PWA y tiempos de respuesta.

---

## Prerrequisitos

```bash
# Docker corriendo
docker-compose up -d

# Verificar que los tests del día 2 pasan
docker-compose exec backend pytest tests/ -v --tb=short
```

---

## 4.12 — Revisar Índices en Tablas Críticas

### Tablas a revisar
- `solicitudpipas` → búsquedas por `spp_estatus`, `spp_vhe_id`, `spp_horaentrega`
- `ubicacionespipa` → búsquedas por `ubp_vhe_id`, `ubp_timestamp`
- `registros_combustible` → búsquedas por `reg_com_vhe_id`, `reg_com_fecha`
- `historial_solicitud` → búsquedas por `histsol_spp_id`

### Verificar índices actuales

```bash
# Conectar a la BD y verificar índices
docker-compose exec db psql -U postgres -d dapatlqdb -c "
SELECT
  schemaname,
  tablename,
  indexname,
  indexdef
FROM pg_indexes
WHERE tablename IN ('solicitudpipas', 'ubicacionespipa', 'registros_combustible', 'historial_solicitud', 'asignaciones')
ORDER BY tablename, indexname;
"
```

### Índices que deberían existir (verificar en `models.py`):

| Tabla | Índice | Columnas | Justificación |
|-------|--------|----------|---------------|
| `solicitudpipas` | `ix_solicitudpipas_vhe_id` | `spp_vhe_id` | JOIN con vehiculos |
| `solicitudpipas` | `ix_solicitudpipas_estatus` | `spp_estatus` | Filtro por estado en Kanban |
| `ubicacionespipa` | `ix_ubicacionespipa_vhe_id` | `ubp_vhe_id` | Última ubicación por vehículo |
| `registros_combustible` | `ix_registros_combustible_vhe_id` | `reg_com_vhe_id` | Filtro por vehículo |
| `historial_solicitud` | `ix_historial_solicitud_spp_id` | `histsol_spp_id` | Historial de una solicitud |
| `asignaciones` | `ix_asignaciones_spp_id` | `asg_spp_id` | Asignación de solicitud |
| `asignaciones` | `ix_asignaciones_vhe_id` | `asg_vhe_id` | Asignación de vehículo |

### Agregar índices faltantes

Si falta algún índice, crear migración:

```bash
# Crear archivo de migración
cat > db/migrations/006_add_indexes.sql << 'EOF'
-- Índices para optimización de rendimiento Sprint 4

-- Índice compuesto para el reporte de combustible (fecha + vehículo)
CREATE INDEX IF NOT EXISTS ix_registros_combustible_fecha_vhe
  ON registros_combustible (reg_com_fecha, reg_com_vhe_id);

-- Índice para ubicaciones por timestamp (limpieza y reportes)
CREATE INDEX IF NOT EXISTS ix_ubicacionespipa_timestamp
  ON ubicacionespipa (ubp_timestamp DESC);

-- Índice compuesto para solicitudes por estado y fecha
CREATE INDEX IF NOT EXISTS ix_solicitudpipas_estatus_fecha
  ON solicitudpipas (spp_estatus, spp_horaentrega);

-- Índice para historial por solicitud y fecha
CREATE INDEX IF NOT EXISTS ix_historial_solicitud_spp_fecha
  ON historial_solicitud (histsol_spp_id, histsol_fecha DESC);

-- Índice para rutas por solicitud
CREATE INDEX IF NOT EXISTS ix_rutas_solicitud_spp_id
  ON rutas_solicitud (rtsol_spp_id);
EOF

# Aplicar migración
docker-compose exec db psql -U postgres -d dapatlqdb -f /docker-entrypoint-initdb.d/006_add_indexes.sql
```

### Verificar planes de consulta

```bash
# Verificar que PostgreSQL usa los índices
docker-compose exec db psql -U postgres -d dapatlqdb -c "
EXPLAIN ANALYZE
SELECT * FROM solicitudpipas WHERE spp_estatus = 'Pendiente' ORDER BY spp_horaentrega;
"

docker-compose exec db psql -U postgres -d dapatlqdb -c "
EXPLAIN ANALYZE
SELECT * FROM ubicacionespipa WHERE ubp_vhe_id = 1 ORDER BY ubp_timestamp DESC LIMIT 1;
"

docker-compose exec db psql -U postgres -d dapatlqdb -c "
EXPLAIN ANALYZE
SELECT * FROM registros_combustible WHERE reg_com_vhe_id = 1 AND reg_com_fecha >= '2026-01-01';
"
```

**Esperado:** Los planes deben usar `Index Scan` o `Bitmap Index Scan`, no `Seq Scan`.

---

## 4.13 — Optimizar Listado de Solicitudes (N+1)

### Problema potencial
El listado de solicitudes en el Kanban carga todos los items y luego busca el nombre del vehículo por cada uno (N+1).

### Archivo a revisar
- `frontend/src/pages/DashboardPipas.jsx` → useEffect de vehículos

### Solución

El código actual ya hace un fetch separado de vehículos y los mapea por ID:

```javascript
// DashboardPipas.jsx líneas ~45-52
useEffect(() => {
  getVehiculos({ page: 1, page_size: 100 })
    .then((d) => {
      const mapa = {};
      d.items.forEach((v) => { mapa[v.vhe_id] = v.vhe_descripcion; });
      setVehiculos(mapa);
    })
    .catch(() => {});
}, []);
```

**Optimización:** Agregar los nombres de vehículo directamente en la respuesta del backend:

```python
# En solicitudes_pipa.py, modificar el response para incluir vhe_descripcion:
class SolicitudPipaResponseConVehiculo(SolicitudPipaResponse):
    vhe_descripcion: Optional[str] = None

# O usar eager loading en el repositorio:
async def listar_filtrado(self, estado, fecha_desde, fecha_hasta, skip, limit):
    stmt = (
        select(SolicitudPipaModel)
        .options(selectinload(SolicitudPipaModel.vehiculo))  # ← Eager load
        .where(...)
    )
```

### Verificar que no hay N+1 en el backend

```bash
# Activar log de queries SQL
docker-compose exec backend python -c "
import logging
logging.getLogger('sqlalchemy.engine').setLevel(logging.INFO)
print('SQL logging activado - revisar logs')
"

# Hacer una petición y ver las queries
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/solicitudes_pipa | python -m json.tool | head -5
```

---

## 4.14 — Optimizar Broadcast del WebSocket

### Problema
Cada update de posición envía un mensaje individual a todos los clientes. Con 10 vehículos enviando cada 10s, son 60 mensajes/min.

### Archivo a modificar
- `backend/app/api/v1/ubicacion_ws.py` → batching de posiciones
- `backend/app/core/websocket_manager.py` → broadcast en lote

### Solución

**Paso 1:** Agregar cola de mensajes en el WebSocket manager:

```python
# En websocket_manager.py, agregar:
class ConnectionManager:
    def __init__(self):
        self._conexiones: Set[WebSocket] = set()
        self._lock = asyncio.Lock()
        self._cola: List[Dict[str, Any]] = []
        self._batch_task: Optional[asyncio.Task] = None

    async def enqueue(self, mensaje: Dict[str, Any]) -> None:
        """Agregar mensaje a la cola para batch broadcast."""
        async with self._lock:
            self._cola.append(mensaje)
        if self._batch_task is None or self._batch_task.done():
            self._batch_task = asyncio.create_task(self._flush_batch())

    async def _flush_batch(self) -> None:
        """Esperar 1s y enviar todos los mensajes acumulados."""
        await asyncio.sleep(1.0)
        async with self._lock:
            mensajes = self._cola.copy()
            self._cola.clear()
        if mensajes:
            # Enviar como batch
            batch = {"type": "batch_ubicaciones", "items": mensajes}
            await self.broadcast(batch)
```

**Paso 2:** Modificar el endpoint WebSocket para usar la cola:

```python
# En ubicacion_ws.py:
@websocket.websocket("/ws/ubicacion")
async def ws_ubicacion(websocket: WebSocket, token: str = Query(...)):
    # ... conexión y validación ...
    while True:
        data = await websocket.receive_json()
        if data.get("type") == "ubicacion" or "lat" in data:
            # Usar cola en vez de broadcast directo
            await manager.enqueue({
                "type": "ubicacion",
                "vhe_id": data["vhe_id"],
                "lat": data["lat"],
                "lng": data["lng"],
            })
```

**Paso 3:** Actualizar el frontend para manejar batches:

```javascript
// En DashboardPipas.jsx y MapaSupervision.jsx:
ws.onmessage = (ev) => {
  const msg = JSON.parse(ev.data);
  if (msg.type === 'batch_ubicaciones') {
    msg.items.forEach(item => {
      setVehiculos(prev => ({ ...prev, [item.vhe_id]: { lat: item.lat, lng: item.lng } }));
    });
  } else if (msg.type === 'ubicacion') {
    setVehiculos(prev => ({ ...prev, [msg.vhe_id]: { lat: msg.lat, lng: msg.lng } }));
  }
};
```

---

## 4.15 — Optimizar Batería de la PWA (GPS)

### Problema
La PWA envía posición GPS cada 10 segundos sin importar si el vehículo se está moviendo o no.

### Archivo a modificar
- `frontend/src/pages/AppChofer.jsx` → intervalo adaptativo de GPS

### Solución

**Paso 1:** Agregar detección de movimiento:

```javascript
// En AppChofer.jsx, reemplazar el useEffect de envío (línea ~140):
useEffect(() => {
  if (!token) return;

  let ultimoEnvio = null;
  let ultimaPosicion = null;
  const VELOCIDAD_UMBRAL = 0.5; // m/s (~1.8 km/h) - si se mueve más rápido, enviar cada 5s
  const INTERVALO_MOVIMIENTO = 5000;  // 5s cuando se mueve
  const INTERVALO_REPOSO = 30000;     // 30s cuando está quieto

  const timer = setInterval(() => {
    const p = posRef.current || pos;
    if (!p) return;

    const msg = { vhe_id: vheId, lat: p.lat, lng: p.lng, ts: new Date().toISOString() };
    const ws = wsRef.current;

    // Determinar si se está moviendo
    let enMovimiento = true;
    if (ultimaPosicion) {
      const dx = (p.lat - ultimaPosicion.lat) * 111000; // metros
      const dy = (p.lng - ultimaPosicion.lng) * 111000 * Math.cos(p.lat * Math.PI / 180);
      const dist = Math.sqrt(dx * dx + dy * dy);
      enMovimiento = dist > 5; // más de 5 metros de desplazamiento
    }

    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(msg));
    } else {
      encolarPosicion(msg);
    }

    ultimaPosicion = p;
  }, enMovimiento ? INTERVALO_MOVIMIENTO : INTERVALO_REPOSO);

  return () => clearInterval(timer);
}, [token, vheId, pos]);
```

**Paso 2:** Agregar pausa automática cuando el GPS no cambia:

```javascript
// Detectar si el GPS está devolviendo la misma posición
const posSinCambio = useRef(0);

// En el watchPosition:
watchPosition(
  (p) => {
    const nueva = { lat: p.coords.latitude, lng: p.coords.longitude };
    if (ultimaPosicion &&
        Math.abs(nueva.lat - ultimaPosicion.lat) < 0.0001 &&
        Math.abs(nueva.lng - ultimaPosicion.lng) < 0.0001) {
      posSinCambio.current++;
    } else {
      posSinCambio.current = 0;
    }
    ultimaPosicion = nueva;
    posRef.current = nueva;
    setPos(nueva);
  },
  () => {},
  { enableHighAccuracy: true, timeout: 10000, maximumAge: 5000 },
);
```

---

## 4.16 — Cachear Catálogos de Baja Volatilidad

### Archivos a modificar
- `frontend/src/services/vehiculoService.js` → caché de vehículos
- `frontend/src/services/pozoService.js` → caché de pozos
- `frontend/src/services/vehiculoMarcaService.js` → caché de marcas

### Solución

**Paso 1:** Crear un utilitario de caché simple:

```javascript
// Crear frontend/src/utils/cache.js:
const cache = new Map();

export function getCached(key, fetchFn, ttlMs = 60000) {
  const entry = cache.get(key);
  if (entry && Date.now() - entry.ts < ttlMs) {
    return Promise.resolve(entry.data);
  }
  return fetchFn().then(data => {
    cache.set(key, { data, ts: Date.now() });
    return data;
  });
}

export function invalidateCache(key) {
  cache.delete(key);
}
```

**Paso 2:** Aplicar en los servicios:

```javascript
// En vehiculoService.js:
import { getCached } from '../utils/cache';

export function getVehiculos(params = {}) {
  const qs = new URLSearchParams(params).toString();
  const key = `vehiculos_${qs}`;
  return getCached(key, () => api.get(`/vehiculos${qs ? `?${qs}` : ''}`), 120000); // 2 min TTL
}
```

### Verificar tiempos de respuesta

```bash
# Medir tiempo de respuesta del listado de vehículos
time curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/vehiculos > /dev/null

# Medir tiempo del Kanban
time curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/solicitudes_pipa > /dev/null

# Medir tiempo de reportes
time curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/reportes/combustible > /dev/null
```

**Esperado:** Listados < 300ms.

---

## Tests de Rendimiento

```bash
# Ejecutar todos los tests para verificar que las optimizaciones no rompieron nada
docker-compose exec backend pytest tests/ -v --tb=short

# Verificar que los endpoints de reportes siguen funcionando
docker-compose exec backend pytest tests/test_reportes_combustible.py -v
docker-compose exec backend pytest tests/test_reportes_tiempos.py -v
```

---

## Comandos Útiles del Día 3

```bash
# Medir tiempos de respuesta
time curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/solicitudes_pipa > /dev/null

# Ver queries SQL lentas
docker-compose exec db psql -U postgres -d dapatlqdb -c "
SELECT query, mean_time, calls
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;
"

# Verificar índices
docker-compose exec db psql -U postgres -d dapatlqdb -c "
SELECT relname, indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan DESC;
"

# Reiniciar backend después de cambios
docker-compose restart backend
```

---

## Entregable del Día 3

- [ ] Índices SQL verificados y faltantes agregados
- [ ] Planes de consulta optimizados (Index Scan)
- [ ] N+1 eliminado en listado de solicitudes
- [ ] Broadcast WebSocket con batching
- [ ] GPS adaptativo en PWA (batería optimizada)
- [ ] Caché de catálogos implementado
- [ ] Tiempos de respuesta < 300ms en listados
- [ ] Todos los tests pasan
- [ ] `documentacion/issues/sprint4-bugs.md` actualizado

---

**Tiempo estimado:** 6 horas (8:00 — 14:00)
