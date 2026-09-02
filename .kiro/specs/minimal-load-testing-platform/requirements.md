# Requisitos: plataforma mínima de pruebas de carga

## Objetivo

Crear una réplica mínima, instalable por empresa, para ejecutar pruebas de carga Locust controladas desde una interfaz web. El producto debe poder funcionar localmente con Docker Compose y no contener marca, dominios, identidades ni infraestructura específicas de una organización.

## Alcance del MVP

Un usuario autenticado con rol `operator` puede crear una ejecución a partir de una plantilla de prueba aprobada, definir usuarios, tasa de llegada, duración y número de workers, iniciarla, seguir su estado y ver el resultado agregado. Un usuario `admin` puede administrar las plantillas y los límites de la instalación.

La primera implementación ejecutará Locust mediante Docker Compose. El código debe aislar esa decisión detrás de una interfaz para poder añadir Kubernetes posteriormente.

## Actores y permisos

| Actor | Permisos |
| --- | --- |
| `admin` | Administrar plantillas, límites y allowlist; ver y detener cualquier ejecución. |
| `operator` | Crear, consultar y detener sus propias ejecuciones. |

Para el MVP local se permitirá autenticación de desarrollo mediante cabeceras configurables. La interfaz de autenticación debe permitir añadir OIDC sin cambiar reglas de autorización ni handlers de negocio.

## Requisitos funcionales

### Plantillas

1. El sistema debe almacenar plantillas aprobadas de Locust con nombre, descripción, código y fecha de creación.
2. Solo `admin` puede crear, editar o archivar plantillas.
3. Una ejecución debe guardar una copia inmutable del código de su plantilla para que los resultados sean reproducibles.

### Ejecuciones

1. `operator` debe poder crear una ejecución con: plantilla, host objetivo, usuarios, spawn rate, duración y workers.
2. El sistema debe validar antes de crearla que:
   - el host está en una allowlist de destinos;
   - el destino no es localhost, una IP privada, link-local, metadata service o control plane;
   - usuarios, spawn rate, duración y workers están dentro de los límites configurados;
   - el usuario no supera la cuota de ejecuciones simultáneas.
3. Una ejecución pasa por los estados `pending`, `running`, `succeeded`, `failed`, `stopped`.
4. El creador o un `admin` puede detener una ejecución `pending` o `running`.
5. Toda ejecución debe tener una fecha límite; el runtime debe eliminar sus recursos aunque el proceso de control falle.
6. La API debe guardar hora de inicio, fin, motivo de fallo y un resumen de métricas cuando estén disponibles.

### Resultados

1. La pantalla de detalle debe mostrar estado, configuración, tiempos y un resumen con requests, fallos, RPS, tiempo medio, p95 y p99.
2. Si Locust produce CSV/HTML, el sistema debe conservarlos como artefactos de la ejecución y exponer enlaces de descarga autorizados.

### API y UI

1. La API debe exponer endpoints versionados bajo `/api/v1` y JSON como formato exclusivo.
2. La UI debe tener las vistas: listado de ejecuciones, nueva ejecución, detalle de ejecución y administración de plantillas.
3. La UI debe mostrar errores de validación de forma comprensible.
4. `/healthz` debe indicar que el proceso está vivo; `/readyz` debe verificar PostgreSQL y el servicio que consume trabajos.

## Requisitos no funcionales y de seguridad

1. Todo valor variable por instalación debe venir de variables de entorno o un archivo de configuración documentado.
2. No se deben guardar secretos en el repositorio ni en imágenes de contenedor.
3. Los procesos de runtime deben ejecutarse como usuario no root, con filesystem de solo lectura cuando sea posible y sin credenciales cloud por defecto.
4. El código de una plantilla se considera no confiable: nunca puede montarse el socket Docker ni directorios del host en su contenedor.
5. La API debe registrar auditoría de creación, inicio, detención y finalización con actor, ejecución y timestamp.
6. Cada empresa será una instalación independiente; el MVP no promete multitenancy dentro de la aplicación.
7. La aplicación debe ser utilizable en español o inglés mediante textos centralizados; iniciar con inglés neutral.

## Criterios de aceptación del MVP

1. Con `docker compose up --build`, un `operator` puede iniciar una prueba aprobada contra un dominio permitido y consultar su resultado.
2. Un host no permitido, una IP local/privada, duración excesiva o workers excesivos se rechazan con HTTP 400 y un mensaje claro.
3. Detener una prueba elimina el contenedor Locust asociado y deja la ejecución en estado `stopped`.
4. Una prueba que alcance su duración termina sola y conserva el resumen de resultados.
5. Los endpoints de disponibilidad fallan si PostgreSQL o el worker no están listos.
6. La instalación se configura sin editar el código fuente.

## Fuera de alcance

- Ejecución distribuida en Kubernetes/EKS.
- Aprovisionamiento cloud/Terraform.
- SSO/OIDC real y gestión corporativa de usuarios.
- Cobro, multitenancy lógico, equipos, notificaciones y comparativas históricas.
- Editor avanzado de scripts en el navegador.
