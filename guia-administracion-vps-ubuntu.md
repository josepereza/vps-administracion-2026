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

### Ver consumo por servicio con `systemd-cgtop`

`top` muestra procesos individuales, pero una API Node.js puede tener varios procesos y Nginx varios trabajadores. `systemd-cgtop` los agrupa por servicio, igual que los administra `systemd`:

```bash
# Vista en tiempo real; pulsa q para salir
sudo systemd-cgtop

# Una sola captura, ordenada por memoria
sudo systemd-cgtop --iterations=1 --order=memory
```

El resultado agrupa recursos de unidades como `mi-api.service`, `nginx.service` o `postgresql.service`. Si `mi-api.service` aparece usando mucha memoria, revisa sus logs y la aplicación; si `nginx.service` consume CPU durante un pico, compáralo con el número de conexiones mediante `ss`.

Una salida simplificada podría verse así:

```text
Control Group       Tasks   %CPU   Memory
/system.slice/mi-api.service     8    2.1%   210.5M
/system.slice/nginx.service      5    0.3%    18.0M
```

Los valores cambian mientras la carga cambia, así que busca tendencias y compara con el estado habitual del VPS. Esta herramienta solo observa: no reinicia, limita ni mata servicios. No decidas aumentar RAM o reiniciar una API por una sola lectura puntual.

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

### Detectar ciclos de reinicio de una API

Con `Restart=always`, `systemd` puede levantar una API después de que falle. Eso es útil, pero una aplicación que arranca y cae una y otra vez puede parecer disponible durante unos segundos. Consulta el contador de reinicios y el estado actual:

```bash
sudo systemctl show mi-api \
  -p ActiveState -p SubState -p NRestarts -p RestartUSec

sudo journalctl -u mi-api -b -n 60 --no-pager
```

Ejemplo:

```text
ActiveState=activating
SubState=auto-restart
NRestarts=7
RestartUSec=5s
```

`NRestarts=7` significa que `systemd` ha intentado reiniciar la unidad siete veces desde que se creó su estado actual; `auto-restart` indica que está esperando antes de otro intento. En una API estable, lo habitual es `ActiveState=active`, `SubState=running` y un contador que no aumenta mientras observas el servicio.

No solucionas un ciclo reiniciando repetidamente o ejecutando `reset-failed` sin más: revisa primero las últimas líneas del journal, variables de entorno, puerto `3000`, permisos y conexión a la base de datos. Un valor `NRestarts=0` tampoco descarta fallos de arranques anteriores; `journalctl -u mi-api -b` limita la revisión al arranque actual del VPS.

### Encontrar qué retrasa el arranque del VPS

Después de reiniciar, una API Node.js o Nginx puede tardar en estar lista porque otro servicio o una dependencia aún está arrancando. `systemd-analyze` permite investigarlo sin modificar nada:

```bash
# Las 15 unidades que más tiempo declararon tardar al arrancar
systemd-analyze blame | head -n 15

# Cadena de dependencias que llevó hasta una API concreta
systemd-analyze critical-chain mi-api.service
```

La primera orden puede mostrar una salida parecida a esta:

```text
3.421s postgresql.service
1.207s mi-api.service
 842ms nginx.service
```

`blame` ordena las unidades por su tiempo de inicio; `critical-chain` enseña la ruta de dependencias y los momentos en que se activaron. Por ejemplo, si `mi-api.service` espera a la red o a PostgreSQL, esa relación aparecerá en la cadena. Es útil para decidir si debes revisar una base de datos lenta, una dependencia de red o el tiempo de inicialización de tu aplicación.

**Precaución:** el primer puesto de `blame` no siempre es la causa de que el VPS tarde más: muchas unidades arrancan en paralelo. No desactives ni cambies el orden de un servicio solo por aparecer lento; confirma primero la cadena crítica y revisa los logs del servicio afectado.

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

### Revisar el límite de archivos abiertos de una API

Cada conexión HTTP, archivo de log o socket usa un descriptor de archivo. Si una API recibe muchas conexiones y llega a su límite, puede fallar con errores como `EMFILE: too many open files` aunque todavía tenga RAM disponible.

```bash
# Límite de la sesión actual de tu terminal
ulimit -n

# Límite real aplicado al servicio por systemd
sudo systemctl show mi-api -p LimitNOFILE

# Número de descriptores que usa ahora la API
PID=$(sudo systemctl show mi-api -p MainPID --value)
sudo ls /proc/$PID/fd | wc -l
```

Un resultado posible sería:

```text
LimitNOFILE=1024
87
```

El primer valor es el máximo permitido para el servicio y el segundo son los descriptores usados en ese momento. Si el uso se acerca al límite de forma constante, crea un ajuste específico para la API:

```bash
sudo systemctl edit mi-api
```

En el editor añade:

```ini
[Service]
LimitNOFILE=65536
```

Después aplica el cambio y verifica:

```bash
sudo systemctl daemon-reload
sudo systemctl restart mi-api
sudo systemctl show mi-api -p LimitNOFILE
```

No aumentes límites de todos los servicios sin necesidad. Primero confirma con logs y métricas que existe presión real; un límite mayor no arregla una fuga de conexiones, archivos o sockets que la aplicación haya dejado abiertos.

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

### Contar conexiones activas hacia Node.js con `ss`

Un puerto puede estar abierto y la API seguir teniendo problemas por demasiadas conexiones simultáneas. `ss` permite ver las conexiones TCP que llegan al puerto interno de Node.js:

```bash
# Puerto en escucha y proceso propietario
sudo ss -tanlp '( sport = :3000 )'

# Solo conexiones ya establecidas con la API
sudo ss -tanp state established '( sport = :3000 )'

# Contarlas, sin incluir la cabecera
sudo ss -tan state established '( sport = :3000 )' | tail -n +2 | wc -l
```

El último comando devuelve un número, por ejemplo `12`: son doce conexiones TCP activas entre Nginx u otros clientes y la API. Durante poco tráfico es normal que sea bajo; compáralo con el valor habitual cuando recibas un `502`, lentitud o consumo alto de archivos abiertos.

Para una vista rápida de los estados de conexión vinculados al puerto, usa:

```bash
sudo ss -tan '( sport = :3000 )' | awk 'NR > 1 {print $1}' | sort | uniq -c
```

Verás estados como `ESTAB` (conexión activa) o `TIME-WAIT` (conexión recién cerrada). Muchos `TIME-WAIT` por sí solos no son un fallo: pueden ser tráfico normal. No cierres conexiones a mano ni abras el puerto `3000` en el firewall para hacer esta prueba; ejecútala dentro del VPS y deja Nginx como único punto público.

### Entender `sport` y `dport` en filtros de `ss`

En una conexión hay dos extremos. Para la API que actúa como servidor, el extremo local es `127.0.0.1:3000` y el extremo remoto suele ser un puerto temporal de Nginx, por ejemplo `127.0.0.1:49822`.

```text
Nginx: 127.0.0.1:49822  ───►  API Node.js: 127.0.0.1:3000
        puerto remoto                 puerto local
```

- `sport` significa *source port* y, en la salida de `ss` de ese proceso, coincide con el puerto local.
- `dport` significa *destination port* y coincide con el puerto del otro extremo o cliente.

Por eso, para ver las conexiones que **recibe** la API local, el filtro correcto es:

```bash
sudo ss -tnp state established '( sport = :3000 )'
```

En cambio, `dport = :3000` sirve cuando tu VPS es el cliente que se conecta al puerto `3000` de otra máquina:

```bash
sudo ss -tnp state established '( dport = :3000 )'
```

Si el segundo comando no muestra nada en un VPS donde Node.js es el servidor, es normal: las conexiones entrantes tienen `3000` como puerto local (`sport`), no como puerto remoto. Usa `ss` solo para observar y diagnosticar; los filtros no abren puertos ni cambian la red.

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

### Medir tiempos de respuesta con `curl`

Una API puede responder con `200` y, aun así, tardar demasiado. Este comando descarta el cuerpo de la respuesta y muestra los tiempos principales:

```bash
curl -s -o /dev/null -w 'HTTP %{http_code}\nDNS: %{time_namelookup}s\nConexión: %{time_connect}s\nPrimer byte: %{time_starttransfer}s\nTotal: %{time_total}s\n' http://localhost/api/health
```

Un resultado típico en una ruta local sana sería:

```text
HTTP 200
DNS: 0.000s
Conexión: 0.001s
Primer byte: 0.024s
Total: 0.024s
```

`time_starttransfer` mide cuánto tarda en llegar el primer byte. Si aumenta, suele indicar que Node.js, una consulta a la base de datos o una dependencia externa está demorando la respuesta. Para separar la API del proxy, compara ambas rutas:

```bash
curl -s -o /dev/null -w 'API directa: %{time_total}s\n' http://127.0.0.1:3000/health
curl -s -o /dev/null -w 'Por Nginx: %{time_total}s\n' http://localhost/api/health
```

En una ruta de salud local los tiempos normalmente son de milisegundos; conserva una referencia de antes del despliegue y repite la prueba varias veces. No expongas `127.0.0.1:3000` a Internet solo para medirlo y no saques conclusiones con una única muestra.

### Comprobar una API tras desplegar con reintentos

Durante un despliegue, Node.js puede tardar unos segundos en estar disponible. `curl` puede esperar y reintentar una ruta de salud sin ocultar un error real:

```bash
curl --fail --silent --show-error \
  --retry 3 --retry-all-errors \
  --connect-timeout 5 --max-time 15 \
  http://localhost/api/health
```

`--fail` hace que los estados HTTP `4xx` y `5xx` cuenten como error; `--retry 3` repite la comprobación hasta tres veces y `--connect-timeout 5` evita esperar indefinidamente a que se abra la conexión. `--max-time 15` limita cada petición a quince segundos.

Si la ruta responde bien, verás su JSON habitual y el comando acabará con código `0`:

```bash
curl --fail --silent --show-error http://localhost/api/health
echo $?
```

El resultado esperado de `echo $?` es `0`. Puedes usarlo en un script de despliegue para detener el proceso si la API no está lista. Aplícalo solo a peticiones seguras de lectura, como `GET /health`: reintentar automáticamente un `POST` podría crear un pedido, un usuario u otra acción dos veces.

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

### Comprobar cabeceras HTTP básicas de seguridad

Antes de añadir reglas, mira las cabeceras que entrega tu web:

```bash
curl -I https://mi-dominio.ch
```

En el bloque `server` de Nginx puedes añadir estas cabeceras de bajo riesgo:

```nginx
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Después valida, recarga y comprueba que aparecen:

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -I https://mi-dominio.ch
```

`nosniff` reduce interpretaciones inesperadas de archivos, `SAMEORIGIN` limita que otras webs incrusten la página y `Referrer-Policy` reduce la información de URL enviada a otros sitios. No añadas una Content-Security-Policy genérica sin probarla: una política demasiado estricta puede romper Angular, APIs externas, fuentes o imágenes.

### Probar un dominio antes de cambiar DNS con `curl --resolve`

`--resolve` hace que `curl` conecte un dominio a una IP concreta solo para esa prueba. Es útil para comprobar un nuevo VPS o un bloque de Nginx antes de modificar el DNS público:

```bash
curl -I --resolve mi-dominio.ch:80:203.0.113.10 http://mi-dominio.ch/
curl -I --resolve mi-dominio.ch:443:203.0.113.10 https://mi-dominio.ch/
```

Lo esperado es una respuesta como `HTTP/1.1 200 OK` o una redirección `301/308` hacia HTTPS. Conserva el nombre del dominio en la URL: así Nginx recibe el `Host` correcto y puede seleccionar su bloque `server` adecuado.

Sustituye `203.0.113.10` por la IP real del VPS. No uses `-k` para ocultar errores de certificado; si HTTPS falla, revisa el certificado, `server_name` y la configuración TLS antes de publicar el cambio DNS.

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

### Identificar la versión exacta que vas a desplegar con Git

Antes de instalar dependencias o reiniciar la API, confirma el commit que está en el VPS. Esto evita investigar una incidencia sin saber qué código se publicó:

```bash
cd /var/www/mi-api

# No debe mostrar nada si no hay cambios locales sin guardar
git status --short

# Commit actual en formato corto y último cambio registrado
git rev-parse --short HEAD
git log -1 --format='%h | %ci | %s'
```

Ejemplo de resultado:

```text
a1b2c3d
a1b2c3d | 2026-11-22 10:34:18 +0100 | Corrige el proxy de pedidos
```

Si `git status --short` no devuelve nada, el árbol de trabajo está limpio. Si aparecen líneas como `M src/app.service.ts` o `?? archivo.txt`, hay cambios locales que no pertenecen necesariamente al commit actual. Revisa esos cambios antes de continuar.

Después de actualizar el código desde una fuente revisada, ejecuta de nuevo `git rev-parse --short HEAD` y anota ese identificador junto al despliegue. Si el servidor solo recibe `dist/` o archivos compilados y no contiene la carpeta `.git`, estos comandos no funcionarán: guarda entonces el hash del commit en tu proceso de despliegue, no inventes una versión.

**Precaución:** no ejecutes `git pull` a ciegas en producción cuando existan cambios locales; puede producir conflictos o mezclar código no probado. Publica un commit conocido, compila y verifica `/health` después del reinicio.

### Revisar dependencias de producción con npm

Antes de actualizar paquetes, mira qué dependencias tiene instalada la API y si npm detecta vulnerabilidades conocidas:

```bash
cd /var/www/mi-api
npm ls --omit=dev --depth=0
npm audit --omit=dev
```

`npm ls` muestra solo las dependencias directas necesarias en producción. `npm audit` revisa la información de vulnerabilidades conocida por npm; un resultado sin problemas termina con `found 0 vulnerabilities`.

Si necesitas instalar únicamente dependencias de ejecución en un servidor donde el proyecto ya está compilado, usa:

```bash
npm ci --omit=dev
```

No lo uses antes de `npm run build` si tu compilación necesita herramientas de desarrollo como Angular CLI o TypeScript. Revisa cada aviso de `npm audit` y actualiza primero en una rama o entorno de prueba. Evita `npm audit fix --force` en producción: puede introducir cambios incompatibles.

### Permisos de las carpetas

El usuario del servicio necesita leer el proyecto y, si guardas imágenes en el servidor, escribir solo en el directorio de uploads:

```bash
sudo chown -R www-data:www-data /var/www/mi-api/uploads
sudo chmod 750 /var/www/mi-api/uploads
```

No des permisos `777`: parecen resolver un error, pero abren una puerta de seguridad innecesaria.

### Encontrar el permiso que bloquea una ruta con `namei`

Un archivo puede tener permisos correctos y seguir dando `permission denied` si alguno de sus directorios padre no permite atravesarlo. `namei` muestra cada parte de la ruta con su propietario y permisos:

```bash
namei -l /var/www/mi-api/uploads/foto.jpg
```

Ejemplo abreviado:

```text
drwxr-xr-x root     root     /
drwxr-x--- deploy   deploy   var
drwxr-x--- www-data www-data uploads
-rw-r----- www-data www-data foto.jpg
```

Para que Nginx o tu servicio llegue al archivo, necesita permiso `x` en cada directorio de la ruta y permiso `r` en el archivo. Corrige solamente el componente que falla, por ejemplo el directorio de uploads:

```bash
sudo chown -R www-data:www-data /var/www/mi-api/uploads
sudo chmod 750 /var/www/mi-api/uploads
```

No uses `chmod -R 777` para ocultar el problema. Revisa primero con qué usuario se ejecuta el servicio (`systemctl show mi-api -p User`) y no concedas acceso a rutas que contienen secretos.

---

## 12. Base de datos: comprobaciones básicas

Nunca expongas MySQL (`3306`) o PostgreSQL (`5432`) al público salvo necesidad muy justificada y una restricción fuerte por IP/VPN.

### PostgreSQL

```bash
sudo systemctl status postgresql
sudo -u postgres psql
```

#### Comprobar si PostgreSQL está expuesto en la red

Primero mira la dirección real donde escucha el proceso y, después, la configuración que PostgreSQL tiene cargada:

```bash
# Socket TCP que está escuchando en el puerto de PostgreSQL
sudo ss -ltnp '( sport = :5432 )'

# Dirección y puerto configurados en PostgreSQL
sudo -u postgres psql -c "SHOW listen_addresses;"
sudo -u postgres psql -c "SHOW port;"
```

Para una base de datos usada únicamente por aplicaciones del mismo VPS, lo normal es ver `127.0.0.1:5432` y `listen_addresses` con `localhost`. Eso permite que Node.js se conecte localmente, pero bloquea conexiones directas desde Internet.

```text
LISTEN 0 200 127.0.0.1:5432 0.0.0.0:*
 listen_addresses
------------------
 localhost
```

Si aparece `0.0.0.0:5432` o `[::]:5432`, PostgreSQL acepta conexiones desde cualquier interfaz de red; no significa que cualquiera pueda entrar, pero aumenta la superficie de ataque. Además de `listen_addresses`, PostgreSQL usa `pg_hba.conf` para decidir quién puede autenticarse. No cambies esas opciones ni abras `5432` en UFW para "probar" una conexión: usa primero el socket local, una VPN o una regla de firewall limitada a una IP necesaria.

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

### Comprobar tareas automáticas con `systemd`

Ubuntu usa *timers* de `systemd` para lanzar tareas programadas, como actualizaciones o rotación de logs. Para ver las próximas ejecuciones y la última vez que se ejecutaron:

```bash
systemctl list-timers --all
systemctl list-timers --all | grep -E 'apt|logrotate'
```

Ejemplo simplificado:

```text
NEXT                         LAST                         UNIT
Wed 2026-11-11 06:25:00 CET  Tue 2026-11-10 06:25:02 CET  apt-daily.timer
```

Para inspeccionar una tarea concreta:

```bash
systemctl status apt-daily.timer --no-pager
systemctl status logrotate.timer --no-pager
```

`NEXT` indica cuándo volverá a ejecutarse y `LAST` cuándo terminó la última ejecución. Un timer puede estar activo aunque su tarea asociada haya fallado, así que revisa también su servicio si algo no se actualiza o no rota:

```bash
sudo systemctl status apt-daily.service --no-pager
sudo systemctl status logrotate.service --no-pager
```

No desactives timers del sistema sin saber qué protegen: podrías dejar de recibir actualizaciones de seguridad o acumular logs. Estos comandos solo consultan el estado y no modifican nada.

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
