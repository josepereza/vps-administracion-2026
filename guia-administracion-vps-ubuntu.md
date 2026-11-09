# Guía práctica: administrar un VPS Ubuntu

Guía de referencia para un VPS con **Ubuntu**, una web **Angular**, una API **Node.js/NestJS**, **Nginx** como proxy inverso y, si aplica, MySQL o PostgreSQL. Sustituye los ejemplos `mi-api`, `mi-dominio.ch` y las rutas por las tuyas.

> Regla de oro: antes de modificar Nginx o servicios, valida, guarda una copia y prueba localmente. No expongas puertos de Node.js ni de la base de datos a Internet si Nginx o la propia máquina pueden acceder a ellos internamente.

---

## 1. Acceso seguro y datos del servidor

```bash
ssh usuario@IP_DEL_VPS
whoami
hostnamectl
uptime
```

- `whoami`: usuario con el que has entrado.
- `hostnamectl`: versión de Ubuntu y nombre del servidor.
- `uptime`: tiempo encendido y carga media. Una carga alta mantenida merece revisión.

Actualiza los paquetes con cuidado:

```bash
sudo apt update
sudo apt upgrade
sudo apt autoremove
```

Lee primero lo que Ubuntu propone eliminar. Evita actualizar en mitad de un despliegue o sin una copia reciente de datos importantes.

### Usuarios y permisos

```bash
getent passwd | cut -d: -f1
groups miusuario
sudo -l
```

Da acceso de administrador solo a usuarios de confianza:

```bash
sudo usermod -aG sudo miusuario
```

Después, el usuario debe cerrar sesión y entrar de nuevo. No ejecutes tus aplicaciones como `root`.

### SSH: recomendaciones mínimas

Usa claves SSH y conserva una segunda sesión abierta antes de cambiar la configuración.

```bash
sudo nano /etc/ssh/sshd_config
sudo sshd -t
sudo systemctl reload ssh
```

Valores recomendables cuando ya confirmaste que las claves funcionan:

```text
PermitRootLogin no
PasswordAuthentication no
```

**Nunca desactives `PasswordAuthentication` antes de comprobar que puedes entrar con tu clave SSH.**

---

## 2. Firewall con UFW

Ver reglas actuales:

```bash
sudo ufw status verbose
sudo ufw status numbered
```

Configuración habitual y segura para Angular + API tras Nginx:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
sudo ufw status
```

Esto permite:

| Puerto | Uso | Público |
|---|---|---|
| 22 | SSH | Sí, solo para administrar |
| 80 | HTTP | Sí |
| 443 | HTTPS | Sí |
| 3000 | Node/NestJS | No |
| 3306 / 5432 | MySQL / PostgreSQL | No |

Para retirar una regla, identifica primero su número y luego bórrala:

```bash
sudo ufw status numbered
sudo ufw delete NUMERO
```

No habilites UFW sin permitir antes SSH: podrías perder el acceso remoto.

---

## 3. Comprobar recursos y procesos

```bash
df -h
free -h
top
ps aux --sort=-%mem | head
ps aux --sort=-%cpu | head
```

- `df -h`: uso de disco. Investiga a partir de 80%; actúa con urgencia sobre 90%.
- `free -h`: memoria RAM y swap.
- `top`: procesos en vivo; pulsa `q` para salir.
- `ps`: procesos que consumen más memoria o CPU.

Encontrar carpetas grandes sin borrar nada:

```bash
sudo du -sh /var/* 2>/dev/null | sort -h
sudo du -sh /var/www/* 2>/dev/null | sort -h
sudo du -sh /var/log/* 2>/dev/null | sort -h
```

No uses `rm -rf` para liberar espacio a ciegas. Los lugares frecuentes a revisar son builds antiguos, copias no necesarias y logs que hayan crecido demasiado.

---

## 4. Revisar consumo de CPU, RAM, espacio y transferencia

Esta sección sirve para saber si el VPS está llegando a su límite antes de que la web o la API fallen.

### CPU y carga del servidor

```bash
uptime
top
ps aux --sort=-%cpu | head -n 10
```

Ejemplo de `uptime`:

```text
load average: 0.18, 0.12, 0.09
```

Los tres valores son la carga media de los últimos 1, 5 y 15 minutos. Compárala con los núcleos disponibles:

```bash
nproc
lscpu | grep '^CPU(s):'
```

En un VPS de 2 CPU, una carga cercana o superior a `2.00` durante varios minutos indica que conviene investigar. Un pico breve no suele ser un problema.

Para una vista más cómoda puedes instalar `htop`:

```bash
sudo apt update
sudo apt install htop
htop
```

En `top` o `htop`, busca un proceso `node`, `nginx`, una base de datos o un proceso desconocido que consuma CPU de forma continua. Pulsa `q` para salir de `top`; en `htop`, `F10`.

### RAM y swap

```bash
free -h
vmstat 1 5
ps aux --sort=-%mem | head -n 10
```

Ejemplo simplificado:

```text
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       1.9Gi       210Mi        50Mi       1.7Gi       1.5Gi
Swap:          1.0Gi          0B       1.0Gi
```

Fíjate especialmente en `available`, no solo en `free`: Linux usa RAM libre como caché y la libera cuando una aplicación la necesita. Si `available` es muy baja de forma continua y la swap crece, el VPS puede ir lento o terminar procesos por falta de memoria.

En `vmstat`, una actividad alta repetida en las columnas `si` y `so` indica que el sistema está moviendo memoria entre RAM y swap. No reinicies sin conocer la causa: primero identifica el proceso que consume memoria.

### Espacio de disco e inodos

```bash
df -h
df -i
sudo du -sh /var/* 2>/dev/null | sort -h
sudo du -sh /var/log/* 2>/dev/null | sort -h
```

- `df -h`: espacio ocupado en GB/MB.
- `df -i`: inodos disponibles; un VPS puede quedar bloqueado si hay millones de archivos pequeños aunque el disco parezca tener espacio.
- `du`: muestra qué carpetas ocupan más.

Como regla práctica, investiga a partir de 80% de disco y actúa antes de llegar al 90%. Revisa logs, builds antiguos, copias de seguridad locales y archivos subidos; no borres nada sin confirmar su función.

### Transferencia y tráfico de red

Ver interfaces y contadores acumulados desde el último reinicio:

```bash
ip -s link
cat /proc/net/dev
```

Normalmente la interfaz se llama `eth0`, `ens3` o similar. Para ver solo una:

```bash
ip -s link show eth0
```

Las columnas `RX` son datos recibidos y `TX` datos enviados. Ejecuta el mismo comando de nuevo pasados uno o dos minutos para observar si aumentan mucho.

Resumen de conexiones actuales:

```bash
ss -s
sudo ss -tulpn
```

Para observar velocidad de entrada y salida en tiempo real, instala `iftop` o `nload`:

```bash
sudo apt update
sudo apt install iftop nload
sudo iftop -i eth0
nload eth0
```

`iftop` muestra qué direcciones IP están consumiendo transferencia; `nload` enseña gráficos sencillos de entrada y salida. Sal con `q`.

Para conservar estadísticas de tráfico por día o mes, `vnStat` es útil:

```bash
sudo apt install vnstat
sudo systemctl enable --now vnstat
vnstat
vnstat -d
vnstat -m
```

`vnstat -d` muestra tráfico diario y `vnstat -m` mensual. Es práctico si tu proveedor tiene un límite de transferencia.

### Comprobación rápida en una sola sesión

```bash
uptime
free -h
df -h
ip -s link
ss -s
```

Precauciones:

- `iftop` revela IP y conexiones activas: no compartas capturas o salidas completas públicamente.
- Instala herramientas solo desde los repositorios oficiales de Ubuntu (`apt`).
- Un aumento de tráfico inesperado puede ser un pico real de visitas, bots o un problema; revisa también `/var/log/nginx/access.log` antes de bloquear nada.

---

## 5. Servicios con systemd

`systemd` controla los servicios del VPS: Nginx, tu API y bases de datos.

```bash
sudo systemctl status nginx
sudo systemctl is-active nginx
sudo systemctl restart nginx
sudo systemctl reload nginx
sudo systemctl enable nginx
```

| Acción | Uso |
|---|---|
| `status` | Ver estado y últimas líneas del log |
| `start` / `stop` | Arrancar o detener |
| `restart` | Detiene brevemente y arranca de nuevo |
| `reload` | Lee configuración nueva sin cortar conexiones activas |
| `enable` | Arranca tras reiniciar el VPS |

Para cambios normales de Nginx, usa `reload`, no `restart`.

### Inspeccionar por qué un servicio no arranca

Antes de editar o reiniciar repetidamente una API, comprueba qué configuración está usando realmente `systemd` y qué ocurrió desde el último arranque del VPS:

```bash
# Muestra el archivo .service y cualquier ajuste adicional (drop-in)
sudo systemctl cat mi-api

# Muestra datos clave de la unidad, sin abrir un editor
sudo systemctl show mi-api -p FragmentPath -p User -p WorkingDirectory -p ExecStart

# Estado breve, PID principal y últimas líneas del arranque actual
sudo systemctl show mi-api -p ActiveState -p SubState -p MainPID
sudo journalctl -u mi-api -b -n 80 --no-pager
```

- `systemctl cat`: enseña el archivo principal y los ajustes que lo sobrescriben.
- `FragmentPath`: indica la ruta del `.service` que se está cargando.
- `-b`: limita el log al arranque actual; evita mezclar errores antiguos.
- `MainPID`: es el proceso principal del servicio cuando está activo.

Lo normal es ver `ActiveState=active`, `SubState=running` y un PID mayor que cero. Si aparece `failed` varias veces, arregla primero el error del log. Después de modificar un archivo `.service`, aplica el cambio de forma ordenada:

```bash
sudo systemctl daemon-reload
sudo systemctl restart mi-api
sudo systemctl status mi-api --no-pager
```

Si `systemd` ha bloqueado el servicio por demasiados reinicios, usa `sudo systemctl reset-failed mi-api` **solo después** de corregir la causa. No publiques la salida completa de `systemctl cat` si incluye secretos directamente en el servicio; guarda las claves en un `EnvironmentFile` con permisos restringidos.

### Servicio para una API NestJS

Archivo `/etc/systemd/system/mi-api.service`:

```ini
[Unit]
Description=API Node.js / NestJS
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/var/www/mi-api
ExecStart=/usr/bin/node dist/main.js
Restart=always
RestartSec=5
Environment=NODE_ENV=production
EnvironmentFile=/etc/mi-api.env

[Install]
WantedBy=multi-user.target
```

Aplicar y arrancar:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mi-api
sudo systemctl status mi-api
sudo journalctl -u mi-api -n 50 --no-pager
```

Variables sensibles en `/etc/mi-api.env`:

```bash
sudo nano /etc/mi-api.env
sudo chmod 600 /etc/mi-api.env
sudo chown root:root /etc/mi-api.env
```

Ejemplo de contenido:

```text
DATABASE_URL=...
JWT_SECRET=una_clave_larga_y_privada
```

No pongas secretos en Git ni dentro del archivo `.service`.

---

## 6. Logs: tu primera herramienta para diagnosticar

```bash
sudo tail -n 50 /var/log/nginx/error.log
sudo tail -n 50 /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
sudo journalctl -u mi-api -n 100 --no-pager
sudo journalctl -u mi-api -f
```

`tail -f` sigue el log en tiempo real; se sale con `Ctrl+C`.

Errores típicos:

| Mensaje | Causa probable | Primera comprobación |
|---|---|---|
| `connect() failed (111)` | La API no escucha o está caída | `systemctl status mi-api` |
| `502 Bad Gateway` | Nginx no alcanza Node | `ss -tulpn`, logs API |
| `permission denied` | Usuario sin permiso sobre archivos | propietario/permisos de la ruta |
| `no space left on device` | Disco lleno | `df -h` |

Los logs pueden contener IP, rutas o datos personales: no los publiques completos ni los borres manualmente.

### Buscar el dato relevante con `grep`

`grep` filtra líneas por texto. Sirve para no tener que leer un log entero ni buscar a mano en muchos archivos de configuración.

```bash
# Errores de Nginx sin distinguir mayúsculas de minúsculas
sudo grep -i "error" /var/log/nginx/error.log

# Dónde se menciona el puerto 3000 en la configuración de Nginx
sudo grep -Rni "3000" /etc/nginx/

# Dos líneas antes y después de cada respuesta 502
sudo grep -C 2 "502" /var/log/nginx/access.log
```

- `-i`: ignora mayúsculas/minúsculas.
- `-R`: busca de forma recursiva en subdirectorios.
- `-n`: muestra el número de línea.
- `-C 2`: añade dos líneas de contexto antes y después.

Lo esperado es localizar el archivo y el momento exacto del problema. No uses `grep` sobre directorios enormes como `/` y evita compartir resultados que contengan tokens, cookies o datos personales.

### Evitar que los logs llenen el disco

Los logs antiguos pueden consumir todo el espacio del VPS. Ubuntu usa `logrotate` para comprimirlos, conservar solo un número limitado y crear archivos nuevos sin detener Nginx.

```bash
# Simula la rotación: no modifica nada
sudo logrotate -d /etc/logrotate.conf

# Comprueba el espacio ocupado por el journal de systemd
sudo journalctl --disk-usage
```

Un resultado normal de `logrotate -d` enumera los ficheros que revisaría. Si el journal ocupa demasiado y ya has comprobado que no necesitas logs antiguos para investigar una incidencia, puedes conservar solo los últimos 14 días:

```bash
sudo journalctl --vacuum-time=14d
sudo journalctl --disk-usage
```

`--vacuum-time=14d` elimina entradas antiguas del journal; no afecta a los archivos de Nginx en `/var/log/nginx/`. Antes de usarlo, revisa que una incidencia reciente no necesite esos logs y guarda fuera del VPS cualquier evidencia importante. No borres archivos de `/var/log` a mano mientras un servicio los está escribiendo.

---

## 7. Puertos y conectividad local

```bash
sudo ss -tulpn
sudo ss -tulpn | grep 3000
curl -I http://localhost
curl -i http://127.0.0.1:3000/api/prueba
sudo lsof -i :3000
```

Resultado deseado aproximado:

```text
127.0.0.1:3000  ... node
0.0.0.0:80      ... nginx
0.0.0.0:443     ... nginx
```

Si Node escucha en `127.0.0.1:3000`, solo Nginx y el propio VPS pueden acceder a la API. Es la opción recomendada si Nginx es el proxy inverso.

`lsof` muestra qué proceso posee un puerto. Si aparece `node`, junto a su PID, ese proceso está usando el puerto; si no devuelve nada, nadie escucha ahí. Para ver el comando completo del proceso puedes usar:

```bash
ps aux | grep node
```

Si el puerto está ocupado por una instancia antigua, revisa primero el servicio:

```bash
sudo systemctl restart mi-api
sudo lsof -i :3000
```

No mates procesos con `kill -9` como primera opción: puede dejar archivos o conexiones en un estado inconsistente. Confirma también que la API escuche en `127.0.0.1`, no en `0.0.0.0`, salvo que necesites exponerla deliberadamente.

### Comprobar una ruta de salud con `curl`

Una ruta `GET /health` que no requiere credenciales es una comprobación pequeña y muy útil tras desplegar.

```bash
# Comprueba la API directamente, sin pasar por Nginx
curl -i http://127.0.0.1:3000/health

# Comprueba el proxy de Nginx y la ruta pública interna
curl -i http://localhost/api/health

# Muestra solo el código HTTP, útil en scripts de despliegue
curl -s -o /dev/null -w "%{http_code}\\n" http://localhost/api/health
```

Lo normal es recibir `200 OK`. Si la primera llamada funciona y la segunda no, revisa `location /api/`, `proxy_pass` y el log de error de Nginx. Si ambas fallan, revisa el servicio Node.js, sus variables de entorno y la conexión a la base de datos. No incluyas secretos ni datos privados en la respuesta de `/health`.

---

## 8. Nginx: publicar Angular y conectar la API

Archivos habituales:

```bash
ls -la /etc/nginx/sites-available
ls -la /etc/nginx/sites-enabled
sudo nginx -t
```

Configuración ejemplo `/etc/nginx/sites-available/mi-dominio`:

```nginx
server {
    listen 80;
    server_name mi-dominio.ch www.mi-dominio.ch;

    root /var/www/mi-frontend/browser;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:3000/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Activar el sitio y aplicar de forma segura:

```bash
sudo ln -s /etc/nginx/sites-available/mi-dominio /etc/nginx/sites-enabled/mi-dominio
sudo nginx -t
sudo systemctl reload nginx
curl -I http://localhost
```

Antes de editar, crea una copia:

```bash
sudo cp /etc/nginx/sites-available/mi-dominio /etc/nginx/sites-available/mi-dominio.bak
```

### Ver la configuración efectiva con `nginx -T`

Un sitio puede cargar archivos desde `sites-enabled`, `conf.d` o configuraciones incluidas. `nginx -T` valida la sintaxis y muestra la configuración completa que Nginx usaría. Es muy útil cuando no sabes qué bloque `server` atiende un dominio o de dónde sale un `proxy_pass`.

```bash
# Revisa toda la configuración de forma cómoda; pulsa q para salir
sudo nginx -T 2>&1 | less

# Localiza nombres de dominio y proxies activos
sudo nginx -T 2>&1 | grep -nE 'server_name|proxy_pass'
```

Ejemplo de resultado:

```text
58:    server_name mi-dominio.ch www.mi-dominio.ch;
71:        proxy_pass http://127.0.0.1:3000/;
```

Si ves varios bloques con el mismo `server_name`, revisa cuál debe permanecer activo y los enlaces de `/etc/nginx/sites-enabled/`. Antes de hacer cambios, valida siempre con `sudo nginx -t` y recarga solo si la prueba termina correctamente.

**Precaución:** `nginx -T` puede mostrar rutas internas, certificados o valores incluidos en la configuración. No pegues su salida completa en foros o chats; comparte únicamente las líneas necesarias y oculta cualquier dato sensible.

### El detalle importante de `proxy_pass`

Con este bloque:

```nginx
location /api/ { proxy_pass http://127.0.0.1:3000/; }
```

La petición pública `/api/productos` llega a Node como `/productos`. Si tu backend necesita conservar `/api`, usa `proxy_pass http://127.0.0.1:3000;` (sin la barra final). Prueba una ruta real tras cada cambio.

---

## 9. HTTPS con Certbot

Antes: el dominio debe apuntar a la IP del VPS y Nginx debe responder por HTTP.

```bash
sudo apt install certbot python3-certbot-nginx
sudo nginx -t
sudo certbot --nginx -d mi-dominio.ch -d www.mi-dominio.ch
sudo certbot renew --dry-run
```

Certbot suele crear la redirección de HTTP a HTTPS. Comprueba el certificado:

```bash
sudo certbot certificates
curl -I https://mi-dominio.ch
```

No ejecutes Certbot si `nginx -t` falla. El puerto 80 debe estar abierto temporalmente para la validación HTTP normal.

### Comprobar la fecha de caducidad desde el VPS

Puedes consultar el certificado que está sirviendo el dominio sin abrir un navegador:

```bash
echo | openssl s_client -connect mi-dominio.ch:443 -servername mi-dominio.ch 2>/dev/null \
  | openssl x509 -noout -dates -issuer -subject
```

Ejemplo de resultado:

```text
notBefore=Nov  1 12:00:00 2026 GMT
notAfter=Jan 30 12:00:00 2027 GMT
issuer=C = US, O = Let's Encrypt, CN = E8
subject=CN = mi-dominio.ch
```

`notAfter` es la fecha límite. `-servername` envía el nombre del sitio durante la conexión TLS; es importante en un VPS con varios dominios HTTPS. Si el dominio no resuelve hacia este VPS o el puerto 443 está cerrado, la conexión fallará.

**Precaución:** sustituye siempre `mi-dominio.ch` por un dominio que controles. Este comando solo inspecciona el certificado público; no sustituye `sudo certbot renew --dry-run`, que comprueba la renovación automática.

---

## 10. Desplegar Angular

En tu proyecto local o en el VPS, la compilación de producción genera archivos estáticos:

```bash
npm ci
npm run build
```

Después copia el resultado a la ruta configurada en `root` de Nginx. En Angular moderno puede ser, por ejemplo, `dist/mi-app/browser`.

Comprobaciones después del despliegue:

```bash
ls -la /var/www/mi-frontend/browser
curl -I http://localhost
sudo nginx -t
```

Para una SPA Angular, `try_files ... /index.html` es necesario: permite que al abrir directamente `/admin/productos`, Nginx entregue Angular en vez de un error 404.

Evita subir `node_modules` al servidor si solo necesitas archivos estáticos compilados.

### Copiar una nueva versión con `rsync`

`rsync` copia solo los archivos que han cambiado. Es útil para actualizar la carpeta que Nginx publica sin subir manualmente todo el proyecto.

Primero simula el despliegue:

```bash
sudo rsync -av --delete --dry-run dist/mi-app/browser/ /var/www/mi-app/
```

- `-a`: conserva la estructura y fechas de los archivos.
- `-v`: muestra los archivos procesados.
- `--delete`: elimina del destino archivos antiguos que ya no pertenecen a la nueva compilación.
- `--dry-run`: solo muestra qué ocurriría; todavía no modifica nada.

Si la lista es correcta, repite el comando sin `--dry-run`:

```bash
sudo rsync -av --delete dist/mi-app/browser/ /var/www/mi-app/
curl -I http://localhost
```

Lo esperado es una respuesta `HTTP/1.1 200 OK` y que la web muestre la nueva versión.

**Precaución:** revisa siempre primero la simulación. `--delete` puede borrar archivos en el destino si indicas una ruta incorrecta. La barra final de `browser/` también importa: copia el contenido de esa carpeta, no la carpeta como un nivel adicional.

---

## 11. Desplegar Node.js / NestJS

Flujo habitual:

```bash
cd /var/www/mi-api
npm ci
npm run build
sudo systemctl restart mi-api
sudo systemctl status mi-api
sudo journalctl -u mi-api -n 50 --no-pager
```

Antes de reiniciar, confirma que la compilación terminó sin errores. Tras reiniciar, verifica localmente y desde Nginx:

```bash
curl -i http://127.0.0.1:3000/health
curl -i http://localhost/api/health
```

Si el primer comando funciona y el segundo no, el problema está en Nginx. Si ninguno funciona, está en Node.js, sus variables o su base de datos.

### Permisos de las carpetas

El usuario del servicio necesita leer el proyecto y, si guardas imágenes en el servidor, escribir solo en el directorio de uploads:

```bash
sudo chown -R www-data:www-data /var/www/mi-api/uploads
sudo chmod 750 /var/www/mi-api/uploads
```

No des permisos `777`: parecen resolver un error, pero abren una puerta de seguridad innecesaria.

---

## 12. Base de datos: comprobaciones básicas

Nunca expongas MySQL (`3306`) o PostgreSQL (`5432`) al público salvo necesidad muy justificada y una restricción fuerte por IP/VPN.

### PostgreSQL

```bash
sudo systemctl status postgresql
sudo -u postgres psql
```

Dentro de `psql`:

```sql
\l
\du
\c mi_base
\dt
```

### MySQL

```bash
sudo systemctl status mysql
sudo mysql
```

Dentro:

```sql
SHOW DATABASES;
SELECT user, host FROM mysql.user;
```

Para datos reales, crea copias antes de migraciones y prueba las migraciones primero en desarrollo.

---

## 13. Copias de seguridad

Una copia útil es la que puedes restaurar. Guarda las copias fuera del VPS cuando sea posible.

### PostgreSQL

```bash
sudo -u postgres pg_dump -Fc mi_base > /var/backups/mi_base_$(date +%F).dump
```

Restauración de prueba:

```bash
sudo -u postgres createdb mi_base_prueba
sudo -u postgres pg_restore -d mi_base_prueba /var/backups/mi_base_2026-10-31.dump
```

Antes de restaurar, comprueba que el fichero se creó y puede leerse:

```bash
ls -lh /var/backups/mi_base_$(date +%F).dump
sudo -u postgres pg_restore -l /var/backups/mi_base_2026-10-31.dump | head
```

`pg_restore -l` lista el contenido sin modificar ninguna base de datos. Una restauración de prueba debe hacerse en una base vacía con otro nombre, nunca sobre producción. Cuando termines, borra la base de prueba solo si has confirmado el resultado y ya no la necesitas.

### MySQL

```bash
mysqldump -u USUARIO -p --single-transaction mi_base > /var/backups/mi_base_$(date +%F).sql
```

Incluye también variables de entorno, archivos subidos y configuración Nginx. Las claves privadas no deben quedar en copias públicas ni repositorios.

---

## 14. Checklist de diagnóstico rápido

### La web no abre

```bash
sudo systemctl status nginx
sudo nginx -t
sudo tail -n 50 /var/log/nginx/error.log
sudo ufw status
```

### Sale `502 Bad Gateway`

```bash
sudo systemctl status mi-api
sudo journalctl -u mi-api -n 50 --no-pager
sudo ss -tulpn | grep 3000
curl -i http://127.0.0.1:3000/health
sudo tail -n 50 /var/log/nginx/error.log
```

### Tras editar Nginx deja de funcionar

```bash
sudo nginx -t
sudo cp /etc/nginx/sites-available/mi-dominio.bak /etc/nginx/sites-available/mi-dominio
sudo nginx -t
sudo systemctl reload nginx
```

### El disco está lleno

```bash
df -h
sudo du -sh /var/* 2>/dev/null | sort -h
sudo journalctl --disk-usage
```

Primero identifica qué creció. No borres bases de datos, logs o directorios de aplicación sin confirmar qué contienen.

---

## 15. Actualizar Ubuntu de forma segura

Actualizar corrige fallos de seguridad y errores, pero conviene hacerlo con una sesión SSH estable y una copia reciente de la base de datos antes de tocar paquetes importantes.

```bash
# Actualiza el catálogo de paquetes: todavía no instala nada
sudo apt update

# Revisa qué paquetes tienen actualización disponible
apt list --upgradable

# Instala actualizaciones normales
sudo apt upgrade
```

Al terminar, comprueba que no haya servicios fallidos y que la web siga respondiendo:

```bash
sudo systemctl --failed
sudo systemctl status nginx mi-api --no-pager
curl -s -o /dev/null -w "%{http_code}\\n" http://localhost/api/health
```

Se espera una lista vacía de unidades fallidas y un código `200`. Lee con cuidado el resumen de `apt upgrade` antes de aceptar: no uses `apt full-upgrade`, ni reinicies, sin entender qué paquetes se eliminarían o sustituirían. Si el sistema indica que hace falta reiniciar, planifica una ventana breve y verifica el acceso SSH antes.

---

## 16. Rutina recomendada

### Cada semana

```bash
sudo apt update
sudo apt upgrade
df -h
sudo systemctl --failed
sudo ufw status
sudo certbot renew --dry-run
```

### Antes de cada despliegue

1. Haz copia de la base de datos y de las variables importantes.
2. Comprueba que el código compila (`npm run build`).
3. Revisa cambios de migraciones y variables de entorno.
4. Despliega y reinicia solo el servicio necesario.
5. Comprueba logs, una ruta de salud y la web desde el navegador.

### Después de cada cambio de Nginx

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Ese `&&` hace que la recarga solo se ejecute si la validación fue correcta.

---

## 17. Comandos que conviene recordar

```bash
# Servicios y logs
sudo systemctl status NOMBRE
sudo systemctl restart NOMBRE
sudo journalctl -u NOMBRE -f

# Nginx
sudo nginx -t
sudo systemctl reload nginx
sudo tail -f /var/log/nginx/error.log

# Red y recursos
sudo ss -tulpn
sudo lsof -i :3000
curl -I http://localhost
df -h
free -h

# Seguridad
sudo ufw status numbered
sudo apt update && sudo apt upgrade
```

Mantén esta guía como referencia, pero confirma siempre las rutas, nombres de servicios y dominios reales de tu VPS antes de ejecutar un comando.
