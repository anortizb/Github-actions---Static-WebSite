# SmartMedia Labs — Plataforma de Procesamiento Inteligente de Imágenes

Plataforma serverless basada en eventos para carga, optimización y almacenamiento de imágenes, desplegada sobre AWS mediante Infrastructure as Code (CloudFormation).

> **Estado actual:** infraestructura base desplegada. Pendiente: funciones Lambda y conexión S3 trigger → Lambda de procesamiento.

---

## Arquitectura

```
Usuario
  │
  ▼
[S3 — sml-frontend]
  │  (sitio estático: http://sml-frontend.s3-website-us-east-1.amazonaws.com)
  │
  ▼
[API Gateway — sml-api]
  │  (https://bg7yhanxyg.execute-api.us-east-1.amazonaws.com/prod)
  │
  ├── POST /upload      ──► [Lambda — sml-generate-presigned-url]  ⚠️ PENDIENTE
  ├── GET  /history     ──► [Lambda — sml-get-history]             ⚠️ PENDIENTE
  └── GET  /status/{id} ──► [Lambda — sml-get-history]             ⚠️ PENDIENTE
         │                        │
         │                        └──► [DynamoDB — sml-image-metadata]  ✅
         │
         ▼
[S3 — sml-images-input]  ✅
  │  (acceso privado, CORS configurado)
  │
  │  (S3 Event Notification → trigger)  ⚠️ PENDIENTE (requiere Lambda)
  ▼
[Lambda — sml-process-image]  ⚠️ PENDIENTE
  │
  ├──► [S3 — sml-images-output]        ✅  (imagen optimizada, lectura pública)
  ├──► [DynamoDB — sml-image-metadata] ✅  (metadatos + estado)
  └──► [SNS — sml-image-notifications] ✅  (notificación URL procesada)
```

### Leyenda

| Símbolo | Significado |
|---|---|
| ✅ | Recurso desplegado y operativo |
| ⚠️ PENDIENTE | Recurso o integración por implementar |

---

## Recursos AWS

### Cuenta y región

| Campo | Valor |
|---|---|
| Account ID | `372123585270` |
| Región | `us-east-1` |

---

### S3 Buckets

| Bucket | ARN | Acceso | Estado |
|---|---|---|---|
| `sml-frontend` | `arn:aws:s3:::sml-frontend` | Lectura pública | ✅ |
| `sml-images-input` | `arn:aws:s3:::sml-images-input` | Privado + CORS | ✅ |
| `sml-images-output` | `arn:aws:s3:::sml-images-output` | Lectura pública (`s3:GetObject`) | ✅ |

**URL frontend:** `http://sml-frontend.s3-website-us-east-1.amazonaws.com`  
**URL patrón imagen procesada:** `https://sml-images-output.s3.us-east-1.amazonaws.com/{imageId}.jpg`

---

### DynamoDB

| Campo | Valor |
|---|---|
| Tabla | `sml-image-metadata` |
| ARN | `arn:aws:dynamodb:us-east-1:372123585270:table/sml-image-metadata` |
| Partition key | `imageId` (String) |
| Billing mode | `PAY_PER_REQUEST` |
| Deletion protection | Habilitada |
| Estado | ✅ |

**Esquema de item:**

```json
{
  "imageId":       "uuid-v4",
  "fileName":      "foto.jpg",
  "originalSize":  4523891,
  "processedSize": 845712,
  "status":        "PROCESSED",
  "outputUrl":     "https://sml-images-output.s3.us-east-1.amazonaws.com/uuid-v4.jpg",
  "createdAt":     "2026-05-10T15:30:00Z",
  "processedAt":   "2026-05-10T15:30:08Z"
}
```

**Estados válidos:** `PENDING` → `PROCESSING` → `PROCESSED` / `FAILED`

---

### SNS

| Campo | Valor |
|---|---|
| Topic | `sml-image-notifications` |
| ARN | `arn:aws:sns:us-east-1:372123585270:sml-image-notifications` |
| Tipo | Standard |
| Suscripción activa | `omaruzgonzalez@gmail.com` (EMAIL, confirmada) |
| Estado | ✅ |

---

### IAM Role (Lambdas)

| Campo | Valor |
|---|---|
| Role | `sml-lambda-execution-role` |
| ARN | `arn:aws:iam::372123585270:role/sml-lambda-execution-role` |
| Trusted entity | `lambda.amazonaws.com` |
| Estado | ✅ |

**Políticas adjuntas:**

| Política | Alcance |
|---|---|
| `AWSLambdaBasicExecutionRole` | Logs en CloudWatch |
| `AmazonS3FullAccess` | Buckets `sml-images-input` / `sml-images-output` |
| `AmazonDynamoDBFullAccess` | Tabla `sml-image-metadata` |
| `AmazonSNSFullAccess` | Topic `sml-image-notifications` |

---

### API Gateway

| Campo | Valor |
|---|---|
| Stack | `sml-api` |
| API ID | `bg7yhanxyg` |
| Stage | `prod` |
| URL base | `https://bg7yhanxyg.execute-api.us-east-1.amazonaws.com/prod` |
| Estado | ✅ (endpoints activos con integración MOCK — pendiente conectar Lambdas) |

**Endpoints:**

| Method | Path | Integración actual | Integración final |
|---|---|---|---|
| POST | `/upload` | MOCK | `sml-generate-presigned-url` ⚠️ |
| GET | `/history` | MOCK | `sml-get-history` ⚠️ |
| GET | `/status/{id}` | MOCK | `sml-get-history` ⚠️ |

CORS (OPTIONS preflight) habilitado en los tres endpoints.

---

### Lambda Functions

| Función | Runtime | Trigger | Estado |
|---|---|---|---|
| `sml-generate-presigned-url` | Node.js 18.x | API Gateway `POST /upload` | ⚠️ PENDIENTE |
| `sml-process-image` | Node.js 18.x | S3 event (`sml-images-input`) | ⚠️ PENDIENTE |
| `sml-get-history` | Node.js 18.x | API Gateway `GET /history`, `GET /status/{id}` | ⚠️ PENDIENTE |

---

## Estructura del repositorio

```
smartmedia-labs/
├── backend/
│   ├── infra/
│   │   └── api-gateway.yaml              # CloudFormation — API Gateway (stack sml-api) ✅
│   ├── sml-generate-presigned-url/
│   │   └── index.js                      # Lambda: genera presigned URL
│   ├── sml-process-image/
│   │   └── index.js                      # Lambda: comprime y optimiza imagen
│   └── sml-get-history/
│       └── index.js                      # Lambda: consulta DynamoDB
├── frontend/
│   ├── index.html
│   ├── app.js
│   └── styles.css
├── .github/
│   └── workflows/
│       └── deploy.yml                    # Pipeline CI/CD (GitHub Actions)
└── README.md
```

---

## Pipeline CI/CD

Definido en `.github/workflows/deploy.yml` usando **GitHub Actions**.  
Las credenciales de despliegue corresponden al usuario IAM `sml-cicd-user`.

### Etapas

```
push a main
    │
    ▼
1. Checkout del código
    │
    ▼
2. Configurar credenciales AWS (GitHub Secrets)
    │
    ▼
3. npm install en cada función Lambda
    │
    ▼
4. Desplegar / actualizar stack CloudFormation (sml-api)
    │
    ▼
5. Desplegar funciones Lambda (aws lambda update-function-code)
    │
    ▼
6. aws s3 sync frontend/ → s3://sml-frontend/
```

### Secrets requeridos en GitHub

| Secret | Descripción |
|---|---|
| `AWS_ACCESS_KEY_ID` | Access key de `sml-cicd-user` |
| `AWS_SECRET_ACCESS_KEY` | Secret key de `sml-cicd-user` |
| `AWS_REGION` | `us-east-1` |

---

## Despliegue manual

### Actualizar stack API Gateway

```bash
aws cloudformation deploy \
  --template-file backend/infra/api-gateway.yaml \
  --stack-name sml-api \
  --region us-east-1
```

### Sincronizar frontend

```bash
aws s3 sync ./frontend/dist s3://sml-frontend/ --delete
```

### Configurar trigger S3 → Lambda (ejecutar una vez desplegada `sml-process-image`)

```bash
aws lambda add-permission \
  --function-name sml-process-image \
  --statement-id s3-trigger \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::sml-images-input

aws s3api put-bucket-notification-configuration \
  --bucket sml-images-input \
  --notification-configuration file://s3-event.json
```

---

## Verificación de recursos

```bash
# Identidad activa
aws sts get-caller-identity

# Buckets
aws s3 ls | grep sml-

# DynamoDB
aws dynamodb describe-table \
  --table-name sml-image-metadata \
  --query "Table.{Status:TableStatus,ARN:TableArn}"

# SNS
aws sns list-subscriptions-by-topic \
  --topic-arn arn:aws:sns:us-east-1:372123585270:sml-image-notifications

# IAM Role
aws iam list-attached-role-policies --role-name sml-lambda-execution-role

# API Gateway
aws apigateway get-rest-api --rest-api-id bg7yhanxyg --region us-east-1
```

---

## Cleanup

```bash
# S3 (vaciar antes de borrar)
for bucket in sml-images-input sml-images-output sml-frontend; do
  aws s3 rm s3://$bucket --recursive
  aws s3api delete-bucket --bucket $bucket
done

# DynamoDB (deshabilitar deletion protection primero)
aws dynamodb update-table \
  --table-name sml-image-metadata \
  --no-deletion-protection-enabled
aws dynamodb delete-table --table-name sml-image-metadata

# SNS
aws sns delete-topic \
  --topic-arn arn:aws:sns:us-east-1:372123585270:sml-image-notifications

# IAM Role
for policy in AWSLambdaBasicExecutionRole AmazonS3FullAccess AmazonDynamoDBFullAccess AmazonSNSFullAccess; do
  aws iam detach-role-policy \
    --role-name sml-lambda-execution-role \
    --policy-arn arn:aws:iam::aws:policy/$policy
done
aws iam delete-role --role-name sml-lambda-execution-role

# Stack API Gateway
aws cloudformation delete-stack --stack-name sml-api
```



_______________________________________________________________________________


# SmartMedia Labs — Plataforma de Procesamiento Inteligente de Imágenes

Plataforma serverless basada en eventos para procesamiento automático de imágenes, desplegada sobre AWS con infraestructura como código (Terraform) y pipeline CI/CD.

---

## Tabla de Contenidos

- [Arquitectura](#arquitectura)
- [Recursos AWS](#recursos-aws)
- [Estructura del Repositorio](#estructura-del-repositorio)
- [Prerrequisitos](#prerrequisitos)
- [Variables de Entorno y Configuración](#variables-de-entorno-y-configuración)
- [Despliegue con Terraform](#despliegue-con-terraform)
- [Pipeline CI/CD](#pipeline-cicd)
- [Monitoreo y Observabilidad](#monitoreo-y-observabilidad)
- [Flujo de Procesamiento](#flujo-de-procesamiento)

---

## Arquitectura

```
Usuario
  │
  ▼
[S3 — Frontend Estático]
  │  (carga imagen vía pre-signed URL)
  ▼
[S3 — Bucket de Entrada]
  │  (evento s3:ObjectCreated)
  ▼
[AWS Lambda — Procesador de Imágenes]
  │  ├─► [S3 — Bucket de Salida]  ←── imagen optimizada
  │  ├─► [DynamoDB]               ←── metadatos del procesamiento
  │  └─► [SNS]                    ←── notificación con URL final
  │
  ▼
[API Gateway]  ←── consulta de historial / estado
  │
  ▼
[AWS Lambda — API Handler]
  │
  ▼
[DynamoDB]
```

### Decisiones de diseño

- **Desacoplamiento por eventos**: el trigger de Lambda es un evento nativo de S3, no una llamada sincrónica desde la API.
- **Pre-signed URLs**: la carga de imágenes se hace directamente al bucket de entrada desde el cliente, sin pasar por Lambda ni API Gateway (reduce latencia y costo).
- **Dos Lambdas separadas**: una para procesamiento de imágenes (triggered por S3) y otra para exponer la API REST (triggered por API Gateway). Principio de responsabilidad única.
- **IaC completa con Terraform**: todos los recursos son reproducibles y versionados.

---

## Recursos AWS

### Cómputo

| Recurso | Nombre lógico | Propósito |
|---|---|---|
| AWS Lambda | `image-processor` | Optimiza la imagen y persiste resultados |
| AWS Lambda | `api-handler` | Expone endpoints REST para historial y pre-signed URL |

### Almacenamiento

| Recurso | Nombre lógico | Propósito |
|---|---|---|
| S3 Bucket | `smartmedia-frontend` | Hosting del frontend estático |
| S3 Bucket | `smartmedia-images-input` | Recibe imágenes originales cargadas por usuarios |
| S3 Bucket | `smartmedia-images-output` | Almacena imágenes optimizadas |
| DynamoDB Table | `image-metadata` | Metadatos: nombre, tamaño, estado, URLs, timestamps |

### Mensajería y API

| Recurso | Nombre lógico | Propósito |
|---|---|---|
| Amazon SNS | `image-processed-topic` | Publica notificación con URL de imagen procesada |
| API Gateway (HTTP API) | `smartmedia-api` | Expone endpoints: `POST /upload-url`, `GET /images` |

### Seguridad y Accesos

| Recurso | Propósito |
|---|---|
| IAM Role — `image-processor-role` | Permisos mínimos para Lambda procesadora (S3 read/write, DynamoDB write, SNS publish) |
| IAM Role — `api-handler-role` | Permisos mínimos para Lambda API (S3 presign, DynamoDB read) |
| S3 Bucket Policy | Acceso público de solo lectura al bucket de salida (imágenes servidas directamente) |
| S3 Block Public Access | Activo en buckets de entrada y frontend |

### Monitoreo y Observabilidad

| Recurso | Propósito |
|---|---|
| CloudWatch Log Groups | Logs estructurados de cada Lambda (retención: 14 días) |
| CloudWatch Metrics | Métricas nativas: invocaciones, errores, duración, throttles |
| CloudWatch Alarm — `lambda-errors` | Alarma si errores Lambda > 5 en 5 minutos |
| CloudWatch Alarm — `lambda-duration` | Alarma si P95 de duración supera el 80% del timeout configurado |
| CloudWatch Dashboard | Panel unificado: errores, latencia, tamaño reducido (antes/después) |
| AWS X-Ray | Trazabilidad distribuida en ambas Lambdas |
| SNS Alarm Topic | Canal dedicado para notificaciones de alarmas operacionales |

> **Nota**: X-Ray se habilita con `tracing_config { mode = "Active" }` en el recurso `aws_lambda_function` de Terraform. El dashboard de CloudWatch consolida métricas clave en una sola vista.

---

## Estructura del Repositorio

```
smartmedia-labs/
├── terraform/
│   ├── main.tf               # Provider y backend remoto
│   ├── variables.tf          # Definición de variables
│   ├── outputs.tf            # Outputs: URLs, ARNs, nombres de recursos
│   ├── s3.tf                 # Buckets y políticas
│   ├── lambda.tf             # Funciones Lambda e IAM roles
│   ├── dynamodb.tf           # Tabla de metadatos
│   ├── api_gateway.tf        # HTTP API y rutas
│   ├── sns.tf                # Tópico y suscripciones
│   ├── monitoring.tf         # Log Groups, Alarms, Dashboard, X-Ray
│   └── terraform.tfvars.example
├── backend/
│   ├── image-processor/
│   │   ├── handler.py        # Lambda: descarga, optimiza, persiste
│   │   └── requirements.txt  # Pillow o similar
│   └── api-handler/
│       ├── handler.py        # Lambda: genera pre-signed URL, consulta historial
│       └── requirements.txt
├── frontend/
│   ├── index.html
│   ├── app.js
│   └── styles.css
├── .github/
│   └── workflows/
│       └── deploy.yml        # Pipeline CI/CD
└── README.md
```

---

## Prerrequisitos

- [Terraform](https://developer.hashicorp.com/terraform/install) >= 1.6
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) configurado con perfil válido
- Python >= 3.11 (para empaquetar Lambdas)
- Credenciales AWS con permisos suficientes para crear los recursos listados

---

## Variables de Entorno y Configuración

Copiar el archivo de ejemplo y ajustar los valores:

```bash
cp terraform/terraform.tfvars.example terraform/terraform.tfvars
```

**`terraform.tfvars.example`**:

```hcl
aws_region       = "us-east-1"
environment      = "dev"
project_name     = "smartmedia"
sns_email        = "ops@smartmedia.com"      # Receptor de notificaciones de alarmas
image_quality    = 75                         # Calidad JPEG para optimización (0-100)
lambda_timeout   = 30                         # Segundos
lambda_memory    = 512                        # MB
log_retention_days = 14
```

> **No versionar `terraform.tfvars`**. Está incluido en `.gitignore`.

---

## Despliegue con Terraform

### 1. Inicializar

```bash
cd terraform
terraform init
```

### 2. Validar y planificar

```bash
terraform validate
terraform plan -var-file="terraform.tfvars"
```

### 3. Aplicar

```bash
terraform apply -var-file="terraform.tfvars"
```

### 4. Obtener outputs

```bash
terraform output
# Ejemplo de outputs:
# frontend_url        = "http://smartmedia-frontend.s3-website-us-east-1.amazonaws.com"
# api_endpoint        = "https://xxxx.execute-api.us-east-1.amazonaws.com"
# output_bucket_name  = "smartmedia-images-output-dev"
```

### 5. Destruir infraestructura

```bash
terraform destroy -var-file="terraform.tfvars"
```

### Backend remoto (recomendado para equipos)

Configurar el backend de Terraform en S3 + DynamoDB para estado compartido y lock:

```hcl
# terraform/main.tf
terraform {
  backend "s3" {
    bucket         = "smartmedia-tfstate"
    key            = "smartmedia/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "smartmedia-tfstate-lock"
    encrypt        = true
  }
}
```

---

## Pipeline CI/CD

El pipeline se ejecuta en **GitHub Actions** y cubre dos etapas:

### Triggers

- `push` a rama `main` → despliegue a producción
- `push` a rama `develop` → despliegue a entorno de staging
- `pull_request` → solo validación (no despliegue)

### Etapas

```
1. Checkout del código
2. Setup Python + instalar dependencias de Lambdas
3. Empaquetar Lambdas (zip)
4. Setup Terraform
5. terraform fmt --check
6. terraform validate
7. terraform plan (en PRs: solo plan, sin apply)
8. terraform apply --auto-approve (en push a main/develop)
9. Sync frontend a S3
10. Invalidar caché de CloudFront (si aplica)
```

### Secrets requeridos en GitHub

| Secret | Descripción |
|---|---|
| `AWS_ACCESS_KEY_ID` | Credencial de IAM para CI/CD |
| `AWS_SECRET_ACCESS_KEY` | Credencial de IAM para CI/CD |
| `AWS_REGION` | Región de despliegue |

> Usar un IAM user/role dedicado para el pipeline con permisos acotados a los recursos del proyecto.

---

## Monitoreo y Observabilidad

### CloudWatch Dashboard

Acceder desde la consola AWS: **CloudWatch → Dashboards → smartmedia-dashboard**

Widgets incluidos:
- Invocaciones y errores de cada Lambda (últimas 24h)
- Duración promedio y P95 de `image-processor`
- Tamaño promedio de reducción (antes vs. después, desde métricas custom)
- Estado de alarmas activas

### Alarmas configuradas

| Alarma | Condición | Acción |
|---|---|---|
| `lambda-processor-errors` | Errores > 5 en 5 min | Notifica al SNS de ops |
| `lambda-processor-duration` | P95 > 80% del timeout | Notifica al SNS de ops |
| `lambda-api-errors` | Errores > 10 en 5 min | Notifica al SNS de ops |

### Métricas custom (emitidas desde Lambda)

```python
# Emitidas con boto3 CloudWatch put_metric_data
# Namespace: SmartMedia/ImageProcessing
- ImageSizeBeforeBytes
- ImageSizeAfterBytes
- CompressionRatioPct
```

### Trazas con X-Ray

Habilitado en ambas Lambdas. Permite visualizar la cadena completa:
`API Gateway → Lambda API → [S3 presign]` y `S3 Event → Lambda Processor → S3 → DynamoDB → SNS`

Acceder desde: **CloudWatch → X-Ray traces → Service Map**

---

## Flujo de Procesamiento

```
1. Usuario solicita pre-signed URL  →  GET /upload-url
2. Frontend carga imagen directamente a S3 input bucket usando la pre-signed URL
3. S3 genera evento s3:ObjectCreated
4. Lambda image-processor se activa automáticamente
5. Lambda descarga la imagen original, la optimiza (reduce tamaño en bytes)
6. Lambda sube imagen optimizada al bucket de salida
7. Lambda registra metadatos en DynamoDB:
   { filename, size_before, size_after, status, output_url, timestamp }
8. Lambda publica en SNS: { output_url, filename, compression_ratio }
9. Frontend consulta historial  →  GET /images  →  Lambda api-handler  →  DynamoDB
```

### Estados del procesamiento (DynamoDB)

| Estado | Descripción |
|---|---|
| `PENDING` | Imagen subida, procesamiento no iniciado |
| `PROCESSING` | Lambda activa procesando |
| `DONE` | Procesamiento exitoso |
| `ERROR` | Fallo durante el procesamiento |

---

## Buenas Prácticas Aplicadas

- **Principio de mínimo privilegio**: cada Lambda tiene un IAM role con solo los permisos que necesita.
- **Separación de buckets**: entrada y salida en buckets distintos para evitar loops de eventos.
- **Pre-signed URLs**: la carga no transita por la API, reduciendo costo y latencia.
- **IaC idempotente**: todos los recursos definidos en Terraform; sin configuración manual.
- **Estado remoto**: backend S3 + lock DynamoDB para trabajo en equipo.
- **Retención de logs acotada**: 14 días para controlar costos de CloudWatch Logs.
- **Monitoreo desde el inicio**: alarmas y dashboard definidos en el mismo IaC, no como afterthought.
- **Variables parametrizadas**: sin valores hardcodeados en el código Terraform.
