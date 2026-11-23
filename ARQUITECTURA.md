# Arquitectura del proyecto integrador — Variante B

**Proyecto:** reservas-app
**Curso:** Contenedores, Docker y Orquestación con Kubernetes
**Institución:** FACET — Universidad Nacional de Tucumán
**Fecha de entrega:** 23 de noviembre de 2026
**Tecnologías:** React, Vite, Nginx, Node.js 20, Express, MySQL 8.4, Docker, Docker Compose, Kubernetes y Helm.

---

## 1. Descripción general de la arquitectura

`reservas-app` es una aplicación web destinada a gestionar reservas de salas y laboratorios por fecha y franja horaria.

El sistema se desarrolló siguiendo una arquitectura de tres capas:

```text
                     Navegador
                         |
                         | HTTP
                         v
                reservas-frontend
                 React + Nginx
                    Puerto 8080
                         |
                         | /api
                         v
                   reservas-api
                Node.js + Express
                    Puerto 3000
                         |
                         | SQL
                         v
                   reservas-db
                     MySQL 8.4
                    Puerto 3306
```

El frontend presenta la interfaz de usuario y envía solicitudes HTTP a la API. La API contiene la lógica de negocio, valida las operaciones y realiza consultas sobre MySQL. La base de datos almacena las salas y las reservas.

El proyecto evolucionó a lo largo del curso desde contenedores ejecutados manualmente hasta un despliegue declarativo sobre Kubernetes, administrado mediante Helm.

En su configuración final se utilizaron:

- Tres Deployments: frontend, API y base de datos.
- Tres Services principales para la comunicación entre componentes.
- Dos ConfigMaps para configuración no sensible.
- Un Secret para las credenciales de MySQL.
- Un PersistentVolumeClaim para conservar los datos.
- Un Ingress para publicar la aplicación.
- Un HorizontalPodAutoscaler para escalar la API.

Estos recursos conforman los doce objetos principales del proyecto.

El Service adicional `reservas-api-nodeport`, utilizado para depuración durante el TP6, no pertenece al chart final.

---

## 2. Configuración del frontend en tiempo de ejecución

### 2.1. ¿Por qué se utiliza API_HOST y no VITE_API_URL?

La configuración del frontend se resolvió mediante variables de entorno interpretadas por Nginx al iniciar el contenedor.

En el Dockerfile se definieron las variables:

```dockerfile
ENV API_HOST=reservas-api \
    API_PORT=3000 \
    NGINX_ENVSUBST_FILTER=^API_
```

La imagen final utiliza `nginxinc/nginx-unprivileged:1.27-alpine` y contiene los archivos estáticos generados por Vite.

Al iniciar el contenedor, Nginx procesa su plantilla de configuración mediante `envsubst`. De esta manera, el destino del proxy se determina en tiempo de ejecución a partir de `API_HOST` y `API_PORT`.

La variable `NGINX_ENVSUBST_FILTER=^API_` limita la sustitución a las variables cuyo nombre comienza con `API_`, evitando reemplazos accidentales de variables propias de Nginx.

Por otra parte, las variables `VITE_*` son incorporadas por Vite durante la construcción del frontend.

Si se utilizara `VITE_API_URL` para establecer una URL absoluta, esa dirección quedaría integrada en los archivos JavaScript generados. Para cambiar de desarrollo a producción, o modificar posteriormente el destino de la API, sería necesario volver a construir el frontend y publicar una nueva imagen.

La configuración actual permite utilizar una misma imagen en diferentes entornos. Solamente es necesario proporcionar las variables adecuadas cuando se inicia el contenedor.

### 2.2. ¿Cómo se implementó esta decisión?

El archivo `frontend/src/api.js` utiliza:

```javascript
const BASE = "/api";
```

Las solicitudes se realizan mediante rutas relativas, por ejemplo:

```text
/api/health
/api/salas
/api/reservas
```

El navegador no necesita conocer el nombre interno del Service de Kubernetes ni el puerto de Express.

Nginx recibe esas solicitudes y las redirige internamente hacia la API.

Esta separación permite mantener la imagen del frontend independiente de la configuración de red propia de cada entorno.

---

## 3. Comunicación entre componentes y CORS

### 3.1. ¿Por qué la arquitectura no necesita CORS?

CORS es un mecanismo de seguridad aplicado por los navegadores a solicitudes HTTP entre distintos orígenes.

Un origen se define por la combinación de protocolo, nombre de host y puerto.

En el despliegue final, el usuario accede a la aplicación mediante:

```text
http://reservas.local
```

Cuando el frontend consulta las reservas, el navegador solicita:

```text
http://reservas.local/api/reservas
```

Ambas solicitudes corresponden al mismo origen.

El navegador solamente se comunica con el frontend publicado mediante el Ingress. La comunicación posterior con `reservas-api:3000` es realizada por Nginx dentro del clúster.

Por esta razón, la API no necesita encabezados CORS para atender las solicitudes habituales de esta aplicación.

### 3.2. ¿Qué cambiaría si el frontend consultara directamente la API?

Si la aplicación React realizara solicitudes a una dirección diferente, por ejemplo:

```text
http://reservas.local
http://api.reservas.local
```

el navegador detectaría dos orígenes diferentes.

En ese caso, la API debería incorporar una configuración CORS que definiera:

- Los orígenes autorizados.
- Los métodos HTTP permitidos.
- Los encabezados admitidos.
- El tratamiento de solicitudes preflight.
- Las credenciales, únicamente si fueran necesarias.

No sería correcto habilitar indiscriminadamente todos los orígenes en una aplicación que manejara autenticación o información sensible.

La utilización de un proxy inverso simplificó la arquitectura y evitó introducir una dependencia innecesaria de CORS.

CORS tampoco sustituye los controles de autenticación y autorización propios de la aplicación.

---

## 4. Escalado automático mediante HPA

### 4.1. ¿Por qué existe un único autoescalador?

Entre los doce objetos principales se incorporó un solo HorizontalPodAutoscaler, denominado `reservas-api-hpa`.

Su objetivo es el Deployment `reservas-api`.

La API constituye una capa adecuada para el escalado horizontal porque las solicitudes pueden distribuirse entre varias réplicas y el estado persistente se mantiene en MySQL.

Durante el TP8 se configuraron los siguientes recursos por contenedor:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 64Mi
  limits:
    cpu: 250m
    memory: 128Mi
```

El HPA se configuró con:

```yaml
minReplicas: 2
maxReplicas: 6
averageUtilization: 50
```

El porcentaje de utilización se calcula respecto del CPU solicitado mediante `resources.requests`, no respecto del límite del contenedor.

En este caso, una utilización del 50 % corresponde aproximadamente a 50 millicores por réplica, tomando como referencia un request de 100m.

La información de consumo se obtuvo mediante metrics-server.

La prueba de carga permitió comprobar que la API aumentó de dos a cuatro y posteriormente a seis réplicas. Al detener la carga, el consumo de CPU disminuyó y el HPA regresó automáticamente a dos réplicas.

La reducción no fue inmediata debido a la ventana de estabilización del autoescalador.

La evidencia completa se encuentra en `informe/hpa-carga.log`.

### 4.2. ¿Qué ocurriría si se asignara un HPA a MySQL?

MySQL es un servicio con estado: almacena información persistente y mantiene estructuras internas que requieren consistencia.

Aumentar automáticamente las réplicas de un Deployment de MySQL no genera por sí solo una base de datos distribuida.

Si varias instancias independientes intentaran utilizar simultáneamente el mismo directorio de datos, podrían producirse errores de acceso, conflictos de archivos y riesgos para la integridad de la información.

Además, un PVC con modo `ReadWriteOnce` no constituye un sistema de replicación ni de coordinación entre instancias de una base de datos.

Para implementar alta disponibilidad en MySQL sería necesario definir una arquitectura específica de replicación, coordinación y recuperación, mediante una solución administrada o herramientas especializadas.

Por lo tanto, aplicar el mismo mecanismo de autoescalado utilizado por la API no sería una solución adecuada.

### 4.3. ¿Por qué tampoco se incorporó un HPA al frontend?

El frontend utiliza Nginx para entregar archivos estáticos y reenviar solicitudes.

Esta tarea presenta un consumo de CPU comparativamente bajo respecto de la lógica ejecutada por la API y de las consultas a MySQL.

Durante el proyecto no se identificó una necesidad concreta de escalar automáticamente el frontend.

Por esa razón, se mantuvo un Deployment con dos réplicas configuradas de manera declarativa.

En un sistema de producción con tráfico elevado podría evaluarse su escalado, pero debería justificarse mediante mediciones de consumo, concurrencia y tiempos de respuesta.

---

## 5. Concurrencia e integridad de las reservas

### 5.1. ¿Qué deja de estar garantizado al pasar de una a tres réplicas?

Al ejecutar varias réplicas de la API, diferentes solicitudes pueden procesarse simultáneamente en procesos independientes.

El problema aparece cuando dos o más usuarios intentan reservar la misma sala, en la misma fecha y franja horaria.

La API realiza una consulta previa para comprobar si la combinación se encuentra disponible.

Sin embargo, esta validación por sí sola no garantiza la ausencia de reservas duplicadas.

Puede producirse la siguiente secuencia:

```text
Réplica A                    Réplica B
---------                    ---------
SELECT: está libre
                             SELECT: está libre
INSERT de la reserva
                             INSERT de la reserva
```

Ambas réplicas pueden consultar el mismo estado antes de que cualquiera haya confirmado una inserción.

Este comportamiento constituye una condición de carrera, también denominada race condition.

La consulta previa y la inserción son operaciones separadas. Por lo tanto, no forman una verificación atómica.

### 5.2. ¿Qué mecanismo garantiza realmente la integridad?

La garantía principal se implementó en MySQL mediante una restricción de unicidad.

En el archivo `backend/db.js`, la tabla `reservas` contiene:

```sql
CONSTRAINT sin_solapamiento
UNIQUE (sala_id, fecha, franja)
```

La restricción impide almacenar dos registros con la misma combinación de sala, fecha y franja.

A diferencia del SELECT previo, esta regla es aplicada por el motor de base de datos, independientemente de la réplica de Express que atienda cada solicitud.

Cuando una inserción viola la restricción, MySQL produce el error `ER_DUP_ENTRY`.

La API identifica esa condición y responde con el código HTTP 409 Conflict.

De este modo, la consulta previa mejora la respuesta de la aplicación, pero la restricción UNIQUE protege la integridad de los datos.

### 5.3. Evidencia experimental de la sexta entrega

Durante el TP6 se escaló `reservas-api` a tres réplicas y se ejecutaron treinta solicitudes POST concurrentes sobre la misma combinación:

- Sala: 1.
- Fecha: 2026-12-01.
- Franja: 5.
- Responsables: R1 a R30.

Se registraron los siguientes resultados:

```text
HTTP 201 Created: 1 solicitud
HTTP 409 Conflict: 29 solicitudes
```

La consulta posterior a MySQL confirmó que existía exactamente una reserva para la combinación utilizada.

Esta prueba demostró que el comportamiento correcto no depende de que las solicitudes sean atendidas por una única réplica.

La evidencia y su interpretación se presentan también en `INFORME.md` y en la sección «Escalado y prueba de concurrencia» del README correspondiente al TP6.

---

## 6. Dificultades reales encontradas con MySQL

### 6.1. Inicialización fallida sobre un directorio de datos no vacío

**Manifestación del problema**

Durante el TP7 se creó un PersistentVolumeClaim de 1 GiB utilizando la StorageClass `local-path`.

Como parte de la prueba de almacenamiento, un Pod BusyBox escribió previamente un archivo sobre el volumen.

Cuando se intentó montar ese volumen completo como directorio de datos de MySQL, la inicialización falló.

El contenedor informó:

```text
--initialize specified but the data directory
has files in it. Aborting.
```

**Diagnóstico**

La revisión de los logs permitió identificar que el directorio utilizado por MySQL contenía archivos que no pertenecían a una base previamente inicializada.

MySQL requiere un directorio de datos apropiado para realizar su inicialización.

**Solución aplicada**

Se conservó el PVC y se configuró un subdirectorio específico mediante:

```yaml
volumeMounts:
  - name: datos
    mountPath: /var/lib/mysql
    subPath: mysql
```

De esta forma, MySQL utilizó un directorio separado dentro del volumen persistente.

Después de aplicar esta modificación, MySQL consiguió inicializarse y aceptar conexiones.

Posteriormente se comprobó que una reserva sobrevivía a la eliminación y recreación del Pod.

### 6.2. Healthcheck: expansión de variables y tiempo de inicialización

**Manifestación del problema**

En Docker Compose fue necesario definir una comprobación de estado confiable para MySQL.

La inicialización inicial del motor puede demorar, por lo que el simple hecho de que el contenedor esté ejecutándose no garantiza que la base de datos acepte conexiones.

Además, Docker Compose interpreta las referencias a variables de entorno al procesar el archivo YAML.

Esto podía alterar la forma en que se entregaba la contraseña al comando de comprobación.

**Diagnóstico**

Se identificaron dos aspectos diferentes:

1. La contraseña debía expandirse dentro del contenedor y no durante la interpretación del archivo Compose.
2. El servicio necesitaba un período de gracia para completar su inicialización.

**Solución aplicada**

Se configuró el healthcheck de MySQL utilizando:

```yaml
healthcheck:
  test:
    - CMD-SHELL
    - mysqladmin ping -h 127.0.0.1 -u root -p"$$MYSQL_ROOT_PASSWORD" --silent
  interval: 5s
  timeout: 5s
  retries: 20
  start_period: 40s
```

El uso de `$$` evita que Compose reemplace anticipadamente la variable. El símbolo `$` se conserva para que la expansión ocurra dentro del contenedor.

El período `start_period: 40s` evita penalizar los primeros intentos fallidos mientras MySQL todavía se inicializa.

La dirección `127.0.0.1` permite comprobar la conexión TCP al servidor.

Esta configuración se complementó con dependencias condicionadas al estado saludable de MySQL y con reintentos de conexión en la API.

### 6.3. Pérdida de información antes de incorporar el PVC

**Manifestación del problema**

Durante el TP6, MySQL todavía utilizaba almacenamiento efímero dentro del contenedor.

Se eliminó su Pod y Kubernetes creó automáticamente uno nuevo mediante el Deployment.

Aunque el nuevo Pod alcanzó el estado Running, las reservas anteriores habían desaparecido.

Incluso se registró el error:

```text
ERROR 1146 (42S02):
Table 'reservasdb.reservas' doesn't exist
```

**Diagnóstico**

La recreación del Pod restauró el proceso de MySQL, pero no sus archivos anteriores.

Se comprobó así que un Deployment mantiene la cantidad deseada de Pods, pero no garantiza la persistencia de los datos.

**Solución aplicada**

Durante el TP7 se incorporó el PersistentVolumeClaim `reservas-db-pvc`, con una capacidad de 1 GiB.

Se montó el almacenamiento mediante `subPath: mysql` y se repitió la prueba de eliminación del Pod.

La reserva de prueba permaneció disponible después de la recreación.

La experiencia permitió diferenciar claramente la disponibilidad del proceso de la durabilidad de sus datos.

### 6.4. Tratamiento de fechas entre MySQL y JavaScript

La columna `fecha` de MySQL utiliza el tipo `DATE`, que representa un día calendario sin información de zona horaria.

La conversión automática a objetos `Date` de JavaScript puede introducir desplazamientos cuando intervienen distintas zonas horarias, como la zona UTC de un contenedor.

Para evitar este comportamiento, el pool de conexiones de `mysql2` se configuró con:

```javascript
dateStrings: true
```

De esta manera, las fechas se reciben como cadenas y se evita una conversión innecesaria de zona horaria.

Esta medida resulta especialmente importante porque el día de la reserva forma parte de la restricción de unicidad del sistema.

---

## 7. Persistencia y administración mediante Helm

### 7.1. Persistencia de MySQL

Se configuró el PVC `reservas-db-pvc` con las siguientes características:

- Capacidad: 1 GiB.
- Modo de acceso: ReadWriteOnce.
- StorageClass: local-path.
- Directorio de montaje: `/var/lib/mysql`.
- Subdirectorio persistente: `mysql`.

Durante el TP7 se comprobó que la reserva identificada con el ID 7 sobrevivía a la eliminación y recreación del Pod de MySQL.

En el TP8 se verificó nuevamente la persistencia durante la transición hacia Helm.

Antes de instalar la release se registraron cuatro reservas en MySQL.

Después de la instalación y del rollback, la consulta continuó devolviendo cuatro reservas.

El PVC permaneció en estado Bound.

### 7.2. Empaquetado del despliegue

Se creó un chart Helm denominado `reservas-app` con:

- `Chart.yaml`.
- `values.yaml`.
- Doce plantillas en `templates/`.

Se parametrizaron los repositorios y tags de las imágenes propias, las cantidades de réplicas, los recursos de la API, los umbrales del HPA, los puertos, la configuración de los ConfigMaps, el host del Ingress y el tamaño solicitado del PVC.

Mediante `helm template` se comprobó que el chart generaba exactamente doce objetos.

También se verificó que modificar parámetros con `--set` producía cambios reales en los manifiestos renderizados.

Por lo tanto, el archivo `values.yaml` no cumple únicamente una función descriptiva: sus valores participan efectivamente en la generación de los recursos.

### 7.3. Instalación sobre recursos existentes

Los objetos de Kubernetes ya habían sido creados durante las entregas anteriores.

Para evitar la eliminación de los recursos y conservar la base de datos, se utilizó la opción `--take-ownership`, disponible en la versión de Helm utilizada.

Antes de realizar la instalación:

1. Se comprobó que la aplicación estaba funcionando.
2. Se verificó que el PVC estaba en estado Bound.
3. Se realizó un respaldo lógico de MySQL mediante `mysqldump`.
4. Se preservaron las credenciales existentes en un archivo privado, ubicado fuera del repositorio.
5. Se ejecutó una simulación mediante `--dry-run=server`.

La instalación real finalizó con:

```text
Release: reservas-app
Namespace: reservas-app
Revision: 1
Status: deployed
```

### 7.4. Actualización y reversión

Se construyó y publicó una imagen `laureanoa/reservas-api:v2`, utilizando el argumento de construcción `APP_VERSION=v2`.

Posteriormente se ejecutó un `helm upgrade` para modificar exclusivamente el tag de la API.

La nueva release alcanzó la revisión 2 y el endpoint `/api/health` informó:

```json
{
  "status": "ok",
  "db": "ok",
  "version": "v2"
}
```

Después se realizó un `helm rollback` hacia la revisión 1.

Helm creó una tercera revisión con la configuración anterior y la API volvió a informar `version: v1`.

El historial final quedó compuesto por:

```text
Revision 1: Install complete
Revision 2: Upgrade complete
Revision 3: Rollback to 1
```

El tag de la imagen del frontend no se modificó durante este procedimiento.

Esto demostró que la API y el frontend poseen ciclos de vida independientes.

El campo `APP VERSION` del historial de Helm permaneció en `v1`, ya que corresponde a `appVersion` del archivo `Chart.yaml`, y no al tag efectivo de la imagen de la API.

---

## 8. Aspectos pendientes para un entorno de producción

### 8.1. Observabilidad

Sería necesario incorporar métricas históricas, paneles de monitoreo, registros centralizados y alertas.

Herramientas como Prometheus y Grafana permitirían analizar tendencias de consumo, tiempos de respuesta, errores y comportamiento del autoescalado.

La observación realizada mediante `kubectl top` y `kubectl get hpa` resulta adecuada para el laboratorio, pero es insuficiente para una operación permanente.

### 8.2. Integración y despliegue continuos

Se debería implementar un pipeline de CI/CD que automatizara:

- La ejecución de pruebas.
- La construcción de imágenes.
- El análisis de vulnerabilidades.
- La publicación de imágenes versionadas.
- La validación del chart Helm.
- El despliegue controlado.
- La verificación posterior a la actualización.

La automatización reduciría los errores asociados a procedimientos manuales.

### 8.3. Gestión externa de secretos

El proyecto utiliza un Secret de Kubernetes para separar las credenciales de los manifiestos de los Deployments.

Sin embargo, Base64 es una codificación reversible y no constituye cifrado.

Para producción se requerirían controles RBAC adecuados, protección de los datos almacenados en etcd y una solución externa de gestión de secretos.

El chart utiliza valores ficticios de laboratorio como configuración predeterminada. Las credenciales reales empleadas durante las pruebas no se incorporaron al repositorio.

### 8.4. Copias de seguridad y recuperación

Un PVC permite persistencia frente a la recreación de Pods, pero no sustituye un sistema de backups.

Antes de instalar Helm se realizó un respaldo lógico mediante `mysqldump`, almacenado fuera del repositorio.

En producción sería necesario programar respaldos periódicos, definir retención, almacenar copias fuera del nodo y realizar pruebas de restauración.

Durante el laboratorio también se verificó que el PersistentVolume utilizaba la política `ReclaimPolicy: Delete`, lo cual refuerza la necesidad de contar con respaldos independientes.

### 8.5. Arquitectura de base de datos

MySQL se ejecuta actualmente mediante un Deployment y un PVC de tipo `local-path`.

Esta solución permite demostrar persistencia, pero no ofrece por sí sola alta disponibilidad ni recuperación ante la pérdida del nodo que aloja el almacenamiento.

Para un entorno productivo sería conveniente utilizar una base de datos administrada o una arquitectura específica con StatefulSet, almacenamiento apropiado, replicación, backups y procedimientos de recuperación.

Un StatefulSet mejora la identidad estable y la administración del almacenamiento, pero no implementa automáticamente la replicación ni la alta disponibilidad del motor MySQL.

### 8.6. Seguridad y disponibilidad

También sería necesario incorporar HTTPS mediante TLS, políticas de acceso a la red, controles de permisos, pruebas automatizadas y una estrategia de recuperación ante fallas.

Las decisiones deberían basarse en los requisitos reales de disponibilidad, volumen de tráfico y criticidad de la información.

---

## 9. Conclusión

El proyecto permitió integrar los conceptos desarrollados durante las ocho semanas del curso en una aplicación funcional de tres capas.

Las decisiones arquitectónicas fundamentales fueron:

1. Separar la configuración de las imágenes de contenedor.
2. Utilizar Nginx como punto de entrada y proxy de la API.
3. Centralizar la garantía de integridad de las reservas en MySQL.
4. Incorporar almacenamiento persistente para conservar los datos.
5. Escalar horizontalmente la capa de API mediante HPA.
6. Parametrizar y administrar los doce objetos mediante Helm.
7. Verificar el ciclo de instalación, actualización y reversión de una release.

La arquitectura resultante cumple los objetivos del entorno académico y permite identificar con claridad las mejoras necesarias para una implementación productiva.

Las pruebas experimentales y los resultados que respaldan estas decisiones se documentan en `INFORME.md` y en el historial de entregas del `README.md`.
