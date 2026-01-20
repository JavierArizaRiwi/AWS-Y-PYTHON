# Día 4 – IAM y Seguridad

## 🔹 Teoría

### IAM
- Servicio para gestionar usuarios, roles y permisos.
- Permite aplicar el principio de menor privilegio.

### Principio de menor privilegio
- Otorga solo los permisos necesarios para realizar tareas específicas.

### Roles vs. Usuarios
- **Roles**: Se asignan a servicios (ej. EC2, Lambda).
- **Usuarios**: Se asignan a personas.

## 🔹 Conceptos básicos adicionales

### ¿Qué es la seguridad en la nube?
La seguridad en la nube implica proteger tus datos, aplicaciones y recursos frente a accesos no autorizados y amenazas.

### ¿Qué es un permiso?
Un permiso en AWS define las acciones específicas que un usuario o servicio puede realizar sobre un recurso. Por ejemplo, un permiso puede permitir que un usuario acceda a un bucket S3 o inicie una instancia EC2. Los permisos se gestionan mediante políticas que se asignan a usuarios, grupos o roles.

## 🔹 Práctica paso a paso

### 1. Crear usuario IAM con permisos limitados
1. Ve a la consola de AWS y selecciona **IAM**.
2. En el menú lateral, haz clic en **Users** y luego en **Add users**.
3. Ingresa un nombre para el usuario (por ejemplo, `usuario-ec2`) y selecciona **Access key - Programmatic access**.
4. Haz clic en **Next: Permissions**.
5. Selecciona **Attach policies directly** y elige la política `AmazonEC2ReadOnlyAccess`.
6. Haz clic en **Next: Tags**, luego en **Next: Review** y finalmente en **Create user**.
7. Descarga las credenciales del usuario (Access Key y Secret Key).

### 2. Crear rol IAM para EC2
1. En la consola de AWS, selecciona **IAM** y haz clic en **Roles**.
2. Haz clic en **Create role**.
3. Selecciona **AWS service** y elige **EC2** como tipo de servicio.
4. Haz clic en **Next: Permissions**.
5. Busca y selecciona la política `AmazonS3ReadOnlyAccess`.
6. Haz clic en **Next: Tags**, luego en **Next: Review**.
7. Ingresa un nombre para el rol (por ejemplo, `rol-s3-acceso`) y haz clic en **Create role**.

### 3. Asignar rol a instancia EC2
1. Ve a la consola de AWS y selecciona **EC2**.
2. En el menú lateral, haz clic en **Instances** y selecciona tu instancia.
3. Haz clic en **Actions > Security > Modify IAM Role**.
4. Selecciona el rol creado (`rol-s3-acceso`) y haz clic en **Update IAM role**.

### 4. Probar acceso restringido
1. Conéctate a tu instancia EC2 mediante SSH.
2. Intenta listar los buckets S3 sin permisos:
   ```bash
   aws s3 ls
   ```
   Deberías recibir un error de acceso denegado.
3. Asigna el rol IAM a la instancia (como se explicó en el paso anterior).
4. Intenta nuevamente listar los buckets S3:
   ```bash
   aws s3 ls
   ```
   Ahora deberías ver los buckets disponibles.

### 5. Revisar logs de acceso
1. Ve a la consola de AWS y selecciona **IAM**.
2. En el menú lateral, haz clic en **Access Analyzer > Access history**.
3. Filtra por el usuario o rol para ver los intentos de acceso.
4. Revisa los detalles de los intentos exitosos y fallidos.

## 🔹 Ejercicio adicional

### Crear una política personalizada
1. Ve a la consola de IAM y selecciona **Policies**.
2. Crea una nueva política con el siguiente JSON:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::example-bucket"
        }
    ]
}
```
3. Asigna esta política a un usuario o rol.
4. Prueba el acceso intentando listar los buckets S3.