# Día 5 (02/02/2026) – Despliegue Controlado y Soporte

## 🔹 Teoría

### Ciclo de vida del despliegue

```
Desarrollo (local)
    ↓
Staging (preproducción)
    ↓
Testing final + validación
    ↓
Producción (usuarios reales)
    ↓
Monitoreo y alertas
    ↓
Soporte y debugging
    ↓
Actualización o Rollback
```

### Estrategias de despliegue

#### Blue-Green Deployment
```
Versión actual (BLUE) → usuarios
Versión nueva (GREEN) → sin tráfico
    ↓
Validaciones
    ↓
Switch al GREEN
    ↓
BLUE permanece como fallback
```

**Ventaja**: Rollback instantáneo

#### Canary Deployment
```
Versión actual → 95% del tráfico
Versión nueva → 5% del tráfico
    ↓
Monitorear métricas
    ↓
Si OK → aumentar a 50%/100%
    ↓
Si error → rollback
```

**Ventaja**: Reduce riesgo con usuarios reales

#### Rolling Deployment
```
Instancia 1: actual → nueva
Instancia 2: actual → nueva
Instancia 3: actual → nueva

Sin downtime
Sin exceso de recursos
```

### Elementos de una arquitectura productiva

```
Internet
    ↓
DNS (Route 53)
    ↓
Load Balancer (ALB/NLB)
    ↓
Auto Scaling Group (EC2 x3+)
    ├─ EC2-1: Docker + App
    ├─ EC2-2: Docker + App
    └─ EC2-3: Docker + App
    ↓
RDS (Base de datos)
S3 (Almacenamiento)
ElastiCache (Cache)
    ↓
CloudWatch (Logs, métricas)
CloudTrail (Auditoría)
```

### Health checks y readiness
```
Health check (cada 5 seg):
GET /health → HTTP 200
 Aplicación está viva

Readiness check:
GET /ready → Verifica:
- BD conectada
- Cache accesible
- S3 reachable
- Sin migraciones pendientes
```

## 🔹 Conceptos Clave

### Build: Crear artefacto
```bash
# Opción 1: Imagen Docker
docker build -t myapp:v1.2.3 .
docker push myregistry/myapp:v1.2.3

# Opción 2: ZIP para Lambda
zip -r function.zip app/ requirements.txt
aws lambda update-function-code \
  --function-name my-function \
  --zip-file fileb://function.zip

# Opción 3: Ejecutable
pyinstaller main.py --onefile
```

### Deploy: Desplegar a producción
```bash
# 1. Detener instancia antigua
aws ec2 stop-instances --instance-ids i-123456

# 2. Lanzar nueva instancia
aws ec2 run-instances --image-id ami-123456 \
  --instance-type t3.medium \
  --security-group-ids sg-123456 \
  --user-data file://user_data.sh

# 3. Registrar en load balancer
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --targets Id=i-newinstance
```

### Configuración de aplicación
```python
# main.py - diferencias por entorno
import os
from config import DevelopmentConfig, ProductionConfig

env = os.getenv("ENVIRONMENT", "development")

if env == "production":
    app.config.from_object(ProductionConfig)
else:
    app.config.from_object(DevelopmentConfig)

# config.py
class ProductionConfig:
    DEBUG = False
    DATABASE_URL = os.getenv("DATABASE_URL")  # De AWS Secrets
    SECRET_KEY = os.getenv("SECRET_KEY")
    LOG_LEVEL = "INFO"
    ALLOWED_HOSTS = ["api.example.com"]

class DevelopmentConfig:
    DEBUG = True
    DATABASE_URL = "postgresql://localhost/myapp"
    LOG_LEVEL = "DEBUG"
```

### Logs centralizados
```
Aplicación (stdout)
    ↓
CloudWatch Agent → CloudWatch Logs
    ↓
Log Group: /aws/ec2/myapp
Log Streams: instance-1, instance-2, instance-3
    ↓
Búsqueda, filtrado, alarmas
```

### Monitoreo y alertas
```
Métricas críticas:
- CPU > 80% → escalable
- Memoria > 85% → alerta
- Disco > 90% → crítica
- Response time > 500ms → degradación
- Error rate > 1% → alerta

Alarmas → SNS → Email/Slack/Pagerduty
```

### Checklist de lanzamiento
```
[ ] Código revisado
[ ] Tests verdes
[ ] Base de datos migrada
[ ] Variables de entorno configuradas
[ ] Secrets en Secrets Manager
[ ] Certificates validados
[ ] Logs centralizados
[ ] Alarmas configuradas
[ ] Rollback plan listo
[ ] Equipo de guardia disponible
[ ] Monitoreo activo durante 1h
```

## 🔹 Ejemplo Real: Despliegue EC2 vs Lambda

### Despliegue en EC2 (Blue-Green)
```bash
#!/bin/bash
# deploy-ec2.sh

set -e

VERSION=$1  # v1.2.3
ENVIRONMENT=${2:-production}
AWS_REGION=us-east-1

echo "=== Desplegando $VERSION a EC2 ==="

# 1. Compilar imagen Docker
echo "1. Compilando imagen..."
docker build -t myregistry/myapp:$VERSION .
docker push myregistry/myapp:$VERSION

# 2. Actualizar Launch Template
echo "2. Actualizando Launch Template..."
aws ec2 create-launch-template-version \
  --launch-template-name myapp \
  --source-version '$Latest' \
  --overrides ImageId=$(docker inspect myregistry/myapp:$VERSION --format='{{.Id}}') \
  --region $AWS_REGION

# 3. Crear nuevo ASG (GREEN)
echo "3. Lanzando GREEN ASG..."
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name myapp-asg-$VERSION \
  --launch-template LaunchTemplateName=myapp,Version='$Latest' \
  --min-size 2 \
  --max-size 6 \
  --desired-capacity 3 \
  --vpc-zone-identifier "subnet-123,subnet-456" \
  --region $AWS_REGION

# 4. Esperar a que instancias pasen health check
echo "4. Esperando health checks..."
aws autoscaling wait group-has-desired-capacity \
  --auto-scaling-group-names myapp-asg-$VERSION \
  --region $AWS_REGION

# 5. Validar endpoints
echo "5. Validando endpoints..."
for i in {1..10}; do
  if curl -f http://internal-lb:8000/health > /dev/null; then
    echo "✓ Health check OK"
    break
  fi
  echo "Intento $i..."
  sleep 5
done

# 6. Switch traffic (Blue to Green)
echo "6. Cambiando tráfico..."
aws elbv2 modify-target-group \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --target-type instance \
  --region $AWS_REGION

# Registrar nuevas instancias
INSTANCE_IDS=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names myapp-asg-$VERSION \
  --query 'AutoScalingGroups[0].Instances[*].InstanceId' \
  --output text \
  --region $AWS_REGION)

for instance_id in $INSTANCE_IDS; do
  aws elbv2 register-targets \
    --target-group-arn arn:aws:elasticloadbalancing:... \
    --targets Id=$instance_id \
    --region $AWS_REGION
done

# 7. Monitoreo durante 5 minutos
echo "7. Monitoreando error rate..."
for i in {1..60}; do
  ERROR_RATE=$(aws cloudwatch get-metric-statistics \
    --namespace MyApp \
    --metric-name ErrorRate \
    --start-time $(date -u -d '1 minute ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 60 \
    --statistics Average \
    --query 'Datapoints[0].Average' \
    --output text \
    --region $AWS_REGION)
  
  if (( $(echo "$ERROR_RATE > 5" | bc -l) )); then
    echo "❌ Error rate alto: $ERROR_RATE%"
    echo "Ejecutar: bash rollback-ec2.sh $VERSION"
    exit 1
  fi
  
  sleep 5
done

# 8. Eliminar ASG anterior (BLUE)
echo "8. Limpiando ASG anterior..."
aws autoscaling delete-auto-scaling-group \
  --auto-scaling-group-name myapp-asg-old \
  --force-delete \
  --region $AWS_REGION

echo "✓ Despliegue completado exitosamente"
```

### Despliegue en Lambda (Canary)
```yaml
# template.yaml - Canary deployment
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: python3.11
    Timeout: 30

Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: .
      Handler: lambda_function.lambda_handler
      AutoPublishAlias: live
      DeploymentPreference:
        Type: Canary10Percent5Minutes  # 10% al inicio, esperar 5 min
        Alarms:
          - !Ref CanaryErrorsAlarm
        TriggerConfigurations:
          - DeploymentTrigger:
              LogicalId: MyFunction
              Type: DeploymentSuccess
      Environment:
        Variables:
          TABLE_NAME: Users

  CanaryErrorsAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: MyFunction-Canary-Errors
      AlarmDescription: Rollback si error rate > 5%
      MetricName: Errors
      Namespace: AWS/Lambda
      Statistic: Sum
      Period: 300
      EvaluationPeriods: 1
      Threshold: 100
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: FunctionName
          Value: !Ref MyFunction
        - Name: Resource
          Value: !Sub '${MyFunction}:live'
      AlarmActions:
        - !Ref RollbackTopic

  RollbackTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: lambda-canary-rollback

Outputs:
  FunctionArn:
    Value: !GetAtt MyFunction.Arn
  LiveAlias:
    Value: !Ref MyFunctionAlias
```

**Monitoreo durante Canary**:
```bash
# Ver métricas en tiempo real
watch -n 5 'aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Duration \
  --dimensions Name=FunctionName,Value=MyFunction \
  --start-time $(date -u -d "10 minutes ago" +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Average,Maximum'

# Ver invocaciones por versión
aws lambda get-alias --function-name MyFunction --name live

# Rollback manual si es necesario
aws lambda update-alias \
  --function-name MyFunction \
  --name live \
  --function-version 5  # Versión anterior
```

## 🔹 Práctica Paso a Paso

### 1. Preparar código para producción
Crea `config.py`:
```python
import os
from datetime import timedelta

class BaseConfig:
    """Configuración compartida"""
    SECRET_KEY = os.getenv("SECRET_KEY", "dev-secret")
    JSON_SORT_KEYS = False
    JSONIFY_PRETTYPRINT_REGULAR = False
    SESSION_COOKIE_SECURE = True
    SESSION_COOKIE_HTTPONLY = True
    PERMANENT_SESSION_LIFETIME = timedelta(days=7)

class DevelopmentConfig(BaseConfig):
    """Desarrollo local"""
    DEBUG = True
    TESTING = False
    SQLALCHEMY_ECHO = True
    LOG_LEVEL = "DEBUG"
    DATABASE_URL = os.getenv(
        "DATABASE_URL",
        "postgresql://postgres:password@localhost/myapp_dev"
    )
    AWS_REGION = "us-east-1"
    ENVIRONMENT = "development"

class StagingConfig(BaseConfig):
    """Preproducción"""
    DEBUG = False
    TESTING = False
    SQLALCHEMY_ECHO = False
    LOG_LEVEL = "INFO"
    DATABASE_URL = os.getenv("DATABASE_URL")
    AWS_REGION = os.getenv("AWS_REGION", "us-east-1")
    ENVIRONMENT = "staging"

class ProductionConfig(BaseConfig):
    """Producción"""
    DEBUG = False
    TESTING = False
    SQLALCHEMY_ECHO = False
    LOG_LEVEL = "WARNING"
    DATABASE_URL = os.getenv("DATABASE_URL")
    AWS_REGION = os.getenv("AWS_REGION", "us-east-1")
    ENVIRONMENT = "production"
    SENTRY_DSN = os.getenv("SENTRY_DSN")  # Error tracking

config = {
    "development": DevelopmentConfig,
    "staging": StagingConfig,
    "production": ProductionConfig,
    "default": DevelopmentConfig
}
```

### 2. Crear Dockerfile optimizado
Crea `Dockerfile`:
```dockerfile
# Etapa 1: Build
FROM python:3.11-slim as builder

WORKDIR /build

# Instalar dependencias de compilación
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Copiar y instalar Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Etapa 2: Runtime (ligero)
FROM python:3.11-slim

WORKDIR /app

# Instalar dependencias de runtime solo
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Copiar dependencias instaladas desde builder
COPY --from=builder /root/.local /root/.local

# Copiar código
COPY app/ app/
COPY main.py .
COPY config.py .

# User no-privilegiado
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Environment
ENV PATH=/root/.local/bin:$PATH \
    PYTHONUNBUFFERED=1 \
    ENVIRONMENT=production

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

EXPOSE 8000

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "--timeout", "60", "main:app"]
```

### 3. User data script para EC2
Crea `scripts/user_data.sh`:
```bash
#!/bin/bash
set -e

# Log everything
exec > >(tee -a /var/log/user_data.log)
exec 2>&1

echo "=== Iniciando user data script ==="

# Variables de entorno
export ENVIRONMENT=${ENVIRONMENT:-production}
export AWS_REGION=${AWS_REGION:-us-east-1}

# Actualizar sistema
sudo yum update -y
sudo yum install -y docker git

# Iniciar Docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user

# Obtener secretos de Secrets Manager
DB_PASSWORD=$(aws secretsmanager get-secret-value \
  --secret-id myapp/db-password \
  --region $AWS_REGION \
  --query SecretString --output text)

API_KEY=$(aws secretsmanager get-secret-value \
  --secret-id myapp/api-key \
  --region $AWS_REGION \
  --query SecretString --output text)

# Crear archivo de variables de entorno
cat > /home/ec2-user/.env << EOF
ENVIRONMENT=$ENVIRONMENT
DATABASE_URL=postgresql://admin:$DB_PASSWORD@mydb.region.rds.amazonaws.com:5432/myapp
API_KEY=$API_KEY
AWS_REGION=$AWS_REGION
LOG_LEVEL=INFO
EOF

# Cambiar permisos
sudo chown ec2-user:ec2-user /home/ec2-user/.env
sudo chmod 600 /home/ec2-user/.env

# Descargar imagen Docker
docker pull myregistry/myapp:${APP_VERSION:-latest}

# Lanzar contenedor
docker run -d \
  --name myapp \
  --restart always \
  -p 8000:8000 \
  --env-file /home/ec2-user/.env \
  -v /var/log/myapp:/app/logs \
  myregistry/myapp:${APP_VERSION:-latest}

# Esperar a que la app esté lista
echo "Esperando a que la app esté lista..."
for i in {1..30}; do
  if curl -f http://localhost:8000/health > /dev/null 2>&1; then
    echo "App lista!"
    break
  fi
  echo "Intento $i..."
  sleep 2
done

echo "=== User data script completado ==="
```

### 4. Health check y readiness endpoints
Crea `app/routes/system.py`:
```python
from flask import Blueprint, jsonify
from app.utils.logger import setup_logger
from sqlalchemy import text

system_bp = Blueprint("system", __name__, url_prefix="/")
logger = setup_logger(__name__)

def check_database():
    """Verifica conexión a BD"""
    try:
        from app.models.database import SessionLocal
        db = SessionLocal()
        db.execute(text("SELECT 1"))
        db.close()
        return True
    except Exception as e:
        logger.error(f"DB check failed: {str(e)}")
        return False

def check_s3():
    """Verifica acceso a S3"""
    try:
        import boto3
        s3 = boto3.client("s3")
        s3.head_bucket(Bucket="myapp-bucket")
        return True
    except Exception as e:
        logger.error(f"S3 check failed: {str(e)}")
        return False

@system_bp.route("/health", methods=["GET"])
def health_check():
    """Liveness probe - ¿está viva la app?"""
    logger.debug("Health check")
    return jsonify({"status": "healthy"}), 200

@system_bp.route("/ready", methods=["GET"])
def readiness_check():
    """Readiness probe - ¿está lista para tráfico?"""
    
    checks = {
        "database": check_database(),
        "s3": check_s3()
    }
    
    logger.debug(f"Readiness check: {checks}")
    
    # Si todas las dependencias están OK
    if all(checks.values()):
        return jsonify({
            "status": "ready",
            "checks": checks
        }), 200
    else:
        return jsonify({
            "status": "not_ready",
            "checks": checks
        }), 503  # Service Unavailable

@system_bp.route("/version", methods=["GET"])
def get_version():
    """Retorna versión de la aplicación"""
    return jsonify({
        "version": "1.2.3",
        "environment": "production",
        "timestamp": "2026-02-02T10:30:00Z"
    }), 200
```

### 5. Logging centralizado en CloudWatch
Crea `scripts/cloudwatch-config.json`:
```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "cwagent"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/myapp/*.log",
            "log_group_name": "/aws/ec2/myapp",
            "log_stream_name": "{instance_id}",
            "timestamp_format": "%Y-%m-%d %H:%M:%S"
          },
          {
            "file_path": "/var/log/docker.log",
            "log_group_name": "/aws/ec2/docker",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  },
  "metrics": {
    "namespace": "MyApp",
    "metrics_collected": {
      "cpu": {
        "measurement": [
          {
            "name": "cpu_usage_idle",
            "rename": "CPU_IDLE",
            "unit": "Percent"
          },
          {
            "name": "cpu_usage_user",
            "rename": "CPU_USER",
            "unit": "Percent"
          }
        ],
        "metrics_collection_interval": 60
      },
      "mem": {
        "measurement": [
          {
            "name": "mem_used_percent",
            "rename": "MEM_USED",
            "unit": "Percent"
          }
        ],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": [
          {
            "name": "used_percent",
            "rename": "DISK_USED",
            "unit": "Percent"
          }
        ],
        "metrics_collection_interval": 60,
        "resources": ["/"]
      }
    }
  }
}
```

### 6. CloudFormation/Terraform para Infrastructure
Crea `infra/main.tf`:
```hcl
# AWS provider
provider "aws" {
  region = var.aws_region
}

# Security Group
resource "aws_security_group" "app" {
  name = "myapp-sg"
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Launch Template (configuración)
resource "aws_launch_template" "app" {
  name_prefix   = "myapp-"
  image_id      = var.ami_id
  instance_type = "t3.medium"
  
  vpc_security_group_ids = [aws_security_group.app.id]
  
  iam_instance_profile {
    name = aws_iam_instance_profile.ec2.name
  }
  
  user_data = base64encode(file("${path.module}/../scripts/user_data.sh"))
  
  monitoring {
    enabled = true
  }
  
  tag_specifications {
    resource_type = "instance"
    tags = {
      Name        = "myapp-instance"
      Environment = var.environment
    }
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "app" {
  name                = "myapp-asg"
  vpc_zone_identifier = var.subnet_ids
  min_size            = 2
  max_size            = 6
  desired_capacity    = 3
  
  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
  
  health_check_type          = "ELB"
  health_check_grace_period  = 300
  
  tag {
    key                 = "Name"
    value               = "myapp-asg"
    propagate_launch_template = true
  }
}

# CloudWatch Alarms
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "myapp-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_actions       = [var.sns_topic_arn]
}
```

### 7. Monitoreo y dashboards
```python
# Crear métrica personalizada
import boto3

cloudwatch = boto3.client("cloudwatch")

cloudwatch.put_metric_data(
    Namespace="MyApp",
    MetricData=[
        {
            "MetricName": "RequestLatency",
            "Value": response_time_ms,
            "Unit": "Milliseconds",
            "Timestamp": datetime.utcnow()
        },
        {
            "MetricName": "Errors",
            "Value": 1,
            "Unit": "Count",
            "Timestamp": datetime.utcnow()
        }
    ]
)
```

### 8. Plan de rollback
Crea `scripts/rollback.sh`:
```bash
#!/bin/bash
# Rollback a versión anterior

OLD_VERSION=${1:-v1.2.2}
NEW_VERSION=${2:-v1.2.3}

echo "Rollback de $NEW_VERSION a $OLD_VERSION"

# 1. Parar instancias nuevas
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name myapp-asg-new \
  --desired-capacity 0

# 2. Esperar a que se terminen
sleep 30

# 3. Volver a versión anterior
aws ec2 create-launch-template-version \
  --launch-template-name myapp \
  --source-version 1  # Versión anterior

# 4. Actualizar ASG
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name myapp-asg \
  --launch-template LaunchTemplateName=myapp,Version='$Latest'

# 5. Incrementar instancias
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name myapp-asg \
  --desired-capacity 3

echo "Rollback completado"
```

### 9. Checklist pre-deploy
Crea `DEPLOY_CHECKLIST.md`:
```markdown
# Pre-Deploy Checklist v1.2.3

## Código y Testing
- [ ] Última versión de main branch
- [ ] Todos los tests en verde (pytest -v)
- [ ] Linting pasado (pylint, black)
- [ ] No hay deuda técnica crítica (SonarQube)
- [ ] Code review aprobado

## Base de Datos
- [ ] Migraciones preparadas (alembic)
- [ ] Plan de rollback de BD definido
- [ ] Backups recientes
- [ ] Cambios de schema compatibles con versión anterior

## Configuración
- [ ] Variables de entorno documentadas
- [ ] Secrets creados en AWS Secrets Manager
- [ ] Certificates válidos
- [ ] DNS propagado

## Infraestructura
- [ ] Capacidad de recursos verificada
- [ ] Límites de rate aumentados si necesario
- [ ] Auto Scaling policies configuradas
- [ ] Load Balancer health checks OK

## Monitoreo
- [ ] CloudWatch alarms configuradas
- [ ] Logs centralizados habilitados
- [ ] Dashboards preparados
- [ ] Sentry/APM funcionando

## Documentación y Soporte
- [ ] Release notes preparadas
- [ ] Runbook de issues comunes
- [ ] Equipo de guardia asignado
- [ ] Canales de comunicación listos (Slack)

## Go/No-Go Decision
- [ ] PM: Go
- [ ] Tech Lead: Go
- [ ] DevOps: Go
- [ ] QA: Go

**Autorizado por**: _______
**Fecha**: _______
**Versión anterior para rollback**: v1.2.2
```

### 10. Debugging en producción
```python
# Configurar logging detallado sin reiniciar
import logging

def enable_debug_logs():
    logging.getLogger("app").setLevel(logging.DEBUG)
    return {"status": "debug mode enabled"}

# Metrics/traces temporales
@app.before_request
def log_request():
    import time
    flask.g.start_time = time.time()

@app.after_request
def log_response(response):
    duration = time.time() - flask.g.start_time
    logger.info(f"{request.path} - {response.status_code} - {duration:.2f}s")
    return response
```

## 🔹 Ejercicio Práctico

### Tarea 1: Preparar código
1. Separar configuración por ambiente
2. Crear Dockerfile optimizado
3. Agregar health y readiness checks

### Tarea 2: Deploy manual
1. Construir imagen Docker
2. Ejecutar en local con vars de producción
3. Probar endpoints críticos

### Tarea 3: Monitoreo
1. Configurar CloudWatch Logs
2. Crear 3 alarmas (CPU, Memory, Error rate)
3. Crear dashboard

### Tarea 4: Plan de rollback
1. Documentar pasos de rollback
2. Testear rollback (dev/staging)
3. Preparar runbook

## 🔹 Verificación de Aprendizaje

- [ ] Entiendo diferencia entre blue-green y canary
- [ ] Puedo crear imagen Docker optimizada
- [ ] He configurado health checks
- [ ] Sé cómo usar CloudWatch para monitoreo
- [ ] Tengo plan de rollback documentado
- [ ] Puedo debuggear problemas en producción

## 🔹 Recursos Complementarios

- [AWS EC2 Best Practices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-best-practices.html)
- [CloudWatch User Guide](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/)
- [Docker best practices](https://docs.docker.com/develop/dev-best-practices/)
- [Kubernetes deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Site Reliability Engineering](https://sre.google/books/)
