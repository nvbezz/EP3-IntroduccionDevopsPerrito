# Tienda Perritos — Despliegue en AWS EKS

**Asignatura:** Introducción a Herramientas DevOps (ISY1101)  
**Evaluación:** Evaluación Final Transversal (EFT)  
**Integrantes:** Yeider Catari · Yaquelin Rugel  
**Sede:** Duoc UC — Plaza Oeste

---

## Descripción

Aplicación CRUD de productos para una tienda de alimento de perros. Arquitectura de 3 capas (frontend, backend y base de datos relacional) contenerizada con Docker, orquestada en **Amazon EKS** (Auto Mode) y automatizada de punta a punta con **GitHub Actions**. El pipeline construye las imágenes, las publica en **Amazon ECR** y despliega los servicios en el clúster sin intervención manual.

---

## Arquitectura

```
Internet
   │
   ▼
Network Load Balancer (NLB · internet-facing)
   │  provisionado por el controlador nativo de EKS Auto Mode
   ▼
Frontend (nginx) — 2 réplicas mínimas — namespace: tienda
   │  proxy /api/ → tienda-backend:3001 (DNS interno del clúster)
   ▼
Backend (Node.js + Express) — 2–10 réplicas (HPA 70% CPU)
   │
   ▼
Base de datos (MySQL 8) — 1 réplica
```

| Elemento | Detalle |
|---|---|
| **Clúster** | `tienda_perritos_eks` — Kubernetes 1.36 — EKS Auto Mode — `us-east-1` |
| **Nodos** | EC2 provisionados automáticamente por Auto Mode (pools `general-purpose` + `system`) |
| **VPC** | VPC por defecto — subredes públicas en us-east-1a/b/c/d/f etiquetadas con `kubernetes.io/role/elb=1` |
| **Roles IAM** | `LabEksClusterRole` (plano de control) · `LabEksNodeRole` (nodos) |
| **Registro** | Amazon ECR — cuenta `<AWS_ACCOUNT_ID>` — región `us-east-1` |

### Justificación de la arquitectura

Se eligió **EKS** sobre ECS porque permite gestión nativa de Kubernetes (HPA, Deployments, Services, Secrets) con mayor control sobre el orquestador. **EKS Auto Mode** simplifica el aprovisionamiento de nodos sin necesidad de crear grupos de nodos manualmente, lo que resulta compatible con las restricciones de roles IAM del entorno AWS Academy (Learner Lab). El NLB internet-facing es provisionado automáticamente por el controlador de balanceo integrado en Auto Mode (`loadBalancerClass: eks.amazonaws.com/nlb`), sin requerir instalación adicional del AWS Load Balancer Controller.

---

## Tecnologías

| Componente | Tecnología |
|---|---|
| Orquestador | Amazon EKS (Kubernetes 1.36, Auto Mode) |
| Registro de imágenes | Amazon ECR |
| CI/CD | GitHub Actions |
| Frontend | nginx:alpine (imagen personalizada) |
| Backend | Node.js 18-alpine + Express |
| Base de datos | MySQL 8 |
| Autoscaling | Horizontal Pod Autoscaler (HPA) + Metrics Server |
| Monitoreo | Amazon CloudWatch (logs del plano de control) |

---

## Requisitos previos

- Cuenta AWS Academy con Learner Lab activo
- AWS CLI v2 instalado y configurado
- kubectl v1.36+ instalado
- Docker Desktop (para build local y docker-compose)
- Git

---

## Entorno de desarrollo local (docker-compose)

Para levantar toda la plataforma localmente sin necesidad de AWS:

```bash
docker compose up --build -d
```

Esto levanta los 3 servicios en una red interna dedicada (`tienda-net`):

| Servicio | Puerto local | Descripción |
|---|---|---|
| `tienda-db` | 3306 | MySQL 8 con datos iniciales precargados |
| `tienda-backend` | 3001 | API REST Node.js + Express |
| `tienda-frontend` | 8080 | nginx sirviendo el HTML estático + proxy inverso a `/api/` |

El backend arranca solo cuando MySQL pasa el healthcheck (`service_healthy`). Los datos de la base se persisten en un volumen nombrado (`db-data`).

```bash
docker compose ps                         # verificar estado
curl http://localhost:3001/api/health     # health del backend
# Abrir http://localhost:8080 en el navegador para ver el frontend
docker compose down -v                    # apagar y limpiar
```

### Buenas prácticas de contenerización aplicadas

- **Imágenes base livianas:** `node:18-alpine` (backend), `nginx:alpine` (frontend), `mysql:8` (DB).
- **`.dockerignore`** en cada componente para excluir `node_modules`, `.git` y archivos innecesarios, reduciendo el tamaño del contexto de build.
- **Healthcheck** en el servicio de base de datos para garantizar que el backend solo arranca cuando MySQL está listo.
- **Red aislada** (`tienda-net`, driver bridge) para que los contenedores se comuniquen por nombre de servicio sin exponer puertos innecesariamente.
- **Volumen nombrado** (`db-data`) para persistir los datos de MySQL entre reinicios.

---

## Variables y Secrets

### GitHub Secrets (requeridos para el pipeline)

| Secret | Descripción | Cambia |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | Credencial del Learner Lab | Cada sesión |
| `AWS_SECRET_ACCESS_KEY` | Credencial del Learner Lab | Cada sesión |
| `AWS_SESSION_TOKEN` | Token de sesión del Learner Lab | Cada sesión |
| `AWS_REGION` | `us-east-1` | Fijo |
| `EKS_CLUSTER_NAME` | `tienda_perritos_eks` | Fijo |
| `EKS_NAMESPACE` | `tienda` | Fijo |

Las credenciales AWS **nunca se almacenan en el repositorio**. Se actualizan en GitHub Secrets al inicio de cada sesión del Learner Lab, siguiendo el principio de mínimo privilegio.

### Kubernetes Secret

La contraseña de MySQL se almacena en un `Secret` de Kubernetes (`mysql-secret`) codificada en base64. El backend la consume mediante `secretKeyRef`, sin valores en texto plano en los manifests ni en el código fuente.

```bash
kubectl get secret mysql-secret -n tienda
```

---

## Pipeline CI/CD

El pipeline se define en `.github/workflows/deploy-eks.yml` y se ejecuta automáticamente en cada push a `main` o manualmente desde la pestaña Actions (`workflow_dispatch`).

### Pasos del pipeline

```
 1. Checkout del código
 2. Configurar credenciales AWS (desde GitHub Secrets)
 3. Login a Amazon ECR
 4. Definir tag de imagen: ${GITHUB_SHA::7}
 5. Build & push FRONTEND → ECR
 6. Build & push BACKEND  → ECR
 7. Build & push DB       → ECR
 8. Instalar kubectl v1.36.0
 9. aws eks update-kubeconfig → conectar al clúster
10. kubectl apply: namespace, secret, mysql, backend, frontend
11. kubectl set image: actualizar imagen en cada deployment
12. kubectl rollout status: verificar que el deploy fue exitoso
13. kubectl apply: HPA de backend y frontend
14. kubectl get pods/svc/hpa: estado final del clúster
```

El tag de imagen es el hash corto del commit (`${GITHUB_SHA::7}`), lo que permite trazabilidad directa entre cada despliegue y su código fuente.

### Trigger manual

GitHub → Actions → CI/CD Tienda Perritos EKS → **Run workflow** → Branch: main → Run workflow

---

## Despliegue manual (sin pipeline)

### 1. Configurar credenciales AWS

```bash
aws configure set aws_access_key_id TU_ACCESS_KEY
aws configure set aws_secret_access_key TU_SECRET_KEY
aws configure set aws_session_token TU_SESSION_TOKEN
aws configure set default.region us-east-1
```

### 2. Conectar kubectl al clúster

```bash
aws eks update-kubeconfig --region us-east-1 --name tienda_perritos_eks
kubectl get nodes
```

### 3. Aplicar manifests

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/mysql-secret.yaml -n tienda
kubectl apply -f k8s/mysql-deployment.yaml -n tienda
kubectl apply -f k8s/mysql-service.yaml -n tienda
kubectl apply -f k8s/backend-deployment.yaml -n tienda
kubectl apply -f k8s/backend-service.yaml -n tienda
kubectl apply -f k8s/frontend-deployment.yaml -n tienda
kubectl apply -f k8s/frontend-service.yaml -n tienda
kubectl apply -f k8s/backend-hpa.yaml -n tienda
kubectl apply -f k8s/frontend-hpa.yaml -n tienda
```

---

## Validación del despliegue

### Verificar pods y servicios

```bash
kubectl get pods -n tienda
kubectl get svc -n tienda
kubectl get hpa -n tienda
```

### Health check del backend (desde dentro del clúster)

```bash
kubectl run curl-test --rm -it --image=curlimages/curl -n tienda -- \
  curl -s http://tienda-backend:3001/api/health
```

Respuesta esperada:
```json
{"status":"ok","message":"Backend de tienda de perritos en ejecución."}
```

### Endpoints del backend

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/api/productos` | Listar todos los productos |
| GET | `/api/productos/:id` | Obtener un producto |
| POST | `/api/productos` | Crear producto |
| PUT | `/api/productos/:id` | Actualizar producto |
| DELETE | `/api/productos/:id` | Eliminar producto |
| GET | `/api/health` | Health check |

---

## Autoscaling (HPA)

El **Horizontal Pod Autoscaler** escala automáticamente los pods según el uso de CPU. El **Metrics Server** viene como addon nativo del clúster.

| Deployment | CPU umbral | Réplicas mín. | Réplicas máx. |
|---|---|---|---|
| tienda-backend | 70% | 2 | 10 |
| tienda-frontend | 60% | 2 | 6 |

### Justificación de los umbrales

- **Backend 70%:** umbral alto que permite aprovechar el nodo antes de escalar. El backend maneja lógica de negocio y consultas SQL; escalar a 70% da margen para absorber picos moderados sin escalar innecesariamente, reduciendo costos.
- **Frontend 60%:** umbral más bajo porque nginx es stateless y escala sin impacto en el estado de la aplicación. Escalar antes garantiza que el servidor web no se convierta en cuello de botella ante tráfico variable.

### Simular carga para demostrar escalado

```bash
kubectl run load-generator \
  --image=busybox:1.28 \
  --restart=Never \
  -n tienda \
  -- /bin/sh -c "while true; do wget -q -O- http://tienda-backend:3001/api/productos; done"
```

Eliminar generador:
```bash
kubectl delete pod load-generator -n tienda
```

---

## Logs y monitoreo

### Logs de la aplicación

```bash
kubectl logs deployment/tienda-backend -n tienda --tail=20
kubectl logs deployment/tienda-frontend -n tienda --tail=20
```

### Métricas de recursos

```bash
kubectl top pods -n tienda
kubectl top nodes
```

### CloudWatch

Los logs del plano de control se envían automáticamente a CloudWatch:

- AWS Console → CloudWatch → Grupos de registros → `/aws/eks/tienda_perritos_eks/cluster`

Flujos disponibles: `kube-apiserver`, `kube-apiserver-audit`, `authenticator`, `kube-controller-manager`, `kube-scheduler`.

---

## Estructura del repositorio

```
.github/
  workflows/
    deploy-eks.yml            # Pipeline CI/CD completo
backend/
  .dockerignore               # Excluye node_modules, .git, .env
  Dockerfile                  # node:18-alpine + Express
  package.json
  server.js                   # API REST (CRUD productos + health check)
db/
  .dockerignore
  Dockerfile                  # mysql:8 + script de inicialización
  init.sql                    # Tabla productos + datos de ejemplo
docker-compose.yml            # Orquestación local (frontend + backend + db)
frontend/
  .dockerignore               # Excluye .git, node_modules
  Dockerfile                  # nginx:alpine + archivos estáticos + proxy config
  app.js                      # Lógica CRUD del frontend
  default.conf                # nginx: proxy /api/ → tienda-backend:3001
  index.html
k8s/
  namespace.yaml              # Namespace 'tienda'
  mysql-secret.yaml           # Secret con la contraseña de MySQL (base64)
  mysql-deployment.yaml
  mysql-service.yaml          # Headless service (ClusterIP: None)
  backend-deployment.yaml     # 2 réplicas, probes, resource limits
  backend-service.yaml        # ClusterIP interno
  backend-hpa.yaml            # HPA 70% CPU, 2–10 réplicas
  frontend-deployment.yaml    # 2 réplicas, probes, resource limits
  frontend-service.yaml       # LoadBalancer + NLB internet-facing
  frontend-hpa.yaml           # HPA 60% CPU, 2–6 réplicas
README.md
```
