# Práctica: Instalación, configuración y securización de Apache en Ubuntu 24.04

**Alumno:** Daviti Gvaramia
**Ciclo:** 2º CFGM Sistemas Microinformáticos y Redes (SMR)
**Módulo:** Aplicaciones Web
**Fecha:** 16/10/2026

---

## Índice

1. [Preparación del sistema](#apartado-1-preparación-del-sistema)
2. [Instalación de Apache](#apartado-2-instalación-de-apache)
3. [Comprobación del funcionamiento](#apartado-3-comprobación-del-funcionamiento)
4. [Comandos principales de administración](#apartado-4-comandos-principales-de-administración)
5. [Ficheros y directorios importantes](#apartado-5-ficheros-y-directorios-importantes)
6. [Modificaciones típicas del servicio](#apartado-6-modificaciones-típicas-del-servicio)
7. [Módulos de Apache](#apartado-7-módulos-de-apache)
8. [Varios sitios en un mismo servidor (Virtual Hosts)](#apartado-8-varios-sitios-en-un-mismo-servidor-virtual-hosts)
9. [Permisos y propietarios](#apartado-9-permisos-y-propietarios)
10. [Conclusiones y problemas encontrados](#conclusiones-y-problemas-encontrados)

### Entorno de trabajo

| Elemento | Valor |
|---|---|
| Sistema operativo | Ubuntu 24.04.5 LTS (*noble*) en máquina virtual |
| Servidor web | Apache/2.4.58 (Ubuntu) |
| Usuario | `daviti_` (con permisos de `sudo`) |
| Nombre del equipo | `daviti` |

---

## Apartado 1. Preparación del sistema

Antes de instalar cualquier servicio conviene tener el sistema al día. Así se instalan las versiones más recientes de los paquetes y se corrigen fallos de seguridad conocidos.

### 1.1. Actualizar la lista de paquetes

```bash
sudo apt update
```

![sudo apt update](img/01-apt-update.png)

`apt update` **no instala nada**: solo descarga de los repositorios la lista actualizada de paquetes y versiones disponibles. Es necesario hacerlo primero, porque si no, `apt` trabajaría con una lista antigua y podría intentar instalar versiones que ya no existen.

### 1.2. Actualizar los paquetes instalados

```bash
sudo apt upgrade -y
```

![sudo apt upgrade -y](img/02-apt-upgrade.png)

`apt upgrade` compara lo que tengo instalado con la lista que acabo de descargar y **actualiza todos los paquetes que tengan una versión nueva**. La opción `-y` responde "sí" automáticamente a la confirmación, para que no se quede esperando.

### 1.3. Comprobar la versión del sistema

```bash
lsb_release -a
```

```text
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 24.04.5 LTS
Release:        24.04
Codename:       noble
```

![Versión de Ubuntu](img/03-lsb-release.png)

`lsb_release -a` muestra la información de la distribución. Confirma que trabajo sobre **Ubuntu 24.04.5 LTS**, cuyo nombre en clave es *noble*, que es la versión que pide la práctica. Al ser LTS (*Long Term Support*) tiene soporte durante varios años, por eso es la recomendada para servidores. El mensaje *"No LSB modules are available"* es solo informativo y no indica ningún error.

---

## Apartado 2. Instalación de Apache

### 2.1. Instalar el paquete

```bash
sudo apt install apache2 -y
```

![sudo apt install apache2 -y](img/04-apt-install-apache2.png)

Con este comando se descarga e instala el servidor web Apache desde los repositorios oficiales de Ubuntu. Al terminar la instalación, Ubuntu **arranca el servicio automáticamente** y lo deja activado para que se inicie con el sistema.

### 2.2. Comprobar la versión instalada

```bash
apache2 -v
```

```text
Server version: Apache/2.4.58 (Ubuntu)
Server built:   2026-07-06T16:27:55
```

![apache2 -v](img/05-apache2-version.png)

La opción `-v` muestra la versión del servidor. Tengo instalada la **versión 2.4.58**, que pertenece a la rama 2.4, la estable y actual de Apache.

### Pregunta de reflexión: ¿Qué paquetes adicionales se han instalado como dependencias?

Al instalar `apache2`, `apt` instala también otros paquetes que Apache necesita para funcionar. En Ubuntu 24.04 los principales son:

| Paquete | Para qué sirve |
|---|---|
| `apache2-bin` | El programa ejecutable del servidor y sus módulos |
| `apache2-data` | Ficheros comunes: página por defecto, iconos, páginas de error |
| `apache2-utils` | Utilidades como `ab` (pruebas de rendimiento), `htpasswd` o `htcacheclean` |
| `libapr1t64`, `libaprutil1t64` | Librerías APR (*Apache Portable Runtime*), la base sobre la que funciona Apache |
| `libaprutil1-dbd-sqlite3`, `libaprutil1-ldap` | Extensiones de APR para bases de datos y LDAP |
| `liblua5.4-0` | Librería de Lua, que usa el módulo `mod_lua` |
| `ssl-cert` | Crea un certificado autofirmado de ejemplo para poder probar HTTPS |

Se pueden ver en la salida de `apt`, en la línea *"Se instalarán los siguientes paquetes adicionales"*, o después con:

```bash
grep -A3 "install apache2" /var/log/apt/history.log
```

---

## Apartado 3. Comprobación del funcionamiento

### 3.1. Estado del servicio

```bash
sudo systemctl status apache2
```

```text
● apache2.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/apache2.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-09-30 07:39:04 UTC; 2min 10s ago
   Main PID: 7632 (apache2)
      Tasks: 55 (limit: 4601)
     Memory: 5.2M (peak: 5.4M)
```

![Estado del servicio Apache](img/06-systemctl-status.png)

Lo que hay que fijarse en esta salida:

- **`Loaded: ... enabled`**: el servicio está activado para arrancar solo cada vez que se enciende la máquina.
- **`Active: active (running)`**: Apache está funcionando ahora mismo.
- **`Main PID: 7632`**: es el proceso principal. Debajo aparecen otros procesos hijos (7633, 7635), que son los que atienden las peticiones de los clientes.
- Al final aparece el aviso **`AH00558: Could not reliably determine the server's fully qualified domain name`**. No es un error: Apache funciona, pero avisa de que no tiene definido un `ServerName`. Lo soluciono en el [apartado 6.3](#63-definir-el-nombre-del-servidor).

### 3.2. Puertos en escucha

```bash
sudo ss -tulpn | grep apache2
```

```text
tcp   LISTEN 0   511   *:80   *:*   users:(("apache2",pid=7635,fd=4),("apache2",pid=7633,fd=4),("apache2",pid=7632,fd=4))
```

![Puertos en escucha](img/07-ss-puertos.png)

`ss` muestra los *sockets* (conexiones) del sistema. Las opciones significan: `-t` TCP, `-u` UDP, `-l` solo los que están escuchando, `-p` el proceso que lo usa y `-n` números en vez de nombres. Con `grep apache2` filtro solo las líneas de Apache.

El resultado confirma que **Apache escucha en el puerto 80 (HTTP) por TCP** y en todas las interfaces de red (`*`), así que acepta conexiones tanto desde la propia máquina como desde otros equipos.

### 3.3. Prueba desde el terminal

```bash
curl -I http://localhost
```

```text
HTTP/1.1 200 OK
Server: Apache/2.4.58 (Ubuntu)
Content-Length: 10671
Content-Type: text/html
```

![curl -I http://localhost](img/08-curl-localhost.png)

`curl -I` hace una petición al servidor y muestra **solo las cabeceras** de la respuesta, sin el contenido de la página. El código **`200 OK`** significa que la petición ha funcionado y que Apache ha devuelto la página. La cabecera `Server` confirma que es Apache quien responde, y `Content-Length: 10671` es el tamaño de la página por defecto de Ubuntu (*Apache2 Ubuntu Default Page*).

Desde el navegador, entrando en `http://IP_DE_MI_SERVIDOR` (la IP se consulta con `ip a`), aparece esa misma página por defecto.

### 3.4. Firewall

```bash
sudo ufw status
```

```text
Status: inactive
```

![sudo ufw status](img/09-ufw-status.png)

UFW (*Uncomplicated Firewall*) es el cortafuegos de Ubuntu. En mi máquina está **inactivo**, por eso Apache ya era accesible sin hacer nada. Aun así, añado la regla para dejarlo preparado por si se activa más adelante:

```bash
sudo ufw allow 'Apache'
```

```text
Rules updated
Rules updated (v6)
```

![sudo ufw allow 'Apache'](img/10-ufw-allow-apache.png)

La regla se añade dos veces: una para IPv4 y otra para IPv6 (`v6`). Mientras el firewall esté desactivado la regla no tiene efecto, pero quedará aplicada en cuanto se active con `sudo ufw enable`.

### Pregunta de reflexión: ¿Qué diferencia hay entre los perfiles Apache, Apache Full y Apache Secure?

Son perfiles que Apache registra en UFW para no tener que abrir los puertos a mano. Se pueden ver con `sudo ufw app list`.

| Perfil | Puertos que abre | Cuándo usarlo |
|---|---|---|
| **Apache** | 80/tcp (HTTP) | Solo web sin cifrar. Es el que he usado en esta práctica |
| **Apache Secure** | 443/tcp (HTTPS) | Solo web cifrada con SSL/TLS |
| **Apache Full** | 80/tcp y 443/tcp | HTTP y HTTPS a la vez. Lo normal en un servidor real, para redirigir de HTTP a HTTPS |

---

## Apartado 4. Comandos principales de administración

Apache se gestiona con dos tipos de herramientas: **`systemctl`**, que controla el servicio dentro del sistema, y las herramientas propias de Apache (**`apache2ctl`** y los comandos **`a2...`**).

| Comando | Función | Explicación |
|---|---|---|
| `sudo systemctl start apache2` | Inicia el servicio | Arranca Apache si estaba parado |
| `sudo systemctl stop apache2` | Detiene el servicio | Para Apache; la web deja de estar disponible |
| `sudo systemctl restart apache2` | Reinicia el servicio | Lo para y lo vuelve a arrancar. Corta las conexiones abiertas |
| `sudo systemctl reload apache2` | Recarga la configuración | Lee de nuevo los ficheros de configuración **sin cortar** a los usuarios conectados |
| `sudo systemctl enable apache2` | Arranque automático | Hace que Apache se inicie solo al encender el equipo |
| `sudo systemctl disable apache2` | Quita el arranque automático | Apache ya no se iniciará solo, habrá que arrancarlo a mano |
| `apache2ctl configtest` | Comprueba la sintaxis | Revisa la configuración sin aplicarla. Si todo está bien responde `Syntax OK` |
| `apache2ctl -S` | Muestra los virtual hosts | Lista los sitios cargados, en qué puerto y desde qué fichero |
| `apache2ctl -M` | Lista los módulos cargados | Indica si cada módulo es `static` (incluido en el programa) o `shared` (cargado aparte) |
| `a2enmod` / `a2dismod` | Activa / desactiva módulos | Crean o borran el enlace en `mods-enabled/` |
| `a2ensite` / `a2dissite` | Activa / desactiva sitios | Crean o borran el enlace en `sites-enabled/` |
| `a2enconf` / `a2disconf` | Activa / desactiva configuraciones | Crean o borran el enlace en `conf-enabled/` |

A lo largo de la práctica he usado todos estos comandos: `configtest` y `reload` después de cada cambio (apartados 6 y 8), `-M`, `a2enmod` y `a2dismod` en el apartado 7, y `-S`, `a2ensite` y `a2dissite` en el apartado 8.

> **Buena costumbre:** ejecutar siempre `apache2ctl configtest` **antes** de recargar o reiniciar. Si hay un error de sintaxis y se reinicia, Apache no vuelve a arrancar y la web se queda caída.

### Pregunta de reflexión: ¿Cuándo conviene usar reload en lugar de restart?

**`reload`** se usa cuando solo he cambiado la configuración (un virtual host, un `ServerName`, una página de error…). Apache relee los ficheros y aplica los cambios mientras termina de atender las peticiones que ya tenía abiertas, así que **los usuarios no notan ningún corte**. En un servidor en producción es la opción preferida.

**`restart`** es necesario cuando el cambio afecta a cómo arranca el proceso: por ejemplo, al **activar o desactivar módulos** (`a2enmod` lo indica en su mensaje: *"you need to run: systemctl restart apache2"*), al cambiar el MPM o si Apache se ha quedado bloqueado. Como para el servicio por completo, corta todas las conexiones.

---

## Apartado 5. Ficheros y directorios importantes

### 5.1. Estructura de `/etc/apache2/`

```bash
ls -l /etc/apache2/
```

```text
-rw-r--r-- 1 root root  7178 Jul  6 16:27 apache2.conf
drwxr-xr-x 2 root root  4096 Sep 30 07:38 conf-available
drwxr-xr-x 2 root root  4096 Sep 30 07:39 conf-enabled
-rw-r--r-- 1 root root  1782 Mar 18  2024 envvars
-rw-r--r-- 1 root root 31063 Mar 18  2024 magic
drwxr-xr-x 2 root root 12288 Sep 30 07:39 mods-available
drwxr-xr-x 2 root root  4096 Sep 30 07:39 mods-enabled
-rw-r--r-- 1 root root   274 Mar 18  2024 ports.conf
drwxr-xr-x 2 root root  4096 Sep 30 07:39 sites-available
drwxr-xr-x 2 root root  4096 Sep 30 07:39 sites-enabled
```

![Contenido de /etc/apache2](img/11-ls-etc-apache2.png)

Toda la configuración de Apache está en `/etc/apache2/`. Los ficheros (empiezan por `-`) y directorios (empiezan por `d`) pertenecen a **root** y solo él puede modificarlos (`rw-r--r--`), por eso hay que usar `sudo` para editarlos. Se ve que Ubuntu organiza la configuración en tres parejas de directorios `*-available` / `*-enabled`.

### 5.2. Tabla de ficheros y directorios clave

| Ruta | Descripción (con mis palabras) |
|---|---|
| `/etc/apache2/apache2.conf` | Fichero de configuración **principal**. Define los ajustes globales (tiempo de espera, usuario, permisos por defecto de directorios) y es el que incluye a todos los demás ficheros |
| `/etc/apache2/ports.conf` | Indica en qué **puertos escucha** Apache con la directiva `Listen` (por defecto el 80, y el 443 si se activa SSL) |
| `/etc/apache2/sites-available/` | Aquí se guardan **todos los sitios web definidos**, estén activos o no. Cada sitio es un fichero `.conf` con su `<VirtualHost>` |
| `/etc/apache2/sites-enabled/` | Contiene solo los **sitios activos**. No son copias, sino enlaces simbólicos a los ficheros de `sites-available` |
| `/etc/apache2/mods-available/` y `mods-enabled/` | Igual que los sitios, pero para **módulos**: en `available` están todos los que se pueden usar y en `enabled` los que Apache carga al arrancar. Cada módulo tiene un `.load` (carga el módulo) y a veces un `.conf` (sus ajustes) |
| `/etc/apache2/conf-available/` y `conf-enabled/` | Fragmentos de **configuración extra** que se pueden activar o desactivar por separado (por ejemplo, el `servername.conf` que creo en el apartado 6.3) |
| `/etc/apache2/envvars` | Define **variables de entorno** que usa Apache, como el usuario y grupo con el que se ejecuta (`www-data`) o la carpeta de logs |
| `/var/www/html/` | Carpeta raíz por defecto (**DocumentRoot**). Lo que se pone aquí es lo que se ve al entrar en `http://servidor/` |
| `/var/log/apache2/access.log` | Registro de **todas las visitas**: IP del cliente, fecha, página pedida, código de respuesta y navegador |
| `/var/log/apache2/error.log` | Registro de **errores y avisos**. Es lo primero que hay que mirar cuando algo falla |

### 5.3. Comprobar los enlaces simbólicos de `sites-enabled`

```bash
ls -l /etc/apache2/sites-enabled/
```

```text
lrwxrwxrwx 1 root root 35 Sep 30 07:39 000-default.conf -> ../sites-available/000-default.conf
```

![Enlaces simbólicos en sites-enabled](img/12-sites-enabled-enlaces.png)

La `l` al principio de los permisos y la flecha `->` indican que **`000-default.conf` es un enlace simbólico** que apunta al fichero real que está en `sites-available`. El fichero existe una sola vez; en `sites-enabled` solo hay un "acceso directo" a él.

### Pregunta de reflexión: ¿Por qué Apache usa enlaces simbólicos entre los directorios `*-available` y `*-enabled`?

Porque permite **activar y desactivar** sitios, módulos o configuraciones **sin borrar ni copiar ficheros**:

- **Activar** = crear el enlace en `*-enabled` (lo hacen `a2ensite`, `a2enmod`, `a2enconf`).
- **Desactivar** = borrar solo el enlace (`a2dissite`, `a2dismod`, `a2disconf`). El fichero original sigue en `*-available`, con toda su configuración, listo para volver a activarlo.

Además, como el fichero solo existe una vez, **no hay copias desactualizadas**: si edito `sites-available/sitio1.conf`, el cambio se aplica directamente porque el enlace apunta a ese mismo fichero. Apache solo lee lo que hay en los directorios `*-enabled`.

---

## Apartado 6. Modificaciones típicas del servicio

### 6.0. Copia de seguridad antes de modificar

```bash
sudo cp /etc/apache2/apache2.conf /etc/apache2/apache2.conf.bak
```

![Copia de seguridad de apache2.conf](img/13-backup-apache2conf.png)

Antes de tocar cualquier fichero de configuración hago una copia con la extensión `.bak`. Si me equivoco y Apache deja de funcionar, puedo restaurar el original con `sudo cp apache2.conf.bak apache2.conf`. Es una costumbre básica de cualquier administrador de sistemas.

### 6.1. Cambiar la página de inicio

```bash
echo "<h1>Servidor de DAVITI</h1>" | sudo tee /var/www/html/index.html
```

```text
<h1>Servidor de DAVITI</h1>
```

![Nueva página de inicio](img/14-nueva-index.png)

`echo` genera el texto HTML y `tee` lo escribe en el fichero `index.html`, sustituyendo la página por defecto de Ubuntu. Se usa `sudo tee` en lugar de `sudo echo ... >` porque la redirección `>` la hace mi usuario normal, que no tiene permiso para escribir en `/var/www/html/`; en cambio `tee` sí se ejecuta como root.

El cambio se comprueba en la captura del apartado 6.2: la respuesta ya no mide 10671 bytes como la página de Ubuntu, sino **`Content-Length: 28`**, que es exactamente el tamaño de mi nueva página.

### 6.2. Cambiar el puerto de escucha al 8080

Edito los dos ficheros que indican el puerto:

```bash
sudo nano /etc/apache2/ports.conf
```

![Editar ports.conf](img/15-nano-ports-conf.png)

En `ports.conf` cambio `Listen 80` por **`Listen 8080`**. Esto le dice a Apache en qué puerto tiene que escuchar.

```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

![Editar 000-default.conf](img/16-nano-000-default.png)

En el sitio por defecto cambio `<VirtualHost *:80>` por **`<VirtualHost *:8080>`**, para que el sitio responda en el nuevo puerto. Hay que cambiar los dos ficheros: si solo cambio uno, Apache escucharía en un puerto pero el sitio estaría configurado en otro.

Después compruebo la sintaxis, recargo y pruebo:

```bash
sudo apache2ctl configtest
sudo systemctl reload apache2
curl -I http://localhost:8080
```

```text
Syntax OK
HTTP/1.1 200 OK
Server: Apache/2.4.58 (Ubuntu)
Content-Length: 28
Content-Type: text/html
```

![Apache funcionando en el puerto 8080](img/17-puerto-8080-curl.png)

`configtest` devuelve **`Syntax OK`** (con el aviso AH00558, que aún no había corregido) y la petición a `localhost:8080` responde **`200 OK`**, así que el cambio de puerto funciona. Al final de la captura se ve que vuelvo a abrir `000-default.conf` para **dejar otra vez el puerto 80**, como pide la práctica (lo compruebo en el apartado 6.5, donde `ss` vuelve a mostrar `*:80`).

### 6.3. Definir el nombre del servidor

Para quitar el aviso *"Could not reliably determine the server's fully qualified domain name"* creo un fragmento de configuración con la directiva `ServerName`:

```bash
echo "ServerName localhost" | sudo tee /etc/apache2/conf-available/servername.conf
sudo a2enconf servername
sudo systemctl reload apache2
```

```text
ServerName localhost
Conf servername already enabled
```

![Definir ServerName](img/18-servername.png)

Lo creo en `conf-available` y lo activo con `a2enconf`, en vez de escribirlo directamente en `apache2.conf`, así el fichero principal queda limpio y puedo desactivarlo cuando quiera. El mensaje *"already enabled"* indica que ya lo había activado en un intento anterior, así que no hacía falta repetirlo.

Comprobación:

```bash
sudo apache2ctl configtest
```

```text
Syntax OK
```

![configtest sin aviso](img/19-configtest-sin-aviso.png)

Ahora `configtest` responde solo **`Syntax OK`**, sin el aviso AH00558. Si lo comparo con la captura del apartado 6.2, se ve claramente que el problema ha desaparecido.

```bash
sudo systemctl reload apache2
sudo systemctl status apache2
```

![Recarga y estado del servicio](img/20-reload-status.png)

En el estado del servicio aparece la línea **`ExecReload=/usr/sbin/apachectl graceful (code=exited, status=0/SUCCESS)`**: la recarga se ha hecho de forma *graceful* (sin cortar conexiones) y ha terminado correctamente. También se ve que el proceso principal sigue siendo el mismo (**PID 7632**) desde el 30 de septiembre: con `reload` Apache nunca se ha llegado a parar.

### 6.4. Cambiar el correo del administrador

```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

![Abrir 000-default.conf](img/21-nano-000-default.png)

Dentro del VirtualHost cambio la directiva `ServerAdmin`:

```apache
ServerAdmin daviti@apache2.com
```

![ServerAdmin modificado](img/23-serveradmin.png)

`ServerAdmin` es la dirección de correo del responsable del servidor. Apache la muestra en algunas páginas de error que genera él mismo (por ejemplo, en un error 500), para que el usuario sepa a quién avisar. Por defecto viene `webmaster@localhost` y la he cambiado por **`daviti@apache2.com`**.

### 6.5. Personalizar la página de error 404

En el mismo fichero añado la directiva `ErrorDocument` antes de cerrar el VirtualHost:

```apache
ErrorDocument 404 /404.html
```

![000-default.conf con ErrorDocument](img/22-000-default-errordocument.png)

En la captura se ve el fichero completo con los tres cambios: **`<VirtualHost *:80>`** (puerto ya restaurado), **`ServerAdmin daviti@apache2.com`** y **`ErrorDocument 404 /404.html`**. Esta última línea indica que, cuando alguien pida una página que no existe (error 404), Apache le muestre mi página `/404.html` en lugar de la página de error genérica.

Creo la página de error y compruebo la sintaxis:

```bash
sudo apache2ctl configtest
echo "<h1>Error 404: Página no encontrada - Servidor de Daviti</h1>" | sudo tee /var/www/html/404.html
```

![Crear la página 404](img/24-pagina-404.png)

Recargo Apache y lo pruebo pidiendo una ruta que no existe:

```bash
sudo ss -tulpn | grep apache2
curl -I http://localhost/ruta-inexistente
```

```text
tcp  LISTEN 0  511  *:80  *:*  users:(("apache2",pid=10654,fd=4),...)
HTTP/1.1 404 Not Found
Server: Apache/2.4.58 (Ubuntu)
Content-Length: 61
Content-Type: text/html
```

![Error 404 personalizado](img/25-curl-404-puerto80.png)

Esta captura confirma dos cosas:

1. **Apache vuelve a escuchar en el puerto 80** (`*:80`), así que he deshecho bien el cambio del apartado 6.2.
2. La ruta inexistente devuelve **`404 Not Found`**, pero con **`Content-Length: 61`** y una cabecera **`Last-Modified`** (fecha de modificación de un fichero). La página de error genérica de Apache es más grande y la genera él mismo, sin fecha de fichero; aquí, en cambio, Apache está enviando un fichero pequeño que yo he creado. Es decir, el `ErrorDocument` funciona y el visitante ve mi mensaje personalizado.

---

## Apartado 7. Módulos de Apache

Apache tiene un núcleo pequeño y casi todas sus funciones (HTTPS, redirecciones, autenticación, PHP…) se añaden con **módulos**. Así solo se carga lo que se necesita, lo que ahorra memoria y reduce los posibles fallos de seguridad.

### 7.1. Módulos cargados y disponibles

```bash
apache2ctl -M
ls /etc/apache2/mods-available/
```

![Módulos cargados y disponibles](img/26-modulos-cargados.png)

`apache2ctl -M` muestra los módulos **que están funcionando ahora**. Hay dos tipos:

- **`static`**: vienen incluidos dentro del propio programa de Apache y no se pueden desactivar (`core_module`, `http_module`, `log_config_module`…).
- **`shared`**: se cargan aparte desde `mods-enabled/` y se pueden activar o desactivar (`alias_module`, `dir_module`, `mime_module`, `mpm_event_module`…).

El listado de `mods-available/` es mucho más largo: son **todos los módulos que tengo instalados** y podría activar. Se ve que muchos aparecen dos veces, con `.load` y con `.conf`.

### 7.2. Activar módulos habituales y desactivar uno

```bash
sudo a2enmod rewrite
sudo a2enmod headers
sudo a2enmod ssl
sudo systemctl restart apache2
apache2ctl -M | grep -E "rewrite|headers|ssl"
```

```text
Enabling module rewrite.
Enabling module headers.
Considering dependency mime for ssl:
Module mime already enabled
Considering dependency socache_shmcb for ssl:
Enabling module socache_shmcb.
Enabling module ssl.

 headers_module (shared)
 rewrite_module (shared)
 ssl_module (shared)
```

```bash
sudo a2dismod status
sudo systemctl restart apache2
apache2ctl -M | grep status
```

```text
Module status disabled.
```

![Activar y desactivar módulos](img/27-a2enmod-a2dismod.png)

Lo que se observa en esta captura:

- Cada `a2enmod` avisa de que hay que hacer **`systemctl restart apache2`**: los módulos se cargan al arrancar, por eso aquí no basta con `reload`.
- Al activar **`ssl`**, `a2enmod` revisa sus **dependencias** automáticamente: `mime` ya estaba activo y `socache_shmcb` (la caché de sesiones SSL) no, así que lo activa también.
- Después del reinicio, `grep` confirma que **`headers`, `rewrite` y `ssl` están cargados**.
- Al desactivar **`status`** y reiniciar, `apache2ctl -M | grep status` **no devuelve nada**: el módulo ya no se carga. El efecto es que la página de estado `http://servidor/server-status` deja de existir.

### 7.3. Tabla de módulos

| Módulo | Función | Explicación |
|---|---|---|
| `mod_rewrite` | Reescritura de URL | Cambia las URL según reglas. Se usa para URL "bonitas" (`/producto/5` en vez de `/producto.php?id=5`) o para redirigir de HTTP a HTTPS |
| `mod_headers` | Manipulación de cabeceras HTTP | Permite añadir, cambiar o quitar cabeceras de las peticiones y respuestas. Muy útil para seguridad (`X-Frame-Options`, `Content-Security-Policy`…) |
| `mod_ssl` | Soporte HTTPS | Cifra la comunicación entre navegador y servidor con SSL/TLS. Necesita un certificado |
| `mod_dir` | Ficheros índice | Hace que al pedir una carpeta (`/`) se muestre automáticamente su `index.html` |
| `mod_alias` | Alias y redirecciones | Asocia una URL a una carpeta distinta del DocumentRoot (`Alias`) o redirige a otra dirección (`Redirect`) |
| `mod_auth_basic` | Autenticación básica | Pide usuario y contraseña para entrar en una zona de la web. Los usuarios se crean con `htpasswd` |
| `mod_status` | Página de estado | Muestra en `/server-status` las conexiones activas, la carga y los procesos de Apache. Lo he desactivado en esta práctica |
| `mod_mime` | Tipos de fichero | Indica al navegador qué tipo de contenido envía según la extensión (`.html` es `text/html`, `.png` es `image/png`) |
| `mod_mpm_event` | Gestión de procesos | Decide cómo reparte Apache el trabajo entre procesos e hilos. `event` es el que usa Ubuntu por defecto, el más eficiente |
| `mod_php` / PHP-FPM | Ejecución de PHP | Permite ejecutar páginas dinámicas en PHP. No viene instalado por defecto (opcional) |

### Pregunta de reflexión: ¿Qué ocurre si desactivas un módulo del que depende otro?

Que el módulo que depende de él **dejaría de funcionar** y Apache podría no arrancar, porque las directivas de ese módulo ya no se entenderían.

Las herramientas de Debian/Ubuntu ayudan a evitarlo:

- **Al activar**, `a2enmod` activa también las dependencias. Lo he visto en la práctica: al activar `ssl` se activó solo `socache_shmcb`.
- **Al desactivar**, `a2dismod` comprueba si otro módulo activo depende del que quiero quitar. Si es así, **muestra un error y no lo desactiva** (*"The following modules depend on ... and need to be disabled first"*): primero hay que desactivar los módulos que dependen de él. Solo se puede saltar esta comprobación forzándolo con `-f`, algo que no conviene hacer.

Aun así hay que tener cuidado: si un sitio usa directivas de un módulo desactivado (por ejemplo, `RewriteRule` sin `mod_rewrite`), `apache2ctl configtest` dará un error de sintaxis del tipo *"Invalid command 'RewriteRule'"*. Por eso siempre hay que hacer `configtest` antes de reiniciar.

---

## Apartado 8. Varios sitios en un mismo servidor (Virtual Hosts)

Un **virtual host** permite que un solo servidor Apache, con una sola IP, aloje **varias webs distintas**. Apache sabe cuál mostrar mirando el nombre de dominio que pide el navegador. En este apartado creo dos sitios: `sitio1.local` y `sitio2.local`.

### 8.1. Crear los directorios y las páginas

```bash
sudo mkdir -p /var/www/sitio1 /var/www/sitio2
echo "<h1>Bienvenido a SITIO 1</h1>" | sudo tee /var/www/sitio1/index.html
echo "<h1>Bienvenido a SITIO 2</h1>" | sudo tee /var/www/sitio2/index.html
```

![Directorios y páginas de los sitios](img/28-vhost-directorios.png)

Cada sitio necesita su propia carpeta, que será su **DocumentRoot**. `mkdir -p` crea las dos carpetas de una vez (y no da error si ya existen). Luego pongo en cada una un `index.html` con un texto diferente, para poder distinguir fácilmente qué sitio estoy viendo.

### 8.2. Crear los ficheros de configuración

```bash
sudo nano /etc/apache2/sites-available/sitio1.conf
```

```apache
<VirtualHost *:80>
    ServerName sitio1.local
    ServerAdmin admin@sitio1.local
    DocumentRoot /var/www/sitio1
    <Directory /var/www/sitio1>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/sitio1_error.log
    CustomLog ${APACHE_LOG_DIR}/sitio1_access.log combined
</VirtualHost>
```

![sitio1.conf](img/29-sitio1-conf.png)

Qué hace cada directiva:

| Directiva | Significado |
|---|---|
| `<VirtualHost *:80>` | El sitio responde en el puerto 80, en cualquier IP del servidor |
| `ServerName sitio1.local` | **La clave del virtual host**: Apache usa este sitio cuando el navegador pide `sitio1.local` |
| `ServerAdmin` | Correo del responsable de este sitio |
| `DocumentRoot /var/www/sitio1` | Carpeta de donde salen los ficheros de este sitio |
| `Options -Indexes` | Si falta el `index.html`, **no muestra el listado de ficheros** de la carpeta (medida de seguridad) |
| `Options +FollowSymLinks` | Permite seguir enlaces simbólicos dentro de la carpeta |
| `AllowOverride None` | Ignora los ficheros `.htaccess`. Es más seguro y más rápido |
| `Require all granted` | Permite el acceso a todo el mundo |
| `ErrorLog` / `CustomLog` | Cada sitio guarda **sus propios logs**, así es más fácil saber de qué web viene un error |

```bash
sudo nano /etc/apache2/sites-available/sitio2.conf
```

![sitio2.conf](img/30-sitio2-conf.png)

`sitio2.conf` es igual pero cambiando `sitio1` por `sitio2` en el nombre, el correo, la carpeta y los logs.

### 8.3. Activar los sitios y desactivar el de por defecto

```bash
sudo a2ensite sitio1.conf sitio2.conf
sudo a2dissite 000-default.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
apache2ctl -S
```

```text
Enabling site sitio1.
Enabling site sitio2.
Syntax OK

VirtualHost configuration:
*:80                   is a NameVirtualHost
         default server sitio1.local (/etc/apache2/sites-enabled/sitio1.conf:1)
         port 80 namevhost sitio1.local (/etc/apache2/sites-enabled/sitio1.conf:1)
         port 80 namevhost sitio2.local (/etc/apache2/sites-enabled/sitio2.conf:1)
...
User: name="www-data" id=33
Group: name="www-data" id=33
```

`a2ensite` crea los enlaces de los dos sitios en `sites-enabled` y `a2dissite` quita el del sitio por defecto. Después compruebo la sintaxis y recargo (para activar sitios basta con `reload`, como indica el propio `a2ensite`).

La salida de `apache2ctl -S` confirma que todo está bien:

- **`*:80 is a NameVirtualHost`**: en el puerto 80 hay varios sitios diferenciados por nombre.
- Aparecen **`sitio1.local` y `sitio2.local`**, cada uno con su fichero de configuración.
- **`000-default` ya no aparece**, así que se ha desactivado correctamente.
- **`default server sitio1.local`**: si llega una petición con un nombre que no coincide con ningún sitio (por ejemplo, entrando por la IP), Apache mostrará `sitio1`, porque es el primero que carga (orden alfabético).
- Apache se ejecuta como **`www-data`**, dato que uso en el apartado 9.

### 8.4. Resolución de nombres

Los dominios `.local` no existen en Internet, así que hay que decirle al equipo a qué IP corresponden. Lo hago añadiendo esta línea en `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

```text
127.0.0.1     sitio1.local sitio2.local
```

`/etc/hosts` se consulta **antes que el DNS**, así que al escribir `sitio1.local` el sistema lo resuelve a `127.0.0.1` (el propio servidor). Para probar desde otro equipo de la red, en su `hosts` habría que poner la IP real del servidor en lugar de `127.0.0.1` (en Windows el fichero está en `C:\Windows\System32\drivers\etc\hosts`).

### 8.5. Pruebas

```bash
curl http://sitio1.local
curl http://sitio2.local
```

```text
<h1>Bienvenido a SITIO 1</h1>
<h1>Bienvenido a SITIO 2</h1>
```

![Virtual hosts funcionando y apache2ctl -S](img/31-a2ensite-apache2ctl-S-curl.png)

Cada dominio devuelve **su propia página**, aunque los dos están en el mismo servidor, en la misma IP y en el mismo puerto. Los virtual hosts funcionan correctamente.

### Pregunta de reflexión: ¿Qué diferencia hay entre virtual hosts basados en nombre y basados en IP? ¿Cuál has usado?

| | Basados en **nombre** | Basados en **IP** |
|---|---|---|
| Cómo distingue los sitios | Por el nombre de dominio que envía el navegador en la cabecera `Host` | Por la dirección IP a la que llega la petición |
| IPs necesarias | Una sola IP para todos los sitios | Una IP distinta para cada sitio |
| Configuración | `<VirtualHost *:80>` + `ServerName` | `<VirtualHost 192.168.1.10:80>`, `<VirtualHost 192.168.1.11:80>`… |
| Uso actual | Es lo habitual (hosting compartido, varias webs en un servidor) | Poco usado; solo en casos especiales |

**Yo he usado virtual hosts basados en nombre.** Los dos sitios usan `<VirtualHost *:80>` y se diferencian solo por su `ServerName`. Lo confirma la línea `*:80 is a NameVirtualHost` de `apache2ctl -S`.

---

## Apartado 9. Permisos y propietarios

### 9.1. Usuario con el que se ejecuta Apache

```bash
ps aux | grep apache2
grep -E "APACHE_RUN_(USER|GROUP)" /etc/apache2/envvars
ls -ld /var/www/sitio1 /var/www/sitio1/index.html
```

```text
root      11526  ...  /usr/sbin/apache2 -k start
www-data  11783  ...  /usr/sbin/apache2 -k start
www-data  11784  ...  /usr/sbin/apache2 -k start

export APACHE_RUN_USER=www-data
export APACHE_RUN_GROUP=www-data

drwxr-xr-x 2 root root 4096 Oct 16 07:40 /var/www/sitio1
-rw-r--r-- 1 root root   30 Oct 16 07:40 /var/www/sitio1/index.html
```

![Procesos, usuario de Apache y permisos](img/32-ps-envvars-permisos.png)

Esta captura muestra cómo funciona Apache por dentro:

- Hay **un proceso padre que se ejecuta como `root`** (PID 11526). Lo necesita para poder abrir el puerto 80, porque los puertos por debajo del 1024 solo los puede usar root.
- Los **procesos hijos se ejecutan como `www-data`** (11783, 11784). Son los que atienden a los visitantes. Así, si alguien encontrara un fallo en la web, solo tendría los permisos de `www-data`, no los de root.
- En `envvars` se ve que este usuario y grupo están definidos en **`APACHE_RUN_USER`** y **`APACHE_RUN_GROUP`**.
- La carpeta y el `index.html` de `sitio1` **pertenecen a `root`**, porque los creé con `sudo`.

> En la salida de `ps` también aparecen varios procesos `systemctl status apache2` en estado `T` (detenido). No son de Apache: son consultas de estado que dejé suspendidas al pulsar `Ctrl+Z` en vez de `q`. Lo explico en [problemas encontrados](#problemas-encontrados-y-cómo-los-resolví).

### 9.2. Aplicar una política de permisos razonable

```bash
sudo chown -R www-data:www-data /var/www/sitio1
sudo find /var/www/sitio1 -type d -exec chmod 755 {} \;
sudo find /var/www/sitio1 -type f -exec chmod 644 {} \;
```

![chown y chmod](img/33-chown-chmod.png)

- **`chown -R www-data:www-data`** cambia el propietario y el grupo de la carpeta y de todo su contenido (`-R` = recursivo) a `www-data`.
- **`find ... -type d -exec chmod 755`** busca solo los **directorios** y les pone permisos 755.
- **`find ... -type f -exec chmod 644`** busca solo los **ficheros** y les pone permisos 644.

Se usa `find` en lugar de `chmod -R` porque los directorios y los ficheros necesitan permisos distintos: un `chmod -R 755` daría permiso de ejecución a todos los ficheros, cosa que no deben tener.

### 9.3. Prueba del error 403

```bash
sudo chmod 000 /var/www/sitio1/index.html
curl -I http://sitio1.local
sudo tail /var/log/apache2/sitio1_error.log
```

Al quitar todos los permisos al `index.html`, Apache (que trabaja como `www-data`) ya no puede leerlo y responde **`403 Forbidden`**. En el log de errores del sitio aparece una línea con el código **`AH00132: file permissions deny server access: /var/www/sitio1/index.html`**, que indica exactamente qué fichero no se ha podido leer y por qué. Esto demuestra que **los logs son la primera herramienta para diagnosticar un fallo**.

Después restauro los permisos correctos:

```bash
sudo chmod 644 /var/www/sitio1/index.html
```

### Qué significan 755 y 644

Cada cifra corresponde a un grupo de usuarios: **propietario**, **grupo** y **otros**. Cada cifra es la suma de: lectura (**r = 4**), escritura (**w = 2**) y ejecución (**x = 1**).

| Permiso | Propietario | Grupo | Otros | Se aplica a |
|---|---|---|---|---|
| **755** (`rwxr-xr-x`) | 7 = leer, escribir, entrar | 5 = leer, entrar | 5 = leer, entrar | **Directorios**: en una carpeta, la `x` significa poder entrar en ella |
| **644** (`rw-r--r--`) | 6 = leer, escribir | 4 = solo leer | 4 = solo leer | **Ficheros**: solo el propietario modifica, el resto solo lee |

Con esto, Apache puede leer y servir la web, pero solo el propietario puede modificarla.

### Por qué nunca se deben usar permisos 777

**777 (`rwxrwxrwx`) significa que cualquier usuario del sistema puede leer, modificar y ejecutar** ese fichero o carpeta. En un servidor web es muy peligroso:

- Si un atacante consigue entrar por un fallo de la web (por ejemplo, un formulario de subida mal hecho), podría **modificar las páginas** (*defacement*) o **subir un script malicioso** y ejecutarlo.
- Cualquier otro usuario o servicio del servidor podría cambiar o borrar la web.
- Es una mala práctica que suele usarse para "arreglar rápido" un problema de permisos, pero en realidad abre un agujero de seguridad. Lo correcto es averiguar qué usuario necesita el permiso y dárselo solo a él.

### Por qué el contenido web no debe pertenecer a root ni ser escribible por todos

- **No debe pertenecer a root** porque se aplica el **principio de mínimo privilegio**: cada usuario debe tener solo los permisos que necesita. Para editar la web no hace falta ser root, y trabajar siempre como root aumenta el riesgo de romper el sistema por un error.
- **No debe ser escribible por todos** porque, si un atacante consigue ejecutar código a través de Apache (como `www-data`) o desde otra cuenta, podría **modificar la web** y usarla para engañar a los visitantes o repartir malware.
- Lo ideal es que Apache pueda **leer** el contenido, y que solo el usuario o grupo que administra la web pueda **escribir**. En entornos más estrictos incluso se deja el contenido con un propietario distinto de `www-data` (por ejemplo, el usuario que despliega la web) y `www-data` solo con lectura, para que Apache no pueda modificar los ficheros que sirve.

---

## Conclusiones y problemas encontrados

### Conclusiones

Con esta práctica he montado un servidor web Apache completo en Ubuntu 24.04, desde la instalación hasta tener **dos webs distintas funcionando en el mismo servidor**. Lo más importante que he aprendido es:

- **Comprobar siempre**: `systemctl status`, `ss`, `curl` y `apache2ctl -S` permiten saber si el servidor funciona sin necesidad de abrir un navegador.
- **La organización de Ubuntu** en `*-available` / `*-enabled` con enlaces simbólicos hace muy fácil activar y desactivar sitios y módulos sin perder la configuración.
- **`apache2ctl configtest` antes de cada recarga** evita dejar el servidor caído por un error de escritura.
- **`reload` es mejor que `restart`** cuando solo cambia la configuración, porque no corta a los usuarios; `restart` solo hace falta con los módulos.
- **Los permisos son parte de la seguridad**: Apache trabaja como `www-data`, los ficheros deben tener 644, las carpetas 755 y nunca 777.
- **Copia de seguridad antes de editar**: un simple `cp fichero fichero.bak` puede ahorrar mucho trabajo.

### Problemas encontrados y cómo los resolví

| Problema | Causa | Solución |
|---|---|---|
| Aviso **`AH00558: Could not reliably determine the server's fully qualified domain name`** al arrancar y al hacer `configtest` | Apache no tenía definido un `ServerName` global y usaba `127.0.1.1` | Creé `servername.conf` con `ServerName localhost` y lo activé con `a2enconf` (apartado 6.3). Después `configtest` solo dice `Syntax OK` |
| La terminal se quedaba "parada" tras `systemctl status` y aparecía **`[1]+ Stopped`** | La salida se abre en un paginador y yo pulsaba `Ctrl+Z`, que **suspende** el proceso en vez de cerrarlo. Por eso en `ps aux` salían varios `systemctl status` en estado `T` | Salir del paginador con la tecla **`q`**. Los procesos suspendidos se pueden recuperar con `fg` o cerrar con `kill %1` |
| Mensajes **`already enabled`** al ejecutar `a2enconf servername` y `a2ensite` | Había ejecutado el comando antes y el enlace ya existía | No es un error: el comando simplemente no hace nada. Se puede comprobar con `ls -l` en el directorio `*-enabled` |
| Después de cambiar el puerto había que acordarse de **volver al 80** en dos ficheros | El puerto se define en `ports.conf` y en cada `<VirtualHost>` | Restauré los dos ficheros y lo comprobé con `ss -tulpn` (vuelve a salir `*:80`) |
| Riesgo de que `sudo echo "..." > fichero` dé **"Permiso denegado"** | La redirección `>` la ejecuta el usuario normal, no `sudo` | Usar `echo "..." \| sudo tee fichero`, como en los apartados 6 y 8 |

---

<div align="center">

**Daviti Gvaramia**, 2º SMR, Aplicaciones Web, 2026

[Volver al README](README.md)

</div>
