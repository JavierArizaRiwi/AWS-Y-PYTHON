# Día 4 (30/01/2026) – Python - Base de Datos

## 🔹 Teoría

### Tipos de bases de datos en AWS

#### RDS (Relational Database Service)
- **PostgreSQL**: Relacional, ACID, JSON support
- **MySQL**: Relacional, rápido, amplio soporte
- **Oracle**: Empresarial, características avanzadas
- **MariaDB**: Fork de MySQL, rendimiento

**Ideal para**:
- Datos estructurados
- Relaciones entre tablas
- ACID (transacciones)
- Reportes complejos

#### DynamoDB
- **NoSQL**: Key-value y documento
- **Serverless**: Sin provisionar capacidad
- **Escalable**: Automática según demanda
- **Rápida**: Baja latencia

**Ideal para**:
- Datos no estructurados
- Alta velocidad
- Escalabilidad masiva
- Acceso por clave primaria

### Modelos de datos

#### Relacional (RDS)
```sql
-- Tabla users
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(100) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tabla orders
CREATE TABLE orders (
  id INT PRIMARY KEY AUTO_INCREMENT,
  user_id INT NOT NULL,
  total DECIMAL(10, 2),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id)
);
```

Relaciones:
- 1:1 (Un usuario - un perfil)
- 1:N (Un usuario - muchos pedidos)
- M:N (Muchos usuarios - muchos productos via tabla de unión)

#### NoSQL (DynamoDB)
```python
# Documento de usuario en DynamoDB
{
  "user_id": "user-123",           # Clave de partición
  "email": "juan@example.com",     # Clave de ordenamiento
  "name": "Juan",
  "preferences": {
    "language": "es",
    "notifications": True
  },
  "orders": [
    {"id": "order-1", "total": 100},
    {"id": "order-2", "total": 200}
  ]
}
```

### Conexión y ciclo de vida

```
Aplicación
    ↓
Connection Pool (reutiliza conexiones)
    ↓
BD
    ↓
Resultado
    ↓
Liberar conexión (back to pool)
```

### Errores comunes de conexión
- **Connection refused**: BD no está corriendo
- **Authentication failed**: Credenciales incorrectas
- **Network timeout**: BD no responde en tiempo
- **Too many connections**: Pool lleno
- **Connection lost**: Conexión intermitente

## 🔹 Conceptos Clave

### SQLAlchemy ORM
```python
from sqlalchemy import create_engine, Column, Integer, String, DateTime, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, relationship
from datetime import datetime

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    name = Column(String(100), nullable=False)
    email = Column(String(100), unique=True, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    # Relación con órdenes
    orders = relationship("Order", back_populates="user")
    
    def __repr__(self):
        return f"<User(id={self.id}, name={self.name})>"

class Order(Base):
    __tablename__ = "orders"
    
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    total = Column(Float, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    # Relación con usuario
    user = relationship("User", back_populates="orders")

# Crear motor de BD
DATABASE_URL = "postgresql://user:pass@localhost/myapp"
engine = create_engine(DATABASE_URL, echo=True)  # echo=True para ver SQL
Base.metadata.create_all(engine)  # Crear tablas

# Crear sesión (transacción)
Session = sessionmaker(bind=engine)
```

### CRUD (Create, Read, Update, Delete)
```python
from sqlalchemy.orm import Session

class UserRepository:
    def __init__(self, db: Session):
        self.db = db
    
    # CREATE
    def create(self, name: str, email: str) -> User:
        user = User(name=name, email=email)
        self.db.add(user)
        self.db.commit()
        self.db.refresh(user)  # Recargar desde BD
        return user
    
    # READ
    def get_by_id(self, user_id: int) -> User | None:
        return self.db.query(User).filter(User.id == user_id).first()
    
    def get_by_email(self, email: str) -> User | None:
        return self.db.query(User).filter(User.email == email).first()
    
    def get_all(self, skip: int = 0, limit: int = 10) -> list[User]:
        return self.db.query(User).offset(skip).limit(limit).all()
    
    # UPDATE
    def update(self, user_id: int, **kwargs) -> User | None:
        user = self.get_by_id(user_id)
        if not user:
            return None
        
        for key, value in kwargs.items():
            setattr(user, key, value)
        
        self.db.commit()
        self.db.refresh(user)
        return user
    
    # DELETE
    def delete(self, user_id: int) -> bool:
        user = self.get_by_id(user_id)
        if not user:
            return False
        
        self.db.delete(user)
        self.db.commit()
        return True
```

### Transacciones
```python
def transfer_money(from_user_id: int, to_user_id: int, amount: float):
    """Transacción: debe ser todo o nada"""
    try:
        # Restar de cuenta origen
        from_user = db.query(User).filter(User.id == from_user_id).first()
        from_user.balance -= amount
        
        # Sumar a cuenta destino
        to_user = db.query(User).filter(User.id == to_user_id).first()
        to_user.balance += amount
        
        # Si ambas operaciones exitosas, commit
        db.commit()
        logger.info(f"Transferencia exitosa: {from_user_id} -> {to_user_id}")
        
    except Exception as e:
        # Si algo falla, rollback de ambas
        db.rollback()
        logger.error(f"Transferencia fallida: {str(e)}")
        raise
```

### Boto3 para DynamoDB
```python
import boto3
from boto3.dynamodb.conditions import Key

# Crear cliente
dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
table = dynamodb.Table("Users")

# CREATE
table.put_item(Item={
    "user_id": "user-123",
    "name": "Juan",
    "email": "juan@example.com"
})

# READ
response = table.get_item(Key={"user_id": "user-123"})
user = response.get("Item")

# UPDATE
table.update_item(
    Key={"user_id": "user-123"},
    UpdateExpression="SET #n = :val",
    ExpressionAttributeNames={"#n": "name"},
    ExpressionAttributeValues={":val": "Juan Pérez"}
)

# DELETE
table.delete_item(Key={"user_id": "user-123"})

# QUERY
response = table.query(
    KeyConditionExpression=Key("user_id").eq("user-123")
)

# SCAN (búsqueda sin clave primaria)
response = table.scan(
    FilterExpression=Key("age").gte(18)
)
```

## 🔹 Ejemplos Reales: EC2 vs Lambda

### Conexión a BD en EC2 (RDS PostgreSQL)
```python
# app/models/database.py - EC2
import os
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# URL de RDS (desde variables de entorno)
DATABASE_URL = os.getenv(
    "DATABASE_URL",
    "postgresql://admin:password@mydb.123456789.us-east-1.rds.amazonaws.com:5432/myapp"
)

# Connection Pool para múltiples requests
engine = create_engine(
    DATABASE_URL,
    pool_size=10,           # Máximo de conexiones simultáneas
    max_overflow=20,        # Conexiones extra si es necesario
    pool_timeout=30,        # Timeout en segundos
    pool_pre_ping=True,     # Verificar conexión antes de usar
)

SessionLocal = sessionmaker(bind=engine)

# En app/services/product_service.py
class ProductService:
    def __init__(self):
        self.db = SessionLocal()
    
    def get_product(self, product_id: int):
        """Utiliza conexión persistente del pool"""
        try:
            product = self.db.query(Product).filter(
                Product.id == product_id
            ).first()
            return product
        except Exception as e:
            logger.error(f"Error en BD: {e}")
            raise
        finally:
            self.db.close()  # Retorna conexión al pool
```

### Conexión a BD en Lambda (RDS Proxy)
```python
# lambda_function.py - Lambda con RDS Proxy
import psycopg2
import os
import json
from datetime import datetime

# RDS Proxy actúa como intermediario
DB_ENDPOINT = os.environ['DB_ENDPOINT']  # myproxy.proxy-123.us-east-1.rds.amazonaws.com
DB_NAME = os.environ['DB_NAME']
DB_USER = os.environ['DB_USER']
DB_PASSWORD = os.environ['DB_PASSWORD']

# Crear conexión (RDS Proxy maneja el pooling)
def get_db_connection():
    return psycopg2.connect(
        host=DB_ENDPOINT,
        port=5432,
        database=DB_NAME,
        user=DB_USER,
        password=DB_PASSWORD,
        connect_timeout=5
    )

def lambda_handler(event, context):
    """Lambda que accede RDS via RDS Proxy"""
    
    logger.info(f"Event: {json.dumps(event)}")
    
    conn = None
    try:
        conn = get_db_connection()
        
        user_id = event['pathParameters']['user_id']
        product = get_product_from_db(conn, user_id)
        
        return {
            'statusCode': 200,
            'body': json.dumps(product)
        }
    
    except psycopg2.Error as e:
        logger.error(f"DB error: {e}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': 'Database error'})
        }
    
    finally:
        if conn:
            conn.close()  # RDS Proxy retorna a pool

def get_product_from_db(conn, product_id):
    """Ejecuta query en RDS"""
    cur = conn.cursor()
    try:
        cur.execute(
            "SELECT id, name, price FROM products WHERE id = %s",
            (product_id,)
        )
        row = cur.fetchone()
        
        if row:
            return {
                'id': row[0],
                'name': row[1],
                'price': float(row[2])
            }
        return None
    finally:
        cur.close()
```

**SAM Template con RDS Proxy**:
```yaml
Resources:
  LambdaFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: python3.11
      Environment:
        Variables:
          DB_ENDPOINT: !GetAtt DBProxy.Endpoint
          DB_NAME: myapp
          DB_USER: !Sub '{{resolve:secretsmanager:db-user:SecretString:username}}'
          DB_PASSWORD: !Sub '{{resolve:secretsmanager:db-password:SecretString:password}}'
      VpcConfig:
        SecurityGroupIds:
          - !Ref LambdaSecurityGroup
        SubnetIds:
          - !Ref PrivateSubnet1
          - !Ref PrivateSubnet2

  DBProxy:
    Type: AWS::RDS::DBProxy
    Properties:
      DBProxyName: myapp-proxy
      RoleArn: !GetAtt ProxyRole.Arn
      EngineFamily: POSTGRESQL
      TargetGroupName: default
      MaxIdleConnectionsPercent: 100
      MaxConnectionsPercent: 100
      ConnectionBorrowTimeout: 120
      SessionPinningFilters:
        - "EXCLUDE_VARIABLE_SETS"
      Auth:
        - AuthScheme: SECRETS
          SecretArn: !GetAtt DBSecret.Arn
          IAMAuth: DISABLED
      DBProxyEndpointName: default
      RequireTLS: false
      Targets:
        DBInstanceIdentifiers:
          - !Ref RDSDatabase
```

## 🔹 Práctica Paso a Paso

### 1. Instalar herramientas
```bash
# PostgreSQL localmente (macOS)
brew install postgresql@15
brew services start postgresql@15

# O con Docker
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=password \
  -p 5432:5432 \
  postgres:15

# Python packages
pip install sqlalchemy psycopg2-binary boto3
```

### 2. Crear modelos con SQLAlchemy
Crea `app/models/database.py`:
```python
from sqlalchemy import create_engine, Column, Integer, String, Float, DateTime, ForeignKey, Boolean
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, relationship
from datetime import datetime
import os

DATABASE_URL = os.getenv(
    "DATABASE_URL",
    "postgresql://postgres:password@localhost/myapp"
)

engine = create_engine(DATABASE_URL, echo=True)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    name = Column(String(100), nullable=False)
    email = Column(String(100), unique=True, nullable=False)
    password_hash = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    # Relación
    orders = relationship("Order", back_populates="user")
    
    def __repr__(self):
        return f"<User {self.email}>"

class Product(Base):
    __tablename__ = "products"
    
    id = Column(Integer, primary_key=True)
    name = Column(String(100), nullable=False)
    price = Column(Float, nullable=False)
    description = Column(String(500))
    stock = Column(Integer, default=0)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    # Relación
    order_items = relationship("OrderItem", back_populates="product")

class Order(Base):
    __tablename__ = "orders"
    
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    total = Column(Float, nullable=False)
    status = Column(String(20), default="pending")  # pending, confirmed, shipped
    created_at = Column(DateTime, default=datetime.utcnow)
    
    # Relaciones
    user = relationship("User", back_populates="orders")
    items = relationship("OrderItem", back_populates="order")

class OrderItem(Base):
    __tablename__ = "order_items"
    
    id = Column(Integer, primary_key=True)
    order_id = Column(Integer, ForeignKey("orders.id"), nullable=False)
    product_id = Column(Integer, ForeignKey("products.id"), nullable=False)
    quantity = Column(Integer, nullable=False)
    price = Column(Float, nullable=False)
    
    # Relaciones
    order = relationship("Order", back_populates="items")
    product = relationship("Product", back_populates="order_items")

# Crear tablas
def init_db():
    Base.metadata.create_all(bind=engine)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### 3. Repository pattern para acceso a datos
Crea `app/repositories/user_repository.py`:
```python
from sqlalchemy.orm import Session
from sqlalchemy.exc import IntegrityError
from app.models.database import User
from app.utils.logger import setup_logger

logger = setup_logger(__name__)

class UserRepository:
    def __init__(self, db: Session):
        self.db = db
    
    def create(self, name: str, email: str, password_hash: str) -> User:
        """Crea nuevo usuario"""
        try:
            user = User(
                name=name,
                email=email,
                password_hash=password_hash
            )
            self.db.add(user)
            self.db.commit()
            self.db.refresh(user)
            logger.info(f"Usuario creado: {user.id}")
            return user
        except IntegrityError:
            self.db.rollback()
            logger.error(f"Email duplicado: {email}")
            raise ValueError(f"Email {email} ya existe")
        except Exception as e:
            self.db.rollback()
            logger.exception("Error creando usuario")
            raise
    
    def get_by_id(self, user_id: int) -> User | None:
        """Obtiene usuario por ID"""
        return self.db.query(User).filter(User.id == user_id).first()
    
    def get_by_email(self, email: str) -> User | None:
        """Obtiene usuario por email"""
        return self.db.query(User).filter(User.email == email).first()
    
    def get_all(self, skip: int = 0, limit: int = 10) -> list[User]:
        """Lista usuarios con paginación"""
        return self.db.query(User)\
            .offset(skip)\
            .limit(limit)\
            .all()
    
    def count(self) -> int:
        """Cuenta total de usuarios"""
        return self.db.query(User).count()
    
    def update(self, user_id: int, **kwargs) -> User | None:
        """Actualiza usuario"""
        try:
            user = self.get_by_id(user_id)
            if not user:
                return None
            
            for key, value in kwargs.items():
                if hasattr(user, key):
                    setattr(user, key, value)
            
            self.db.commit()
            self.db.refresh(user)
            logger.info(f"Usuario actualizado: {user_id}")
            return user
        except Exception as e:
            self.db.rollback()
            logger.exception(f"Error actualizando usuario {user_id}")
            raise
    
    def delete(self, user_id: int) -> bool:
        """Elimina usuario"""
        try:
            user = self.get_by_id(user_id)
            if not user:
                return False
            
            self.db.delete(user)
            self.db.commit()
            logger.info(f"Usuario eliminado: {user_id}")
            return True
        except Exception as e:
            self.db.rollback()
            logger.exception(f"Error eliminando usuario {user_id}")
            raise
```

### 4. Manejo de errores de conexión
Crea `app/utils/db.py`:
```python
import time
from sqlalchemy import create_engine, event, exc
from sqlalchemy.pool import Pool
from app.utils.logger import setup_logger

logger = setup_logger(__name__)

def create_engine_with_retry(url, max_retries=3):
    """Crea motor con reintentos de conexión"""
    
    for attempt in range(max_retries):
        try:
            engine = create_engine(url)
            
            # Probar conexión
            with engine.connect() as conn:
                logger.info("Conexión a BD exitosa")
                return engine
                
        except exc.OperationalError as e:
            logger.warning(f"Intento {attempt + 1}: {str(e)}")
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Backoff exponencial
    
    raise Exception("No se pudo conectar a la BD después de reintentos")

def setup_connection_listeners(engine):
    """Configura listeners para eventos de conexión"""
    
    @event.listens_for(Pool, "connect")
    def receive_connect(dbapi_conn, connection_record):
        logger.debug("Conexión establecida")
    
    @event.listens_for(Pool, "checkin")
    def receive_checkin(dbapi_conn, connection_record):
        logger.debug("Conexión retornada al pool")
    
    @event.listens_for(Pool, "checkout")
    def receive_checkout(dbapi_conn, connection_record, connection_proxy):
        logger.debug("Conexión sacada del pool")
```

### 5. Migraciones con Alembic
```bash
# Instalar
pip install alembic

# Inicializar migraciones
alembic init migrations

# Crear migración
alembic revision --autogenerate -m "Agregar tabla users"

# Aplicar migraciones
alembic upgrade head

# Ver historial
alembic history
```

### 6. Testing con BD en memoria
Crea `tests/test_db.py`:
```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.models.database import Base, User
from app.repositories.user_repository import UserRepository

# BD en memoria para tests
TEST_SQLALCHEMY_DATABASE_URL = "sqlite:///:memory:"

@pytest.fixture
def db():
    """Crea BD de test"""
    engine = create_engine(TEST_SQLALCHEMY_DATABASE_URL)
    Base.metadata.create_all(bind=engine)
    
    SessionLocal = sessionmaker(bind=engine)
    db = SessionLocal()
    
    yield db
    
    db.close()

def test_create_user(db):
    """Test crear usuario"""
    repo = UserRepository(db)
    user = repo.create(
        name="Juan",
        email="juan@example.com",
        password_hash="hashed_password"
    )
    
    assert user.id is not None
    assert user.email == "juan@example.com"

def test_get_user(db):
    """Test obtener usuario"""
    repo = UserRepository(db)
    user = repo.create(
        name="Juan",
        email="juan@example.com",
        password_hash="hashed_password"
    )
    
    retrieved = repo.get_by_id(user.id)
    assert retrieved.name == "Juan"

def test_duplicate_email(db):
    """Test email duplicado"""
    repo = UserRepository(db)
    repo.create("Juan", "juan@example.com", "pass1")
    
    with pytest.raises(ValueError):
        repo.create("Juan2", "juan@example.com", "pass2")
```

### 7. Usar DynamoDB localmente
```bash
# Descargar DynamoDB local
wget https://s3-us-west-2.amazonaws.com/dynamodb-local/dynamodb_local_latest.zip
unzip dynamodb_local_latest.zip

# Ejecutar
java -Djava.library.path=./DynamoDBLocal_lib -jar DynamoDBLocal.jar

# Conectar desde Python
import boto3

dynamodb = boto3.resource(
    "dynamodb",
    endpoint_url="http://localhost:8000",  # Local
    region_name="us-east-1"
)
```

### 8. Health check para BD
```python
from sqlalchemy import text
from app.utils.logger import setup_logger

logger = setup_logger(__name__)

def check_database_health(db):
    """Verifica si BD está conectada"""
    try:
        db.execute(text("SELECT 1"))
        logger.info("BD está saludable")
        return True
    except Exception as e:
        logger.error(f"BD no accesible: {str(e)}")
        return False
```

## 🔹 Ejercicio Práctico

### Tarea 1: Diseñar schema
1. Dibuja diagrama E-R para tu aplicación
2. Define al menos 3 tablas con relaciones
3. Identifica tipos de datos para cada columna

### Tarea 2: Crear modelos SQLAlchemy
1. Define modelos para 3 entidades
2. Especifica relaciones
3. Crea modelos Pydantic para requests/responses

### Tarea 3: CRUD completo
1. Implementa repository para 2 modelos
2. Crea tests unitarios
3. Prueba con Postman o curl

### Tarea 4: Manejo de errores
1. Identifica 3 puntos de fallo potencial
2. Implementa try-except apropiado
3. Agrega logging en cada operación

## 🔹 Verificación de Aprendizaje

- [ ] Entiendo diferencia entre RDS y DynamoDB
- [ ] Puedo diseñar un schema relacional
- [ ] He usado SQLAlchemy para CRUD
- [ ] Sé cómo manejar transacciones
- [ ] Puedo testear código que accede BD
- [ ] Entiendo connection pooling y timeouts

## 🔹 Recursos Complementarios

- [PostgreSQL documentation](https://www.postgresql.org/docs/)
- [SQLAlchemy ORM](https://docs.sqlalchemy.org/en/20/orm/)
- [Alembic migrations](https://alembic.sqlalchemy.org/)
- [Boto3 DynamoDB](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/dynamodb.html)
- [AWS RDS best practices](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/BestPractices.md)
