# Proyecto Innovatech Chile — EP3
## Orquestación y Automatización en AWS EKS

**Asignatura:** Introducción a Herramientas DevOps — ISY1101  
**Integrantes:** Diego Manríquez — Rodrigo Jerez  
**Evaluación:** Parcial N°3

---

## Descripción General

Este proyecto implementa la tercera etapa de modernización de Innovatech Chile. Tras contenedorizar los servicios en EP2 y montar la infraestructura base en EP1, esta etapa lleva la aplicación a un entorno de **orquestación productivo y automatizado** en AWS EKS.

La solución despliega tres servicios contenerizados (Frontend React + dos Backends Spring Boot) en un clúster Kubernetes gestionado en AWS, con autoscaling automático, pipeline CI/CD completo mediante GitHub Actions y base de datos MySQL en Amazon RDS.

---

## Estructura del Repositorio

```
DevOps-Experiencia-3/
├── .github/
│   └── workflows/
│       └── deploy.yml              # Pipeline CI/CD completo
├── k8s/
│   ├── frontend.yaml               # Deployment + Service + HPA del frontend
│   ├── backend.yaml                # Deployment + Service + HPA del backend ventas
│   ├── backend-despachos.yaml      # Deployment + Service + HPA del backend despachos
│   └── db-secret.yaml              # Secret con credenciales de base de datos
├── terraform/
│   ├── main.tf                     # Infraestructura AWS (EKS, VPC, RDS, ECR, SG)
│   ├── variables.tf                # Variables configurables
│   └── outputs.tf                  # Outputs de la infraestructura
├── front-despacho/                 # Aplicación React (Frontend)
├── back-ventas-springboot/         # API REST Spring Boot (Backend Ventas)
└── back-bespachos-springboot/      # API REST Spring Boot (Backend Despachos)
```

---

## Arquitectura

```
GitHub → GitHub Actions → Amazon ECR → AWS EKS → Usuario
                                           │
                          ┌────────────────┼────────────────┐
                          │                │                 │
                     Frontend          Backend           Backend
                     (React)           Ventas           Despachos
                     HPA: 2-10        HPA: 2-10         HPA: 2-10
                     pods             pods              pods
                          │                │                 │
                     LoadBalancer     ClusterIP         ClusterIP
                     Service          Service           Service
                                           │
                                      Amazon RDS
                                       (MySQL 8.0)
```

**Red:** VPC con CIDR `10.0.0.0/16`, dos subredes públicas (`10.0.101.0/24`, `10.0.102.0/24`) y dos privadas (`10.0.1.0/24`, `10.0.2.0/24`) en zonas `us-east-1a` y `us-east-1b`. Los nodos EKS y la base de datos RDS se ubican en subredes privadas. El frontend se expone mediante un LoadBalancer público.

---

## Infraestructura (Terraform)

Toda la infraestructura está definida como código en `/terraform`.

| Recurso | Descripción |
|---|---|
| VPC | CIDR `10.0.0.0/16` con subredes públicas y privadas |
| NAT Gateway | Permite salida a internet desde subredes privadas |
| EKS Cluster | `devops-eks-cluster`, versión Kubernetes 1.30 |
| Node Group | Instancias `t3.medium`, mínimo 1, deseado 2, máximo 4 nodos |
| Amazon RDS | MySQL 8.0, instancia `db.t3.micro`, 20GB, en subred privada |
| ECR | `frontend-repo` y `backend-repo` para almacenar imágenes Docker |
| IAM | Rol `LabRole` reutilizado para clúster y nodos |
| Security Groups | SG para EKS (tráfico general) y RDS (solo MySQL port 3306 desde VPC) |

### Levantar la infraestructura

```bash
cd terraform
terraform init
terraform plan -var="db_password=TuPasswordSegura"
terraform apply -var="db_password=TuPasswordSegura"
```

---

## Servicios y Manifiestos Kubernetes

### Frontend (`k8s/frontend.yaml`)
- **Imagen:** `frontend-repo` en ECR
- **Réplicas:** 2 iniciales
- **Puerto:** 80 (Nginx)
- **Variable de entorno:** `BACKEND_URL` apuntando al servicio interno del backend ventas
- **Servicio:** `LoadBalancer` — expone el frontend públicamente
- **HPA:** 2–10 réplicas, umbral 50% CPU

### Backend Ventas (`k8s/backend.yaml`)
- **Imagen:** `backend-repo:ventas-*` en ECR
- **Réplicas:** 2 iniciales
- **Puerto:** 8080
- **Credenciales:** cargadas desde `db-secret` (sin hardcodeo)
- **Probes:** readiness (delay 45s) y liveness (delay 60s) sobre TCP 8080
- **Servicio:** `ClusterIP` — accesible solo dentro del clúster como `backend-service`
- **HPA:** 2–10 réplicas, umbral 50% CPU

### Backend Despachos (`k8s/backend-despachos.yaml`)
- **Imagen:** `backend-repo:despachos-*` en ECR
- **Réplicas:** 2 iniciales
- **Puerto:** 8080
- **Credenciales:** cargadas desde `db-secret` (sin hardcodeo)
- **Probes:** readiness (delay 45s) y liveness (delay 60s) sobre TCP 8080
- **Servicio:** `ClusterIP` — accesible solo dentro del clúster como `backend-despachos-service`
- **HPA:** 2–10 réplicas, umbral 50% CPU

### Autoscaling — Justificación del umbral

Se eligió **50% de uso de CPU** como umbral de escalado para los tres servicios. Este valor permite que Kubernetes reaccione antes de que los pods se saturen, dando margen para levantar nuevas réplicas sin que el servicio se degrade. Un valor más alto (ej. 80%) reaccionaría tarde; uno más bajo (ej. 20%) provocaría escalado innecesario y mayor costo.

---

## Gestión de Secrets y Credenciales

Las credenciales de base de datos **nunca están hardcodeadas** en el código ni en los manifiestos finales. Se gestionan mediante un Kubernetes Secret (`db-secret`) que los pods consumen como variables de entorno:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

El Secret debe aplicarse manualmente antes del primer despliegue:

```bash
# Editar k8s/db-secret.yaml con los valores reales del RDS
kubectl apply -f k8s/db-secret.yaml
```

Las credenciales AWS del pipeline (access key, secret key, session token) se almacenan como **GitHub Secrets** y se inyectan en el workflow sin exponerse en el código.

---

## Pipeline CI/CD (`.github/workflows/deploy.yml`)

El pipeline se activa automáticamente ante cada `push` a la rama `main` y ejecuta los siguientes pasos en orden:

```
1. Checkout del código
2. Configuración de credenciales AWS (desde GitHub Secrets)
3. Login a Amazon ECR
4. Build y push de imagen Frontend (tag desde archivo VERSION)
5. Build y push de imagen Backend Ventas (tag: ventas-*)
6. Build y push de imagen Backend Despachos (tag: despachos-*)
7. Actualización del kubeconfig para conectar con EKS
8. Sustitución de placeholders de imagen en los manifiestos k8s
9. kubectl apply de los tres deployments
10. Verificación del rollout (kubectl rollout status, timeout 300s)
```

### Control de versiones de imágenes

Cada servicio tiene dos archivos de control:
- `VERSION`: versión que se construye y sube a ECR en el paso de build.
- `DEPLOY_VERSION`: versión que se despliega en el clúster en el paso de deploy.

Esto permite construir una versión nueva sin desplegarla inmediatamente, controlando exactamente qué versión está corriendo en producción.

---

## Comunicación entre Servicios

El frontend se comunica con el backend ventas usando DNS interno de Kubernetes:

```
http://backend-service.default.svc.cluster.local
```

El backend despachos es accesible internamente como:

```
http://backend-despachos-service.default.svc.cluster.local
```

Ambos backends solo aceptan tráfico interno del clúster (tipo `ClusterIP`). Solo el frontend está expuesto públicamente mediante `LoadBalancer`.

---

## Despliegue Manual (primera vez)

```bash
# 1. Levantar infraestructura
cd terraform
terraform apply -var="db_password=TuPassword"

# 2. Configurar kubectl
aws eks update-kubeconfig --region us-east-1 --name devops-eks-cluster

# 3. Aplicar secret con datos reales del RDS
kubectl apply -f k8s/db-secret.yaml

# 4. Aplicar manifiestos (o dejar que el pipeline lo haga)
kubectl apply -f k8s/backend.yaml
kubectl apply -f k8s/backend-despachos.yaml
kubectl apply -f k8s/frontend.yaml

# 5. Verificar pods
kubectl get pods
kubectl get svc
kubectl get hpa
```

---

## Herramientas Utilizadas

| Herramienta | Uso |
|---|---|
| AWS EKS | Orquestación de contenedores |
| AWS ECR | Registro de imágenes Docker |
| AWS RDS (MySQL 8.0) | Base de datos relacional |
| Terraform | Infraestructura como código |
| GitHub Actions | Pipeline CI/CD |
| kubectl | Gestión del clúster |
| Docker | Contenerización de servicios |
| React + Vite | Frontend |
| Spring Boot | Backends REST |
