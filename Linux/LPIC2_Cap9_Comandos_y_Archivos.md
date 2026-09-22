# LPIC-2 · Capítulo 9: Offering Web Services
### Recopilación de comandos y archivos de configuración

Objetivos del examen cubiertos: 208.1 (Basic Apache configuration), 208.2 (Apache configuration for HTTPS), 208.3 (Implementing Squid as a caching proxy), 208.4 (Implementing Nginx as a web server and a reverse proxy)

---

## 1. Conceptos básicos

| Estándar/Protocolo | Función |
|---|---|
| HTML | Define el formato/contenido de una página web |
| HTTP | Define cómo el cliente pide y el servidor responde (peticiones/respuestas) |
| SSL / TLS | Protocolos de cifrado del tráfico |

**Códigos de respuesta HTTP habituales:** `200` (éxito), `403` (sin permiso), `404` (fichero no encontrado), `500` (error del servidor).

---

## 2. Apache: instalación y ficheros de configuración

| Distribución | Paquete | Fichero principal | Directorio de configuración |
|---|---|---|---|
| **Debian** | `apache2` | `apache2.conf` | `/etc/apache2/` |
| **Red Hat** | `httpd` | `httpd.conf` | `/etc/httpd/conf/` |

(Instalaciones antiguas Apache 1.3.x en Debian usaban `/etc/apache/apache.conf`.)

Ambas familias permiten repartir la configuración en varios ficheros mediante la directiva `Include`.

Directorio por defecto de contenido web (`DocumentRoot`) en ambas familias: `/var/www/html`.

### 2.1 Directivas de configuración más comunes (Tabla 9.4)

| Directiva | Función |
|---|---|
| `Listen` | Puerto (y opcionalmente IP) donde escucha |
| `User` / `Group` | Cuenta con la que se ejecuta el demonio |
| `ServerAdmin` | Email del administrador |
| `ServerName` | Nombre de dominio del servidor |
| `ServerRoot` | Ubicación de los ficheros de configuración base |
| `DocumentRoot` | Carpeta de contenido servida por defecto |
| `DirectoryIndex` | Fichero servido por defecto al pedir un directorio |
| `ErrorDocument` | Fichero a servir cuando ocurre un tipo de error concreto |
| `ErrorLog` | Ubicación del log de errores |
| `LogFormat` | Formato de cada entrada del log |
| `AccessFileName` | Nombre del fichero de restricciones por carpeta (`.htaccess`) |
| `Include` | Incluye configuración de otro fichero |
| `StartServers` / `MaxClients` / `MinSpareServers` / `MaxSpareServers` | Control de procesos concurrentes |
| `LoadModule` | Carga un módulo de funcionalidad |

### 2.2 Control del servicio: `apache2ctl` / `apachectl`

| Comando | Función |
|---|---|
| `start` | Arranca el servidor |
| `stop` | Para el servidor, cerrando conexiones activas |
| `restart` | Envía SIGHUP y reinicia, cerrando conexiones activas |
| `graceful` | Reinicia sin cerrar las conexiones activas |
| `gracefulstop` | Para sin cerrar las conexiones activas |
| `fullstatus` | Informe completo de estado (requiere navegador de texto, ej. Lynx) |
| `status` | Informe breve de estado |
| `configtest` | Comprueba la sintaxis de la configuración sin arrancar el servidor |
| `help` | Lista de comandos |

Nota: algunas distribuciones Red Hat (ej. CentOS) usan `apachectl` incluso para Apache 2.x.

### 2.3 Ficheros de log

| Log | Debian | Red Hat | Contenido |
|---|---|---|---|
| Acceso | `access.log` | `access_log` | Todas las peticiones de los clientes |
| Errores | `error.log` | `error_log` | Errores y avisos del servidor |

Ubicación habitual: dentro de `/var/log/` (carpeta exacta configurable con `ErrorLog`).

---

## 3. Alojamiento web para usuarios

| Directiva | Función |
|---|---|
| `UserDir nombre_carpeta` | Permite que cada usuario publique contenido desde `~/nombre_carpeta` (por defecto `public_html`) |

Acceso vía URL: `http://servidor/~usuario/fichero`. Requiere permisos de lectura/ejecución para el usuario/grupo de Apache tanto en `public_html` como en el `HOME` del usuario.

---

## 4. Hosting virtual (virtual hosting)

### 4.1 Basado en nombre (name-based)

```
NameVirtualHost 192.168.1.77
<VirtualHost 192.168.1.77>
    ServerName www.myhost1.com
    DocumentRoot /var/www/html/host1
</VirtualHost>
<VirtualHost 192.168.1.77>
    ServerName www.myhost2.com
    DocumentRoot /var/www/html/host2
</VirtualHost>
```
Todas las webs comparten la misma IP; Apache distingue por el nombre de host que pide el cliente.

### 4.2 Basado en IP (IP-based)

```
Listen 192.168.1.77:80
Listen 192.168.1.78:80
<VirtualHost www.myhost1.com>
    ServerName www.myhost1.com
    DocumentRoot /var/www/html/myhost1
</VirtualHost>
<VirtualHost www.myhost2.com>
    ServerName www.myhost2.com
    DocumentRoot /var/www/html/myhost2
</VirtualHost>
```
Requiere varias IPs asignadas al servidor (varias tarjetas o IPs alias en la misma). Ventaja: se pueden ejecutar dos procesos Apache distintos, cada uno con su propia configuración.

---

## 5. Restricción de acceso

### 5.1 Autenticación por usuario/contraseña

Módulos relevantes: `mod_authn_file` (sustituto moderno de `mod_auth`/`mod_auth_basic`), `mod_authn_anon`, `mod_authn_db`, `mod_authn_dbm`, `mod_authnz_ldap`, `mod_authnz_mysql`.

**Crear el fichero de usuarios:**
```
htpasswd -c /var/www/html/passwords rich
```
(`-c` solo la primera vez, para crear el fichero; en usos posteriores se omite para añadir más usuarios sin sobrescribir).

**Proteger un directorio (en el fichero principal o en `.htaccess`):**
```
<Directory /var/www/html>
    AuthName "Restricted Area"
    AuthType Basic
    AuthUserFile /var/www/html/passwords
    Require valid-user
    DocumentRoot /var/www/html
</Directory>
```

| Directiva | Función |
|---|---|
| `AuthName` | Título del cuadro de login |
| `AuthType` | Tipo de autenticación (`Basic`) |
| `AuthUserFile` | Ruta al fichero de usuarios/contraseñas |
| `AuthGroupFile` | Agrupa usuarios en grupos |
| `Require valid-user` | Exige un usuario válido del fichero |
| `AllowOverride` | Necesaria en el `<Directory>` para permitir que `.htaccess` tenga efecto |

### 5.2 Restricción por IP (`mod_access` / módulos equivalentes)

```
<Directory /var/www/html>
    Order Deny,Allow
    Deny from All
    Allow from 192.168.1.0/255.255.255.0
    DocumentRoot /var/www/html
</Directory>
```

Módulos: `mod_access` (lista de IPs/hosts/dominios), `mod_access_compat` y `mod_authz_host` (basados en hostname/IP del cliente).

---

## 6. Contenido dinámico

| Método | Descripción |
|---|---|
| CGI (Common Gateway Interface) | Apache pasa el fichero a un intérprete externo (Perl, Python...) que procesa el código y devuelve el resultado |
| `mod_perl`, `mod_php` | Módulos que procesan el código embebido directamente dentro del propio proceso Apache (sin proceso externo) |
| `AddHandler` | Directiva que indica cómo tratar los ficheros según su extensión (ej. redirigirlos al intérprete PHP) |

---

## 7. HTTPS (SSL/TLS) en Apache

### 7.1 Instalación

| Distribución | Notas |
|---|---|
| Debian | `mod_ssl` viene incluido en la instalación básica de `apache2` |
| Red Hat | Hay que instalar el paquete `mod_ssl` aparte (`yum install mod_ssl`) |

Además se necesita el paquete `openssl`. Ubicación habitual de certificados: `/etc/ssl/` (o `/etc/pki/` en algunas distros).

### 7.2 Generar clave y certificado

| Paso | Comando |
|---|---|
| 1. Generar clave privada | `openssl genrsa -des3 -out server.key 2048` |
| 2. Generar CSR (Certificate Signing Request) | (vía `openssl req` o el asistente) |
| 3. Actuar como tu propia CA (para pruebas) | `/usr/lib/ssl/misc/CA.pl -newca` |
| 4. Firmar el certificado (autofirmado, para pruebas) | Script `CA.pl`, genera `newcert.pem` |

Un certificado autofirmado no es de confianza para los navegadores por defecto (mostrarán aviso); para producción hace falta una CA comercial reconocida.

### 7.3 Instalar y activar en Apache

```
mkdir /etc/apache2/certs
cp server.key /etc/apache2/certs
cp newcert.pem /etc/apache2/certs
```

```
Listen 443

<VirtualHost ...>
    SSLEngine On
    SSLCertificateFile /etc/apache2/certs/newcert.pem
    SSLCertificateKeyFile /etc/apache2/certs/server.key
</VirtualHost>
```

**Otras directivas SSL relevantes:**

| Directiva | Función |
|---|---|
| `SSLCACertificateFile` / `SSLCACertificatePath` | Certificado(s) de la CA para validar certificados de cliente |
| `SSLCertificateChainFile` | Cadena de certificados CA concatenados |
| `SSLProtocol` | Versiones de SSL/TLS soportadas |
| `SSLCipherSuite` | Protocolos de cifrado soportados |
| `ServerTokens` | Controla si se informa del SO en las respuestas |
| `ServerSignature` | Controla el pie de página con info del servidor |
| `TraceEnable` | Permite o no el comando HTTP TRACE |

Nota: con hosting virtual basado en nombre, un mismo certificado no vale para varios hosts virtuales; la extensión **SNI** (Server Name Indication) permite que el cliente indique el hostname al inicio del handshake SSL para que Apache elija el certificado correcto.

---

## 8. Squid: servidor proxy caché

### 8.1 Instalación y arranque

| Distribución | Directorio de configuración | Arranque |
|---|---|---|
| Debian | `/etc/squid3/` | Se inicia automáticamente al instalar |
| Red Hat | `/etc/squid/` | `systemctl start squid` + `systemctl enable squid` |

Fichero principal: `squid.conf` (soporta `include` como Apache). Puerto por defecto: TCP **3128**.

### 8.2 Directivas principales

| Directiva | Función |
|---|---|
| `http_port` | Puerto de escucha |
| `cache_dir` | Carpeta/partición de caché: tipo de FS, ruta, espacio en MB, nº de carpetas de primer y segundo nivel |
| `acl` | Define una lista de control de acceso |
| `http_access` | Regla que permite o deniega el acceso a una ACL |
| `auth_param` | Método de autenticación de clientes |
| `redirect_program` | Programa externo al que redirigir las peticiones |

### 8.3 Caché web (ejemplo mínimo)

```
http_port 3128
cache_dir ufs /var/spool/squid3 100 16 256
```

### 8.4 Tipos de ACL

`src` (IP origen), `dst` (IP destino), `port`, `srcdomain`, `dstdomain`, `time`, `proto`, `browser`.

Ejemplo de control de acceso:
```
acl ourhosts src 192.168.2.0/255.255.255.0
http_access allow ourhosts
http_access deny all
```

Ejemplo con dominio y horario:
```
acl socialmedia dstdomain www.facebook.com www.twitter.com
acl lunch MTWHF 12:00-13:00
http_access allow socialmedia lunch
http_access deny socialmedia
```

Importante: la acción por defecto de Squid es la **contraria** a la última regla `http_access` definida (si la última es `allow`, por defecto se deniega; si es `deny`, por defecto se permite).

### 8.5 Autenticación de clientes

```
auth_param basic /usr/lib/squid/pam_auth
auth_param basic children 5 startup=5 idle=1
auth_param basic realm Squid proxy-caching web server
auth_param basic credentialsttl 2 hours

acl ourhosts proxy_auth REQUIRED
```

### 8.6 Configuración del cliente

Cada navegador debe configurarse para usar el proxy (IP y puerto del servidor Squid). Para forzar su uso, hay que bloquear el acceso directo a Internet a nivel de firewall/router.

---

## 9. Nginx: servidor web y proxy inverso

### 9.1 Características

- No usa un hilo por cliente como Apache; usa una arquitectura **asíncrona** dentro del mismo proceso, lo que reduce el consumo de memoria por cliente y permite atender a más clientes con el mismo servidor.
- Puede funcionar como **reverse proxy** (proxy inverso): recibe peticiones de clientes y las reparte entre varios servidores backend (**load balancing**), al contrario que un proxy normal (que atiende a varios clientes hacia un único servidor externo).

### 9.2 Instalación

```
sudo apt-get install nginx
```
(en Debian/Ubuntu está en el repositorio estándar; en distribuciones Red Hat puede no estarlo por defecto). Arranca automáticamente en el puerto 80 — si Apache ya lo usa, hay que parar uno de los dos o cambiar puertos.

### 9.3 Estructura básica de configuración

```
server {
    listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;
    root /usr/share/nginx/html;
    index index.html index.htm;
    server_name localhost;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

| Directiva | Función |
|---|---|
| `server { ... }` | Bloque de configuración de un servidor (equivalente a `VirtualHost`) |
| `listen` | Puerto(s)/dirección(es) de escucha |
| `root` | Carpeta de contenido (equivalente a `DocumentRoot`) |
| `index` | Ficheros por defecto para peticiones de directorio |
| `server_name` | Nombre de host del servidor |
| `location` | Configuración específica para una ruta concreta; aquí se definen, entre otras cosas, las direcciones proxy hacia los servidores backend para implementar el reverse proxy |

---

## 10. Resumen de diferencias Debian vs Red Hat en este capítulo

| Aspecto | Debian | Red Hat |
|---|---|---|
| Paquete de Apache | `apache2` | `httpd` |
| Fichero principal de Apache | `/etc/apache2/apache2.conf` | `/etc/httpd/conf/httpd.conf` |
| Logs de Apache | `access.log` / `error.log` | `access_log` / `error_log` |
| Instalación de `mod_ssl` | Incluido con `apache2` | Paquete `mod_ssl` aparte |
| Directorio de Squid | `/etc/squid3/` | `/etc/squid/` |
| Arranque de Squid | Automático al instalar | `systemctl start squid` + `enable` |
| Resto del capítulo (directivas Apache, HTTPS, Squid ACLs, Nginx) | Igual | Igual |

---

*Documento generado a partir del Capítulo 9 ("Offering Web Services") del libro LPIC-2: Linux Professional Institute Certification Study Guide (Bresnahan & Blum).*
