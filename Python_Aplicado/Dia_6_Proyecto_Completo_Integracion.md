# Día 6 – Proyecto Completo: Integración End-to-End

## 🔹 Introducción

Este README integra **todo lo aprendido** en los 5 días anteriores construyendo una aplicación real: **Sistema de Ordenes de E-commerce**.

Abordaremos:
1. Lectura y comprensión de código existente
2. Manejo de errores y logging
3. Integración con APIs externas
4. Acceso a base de datos
5. Despliegue en producción

**Escenario**: Necesitamos entender, mejorar y desplegar una aplicación que ya funciona en staging.

---

## 🔹 Arquitectura del Proyecto

```
Cliente Web
    ↓
Frontend (React/Vue)
    ↓
┌─────────────────────────────────────┐
│     API REST (FastAPI/Flask)        │
│  ├─ POST /orders (crear orden)      │
│  ├─ GET /orders/{id} (obtener)      │
│  ├─ PUT /orders/{id} (actualizar)   │
│  └─ GET /health (verificar)         │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  Servicios de Negocio               │
│  ├─ OrderService                    │
│  ├─ PaymentService                  │
│  └─ NotificationService             │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  Dependencias Externas              │
│  ├─ RDS PostgreSQL (BD)             │
│  ├─ Stripe (pagos)                  │
│  ├─ SendGrid (emails)               │
│  └─ S3 (imágenes)                   │
└─────────────────────────────────────┘
    ↓
CloudWatch Logs + Metrics
```

---

## 🔹 Estructura del Proyecto

```
ecommerce-app/
├── main.py                  # Punto de entrada
├── config.py                # Configuración por ambiente
├── requirements.txt         # Dependencias
├── .env.example             # Variables de entorno
├── Dockerfile               # Para EC2/ECS
├── lambda_function.py       # Para AWS Lambda
├── docker-compose.yml       # Local development
│
├── app/
│   ├── __init__.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── database.py      # SQLAlchemy models
│   │   ├── user.py
│   │   ├── product.py
│   │   └── order.py
│   │
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── user.py          # Pydantic schemas
│   │   ├── product.py
│   │   └── order.py
│   │
│   ├── repositories/
│   │   ├── __init__.py
│   │   ├── user_repository.py
│   │   ├── product_repository.py
│   │   └── order_repository.py
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── order_service.py      # Lógica de negocio
│   │   ├── payment_service.py    # Integración Stripe
│   │   └── notification_service.py  # Emails
│   │
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── orders.py
│   │   ├── products.py
│   │   └── users.py
│   │
│   └── utils/
│       ├── __init__.py
│       ├── logger.py        # Logging centralizado
│       ├── db.py            # Conexión a BD
│       ├── http_client.py   # Cliente HTTP
│       └── decorators.py    # Decorators útiles
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py          # Fixtures pytest
│   ├── test_models.py
│   ├── test_services.py
│   ├── test_api.py
│   └── test_integration.py
│
├── migrations/              # Alembic para BD
│   ├── alembic.ini
│   └── versions/
│
├── scripts/
│   ├── seed_db.py          # Datos de prueba
│   ├── deploy.sh           # EC2 deploy
│   └── rollback.sh         # EC2 rollback
│
└── docs/
    ├── README.md           # Documentación
    ├── API.md              # API documentation
    └── DEPLOYMENT.md       # Guía de despliegue
```

---

## 🔹 Paso 1: Lectura y Análisis del Código Existente

### 1.1 Entender el flujo de una orden

```python
# Flujo: Cliente → Crear Orden → Procesar Pago → Enviar Email

# app/models/order.py
from sqlalchemy import Column, Integer, String, Float, DateTime, ForeignKey, Enum
from app.models.database import Base
from datetime import datetime
import enum

class OrderStatus(str, enum.Enum):
    PENDING = "pending"
    PAID = "paid"
    PROCESSING = "processing"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class Order(Base):
    __tablename__ = "orders"
    
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    status = Column(String(20), default=OrderStatus.PENDING.value)
    total = Column(Float, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    user = relationship("User", back_populates="orders")
    items = relationship("OrderItem", back_populates="order")
```

### 1.2 Analizar servicio de órdenes

```python
# app/services/order_service.py
from app.repositories.order_repository import OrderRepository
from app.repositories.product_repository import ProductRepository
from app.services.payment_service import PaymentService
from app.services.notification_service import NotificationService
from app.utils.logger import setup_logger

logger = setup_logger(__name__)

class OrderService:
    def __init__(self, db):
        self.db = db
        self.order_repo = OrderRepository(db)
        self.product_repo = ProductRepository(db)
        self.payment_service = PaymentService()
        self.notification_service = NotificationService()
    
    def create_order(self, user_id: int, items: list) -> dict:
        """
        Crear orden y procesar pago
        Flujo: Validar → Calcular total → Crear orden → Procesar pago → Notificar
        """
        try:
            # 1. Validar productos y stock
            logger.info(f"Validando {len(items)} items para usuario {user_id}")
            total = 0
            order_items = []
            
            for item in items:
                product = self.product_repo.get_by_id(item['product_id'])
                if not product:
                    raise ValueError(f"Producto {item['product_id']} no existe")
                
                if product.stock < item['quantity']:
                    raise ValueError(f"Stock insuficiente para {product.name}")
                
                item_total = product.price * item['quantity']
                total += item_total
                order_items.append({
                    'product': product,
                    'quantity': item['quantity'],
                    'price': product.price
                })
            
            # 2. Crear orden (aún sin pagar)
            logger.info(f"Creando orden con total ${total}")
            order = self.order_repo.create(
                user_id=user_id,
                total=total,
                status="pending"
            )
            
            # 3. Procesar pago (transacción crítica)
            try:
                logger.info(f"Procesando pago de orden {order.id}")
                payment_result = self.payment_service.process_payment(
                    order_id=order.id,
                    amount=total,
                    user_id=user_id
                )
                
                # 4. Actualizar estado a PAID
                self.order_repo.update(order.id, status="paid")
                logger.info(f"Pago procesado exitosamente para orden {order.id}")
                
                # 5. Descontar stock
                for item in order_items:
                    self.product_repo.decrease_stock(
                        item['product'].id,
                        item['quantity']
                    )
                
                # 6. Enviar confirmación (async)
                self.notification_service.send_order_confirmation(
                    user_id=user_id,
                    order=order
                )
                
                logger.info(f"Orden {order.id} completada exitosamente")
                return {"id": order.id, "status": "paid", "total": total}
            
            except Exception as payment_error:
                # Si el pago falla, cancelar orden
                logger.error(f"Error en pago: {payment_error}")
                self.order_repo.update(order.id, status="cancelled")
                raise
        
        except ValueError as e:
            logger.warning(f"Error de validación: {e}")
            raise
        except Exception as e:
            logger.exception(f"Error creando orden para usuario {user_id}")
            raise
```

### 1.3 Revisar rutas (endpoints)

```python
# app/routes/orders.py
from fastapi import APIRouter, Depends, HTTPException, status
from app.schemas.order import OrderCreate, OrderResponse
from app.services.order_service import OrderService
from app.models.database import SessionLocal
from app.utils.logger import setup_logger

orders_bp = APIRouter(prefix="/api/orders", tags=["orders"])
logger = setup_logger(__name__)

@orders_bp.post("/", response_model=OrderResponse, status_code=status.HTTP_201_CREATED)
async def create_order(
    order_data: OrderCreate,
    db = Depends(lambda: SessionLocal())
):
    """Crear nueva orden"""
    try:
        service = OrderService(db)
        result = service.create_order(
            user_id=order_data.user_id,
            items=order_data.items
        )
        logger.info(f"Orden creada: {result['id']}")
        return result
    
    except ValueError as e:
        logger.warning(f"Validación fallida: {e}")
        raise HTTPException(status_code=400, detail=str(e))
    except Exception as e:
        logger.exception("Error creando orden")
        raise HTTPException(status_code=500, detail="Error interno")

@orders_bp.get("/{order_id}", response_model=OrderResponse)
async def get_order(
    order_id: int,
    db = Depends(lambda: SessionLocal())
):
    """Obtener estado de orden"""
    try:
        service = OrderService(db)
        order = service.order_repo.get_by_id(order_id)
        
        if not order:
            raise HTTPException(status_code=404, detail="Orden no encontrada")
        
        return order
    except Exception as e:
        logger.exception(f"Error obteniendo orden {order_id}")
        raise HTTPException(status_code=500, detail="Error interno")
```

---

## 🔹 Paso 2: Integrar Manejo de Errores y Logging

### 2.1 Crear excepciones personalizadas

```python
# app/exceptions.py
class OrderError(Exception):
    """Error base para órdenes"""
    def __init__(self, message, code=500):
        self.message = message
        self.code = code
        super().__init__(self.message)

class OrderValidationError(OrderError):
    """Error de validación"""
    def __init__(self, message):
        super().__init__(message, 400)

class OrderNotFoundError(OrderError):
    """Orden no existe"""
    def __init__(self, order_id):
        super().__init__(f"Orden {order_id} no encontrada", 404)

class PaymentError(OrderError):
    """Error en procesamiento de pago"""
    def __init__(self, message):
        super().__init__(message, 402)  # Payment Required

class InsufficientStockError(OrderValidationError):
    """Stock insuficiente"""
    def __init__(self, product_name, available, requested):
        msg = f"{product_name}: disponible {available}, solicitado {requested}"
        super().__init__(msg)
```

### 2.2 Mejorar servicio con manejo de errores

```python
# app/services/order_service.py (mejorado)
from app.exceptions import (
    OrderValidationError,
    InsufficientStockError,
    PaymentError
)
import logging

logger = logging.getLogger(__name__)

class OrderService:
    def create_order_with_error_handling(self, user_id: int, items: list):
        """Versión mejorada con validaciones exhaustivas"""
        
        transaction_id = str(uuid.uuid4())
        logger.info(f"[{transaction_id}] Iniciando orden para usuario {user_id}")
        
        try:
            # Validar usuario existe
            user = self.user_repo.get_by_id(user_id)
            if not user:
                raise OrderValidationError(f"Usuario {user_id} no encontrado")
            
            # Validar items
            if not items or len(items) == 0:
                raise OrderValidationError("Orden debe tener al menos 1 item")
            
            # Validar cada item
            total = 0
            validated_items = []
            
            for idx, item in enumerate(items):
                if not item.get('product_id') or not item.get('quantity'):
                    raise OrderValidationError(
                        f"Item {idx}: falta product_id o quantity"
                    )
                
                product = self.product_repo.get_by_id(item['product_id'])
                if not product:
                    raise OrderValidationError(
                        f"Producto {item['product_id']} no existe"
                    )
                
                quantity = item['quantity']
                if quantity <= 0:
                    raise OrderValidationError(f"Cantidad debe ser > 0")
                
                if product.stock < quantity:
                    raise InsufficientStockError(
                        product.name,
                        product.stock,
                        quantity
                    )
                
                item_total = product.price * quantity
                total += item_total
                validated_items.append({
                    'product': product,
                    'quantity': quantity
                })
                logger.debug(f"[{transaction_id}] Item validado: {product.name} x{quantity}")
            
            # Crear orden
            logger.info(f"[{transaction_id}] Creando orden con total ${total}")
            order = self.order_repo.create(
                user_id=user_id,
                total=total,
                status="pending"
            )
            logger.info(f"[{transaction_id}] Orden creada: {order.id}")
            
            # Procesar pago (crítico)
            try:
                logger.info(f"[{transaction_id}] Procesando pago de ${total}")
                payment = self.payment_service.process_payment(
                    order_id=order.id,
                    amount=total,
                    user_id=user_id
                )
                logger.info(f"[{transaction_id}] Pago exitoso: {payment['transaction_id']}")
                
            except PaymentError as e:
                logger.error(f"[{transaction_id}] Pago rechazado: {e.message}")
                # Marcar como cancelada
                self.order_repo.update(order.id, status="cancelled")
                raise
            
            # Actualizar estado y descontar stock
            self.order_repo.update(order.id, status="paid")
            
            for item in validated_items:
                self.product_repo.decrease_stock(
                    item['product'].id,
                    item['quantity']
                )
                logger.debug(f"[{transaction_id}] Stock descontado: {item['product'].id}")
            
            # Notificar (no-blocking)
            try:
                self.notification_service.send_order_confirmation(
                    user_id=user_id,
                    order_id=order.id
                )
            except Exception as e:
                # Log pero no fallar la orden
                logger.warning(f"[{transaction_id}] Error enviando email: {e}")
            
            logger.info(f"[{transaction_id}] Orden completada exitosamente")
            return {
                "id": order.id,
                "status": "paid",
                "total": total,
                "transaction_id": transaction_id
            }
        
        except (OrderValidationError, PaymentError) as e:
            logger.warning(f"[{transaction_id}] Error esperado: {e.message}")
            raise
        except Exception as e:
            logger.exception(f"[{transaction_id}] Error inesperado en orden")
            raise OrderError("Error procesando orden")
```

---

## 🔹 Paso 3: Implementar Integraciones Externas

### 3.1 Integración con Stripe (pagos)

```python
# app/services/payment_service.py
import stripe
from app.utils.logger import setup_logger
from app.exceptions import PaymentError
import os

logger = setup_logger(__name__)
stripe.api_key = os.getenv("STRIPE_SECRET_KEY")

class PaymentService:
    def process_payment(self, order_id: int, amount: float, user_id: int):
        """
        Procesar pago con Stripe
        """
        try:
            logger.info(f"Iniciando pago de ${amount} para orden {order_id}")
            
            # Crear payment intent
            intent = stripe.PaymentIntent.create(
                amount=int(amount * 100),  # Stripe usa centavos
                currency="usd",
                metadata={
                    "order_id": order_id,
                    "user_id": user_id
                }
            )
            
            logger.info(f"Payment Intent creado: {intent.id}")
            
            # En producción, el cliente enviaría el stripe token
            # Por ahora simulamos confirmación
            intent = stripe.PaymentIntent.confirm(
                intent.id,
                payment_method="pm_card_visa"  # Test card
            )
            
            if intent.status == "succeeded":
                logger.info(f"Pago confirmado: {intent.id}")
                return {
                    "transaction_id": intent.id,
                    "status": "succeeded",
                    "amount": amount
                }
            else:
                raise PaymentError(f"Pago no procesado: {intent.status}")
        
        except stripe.error.CardError as e:
            logger.error(f"Error de tarjeta: {e.user_message}")
            raise PaymentError(e.user_message)
        except stripe.error.RateLimitError:
            logger.error("Rate limit en Stripe")
            raise PaymentError("Intenta de nuevo en unos momentos")
        except stripe.error.StripeConnectionError:
            logger.error("Error de conexión con Stripe")
            raise PaymentError("Error de conexión")
        except Exception as e:
            logger.exception(f"Error procesando pago: {e}")
            raise PaymentError("Error procesando pago")
```

### 3.2 Integración con SendGrid (emails)

```python
# app/services/notification_service.py
from sendgrid import SendGridAPIClient
from sendgrid.helpers.mail import Mail
from app.utils.logger import setup_logger
import os

logger = setup_logger(__name__)
sg = SendGridAPIClient(os.environ.get('SENDGRID_API_KEY'))

class NotificationService:
    def send_order_confirmation(self, user_id: int, order_id: int):
        """Enviar email de confirmación"""
        try:
            # Obtener datos de usuario y orden
            user = self.user_repo.get_by_id(user_id)
            order = self.order_repo.get_by_id(order_id)
            
            if not user or not order:
                raise ValueError("Usuario u orden no encontrada")
            
            # Construir email
            message = Mail(
                from_email='orders@myapp.com',
                to_emails=user.email,
                subject=f'Confirmar orden #{order.id}',
                html_content=f'''
                <h1>Orden confirmada!</h1>
                <p>Gracias por tu compra.</p>
                <p>Orden: #{order.id}</p>
                <p>Total: ${order.total}</p>
                <p><a href="https://myapp.com/orders/{order.id}">Ver orden</a></p>
                '''
            )
            
            # Enviar
            response = sg.send(message)
            logger.info(f"Email enviado a {user.email} (status {response.status_code})")
        
        except Exception as e:
            logger.exception(f"Error enviando email: {e}")
            raise
```

---

## 🔹 Paso 4: Estructura de BD Completa

### 4.1 Modelos

```python
# app/models/order.py (completo)
from sqlalchemy import Column, Integer, String, Float, DateTime, ForeignKey, Table
from sqlalchemy.orm import relationship
from app.models.database import Base
from datetime import datetime

# Tabla de asociación para Many-to-Many
order_items = Table(
    'order_items',
    Base.metadata,
    Column('order_id', Integer, ForeignKey('orders.id'), primary_key=True),
    Column('product_id', Integer, ForeignKey('products.id'), primary_key=True)
)

class Order(Base):
    __tablename__ = "orders"
    
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    total = Column(Float, nullable=False)
    status = Column(String(20), default="pending")
    stripe_payment_id = Column(String(100), unique=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    user = relationship("User", back_populates="orders")
    products = relationship("Product", secondary=order_items)

class OrderItem(Base):
    __tablename__ = "order_items"
    
    order_id = Column(Integer, ForeignKey("orders.id"), primary_key=True)
    product_id = Column(Integer, ForeignKey("products.id"), primary_key=True)
    quantity = Column(Integer, nullable=False)
    price = Column(Float, nullable=False)  # Precio al momento de la orden
```

---

## 🔹 Paso 5: Despliegue EC2 vs Lambda

### 5.1 Despliegue en EC2

```bash
#!/bin/bash
# scripts/deploy-ec2.sh

set -e
VERSION=$1

docker build -t myregistry/ecommerce:$VERSION .
docker push myregistry/ecommerce:$VERSION

# Actualizar instancias
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name ecommerce-asg \
  --launch-template LaunchTemplateName=ecommerce,Version='$Latest'

echo "✓ Despliegue a EC2 completado: v$VERSION"
```

### 5.2 Despliegue en Lambda

```yaml
# template.yaml para Lambda
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  OrderFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: .
      Handler: lambda_function.lambda_handler
      Runtime: python3.11
      Timeout: 60
      MemorySize: 1024
      Environment:
        Variables:
          DATABASE_URL: !Sub '{{resolve:secretsmanager:db-url:SecretString}}'
          STRIPE_API_KEY: !Sub '{{resolve:secretsmanager:stripe-key:SecretString}}'
      Events:
        CreateOrder:
          Type: Api
          Properties:
            Path: /orders
            Method: post
        GetOrder:
          Type: Api
          Properties:
            Path: /orders/{id}
            Method: get

Outputs:
  ApiEndpoint:
    Value: !Sub 'https://${ServerlessApi}.execute-api.${AWS::Region}.amazonaws.com/Prod'
```

---

## 🔹 Paso 6: Testing Completo

```python
# tests/test_integration.py
import pytest
from fastapi.testclient import TestClient
from app.main import app
from app.models.database import SessionLocal

client = TestClient(app)

@pytest.fixture
def test_user():
    """Crear usuario de test"""
    response = client.post("/api/users", json={
        "name": "Test User",
        "email": "test@example.com"
    })
    return response.json()

@pytest.fixture
def test_product():
    """Crear producto de test"""
    response = client.post("/api/products", json={
        "name": "Test Product",
        "price": 99.99,
        "stock": 100
    })
    return response.json()

def test_complete_order_flow(test_user, test_product):
    """Test del flujo completo de orden"""
    
    # 1. Crear orden
    order_data = {
        "user_id": test_user['id'],
        "items": [
            {
                "product_id": test_product['id'],
                "quantity": 2
            }
        ]
    }
    
    response = client.post("/api/orders", json=order_data)
    assert response.status_code == 201
    order = response.json()
    
    # 2. Verificar orden está pagada
    assert order['status'] == 'paid'
    assert order['total'] == 199.98
    
    # 3. Obtener orden
    response = client.get(f"/api/orders/{order['id']}")
    assert response.status_code == 200
    assert response.json()['id'] == order['id']
    
    # 4. Verificar stock fue descontado
    response = client.get(f"/api/products/{test_product['id']}")
    updated_product = response.json()
    assert updated_product['stock'] == 98  # 100 - 2
```

---

## 🔹 Ejercicio Práctico

### Tarea 1: Ejecutar localmente
```bash
# 1. Clonar proyecto
git clone https://github.com/tu-org/ecommerce-app.git
cd ecommerce-app

# 2. Configurar entorno
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 3. Crear archivo .env
cp .env.example .env
# Editar .env con credenciales

# 4. Inicializar BD
docker-compose up postgres
alembic upgrade head

# 5. Ejecutar aplicación
python main.py

# 6. Probar endpoints
curl -X POST http://localhost:8000/api/orders \
  -H "Content-Type: application/json" \
  -d '{"user_id": 1, "items": [{"product_id": 1, "quantity": 2}]}'
```

### Tarea 2: Debugging de error real
Caso: Usuario reporta que le cobran dos veces

**Investigación**:
```bash
# 1. Ver logs
tail -f logs/app.log | grep "order"

# 2. Verificar stripe
aws dynamodb query --table-name stripe_payments \
  --key-condition-expression "user_id = :uid" \
  --expression-attribute-values '{":uid":{"S":"123"}}'

# 3. Encontrar duplicado
# Ver query a BD para ver si hay dos órdenes con mismo user + fecha cercana
```

### Tarea 3: Mejorar manejo de errores
Identifica 3 puntos donde puede fallar y agrega try-except

### Tarea 4: Agregar nueva funcionalidad
Implementar "cancelar orden" respetando el flujo completo

---

## 🔹 Verificación de Aprendizaje

- [ ] Puedo leer y entender código de proyecto existente
- [ ] Sé cómo agregar logging exhaustivo
- [ ] Puedo integrar con APIs externas (Stripe, SendGrid)
- [ ] Entiendo transacciones en BD
- [ ] Puedo testear flujos completos
- [ ] Sé desplegar tanto en EC2 como Lambda

## 🔹 Recursos Complementarios

- [FastAPI Full Stack Tutorial](https://fastapi.tiangolo.com/deployment/)
- [Stripe Python SDK](https://stripe.com/docs/libraries/python)
- [SendGrid Python Library](https://sendgrid.com/docs/for-developers/sending-email/v3-python-mail-send/)
- [SQLAlchemy Relationships](https://docs.sqlalchemy.org/en/20/orm/basic_relationships.html)
- [Pytest Fixtures](https://docs.pytest.org/en/stable/fixture.html)
