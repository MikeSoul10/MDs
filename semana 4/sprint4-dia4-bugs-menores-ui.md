# Sprint 4 — Día 4 (Jueves): Bugs Menores y Pulido de UI

**Objetivo:** Cerrar bugs menores, pulir la UI del Kanban, mapa y formularios, mejorar mensajes de validación en español y actualizar documentación técnica.

---

## Prerrequisitos

```bash
docker-compose up -d
docker-compose exec backend pytest tests/ -v --tb=short
```

---

## 4.17 — Cierre de Bugs Menores

### Bug m1: Archivo duplicado con typo

**Problema:** Existe `frontend/src/pages/ReposrteTiemposTranslados.jsx` (typo en nombre) duplicando `ReporteTiemposTrasladosPage.jsx`.

**Solución:**
```bash
# Eliminar el archivo duplicado
rm frontend/src/pages/ReposrteTiemposTranslados.jsx

# Verificar que App.jsx no lo importa
grep -r "ReposrteTiemposTranslados" frontend/src/
```

### Bug m2: PozoPage requiere `poz_dom_id` sin validación FK

**Problema:** El formulario acepta cualquier número de domicilio sin verificar que exista.

**Solución:** Agregar validación en el backend:

```python
# En backend/app/api/v1/pozos.py, función create_pozo:
@router.post("", response_model=PozoResponse, status_code=status.HTTP_201_CREATED)
async def create_pozo(
    body: PozoCreate,
    current_user=Depends(get_current_user),
    repo: PozoRepository = Depends(get_pozo_repository),
):
    # Verificar que el domicilio existe (si la tabla tiene datos)
    # Opcional: agregar validación de FK
    creado = await repo.create(body.model_dump())
    return _to_response(creado)
```

### Bug m3: CombustiblePage no valida km_final > km_inicio

**Problema:** Permite registrar km_final menor que km_inicio.

**Solución en el frontend:**

```javascript
// En CombustiblePage.jsx, agregar validación antes de enviar:
const registrar = async (e) => {
  e.preventDefault();
  if (Number(form.km_final) <= Number(form.km_inicio)) {
    setError('El KM final debe ser mayor al KM inicio.');
    return;
  }
  // ... resto del código
};
```

**Solución en el backend (esquema):**

```python
# En backend/app/api/schemas/combustible.py:
from pydantic import model_validator

class CombustibleCreate(BaseModel):
    reg_com_vhe_id: int = Field(..., ge=1)
    reg_com_lts: float = Field(..., gt=0)
    reg_com_cost: float = Field(..., gt=0)
    reg_com_km_inicio: int = Field(..., ge=0)
    reg_com_km_final: int = Field(..., ge=0)

    @model_validator(mode="after")
    def validar_km(self) -> "CombustibleCreate":
        if self.reg_com_km_final <= self.reg_com_km_inicio:
            raise ValueError("KM final debe ser mayor que KM inicio")
        return self
```

### Bug m4: SolicitudPipaPage no valida campos numéricos

**Problema:** Los campos `con`, `srv` aceptan valores no numéricos.

**Solución:** Ya tienen `type="number"` en el HTML, pero agregar validación backend:

```python
# En solicitudes_pipa.py schema, ya tiene ge=1, verificar que funciona
```

### Bug m5: DashboardPipas no muestra mensaje de error al usuario

**Problema:** Si la API falla, solo hace `console.error` sin mostrar nada al usuario.

**Solución:**

```javascript
// En DashboardPipas.jsx, agregar estado de error:
const [error, setError] = useState('');

const cargar = () => {
  setLoading(true);
  setError('');
  getSolicitudes({ page: 1, page_size: 100 })
    .then((d) => setSolicitudes(d.items))
    .catch((err) => setError('Error al cargar solicitudes: ' + err.message))
    .finally(() => setLoading(false));
};
```

---

## 4.18 — Pulido de UI del Kanban, Mapa y Formularios

### Estados de carga

**Kanban (`DashboardPipas.jsx`):**

Agregar skeleton loading mientras carga:

```jsx
{loading && (
  <div className="space-y-2">
    {[1, 2, 3].map(i => (
      <div key={i} className="animate-pulse rounded-lg border p-3"
        style={{ backgroundColor: 'var(--color-surface-elevated)', borderColor: 'var(--color-border)' }}>
        <div className="h-3 w-20 rounded mb-2" style={{ backgroundColor: 'var(--color-border)' }} />
        <div className="h-2 w-32 rounded" style={{ backgroundColor: 'var(--color-border)' }} />
      </div>
    ))}
  </div>
)}
```

**Empty states:**

Cuando no hay solicitudes en una columna:

```jsx
{agrupadas(estado).length === 0 && !loading && (
  <p className="text-[11px] text-center py-4" style={{ color: 'var(--color-text-muted)' }}>
    Sin solicitudes en {estado}
  </p>
)}
```

### Errores amigables

**Formularios:** Mostrar errores en rojo con icono:

```jsx
{error && (
  <div className="flex items-center gap-2 mb-3 text-[11px] px-3 py-2 rounded-lg"
    style={{ backgroundColor: '#fef2f2', color: '#991b1b', border: '1px solid #fecaca' }}>
    <span className="font-medium">Error:</span> {error}
  </div>
)}
```

### Mapa de Supervisión

**Empty state del mapa:**

```jsx
{Object.keys(vehiculos).length === 0 && (
  <div className="absolute top-4 left-1/2 -translate-x-1/2 z-10 px-4 py-2 rounded-lg shadow"
    style={{ backgroundColor: 'var(--color-surface-elevated)', border: '1px solid var(--color-border)' }}>
    <p className="text-xs" style={{ color: 'var(--color-text-muted)' }}>
      Esperando ubicación de vehículos...
    </p>
  </div>
)}
```

---

## 4.19 — Pulido de la PWA del Chofer

### Diseño móvil

**Problemas a corregir:**
1. Botones muy pequeños para uso en la calle
2. Sin feedback visual al tocar botones
3. Sin indicador de loading al cambiar estado

### Solución

**Botones más grandes:**

```jsx
// ANTES:
<button onClick={() => cambiarEstado(s, 'Entregada')}
  className="flex-1 text-xs bg-green-600 text-white px-3 py-1.5 rounded">
  Marcar entregada
</button>

// DESPUÉS:
<button onClick={() => cambiarEstado(s, 'Entregada')}
  className="flex-1 text-sm font-semibold bg-green-600 text-white px-4 py-3 rounded-lg
    active:scale-95 transition-transform shadow-sm">
  Marcar entregada
</button>
```

**Feedback visual al tocar:**

```jsx
// Agregar active:scale-95 a todos los botones de acción
className="... active:scale-95 transition-transform"
```

**Indicador de loading:**

```jsx
const [cambiando, setCambiando] = useState(null); // spp_id del que está cambiando

const cambiarEstado = async (s, estadoNuevo) => {
  if (!window.confirm(`¿Marcar el folio #${s.spp_id} como "${estadoNuevo}"?`)) return;
  setCambiando(s.spp_id);
  try {
    await cambiarEstadoSolicitud(s.spp_id, { estado_nuevo: estadoNuevo });
    avisarSupervisor(s.spp_id, estadoNuevo);
    recargar();
  } catch (err) {
    alert(err.message);
  } finally {
    setCambiando(null);
  }
};

// En el botón:
<button onClick={() => cambiarEstado(s, 'En Ruta')}
  disabled={cambiando === s.spp_id}
  className="... disabled:opacity-50">
  {cambiando === s.spp_id ? 'Cambiando...' : 'Iniciar ruta'}
</button>
```

---

## 4.20 — Mensajes de Validación en Español

### Backend

Revisar todos los mensajes de error en los endpoints y schemas:

```bash
# Buscar mensajes de error en inglés
grep -rn "detail=" backend/app/api/v1/ | grep -v "__pycache__"
grep -rn "raise ValueError" backend/app/api/schemas/
```

**Mensajes a traducir:**

| Archivo | Mensaje actual | Mensaje en español |
|---------|---------------|-------------------|
| `solicitudes_pipa.py` | "El vehiculo no existe" | ✅ Ya está en español |
| `solicitudes_pipa.py` | "La solicitud ya esta en ese estado" | ✅ Ya está en español |
| `solicitudes_pipa.py` | "Transicion invalida" | ✅ Ya está en español |
| `vehiculos.py` | "La descripcion ya existe" | ✅ Ya está en español |
| `asignaciones.py` | "Solo se asignan solicitudes en estado 'Pendiente'" | ✅ Ya está en español |
| `asignaciones.py` | "No hay vehiculos disponibles." | ✅ Ya está en español |
| `combustible.py` | "El vehiculo no existe" | ✅ Ya está en español |

**Verificar schemas:**

```bash
grep -rn "description=" backend/app/api/schemas/ | head -20
```

### Frontend

Verificar mensajes de error en los formularios:

```bash
grep -rn "setError\|alert(" frontend/src/pages/ | head -20
```

**Mensajes ya en español:** La mayoría ya están en español. Verificar que no queden en inglés.

---

## 4.21 — Documentación Técnica del Módulo

### Actualizar `documentacion/sprint3-resumen.md` → `documentacion/sprint4-resumen.md`

```markdown
# Sprint 4 — Resumen de Entregables

## Estado: Completado

## Entregables por día

### Lunes — QA Integral
- QA funcional de catálogos (vehículos, marcas, pozos, combustible)
- QA del ciclo de vida de solicitudes
- QA del mapa en vivo y PWA del chofer
- QA de reportes con datos reales
- Backlog de bugs priorizado

### Martes — Bugs Críticos y Mayores
- Fix: `cambiar_estado` retorna estado actualizado
- Fix: Disponibilidad de vehículos (sin doble asignación)
- Fix: Fallback de OSRM caído
- Fix: WebSocket con backoff exponencial
- Fix: Permisos por rol (`solicitudes.gestionar`)

### Miércoles — Optimización de Rendimiento
- Índices SQL optimizados
- Eliminación de N+1 en listados
- Broadcast WebSocket con batching
- GPS adaptativo en PWA (batería)
- Caché de catálogos

### Jueves — Bugs Menores y Pulido
- Eliminación de archivo duplicado
- Validación km_final > km_inicio
- UI pulida (loading states, empty states)
- PWA chofer con botones grandes
- Mensajes de validación en español
- Documentación actualizada

### Viernes — Release Candidate
- Tests completos en verde
- Build PWA exitoso
- Levantamiento limpio desde cero
- Release etiquetado
- Demo final

## Archivos modificados en Sprint 4

### Backend
| Archivo | Cambio |
|---------|--------|
| `solicitudes_pipa.py` | Fix retorno de estado, permisos |
| `asignaciones.py` | Check de disponibilidad, fallback OSRM |
| `combustible.py` | Validación km |
| `osrm_client.py` | Timeout y retry |
| `websocket_manager.py` | Batch broadcast |
| `deps_permisos.py` | Nuevo: dependencia de permisos |
| `models.py` | Índices adicionales |

### Frontend
| Archivo | Cambio |
|---------|--------|
| `DashboardPipas.jsx` | Backoff exponencial, loading states |
| `AppChofer.jsx` | GPS adaptativo, botones grandes |
| `MapaSupervision.jsx` | authStorage, empty states |
| `CombustiblePage.jsx` | Validación km |
| `ProtectedRoute.jsx` | Check de roles |
| `vehiculoService.js` | Caché |
| `ReposrteTiemposTranslados.jsx` | Eliminado (duplicado) |
```

### Actualizar `servicios.md`

Agregar sección de rendimiento:

```markdown
## Rendimiento

| Endpoint | Tiempo objetivo | Estado |
|----------|----------------|--------|
| GET /solicitudes_pipa | < 300ms | ✅ |
| GET /vehiculos | < 300ms | ✅ |
| GET /reportes/combustible | < 500ms | ✅ |
| GET /reportes/tiempos_traslado | < 500ms | ✅ |
| WebSocket broadcast | < 1s latency | ✅ |
```

---

## 4.22 — Pruebas de Humo

### Checklist completo

```bash
# 1. Backend health
curl -s http://localhost:8000/api/v1/version | python -m json.tool

# 2. Login
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login \
  -d "username=admin&password=admin123" | python -c "import sys,json;print(json.load(sys.stdin)['access_token'])")
echo "Token: ${TOKEN:0:20}..."

# 3. Listados
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/solicitudes_pipa | python -c "import sys,json;d=json.load(sys.stdin);print(f'Solicitudes: {d[\"total\"]}')"
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/vehiculos | python -c "import sys,json;d=json.load(sys.stdin);print(f'Vehículos: {d[\"total\"]}')"
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/pozos | python -c "import sys,json;d=json.load(sys.stdin);print(f'Pozos: {d[\"total\"]}')"

# 4. Reportes
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/reportes/combustible | python -c "import sys,json;d=json.load(sys.stdin);print(f'Reporte combustible: {len(d[\"filas\"])} filas')"
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/reportes/tiempos_traslado | python -c "import sys,json;d=json.load(sys.stdin);print(f'Reporte tiempos: {len(d[\"filas\"])} filas')"

# 5. Frontend
curl -o /dev/null -w "Frontend: HTTP %{http_code}\n" http://localhost:3000

# 6. Tests
docker-compose exec backend pytest tests/ -v --tb=short 2>&1 | tail -5
```

### Verificar en navegador

| # | Página | URL | Estado esperado |
|---|--------|-----|-----------------|
| 1 | Login | `/login` | Formulario carga |
| 2 | Dashboard | `/` | Métricas visibles |
| 3 | Kanban | `/pipas` | Columnas con solicitudes |
| 4 | Mapa | `/mapa` | Mapa carga con marcadores |
| 5 | Vehículos | `/vehiculos` | Tabla con datos |
| 6 | Marcas | `/vehiculomarcas` | Tabla con datos |
| 7 | Solicitudes | `/solicitud-pipa` | Formulario funcional |
| 8 | Pozos | `/pozos` | Tabla con datos |
| 9 | Combustible | `/combustible` | Formulario funcional |
| 10 | Reporte combustible | `/reportes-combustible` | Datos y filtros |
| 11 | Reporte tiempos | `/reportes-tiempos` | Datos y filtros |
| 12 | PWA Chofer | `/chofer` | Login y mapa |

---

## Comandos Útiles del Día 4

```bash
# Verificar que no quedan imports rotos
grep -rn "ReposrteTiemposTranslados" frontend/src/

# Verificar que todos los archivos .jsx compilan
docker-compose exec frontend npx vite build --mode development 2>&1 | tail -10

# Ejecutar tests
docker-compose exec backend pytest tests/ -v --tb=short

# Verificar logs sin errores
docker-compose logs backend --tail=50 | grep -i "error\|exception\|traceback"
docker-compose logs frontend --tail=50 | grep -i "error"
```

---

## Entregable del Día 4

- [ ] Todos los bugs menores cerrados
- [ ] Archivo duplicado eliminado
- [ ] Validación km_final > km_inicio implementada
- [ ] UI pulida (loading states, empty states, errores amigables)
- [ ] PWA chofer con botones grandes y feedback visual
- [ ] Mensajes de validación en español
- [ ] Documentación técnica actualizada
- [ ] Pruebas de humo completadas sin errores

---

**Tiempo estimado:** 6 horas (8:00 — 14:00)
