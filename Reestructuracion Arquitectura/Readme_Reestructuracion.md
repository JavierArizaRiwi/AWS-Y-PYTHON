# Reestructuración Profesional de una Instancia EC2 con Docker y Entornos Staging/Producción

Guía paso a paso para transformar una instancia EC2 en un entorno profesional de despliegue para aplicaciones Python usando Docker, control de versiones, entornos separados (staging y producción), rollback y buenas prácticas de seguridad.

---

## Instalación y Configuración de NGINX en EC2

### ¿Por qué usar NGINX?

**NGINX** es un servidor web y proxy inverso ampliamente utilizado en arquitecturas modernas por su:
- Alto rendimiento y bajo consumo de recursos
- Capacidad de servir contenido estático rápidamente
- Balanceo de carga y proxy inverso para aplicaciones (Flask, Django, Node, etc.)
- Seguridad adicional (limitación de métodos, headers, protección DDOS básica)
- Soporte nativo para HTTPS y certificados SSL
- Fácil integración con Docker y despliegues automatizados

**Ventajas de NGINX en EC2:**
- Permite exponer tu aplicación Python solo en localhost y publicar al exterior solo por NGINX (más seguro)
- Puedes tener múltiples apps en la misma instancia, cada una con su propio proxy
- Facilita la gestión de HTTPS y redirecciones
- Permite cachear contenido estático y mejorar la velocidad

---

### Instalación Paso a Paso

1. **Actualizar el sistema:**
  ```bash
  sudo apt update && sudo apt upgrade -y
  ```

2. **Instalar NGINX:**
  ```bash
  sudo apt install nginx -y
  ```

3. **Habilitar y arrancar el servicio:**
  ```bash
  sudo systemctl enable nginx
  sudo systemctl start nginx
  sudo systemctl status nginx
  ```

4. **Configurar el firewall (opcional):**
  ```bash
  sudo ufw allow 'Nginx Full'
  sudo ufw reload
  ```

5. **Configurar un sitio para tu app (reverse proxy):**
  ```bash
  sudo nano /etc/nginx/sites-available/miapp
  ```
  Ejemplo de configuración para una app Python en Docker (puerto 8000):
  ```nginx
  server {
     listen 80;
     server_name tu-dominio.com;

     location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
     }

     location /static/ {
        alias /opt/apps/miapp/static/;
        expires 30d;
     }
  }
  ```

6. **Habilitar el sitio y recargar NGINX:**
  ```bash
  sudo ln -s /etc/nginx/sites-available/miapp /etc/nginx/sites-enabled/
  sudo nginx -t
  sudo systemctl reload nginx
  ```

7. **(Opcional) Configurar HTTPS con Let's Encrypt:**
  ```bash
  sudo apt install certbot python3-certbot-nginx -y
  sudo certbot --nginx -d tu-dominio.com
  ```

---

### Buenas Prácticas con NGINX
- Mantener solo el puerto 80/443 abierto al exterior, el resto solo accesible localmente
- Usar headers de seguridad (X-Frame-Options, X-Content-Type-Options, etc.)
- Limitar tamaño de uploads y peticiones
- Configurar logs separados por app
- Automatizar la configuración con scripts o Ansible

---
---

## 1. Objetivo

- Deploy por incrementos funcionales
- Separación de entornos (staging → producción)
- Rollback inmediato
- Configuración por variables de entorno
- Evitar instalaciones manuales repetitivas
- Seguridad y orden en la instancia

---

## 2. Limpieza del Repositorio

### .gitignore recomendado

```gitignore
# Python
venv/
__pycache__/
*.pyc

# Node
node_modules/

# Secrets
.env
*.pem
*_key
deep_key

# Logs / temp
nohup.out
temp_files/
ArchivosDescargados/
PDFDescargado/
*.rpm
```

Nunca almacenar llaves ni secretos dentro del repositorio.

---

## 3. Estructura Profesional en la EC2

```text
/opt/apps/miapp        -> Contenedores y docker-compose
/etc/miapp             -> Variables de entorno (.env)
/var/lib/miapp         -> Datos temporales y persistentes
/var/log/miapp         -> Logs (opcional)
```

Crear carpetas:

```bash
sudo mkdir -p /opt/apps/miapp /etc/miapp /var/lib/miapp/{temp_files,ArchivosDescargados,PDFDescargado}
sudo chown -R ec2-user:ec2-user /opt/apps/miapp /etc/miapp /var/lib/miapp
```

---

## 4. Instalación de Docker

### Amazon Linux 2023

```bash
sudo dnf update -y
sudo dnf install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
logout
```

Ingresar nuevamente y validar:

```bash
docker --version
```

---

## 5. Dockerización del Proyecto

### .dockerignore

```dockerignore
venv
__pycache__
*.pyc
node_modules
.git
.env
*.pem
*_key
deep_key
nohup.out
temp_files
ArchivosDescargados
PDFDescargado
*.rpm
```

### Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

CMD ["python", "main.py"]
```

---

## 6. Separación de Entornos

### Variables de entorno

`/etc/miapp/staging.env`
```env
APP_ENV=staging
LOG_LEVEL=DEBUG
```

`/etc/miapp/prod.env`
```env
APP_ENV=prod
LOG_LEVEL=INFO
```

---


## 7. Docker Compose con Staging y Producción (Puertos solo en loopback)

```yaml
services:
  miapp-prod:
    image: miapp:prod
    env_file:
      - /etc/miapp/prod.env
    restart: unless-stopped
    ports:
      - "127.0.0.1:8080:8000"

  miapp-staging:
    image: miapp:staging
    env_file:
      - /etc/miapp/staging.env
    restart: unless-stopped
    ports:
      - "127.0.0.1:8081:8000"
```

✅ Esto significa: solo la misma EC2 puede llegar a 8080/8081. Nadie desde fuera puede acceder directamente a esos puertos.

---

## 8. Instalar y Configurar NGINX en EC2

### Instalación

**Amazon Linux 2:**
```bash
sudo yum install -y nginx
sudo systemctl enable --now nginx
```

**Amazon Linux 2023:**
```bash
sudo dnf install -y nginx
sudo systemctl enable --now nginx
```

---

### Configuración de NGINX por dominios (prod y staging)

Crea el archivo de configuración:

```bash
sudo nano /etc/nginx/conf.d/miapp.conf
```

Pega esto (ajusta los dominios):

```nginx
# PROD: api.tudominio.com -> localhost:8080
server {
  listen 80;
  server_name api.tudominio.com;

  location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}

# STAGING: staging.tudominio.com -> localhost:8081
server {
  listen 80;
  server_name staging.tudominio.com;

  location / {
    proxy_pass http://127.0.0.1:8081;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

Revisar y recargar NGINX:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 9. DNS en Route 53

Debes tener:

- `api.tudominio.com` → A record a la IP pública de EC2 (o ALB)
- `staging.tudominio.com` → A record a la misma IP (o ALB)

Route 53 solo resuelve DNS, no interactúa con Docker ni NGINX.

---

## 10. Seguridad: Security Group

- Abre solo el puerto 80 (y luego 443 para HTTPS) a internet
- NO abras 8080/8081 (ya no hace falta, quedan protegidos en loopback)

---

## 11. HTTPS con Let’s Encrypt (opcional pero recomendado)

Cuando ya esté respondiendo por HTTP, instala certbot:

**Amazon Linux 2:**
```bash
sudo yum install -y certbot python3-certbot-nginx
```

**Amazon Linux 2023:**
```bash
sudo dnf install -y certbot python3-certbot-nginx
```

Luego ejecuta:

```bash
sudo certbot --nginx -d api.tudominio.com -d staging.tudominio.com
```

Esto configurará HTTPS automáticamente para ambos dominios.

---

## 12. Verificación de servicios y puertos

Para comprobar qué servicios están escuchando:

```bash
sudo ss -ltnp | egrep ':80|:443|:8080|:8081'
```

- :80 debe ser nginx
- :8080 y :8081 deben ser docker-proxy pero en 127.0.0.1
- Si ves python directo en 80/8000: eso es lo viejo y hay que apagarlo

---

---

## 8. Flujo de Despliegue por Versiones

### Build local

```bash
docker build -t miapp:1.0.0 .
```

### Exportar e importar en EC2

```bash
docker save miapp:1.0.0 -o miapp_1.0.0.tar
scp miapp_1.0.0.tar ec2-user@IP:/opt/apps/miapp/
```

En EC2:

```bash
docker load -i miapp_1.0.0.tar
```

---

## 9. Staging First

```bash
docker tag miapp:1.0.0 miapp:staging
docker compose up -d miapp-staging
docker compose logs -f miapp-staging
```

Validar funcionamiento.

---

## 10. Promoción a Producción

```bash
docker tag miapp:1.0.0 miapp:prod
docker compose up -d miapp-prod
docker compose logs -f miapp-prod
```

---

## 11. Rollback

```bash
docker tag miapp:1.0.0 miapp:prod
docker compose up -d miapp-prod
```

---

## 12. Buenas Prácticas

- Usar IAM Roles en vez de llaves locales
- Variables sensibles en /etc/miapp
- Logs con docker logs o CloudWatch
- Healthchecks y monitoreo
- Nginx + HTTPS para staging y prod

---

## 13. Flujo Profesional Final

```text
Local Dev → Docker Build → Staging → Validación → Promoción a Producción → Monitoreo → Rollback si falla
```

---

Autor: Javier Ariza