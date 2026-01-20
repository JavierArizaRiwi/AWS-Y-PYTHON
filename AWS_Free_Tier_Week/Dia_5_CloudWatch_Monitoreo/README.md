# Día 5 – CloudWatch y Monitoreo

## 🔹 Teoría

### CloudWatch
- Servicio para recopilar métricas, logs y eventos de tus recursos en la nube.

### Alarmas
- Notifican cuando una métrica supera un umbral definido.

### Dashboards
- Visualizan métricas en tiempo real.

## 🔹 Conceptos básicos adicionales

### ¿Qué es una métrica?
Una métrica es un dato que mide el rendimiento o estado de un recurso, como el uso de CPU o la latencia de red.

### ¿Qué es un dashboard?
Un dashboard es una interfaz visual que muestra métricas y gráficos en tiempo real.

## 🔹 Práctica paso a paso

### 1. Configurar una alarma de CPU
1. Ve a la consola de AWS y selecciona **CloudWatch**.
2. Haz clic en **Alarms > Create Alarm**.
3. Selecciona la métrica de tu instancia EC2:
   - **Browse > EC2 > Per-Instance Metrics**.
   - Elige la métrica `CPUUtilization` de tu instancia.
4. Configura la alarma:
   - **Threshold type**: Static.
   - **Whenever CPUUtilization is...**: Greater than 70%.
   - **For...**: 5 consecutive periods.
5. Configura las notificaciones (opcional) y haz clic en **Create Alarm**.

### 2. Simular carga para disparar la alarma
1. Conéctate a tu instancia EC2 mediante SSH.
2. Genera carga en la CPU:
   ```bash
   yes > /dev/null &
   ```
3. Observa cómo la alarma se activa en la consola de CloudWatch.

### 3. Crear un dashboard
1. En la consola de CloudWatch, selecciona **Dashboards > Create Dashboard**.
2. Ingresa un nombre para el dashboard y haz clic en **Create Dashboard**.
3. Agrega widgets:
   - Haz clic en **Add widget**.
   - Selecciona **Line** y elige métricas como `CPUUtilization`.
4. Guarda el dashboard y revisa las métricas en tiempo real.

### 4. Revisar logs en CloudWatch Logs
- Accede a los logs generados por la instancia EC2.

### 5. Documentar respuesta a alertas
- Registra los pasos para identificar y resolver problemas.

## 🔹 Ejercicio adicional

### Crear una métrica personalizada
1. En tu instancia EC2, crea un script para enviar métricas a CloudWatch:
```bash
#!/bin/bash
aws cloudwatch put-metric-data \
--metric-name CustomCPUUsage \
--namespace CustomMetrics \
--value 75
```
2. Haz el script ejecutable:
```bash
chmod +x custom_metric.sh
```
3. Ejecútalo para enviar la métrica.
4. Ve a la consola de CloudWatch y verifica la métrica en la sección **Custom Metrics**.