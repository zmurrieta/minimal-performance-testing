# Diseño: plataforma mínima de pruebas de carga

## Decisiones

- Mantener Go para API y worker, Vue 3 para la interfaz y PostgreSQL como estado durable.
- Ejecutar inicialmente un proceso Locust por ejecución usando Docker Compose.
- Usar un directorio o volumen de artefactos local; modelarlo como `ArtifactStore` para reemplazarlo por S3 después.
- Crear una abstracción `TestRuntime`; `DockerRuntime` será la primera implementación y `KubernetesRuntime` una extensión futura.
- No incorporar NATS en este MVP salvo que resulte necesario para desacoplar el worker. Un worker basado en polling de PostgreSQL reduce servicios locales y complejidad.

## Componentes

```text
Vue web
  | REST /api/v1
Go API -------------------------- PostgreSQL
  | crea solicitudes                    ^
  v                                    |
Go worker -- DockerRuntime -- Locust ---+
  |
  +-- volumen local de artefactos
```

El API nunca ejecuta comandos Docker directamente. Crea una fila `test_runs` en estado `pending`. El worker reclama de forma transaccional una fila pendiente, aplica validaciones defensivas, ejecuta Locust y persiste los cambios de estado y métricas.

## Modelo de datos

### `test_suites`

| Campo | Descripción |
| --- | --- |
| `id` | UUID. |
| `name`, `description` | Metadatos visibles. |
| `locustfile` | Código de plantilla aprobado. |
| `archived_at` | Una plantilla archivada no permite nuevas ejecuciones. |
| `created_by`, `created_at`, `updated_at` | Auditoría básica. |

### `test_runs`

| Campo | Descripción |
| --- | --- |
| `id`, `test_suite_id`, `status` | Identidad, relación y ciclo de vida. |
| `target_host`, `users`, `spawn_rate`, `duration_seconds`, `workers` | Configuración inmutable. |
| `locustfile_snapshot` | Copia de la plantilla usada. |
| `requested_by`, `started_at`, `finished_at`, `error_message` | Auditoría y diagnóstico. |
| `total_requests`, `total_failures`, `requests_per_second`, `avg_response_ms`, `p95_response_ms`, `p99_response_ms` | Resumen final nullable. |
| `runtime_ref` | Identificador del contenedor/proceso para detenerlo. |

### `audit_events`

Almacena `id`, `actor`, `action`, `entity_type`, `entity_id`, `metadata` JSON y `created_at`. No se actualiza ni elimina desde la aplicación.

## Contrato HTTP

| Método y ruta | Rol | Uso |
| --- | --- | --- |
| `GET /api/v1/version` | público | Versión. |
| `GET /api/v1/test-suites` | operator | Lista plantillas activas. |
| `POST /api/v1/test-suites` | admin | Crea plantilla. |
| `PATCH /api/v1/test-suites/{id}` | admin | Edita o archiva plantilla. |
| `GET /api/v1/runs` | operator | Lista ejecuciones; un operador ve solo las propias. |
| `POST /api/v1/runs` | operator | Crea una ejecución pendiente. |
| `GET /api/v1/runs/{id}` | operator | Detalle y métricas. |
| `POST /api/v1/runs/{id}/stop` | owner/admin | Solicita detención. |
| `GET /api/v1/runs/{id}/artifacts/{name}` | owner/admin | Descarga un artefacto permitido. |

`POST /runs` recibe:

```json
{
  "test_suite_id": "uuid",
  "target_host": "https://service.example.com",
  "users": 20,
  "spawn_rate": 2,
  "duration_seconds": 120,
  "workers": 1
}
```

## Ciclo de vida de una ejecución

1. API autentica, autoriza, valida límites y destino; inserta `pending` y auditoría.
2. Worker reclama atómicamente la fila y la cambia a `running`.
3. Worker crea un directorio temporal aislado, escribe el snapshot como `locustfile.py` y arranca Locust con `--headless`, duración y CSV configurados.
4. Worker supervisa finalización, solicitud de stop y deadline.
5. Worker analiza CSV, guarda artefactos, actualiza estado y métricas, y registra auditoría.
6. Worker elimina contenedor y archivos temporales. Una tarea de recuperación elimina ejecuciones `running` huérfanas.

## Seguridad de destinos

La configuración incluye `ALLOWED_TARGET_HOSTS` como lista de nombres DNS exactos o sufijos controlados. Al crear y al ejecutar una prueba se debe:

1. exigir URL `http` o `https` sin credenciales;
2. validar host contra allowlist;
3. resolver DNS y rechazar cualquier IPv4/IPv6 loopback, privada, link-local, multicast, unspecified o reservada;
4. bloquear explícitamente `169.254.169.254`, nombres de metadata y dominios internos configurados;
5. revalidar antes de arrancar el runtime para reducir ataques DNS rebinding.

## Configuración mínima

| Variable | Ejemplo | Uso |
| --- | --- | --- |
| `APP_HTTP_ADDR` | `:8080` | API. |
| `DATABASE_URL` | `postgres://...` | Persistencia. |
| `ARTIFACTS_DIR` | `/data/artifacts` | Artefactos locales. |
| `ALLOWED_TARGET_HOSTS` | `api.example.com,.staging.example.com` | Allowlist. |
| `MAX_RUN_DURATION` | `15m` | Límite. |
| `MAX_WORKERS` | `4` | Límite. |
| `MAX_USERS` | `1000` | Límite. |
| `MAX_CONCURRENT_RUNS_PER_USER` | `2` | Cuota. |
| `AUTH_MODE` | `development` | `development` u `oidc` futuro. |

## Estructura propuesta

```text
apps/api/cmd/api                 proceso HTTP
apps/worker/cmd/worker           proceso asíncrono
internal/domain                  entidades y reglas
internal/ports                   repositorio, runtime, artefactos, identidad
internal/adapters/postgres       persistencia
internal/adapters/docker         DockerRuntime
apps/web                         interfaz Vue
db/migrations                    migraciones versionadas
compose.yaml                     api, worker, web, postgres
```

## Pruebas obligatorias

- Unitarias para reglas de límites, estados y validación DNS/allowlist.
- Integración de repositorios contra PostgreSQL en Compose.
- Integración del worker con un runtime falso; Docker real solo en una prueba de smoke opcional.
- End-to-end: crear plantilla, iniciar, completar y detener ejecución.
