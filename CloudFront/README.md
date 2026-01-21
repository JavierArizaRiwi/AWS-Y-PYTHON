# AWS CloudFront

AWS CloudFront es un servicio de red de entrega de contenido (CDN) que permite distribuir contenido estático y dinámico a usuarios finales con baja latencia y altas velocidades de transferencia.

## Características Principales

- **Distribución Global:** Utiliza una red de ubicaciones distribuidas globalmente para entregar contenido más cerca de los usuarios.
- **Compatibilidad con Contenido Estático y Dinámico:** Puede entregar tanto archivos estáticos (imágenes, videos, etc.) como contenido dinámico generado por aplicaciones.
- **Integración con Otros Servicios de AWS:** Funciona perfectamente con S3, EC2, Elastic Load Balancing, y Route 53.
- **Seguridad:** Ofrece HTTPS, protección contra ataques DDoS mediante AWS Shield, y soporte para AWS WAF.
- **Optimización de Costos:** Paga solo por lo que usas, sin tarifas iniciales.

## Casos de Uso

- **Entrega de Contenido Estático:** Imágenes, videos, archivos CSS/JS.
- **Streaming de Video:** Transmisión en vivo o bajo demanda.
- **Aceleración de Aplicaciones Web:** Mejora el rendimiento de aplicaciones dinámicas.
- **Seguridad de Contenido:** Protección de contenido mediante HTTPS y políticas de acceso.

## Cómo Funciona

1. **Origen:** Configura un origen (por ejemplo, un bucket de S3 o un servidor HTTP).
2. **Distribución:** CloudFront crea una distribución que define cómo se entregará el contenido.
3. **Puntos de Presencia:** Los usuarios acceden al contenido desde el punto de presencia más cercano.
4. **Caché:** CloudFront almacena en caché el contenido para reducir la carga en el origen y mejorar la velocidad.

## Precios

El costo de CloudFront se basa en:
- **Datos Transferidos:** Cantidad de datos entregados a través de la CDN.
- **Solicitudes:** Número de solicitudes realizadas.
- **Región:** Los precios varían según la región.

Para más detalles, consulta la [página oficial de precios de CloudFront](https://aws.amazon.com/cloudfront/pricing/).

---

### Recursos Adicionales
- [Documentación Oficial de CloudFront](https://docs.aws.amazon.com/cloudfront)
- [Tutoriales de CloudFront](https://aws.amazon.com/cloudfront/getting-started/)