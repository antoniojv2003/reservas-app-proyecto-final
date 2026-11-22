# reservas-app — Proyecto final (Variante B)

Sistema de reservas de salas por franja horaria, contenerizado con Docker y orquestado en Kubernetes.
Proyecto final del curso **Contenedores, Docker y Orquestación con Kubernetes** (FACET – UNT).

| Capa | Tecnología | Imagen | Puerto interno |
| --- | --- | --- | --- |
| `reservas-frontend` | React + Vite, servido por nginx sin privilegios | `antoniojv03/reservas-frontend:v1` | 8080 |
| `reservas-api` | Node 20 + Express | `antoniojv03/reservas-api:v1` (y `v2`) | 3000 |
| `reservas-db` | MySQL 8.4 | `mysql:8.4` | 3306 |

```
navegador ──> [Ingress reservas.local] ──> reservas-frontend (nginx) ──/api/──> reservas-api ──> reservas-db
                                           sirve la SPA y hace de          Node + Express     MySQL + PVC
                                           proxy inverso hacia la API      (2 a 6 réplicas,
                                                                            HPA sobre CPU)
```

El frontend es el único punto de entrada. El navegador nunca habla con la API directamente: pide
`/api/...` a su propio origen y nginx lo reenvía a `reservas-api`. Por eso no hace falta CORS y la
misma imagen del frontend sirve para cualquier entorno (ver [ARQUITECTURA.md](ARQUITECTURA.md)).

## Contenido del repositorio

| Ruta | Qué es |
| --- | --- |
| [`backend/`](backend) | Código de la API y su `Dockerfile` multietapa (`Dockerfile.ingenuo` es la versión de una sola etapa usada como línea base en la entrega 2) |
| [`frontend/`](frontend) | Código de la SPA, su `Dockerfile` multietapa y la plantilla de nginx (`nginx/templates/default.conf.template`) |
| [`docker-compose.yaml`](docker-compose.yaml) | Las tres capas en una sola máquina: red propia, volumen de la base y comprobaciones de estado |
| [`.env.example`](.env.example) | Contrato de variables que consume Compose (el `.env` real está en `.gitignore`) |
| [`k8s/`](k8s) | Los **12 objetos** de Kubernetes, un archivo por objeto |
| [`k8s/historico/`](k8s/historico) | Manifiestos de entregas intermedias que **no** forman parte del proyecto (Pods sueltos de la entrega 5 y el NodePort de depuración de la entrega 6). Se conservan solo como referencia; `kubectl apply -f k8s/` no los aplica porque no es recursivo |
| [`reservas-app/`](reservas-app) | Chart de Helm: `Chart.yaml`, `values.yaml` y los 12 objetos como plantillas parametrizadas |
| [`ARQUITECTURA.md`](ARQUITECTURA.md) | Respuestas a las preguntas de arquitectura |
| [`INFORME.md`](INFORME.md) | Demostración de que el despliegue funciona, con capturas en [`informe/`](informe) |

### Los 12 objetos de Kubernetes

| # | Objeto | Nombre | Manifiesto | Plantilla del chart |
| --- | --- | --- | --- | --- |
| 1 | Deployment | `reservas-frontend` | `k8s/deploy-reservas-frontend.yaml` | `deployment-reservas-frontend.yaml` |
| 2 | Deployment | `reservas-api` | `k8s/deploy-reservas-api.yaml` | `deployment-reservas-api.yaml` |
| 3 | Deployment | `reservas-db` | `k8s/deploy-reservas-db.yaml` | `deployment-reservas-db.yaml` |
| 4 | Service (NodePort 30301) | `reservas-frontend` | `k8s/svc-reservas-frontend.yaml` | `service-reservas-frontend.yaml` |
| 5 | Service (ClusterIP) | `reservas-api` | `k8s/svc-reservas-api.yaml` | `service-reservas-api.yaml` |
| 6 | Service (ClusterIP) | `reservas-db` | `k8s/svc-reservas-db.yaml` | `service-reservas-db.yaml` |
| 7 | ConfigMap | `reservas-frontend-config` | `k8s/reservas-frontend-config.yaml` | `configmap-reservas-frontend.yaml` |
| 8 | ConfigMap | `reservas-api-config` | `k8s/reservas-api-config.yaml` | `configmap-reservas-api.yaml` |
| 9 | Secret | `reservas-db-secret` | `k8s/reservas-db-secret.yaml` | `secret-reservas-db.yaml` |
| 10 | PersistentVolumeClaim | `reservas-db-pvc` | `k8s/reservas-db-pvc.yaml` | `pvc-reservas-db.yaml` |
| 11 | Ingress | `reservas-app-ingress` | `k8s/reservas-app-ingress.yaml` | `ingress-reservas-app.yaml` |
| 12 | HorizontalPodAutoscaler | `reservas-api-hpa` | `k8s/reservas-api-hpa.yaml` | `hpa-reservas-api.yaml` |

> **Credenciales:** el Secret (`k8s/reservas-db-secret.yaml` y `db.secret` en `values.yaml`) y
> `.env.example` contienen **valores de laboratorio**, no credenciales reales. Cómo se gestionarían en
> un entorno real está explicado en [ARQUITECTURA.md, sección 4](ARQUITECTURA.md#4-qué-quedaría-pendiente-para-un-entorno-de-producción-real).

---

## Camino 1: Docker Compose (una sola máquina)

Requisitos: Docker Engine con el plugin `docker compose`.

```bash
git clone https://github.com/antoniojv2003/reservas-app-proyecto-final.git
cd reservas-app-proyecto-final

# 1. Crear el .env a partir del contrato (y cambiar las contraseñas)
cp .env.example .env

# 2. Construir las imágenes y levantar las tres capas
docker compose up -d --build

# 3. Esperar a que los tres servicios estén (healthy)
docker compose ps
```

- Aplicación: <http://localhost:3000>
- API directa, solo para depuración: <http://localhost:3001/health>
- La base no publica ningún puerto: solo es alcanzable desde la red `reservas-net`.

Compose arranca en orden por las comprobaciones de estado: `reservas-db` (healthy) → `reservas-api` (healthy) → `reservas-frontend`.

```bash
docker compose logs -f reservas-api   # logs de un servicio
docker compose down                   # elimina contenedores y red; conserva el volumen reservas-db-data
docker compose down -v                # además elimina el volumen: se pierden los datos
```

---

## Camino 2: Helm sobre el clúster

### Requisitos del clúster

| Requisito | Para qué | Comprobación |
| --- | --- | --- |
| Ingress Controller con clase `nginx` | Atender el Ingress `reservas-app-ingress` | `kubectl get ingressclass` |
| metrics-server | Que el HPA tenga métricas de CPU | `kubectl top pods -A` |
| StorageClass por defecto (por ej. `local-path`) | Aprovisionar el PVC de la base | `kubectl get storageclass` |
| Acceso a Docker Hub | Descargar `antoniojv03/reservas-api` y `antoniojv03/reservas-frontend` (públicas) | — |

### Instalación

```bash
git clone https://github.com/antoniojv2003/reservas-app-proyecto-final.git
cd reservas-app-proyecto-final

# 1. Validar antes de instalar: lint y renderizado (no toca el clúster)
helm lint ./reservas-app
helm template reservas-app ./reservas-app -n reservas-app | grep -E '^kind:' | wc -l   # -> 12

# 2. Instalar la release (crea el namespace)
helm install reservas-app ./reservas-app -n reservas-app --create-namespace

# 3. Verificar los 12 objetos
kubectl get all,cm,secret,pvc,ingress,hpa -n reservas-app
```

### Acceso

```bash
# IP del Ingress Controller (columna ADDRESS)
kubectl get ingress reservas-app-ingress -n reservas-app

# Resolver reservas.local hacia esa IP
echo "<ADDRESS-del-ingress> reservas.local" | sudo tee -a /etc/hosts
```

- Aplicación por el Ingress: <http://reservas.local>
- Alternativa por NodePort: `http://<IP-del-nodo>:30301`
- Verificación válida de la API a través del Ingress: `curl -s http://reservas.local/api/reservas` (devuelve JSON).
  `curl http://reservas.local/reservas`, sin `/api`, devuelve `200` con el `index.html` de la SPA: es un falso positivo.

### Parámetros principales de `values.yaml`

| Clave | Valor por defecto | Qué controla |
| --- | --- | --- |
| `frontend.image.tag` / `api.image.tag` | `v1` / `v1` | Tag de cada imagen propia (ciclos de vida independientes) |
| `frontend.replicas` / `api.replicas` / `db.replicas` | `2` / `2` / `1` | Réplicas iniciales (las de la API las administra luego el HPA) |
| `api.hpa.minReplicas` / `maxReplicas` / `targetCPUUtilizationPercentage` | `2` / `6` / `50` | Umbrales del autoescalador |
| `api.resources.requests` / `limits` | `100m`, `128Mi` / `300m`, `256Mi` | El request de CPU es el denominador del porcentaje del HPA |
| `api.config.*` / `frontend.config.*` | ver archivo | Contenido de los dos ConfigMaps |
| `db.pvc.storage` | `1Gi` | Tamaño del volumen de la base |
| `ingress.host` / `ingress.className` | `reservas.local` / `nginx` | Host y clase del Ingress |
| `frontend.service.nodePort` | `30301` | Puerto de nodo del frontend |

Comprobar que un parámetro es real: `helm template reservas-app ./reservas-app --set api.image.tag=v2 | grep image:`
tiene que mostrar el cambio.

### Ciclo de vida de la release

```bash
helm upgrade reservas-app ./reservas-app -n reservas-app --set api.image.tag=v2   # revisión 2
helm history reservas-app -n reservas-app
helm rollback reservas-app 1 -n reservas-app                                      # crea la revisión 3
helm uninstall reservas-app -n reservas-app   # ¡también borra el PVC y, con él, los datos!
```

`helm upgrade` **no** conserva los `--set` de la instalación: hay que repetirlos o usar `--reuse-values`.

### Alternativa: aplicar los manifiestos sueltos

Los mismos 12 objetos sin Helm (no combinar con la release en el mismo namespace):

```bash
kubectl create namespace reservas-app
kubectl apply -f k8s/
```

---

## Historial del proyecto

Las ocho entregas semanales se resolvieron en el repositorio
[antoniojv2003/reservas-app](https://github.com/antoniojv2003/reservas-app), que conserva el registro
punto por punto de cada trabajo práctico. Este repositorio reúne el estado final para la entrega.
