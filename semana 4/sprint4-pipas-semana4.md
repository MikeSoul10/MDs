# Sprint 4 — Semana 4: Corrección de Errores y Hardening

- **Duración:** 5 días hábiles
- **Objetivo:** Estabilizar el módulo de pipas: QA integral (incluyendo la PWA), corrección de bugs, optimización de rendimiento y entrega del release candidate.
- **Branches:** `fix/pipas-*` según el bug corregido.
- **Dependencia:** Semanas 1 a 3 terminadas.

---

## Lunes — QA Integral y Backlog de Bugs

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 4.1 | QA funcional de catálogos (vehículos, marcas, pozos) contra la API | terminal + navegador | Lista de bugs documentada |
| 4.2 | QA del ciclo de vida de solicitudes (todos los cambios de estado posibles y los inválidos) | navegador | Transiciones inválidas bloqueadas |
| 4.3 | QA del mapa en vivo y la PWA del chofer: reconexión del WebSocket, múltiples vehículos, sesiones largas, GPS real del teléfono | navegador + dispositivo móvil | Lista de bugs documentada |
| 4.4 | QA de la PWA: instalación en Android/iOS, modo offline, notificaciones push, batería con GPS activo | dispositivo móvil | Lista de bugs documentada |
| 4.5 | QA de reportes (combustible, tiempos) con volúmenes de datos reales | navegador | Cifras coherentes |
| 4.6 | Priorizar bugs: críticos (bloquean uso) / mayores / menores y registrarlos | `documentacion/issues/` o GitHub | Backlog priorizado con reproducción de cada bug |

---

## Martes — Corrección de Bugs Críticos y Mayores

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 4.7 | Fix de bugs críticos de estados: evitar saltos inválidos y estados fantasma en `historial_solicitud` | `backend/app/` | Tests de regresión verdes |
| 4.8 | Fix de disponibilidad: vehículo con solicitud activa no debe asignarse de nuevo | `backend/app/domain/` | Caso de doble asignación cubierto |
| 4.9 | Fix de rutas: manejar OSRM caído o timeout sin romper la asignación | `backend/app/infrastructure/` | Fallback documentado y testeado |
| 4.10 | Fix de WebSocket/PWA: reconexión automática del frontend, cola de posiciones offline y limpieza de conexiones | `frontend/src/` + `backend/app/` | Reconexión sin interacción manual, posiciones no perdidas |
| 4.11 | Fix de CORS/red entre contenedores y de permisos por rol (403 correcto) | `docker-compose.yml` + `backend/app/api/v1/` | Accesos no autorizados rechazados |

---

## Miércoles — Optimización de Rendimiento

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 4.12 | Revisar índices en `solicitudpipas`, `ubicaciones_pipa`, `registros_combustible` y `historial_solicitud` | `db/migrations/` | Planes de consulta con los índices correctos |
| 4.13 | Optimizar el listado de solicitudes (paginación, filtros, carga perezosa de relaciones) | `backend/app/api/v1/` | Listado sin consultas N+1 |
| 4.14 | Optimizar broadcast del WebSocket (batch de posiciones en lugar de mensaje por update) | `backend/app/` | Menos mensajes, mismo dato en vivo |
| 4.15 | Optimizar la PWA: reducir batería del GPS (ajustar intervalo de envío, pausa al detenerse) | `frontend/src/` | Menor consumo en uso real |
| 4.16 | Cachear catálogos de baja volatilidad (marcas, pozos) si aplica y validar tiempos de respuesta (< 300 ms en listados) | `backend/app/` | Métricas documentadas |

---

## Jueves — Bugs Menores y Pulido de UI

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 4.17 | Cierre de bugs menores de la lista priorizada | `frontend/src/` + `backend/app/` | Bugs menores cerrados |
| 4.18 | Pulido de UI del Kanban, mapa y formularios (estados de carga, errores amigables, empty states) | `frontend/src/` | UI consistente con el resto del sistema |
| 4.19 | Pulido de la PWA del chofer (diseño móvil, botones grandes, feedback al marcar entregas) | `frontend/src/` | PWA cómoda de usar en la calle |
| 4.20 | Mensajes de validación en español y consistentes | `frontend/src/` + `backend/app/api/schemas/` | Validaciones claras para el usuario |
| 4.21 | Completar documentación técnica del módulo (modelo de datos, endpoints, WebSocket, OSRM, PWA) | `documentacion/` | Documentación final revisada |
| 4.22 | Pruebas de humo de todo el sistema (backend + frontend + DB + mapa + PWA) | terminal + navegador | Sin regresiones |

---

## Viernes — Release Candidate y Demo Final

| # | Tarea | Archivo destino | Definición de Terminado |
|---|-------|-----------------|--------------------------|
| 4.23 | Correr `pytest` completo + `npm run build` (PWA) + `pre-commit` | terminal | Todo verde |
| 4.24 | Verificar que `docker-compose up` desde cero funciona (migraciones + seeds) | terminal | Levantamiento limpio sin errores |
| 4.25 | Revisión final de seguridad: JWT, permisos, validación de entradas, WebSocket autenticado | `backend/app/` | Sin hallazgos críticos |
| 4.26 | Etiquetar el release (ej. `v0.1.0`) y merge a `main` vía PR | GitHub | Release etiquetado |
| 4.27 | Demo final del módulo de pipas (panel de Sandra + PWA del chofer) a los interesados | navegador + dispositivo móvil | Aprobación y retroalimentación registrada |

---

## Criterios de Aceptación del Sprint

1. Cero bugs críticos abiertos; bugs mayores y menores cerrados o con plan.
2. La PWA del chofer es estable en dispositivos reales (GPS, offline, reconexión).
3. El sistema responde estable bajo uso concurrente (listados < 300 ms).
4. Levantamiento limpio desde cero con `docker-compose up`.
5. Suite completa de pruebas en verde y release etiquetado.
6. Demo final aprobada.

## Comandos útiles

```bash
docker-compose up -d --build
docker-compose exec backend pytest
npm --prefix frontend run build
docker-compose down -v && docker-compose up -d --build   # prueba desde cero
```
