# Optimización Avanzada: CloudWatch, API Gateway y Auto Scaling para EC2

Guía completa para mejorar la reestructuración existente con monitoreo inteligente, gestión de tráfico y reducción de costos mediante escalado automático y apagado programado de instancias.

---

## Índice
1. [CloudWatch: Monitoreo Integral](#1-cloudwatch-monitoreo-integral)
2. [API Gateway: Gestión de Tráfico](#2-api-gateway-gestión-de-tráfico)
3. [Auto Scaling: Escalado Automático](#3-auto-scaling-escalado-automático)
4. [Cost Optimization: Apagado/Encendido Programado](#4-cost-optimization-apagadoencendido-programado)
5. [Arquitectura Completa Integrada](#5-arquitectura-completa-integrada)
6. [Casos de Uso Reales](#6-casos-de-uso-reales)

---

## 1. CloudWatch: Monitoreo Integral

### ¿Por qué CloudWatch?

CloudWatch es el servicio nativo de AWS para monitoreo, logging y alertas. Integrado con Auto Scaling permite:
- Monitorear CPU, memoria, disco, red en tiempo real
- Crear dashboards personalizados
- Configurar alarmas automáticas
- Almacenar logs centralizados
- Generar métricas personalizadas desde tu app

---

### 1.1 Instalación del CloudWatch Agent en EC2

#### A. Descargar e instalar el agente

```bash
# En Amazon Linux 2023
wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
sudo rpm -U ./amazon-cloudwatch-agent.rpm

# En Ubuntu/Debian
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i -E ./amazon-cloudwatch-agent.deb
```

#### B. Crear archivo de configuración

`/opt/aws/amazon-cloudwatch-agent/etc/config.json`

```json
{
  "metrics": {
    "namespace": "MiApp",
    "metrics_collected": {
      "cpu": {
        "measurement": [
          {
            "name": "cpu_usage_idle",
            "rename": "CPU_USAGE_IDLE",
            "unit": "Percent"
          },
          {
            "name": "cpu_usage_iowait",
            "rename": "CPU_USAGE_IOWAIT",
            "unit": "Percent"
          },
          "cpu_time_guest"
        ],
        "totalcpu": false,
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": [
          {
            "name": "used_percent",
            "rename": "DISK_USED_PERCENT",
            "unit": "Percent"
          }
        ],
        "metrics_collection_interval": 60,
        "resources": [
          "*"
        ]
      },
      "mem": {
        "measurement": [
          {
            "name": "mem_used_percent",
            "rename": "MEM_USED_PERCENT",
            "unit": "Percent"
          }
        ],
        "metrics_collection_interval": 60
      },
      "netstat": {
        "measurement": [
          {
            "name": "tcp_established",
            "rename": "TCP_ESTABLISHED",
            "unit": "Count"
          }
        ],
        "metrics_collection_interval": 60
      }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/miapp/app.log",
            "log_group_name": "/aws/ec2/miapp/application",
            "log_stream_name": "{instance_id}"
          },
          {
            "file_path": "/var/log/nginx/access.log",
            "log_group_name": "/aws/ec2/miapp/nginx-access",
            "log_stream_name": "{instance_id}"
          },
          {
            "file_path": "/var/log/nginx/error.log",
            "log_group_name": "/aws/ec2/miapp/nginx-error",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
```

#### C. Iniciar el agente

```bash
# Validar configuración
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json
```

---

### 1.2 Enviar Métricas Personalizadas desde tu App

Integra CloudWatch en tu aplicación Python para enviar métricas de negocio:

```python
import boto3
from datetime import datetime

cloudwatch = boto3.client('cloudwatch')

def registrar_metrica_custom(nombre_metrica, valor, unidad='Count'):
    """
    Envia una métrica personalizada a CloudWatch
    
    Ejemplos de uso:
    - registrar_metrica_custom('RequestsProcessados', 150, 'Count')
    - registrar_metrica_custom('TiempoProcesamiento', 2.5, 'Seconds')
    - registrar_metrica_custom('TasaError', 1.2, 'Percent')
    """
    try:
        cloudwatch.put_metric_data(
            Namespace='MiApp/Negocio',
            MetricData=[
                {
                    'MetricName': nombre_metrica,
                    'Value': valor,
                    'Unit': unidad,
                    'Timestamp': datetime.utcnow()
                }
            ]
        )
        print(f"Métrica '{nombre_metrica}' enviada: {valor} {unidad}")
    except Exception as e:
        print(f"Error enviando métrica: {e}")

# Ejemplo en tu app
def procesar_pedido(pedido_id):
    inicio = time.time()
    
    # Tu lógica aquí
    resultado = procesar_pedido_interno(pedido_id)
    
    tiempo_procesamiento = time.time() - inicio
    registrar_metrica_custom('TiempoProcesamiento', tiempo_procesamiento, 'Seconds')
    
    return resultado
```

---

### 1.3 Crear Dashboard en CloudWatch

Accede a **CloudWatch Console → Dashboards → Create Dashboard**

Ejemplo de dashboard útil:
- CPU Utilization (%)
- Memory Usage (%)
- Network In/Out (bytes)
- Disk Usage (%)
- Requests/segundo
- Tiempo de respuesta promedio
- Tasa de errores

---

## 2. API Gateway: Gestión de Tráfico

### ¿Por qué usar API Gateway?

- **Load Balancing global**: distribuir tráfico inteligentemente
- **Rate Limiting**: proteger contra ataques y uso excesivo
- **Throttling**: limitar requests por cliente/IP
- **Caché**: reducir carga en EC2
- **WAF integrado**: protección contra amenazas
- **Transformación de requests**: validar antes de llegar a EC2

---

### 2.1 Crear API Gateway REST API

#### A. Crear la API

1. AWS Console → API Gateway → Create API → REST API
2. Nombre: `miapp-api`
3. Endpoint type: `Regional`

#### B. Crear recurso proxy

```
/
└── {proxy+} (Greedy path parameter)
```

Esto captura todas las rutas y las envía a tu EC2.

#### C. Crear método ANY

- Seleccionar `{proxy+}`
- Acciones → Create Method → ANY
- Integration type: HTTP
- HTTP Method: ANY
- Endpoint URL: `http://tu-dominio-o-ip-ec2/` (sin incluir path, API Gateway lo añade)

#### D. Habilitar throttling

- Acciones → Resource → Throttle Settings
- Rate: 10000 (requests por segundo)
- Burst: 5000

---

### 2.2 Configuración de Caché

En el método ANY:
- Method Execution → Integration Response → 200
- Enable API cache: ✓
- Cache TTL: 300 segundos (5 minutos)

```
Cache Settings:
- Cache cluster size: 0.5 GB
- Encrypt cache data: ✓
```

---

### 2.3 Deployment

1. Actions → Deploy API
2. Stage: `prod` (o crear nuevo)
3. Invoke URL: `https://xxxxx.execute-api.region.amazonaws.com/prod`

**Ahora tus clientes acceden vía API Gateway, no directamente a EC2.**

---

### 2.4 IAM Role para EC2 (opcional pero recomendado)

Si tu app en EC2 necesita comunicarse con API Gateway:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "apigateway:GET",
        "apigateway:POST",
        "apigateway:PUT",
        "apigateway:DELETE"
      ],
      "Resource": "arn:aws:apigateway:region::/restapis/*"
    }
  ]
}
```

---

## 3. Auto Scaling: Escalado Automático

### ¿Por qué Auto Scaling?

- **Escalado horizontal**: agregar más instancias según demanda
- **Reducción de costos**: eliminar instancias cuando no se necesitan
- **Alta disponibilidad**: mantener aplicación online ante fallos
- **Automático**: basado en métricas (CPU, memoria, requests)

---

### 3.1 Crear Launch Template

1. EC2 Console → Launch Templates → Create Launch Template
2. **Nombre**: `miapp-template`
3. **AMI**: selecciona tu AMI customizada (con Docker, NGINX, CloudWatch agent)
4. **Instance type**: t3.medium (ajusta según tu app)
5. **Key pair**: tu par de claves
6. **Network settings**: tu VPC y subnet
7. **Security groups**: permitir tráfico necesario
8. **IAM instance profile**: asignar rol con permisos CloudWatch

**User Data Script** (importante):

```bash
#!/bin/bash
set -e

# Actualizar sistema
sudo dnf update -y

# Iniciar Docker
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user

# Clonar repositorio
cd /opt/apps/miapp
git pull origin main

# Construir imagen Docker
docker build -t miapp:latest .

# Iniciar contenedor con docker-compose
cd /opt/apps/miapp
docker-compose -f docker-compose.prod.yml up -d

# Iniciar CloudWatch Agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json
```

---

### 3.2 Crear Auto Scaling Group

1. EC2 Console → Auto Scaling Groups → Create Auto Scaling Group
2. **Nombre**: `miapp-asg`
3. **Launch Template**: seleccionar `miapp-template`
4. **Desired capacity**: 2 (instancias normales)
5. **Min capacity**: 1 (mínimo en horas bajas)
6. **Max capacity**: 5 (máximo ante alto tráfico)
7. **VPC**: seleccionar VPC y subnets (preferiblemente en AZs diferentes)

---

### 3.3 Crear Políticas de Escalado

#### A. Target Tracking Scaling Policy

Es la más simple y recomendada:

1. Auto Scaling Group → Scaling policies → Create scaling policy
2. **Policy type**: Target tracking scaling policy
3. **Metric type**: CPU Utilization
4. **Target value**: 70% (agregará instancias cuando CPU > 70%)

```
CPU < 70% → quitar instancias gradualmente
CPU > 70% → agregar instancias gradualmente
```

#### B. Política adicional basada en custom metrics

```python
# CLI o boto3
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name miapp-asg \
  --policy-name scale-based-requests \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "TargetValue": 5000,
    "CustomizedMetricSpecification": {
      "MetricName": "RequestsPerSecond",
      "Namespace": "MiApp/Negocio",
      "Statistic": "Average"
    }
  }'
```

---

### 3.4 Validar Auto Scaling

```bash
# Ver instancias del ASG
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names miapp-asg

# Ver actividades de escalado
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name miapp-asg

# Ver capacidad actual
aws ec2 describe-instances \
  --filters "Name=tag:aws:autoscaling:groupName,Values=miapp-asg" \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType]' \
  --output table
```

---

## 4. Cost Optimization: Apagado/Encendido Programado

### ¿Por qué? Ahorra 30-50% de costos

Si tu app tiene:
- **Horario comercial**: solo activa 9-18h
- **Fines de semana**: tráfico bajo o nulo
- **Ambiente staging**: no necesita estar siempre activo

---

### 4.1 EC2 Scheduled Actions (Automático)

Integrado con Auto Scaling, permite apagar/encender por horarios:

```bash
# Apagar a las 18:00 todos los días (reducir a 0 instancias)
aws autoscaling put-scheduled-action \
  --auto-scaling-group-name miapp-asg \
  --scheduled-action-name miapp-scale-down-evening \
  --recurrence "0 18 * * *" \
  --min-size 0 \
  --desired-capacity 0 \
  --max-size 5

# Encender a las 08:00 (volver a capacidad normal)
aws autoscaling put-scheduled-action \
  --auto-scaling-group-name miapp-asg \
  --scheduled-action-name miapp-scale-up-morning \
  --recurrence "0 8 * * MON-FRI" \
  --min-size 1 \
  --desired-capacity 2 \
  --max-size 5
```

---

### 4.2 EventBridge + Lambda (Más Control)

Para apagar/encender instancias específicas con lógica personalizada:

#### A. Crear función Lambda

```python
import boto3
import json
from datetime import datetime

ec2 = boto3.client('ec2')
asg = boto3.client('autoscaling')

def lambda_handler(event, context):
    """
    Apaga/enciende instancias según hora
    """
    action = event.get('action')  # 'stop' o 'start'
    asg_name = 'miapp-asg'
    
    # Obtener instancias del ASG
    response = asg.describe_auto_scaling_groups(
        AutoScalingGroupNames=[asg_name]
    )
    
    if not response['AutoScalingGroups']:
        return {'statusCode': 404, 'body': 'ASG no encontrado'}
    
    instance_ids = [
        i['InstanceId'] 
        for i in response['AutoScalingGroups'][0]['Instances']
    ]
    
    if action == 'stop':
        ec2.stop_instances(InstanceIds=instance_ids)
        print(f"Deteniendo instancias: {instance_ids}")
    elif action == 'start':
        ec2.start_instances(InstanceIds=instance_ids)
        print(f"Iniciando instancias: {instance_ids}")
    
    return {
        'statusCode': 200,
        'body': json.dumps(f'{action} completado: {instance_ids}')
    }
```

#### B. Crear reglas EventBridge

1. EventBridge Console → Rules → Create Rule
2. **Name**: `miapp-stop-evening`
3. **Pattern**: Schedule → Cron: `0 18 ? * MON-FRI *` (18:00 L-V)
4. **Target**: Lambda function → `miapp-scheduler`
5. **Input**: JSON constante:
   ```json
   {"action": "stop"}
   ```

6. Repetir para encender: Cron: `0 8 ? * MON-FRI *`

---

### 4.3 Instance Scheduler (Recomendado para múltiples instancias)

AWS ofrece una solución native:

1. AWS Console → Systems Manager → Parameter Store
2. Crear parámetro: `miapp/instance-scheduler-config`
3. Usar servicio AWS Instance Scheduler (versión nativa)

---

## 5. Arquitectura Completa Integrada

```
┌─────────────────────────────────────────────────────────┐
│                   Internet Users                        │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
        ┌────────────────────────┐
        │    CloudFront (CDN)    │ ← Cachea contenido estático
        │  (opcional, más globales)
        └────────────┬───────────┘
                     │
                     ▼
        ┌────────────────────────────┐
        │     API Gateway            │
        │  • Rate Limiting: 10k/s    │
        │  • Cache: 300s TTL         │
        │  • WAF Integrado           │
        │  • Transformación requests │
        └────────────┬───────────────┘
                     │
                     ▼
        ┌──────────────────────────────┐
        │  Application Load Balancer   │ ← Distribuye tráfico
        │  (ALB) en subnets públicas   │
        └────────────┬─────────────────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
         ▼           ▼           ▼
    ┌────────┐  ┌────────┐  ┌────────┐
    │ EC2-1  │  │ EC2-2  │  │ EC2-3* │ ← Auto Scaling Group
    │ Docker │  │ Docker │  │ Docker │   (2-5 instancias)
    │ App    │  │ App    │  │ App    │
    │ NGINX  │  │ NGINX  │  │ NGINX  │
    └────────┘  └────────┘  └────────┘
         │           │           │
         └───────────┼───────────┘
                     │
    ┌────────────────┴─────────────────┐
    │                                  │
    ▼                                  ▼
┌─────────────────┐         ┌──────────────────┐
│  CloudWatch     │         │  Auto Scaling    │
│  • Métricas     │         │  • CPU > 70%     │
│  • Logs         │────────▶│    → +1 instancia│
│  • Dashboards   │         │  • CPU < 30%     │
│  • Alarmas      │         │    → -1 instancia│
└─────────────────┘         └──────────────────┘
    │
    ▼
┌─────────────────┐
│  EventBridge    │
│  + Lambda       │
│  • 18:00: STOP  │
│  • 08:00: START │
└─────────────────┘
```

---

## 6. Casos de Uso Reales

### Caso 1: E-commerce con tráfico variable

**Configuración óptima:**
- **Horario comercial (9-21h)**: Min=2, Desired=3, Max=8
- **Horas bajas (21-9h)**: Min=1, Desired=1, Max=3
- **Fines de semana**: Min=1, Desired=2, Max=5
- **Black Friday**: Min=5, Desired=10, Max=20

**Ahorros**: ~40% con apagado nocturno

---

### Caso 2: API interno con picos predecibles

**Lunes-Viernes 08:00**: Spike de requests (reportes diarios)
- Configurar Target Tracking en 70% CPU
- EventBridge dispara Lambda para pre-scale antes del pico
- A las 12:00 vuelve a normalidad

**Script para pre-scale:**
```bash
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name miapp-asg \
  --desired-capacity 5 \
  --no-honor-cooldown  # Ignora período de espera
```

---

### Caso 3: Staging + Producción con costo mínimo

**Staging**:
- Instancia única t2.micro
- Apaga 18-9h (solo laboral)
- Sin Auto Scaling

**Producción**:
- ASG: Min=2, Max=10
- Auto Scaling por CPU
- Apaga 50% de instancias 18-9h

---

## 7. Monitoreo y Alertas Prácticas

### 7.1 Crear Alarma de CPU

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name miapp-cpu-high \
  --alarm-description "CPU alta en miapp" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:region:account:topic-email
```

---

### 7.2 Crear Alarma de Memoria

Requiere CloudWatch Agent (ya instalado):

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name miapp-memory-high \
  --alarm-description "Memoria alta en miapp" \
  --metric-name MEM_USED_PERCENT \
  --namespace MiApp \
  --statistic Average \
  --period 300 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:region:account:topic-email
```

---

### 7.3 Crear Alarma de Aplicación Personalizada

Si tu app no responde en 5 minutos:

```python
# En tu app
import requests
import time

def healthcheck_lambda():
    """
    Función para verificar salud de la app cada 5 min
    (Ejecutar vía CloudWatch Events)
    """
    try:
        response = requests.get('http://localhost:8000/health', timeout=5)
        
        if response.status_code == 200:
            cloudwatch.put_metric_data(
                Namespace='MiApp/Health',
                MetricData=[{
                    'MetricName': 'AppHealthy',
                    'Value': 1,
                    'Unit': 'None'
                }]
            )
        else:
            cloudwatch.put_metric_data(
                Namespace='MiApp/Health',
                MetricData=[{
                    'MetricName': 'AppHealthy',
                    'Value': 0,
                    'Unit': 'None'
                }]
            )
    except Exception as e:
        print(f"Health check failed: {e}")
```

---

## 8. Implementación Step-by-Step

### Fase 1: Monitoreo (Semana 1)
-  Instalar CloudWatch Agent
-  Configurar métricas básicas (CPU, memoria, disco)
-  Crear dashboard
-  Configurar alarmas

### Fase 2: API Gateway (Semana 2)
-  Crear REST API
-  Configurar proxy
-  Habilitar caché
-  Deployer en prod

### Fase 3: Auto Scaling (Semana 3)
-  Crear Launch Template
-  Crear Auto Scaling Group
-  Configurar Target Tracking Policy
-  Pruebas de carga

### Fase 4: Cost Optimization (Semana 4)
-  Configurar Scheduled Actions
-  Crear EventBridge + Lambda
-  Validar ahorro de costos
-  Monitoreo final

---

## 9. Costos Estimados

### Ejemplo: 2 instancias t3.medium (prod) + 1 t2.micro (staging)

**Sin optimización**:
- 3 instancias × 24h × 30 días = ~$90/mes

**Con Auto Scaling + Apagado nocturno**:
- Prod: 2-4 instancias × 15h/día (10-70% del tiempo en pico)
- Staging: apaga 18-9h
- **Estimado: ~$45-50/mes (50% ahorro)**

**Con CloudFront + API Gateway caching**:
- Reduce tráfico a instancias 40-60%
- Menor consumo CPU, faster de-scaling
- **Ahorro adicional: 10-15%**

---

## 10. Troubleshooting

| Problema | Causa | Solución |
|----------|-------|----------|
| Auto Scaling no funciona | IAM permissions insuficientes | Verificar rol EC2 con permisos `autoscaling:*` |
| CloudWatch sin datos | Agent no corriendo | `sudo systemctl status amazon-cloudwatch-agent` |
| API Gateway lento | Cache deshabilitado | Habilitar cache TTL > 300s |
| Instancias no se apagan | Scheduled action mal configurada | Revisar timezone y recurrence en UTC |
| Picos de tráfico sin escalar | Target value muy alto | Reducir de 80% a 70% CPU |

---

## Conclusión

Esta configuración te proporciona:
-  **Visibilidad**: Monitoreo completo con CloudWatch
-  **Escalabilidad**: Auto Scaling basado en demanda
-  **Confiabilidad**: Multi-AZ y health checks
-  **Costo-eficiencia**: Apagado programado y escalado inteligente
-  **Protección**: API Gateway con WAF y rate limiting

**Resultado final**: Infraestructura profesional, resiliente y económica.
