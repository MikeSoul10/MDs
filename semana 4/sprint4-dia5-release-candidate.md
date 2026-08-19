# Sprint 4 — Día 5 (Viernes): Release Candidate y Demo Final

**Objetivo:** Cerrar el sprint con todos los tests en verde, build de PWA exitoso, levantamiento limpio desde cero, release etiquetado y demo funcional.

---

## Prerrequisitos

```bash
# Docker corriendo
docker-compose up -d

# Verificar que no hay errores en logs
docker-compose logs backend --tail=20 | grep -i "error\|exception"
docker-compose logs frontend --tail=20 | grep -i "error"
```

---

## 4.23 — Correr pytest + npm run build + pre-commit

### Paso 1: pytest completo

```bash
docker-compose exec backend pytest tests/ -v --tb=short
```

**Resultado esperado:** Todos los tests en verde.

Si algún test falla:
```bash
# Ver el traceback completo
docker-compose exec backend pytest tests/test_fallido.py -v --tb=long

# Ejecutar solo el test fallido para depurar
docker-compose exec backend pytest tests/test_fallido.py::test_nombre -v -s
```

### Paso 2: Build de la PWA

```bash
docker-compose exec frontend npm run build
```

**Resultado esperado:** Build exitoso sin errores. Se genera `dist/` con service worker.

Verificar:
```bash
docker-compose exec frontend ls -la dist/
docker-compose exec frontend grep -l "workbox" dist/assets/*.js 2>/dev/null && echo "SW integrado"
```

### Paso 3: Verificar pre-commit

```bash
# El archivo .pre-commit-config.yaml está vacío en este proyecto
# Verificar que no hay archivos sensibles
git status
git diff --cached

# Verificar que no hay secrets
grep -rn "password\|secret\|token" .env.example
```

### Resumen de tests esperados

| Archivo | Tests | Estado |
|---------|-------|--------|
| `test_health.py` | 3 | ✅ |
| `test_auth.py` | 5 | ✅ |
| `test_users.py` | 4 | ✅ |
| `test_roles.py` | 3 | ✅ |
| `test_files.py` | 4 | ✅ |
| `test_chat.py` | 5 | ✅ |
| `test_disponibilidad.py` | 3 | ✅ |
| `test_asignacion.py` | 5 | ✅ |
| `test_agrupacion_geografica.py` | 4 | ✅ |
| `test_solicitud_pipa.py` | 9 | ✅ (1 nuevo: retorno actualizado) |
| `test_vehiculo.py` | 7 | ✅ |
| `test_vehiculomarcas.py` | 6 | ✅ |
| `test_websocket_ubicacion.py` | 5 | ✅ |
| `test_reportes_combustible.py` | 5 | ✅ |
| `test_reportes_tiempos.py` | 4 | ✅ |
| `test_flujo_completo.py` | 2 | ✅ |
| `test_permisos_solicitudes.py` | 3 | ✅ |
| **TOTAL** | **~82** | |

---

## 4.24 — Verificar Levantamiento Limpio desde Cero

### Paso 1: Down completo

```bash
docker-compose down -v
```

**Verificar que se eliminaron volúmenes:**
```bash
docker volume ls | grep dapa
# No debe aparecer ningún volumen dapa
```

### Paso 2: Up desde cero

```bash
docker-compose up -d --build
```

**Verificar cada servicio:**
```bash
# Esperar a que todo esté up (puede tomar 1-2 minutos)
docker-compose ps

# Verificar que db-migrate terminó
docker-compose logs db-migrate | tail -5

# Verificar que el backend responde
sleep 10
curl -s http://localhost:8000/api/v1/version

# Verificar que el frontend responde
curl -o /dev/null -w "Frontend: HTTP %{http_code}\n" http://localhost:3000

# Verificar que pgAdmin responde
curl -o /dev/null -w "pgAdmin: HTTP %{http_code}\n" http://localhost:5051
```

### Paso 3: Verificar que los seeds se aplicaron

```bash
docker-compose exec db psql -U postgres -d dapatlqdb -c "
SELECT 'vehiculos' as tabla, count(*) FROM vehiculos
UNION ALL
SELECT 'solicitudes', count(*) FROM solicitudpipas
UNION ALL
SELECT 'pozos', count(*) FROM pozos
UNION ALL
SELECT 'usuarios', count(*) FROM usuarios
UNION ALL
SELECT 'roles', count(*) FROM roles;
"
```

### Paso 4: Login funcional

```bash
# Login con admin
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login \
  -d "username=admin&password=admin123" | python -c "import sys,json;print(json.load(sys.stdin)['access_token'])")
echo "Login OK: ${TOKEN:0:20}..."
```

### Paso 5: Tests pasan con DB limpia

```bash
docker-compose exec backend pytest tests/ -v --tb=short
```

---

## 4.25 — Revisión Final de Seguridad

### Checklist de seguridad

| # | Área | Verificación | Comando |
|---|------|-------------|---------|
| 1 | JWT | Tokens expiran correctamente | Verificar `ACCESS_TOKEN_EXPIRE_MINUTES` en `.env` |
| 2 | JWT | Refresh token rotation funciona | `curl -X POST /api/v1/auth/refresh` |
| 3 | Permisos | Roles sin permiso reciben 403 | `curl -H "Authorization: Bearer $USER_TOKEN" -X PATCH /solicitudes_pipa/1/estado` |
| 4 | Validación | Inputs inválidos reciben 422 | `curl -X POST /solicitudes_pipa -d '{}'` |
| 5 | WebSocket | Conexión sin token rechazada | Verificar en logs del backend |
| 6 | CORS | Orígenes no permitidos rechazados | Verificar `CORS_ORIGINS` en `.env` |
| 7 | Passwords | No se guardan en texto plano | Verificar hash en tabla `usuarios` |
| 8 | Secrets | No hay secrets en el código | `grep -rn "P4ssw0rd" --include="*.py" --include="*.js" .` |

### Verificar JWT

```bash
# Token debe expirar en 30 minutos (default)
docker-compose exec backend python -c "
from app.config import settings
print(f'ACCESS_TOKEN_EXPIRE_MINUTES: {settings.ACCESS_TOKEN_EXPIRE_MINUTES}')
print(f'ALGORITHM: {settings.ALGORITHM}')
print(f'SECRET_KEY length: {len(settings.SECRET_KEY)}')
"
```

### Verificar permisos

```bash
# Crear usuario sin permisos
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login \
  -d "username=admin&password=admin123" | python -c "import sys,json;print(json.load(sys.stdin)['access_token'])")

# Intentar cambiar estado sin permiso (debería fallar si el permiso está implementado)
curl -s -X PATCH http://localhost:8000/api/v1/solicitudes_pipa/1/estado \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"estado_nuevo":"En Ruta"}' | python -m json.tool
```

### Verificar que no hay secrets en el código

```bash
# Buscar passwords hardcodeadas
grep -rn "P4ssw0rd\|password.*=.*['\"]" --include="*.py" --include="*.js" --include="*.jsx" . | grep -v node_modules | grep -v .git

# Buscar tokens hardcodeados
grep -rn "Bearer.*eyJ" --include="*.py" --include="*.js" . | grep -v node_modules
```

---

## 4.26 — Etiquetar Release y Merge a Main

### Paso 1: Verificar estado del repo

```bash
git status
git log --oneline -10
```

### Paso 2: Crear branch si no existe

```bash
git checkout -b fix/pipas-sprint4
```

### Paso 3: Agregar todos los cambios

```bash
# Agregar todo
git add -A

# Verificar qué se va a commitear
git status

# Commitear
git commit -m "fix(sprint4): QA, corrección de bugs, optimización y pulido

- Fix: cambiar_estado retorna estado actualizado
- Fix: disponibilidad de vehículos (sin doble asignación)
- Fix: fallback de OSRM caído
- Fix: WebSocket con backoff exponencial
- Fix: permisos por rol (solicitudes.gestionar)
- Opt: índices SQL para tablas críticas
- Opt: batch broadcast WebSocket
- Opt: GPS adaptativo en PWA (batería)
- Opt: caché de catálogos
- UI: loading states, empty states, errores amigables
- UI: PWA chofer con botones grandes
- Val: km_final > km_inicio, mensajes en español
- Docs: sprint4-resumen.md, servicios.md actualizado"
```

### Paso 4: Push

```bash
git push origin fix/pipas-sprint4
```

### Paso 5: Crear PR

```bash
gh pr create \
  --base main \
  --head fix/pipas-sprint4 \
  --title "Sprint 4: QA, Corrección de Bugs y Release Candidate" \
  --body "## Resumen
- QA integral del módulo de pipas
- Corrección de bugs críticos y mayores
- Optimización de rendimiento (índices, WebSocket, GPS)
- Pulido de UI y PWA
- Release candidate listo para demo

## Tests
- 82 tests en verde
- Build PWA exitoso
- Levantamiento limpio desde cero verificado"
```

### Paso 6: Etiquetar release

```bash
# Después de que el PR sea aprobado y mergeado
git checkout main
git pull origin main
git tag -a v0.1.0 -m "Sprint 4: Release Candidate - Módulo de Pipas"
git push origin v0.1.0
```

---

## 4.27 — Demo Final del Módulo de Pipas

### Preparación de la demo

**Ambiente:**
```bash
docker-compose down -v && docker-compose up -d --build
# Esperar 2 minutos a que todo esté listo
```

**Datos de prueba:**
```bash
# Asegurar que hay solicitudes, vehículos y pozos
docker-compose exec backend pytest tests/ -v --tb=short
```

### Guion de la demo

#### Parte 1: Panel de Sandra (Kanban) — 5 min

1. **Login** → `http://localhost:3000/login` → usuario admin
2. **Dashboard** → Ver métricas principales
3. **Kanban** → `http://localhost:3000/pipas`
   - Mostrar las 4 columnas: Pendiente / En Ruta / Entregada / Cancelada
   - Crear solicitud nueva en `/solicitud-pipa`
   - Volver al Kanban y ver que apareció
   - Asignar vehículo (click "Asignar")
   - Drag & drop a "En Ruta"
   - Verificar que el badge del contador cambió
4. **Filtros** → Filtrar por fecha y vehículo

#### Parte 2: Catálogos — 3 min

5. **Vehículos** → `/vehiculos` → CRUD rápido
6. **Marcas** → `/vehiculomarcas` → Listar
7. **Pozos** → `/pozos` → Crear uno nuevo
8. **Combustible** → `/combustible` → Registrar gasto

#### Parte 3: Mapa en Vivo — 3 min

9. **Mapa** → `http://localhost:3000/mapa`
   - Verificar que muestra marcadores de solicitudes
   - Verificar que se actualiza en tiempo real (abrir PWA del chofer)

#### Parte 4: PWA del Chofer — 5 min

10. **PWA Chofer** → `http://localhost:3000/chofer`
    - Login como chofer
    - Ver mapa con posición GPS
    - Ver entregas asignadas
    - Click "Iniciar ruta" → estado cambia a "En Ruta"
    - Volver al Kanban de Sandra → verificar que se actualizó
    - Click "Marcar entregada" → estado cambia a "Entregada"
    - Volver al Kanban → verificar

#### Parte 5: Reportes — 3 min

11. **Reporte combustible** → `/reportes-combustible`
    - Filtrar por fecha
    - Exportar CSV
12. **Reporte tiempos** → `/reportes-tiempos`
    - Ver entregas por día
    - Ver km recorridos

#### Parte 6: Tests — 2 min

13. **Ejecutar tests en vivo:**
    ```bash
    docker-compose exec backend pytest tests/ -v --tb=short
    ```
    - Mostrar todos los tests en verde

### Checklist de la demo

```
□ Login funciona
□ Dashboard muestra métricas
□ Kanban carga con 4 columnas
□ Crear solicitud aparece en Kanban
□ Asignar vehículo funciona
□ Drag & drop cambia estado
□ Filtros funcionan
□ CRUD vehículos funciona
□ CRUD marcas funciona
□ CRUD pozos funciona
□ Registrar combustible funciona
║ Mapa muestra marcadores
□ Mapa actualiza en tiempo real
□ PWA chofer carga
□ PWA chofer muestra mapa
□ PWA chofer cambiar estado funciona
□ Kanban se actualiza al cambiar estado desde PWA
□ Reporte combustible muestra datos
□ Reporte tiempos muestra datos
□ Exportar CSV funciona
□ Tests pasan en vivo
□ Build PWA exitoso
□ Levantamiento limpio desde cero
```

---

## Comandos Rápidos de Referencia

```bash
# === DIA 5: Comandos esenciales ===

# 1. Tests completos
docker-compose exec backend pytest tests/ -v --tb=short

# 2. Build PWA
docker-compose exec frontend npm run build

# 3. Levantamiento limpio
docker-compose down -v && docker-compose up -d --build

# 4. Verificar servicios
docker-compose ps
curl -s http://localhost:8000/api/v1/version
curl -o /dev/null -w "%{http_code}" http://localhost:3000

# 5. Git
git status
git add .
git commit -m "fix(sprint4): release candidate"
git push origin fix/pipas-sprint4

# 6. Demo
docker-compose down -v && docker-compose up -d --build
# Esperar 2 min
docker-compose exec backend pytest tests/ -v --tb=short
```

---

## Solución de Problemas Comunes

### Tests fallan después de los cambios del sprint
```bash
# Verificar si es problema de la BD
docker-compose down -v && docker-compose up -d --build
docker-compose exec backend pytest tests/ -v --tb=short
```

### Build de PWA falla
```bash
docker-compose exec frontend rm -rf node_modules dist
docker-compose exec frontend npm install
docker-compose exec frontend npm run build
```

### Docker no levanta desde cero
```bash
# Verificar que no hay contenedores huérfanos
docker-compose down -v
docker system prune -f
docker-compose up -d --build
```

### WebSocket no conecta
```bash
# Verificar que el backend está corriendo
docker-compose logs backend --tail=20 | grep -i "ws\|websocket"
```

---

## Entregable del Día 5

- [ ] 82+ tests en verde
- [ ] Build PWA exitoso con service worker
- [ ] Levantamiento limpio desde cero verificado
- [ ] Revisión de seguridad completada
- [ ] PR creado y mergeado a main
- [ ] Release etiquetado (v0.1.0)
- [ ] Demo funcional grabada/lista
- [ ] Documentación actualizada

---

**Tiempo estimado:** 6 horas (8:00 — 14:00)
