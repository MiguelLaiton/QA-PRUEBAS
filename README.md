# FIXIA — Entorno Docker y guía de pruebas de QA

Guía paso a paso para (1) levantar todo el entorno de FIXIA con Docker y (2) ejecutar las pruebas unitarias y de
integración de QA. Todo corre dentro de contenedores: la única herramienta que necesita la máquina es **Docker
Desktop** (con Docker Compose v2) y **Git**. No hace falta instalar Java, Maven, Python ni Go.

Documentos de referencia: `Fixia_SRS_V4.docx.md` (SRS V4.0), `DD V3.docx.md` (DD V3) y `TD_V1.docx` (documento de
aseguramiento, en esta misma carpeta).

> Los comandos están escritos para **PowerShell** (Windows). Donde cambia, se indica la variante para Bash
> (Linux/macOS/Git Bash). Ejecútalos desde la carpeta raíz del proyecto (`FIXIA-QA`), salvo que se indique otra.

---

## 1. Mapa del proyecto

| Carpeta | Tecnología | Rol | Pruebas Java de QA |
|---|---|---|---|
| `ms-orchestrator` | Docker Compose, Traefik | Orquestación, gateway, infraestructura, observabilidad | — |
| `ms-users` | Java 21 + Spring Boot 3.2 | Identidad: registro de cliente y técnico, login, sesión, perfil profesional (RF-001/002/004/005/007/009) | 291 pruebas |
| `ms-payments` | Java 21 + Spring Boot 3.2 | Pagos (hoy solo el esqueleto de arranque) | 4 pruebas de humo |
| `ms-matching-geo` | Python + FastAPI | Matching y geolocalización | no aplica (no es Java) |
| `ms-intake-lakehouse` | Go | Ingesta Kafka → RustFS | no aplica (no es Java) |
| `ms-frontend` | Flutter web + Nginx | Interfaz web | no aplica (no es Java) |
| `ms-apigateway`, `ms-services` | — | Sin código todavía | — |

---

## 2. Levantar el entorno completo con Docker

### 2.1 Primera vez (clonar y configurar)

```powershell
git clone https://github.com/Grupo-Fixia/ms-orchestrator.git
cd ms-orchestrator
```

El script `setup-dev.sh` clona los demás repositorios, los pasa a `develop`, genera el archivo `.env` (incluidas las
claves JWT RS256 de `ms-users`) y levanta todo con `docker compose --profile full up -d --build`.
En Windows ejecútalo con **Git Bash**:

```powershell
& "C:\Program Files\Git\bin\bash.exe" ./setup-dev.sh
```

En Linux/macOS: `chmod +x setup-dev.sh && ./setup-dev.sh`.

> **Windows:** si el build falla con `docker-credential-desktop: executable file not found`, añade la carpeta de
> Docker Desktop al PATH **de esa sesión** y vuelve a ejecutar:
> `$env:PATH = "$env:LOCALAPPDATA\Programs\DockerDesktop\resources\bin;" + $env:PATH`
>
> No uses `bash` a secas en PowerShell: en Windows abre el stub de WSL, no Git Bash.

### 2.2 Levantar, parar y reiniciar (día a día)

Desde `ms-orchestrator`:

```powershell
# Todo (infraestructura + servicios + frontend)
docker compose --profile full up -d --build

# Todo + monitoreo (Prometheus y Grafana)
docker compose --profile full --profile observability up -d --build

# Solo infraestructura (Postgres, Kafka, RustFS, gateway)
docker compose --profile infra up -d

# Parar (conserva los datos)
docker compose --profile full --profile observability down

# Parar y BORRAR los datos de las bases (empezar de cero)
docker compose --profile full --profile observability down -v
```

> **Importante desde el 8/10/2026 — ajuste local del puerto del front.** DevOps cambió el `nginx.conf` del front de `develop` para escuchar en el puerto **8080**
> (lo necesita Kubernetes), pero el `docker-compose.yml` de `ms-orchestrator` sigue enviando Traefik al **80**: con el compose tal cual, `http://localhost`
> responde **502 Bad Gateway**. Hasta que se corrija, en tu computador usa un archivo de ajuste aparte, **sin editar nada de los repositorios**
> y **sin subirlo a ningún repositorio ni a QA/DevOps** (lo acordó Juan Pablo).
>
> 1. Crea `FIXIA-QA/docker-compose.qa-local.yml` (en la carpeta `FIXIA-QA`, que no es un repo):
>
>    ```yaml
>    services:
>      ms-frontend:
>        labels:
>          - "traefik.http.services.ms-frontend.loadbalancer.server.port=8080"
>    ```
>
> 2. Desde la carpeta `FIXIA-QA` (no desde `ms-orchestrator`), levanta con:
>
>    ```powershell
>    docker compose --project-directory ms-orchestrator -f ms-orchestrator/docker-compose.yml -f docker-compose.qa-local.yml --profile full --profile tools --profile observability up -d --build
>    ```
>
> 3. **Antes de levantar una rama del front, revisa su nginx** (`Select-String ms-frontend\nginx.conf -Pattern listen`):
>    si dice `listen 8080;` usa el ajuste; si dice `listen 80;` (rama de PR sin actualizar con `develop`) **no lo uses**, porque daría 502.
>
> Cuando DevOps corrija la etiqueta `ms-frontend.loadbalancer.server.port` del compose (de `80` a `8080`), borra `docker-compose.qa-local.yml`.
> Los comandos de esta sección 2.2 y de las siguientes se escriben desde `ms-orchestrator` y sin ajuste: con el `develop` del 8/10/2026 cámbialos por el comando de arriba.

### 2.3 Verificar que todo quedó arriba

```powershell
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Deben aparecer en estado `Up` (y `healthy` donde aplica): `fixia-apigateway`, `ms-frontend`, `ms-users`,
`ms-payments`, `ms-matching-geo`, `ms-intake-lakehouse`, `fixia-db-users`, `fixia-db-payments`, `fixia-kafka`,
`fixia-rustfs` y, con el perfil de observabilidad, `fixia-prometheus`, `fixia-blackbox` y `fixia-grafana`.
`fixia-rustfs-init` termina con `Exited (0)`: es un contenedor de inicialización y es lo esperado.

Prueba rápida del gateway:

```powershell
curl.exe -s -o NUL -w "%{http_code}`n" http://localhost/                      # frontend: 200
curl.exe -s -o NUL -w "%{http_code}`n" http://localhost/api/users/me          # sin token: 401
curl.exe -s -o NUL -w "%{http_code}`n" http://localhost/actuator/health       # no expuesto por el gateway: 404
```

### 2.4 Puertos y accesos

| Servicio | URL / puerto | Notas |
|---|---|---|
| Gateway (Traefik) | http://localhost (80) | Punto de entrada: `/` frontend, `/api/users`, `/api/payments`, `/api/geo` |
| Dashboard de Traefik | http://localhost:8080 | Solo desarrollo |
| PostgreSQL users / payments | `127.0.0.1:5432` / `127.0.0.1:5433` | Usuario y contraseña en `ms-orchestrator/.env` |
| Kafka (desde el host) | `127.0.0.1:29092` | |
| RustFS (S3) y consola | `127.0.0.1:9000` / `127.0.0.1:9001` | `rustfsadmin` / `rustfsadmin` por defecto |
| Adminer (visor web de la base de datos) | http://localhost:8081 | Perfil `tools`. Servidor `postgres-users`, usuario `user`, contraseña `password`, base `users_db` |
| Prometheus | http://localhost:9090 | Perfil `observability` |
| Grafana | http://localhost:3000 | `admin` / `admin` (cambiables con `GRAFANA_ADMIN_USER` / `GRAFANA_ADMIN_PASSWORD` en `.env`) |

Con el perfil `observability`, Grafana trae el dashboard **Fixia → Fixia - Disponibilidad** (estado y latencia de cada
servicio). Prometheus sondea 3 endpoints HTTP y 6 puertos TCP mediante `blackbox-exporter`
(configuración en `ms-orchestrator/config/observability/`). Cómo verlo y comprobarlo: sección 2.8.

> Las métricas de CPU/memoria por contenedor (cAdvisor) no están incluidas: su imagen solo se publica en `gcr.io`,
> que no siempre es accesible desde Docker Desktop. Los servicios Spring tampoco exponen aún métricas de aplicación.

### 2.5 Ver la web y comprobar la conexión con la base de datos

> Para una versión corta, paso a paso y con puertos, usa **`GUIA_VERIFICAR_DATOS.md`** (en esta misma carpeta).

> **Estado a 8/10/2026.** La rama `develop` de `ms-frontend` ya incluye la **página de inicio**, el **login**, la **sesión**, el **registro de cliente** (GC-253) y el
> **registro de técnico conectado al back** (GC-255 y GC-256, PR #27 y #28). Usa `develop`, **también en `ms-users`** (el back de técnicos, GC-254/261/262, ya está en `develop`;
> si tu carpeta `ms-users` está en una rama anterior, la web mostrará el formulario pero el endpoint de técnicos no existirá):
>
> ```bash
> git -C ms-frontend fetch --all
> git -C ms-frontend checkout develop && git -C ms-frontend pull
> git -C ms-users fetch --all
> git -C ms-users checkout develop && git -C ms-users pull
> ```
>
> (Si tienes cambios sin commit en `ms-users`, como tus pruebas de QA, guárdalos antes con `git -C ms-users stash -u`, o haz primero el commit de tu rama; sección 7.)
> El recorrido del técnico con la base de datos está en `GUIA_VERIFICAR_DATOS.md`, sección 10. Para comprobar que las pantallas se ven bien, ver la sección 11 de esa guía y el apartado 2.7.

**1. Levantar back + base de datos + front** (desde `ms-orchestrator`):

```powershell
docker compose --profile full up -d --build
```

Si solo cambió el front: `docker compose --profile full up -d --build ms-frontend`. El primer build del front tarda varios
minutos (descarga Flutter 3.22.0 y compila la web).

**2. Abrir la web:** http://localhost — carga la **página de inicio**; desde ahí se llega al **registro de cliente**, al **de técnico** y al **login**, y tras iniciar sesión aparece la pantalla
de **sesión** con nombre, correo y rol.

> En `develop` los formularios de **registro de cliente y de técnico** ya guardan en la base de datos (el técnico queda con verificación `PENDING`). También puedes crear la cuenta por la API (paso 3).

**3. Crear un usuario por la API (a través del gateway) e iniciar sesión en la web con él:**

```powershell
$body = @{ firstName="Ana"; lastName="Prueba"; documentType="CC"; documentNumber="QA900001"; email="ana.prueba@example.com"; phone="+573001112233"; password="Clave1234"; policyVersion="v1.0"; consentAccepted=$true } | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri http://localhost/api/users/clients -ContentType "application/json" -Body $body
```

Ahora en http://localhost entra con `ana.prueba@example.com` / `Clave1234`. Debe mostrar la sesión con rol `CLIENT`; "Cerrar
sesión" vuelve al login, y recargar la página con "mantener sesión" activo conserva la sesión.

**4. Comprobar que los datos llegaron a PostgreSQL** (contenedor `fixia-db-users`, base `users_db`):

```powershell
# Cuenta creada (la contraseña se guarda como hash BCrypt $2a$12$..., nunca en claro)
docker exec fixia-db-users psql -U user -d users_db -c "select email, role, status, consent_policy_version, left(password_hash,7) as hash, created_at from users"

# Sesiones: un refresh token activo por inicio de sesión; al cerrar sesión aparece un access token revocado
docker exec fixia-db-users psql -U user -d users_db -c "select count(*) filter (where revoked_at is null) as activos, count(*) filter (where revoked_at is not null) as revocados from refresh_tokens"
docker exec fixia-db-users psql -U user -d users_db -c "select jti, revoked_at from revoked_tokens"

# Migraciones de Flyway aplicadas y tablas existentes
docker exec fixia-db-users psql -U user -d users_db -c "select version, description, success from flyway_schema_history order by installed_rank"
docker exec fixia-db-users psql -U user -d users_db -c "\dt"
```

Si el usuario aparece en `users` y, tras iniciar y cerrar sesión desde la web, cambian `refresh_tokens` y `revoked_tokens`, el
front, el back y la base de datos están conectados. También puedes abrir la base con un cliente SQL (DBeaver, etc.) en
`127.0.0.1:5432`, base `users_db`, usuario y contraseña de `ms-orchestrator/.env`.

**5. Borrar los datos de prueba:**

```powershell
docker exec fixia-db-users psql -U user -d users_db -c "delete from users where email='ana.prueba@example.com'; delete from revoked_tokens;"
```

(`refresh_tokens` se borra en cascada con el usuario.)

### 2.6 Probar la rama de un PR del back (ms-users)

Sirve para probar la rama de un PR **pendiente** del back. Desde el 8/10/2026 el back de técnicos (GC-254, GC-261 y GC-262) ya está en `develop`, así que su flujo se prueba con `develop` sin cambiar de rama. Ejemplo (cambia el nombre de la rama por la del PR que revises):

```powershell
git -C ms-users fetch --all
git -C ms-users checkout <rama-del-pr>
cd ms-orchestrator
docker compose --profile full up -d --build ms-users
```

Flujo de técnico (registro → login → perfil) a través del gateway:

```powershell
$reg = @{ firstName="Tec"; lastName="Nico"; documentType="CC"; documentNumber="QT1"; email="tec@example.com"; phone="+573001112233"; password="Clave1234"; policyVersion="v1.0"; consentAccepted=$true } | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri http://localhost/api/users/technicians -ContentType "application/json" -Body $reg

$s = Invoke-RestMethod -Method Post -Uri http://localhost/api/users/auth/login -ContentType "application/json" -Body (@{email="tec@example.com";password="Clave1234"}|ConvertTo-Json)
$h = @{ Authorization = "Bearer $($s.accessToken)" }

Invoke-RestMethod -Uri http://localhost/api/users/technicians/me/profile -Headers $h
Invoke-RestMethod -Method Put -Uri http://localhost/api/users/technicians/me/profile -Headers $h -ContentType "application/json" -Body (@{ professionalDescription="Plomero con 5 anios"; yearsOfExperience=5; categories=@("PLUMBING","MAINTENANCE") } | ConvertTo-Json)

docker exec fixia-db-users psql -U user -d users_db -c "select u.email, u.role, t.verification_status, t.professional_description, t.years_of_experience from technicians t join users u on u.id=t.user_id"
docker exec fixia-db-users psql -U user -d users_db -c "select category from technician_categories"
```

Categorías válidas: `PLUMBING`, `ELECTRICAL`, `MAINTENANCE`, `LOCKSMITHING`, `PAINTING`, `CARPENTRY`.

> **Antes de volver a `develop`:** si la rama añade migraciones de Flyway que `develop` no tiene, el `ms-users` de `develop` se negará a arrancar. Para volver, borra los datos de las bases y reconstruye:
> `docker compose --profile full --profile observability down -v`, luego `git -C ms-users checkout develop` y `docker compose --profile full up -d --build`.

> **Registro de técnico en el front:** ya está en `develop` (GC-255 y GC-256, 8/10/2026). Cómo probarlo desde la landing y comprobar `users`, `technicians` y `technician_categories`:
> `GUIA_VERIFICAR_DATOS.md`, sección 10.

### 2.7 Ver que las pantallas se ven bien (inicio, cliente y técnico)

Resumen (detalle completo en `GUIA_VERIFICAR_DATOS.md`, sección 11):

| Pantalla | Dirección |
|---|---|
| Página de inicio | http://localhost/#/ |
| Inicio de sesión | http://localhost/#/login |
| Registro de cliente | http://localhost/#/registro-cliente |
| Registro de técnico | http://localhost/#/registro-tecnico (o, desde la landing: **Registrarse** → **Registrarme como técnico**) |

1. Abre cada pantalla y pulsa `F12`; activa la **vista de dispositivo** (`Ctrl + Shift + M`; Mac: `Cmd + Shift + M`).
2. Revísala en **1280 px** (computador), **768 px** (tableta) y **390 / 360 / 320 px** (celular), y con **zoom al 200 %**.
3. Comprueba: logo visible, sin textos montados ni barra horizontal, mensajes de error **en rojo bajo cada campo** al enviar vacío, colores coherentes entre pantallas, menús desplegables completos y recorrido con **Tab** y **Enter**.
4. Haz el recorrido con datos reales: registrar cliente → cuenta creada → iniciar sesión → cerrar sesión, y mira la base de datos en cada paso (`GUIA_VERIFICAR_DATOS.md`, pasos 4 a 7).
5. Técnico: el formulario ya está conectado al back. Haz el recorrido registrar técnico → cuenta creada → iniciar sesión (*Rol: Técnico*) → cerrar sesión, y mira `users` y `technicians` en la base de datos (`GUIA_VERIFICAR_DATOS.md`, sección 10). El perfil profesional todavía no tiene pantalla: se completa por la API (sección 10.4).

### 2.8 Ver y verificar el monitoreo (Grafana y Prometheus)

El monitoreo mide si cada servicio está arriba y su latencia. Resumen (detalle y solución de problemas en `GUIA_VERIFICAR_DATOS.md`, sección 12):

1. **Levantarlo** (desde `ms-orchestrator`; el perfil `observability` no se activa solo):
   ```powershell
   docker compose --profile full --profile tools --profile observability up -d
   ```
2. **Abrir Grafana:** http://localhost:3000 con usuario `admin` y contraseña `admin`.
3. **Abrir el panel:** http://localhost:3000/d/fixia-overview/fixia-disponibilidad (o **Dashboards → Fixia → Fixia - Disponibilidad**). Con todo sano, cada servicio aparece en verde con valor `1` y se ve su latencia.
4. **Comprobar que mide de verdad:** `docker stop ms-payments`; a los ~15 s el panel lo muestra en rojo (0); `docker start ms-payments` y a los ~20 s vuelve a verde. (Probado el 7/10/2026: 11 s para marcarlo caído y ~20 s para recuperarlo.)
5. **Sin Grafana:** http://localhost:9090/targets (todos **UP**) o, en http://localhost:9090/graph, la consulta `probe_success` (1 = arriba, 0 = caído) y `probe_duration_seconds` (latencia).

Desde el 8/10/2026 la web se sondea en el puerto **8080** (`http://ms-frontend:8080/`), porque el nginx del front cambió de 80 a 8080; si Grafana la muestra en rojo aunque abre, revisa `config/observability/prometheus.yml` y reinicia con `docker restart fixia-prometheus`.

Limitaciones: solo mide disponibilidad y latencia de 9 destinos (3 HTTP y 6 TCP). No hay logs centralizados, alertas ni métricas de la aplicación (`TD_V1.docx`, OBS-03 y OBS-04). Los puertos 3000 y 9090 solo son accesibles desde tu computador (`127.0.0.1`).

### 2.9 Traefik (el gateway): cómo explicarlo y probarlo

Traefik es el único punto de entrada: todo entra por `http://localhost` (puerto 80) y Traefik reparte según la ruta. Detalle completo en `GUIA_VERIFICAR_DATOS.md`, sección 13.

| Ruta | Va a | Prioridad |
|---|---|---|
| `/api/users/...` | `ms-users` (8080) | 10 |
| `/api/payments/...` | `ms-payments` (8080) | 10 |
| `/api/geo/...` | `ms-matching-geo` (8000) | 10 |
| cualquier otra | `ms-frontend` (la web) | 1 |

La prioridad hace que `/api/users` gane sobre `/`, que captura todo lo demás. Las rutas se declaran con `labels` en `ms-orchestrator/docker-compose.yml` y Traefik las descubre solo (`--providers.docker=true`).

**Ver el panel:** http://localhost:8080 → *HTTP Routers*. Con Traefik 3.6.25 debe mostrar 2 entrypoints (`:80` y `:8080`), 6 routers (4 de Fixia y 2 internos), 7 services y 2 middlewares (del propio panel), todo en verde.

**Probarlo** (PowerShell; usa `curl.exe`):

```powershell
curl.exe -i http://localhost/                      # 200, HTML: va a la web
curl.exe -i http://localhost/api/users/me          # 401: llegó a ms-users, que pide token
curl.exe -i -X POST -H "Content-Type: application/json" -d "{}" http://localhost/api/users/clients   # 400: ms-users valida el cuerpo
curl.exe -i http://localhost/api/payments/x        # 404 en JSON: llegó a ms-payments
```

**Descubrimiento automático:** con el panel abierto, `docker stop ms-payments` hace desaparecer su router en segundos y `docker start ms-payments` lo devuelve (probado el 7/10/2026).

**Ojo:** con un servicio apagado, su ruta `/api/...` cae en la regla `/` y responde **200 con HTML** de la web en vez de un error. Es una consecuencia del diseño actual; ver la sección 13.5 de la guía. El panel (`--api.insecure=true`) es solo para desarrollo.

---

## 3. Ejecutar las pruebas (unitarias y de integración)

Las pruebas viven en los repositorios, en `src/test/java`:

- **Unitarias** (etiqueta `@Tag("unit")`): JUnit 5 + Mockito, sin Spring ni base de datos. Se ejecutan en segundos.
- **Integración** (el resto): Spring Boot + MockMvc + base H2 en memoria (modo PostgreSQL) con las migraciones reales de
  Flyway.

Cada comando usa una imagen oficial de Maven y **copia el repositorio dentro del contenedor** antes de ejecutar, de modo
que no se escribe nada (ni `target/`) en tu carpeta. El volumen `fixia-m2` guarda la caché de dependencias para que las
siguientes ejecuciones sean rápidas.

### Cómo se ve la salida (qué evalúa cada prueba y si pasó)

Cada prueba imprime en la terminal una línea con **qué evalúa** (el nombre de la prueba convertido en frase) y su resultado.
Si falla, muestra el **motivo** (lo esperado frente a lo obtenido). Cada clase abre con un encabezado que indica su tipo
(`unit` / `integration`) y el requisito del SRS o del DD que verifica:

```
==== AuthServiceUnitTest [unit] - CA-14 / RF-004: credenciales válidas inician sesión; las inválidas no revelan el motivo del rechazo.
  [PASO]    Con credenciales válidas emite los tokens
  [PASO]    Una cuenta deshabilitada no inicia sesión aunque la contraseña sea correcta
  [FALLO]   El refresh rechaza un token expirado
            Motivo: expected: "valor esperado" | but was: "valor real"
  [OMITIDA] Cerrar sesión varias veces de forma simultánea es idempotente
            Motivo: DEF-QA-01: el logout concurrente con el mismo token responde 409 en lugar de ser idempotente
```

Es automático: lo hacen `support/ReporteDePruebas`, `support/NombresLegibles` y `src/test/resources/junit-platform.properties`,
y también aplica a las pruebas que ya existían. Para que se vean bien las tildes en PowerShell ejecuta antes, en esa ventana:

```powershell
[Console]::OutputEncoding = [Text.Encoding]::UTF8
```

Al final, Maven muestra el total (`Tests run: ..., Failures: ..., Skipped: ...`) y si se cumplió la cobertura.

### 3.1 ms-users

**Solo pruebas unitarias** (144 pruebas):

```powershell
docker run --rm -v "${PWD}\ms-users:/src:ro" -v fixia-m2:/root/.m2 maven:3.9.9-eclipse-temurin-21 sh -c "cp -r /src /w && cd /w && mvn -B test -Dgroups=unit"
```

**Solo pruebas de integración** (todas menos las 144 unitarias; 1 omitida; incluye las que ya existían en el repositorio):

```powershell
docker run --rm -v "${PWD}\ms-users:/src:ro" -v fixia-m2:/root/.m2 maven:3.9.9-eclipse-temurin-21 sh -c "cp -r /src /w && cd /w && mvn -B test -DexcludedGroups=unit"
```

**Todas las pruebas + verificación de cobertura** (291 pruebas sobre el `develop` del 8/10/2026; 1 omitida). `verify` además exige un mínimo de **85 %
de líneas** con JaCoCo y falla el build si no se cumple:

```powershell
docker run --rm -v "${PWD}\ms-users:/src:ro" -v fixia-m2:/root/.m2 maven:3.9.9-eclipse-temurin-21 sh -c "cp -r /src /w && cd /w && mvn -B verify"
```

Resultado esperado sobre el `develop` del 8/10/2026: `Tests run: 291, Failures: 0, Errors: 0, Skipped: 1`, `All coverage checks have been met.` y
`BUILD SUCCESS`. Con el `develop` anterior (sin el back de técnicos) eran 269 pruebas y 4 omitidas, con **98,3 % de líneas y 94,6 % de ramas**; con el back de técnicos la cobertura fue de **98,2 % de líneas** (rama GC-262).

**Una clase o un método concreto** (cambia el nombre):

```powershell
docker run --rm -v "${PWD}\ms-users:/src:ro" -v fixia-m2:/root/.m2 maven:3.9.9-eclipse-temurin-21 sh -c "cp -r /src /w && cd /w && mvn -B test -Dtest=TokenServiceUnitTest"
docker run --rm -v "${PWD}\ms-users:/src:ro" -v fixia-m2:/root/.m2 maven:3.9.9-eclipse-temurin-21 sh -c "cp -r /src /w && cd /w && mvn -B test -Dtest=AuthServiceUnitTest#conContrasenaIncorrectaRechazaYNoEmiteTokens"
```

**Ver el reporte HTML de cobertura** (opcional; esto sí escribe archivos, en la carpeta `qa-reports` que tú elijas):

```powershell
New-Item -ItemType Directory -Force qa-reports | Out-Null
docker run --rm -v "${PWD}\ms-users:/src:ro" -v "${PWD}\qa-reports:/out" -v fixia-m2:/root/.m2 maven:3.9.9-eclipse-temurin-21 sh -c "cp -r /src /w && cd /w && mvn -B verify && cp -r target/site/jacoco /out/ms-users"
start qa-reports\ms-users\index.html
```

### 3.2 ms-payments

Hoy `ms-payments` solo tiene la clase de arranque, así que hay 4 pruebas de humo (el servicio arranca, puerto y nombre
esperados). Cobertura: 100 % de sus 4 líneas, umbral JaCoCo del 85 %.

```powershell
docker run --rm -v "${PWD}\ms-payments:/src:ro" -v fixia-m2:/root/.m2 maven:3.9.9-eclipse-temurin-21 sh -c "cp -r /src /w && cd /w && mvn -B verify"
```

### 3.3 Variante Bash (Linux/macOS)

Cambia `"${PWD}\ms-users:/src:ro"` por `"$(pwd)/ms-users:/src:ro"`. En Git Bash para Windows antepón
`MSYS_NO_PATHCONV=1` y usa `"$(pwd -W)/ms-users:/src:ro"`.

### 3.4 Defecto conocido (prueba omitida)

La prueba `ConcurrencyIntegrationTest.cerrarSesionVariasVecesDeFormaSimultaneaEsIdempotente` está marcada con
`@Disabled` porque documenta el defecto **DEF-QA-01**: dos cierres de sesión simultáneos con el mismo token responden
`409 "Cuenta existente"` en lugar de `204`/`401` (condición de carrera sobre la clave primaria de `revoked_tokens`).
Para reproducirlo, quita la anotación `@Disabled` de esa prueba y ejecuta:

```powershell
docker run --rm -v "${PWD}\ms-users:/src:ro" -v fixia-m2:/root/.m2 maven:3.9.9-eclipse-temurin-21 sh -c "cp -r /src /w && cd /w && mvn -B test -Dtest=ConcurrencyIntegrationTest"
```

---

## 4. Qué cubren las pruebas de ms-users

| Archivo | Tipo | Verifica |
|---|---|---|
| `service/TokenServiceUnitTest` | Unitaria | Claims del JWT RS256, hash del refresh token, rotación de un solo uso, carreras |
| `service/AuthServiceUnitTest` | Unitaria | Login, normalización del correo, tiempo constante, mensaje único de error |
| `service/ClientRegistrationServiceUnitTest` | Unitaria | Alta de cliente, rol CLIENT, consentimiento, duplicados |
| `service/LogoutServiceUnitTest` | Unitaria | Revocación del access token y del refresh token propio |
| `service/TokenHasherTest` | Unitaria | SHA-256 contra vectores conocidos |
| `domain/DomainModelUnitTest` | Unitaria | Entidades y valores de enumeración del DD |
| `dto/RequestValidationUnitTest` | Unitaria | Valores límite de cada campo de registro, login y tokens |
| `dto/ResponseMappingUnitTest` | Unitaria | Las respuestas no exponen datos sensibles |
| `config/*UnitTest` | Unitaria | Cargador de claves, validador de revocación, respuesta 401 única |
| `exception/GlobalExceptionHandlerUnitTest` | Unitaria | Errores RFC 7807 sin detalles internos |
| `integration/AuthenticationFlowIntegrationTest` | Integración | Registro → login → perfil → refresh → logout; aislamiento; no escalar a ADMIN |
| `integration/ConcurrencyIntegrationTest` | Integración | Registros, refresh e inicios de sesión simultáneos |
| `integration/DatabaseContractIntegrationTest` | Integración | Esquema contra las tablas 1, 8 y 9 del DD y restricciones de la BD |
| `integration/SecurityHardeningIntegrationTest` | Integración | Superficie expuesta, entradas hostiles, fugas de información |
| `integration/FrontendContractIntegrationTest` | Integración | Contrato front-back (GC-253 y GC-255): el cuerpo exacto que envían los formularios del front y cómo responde y qué guarda `ms-users`; las de técnico se ejecutan desde el 8/10/2026, porque el endpoint ya está en `develop` |
| `support/TestData`, `support/ApiClient` | Soporte | Datos y cliente HTTP compartidos |

El detalle de trazabilidad (requisito → prueba) está en `TD_V1.docx`.

### Lo que NO se puede probar todavía

Las pruebas solo pueden verificar lo que está implementado. Desde el 8/10/2026 el registro de técnico (RF-002, CA-13) y el perfil profesional
(RF-009, CA-15) existen en el back y el registro de técnico está conectado en la web; el perfil aún no tiene pantalla. Hoy **no existe código** para:
2FA (RF-008), la aprobación del técnico, publicaciones, postulaciones, cotizaciones, pagos y billetera (RF-020 a RF-073, CA-01 a CA-11). Esos requisitos figuran como *brecha* en `TD_V1.docx` y se deben probar
cuando se implementen.

---

## 5. Git Flow aplicado

Según la wiki (`05-git-control-versiones/estrategias-ramas.md`):

| Rama | Entorno | Protección |
|---|---|---|
| `main` | Producción | PR + 2 aprobaciones + CI en verde |
| `release/*` | Staging | PR; solo ajustes finales |
| `develop` | QA / Dev | PR + 1 aprobación + CI en verde |
| `feature/<CLAVE-JIRA>-<descripción>` | — | Se crea desde `develop` y vuelve a `develop` |

Los cambios de QA están en la rama local `feature/GC-275-pruebas-qa-td-v1` de `ms-users`, `ms-payments` y `ms-orchestrator`
(creada desde `develop`; la clave `GC-275` es la historia del SRS que reúne la evidencia de QA). **Aún no hay commits ni push.**
El paso a paso para hacerlos, con los archivos exactos de cada commit y los textos de los PR, está en la **sección 7**.

---

## 6. Solución de problemas

| Síntoma | Causa y solución |
|---|---|
| `docker-credential-desktop: executable file not found` | Añade `...\DockerDesktop\resources\bin` al PATH de la sesión (ver 2.1). |
| `no such host` / `lookup ... production.cloudfront.docker.com` | DNS intermitente de Docker Desktop. Reintenta; si persiste, reinicia Docker Desktop y revisa DNS/VPN/proxy. |
| `Non-resolvable parent POM` / `No address associated with hostname` al correr las pruebas | Mismo DNS intermitente, ahora al descargar dependencias de Maven. Vuelve a ejecutar el comando; la descarga continúa y queda en la caché `fixia-m2`. |
| `failed to connect to the docker API ... dockerDesktopLinuxEngine` | Docker Desktop no está abierto o aún arranca. Ábrelo y espera a "Engine running". |
| `falta JWT_PRIVATE_KEY en .env` | Ejecuta `setup-dev.sh` de la rama `develop`: genera las claves y no pisa el resto del `.env`. |
| `$'\r': command not found` al ejecutar un `.sh` | Estás usando el `bash` de WSL. Usa Git Bash (ver 2.1). |
| `http://localhost` responde **502 Bad Gateway** | El nginx del front (`develop`) escucha en 8080 y el compose enruta Traefik al 80. Levanta con el ajuste local de la sección 2.2. Si estás en una rama de PR con `listen 80;`, levanta **sin** el ajuste. |
| Puerto 80, 8080, 3000 o 9090 ocupado | Cierra el proceso que lo usa o cambia el mapeo en `docker-compose.yml`. |
| Primera ejecución de pruebas muy lenta | Descarga las dependencias de Maven; el volumen `fixia-m2` las conserva para las siguientes. |

---

## 7. Cómo subir los cambios: commits y Pull Requests

Esta sección es el paso a paso para que **tú** hagas los commits y los PR de lo trabajado en QA. Nada se sube solo.
Reglas de la wiki (`05-git-control-versiones`): un PR por tarea de Jira, mensajes `tipo(alcance): descripción [CLAVE]`
(descripción en minúsculas, imperativo, sin punto final), PR hacia `develop` con *Squash and Merge* y 1 aprobación.
Los comandos de `git` son iguales en Windows (PowerShell o Git Bash), Mac y Linux.

### 7.1 ¿Lo que se agregó pisa o rompe algo de lo que ya estaba en los repositorios?

**No.** Todo es aditivo, y se comprobó así:

| Repositorio | Qué ya existía | Qué se cambió | Comprobado |
|---|---|---|---|
| `ms-users` | 60 pruebas, `pom.xml`, `application-test.yml` | Se **agregan** archivos de prueba. Solo se modifican 2 archivos: `pom.xml` (1 línea: umbral de cobertura 0,80 → 0,85) y `application-test.yml` (+7 líneas de logs) | Las 60 pruebas previas siguen intactas y pasan junto con las nuevas (269 en total). La imagen Docker construye bien |
| `ms-payments` | Solo la clase de arranque | `pom.xml` (+58 líneas: dependencias **solo de prueba** y plugin JaCoCo) y pruebas nuevas | La imagen Docker construye bien. La verificación de cobertura corre solo con `mvn verify`, no en el build de Docker |
| `ms-orchestrator` | 12 servicios en el compose | `docker-compose.yml` (+70 líneas) y 5 archivos nuevos en `config/observability/`. Se agregan 4 servicios (Prometheus, Blackbox, Grafana, Adminer) en perfiles **opcionales** | Los 12 servicios existentes quedan sin cambios y no se quitó ninguno; `docker compose config` es válido; el perfil `full` no cambia |

### 7.2 ¿Qué PR hay que hacer?

| PR | ¿Hace falta? | Por qué |
|---|---|---|
| **`ms-users`** | **Sí** | Es donde viven las pruebas de Java (209 nuevas) |
| `ms-orchestrator` | **Recomendado** | Las pruebas de `ms-users` **no lo necesitan** (corren solas). Pero `GUIA_VERIFICAR_DATOS.md` usa el perfil `tools` (Adminer) y la observabilidad pedida en RNF-028; sin este PR, otra persona no tendría esos perfiles |
| `ms-payments` | Opcional | Solo agrega una prueba de humo al esqueleto del servicio |

Cada repositorio tiene su rama local `feature/GC-275-pruebas-qa-td-v1`, ya creada desde `develop` y al día con `origin/develop`.

Los archivos de la carpeta `FIXIA-QA` (`README.md`, `GUIA_*.md`, `TD_V1.docx`) **no pertenecen a ningún repositorio** (esa carpeta no es un repo de git):
no van en ningún PR. Decide con el equipo dónde guardarlos.

### 7.3 Antes de empezar

1. **Confirma la clave de Jira.** Usé `GC-275` (la historia que el SRS asocia con la evidencia de QA y el TD V1; es el prefijo que usa el historial de los
   repos). Si la tuya es otra, renombra la rama en cada repo antes de subirla (`git branch -m feature/GC-XXX-pruebas-qa-td-v1`) y usa esa clave
   en los commits y en los títulos.
2. **Ejecuta las pruebas** y confirma que dan lo esperado. Sobre el `develop` del 8/10/2026 (ya con el back de técnicos): `ms-users` → `Tests run: 291, Failures: 0, Skipped: 1` y `BUILD SUCCESS`
   (con el `develop` anterior eran 269 y 4 omitidas; sección 3.1 y `GUIA_VERIFICAR_DATOS.md`, paso 9). **Antes de subir, rebasa tu rama sobre `origin/develop`**: se creó a partir de un `develop` anterior.
3. Desde el 8/10/2026 `ms-users` y `ms-frontend` **sí tienen CI** (`.github/workflows`): `ms-users` corre `mvn test` y `ms-frontend` corre `flutter analyze`, `flutter test --coverage` y un mínimo de cobertura de 80 %. Pega en el PR también el resultado de tu ejecución. Nota: `mvn test` no aplica el mínimo de cobertura de JaCoCo (eso ocurre en `mvn verify`).
   **No subas `docker-compose.qa-local.yml`** (el ajuste local del puerto, sección 2.2): es solo para tu computador.
4. En Windows git puede avisar `LF will be replaced by CRLF`: es normal (`core.autocrlf=true`), no hagas nada.

En cada repo, **antes de cada commit** comprueba la rama y qué hay pendiente:

```bash
git branch --show-current
git status --short
```

**Usa siempre `git add` con archivos o carpetas concretos, nunca `git add .`**, para no subir nada de más.

### 7.4 Los commits

#### `ms-users` — 2 commits

```bash
cd ms-users
git branch --show-current          # debe decir: feature/GC-275-pruebas-qa-td-v1
```

Commit 1, las pruebas (incluye los 2 archivos nuevos de `src/test/resources` y el cambio de `application-test.yml`):

```bash
git add src/test/java src/test/resources
git status --short                 # `pom.xml` NO debe aparecer en verde todavía
git commit -m "test(users): agregar pruebas unitarias y de integracion de QA [GC-275]"
```

Commit 2, el umbral de cobertura:

```bash
git add pom.xml
git commit -m "chore(users): subir umbral de cobertura jacoco a 85% [GC-275]"
git status --short                 # no debe quedar nada pendiente
git log --oneline -3
```

#### `ms-orchestrator` — 1 commit (recomendado)

```bash
cd ../ms-orchestrator
git branch --show-current          # debe decir: feature/GC-275-pruebas-qa-td-v1
git add docker-compose.yml config/observability
git status --short                 # solo esos archivos; el `.env` NO debe aparecer (está ignorado y tiene claves)
git commit -m "feat(compose): agregar perfiles de observabilidad y visor de base de datos [GC-275]"
```

Si prefieres un commit por perfil, usa `git add -p docker-compose.yml` y elige por bloques.

#### `ms-payments` — 2 commits (opcional; primero el `pom.xml`, porque la prueba lo necesita)

```bash
cd ../ms-payments
git branch --show-current          # debe decir: feature/GC-275-pruebas-qa-td-v1
git add pom.xml
git commit -m "chore(payments): agregar dependencias de prueba y jacoco con umbral de 85% [GC-275]"
git add src/test
git commit -m "test(payments): agregar prueba de humo del arranque del servicio [GC-275]"
git status --short                 # no debe quedar nada pendiente
```

### 7.5 Subir cada rama y abrir el PR

Repite en cada repo:

```bash
git fetch origin
git rebase origin/develop          # la wiki exige la rama rebasada antes del PR; hoy ya está al día
git push -u origin feature/GC-275-pruebas-qa-td-v1
```

- Si el `rebase` avisa de conflictos, **detente**: `git rebase --abort` deshace el intento.
- Tras el `push`, git imprime un enlace `https://github.com/Grupo-Fixia/<repo>/pull/new/<rama>`. Ábrelo, o entra a
  `https://github.com/Grupo-Fixia/<repo>/compare/develop...<rama>?expand=1`.
- Configura **base `develop`** y la rama como comparación.
- Pide **1 revisión** de un Senior / Tech Lead. Aprobado: **Squash and merge** y borra la rama.

### 7.6 Títulos y descripciones listos para pegar

El título lleva el mismo formato que el commit. La descripción sigue la plantilla de la wiki. Sustituye el texto entre corchetes por tus resultados.

#### PR de `ms-users`

**Título:** `test(users): agregar pruebas unitarias y de integracion de QA [GC-275]`

```markdown
## 📌 Resumen del Cambio
Agrega 209 pruebas nuevas a ms-users (144 unitarias y 65 de integración) y sube el umbral de cobertura de JaCoCo de 80 % a 85 % (RNF-021 del SRS).
Cubren registro de cliente, inicio y cierre de sesión, tokens JWT, validaciones, contrato del esquema de BD contra el DD, concurrencia y endurecimiento de seguridad.
Cada prueba imprime en la terminal qué evalúa y si pasó o falló (con el motivo).

## 🔗 Ticket de Jira
- **Ticket:** GC-275

## 🛠️ Tipo de Cambio
- [x] ⚙️ Tarea técnica o mantenimiento (`chore`)  (pruebas y configuración de cobertura)

## 🧪 Evidencias y Pruebas
- `mvn verify` sobre el `develop` del 8/10/2026: 291 pruebas (82 existentes + 209 nuevas), 0 fallos, 1 omitida. El mínimo de cobertura (85 %) se cumple. BUILD SUCCESS.
- 1 prueba omitida: `ConcurrencyIntegrationTest.cerrarSesionVariasVecesDeFormaSimultaneaEsIdempotente` (`@Disabled`; documenta el defecto DEF-QA-01:
  dos cierres de sesión simultáneos con el mismo token responden 409 en lugar de 204/401). Las 3 pruebas de técnico de `FrontendContractIntegrationTest` ya se ejecutan y pasan, porque `POST /api/users/technicians` está en `develop`. No se modificó código de producción.
- Las pruebas existentes no se modificaron.
- Archivos existentes modificados: `pom.xml` (umbral 0.80 → 0.85) y `src/test/resources/application-test.yml` (logs en WARN durante las pruebas).
- [Pega aquí el resumen de tu ejecución local: las líneas `Tests run:` y `BUILD SUCCESS`]

## 📋 Checklist de Autoevaluación
- [x] El título del PR sigue el formato `<tipo>(<alcance>): <descripción> [GC-XXX]`.
- [ ] La rama origen está actualizada mediante `git rebase` con la rama de destino.
- [x] No se subieron archivos innecesarios, temporales o secretos.
- [x] El código cumple con las reglas de estilo del proyecto.
- [ ] Todos los tests se ejecutaron en verde (no hay CI configurado; ejecutadas localmente).
```

#### PR de `ms-orchestrator` (recomendado)

**Título:** `feat(compose): agregar perfiles de observabilidad y visor de base de datos [GC-275]`

```markdown
## 📌 Resumen del Cambio
Agrega dos perfiles opcionales al docker-compose, sin cambiar los servicios existentes ni el perfil `full`:
- `observability` (RNF-028): Prometheus, Blackbox Exporter y Grafana con datasource y dashboard "Fixia - Disponibilidad". Sondea 9 servicios (3 HTTP y 6 TCP).
- `tools`: Adminer (visor web de PostgreSQL) en http://localhost:8081, solo desarrollo.

## 🔗 Ticket de Jira
- **Ticket:** GC-275

## 🛠️ Tipo de Cambio
- [x] 🚀 Nueva funcionalidad (`feat`)

## 🧪 Evidencias y Pruebas
- `docker compose config -q` sin errores. Los 12 servicios existentes no cambian; se agregan 4.
- `docker compose --profile full --profile observability --profile tools up -d`: Prometheus muestra `probe_success = 1` en los 9 destinos; Grafana carga el dashboard; Adminer entra a `postgres-users`.
- Puertos solo en `127.0.0.1` (3000 Grafana, 9090 Prometheus, 8081 Adminer). No incluye cAdvisor (su imagen está solo en gcr.io).
- Archivo existente modificado: `docker-compose.yml` (+70 líneas).

## 📋 Checklist de Autoevaluación
- [x] El título sigue el formato de la wiki.
- [ ] La rama está rebasada con `develop`.
- [x] No se subieron archivos innecesarios, temporales o secretos (`.env` no incluido).
```

#### PR de `ms-payments` (opcional)

**Título:** `chore(payments): agregar pruebas de humo y cobertura jacoco de 85% [GC-275]`

```markdown
## 📌 Resumen del Cambio
ms-payments no tenía dependencias de prueba. Se agregan `spring-boot-starter-test` y `h2` (solo para pruebas), el plugin JaCoCo con umbral de 85 % y 4 pruebas
de humo (el servicio arranca con su configuración). Los requisitos de pagos (RF-057 a RF-073) aún no tienen código, por lo que no hay más que probar hoy.

## 🔗 Ticket de Jira
- **Ticket:** GC-275

## 🛠️ Tipo de Cambio
- [x] ⚙️ Tarea técnica o mantenimiento (`chore`)

## 🧪 Evidencias y Pruebas
- `mvn verify`: 4 pruebas, 0 fallos. Cobertura 100 % (4 de 4 líneas; poco significativa porque el servicio solo tiene la clase de arranque). BUILD SUCCESS.
- La imagen Docker del servicio sigue construyendo. Archivo existente modificado: `pom.xml` (+58 líneas, solo dependencias de prueba y plugin JaCoCo).
- [Pega aquí el resumen de tu ejecución local]

## 📋 Checklist de Autoevaluación
- [x] El título sigue el formato de la wiki.
- [ ] La rama está rebasada con `develop`.
- [x] No se subieron archivos innecesarios, temporales o secretos.
```

### 7.7 Orden y relación con otros PR abiertos

1. `ms-users`, `ms-orchestrator` y `ms-payments` son **independientes** entre sí: se pueden abrir a la vez.
2. **PR del equipo ya abiertos en `ms-users` (GC-254, GC-261, GC-262):** no chocan con estos archivos; mis pruebas se verificaron sobre GC-262 y esas
   ramas cumplen el umbral de 85 % (96–97 % de líneas).
3. Abre en Jira los defectos **DEF-QA-01** y **DEF-QA-02** y enlázalos desde los PR.

### 7.8 Si te equivocas

| Situación | Solución |
|---|---|
| Hiciste `git add` de un archivo que no debía ir | `git restore --staged <archivo>` (el archivo queda intacto en tu carpeta) |
| El último commit tiene un error y **aún no hiciste push** | `git commit --amend -m "nuevo mensaje"` (o `git reset --soft HEAD~1` para deshacerlo y empezar de nuevo) |
| Ya hiciste push y quieres corregir | Un commit nuevo encima y otro `git push`. `git push --force-with-lease` solo en **tu propia rama** y tras un rebase, nunca en `develop` ni `main` |
| Estás en la rama equivocada | `git branch --show-current` antes de cada commit; si ya commiteaste en otra, pide ayuda antes de mover nada |
| El `rebase` dio conflictos | `git rebase --abort` y vuelves al estado anterior |
| No sabes qué se va a subir | `git diff --cached --stat` (lo agregado) y `git log origin/develop..HEAD --stat` (lo que tendrá el PR) |
