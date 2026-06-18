# EA-Deploy_G3

Proyecto de despliegue de ViveBook mediante Docker Compose.

Este repositorio contiene la configuracion necesaria para levantar en una maquina los servicios principales de ViveBook:

- Base de datos MongoDB.
- Backend Node.js/Express.
- Web publica React/Vite.
- BackOffice Angular.

El objetivo es que la maquina de produccion solo necesite Docker, este `compose.yaml` y un `.env` correctamente configurado.

## Estructura

```text
EA-Deploy_G3/
+-- compose.yaml
+-- .env
+-- README.md
```

## Servicios

### mongodb

Servicio de base de datos usando la imagen oficial `mongo:latest`.

- Contenedor: `vivebook_db`.
- No expone el puerto `27017` al host.
- Solo es accesible desde la red interna `backend_internal`.
- Usa el volumen persistente `mongo_data`.

Esto evita que MongoDB quede abierto fuera de Docker.

### backend

Servicio API principal de ViveBook.

- Imagen: `victorpk/vivebook-backend-ea:latest`.
- Contenedor: `vivebook_backend`.
- Puerto host: `1337`.
- Puerto contenedor: `1337`.
- Carga variables desde `.env`.
- Se conecta a MongoDB mediante la red interna.
- Es accesible por Web y BackOffice mediante `frontend_net`.

URL esperada:

```text
https://ea3-api.upc.edu
```

En local, si se levanta directamente en una maquina sin proxy, quedara disponible en:

```text
http://localhost:1337
```

### web

Aplicacion publica React/Vite.

- Imagen: `victorpk/vivebook-web-ea:latest`.
- Contenedor: `vivebook_web`.
- Puerto host: `80`.
- Puerto contenedor: `80`.

En local:

```text
http://localhost
```

En produccion deberia exponerse mediante el dominio configurado para la Web.

### backoffice

Panel administrativo Angular.

- Imagen: `victorpk/vivebook-backoffice-ea:latest`.
- Contenedor: `vivebook_backoffice`.
- Puerto host: `8080`.
- Puerto contenedor: `80`.

En local:

```text
http://localhost:8080
```

## Redes Docker

El compose define dos redes:

### backend_internal

Red interna usada por Backend y MongoDB.

```yaml
internal: true
```

Esto significa que los contenedores conectados a esta red no quedan expuestos directamente al exterior.

### frontend_net

Red usada para que Web y BackOffice puedan convivir con el Backend dentro del despliegue.

## Volumenes

### mongo_data

Volumen persistente de MongoDB.

Gracias a este volumen, los datos de la base de datos sobreviven aunque se apaguen o recrean los contenedores.

No eliminar este volumen salvo que se quiera borrar la base de datos.

## Variables de entorno

El archivo `.env` contiene la configuracion compartida del despliegue.

Variables principales:

| Variable | Uso |
| --- | --- |
| `MONGO_USERNAME` | Usuario administrativo de MongoDB, si se usa autenticacion. |
| `MONGO_PASSWORD` | Password administrativo de MongoDB, si se usa autenticacion. |
| `MONGO_URI` | URI usada por el Backend para conectar con MongoDB. |
| `SERVER_PORT` | Puerto interno del Backend. |
| `JWT_ACCESS_SECRET` | Secreto para firmar access tokens. |
| `JWT_REFRESH_SECRET` | Secreto para firmar refresh tokens. |
| `JWT_ACCESS_EXPIRES_IN` | Duracion del access token. |
| `JWT_REFRESH_EXPIRES_IN` | Duracion del refresh token. |
| `SWAGGER_URL` | URL usada por Swagger/OpenAPI. |
| `VITE_API_URL` | URL publica del Backend usada por la Web durante la construccion de imagen. |
| `VITE_SOCKET_URL` | URL publica de Socket.IO usada por la Web durante la construccion de imagen. |
| `BACKOFFICE_API_URL` | URL publica del Backend usada por el BackOffice durante la construccion de imagen. |
| `CLOUDINARY_API_KEY` | API key publica de Cloudinary. |
| `CLOUDINARY_SECRET` | API secret privada de Cloudinary, usada solo por Backend. |
| `CLOUDINARY_NAME` | Nombre de nube de Cloudinary. |

Importante:

- En produccion, `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET` y `CLOUDINARY_SECRET` deben tener valores reales y privados.
- No reutilizar secretos temporales en produccion.
- Las variables `VITE_*` y `BACKOFFICE_API_URL` deben apuntar a la API publica accesible desde el navegador.
- Para este despliegue se espera que la API publica sea:

```text
https://ea3-api.upc.edu
```

## Requisitos previos

En la maquina de despliegue debe estar instalado:

- Docker.
- Docker Compose v2.
- Acceso a internet para descargar las imagenes desde Docker Hub.
- Puertos disponibles:
  - `80` para Web.
  - `8080` para BackOffice.
  - `1337` para Backend, salvo que se use un proxy inverso.

Comprobar instalacion:

```bash
docker --version
docker compose version
```

## Pasos para levantar el despliegue

### 1. Entrar en el proyecto

```bash
cd EA-Deploy_G3
```

### 2. Revisar el `.env`

Antes de arrancar, comprobar que el `.env` contiene los valores correctos para produccion.

Especialmente:

```env
MONGO_URI=mongodb://mongodb:27017/vivebook_DB
SERVER_PORT=1337
JWT_ACCESS_SECRET=...
JWT_REFRESH_SECRET=...
VITE_API_URL=https://ea3-api.upc.edu
VITE_SOCKET_URL=https://ea3-api.upc.edu
BACKOFFICE_API_URL=https://ea3-api.upc.edu
CLOUDINARY_API_KEY=...
CLOUDINARY_SECRET=...
CLOUDINARY_NAME=...
```

### 3. Descargar imagenes

```bash
docker compose pull
```

### 4. Levantar todos los servicios

```bash
docker compose up -d
```

### 5. Comprobar contenedores

```bash
docker compose ps
```

Deberian aparecer activos:

```text
vivebook_db
vivebook_backend
vivebook_web
vivebook_backoffice
```

### 6. Revisar logs

Logs generales:

```bash
docker compose logs -f
```

Logs de un servicio concreto:

```bash
docker compose logs -f backend
docker compose logs -f mongodb
docker compose logs -f web
docker compose logs -f backoffice
```

### 7. Verificar acceso

Backend:

```text
http://localhost:1337
```

Web:

```text
http://localhost
```

BackOffice:

```text
http://localhost:8080
```

En produccion, si existe proxy inverso o balanceador, deberia apuntar a estos servicios segun la configuracion de la maquina.

## Actualizar despliegue

Cuando haya nuevas imagenes publicadas en Docker Hub con etiqueta `latest`:

```bash
docker compose pull
docker compose up -d
```

Docker descargara las nuevas imagenes y recreara los contenedores que hayan cambiado.

Para revisar que imagen esta usando cada contenedor:

```bash
docker compose images
```

## Parar servicios

Parar contenedores sin borrar datos:

```bash
docker compose down
```

Esto detiene y elimina los contenedores, pero mantiene el volumen `mongo_data`.

## Borrar completamente la base de datos

Solo si se quiere eliminar toda la informacion persistida:

```bash
docker compose down -v
```

Advertencia: este comando elimina el volumen `mongo_data` y, por tanto, los datos de MongoDB.

## Comandos utiles

Reiniciar un servicio:

```bash
docker compose restart backend
```

Ver uso de recursos:

```bash
docker stats
```

Entrar en un contenedor:

```bash
docker exec -it vivebook_backend sh
docker exec -it vivebook_db mongosh
```

Ver redes creadas:

```bash
docker network ls
```

Ver volumenes:

```bash
docker volume ls
```

## Consideraciones de produccion

- MongoDB no esta expuesto al exterior, lo cual es correcto.
- El Backend si expone el puerto `1337`; en produccion puede dejarse asi o protegerse detras de un proxy inverso.
- Web ocupa el puerto `80`.
- BackOffice ocupa el puerto `8080`.
- Los secretos del `.env` deben mantenerse fuera de repositorios publicos.
- Las imagenes usan la etiqueta `latest`, por lo que actualizar es sencillo, pero conviene controlar bien que version se esta desplegando.
- Si se necesita trazabilidad exacta, se puede cambiar en el futuro a etiquetas versionadas.

## Flujo recomendado de despliegue

1. Publicar nuevas imagenes desde los workflows de GitHub Actions.
2. Entrar en la maquina de produccion.
3. Ir al directorio `EA-Deploy_G3`.
4. Ejecutar:

```bash
docker compose pull
docker compose up -d
docker compose ps
```

5. Revisar logs del Backend:

```bash
docker compose logs -f backend
```

6. Comprobar Web, BackOffice y API desde navegador.

## Resolucion de problemas

### El Backend no conecta con MongoDB

Comprobar:

- Que `mongodb` esta levantado.
- Que `MONGO_URI` usa el nombre del servicio Docker:

```env
MONGO_URI=mongodb://mongodb:27017/vivebook_DB
```

- Que ambos servicios comparten la red `backend_internal`.

### Web o BackOffice no conectan con el Backend

Comprobar:

- Que las imagenes fueron construidas con la URL publica correcta.
- Que las variables son:

```env
VITE_API_URL=https://ea3-api.upc.edu
VITE_SOCKET_URL=https://ea3-api.upc.edu
BACKOFFICE_API_URL=https://ea3-api.upc.edu
```

- Que el dominio `https://ea3-api.upc.edu` apunta correctamente al Backend.
- Que el Backend permite CORS desde los dominios usados por Web y BackOffice.

### Cloudinary falla

Comprobar:

- `CLOUDINARY_API_KEY`.
- `CLOUDINARY_SECRET`.
- `CLOUDINARY_NAME`.

El secret de Cloudinary debe estar solo en Backend/despliegue, nunca en Frontend Web, BackOffice o APP.

### Cambios en `.env` no se aplican

Recrear los contenedores:

```bash
docker compose up -d --force-recreate
```

Si el cambio afecta a variables usadas durante la construccion de imagenes frontend, no basta con cambiar el `.env` del compose: hay que reconstruir y republicar la imagen correspondiente.
