# AWS-Y-PYTHON

# Plan de Trabajo: Upskilling Intruncks

## Semana 1: AWS Free Tier

### Día 1 – 20/01/2026: AWS EC2 - Fundamentos
- **Objetivo**: Comprender qué es EC2, tipos de instancias y despliegue básico en producción.
- **Estrategias**: Creación guiada de instancia, conexión SSH, revisión de estado, configuración de security groups.
- **Ejercicio**: Levantar un servicio simple en Python.
- **Enfoque**: Entorno real, no sandbox.

### Día 2 – 21/01/2026: AWS EC2 - Operación y Troubleshooting
- **Objetivo**: Diagnosticar problemas comunes (CPU, memoria, disco, red).
- **Estrategias**: Revisión de métricas, logs del sistema, simulación de caída de servicio y recuperación.
- **Observaciones**: Usar ejemplos reales de fallas previas.

### Día 3 – 22/01/2026: Route 53 y DNS
- **Objetivo**: Entender resolución DNS y enrutamiento hacia EC2.
- **Estrategias**: Configuración de dominio, registros A/CNAME, pruebas de health check.
- **Observaciones**: Validar con dominios usados en producción.

### Día 4 – 23/01/2026: IAM y Seguridad
- **Objetivo**: Gestionar accesos seguros mediante roles y policies.
- **Estrategias**: Creación de usuarios, roles para EC2, prueba de permisos mínimos, revisión de buenas prácticas.
- **Observaciones**: Enfatizar errores comunes y cómo evitarlos.

### Día 5 – 26/01/2026: CloudWatch y Monitoreo
- **Objetivo**: Monitorear servicios y detectar fallas proactivamente.
- **Estrategias**: Configuración de alarmas, análisis de logs, simulación de alerta por consumo alto.
- **Observaciones**: Revisar alertas reales y su impacto.

---

## Semana 2: Python

### Día 6 – 27/01/2026: Python - Lectura de Proyecto Real
- **Objetivo**: Comprender estructura y flujo de una aplicación existente.
- **Estrategias**: Visión general de Lambda, S3, RDS/DynamoDB y API Gateway: cuándo se usan, cómo se integran con EC2 y flujos típicos.
- **Observaciones**: Usar código de proyectos actuales.

### Día 7 – 28/01/2026: Python - Manejo de Errores y Logs
- **Objetivo**: Implementar manejo de excepciones y logging productivo.
- **Estrategias**: Code walkthrough, explicación de módulos, dependencias, entorno virtual. Refactor para agregar try/except, logging estructurado, revisión de stack traces.
- **Observaciones**: Aplicar sobre código en producción.

### Día 8 – 29/01/2026: Python - Integración con APIs
- **Objetivo**: Consumir y exponer servicios REST.
- **Estrategias**: Uso de requests/FastAPI, pruebas con Postman, manejo de timeouts y errores HTTP.
- **Observaciones**: Validar contra APIs internas reales.

### Día 9 – 30/01/2026: Python - Base de Datos
- **Objetivo**: Conectar y operar con base de datos (RDS o local).
- **Estrategias**: Conexión, consultas básicas, manejo de errores de conexión y transacciones.
- **Observaciones**: Usar bases reales o réplicas.

### Día 10 – 02/02/2026: Despliegue Controlado y Soporte
- **Objetivo**: Desplegar una app Python en EC2 y validar operación.
- **Estrategias**: Build, deploy, verificación, rollback simulado, checklist de soporte operativo. Revisión arquitectónica completa: interacción EC2, Lambda, API Gateway, S3, RDS/DynamoDB, IAM y CloudWatch en un flujo end-to-end.
- **Observaciones**: Simular escenarios de soporte y debugging.
