# Día 3 – Route 53 y DNS

## 🔹 Teoría

### DNS
- Traduce nombres de dominio a direcciones IP.
- Permite que los usuarios accedan a servicios usando nombres fáciles de recordar.

### Registros
- **A**: Apunta a una dirección IP específica.
- **CNAME**: Alias de otro dominio.

### Health checks
- Validan la disponibilidad de recursos.
- Alertan sobre problemas de conectividad.

## 🔹 Conceptos básicos adicionales

### ¿Qué es un dominio?
Un dominio es un nombre único que identifica un sitio web en Internet, como `midominio.com`.

### ¿Qué es Route 53?
Route 53 es el servicio de DNS de AWS que permite gestionar nombres de dominio y enrutar tráfico a recursos específicos.

### ¿Qué es DNS?
El Sistema de Nombres de Dominio (DNS) traduce nombres de dominio como `midominio.com` a direcciones IP que las computadoras pueden entender.

## 🔹 Práctica paso a paso

### 1. Registrar un dominio en Route 53
1. Ve a la consola de AWS y selecciona **Route 53**.
2. Haz clic en **Domains > Register Domain**.
3. Busca un dominio disponible y sigue los pasos para registrarlo.
4. Completa el pago y espera la confirmación del registro.

### 2. Crear un registro A
1. En la consola de Route 53, selecciona tu dominio registrado.
2. Haz clic en **Create Record**.
3. Configura el registro de la siguiente manera:
   - **Record name**: Deja en blanco para el dominio raíz o escribe un subdominio (por ejemplo, `www`).
   - **Record type**: A.
   - **Value**: Ingresa la IP pública de tu instancia EC2.
   - **TTL**: 300 (valor predeterminado).
4. Haz clic en **Create records**.

### 3. Validar configuración
1. Abre una terminal y ejecuta:
   ```bash
   nslookup midominio.com
   ```
2. Verifica que la IP devuelta coincida con la de tu instancia EC2.

### 4. Configurar un health check
1. En la consola de Route 53, selecciona **Health checks** y haz clic en **Create health check**.
2. Configura los siguientes parámetros:
   - **Monitor endpoint**: Ingresa la IP pública de tu instancia.
   - **Protocol**: HTTP.
   - **Path**: `/`.
3. Haz clic en **Create health check**.
4. Verifica el estado del health check en la consola.

## 🔹 Ejercicio adicional

### Configurar un subdominio
1. Ve a la consola de Route 53.
2. Selecciona tu dominio registrado.
3. Crea un nuevo registro **A** para un subdominio (por ejemplo, `api.midominio.com`) apuntando a la IP pública de tu instancia EC2.
4. Valida la configuración con:
```bash
nslookup api.midominio.com
dig api.midominio.com
```