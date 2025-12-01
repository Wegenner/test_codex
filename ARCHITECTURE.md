# Sistema de Gestión Financiera Personal

Este documento describe la arquitectura propuesta para una aplicación full-stack de gestión financiera con integración a n8n. Está orientada a React en el frontend y NestJS + Prisma en el backend, con PostgreSQL como base de datos.

## Stack

- **Frontend:** React + TypeScript (Vite), UI Mantine/MUI, estado global con Zustand o RTK Query Toolkit, fetching de datos con React Query/RTK Query.
- **Backend:** Node.js + TypeScript con NestJS (o Express modular), ORM Prisma sobre PostgreSQL, autenticación JWT + refresh tokens.
- **Infra:** Docker Compose (API, DB, frontend), GitHub Actions para CI (lint, test, build), integración Webhook/REST para n8n.

## Modelo de dominio

- **Usuario**: credenciales y preferencias.
- **Cuenta**: nombre, tipo, saldo inicial, moneda.
- **Tarjeta**: límite, saldo, fecha de corte, relación a Cuenta.
- **Categoría**: ingreso/gasto, nombre, color.
- **Transacción**: monto, fecha, nota, categoría, estado (pendiente/conciliada), referencia a Cuenta y/o Tarjeta.
- **InboxItem**: movimientos sin asignar, payload de origen, estado (pendiente/asignado/descartado), regla sugerida.
- **CargoRecurrente**: monto, periodicidad (cron/intervalo), límite opcional, próxima fecha estimada, cuenta/tarjeta destino.
- **Regla de asignación** (opcional): patrón de texto/fuente, categoría/cuenta sugerida.
- **Vistas calculadas**: métricas de dashboard (balance, gasto por categoría, próximos cargos). Se modelan como consultas agregadas en Prisma/SQL.

## Backend

### Endpoints CRUD

| Recurso | Endpoints clave |
| --- | --- |
| Cuentas | `GET/POST /accounts`, `GET/PUT/PATCH/DELETE /accounts/:id` |
| Categorías | `GET/POST /categories`, `GET/PUT/PATCH/DELETE /categories/:id` |
| Transacciones | `GET/POST /transactions`, `GET/PUT/PATCH/DELETE /transactions/:id` |
| Tarjetas | `GET/POST /cards`, `GET/PUT/PATCH/DELETE /cards/:id` |
| Cargos recurrentes | `GET/POST /recurrings`, `GET/PUT/PATCH/DELETE /recurrings/:id` |
| Inbox | `GET/POST /inbox`, `POST /inbox/:id/assign` |

### Flujos especiales

- **Inbox**
  - Creación vía API o webhook n8n (payload libre). 
  - Endpoint de asignación (`POST /inbox/:id/assign`) que valida la relación con el usuario, mapea los campos y crea la Transacción resultante; marca el inbox como asignado.
  - Listado con filtros (estado, fecha de creación, origen).
- **Cargos recurrentes**
  - Scheduler (cron/worker) que lee cargos activos y genera Transacciones según periodicidad, respetando límites (cantidad máxima o monto acumulado).
  - Endpoint para consultar próximos cargos (sumatoria por tarjeta/cuenta) y fecha prevista de ejecución.
- **Integración n8n**
  - Webhook REST `POST /integrations/n8n/inbox` para crear InboxItems.
  - Endpoints de estado `GET /integrations/n8n/inbox/:id` y documentación OpenAPI/Swagger.
- **Seguridad y validaciones**
  - Policies por usuario/tenant en cada endpoint.
  - Validación de montos (no negativos salvo reembolsos), fechas (rango), y paginación con filtros de fecha/categoría/cuenta.
  - JWT access + refresh; rotación de refresh tokens.

### Persistencia y lógica

- Reglas de asignación: se aplican al crear InboxItem para sugerir categoría/cuenta; se almacenan para auditoría y override manual.
- Reconciliación: Transacción puede marcarse como conciliada/no; derivado de su origen en Inbox.
- Cálculos: consultas agregadas para totales por categoría/cuenta, disponibles de tarjeta y próximos cargos recurrentes.

## Frontend

- **Dashboard**: KPIs (balance, gasto/ingreso), gráfica de gasto/ingreso, panel de tarjetas (límite, disponible, fecha de corte), listado de cargos recurrentes con total próximo a pagar.
- **Inbox**: tabla/lista con filtros; wizard/modal para asignar Inbox→Transacción (selección de cuenta, categoría, fecha, nota, reconciliación).
- **Transacciones**: tabla con filtros por fecha/categoría/cuenta, búsqueda, paginación, edición rápida y conciliación.
- **Categorías**: CRUD con colores/íconos.
- **Cargos recurrentes**: cards con periodicidad/límite/total próximo, creación/edición, alertas/notificaciones antes del cargo.
- **Cuentas/Tarjetas**: detalle y movimientos, cálculo de disponible de tarjeta.

## Automatización y pruebas

- **Lint/format**: ESLint + Prettier.
- **Pruebas unitarias**: Jest/Testing Library (frontend/backend).
- **Pruebas de API**: Supertest con base de datos de prueba de Prisma (seed inicial).
- **Seeds**: datos demo para sandbox local.
- **n8n**: ejemplos de workflow (enviar JSON al webhook de inbox, leer cargos recurrentes) con colección exportada.

## Despliegue

- **Local**: Docker Compose con servicios `api`, `db` y `web`; variables de entorno (DATABASE_URL, JWT_SECRET, REFRESH_SECRET, CORS_ORIGIN, PORTS).
- **CI**: GitHub Actions ejecuta lint/test/build en cada push/PR.
- **Prod opcional**: Railway/Render/Fly; CDN para assets; revisión de variables de entorno y migraciones de Prisma en el deploy.
