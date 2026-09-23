# Error #7 — `levantar-db.sh` publicaba el puerto 5433→5433 y Postgres escucha en el 5432 interno

- **Fecha:** 2026-09-23
- **Fase/Módulo:** Módulo 4 (despliegue y conexión de PostgreSQL con el Proxy)

## Síntoma

El `POST /query` del Proxy devolvía 500 al consultar la base de datos:

```
error interno: connection failed: connection to server at "127.0.0.1", port 5433 failed:
server closed the connection unexpectedly
```

En cambio, `./infra/levantar-db.sh status` mostraba la DB "ACTIVA" con usuarios y tablas:
`status` usa `docker exec` (conexión *dentro* del contenedor), que nunca pasa por el mapeo de puertos.

## Causa raíz

`levantar-db.sh` creaba el contenedor con `-p "127.0.0.1:${PORT}:${PORT}"` (PORT=5433), es decir,
publicaba el puerto de HOST 5433 hacia el puerto 5433 del CONTENEDOR. Pero la imagen oficial
`postgres:16-alpine` escucha SIEMPRE en el puerto interno **5432** (no hay env var que lo cambie;
`POSTGRES_PORT` no forma parte de la imagen).

Efecto: el proxy se conectaba al puerto publicada de Docker (`docker-proxy`), que intentaba
reenviar al puerto 5433 interno donde NO había ningún listener → la conexión TCP se cerraba al
instante → libpq reportaba "server closed the connection unexpectedly".

Diagnóstico:

```
docker ps            # 127.0.0.1:5433->5433/tcp  (mapeo a un puerto interno muerto)
docker exec db-tfg ss -ltn   # dentro solo escucha 0.0.0.0:5432
docker inspect db-tfg …      # {"5432/tcp":null,"5433/tcp":[{…host 5433}]}
```

## Solución aplicada

Publicar el puerto del host hacia el 5432 interno del contenedor y recrearlo (el volumen
`tfg-pgdata` conserva datos y roles; `init.sql` NO se re-ejecuta al recrear con el mismo volumen):

```bash
# en infra/levantar-db.sh
-p "127.0.0.1:${PORT}:5432" \

./infra/levantar-db.sh stop
sg docker -c "docker rm db-tfg"   # (o `docker rm` si se tiene permiso)
./infra/levantar-db.sh start
```

Verificación: `docker ps` muestra `127.0.0.1:5433->5432/tcp`, la conexión psycopg desde el host
funciona y `POST /query` del Proxy devuelve filas.

## Lección para la memoria

> Al publicar el puerto de un contenedor hay que distinguir el puerto de HOST del de CONTENEDOR
> (`-p HOST:CONTENEDOR`). La imagen oficial de PostgreSQL expone el 5432 interno aunque el host
> use otro puerto. Un fallo de conexión en un servicio Dockerizado que reporta "server closed
> the connection unexpectedly" se diagnostica comparando `docker ps` (mapeo real) con los
> listeners internos del contenedor (`docker exec … ss -ltn`), no asumiendo que la DB está caída.