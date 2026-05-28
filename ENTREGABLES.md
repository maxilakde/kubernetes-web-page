# Entregables — Migración Minikube → EKS

## Estado del despliegue

| Paso | Estado |
|---|---|
| AWS CLI configurado | Completado (`maxilakde`, cuenta `347685736828`) |
| Imágenes Docker Hub | Completado (`maximilianoastor/backend:2.0`, `maximilianoastor/frontend:2.0`) |
| Manifiesto EKS | Completado ([`k8s_eks.yaml`](k8s_eks.yaml)) |
| Clúster EKS | **Pendiente** — error de permisos IAM (ver abajo) |
| Despliegue y validación | Pendiente (después de crear el clúster) |

---

## Bloqueo actual: permisos IAM

Al ejecutar `eksctl create cluster` aparece:

```
AccessDeniedException: User arn:aws:iam::347685736828:user/maxilakde
is not authorized to perform: eks:DescribeClusterVersions
```

### Solución (en consola AWS)

1. Ir a **IAM** → **Users** → `maxilakde`
2. **Add permissions** → **Attach policies directly**
3. Adjuntar una de estas opciones (según lo que permita el curso):
   - **Opción A (recomendada para laboratorio):** `AdministratorAccess`
   - **Opción B (más restrictiva):** políticas mínimas de [eksctl](https://eksctl.io/usage/minimum-iam-policies/):
     - `AmazonEKSClusterPolicy`
     - `AmazonEKSServicePolicy`
     - `AmazonEC2FullAccess`
     - `IAMFullAccess` (eksctl crea roles IAM)
     - `CloudFormationFullAccess`

4. Esperar 1–2 minutos y volver a ejecutar:

```powershell
eksctl create cluster `
  --name devops-web-app `
  --region us-east-1 `
  --nodegroup-name standard-workers `
  --node-type t3.small `
  --nodes 2 `
  --nodes-min 1 `
  --nodes-max 3 `
  --managed
```

---

## Comandos restantes (después de crear el clúster)

```powershell
# Verificar clúster
kubectl get nodes
kubectl cluster-info

# Desplegar aplicación
kubectl apply -f k8s_eks.yaml

# Esperar pods y LoadBalancer
kubectl get pods -w
kubectl get svc frontend-service -w

# Obtener URL pública
kubectl get svc frontend-service
```

---

## URL de acceso

```
http://<EXTERNAL-IP-o-hostname-del-LoadBalancer>/
```

Credenciales de prueba: `admin` / `1234`

Verificar balanceo: `http://<LB>/whoami` (debe alternar hostnames de pods)

---

## Archivos YAML entregables

| Archivo | Entorno | Descripción |
|---|---|---|
| [`k8s.yaml`](k8s.yaml) | Minikube | Imágenes locales, ClusterIP, `imagePullPolicy: Never` |
| [`k9s_ec2.yaml`](k9s_ec2.yaml) | EC2/NodePort | Imágenes Docker Hub, NodePort 30000/30001 |
| [`k8s_eks.yaml`](k8s_eks.yaml) | **EKS** | Imágenes Docker Hub, backend ClusterIP, frontend LoadBalancer |

---

## Breve explicación del proceso (para la entrega)

La aplicación full stack (Nginx + Express) se migró de Minikube a EKS. En local, las imágenes se construían con `minikube docker-env` y se desplegaban con Services ClusterIP e `imagePullPolicy: Never`. Para EKS, las imágenes se publicaron en Docker Hub bajo `maximilianoastor/backend:2.0` y `maximilianoastor/frontend:2.0`, y el manifiesto `k8s_eks.yaml` usa `imagePullPolicy: Always`.

El backend permanece como Service ClusterIP (acceso solo interno al cluster). El frontend se expone con un Service tipo LoadBalancer, que provisiona un Elastic Load Balancer de AWS. Nginx en el frontend proxifica las peticiones `/api/` al Service `backend-service:3000` usando el DNS interno de Kubernetes, sin cambios en el código de la aplicación.

Se desplegaron 4 réplicas del frontend y 1 del backend. La validación incluye login (`admin`/`1234`), comunicación frontend→backend y balanceo de carga mediante el endpoint `/whoami`.

---

## Capturas sugeridas para el curso

1. Consola AWS → EKS → cluster `devops-web-app` + `kubectl get nodes`
2. `kubectl get pods`, `kubectl get svc`, `kubectl get deployments`
3. Pantalla de login y dashboard en el navegador
4. Resultado de `/whoami` mostrando distintos pods

---

## Limpieza (al terminar la práctica)

```powershell
kubectl delete -f k8s_eks.yaml
eksctl delete cluster --name devops-web-app --region us-east-1
```
