<div align="center">

# Práctica: Instalación, configuración y securización de Apache en Ubuntu 24.04

**Alumno:** Daviti Gvaramia
**Ciclo:** 2º CFGM Sistemas Microinformáticos y Redes (SMR)
**Módulo:** Aplicaciones Web
**Fecha:** octubre de 2026

![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04_LTS-E95420?logo=ubuntu&logoColor=white)
![Apache](https://img.shields.io/badge/Apache-2.4.58-D22128?logo=apache&logoColor=white)

</div>

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

> **Leyenda:** en cada paso aparece el comando en un bloque de código, la salida relevante, la captura y una explicación propia. Las preguntas de reflexión están marcadas con ❓.

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

### 1.3. Comprobar la versión del sistema 📸

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

### ❓ ¿Qué paquetes adicionales se han instalado como dependencias?

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

### 3.1. Estado del servicio 📸

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

- **`Loaded: ... enabled`** → el servicio está activado para arrancar solo cada vez que se enciende la máquina.
- **`Active: active (running)`** → Apache está funcionando ahora mismo.
- **`Main PID: 7632`** → es el proceso principal. Debajo aparecen otros procesos hijos (7633, 7635), que son los que atienden las peticiones de los clientes.
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

### ❓ ¿Qué diferencia hay entre los perfiles Apache, Apache Full y Apache Secure?

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

### ❓ ¿Cuándo conviene usar reload en lugar de restart?

**`reload`** se usa cuando solo he cambiado la configuración (un virtual host, un `ServerName`, una página de error…). Apache relee los ficheros y aplica los cambios mientras termina de atender las peticiones que ya tenía abiertas, así que **los usuarios no notan ningún corte**. En un servidor en producción es la opción preferida.

**`restart`** es necesario cuando el cambio afecta a cómo arranca el proceso: por ejemplo, al **activar o desactivar módulos** (`a2enmod` lo indica en su mensaje: *"you need to run: systemctl restart apache2"*), al cambiar el MPM o si Apache se ha quedado bloqueado. Como para el servicio por completo, corta todas las conexiones.

---
