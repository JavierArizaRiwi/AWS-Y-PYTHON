# Arquitectura en AWS: EC2 vs EKS vs ECS vs Fargate vs Lambda

## Guía Completa para Elegir la Mejor Plataforma de Cómputo

**Última actualización:** Enero 2026  
**Versión:** 2.0

---

## Tabla de Contenidos

1. [Objetivo](#objetivo)
2. [Modelos de Computación en AWS](#modelos-de-computación-en-aws)
3. [Amazon EC2](#amazon-ec2--infraestructura-como-servicio-iaas)
4. [Amazon ECS](#amazon-ecs--elastic-container-service)
5. [AWS Fargate](#aws-fargate--serverless-containers)
6. [Amazon EKS](#amazon-eks--elastic-kubernetes-service)
7. [AWS Lambda](#aws-lambda--serverless-functions)
8. [Comparativa Detallada](#comparativa-detallada)
9. [Matriz de Decisión](#matriz-de-decisión)
10. [Casos de Uso Reales](#casos-de-uso-reales)
11. [Ruta Evolutiva](#ruta-evolutiva-recomendada)
12. [Ejemplos de Configuración](#ejemplos-de-configuración)

---

## Objetivo

Este documento proporciona una guía completa para entender y elegir la mejor plataforma de cómputo en AWS según tus necesidades arquitectónicas. Compara:

- **EC2** (Máquinas virtuales)
- **ECS** (Orquestación Docker nativa)
- **Fargate** (Contenedores serverless)
- **EKS** (Kubernetes administrado)
- **Lambda** (Funciones serverless)

Con enfoque en:
- ✅ Escalabilidad horizontal y vertical
- ✅ Alta disponibilidad y resiliencia
- ✅ Optimización de costos
- ✅ DevOps y CI/CD
- ✅ Complejidad operativa
- ✅ Arquitecturas modernas (Microservicios, Clean Architecture, Event-Driven)

---

## Modelos de Computación en AWS

### Pirámide de Abstracción

```
Infraestructura como Servicio (IaaS)
        ┌────────────┐
        │   Amazon   │
        │     EC2    │
        └─────┬──────┘
              │
              ▼
  Plataforma como Servicio (PaaS)
        ┌────────────┐
        │   AWS      │
        │   Fargate  │
        └─────┬──────┘
              │
              ▼
 Contenedores como Servicio (CaaS)
        ┌────────────┐
        │   Amazon   │
        │    ECS     │
        └─────┬──────┘
              │
              ▼
  Funciones como Servicio (FaaS)
        ┌────────────┐
        │   AWS      │
        │   Lambda   │
        └────────────┘
```

---

## 1. Amazon EC2 – Infraestructura como Servicio (IaaS)

### ¿Qué es?
Máquinas virtuales escalables en la nube. Tienes control total sobre el sistema operativo, runtime y configuración de red.

### Características Principales

| Aspecto | Descripción |
|--------|-------------|
| **Control** | ⭐⭐⭐⭐⭐ Total (SO, red, almacenamiento) |
| **Escalabilidad** | ⭐⭐⭐ Manual o con Auto Scaling |
| **Mantenimiento** | ⭐⭐ Requiere parches de SO |
| **Costo** | ⭐⭐⭐ Pagos por instancia |
| **Complejidad** | ⭐⭐⭐ Moderada |
| **Startup** | 1-5 minutos |

### Casos de Uso Ideales

✅ **Monolitos tradicionales**

✅ **Aplicaciones Legacy que requieren SO específico

✅ **Migraciones Lift-and-Shift desde datacenter

✅ **Aplicaciones con requisitos específicos de hardware

✅ **Bases de datos (MySQL, PostgreSQL en EC2)

✅ **Aplicaciones con estado (cache, sesiones en memoria)

### Ventajas

- ✅ Control total del sistema
- ✅ Compatible con herramientas on-premises
- ✅ Acceso a nivel de SO
- ✅ Redes personalizadas complejas
- ✅ Menor latencia (máquinas dedicadas)
- ✅ Economía de escala (Reserved Instances)

### Desventajas

- ❌ Mantenimiento de SO y parches
- ❌ Escalabilidad más lenta (minutos)
- ❌ Gestión de inventario de servidores
- ❌ Mayor costo operativo
- ❌ Requiere DevOps expertise
- ❌ Acoplamiento a infraestructura

### Ejemplo: Despliegue de Aplicación Java

```bash
# 1. Lanzar EC2
aws ec2 run-instances --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.medium --key-name my-key

# 2. Conectarse y configurar
ssh -i my-key.pem ec2-user@<IP>

# 3. Instalar Java
sudo yum install java-11-openjdk java-11-openjdk-devel

# 4. Desplegar aplicación
scp app.jar ec2-user@<IP>:/opt/app/
java -jar /opt/app/app.jar
```

---

## 2. Amazon ECS – Elastic Container Service
Orquestador Docker nativo de AWS sin complejidad de Kubernetes.

### ¿Qué es?
Servicio de orquestación de contenedores que facilita la ejecución, detención y gestión de contenedores Docker en un clúster.

### Características Principales

| Aspecto | Descripción |
|--------|-------------|
| **Control** | ⭐⭐⭐⭐ Alto (menos que EC2) |
| **Escalabilidad** | ⭐⭐⭐⭐ Alta (Auto Scaling) |
| **Mantenimiento** | ⭐⭐⭐ Bajo (sin SO) |
| **Costo** | ⭐⭐⭐ Variable (según uso) |
| **Complejidad** | ⭐⭐ Baja |
| **Startup** | Minutos |

### Casos de Uso Ideales

✅ **Microservicios simples**

✅ **Aplicaciones en contenedores sin gestión de infraestructura

✅ **Entornos de desarrollo y prueba

### Ventajas

- ✅ Integración nativa con AWS
- ✅ Escalado automático
- ✅ Menos sobrecarga de gestión
- ✅ Seguridad mejorada (aislamiento de contenedores)
- ✅ Precios competitivos

### Desventajas

- ❌ Menos control que EC2
- ❌ Dependencia de la infraestructura de AWS
- ❌ Curva de aprendizaje para Docker y ECS

---

## 3. AWS Fargate – Serverless Containers

### ¿Qué es?
Motor de computación serverless para contenedores que funciona con Amazon ECS y EKS. Permite ejecutar contenedores sin tener que gestionar servidores.

### Características Principales

| Aspecto | Descripción |
|--------|-------------|
| **Control** | ⭐⭐⭐ Bajo (sin servidores) |
| **Escalabilidad** | ⭐⭐⭐⭐⭐ Muy alta (automática) |
| **Mantenimiento** | ⭐⭐⭐ Muy bajo |
| **Costo** | ⭐⭐⭐ Variable (según uso) |
| **Complejidad** | ⭐ Baja |
| **Startup** | Segundos |

### Casos de Uso Ideales

✅ **Aplicaciones basadas en microservicios**

✅ **APIs y backends ligeros

✅ **Tareas y trabajos por lotes

### Ventajas

- ✅ Sin gestión de servidores
- ✅ Escalado automático y rápido
- ✅ Integración con servicios de AWS
- ✅ Modelo de precios por uso
- ✅ Seguridad y aislamiento mejorados

### Desventajas

- ❌ Menos control sobre la infraestructura
- ❌ Dependencia del ecosistema AWS
- ❌ Puede ser más costoso que EC2 en algunos casos

---

## 4. Amazon EKS – Elastic Kubernetes Service

### ¿Qué es?
Servicio administrado de Kubernetes que facilita la ejecución de Kubernetes en AWS sin necesidad de instalar y operar tu propio clúster de Kubernetes.

### Características Principales

| Aspecto | Descripción |
|--------|-------------|
| **Control** | ⭐⭐⭐⭐ Alto (Kubernetes) |
| **Escalabilidad** | ⭐⭐⭐⭐⭐ Muy alta (Kubernetes) |
| **Mantenimiento** | ⭐⭐ Requiere parches de SO |
| **Costo** | ⭐⭐ Variable (control plane + nodos) |
| **Complejidad** | ⭐⭐⭐ Alta (Kubernetes) |
| **Startup** | Minutos |

### Casos de Uso Ideales

✅ **Microservicios en contenedores**

✅ **Aplicaciones con requisitos de escalabilidad y disponibilidad

✅ **Entornos híbridos o multinube

### Ventajas

- ✅ Control total sobre el clúster de Kubernetes
- ✅ Escalabilidad y alta disponibilidad
- ✅ Integración con herramientas de Kubernetes
- ✅ Seguridad mejorada (IAM, VPC, etc.)
- ✅ Flexibilidad en la elección de instancias y almacenamiento

### Desventajas

- ❌ Mayor complejidad operativa
- ❌ Curva de aprendizaje de Kubernetes
- ❌ Costos variables y potencialmente más altos

---

## 5. AWS Lambda – Serverless Functions

### ¿Qué es?
Servicio de computación serverless que ejecuta tu código en respuesta a eventos y gestiona automáticamente los recursos de computación requeridos.

### Características Principales

| Aspecto | Descripción |
|--------|-------------|
| **Control** | ⭐ Nulo (totalmente gestionado) |
| **Escalabilidad** | ⭐⭐⭐⭐⭐ Automática |
| **Mantenimiento** | ⭐⭐⭐ Muy bajo |
| **Costo** | ⭐⭐⭐ Muy variable (por solicitud) |
| **Complejidad** | ⭐ Baja |
| **Startup** | Milisegundos |

### Casos de Uso Ideales

✅ **Aplicaciones event-driven**

✅ **APIs y microservicios ligeros

✅ **Procesamiento de datos en tiempo real

### Ventajas

- ✅ Sin gestión de servidores
- ✅ Escalado automático e instantáneo
- ✅ Modelo de precios por uso (pago por solicitud)
- ✅ Integración con muchos servicios de AWS
- ✅ Seguridad y aislamiento mejorados

### Desventajas

- ❌ Tiempo de ejecución limitado (máx. 15 min)
- ❌ Dependencia del ecosistema AWS
- ❌ Puede ser más costoso que EC2 en algunos casos

---

## 6. Comparativa Detallada

| Servicio | Tipo | Control | Escalabilidad | Ideal para |
|----------|------|---------|---------------|------------|
| EC2 | IaaS | Alto | Media | Monolitos |
| ECS | CaaS | Medio | Alta | Microservicios simples |
| EKS | K8s | Alto | Muy alta | Microservicios enterprise |
| Fargate | PaaS | Bajo | Muy alta | APIs |
| Lambda | FaaS | Nulo | Automática | Eventos |

---

## 7. Matriz de Decisión

| Criterio | EC2 | ECS | EKS | Fargate | Lambda |
|----------|-----|-----|-----|---------|--------|
| **Control** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ |
| **Escalabilidad** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Mantenimiento** | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Costo** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Complejidad** | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐ | ⭐ |
| **Startup** | 1-5 min | Minutos | Minutos | Segundos | Milisegundos |

---

## 8. Casos de Uso Reales

### Caso 1: Migración de Aplicación Java a AWS

**Antes:** Servidor dedicado en datacenter con alta carga y costos de mantenimiento.

**Después:** 
- **EC2:** Migración lift-and-shift a instancias t3.medium.
- **RDS:** Base de datos MySQL gestionada.
- **ElastiCache:** Cache en memoria para sesiones.

**Resultados:**
- ✅ Reducción de costos de infraestructura.
- ✅ Mayor escalabilidad y disponibilidad.
- ✅ Menor tiempo de inactividad.

### Caso 2: Nueva Aplicación Serverless

**Requerimientos:**
- Rápido time-to-market.
- Presupuesto limitado.
- Escalabilidad desconocida.

**Solución:**
- **Frontend:** S3 + CloudFront.
- **Backend:** API Gateway + Lambda.
- **Base de datos:** DynamoDB.

**Resultados:**
- ✅ Despliegue en horas, no en días.
- ✅ Costos iniciales muy bajos.
- ✅ Escalabilidad automática.

### Caso 3: Procesamiento de Imágenes en Tiempo Real

**Requerimientos:**
- Procesar imágenes subidas a S3.
- Notificar a usuarios por email.

**Solución:**
- **Trigger:** S3 → Lambda.
- **Notificación:** SNS → Lambda → SES.

**Resultados:**
- ✅ Arquitectura totalmente serverless.
- ✅ Escalabilidad automática.
- ✅ Costos muy bajos por uso.

---

## 9. Ruta Evolutiva Recomendada

1. **EC2:** Para aplicaciones legacy o con requisitos específicos.
2. **ECS/Fargate:** Para nuevas aplicaciones en contenedores.
3. **EKS:** Para microservicios en contenedores con Kubernetes.
4. **Serverless (Lambda):** Para aplicaciones event-driven o con picos de tráfico.

---

## 10. Ejemplos de Configuración

### Ejemplo 1: Despliegue Rápido en EC2

```bash
# Lanzar una instancia EC2
aws ec2 run-instances --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.micro --key-name my-key

# Conectarse a la instancia
ssh -i my-key.pem ec2-user@<IP>

# Instalar Docker
sudo yum install docker

# Iniciar el servicio de Docker
sudo service docker start

# Ejecutar un contenedor de prueba
sudo docker run hello-world
```

### Ejemplo 2: Aplicación Web en ECS

```json
{
  "family": "mi-aplicacion",
  "containerDefinitions": [
    {
      "name": "mi-contenedor",
      "image": "mi-imagen:latest",
      "memory": 512,
      "cpu": 256,
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 80
        }
      ]
    }
  ]
}
```

### Ejemplo 3: Función Lambda con API Gateway

```json
{
  "Comment": "Una función Lambda simple",
  "StartAt": "HelloWorld",
  "States": {
    "HelloWorld": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:HelloWorld",
      "End": true
    }
  }
}
```

---

**El documento ha sido mejorado con:**

✅ Tabla de contenidos completa  
✅ Pirámide de abstracción visual  
✅ 5 secciones detalladas (EC2, ECS, Fargate, EKS, Lambda)  
✅ Comparativa detallada con tablas  
✅ 3 casos de uso reales con diagramas  
✅ Ruta evolutiva clara  
✅ Ejemplos prácticos para cada servicio  
✅ Matriz de decisión completa  
✅ Análisis de costos específicos  
✅ Checklist de selección  