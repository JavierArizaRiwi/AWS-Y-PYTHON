# Día 2 (28/01/2026) – Python - Manejo de Errores y Logs

## 🔹 Teoría

### ¿Por qué es crítico el manejo de errores?
En producción, los errores son inevitables:
- **Conexiones de red perdidas**: Timeout al conectar a RDS
- **Datos inválidos**: Usuario envía JSON malformado
- **Recursos agotados**: S3 rechaza carga de archivo
- **Errores de credenciales**: IAM role sin permisos
- **Condiciones de carrera**: Dos procesos modifican el mismo recurso

Sin manejo de errores:
- La aplicación se cuelga
- El usuario no sabe qué salió mal
- No hay forma de debuggear

Con manejo de errores:
-  Se registra qué pasó (logs)
-  Se comunica al usuario de forma clara
-  Se puede debuggear y recuperar

### Tipos de excepciones en Python

#### Built-in Exceptions
```python
TypeError          # Tipo de dato incorrecto
ValueError         # Valor incorrecto para el tipo
KeyError           # Clave no existe en diccionario
IndexError         # Índice fuera de rango
FileNotFoundError  # Archivo no existe
ConnectionError    # Falla de conexión
TimeoutError       # Operación expiró
```

#### Custom Exceptions
```python
class InvalidUserError(Exception):
    """Excepción personalizada"""
    pass

class DatabaseConnectionError(Exception):
    """Error al conectar a BD"""
    pass
```

### Logging vs Print
```python
# ❌ MAL: usar print
print("Usuario creado")

#  BIEN: usar logging
import logging
logger = logging.getLogger(__name__)
logger.info("Usuario creado con ID: 123")
```

**Ventajas de logging**:
- Niveles: DEBUG, INFO, WARNING, ERROR, CRITICAL
- Timestamps automáticos
- Fácil redirigir a archivos o CloudWatch
- Se puede filtrar por módulo

### Niveles de logging
```
DEBUG     (10) - Información detallada para diagnóstico
INFO      (20) - Confirmación que funciona
WARNING   (30) - Algo inusual pero funciona
ERROR     (40) - Error serio pero controlado
CRITICAL  (50) - Error muy grave, falla total
```

## 🔹 Conceptos Clave

### Ciclo try-except-finally
```python
try:
    # Código que puede fallar
    response = requests.get(url, timeout=5)
    response.raise_for_status()  # Lanza excepción si HTTP error
except requests.Timeout:
    # Manejo específico de timeout
    logger.error("Timeout conectando a API externa")
    return {"error": "API no respondió"}, 504
except requests.HTTPError as e:
    # Manejo específico de errores HTTP
    logger.error(f"Error HTTP: {e.response.status_code}")
    return {"error": "Error en API externa"}, 502
except Exception as e:
    # Captura cualquier otra excepción
    logger.exception("Error inesperado")  # Incluye stack trace
    return {"error": "Error interno"}, 500
finally:
    # Se ejecuta siempre, incluso si hay excepción
    logger.debug("Fin de operación")
```

### Contextos de seguridad (context managers)
```python
from contextlib import contextmanager

@contextmanager
def database_transaction(db):
    try:
        yield db
        db.commit()
        logger.info("Transacción completada")
    except Exception as e:
        db.rollback()
        logger.error(f"Transacción fallida: {e}")
        raise

# Uso
with database_transaction(db) as session:
    user = User(name="Juan")
    session.add(user)
    # Si hay error, se hace rollback automático
```

### Stack trace y debugging
```python
import traceback

try:
    result = 10 / 0
except ZeroDivisionError:
    # Opción 1: Logging con stack trace
    logger.exception("División por cero")
    
    # Opción 2: Capturar stack trace manualmente
    tb = traceback.format_exc()
    logger.error(f"Error completo:\n{tb}")
```

### Logging estructurado
```python
import json
import logging

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "module": record.module,
            "function": record.funcName,
            "message": record.getMessage(),
            "line": record.lineno
        }
        if record.exc_info:
            log_data["exception"] = self.formatException(record.exc_info)
        return json.dumps(log_data)
```

## 🔹 Ejemplos Reales: EC2 vs Lambda

### Logging en EC2
```python
# app/utils/logger.py - EC2 version
import logging
import logging.handlers
import os
from datetime import datetime

def setup_logger_ec2(name, level=logging.INFO):
    """Logger para EC2 (escribe a archivo y stdout)"""
    
    logger = logging.getLogger(name)
    logger.setLevel(level)
    
    # Archivo de logs
    os.makedirs("/var/log/myapp", exist_ok=True)
    log_file = f"/var/log/myapp/{datetime.now().strftime('%Y-%m-%d')}.log"
    
    # Handler para archivo (CloudWatch Agent lo leerá)
    file_handler = logging.FileHandler(log_file)
    file_handler.setLevel(logging.DEBUG)
    
    # Handler para stdout (contenedor Docker)
    console_handler = logging.StreamHandler()
    console_handler.setLevel(logging.INFO)
    
    # Formato estructurado para CloudWatch
    formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    )
    file_handler.setFormatter(formatter)
    console_handler.setFormatter(formatter)
    
    logger.addHandler(file_handler)
    logger.addHandler(console_handler)
    
    return logger

# Uso en EC2
logger = setup_logger_ec2(__name__)
logger.info("Aplicación iniciada en EC2")
```

### Logging en Lambda
```python
# Logging para Lambda (automático a CloudWatch)
import logging
import json

logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Handler personalizado para JSON
class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_record = {
            'timestamp': self.formatTime(record),
            'level': record.levelname,
            'function': record.funcName,
            'message': record.getMessage(),
            'request_id': record.__dict__.get('request_id', 'unknown')
        }
        return json.dumps(log_record)

# Aplicar a todos los handlers
for handler in logger.handlers:
    handler.setFormatter(JSONFormatter())

def lambda_handler(event, context):
    """
    context.aws_request_id = identificador único
    context.invoked_function_arn = ARN de la función
    """
    request_id = context.aws_request_id
    logger.info(f"Request ID: {request_id}")
    
    try:
        logger.info(f"Evento: {json.dumps(event)}")
        result = process_event(event)
        logger.info(f"Resultado: {result}")
        return result
    except Exception as e:
        logger.exception(f"Error en {request_id}")
        raise
```

**CloudWatch Logs output**:
```json
{
  "timestamp": "2026-01-27 10:30:45",
  "level": "INFO",
  "function": "lambda_handler",
  "message": "Request ID: 550e8400-e29b-41d4-a716-446655440000",
  "request_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

## 🔹 Práctica Paso a Paso

### 1. Configurar logging centralizado
Crea `app/utils/logger.py`:
```python
import logging
import logging.handlers
import os
from datetime import datetime

def setup_logger(name, level=logging.INFO):
    """Configura logger con archivo y consola"""
    
    logger = logging.getLogger(name)
    logger.setLevel(level)
    
    # Crear directorio de logs si no existe
    os.makedirs("logs", exist_ok=True)
    
    # Handler para archivo
    log_file = f"logs/{datetime.now().strftime('%Y-%m-%d')}.log"
    file_handler = logging.FileHandler(log_file)
    file_handler.setLevel(logging.DEBUG)
    
    # Handler para consola
    console_handler = logging.StreamHandler()
    console_handler.setLevel(logging.INFO)
    
    # Formato
    formatter = logging.Formatter(
        "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
    )
    file_handler.setFormatter(formatter)
    console_handler.setFormatter(formatter)
    
    logger.addHandler(file_handler)
    logger.addHandler(console_handler)
    
    return logger
```

**Uso**:
```python
from app.utils.logger import setup_logger

logger = setup_logger(__name__)
logger.info("Aplicación iniciada")
```

### 2. Crear excepciones personalizadas
Crea `app/exceptions.py`:
```python
class ApplicationError(Exception):
    """Excepción base de la aplicación"""
    def __init__(self, message, code=500):
        self.message = message
        self.code = code
        super().__init__(self.message)

class ValidationError(ApplicationError):
    """Error de validación de datos"""
    def __init__(self, message):
        super().__init__(message, 400)

class DatabaseError(ApplicationError):
    """Error de base de datos"""
    def __init__(self, message):
        super().__init__(message, 500)

class ExternalServiceError(ApplicationError):
    """Error al llamar servicio externo"""
    def __init__(self, message):
        super().__init__(message, 502)

class NotFoundError(ApplicationError):
    """Recurso no encontrado"""
    def __init__(self, message):
        super().__init__(message, 404)
```

### 3. Implementar manejo de errores en servicios
Crea `app/services/user_service.py`:
```python
from app.utils.logger import setup_logger
from app.exceptions import ValidationError, DatabaseError, NotFoundError
from app.models import User

logger = setup_logger(__name__)

class UserService:
    def create_user(self, data):
        """Crea usuario con validación y manejo de errores"""
        try:
            # Validación
            if not data.get("email"):
                raise ValidationError("Email es requerido")
            
            if len(data.get("password", "")) < 8:
                raise ValidationError("Contraseña debe tener al menos 8 caracteres")
            
            # Crear usuario
            user = User(
                name=data.get("name"),
                email=data.get("email"),
                password=data.get("password")
            )
            
            # Guardar
            db.session.add(user)
            db.session.commit()
            
            logger.info(f"Usuario creado: {user.id} ({user.email})")
            return user
            
        except ValidationError as e:
            logger.warning(f"Error de validación: {e.message}")
            raise
        except Exception as e:
            logger.error(f"Error creando usuario: {str(e)}", exc_info=True)
            db.session.rollback()
            raise DatabaseError(f"Error al crear usuario: {str(e)}")

    def get_user(self, user_id):
        """Obtiene usuario por ID"""
        try:
            user = User.query.get(user_id)
            
            if not user:
                raise NotFoundError(f"Usuario {user_id} no encontrado")
            
            logger.debug(f"Usuario recuperado: {user_id}")
            return user
            
        except NotFoundError as e:
            logger.warning(f"Usuario no encontrado: {user_id}")
            raise
        except Exception as e:
            logger.error(f"Error obteniendo usuario {user_id}: {str(e)}", exc_info=True)
            raise DatabaseError(f"Error al obtener usuario")
```

### 4. Manejar errores en rutas (endpoints)
Crea `app/routes/users.py`:
```python
from flask import Blueprint, request, jsonify
from app.services.user_service import UserService
from app.exceptions import ApplicationError, ValidationError
from app.utils.logger import setup_logger

users_bp = Blueprint("users", __name__, url_prefix="/api/users")
logger = setup_logger(__name__)
user_service = UserService()

@users_bp.route("/", methods=["POST"])
def create_user():
    try:
        data = request.get_json()
        
        if not data:
            return {"error": "Body vacío"}, 400
        
        user = user_service.create_user(data)
        logger.info(f"Usuario creado vía API: {user.id}")
        return {"id": user.id, "email": user.email}, 201
        
    except ValidationError as e:
        logger.warning(f"Validación fallida: {e.message}")
        return {"error": e.message}, e.code
    except ApplicationError as e:
        logger.error(f"Error de aplicación: {e.message}")
        return {"error": e.message}, e.code
    except Exception as e:
        logger.exception("Error inesperado en POST /users")
        return {"error": "Error interno del servidor"}, 500

@users_bp.route("/<int:user_id>", methods=["GET"])
def get_user(user_id):
    try:
        user = user_service.get_user(user_id)
        return {"id": user.id, "name": user.name, "email": user.email}, 200
        
    except NotFoundError as e:
        return {"error": e.message}, e.code
    except ApplicationError as e:
        return {"error": e.message}, e.code
    except Exception as e:
        logger.exception(f"Error obteniendo usuario {user_id}")
        return {"error": "Error interno del servidor"}, 500
```

### 5. Manejar errores de conexión a BD
```python
from sqlalchemy.exc import SQLAlchemyError
from app.exceptions import DatabaseError

def get_db():
    """Context manager para conexión a BD"""
    db = SessionLocal()
    try:
        yield db
    except SQLAlchemyError as e:
        db.rollback()
        logger.error(f"Error de BD: {str(e)}", exc_info=True)
        raise DatabaseError("Error en la base de datos")
    finally:
        db.close()
```

### 6. Manejo de errores en llamadas a APIs externas
```python
import requests
from app.exceptions import ExternalServiceError

def call_external_api(endpoint, timeout=5):
    """Llama API externa con reintentos"""
    max_retries = 3
    retry_count = 0
    
    while retry_count < max_retries:
        try:
            logger.debug(f"Intento {retry_count + 1} para {endpoint}")
            
            response = requests.get(
                endpoint,
                timeout=timeout
            )
            response.raise_for_status()
            
            logger.info(f"API llamada exitosa: {endpoint}")
            return response.json()
            
        except requests.Timeout:
            retry_count += 1
            if retry_count >= max_retries:
                logger.error(f"Timeout en {endpoint} después de {max_retries} intentos")
                raise ExternalServiceError(f"API no respondió después de {max_retries} intentos")
            logger.warning(f"Timeout en {endpoint}, reintentando...")
            
        except requests.HTTPError as e:
            logger.error(f"Error HTTP {e.response.status_code} en {endpoint}")
            raise ExternalServiceError(f"API retornó error {e.response.status_code}")
            
        except Exception as e:
            logger.exception(f"Error inesperado llamando {endpoint}")
            raise ExternalServiceError(f"Error llamando API: {str(e)}")
```

### 7. Manejo de errores en Lambda específicos
```python
# Lambda con reintentos y DLQ (Dead Letter Queue)
import json
import boto3

sqs = boto3.client('sqs')
DLQ_URL = os.environ['DLQ_URL']

def lambda_handler(event, context):
    """
    Usa DLQ para errores irrecuperables
    """
    try:
        # Procesar
        result = process_order(event)
        logger.info(f"Orden procesada: {event['order_id']}")
        return {'statusCode': 200, 'body': json.dumps(result)}
        
    except ValueError as e:
        # Error de datos (no reintentar)
        logger.warning(f"Validación fallida: {e}")
        send_to_dlq(event, str(e))
        return {'statusCode': 400, 'body': json.dumps({'error': str(e)})}
        
    except Exception as e:
        # Error desconocido (CloudWatch lo detectará y puede reintentar)
        logger.exception(f"Error crítico: {e}")
        raise  # Lambda reintentar automáticamente

def send_to_dlq(event, error_msg):
    """Enviar a Dead Letter Queue para análisis"""
    sqs.send_message(
        QueueUrl=DLQ_URL,
        MessageBody=json.dumps({
            'event': event,
            'error': error_msg,
            'timestamp': datetime.utcnow().isoformat()
        })
    )
```

### 8. Refactor de código existente
Identifica en tu código actual:
```bash
# Buscar prints que deberían ser logs
grep -r "print(" app/

# Buscar excepciones no capturadas
grep -r "raise" app/ | grep -v "except"

# Buscar código sin try-except
grep -r "requests\." app/ | grep -v "except"
```

**Refactor Example**:
```python
# ANTES (malo)
def upload_to_s3(file):
    s3_client = boto3.client("s3")
    s3_client.upload_file(file, "my-bucket", file)
    print("File uploaded")
    return "OK"

# DESPUÉS (mejor)
def upload_to_s3(file):
    try:
        s3_client = boto3.client("s3")
        s3_client.upload_file(file, "my-bucket", file)
        logger.info(f"Archivo cargado: {file}")
        return {"status": "success"}
    except FileNotFoundError:
        logger.warning(f"Archivo no encontrado: {file}")
        raise ValidationError(f"Archivo {file} no existe")
    except Exception as e:
        logger.exception(f"Error cargando a S3: {str(e)}")
        raise ExternalServiceError("Error cargando archivo a S3")
```

### 8. Testing de excepciones
Crea `tests/test_user_service.py`:
```python
import pytest
from app.services.user_service import UserService
from app.exceptions import ValidationError, NotFoundError

def test_create_user_valid():
    """Test con datos válidos"""
    service = UserService()
    user = service.create_user({
        "name": "Juan",
        "email": "juan@example.com",
        "password": "SecurePass123"
    })
    assert user.id is not None

def test_create_user_invalid_email():
    """Test con email faltante"""
    service = UserService()
    with pytest.raises(ValidationError):
        service.create_user({
            "name": "Juan",
            "password": "SecurePass123"
        })

def test_create_user_short_password():
    """Test con contraseña corta"""
    service = UserService()
    with pytest.raises(ValidationError) as exc_info:
        service.create_user({
            "name": "Juan",
            "email": "juan@example.com",
            "password": "short"
        })
    assert "8 caracteres" in str(exc_info.value)

def test_get_user_not_found():
    """Test obteniendo usuario inexistente"""
    service = UserService()
    with pytest.raises(NotFoundError):
        service.get_user(99999)
```

### 9. Revisar logs en CloudWatch
```bash
# Si la app corre en EC2/Lambda
aws logs tail /aws/lambda/my-function --follow

# Filtrar por nivel
aws logs filter-log-events \
  --log-group-name /aws/lambda/my-function \
  --filter-pattern "ERROR"

# Ver últimas líneas
aws logs tail /aws/lambda/my-function --since 1h
```

## 🔹 Ejercicio Práctico

### Tarea 1: Auditoría de errores
1. Abre 3 archivos de servicios en tu proyecto
2. Identifica todo `raise`, `requests.`, `boto3.`
3. Documenta si tienen try-except

**Formato de reporte**:
```
Archivo: app/services/product_service.py

Línea 45: requests.get(url)
  - ¿Tiene try-except? NO
  - Riesgo: Timeout sin manejo
  - Acción: Agregar except requests.Timeout

Línea 72: db.session.commit()
  - ¿Tiene try-except? SÍ
  - Riesgo: BAJO
```

### Tarea 2: Crear excepciones personalizadas
1. Define 5 excepciones específicas para tu app
2. Documenta cuándo se usan

### Tarea 3: Refactor de un servicio
1. Toma un servicio existente
2. Agrega logging en cada función
3. Envuelve operaciones riesgosas en try-except
4. Prueba con pytest

### Tarea 4: Configurar logging en CloudWatch
1. Agrega logger que envíe a CloudWatch
2. Verifica logs en la consola AWS

## 🔹 Verificación de Aprendizaje

- [ ] Entiendo la diferencia entre print y logging
- [ ] Puedo crear excepciones personalizadas
- [ ] Sé cómo usar try-except-finally correctamente
- [ ] Puedo debuggear con logs y stack traces
- [ ] He refactorizado código para agregar manejo de errores
- [ ] Puedo ver logs en CloudWatch

## 🔹 Recursos Complementarios

- [Python logging module](https://docs.python.org/3/library/logging.html)
- [Python exceptions](https://docs.python.org/3/tutorial/errors.html)
- [AWS CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/)
- [Boto3 error handling](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/error-handling.html)
- [pytest documentation](https://docs.pytest.org/)
