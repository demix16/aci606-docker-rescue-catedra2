# Bitácora técnica de diagnóstico, reparaciones y validación — ACI606

**Asignatura:** Especialidad II: Cloud Computing (ACI606)
**Evaluación:** Cátedra 2 — Docker Rescue Challenge
**Proyecto:** `demix16/aci606-docker-rescue-catedra2`
**Repositorio:** https://github.com/demix16/aci606-docker-rescue-catedra2
**Entorno de pruebas:** GitHub Codespaces, Docker Engine y Docker Compose
**Fecha de pruebas documentadas:** 22 de noviembre de 2026

> **Alcance y trazabilidad.** En esta bitácora registramos los problemas encontrados durante la recuperación del proyecto, sus causas, los cambios realizados y las pruebas que permitieron verificar cada corrección. Como respaldo utilizamos las salidas de terminal y los logs observados en GitHub Codespaces, junto con el historial de commits del repositorio.

## 1. Arquitectura del sistema

- **Frontend:** Nginx, sirve `index.html` en el puerto 80 dentro del contenedor y publica `8080:80` mediante Compose.
- **Backend:** Python 3.12, FastAPI, Uvicorn, SQLAlchemy y `psycopg2-binary`, escucha en el puerto 8000 interno.
- **Base de datos:** PostgreSQL 16, servicio Compose `db`, puerto 5432 interno.
- **Comunicación:** navegador/cliente → Nginx (`/api/`) → FastAPI → PostgreSQL.
- **Persistencia final:** volumen nombrado `postgres_data` montado en `/var/lib/postgresql/data`.
- **Configuración:** `compose.yaml` obtiene parámetros de `.env`; `.env.example` sirve de plantilla pública.

## 2. Método de diagnóstico utilizado

Para cada falla se siguió la secuencia **síntoma → evidencia → causa raíz → reparación → justificación → verificación → impacto**. Las fallas se corrigieron progresivamente para conservar el historial en Git y observar el siguiente error real solo cuando el anterior estaba superado.

Comandos principales utilizados:

```bash
docker compose config --quiet
docker compose up --build -d
docker compose ps -a
docker compose logs --tail=80 backend
docker compose logs --tail=80 frontend
docker compose logs --tail=80 db
curl -i --max-time 5 http://127.0.0.1:8080/api/health
```

## 3. Incidencias y reparaciones — Sección 6 de la pauta

### R-01 — Punto de entrada ASGI incorrecto

1. **Síntoma:** el contenedor `backend` se detuvo con estado `Exited (1)` después de la construcción inicial.
2. **Evidencia inicial:** `docker compose logs --tail=80 backend` mostró `ERROR: Error loading ASGI app. Could not import module "app".`
3. **Causa raíz:** el archivo `backend/Dockerfile` invocaba `uvicorn app:app`, pero la aplicación FastAPI está definida en `backend/main.py` como `app`.
4. **Elemento modificado:** `backend/Dockerfile`; instrucción `CMD` que inicia Uvicorn.
5. **Reparación realizada:** en `backend/Dockerfile` se sustituyó `app:app` por `main:app` en `CMD`.
6. **Justificación:** Uvicorn debe importar el módulo Python `main` y obtener su objeto `app`; antes intentaba importar un módulo inexistente.
7. **Verificación:** después de `docker compose up --build -d`, desapareció `Could not import module "app"`. La ejecución avanzó y reveló una dependencia no instalada (`psycopg2`).
8. **Impacto si no se corrige:** la API no inicia y Nginx no puede completar ninguna operación `/api/`.
9. **Trazabilidad Git:** se registró el cambio mediante el commit `fix(R-01)` en `git log --oneline`.

### R-02 — Controlador PostgreSQL ausente en el backend

1. **Síntoma:** el backend siguió detenido tras corregir el módulo de Uvicorn.
2. **Evidencia inicial:** `docker compose logs --tail=80 backend` mostró `ModuleNotFoundError: No module named 'psycopg2'`.
3. **Causa raíz:** `backend/requirements.txt` incluía SQLAlchemy, pero no instalaba el controlador DBAPI requerido para la URL `postgresql://` usada en el proyecto.
4. **Elemento modificado:** `backend/requirements.txt`; listado de dependencias de Python.
5. **Reparación realizada:** se agregó `psycopg2-binary` a `backend/requirements.txt`.
6. **Justificación:** SQLAlchemy utiliza el controlador `psycopg2` para establecer conexiones PostgreSQL bajo el dialecto configurado.
7. **Verificación:** la imagen reconstruida dejó de registrar `ModuleNotFoundError` y alcanzó la conexión PostgreSQL; surgió entonces `Connection refused` hacia `localhost:5432`.
8. **Impacto si no se corrige:** la aplicación no llega a inicializar el motor de acceso a la base de datos.
9. **Trazabilidad Git:** se registró el cambio mediante el commit `fix(R-02)`.

### R-03 — Host de base de datos incorrecto

1. **Síntoma:** el backend falló al intentar conectar a `localhost` por el puerto 5432.
2. **Evidencia inicial:** los logs mostraron `connection to server at "localhost" ... port 5432 failed: Connection refused`.
3. **Causa raíz:** `localhost` dentro del contenedor backend representa al propio backend, no al contenedor de PostgreSQL.
4. **Elemento modificado:** `compose.yaml`; variable `DATABASE_URL` del servicio `backend`.
5. **Reparación realizada:** en `compose.yaml`, se cambió el host de `DATABASE_URL` de `localhost` a `db`, conservando el puerto `5432` y el nombre de base `appdb`.
6. **Justificación:** Docker Compose proporciona resolución DNS por nombre del servicio en la red del proyecto; `db` identifica al contenedor PostgreSQL.
7. **Verificación:** el backend llegó al servidor `db` (IP de red Docker) y el mensaje cambió a `FATAL: password authentication failed for user "appuser"`, lo que confirmó conectividad y dejó expuesta la siguiente falla.
8. **Impacto si no se corrige:** FastAPI no puede conectar a PostgreSQL y no completa su arranque.
9. **Trazabilidad Git:** se registró el cambio mediante el commit `fix(R-03)`.

### R-04 — Credenciales inconsistentes entre FastAPI y PostgreSQL

1. **Síntoma:** la base de datos rechazó el intento de autenticación desde el backend.
2. **Evidencia inicial:** `docker compose logs --tail=80 backend` mostró `FATAL: password authentication failed for user "appuser"`.
3. **Causa raíz:** la contraseña especificada en `DATABASE_URL` no coincidía con la configurada inicialmente para PostgreSQL en `POSTGRES_PASSWORD`.
4. **Elemento modificado:** `compose.yaml`; credencial de conexión utilizada en `DATABASE_URL`.
5. **Reparación realizada:** se alineó la contraseña usada por el backend con la ya configurada en PostgreSQL. El valor específico de esa contraseña no se reproduce aquí.
6. **Justificación:** la autenticación requiere credenciales válidas del usuario definido en la base de datos; cambiar exclusivamente la URL del backend permite mantener los datos existentes.
7. **Verificación:** `docker compose ps -a` mostró `backend Up` y los logs reportaron `Application startup complete` y `Uvicorn running on http://0.0.0.0:8000`.
8. **Impacto si no se corrige:** el backend no inicia, aunque el servidor PostgreSQL esté en línea.
9. **Trazabilidad Git:** el cambio quedó registrado como `fix(R-04)`. El commit `b7a0e51` tiene un mensaje similar por error, pero corresponde a R-05, como se explica en la sección 6.

### R-05 — Publicación incorrecta del puerto del frontend

1. **Síntoma:** el frontend no se podía consultar correctamente por el puerto 8080 del Codespace.
2. **Evidencia inicial:** `compose.yaml` publicaba `8080:8080` mientras `frontend/nginx.conf` tenía `listen 80;`. La consulta mediante `curl` falló antes de corregir el mapeo.
3. **Causa raíz:** el puerto **interno** de destino no coincidía con el puerto real de escucha del proceso Nginx.
4. **Elemento modificado:** `compose.yaml`; publicación de puertos del servicio `frontend`.
5. **Reparación realizada:** en `compose.yaml`, se reemplazó `8080:8080` por `8080:80`.
6. **Justificación:** el puerto 8080 del anfitrión debe redirigirse al puerto 80 del contenedor Nginx.
7. **Verificación:** `curl -i http://127.0.0.1:8080/` devolvió `HTTP/1.1 200 OK` y el HTML del frontend.
8. **Impacto si no se corrige:** la página web no queda disponible a través del puerto publicado esperado.
9. **Trazabilidad Git:** el commit `b7a0e51` contiene esta corrección, **aunque su mensaje indica por error R-04**; se conserva y documenta la aclaración.

### R-06 — Proxy inverso Nginx apuntando al contenedor equivocado

1. **Síntoma:** la página HTML respondía, pero `/api/health` devolvía `HTTP/1.1 502 Bad Gateway`.
2. **Evidencia inicial:** `curl -i --max-time 5 http://127.0.0.1:8080/api/health` devolvió 502; los logs de Nginx mostraron errores `connect() failed (111: Connection refused) while connecting to upstream` hacia `localhost:8000`.
3. **Causa raíz:** `frontend/nginx.conf` configuraba `proxy_pass http://localhost:8000/;`; ese `localhost` identifica al propio contenedor Nginx y no a FastAPI.
4. **Elemento modificado:** `frontend/nginx.conf`; directiva `proxy_pass` dentro de `location /api/`.
5. **Reparación realizada:** se sustituyó por `proxy_pass http://backend:8000/;`, conservando la barra final `/`.
6. **Justificación:** Compose permite resolver `backend` dentro de la red interna. La barra final facilita la transformación `/api/health` → `/health` con esta ubicación Nginx.
7. **Verificación:** `curl -i http://127.0.0.1:8080/api/health` devolvió HTTP 200 y `{"status":"ok","database":"reachable"}`; `GET /api/items` respondió HTTP 200.
8. **Impacto si no se corrige:** la interfaz estática carga, pero las operaciones dependientes de la API fallan.
9. **Trazabilidad Git:** el cambio se registró en un commit identificado con `fix(R-06)`.

## 4. Mejoras obligatorias — Sección 7 de la pauta

### M-01 — Persistencia mediante volumen nombrado

- **Implementación:** `compose.yaml` declara el volumen `postgres_data` y lo monta en `/var/lib/postgresql/data` del servicio `db`.
- **Migración controlada:** se creó el respaldo lógico `respaldo_antes_volumen.sql` con `pg_dump`; se detuvieron frontend y backend, se recreó `db` con el volumen y se restauró el respaldo con `psql`.
- **Comprobación:** `SELECT id, name, created_at FROM items ORDER BY id;` mostró el registro ID 1, `Prueba ACI606`. Después de `docker compose up -d --no-deps --force-recreate db`, el mismo registro permaneció. La API volvió a listarlo tras levantar los servicios.
- **Trazabilidad Git:** commit `7eb06c1` — `feat(persistencia): agregar volumen nombrado PostgreSQL`.
- **Evidencia documentada:** la consulta SQL y la respuesta de `GET /api/items` mostraron el registro después de recrear el servicio `db`; `docker volume ls` confirmó la presencia del volumen del proyecto.
- **Observación real:** durante la migración se ejecutó una segunda restauración por error y apareció `relation "items" already exists`. La primera restauración ya había creado la tabla y los datos; después comprobamos que el registro seguía presente.

### M-02 — Externalización de configuración

- **Implementación:** `compose.yaml` consume `DATABASE_URL`, `POSTGRES_USER`, `POSTGRES_PASSWORD` y `POSTGRES_DB` desde variables externas; `.env` está excluido en `.gitignore`; `.env.example` contiene marcadores de ejemplo.
- **Comprobación:** `git check-ignore -v .env`, `docker compose config --quiet` y `git status --short` confirmaron exclusión, sintaxis y archivos versionados esperados. Tras `docker compose up -d --force-recreate`, se observaron los servicios `Up`, `/api/health` saludable y `Prueba ACI606` conservada.
- **Trazabilidad Git:** commit `6a191bc` — `feat(config): externalizar variables de entorno`.
- **Evidencia documentada:** `docker compose config --quiet` terminó sin errores; la verificación de `.gitignore` confirmó que `.env` estaba excluido, y la API mantuvo el acceso a PostgreSQL tras recrear los contenedores.
- **Limitación identificada:** aunque Compose obtiene `DATABASE_URL` desde `.env`, `backend/main.py` conserva una URL de conexión de respaldo con una contraseña de pruebas. En la ejecución actual de Compose se utiliza la variable configurada, pero sería más seguro retirar ese valor por defecto. Además, la externalización no elimina las credenciales que aparecieron en commits anteriores; no deben reutilizarse fuera de este ejercicio.

### M-03 — Healthcheck del backend

- **Implementación:** `healthcheck` en `compose.yaml` que usa Python `urllib.request` para consultar `http://127.0.0.1:8000/health`; `interval: 10s`, `timeout: 5s`, `retries: 5`, `start_period: 15s`.
- **Comprobación:** `docker compose ps` mostró el backend como `Up (healthy)` y los otros dos servicios `Up`.
- **Trazabilidad Git:** commit `8be1429` — `feat(healthcheck): agregar comprobacion de salud al backend`.
- **Evidencia documentada:** la salida de `docker compose ps` registró el estado `Up (healthy)` para el backend después de aplicar el cambio.

### M-04 — README reproducible

- **Implementación:** `README.md` técnico con arquitectura, requisitos, `.env`, construcción, API REST, persistencia, diagnóstico, fallo controlado y recuperación.
- **Comprobación:** se verificó la presentación de tablas y diagrama Mermaid en vista previa; el contenido de la interrupción documenta los resultados observados reales (`curl (28)` y `499` en Nginx) y la recuperación.
- **Trazabilidad Git:** commit `baabd2b` — `docs(readme): documentar arquitectura pruebas y recuperacion`.
- **Evidencia documentada:** archivo `README.md` publicado en GitHub y commit `baabd2b`; se comprobó también la representación del diagrama Mermaid y la explicación del fallo controlado.

## 5. Pruebas obligatorias — Sección 8 de la pauta

| ID | Prueba | Procedimiento ejecutado | Resultado observado | Estado |
|---|---|---|---|---|
| P-01 | Arranque limpio de contenedores | `docker compose down` y luego `docker compose up --build -d` | Tres servicios `Up`, backend `healthy`, HTTP 200 en `/api/health` | Superada en el Codespace existente |
| P-02 | Comunicación integral | `GET /`, `GET /api/health`, `POST /api/items`, `GET /api/items` | HTML 200, salud 200, POST 201 y registro ID 1 consultable | Superada |
| P-03 | Persistencia | Crear y respaldar registro; montar `postgres_data`; recrear contenedor `db`; consultar otra vez | `Prueba ACI606` conservó ID, nombre y fecha | Superada |
| P-04 | Fallo controlado | `docker compose stop backend` y solicitar `/api/health` | Backend `Exited (0)` mientras frontend y db seguían `Up`; cliente agotó espera (`curl (28)`) | Superada |
| P-05 | Diagnóstico | `docker compose ps -a`, `docker compose logs --tail=50 frontend`, `curl` | Nginx registró `GET /api/health ... 499` al cerrarse la conexión del cliente | Superada |
| P-06 | Recuperación | `docker compose start backend`, `docker compose ps` y `curl /api/health` | Backend regresó a `Up (healthy)` y la API respondió HTTP 200 | Superada |

> La prueba P-01 confirma reconstrucción en un Codespace con imágenes y entorno ya disponibles; **no debe presentarse como validación desde un Codespace nuevo o completamente vacío** hasta realizar esa prueba de forma independiente.

### Evidencia del fallo controlado

- **Antes de detener:** `GET /api/health` respondió HTTP 200.
- **Interrupción:** `docker compose stop backend`; `backend` quedó `Exited (0)`, `frontend` y `db` permanecieron `Up`.
- **Síntoma observado:** `curl: (28) Operation timed out after 5002 milliseconds with 0 bytes received`.
- **Registro Nginx:** `GET /api/health HTTP/1.1" 499` — el cliente cerró la conexión antes de obtener una respuesta.
- **Recuperación:** `docker compose start backend`; backend `Up (healthy)`; `/api/health` volvió a HTTP 200 con estado `ok` y base `reachable`.
- **Interpretación:** la indisponibilidad del backend afecta a las rutas `/api/`, aunque el frontend y PostgreSQL permanezcan encendidos. Un `502 Bad Gateway` es posible en otros intentos o condiciones, pero **no fue el resultado HTTP observado en esta prueba controlada**.

## 6. Historial Git y aclaración de R-05

El commit `b7a0e51` fue registrado accidentalmente con el mensaje:

```text
fix(R-04): corregir autenticacion PostgreSQL
```

Sin embargo, **su cambio corresponde a R-05**, la corrección de puertos en `compose.yaml` de `8080:8080` a `8080:80`. Se conserva el historial publicado sin forzar cambios y se declara aquí la diferencia para que el evaluador pueda rastrear la modificación real.

Comandos que permiten consultar el historial de la reparación:

```bash
git log --oneline --decorate -15
git show b7a0e51 -- compose.yaml
git status --short
```

**Cierre de la bitácora.** Con las seis reparaciones y las mejoras descritas, la aplicación quedó operativa en el Codespace utilizado: el frontend se comunicó con la API, FastAPI informó acceso a PostgreSQL, el registro de prueba persistió tras recrear los contenedores y el backend pudo recuperarse después de una interrupción. Las pruebas del arranque se realizaron sobre ese Codespace existente, por lo que no se presenta como comprobada una instalación desde un Codespace totalmente nuevo.
