# Día 1 (27/01/2026) – Python - Lectura de Proyecto Real

## 🔹 Teoría

### Anatomía de una aplicación Python en AWS
Una aplicación completa en AWS típicamente está compuesta por varios servicios que interactúan entre sí:

- **Lambda**: Funciones serverless para procesamiento de eventos.
- **S3**: Almacenamiento de archivos estáticos y datos.
- **RDS/DynamoDB**: Bases de datos relacionales o NoSQL.
- **API Gateway**: Puerta de entrada para endpoints REST.
- **EC2**: Servidores virtuales para aplicaciones persistentes.
- **CloudWatch**: Monitoreo y logs de toda la arquitectura.

### Patrones de comunicación
- **Síncrono**: Lambda → RDS (consultas directas).
- **Asíncrono**: S3 → Lambda (eventos de carga).
- **REST**: API Gateway → Lambda/EC2.
- **Colas**: SQS para procesamiento desacoplado.

### Estructura típica de un proyecto Python
```
proyecto/
├── app/
│   ├── __init__.py
│   ├── main.py              # Punto de entrada
│   ├── config.py            # Configuración (dev/prod)
│   ├── models/              # Definición de datos
│   │   ├── user.py
│   │   └── product.py
│   ├── routes/              # Endpoints REST
│   │   ├── users.py
│   │   └── products.py
│   ├── services/            # Lógica de negocio
│   │   ├── auth_service.py
│   │   └── product_service.py
│   └── utils/               # Utilidades
│       ├── db.py            # Conexión a BD
│       ├── s3_client.py     # Interacción con S3
│       └── logger.py        # Logging centralizado
├── lambda_functions/        # Funciones Lambda independientes
│   ├── process_image.py
│   └── send_email.py
├── tests/                   # Tests unitarios e integración
├── requirements.txt         # Dependencias Python
├── docker-compose.yml       # Entorno local
├── Dockerfile
└── README.md
```

## 🔹 Conceptos Clave

### ¿Qué es una aplicación real?
No es solo código aislado. Es:
- Múltiples módulos interdependientes
- Manejo de bases de datos reales
- Integración con servicios externos (AWS, APIs)
- Configuración para diferentes entornos
- Tests y validación
- Logs y monitoreo

### Entendiendo el flujo de datos
```
Solicitud HTTP → API Gateway → Lambda/EC2
                                    ↓
                                   RDS/DynamoDB
                                    ↓
                                   S3 (si aplica)
                                    ↓
                                  Respuesta JSON
                                    ↓
                                  CloudWatch Logs
```

### Dependencias comunes
- **Flask/FastAPI**: Frameworks web
- **SQLAlchemy**: ORM para bases de datos
- **boto3**: SDK de AWS para Python
- **requests**: Cliente HTTP
- **python-dotenv**: Manejo de variables de entorno
- **pydantic**: Validación de datos
- **pytest**: Testing

### Variables de entorno
Nunca hardcodees credenciales. Usa variables de entorno:
```python
import os
from dotenv import load_dotenv

load_dotenv()
DB_USER = os.getenv("DB_USER")
DB_PASSWORD = os.getenv("DB_PASSWORD")
AWS_REGION = os.getenv("AWS_REGION", "us-east-1")
```

## 🔹 Práctica Paso a Paso

### 1. Clonar o descargar proyecto existente
```bash
# Opción A: Desde un repositorio
git clone https://github.com/tu-org/proyecto.git
cd proyecto

# Opción B: Desde un archivo ZIP
unzip proyecto.zip
cd proyecto
```

### 2. Revisar la estructura del proyecto
```bash
# Ver el árbol de carpetas
find . -type f -name "*.py" | head -20

# Ver dependencias
cat requirements.txt

# Ver configuración de Docker
cat Dockerfile
```

### 3. Entender el archivo de configuración
```python
# config.py - Ejemplo
import os

class Config:
    """Base configuration"""
    SECRET_KEY = os.getenv("SECRET_KEY", "dev-key")
    DB_URL = os.getenv("DATABASE_URL")
    AWS_REGION = os.getenv("AWS_REGION", "us-east-1")

class DevelopmentConfig(Config):
    """Development environment"""
    DEBUG = True
    DB_URL = "postgresql://localhost/myapp_dev"

class ProductionConfig(Config):
    """Production environment"""
    DEBUG = False
    DB_URL = os.getenv("DATABASE_URL")  # RDS en AWS

config = {
    "development": DevelopmentConfig,
    "production": ProductionConfig,
    "default": DevelopmentConfig
}
```

### 4. Analizar el punto de entrada
```python
# main.py - Ejemplo
from flask import Flask
from app.config import config

app = Flask(__name__)
app.config.from_object(config["development"])

# Importar blueprints (colecciones de rutas)
from app.routes import users_bp, products_bp
app.register_blueprint(users_bp)
app.register_blueprint(products_bp)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### 5. Examinar conexión a base de datos
```python
# utils/db.py - Ejemplo
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "postgresql://user:pass@localhost/myapp"
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(bind=engine)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### 6. Revisar integración con AWS
```python
# utils/s3_client.py - Ejemplo
import boto3

class S3Client:
    def __init__(self):
        self.s3 = boto3.client("s3", region_name="us-east-1")
        self.bucket = "my-app-bucket"
    
    def upload_file(self, file_path, key):
        self.s3.upload_file(file_path, self.bucket, key)
    
    def download_file(self, key, file_path):
        self.s3.download_file(self.bucket, key, file_path)
```

### 7. Inspeccionar rutas y endpoints
```python
# routes/users.py - Ejemplo
from flask import Blueprint, request, jsonify
from app.services.auth_service import AuthService

users_bp = Blueprint("users", __name__, url_prefix="/api/users")
auth_service = AuthService()

@users_bp.route("/", methods=["GET"])
def list_users():
    users = auth_service.get_all_users()
    return jsonify(users)

@users_bp.route("/<user_id>", methods=["GET"])
def get_user(user_id):
    user = auth_service.get_user_by_id(user_id)
    if not user:
        return {"error": "User not found"}, 404
    return jsonify(user)

@users_bp.route("/", methods=["POST"])
def create_user():
    data = request.json
    user = auth_service.create_user(data)
    return jsonify(user), 201
```

### 8. Establecer entorno local
```bash
# Crear entorno virtual
python3 -m venv venv
source venv/bin/activate  # En Windows: venv\Scripts\activate

# Instalar dependencias
pip install -r requirements.txt

# Crear archivo .env
cat > .env << EOF
FLASK_ENV=development
DATABASE_URL=postgresql://user:pass@localhost/myapp_dev
AWS_REGION=us-east-1
SECRET_KEY=dev-secret-key
EOF

# Correr migraciones de base de datos (si aplica)
flask db upgrade

# Iniciar la aplicación
python main.py
```

### 9. Probar endpoints con curl
```bash
# Listar usuarios
curl http://localhost:5000/api/users

# Obtener usuario específico
curl http://localhost:5000/api/users/1

# Crear usuario
curl -X POST http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Juan", "email": "juan@example.com"}'
```

### 10. Revisar logs y salida de ejecución
```bash
# Ver logs en tiempo real
tail -f logs/app.log

# Buscar errores específicos
grep "ERROR" logs/app.log

# Ver logs de CloudWatch (si está en AWS)
aws logs tail /aws/lambda/my-function --follow
```

## 🔹 Ejemplo Real: EC2 vs Lambda

### Arquitectura en EC2
```
Cliente HTTP
    ↓
Load Balancer (ALB) - puerto 443
    ↓
EC2 Instance (t3.medium)
├── Docker container
│   ├── FastAPI app (puerto 8000)
│   ├── Gunicorn workers
│   └── SQLAlchemy ORM
├── PostgreSQL RDS connection pool
└── S3 client (boto3)
    ↓
Base de datos RDS
Storage S3
CloudWatch Logs
```

**Estructura de proyecto en EC2**:
```bash
# En /opt/myapp (EC2)
myapp/
├── main.py              # ASGI app (FastAPI)
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
├── app/
│   ├── models/          # SQLAlchemy ORM
│   ├── routes/          # Endpoints FastAPI
│   └── services/        # Lógica de negocio
└── logs/
    └── app.log          # Logs locales (CloudWatch Agent)
```

**Iniciar en EC2**:
```bash
# En la instancia EC2
docker pull myregistry/myapp:v1.0.0
docker run -d -p 8000:8000 \
  -e DATABASE_URL=postgresql://user:pass@rds.aws/myapp \
  -e ENVIRONMENT=production \
  myregistry/myapp:v1.0.0

# Verificar salud
curl http://localhost:8000/health
# {"status": "healthy"}
```

### Arquitectura en Lambda
```
Cliente HTTP
    ↓
API Gateway (endpoint: /api/products)
    ↓
Lambda Function (Python 3.11)
├── Event handler
├── SQLAlchemy ORM (RDS Proxy)
└── boto3 (S3 client)
    ↓
RDS (via RDS Proxy para connection pooling)
Storage S3
CloudWatch Logs (automático)
```

**Estructura de Lambda**:
```bash
# Código para Lambda
lambda_function.zip
├── lambda_function.py   # Handler principal
├── requirements.txt
└── app/
    ├── models/
    ├── services/
    └── utils/
```

**lambda_function.py - Ejemplo real**:
```python
import json
import boto3
import psycopg2
from psycopg2.pool import SimpleConnectionPool
import os
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Connection pool (reutilizar entre invocaciones)
db_pool = None

def get_db_pool():
    global db_pool
    if db_pool is None:
        db_pool = SimpleConnectionPool(
            1, 5,  # min, max connections
            host=os.environ['DB_HOST'],
            port=5432,
            database=os.environ['DB_NAME'],
            user=os.environ['DB_USER'],
            password=os.environ['DB_PASSWORD']
        )
    return db_pool

def lambda_handler(event, context):
    """
    API Gateway trigger
    
    event = {
        'httpMethod': 'GET',
        'path': '/api/products',
        'queryStringParameters': {'page': '1'},
        'headers': {'Content-Type': 'application/json'}
    }
    """
    
    logger.info(f"Evento recibido: {event['httpMethod']} {event['path']}")
    
    try:
        http_method = event['httpMethod']
        path = event['path']
        
        # Routing
        if path == '/api/products' and http_method == 'GET':
            return get_products(event)
        elif path == '/api/products' and http_method == 'POST':
            return create_product(event)
        else:
            return {
                'statusCode': 404,
                'body': json.dumps({'error': 'Not found'})
            }
    
    except Exception as e:
        logger.exception(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': 'Internal server error'})
        }

def get_products(event):
    """GET /api/products"""
    pool = get_db_pool()
    conn = pool.getconn()
    
    try:
        with conn.cursor() as cur:
            # Paginación
            page = int(event.get('queryStringParameters', {}).get('page', 1))
            limit = 10
            offset = (page - 1) * limit
            
            cur.execute(
                "SELECT id, name, price FROM products LIMIT %s OFFSET %s",
                (limit, offset)
            )
            products = cur.fetchall()
            
            logger.info(f"Productos obtenidos: {len(products)}")
            
            return {
                'statusCode': 200,
                'headers': {'Content-Type': 'application/json'},
                'body': json.dumps({
                    'products': [
                        {'id': p[0], 'name': p[1], 'price': p[2]}
                        for p in products
                    ]
                })
            }
    finally:
        pool.putconn(conn)

def create_product(event):
    """POST /api/products"""
    pool = get_db_pool()
    conn = pool.getconn()
    
    try:
        body = json.loads(event.get('body', '{}'))
        
        # Validación
        if not body.get('name') or not body.get('price'):
            return {
                'statusCode': 400,
                'body': json.dumps({'error': 'Missing name or price'})
            }
        
        with conn.cursor() as cur:
            cur.execute(
                "INSERT INTO products (name, price) VALUES (%s, %s) RETURNING id",
                (body['name'], float(body['price']))
            )
            product_id = cur.fetchone()[0]
            conn.commit()
            
            logger.info(f"Producto creado: {product_id}")
            
            return {
                'statusCode': 201,
                'body': json.dumps({'id': product_id})
            }
    except ValueError as e:
        return {
            'statusCode': 400,
            'body': json.dumps({'error': str(e)})
        }
    finally:
        pool.putconn(conn)
```

**Template.yaml para Lambda**:
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: python3.11
    Timeout: 30
    MemorySize: 512
    Environment:
      Variables:
        DB_HOST: !GetAtt RDSDatabase.Endpoint.Address
        DB_NAME: myapp
        DB_USER: admin
        DB_PASSWORD: !Sub '{{resolve:secretsmanager:db-password:SecretString:password}}'

Resources:
  GetProductsFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: .
      Handler: lambda_function.lambda_handler
      Events:
        ApiGet:
          Type: Api
          Properties:
            RestApiId: !Ref MyApi
            Path: /api/products
            Method: get
        ApiPost:
          Type: Api
          Properties:
            RestApiId: !Ref MyApi
            Path: /api/products
            Method: post
      Policies:
        - AWSLambdaVPCAccessExecutionRole  # Para RDS en VPC
        - SecretsManagerReadWrite
      VpcConfig:
        SecurityGroupIds:
          - !Ref LambdaSecurityGroup
        SubnetIds:
          - !Ref PrivateSubnet1
          - !Ref PrivateSubnet2

  MyApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      Auth:
        ApiKeyRequired: true

Outputs:
  ApiEndpoint:
    Value: !Sub 'https://${MyApi}.execute-api.${AWS::Region}.amazonaws.com/prod'
```

## 🔹 Ejercicio Práctico

### Tarea 1: Mapeo de arquitectura
Crea un diagrama simple de tu proyecto con:
- Componentes (EC2, Lambda, RDS, S3, API Gateway)
- Flujos de datos entre ellos
- Dependencias externas

**Ejemplo EC2**:
```
Cliente HTTP
    ↓
ALB (puerto 443)
    ↓
EC2 Auto Scaling Group
├── Docker: FastAPI
├── Connection Pool RDS
└── S3 Client
    ↓
RDS PostgreSQL
S3 Bucket
CloudWatch Logs
```

**Ejemplo Lambda**:
```
Cliente HTTP
    ↓
API Gateway (endpoint: /api/products)
    ↓
Lambda (Python 3.11, 512MB)
├── Event handler
├── DB Pool (5 conexiones)
└── boto3
    ↓
RDS (via RDS Proxy)
S3 Bucket
CloudWatch Logs (automático)
```

### Tarea 2: Análisis de dependencias
1. Abre `requirements.txt` y lista los primeros 10 paquetes.
2. Investiga qué hace cada uno:
   - `flask`: Framework web
   - `sqlalchemy`: ORM
   - `boto3`: SDK de AWS
   - Etc.

### Tarea 3: Ejecución local
1. Configura tu entorno virtual.
2. Instala dependencias.
3. Arranca la aplicación localmente.
4. Prueba al menos 3 endpoints con curl o Postman.
5. Documenta qué hace cada uno.

### Tarea 4: Inspección de código
1. Abre `config.py` y explica las variables de cada entorno.
2. Abre `models/` y describe qué entidades de BD se definen.
3. Abre `services/` y explica la lógica de negocio de 2 servicios.

## 🔹 Verificación de Aprendizaje

- [ ] Entiendo la estructura típica de un proyecto Python
- [ ] Puedo explicar cómo se integran EC2, Lambda, RDS, S3 y API Gateway
- [ ] He corrido localmente la aplicación y probado endpoints
- [ ] Puedo navegar el código y entender su flujo
- [ ] Conozco dónde buscar logs y errores

## 🔹 Recursos Complementarios

- [Flask Documentation](https://flask.palletsprojects.com/)
- [SQLAlchemy ORM](https://docs.sqlalchemy.org/)
- [Boto3 AWS SDK](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html)
- [Python Logging](https://docs.python.org/3/library/logging.html)
- [AWS Architecture Patterns](https://aws.amazon.com/architecture/reference-architecture-diagrams/)
