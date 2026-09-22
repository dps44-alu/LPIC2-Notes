# LPIC-2 · Capítulo 9: Offering Web Services
### Equivalencias en FreeBSD 15

Este documento recoge, para cada concepto visto en el Capítulo 9 del libro LPIC-2 (Apache, HTTPS, Squid y Nginx), cuál es su equivalente en **FreeBSD 15**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo: Apache, Squid y Nginx son **los mismos programas** en FreeBSD, con las mismas directivas. Lo que cambia es:
- Se instalan como **paquetes**, así que su configuración está en **`/usr/local/etc/`** y el contenido web en **`/usr/local/www/`**.
- Se activan en **`/etc/rc.conf`** (`nombre_enable="YES"`) y se controlan con `service`.
- El usuario con el que se ejecutan los servidores web es **`www`** (en Debian es `www-data` y en Red Hat `apache`).

---

## 1. Conceptos básicos

Los conceptos (HTML, HTTP, SSL/TLS) y los códigos de respuesta (`200`, `403`, `404`, `500`) son **los mismos**: son estándares de Internet, no dependen del sistema operativo.

---

## 2. Apache: instalación y ficheros de configuración

| Elemento | Debian | Red Hat | FreeBSD 15 |
|---|---|---|---|
| Paquete | `apache2` | `httpd` | **`apache24`** |
| Fichero principal | `/etc/apache2/apache2.conf` | `/etc/httpd/conf/httpd.conf` | **`/usr/local/etc/apache24/httpd.conf`** |
| Configuración adicional | `/etc/apache2/conf-enabled/`, `sites-enabled/` | `/etc/httpd/conf.d/` | **`/usr/local/etc/apache24/Includes/`** (todo `.conf` que se deje aquí se carga automáticamente) |
| Configuraciones de ejemplo | — | — | `/usr/local/etc/apache24/extra/` (vhosts, SSL, userdir...; se activan con `Include`) |
| Módulos | `/usr/lib/apache2/modules/` | `/usr/lib64/httpd/modules/` | `/usr/local/libexec/apache24/` |
| `DocumentRoot` por defecto | `/var/www/html` | `/var/www/html` | **`/usr/local/www/apache24/data`** |
| Usuario / grupo | `www-data` | `apache` | **`www`** |

**Instalar y arrancar:**
```
pkg install apache24
sysrc apache24_enable="YES"
service apache24 start
```

**Filtro de aceptación HTTP (propio de FreeBSD):** Apache avisa al arrancar si no encuentra el filtro `accf_http`. Es una función del kernel de FreeBSD que retiene las conexiones hasta que llega la petición HTTP completa, ahorrando trabajo al servidor. Se activa así:
```
sysrc apache24_http_accept_enable="YES"
```

### 2.1 Directivas de configuración (Tabla 9.4)

**Idénticas** a las del libro (`Listen`, `User`, `Group`, `ServerAdmin`, `ServerName`, `ServerRoot`, `DocumentRoot`, `DirectoryIndex`, `ErrorDocument`, `ErrorLog`, `LogFormat`, `AccessFileName`, `Include`, `LoadModule`). Valores por defecto en FreeBSD:

| Directiva | Valor en FreeBSD 15 |
|---|---|
| `ServerRoot` | `"/usr/local"` |
| `User` / `Group` | `www` / `www` |
| `DocumentRoot` | `"/usr/local/www/apache24/data"` |
| `LoadModule` | Rutas del tipo `libexec/apache24/mod_xxx.so`. Muchos módulos vienen **comentados** en `httpd.conf`: para activarlos basta con quitar el `#` |

Nota de versión (afecta también a Linux): en Apache 2.4 la directiva `MaxClients` del libro se llama **`MaxRequestWorkers`**.

### 2.2 Control del servicio

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `apachectl start` / `systemctl start` | `service apache24 start` | Arranca |
| `apachectl stop` | `service apache24 stop` | Para |
| `apachectl restart` | `service apache24 restart` | Reinicia |
| `apachectl graceful` | `service apache24 graceful` | Reinicia sin cortar conexiones |
| `apachectl gracefulstop` | `service apache24 gracefulstop` | Para sin cortar conexiones |
| `apachectl configtest` | `service apache24 configtest` (o `apachectl configtest`) | Comprueba la sintaxis |
| `apachectl status` / `fullstatus` | `apachectl status` / `fullstatus` | Informe de estado (necesita `mod_status` y un navegador de texto, como el paquete `lynx`) |

`apachectl` está disponible en `/usr/local/sbin/apachectl`, igual que en Linux.

### 2.3 Ficheros de log

| Log | Debian | Red Hat | FreeBSD 15 |
|---|---|---|---|
| Acceso | `/var/log/apache2/access.log` | `/var/log/httpd/access_log` | **`/var/log/httpd-access.log`** |
| Errores | `/var/log/apache2/error.log` | `/var/log/httpd/error_log` | **`/var/log/httpd-error.log`** |

Para que estos logs roten automáticamente, se añaden a `/etc/newsyslog.conf` (o a un archivo en `/usr/local/etc/newsyslog.conf.d/`). `newsyslog` es el equivalente de FreeBSD a `logrotate`.

---

## 3. Alojamiento web para usuarios

La directiva `UserDir` funciona **igual**. En FreeBSD viene preparada en un archivo de ejemplo que hay que activar:

1. En `httpd.conf`, quitar el `#` de:
   ```
   LoadModule userdir_module libexec/apache24/mod_userdir.so
   Include etc/apache24/extra/httpd-userdir.conf
   ```
2. El archivo `extra/httpd-userdir.conf` ya contiene `UserDir public_html`.
3. `service apache24 graceful`.

Acceso: `http://servidor/~usuario/`. Los permisos necesarios son los mismos que en Linux (el usuario `www` debe poder entrar en el `HOME` y leer `public_html`).

---

## 4. Hosting virtual

La sintaxis de `<VirtualHost>` es **la misma**. En FreeBSD, lo cómodo es crear un archivo por sitio en `/usr/local/etc/apache24/Includes/`, por ejemplo `Includes/myhost1.conf`:

```
<VirtualHost *:80>
    ServerName www.myhost1.com
    DocumentRoot /usr/local/www/myhost1
    ErrorLog /var/log/myhost1-error.log
    CustomLog /var/log/myhost1-access.log combined
</VirtualHost>
```

Nota de versión (afecta también a Linux): en Apache 2.4 la línea **`NameVirtualHost` del libro ya no es necesaria** (está obsoleta); Apache distingue por nombre automáticamente.

**Hosting por IP:** igual que en el libro (`Listen IP:80` y un `<VirtualHost IP:80>` por dirección). Las IPs adicionales se añaden a la tarjeta como alias en `/etc/rc.conf` (Capítulo 6):
```
ifconfig_em0_alias0="inet 192.168.1.78/32"
```

---

## 5. Restricción de acceso

### 5.1 Autenticación por usuario/contraseña

**Idéntica** al libro. Los módulos (`mod_authn_file`, `mod_authnz_ldap`, etc.) vienen en el paquete `apache24`, y `htpasswd` está en `/usr/local/bin/`:

```
htpasswd -c /usr/local/etc/apache24/passwords rich
```

Buena práctica: guardar el fichero de contraseñas **fuera** del `DocumentRoot` (el ejemplo del libro lo deja dentro de `/var/www/html`, donde podría descargarse).

```
<Directory "/usr/local/www/apache24/data/privado">
    AuthName "Restricted Area"
    AuthType Basic
    AuthUserFile /usr/local/etc/apache24/passwords
    Require valid-user
</Directory>
```

Las directivas `AuthName`, `AuthType`, `AuthUserFile`, `AuthGroupFile`, `Require valid-user` y `AllowOverride` funcionan igual.

### 5.2 Restricción por IP

Nota de versión importante (afecta también a Linux): la sintaxis del libro (`Order`, `Deny from`, `Allow from`) es de **Apache 2.2**. En Apache 2.4 solo funciona si se carga `mod_access_compat`. La forma actual usa **`Require`** (módulo `mod_authz_host`):

| Apache 2.2 (libro) | Apache 2.4 (actual) |
|---|---|
| `Order Deny,Allow` + `Deny from All` + `Allow from 192.168.1.0/255.255.255.0` | `Require ip 192.168.1.0/24` |
| `Allow from all` | `Require all granted` |
| `Deny from all` | `Require all denied` |
| — | `Require host example.com` (por nombre de dominio) |

Ejemplo equivalente al del libro:
```
<Directory "/usr/local/www/apache24/data">
    Require ip 192.168.1.0/24
</Directory>
```

---

## 6. Contenido dinámico

| Método | En FreeBSD 15 |
|---|---|
| CGI | Igual. Carpeta por defecto para scripts: `/usr/local/www/apache24/cgi-bin/`. Hay que activar `mod_cgi` o `mod_cgid` en `httpd.conf` |
| `mod_php` | Paquete `mod_phpXX`, donde `XX` es la versión de PHP (buscar con `pkg search mod_php`). Tras instalarlo, el paquete muestra las líneas que hay que añadir a Apache |
| PHP-FPM (forma recomendada hoy) | Paquete `phpXX` + `sysrc php_fpm_enable="YES"` + `service php_fpm start`; Apache le pasa las peticiones con `mod_proxy_fcgi` |
| `mod_perl` | Paquete `ap24-mod_perl2` |
| `AddHandler` | Igual |

Ejemplo para enviar los `.php` a PHP-FPM (en un archivo de `Includes/`):
```
<FilesMatch "\.php$">
    SetHandler "proxy:fcgi://127.0.0.1:9000"
</FilesMatch>
```

---

## 7. HTTPS (SSL/TLS) en Apache

### 7.1 Instalación

| Distribución | Notas |
|---|---|
| Debian | `mod_ssl` incluido con `apache2` |
| Red Hat | Paquete `mod_ssl` aparte |
| **FreeBSD 15** | **`mod_ssl` incluido en el paquete `apache24`**; solo hay que activarlo |

OpenSSL viene en el **sistema base** de FreeBSD (comando `openssl`), no hace falta instalarlo.

| Elemento | Ubicación en FreeBSD 15 |
|---|---|
| Certificados de CA de confianza del sistema | `/etc/ssl/certs/` (se gestionan con `certctl`, ej. `certctl rehash`) |
| Certificados propios de Apache (recomendado) | Una carpeta como `/usr/local/etc/apache24/ssl/` |
| Configuración SSL de ejemplo | `/usr/local/etc/apache24/extra/httpd-ssl.conf` |

### 7.2 Generar clave y certificado

Los comandos `openssl` son **los mismos**:

| Paso | Comando |
|---|---|
| 1. Generar la clave privada | `openssl genrsa -out server.key 2048` (con `-des3` la clave queda protegida por contraseña, como en el libro, pero entonces Apache la pide en cada arranque) |
| 2. Generar la petición de firma (CSR) | `openssl req -new -key server.key -out server.csr` |
| 3. Certificado autofirmado para pruebas (en un paso, sustituye a `CA.pl`) | `openssl req -x509 -newkey rsa:2048 -nodes -keyout server.key -out server.crt -days 365` |

El script `CA.pl` del libro no se usa en FreeBSD (no viene con el OpenSSL del sistema base); el comando del paso 3 hace lo mismo para pruebas.

Para certificados reales y gratuitos, lo habitual hoy es **Let's Encrypt**, con los paquetes `certbot` (buscar con `pkg search certbot`) o `acme.sh`.

### 7.3 Instalar y activar en Apache

```
mkdir -p /usr/local/etc/apache24/ssl
cp server.key server.crt /usr/local/etc/apache24/ssl/
chmod 600 /usr/local/etc/apache24/ssl/server.key
```

En `httpd.conf`, quitar el `#` de:
```
LoadModule ssl_module libexec/apache24/mod_ssl.so
LoadModule socache_shmcb_module libexec/apache24/mod_socache_shmcb.so
Include etc/apache24/extra/httpd-ssl.conf
```

Y en `extra/httpd-ssl.conf` (que ya incluye `Listen 443` y un `<VirtualHost _default_:443>`), ajustar las rutas:
```
SSLEngine on
SSLCertificateFile "/usr/local/etc/apache24/ssl/server.crt"
SSLCertificateKeyFile "/usr/local/etc/apache24/ssl/server.key"
```

Comprobar y aplicar: `service apache24 configtest` y `service apache24 graceful`.

Las demás directivas del libro (`SSLCACertificateFile`, `SSLCertificateChainFile`, `SSLProtocol`, `SSLCipherSuite`, `ServerTokens`, `ServerSignature`, `TraceEnable`) y el uso de **SNI** son **idénticos**.

---

## 8. Squid: servidor proxy caché

### 8.1 Instalación y arranque

| Elemento | Debian | Red Hat | FreeBSD 15 |
|---|---|---|---|
| Paquete | `squid` (antes `squid3`) | `squid` | **`squid`** |
| Configuración | `/etc/squid/` | `/etc/squid/` | **`/usr/local/etc/squid/squid.conf`** |
| Caché por defecto | `/var/spool/squid` | `/var/spool/squid` | **`/var/squid/cache`** |
| Logs | `/var/log/squid/` | `/var/log/squid/` | `/var/log/squid/` (ej. `access.log`, `cache.log`) |
| Usuario | `proxy` | `squid` | `squid` |
| Arranque | Automático | `systemctl start/enable squid` | `sysrc squid_enable="YES"` + `service squid start` |
| Puerto | 3128 | 3128 | 3128 (igual) |

Si se añade o cambia un `cache_dir`, hay que crear su estructura de carpetas antes de arrancar: `squid -z` (el script de arranque de FreeBSD lo hace solo la primera vez).

Recargar la configuración sin parar el servicio: `squid -k reconfigure` (o `service squid reload`). Comprobar la sintaxis: `squid -k parse`.

### 8.2 a 8.4 Directivas y ACL

**Idénticas** a las del libro (`http_port`, `cache_dir`, `acl`, `http_access`, `auth_param`; tipos de ACL `src`, `dst`, `port`, `srcdomain`, `dstdomain`, `time`, `proto`, `browser`). Solo cambia la ruta de la caché:
```
http_port 3128
cache_dir ufs /var/squid/cache 100 16 256
```

Nota: en el ejemplo del horario del libro falta la palabra `time`. La forma correcta es:
```
acl lunch time MTWHF 12:00-13:00
```

La regla del libro sobre la acción por defecto (la contraria a la última `http_access`) se aplica igual.

### 8.5 Autenticación de clientes

Igual, cambiando la ruta del programa auxiliar, que en las versiones actuales se llama **`basic_pam_auth`** (el `pam_auth` del libro es el nombre antiguo):
```
auth_param basic program /usr/local/libexec/squid/basic_pam_auth
auth_param basic children 5 startup=5 idle=1
auth_param basic realm Squid proxy-caching web server
auth_param basic credentialsttl 2 hours

acl ourhosts proxy_auth REQUIRED
```

### 8.6 Configuración del cliente

Igual que en el libro. Para obligar a usar el proxy, se bloquea la salida directa a Internet en el cortafuegos; en FreeBSD con **`pf`** también se puede redirigir el tráfico web al proxy de forma transparente (Capítulo 12).

---

## 9. Nginx: servidor web y proxy inverso

### 9.1 Características

Las mismas que describe el libro (arquitectura asíncrona, proxy inverso y reparto de carga).

### 9.2 Instalación

| Elemento | Debian | FreeBSD 15 |
|---|---|---|
| Instalar | `apt-get install nginx` | `pkg install nginx` |
| Arranque | Automático | `sysrc nginx_enable="YES"` + `service nginx start` |
| Fichero principal | `/etc/nginx/nginx.conf` | **`/usr/local/etc/nginx/nginx.conf`** |
| Sitios adicionales | `/etc/nginx/sites-enabled/` | No existe por defecto. Se suele crear `/usr/local/etc/nginx/conf.d/` y añadir `include conf.d/*.conf;` dentro del bloque `http` |
| Contenido por defecto | `/usr/share/nginx/html` o `/var/www/html` | **`/usr/local/www/nginx/`** |
| Logs | `/var/log/nginx/` | `/var/log/nginx/access.log` y `error.log` |
| Usuario | `www-data` | `www` |

Igual que en Linux, si Apache ya usa el puerto 80, hay que parar uno de los dos o cambiar el puerto.

**Comandos útiles:**

| Comando | Función |
|---|---|
| `nginx -t` o `service nginx configtest` | Comprueba la sintaxis |
| `service nginx reload` (o `nginx -s reload`) | Recarga la configuración sin cortar conexiones |
| `service nginx restart` | Reinicia |

### 9.3 Estructura básica de configuración

**Idéntica** al libro (`server`, `listen`, `root`, `index`, `server_name`, `location`), cambiando la ruta de `root`:
```
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    root /usr/local/www/nginx;
    index index.html index.htm;
    server_name localhost;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

**Ejemplo de proxy inverso con reparto de carga** (lo que el libro menciona dentro de `location`):
```
upstream backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}

server {
    listen 80;
    server_name www.example.com;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Otros programas de proxy inverso y reparto de carga muy usados en FreeBSD: `haproxy` y `varnish` (paquetes).

---

## 10. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 9) | Equivalente en FreeBSD 15 |
|---|---|
| Paquete `apache2` / `httpd` | Paquete `apache24` |
| `/etc/apache2/apache2.conf` / `/etc/httpd/conf/httpd.conf` | `/usr/local/etc/apache24/httpd.conf` |
| `conf.d/`, `sites-enabled/` | `/usr/local/etc/apache24/Includes/` |
| `/var/www/html` | `/usr/local/www/apache24/data` |
| Usuario `www-data` / `apache` | Usuario `www` |
| `systemctl start apache2` / `httpd` | `service apache24 start` |
| `apachectl configtest` / `graceful` | `service apache24 configtest` / `graceful` |
| `access.log` / `access_log` | `/var/log/httpd-access.log` |
| `error.log` / `error_log` | `/var/log/httpd-error.log` |
| `logrotate` | `newsyslog` |
| `a2enmod` / `LoadModule` | Quitar el `#` de la línea `LoadModule` en `httpd.conf` |
| `NameVirtualHost` | Obsoleto en Apache 2.4 |
| `Order` / `Allow from` / `Deny from` | `Require ip` / `Require all granted` / `Require all denied` |
| `mod_php` | Paquete `mod_phpXX` o PHP-FPM (`phpXX`) |
| Paquete `mod_ssl` (Red Hat) | Incluido en `apache24` |
| `openssl` (paquete) | `openssl` del sistema base |
| `CA.pl -newca` | `openssl req -x509 ...` |
| `/etc/squid/`, `/var/spool/squid` | `/usr/local/etc/squid/`, `/var/squid/cache` |
| `pam_auth` (Squid) | `/usr/local/libexec/squid/basic_pam_auth` |
| `/etc/nginx/nginx.conf` | `/usr/local/etc/nginx/nginx.conf` |
| `/usr/share/nginx/html` | `/usr/local/www/nginx` |
| — | Filtro `accf_http` (`apache24_http_accept_enable="YES"`) |

---

*Documento elaborado como contrapartida en FreeBSD 15 al Capítulo 9 ("Offering Web Services") del libro LPIC-2. Fuentes: FreeBSD Handbook (sección "Apache HTTP Server" del capítulo "Network Servers") y páginas de manual de FreeBSD: rc.conf(5), accf_http(9), newsyslog.conf(5), certctl(8), openssl(1), además de la documentación de Apache 2.4, Squid y Nginx incluida en sus paquetes.*
