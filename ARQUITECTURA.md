# ARQUITECTURA — reservas-app (Variante B)

Respuestas a las preguntas de arquitectura del proyecto final. Cada respuesta remite a la evidencia
de [INFORME.md](INFORME.md) cuando la hay.

---

## 1. Las tres preguntas obligatorias del curso

### a) ¿Por qué el frontend se configura con `API_HOST` en tiempo de ejecución y no con `VITE_API_URL` en tiempo de construcción?

**Porque el código del frontend no corre en el contenedor: corre en el navegador.** Vite resuelve
las variables `import.meta.env.VITE_*` cuando ejecuta `npm run build` y las reemplaza por su valor
literal dentro del JavaScript del bundle. Lo que se copia a la imagen final (`/usr/share/nginx/html`)
son archivos estáticos que ya traen la URL escrita adentro. Una vez construida la imagen, ninguna
variable de entorno del contenedor puede cambiar ese valor, porque el navegador que ejecuta el
JavaScript no tiene acceso al entorno del contenedor.

En este proyecto la SPA no conoce ninguna URL absoluta. `frontend/src/api.js` usa siempre la ruta
relativa `/api`, del mismo origen del que se descargó. La decisión de a qué backend reenviar la toma
**nginx**, no el JavaScript:

- La imagen `nginx-unprivileged` ejecuta al arrancar `/docker-entrypoint.d/20-envsubst-on-templates.sh`.
  Ese script toma `nginx/templates/default.conf.template`, sustituye `${API_HOST}` y `${API_PORT}` con
  las variables del contenedor y genera la configuración real:
  `proxy_pass http://${API_HOST}:${API_PORT}/;`.
- `NGINX_ENVSUBST_FILTER=^API_` limita la sustitución a las variables que empiezan con `API_`. Sin
  ese filtro, envsubst también reemplazaría variables propias de nginx como `$uri`, `$host` o
  `$proxy_add_x_forwarded_for` por cadenas vacías, y la configuración quedaría rota en silencio.

Resultado: **una sola imagen (`antoniojv03/reservas-frontend:v1`) para todos los entornos.** En
Compose, `API_HOST=reservas-api` resuelve por el DNS de la red `reservas-net`. En Kubernetes, el mismo
valor llega desde el ConfigMap `reservas-frontend-config` y resuelve al Service `reservas-api`. Se
cambia de entorno cambiando configuración, sin tocar la imagen.

**Si fuera al revés** (con `VITE_API_URL` en el build), pasar de desarrollo a producción obligaría a:

1. Volver a ejecutar `npm run build` con `VITE_API_URL` apuntando a la URL de producción.
2. Construir una imagen nueva del frontend, con otro tag, y publicarla en el registry.
3. Volver a desplegar el frontend con esa imagen (otro `helm upgrade` cambiando `frontend.image.tag`).

Eso significa una imagen por entorno, y que **la imagen que se probó no es la que llega a
producción**, lo que rompe el principio de artefacto inmutable que el curso sostiene desde la entrega 2.
Además, una URL absoluta hacia la API pone a la SPA y a la API en orígenes distintos, y eso obliga a
configurar CORS (punto b) y a exponer la API públicamente.

### b) ¿Por qué esta arquitectura no necesita CORS? ¿Qué habría que agregar si la SPA se comunicara directamente con la API?

**CORS es un control del navegador, y en esta arquitectura el navegador nunca hace una petición a
otro origen.** La política del mismo origen compara esquema, host y puerto. El navegador descarga la
SPA desde `http://reservas.local` y luego pide `http://reservas.local/api/reservas`: mismo esquema,
mismo host, mismo puerto. Para el navegador es una petición del mismo origen, así que no aplica la
política CORS ni se dispara la comprobación previa (`OPTIONS`). Se verificó en la entrega 3 con las
herramientas de desarrollo: el `POST /api/reservas` sale hacia el mismo origen y no hay preflight.

El salto hacia la API lo hace nginx **del lado del servidor** (`location /api/` → `proxy_pass
http://reservas-api:3000/`). La barra final recorta el prefijo `/api`, así que Express recibe
`/reservas`. Una petición de servidor a servidor no pasa por CORS porque no hay navegador que la
controle. Además, la API ni siquiera es alcanzable desde afuera: `reservas-api` es un Service
ClusterIP, y el Ingress tiene una única regla `/` hacia `reservas-frontend:8080`.

**Si la SPA hablara directamente con la API** (por ejemplo `http://api.reservas.local` o
`http://<nodo>:30501`), cambiaría el host o el puerto, y por lo tanto el origen. Habría que agregar:

1. **Exponer la API** fuera del clúster: otra regla de Ingress, otro host o un NodePort.
2. **Encabezados CORS en Express** (por ejemplo, el middleware `cors`):
   - `Access-Control-Allow-Origin` con el origen exacto del frontend (no `*`), tomado de configuración por entorno;
   - responder el **preflight `OPTIONS`**, que acá siempre se dispara porque los `POST` llevan `Content-Type: application/json` y eso no es una petición "simple";
   - `Access-Control-Allow-Methods: GET, POST, DELETE` y `Access-Control-Allow-Headers: Content-Type`.
3. **Configurar en la SPA la URL absoluta de la API**, con lo que se vuelve al problema del punto a:
   la URL queda horneada en el build.

### c) ¿Por qué hay un solo autoescalador entre los doce objetos? ¿Qué ocurriría si se le asignara uno a `reservas-db`? ¿Y por qué tampoco lo tiene el frontend?

**Escalar horizontalmente es una propiedad del diseño de cada capa, no un botón de Kubernetes.** Un
HPA solo tiene sentido donde se cumplen dos condiciones: que las réplicas sean intercambiables y que
la métrica observada crezca con la carga. En este proyecto solo `reservas-api` cumple las dos.

- **`reservas-api` sí.** No guarda estado propio: todo el estado vive en MySQL, así que cualquier
  réplica puede atender cualquier pedido y el Service reparte entre ellas. Su CPU sube con la carga
  porque `GET /reservas` resuelve un `JOIN` entre `reservas` y `salas`. El HPA calcula el porcentaje
  contra `resources.requests.cpu: 100m`, que es el denominador; sin request quedaría en `<unknown>`.
  Se demostró bajo carga: pasó de 2 a 6 réplicas con picos de 279 % y volvió a 2 tras la ventana de
  estabilización (ver [INFORME.md, punto 2](INFORME.md#2-autoescalado-bajo-carga-kubectl-get-hpa--w)).

- **`reservas-db` no: si tuviera un HPA, al escalar se rompería la base.** Es un Deployment con un PVC
  `ReadWriteOnce` en `local-path` (el volumen es un directorio de un nodo). Una segunda réplica caería
  en uno de estos casos:
  - **En otro nodo:** no puede montar el volumen (`Multi-Attach error`) y queda en `ContainerCreating`.
  - **En el mismo nodo:** `ReadWriteOnce` restringe por nodo, no por Pod, así que el montaje se
    permite. Entonces dos `mysqld` abrirían el mismo directorio de datos. InnoDB lo impide con un
    bloqueo de archivo (`Unable to lock ./ibdata1`) y el segundo Pod no arranca. Si lo lograra, el
    resultado sería corrupción.
  - **Con volúmenes separados:** serían dos bases independientes, sin replicación. El Service
    `reservas-db` repartiría las conexiones entre ellas: cada reserva caería en una u otra, el listado
    sería distinto según a cuál se conecte cada réplica de la API, y la restricción `UNIQUE` solo se
    cumpliría dentro de cada instancia, con lo que volverían las dobles reservas (sección 2).

  Escalar una base de datos requiere replicación (primario/réplicas, StatefulSet, un operador), no un
  HPA sobre CPU. Por la misma razón el Deployment de la base usa `strategy: Recreate`: ni siquiera
  durante una actualización debe haber dos `mysqld` sobre el mismo volumen.

- **`reservas-frontend` tampoco, porque no hay carga que lo haga escalar.** Técnicamente podría:
  también es sin estado. Pero nginx sirviendo archivos estáticos y reenviando `/api` consume casi
  nada. En la prueba de carga de la entrega 8, mientras las réplicas de la API consumían entre 46m y
  146m de CPU, los Pods del frontend marcaban 0–1m ([captura](informe/i2-hpa-4-top-pods-bajo-carga.png)).
  Un HPA sobre esa métrica nunca se dispararía, y un autoescalador que no se puede demostrar es peor
  que no tenerlo. La disponibilidad del frontend la dan sus dos réplicas fijas. Además, su Deployment
  no declara `requests`, así que un HPA de CPU quedaría en `<unknown>`.

---

## 2. ¿Qué deja de estar garantizado al pasar de una réplica de la API a tres?

Deja de estar garantizado que **"consultar si el turno está libre y después insertar" sea una
operación indivisible**. La invariante del sistema es que una sala no puede reservarse dos veces en
la misma fecha y franja. La consulta previa a la inserción no sostiene esa invariante.

`POST /reservas` (en `backend/server.js`) hace:

```js
// 1) consulta previa
const [tomadas] = await pool.query(
  'SELECT id FROM reservas WHERE sala_id = ? AND fecha = ? AND franja = ?', ...);
if (tomadas.length > 0) return res.status(409)...
// 2) inserción
await pool.query('INSERT INTO reservas (sala_id, fecha, franja, ...) VALUES (...)', ...);
```

Entre el paso 1 y el 2 hay una ventana. Con tres réplicas, cada una es un proceso independiente con su
propio pool de conexiones, y el Service reparte los pedidos entre ellas. Si llegan tres pedidos por el
mismo turno al mismo tiempo, **los tres procesos pueden ejecutar el `SELECT` antes de que cualquiera
haya hecho su `INSERT`**. Los tres leen "libre" y los tres avanzan a insertar. Ninguna réplica sabe lo
que hacen las otras: no comparten memoria ni bloqueos.

La ventana existe incluso con una sola réplica, porque Node atiende otro pedido mientras este espera
el `await` del `SELECT`. Pero con un solo proceso la coincidencia es rara. Con tres procesos en
paralelo pasa a ser lo esperable bajo carga, que es justamente cuando el HPA agrega réplicas.

**Lo que impide la doble reserva es la restricción de unicidad que aplica el motor de base de
datos**, declarada en `backend/db.js`:

```sql
CONSTRAINT sin_solapamiento UNIQUE (sala_id, fecha, franja)
```

InnoDB respalda esa restricción con un índice único y la verifica **de forma atómica al escribir**.
Cuando dos transacciones intentan insertar la misma clave, la segunda queda bloqueada por la primera
y, al confirmarse esta, falla con `ER_DUP_ENTRY` (error 1062). No importa desde qué réplica ni desde
qué conexión llegue: el motor es único y serializa las escrituras sobre esa clave. La API traduce ese
error a `409 Conflict` (`traducirError`), con el mismo mensaje que devuelve cuando el `SELECT`
encuentra el turno ocupado. El `SELECT` previo queda solo como cortesía para dar un mensaje claro en
el caso común; no es parte de la garantía.

Esto también explica por qué la base no puede escalarse como la API (sección 1c): la garantía depende
de que haya **un único** motor que vea todas las inserciones.

**Prueba (entrega 6, punto 8):** con la API escalada a 3 réplicas se lanzaron 30 `POST` en paralelo
por la misma sala, fecha y franja. Resultado: **1 respuesta `201` y 29 respuestas `409`**. El recuento
en la base dio **exactamente 1** registro. Evidencia completa en [INFORME.md, punto 5](INFORME.md#5-prueba-de-concurrencia-de-la-sexta-entrega).

---

## 3. Dificultades propias de MySQL que se presentaron durante el curso

### 3.1 La contraseña de root es obligatoria (entrega 1)

- **Cómo se manifestó:** `docker run -d --name reservas-db mysql:8.4`, sin variables, dejó el
  contenedor en `Exited (1)` a los pocos segundos.
- **Cómo se diagnosticó:** `docker ps -a` mostró el estado `Exited (1)` y `docker logs reservas-db`
  dio la causa: *"Database is uninitialized and password option is not specified. You need to specify
  one of the following as an environment variable: MYSQL_ROOT_PASSWORD, MYSQL_ALLOW_EMPTY_PASSWORD,
  MYSQL_RANDOM_ROOT_PASSWORD"* ([captura](informe/q3-mysql-root-password-obligatoria.png)).
- **Causa:** el entrypoint de la imagen oficial, cuando encuentra un directorio de datos vacío, tiene
  que inicializarlo y crear la cuenta `root`. Se niega a hacerlo sin una decisión explícita sobre su
  contraseña, aunque la aplicación nunca use `root` (se conecta como `reservas_user`).
- **Cómo se resolvió:** con las cuatro variables `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER`
  y `MYSQL_PASSWORD`. Esa obligación se arrastró a todo el proyecto: está en `.env.example`, en el
  Secret `reservas-db-secret` (que la base consume entero con `envFrom`), y es la que usa la
  comprobación de estado de Compose (`mysqladmin ping -u root`).

### 3.2 El `subPath` que exige el directorio de datos vacío (entrega 7)

- **Cómo se manifestó:** al montar el PVC `reservas-db-pvc` directamente en `/var/lib/mysql`, el Pod
  de `reservas-db` quedó en `Error` / `CrashLoopBackOff`. (Para provocarlo de forma controlada, antes se
  escribió `viejo.txt` en el volumen con un Pod `busybox`.)
- **Cómo se diagnosticó:** `kubectl get pods` mostró los reinicios y `kubectl logs -l app=reservas-db`
  la causa: *"--initialize specified but the data directory has files in it. Aborting."*
  ([captura](informe/q3-subpath-data-directory-has-files.png)).
- **Causa:** MySQL solo inicializa sobre un directorio **completamente vacío**. La raíz de un volumen
  no siempre lo está: en este caso tenía `viejo.txt`, y en muchos sistemas de archivos trae
  `lost+found`. Cualquier archivo que MySQL no reconoce aborta la inicialización.
- **Cómo se resolvió:** con `subPath: mysql` en el `volumeMount`. MySQL ve como `/var/lib/mysql` el
  subdirectorio `mysql/` del volumen, que Kubernetes crea vacío. El Pod pasó a `Running` y luego se
  comprobó la persistencia borrando el Pod
  ([INFORME, persistencia](INFORME.md#c4-configuración-externalizada-y-persistencia)).

### 3.3 `cap_drop: [ALL]` y el cambio de usuario del entrypoint (entrega 4)

- **Cómo se manifestó:** al aplicar `cap_drop: [ALL]` a los tres servicios, la API y el frontend
  siguieron funcionando, pero `reservas-db` terminó con código 1. Por la dependencia `service_healthy`,
  tampoco arrancaron las otras dos capas ([captura](informe/q3-cap-drop-mysql-falla.png)).
- **Cómo se diagnosticó:** `docker compose logs reservas-db` mostró
  *"[ERROR] [MY-010126] [Server] setgid: Operation not permitted"*
  ([captura](informe/q3-cap-drop-mysql-setgid.png)).
- **Causa:** a diferencia de las imágenes propias, que arrancan directamente como usuario sin
  privilegios, el entrypoint de MySQL arranca como `root`. Ajusta dueño y permisos del directorio de
  datos y después **baja** al usuario `mysql` con `setuid`/`setgid`. Sin esas capabilities, el cambio
  de usuario falla.
- **Cómo se resolvió:** devolviendo solo las capabilities que necesita (`cap_add: CHOWN, SETUID,
  SETGID, DAC_OVERRIDE, FOWNER`) y manteniendo `cap_drop: [ALL]` para todo lo demás.

### 3.4 La comprobación de estado: el símbolo `$` y el período de gracia (entrega 4)

- **Cómo se manifestó:** la comprobación de estado necesita la contraseña de root:
  `mysqladmin ping -h 127.0.0.1 -u root -p"$$MYSQL_ROOT_PASSWORD" --silent`. Con un solo `$`, Compose
  interpola la variable **al leer el archivo** y deja la contraseña escrita en el comando del
  healthcheck, visible con `docker inspect`. Si la variable no estuviera en el `.env` de Compose, el
  comando quedaría con `-p""` y fallaría siempre.
- **Cómo se diagnosticó:** es una trampa de la interpolación de Compose, no un error que avise: el
  servicio puede quedar `healthy` igual. Se detecta leyendo el archivo ya interpolado con
  `docker compose config`, o el comando almacenado en el contenedor con `docker inspect`.
- **Cómo se resolvió:** con `$$`, Compose escribe un `$` literal y la variable la expande el intérprete
  `CMD-SHELL` **dentro** del contenedor, que ya la tiene en su entorno.
- **Período de gracia:** la primera inicialización del directorio de datos de MySQL tarda bastante:
  arranca un servidor temporal (el primer *ready for connections*, con `port: 0`), crea usuarios y
  base, y reinicia. Con el volumen ya inicializado, la base estuvo lista en unos 6 segundos. Con el
  volumen vacío tarda varias veces más. Sin `start_period: 40s`, los fallos de ese arranque contarían
  contra los `retries` y el contenedor podría quedar `unhealthy` cuando en realidad estaba arrancando,
  y con él la API y el frontend, que dependen de `service_healthy`.

### 3.5 Complemento: el esquema se crea solo al arrancar la API (entrega 6)

Cuando la base todavía no tenía volumen, al recrear su Pod el directorio de datos volvió a estar vacío
y las réplicas de la API, que siguieron vivas, respondían
`ERROR 1146: Table 'reservasdb.reservas' doesn't exist`. La API crea las tablas (`CREATE TABLE IF NOT
EXISTS`) una sola vez, al arrancar (`initDb()`), y no las recrea si la base se reinicia vacía. Se
resolvió con `kubectl rollout restart deployment/reservas-api`. La solución de fondo llegó en la
entrega 7 con el PVC.

---

## 4. Qué quedaría pendiente para un entorno de producción real

### Observabilidad

Hoy el proyecto se observa con `kubectl top` (metrics-server), `kubectl logs` y `kubectl get events`.
Alcanza para depurar a mano, pero no deja historial ni avisa de nada. En producción haría falta:

- **Métricas con historial y alertas:** Prometheus + Grafana, con métricas propias de la API (latencia,
  tasa de 5xx, cantidad de 409) y de MySQL (`mysqld-exporter`). También alertas, por ejemplo HPA en
  `maxReplicas` durante mucho tiempo o PVC casi lleno.
- **Logs centralizados:** Loki o EFK. Hoy los logs de cada réplica viven en su Pod y se pierden al
  reemplazarlo.
- **Trazas** (OpenTelemetry), para seguir un pedido navegador → nginx → API → MySQL.
- **Probes de Kubernetes:** Compose tiene comprobaciones de estado en las tres capas, pero los
  Deployments no declaran `readinessProbe` ni `livenessProbe`. Se agregaría `readinessProbe` sobre
  `/health` en la API (que ya consulta la base) y sobre `/healthz` en el frontend, para que el Service
  no mande tráfico a una réplica que todavía no está lista.

### Integración y despliegue continuos

Todo el ciclo se hizo a mano: `docker build`, `docker push`, `helm upgrade`. Un pipeline (GitHub
Actions, GitLab CI, o Argo CD para GitOps) lo automatizaría con cada commit o tag:

1. tests de la API;
2. `docker build` de ambas imágenes, con tags inmutables (por ejemplo el SHA del commit; nunca `latest`);
3. escaneo de vulnerabilidades con Trivy y **corte del pipeline** si hay críticas. En la entrega 4 el
   escaneo marcó 2 críticas en la base Alpine del frontend y 2 en dependencias de Node de la API;
4. `docker push` al registry;
5. `helm lint` + `helm template`, y `helm upgrade --install --atomic` (que revierte solo si falla) a
   un entorno de staging y luego a producción.

### Gestión externa de secretos

El Secret `reservas-db-secret` **no está cifrado: está en base64**. Se comprobó en la entrega 7
decodificando `MYSQL_PASSWORD` con `base64 -d` ([captura](informe/c4-secret-base64-decodificado.png)).
Además los valores están en el repositorio (`k8s/reservas-db-secret.yaml`, `values.yaml`). Por eso
son valores de laboratorio. En un entorno real:

- Las credenciales **no se versionan**. Se guardan en un gestor externo (HashiCorp Vault, AWS Secrets
  Manager, Google Secret Manager) y se sincronizan al clúster con **External Secrets Operator**, o se
  versionan cifradas con **Sealed Secrets** o **SOPS**.
- Se habilita el **cifrado en reposo de etcd** y se restringe con **RBAC** quién puede leer Secrets
  del namespace.
- Se rotan periódicamente, y la contraseña de `root` no debería existir como variable al alcance de
  cualquier Pod.

### Copias de respaldo de la base

Hoy no hay ninguna copia automática. El PVC usa `local-path` con política `Delete`: si el nodo se
pierde, se pierden los datos, y un `helm uninstall` borra el PVC junto con la release. Haría falta:

- Un **CronJob** con `mysqldump` (o `xtrabackup`) que suba las copias a almacenamiento externo (S3 o
  similar) con retención definida.
- **Binlogs** para restaurar a un punto en el tiempo, o snapshots de volumen si el almacenamiento lo
  soporta.
- **Probar la restauración** periódicamente: una copia que nunca se restauró no es una copia. En la
  entrega 4 se hizo a mano el ciclo completo con Compose: `tar` del volumen, `down -v`, restauración en
  un volumen nuevo y verificación de los datos.

### Por qué una base como Deployment con un volumen no es la solución de un entorno serio

Un Deployment está pensado para **réplicas intercambiables y sin estado**: Pods sin identidad estable
que se crean y destruyen libremente. Una base de datos es exactamente lo contrario:

- **No escala ni tiene alta disponibilidad.** Con un PVC `ReadWriteOnce` solo puede haber una réplica
  (sección 1c), y no hay réplica de reserva ni failover. Si el Pod o el nodo fallan, la base no está
  disponible hasta que el Pod se recrea, y con `local-path` solo puede recrearse en el **mismo nodo**
  donde está el directorio. Si el nodo muere, los datos mueren con él.
- **Las actualizaciones obligan a un corte.** El `RollingUpdate` por defecto intentaría levantar el
  Pod nuevo antes de bajar el viejo, con dos `mysqld` sobre el mismo volumen. Por eso este proyecto usa
  `strategy: Recreate`, que evita la corrupción a costa de dejar la base fuera de servicio durante
  cada actualización.
- **La identidad y el disco no van juntos.** Lo correcto en Kubernetes es un **StatefulSet**: nombre de
  red estable por réplica (`reservas-db-0`), un PVC propio por réplica (`volumeClaimTemplates`) y
  arranque ordenado. Esa es la base sobre la que se monta replicación primario/réplicas.
- **En la práctica**, además del StatefulSet, la replicación, el failover y las copias de respaldo se
  delegan en un **operador** (MySQL Operator / InnoDB Cluster, Percona Operator) o directamente en un
  servicio administrado (Amazon RDS, Cloud SQL), y la base queda fuera del ciclo de vida de la
  aplicación.

Otros pendientes menores para producción: TLS en el Ingress (cert-manager), `NetworkPolicy` para que
solo la API pueda llegar al puerto 3306, `requests`/`limits` también en la base y el frontend, y un
`PodDisruptionBudget` para la API.
