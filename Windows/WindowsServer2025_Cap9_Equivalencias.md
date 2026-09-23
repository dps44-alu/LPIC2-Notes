# LPIC-2 · Capítulo 9: Offering Web Services
### Equivalencias en Windows Server 2025

Este documento recoge, para cada concepto visto en el Capítulo 9 del libro LPIC-2 (Apache, HTTPS, Squid y Nginx), cuál es su equivalente en **Windows Server 2025**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo: el servidor web de Windows Server es **IIS** (Internet Information Services, versión 10.0), que es un **rol** del sistema. Cumple el papel de Apache y también, con dos complementos gratuitos de Microsoft, el de Nginx como proxy inverso. Diferencias principales con Apache:
- La configuración no está en `httpd.conf`, sino en archivos **XML**: `applicationHost.config` (todo el servidor) y **`web.config`** (por sitio o carpeta, parecido a `.htaccess`).
- Se administra con el **Administrador de IIS** (`inetmgr`), con la herramienta de texto **`appcmd`** o con PowerShell (módulos **IISAdministration** y **WebAdministration**).
- Quien escucha en los puertos no es el propio servidor web, sino un controlador del kernel, **`http.sys`**, que reparte las peticiones a los **grupos de aplicaciones** (procesos `w3wp.exe`).
- Los usuarios que se autentican son **cuentas de Windows o de Active Directory**, no un archivo de contraseñas.
- No hay equivalente integrado de **Squid** (apartado 8).

Todos los comandos se ejecutan en una consola **como Administrador**.

---

## 1. Conceptos básicos

HTML, HTTP y TLS son estándares, así que son **los mismos**. Los códigos de respuesta también (`200`, `403`, `404`, `500`), pero IIS añade **subcódigos** que ayudan mucho a encontrar el problema:

| Código | Significado en IIS |
|---|---|
| `401.1` / `401.2` | Inicio de sesión incorrecto / método de autenticación no permitido |
| `403.14` | Se ha pedido una carpeta sin documento predeterminado y sin permiso para listar su contenido |
| `404.3` | Tipo de archivo sin tipo MIME configurado (IIS no lo sirve) |
| `500.19` | Error en un archivo de configuración (`web.config` o `applicationHost.config` mal escrito) |
| `503` | El grupo de aplicaciones está parado |

Los subcódigos solo se ven en las páginas de error detalladas (por defecto, al navegar desde el propio servidor) y en los registros.

---

## 2. IIS: instalación y ficheros de configuración

| Elemento | Debian | Red Hat | Windows Server 2025 |
|---|---|---|---|
| Instalación | `apt-get install apache2` | `yum install httpd` | `Install-WindowsFeature Web-Server -IncludeManagementTools` |
| Fichero principal | `/etc/apache2/apache2.conf` | `/etc/httpd/conf/httpd.conf` | `C:\Windows\System32\inetsrv\config\applicationHost.config` |
| Configuración por carpeta | `.htaccess` | `.htaccess` | `web.config` |
| Contenido web por defecto | `/var/www/html` | `/var/www/html` | `C:\inetpub\wwwroot` |
| Servicio | `apache2` | `httpd` | **Servicio de publicación World Wide Web** (`W3SVC`) y **Servicio WAS** (`WAS`) |
| Proceso que atiende las peticiones | `apache2` | `httpd` | `w3wp.exe` (uno por grupo de aplicaciones) |
| Usuario del proceso | `www-data` | `apache` | Identidad del grupo de aplicaciones (por defecto `IIS AppPool\DefaultAppPool`) |
| Usuario anónimo | — | — | `IUSR` |

Funciones adicionales (autenticación, CGI, restricción por IP...) se instalan como **servicios de rol**, el equivalente a instalar módulos. Para ver cuáles hay: `Get-WindowsFeature Web-*`.

**Herramientas de administración:**

| Herramienta | Descripción |
|---|---|
| `inetmgr` | Administrador de IIS (gráfico; en Server Core se usa en remoto con el servicio de rol `Web-Mgmt-Service`) |
| `C:\Windows\System32\inetsrv\appcmd.exe` | Herramienta clásica de texto |
| Módulo **IISAdministration** | Cmdlets modernos: `Get-IISSite`, `New-IISSite`, `Start-IISSite`, `Get-IISAppPool`, `Get-IISConfigSection`... |
| Módulo **WebAdministration** | Cmdlets anteriores, todavía muy usados: `New-Website`, `New-WebAppPool`, `Set-WebConfigurationProperty` y la unidad `IIS:\` |

### 2.1 Directivas de configuración (equivalente a la Tabla 9.4)

| Directiva de Apache | Equivalente en IIS |
|---|---|
| `Listen` | **Enlaces** (*bindings*) de cada sitio, con el formato `IP:puerto:nombre`: `New-IISSiteBinding -Name "Default Web Site" -BindingInformation "*:8080:" -Protocol http` |
| `User` / `Group` | **Identidad del grupo de aplicaciones**: `ApplicationPoolIdentity` (recomendada), `NetworkService`, `LocalSystem` o una cuenta concreta |
| `ServerAdmin` | No existe |
| `ServerName` | **Nombre de host** en el enlace del sitio (`"*:80:www.empresa.com"`) |
| `ServerRoot` | `C:\Windows\System32\inetsrv\` |
| `DocumentRoot` | **Ruta física** del sitio: `Set-ItemProperty "IIS:\Sites\Default Web Site" -Name physicalPath -Value D:\Web` |
| `DirectoryIndex` | **Documento predeterminado**: `Add-WebConfigurationProperty -PSPath "IIS:\Sites\Default Web Site" -Filter system.webServer/defaultDocument/files -Name "." -Value @{value="inicio.html"}` |
| `ErrorDocument` | **Páginas de error**, sección `httpErrors` (ver ejemplo abajo) |
| `ErrorLog` / `LogFormat` | **Registro** de cada sitio: carpeta, formato (W3C, IIS o NCSA) y campos (apartado 2.3) |
| `AccessFileName` | Siempre `web.config` (no se puede cambiar) |
| `Include` | Atributo `configSource` (llevar una sección a otro archivo) y etiquetas `<location>` |
| `StartServers`, `MaxClients`... | Ajustes del **grupo de aplicaciones** (número de procesos, cola de peticiones, reciclado, tiempo de inactividad) y límites del sitio (`limits.maxConnections`) |
| `LoadModule` | **Servicios de rol** (`Install-WindowsFeature Web-...`) y módulos: `Get-WebGlobalModule` (ver), `Enable-WebGlobalModule` (activar) |

Ejemplo de `ErrorDocument 404 /404.html` en `web.config`:
```xml
<configuration>
  <system.webServer>
    <httpErrors errorMode="Custom">
      <remove statusCode="404" />
      <error statusCode="404" path="/404.html" responseMode="ExecuteURL" />
    </httpErrors>
  </system.webServer>
</configuration>
```

### 2.2 Control del servicio (equivalente a `apache2ctl` / `apachectl`)

| Comando `apachectl` | Equivalente en Windows Server 2025 |
|---|---|
| `start` | `iisreset /start` (todo IIS) o `Start-IISSite -Name "Default Web Site"` (un sitio) |
| `stop` | `iisreset /stop` o `Stop-IISSite -Name "Default Web Site"` |
| `restart` | `iisreset` (reinicia todos los servicios de IIS; corta las conexiones) |
| `graceful` | **Reciclar el grupo de aplicaciones**: `Restart-WebAppPool DefaultAppPool`. Por defecto el reciclado es "solapado": arranca un proceso nuevo antes de cerrar el antiguo, así que no se pierden peticiones |
| `gracefulstop` | `Stop-WebAppPool DefaultAppPool` (termina las peticiones en curso dentro del tiempo de cierre configurado) |
| `status` | `iisreset /status`, `Get-IISSite`, `appcmd list site` |
| `fullstatus` | `appcmd list wp` (procesos en marcha) y `appcmd list requests` (peticiones que se están atendiendo en este momento) |
| `configtest` | No existe. Si un archivo de configuración tiene un error, IIS responde con el error **500.19** y lo anota en el Visor de eventos |
| `help` | `appcmd /?`, `Get-Command -Module IISAdministration` |

**Copias de la configuración** (muy útil, ya que no hay `configtest`):

| Comando | Función |
|---|---|
| `appcmd add backup AntesDelCambio` | Guarda una copia de toda la configuración de IIS |
| `appcmd list backup` | Lista las copias |
| `appcmd restore backup AntesDelCambio` | Restaura una copia |
| `C:\inetpub\history\` | IIS guarda además **copias automáticas** de `applicationHost.config` cada vez que cambia (servicio de historial de configuración) |

### 2.3 Ficheros de log

| Log de Apache | Equivalente en IIS | Ubicación |
|---|---|---|
| `access.log` / `access_log` | **Registro de IIS** de cada sitio (un archivo por día, formato W3C por defecto) | `C:\inetpub\logs\LogFiles\W3SVC1\u_exAAMMDD.log` (`W3SVC1` = sitio con ID 1) |
| `error.log` / `error_log` | Visor de eventos (registro **Sistema** y **Aplicación**, orígenes `W3SVC` y `WAS`) | — |
| — | **Registro de errores de `http.sys`** (peticiones rechazadas antes de llegar a IIS) | `C:\Windows\System32\LogFiles\HTTPERR\` |
| — | **Seguimiento de solicitudes con error** (paso a paso de cada petición que falla; se activa por sitio) | `C:\inetpub\logs\FailedReqLogFiles\` |

El formato **NCSA** de IIS es el mismo "formato común" de los registros de Apache. Cambiar la carpeta de registro de un sitio:
```
Set-ItemProperty "IIS:\Sites\Default Web Site" -Name logFile.directory -Value "D:\LogsWeb"
```
IIS **no borra ni comprime** los registros antiguos (no hay equivalente de `logrotate`): hay que hacerlo con una tarea programada.

---

## 3. Alojamiento web para usuarios

IIS **no tiene equivalente de `UserDir`**. Lo más parecido es crear un **directorio virtual** por usuario que apunte a una carpeta suya:
```
New-WebVirtualDirectory -Site "Default Web Site" -Name "juan" -PhysicalPath "C:\Users\juan\public_html"
```
Acceso: `http://servidor/juan/archivo`. Igual que en Linux, la identidad del grupo de aplicaciones (`IIS AppPool\DefaultAppPool`) y el usuario anónimo (`IUSR`) necesitan permiso de lectura en esa carpeta (permisos NTFS, por ejemplo con `icacls`).

---

## 4. Hosting virtual

En IIS cada "host virtual" es un **sitio** (*site*) independiente, con sus enlaces, su carpeta y su configuración.

### 4.1 Basado en nombre

Todos los sitios comparten IP y puerto, y se distinguen por el **nombre de host** del enlace (equivale a varios `<VirtualHost>` con distinto `ServerName`):
```
New-IISSite -Name "host1" -PhysicalPath "C:\inetpub\host1" -BindingInformation "192.168.1.77:80:www.myhost1.com"
New-IISSite -Name "host2" -PhysicalPath "C:\inetpub\host2" -BindingInformation "192.168.1.77:80:www.myhost2.com"
```
Con `appcmd`: `appcmd add site /name:host1 /bindings:http/192.168.1.77:80:www.myhost1.com /physicalPath:C:\inetpub\host1`.

`NameVirtualHost` no tiene equivalente: no hace falta.

### 4.2 Basado en IP

Cada sitio usa una IP distinta del servidor y el nombre de host se deja vacío:
```
New-IISSite -Name "myhost1" -PhysicalPath "C:\inetpub\myhost1" -BindingInformation "192.168.1.77:80:"
New-IISSite -Name "myhost2" -PhysicalPath "C:\inetpub\myhost2" -BindingInformation "192.168.1.78:80:"
```

La ventaja que menciona el libro (procesos separados para cada sitio) en IIS se consigue con **cualquier** tipo de hosting virtual: basta con dar a cada sitio su propio **grupo de aplicaciones**:
```
New-WebAppPool -Name "host1"
Set-ItemProperty "IIS:\Sites\host1" -Name applicationPool -Value "host1"
```
Cada grupo es un proceso `w3wp.exe` distinto, con su propia identidad: si un sitio falla, no afecta a los demás.

---

## 5. Restricción de acceso

### 5.1 Autenticación por usuario/contraseña

En IIS los usuarios son **cuentas de Windows** (locales o de Active Directory), así que **no existe `htpasswd`**: los usuarios se crean como cualquier cuenta (`New-LocalUser`, o en Active Directory) y se agrupan en grupos de Windows.

| Módulo de Apache | Servicio de rol de IIS | Descripción |
|---|---|---|
| `mod_auth_basic` / `mod_authn_file` | **Autenticación básica** (`Web-Basic-Auth`) | Usuario y contraseña de Windows. Viajan sin cifrar: usar siempre con HTTPS |
| `mod_authnz_ldap` | **Autenticación de Windows** (`Web-Windows-Auth`) | Kerberos/NTLM con Active Directory; inicio de sesión automático para los equipos del dominio |
| — | **Autenticación de certificado de cliente** (`Web-Cert-Auth`) | Con certificados |
| `Require` | **Autorización de URL** (`Web-Url-Auth`) | Qué usuarios o grupos pueden entrar |
| `mod_authn_anon` | Autenticación anónima (activada por defecto, con la cuenta `IUSR`) | Acceso sin usuario |

La autenticación implícita (*Digest*) está obsoleta en Windows Server 2025.

**Proteger una carpeta** (equivalente al ejemplo del libro): desactivar el acceso anónimo y activar la autenticación básica:
```
Install-WindowsFeature Web-Basic-Auth, Web-Url-Auth
Set-WebConfigurationProperty -PSPath IIS:\ -Location "Default Web Site/privado" -Filter system.webServer/security/authentication/anonymousAuthentication -Name enabled -Value $false
Set-WebConfigurationProperty -PSPath IIS:\ -Location "Default Web Site/privado" -Filter system.webServer/security/authentication/basicAuthentication -Name enabled -Value $true
```
Con el acceso anónimo desactivado, cualquier usuario válido puede entrar (equivale a `Require valid-user`). Para permitir solo un grupo, se añade una regla de autorización en el `web.config` de la carpeta:
```xml
<configuration>
  <system.webServer>
    <security>
      <authorization>
        <remove users="*" roles="" verbs="" />
        <add accessType="Allow" roles="UsuariosWeb" />
      </authorization>
    </security>
  </system.webServer>
</configuration>
```

| Directiva de Apache | Equivalente en IIS |
|---|---|
| `AuthName` | Atributo `realm` de la autenticación básica |
| `AuthType Basic` | `basicAuthentication enabled="true"` |
| `AuthUserFile` | No aplica: cuentas de Windows / Active Directory |
| `AuthGroupFile` | Grupos de Windows / Active Directory |
| `Require valid-user` | Anónimo desactivado (o regla `<add accessType="Allow" users="*" />`) |
| `Require group` | Regla `<add accessType="Allow" roles="Grupo" />` |
| `AllowOverride` | **Delegación de características**: cada sección de configuración está "bloqueada" o "desbloqueada" para los `web.config`. Las secciones de autenticación vienen **bloqueadas** por defecto; por eso el ejemplo las escribe en `applicationHost.config` con `-PSPath IIS:\ -Location`. Desbloquear: `appcmd unlock config -section:system.webServer/security/authentication/basicAuthentication` |

### 5.2 Restricción por IP

Se instala el servicio de rol **Restricciones de dominio y direcciones IP** (`Install-WindowsFeature Web-IP-Security`). El ejemplo del libro (`Deny from All` + `Allow from 192.168.1.0/24`) queda así:
```
Set-WebConfigurationProperty -PSPath IIS:\ -Location "Default Web Site" -Filter system.webServer/security/ipSecurity -Name allowUnlisted -Value $false
Add-WebConfigurationProperty -PSPath IIS:\ -Location "Default Web Site" -Filter system.webServer/security/ipSecurity -Name "." -Value @{ipAddress="192.168.1.0"; subnetMask="255.255.255.0"; allowed="true"}
```
`allowUnlisted = false` equivale a `Deny from All`: se deniega todo lo que no esté en la lista.

| Apache | IIS |
|---|---|
| `Order Deny,Allow` + `Deny from All` | `allowUnlisted="false"` |
| `Allow from IP/red` | `<add ipAddress="..." subnetMask="..." allowed="true" />` |
| `Deny from IP` | `<add ipAddress="..." allowed="false" />` |
| Restricción por nombre de dominio | `enableReverseDns="true"` + `<add domainName="..." />` (lento: hace una consulta DNS inversa por petición) |
| — | **Restricciones dinámicas de IP**: bloquea automáticamente las IP que hacen demasiadas peticiones (sección `dynamicIpSecurity`) |

---

## 6. Contenido dinámico

| Método en Apache | Equivalente en IIS |
|---|---|
| CGI | Servicio de rol **CGI** (`Web-CGI`), que incluye también **FastCGI** (procesos del intérprete que se reutilizan, mucho más rápido que CGI) |
| `mod_php` | **PHP con FastCGI**: se descarga PHP para Windows (versión *Non Thread Safe*) de php.net y se asocia `php-cgi.exe` a los archivos `.php` |
| `mod_perl` | Perl para Windows mediante FastCGI o CGI |
| Aplicaciones .NET (sin equivalente en Linux) | **ASP.NET** (`Web-Asp-Net45`) y **ASP.NET Core** (con el *Hosting Bundle* de .NET), que se ejecutan dentro de los procesos de IIS |
| `AddHandler` | **Asignaciones de controlador** (*handler mappings*) |

Ejemplo, asociar `.php` a PHP (equivalente a `AddHandler`):
```
Add-WebConfiguration -PSPath IIS:\ -Filter system.webServer/fastCgi -Value @{fullPath="C:\PHP\php-cgi.exe"}
New-WebHandler -Name "PHP" -Path "*.php" -Verb "*" -Modules FastCgiModule -ScriptProcessor "C:\PHP\php-cgi.exe" -ResourceType File
```

---

## 7. HTTPS (SSL/TLS) en IIS

### 7.1 Instalación

**No hay que instalar nada**: el cifrado lo hace Windows (el componente **Schannel**) junto con `http.sys`. No existe un paquete como `mod_ssl` ni hace falta `openssl`.

| Linux | Windows Server 2025 |
|---|---|
| Paquete `mod_ssl` | Integrado |
| `openssl` | Integrado en Windows: cmdlets de certificados (`New-SelfSignedCertificate`, `Import-PfxCertificate`), `certreq`, `certutil` |
| `/etc/ssl/`, `/etc/pki/` | **Almacén de certificados** del equipo: `Cert:\LocalMachine\My` en PowerShell, o la consola `certlm.msc` |

Las **versiones de TLS** y los **algoritmos de cifrado** se configuran para **todo el sistema**, no en el servidor web:
- TLS 1.0 y TLS 1.1 vienen **desactivados por defecto** en Windows Server 2025. Las versiones se controlan en el Registro, en `HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols`.
- Algoritmos: `Get-TlsCipherSuite`, `Disable-TlsCipherSuite -Name "..."`, `Enable-TlsCipherSuite`, o la directiva de grupo "Orden de conjuntos de cifrado SSL".

### 7.2 Generar clave y certificado

| Paso del libro | Equivalente en Windows Server 2025 |
|---|---|
| Clave + certificado autofirmado (pruebas) | `New-SelfSignedCertificate -DnsName www.empresa.com -CertStoreLocation Cert:\LocalMachine\My -NotAfter (Get-Date).AddYears(1)` (crea la clave y el certificado en el almacén) |
| Generar una solicitud (CSR) | `certreq -new solicitud.inf solicitud.csr` (el archivo `.inf` indica el nombre, el tamaño de clave...) o Administrador de IIS → **Certificados de servidor** → Crear solicitud de certificado |
| Instalar el certificado que devuelve la CA | `certreq -accept certificado.cer` |
| Ser tu propia CA (`CA.pl -newca`) | Rol **Servicios de certificados de Active Directory** (AD CS): `Install-WindowsFeature ADCS-Cert-Authority -IncludeManagementTools` y después `Install-AdcsCertificationAuthority -CAType StandaloneRootCA` |
| Importar un certificado con su clave | `Import-PfxCertificate -FilePath C:\certs\web.pfx -CertStoreLocation Cert:\LocalMachine\My -Password (Read-Host -AsSecureString)` |
| Juntar un `.cer` y su `.key` (formato de Linux) en un `.pfx` | `certutil -mergepfx web.cer web.pfx` (con `web.key` en la misma carpeta) |
| Certificados gratuitos de Let's Encrypt | Clientes ACME de terceros para Windows, como win-acme, que también los renuevan solos |

La clave privada se guarda **protegida dentro del almacén**, no en un archivo `server.key` suelto.

### 7.3 Instalar y activar en IIS

Se añade un **enlace HTTPS** al sitio con la huella (*thumbprint*) del certificado:
```
$cert = Get-ChildItem Cert:\LocalMachine\My | Where-Object Subject -like "*www.empresa.com*"
New-IISSiteBinding -Name "Default Web Site" -BindingInformation "*:443:www.empresa.com" -Protocol https -CertificateThumbPrint $cert.Thumbprint -CertStoreLocation "Cert:\LocalMachine\My" -SslFlag Sni
```
Comprobar los certificados asociados a cada puerto en `http.sys`: `netsh http show sslcert`.

**Equivalencia de las directivas SSL:**

| Directiva de Apache | Equivalente en IIS / Windows Server 2025 |
|---|---|
| `Listen 443` + `SSLEngine On` | Enlace `https` en el puerto 443 |
| `SSLCertificateFile` / `SSLCertificateKeyFile` | Certificado (con su clave) en `Cert:\LocalMachine\My`, indicado por su huella |
| `SSLCertificateChainFile` | Certificados intermedios en el almacén **Entidades de certificación intermedias** (`Cert:\LocalMachine\CA`) |
| `SSLCACertificateFile` (certificados de cliente) | Almacén de **entidades raíz de confianza** + ajuste de certificados de cliente del sitio (Configuración de SSL → Requerir) |
| `SSLProtocol` / `SSLCipherSuite` | Configuración de Schannel para todo el sistema (apartado 7.1) |
| `ServerTokens` (ocultar la versión) | `Set-WebConfigurationProperty -PSPath IIS:\ -Filter system.webServer/security/requestFiltering -Name removeServerHeader -Value $true` (quita la cabecera `Server`) |
| `ServerSignature` | Por defecto las páginas de error detalladas solo se ven desde el propio servidor (`errorMode="DetailedLocalOnly"`) |
| `TraceEnable Off` | Filtrado de solicitudes: `<requestFiltering><verbs><add verb="TRACE" allowed="false" /></verbs></requestFiltering>` |
| SNI | Admitido: opción `-SslFlag Sni` en el enlace |
| — | **HSTS** integrado en el sitio (atributos `hsts enabled`, `max-age`, `redirectHttpToHttps`) |

**Almacén centralizado de certificados** (servicio de rol `Web-CertProvider`): IIS puede leer los certificados en formato `.pfx` desde una **carpeta compartida**, uno por nombre de host. Es lo más parecido a la forma de trabajar de Apache (certificados en archivos) y es útil cuando varios servidores web comparten los mismos certificados.

---

## 8. Squid: servidor proxy caché

**Windows Server 2025 no incluye un proxy caché de navegación** como Squid. Microsoft tuvo productos de este tipo (ISA Server y después Forefront TMG), pero están retirados hace años.

| Opción | Descripción |
|---|---|
| Squid para Windows | Existen compilaciones de Squid para Windows mantenidas por terceros; las directivas de `squid.conf` son las mismas del libro |
| IIS + **Application Request Routing** (ARR) | El complemento ARR puede configurarse como proxy de reenvío con caché. Es una opción limitada y ARR apenas recibe actualizaciones |
| Máquina virtual Linux con Squid | La opción más completa si se necesita Squid tal cual |
| Proxy en la nube o en el firewall perimetral | Solución habitual hoy en empresas |

| Directiva / ACL de Squid | Situación en Windows Server 2025 |
|---|---|
| `http_port`, `cache_dir` | Solo con Squid (para Windows o en Linux) |
| `acl ... dstdomain` + `http_access deny` (bloquear sitios) | Sin proxy: bloqueo por DNS con directivas de resolución de consultas (Capítulo 8) o reglas del firewall |
| `acl ... time` (horarios) | Las directivas DNS admiten condiciones de hora (`-TimeOfDay`) |
| `auth_param basic ... pam_auth` | En Squid para Windows, autenticación contra Windows / Active Directory mediante sus asistentes (*helpers*) |

### 8.6 Configuración del cliente

| Método | Comando / herramienta |
|---|---|
| Proxy para los programas y servicios del sistema (WinHTTP) | `netsh winhttp set proxy proxy-server="proxy.empresa.local:3128" bypass-list="*.empresa.local"`. Ver: `netsh winhttp show proxy`. Quitar: `netsh winhttp reset proxy` |
| Proxy de los usuarios (navegadores) | Configuración → Red e Internet → Proxy, o en masa con **Directiva de grupo** |
| Configuración automática | Archivo **PAC** o descubrimiento **WPAD** |

Igual que dice el libro: para obligar a usar el proxy hay que bloquear en el firewall o el router la salida directa a Internet.

---

## 9. Nginx: servidor web y proxy inverso

### 9.1 Características

- La arquitectura asíncrona de Nginx la tiene también **IIS**: `http.sys` recibe las conexiones en el kernel y los procesos `w3wp.exe` atienden muchas peticiones a la vez con un conjunto de hilos, sin un proceso o hilo fijo por cliente.
- El papel de **proxy inverso con balanceo de carga** lo cumple IIS con dos complementos gratuitos de Microsoft: **URL Rewrite** (reescribe y reenvía peticiones) y **Application Request Routing** (ARR: granjas de servidores, balanceo, comprobación de estado y caché). Se descargan de la web oficial de IIS e instalan como paquetes MSI.

### 9.2 Instalación

| Opción | Situación |
|---|---|
| Nginx para Windows | Existe versión oficial, pero su propia documentación la considera **beta**, con limitaciones de rendimiento; no se recomienda en producción |
| IIS + URL Rewrite + ARR | Opción nativa recomendada en Windows |

Tras instalar ARR, hay que activar la función de proxy:
```
Set-WebConfigurationProperty -PSPath MACHINE/WEBROOT/APPHOST -Filter system.webServer/proxy -Name enabled -Value $true
```

Igual que en el libro con Apache y Nginx: si hay dos servidores web en la misma máquina, no pueden usar los dos el puerto 80 en la misma IP.

### 9.3 Estructura básica de configuración

| Directiva de Nginx | Equivalente en IIS |
|---|---|
| `server { ... }` | **Sitio** de IIS |
| `listen 80` | Enlace del sitio (`*:80:`) |
| `root` | Ruta física del sitio |
| `index` | Documento predeterminado |
| `server_name` | Nombre de host del enlace |
| `location /ruta { ... }` | Etiqueta `<location path="...">` en la configuración, o un `web.config` en esa carpeta |
| `try_files $uri $uri/ =404` | Comportamiento por defecto de IIS |
| `proxy_pass` | Regla de **URL Rewrite** con acción `Rewrite` hacia otro servidor o granja |
| `upstream` (grupo de servidores) | **Granja de servidores** de ARR (`webFarms`) |

**Ejemplo de proxy inverso con balanceo** (equivalente a `upstream` + `proxy_pass` de Nginx). En `applicationHost.config`, la granja:
```xml
<webFarms>
  <webFarm name="Granja1" enabled="true">
    <server address="10.0.0.11" enabled="true" />
    <server address="10.0.0.12" enabled="true" />
  </webFarm>
</webFarms>
```
Y una regla de URL Rewrite que envía todo a la granja:
```xml
<rewrite>
  <globalRules>
    <rule name="ProxyGranja1" stopProcessing="true">
      <match url="(.*)" />
      <action type="Rewrite" url="http://Granja1/{R:1}" />
    </rule>
  </globalRules>
</rewrite>
```
Lo mismo se puede hacer desde el Administrador de IIS (nodo **Granjas de servidores**), que crea la regla automáticamente.

---

## 10. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 9) | Equivalente en Windows Server 2025 |
|---|---|
| Paquete `apache2` / `httpd` | Rol IIS (`Install-WindowsFeature Web-Server`) |
| `apache2.conf` / `httpd.conf` | `C:\Windows\System32\inetsrv\config\applicationHost.config` |
| `.htaccess` | `web.config` |
| `/var/www/html` | `C:\inetpub\wwwroot` |
| Usuario `www-data` / `apache` | Identidad del grupo de aplicaciones (`IIS AppPool\...`) e `IUSR` |
| Proceso `apache2` / `httpd` | `w3wp.exe` (grupos de aplicaciones) + `http.sys` |
| `LoadModule` / `a2enmod` | Servicios de rol (`Install-WindowsFeature Web-...`), `Enable-WebGlobalModule` |
| `Listen` / `ServerName` | Enlaces del sitio (`IP:puerto:nombre`) |
| `DocumentRoot` / `DirectoryIndex` | Ruta física / documento predeterminado |
| `apachectl start/stop/restart` | `iisreset`, `Start-IISSite` / `Stop-IISSite` |
| `apachectl graceful` | Reciclar el grupo de aplicaciones (`Restart-WebAppPool`) |
| `apachectl configtest` | No existe (error 500.19); copias con `appcmd add backup` |
| `access.log` | `C:\inetpub\logs\LogFiles\W3SVC<n>\` |
| `error.log` | Visor de eventos + `HTTPERR` + seguimiento de solicitudes con error |
| `logrotate` | Tarea programada propia |
| `UserDir` | Directorio virtual por usuario |
| `<VirtualHost>` | Sitio de IIS (por nombre de host o por IP) |
| `htpasswd` + `AuthUserFile` | Cuentas de Windows / Active Directory |
| `AuthType Basic` + `Require` | Autenticación básica / de Windows + Autorización de URL |
| `AllowOverride` | Delegación de características (bloqueo de secciones) |
| `Order` / `Allow from` / `Deny from` | Restricciones de dominio y direcciones IP (`ipSecurity`) |
| CGI / `mod_php` / `AddHandler` | CGI y FastCGI / PHP con FastCGI / asignaciones de controlador |
| `mod_ssl` + `openssl` | Integrados (Schannel, cmdlets de certificados, `certreq`, `certutil`) |
| `/etc/ssl/` | Almacén `Cert:\LocalMachine\My` (`certlm.msc`) |
| `CA.pl -newca` | AD CS o `New-SelfSignedCertificate` |
| `SSLProtocol` / `SSLCipherSuite` | Schannel (Registro) y `Disable-TlsCipherSuite` |
| `ServerTokens` / `TraceEnable` | `removeServerHeader` / filtrado de verbos |
| SNI | `-SslFlag Sni` |
| Squid | Sin equivalente integrado (Squid para Windows, ARR o Linux) |
| Proxy en el cliente | `netsh winhttp set proxy`, Directiva de grupo, PAC/WPAD |
| Nginx (proxy inverso) | IIS + URL Rewrite + Application Request Routing |
| `upstream` / `proxy_pass` | Granja de servidores de ARR / regla de URL Rewrite |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 9 ("Offering Web Services") del libro LPIC-2. Fuentes: documentación oficial de Microsoft Learn e iis.net (IIS 10.0, applicationHost.config y web.config, appcmd, módulos IISAdministration y WebAdministration, autenticación y autorización, restricciones de IP, FastCGI, enlaces HTTPS y SNI, HSTS, almacén centralizado de certificados, Schannel y TLS en Windows Server, URL Rewrite y Application Request Routing).*
