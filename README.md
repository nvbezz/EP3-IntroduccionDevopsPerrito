# Tienda Perritos — Despliegue en AWS EKS

**Asignatura:** Introducción a Herramientas DevOps (ISY1101)  
**Evaluación:** Parcial N°3  
**Integrantes:** Yeider Catari · Yaquelin Rugel

---

## Descripción

Aplicación CRUD de productos para una tienda de alimento de perros. Arquitectura de 3 capas desplegada en **Amazon EKS** con automatización completa mediante **GitHub Actions**.

---

## Arquitectura

```
Internet
   │
   ▼
Network Load Balancer (NLB · internet-facing)
   │
   ▼
Frontend (nginx) — 2 réplicas mínimas — namespace: tienda
   │  proxy /api/ → tienda-backend:3001
   ▼
Backend (Node.js + Express) — 2–10 réplicas (HPA)
   │
   ▼
Base de datos (MySQL 8) — 1 réplica
```

**Clúster:** `tienda_perritos_eks` — Kubernetes 1.36 — EKS Auto Mode — región `us-east-1`  
**Nodos:** EC2 provisionados automáticamente por EKS Auto Mode (`general-purpose` + `system`)  
**VPC:** VPC por defecto — 5 subredes públicas en us-east-1a/b/c/d/f  
**Roles IAM:** `LabEksClusterRole` (plano de control) · `LabEksNodeRole` (nodos)  
**Registro de imágenes:** Amazon ECR — cuenta `047157437257` — región `us-east-1`

### Justificación de la arquitectura

Se eligió **EKS** sobre ECS porque permite gestión nativa de Kubernetes (HPA, Deployments, Services, Secrets) con mayor control sobre el orquestador. **EKS Auto Mode** simplifica el aprovisionamiento de nodos sin necesidad de crear grupos de nodos manualmente, lo que es compatible con las restricciones de roles IAM del entorno AWS Academy (Learner Lab). El NLB internet-facing fue provisionado automáticamente por el controlador de balanceo integrado en Auto Mode, sin requerir instalación adicional del AWS Load Balancer Controller.

---

## Tecnologías

| Componente | Tecnología |
|---|---|
| Orquestador | Amazon EKS (Kubernetes 1.36, Auto Mode) |
| Registro de imágenes | Amazon ECR |
| CI/CD | GitHub Actions |
| Frontend | nginx (imagen personalizada) |
| Backend | Node.js 18 + Express |
| Base de datos | MySQL 8 |
| Autoscaling | Horizontal Pod Autoscaler (HPA) + Metrics Server |
| Monitoreo | Amazon CloudWatch + OTel Container Insights |

---

## Requisitos previos

- Cuenta AWS Academy con Learner Lab activo
- AWS CLI v2 instalado y configurado
- kubectl instalado (v1.36+)
- Docker Desktop (para build local)
- Git

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

Las credenciales AWS **nunca se almacenan en el repositorio**. Se actualizan en GitHub Secrets al inicio de cada sesión del Learner Lab.

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
7. Build & push DB        → ECR
8. Instalar kubectl v1.36.0
9. aws eks update-kubeconfig → conectar al clúster
10. kubectl apply: namespace, secret, mysql, backend, frontend
11. kubectl set image: actualizar imagen en cada deployment
12. kubectl rollout status: verificar que el deploy fue exitoso
13. kubectl apply: HPA de backend y frontend
14. kubectl get pods/svc/hpa: estado final del clúster
```

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

### URL pública del frontend

```
http://k8s-tienda-tiendafr-b667c9954b-422ee71ce91221aa.elb.us-east-1.amazonaws.com
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

El **Horizontal Pod Autoscaler** escala automáticamente los pods según el uso de CPU. El **Metrics Server** fue instalado como addon nativo del clúster (v0.8.1).

| Deployment | CPU umbral | Réplicas mín. | Réplicas máx. |
|---|---|---|---|
| tienda-backend | 70% | 2 | 10 |
| tienda-frontend | 60% | 2 | 6 |

### Justificación de los umbrales

- **Backend 70%:** umbral alto que permite aprovechar el nodo antes de escalar. El backend maneja lógica de negocio y consultas SQL; escalar a 70% da margen para absorber picos moderados sin escalar innecesariamente, reduciendo costos.
- **Frontend 60%:** umbral más bajo porque nginx es stateless y escala sin impacto en el estado de la aplicación. Escalar antes garantiza que el servidor web no se convierta en cuello de botella ante tráfico variable.

### Verificar autoscaling

```bash
kubectl get hpa -n tienda -w
```

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

## Recuperación post-deploy

Para demostrar que el clúster se recupera automáticamente tras un redeploy:

```bash
kubectl rollout restart deployment/tienda-backend -n tienda
kubectl rollout status deployment/tienda-backend -n tienda
```

El rolling update reemplaza los pods uno a uno sin downtime.

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
    deploy-eks.yml        # Pipeline CI/CD
backend/
  Dockerfile
  package.json
  server.js               # API REST Node.js + Express
db/
  Dockerfile
  init.sql                # Creación de tabla y datos iniciales
frontend/
  Dockerfile
  app.js                  # Lógica CRUD del frontend
  default.conf            # Configuración nginx + proxy /api/
  index.html
k8s/
  namespace.yaml
  mysql-secret.yaml
  mysql-deployment.yaml
  mysql-service.yaml
  backend-deployment.yaml
  backend-service.yaml
  backend-hpa.yaml
  frontend-deployment.yaml
  frontend-service.yaml
  frontend-hpa.yaml
README.md
```
