# Tablón de notas

Aplicación web mínima (Flask + Redis) que muestra un tablón de notas. Las notas
se guardan en Redis y se conservan entre reinicios gracias a un volumen de
Docker. La imagen se construye con `python:3.12-slim`, corre con Gunicorn y
arranca con un usuario sin privilegios.

- **Servicio web:** Flask servido por Gunicorn en el puerto `8000`.
- **Servicio de caché/datos:** Redis 7.
- **Comunicación interna:** el servicio `web` se conecta a Redis por el nombre
  de servicio `cache` dentro de la red interna de Compose (Redis **no** se
  publica al host).

---

## 1. Requisitos previos

- **Docker** (versión 20.10 o superior).
- **Docker Compose v2** (se invoca como `docker compose`, no `docker-compose`).
- Puerto **8000** libre en el host.
- (Opcional) `curl` en el host para las pruebas de comprobación.

Verificación rápida:

```bash
docker --version
docker compose version
```

No hace falta tener Python ni Redis instalados en el host: todo corre en
contenedores.

---

## 2. Variables de entorno necesarias

El `docker-compose.yml` define las siguientes variables para el servicio `web`:

| Variable | Obligatoria | Valor por defecto | Descripción |
|----------|-------------|-------------------|-------------|
| `CLAVE_ADMIN` | **Sí** | — (el compose falla si no se define) | Clave de administración de la aplicación. |
| `REDIS_URL` | No | `redis://cache:6379` | URL de conexión a Redis. |
| `REDIS_HOST` | No | `cache` | Host de Redis (informativo). |
| `REDIS_PORT` | No | `6379` | Puerto de Redis (informativo). |
| `TITULO_APP` | No | `Tablón de notas` | Título mostrado en la página. |

Como `CLAVE_ADMIN` no debe quedar escrita en el repositorio, se pasa desde un
archivo `.env` (que está excluido en `.gitignore` y `.dockerignore`).

Crea un archivo `.env` en la raíz del proyecto:

```bash
cat > .env <<'EOF'
CLAVE_ADMIN=cambia-esta-clave-por-una-segura
EOF
```

> **Seguridad:** no subas `.env` al repositorio ni reutilices contraseñas
> débiles. Para producción, usa un gestor de secretos (Docker secrets, Vault,
> etc.) en lugar de variables de entorno en texto plano.

---

## 3. Pasos para la puesta en marcha

1. Clona/entra en el directorio del proyecto:

   ```bash
   cd practica_1_docker_openCode/recursos/notas
   ```

2. Crea el archivo `.env` con `CLAVE_ADMIN` (ver sección 2).

3. Construye y levanta los servicios en segundo plano:

   ```bash
   docker compose up -d --build
   ```

4. Comprueba que los servicios están arriba y saludables:

   ```bash
   docker compose ps
   ```

5. Abre la aplicación en el navegador:

   ```
   http://localhost:8000
   ```

Para reconstruir solo la imagen web tras cambios en el código:

```bash
docker compose build web
docker compose up -d web
```

---

## 4. Pruebas de comprobación

**Estado de los servicios** (deben aparecer como `running`/`healthy`):

```bash
docker compose ps
```

**Endpoint de salud de la aplicación** (responde `{"estado":"ok",...}` con 200):

```bash
curl -fsS http://localhost:8000/salud
```

**Comprobar que la web responde:**

```bash
curl -I http://localhost:8000/
```

**Crear una nota y verla reflejada:**

```bash
curl -fsS -X POST -d "texto=Nota de prueba" http://localhost:8000/notas
curl -fsS http://localhost:8000/
```

**Comprobar que Redis responde por dentro de la red interna:**

```bash
docker compose exec cache redis-cli ping
# Debe responder: PONG
```

**Comprobar que el contenedor NO corre como root** (debe devolver `appuser`):

```bash
docker compose exec web whoami
```

---

## 5. Dónde se guardan los datos persistentes

- Las notas se almacenan en **Redis**, en la clave de lista `notas`.
- Redis persiste en disco mediante **AOF** (`--appendonly yes`) dentro del
  contenedor en la ruta `/data`.
- Ese directorio está respaldado por el **volumen nombrado `datos-redis`**,
  gestionado por Docker, por lo que los datos **sobreviven** a
  `docker compose down` y a la recreación de contenedores.

Dónde vive físicamente el volumen en el host (ruta gestionada por Docker):

```bash
docker volume inspect notas_datos-redis
```

> El prefijo (`notas_`) corresponde al nombre del proyecto Compose (nombre del
> directorio). Ajusta el comando si tu carpeta se llama distinto; puedes listar
> los volúmenes con `docker volume ls`.

Ver el contenido de la clave `notas`:

```bash
docker compose exec cache redis-cli lrange notas 0 -1
```

> **Importante:** los datos solo se pierden si borras el volumen explícitamente
> (ver sección 7, comando con `-v`).

---

## 6. Comandos para ver los registros (logs)

**Logs de todos los servicios (en vivo):**

```bash
docker compose logs -f
```

**Logs solo del servicio web:**

```bash
docker compose logs -f web
```

**Logs solo de Redis:**

```bash
docker compose logs -f cache
```

**Últimas 100 líneas y sin seguir el flujo:**

```bash
docker compose logs --tail=100 web
```

**Logs con marcas de tiempo:**

```bash
docker compose logs -ft web
```

---

## 7. Comandos exactos para parar y limpiar todo el entorno

**Parar los servicios conservando los datos** (los contenedores se eliminan,
el volumen `datos-redis` se conserva):

```bash
docker compose down
```

**Parar y eliminar también el volumen (⚠️ borra todas las notas):**

```bash
docker compose down -v
```

**Parar sin eliminar los contenedores** (solo detener):

```bash
docker compose stop
```

**Reanudar tras un `stop`:**

```bash
docker compose start
```

**Limpieza total del entorno de este proyecto** (contenedores, red, volumen e
imágenes construidas por Compose):

```bash
docker compose down -v --rmi local
```

**Eliminar además imágenes huérfanas y recursos no usados de Docker:**

```bash
docker compose down -v --rmi local --remove-orphans
docker system prune -f
```

> `docker system prune -f` afecta a **todo** Docker en la máquina, no solo a
> este proyecto. Úsalo con precaución.
