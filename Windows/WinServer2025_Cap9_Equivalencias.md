# LPIC-2 · Capítulo 9: Offering Web Services
### Equivalencias en Windows Server 2025

El servidor web nativo de Windows Server es **IIS** (Internet Information Services), un rol integrado desde hace décadas, conceptualmente equivalente a Apache. Windows no tiene un proxy caché nativo comparable a Squid; se cubre con el módulo **Application Request Routing (ARR)** de IIS o con software portado/de terceros.

---

## 1. Instalación y ficheros de configuración de IIS

| Concepto Linux (Apache) | Equivalente en Windows Server 2025 (IIS) |
|---|---|
| Paquete `apache2`/`httpd` | Rol **Web Server (IIS)** |
| Instalación | `Install-WindowsFeature -Name Web-Server -IncludeManagementTools` |
| `apache2.conf`/`httpd.conf` (config. global) | **`applicationHost.config`** (`%SystemRoot%\System32\inetsrv\config\`) — configuración a nivel de todo el servidor IIS |
| `.htaccess` (config. por carpeta) | **`web.config`** — fichero XML que puede colocarse en cualquier carpeta del sitio para sobrescribir configuración solo en esa carpeta y sus subcarpetas, igual que `.htaccess` |
| `Include` (repartir configuración en varios ficheros) | La configuración de IIS ya está jerarquizada de serie: `applicationHost.config` (servidor) → `web.config` de sitio → `web.config` de subcarpeta, en cascada |
| `DocumentRoot` | Carpeta física del sitio (`PhysicalPath`), configurable por sitio o por directorio virtual |

**Formato de `web.config` (XML) frente al de Apache (directivas de texto plano):** es la diferencia estructural más notable; donde Apache usa líneas tipo `DirectiveName valor`, IIS usa XML anidado, por ejemplo:
```xml
<configuration>
  <system.webServer>
    <defaultDocument>
      <files>
        <add value="index.html" />
      </files>
    </defaultDocument>
  </system.webServer>
</configuration>
```

---

## 2. Gestión del servicio y del sitio

| Concepto Linux (`apache2ctl`) | Equivalente en Windows Server 2025 |
|---|---|
| `apache2ctl start`/`stop`/`restart` | `Start-Service W3SVC` / `Stop-Service W3SVC` / `Restart-Service W3SVC` (el servicio de IIS se llama **World Wide Web Publishing Service**) |
| `apache2ctl graceful` (reinicio sin cortar conexiones) | **`iisreset /noforce`** (espera a que terminen las peticiones activas antes de reiniciar) |
| `apache2ctl configtest` | No hay un "test de sintaxis" independiente: `applicationHost.config`/`web.config` son XML, y un error de XML hace fallar la carga inmediatamente, mostrando el error en el navegador o en el Visor de eventos |
| `apache2ctl status`/`fullstatus` | **IIS Manager** (`inetmgr.exe`), o `Get-Website`/`Get-WebAppPoolState` (PowerShell) |
| **`iisreset`** (sin opciones) | Reinicia IIS por completo (equivalente más brusco a `apache2ctl restart`) |
| **`appcmd.exe`** | Herramienta de línea de comandos clásica de IIS (equivalente funcional a `apache2ctl` + edición de configuración desde consola): `appcmd list sites`, `appcmd start site "nombre"`, `appcmd set config` |

---

## 3. Logs

| Concepto Apache | Equivalente IIS |
|---|---|
| `access.log`/`access_log` | Logs de IIS en `%SystemDrive%\inetpub\logs\LogFiles\W3SVC<n>\`, en formato W3C extendido (configurable) |
| `error.log`/`error_log` | IIS registra los errores en el mismo log W3C (con el código de estado HTTP) y además en el **Visor de eventos** para errores graves del propio servicio |

---

## 4. Alojamiento de sitios (equivalente a `UserDir` y hosting virtual)

### 4.1 Directorios virtuales (equivalente aproximado a `UserDir`)

| Concepto Apache | Equivalente IIS |
|---|---|
| `UserDir public_html` | **Directorio virtual** (Virtual Directory): una ruta URL que apunta a una carpeta física distinta de la del sitio principal, creado con `New-WebVirtualDirectory -Site "nombre" -Name "usuario" -PhysicalPath "C:\ruta"` |

### 4.2 Hosting virtual (equivalente a `VirtualHost`)

IIS no distingue entre "basado en nombre" y "basado en IP" como dos mecanismos separados: cualquier sitio se define por una combinación de **IP + puerto + encabezado de host (hostname)**, y basta con variar cuál de esos tres campos se deja fijo o distinto para lograr cualquiera de los dos modelos del libro.

| Concepto Apache | Equivalente IIS |
|---|---|
| `NameVirtualHost` + varios `<VirtualHost IP>` con distinto `ServerName` | Varios sitios con la misma IP:puerto pero distinto **encabezado de host** en el binding |
| `Listen IP:80` + varios `<VirtualHost IP_distinta>` | Varios sitios, cada uno con un **binding** a una IP distinta |
| Crear un sitio nuevo | `New-Website -Name "MiSitio" -PhysicalPath "C:\inetpub\misitio" -Port 80 -HostHeader "www.midominio.com"` |
| Añadir un binding adicional a un sitio existente | `New-WebBinding -Name "MiSitio" -IPAddress "*" -Port 80 -HostHeader "www.otrodominio.com"` |
| Consultar los bindings de un sitio | `Get-WebBinding -Name "MiSitio"` |

---

## 5. Restricción de acceso

### 5.1 Autenticación por usuario/contraseña

| Concepto Apache | Equivalente IIS |
|---|---|
| `mod_authn_file`, `htpasswd`, `AuthUserFile` | Característica **Autenticación básica** (Basic Authentication), que usa directamente las cuentas de usuario de Windows (local o de dominio) en vez de un fichero de contraseñas independiente |
| Instalar el módulo | `Install-WindowsFeature Web-Basic-Auth` |
| Activarlo en un sitio | `Set-WebConfigurationProperty -Filter /system.webServer/security/authentication/basicAuthentication -Name enabled -Value true -PSPath "IIS:\Sites\MiSitio"` |
| `Require valid-user` | Se controla con los **permisos NTFS** de la carpeta física del sitio, no con una directiva de Apache: solo los usuarios/grupos con permiso de lectura en esa carpeta pueden autenticarse con éxito |
| — (sin equivalente directo en el libro, propio de entornos Windows) | **Autenticación de Windows** (Windows Authentication / Kerberos), que permite inicio de sesión único (SSO) transparente para usuarios de un dominio Active Directory, sin pedir contraseña de nuevo |

### 5.2 Restricción por IP

| Concepto Apache | Equivalente IIS |
|---|---|
| `mod_access`, `Order Deny,Allow`, `Deny from All` | Característica **Restricciones de IP y dominio** (`Web-IP-Security`), gestionable desde IIS Manager o con `Add-WebConfiguration` sobre la sección `ipSecurity` |
| Instalar la característica | `Install-WindowsFeature Web-IP-Security` |

---

## 6. Contenido dinámico

| Concepto Apache | Equivalente IIS |
|---|---|
| CGI (proceso externo) | IIS también soporta CGI (`Web-CGI`), aunque en desuso |
| `mod_perl`, `mod_php` (módulo embebido en el proceso) | **ASP.NET** (embebido de forma nativa en el proceso de trabajo de IIS, `Web-Asp-Net45`), o **PHP** vía FastCGI (`Web-CGI` + configuración de PHP como manejador FastCGI) |
| `AddHandler` | Sección `<handlers>` en `web.config`, que asocia extensiones de fichero a un módulo/intérprete concreto |

---

## 7. HTTPS en IIS

| Concepto Apache (mod_ssl/OpenSSL) | Equivalente IIS |
|---|---|
| Instalar `mod_ssl` | No hace falta instalar nada aparte: el soporte HTTPS viene integrado en el propio rol Web-Server |
| `openssl genrsa`/`openssl req` (generar clave y CSR) | **`New-SelfSignedCertificate`** (PowerShell), para certificados de prueba; para producción, un CSR se genera desde IIS Manager ("Crear solicitud de certificado") y se envía a una CA |
| Actuar como tu propia CA de pruebas (`CA.pl -newca`) | No hay un equivalente de "CA casera" tan directo: se recurre a `New-SelfSignedCertificate` para pruebas, o a los **Servicios de certificados de Active Directory (AD CS)** si se necesita una CA interna real |
| Copiar certificado/clave a una carpeta y activar `SSLEngine On` | El certificado se instala en el **almacén de certificados de Windows** (Personal/Equipo local), y se asocia al sitio mediante un **binding HTTPS** |
| `SSLCertificateFile`/`SSLCertificateKeyFile` en el `VirtualHost` | `New-WebBinding -Name "MiSitio" -Protocol https -Port 443 -IPAddress "*"` seguido de `(Get-WebBinding ...).AddSslCertificate(huella_certificado, "my")` |
| `SSLProtocol`, `SSLCipherSuite` | Configuración de protocolos/cifrados TLS a nivel de sistema operativo (no por sitio), vía Registro (`SCHANNEL`) o la herramienta gráfica **IIS Crypto** (de terceros, muy usada para esta tarea) |
| SNI (varios hosts virtuales, un certificado por cada uno) | IIS soporta SNI de forma nativa desde IIS 8: al crear el binding HTTPS, se marca la opción "Requerir indicación de nombre de servidor (SNI)" |

**Ejemplo completo (equivalente al flujo del libro con Apache):**
```powershell
$cert = New-SelfSignedCertificate -DnsName "www.midominio.com" -CertStoreLocation "cert:\LocalMachine\My"
New-WebBinding -Name "MiSitio" -IPAddress "*" -Port 443 -Protocol https -SslFlags 1
(Get-WebBinding -Name "MiSitio" -Protocol https).AddSslCertificate($cert.Thumbprint, "my")
```

---

## 8. Squid → sin equivalente nativo directo

Windows Server no incluye un proxy caché propio comparable a Squid. Las opciones son:

| Opción | Descripción |
|---|---|
| **Application Request Routing (ARR)** para IIS | Módulo oficial de Microsoft (gratuito, descarga aparte) que convierte IIS en un **proxy inverso con caché**, cubriendo el mismo caso de uso de "caché web" del capítulo, aunque orientado sobre todo a reverse proxy más que a proxy de salida para clientes internos |
| **Squid para Windows** | El propio proyecto Squid tiene una versión portada para Windows, con la misma sintaxis de `squid.conf`, ACLs y `http_access` descritas en el libro — la opción más fiel si se necesita reproducir exactamente el comportamiento de Squid |
| Soluciones de terceros (WinGate, CCProxy) | Proxies comerciales para Windows con funciones de caché y control de acceso similares |

---

## 9. Nginx en Windows Server

Nginx es multiplataforma y **tiene una versión nativa para Windows**, descargable directamente del proyecto oficial; se instala y configura de forma prácticamente idéntica a como se describe en el libro (mismo fichero `nginx.conf`, misma sintaxis de bloques `server { }` y `location { }`), sin necesidad de WSL ni de una capa de compatibilidad. Para usarlo como servidor web principal o como reverse proxy en Windows Server 2025 se puede desplegar sin más adaptación que ajustar rutas de Windows en la configuración.

| Concepto | Windows Server 2025 |
|---|---|
| Papel principal de la plataforma | IIS (equivalente nativo a Apache) |
| Nginx como alternativa | Funciona igual que en Linux, instalado como servicio de Windows (con NSSM u otro gestor de servicios, ya que Nginx no se registra como servicio de forma nativa en Windows) |

---

## 10. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 9) | Equivalente en Windows Server 2025 |
|---|---|
| `apache2`/`httpd` | Rol **Web Server (IIS)** |
| `httpd.conf`/`apache2.conf` | `applicationHost.config` |
| `.htaccess` | `web.config` |
| `apache2ctl` | `iisreset`, `appcmd.exe`, cmdlets `*-Website`/`*-WebAppPoolState` |
| `VirtualHost` (name/IP-based) | Bindings de sitio (IP + puerto + encabezado de host) |
| `UserDir` | Directorios virtuales (`New-WebVirtualDirectory`) |
| `htpasswd`/`AuthUserFile` | Autenticación básica sobre cuentas Windows; alternativa nativa: Autenticación de Windows (Kerberos/SSO) |
| `mod_access` (restricción por IP) | Característica "Restricciones de IP y dominio" |
| `mod_ssl` + OpenSSL | Soporte HTTPS integrado + `New-SelfSignedCertificate` + bindings HTTPS |
| Squid | Sin equivalente nativo: **ARR** (reverse proxy en IIS) o **Squid para Windows** |
| Nginx | Misma herramienta, con versión nativa para Windows |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 9 ("Offering Web Services") del libro LPIC-2. Fuentes: conocimiento general de administración de Windows Server, verificado con documentación oficial de Microsoft Learn (`Install-WindowsFeature`, `New-WebBinding`, `New-IISSiteBinding` para Windows Server 2025).*
