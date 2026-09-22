# LPIC-2 · Capítulo 10: Sharing Files
### Equivalencias en Windows Server 2025

Aviso para este capítulo: Samba es, precisamente, la implementación en Linux del protocolo **SMB/CIFS que es nativo de Windows**. Por eso esta sección no traduce herramientas de Linux a Windows en el sentido habitual: describe cómo Windows Server ofrece de forma **nativa** lo mismo que Samba imita en Linux. NFS y FTP sí son roles opcionales instalables en Windows, igual que en Linux.

---

## 1. Compartición de archivos SMB (equivalente nativo a Samba)

### 1.1 El rol y el servicio

| Concepto Linux (Samba) | Equivalente en Windows Server 2025 |
|---|---|
| Paquete `samba` | Rol **File and Storage Services** (o simplemente activar la compartición desde el propio Explorador de archivos, que ya viene con el sistema) |
| Daemon `smbd` | Servicio **Server** (`LanmanServer`), integrado en el núcleo del sistema, no un proceso de terceros |
| Daemon `nmbd` (NetBIOS) | Servicio **Computer Browser** / resolución NetBIOS, integrado igualmente en el sistema |
| Daemon `winbindd` (traducir usuarios de AD) | No hace falta: en Windows, los usuarios y grupos de Active Directory **son** directamente los mismos usuarios del sistema, sin necesidad de una capa de traducción |
| Base de datos de usuarios (`tdbsam`/`ldapsam`) | Cuentas de usuario locales o de **Active Directory**, las mismas que para cualquier inicio de sesión en Windows |

### 1.2 Configuración: sin `smb.conf`

Diferencia clave: no existe un fichero de texto único equivalente a `smb.conf`. Cada recurso compartido se define como un objeto independiente (con su ruta, permisos y opciones), gestionado gráficamente desde el **Explorador de archivos** (pestaña Compartir) o el **Administrador del servidor**, o mediante el módulo PowerShell **`SmbShare`**.

**Crear un recurso compartido (equivalente a una sección `[nombre_share]` de `smb.conf`):**
```powershell
New-SmbShare -Name "Datos" -Path "D:\Datos" `
  -FullAccess "DOMINIO\Administradores" `
  -ChangeAccess "DOMINIO\Empleados" `
  -ReadAccess "DOMINIO\Invitados" `
  -Description "Recurso compartido del departamento"
```

| Cmdlet | Equivalente conceptual a... |
|---|---|
| `New-SmbShare` | Crear una sección `[share]` en `smb.conf` |
| `Get-SmbShare` | Ver los recursos definidos (equivalente a `testparm` mostrando la configuración) |
| `Set-SmbShare` | Modificar las opciones de un recurso ya creado |
| `Remove-SmbShare` | Eliminar un recurso compartido |
| `Get-SmbShareAccess` | Consultar los permisos ACL de un recurso (equivalente a `smbcacls`) |
| `Grant-SmbShareAccess` / `Revoke-SmbShareAccess` | Añadir/quitar permisos de acceso a un recurso ya creado |
| `Get-SmbSession` | Ver las conexiones activas al servidor (equivalente a `smbstatus`) |
| `Get-SmbOpenFile` | Ver qué ficheros concretos tiene abiertos cada cliente |
| `Close-SmbSession` | Cierra una sesión de cliente concreta a la fuerza |

### 1.3 Permisos: doble capa (compartido + NTFS)

Diferencia importante frente a Samba: en Windows, el acceso final a un recurso compartido resulta de **combinar dos capas de permisos**: los del propio recurso compartido (`FullAccess`/`ChangeAccess`/`ReadAccess`, arriba) y los permisos **NTFS** de la carpeta física subyacente (equivalentes a los permisos Unix que en Linux ya delimitan el acceso real bajo Samba). El permiso efectivo es siempre el más restrictivo de ambas capas.

### 1.4 Montar un recurso SMB en el cliente

| Concepto Linux | Equivalente en Windows |
|---|---|
| `mount.cifs`, entrada `cifs` en `/etc/fstab` | `New-SmbMapping -LocalPath "Z:" -RemotePath "\\servidor\recurso" -Persistent $true` (equivalente a montar y hacerlo persistente entre reinicios) |
| `smbclient` (acceso tipo FTP a un recurso) | Simplemente `\\servidor\recurso` desde el Explorador de archivos, o `Get-ChildItem \\servidor\recurso` desde PowerShell |
| Fichero de credenciales (`credentials=`) | `New-SmbMapping ... -UserName "usuario" -Password "clave"`, o gestión de credenciales guardadas con `cmdkey` |

### 1.5 Puertos

| Concepto Linux (Samba) | Windows Server 2025 |
|---|---|
| 137/138/139 (NetBIOS) | Igual, si se mantiene compatibilidad con clientes muy antiguos |
| 445 (SMB sobre TCP, moderno) | Igual, es el puerto por defecto de cualquier recurso compartido de Windows |
| 445 también para SMB sobre QUIC (novedad reciente) | **SMB sobre QUIC**, disponible en Windows Server 2022+ y ampliado en 2025, permite acceder a recursos compartidos de forma cifrada a través de Internet sin VPN, sin equivalente directo en el libro |

### 1.6 Solución de problemas

| Concepto Linux (Samba) | Equivalente Windows |
|---|---|
| `testparm` | `Get-SmbShare`, o simplemente el Administrador del servidor mostrando el recurso |
| `smbstatus` | `Get-SmbSession`, `Get-SmbOpenFile` |
| `nmblookup` | `nbtstat -A IP` (resolución NetBIOS clásica) |
| Log de Samba (`log level`) | Visor de eventos, registro relacionado con SMB Server (`Microsoft-Windows-SMBServer`) |

---

## 2. NFS

Windows Server también incluye un rol **NFS Server** nativo (Server for NFS), pensado para compartir recursos con clientes Unix/Linux — el escenario inverso al habitual, pero perfectamente soportado.

| Concepto Linux (servidor NFS) | Equivalente en Windows Server 2025 |
|---|---|
| Paquete del servidor NFS | Rol **Server for NFS**: `Install-WindowsFeature FS-NFS-Service` |
| `/etc/exports` | No hay fichero de texto: cada recurso NFS se crea como objeto independiente con el módulo PowerShell **`NFS`** |
| `exportfs -a` | `New-NfsShare` |
| Opciones `rw`/`ro` | Parámetro `-Permission` (`ReadWrite`/`ReadOnly`/`NoAccess`) |
| `root_squash`/`no_root_squash`/`all_squash` | Parámetros `-AllowRootAccess`, `-EnableAnonymousAccess`, `-AnonymousUid`/`-AnonymousGid` |
| Autenticación (`sys`, Kerberos) | Parámetro `-Authentication` (`Sys`, `Krb5`, `Krb5i`, `Krb5p`) |

**Crear un recurso NFS (equivalente a una línea de `/etc/exports`):**
```powershell
New-NfsShare -Name "datos_unix" -Path "D:\DatosUnix" `
  -Permission ReadWrite -AllowRootAccess $false -Authentication Sys
```

| Cmdlet | Equivalente conceptual a... |
|---|---|
| `New-NfsShare` | Añadir una línea a `/etc/exports` |
| `Get-NfsShare` | `exportfs -v` (ver recursos exportados) |
| `Set-NfsShare` | Modificar opciones de un export existente |
| `Remove-NfsShare` | Desexportar un recurso |
| `Get-NfsSession` | Ver clientes NFS conectados (equivalente a `showmount -a`) |
| `Get-NfsMappingStore` / mapeo de identidades | Sustituye a la resolución de UID/GID entre sistemas, ya que Windows y Unix no comparten el mismo espacio de identificadores de usuario por defecto |

---

## 3. FTP

### 3.1 El rol FTP de IIS

| Concepto Linux (vsftpd) | Equivalente en Windows Server 2025 |
|---|---|
| Paquete `vsftpd` | Característica **FTP Server** dentro del rol Web Server (IIS): `Install-WindowsFeature Web-Ftp-Server` |
| `vsftpd.conf` | Configuración vía **IIS Manager**, `appcmd.exe`, o el módulo `WebAdministration` de PowerShell — sin fichero de texto único |
| Directorio raíz por defecto (`/var/ftp`) | Ruta física del sitio FTP, definida al crearlo (cualquier carpeta) |

**Crear un sitio FTP (equivalente a levantar vsftpd con su configuración básica):**
```powershell
Import-Module WebAdministration
New-WebFtpSite -Name "MiFTP" -Port 21 -PhysicalPath "D:\FTP" -IPAddress "*"
```

### 3.2 Correspondencia de directivas de `vsftpd.conf`

| Directiva vsftpd | Equivalente en IIS FTP |
|---|---|
| `anonymous_enable` | Habilitar **Autenticación anónima** en la configuración de autenticación del sitio FTP |
| `local_enable` | Habilitar **Autenticación básica**, usando cuentas locales/de dominio de Windows |
| `write_enable`, `anon_upload_enable` | **Autorización FTP** (FTP Authorization Rules): reglas de permiso de lectura/escritura por usuario o rol, configurables con `Add-WebConfiguration` sobre la sección `system.ftpServer/security/authorization` |
| `chroot_local_user` | **Aislamiento de usuario** (User Isolation), función nativa de IIS FTP que restringe a cada usuario a su propia carpeta, sin necesidad de configurarlo como un "chroot" manual |
| `anon_root` | Ruta física asignada al sitio (o a la carpeta del usuario anónimo dentro del aislamiento de usuario) |
| Modo pasivo (`pasv_...` en otros FTP) | Configuración de **Firewall Support** en las propiedades del sitio FTP (rango de puertos pasivos) |

### 3.3 HTTPS/FTPS (equivalente al `SSL/TLS Setting` del libro)

Igual que con IIS web (Capítulo 9), el cifrado FTP (**FTPS**) se activa asociando un certificado SSL/TLS al sitio FTP desde sus propiedades de "SSL", reutilizando el mismo modelo de certificados del almacén de Windows.

---

## 4. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 10) | Equivalente en Windows Server 2025 |
|---|---|
| Samba (`smbd`, `smb.conf`) | SMB **nativo** del sistema (servicio Server, módulo `SmbShare`) |
| Secciones `[share]` de `smb.conf` | `New-SmbShare`/`Set-SmbShare` |
| `smbstatus` | `Get-SmbSession`, `Get-SmbOpenFile` |
| `testparm` | `Get-SmbShare` |
| Montar recurso SMB (`mount.cifs`) | `New-SmbMapping` |
| Servidor NFS (`/etc/exports`) | Rol **Server for NFS**, `New-NfsShare` |
| `exportfs`/`showmount` | `Get-NfsShare`/`Get-NfsSession` |
| vsftpd / Pure-FTPd | Característica **FTP Server** de IIS |
| `chroot_local_user` | Aislamiento de usuario (User Isolation) nativo de IIS FTP |
| FTPS (SSL/TLS) | Certificado SSL/TLS asociado al sitio FTP en IIS |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 10 ("Sharing Files") del libro LPIC-2. Fuentes: conocimiento general de administración de Windows Server, verificado con documentación oficial de Microsoft Learn (`New-SmbShare`, `New-NfsShare`, `New-WebFtpSite` para Windows Server 2025).*
