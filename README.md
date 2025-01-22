- [NGINX Nodo Multimedia](#nginx-nodo-multimedia)
  * [NGINX puede ser muy útil para tu servidor multimedia con Docker por varias razones:](#nginx-puede-ser-muy--til-para-tu-servidor-multimedia-con-docker-por-varias-razones-)
    + [DOCKER](#docker)
    + [LOCAL](#local)
    + [PI-HOLE](#pi-hole)
    + [Consideraciones importantes:](#consideraciones-importantes-)

# NGINX Nodo Multimedia

Configuracion de NGINX para router inverso de los endpoints del nodo multimedia en red local.

## NGINX puede ser muy útil para tu servidor multimedia con Docker por varias razones:

1. Proxy Inverso

- Permite acceder a todos tus contenedores a través de un único punto de entrada
- En lugar de recordar diferentes puertos (ej: servidor1:32400, servidor2:8096), podrías acceder con URLs más amigables como:
  - multimedia.tudominio.com/plex
  - multimedia.tudominio.com/jellyfin
  - multimedia.tudominio.com/sonarr

2. SSL/TLS centralizado

- Puedes configurar certificados SSL una sola vez en NGINX
- NGINX se encarga de la terminación SSL y redirige el tráfico de forma segura a tus contenedores
- Evitas tener que configurar SSL en cada servicio por separado

3. Balanceo de carga

- Si tienes servicios replicados, NGINX puede distribuir el tráfico
- Útil si algunos de tus servicios multimedia requieren más recursos

4. Caché

- NGINX puede cachear contenido estático
- Reduce la carga en tus servicios multimedia
- Mejor rendimiento para los usuarios

5. Seguridad adicional

- Actúa como una capa extra de seguridad
- Puedes implementar reglas de firewall a nivel de aplicación
- Control de acceso más granular
- Protección contra ataques DDoS

### DOCKER

Primero, necesitamos crear los archivos de configuración necesarios.

### Estructura de Directorios

```bash
mkdir -p nginx/conf.d
```

### Archivo `docker-compose.yml`

```yaml
services:
  nginx:
    image: nginx:latest
    container_name: nginx-proxy
    ports:
      # - HOST_PORT:CONTAINER_PORT
      # 8080:80
      # Ahora, cualquier tráfico enviado al puerto 8080 en su máquina anfitriona 
      # será reenviado al puerto 80 dentro del contenedor.
      - "8181:80"
      - "4433:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      # - ./letsencrypt:/etc/letsencrypt
    restart: unless-stopped
    networks:
      - proxy-network

networks:
  proxy-network:
    driver: bridge
```

### Archivo `nginx/conf.d/default.conf`

```nginx
server {
    listen 80;
    server_name plex.example.com;

    # Configuración para Plex
    location / {
        proxy_pass http://192.168.0.84:32400/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_buffering off;
        proxy_redirect off;
        proxy_read_timeout 90s;
        proxy_connect_timeout 90s;

        client_max_body_size 100M;
        proxy_buffer_size 4k;
        proxy_buffers 4 32k;
        proxy_busy_buffers_size 64k;
    }

}

server {
    listen 80;
    server_name jellyfin.example.com;

    location / {
        proxy_pass http://192.168.0.84:8096/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Para implementar esta configuración, sigue estos pasos:

1. Crea una estructura de directorios:

```bash
mkdir -p nginx/conf.d
```

2. Copia los archivos de configuración mostrados arriba en sus respectivas ubicaciones:

- `docker-compose.yml` en el directorio raíz
- `default.conf` en el directorio `nginx/conf.d`

3. Inicia el contenedor:

```bash
docker-compose up -d
```

4. Modifica tu archivo hosts (para pruebas locales):

```bash
sudo nano /etc/hosts
```

Y agrega:

```bash
192.168.0.84 example.com
```

Ahora podrás acceder a tu servicio Plex a través de `http://example.com/plex`

Para agregar más servicios, simplemente añade más bloques `location` en el archivo `default.conf`. Por ejemplo, para Jellyfin:

```nginx
location /jellyfin/ {
    proxy_pass http://192.168.0.84:8096/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Algunas notas importantes:

1. Asegúrate de que todos los servicios estén en la misma red de Docker
2. Los puertos 80 y 443 deben estar libres en tu máquina
3. Si necesitas HTTPS, deberás configurar certificados SSL (podemos cubrir esto si lo necesitas)
4. Algunos servicios pueden requerir configuración adicional de headers

### LOCAL

Si tienes NGINX instalado directamente en tu sistema (no en Docker), la configuración es similar pero los archivos estarán en diferentes ubicaciones. Aquí te explico cómo hacerlo:

1. La configuración principal de NGINX normalmente está en:

```bash
/etc/nginx/nginx.conf
```

2. Las configuraciones de sitios específicos van en:

```bash
/etc/nginx/sites-available/
/etc/nginx/sites-enabled/
```

Vamos a crear una nueva configuración para tus servicios:

1. Crea un nuevo archivo en sites-available:

```bash
sudo nano /etc/nginx/sites-available/multimedia
```

2. Añade esta configuración:

```nginx
server {
    listen 80;
    server_name example.com;

    # Configuración para Plex
    location /plex/ {
        proxy_pass http://192.168.0.84:32400/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Puedes agregar más servicios aquí
    # Ejemplo para Jellyfin
    location /jellyfin/ {
        proxy_pass http://192.168.0.84:8096/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

3. Crea un enlace simbólico a sites-enabled:

```bash
sudo ln -s /etc/nginx/sites-available/multimedia /etc/nginx/sites-enabled/
```

4. Verifica que la configuración sea correcta:

```bash
sudo nginx -t
```

5. Si todo está bien, reinicia NGINX:

```bash
sudo systemctl restart nginx
```

6. Modifica tu archivo hosts:

```bash
sudo nano /etc/hosts
```

Añade:

```bash
192.168.0.84 example.com
```

Para verificar que NGINX está funcionando correctamente:

```bash
sudo systemctl status nginx
```

Para ver los logs en caso de errores:

```bash
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log
```

Consejos adicionales:

1. Asegúrate de que NGINX tiene permisos para acceder a los puertos 80/443:

```bash
sudo apt-get install libcap2-bin
sudo setcap cap_net_bind_service=+ep /usr/sbin/nginx
```

2. Si tienes un firewall activo, permite el tráfico HTTP/HTTPS:

```bash
sudo ufw allow 'Nginx Full'
```

Para acceder desde tu smartphone a example.com/plex necesitas considerar varios aspectos:

1. Red Local (Sí funcionará):

- Si tu smartphone está conectado a la misma red WiFi que tu servidor
- Deberás agregar la entrada en el archivo hosts de tu smartphone (esto puede requerir root en Android)
- O usar la IP directamente: http://192.168.0.84/plex

2. Fuera de tu red local (No funcionará con solo NGINX):

Para acceder desde fuera necesitarías:

- Un dominio real (comprar example.com)
- Configurar DNS para que apunte a tu IP pública
- Abrir/redireccionar puertos en tu router (port forwarding)
- Idealmente, configurar un certificado SSL

La SOLUCIÓN más práctica sería:

A) Para uso solo en red local:

- Usar directamente la IP: http://192.168.0.84/plex
- O configurar un DNS local (Pi-hole o tu router) para resolver example.com

B) Para acceso desde cualquier lugar:

1. Comprar un dominio
2. Configurar DDNS si no tienes IP fija
3. Configurar port forwarding en tu router
4. Usar Let's Encrypt para SSL

### PI-HOLE

1. Configuración con Pi-hole:

2. Instalar Pi-hole (si aún no está instalado)

```bash
curl -sSL https://install.pi-hole.net | bash
```

3. Acceder a la interfaz web de Pi-hole

```text
Típicamente en http://IP_PIHOLE/admin
```

4. Configurar DNS Local Records
En la interfaz web:
Local DNS → DNS Records
Añadir nuevo registro:

```text
Domain: example.com
IP Address: 192.168.0.84
Add
```

5. Configurar registros adicionales para subdominios (opcional)

```text
Domain: plex.example.com
IP Address: 192.168.0.84
Add

Domain: jellyfin.example.com
IP Address: 192.168.0.84
Add
```

6. Alternativamente, puedes añadir registros vía línea de comandos:

```bash
pihole -a addcustom example.com 192.168.0.84
```

7. Verificar la configuración

```bash
pihole -q example.com
```

8. Configurar tu router para usar Pi-hole como DNS

### Consideraciones importantes

1. Pi-hole:

- Asegúrate de que Pi-hole tiene una IP estática
- Configura una contraseña segura para la interfaz web
- Haz copias de seguridad regulares de la configuración
- Considera usar DHCP desde Pi-hole para mejor control

Para ambos casos:

- Prueba la resolución DNS con `nslookup example.com`
- Verifica que puedes acceder desde diferentes dispositivos
- Monitorea los logs por posibles errores

Para Plex a través de NGINX, te recomiendo esta configuración optimizada para streaming:

```nginx
# En el archivo de configuración del servidor
server {
    listen 80;
    server_name tudominio.local;

    # Configuración específica para Plex
    location /plex/ {
        proxy_pass http://192.168.0.84:32400/;
        
        # Headers necesarios para Plex
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Configuración optimizada para streaming
        proxy_redirect off;
        proxy_buffering off;  # Importante para streaming
        
        # Timeouts más largos para evitar interrupciones
        proxy_read_timeout 90s;
        proxy_connect_timeout 90s;
        
        # Soporte para streaming y websockets
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Tamaños de buffer para streaming HD
        client_max_body_size 100M;
        proxy_buffer_size 4k;
        proxy_buffers 4 32k;
        proxy_busy_buffers_size 64k;
    }
}
```

Las claves de esta configuración son:

1. `proxy_buffering off`: Mejora la experiencia de streaming
2. Timeouts largos: Evita cortes durante la reproducción
3. Soporte websockets: Necesario para algunas funciones de Plex
4. Buffers ajustados: Optimizados para streaming de video

Si experimentas algún problema específico, como:

- Cortes en la reproducción: Aumentar los timeouts
- Problemas con calidad HD: Aumentar los tamaños de buffer
- Problemas de conexión: Verificar la configuración de websockets

################################################

# Configuración de NGINX como Proxy Inverso para Servicios Multimedia con Docker

Este repositorio contiene la configuración de NGINX como proxy inverso para servicios multimedia como Plex, Jellyfin, Sonarr, entre otros, utilizando Docker. La configuración incluye optimizaciones para streaming, seguridad, caché, balanceo de carga y más.

## Características Principales

- **Proxy Inverso**: Accede a todos tus contenedores a través de un único punto de entrada con URLs amigables.
- **SSL/TLS Centralizado**: Configura certificados SSL una sola vez en NGINX.
- **Balanceo de Carga**: Distribuye el tráfico entre múltiples instancias de un servicio.
- **Caché**: Reduce la carga en tus servicios multimedia y mejora el rendimiento.
- **Seguridad Adicional**: Implementa reglas de firewall a nivel de aplicación y protección contra ataques DDoS.

## Configuración Básica

### Estructura de Directorios

```bash
mkdir -p nginx/conf.d
```

### Archivo `docker-compose.yml`

```yaml
version: '3'
services:
  nginx:
    image: nginx:latest
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./letsencrypt:/etc/letsencrypt
    restart: unless-stopped
    networks:
      - proxy-network

networks:
  proxy-network:
    driver: bridge
```

### Archivo `nginx/conf.d/default.conf`

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Configuración para Plex
    location /plex/ {
        proxy_pass http://192.168.0.84:32400/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_buffering off;
        proxy_redirect off;
        proxy_read_timeout 90s;
        proxy_connect_timeout 90s;

        client_max_body_size 100M;
        proxy_buffer_size 4k;
        proxy_buffers 4 32k;
        proxy_busy_buffers_size 64k;
    }

    # Configuración para Jellyfin
    location /jellyfin/ {
        proxy_pass http://192.168.0.84:8096/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Aquí puedes agregar más bloques location para otros servicios
}
```

## Mejoras y Optimizaciones

### 1. **Configuración de SSL/TLS**

Para asegurar las comunicaciones, es crucial configurar SSL/TLS. Puedes usar Let's Encrypt para obtener certificados gratuitos.

### 2. **Mejoras en la Configuración de Plex**

Optimiza el streaming ajustando parámetros adicionales:

```nginx
location /plex/ {
    proxy_pass http://192.168.0.84:32400/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    proxy_buffering off;
    proxy_redirect off;
    proxy_read_timeout 90s;
    proxy_connect_timeout 90s;

    client_max_body_size 100M;
    proxy_buffer_size 4k;
    proxy_buffers 4 32k;
    proxy_busy_buffers_size 64k;
}
```

### 3. **Configuración de Caché**

Configura NGINX para cachear contenido estático y reducir la carga en tus servicios:

```nginx
http {
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=1g inactive=60m use_temp_path=off;

    server {
        location /static/ {
            proxy_cache my_cache;
            proxy_pass http://192.168.0.84:32400/;
            proxy_cache_valid 200 302 10m;
            proxy_cache_valid 404 1m;
        }
    }
}
```

### 4. **Seguridad Adicional**

Agrega directivas de seguridad adicionales para proteger tu servidor:

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    # Prevenir clickjacking
    add_header X-Frame-Options "SAMEORIGIN";

    # Prevenir MIME sniffing
    add_header X-Content-Type-Options "nosniff";

    # Política de seguridad de contenido
    add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data:;";

    # Prevenir XSS
    add_header X-XSS-Protection "1; mode=block";

    # Configuración SSL
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location /plex/ {
        proxy_pass http://192.168.0.84:32400/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### 5. **Uso de Variables para IPs y Puertos**

Haz tu configuración más mantenible usando variables para las IPs y puertos:

```nginx
map $host $backend {
    default         192.168.0.84;
    plex.example.com 192.168.0.84:32400;
    jellyfin.example.com 192.168.0.84:8096;
}

server {
    listen 80;
    server_name example.com;

    location /plex/ {
        proxy_pass http://$backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### 6. **Configuración de Logs**

Configura logs separados para cada servicio para facilitar la depuración:

```nginx
http {
    log_format custom '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log custom;
    error_log /var/log/nginx/error.log;

    server {
        location /plex/ {
            access_log /var/log/nginx/plex_access.log custom;
            error_log /var/log/nginx/plex_error.log;

            proxy_pass http://192.168.0.84:32400/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
}
```

### 7. **Uso de Subdominios**

En lugar de usar rutas (`/plex`, `/jellyfin`), configura subdominios para cada servicio:

```nginx
server {
    listen 80;
    server_name plex.example.com;

    location / {
        proxy_pass http://192.168.0.84:32400/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

server {
    listen 80;
    server_name jellyfin.example.com;

    location / {
        proxy_pass http://192.168.0.84:8096/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 8. **Configuración de Balanceo de Carga**

Si tienes múltiples instancias de un servicio, configura balanceo de carga:

```nginx
upstream plex_servers {
    server 192.168.0.84:32400;
    server 192.168.0.85:32400;
}

server {
    listen 80;
    server_name plex.example.com;

    location / {
        proxy_pass http://plex_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### 9. **Configuración de Rate Limiting**

Protege tus servicios de abusos configurando límites de tasa:

```nginx
http {
    limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;

    server {
        location /plex/ {
            limit_req zone=one burst=5;

            proxy_pass http://192.168.0.84:32400/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
}
```

### 10. **Configuración de Health Checks**

Configura chequeos de salud para tus servicios:

```nginx
upstream plex_servers {
    server 192.168.0.84:32400;
    server 192.168.0.85:32400;

    check interval=3000 rise=2 fall=5 timeout=1000;
}

server {
    listen 80;
    server_name plex.example.com;

    location / {
        proxy_pass http://plex_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

## Conclusión

Esta configuración te permitirá tener un proxy inverso robusto, seguro y optimizado para tus servicios multimedia. Asegúrate de probar cada cambio en un entorno de desarrollo antes de implementarlo en producción. Si tienes alguna pregunta adicional o necesitas más detalles sobre alguna de estas configuraciones, no dudes en preguntar.

---

**Nota**: Asegúrate de reemplazar las direcciones IP y nombres de dominio con los valores correctos para tu entorno.
