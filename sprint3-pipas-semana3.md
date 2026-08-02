# Sprint 3 — Semana 3: Frontend de Operación + Reportes

- **Duración:** 5 días hábiles
- **Objetivo:** Entregar el flujo operativo completo de Sandra (recibir solicitud → asignar → supervisar → reportar) y los reportes de combustible y tiempos de traslado.
- **Branches:** `feature/pipas-dashboard`, `feature/pipas-catalogos`, `feature/pipas-reportes`.
- **Dependencia:** Semanas 1 y 2 terminadas.

---

## Lunes — Dashboard Logístico (Kanban)

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 3.1 | Crear `DashboardPipas.jsx`: tablero Kanban de solicitudes con columnas Pendiente / En Ruta / Entregada / Cancelada | `frontend/src/pages/` | Las solicitudes se agrupan por estado desde la API |
| 3.2 | Componente reutilizable `SolicitudCard.jsx` (folio, vehículo, domicilio, hora de entrega) | `frontend/src/components/` | Tarjeta funcional con acciones |
| 3.3 | Acciones en la tarjeta: Asignar vehículo, Cambiar estado, Cancelar (con motivo) | `frontend/src/` | Cada acción llama al endpoint correspondiente |
| 3.4 | Drag & drop entre columnas que llama a `PATCH /solicitudes_pipa/{id}/estado` | `frontend/src/` | El estado cambia en BD y se refleja en el tablero |
| 3.5 | Filtros por fecha, vehículo y estado + contador por columna | `frontend/src/` | Filtros funcionales |

---

## Miércoles — Pantallas de Catálogos

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 3.6 | Alta/edición de vehículos (reutilizando el patrón de `EstadosPage` y `DataTable`) | `frontend/src/pages/` | CRUD de vehículos contra la API |
| 3.7 | Alta/edición de marcas de vehículo | `frontend/src/pages/` | CRUD de marcas contra la API |
| 3.8 | Alta de solicitud de pipa (formulario: domicilio, cantidad, hora de entrega, vehículo asignado) | `frontend/src/pages/` | `POST /solicitudes_pipa` funciona |
| 3.9 | Alta de pozos y gestión de su disponibilidad (tabla `pozos` existente) | `frontend/src/pages/` | CRUD de pozos contra la API |
| 3.10 | Registro de gasto de combustible por vehículo | `frontend/src/pages/` | `POST /combustible` funciona |

---

## Jueves — Reportes

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 3.11 | Backend: endpoint `GET /reportes/combustible` (filtros por fecha y vehículo, totales por día) | `backend/app/api/v1/` | Devuelve agregados correctos |
| 3.12 | Backend: endpoint `GET /reportes/tiempos_traslado` (duración promedio, entregas por día, km recorridos) | `backend/app/api/v1/` | Calcula desde `rutas_solicitud` y `ubicaciones_pipa` |
| 3.13 | Frontend: página de reporte de combustible con filtros y tabla de resultados | `frontend/src/pages/` | Muestra datos reales de la API |
| 3.14 | Frontend: página de reporte de tiempos de traslado | `frontend/src/pages/` | Muestra datos reales de la API |
| 3.15 | Exportar reportes a CSV/Excel y opción de imprimir (mismo patrón que `EstadosPage`) | `frontend/src/pages/` | Exportación funcional |

---

## Viernes — Pruebas E2E, Documentación y Demo

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 3.16 | Tests backend de los endpoints de reportes y del flujo completo | `backend/tests/` | Pytest verde |
| 3.17 | Prueba E2E del flujo completo: alta de solicitud → asignación → Kanban → reporte | navegador | Todo el flujo operativo de Sandra funciona |
| 3.18 | Validar permisos por rol (solo roles con `solicitudes.gestionar` pueden cambiar estados) | `backend/app/api/v1/` | Roles sin permiso reciben 403 |
| 3.19 | Correr `pytest` + `npm run build` y revisar `pre-commit` | terminal | Todo verde |
| 3.20 | Actualizar `servicios.md` y `documentacion/` con endpoints y pasos de configuración de OSRM/WebSocket | `documentacion/` | Documentación al día |
| 3.21 | PR a `feature/pipas-semana3` y preparar demo funcional | GitHub | PR mergeado + demo grabada/lista |

---

## Criterios de Aceptación del Sprint

1. Kanban funcional: Sandra puede cambiar estados con drag & drop y ver el flujo completo.
2. CRUD de catálogos (vehículos, marcas, pozos, solicitudes, combustible) operando contra la API.
3. Reportes de combustible y tiempos de traslado con datos reales y exportación.
4. Control de permisos por rol aplicado.
5. Documentación actualizada y demo funcional entregada.

## Comandos útiles

```bash
docker-compose up -d --build
docker-compose exec backend pytest
npm --prefix frontend run dev   # desarrollo
npm --prefix frontend run build # producción
```
