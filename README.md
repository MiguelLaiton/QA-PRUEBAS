# QA-PRUEBAS
# README_PRUEBAS — QA Sprint 3 (ms-users + ms-frontend)

Guía para **actualizar, compilar, levantar y verificar** `ms-users` (Spring Boot) y `ms-frontend` (Flutter) junto con las pruebas del Sprint 3 (GC-234 a GC-237 · CA-12 a CA-15).
Todos los comandos son para **PowerShell** y están verificados en equipo (Windows 11) el 06/10/2026. 

```text
FIXIA\
├─ README_PRUEBAS.md                 ← este archivo
├─ FIXIA-QA.code-workspace           ← workspace de VS Code (tareas, Java 21, Flutter, depuración)
├─ ms-users\                         ← backend
│  ├─ mvnw, mvnw.cmd, .mvn\          ← NUEVO: Maven Wrapper (no hace falta instalar Maven)
│  ├─ src\test\java\...\acceptance\  ← NUEVO: Sprint3AcceptanceTest, CorsAndContractObservationsTest
│  └─ qa\smoke-sprint3.ps1, qa\sprint3.http   ← NUEVO: humo y colección REST Client
├─ ms-frontend\
│  └─ test\features\technician_registration\  ← NUEVO: controller / contract / form tests
│     test\integration\                       ← NUEVO: integración real contra ms-users
├─ qa-sprint3\                       ← documentación de QA (casos, datos, defectos, TD V1, evidencias)
│  └─ sonarqube\                     ← NUEVO: docker-compose y analizar.ps1
└─ ms-users\qa\postman\ · sonar-project.properties (ms-users y ms-frontend)  ← NUEVO
```

>  No se modificó ningún archivo versionado.

---

## 1. Requisitos (estado en este equipo)

| Herramienta | Estado | Ubicación / nota |
|---|---|---|
| Git | ✅ | — |
| JDK 21 | ✅ | . **El `java` por defecto es 15** → siempre usar `JAVA_HOME` de 21 (ya configurado en las terminales de VS Code vía el workspace). |
| Maven | ✅ vía wrapper | `.\mvnw.cmd` descarga Maven 3.9.9 la primera vez (queda en `%USERPROFILE%\.m2\wrapper`). |
| Flutter 3.47.6 (Dart 3.13.5) | ✅ |  (instalado en esta sesión). Windows desktop y Chrome disponibles según `flutter doctor`. |
| VS Code + extensiones | recomendadas | Java Pack, Spring Boot Tools, Dart, Flutter, REST Client, PowerShell (el workspace las sugiere al abrirlo). |
| Docker | opcional | PostgreSQL real (§4.3) y SonarQube (§9). Docker Desktop está en `%LOCALAPPDATA%\Programs\DockerDesktop`. |
| Node.js 22 / npx | ✅ | Para Newman (§8.2). |
| Postman (aplicación) | recomendada | No se verificó su instalación; Newman da el mismo resultado sin ella. |

Si trabaja en otro equipo: instale JDK 21 y Flutter estable, y ajuste las rutas  del archivo `FIXIA-QA.code-workspace`.

Para una terminal fuera de VS Code, en cada sesión:

```powershell
$env:JAVA_HOME = "C:"
$env:Path = "C:"
```

## 2. Git — actualizar todas las ramas distintas de `main`

Cada microservicio es **su propio repositorio**. Ramas asociadas al Sprint 3:

| Repo | Ramas (aparte de `main`) | Rama que se prueba |
|---|---|---|
| `ms-users` | `develop`, `feature/GC-254-registro-tecnico`, `feature/GC-259-cierre-sesion`, `feature/GC-261-modelo-perfil-profesional`, `feature/GC-262-api-perfil-tecnico` | `feature/GC-262-api-perfil-tecnico` (contiene a todas las anteriores: historial lineal) |
| `ms-frontend` | `develop`, `feature/GC-235-registro-cuenta-tecnico`, `feature/GC-255-formulario-registro-tecnico` | `feature/GC-255-formulario-registro-tecnico` (contiene a GC-235) |

### 2.1 Opción rápida (script)

```powershell
cd C:
.\qa-sprint3\actualizar-ramas.ps1
```

Hace `fetch --all --prune` y `pull --ff-only` de cada rama (no hace merge ni descarta nada) y deja cada repo en la rama a probar. Si hay cambios locales en archivos versionados, se detiene en ese repo y lo avisa.

### 2.2 Comandos exactos (manual)

```powershell
# ---- ms-users ----
cd C:
git fetch --all --prune
git checkout develop                                    ; git pull --ff-only
git checkout feature/GC-254-registro-tecnico            ; git pull --ff-only
git checkout feature/GC-259-cierre-sesion               ; git pull --ff-only
git checkout feature/GC-261-modelo-perfil-profesional   ; git pull --ff-only
git checkout feature/GC-262-api-perfil-tecnico          ; git pull --ff-only
git log --oneline -3                                    # debe verse 5826fc7 (o más reciente)

# ---- ms-frontend ----
cd C:
git fetch --all --prune
git checkout develop                                    ; git pull --ff-only
git checkout feature/GC-235-registro-cuenta-tecnico     ; git pull --ff-only
git checkout feature/GC-255-formulario-registro-tecnico ; git pull --ff-only
git log --oneline -3                                    # debe verse ea7693f (o más reciente)
```

Comprobar que no hay ramas nuevas en el remoto (si aparece una, agréguela a la lista):

```powershell
git branch -r
```

Si `git pull --ff-only` falla por historiales divergentes, **no** use `--force`: avise al equipo. Los archivos nuevos de QA (sin seguimiento) no estorban para cambiar de rama.

## 3. Abrir el workspace en VS Code

```powershell
cd C:
code .\FIXIA-QA.code-workspace
```

Abre cuatro carpetas: `ms-users`, `ms-frontend`, `qa-sprint3` y la raíz (este README). Con **Ctrl+Shift+P → «Tasks: Run Task»** encuentra:

| Tarea | Hace |
|---|---|
| `BACKEND 1 - Compilar y probar todo (mvnw verify)` | Compila, ejecuta las 183 pruebas y valida cobertura |
| `BACKEND 2 - Solo pruebas de aceptación Sprint 3` | Solo `Sprint3AcceptanceTest` + `CorsAndContractObservationsTest` |
| `BACKEND 3 - Levantar ms-users (perfil local, H2, :8080)` | Arranca el servicio (se queda corriendo) |
| `BACKEND 4 - Smoke Sprint 3` | Humo contra el servicio levantado |
| `POSTMAN - Ejecutar colección con Newman` | 51 requests / 94 aserciones |
| `SONAR 1` / `SONAR 2` | Levantar SonarQube (Docker) / analizar ms-users |
| `FRONTEND 1 … 5` | `pub get` · pruebas · integración real · `analyze` · app en Windows |
| `QA - TODAS las pruebas automáticas` | Backend + frontend (tarea de prueba por defecto: **Ctrl+Shift+P → Tasks: Run Test Task**) |

En **Run and Debug** hay dos lanzadores del frontend (Windows y Chrome) ya apuntando a `http://localhost:8080`.

## 4. Compilar y levantar `ms-users`

### 4.1 Compilar y probar

```powershell
cd C:
$env:JAVA_HOME = "C:"
.\mvnw.cmd -B clean verify              # compila + 183 pruebas + cobertura JaCoCo (≥ 80 %)
.\mvnw.cmd -B clean package -DskipTests # solo empaquetar (target\ms-users-0.0.1-SNAPSHOT.jar)
```

La primera vez descarga Maven y las dependencias (varios minutos). Las pruebas **no necesitan Docker ni PostgreSQL** (H2 en memoria con las mismas migraciones Flyway).

### 4.2 Levantar con el perfil `local` (H2, sin Docker) — recomendado para las pruebas manuales

```powershell
cd C:
$env:JAVA_HOME = "C:"
.\mvnw.cmd -B spring-boot:run "-Dspring-boot.run.useTestClasspath=true" "-Dspring-boot.run.profiles=local"
```

Listo cuando aparece `Started Application in … seconds`. Compruebe desde **otra** terminal:

```powershell
Invoke-RestMethod http://localhost:8080/actuator/health      # status : UP
```

* Los datos viven en memoria: se pierden al detener el servicio (Ctrl+C).
* Claves JWT efímeras: al reiniciar, los tokens anteriores dejan de valer.
* Si el puerto 8080 está ocupado (p. ej. Traefik del orquestador, OBS-05): `$env:SERVER_PORT = "8081"` antes de arrancar y use `-BaseUrl http://localhost:8081` en el humo y `--dart-define=USERS_API_BASE_URL=http://localhost:8081` en Flutter.

### 4.3 (Opcional, no ejecutado en esta verificación) Con PostgreSQL real

```powershell
docker run -d --name fixia-pg -e POSTGRES_DB=ms-users -e POSTGRES_USER=user -e POSTGRES_PASSWORD=password -p 5432:5432 postgres:16-alpine
# Claves JWT (openssl viene con Git for Windows)
$ssl = "C:\Program Files\Git\usr\bin\openssl.exe"
& $ssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out $env:TEMP\jwt-private.pem
& $ssl pkey -in $env:TEMP\jwt-private.pem -pubout -out $env:TEMP\jwt-public.pem
$env:JWT_PRIVATE_KEY = (Get-Content $env:TEMP\jwt-private.pem -Raw)
$env:JWT_PUBLIC_KEY  = (Get-Content $env:TEMP\jwt-public.pem  -Raw)
.\mvnw.cmd -B spring-boot:run          # usa localhost:5432/ms-users, user/password por defecto
```

## 5. Compilar y levantar `ms-frontend` (Flutter)

```powershell
cd C:
$env:Path = "C:;" + $env:Path
flutter pub get
flutter analyze                          # 1 advertencia conocida de entorno (flutter_lints), ver qa-sprint3/05 §4
flutter test --coverage                  # 64 pasan, 7 omitidas
# Ejecutar la app contra el backend local (ms-users debe estar levantado):
flutter run -d windows --dart-define=USERS_API_BASE_URL=http://localhost:8080
```

* La URL por defecto de la app es `https://api.fixia.com`; **sin** `--dart-define` no hablará con su servicio local.
* **Use Windows, no Chrome:** en Chrome la petición es bloqueada por CORS (DEF-01). El lanzador «Chrome» existe solo para reproducir ese defecto.
* Verificado: `flutter build windows --debug` compila y genera `build\windows\x64\runner\Debug\ms_frontend.exe` (≈ 1 min; usa Visual Studio, detectado por `flutter doctor`). La ejecución interactiva de la ventana no se probó.
* Para un ejecutable: `flutter build windows --dart-define=USERS_API_BASE_URL=http://localhost:8080` (salida en `build\windows\x64\runner\Release\`).

## 6. Ejecutar y verificar las pruebas automáticas

### 6.1 Resumen de comandos

| # | Qué | Comando (cwd) | Resultado esperado |
|---|---|---|---|
| 1 | **Backend completo** | `.\mvnw.cmd -B verify` (`ms-users`) | `Tests run: 183, Failures: 0, Errors: 0, Skipped: 0` · `All coverage checks have been met.` · `BUILD SUCCESS` |
| 2 | Solo aceptación Sprint 3 | `.\mvnw.cmd -B test "-Dtest=Sprint3AcceptanceTest,CorsAndContractObservationsTest" "-Djacoco.skip=true"` | `Tests run: 101, Failures: 0` |
| 3 | Un criterio (etiqueta) | `.\mvnw.cmd -B test "-Dgroups=CA-14" "-Djacoco.skip=true"` (también `CA-12`, `CA-13`, `CA-15`, `E2E`, `DEF-01`, `OBS-02`, `OBS-03`) | CA-14 → `Tests run: 9, Failures: 0` |
| 4 | **Frontend completo** | `flutter test --coverage` (`ms-frontend`) | `+64 ~7: All tests passed!` |
| 5 | Un caso del frontend | `flutter test --plain-name "TC-FE-UI-06"` | `All tests passed!` |
| 6 | **Humo contra servicio** (con ms-users levantado) | `.\qa\smoke-sprint3.ps1` (`ms-users`) | `PASS: 20  FAIL: 1` — el FAIL es **TC-S3-SMK-20 (DEF-01)** y es el esperado hoy |
| 7 | **Integración real front↔back** (con ms-users levantado) | `flutter test test/integration --dart-define=USERS_API_BASE_URL=http://localhost:8080 --reporter expanded` (`ms-frontend`) | `+5 -2`: fallan **TC-FE-INT-03 y 04 (DEF-02)**, esperado hoy |
| 8 | Reproducir DEF-02 sin servicio | `flutter test test/features/technician_registration/technician_registration_contract_test.dart --plain-name DEF-02 --run-skipped` | Falla con `Actual: 'El telÃ©fono no es vÃ¡lido'` (esperado hoy) |

> Si ejecuta el humo con PowerShell bloqueado por política: `powershell -NoProfile -ExecutionPolicy Bypass -File .\qa\smoke-sprint3.ps1`.

### 6.2 Dónde ver los reportes

| Reporte | Ruta |
|---|---|
| Resultado por clase (backend) | `ms-users\target\surefire-reports\*.txt` y `*.xml` |
| Cobertura backend (HTML) | `ms-users\target\site\jacoco\index.html` (abrir en el navegador: `start ms-users\target\site\jacoco\index.html`) |
| Cobertura frontend | `ms-frontend\coverage\lcov.info` (extensión «Coverage Gutters» para verla en el editor) |
| Resultado del humo | `ms-users\qa\smoke-sprint3-resultado.csv` |
| Resultado Postman | consola de Newman / `qa-sprint3\evidencias\postman-newman-junit.xml` |

## 7. Verificación manual (interfaz Flutter + API)

Prepare dos terminales: **(A)** `ms-users` levantado (§4.2) y **(B)** la app: tarea `FRONTEND 5` o el comando de §5. Para las llamadas HTTP abra `ms-users\qa\sprint3.http` (extensión REST Client) y seleccione el entorno **local** (Ctrl+Alt+E). Datos: `qa-sprint3\03-datos-de-prueba.md` (D-07).

| Caso | Pasos | Debe verse | ✔ |
|---|---|---|---|
| **MAN-01** Registro de cliente (API; no hay pantalla) | En `sprint3.http` envíe los bloques 1, 2 y 3 | 201 `role: CLIENT` · 400 con `errors[]` (correo y consentimiento) · 409 | ☐ |
| **MAN-02** Formulario vacío | En la app pulse «Crear cuenta de técnico» sin llenar nada | «… es obligatorio» en cada campo y «Debes aceptar el tratamiento de datos.»; no se crea cuenta | ☐ |
| **MAN-03** Registro válido | Llene con D-07, marque el tratamiento de datos (verá versión `v1.0` y fecha), pulse el botón | Aviso «Tu cuenta de técnico fue creada.» y botón deshabilitado. En `sprint3.http` haga login con ese correo (ajuste el JSON del bloque 6): 200 | ☐ |
| **MAN-04** Correo duplicado | Reinicie la app (`R` mayúscula en la terminal de Flutter) y repita MAN-03 con los mismos datos | «Ya existe una cuenta con ese correo o documento.» | ☐ |
| **MAN-05** Servicio caído | Detenga ms-users (Ctrl+C) y envíe el formulario con datos nuevos | «No fue posible conectar con el servicio. Inténtalo de nuevo.» | ☐ |
| **MAN-06** DEF-02 | Con ms-users arriba, correo de 250 `a` + `@a.co` y resto válido | Hoy: texto con `Ã©`/`Ã³` (defecto). Tras corregir: mensaje legible | ☐ |
| **MAN-07** CA-14/15 por API | En `sprint3.http` ejecute los bloques 6 → 15 en orden | Login 200 · credenciales malas 401 · `/me` sin token 401, con token 200 · perfil vacío → PUT 200 → 81 años 400 · logout 204 · access y refresh posteriores 401 | ☐ |

Guarde las capturas como `qa-sprint3\evidencias\MAN-0X-<descripcion>.png` y marque los resultados en `qa-sprint3\02-casos-de-prueba.md` §G.

## 8. Pruebas en Postman (API testing de `ms-users`)

Generado en esta revisión (**no existía nada previo**):

| Archivo (`ms-users\qa\postman\`) | Contenido |
|---|---|
| `Fixia-ms-users-Sprint3.postman_collection.json` | Colección v2.1: **51 requests y 94 aserciones**, organizada por criterio (00 Salud · 01 CA-12 · 02 CA-13 · 03 CA-14 · 04 CA-15 · 05 cierre de sesión). Cada request lleva su caso `TC-PM-xx`. |
| `Fixia-local.postman_environment.json` | Environment `Fixia · local` con `baseUrl=http://localhost:8080` y `password`. |
| `_build_collection.py` | Generador de los dos JSON (editar aquí y ejecutar `python _build_collection.py` si hay que cambiar casos). |

### 8.1 Importar y ejecutar en la aplicación Postman

1. Levante `ms-users` con el perfil local (§4.2) y compruebe `http://localhost:8080/actuator/health`.
2. En Postman: **Import** → arrastre los dos `.json` de `ms-users\qa\postman\` (o *Files → Select Files*).
3. Arriba a la derecha elija el environment **«Fixia · local (ms-users :8080)»**.
4. Clic derecho sobre la colección **«Fixia · ms-users · Sprint 3 QA» → Run collection** (Collection Runner).
5. Deje todas las requests marcadas **en el orden original**, *Iterations = 1*, sin *delay*, y pulse **Run**.
6. Revise *Test Results*: todas las aserciones deben estar en verde. Se puede repetir sin limpiar la base: la primera request genera un sufijo nuevo (`run`) en cada ejecución.
7. Una request suelta solo funciona si antes se ejecutó la secuencia previa (las carpetas se pasan los tokens por variables de colección).

### 8.2 Ejecutar por línea de comandos (Newman) — mismo resultado, apto para CI

```powershell
cd C:
npx --yes newman run qa\postman\Fixia-ms-users-Sprint3.postman_collection.json -e qa\postman\Fixia-local.postman_environment.json
```

O la tarea de VS Code **«POSTMAN - Ejecutar colección con Newman»**. Informe JUnit: añada `--reporters cli,junit --reporter-junit-export qa\postman\newman-junit.xml`.

**Resultado esperado:** `requests 51 / failed 0` y `assertions 94 / failed 0`. Verificado el 06/10/2026 dos veces seguidas contra `ms-users@5826fc7` (evidencia: `qa-sprint3\evidencias\postman-newman*`).

### 8.3 Qué cubre

| Carpeta | Requests | Verifica |
|---|---|---|
| 01 · CA-12 | TC-PM-01…07 | Cliente 201 con rol `CLIENT` y sin contraseña/hash; 400 con `errors[]` (correo, consentimiento, contraseña de 7 caracteres o sin números); 409 por correo (otra capitalización) o documento duplicado con cuerpo genérico; JSON malformado sin filtrar trazas. |
| 02 · CA-13 | TC-PM-10…16 | Técnico 201 `PROFESSIONAL`/`PENDING`; sin consentimiento, sin `documentType` o teléfono inválido → 400; correo de cliente reutilizado → 409; campo `specialty` ignorado (OBS-01). |
| 03 · CA-14 | TC-PM-20…33 | Login técnico y cliente; 401 idéntico para contraseña errónea, correo inexistente e inyección SQL; `/me` sin token, con basura o alterado → 401; refresh con rotación y reuso → 401; logout sin token 401 o sin `refreshToken` 400. |
| 04 · CA-15 | TC-PM-40…54 | Perfil vacío → alta → consulta → actualización con recorte; años −1/81, categorías vacías/inexistentes y 1001 caracteres → 400 sin alterar lo guardado; sin token 401; cliente 403; **aislamiento entre técnicos**. |
| 05 · Cierre | TC-PM-60…66 | Logout 204; access, perfil y refresh quedan inválidos (401); nuevo login conserva el perfil; la sesión del cliente es independiente. |

> No se incluye CORS (DEF-01): Postman no aplica la política de los navegadores. Se prueba con `TC-S3-INT-01/02` y el humo (SMK-20).

## 9. Análisis estático con SonarQube

> **Estado:** configuración lista; **el análisis no se pudo ejecutar contra un servidor en esta revisión**: Docker Desktop no logró descargar la imagen `sonarqube:community` (sin resolución DNS hacia el CDN de Docker Hub desde su máquina virtual). Ejecútelo en una red con acceso a Docker Hub.

| Archivo | Para qué |
|---|---|
| `ms-users\sonar-project.properties` | Proyecto `fixia-ms-users`: fuentes, binarios, cobertura JaCoCo XML (`target\site\jacoco\jacoco.xml`, **verificado: se genera con `mvnw verify`**), resultados Surefire y exclusión de `Application.java`. |
| `ms-frontend\sonar-project.properties` | Proyecto `fixia-ms-frontend` (`lib`, `test`, `coverage\lcov.info`). **SonarQube Community no analiza Dart sin el plugin comunitario `sonar-flutter`**; sin él use `flutter analyze` (tarea `FRONTEND 4`). |
| `qa-sprint3\sonarqube\docker-compose.yml` | Servidor SonarQube Community en **http://localhost:9001** (el 9000 lo usa `fixia-rustfs` del orquestador). |
| `qa-sprint3\sonarqube\analizar.ps1` | `mvnw verify` + análisis + lectura del Quality Gate por API. |
| `C:` | Scanner CLI instalado (para el frontend). |

```powershell
# 1) Docker Desktop abierto. Levantar SonarQube (la primera vez descarga ~1 GB; arranca en 1-3 min)
cd C:
docker compose -f qa-sprint3\sonarqube\docker-compose.yml up -d
Invoke-RestMethod http://localhost:9001/api/system/status        # status : UP

# 2) Abrir http://localhost:9001 → admin / admin → cambiar contraseña
#    My Account → Security → Generate Tokens (User token) → copiar el token (sqp_...)

# 3) Analizar ms-users (compila, prueba, mide cobertura y envía a Sonar)
.\qa-sprint3\sonarqube\analizar.ps1 -Token sqp_XXXXXXXX
#    -Frontend añade ms-frontend (solo útil con el plugin sonar-flutter)
```

También con la tarea **«SONAR 2 - Analizar ms-users»** (pide el token). Equivalente manual:

```powershell
cd ms-users
.\mvnw.cmd -B verify
.\mvnw.cmd -B org.sonarsource.scanner.maven:sonar-maven-plugin:sonar "-Dsonar.host.url=http://localhost:9001" "-Dsonar.token=sqp_XXXXXXXX" "-Dsonar.projectKey=fixia-ms-users"
```

(Comprobado sin servidor: el plugin se resuelve y arranca; solo falla por no haber servidor ni token.)

**Quality Gate esperado** (wiki `06-cicd-ambientes/pipelines-quality-gate.md`, sobre código nuevo): cobertura ≥ 80 % · 0 vulnerabilidades · 100 % de *Security Hotspots* revisados · 0 bugs críticos/bloqueantes · duplicación < 3 % · mantenibilidad A. Referencia ya medida con JaCoCo: 97,1 % de líneas. Anote los resultados reales en `qa-sprint3\06-TD-V1-Documento-de-Pruebas.md` §6.4. Para detener: `docker compose -f qa-sprint3\sonarqube\docker-compose.yml down` (`-v` borra los datos).

## 10. Cómo validar los resultados esperados

1. **Backend:** el resumen final de Maven debe decir `Tests run: 183, Failures: 0, Errors: 0, Skipped: 0` y `BUILD SUCCESS`. Si hay un fallo, abra `target\surefire-reports\<Clase>.txt` y busque el nombre del caso (`TC-S3-…` está en `@DisplayName`).
2. **Frontend:** `flutter test` debe terminar con `All tests passed!` y `~7` omitidas (las 6 de integración sin URL + la de DEF-02). Un `-N` (falla) es un incumplimiento nuevo.
3. **Fallos esperados hoy** (no son regresiones; son los defectos abiertos): humo `TC-S3-SMK-20`; integración `TC-FE-INT-03/04`; DEF-02 forzado. Cualquier **otro** fallo, investigarlo y registrarlo con la plantilla de `qa-sprint3\05-registro-defectos.md`.
4. **Cuando se corrija un defecto:** DEF-01 → invertir `TC-S3-INT-01/02` y `TC-S3-SMK-20` (deben pasar a exigir `Access-Control-Allow-Origin`); DEF-02 → quitar el `skip` de `TC-FE-API-13` y confirmar que `TC-FE-INT-03/04` pasan.
5. **Cobertura:** `ms-users` ≥ 80 % de líneas (JaCoCo la valida en `verify`; hoy 97,1 %) y dominio ≥ 85 % (hoy 95,6 %).
6. **Postman:** Newman/Runner con `failed = 0` en requests y aserciones (51/94). Un fallo indica regresión de contrato: abra el request `TC-PM-xx` y compare con `ms-users\docs\openapi.yaml`.
7. **SonarQube:** Quality Gate en *Passed* (§9); toda vulnerabilidad o bug crítico nuevo se registra como defecto.
8. **Documentación:** actualice `qa-sprint3\06-TD-V1-Documento-de-Pruebas.md` (§6–§8) con las ejecuciones nuevas y use §8 como insumo para la tabla 9.4 de la SRS.

## 11. Solución de problemas

| Síntoma | Causa / solución |
|---|---|
| `release version 21 not supported` o `invalid target release: 21` | Está usando Java 15. Defina `JAVA_HOME` de Corretto 21 (§1). |
| Newman: `ECONNREFUSED` | `ms-users` no está levantado (§4.2). |
| Sonar: `HTTP 403` en `localhost:9000` | El 9000 es de `fixia-rustfs`; SonarQube usa el **9001**. |
| Docker: `no such host … cloudfront.docker.com` | Sin DNS hacia Docker Hub: cambie de red/VPN o configure DNS en Docker Desktop → Settings → Docker Engine. |
| `Port 8080 was already in use` | Detenga el otro proceso: `Get-NetTCPConnection -LocalPort 8080 -State Listen \| % { Stop-Process -Id $_.OwningProcess -Force }`, o use otro `SERVER_PORT`. |
| `JWT_PRIVATE_KEY … obligatoria` al arrancar | Arrancó sin el perfil `local`; use el comando de §4.2 o defina las claves (§4.3). |
| `ClassNotFoundException: org.h2.Driver` | Falta `-Dspring-boot.run.useTestClasspath=true` (H2 es de ámbito *test*). |
| `flutter` no se reconoce | Falta el `Path` (§1) o abra la terminal desde VS Code con el workspace. |
| `Unable to find git in your PATH` / `Filename too long` al instalar Flutter | Ejecutar desde PowerShell (no bash) y `git config --global core.longpaths true`. |
| `flutter test` falla con rutas de otro SDK (`package:flutter/...` no encontrado) | Cambió la ruta del SDK: ejecute `flutter pub get`. |
| La app en Chrome no registra y la consola dice *CORS* | Es **DEF-01**. Use Windows. |
| `no se puede cargar el archivo … smoke-sprint3.ps1` | Política de ejecución: use `-ExecutionPolicy Bypass` (§6.1). |
| Texto con `Ã©` en mensajes de error de la app | Es **DEF-02**. |

## 12. Qué conviene confirmar en git (cuando decida)

Las pruebas nuevas y los scripts están sin seguimiento. Para incorporarlos use una rama de QA (no confirme sobre `feature/GC-262…` ni `GC-255…` sin acordarlo con el equipo):

```powershell
cd C:
git checkout -b qa/sprint3-pruebas
git add mvnw mvnw.cmd .mvn src/test/java/com/fixia/users/acceptance qa sonar-project.properties
git commit -m "test(users): pruebas de aceptacion Sprint 3, humo y maven wrapper"

cd ..\ms-frontend
git checkout -b qa/sprint3-pruebas
git add test/features/technician_registration test/integration sonar-project.properties
git commit -m "test(frontend): pruebas de controlador, contrato, formulario e integracion Sprint 3"
```

(`.vscode/`, `.idea/`, `windows/`, `.metadata`, `analysis_options.yaml` y `ms_frontend.iml` ya existían sin seguimiento; no los agregue sin revisar.)
