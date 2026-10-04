# reservas-app

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

El frontend es el único punto de entrada: nadie le habla a la base
directamente, y a la API le habla el frontend.

## Medición y comparación de imágenes

Para comparar las imágenes se utilizó la columna `CONTENT SIZE` informada por `docker images`, manteniendo la misma métrica en todos los casos.

| Imagen | Imagen base final | Tamaño | Reducción respecto de `reservas-api:naive` |
| --- | --- | ---: | ---: |
| `reservas-api:naive` | `node:20` | 402 MB | — |
| `reservas-api:v1` | `node:20-alpine` | 50 MB | 87,6 % |
| `reservas-frontend:v1` | `nginxinc/nginx-unprivileged:1.27-alpine` | 21 MB | 94,8 % |

La reducción se calculó tomando como referencia la imagen ingenua de la API:

- API definitiva: `(402 MB - 50 MB) / 402 MB × 100 = 87,6 %`
- Frontend: `(402 MB - 21 MB) / 402 MB × 100 = 94,8 %`

### Decisiones adoptadas

- **Uso de `node:20-alpine` en la API:** reemplazar la imagen completa `node:20` por una base mínima redujo considerablemente el tamaño de la imagen final.

- **Construcción multi-etapa de la API:** permitió separar la instalación de dependencias de la etapa de ejecución y copiar a la imagen final únicamente los archivos necesarios para ejecutar la aplicación.

- **Uso de `npm ci --omit=dev`:** instaló únicamente las dependencias de producción definidas en `package-lock.json`, evitando dependencias de desarrollo innecesarias en runtime.

- **Copia de `package.json` y `package-lock.json` antes del código:** permite aprovechar el cache de Docker, de modo que la instalación de dependencias no se repita cuando cambia únicamente el código fuente.

- **Uso de un usuario no privilegiado (`node`) en la API:** no reduce significativamente el tamaño, pero mejora la seguridad al evitar que la aplicación se ejecute como `root`.

- **Multi-stage fuerte en el frontend:** Node, npm, Vite, `node_modules` y el código fuente se utilizan solamente durante la etapa de construcción; a la imagen final se copia únicamente el directorio `dist/`.

- **Uso de `nginxinc/nginx-unprivileged:1.27-alpine`:** proporciona una imagen final mínima para servir los archivos estáticos y, además, ejecuta nginx con un usuario sin privilegios.
