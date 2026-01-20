# Día 1 – AWS EC2 Fundamentos

## 🔹 Teoría

### Qué es EC2
EC2 (Elastic Compute Cloud) es un servicio de cómputo en la nube que permite crear servidores virtuales escalables. Es ideal para ejecutar aplicaciones y servicios en la nube.

### Tipos de instancias
- **Uso general**: t2.micro (incluido en Free Tier).
- **Optimizadas para CPU**: Para cargas de trabajo intensivas en procesamiento.
- **Optimizadas para memoria**: Para aplicaciones que requieren grandes cantidades de memoria.
- **Optimizadas para almacenamiento**: Para aplicaciones que necesitan acceso rápido a datos.

### Free Tier
AWS ofrece 750 horas/mes de uso gratuito de instancias t2.micro durante los primeros 12 meses.

### Security Groups
- Firewalls virtuales que controlan el tráfico entrante y saliente de las instancias.
- Configuración de reglas para permitir o denegar acceso.

## 🔹 Conceptos básicos adicionales

### ¿Qué es la nube?
La nube es un conjunto de servidores distribuidos en diferentes ubicaciones que permiten almacenar y procesar datos de manera remota. AWS es uno de los principales proveedores de servicios en la nube.

### ¿Por qué usar EC2?
- Flexibilidad: Puedes elegir el sistema operativo, la capacidad de almacenamiento y el tipo de instancia.
- Escalabilidad: Ajusta los recursos según las necesidades de tu aplicación.
- Costo: Paga solo por lo que usas.

### ¿Qué es una instancia?
Una instancia es un servidor virtual que puedes configurar y utilizar para ejecutar aplicaciones o servicios.

### ¿Qué es una instancia EC2?
Una instancia EC2 es un servidor virtual en la nube que puedes configurar para ejecutar aplicaciones. Es como tener una computadora en un centro de datos de AWS.

## 🔹 Práctica paso a paso

### 1. Crear cuenta AWS Free Tier
1. Ve a [AWS Free Tier](https://aws.amazon.com/free/) y haz clic en **Create a Free Account**.
2. Ingresa tu correo electrónico y elige una contraseña.
3. Proporciona tus datos personales y de pago (no se te cobrará mientras uses el Free Tier).
4. Verifica tu cuenta con un código enviado a tu teléfono.
5. Accede a la consola de AWS.

### 2. Lanzar instancia EC2
1. En la consola de AWS, selecciona **EC2**.
2. Haz clic en **Launch Instance**.
3. Elige **Amazon Linux 2 AMI**.
4. Selecciona el tipo de instancia **t2.micro** (gratis en el Free Tier).
5. Haz clic en **Review and Launch** y luego en **Launch**.
6. Crea un nuevo par de claves (Key Pair), descárgalo y guárdalo en un lugar seguro.
7. Haz clic en **Launch Instances**.

### 3. Conectarse a la instancia
1. Abre una terminal en tu computadora.
2. Navega al directorio donde guardaste el archivo `.pem`.
3. Conéctate usando el siguiente comando:
   ```bash
   ssh -i ec2-key.pem ec2-user@<IP_PUBLICA>
   ```
   Reemplaza `<IP_PUBLICA>` con la dirección IP de tu instancia (puedes encontrarla en la consola de EC2).

### 4. Instalar Python y Flask
1. Una vez conectado, instala Python:
   ```bash
   sudo yum install python3 -y
   ```
2. Instala Flask:
   ```bash
   pip3 install flask
   ```

### 5. Crear una aplicación Flask
1. Crea un archivo llamado `app.py`:
   ```bash
   nano app.py
   ```
2. Copia el siguiente código:
   ```python
   from flask import Flask
   app = Flask(__name__)

   @app.route("/")
   def home():
       return "Hola desde EC2 Free Tier!"

   app.run(host="0.0.0.0", port=80)
   ```
3. Guarda y cierra el archivo.

### 6. Ejecutar la aplicación
1. Ejecuta el archivo:
   ```bash
   python3 app.py
   ```
2. Abre un navegador y accede a `http://<IP_PUBLICA>` para ver tu aplicación en funcionamiento.

## 🔹 Ejercicio adicional

### Crear un Security Group
1. Ve a la consola de AWS y selecciona **Security Groups**.
2. Crea un nuevo grupo con las siguientes reglas:
   - **Regla entrante**: Permitir tráfico SSH (puerto 22) desde tu IP.
   - **Regla entrante**: Permitir tráfico HTTP (puerto 80) desde cualquier lugar.
3. Asocia este Security Group a tu instancia EC2.