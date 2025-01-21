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

Te ayudaré a configurar NGINX como proxy inverso para tus servicios. Primero, necesitaremos crear los archivos de configuración necesarios.

```yaml
# docker-compose.yml
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
    restart: unless-stopped
    networks:
      - proxy-network

networks:
  proxy-network:
    driver: bridge

```

```nginx
# nginx/conf.d/default.conf
server {
    listen 80;
    server_name virygui.com;

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

    # Aquí puedes agregar más bloques location para otros servicios
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
```
192.168.0.84 virygui.com
```

Ahora podrás acceder a tu servicio Plex a través de `http://virygui.com/plex`

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
    server_name virygui.com;

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
```
192.168.0.84 virygui.com
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

Para acceder desde tu smartphone a virygui.com/plex necesitas considerar varios aspectos:

1. Red Local (Sí funcionará):
- Si tu smartphone está conectado a la misma red WiFi que tu servidor
- Deberás agregar la entrada en el archivo hosts de tu smartphone (esto puede requerir root en Android)
- O usar la IP directamente: http://192.168.0.84/plex

2. Fuera de tu red local (No funcionará con solo NGINX):
Para acceder desde fuera necesitarías:
- Un dominio real (comprar virygui.com)
- Configurar DNS para que apunte a tu IP pública
- Abrir/redireccionar puertos en tu router (port forwarding)
- Idealmente, configurar un certificado SSL

La SOLUCIÓN más práctica sería:

A) Para uso solo en red local:
- Usar directamente la IP: http://192.168.0.84/plex
- O configurar un DNS local (Pi-hole o tu router) para resolver virygui.com

B) Para acceso desde cualquier lugar:
1. Comprar un dominio
2. Configurar DDNS si no tienes IP fija
3. Configurar port forwarding en tu router
4. Usar Let's Encrypt para SSL

### PI-HOLE
1. Configuración con Pi-hole:

```bash
# 1. Instalar Pi-hole (si aún no está instalado)
curl -sSL https://install.pi-hole.net | bash

# 2. Acceder a la interfaz web de Pi-hole
# Típicamente en http://IP_PIHOLE/admin

# 3. Configurar DNS Local Records
# En la interfaz web:
# Local DNS → DNS Records
# Añadir nuevo registro:
Domain: virygui.com
IP Address: 192.168.0.84
Add

# 4. Configurar registros adicionales para subdominios (opcional)
Domain: plex.virygui.com
IP Address: 192.168.0.84
Add

Domain: jellyfin.virygui.com
IP Address: 192.168.0.84
Add

# 5. Alternativamente, puedes añadir registros vía línea de comandos:
pihole -a addcustom virygui.com 192.168.0.84

# 6. Verificar la configuración
pihole -q virygui.com

# 7. Configurar tu router para usar Pi-hole como DNS
# En el TP-Link AX3000:
# Advanced → Network → DHCP Server
# Primary DNS: IP_DE_TU_PIHOLE
# Secondary DNS: 8.8.8.8 (o el que prefieras como respaldo)

```

### Consideraciones importantes:

1. Pi-hole:
- Asegúrate de que Pi-hole tiene una IP estática
- Configura una contraseña segura para la interfaz web
- Haz copias de seguridad regulares de la configuración
- Considera usar DHCP desde Pi-hole para mejor control

Para ambos casos:
- Prueba la resolución DNS con `nslookup virygui.com`
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
