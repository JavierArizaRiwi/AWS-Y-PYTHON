# Día 2 – EC2 Operación y Troubleshooting

## 🔹 Teoría

### Problemas comunes
- **CPU alta**: Procesos que consumen demasiados recursos.
- **Memoria insuficiente**: Falta de RAM para ejecutar aplicaciones.
- **Disco lleno**: Espacio de almacenamiento agotado.
- **Red saturada**: Problemas de conectividad o ancho de banda.

### Herramientas
- `top`: Monitorea el uso de CPU y memoria.
- `free -m`: Muestra el uso de memoria.
- `df -h`: Verifica el espacio en disco.
- `netstat`: Analiza conexiones de red.

### Logs
- `/var/log/messages`: Logs del sistema.
- `/var/log/secure`: Logs de seguridad.

## 🔹 Conceptos básicos adicionales

### ¿Qué es un sistema operativo?
Un sistema operativo es el software que permite que el hardware y las aplicaciones trabajen juntos. En EC2, puedes elegir sistemas operativos como Amazon Linux, Ubuntu, o Windows Server.

### ¿Qué son los logs?
Los logs son registros que documentan eventos y actividades en tu sistema. Son útiles para diagnosticar problemas y monitorear el rendimiento.

### ¿Qué es el monitoreo en EC2?
El monitoreo en EC2 implica observar el rendimiento y el estado de tu instancia para identificar problemas y optimizar su funcionamiento.

## 🔹 Práctica paso a paso

### 1. Simular carga de CPU
1. Conéctate a tu instancia EC2 mediante SSH.
2. Ejecuta el siguiente comando para generar carga en la CPU:
   ```bash
   yes > /dev/null &
   ```
3. Usa el comando `top` para monitorear el uso de CPU:
   ```bash
   top
   ```
   Observa cómo el proceso `yes` consume recursos.

### 2. Simular disco lleno
1. Crea un archivo grande para llenar el disco:
   ```bash
   fallocate -l 1G testfile
   ```
2. Verifica el espacio en disco:
   ```bash
   df -h
   ```
   Observa cómo el archivo ocupa espacio en el disco.

### 3. Detener procesos
1. Identifica el proceso que genera carga con `top`.
2. Detén el proceso usando su ID:
   ```bash
   kill -9 <PID>
   ```
   Reemplaza `<PID>` con el ID del proceso.

### 4. Crear un script de monitoreo
1. Crea un archivo llamado `monitor.sh`:
   ```bash
   nano monitor.sh
   ```
2. Copia el siguiente código:
   ```bash
   #!/bin/bash
   echo "Uso de CPU:" >> monitoreo.log
   top -b -n1 | grep "Cpu(s)" >> monitoreo.log
   echo "Uso de memoria:" >> monitoreo.log
   free -m >> monitoreo.log
   echo "Espacio en disco:" >> monitoreo.log
   df -h >> monitoreo.log
   ```
3. Haz el script ejecutable:
   ```bash
   chmod +x monitor.sh
   ```
4. Ejecútalo:
   ```bash
   ./monitor.sh
   ```
5. Revisa el archivo `monitoreo.log` para ver los resultados.