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
   │  gestionado por EKS Auto Mode (loadBalancerClass: eks.amazonaws.com/nlb)
   ▼
Frontend (nginx) — 2 réplicas mínimas — namespace: tienda
   │  proxy /api/ → tienda-backend:3001 (DNS interno del clúster)
   ▼
Backend (Node.js + Express) — 2–10 réplicas (HPA)
   │
   ▼
Base de datos (MySQL 8) — 1 réplica
```

**Clúster:** `tienda_perritos_eks` — Kubernetes 1.36 — EKS Auto Mode — región `us-east-1`  
**Nodos:** EC2 provisionados automáticamente por EKS Auto Mode (pools `general-purpose` + `system`)  
**VPC:** VPC por defecto — 5 subredes públicas en us-east-1a/b/c/d/f  
**Roles IAM:** `LabEksClusterRole` (plano de control) · `LabEksNodeRole` (nodos)  
**Registro de imágenes:** Amazon ECR — cuenta `047157437257` — región `us-east-1`

### Justificación de la arquitectura

Se eligió **EKS** sobre ECS porque permite gestión nativa de Kubernetes (HPA, Deployments, Services, Secrets) con mayor control sobre el orquestador. **EKS Auto Mode** simplifica el aprovisionamiento de nodos sin necesidad de crear grupos de nodos manualmente, compatible con las restricciones de roles IAM del entorno AWS Academy (Learner Lab). El NLB internet-facing es provisionado automáticamente por el controlador de balanceo **integrado en EKS Auto Mode** (`eks.amazonaws.com/nlb`), sin requerir la instalación por separado del AWS Load Balancer Controller. Las subredes de la VPC fueron taggeadas con `kubernetes.io/role/elb=1` para que el controlador las detecte correctamente.

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
| Autoscaling | Horizontal Pod Autoscaler (HPA) + Metrics Server v0.8.1 |
| Monitoreo | Amazon CloudWatch + OTel Container Insights |

---

## Requisitos previos

- Cuenta AWS Academy con Learner Lab activo
- AWS CLI v2 instalado y configurado
- kubectl v1.36+ instalado
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

La contraseña de MySQL se almacena en un `Secret` de Kubernetes (`mysql-secret`) codificada en base64. El backend la consume mediante `secretKeyRef`, sin valores en texto plano en los manifests.

```bash
kubectl get secret mysql-secret -n tienda
```

### Nota sobre el valor por defecto en backend/server.js

`server.js` define `DB_PASSWORD = "admin123"` como valor de fallback para ejecución local sin variables de entorno. En el clúster EKS, esta variable es **siempre sobreescrita** por el Kubernetes Secret vía `secretKeyRef`, por lo que el valor hardcodeado nunca se usa en producción. No hay credenciales reales expuestas en el repositorio.

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

El **Horizontal Pod Autoscaler** escala automáticamente los pods según el uso de CPU. El **Metrics Server** fue instalado como addon nativo del clúster (v0.8.1-eksbuild.11), sin instalación manual adicional.

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

Durante la prueba de carga el HPA escaló el backend de **2 → 10 réplicas** al superar el umbral de 70% CPU (se registró hasta 344% de uso). Al eliminar el generador, el HPA reduce réplicas gradualmente.

Eliminar generador:
```bash
kubectl delete pod load-generator -n tienda
```

---

## Recuperación post-deploy

El rolling update reemplaza pods uno a uno sin downtime. Para demostrarlo:

```bash
kubectl rollout restart deployment/tienda-backend -n tienda
kubectl rollout status deployment/tienda-backend -n tienda
```

Salida esperada:
```
deployment.apps/tienda-backend restarted
Waiting for deployment "tienda-backend" rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for deployment "tienda-backend" rollout to finish: 1 old replicas are pending termination...
deployment "tienda-backend" successfully rolled out
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

Los logs del plano de control se envían automáticamente a CloudWatch. Grupos disponibles en `/aws/eks/tienda_perritos_eks/cluster`:

| Flujo | Contenido |
|---|---|
| `kube-apiserver` | Todas las llamadas a la API del clúster |
| `kube-apiserver-audit` | Auditoría de acceso y operaciones |
| `authenticator` | Eventos de autenticación IAM |
| `kube-controller-manager` | Estado de los controladores |
| `kube-scheduler` | Decisiones de programación de pods |

AWS Console → CloudWatch → Grupos de registros → `/aws/eks/tienda_perritos_eks/cluster`

### Nota sobre errores de OpenTelemetry en logs

El addon `amazon-cloudwatch-observability` inyecta contenedores init de OTel (`opentelemetry-auto-instrumentation-nodejs`, etc.) en los pods. Al arrancar, intentan reportar trazas a AWS X-Ray y reciben `AccessDeniedException: User is not authorized to perform xray:PutTraceSegments`. Esto ocurre porque el Pod Identity del agente no tiene el permiso de X-Ray en el entorno Learner Lab. **No afecta el funcionamiento de la aplicación** — nginx, Express y MySQL operan con normalidad. Solo impacta la recolección de trazas distribuidas.

---

## Análisis de logs, métricas y tiempos del pipeline (IE6)

### Tiempos del pipeline (run exitoso — 2026-06-29)

| Step | Duración | Observación |
|---|---|---|
| Build & push DB | 14s | El más lento — imagen MySQL base más pesada |
| Build & push Backend | 8s | Node.js con dependencias npm |
| Build & push Frontend | 4s | nginx estático, imagen más liviana |
| Configurar kubeconfig | 4s | Llamada a API de EKS para obtener token |
| Aplicar manifests base | 5s | Apply + rollout status de mysql |
| Actualizar Backend en EKS | 4s | set image + rollout status |
| Actualizar Frontend en EKS | 3s | set image + rollout status |
| **Total pipeline** | **~50s** | Desde checkout hasta estado final |

### Conclusiones

- El paso más lento es el **build & push de imágenes** (26s de 50s totales = 52% del tiempo). Esto es esperado en un entorno sin caché de capas Docker entre runs.
- El **deploy al clúster** es rápido (12s) gracias al rolling update con 2 réplicas — siempre hay un pod disponible mientras el nuevo arranca.
- Los **runs fallidos iniciales** fueron causados por configuración, no por código: credenciales no configuradas, versión de kubectl incompatible y subredes sin tag. Una vez resueltos, el pipeline es estable.
- Durante la prueba de carga, `kubectl top pods` mostró el backend consumiendo hasta **5m CPU por pod** en reposo y escalando hasta 10 réplicas bajo carga, confirmando que el HPA responde correctamente a métricas reales.

---

## Problemas encontrados y solución

| Problema | Causa | Solución aplicada |
|---|---|---|
| Pipeline falla al hacer login ECR (`not authorized`) | Credenciales AWS no configuradas en GitHub Secrets | Se configuraron los 6 secrets requeridos antes del primer run |
| `kubectl rollout status` falla con error de API | kubectl v1.29 incompatible con clúster K8s 1.36 (diferencia de 7 versiones) | PR `fix/kubectl-version-workflow`: actualizar kubectl a v1.36.0 en el workflow |
| `frontend-service` queda en `EXTERNAL-IP: <pending>` indefinidamente | Subredes de la VPC sin el tag `kubernetes.io/role/elb` requerido por el controlador del NLB | Se taggearon las 5 subredes con `kubernetes.io/role/elb=1` vía AWS CLI |
| Pipeline falla con `voc-cancel-cred` (AccessDenied en ECR) | Credenciales del Learner Lab expiraron (~4h de vida útil) | Actualizar los 3 secrets de AWS en GitHub al inicio de cada sesión del Lab |

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
