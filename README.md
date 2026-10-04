## Semana 01
## reservas-app

Aplicacion web para la reserva de salas y laboratorios por franja horaria.

El proyecto utiliza una arquitectura de tres capas compuesta por un frontend desarrollado con React + Vite, una API REST desarrollada con Node.js + Express y una base de datos MySQL 8.4.

## Arquitectura

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
