# Plan de implementación: plataforma mínima de pruebas de carga

## 1. Fundaciones

- [ ] Crear configuración tipada con las variables definidas en el diseño y validarla al arrancar.
- [ ] Crear migraciones para ampliar `test_suites`, `test_runs` y añadir `audit_events`.
- [ ] Implementar repositorios PostgreSQL y pruebas de integración.
- [ ] Implementar health y readiness reales.

## 2. Dominio y seguridad

- [ ] Implementar autenticación de desarrollo y un puerto de identidad extensible para OIDC.
- [ ] Implementar autorización por rol y propiedad de ejecución.
- [ ] Implementar máquina de estados de una ejecución y transiciones válidas.
- [ ] Implementar validación de límites, cuota de concurrencia y allowlist.
- [ ] Implementar resolución DNS segura y sus pruebas para rangos prohibidos.
- [ ] Registrar eventos de auditoría en todas las transiciones relevantes.

## 3. API

- [ ] Implementar CRUD administrativo de plantillas.
- [ ] Implementar creación, listado y detalle de ejecuciones.
- [ ] Implementar solicitud idempotente de detención.
- [ ] Implementar descarga autorizada de artefactos con protección contra path traversal.
- [ ] Documentar ejemplos HTTP y errores de validación en el README.

## 4. Worker y runtime local

- [ ] Crear el worker que reclama ejecuciones pendientes de forma segura.
- [ ] Definir `TestRuntime` y `ArtifactStore` con implementaciones de prueba.
- [ ] Implementar `DockerRuntime` para lanzar Locust headless sin privilegios ni montajes del host.
- [ ] Escribir el snapshot de la plantilla y recoger CSV/HTML en un directorio por ejecución.
- [ ] Analizar métricas CSV, guardar resumen y artefactos.
- [ ] Implementar stop, deadline, limpieza y recuperación de ejecuciones huérfanas.

## 5. Interfaz

- [ ] Crear vistas de ejecuciones, nueva ejecución, detalle y plantillas.
- [ ] Conectar las vistas al contrato API y mostrar estados de carga/error.
- [ ] Mostrar métricas resumidas y enlaces de artefactos.
- [ ] Centralizar textos para permitir localización posterior.

## 6. Operación y verificación

- [ ] Actualizar `compose.yaml` para API, worker, web, PostgreSQL y volumen de artefactos.
- [ ] Proporcionar `.env.example` completo y una plantilla Locust segura de ejemplo.
- [ ] Añadir pruebas unitarias, integración y smoke e2e al pipeline CI.
- [ ] Verificar manualmente los seis criterios de aceptación de `requirements.md`.
- [ ] Mantener Helm/Terraform fuera de esta entrega o marcarlos como extensión futura; no deben ser requisito para el MVP local.

## Definición de terminado

La implementación está terminada cuando todos los criterios de aceptación de `requirements.md` pasan localmente con Docker Compose, las pruebas automatizadas pasan y no hay secretos ni identificadores de una empresa en el código ni en la configuración de ejemplo.
