# Día 7 – Debugging y Troubleshooting en Producción

## 🔹 Introducción

El debugging en producción es diferente a local. No puedes pausar la ejecución con breakpoints mientras usuarios esperan.

**Objetivo**: Aprender técnicas de debugging que funcionan en EC2 y Lambda sin interrumpir servicio.

---

## 🔹 Teoría: Técnicas de Debugging

### Diferencias: Local vs Producción

| Aspecto | Local | Producción |
|---------|-------|-----------|
| **Breakpoints** | ✅ Sí, puedo pausar | ❌ No, usuarios esperan |
| **Restart** | ✅ Rápido | ❌ Afecta servicio |
| **Acceso a código** | ✅ Editor abierto | ❌ Servidor remoto |
| **Logs** | ✅ Stdout directo | ❌ CloudWatch remoto |
| **Datos** | ✅ BD local | ❌ BD en RDS |
| **Performance** | 🚀 Rápido | 🐌 Red latency |

### Estrategias

1. **Logging exhaustivo** (antes de que falle)
2. **Métricas** (detectar anomalías)
3. **Tracing distribuido** (seguir requests)
4. **Queries a BD** (sin código)
5. **SSH a instancia** (último recurso)
6. **Replay de requests** (reproducir problema)

---

## 🔹 Técnica 1: Logging Estratégico

### Niveles de logging

```python
import logging

logger = logging.getLogger(__name__)

# CRITICAL: Sistema no funciona
logger.critical("Base de datos desconectada")

# ERROR: Operación falló
logger.error("Error procesando orden 123")

# WARNING: Comportamiento inesperado pero funciona
logger.warning("Timeout en API externa, reintentando")

# INFO: Eventos importantes
logger.info("Usuario creado exitosamente")

# DEBUG: Información detallada para debugging
logger.debug(f"Query SQL: SELECT * FROM users WHERE id={user_id}")
```

### Logging con contexto

```python
import uuid
from contextvars import ContextVar

# Variable de contexto para rastrear request
request_id: ContextVar[str] = ContextVar('request_id', default='unknown')

@app.middleware("http")
async def add_request_id(request, call_next):
    """Agregar ID único a cada request"""
    req_id = str(uuid.uuid4())
    request_id.set(req_id)
    
    logger.info(f"[{req_id}] {request.method} {request.url.path}")
    response = await call_next(request)
    logger.info(f"[{req_id}] Status: {response.status_code}")
    
    return response

# En cualquier lugar del código
logger.info(f"[{request_id.get()}] Procesando orden")
```

### Structured logging (JSON)

```python
import json
import logging

class JSONFormatter(logging.Formatter):
    """Logs en formato JSON para CloudWatch"""
    
    def format(self, record):
        log_data = {
            'timestamp': self.formatTime(record),
            'level': record.levelname,
            'logger': record.name,
            'function': record.funcName,
            'line': record.lineno,
            'message': record.getMessage(),
            'request_id': request_id.get(),
        }
        
        # Agregar excepción si aplica
        if record.exc_info:
            log_data['exception'] = self.formatException(record.exc_info)
        
        # Agregar datos custom si existen
        if hasattr(record, 'user_id'):
            log_data['user_id'] = record.user_id
        if hasattr(record, 'order_id'):
            log_data['order_id'] = record.order_id
        
        return json.dumps(log_data)

# Uso
logger.info("Usuario creado", extra={
    'user_id': 123,
    'email': 'user@example.com'
})
```

---

## 🔹 Técnica 2: Debugging en EC2

### 2.1 SSH a instancia

```bash
# Obtener IP pública
INSTANCE_IP=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ecommerce-prod" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

# Conectar
ssh -i ~/.ssh/ec2-key.pem ec2-user@$INSTANCE_IP

# Ver logs en tiempo real
tail -f /var/log/myapp/app.log
grep "ERROR" /var/log/myapp/app.log | tail -20

# Ver procesos
ps aux | grep python
docker ps -a

# Ver recursos
free -h           # Memoria
df -h             # Disco
top -b -n 1       # CPU
```

### 2.2 Ejecutar queries directamente

```bash
# Conectar a base de datos desde EC2
psql -h mydb.region.rds.amazonaws.com \
     -U admin \
     -d myapp \
     -c "SELECT COUNT(*) FROM orders WHERE status='pending';"

# Ver órdenes problemáticas
SELECT id, user_id, total, status, created_at, updated_at
FROM orders
WHERE created_at > NOW() - INTERVAL '1 hour'
AND status = 'pending'
ORDER BY created_at DESC;
```

### 2.3 Inspeccionar proceso de Python

```bash
# Ver proceso Python
ps aux | grep python

# PID = 1234
# Adjuntar debugger (gdb)
gdb python 1234

# Ver stack trace en vivo
(gdb) py-bt

# Listar threads
(gdb) info threads

# Continuoar ejecución
(gdb) continue
(gdb) quit
```

### 2.4 Crear puerto de debugging

```bash
# Dentro del contenedor Docker
docker exec -it myapp /bin/bash

# Instalar pdb (Python debugger)
pip install ipdb

# Ver logs
python -m pdb /app/main.py

# (Pdb) help         # Ver comandos
# (Pdb) b 45         # Breakpoint en línea 45
# (Pdb) c            # Continuar
# (Pdb) n            # Siguiente línea
# (Pdb) p variable   # Imprimir variable
```

### 2.5 Monitoreo en tiempo real

```bash
# Ver requests en tiempo real
tail -f /var/log/myapp/app.log | grep "POST /api/orders"

# Contar errores por hora
grep "ERROR" /var/log/myapp/app.log | \
  cut -d' ' -f1 | \
  uniq -c | \
  sort -rn

# Ver errores específicos
grep "Connection refused" /var/log/myapp/app.log

# Ver patrón de latencia
grep "Duration:" /var/log/myapp/app.log | \
  awk '{print $NF}' | \
  sort -n | \
  tail -10  # Top 10 requests más lentos
```

---

## 🔹 Técnica 3: Debugging en Lambda

### 3.1 CloudWatch Logs Insights

```sql
-- Query: Encontrar requests lentos
fields @timestamp, @duration, user_id, order_id
| filter @duration > 1000  -- Más de 1 segundo
| stats avg(@duration), max(@duration) by user_id

-- Query: Contar errores por tipo
fields error_type
| filter ispresent(error_type)
| stats count() by error_type

-- Query: Rastrear request específico
fields @message, @timestamp
| filter request_id = "550e8400-e29b-41d4-a716-446655440000"
| sort @timestamp asc

-- Query: Errores de base de datos
fields @message, @timestamp, @duration
| filter @message like /Connection refused/
| stats count(), avg(@duration) as avg_time
```

### 3.2 X-Ray para tracing distribuido

```python
# lambda_function.py
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

patch_all()  # Automaticamente instrumenta boto3, requests, etc

@xray_recorder.capture('create_order')
def create_order(event, context):
    """Toda esta función será trackeada en X-Ray"""
    
    order_id = create_order_in_db()
    
    process_payment(order_id)  # Subsegmento automático
    
    send_notification(order_id)
    
    return order_id

@xray_recorder.capture('process_payment')
def process_payment(order_id):
    # Lógica de pago
    pass

# Ver traces en AWS Console:
# X-Ray → Service Map (visualiza arquitectura)
# X-Ray → Traces (lista de requests)
# Haz click en un trace para ver timeline detallada
```

### 3.3 Error Tracking con Dead Letter Queue

```python
import json
import boto3
import logging

sqs = boto3.client('sqs')
logger = logging.getLogger()

def lambda_handler(event, context):
    try:
        # Procesar
        result = process_order(event)
        return result
    
    except ValueError as e:
        # Error de validación: no reintentar
        logger.warning(f"Validación fallida: {e}")
        send_to_dlq(event, "VALIDATION_ERROR", str(e))
        raise
    
    except Exception as e:
        # Error desconocido: reintentará automáticamente
        logger.exception(f"Error: {e}")
        raise

def send_to_dlq(event, error_type, error_msg):
    """Enviar a DLQ para análisis manual"""
    dlq_url = os.environ['DLQ_URL']
    
    sqs.send_message(
        QueueUrl=dlq_url,
        MessageBody=json.dumps({
            'error_type': error_type,
            'error_message': error_msg,
            'original_event': event,
            'timestamp': datetime.utcnow().isoformat()
        })
    )

# Monitorear DLQ
aws sqs receive-message --queue-url $DLQ_URL --max-number-of-messages 10
```

### 3.4 Custom Metrics para debugging

```python
import boto3
from datetime import datetime

cloudwatch = boto3.client('cloudwatch')

def lambda_handler(event, context):
    start_time = time.time()
    
    try:
        result = process_order(event)
        
        # Métrica de éxito
        duration = time.time() - start_time
        cloudwatch.put_metric_data(
            Namespace='MyApp',
            MetricData=[
                {
                    'MetricName': 'OrderProcessingTime',
                    'Value': duration * 1000,  # ms
                    'Unit': 'Milliseconds',
                    'Timestamp': datetime.utcnow(),
                    'Dimensions': [
                        {'Name': 'FunctionName', 'Value': context.function_name}
                    ]
                },
                {
                    'MetricName': 'OrdersProcessed',
                    'Value': 1,
                    'Unit': 'Count'
                }
            ]
        )
        
        return result
    
    except Exception as e:
        # Métrica de error
        cloudwatch.put_metric_data(
            Namespace='MyApp',
            MetricData=[
                {
                    'MetricName': 'OrderProcessingErrors',
                    'Value': 1,
                    'Unit': 'Count',
                    'Dimensions': [
                        {'Name': 'ErrorType', 'Value': type(e).__name__}
                    ]
                }
            ]
        )
        raise
```

---

## 🔹 Casos Reales de Debugging

### Caso 1: Timeout en RDS

**Síntoma**: Requests tardan 30+ segundos, luego fallan

```python
# Investigación
logger.debug(f"Conectando a RDS en {DB_HOST}")
start = time.time()
conn = engine.connect()
logger.info(f"Conectado en {time.time() - start:.2f}s")

# Soluciones
# 1. RDS en misma VPC que EC2/Lambda
# 2. RDS Proxy para pooling
# 3. Aumentar timeout: pool_pre_ping=True
# 4. Verificar security groups permiten puerto 5432

# Query a RDS
SELECT datname, usename, count(*) as connections
FROM pg_stat_activity
GROUP BY datname, usename;
```

### Caso 2: Memory leak en Lambda

**Síntoma**: Lambda se vuelve más lento con cada invocación

```python
import tracemalloc
import logging

logger = logging.getLogger()
tracemalloc.start()

def lambda_handler(event, context):
    # Tomar snapshot
    current, peak = tracemalloc.get_traced_memory()
    logger.info(f"Memoria: {current / 1024 / 1024:.1f}MB, Peak: {peak / 1024 / 1024:.1f}MB")
    
    # Procesar
    result = process_order(event)
    
    # Verificar crecimiento
    current_after, peak_after = tracemalloc.get_traced_memory()
    growth = (current_after - current) / 1024 / 1024
    logger.warning(f"Crecimiento de memoria: {growth:.1f}MB")
    
    # Si hay leak, identificar dónde
    snapshot = tracemalloc.take_snapshot()
    top_stats = snapshot.statistics('lineno')
    for stat in top_stats[:3]:
        logger.info(stat)
    
    return result

# Solución: Asegurarse de cerrar conexiones, limpiar variables globales
```

### Caso 3: Error aleatorio (race condition)

**Síntoma**: Funciona 99% de las veces, falla intermitentemente

```python
# Problema: Dos requests actualizando mismo recurso simultáneamente

# ANTES (incorrecto)
product = db.query(Product).filter(Product.id == 1).first()
product.stock -= quantity  # Race condition aquí
db.commit()

# DESPUÉS (correcto - usar transacción)
from sqlalchemy import select, update

def decrease_stock_safe(product_id: int, quantity: int):
    """Usar UPDATE directo (atomic)"""
    db.execute(
        update(Product)
        .where(Product.id == product_id)
        .values(stock=Product.stock - quantity)
    )
    db.commit()

# O usar row-level locking
db.query(Product)\
    .filter(Product.id == 1)\
    .with_for_update()  # Lock hasta fin de transacción
```

### Caso 4: API Externa lenta

**Síntoma**: El 50% de requests a `/api/orders` tardan >5 segundos

```python
# Agregar timing logging
import time
from app.utils.http_client import HTTPClient

class HTTPClient:
    def request(self, method, url, **kwargs):
        start = time.time()
        logger.debug(f"Iniciando {method} {url}")
        
        try:
            response = self.session.request(method, url, **kwargs)
            duration = time.time() - start
            
            if duration > 1:
                logger.warning(f"Solicitud lenta: {method} {url} tomó {duration:.2f}s")
            
            return response
        
        except requests.Timeout:
            duration = time.time() - start
            logger.error(f"Timeout: {method} {url} después de {duration:.2f}s")
            raise

# Monitoreo en CloudWatch
tail -f /var/log/myapp/app.log | grep "Solicitud lenta"
```

---

## 🔹 Herramientas Útiles

### 1. AWS CloudWatch Insights

```bash
# Ver en web console o CLI
aws logs start-query \
  --log-group-name /aws/lambda/myapp \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string "fields @timestamp, @duration | stats avg(@duration) by bin(5m)"

# Esperar resultado
aws logs get-query-results --query-id <QUERY_ID>
```

### 2. AWS X-Ray

```bash
# Ver traces
aws xray get-trace-summaries \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s)

# Detalles de un trace
aws xray get-trace-document --trace-id <TRACE_ID>
```

### 3. Logs en local

```bash
# Descargar logs de CloudWatch
aws logs get-log-events \
  --log-group-name /aws/ec2/myapp \
  --log-stream-name i-123456 \
  --start-time $(date -d '1 day ago' +%s)000 \
  > logs.json

# Procesar logs
cat logs.json | jq '.events[] | select(.message | contains("ERROR"))'
```

### 4. Profiling

```python
# Ver qué funciones consumen más CPU
from cProfile import Profile
from pstats import Stats
import io

profiler = Profile()

def lambda_handler(event, context):
    profiler.enable()
    
    result = process_order(event)
    
    profiler.disable()
    
    # Imprimir stats
    s = io.StringIO()
    ps = Stats(profiler, stream=s).sort_stats('cumulative')
    ps.print_stats(10)
    logger.info(s.getvalue())
    
    return result
```

---

## 🔹 Checklist de Debugging

```
[ ] Revisar logs en CloudWatch
[ ] Buscar error message específico
[ ] Ver timestamp (¿cuándo comenzó?)
[ ] Correlacionar con cambios (¿qué se desplegó?)
[ ] Verificar métricas (CPU, memoria, conexiones)
[ ] Revisar dependencias externas (RDS, APIs)
[ ] Reproducir en staging
[ ] Hacer rollback si es crítico
[ ] Investigación post-mortem
[ ] Agregar logging para prevenir futuros errores
```

---

## 🔹 Ejercicio Práctico

### Tarea 1: Simular error y debuggear

```python
# app/routes/orders.py - Introducir bug
@orders_bp.post("/")
async def create_order(order_data: OrderCreate):
    order = service.create_order(order_data)
    # BUG: olvidé guardar en BD
    # return order  # ERROR: order es None
```

**Debugging**:
1. Ver logs → "TypeError: 'NoneType' object is not subscriptable"
2. Ver línea exacta en stacktrace
3. Identificar que no se guardó
4. Arreglar: `db.session.add(order)` y `db.session.commit()`

### Tarea 2: Investigar timeout

Crear script que simule timeout:

```python
# test_timeout.py
import time
import requests

# Simular endpoint lento
@app.get("/slow")
async def slow_endpoint():
    time.sleep(10)
    return {"status": "ok"}

# Debugging
start = time.time()
try:
    response = requests.get("http://localhost:8000/slow", timeout=5)
except requests.Timeout:
    print(f"Timeout después de {time.time() - start:.2f}s")
```

### Tarea 3: Memory profiling

```python
# Identificar memory leak
for i in range(1000):
    data = requests.get("http://api.example.com/data").json()
    # BUG: no limpiar memoria después de cada iteración
```

---

## 🔹 Verificación de Aprendizaje

- [ ] Sé usar CloudWatch Logs Insights
- [ ] Puedo conectar SSH a EC2 para debugging
- [ ] Entiendo X-Ray para tracing distribuido
- [ ] Puedo identificar memory leaks
- [ ] Sé debuggear race conditions
- [ ] Puedo reproducir errores en staging

## 🔹 Recursos Complementarios

- [AWS CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html)
- [AWS X-Ray Developer Guide](https://docs.aws.amazon.com/xray/latest/devguide/)
- [Python Debugging Tools](https://docs.python.org/3/library/debug.html)
- [Postmortem Culture](https://www.blameless.io/)
- [Observability Engineering](https://www.oreilly.com/library/view/observability-engineering/9781492076438/)
