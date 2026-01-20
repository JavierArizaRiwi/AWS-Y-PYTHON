# Introducción a AWS CLI e Interfaz de AWS

## 🔹 ¿Qué es AWS CLI?
AWS Command Line Interface (CLI) es una herramienta que permite interactuar con los servicios de AWS desde la línea de comandos. Es ideal para automatizar tareas y gestionar recursos de manera eficiente.

### Instalación de AWS CLI

#### En Linux
1. Descarga el instalador:
   ```bash
   curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
   ```
2. Extrae el archivo:
   ```bash
   unzip awscliv2.zip
   ```
3. Instala AWS CLI:
   ```bash
   sudo ./aws/install
   ```
4. Verifica la instalación:
   ```bash
   aws --version
   ```

### Configuración de AWS CLI
1. Ejecuta el comando:
   ```bash
   aws configure
   ```
2. Ingresa las credenciales de tu cuenta AWS:
   - **Access Key ID**: Proporcionada al crear un usuario IAM.
   - **Secret Access Key**: Proporcionada al crear un usuario IAM.
   - **Default region**: Por ejemplo, `us-east-1`.
   - **Output format**: Deja en blanco o elige `json`.

### Comandos básicos de AWS CLI
- Listar instancias EC2:
  ```bash
  aws ec2 describe-instances
  ```
- Crear un bucket S3:
  ```bash
  aws s3 mb s3://mi-nuevo-bucket
  ```
- Subir un archivo a S3:
  ```bash
  aws s3 cp archivo.txt s3://mi-nuevo-bucket
  ```

## 🔹 ¿Qué es la Interfaz de AWS?
La Interfaz de AWS es una consola web que permite gestionar servicios y recursos de AWS de manera visual e intuitiva.

### Navegación básica
1. Accede a la consola en [AWS Console](https://aws.amazon.com/console/).
2. Usa la barra de búsqueda para encontrar servicios como EC2, S3 o IAM.
3. Explora los menús laterales para acceder a configuraciones específicas.

### Funcionalidades clave
- **Dashboard**: Vista general de tus recursos y servicios.
- **Sección de servicios**: Acceso rápido a servicios como EC2, S3, RDS, etc.
- **Notificaciones**: Alertas sobre el estado de tus recursos.

### Ejemplo práctico: Lanzar una instancia EC2
1. Ve a **EC2** desde la consola.
2. Haz clic en **Launch Instance**.
3. Configura los detalles de la instancia (AMI, tipo, almacenamiento, etc.).
4. Revisa y lanza la instancia.

### Ejemplo práctico: Crear un bucket S3
1. Ve a **S3** desde la consola.
2. Haz clic en **Create Bucket**.
3. Ingresa un nombre único para el bucket.
4. Configura las opciones y haz clic en **Create Bucket**.

---

Con esta guía, tendrás el contexto necesario para usar tanto AWS CLI como la interfaz gráfica de AWS. Ambas herramientas son complementarias y te permitirán gestionar tus recursos de manera eficiente.