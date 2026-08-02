# Sprint 2 — Semana 2: Lógica de Negocio + Mapa en Vivo

- **Duración:** 5 días hábiles
- **Objetivo:** Agregar el motor de rutas (OSRM), la asignación automática de vehículos, el WebSocket de ubicación en tiempo real y la vista de mapa de supervisión.
- **Branches:** `feature/pipas-rutas`, `feature/pipas-websocket`, `feature/pipas-mapa`.
- **Dependencia:** Semana 1 terminada (API de `vehiculos`, `vehiculomarcas` y `solicitudpipas` en `/api/v1`).

---

## Lunes — Motor de Rutas OSRM

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 2.1 | Agregar servicio `osrm` al `docker-compose.yml` en la red `dapa-network` (imagen `ghcr.io/project-osrm/osrm-backend`, puerto 5000, volumen de datos del mapa) | `docker-compose.yml` | `docker compose up` levanta OSRM junto a los demás servicios |
| 2.2 | Preparar datos de mapa de la delegación (descargar extract OSM o definir área de trabajo) y cargar el perfil `car.lua` | documento de despliegue / `documentacion/` | `curl http://osrm:5000/route/v1/driving/...` responde |
| 2.3 | Crear cliente HTTP `OsrmClient` en infraestructura (usa `httpx`, ya en `requirements.txt`) | `backend/app/infrastructure/` | Clase con `get_route()`, `get_nearest()` y timeout configurable |
| 2.4 | Crear migración `00056_crea_tabla_rutas_solicitud.sql`: ruta calculada por solicitud (origen, destino, distancia km, duración, geometría GeoJSON) | `db/migrations/` | Migración ejecutable y reversible |
| 2.5 | Crear `RutaRepository` + modelo `RutaSolicitudModel` | `backend/app/infrastructure/` | CRUD funcional |

---

## Martes — Agrupación Geográfica y Disponibilidad

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 2.6 | Crear regla de negocio en `app/domain/`: agrupar solicitudes pendientes por cercanía geográfica (misma colonia / radio configurable) usando las coordenadas del domicilio | `backend/app/domain/` | Función pura probada con casos de agrupación |
| 2.7 | Crear regla de disponibilidad de vehículos: solo `vhe_estatus = TRUE` y sin solicitud activa asignada | `backend/app/domain/` | Query optimizada + test |
| 2.8 | Algoritmo de asignación automática: vehículo disponible + ruta más corta a la solicitud (via OSRM) | `backend/app/domain/` | Asignación correcta en escenarios de 1 y N vehículos |
| 2.9 | Sugerir el pozo de abastecimiento más cercano a la ruta (usando la tabla `pozos` existente) | `backend/app/domain/` | Sugerencia funcional en la respuesta de asignación |
| 2.10 | Endpoint `POST /asignaciones/automatica` y `GET /asignaciones` que consume las reglas | `backend/app/api/v1/` | Endpoint protegido por JWT y documentado en `/docs` |

---

## Miércoles — WebSocket de Ubicación en Tiempo Real

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 2.11 | Configurar endpoint WebSocket `/ws/ubicacion` en FastAPI (el hook `on_startup` de `app/core/events.py` ya existe) | `backend/app/` | Cliente puede conectar y recibir mensajes |
| 2.12 | Persistir lat/lng de cada vehículo en `ubicaciones_pipa` (migración 00055) al recibir mensaje | `backend/app/infrastructure/` | Se inserta un registro por update |
| 2.13 | Broadcast de ubicaciones a todos los clientes conectados (supervisores) | `backend/app/` | Múltiples clientes reciben la misma posición |
| 2.14 | Manejo de desconexión y limpieza de conexiones huérfanas | `backend/app/` | Sin fugas de conexiones en logs |
| 2.15 | Tests del WebSocket (conectar, recibir, desconectar) | `backend/tests/` | Pytest verde |

---

## Jueves — Mapa en el Frontend (React Leaflet)

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 2.16 | Instalar `leaflet` y `react-leaflet` (agregar a `frontend/package.json`) | `frontend/` | `npm install` sin conflictos |
| 2.17 | Crear `MapaSupervision.jsx` (página a pantalla completa) con capa base de OpenStreetMap | `frontend/src/pages/` | Mapa se renderiza con tiles de OSM |
| 2.18 | Renderizar marcadores ("banderitas") de solicitudes pendientes/entregadas desde el servicio de solicitudes | `frontend/src/services/` + `src/pages/` | Marcadores visibles con tooltip del folio |
| 2.19 | Renderizar posición en vivo de cada vehículo conectada al WebSocket | `frontend/src/` | Marcadores de vehículos se mueven con el broadcast |
| 2.20 | Registrar ruta de `MapaSupervision` en el router del frontend y link en el layout | `frontend/src/` | Navegable desde el menú |

---

## Viernes — Integración y Pruebas E2E

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 2.21 | Servicio `asignacionesService.js` + `solicitudesService.js` en el frontend consumiendo `localhost:8000/api/v1` (y `VITE_API_URL` en Docker) | `frontend/src/services/` | Pantallas usan los servicios, no datos quemados |
| 2.22 | Validar CORS/red: dentro de Docker el frontend llama a `http://backend:8000`, fuera a `localhost:8000` | `frontend/src/services/api.js` + `docker-compose.yml` | Sin errores de CORS en consola |
| 2.23 | Prueba E2E: crear solicitud → asignar vehículo → verlo en el mapa → simular ubicación | terminal / navegador | Flujo completo funcional |
| 2.24 | Correr `pytest` completo en backend y build del frontend | terminal | Todo verde, `npm run build` exitoso |
| 2.25 | PR a `feature/pipas-semana2` y revisión | GitHub | PR mergeado con commits `feat:`/`fix:` |

---

## Criterios de Aceptación del Sprint

1. OSRM corre en `dapa-network` y responde rutas.
2. La asignación automática elige un vehículo disponible con la ruta más corta.
3. El WebSocket transmite y persiste posiciones de vehículos en tiempo real.
4. `MapaSupervision` muestra banderitas de solicitudes y movimiento en vivo de vehículos.
5. Flujo E2E completo validado y sin errores de CORS.

## Comandos útiles

```bash
docker-compose up -d --build
curl "http://localhost:5000/route/v1/driving/-103.5,19.2;-103.6,19.3"  # probar OSRM
docker-compose exec backend pytest
npm --prefix frontend run build
```
