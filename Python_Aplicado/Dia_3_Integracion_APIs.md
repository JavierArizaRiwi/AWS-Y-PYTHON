# Día 3 (29/01/2026) – Python - Integración con APIs

## 🔹 Teoría

### ¿Qué es una API REST?
Una API (Application Programming Interface) REST es un conjunto de endpoints HTTP que permiten a aplicaciones comunicarse entre sí:

- **HTTP GET**: Obtener datos
- **HTTP POST**: Crear datos
- **HTTP PUT/PATCH**: Actualizar datos
- **HTTP DELETE**: Eliminar datos

### Componentes de una solicitud HTTP
```
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer token123

{
  "name": "Juan",
  "email": "juan@example.com"
}
```

- **Method**: POST (acción)
- **Endpoint**: /api/users (recurso)
- **Headers**: Metadatos (Content-Type, Authorization)
- **Body**: Datos a enviar

### Códigos de respuesta HTTP
```
2xx - Éxito
  200 OK - Solicitud exitosa
  201 Created - Recurso creado
  204 No Content - Éxito sin contenido

3xx - Redirección
  301 Moved Permanently - Recurso movido
  302 Found - Redirección temporal

4xx - Error del cliente
  400 Bad Request - Solicitud inválida
  401 Unauthorized - Sin autenticación
  403 Forbidden - Sin autorización
  404 Not Found - Recurso no existe
  429 Too Many Requests - Limite de rate

5xx - Error del servidor
  500 Internal Server Error - Error desconocido
  502 Bad Gateway - Problema con proxy
  503 Service Unavailable - Servicio no disponible
```

### Patrones de autenticación
```
1. API Key:
   Authorization: X-API-Key: your-api-key

2. Bearer Token (JWT):
   Authorization: Bearer eyJhbGc...

3. Basic Auth:
   Authorization: Basic base64(user:pass)

4. OAuth2:
   Intercambio de credenciales por token
```

### Timeouts y reintentos
- **Timeout**: Tiempo máximo de espera
- **Reintento**: Volver a intentar si falla
- **Backoff exponencial**: Esperar más entre intentos

```
Intento 1: falla
  Espera 1 segundo
Intento 2: falla
  Espera 2 segundos
Intento 3: falla
  Espera 4 segundos
Intento 4: éxito
```

## 🔹 Conceptos Clave

### Librería requests
```python
import requests

# GET
response = requests.get("https://api.example.com/users")

# POST
response = requests.post(
    "https://api.example.com/users",
    json={"name": "Juan", "email": "juan@example.com"},
    headers={"Authorization": "Bearer token"}
)

# Headers personalizados
headers = {
    "Authorization": f"Bearer {token}",
    "Content-Type": "application/json",
    "User-Agent": "MyApp/1.0"
}

# Query parameters
params = {"page": 1, "limit": 10}
response = requests.get(url, params=params)

# Timeout (segundos)
response = requests.get(url, timeout=5)
```

### FastAPI para exponer APIs
```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel

app = FastAPI()

class User(BaseModel):
    name: str
    email: str

@app.get("/api/users")
async def list_users():
    """Lista todos los usuarios"""
    return [{"id": 1, "name": "Juan"}]

@app.get("/api/users/{user_id}")
async def get_user(user_id: int):
    """Obtiene un usuario por ID"""
    if user_id == 1:
        return {"id": 1, "name": "Juan"}
    raise HTTPException(status_code=404, detail="Usuario no encontrado")

@app.post("/api/users", status_code=status.HTTP_201_CREATED)
async def create_user(user: User):
    """Crea un nuevo usuario"""
    return {"id": 2, "name": user.name, "email": user.email}

@app.put("/api/users/{user_id}")
async def update_user(user_id: int, user: User):
    """Actualiza un usuario"""
    return {"id": user_id, "name": user.name, "email": user.email}

@app.delete("/api/users/{user_id}")
async def delete_user(user_id: int):
    """Elimina un usuario"""
    return {"status": "eliminado"}
```

### Validación con Pydantic
```python
from pydantic import BaseModel, EmailStr, Field

class User(BaseModel):
    name: str = Field(..., min_length=3, max_length=50)
    email: EmailStr
    age: int = Field(ge=18, le=120)

class Product(BaseModel):
    title: str
    price: float = Field(gt=0)
    description: str | None = None

# FastAPI valida automáticamente
@app.post("/users")
async def create_user(user: User):
    # Si los datos no coinciden el schema, FastAPI retorna 422
    return user
```

### Clientes HTTP con reintentos
```python
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session_with_retries(max_retries=3):
    """Crea sesión con reintentos automáticos"""
    session = requests.Session()
    
    retry_strategy = Retry(
        total=max_retries,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504],
        allowed_methods=["HEAD", "GET", "OPTIONS", "POST"]
    )
    
    adapter = HTTPAdapter(max_retries=retry_strategy)
    session.mount("http://", adapter)
    session.mount("https://", adapter)
    
    return session

# Uso
session = create_session_with_retries()
response = session.get("https://api.example.com/data")
```

## 🔹 Ejemplos Reales: EC2 vs Lambda

### API REST en EC2 (FastAPI + Gunicorn)
```python
# app/main.py - EC2
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import logging

app = FastAPI(title="Mi API en EC2")
logger = logging.getLogger(__name__)

# CORS para clientes web
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/health")
async def health():
    """Health check para load balancer"""
    return {"status": "healthy"}

@app.get("/api/users/{user_id}")
async def get_user(user_id: int):
    """Endpoint con conexión persistente a RDS"""
    from app.repositories.user_repository import UserRepository
    from app.models.database import SessionLocal
    
    db = SessionLocal()  # Reutiliza conexión del pool
    try:
        repo = UserRepository(db)
        user = repo.get_by_id(user_id)
        if not user:
            return {"error": "Not found"}, 404
        return user
    finally:
        db.close()

if __name__ == "__main__":
    # Ejecutar con: gunicorn -w 4 -b 0.0.0.0:8000 main:app
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**Dockerfile para EC2**:
```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y postgresql-client && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV PYTHONUNBUFFERED=1

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:8000/health || exit 1

CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:8000", "--timeout", "60", "--access-logfile", "-", "app.main:app"]
```

### API REST en Lambda (API Gateway + Function)
```python
# lambda_function.py - Lambda
import json
import boto3
import logging
from datetime import datetime

logger = logging.getLogger()
logger.setLevel(logging.INFO)

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def lambda_handler(event, context):
    """
    event = {
        'httpMethod': 'GET',
        'path': '/api/users/123',
        'pathParameters': {'user_id': '123'},
        'headers': {'Authorization': 'Bearer token'},
        'body': None,
        'requestContext': {
            'requestId': '550e8400-e29b-41d4-a716-446655440000'
        }
    }
    """
    
    logger.info(f"Request: {event['httpMethod']} {event['path']}")
    
    try:
        http_method = event['httpMethod']
        path = event['path']
        
        # Routing
        if path.startswith('/api/users/') and http_method == 'GET':
            user_id = event['pathParameters']['user_id']
            return get_user(user_id)
        
        elif path == '/api/users' and http_method == 'POST':
            return create_user(event)
        
        else:
            return response_error(404, "Not found")
    
    except Exception as e:
        logger.exception(f"Error: {str(e)}")
        return response_error(500, "Internal server error")

def get_user(user_id):
    """GET /api/users/{user_id}"""
    try:
        response = table.get_item(Key={'user_id': user_id})
        
        if 'Item' not in response:
            return response_error(404, "User not found")
        
        logger.info(f"User retrieved: {user_id}")
        return {
            'statusCode': 200,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps(response['Item'])
        }
    except Exception as e:
        logger.exception(f"Error getting user {user_id}")
        return response_error(500, str(e))

def create_user(event):
    """POST /api/users"""
    try:
        body = json.loads(event.get('body', '{}'))
        
        # Validación
        if not body.get('name') or not body.get('email'):
            return response_error(400, "Missing name or email")
        
        # Guardar en DynamoDB
        table.put_item(Item={
            'user_id': str(datetime.utcnow().timestamp()),
            'name': body['name'],
            'email': body['email'],
            'created_at': datetime.utcnow().isoformat()
        })
        
        logger.info(f"User created: {body['email']}")
        return {
            'statusCode': 201,
            'body': json.dumps({'status': 'created'})
        }
    except ValueError as e:
        return response_error(400, str(e))

def response_error(status_code, message):
    """Helper para respuestas de error"""
    return {
        'statusCode': status_code,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({'error': message})
    }
```

**SAM Template para Lambda**:
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: python3.11
    Timeout: 30
    MemorySize: 512
    Tracing: Active
    Environment:
      Variables:
        TABLE_NAME: Users

Resources:
  ApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: .
      Handler: lambda_function.lambda_handler
      Policies:
        - DynamoDBCrudPolicy:
            TableName: Users
      Events:
        GetUser:
          Type: Api
          Properties:
            RestApiId: !Ref MyApi
            Path: /api/users/{user_id}
            Method: get
        CreateUser:
          Type: Api
          Properties:
            RestApiId: !Ref MyApi
            Path: /api/users
            Method: post

  MyApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      TracingEnabled: true

Outputs:
  ApiEndpoint:
    Value: !Sub 'https://${MyApi}.execute-api.${AWS::Region}.amazonaws.com/prod'
```

## 🔹 Práctica Paso a Paso

### 1. Cliente HTTP para consumir APIs
Crea `app/utils/http_client.py`:
```python
import requests
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry
from app.utils.logger import setup_logger
from app.exceptions import ExternalServiceError

logger = setup_logger(__name__)

class HTTPClient:
    """Cliente HTTP reutilizable con reintentos"""
    
    def __init__(self, base_url="", timeout=5, max_retries=3):
        self.base_url = base_url
        self.timeout = timeout
        self.session = self._create_session(max_retries)
    
    def _create_session(self, max_retries):
        """Crea sesión con estrategia de reintentos"""
        session = requests.Session()
        
        retry = Retry(
            total=max_retries,
            backoff_factor=1,
            status_forcelist=[429, 500, 502, 503, 504]
        )
        
        adapter = HTTPAdapter(max_retries=retry)
        session.mount("http://", adapter)
        session.mount("https://", adapter)
        
        return session
    
    def request(self, method, endpoint, **kwargs):
        """Realiza solicitud HTTP"""
        url = f"{self.base_url}{endpoint}"
        
        try:
            logger.debug(f"{method} {url}")
            
            response = self.session.request(
                method,
                url,
                timeout=self.timeout,
                **kwargs
            )
            
            response.raise_for_status()
            logger.info(f"{method} {endpoint} -> {response.status_code}")
            
            return response
            
        except requests.Timeout:
            logger.error(f"Timeout en {method} {endpoint}")
            raise ExternalServiceError("API no respondió (timeout)")
        except requests.HTTPError as e:
            logger.error(f"HTTP {e.response.status_code} en {endpoint}")
            raise ExternalServiceError(f"API error: {e.response.status_code}")
        except Exception as e:
            logger.exception(f"Error en {method} {endpoint}")
            raise ExternalServiceError(f"Error en API: {str(e)}")
    
    def get(self, endpoint, **kwargs):
        return self.request("GET", endpoint, **kwargs)
    
    def post(self, endpoint, **kwargs):
        return self.request("POST", endpoint, **kwargs)
    
    def put(self, endpoint, **kwargs):
        return self.request("PUT", endpoint, **kwargs)
    
    def delete(self, endpoint, **kwargs):
        return self.request("DELETE", endpoint, **kwargs)
```

### 2. Servicio para consumir API externa
Crea `app/services/external_api_service.py`:
```python
import os
from app.utils.http_client import HTTPClient
from app.utils.logger import setup_logger

logger = setup_logger(__name__)

class ExternalAPIService:
    """Consumidor de APIs externas"""
    
    def __init__(self):
        self.api_base = os.getenv(
            "EXTERNAL_API_BASE",
            "https://api.example.com"
        )
        self.api_key = os.getenv("EXTERNAL_API_KEY")
        self.client = HTTPClient(base_url=self.api_base)
    
    def get_user_data(self, user_id):
        """Obtiene datos de usuario desde API externa"""
        headers = {"Authorization": f"Bearer {self.api_key}"}
        
        response = self.client.get(
            f"/users/{user_id}",
            headers=headers
        )
        
        return response.json()
    
    def create_order(self, user_id, items):
        """Crea un pedido en API externa"""
        headers = {"Authorization": f"Bearer {self.api_key}"}
        
        payload = {
            "user_id": user_id,
            "items": items
        }
        
        response = self.client.post(
            "/orders",
            json=payload,
            headers=headers
        )
        
        return response.json()
    
    def sync_users(self):
        """Sincroniza usuarios desde API externa"""
        headers = {"Authorization": f"Bearer {self.api_key}"}
        
        page = 1
        all_users = []
        
        while True:
            response = self.client.get(
                "/users",
                params={"page": page, "limit": 100},
                headers=headers
            )
            
            data = response.json()
            all_users.extend(data.get("users", []))
            
            if not data.get("has_next"):
                break
            
            page += 1
            logger.info(f"Descargada página {page} de usuarios")
        
        logger.info(f"Total usuarios sincronizados: {len(all_users)}")
        return all_users
```

### 3. Crear API con FastAPI
Crea `app/api/main.py`:
```python
from fastapi import FastAPI, HTTPException, status, Header
from pydantic import BaseModel, EmailStr
from app.services.user_service import UserService
from app.utils.logger import setup_logger

app = FastAPI(title="Mi API")
logger = setup_logger(__name__)
user_service = UserService()

# Modelos de validación
class UserCreate(BaseModel):
    name: str
    email: EmailStr
    password: str

class UserResponse(BaseModel):
    id: int
    name: str
    email: str

class ErrorResponse(BaseModel):
    error: str
    detail: str

# Middleware de logging
@app.middleware("http")
async def log_requests(request, call_next):
    logger.info(f"{request.method} {request.url.path}")
    response = await call_next(request)
    logger.info(f"{request.method} {request.url.path} -> {response.status_code}")
    return response

# Rutas
@app.get("/api/health", tags=["Health"])
async def health_check():
    """Health check endpoint"""
    return {"status": "healthy"}

@app.get("/api/users", response_model=list[UserResponse])
async def list_users(skip: int = 0, limit: int = 10):
    """Lista usuarios con paginación"""
    try:
        users = user_service.get_all_users(skip=skip, limit=limit)
        return users
    except Exception as e:
        logger.exception("Error listando usuarios")
        raise HTTPException(
            status_code=500,
            detail="Error obteniendo usuarios"
        )

@app.get("/api/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: int):
    """Obtiene un usuario por ID"""
    try:
        user = user_service.get_user(user_id)
        if not user:
            raise HTTPException(
                status_code=404,
                detail=f"Usuario {user_id} no encontrado"
            )
        return user
    except Exception as e:
        logger.exception(f"Error obteniendo usuario {user_id}")
        raise HTTPException(status_code=500, detail="Error interno")

@app.post("/api/users", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(user: UserCreate):
    """Crea un nuevo usuario"""
    try:
        new_user = user_service.create_user(
            name=user.name,
            email=user.email,
            password=user.password
        )
        logger.info(f"Usuario creado: {new_user.id}")
        return new_user
    except ValueError as e:
        logger.warning(f"Error de validación: {str(e)}")
        raise HTTPException(status_code=400, detail=str(e))
    except Exception as e:
        logger.exception("Error creando usuario")
        raise HTTPException(status_code=500, detail="Error interno")

@app.put("/api/users/{user_id}", response_model=UserResponse)
async def update_user(user_id: int, user: UserCreate):
    """Actualiza un usuario"""
    try:
        updated = user_service.update_user(
            user_id,
            name=user.name,
            email=user.email,
            password=user.password
        )
        if not updated:
            raise HTTPException(status_code=404, detail="Usuario no encontrado")
        logger.info(f"Usuario actualizado: {user_id}")
        return updated
    except Exception as e:
        logger.exception(f"Error actualizando usuario {user_id}")
        raise HTTPException(status_code=500, detail="Error interno")

@app.delete("/api/users/{user_id}")
async def delete_user(user_id: int):
    """Elimina un usuario"""
    try:
        success = user_service.delete_user(user_id)
        if not success:
            raise HTTPException(status_code=404, detail="Usuario no encontrado")
        logger.info(f"Usuario eliminado: {user_id}")
        return {"status": "eliminado"}
    except Exception as e:
        logger.exception(f"Error eliminando usuario {user_id}")
        raise HTTPException(status_code=500, detail="Error interno")

@app.exception_handler(HTTPException)
async def http_exception_handler(request, exc):
    """Manejador personalizado de excepciones HTTP"""
    logger.error(f"HTTP {exc.status_code}: {exc.detail}")
    return {"error": exc.detail}
```

### 4. Testing de APIs con Postman/curl
```bash
# Obtener health check
curl http://localhost:8000/api/health

# Listar usuarios
curl http://localhost:8000/api/users

# Obtener usuario específico
curl http://localhost:8000/api/users/1

# Crear usuario
curl -X POST http://localhost:8000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Juan", "email": "juan@example.com", "password": "SecurePass123"}'

# Actualizar usuario
curl -X PUT http://localhost:8000/api/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name": "Juan Pérez", "email": "juan@example.com", "password": "NewPass123"}'

# Eliminar usuario
curl -X DELETE http://localhost:8000/api/users/1
```

### 5. Testing de APIs con pytest
Crea `tests/test_api.py`:
```python
from fastapi.testclient import TestClient
from app.api.main import app

client = TestClient(app)

def test_health_check():
    """Test del health check"""
    response = client.get("/api/health")
    assert response.status_code == 200
    assert response.json()["status"] == "healthy"

def test_create_user():
    """Test crear usuario"""
    response = client.post("/api/users", json={
        "name": "Test User",
        "email": "test@example.com",
        "password": "TestPass123"
    })
    assert response.status_code == 201
    assert response.json()["email"] == "test@example.com"

def test_get_user_not_found():
    """Test obtener usuario inexistente"""
    response = client.get("/api/users/99999")
    assert response.status_code == 404

def test_create_user_invalid_email():
    """Test crear usuario con email inválido"""
    response = client.post("/api/users", json={
        "name": "Test User",
        "email": "invalid-email",
        "password": "TestPass123"
    })
    assert response.status_code == 422  # Validation error

def test_list_users_pagination():
    """Test paginación"""
    response = client.get("/api/users?skip=0&limit=5")
    assert response.status_code == 200
```

### 6. Documentación automática
FastAPI genera automáticamente documentación interactiva:

```bash
# Swagger UI (recomendado)
http://localhost:8000/docs

# ReDoc (alternativa)
http://localhost:8000/redoc
```

### 7. Manejo de rate limiting
```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter

@app.get("/api/users")
@limiter.limit("10/minute")
async def list_users(request):
    """Máximo 10 solicitudes por minuto"""
    return []
```

### 8. CORS para acceso desde frontend
```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://example.com"],  # URLs permitidas
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## 🔹 Ejercicio Práctico

### Tarea 1: Consumir API pública
1. Usa `requests` para consumir [JSONPlaceholder](https://jsonplaceholder.typicode.com)
2. Script para obtener usuarios y posts
3. Maneja errores y reintentos

```python
import requests

response = requests.get("https://jsonplaceholder.typicode.com/users/1")
user = response.json()
print(user)
```

### Tarea 2: Crear endpoints FastAPI
1. Crea 3 endpoints GET, POST, DELETE
2. Agrega validación con Pydantic
3. Documenta en Swagger

### Tarea 3: Testing
1. Escribe 5 tests para tus endpoints
2. Prueba casos exitosos y de error
3. Ejecuta con `pytest -v`

### Tarea 4: Cliente HTTP avanzado
1. Crea cliente con reintentos automáticos
2. Agregar timeout
3. Manejar 429 (rate limit)

## 🔹 Verificación de Aprendizaje

- [ ] Entiendo estructura de solicitudes/respuestas HTTP
- [ ] Puedo consumir APIs externas con requests
- [ ] He creado endpoints REST con FastAPI
- [ ] Sé validar datos con Pydantic
- [ ] Puedo testear APIs con pytest
- [ ] Entiendo rate limiting y reintentos

## 🔹 Recursos Complementarios

- [Requests documentation](https://requests.readthedocs.io/)
- [FastAPI tutorial](https://fastapi.tiangolo.com/)
- [Pydantic documentation](https://docs.pydantic.dev/)
- [JSONPlaceholder](https://jsonplaceholder.typicode.com/) - API de prueba
- [HTTP status codes](https://httpwg.org/specs/rfc9110.html#status.codes)
