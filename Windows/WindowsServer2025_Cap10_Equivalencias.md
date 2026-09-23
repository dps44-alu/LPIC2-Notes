# LPIC-2 · Capítulo 10: Sharing Files
### Equivalencias en Windows Server 2025

Este documento recoge, para cada concepto visto en el Capítulo 10 del libro LPIC-2 (Samba, NFS y FTP), cuál es su equivalente en **Windows Server 2025**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo:
- **Samba** es una imitación del protocolo **SMB**, que es el sistema propio de Windows para compartir archivos e impresoras. En Windows Server no hace falta instalar nada: SMB viene **integrado** (servicio **Servidor**, `LanmanServer`). Es la parte del capítulo donde Windows está "en casa".
- **NFS** está disponible como **servicio de rol** (Servidor para NFS) y también hay un **Cliente para NFS**. Funciona bien, pero requiere relacionar los usuarios de Unix (UID/GID) con cuentas de Windows.
- **FTP** lo ofrece **IIS** (servicio de rol Servidor FTP), configurado igual que un sitio web (Capítulo 9).

Todos los comandos se ejecutan en una consola **como Administrador**.

---

## 1. Samba → SMB nativo de Windows

### 1.1 Instalación y ubicación de ficheros

| Elemento en Linux (Samba) | Equivalente en Windows Server 2025 |
|---|---|
| Paquete `samba` | Integrado. El servicio de rol **Servidor de archivos** (`FS-FileServer`) se añade solo al crear la primera carpeta compartida; también se puede instalar con `Install-WindowsFeature FS-FileServer` |
| `/etc/samba/smb.conf` | No hay archivo: configuración global con `Get-SmbServerConfiguration` / `Set-SmbServerConfiguration`, y recursos compartidos guardados en el Registro (`HKLM\SYSTEM\CurrentControlSet\Services\LanmanServer\Shares`) |
| `/var/log/samba/` | Visor de eventos: registros `Microsoft-Windows-SMBServer/Operational`, `/Security` y `/Audit` (y `SMBClient/...` en el cliente) |
| Bases de usuarios (`tdbsam`, `ldapsam`, `smbpasswd`) | Cuentas locales (base de datos **SAM** del equipo, equivale a `tdbsam`) o cuentas de **Active Directory** (equivale a `ldapsam`) |

### 1.2 Daemons de Samba

| Daemon de Samba | Equivalente en Windows Server 2025 |
|---|---|
| `smbd` | Servicio **Servidor** (`LanmanServer`) y el controlador del kernel `srv2.sys`. El cliente es el servicio **Estación de trabajo** (`LanmanWorkstation`) |
| `nmbd` (NetBIOS) | **NetBIOS sobre TCP/IP** (NetBT), integrado en la pila de red. Solo lo necesitan equipos muy antiguos; hoy los nombres se resuelven por DNS |
| `winbindd` | Unir el servidor al dominio (`Add-Computer -DomainName empresa.local`): el servicio **Netlogon** se encarga de la relación con los controladores de dominio. Windows usa directamente las cuentas de AD, sin "traducirlas" |
| Función de servidor WINS | Característica **Servidor WINS** (`Install-WindowsFeature WINS`), considerada antigua |

**SMB1** (la versión antigua e insegura del protocolo) **no viene instalado** en Windows Server 2025 y no debe activarse. Windows usa SMB 2 y SMB 3.

### 1.3 a 1.5 Estructura de `smb.conf` y directivas

**Secciones especiales:**

| Sección de `smb.conf` | Equivalente en Windows Server 2025 |
|---|---|
| `[global]` | `Set-SmbServerConfiguration` (ajustes del servidor SMB) |
| `[nombre_share]` | Un recurso compartido: `New-SmbShare` |
| `[netlogon]` | Recursos `NETLOGON` y `SYSVOL`, que se crean solos en los **controladores de dominio** |
| `[printers]` | Rol **Servidor de impresión** (`Install-WindowsFeature Print-Server`) y compartir cada impresora: `Set-Printer -Name "HP01" -Shared $true -ShareName "HP01"` |
| `[profiles]` | **Perfiles móviles** (itinerantes): carpeta compartida + ruta del perfil en la cuenta de usuario (`Set-ADUser juan -ProfilePath \\SRV01\Perfiles$\juan`) |
| `[homes]` | **Carpetas particulares**: `Set-ADUser juan -HomeDirectory \\SRV01\Usuarios$\juan -HomeDrive H:` |

Windows crea además unos **recursos administrativos** ocultos: `C$` (cada disco), `ADMIN$` (carpeta de Windows) e `IPC$` (comunicación entre procesos), solo accesibles para administradores.

**Directivas globales (Tabla 10.4):**

| Directiva de Samba | Equivalente en Windows Server 2025 |
|---|---|
| `workgroup` | Grupo de trabajo o dominio del equipo: `Add-Computer -WorkgroupName OFICINA` o `Add-Computer -DomainName empresa.local` |
| `server string` | Descripción del equipo: `net config server /srvcomment:"Servidor de archivos"` |
| `interfaces` | Por defecto SMB escucha en todas las tarjetas. Para que no lo haga en una: `Disable-NetAdapterBinding -Name "Ethernet 2" -ComponentID ms_server` (quita "Compartir archivos e impresoras" de esa tarjeta) |
| `hosts allow` | Regla del firewall "Compartir archivos e impresoras (SMB de entrada)" limitada a unas IP: `Set-NetFirewallRule -Name "FPS-SMB-In-TCP" -RemoteAddress 192.168.56.0/24` |
| `log file` / `max log size` / `log level` | Registros de eventos de SMB (tamaño: `wevtutil sl Microsoft-Windows-SMBServer/Operational /ms:104857600`) y opciones de auditoría de `Set-SmbServerConfiguration` |
| `security = user` | Siempre es así: la autenticación es por usuario (la seguridad por recurso "share" desapareció hace décadas) |
| `passdb backend` | Cuentas locales (SAM) o Active Directory |

**Directivas de un recurso compartido (Tabla 10.6):**

| Directiva de Samba | Equivalente en `New-SmbShare` / `Set-SmbShare` |
|---|---|
| `comment` | `-Description "texto"` |
| `browseable = no` | Terminar el nombre en `$` (`Datos$`): el recurso existe pero no aparece en la lista |
| `valid users` | `-FullAccess`, `-ChangeAccess`, `-ReadAccess` (usuarios o grupos) |
| `invalid users` | `-NoAccess` o `Block-SmbShareAccess` |
| `path` | `-Path` |
| `public` / `guest ok` | No recomendable: el acceso de invitado está desactivado por defecto y los clientes Windows modernos lo rechazan |
| `guest only` | No aplica |
| `force group` | No aplica: el propietario y los permisos de lo que se crea dependen de la herencia de permisos NTFS |
| `writable = yes` | Permiso `-ChangeAccess` (cambiar) o `-FullAccess` |
| `read only = yes` | Permiso `-ReadAccess` |
| `write list` | Los usuarios o grupos puestos en `-ChangeAccess` |
| — | `-FolderEnumerationMode AccessBased`: cada usuario **solo ve** las carpetas a las que tiene acceso (enumeración basada en el acceso) |
| — | `-EncryptData $true`: cifra el tráfico SMB de ese recurso |
| — | `-ConcurrentUserLimit 50`: máximo de usuarios conectados a la vez |

**Muy importante — dos capas de permisos:** en Windows, el acceso a un recurso compartido depende de **dos permisos a la vez**: los **permisos del recurso compartido** (los de `New-SmbShare`) y los **permisos NTFS** de la carpeta (los de `icacls`). Se aplica **el más restrictivo** de los dos. Es parecido a Samba, donde también cuentan los permisos Linux de la carpeta. Una práctica habitual es dar permisos amplios en el recurso compartido y controlar el detalle con NTFS.

### 1.6 Pasos para configurar un recurso compartido

| Paso del libro | Equivalente en Windows Server 2025 |
|---|---|
| 1. Crear el directorio (`/srv/`) | `New-Item -ItemType Directory D:\Compartido` |
| 2. Modificar `smb.conf` | `New-SmbShare -Name Compartido -Path D:\Compartido -ChangeAccess "EMPRESA\Ventas" -ReadAccess "EMPRESA\Domain Users"` |
| 3. `testparm` | No hace falta (no hay archivo de texto) |
| 4. Crear el usuario local | `New-LocalUser -Name juan -Password (Read-Host -AsSecureString)` (o una cuenta de Active Directory) |
| 5. `smbpasswd -a` / `pdbedit` | No hace falta: las cuentas de Windows ya sirven para SMB |
| 6. Comprobar la cuenta | `Get-LocalUser juan` / `Get-ADUser juan` |
| 7. Arrancar el demonio | El servicio **Servidor** está siempre en marcha |
| 8. Comprobar los recursos ofrecidos | `Get-SmbShare` o `net share` |
| 9. Arranque automático | Ya lo es |
| 10. Firewall | Al crear el primer recurso compartido, Windows activa solo las reglas de "Compartir archivos e impresoras" |
| (Permisos de la carpeta) | `icacls D:\Compartido /grant "EMPRESA\Ventas:(OI)(CI)M"` (M = modificar, heredado por archivos y subcarpetas) |

Con la herramienta clásica `net`:
```
net share Compartido=D:\Compartido /GRANT:"EMPRESA\Ventas",CHANGE /REMARK:"Carpeta de ventas"
net share Compartido /DELETE
```

### 1.7 Utilidades de administración (Tabla 10.3)

| Utilidad de Samba | Equivalente en Windows Server 2025 |
|---|---|
| `mount.cifs` | `net use Z: \\SRV01\Compartido` o `New-SmbMapping` (Capítulo 4) |
| `net` | **`net`** de Windows (la de Samba imita a esta): `net share`, `net use`, `net session`, `net file`, `net view`, `net user` |
| `nmblookup` | `nbtstat -A 192.168.56.102` (nombres NetBIOS de un equipo), `nbtstat -n` (propios), `nbtstat -c` (caché) |
| `pdbedit -L` | `Get-LocalUser` / `net user` (locales) o `Get-ADUser -Filter *` (dominio) |
| `rpcclient` | No hace falta: Windows usa RPC de forma nativa en todas sus herramientas de administración remota |
| `smbcacls` | `icacls` (permisos NTFS) y `Get-SmbShareAccess` / `Grant-SmbShareAccess` / `Revoke-SmbShareAccess` (permisos del recurso) |
| `smbclient -L //servidor` | `net view \\SRV01 /all` o `Get-SmbShare -CimSession SRV01` |
| `smbclient` (acceso tipo FTP) | No hace falta: se usa directamente la ruta `\\SRV01\Compartido` con cualquier comando (`dir`, `copy`, `robocopy`...) |
| `smbcontrol` | `Set-SmbServerConfiguration`, `Close-SmbSession` (cerrar una sesión), `Close-SmbOpenFile` (cerrar un archivo abierto), `Restart-Service LanmanServer` |
| `smbpasswd` | `Set-LocalUser juan -Password (Read-Host -AsSecureString)` o `net user juan *` |
| `smbstatus` | `Get-SmbSession` (quién está conectado), `Get-SmbOpenFile` (qué archivos tiene abiertos); también `net session` y `net file` |
| `smbtar` | Copias de seguridad de Windows Server o `robocopy` (Capítulo 2) |
| `testparm` | `Get-SmbServerConfiguration` y `Get-SmbShare` (ver la configuración) |
| `wbinfo -u` / `-g` | `Get-ADUser` / `Get-ADGroup`; `nltest /dsgetdc:empresa.local` (controlador de dominio que se usa) y `nltest /sc_verify:empresa.local` (comprobar la relación con el dominio) |
| `samba-tool` (Samba como controlador de AD) | Rol **Servicios de dominio de Active Directory** (`Install-WindowsFeature AD-Domain-Services` + `Install-ADDSForest`) y sus herramientas |

### 1.8 Montar un recurso compartido en un cliente Windows

| Linux | Windows Server 2025 |
|---|---|
| Entrada `cifs` en `/etc/fstab` | `net use Z: \\192.168.56.102\ssharea /persistent:yes` o `New-SmbMapping -LocalPath Z: -RemotePath \\192.168.56.102\ssharea -Persistent $true` |
| Archivo de credenciales (`credentials=`) | **Administrador de credenciales** de Windows: `cmdkey /add:192.168.56.102 /user:usuario /pass` (pide la contraseña y la guarda protegida). Ver: `cmdkey /list` |
| `uid=`, `noperm` | No aplica: se usan los permisos de la cuenta con la que se conecta |
| `iocharset=utf-8` | No hace falta: SMB usa Unicode siempre |
| Ver conexiones activas | `Get-SmbConnection` (servidor, recurso, versión de SMB) o `net use` |

### 1.9 Puertos de SMB (Tabla 10.7)

| Puerto | Uso en Windows Server 2025 |
|---|---|
| TCP **445** | SMB directo sobre TCP. **Es el único necesario** en redes modernas |
| TCP/UDP 137, 138, 139 | NetBIOS: solo para equipos antiguos. Se puede desactivar NetBIOS sobre TCP/IP en cada tarjeta |
| UDP **443** | **SMB sobre QUIC** (SMB cifrado con TLS 1.3, pensado para acceder sin VPN). Disponible en todas las ediciones de Windows Server 2025 |
| 88 / 464, 389 / 636 | Kerberos y LDAP, cuando el servidor está en un dominio (se usan para la autenticación, no para SMB en sí) |

Comprobar que el servidor escucha (equivalente a `ss -utlpn`): `Get-NetTCPConnection -LocalPort 445 -State Listen`.

### 1.10 Solución de problemas

| Linux (Samba) | Windows Server 2025 |
|---|---|
| `log level` en `smb.conf` | Registros `Microsoft-Windows-SMBServer/*` y `SMBClient/*` en el Visor de eventos |
| `smbclient -L //servidor -U usuario` | `net view \\SRV01 /all` / `Get-SmbShare -CimSession SRV01` |
| `ping`, `traceroute` | `ping`, `tracert`, `Test-NetConnection SRV01 -Port 445` |
| Comprobar que los demonios están activos | `Get-Service LanmanServer, LanmanWorkstation` |
| `testparm` | `Get-SmbServerConfiguration` |
| `pdbedit -L`, `wbinfo -u` | `Get-LocalUser`, `Get-ADUser` |
| `nmblookup -A IP` | `nbtstat -A IP` |

**Cambios de seguridad de SMB en Windows Server 2025** (causa frecuente de problemas con equipos o NAS antiguos):

| Cambio | Detalle |
|---|---|
| **Firma SMB obligatoria** | Windows Server 2025 exige que estén firmadas todas sus conexiones SMB **salientes** (cuando actúa como cliente). Un NAS antiguo sin firma dará error `STATUS_INVALID_SIGNATURE`. Como servidor, se puede exigir también: `Set-SmbServerConfiguration -RequireSecuritySignature $true` |
| **Acceso de invitado** | Desactivado; además, la firma obligatoria impide las conexiones de invitado |
| **Limitador de autenticación** | Añade un retraso tras cada intento fallido de contraseña, para frenar ataques de fuerza bruta: `Set-SmbServerConfiguration -InvalidAuthenticationDelayTimeInMs 2000` |
| **Bloqueo de NTLM** en el cliente SMB | Opcional: `Set-SmbClientConfiguration -BlockNTLM $true` (solo Kerberos) |
| Versión mínima y máxima de SMB | Nuevas directivas de grupo para permitir solo, por ejemplo, SMB 3.x |
| Cifrado | Por recurso (`-EncryptData $true`) o en todo el servidor (`Set-SmbServerConfiguration -EncryptData $true`) |

---

## 2. NFS

### 2.0 Activar el servidor y el cliente

| Elemento | Windows Server 2025 |
|---|---|
| Servidor NFS | `Install-WindowsFeature FS-NFS-Service -IncludeManagementTools` (servicio de rol **Servidor para NFS**, servicio `NfsService`). Admite NFSv2, NFSv3 y NFSv4.1 |
| Cliente NFS | `Install-WindowsFeature NFS-Client` (**Cliente para NFS**). Admite **solo NFSv2 y NFSv3** |
| Configuración del servidor | `Get-NfsServerConfiguration` / `Set-NfsServerConfiguration` (por ejemplo, `-EnableNFSV4 $true`) |
| Reiniciar el servidor | `Restart-Service NfsService` o `nfsadmin server stop` / `nfsadmin server start` |

**Asignación de identidades** (sin equivalente en el libro, pero imprescindible): NFS identifica a los usuarios por **UID y GID** de Unix, y Windows por cuentas. El Servidor para NFS tiene que saber qué UID corresponde a qué cuenta de Windows. Opciones:
- **Active Directory**: los atributos `uidNumber` y `gidNumber` de cada usuario y grupo (la opción recomendada en un dominio). Se activa con `Set-NfsMappingStore -EnableADLookup $true -ADDomainName empresa.local`.
- Archivos **`passwd`** y **`group`** con formato Unix en `C:\Windows\System32\drivers\etc\`, para servidores sin dominio.

Los usuarios sin correspondencia se tratan como **anónimos**.

### 2.1 El equivalente de `/etc/exports`

**No hay archivo `/etc/exports`.** Cada exportación se crea con cmdlets (o desde el Administrador del servidor → Recursos compartidos → "Recurso compartido NFS"):
```
New-NfsShare -Name nfs_share_perm -Path D:\nfs_share_perm -Permission no-access
Grant-NfsSharePermission -Name nfs_share_perm -ClientName 192.168.56.101 -ClientType host -Permission readwrite
Grant-NfsSharePermission -Name nfs_share_perm -ClientName 192.168.56.0/24 -ClientType host -Permission readonly
```
(Equivale a las dos líneas de ejemplo del libro: lectura y escritura para un equipo, solo lectura para la red. `-Permission no-access` deja cerrado el acceso por defecto al resto.)

**Opciones de exportación:**

| Opción de Linux | Equivalente en Windows Server 2025 |
|---|---|
| `rw` / `ro` | `-Permission readwrite` / `readonly` (o `no-access`) |
| Cliente (`192.168.56.101`, `192.168.56.*`) | `-ClientName` con `-ClientType host`, `netgroup` o `clientgroup` (grupos de clientes definidos en el servidor: `New-NfsClientgroup`) |
| `root_squash` / `no_root_squash` | `-AllowRootAccess $false` (por defecto) / `$true` en `Grant-NfsSharePermission` |
| `all_squash` | Sin equivalente directo: los usuarios que no tienen correspondencia con una cuenta de Windows ya se tratan como anónimos |
| `anonuid` / `anongid` | `New-NfsShare ... -EnableAnonymousAccess $true -AnonymousUid 65534 -AnonymousGid 65534` |
| `sync` / `async` | No aplica: lo gestiona Windows |
| `fsid`, `subtree_check` | No aplican |
| `sec=krb5` (no aparece en el libro) | `-Authentication krb5,krb5i,krb5p,sys` |

### 2.2 Utilidades NFS (equivalente a la Tabla 10.15)

| Utilidad de Linux | Equivalente en Windows Server 2025 |
|---|---|
| `exportfs` (ver exportaciones) | `Get-NfsShare` y `Get-NfsSharePermission -Name nfs_share_perm` |
| `exportfs -r` (recargar) | No hace falta: los cambios se aplican al momento |
| `exportfs -u` (dejar de exportar) | `Remove-NfsShare -Name nfs_share_perm` |
| `exportfs -o` (exportar por línea de comandos) | `New-NfsShare` |
| `mount.nfs` / `umount.nfs` | `mount` / `umount` del Cliente para NFS (apartado 2.3) |
| `nfsstat` | `Get-NfsStatistics` (servidor) |
| `mountstats` / `nfsiostat` | Contadores de rendimiento del servidor NFS (`Get-Counter -ListSet "*NFS*"`) |
| `showmount -a` (clientes conectados) | `Get-NfsMountedClient` y `Get-NfsSession` |
| `showmount -e servidor` | `showmount -e servidor` (comando incluido con el Cliente para NFS) |
| `rpcinfo -p` | No incluido; se comprueba el puerto con `Test-NetConnection servidor -Port 2049` |
| — | `Get-NfsOpenFile` (archivos abiertos por clientes NFS) |

Herramienta clásica de texto (equivalente a trabajar con `exportfs`): `nfsshare` (por ejemplo, `nfsshare nfs1=D:\nfs1 -o rw=192.168.56.101`).

### 2.3 Opciones de montaje (equivalente a la Tabla 10.17)

Con el Cliente para NFS, el comando `mount` de Windows monta una exportación en una letra de unidad:
```
mount -o anon,nolock,mtype=hard \\192.168.56.102\srv\nfs_share_perm Z:
```
La ruta de la exportación se escribe con barras invertidas: `servidor:/srv/nfs` pasa a ser `\\servidor\srv\nfs`.

| Opción de Linux | Equivalente en el cliente NFS de Windows |
|---|---|
| `intr` / `hard` / `soft` | `mtype=hard` o `mtype=soft` |
| `nfsvers=2\|3\|4` | Solo NFSv2 y NFSv3 (el cliente de Windows no admite NFSv4) |
| `tcp` / `udp` | Por defecto usa TCP y UDP según el servidor |
| `nolock` | `nolock` (igual) |
| `timeo`, `retrans` | `timeout=segundos`, `retry=número` |
| `rsize`, `wsize` | `rsize=KB`, `wsize=KB` |
| Montar como anónimo | `anon` |
| Sensibilidad a mayúsculas | `casesensitive=yes` (Windows no distingue mayúsculas por defecto; Unix sí) |
| `noexec`, `nosuid` | No aplican |

| Comando | Función |
|---|---|
| `mount` (sin parámetros) | Lista las exportaciones NFS montadas |
| `umount Z:` | Desmonta |

No hay `/etc/fstab`: para montar una exportación en cada inicio se usa un script de inicio de sesión o una tarea programada (Capítulo 4).

### 2.4 Comprobación y solución de problemas

| Linux | Windows Server 2025 |
|---|---|
| `rpcinfo -p` | `Get-Service NfsService` y `Test-NetConnection servidor -Port 2049` |
| `exportfs -v` / `showmount -e` | `Get-NfsShare` / `showmount -e servidor` |
| `showmount -a` | `Get-NfsMountedClient` |
| `/var/log/messages` | Visor de eventos → Registros de aplicaciones y servicios → **Microsoft → Windows → ServicesForNFS-Server** |
| Problemas de permisos (el cliente ve archivos de "nobody") | Revisar la asignación de identidades (`Get-NfsMappingStore`, atributos `uidNumber` / `gidNumber` en AD) |

---

## 3. FTP

El servidor FTP de Windows Server es un servicio de rol de **IIS**. Se configura igual que un sitio web (Capítulo 9) y comparte con él el archivo `applicationHost.config`.

### 3.1 El equivalente de vsftpd: Servidor FTP de IIS

| Elemento de vsftpd | Equivalente en Windows Server 2025 |
|---|---|
| Paquete `vsftpd` | `Install-WindowsFeature Web-Ftp-Server -IncludeManagementTools` |
| Servicio | **Servicio FTP de Microsoft** (`FTPSVC`) |
| `/etc/vsftpd.conf` | `C:\Windows\System32\inetsrv\config\applicationHost.config` (secciones `ftpServer` y `system.ftpServer`), gestionado con el Administrador de IIS o PowerShell |
| `/var/ftp/` o `/srv/ftp/` | `C:\inetpub\ftproot` |
| Registros | `C:\inetpub\logs\LogFiles\FTPSVC<n>\` |
| Reiniciar | `Restart-Service FTPSVC` |

**Crear un sitio FTP:**
```
New-WebFtpSite -Name "FTP" -Port 21 -PhysicalPath "C:\inetpub\ftproot"
```

**Directivas de vsftpd (Tabla 10.18) y su equivalente:**

| Directiva de vsftpd | Equivalente en el Servidor FTP de IIS |
|---|---|
| `anonymous_enable` | Autenticación anónima: `Set-ItemProperty "IIS:\Sites\FTP" -Name ftpServer.security.authentication.anonymousAuthentication.enabled -Value $true` |
| `local_enable` | Autenticación básica con cuentas de Windows: `... -Name ftpServer.security.authentication.basicAuthentication.enabled -Value $true` |
| `write_enable` | Regla de autorización con permiso `Write` + permiso NTFS de escritura en la carpeta |
| `anon_upload_enable` | Regla de autorización para usuarios anónimos (`users="?"`) con `Read,Write` + escritura NTFS para `IUSR` |
| `anon_mkdir_write_enable` / `anon_other_write_enable` | No hay opciones separadas en FTP; se controla con **permisos NTFS** detallados de `IUSR` (crear carpetas, eliminar...) |
| `anon_root` | Con aislamiento de usuarios: carpeta `LocalUser\Public` dentro de la raíz del sitio |
| `anon_world_readable_only` | Permisos NTFS de lectura de `IUSR` |
| `chown_uploads` / `chown_username` | No aplica (propietario y permisos según la herencia NTFS) |
| `chroot_local_user` | **Aislamiento de usuarios**: `Set-ItemProperty "IIS:\Sites\FTP" -Name ftpServer.userIsolation.mode -Value IsolateAllDirectories`. Cada usuario queda encerrado en `LocalUser\<usuario>` (o `<DOMINIO>\<usuario>`) |
| `chroot_list_enable` / `chroot_list_file` | No hay lista: el aislamiento se aplica a todo el sitio |
| `ftp_username` | Cuenta anónima: `IUSR` por defecto (`anonymousAuthentication.userName`) |
| `listen` / `listen_ipv6` | Enlaces del sitio FTP (IP, puerto) |
| `userlist_enable` / `userlist_file` | Reglas de autorización de tipo **Denegar** para usuarios o grupos |
| `log_ftp_protocol` | Registro del sitio FTP (campos configurables en `ftpServer.logFile`) |
| TCP Wrappers | **Restricciones de dirección IP y dominio de FTP** (sección `system.ftpServer/security/ipSecurity`) |

**Reglas de autorización** (quién puede leer o escribir):
```
Add-WebConfiguration "/system.ftpServer/security/authorization" -PSPath IIS:\ -Location "FTP" -Value @{accessType="Allow"; users="*"; permissions="Read,Write"}
```
(`users="*"` = todos los usuarios autenticados; `users="?"` = anónimos; también `roles="Grupo"`.)

**Configuraciones típicas del libro:**

| Configuración | Ajustes en IIS |
|---|---|
| Solo usuarios con contraseña (`anonymous_enable=NO`, `local_enable=YES`) | Autenticación básica activada, anónima desactivada, regla `Allow` para los usuarios o grupos |
| Solo anónimo de descarga (`anonymous_enable=YES`, `local_enable=NO`) | Autenticación anónima activada, básica desactivada, regla `Allow` con `users="?"` y `permissions="Read"` |
| Subidas anónimas | Regla con `users="?"` y `permissions="Read,Write"` + permiso NTFS de escritura para `IUSR` |

**Cuentas FTP sin acceso interactivo** (equivalente a darles `/usr/sbin/nologin`): se crean cuentas locales normales y se les **deniega el inicio de sesión local y por Escritorio remoto** con las directivas de seguridad (`secpol.msc` → Asignación de derechos de usuario → "Denegar inicio de sesión local" y "Denegar inicio de sesión a través de Servicios de Escritorio remoto").

**FTP cifrado y firewall:**

| Tarea | Comando / ajuste |
|---|---|
| FTPS (FTP sobre TLS) | `Set-ItemProperty "IIS:\Sites\FTP" -Name ftpServer.security.ssl.serverCertHash -Value HUELLA` y la directiva de SSL: `ftpServer.security.ssl.controlChannelPolicy` y `dataChannelPolicy` (permitir o exigir) |
| Puertos del modo pasivo | `Set-WebConfigurationProperty -PSPath IIS:\ -Filter system.ftpServer/firewallSupport -Name lowDataChannelPort -Value 50000` (y `highDataChannelPort`), y abrir ese rango en el firewall |

### 3.2 Pure-FTPd

**No existe para Windows.** Sus opciones (Tabla 10.19) se corresponden con ajustes del Servidor FTP de IIS:

| Opción de Pure-FTPd | Equivalente en IIS |
|---|---|
| `-4` / `-6` | Enlaces del sitio con una IP v4 o v6 concreta |
| `-A` (chroot para todos) | Aislamiento de usuarios (`IsolateAllDirectories`) |
| `-a gid` (grupo sin chroot) | Sin equivalente directo |
| `-B` (segundo plano) | Siempre funciona como servicio de Windows |

### 3.3 Otras opciones de FTP

| Programa | Notas |
|---|---|
| **SFTP** con OpenSSH | Incluido en Windows Server 2025 (servidor OpenSSH, Capítulo 2). **Opción recomendada**: cifrado y sin los problemas de puertos de FTP. Para encerrar a los usuarios en una carpeta: `ChrootDirectory` y `ForceCommand internal-sftp` en `C:\ProgramData\ssh\sshd_config` |
| FileZilla Server | Servidor FTP/FTPS de terceros, muy usado en Windows |

---

## 4. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 10) | Equivalente en Windows Server 2025 |
|---|---|
| Paquete `samba` | Integrado (servicio `LanmanServer`, servicio de rol Servidor de archivos) |
| `smb.conf` `[global]` | `Set-SmbServerConfiguration` |
| `smb.conf` `[recurso]` | `New-SmbShare` / `net share` |
| `/var/log/samba/` | Registros `Microsoft-Windows-SMBServer/*` |
| `tdbsam` / `ldapsam` | Cuentas locales (SAM) / Active Directory |
| `smbd` / `nmbd` / `winbindd` | `LanmanServer` / NetBIOS sobre TCP/IP / unión al dominio (Netlogon) |
| `valid users`, `writable`, `write list` | `-FullAccess`, `-ChangeAccess`, `-ReadAccess`, `-NoAccess` (+ permisos NTFS) |
| `browseable = no` | Nombre del recurso terminado en `$` |
| `[printers]` | Rol Servidor de impresión (`Set-Printer -Shared`) |
| `testparm` | No hace falta (`Get-SmbServerConfiguration`) |
| `smbpasswd -a` / `pdbedit` | No hace falta / `Get-LocalUser`, `Get-ADUser` |
| `smbstatus` | `Get-SmbSession`, `Get-SmbOpenFile` |
| `smbcacls` | `icacls` + `Grant-SmbShareAccess` |
| `smbclient -L` | `net view \\servidor /all` |
| `nmblookup` | `nbtstat` |
| `mount.cifs` + `/etc/fstab` | `net use /persistent:yes` o `New-SmbMapping -Persistent` |
| Archivo `credentials=` | `cmdkey` (Administrador de credenciales) |
| `samba-tool` | Rol AD DS |
| NFS servidor (`nfs-kernel-server` / `nfs-utils`) | Servidor para NFS (`FS-NFS-Service`) |
| `/etc/exports` | `New-NfsShare` + `Grant-NfsSharePermission` |
| `root_squash` / `no_root_squash` | `-AllowRootAccess $false` / `$true` |
| `anonuid` / `anongid` | `-AnonymousUid` / `-AnonymousGid` |
| `exportfs -r` | No hace falta |
| `showmount -a` | `Get-NfsMountedClient` |
| `nfsstat` | `Get-NfsStatistics` |
| UID/GID de Unix | Asignación de identidades (atributos de AD o archivos `passwd`/`group`) |
| `mount -t nfs` | `mount -o ... \\servidor\export Z:` (Cliente para NFS, solo NFSv2/v3) |
| vsftpd | Servidor FTP de IIS (`Web-Ftp-Server`, servicio `FTPSVC`) |
| `/etc/vsftpd.conf` | `applicationHost.config` (secciones `ftpServer`) |
| `anonymous_enable` / `local_enable` | Autenticación anónima / básica |
| `chroot_local_user` | Aislamiento de usuarios de FTP |
| `/sbin/nologin` | Denegar el inicio de sesión local y por Escritorio remoto |
| TCP Wrappers en vsftpd | Restricciones de IP de FTP |
| SFTP | OpenSSH incluido (`sshd_config` en `C:\ProgramData\ssh\`) |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 10 ("Sharing Files") del libro LPIC-2. Fuentes: documentación oficial de Microsoft Learn (SMB en Windows Server, módulo SmbShare de PowerShell, endurecimiento de la seguridad de SMB y firma SMB en Windows Server 2025, SMB sobre QUIC, Servidor para NFS y Cliente para NFS, módulo NFS de PowerShell, Servidor FTP de IIS y OpenSSH para Windows).*
