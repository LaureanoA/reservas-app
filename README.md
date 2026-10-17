## Semana 01
### reservas-app

Aplicacion web para la reserva de salas y laboratorios por franja horaria.

El proyecto utiliza una arquitectura de tres capas compuesta por un frontend desarrollado con React + Vite, una API REST desarrollada con Node.js + Express y una base de datos MySQL 8.4.

### Arquitectura

navegador ──► reservas-frontend ──────► reservas-api ──────► reservas-db

		React + Vite 	     Node 20 + Express 	      MySQL 8.4
		nginx :8080 		   :3000 		:3306
		(host 3000)           (host 3001, solo       (sin puerto
						depuración)           publicado)

| 	Capa 	      | 	Imagen                 | Puerto interno | Puerto publicado 	 |
| ---  		      | ---         	 	       | ---  		| ---   	         |
| `reservas-frontend` | React + Vite servido por nginx | 8080 		| 3000   		 |
| `reservas-api`      | Node 20 + Express              | 3000 		| 3001 (solo depuración) |
| `reservas-db`       | MySQL 8.4 		       | 3306 		| — 		         |

El frontend es el único punto de entrada: nadie le habla a la base directamente, y a la API le habla el frontend.

---

## Semana 02
### Imágenes construidas

| Imagen | Imagen base | Tamaño | Reducción |
| --- | --- | ---: | ---: |
| `reservas-api:naive` | `node:20` | 402 MB | — |
| `reservas-api:v1` | `node:20-alpine` | 50 MB | 87,6 % |
| `reservas-frontend:v1` | `nginxinc/nginx-unprivileged:1.27-alpine` | 21 MB | 94,8 % |

Decisiones aplicadas:

- `node:20-alpine` reduce el tamaño de la API respecto de `node:20`.
- El multi-stage permite dejar fuera de la imagen final archivos y herramientas innecesarias.
- `npm ci --omit=dev` instala únicamente las dependencias necesarias en producción.
- Copiar `package.json` y `package-lock.json` antes del código permite aprovechar el cache de construcción.
- La API se ejecuta con el usuario no privilegiado `node`.
- En el frontend, Node y Vite se utilizan solo para generar `dist/`; la imagen final utiliza nginx sin privilegios.

### Configuración de nginx

```dockerfile
ENV API_HOST=reservas-api \
    API_PORT=3000 \
    NGINX_ENVSUBST_FILTER=^API_
```

`NGINX_ENVSUBST_FILTER=^API_` limita la sustitución a las variables `API_*` y evita que `envsubst` reemplace variables propias de nginx como `$uri`, `$host` o `$proxy_add_x_forwarded_for`.

### Verificación del frontend

```bash
docker run --rm reservas-frontend:v1 whoami
```

```text
nginx
```

```bash
docker run --rm reservas-frontend:v1 node --version
```

```text
node: not found
```

La primera prueba confirma que el contenedor no se ejecuta como `root` y la segunda que Node no está presente en la imagen final.

### Análisis de capas

```bash
docker history --format '{{.Size}}\t{{.CreatedBy}}' reservas-api:v1 | grep -v '^0B'
```

```text
24.6kB  COPY --chown=node:node server.js db.js ./
49.2kB  COPY --chown=node:node package.json package-lock.json ./
9.85MB  COPY --chown=node:node /app/node_modules ./node_modules
8.19kB  WORKDIR /app
20.5kB  COPY docker-entrypoint.sh /usr/local/bin/
5.48MB  RUN /bin/sh -c apk add --no-cache ...
130MB   RUN /bin/sh -c addgroup -g 1000 node && ...
9.11MB  ADD alpine-minirootfs...
```

La capa de mayor tamaño es de aproximadamente `130 MB` y pertenece a la imagen base `node:20-alpine`. El código propio representa solo unos pocos KB.

### Cache de construcción

Al modificar únicamente `server.js`:

```text
COPY package.json package-lock.json ./    CACHED
RUN npm ci --omit=dev                     CACHED
COPY server.js db.js ./                   reconstruido
```

Al modificar `package.json`:

```text
COPY package.json package-lock.json ./    reconstruido
RUN npm ci --omit=dev                     reconstruido
```

Los manifiestos se copian antes que el código para que los cambios en `server.js` o `db.js` no invaliden la instalación de dependencias. Si cambia `package.json`, la capa de `npm ci` debe reconstruirse.

### Publicación en Docker Hub

```bash
docker tag reservas-api:v1 laureanoa/reservas-api:v1
docker tag reservas-frontend:v1 laureanoa/reservas-frontend:v1

docker push laureanoa/reservas-api:v1
docker push laureanoa/reservas-frontend:v1
```

Imágenes publicadas:

- `laureanoa/reservas-api:v1`
- `laureanoa/reservas-frontend:v1`

---

## Semana 03
## Orden de arranque

El orden de arranque de la aplicación es:

**red → volumen → base de datos → API → frontend**

- **Red (`reservas-net`)**: debe existir antes de crear los contenedores que se conectan a ella. Si no existe, Docker no puede asociarlos a esa red.
- **Volumen (`reservas-db-data`)**: se crea antes de la base para declarar explícitamente la persistencia. Con `-v`, Docker podría crearlo automáticamente si no existe, por lo que no produciría un error inmediato.
- **Base de datos (`reservas-db`)**: debe estar inicializada y aceptando conexiones antes de iniciar la API. Si la API arranca antes, la conexión con MySQL puede fallar.
- **API (`reservas-api`)**: debe estar disponible antes de iniciar el frontend. Si nginx no puede resolver `reservas-api` al arrancar, el frontend termina con el error `host not found in upstream`.
- **Frontend (`reservas-frontend`)**: se inicia al final, cuando los servicios de los que depende ya están disponibles.

Actualmente este orden debe respetarse manualmente y depende de quien levanta la aplicación. Todavía no existe un archivo ejecutable que describa y automatice estas dependencias; esta limitación se resolverá en la siguiente etapa con Docker Compose.

---

## Semana 4
## Docker Compose y seguridad básica

En esta etapa se reemplazó el levantamiento manual de los tres contenedores realizado en la semana anterior por un único archivo `docker-compose.yml`.

El stack queda compuesto por:

- `reservas-db`: MySQL 8.4 con volumen persistente.
- `reservas-api`: API Node.js.
- `reservas-frontend`: frontend servido por nginx.
- Red: `reservas-net`.
- Volumen: `reservas-db-data`.

### Healthchecks

Se incorporaron healthchecks para comprobar el estado real de cada servicio y no solamente que el contenedor se encuentre iniciado.

#### Base de datos

MySQL utiliza:

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

Se utiliza `$$MYSQL_ROOT_PASSWORD` porque un único `$` sería interpretado por Docker Compose al procesar el archivo. Con `$$`, Compose conserva un `$` literal y la variable es expandida posteriormente dentro del contenedor.

El `start_period: 40s` brinda un período de gracia durante la inicialización de MySQL, evitando marcar el servicio como `unhealthy` mientras la base todavía se está preparando.

Además, se utiliza `127.0.0.1` para forzar una conexión TCP al servidor MySQL definitivo y evitar un posible resultado positivo prematuro mediante el socket local.

#### API

La API utiliza el `fetch()` disponible en Node.js para consultar su endpoint `/health`:

```yaml
healthcheck:
  test:
    - CMD
    - node
    - -e
    - "fetch('http://127.0.0.1:3000/health').then(r => process.exit(r.ok ? 0 : 1)).catch(() => process.exit(1))"
```

No se instaló `curl` únicamente para realizar el healthcheck, ya que Node.js ya dispone de la herramienta necesaria. De esta forma se evita agregar dependencias innecesarias a la imagen.

#### Frontend

El frontend comprueba `/healthz` mediante `wget`, disponible en la imagen Alpine utilizada:

```yaml
healthcheck:
  test:
    - CMD
    - wget
    - -q
    - -O
    - "-"
    - http://127.0.0.1:8080/healthz
```

### Orden de dependencias

Se configuró la siguiente cadena:

```text
reservas-db (healthy)
        ↓
reservas-api (healthy)
        ↓
reservas-frontend
```

La API depende de MySQL con:

```yaml
depends_on:
  reservas-db:
    condition: service_healthy
```

Esperar solamente a que el contenedor de MySQL esté iniciado no garantiza que la base ya acepte conexiones. Por eso la API espera a que la DB se encuentre realmente saludable.

La API mantiene además su propia lógica de reintentos de conexión, importante para futuros entornos como Kubernetes, donde no existe `depends_on`.

El frontend depende de la API:

```yaml
depends_on:
  reservas-api:
    condition: service_healthy
```

Esto evita que nginx arranque antes de que exista y esté disponible el upstream `reservas-api`, reduciendo errores iniciales como `host not found in upstream` o respuestas `502`.

### Persistencia: `down` vs. `down -v`

Se comprobó experimentalmente la diferencia entre:

```bash
docker compose down
```

y:

```bash
docker compose down -v
```

`docker compose down` eliminó los contenedores y la red, pero conservó `reservas-db-data`. Al ejecutar nuevamente `docker compose up -d`, las reservas anteriores continuaban almacenadas.

En cambio, `docker compose down -v` eliminó también el volumen. Al volver a levantar el stack, Compose creó un volumen nuevo y MySQL comenzó con una base nueva.

Por lo tanto:

```text
docker compose down
→ elimina contenedores y red
→ conserva volumen y datos

docker compose down -v
→ elimina contenedores, red y volumen
→ elimina los datos persistentes
```

### Variables de entorno y secretos

Se agregó `.env.example` como contrato de las variables requeridas por las tres capas del sistema.

Los valores reales se almacenan localmente en `.env`, archivo excluido mediante `.gitignore`, mientras que `docker-compose.yml` referencia las variables mediante `${VARIABLE}`. De esta forma, las credenciales no quedan versionadas directamente en el repositorio.

`.env.example` sí se versiona porque contiene únicamente valores de ejemplo y permite conocer qué variables necesita la aplicación.

Esta solución evita publicar secretos en Git, aunque las variables cargadas siguen siendo accesibles para Docker durante la ejecución. En Kubernetes esta separación se realizará mediante `ConfigMap` para configuración no sensible y `Secret` para información sensible.

### Revisión básica de seguridad

#### Usuarios de ejecución

Se verificaron los usuarios de los contenedores personalizados:

```text
reservas-api:
uid=1000(node) gid=1000(node)

reservas-frontend:
uid=101(nginx) gid=101(nginx)
```

Por lo tanto, API y frontend se ejecutan como usuarios no root.

#### Puertos publicados

La configuración resultante es:

| Servicio | Puerto publicado al host | Uso |
|---|---|---|
| `reservas-frontend` | `3000 → 8080` | Punto de entrada de la aplicación |
| `reservas-api` | `3001 → 3000` | Acceso temporal para depuración |
| `reservas-db` | Ninguno | Solo accesible desde la red Docker |

La exposición de `3001` para la API se mantiene únicamente con fines de depuración en esta etapa. En etapas posteriores la API será un servicio interno.

#### Escaneo de vulnerabilidades

Se analizaron las dos imágenes propias mediante Trivy el 16/10/2026.

| Imagen | Sistema base | Paquetes de aplicación/lenguaje | Con `Fixed Version` |
|---|---:|---:|---:|
| `reservas-api:v1` | 51 | 40 `node-pkg` | 21 del sistema + 37 Node.js |
| `reservas-frontend:v1` | 159 | 0 | 73 del sistema |

En la API, Trivy informó para Alpine 32 vulnerabilidades LOW, 15 MEDIUM, 4 HIGH y ninguna CRITICAL. Para los paquetes Node.js detectó 3 LOW, 13 MEDIUM, 22 HIGH y 2 CRITICAL.

En el frontend informó 42 LOW, 76 MEDIUM, 39 HIGH y 2 CRITICAL.

Los resultados de un escaneo pueden cambiar a medida que se publican nuevas vulnerabilidades y versiones corregidas. El objetivo no es asumir que una imagen debe tener cero hallazgos, sino identificar, priorizar y actualizar aquellos componentes para los que existe una versión corregida.

### Principio de mínimo privilegio

Se eliminan inicialmente todas las Linux capabilities de los tres servicios:

```yaml
cap_drop:
  - ALL
```

API y frontend funcionan correctamente sin agregar ninguna capability adicional.

MySQL, en cambio, necesitó recuperar únicamente las capacidades requeridas durante su inicialización. Se determinaron experimentalmente, agregándolas una por una a partir de los errores observados:

```text
setgid: Operation not permitted
→ SETGID

setuid: Operation not permitted
→ SETUID

/var/lib/mysql/...: Permission denied
→ DAC_OVERRIDE
```

La configuración final de la base quedó:

```yaml
cap_drop:
  - ALL

cap_add:
  - SETGID
  - SETUID
  - DAC_OVERRIDE
```

Resumen:

| Servicio | `cap_drop` | `cap_add` |
|---|---|---|
| `reservas-api` | `ALL` | Ninguna |
| `reservas-frontend` | `ALL` | Ninguna |
| `reservas-db` | `ALL` | `SETGID`, `SETUID`, `DAC_OVERRIDE` |

MySQL necesita capacidades adicionales porque durante su arranque debe cambiar de usuario/grupo y acceder al directorio persistente de datos. La API y el frontend pueden ejecutar su funcionalidad sin capacidades adicionales.

### Docker Compose vs. ejecución manual

En la semana 3 el stack requería crear los recursos y levantar cada contenedor manualmente mediante distintos comandos `docker run`.

Con Docker Compose desaparecen de la operación cotidiana:

```text
docker network create ...
docker volume create ...
docker run ... reservas-db
docker run ... reservas-api
docker run ... reservas-frontend
```

También dejan de especificarse manualmente en cada ejecución opciones como `--network`, `-v`, `-e` y `-p`.

En su lugar, `docker-compose.yml` incorpora declarativamente:

```text
services
networks
volumes
environment
ports
healthcheck
depends_on
cap_drop / cap_add
```

El stack completo puede levantarse mediante:

```bash
docker compose up -d
```

y eliminarse con:

```bash
docker compose down
```

Esto hace que la configuración sea reproducible, versionable y más fácil de mantener que una secuencia de comandos manuales.
