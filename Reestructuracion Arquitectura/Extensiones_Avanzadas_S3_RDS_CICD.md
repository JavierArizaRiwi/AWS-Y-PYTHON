# Extensiones Avanzadas: S3, CloudFront, RDS, Backups y CI/CD

Complemento de la arquitectura anterior con almacenamiento escalable, distribución global, base de datos administrada, respaldos automáticos y pipeline de despliegue continuo.

---

## Índice
1. [S3 + CloudFront: Contenido Estático Global](#1-s3--cloudfront-contenido-estático-global)
2. [RDS: Base de Datos Administrada](#2-rds-base-de-datos-administrada)
3. [ElastiCache: Caché en Memoria](#3-elasticache-caché-en-memoria)
4. [Backup y Disaster Recovery](#4-backup-y-disaster-recovery)
5. [CI/CD con CodePipeline](#5-cicd-con-codepipeline)
6. [VPC Security y Networking Avanzado](#6-vpc-security-y-networking-avanzado)
7. [Logging Centralizado con ELK](#7-logging-centralizado-con-elk)
8. [Arquitectura Empresarial Completa](#8-arquitectura-empresarial-completa)

---

## 1. S3 + CloudFront: Contenido Estático Global

### ¿Por qué S3 + CloudFront?

**S3 (Simple Storage Service)**:
- Almacenamiento ilimitado y escalable
- Hosting de archivos estáticos (JS, CSS, imágenes, PDFs)
- Versionado y control de acceso
- Muy barato (~$0.023 por GB/mes)

**CloudFront (CDN)**:
- Caché distribuida en 600+ edge locations globales
- Reduce latencia de usuarios distantes
- Protección DDoS integrada
- HTTPS automático con ACM

**Beneficio combinado**:
- Tu EC2 solo sirve aplicación dinámica (API)
- S3 + CloudFront sirven estáticos (imágenes, JS, CSS, PDFs)
- **Resultado**: EC2 usa 70% menos banda, soporta 5x más usuarios

---

### 1.1 Crear y Configurar Bucket S3

#### A. Crear bucket

```bash
# Crear bucket (nombres deben ser únicos globalmente)
aws s3api create-bucket \
  --bucket miapp-static-prod \
  --region us-east-1 \
  --create-bucket-configuration LocationConstraint=us-east-1

# Habilitar versionado (para rollback)
aws s3api put-bucket-versioning \
  --bucket miapp-static-prod \
  --versioning-configuration Status=Enabled

# Bloquear acceso público (importante por seguridad)
aws s3api put-public-access-block \
  --bucket miapp-static-prod \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

#### B. Crear política de acceso para EC2

EC2 necesita permiso para subir/descargar archivos:

```bash
# Crear IAM role con permisos S3
aws iam put-role-policy --role-name ec2-miapp-role --policy-name s3-access \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ],
        "Resource": [
          "arn:aws:s3:::miapp-static-prod",
          "arn:aws:s3:::miapp-static-prod/*"
        ]
      }
    ]
  }'
```

#### C. Configurar ciclo de vida (eliminar archivos temporales)

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket miapp-static-prod \
  --lifecycle-configuration '{
    "Rules": [
      {
        "Id": "DeleteTempFiles",
        "Filter": {"Prefix": "temp/"},
        "Expiration": {"Days": 7},
        "Status": "Enabled"
      },
      {
        "Id": "DeleteIncompleteUploads",
        "Filter": {},
        "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7},
        "Status": "Enabled"
      }
    ]
  }'
```

---

### 1.2 Crear CloudFront Distribution

#### A. Crear distribución desde AWS Console

1. CloudFront → Distributions → Create distribution
2. **Origin domain**: `miapp-static-prod.s3.us-east-1.amazonaws.com`
3. **Restrict bucket access**: ✓ (usar Origin Access Identity)
4. **Viewer protocol policy**: Redirect HTTP to HTTPS
5. **Cache policy**: Managed-CachingOptimized (compresión automática)
6. **CNAME**: `cdn.miapp.com` (dominio personalizado)
7. **SSL certificate**: AWS Certificate Manager (gratuito)
8. **Default TTL**: 86400 segundos (1 día)

#### B. Crear CLI equivalente

```bash
aws cloudfront create-distribution \
  --origin-domain-name miapp-static-prod.s3.us-east-1.amazonaws.com \
  --default-root-object index.html \
  --comment "CDN para contenido estático de MiApp" \
  --enabled \
  --default-cache-behavior '{
    "AllowedMethods": {"Quantity": 2, "Items": ["GET", "HEAD"]},
    "ViewerProtocolPolicy": "redirect-to-https",
    "Compress": true,
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6"
  }'
```

---

### 1.3 Integración en tu Aplicación Python

#### A. Upload automático de archivos estáticos

```python
import boto3
import os
from pathlib import Path

s3 = boto3.client('s3')
BUCKET_NAME = 'miapp-static-prod'

def upload_static_files(local_dir, s3_prefix='static'):
    """
    Sube todos los archivos estáticos a S3
    Ejemplo: upload_static_files('/app/static', 'prod/static')
    """
    for file_path in Path(local_dir).rglob('*'):
        if file_path.is_file():
            relative_path = file_path.relative_to(local_dir)
            s3_key = f"{s3_prefix}/{relative_path}"
            
            try:
                s3.upload_file(
                    str(file_path),
                    BUCKET_NAME,
                    s3_key,
                    ExtraArgs={
                        'ContentType': get_content_type(file_path),
                        'CacheControl': 'max-age=86400',  # 1 día
                    }
                )
                print(f"✓ Subido: {s3_key}")
            except Exception as e:
                print(f"✗ Error subiendo {s3_key}: {e}")

def get_content_type(file_path):
    """Detecta MIME type"""
    import mimetypes
    mime_type, _ = mimetypes.guess_type(file_path)
    return mime_type or 'application/octet-stream'

def get_cloudfront_url(filename):
    """Retorna URL de CloudFront para archivo"""
    # Con versionado para cache-busting
    import hashlib
    file_hash = hashlib.md5(open(filename, 'rb').read()).hexdigest()[:8]
    return f"https://cdn.miapp.com/prod/static/{Path(filename).name}?v={file_hash}"
```

#### B. En template HTML/Jinja2

```html
<!DOCTYPE html>
<html>
<head>
    <!-- Antes: desde EC2 (lento) -->
    <!-- <link rel="stylesheet" href="/static/style.css"> -->
    
    <!-- Después: desde CloudFront (rápido + global) -->
    <link rel="stylesheet" href="https://cdn.miapp.com/prod/static/style.css">
</head>
<body>
    <img src="https://cdn.miapp.com/prod/static/logo.png" alt="Logo">
    <script src="https://cdn.miapp.com/prod/static/app.js"></script>
</body>
</html>
```

---

### 1.4 Invalidar Caché tras Deploy

Cuando haces deploy nuevo, necesitas refrescar CloudFront:

```python
def invalidate_cloudfront(distribution_id, paths=['/prod/static/*']):
    """
    Limpia caché de CloudFront para que sirva archivos nuevos
    """
    cf = boto3.client('cloudfront')
    
    response = cf.create_invalidation(
        DistributionId=distribution_id,
        InvalidationBatch={
            'Paths': {
                'Quantity': len(paths),
                'Items': paths
            },
            'CallerReference': str(time.time())
        }
    )
    
    print(f"Invalidación iniciada: {response['Invalidation']['Id']}")
    return response
```

---

## 2. RDS: Base de Datos Administrada

### ¿Por qué RDS en lugar de DB en EC2?

**Ventajas RDS**:
- Backups automáticos diarios (hasta 35 días)
- Multi-AZ failover automático (alta disponibilidad)
- Mantenimiento automático (parches, actualizaciones)
- Encriptación en reposo y en tránsito
- Monitoreo integrado con CloudWatch
- Read replicas para escalado de lecturas
- No necesitas administrar servidor
- Mucho más seguro

**Ventajas EC2 + DB local**:
- Control total (pero requiere mantenimiento)
- Costo más bajo si bien configurado
- Mejor para dev/testing

**Recomendación**: RDS para producción, Docker local para staging.

---

### 2.1 Crear RDS Instance (PostgreSQL)

```bash
aws rds create-db-instance \
  --db-instance-identifier miapp-prod-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 15.3 \
  --master-username admin \
  --master-user-password "$(openssl rand -base64 32)" \
  --allocated-storage 100 \
  --storage-type gp3 \
  --vpc-security-group-ids sg-xxxxxxxx \
  --db-subnet-group-name default \
  --backup-retention-period 30 \
  --multi-az \
  --storage-encrypted \
  --copy-tags-to-snapshot \
  --enable-cloudwatch-logs-exports postgresql
```

### 2.2 Configurar acceso desde EC2

```bash
# Security Group de RDS debe permitir PostgreSQL (5432) desde EC2
aws ec2 authorize-security-group-ingress \
  --group-id sg-rds-xxxxxxxx \
  --protocol tcp \
  --port 5432 \
  --source-group sg-ec2-xxxxxxxx
```

### 2.3 Conectar desde aplicación Python

```python
import os
import psycopg2
from sqlalchemy import create_engine

# Variables de entorno desde AWS Secrets Manager (mejor práctico)
RDS_HOST = os.getenv('RDS_HOST', 'miapp-prod-db.xxxxx.rds.amazonaws.com')
RDS_PORT = os.getenv('RDS_PORT', '5432')
RDS_USER = os.getenv('RDS_USER', 'admin')
RDS_PASSWORD = os.getenv('RDS_PASSWORD')  # Guardar en Secrets Manager
RDS_DB = os.getenv('RDS_DB', 'miapp_db')

# Conexión con SQLAlchemy
DATABASE_URL = f"postgresql://{RDS_USER}:{RDS_PASSWORD}@{RDS_HOST}:{RDS_PORT}/{RDS_DB}"
engine = create_engine(DATABASE_URL, echo=False, pool_size=20, max_overflow=40)

# Probar conexión
try:
    with engine.connect() as conn:
        result = conn.execute("SELECT 1")
        print("✓ Conectado a RDS correctamente")
except Exception as e:
    print(f"✗ Error conectando a RDS: {e}")
```

---

### 2.4 Configurar Read Replica para escalado

```bash
# Crear read replica en otra AZ
aws rds create-db-instance-read-replica \
  --db-instance-identifier miapp-prod-db-read-1 \
  --source-db-instance-identifier miapp-prod-db \
  --db-instance-class db.t3.micro
```

Usar en aplicación:
```python
# Escrituras a master
MASTER_URL = "postgresql://admin:pw@master.rds.amazonaws.com/db"

# Lecturas a replica (distribuir carga)
REPLICA_URLS = [
    "postgresql://admin:pw@read-1.rds.amazonaws.com/db",
    "postgresql://admin:pw@read-2.rds.amazonaws.com/db"
]

def get_read_engine():
    """Elige replica aleatoria para read-heavy queries"""
    import random
    url = random.choice(REPLICA_URLS)
    return create_engine(url)
```

---

## 3. ElastiCache: Caché en Memoria

### ¿Por qué ElastiCache?

- Reduce carga en RDS y EC2
- Respuestas ultra-rápidas (microsegundos)
- Ideal para sesiones, cache de DB, leaderboards
- Redis o Memcached administrados

---

### 3.1 Crear Redis Cluster

```bash
aws elasticache create-cache-cluster \
  --cache-cluster-id miapp-cache \
  --engine redis \
  --cache-node-type cache.t3.micro \
  --engine-version 7.0 \
  --num-cache-nodes 1 \
  --vpc-security-group-ids sg-xxxxxxxx
```

### 3.2 Integrar en tu App Python

```python
import redis
import json
from functools import wraps
import hashlib

# Conectar a Redis
REDIS_HOST = os.getenv('REDIS_HOST', 'miapp-cache.xxxxx.cache.amazonaws.com')
REDIS_PORT = int(os.getenv('REDIS_PORT', '6379'))

redis_client = redis.Redis(
    host=REDIS_HOST,
    port=REDIS_PORT,
    decode_responses=True,
    socket_connect_timeout=5,
    retry_on_timeout=True
)

def cache_result(ttl=300):
    """
    Decorador para cachear resultados de funciones
    Uso: @cache_result(ttl=600)
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Generar clave basada en función y argumentos
            cache_key = f"{func.__name__}:{hashlib.md5(str((args, kwargs)).encode()).hexdigest()}"
            
            # Intentar obtener del caché
            cached = redis_client.get(cache_key)
            if cached:
                return json.loads(cached)
            
            # Si no está en caché, ejecutar función
            result = func(*args, **kwargs)
            
            # Guardar en caché
            redis_client.setex(cache_key, ttl, json.dumps(result))
            
            return result
        return wrapper
    return decorator

# Ejemplo de uso
@cache_result(ttl=600)  # 10 minutos
def obtener_usuarios(filtro='activos'):
    """Consulta a DB que será cacheada"""
    return db.query(Usuario).filter_by(estado=filtro).all()
```

---

## 4. Backup y Disaster Recovery

### 4.1 Estrategia de Backup

```
┌─────────────────────────┐
│   Datos en producción   │
│    EC2 + RDS + S3       │
└────────────┬────────────┘
             │
      ┌──────┴──────┐
      │             │
      ▼             ▼
  RDS Backup   S3 Versioning
  (automático) (automático)
      │             │
      └──────┬──────┘
             │
      ▼      ▼
   AWS Backup
   (consolidado)
      │
      ▼
   Cross-Region
   (DR en otra región)
```

### 4.2 Backup Automático de RDS

```bash
# Activar backups automáticos (ya incluido en RDS)
aws rds modify-db-instance \
  --db-instance-identifier miapp-prod-db \
  --backup-retention-period 30 \
  --preferred-backup-window "03:00-04:00"
```

### 4.3 Backup de EC2 (AMI Snapshots)

```bash
# Crear snapshot de volumen
VOLUME_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=miapp-prod" \
  --query 'Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.VolumeId' \
  --output text)

aws ec2 create-snapshot \
  --volume-id $VOLUME_ID \
  --description "Backup diario miapp-prod"

# Automatizar con Lambda (diariamente)
```

### 4.4 Backup de S3 a otra región

```bash
# Habilitar replicación de S3 a otra región
aws s3api put-bucket-replication \
  --bucket miapp-static-prod \
  --replication-configuration '{
    "Role": "arn:aws:iam::ACCOUNT:role/s3-replication-role",
    "Rules": [{
      "Status": "Enabled",
      "Priority": 1,
      "DeleteMarkerReplication": {"Status": "Enabled"},
      "Filter": {},
      "Destination": {
        "Bucket": "arn:aws:s3:::miapp-static-backup",
        "ReplicationTime": {"Status": "Enabled", "Time": {"Minutes": 15}}
      }
    }]
  }'
```

### 4.5 Script de Backup Completo

```python
import boto3
import json
from datetime import datetime

ec2 = boto3.client('ec2')
rds = boto3.client('rds')
s3 = boto3.client('s3')

def backup_everything():
    """Ejecutar backup de toda la infraestructura"""
    
    timestamp = datetime.now().isoformat()
    
    print("=== INICIANDO BACKUPS ===")
    
    # 1. RDS Snapshot
    print("\n[RDS] Creando snapshot...")
    try:
        rds.create_db_snapshot(
            DBSnapshotIdentifier=f"miapp-db-backup-{timestamp}",
            DBInstanceIdentifier="miapp-prod-db"
        )
        print("✓ RDS snapshot iniciado")
    except Exception as e:
        print(f"✗ Error en RDS: {e}")
    
    # 2. EC2 AMI
    print("\n[EC2] Creando AMI...")
    try:
        response = ec2.create_image(
            InstanceId="i-xxxxxxxx",
            Name=f"miapp-ami-{timestamp}",
            Description="Backup automático"
        )
        print(f"✓ AMI created: {response['ImageId']}")
    except Exception as e:
        print(f"✗ Error en EC2: {e}")
    
    # 3. S3 Sync
    print("\n[S3] Sincronizando datos...")
    try:
        # Sync a bucket de backup
        import subprocess
        subprocess.run([
            'aws', 's3', 'sync',
            's3://miapp-static-prod',
            's3://miapp-static-backup',
            '--delete'
        ])
        print("✓ S3 sincronizado")
    except Exception as e:
        print(f"✗ Error en S3: {e}")
    
    print("\n=== BACKUP COMPLETADO ===")

# Ejecutar vía Lambda con CloudWatch Events (diariamente a las 2am)
def lambda_handler(event, context):
    backup_everything()
    return {'status': 'success'}
```

---

## 5. CI/CD con CodePipeline

### 5.1 Flujo Automático: Git → Build → Deploy

```
GitHub Push
    │
    ▼
CodePipeline
    │
    ├─→ CodeBuild (test + build Docker image)
    │       │
    │       ├─ git clone
    │       ├─ pip install requirements.txt
    │       ├─ pytest (tests)
    │       ├─ docker build
    │       └─ docker push ECR
    │
    ├─→ CodeDeploy
    │       │
    │       ├─ Pull image de ECR
    │       ├─ docker-compose pull
    │       ├─ docker-compose up
    │       └─ Health check
    │
    └─→ Manual Approval (antes de prod)
            │
            ▼
        Deploy a Producción
            │
            ▼
        CloudWatch Monitoring
```

### 5.2 Crear CodePipeline

#### A. buildspec.yml (en raíz del repo)

```yaml
version: 0.2

env:
  variables:
    AWS_ACCOUNT_ID: "123456789"
    AWS_REGION: "us-east-1"
    IMAGE_REPO_NAME: "miapp"
    IMAGE_TAG: "latest"
  
  parameter-store:
    DOCKER_PASSWORD: /miapp/docker-password

phases:
  pre_build:
    commands:
      - echo "Logging into Docker Hub..."
      - echo $DOCKER_PASSWORD | docker login -u $DOCKER_USER --password-stdin
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$IMAGE_REPO_NAME
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}

  build:
    commands:
      - echo "Building Docker image on `date`"
      - docker build -t $REPOSITORY_URI:$IMAGE_TAG .
      - docker tag $REPOSITORY_URI:$IMAGE_TAG $REPOSITORY_URI:latest
      - echo "Running tests..."
      - docker run --rm $REPOSITORY_URI:$IMAGE_TAG pytest tests/

  post_build:
    commands:
      - echo "Pushing to ECR..."
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - docker push $REPOSITORY_URI:latest
      - echo "Writing image definitions file..."
      - printf '[{"name":"miapp","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files: imagedefinitions.json
  discard-paths: yes

logs:
  files:
    - logs/**/*
```

#### B. Crear Pipeline vía Console

1. AWS CodePipeline → Create pipeline
2. **Name**: miapp-pipeline
3. **Source**: GitHub (repo + main branch)
4. **Build**: CodeBuild → Create new project → miapp-build
5. **Deploy**: CodeDeploy → miapp-prod

---

### 5.3 Configurar CodeDeploy

#### A. appspec.yml (en raíz del repo)

```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::EC2::Instance
      Properties:
        Name: miapp-prod

Hooks:
  BeforeInstall:
    - Location: scripts/before_install.sh
      Timeout: 300
  ApplicationStart:
    - Location: scripts/start_server.sh
      Timeout: 300
  ApplicationStop:
    - Location: scripts/stop_server.sh
      Timeout: 300
```

#### B. scripts/start_server.sh

```bash
#!/bin/bash
set -e

cd /opt/apps/miapp

# Pull última imagen
docker pull $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/miapp:latest

# Actualizar docker-compose
docker-compose -f docker-compose.prod.yml pull
docker-compose -f docker-compose.prod.yml up -d

# Health check
sleep 5
curl -f http://localhost:8000/health || exit 1

echo "✓ Deploy completado exitosamente"
```

---

## 6. VPC Security y Networking Avanzado

### 6.1 Arquitectura de Seguridad en Capas

```
┌──────────────────────────────────────────┐
│         Internet (WWW)                   │
└──────────────────┬───────────────────────┘
                   │
          ┌────────▼─────────┐
          │ AWS WAF          │ ← Protección DDoS/SQL injection
          │ (CloudFront)     │
          └────────┬─────────┘
                   │
          ┌────────▼──────────────┐
          │ Public Subnet         │
          │ ┌──────────────────┐  │
          │ │  ALB (puerto 80) │  │ ← Load Balancer
          │ └────────┬─────────┘  │
          └──────────┼────────────┘
                     │
          ┌──────────▼────────────┐
          │ Private Subnet        │
          │ ┌──────────────────┐  │
          │ │  EC2 Instancias  │  │ ← Sin IP pública
          │ │  (puerto 8000)   │  │
          │ └──────┬───────────┘  │
          └────────┼──────────────┘
                   │
          ┌────────▼──────────────┐
          │ Database Subnet       │
          │ ┌──────────────────┐  │
          │ │  RDS PostgreSQL  │  │ ← Sin acceso directo
          │ │  (puerto 5432)   │  │
          │ └──────────────────┘  │
          └───────────────────────┘
```

### 6.2 Security Groups

```bash
# 1. ALB Security Group
aws ec2 create-security-group \
  --group-name miapp-alb \
  --description "Load Balancer - Allow HTTP/HTTPS from Internet"

aws ec2 authorize-security-group-ingress \
  --group-id sg-alb \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id sg-alb \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# 2. EC2 Security Group
aws ec2 create-security-group \
  --group-name miapp-ec2 \
  --description "EC2 - Only allow from ALB"

aws ec2 authorize-security-group-ingress \
  --group-id sg-ec2 \
  --protocol tcp \
  --port 8000 \
  --source-group sg-alb

# 3. RDS Security Group
aws ec2 create-security-group \
  --group-name miapp-rds \
  --description "RDS - Only allow from EC2"

aws ec2 authorize-security-group-ingress \
  --group-id sg-rds \
  --protocol tcp \
  --port 5432 \
  --source-group sg-ec2
```

### 6.3 VPC Endpoints (acceso privado a servicios AWS)

```bash
# Crear endpoint de S3 (sin salir de VPC)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxxxx \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-xxxxx

# Crear endpoint de ECR
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxxxx \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.ecr.dkr \
  --subnet-ids subnet-xxxxx
```

---

## 7. Logging Centralizado con ELK

### 7.1 Stack ELK en EC2 (Alternativa a CloudWatch)

Para máximo control sobre logs (opcional):

```bash
# Instalación rápida con Docker
docker run -d --name elasticsearch \
  -e discovery.type=single-node \
  docker.elastic.co/elasticsearch/elasticsearch:8.5.0

docker run -d --name kibana \
  -e ELASTICSEARCH_HOSTS=http://elasticsearch:9200 \
  -p 5601:5601 \
  docker.elastic.co/kibana/kibana:8.5.0

docker run -d --name logstash \
  -v $(pwd)/logstash.conf:/usr/share/logstash/pipeline/logstash.conf \
  docker.elastic.co/logstash/logstash:8.5.0
```

### 7.2 Configuración de Filebeat (en tu app EC2)

```yaml
# /etc/filebeat/filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/miapp/app.log
      - /var/log/nginx/access.log

processors:
  - add_kubernetes_metadata: ~

output.elasticsearch:
  hosts: ["localhost:9200"]

logging.level: info
```

---

## 8. Arquitectura Empresarial Completa

### 8.1 Diagrama Completo

```
                            ┌─────────────────┐
                            │   Users (Global) │
                            └────────┬─────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
                    ▼                ▼                ▼
            ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
            │  CloudFront  │  │  CloudFront  │  │  CloudFront  │
            │  (Edge US)   │  │  (Edge EU)   │  │  (Edge APAC) │
            └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
                   └──────────┬──────────────────────┬──┘
                              │
                    ┌─────────▼──────────┐
                    │   S3 (Estáticos)   │
                    │ - Imágenes         │
                    │ - JS/CSS           │
                    │ - PDFs             │
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │   API Gateway      │
                    │ - Cache 300s       │
                    │ - Rate Limit 10k/s │
                    │ - WAF integrado    │
                    └─────────┬──────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
   ┌──────────┐         ┌──────────┐         ┌──────────┐
   │  ALB     │         │  ALB     │         │  ALB     │
   │ (us-east)│         │(eu-west)│         │(ap-south)│
   └─────┬────┘         └─────┬────┘         └─────┬────┘
         │                    │                    │
   ┌─────────────────────────────────────────────┐ │
   │                                             │ │
   ▼                 ▼                 ▼         │ │
┌──────┐        ┌──────┐        ┌──────┐        │ │
│EC2-1 │        │EC2-2 │        │EC2-3 │──┐    │ │
│Docker│        │Docker│        │Docker│  │    │ │
│App   │        │App   │        │App   │  │    │ │
└──┬───┘        └──┬───┘        └──┬───┘  │    │ │
   │               │               │     │    │ │
   └───────────────┼───────────────┘     │    │ │
         ASG (Auto Scaling)              │    │ │
         • Min: 2, Max: 5                │    │ │
         • Target: CPU 70%               │    │ │
         • Apaga 18-9h                   │    │ │
                   │                     │    │ │
     ┌─────────────┴──────────┬──────────┘    │ │
     │                        │               │ │
     ▼                        ▼               │ │
┌──────────────┐        ┌──────────────┐    │ │
│ RDS Master   │        │ RDS Read     │◄───┘ │
│ (Multi-AZ)   │◄──┐    │ Replica      │      │
│ PostgreSQL   │   │    │ (Read-only)  │      │
└──────────────┘   │    └──────────────┘      │
     │             │           │              │
     └─────────────┼───────────┘              │
              Replicación                     │
                   │                          │
     ┌─────────────▼────────────┐            │
     │    ElastiCache Redis     │            │
     │  • Sessions              │◄───────────┘
     │  • Cache queries         │
     │  • Rate limiting         │
     └──────────────────────────┘

┌────────────────────────────────────────────────────┐
│           Monitoring & Logging                     │
│  ┌──────────────────────────────────────────────┐  │
│  │ CloudWatch                                   │  │
│  │ • CPU, Memory, Disk, Network metrics         │  │
│  │ • Custom business metrics                    │  │
│  │ • Dashboards y Alarmas                       │  │
│  │ • Logs centralizados                         │  │
│  └──────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────┐  │
│  │ X-Ray (Tracing)                              │  │
│  │ • Visualizar flujo de requests               │  │
│  │ • Identificar bottlenecks                    │  │
│  └──────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│           CI/CD Pipeline                           │
│  Git Push → CodeBuild → CodeDeploy → Prod          │
│  (test + docker build) → (health check) → (live)   │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│           Backup & Disaster Recovery               │
│  • RDS Snapshots (automáticos, 30 días)            │
│  • EC2 AMI snapshots (diarios)                     │
│  • S3 replicación cross-region                     │
│  • Recuperación RTO: 15 min, RPO: 1 hora           │
└────────────────────────────────────────────────────┘
```

### 8.2 Tabla de Comparativa: Deployment Básico vs Empresarial

| Aspecto | Básico | Empresarial |
|---------|--------|------------|
| **Instancias** | 1 EC2 manual | Auto Scaling 2-5 |
| **BD** | SQLite/MySQL en EC2 | RDS Multi-AZ |
| **Estáticos** | Servidos por NGINX | S3 + CloudFront |
| **Caché** | Redis local | ElastiCache Redis |
| **Tráfico** | ALB simple | API Gateway + ALB |
| **Deploy** | Manual/SSH | CI/CD CodePipeline |
| **Monitoreo** | Logs en archivo | CloudWatch + X-Ray |
| **Backups** | Manual | Automáticos 30 días |
| **Escalado** | Manual | Automático por CPU |
| **Seguridad** | Security groups | WAF + VPC endpoints |
| **Costo/mes** | ~$50-100 | ~$300-500 |
| **Disponibilidad** | ~99% | ~99.99% |
| **Recovery time** | 1+ hora | 15-30 minutos |

---

## Conclusión y Recomendaciones

### ✅ Implementar AHORA (Semanas 1-4)
1. S3 + CloudFront para estáticos
2. RDS para BD (migrar de EC2)
3. Auto Scaling básico (CPU 70%)
4. CloudWatch dashboards

### ⏳ Implementar DESPUÉS (Semanas 5-8)
1. ElastiCache Redis
2. CI/CD con CodePipeline
3. VPC security avanzado
4. Multi-AZ failover

### 🚀 Considerar LARGO PLAZO (Meses 3+)
1. Multi-región (global)
2. Kubernetes (EKS)
3. Machine Learning (SageMaker)
4. Event-driven architecture (SNS/SQS)

**ROI esperado**: Reducción de costos 40-50%, mejora de performance 3-5x, disponibilidad 99.9%+
