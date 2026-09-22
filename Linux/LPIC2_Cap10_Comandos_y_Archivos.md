# LPIC-2 · Capítulo 10: Sharing Files
### Recopilación de comandos y archivos de configuración

Objetivos del examen cubiertos: 209.1 (Samba Server Configuration), 209.2 (NFS Server Configuration), 212.2 (Managing FTP Servers)

---

## 1. Samba

Samba es una implementación de código abierto del protocolo **SMB** (Service Message Block), también conocido como **CIFS**. Permite compartir ficheros e impresoras entre sistemas Linux y Windows.

### 1.1 Ubicación de ficheros y directorios

| Elemento | Ubicación habitual |
|---|---|
| Fichero principal de configuración | `/etc/samba/smb.conf` (versiones antiguas: `/etc/smb.conf`, `/etc/smb/`, `/etc/samba.d/`) |
| Logs | `/var/log/samba/` |
| Bases de datos de usuarios | `/var/lib/samba/` (`smbpasswd`, `tdbsam`, `ldapsam`) |

Recomendación de base de usuarios según tamaño: `tdbsam` para menos de 250 usuarios; `ldapsam` (con AD/LDAP) para entornos grandes; `smbpasswd` en desuso, solo por compatibilidad.

### 1.2 Daemons de Samba

| Daemon | Función |
|---|---|
| `smbd` | Siempre necesario: gestiona los recursos compartidos SMB, bloqueo de ficheros, autenticación de usuarios |
| `nmbd` | Gestiona peticiones NetBIOS (solo necesario con clientes Windows antiguos a Windows 2000) |
| `winbindd` | Conecta un sistema Linux con un controlador de dominio Windows; traduce usuarios/grupos de Active Directory a usuarios Linux; también actúa como WINS para sistemas NetBIOS antiguos |

### 1.3 Estructura del fichero `smb.conf`

- Comentarios tras `;` o `#`; líneas en blanco ignoradas.
- Dividido en secciones entre corchetes `[nombre]` (no distinguen mayúsculas/minúsculas).
- Las directivas tampoco distinguen mayúsculas/minúsculas; los espacios junto al `=` se ignoran; una línea se puede continuar con `\` al final.

**Secciones especiales:**

| Sección | Función |
|---|---|
| `[global]` | Directivas generales: red, logging, resolución de nombres, seguridad, etc. |
| `[nombre_share]` | Un recurso compartido concreto (carpeta o impresora) |
| `[netlogon]` | Necesaria cuando Samba actúa como controlador de dominio |
| `[printers]` | Comparte automáticamente todas las colas de impresión sin definir cada una individualmente |
| `[profiles]` | Perfiles de usuario itinerantes (roaming profiles) |

### 1.4 Directivas globales destacadas (Tabla 10.4)

| Directiva | Función |
|---|---|
| `workgroup` | Grupo de trabajo/dominio Windows al que pertenece (no es un FQDN) |
| `server string` | Descripción del servidor (admite variables como `%v` para la versión) |
| `interfaces` | Interfaces de red a usar |
| `hosts allow` | Redes/IPs con acceso permitido |
| `log file` | Ruta de log (admite `%m` por nombre de máquina cliente) |
| `max log size` | Tamaño máximo del log en KB |
| `security` | Modo de seguridad: `user` (autenticación por usuario) o `share` |
| `passdb backend` | Base de datos de usuarios a usar (`tdbsam`, `ldapsam`, `smbpasswd`) |
| `log level` | Nivel de depuración (0 = sin logging, por defecto) |

### 1.5 Directivas de un recurso compartido (Tabla 10.6)

| Directiva | Función |
|---|---|
| `comment` | Descripción visible al listar recursos |
| `browseable` (o `browsable`) | Si aparece listado como recurso disponible (por defecto `yes`) |
| `valid users` / `invalid users` | Lista de usuarios/grupos (`@grupo`) permitidos/excluidos |
| `path` | Ruta absoluta del recurso |
| `public` (sinónimo `guest ok`) | Si se requiere contraseña (`no` por defecto = sí se requiere) |
| `guest only` (sinónimo `only guest`) | Solo permite conexiones de invitado |
| `group` (sinónimo `force group`) | Grupo primario asignado a los usuarios que se conectan |
| `writable` (antónimo `read only`) | Permite escritura (`no` por defecto) |
| `write list` | Usuarios/grupos con acceso de escritura garantizado |

**Sección `[printers]` típica:**
```
[printers]
    comment = All Printers
    path = /var/spool/samba
    browseable = no
    guest ok = no
    writable = no
    printable = yes
```

### 1.6 Pasos para configurar un recurso compartido

1. Crear el directorio a compartir (recomendado en `/srv/`)
2. Modificar `smb.conf`
3. Comprobar la sintaxis: `testparm`
4. Crear/dar contraseña al usuario local necesario
5. Añadir el usuario a la base de datos de Samba (`smbpasswd -a usuario` o `pdbedit`)
6. Comprobar la cuenta Samba
7. Arrancar el demonio (`systemctl start smb`)
8. Comprobar los recursos ofrecidos
9. Activar el arranque automático
10. Ajustar el firewall si aplica

### 1.7 Utilidades de administración (Tabla 10.3)

| Utilidad | Función |
|---|---|
| `mount.cifs` | Monta un recurso Samba en el cliente |
| `net` | Administra el servidor Samba y servidores remotos (similar a `net` de Windows/DOS) |
| `nmblookup` | Resuelve información NetBIOS (nombres, IPs, grupo de trabajo) |
| `pdbedit` | Gestiona cualquiera de las bases de usuarios (`-L` lista, `-v` detalle, `-u usuario`) |
| `rpcclient` | Ejecuta funciones RPC de Microsoft |
| `smbcacls` | Muestra/modifica las ACLs de un recurso |
| `smbclient` | Conecta, lista recursos y permite acceso estilo FTP a un recurso compartido |
| `smbcontrol` | Gestiona el demonio `smbd` |
| `smbpasswd` | Gestiona la base `smbpasswd`/`tdbsam` |
| `smbstatus` | Muestra las conexiones activas al servidor |
| `smbtar` | Backup/restauración de un recurso a fichero o cinta |
| `testparm` | Comprueba la sintaxis de `smb.conf` |
| `wbinfo` | Información del demonio `winbindd` (`-u` usuarios, `-g` grupos) |
| `samba-tool` | Herramienta principal cuando Samba actúa como controlador de Active Directory |

Ejemplo de comprobación de recursos ofrecidos:
```
smbclient -L //localhost -U usuario
```

### 1.8 Montar un recurso Samba en un cliente Linux

Vía `/etc/fstab`:
```
//192.168.56.102/ssharea /home/usuario/cshare cifs credentials=/etc/samba/credfile,noperm,uid=1004 0 0
```
El fichero de credenciales (`credentials=`) contiene `username=...` y `password=...`, debe pertenecer a root y tener permisos restringidos (solo lectura del propietario). Añadir `iocharset=utf-8` ayuda con problemas de nombres de fichero en entornos mixtos.

### 1.9 Puertos de Samba (Tabla 10.7, selección)

| Puerto | Protocolo(s) | Uso |
|---|---|---|
| 137, 138, 139 | TCP/UDP | NetBIOS (nombre, datagrama, sesión) |
| 389 / 636 | TCP/UDP / TCP | LDAP / LDAP sobre SSL |
| 445 | TCP | SMB sobre TCP (moderno, sin NetBIOS) |
| 88 / 464 | TCP/UDP | Kerberos / Kerberos kpasswd |

Comprobar qué puertos usa realmente el demonio: `ss -utlpn | grep PID_smbd`.

### 1.10 Solución de problemas

- Activar logging: `log level` entre 1 y 10 en `smb.conf`.
- Ver recursos ofrecidos: `smbclient -L //servidor -U usuario`.
- Diagnóstico de red básico: `ping`, `traceroute`.
- Comprobar que los demonios necesarios están activos.
- Comprobar sintaxis: `testparm`.
- Revisar cuentas: `pdbedit -L`, `wbinfo -u`/`-g`.
- Resolución NetBIOS: `nmblookup -A IP`, `nmblookup -S hostname`.

---

## 2. NFS (Network File System)

### 2.1 El fichero `/etc/exports`

Define los recursos exportados por el servidor NFS, uno por línea:
```
/srv/nfs_share_perm 192.168.56.101(rw,...)
/srv/nfs_share_perm 192.168.56.*(ro,...)
```
Importante: **no debe haber espacio** entre el cliente/IP y la lista de opciones entre paréntesis (un espacio de más concede acceso no deseado a todos).

**Opciones destacadas de exportación (selección):**

| Opción | Función |
|---|---|
| `rw` / `ro` | Lectura/escritura o solo lectura |
| `sync` | Espera a que el caché de escritura se vuelque a disco antes de leer/escribir (recomendado para `rw`) |
| `async` | Mejora el rendimiento en `ro`, pero con riesgo de corrupción si el servidor falla |
| `all_squash` | Trata a todos los clientes (incluido root) como usuario anónimo |
| `root_squash` | Mapea el root del cliente a un usuario sin privilegios (`nfsnobody`) |
| `no_root_squash` | Permite que el root del cliente tenga privilegios de superusuario en el export |
| `fsid` | Identifica el export por UUID o número; en NFSv4, `root` o `0` marca la raíz de todos los exports |
| `anonuid` / `anongid` | UID/GID asignado a los usuarios anónimos |
| `subtree_check` / `no_subtree_check` | Activa/desactiva la comprobación de permisos en directorios superiores |

### 2.2 Utilidades NFS (Tabla 10.15)

| Utilidad | Función |
|---|---|
| `exportfs` | Gestiona y muestra los recursos exportados; lee `/etc/exports` al arrancar el servicio |
| `mount.nfs` / `umount.nfs` | Monta/desmonta un export en el cliente |
| `mountstats` | Estadísticas por punto de montaje (desde `/proc/self/mountstats`) — solo acepta rutas absolutas |
| `nfsiostat` | Estadísticas de I/O por punto de montaje |
| `nfsstat` | Estadísticas de actividad cliente/servidor |
| `rpcinfo` | Información de los servicios RPC (puertos, programas) |
| `showmount` | Muestra estado del servidor NFS y clientes conectados; funciona en remoto (no vale con NFSv4 para `-e`) |

**Opciones de `exportfs` (Tabla 10.16):**

| Opción | Función |
|---|---|
| `-a` | Exporta (o `-u` desexporta) todos los recursos de `/etc/exports` |
| `-i` | Ignora `/etc/exports`, usa opciones de línea de comandos |
| `-o` | Exporta un recurso indicado por línea de comandos |
| `-r` | Re-exporta (recarga) todos los recursos de `/etc/exports` |
| `-u` | Desexporta un recurso concreto |
| `-v` | Modo detallado |

Recargar tras modificar `/etc/exports`: `exportfs -r` (algunos administradores usan `exportfs -ra`).

### 2.3 Opciones de montaje en `/etc/fstab` (Tabla 10.17)

| Opción | Función |
|---|---|
| `intr` | Permite interrumpir peticiones si el servidor NFS cae |
| `nfsvers=2\|3\|4` | Versión de NFS a usar (NFSv2 en total desuso; NFSv3 desaconsejado por seguridad) |
| `tcp` | Monta usando TCP (recomendado para `rw`) |
| `udp` | Monta usando UDP (mejora rendimiento en `ro`) |
| `port=número` | Puerto del servidor (`0` fuerza que `rpcbind` indique el puerto) |
| `noacl` / `nolock` | Desactiva ACLs / bloqueo de ficheros (distros/versiones antiguas) |
| `noexec` | Desactiva ejecución de binarios (útil si los binarios no son compatibles entre servidor y cliente) |
| `nosuid` | Desactiva SUID/SGID en el recurso montado |

Ejemplo de entrada en `/etc/fstab`:
```
192.168.56.102:/srv/nfs_share_perm /home/usuario/NFSPerm nfs intr,nfsvers=3,tcp 0 0
```

Nota: para montajes automáticos con mejor rendimiento, se recomienda AutoFS (ver Capítulo 4) en vez de `/etc/fstab` directamente.

### 2.4 Comprobación y solución de problemas

| Comando | Función |
|---|---|
| `rpcinfo -p` | Comprueba que `portmapper`, `mountd` y `nfs` están escuchando |
| `exportfs -v` / `showmount -e` | Ver los recursos ofrecidos (usar `exportfs -v` con NFSv4) |
| `showmount -a` | Ver qué clientes tienen montado qué |
| `ping` / `traceroute` / `nmap` | Diagnóstico de red básico |
| `/var/log/messages` | Log donde el servidor NFS registra sus mensajes |

Mensajes de error típicos: `RPC: Program Not Registered` (falta cargar `/etc/exports` → `exportfs -ra`), `no route to host` (servidor caído o firewall), `connection refused` (el servicio `rpcbind` no está activo).

---

## 3. FTP

### 3.1 vsftpd (Very Secure FTP Daemon)

| Elemento | Detalle |
|---|---|
| Paquete | `vsftpd` (mismo nombre en la mayoría de distribuciones) |
| Fichero de configuración | `/etc/vsftpd.conf` o `/etc/vsftpd/vsftpd.conf`, según distribución |
| Directorio por defecto de contenido | `/var/ftp/` o `/srv/ftp/`, según distribución |

**Directivas destacadas (Tabla 10.18):**

| Directiva | Función |
|---|---|
| `anonymous_enable` | Activa/desactiva el acceso anónimo (`YES` por defecto) |
| `local_enable` | Permite el login con cuentas locales del sistema (`/etc/passwd`/`/etc/shadow`) |
| `write_enable` | Permite operaciones de escritura (subir/borrar ficheros) |
| `anon_upload_enable` | Permite subidas anónimas (requiere `write_enable=YES` y permisos de escritura) |
| `anon_mkdir_write_enable` | Permite crear directorios de forma anónima |
| `anon_other_write_enable` | Permite borrar/renombrar ficheros de forma anónima |
| `anon_root` | Directorio al que entra el usuario anónimo tras conectar |
| `anon_world_readable_only` | Solo permite ver/descargar ficheros legibles por todos (`YES` por defecto) |
| `chown_uploads` / `chown_username` | Cambia el propietario de los ficheros subidos anónimamente |
| `chroot_local_user` | Enjaula (chroot) a los usuarios locales en su directorio HOME |
| `chroot_list_enable` / `chroot_list_file` | Lista de usuarios a incluir/excluir del chroot |
| `ftp_username` | Cuenta del sistema usada para el usuario anónimo (por defecto `ftp`) |
| `listen` / `listen_ipv6` | Modo standalone (`YES`) o gestionado por `xinetd` (`NO`), para IPv4/IPv6 |
| `userlist_enable` / `userlist_file` | Lista de usuarios a los que se deniega el acceso antes de pedir contraseña |
| `log_ftp_protocol` | Registra todas las peticiones y respuestas FTP |

**Configuración típica con usuario/contraseña:**
```
anonymous_enable=NO
local_enable=YES
```
El usuario FTP debe existir en el sistema; conviene asignarle un shell sin acceso interactivo (`/usr/sbin/nologin` o `/sbin/nologin`), y ese shell debe figurar en `/etc/shells` si vsftpd usa autenticación PAM (`pam_service_name`).

**Configuración típica de acceso anónimo (solo descarga):**
```
anonymous_enable=YES
local_enable=NO
```

**Habilitar subidas anónimas:**
```
write_enable=YES
anon_upload_enable=YES
anon_mkdir_write_enable=YES
```

Reinicio tras cambios: `service vsftpd restart` (o `systemctl restart vsftpd`).

**Control de acceso con TCP Wrappers:** vsftpd soporta `libwrap` (comprobar con `ldd /usr/sbin/vsftpd | grep libwrap`), funciona igual que se describió en el Capítulo 6, sustituyendo el nombre de servicio por `vsftpd` en `/etc/hosts.allow`/`/etc/hosts.deny`.

### 3.2 Pure-FTPd (alternativa)

| Distribución | Instalación |
|---|---|
| Debian/Ubuntu | `sudo apt-get install pure-ftpd` (repositorio estándar) |
| Red Hat/CentOS | Requiere el repositorio **EPEL**: descargar e instalar el paquete `epel-release*.rpm` con `rpm -ivh`, y luego `yum --enablerepo=epel install pure-ftpd` |

No usa un único fichero de configuración como vsftpd: se controla principalmente mediante **opciones de línea de comandos** al arrancar el demonio `pure-ftpd` (consultar con `pure-ftpd --help`).

**Opciones destacadas (Tabla 10.19):**

| Opción corta | Opción larga | Función |
|---|---|---|
| `-4` | `--ipv4only` | Solo IPv4 |
| `-6` | `--ipv6only` | Solo IPv6 |
| `-A` | `--chrooteveryone` | Chroot para todos los clientes excepto root |
| `-a gid` | `--trustedgid gid` | No aplica chroot a los clientes de ese GID |
| `-B` | `--daemonize` | Arranca en segundo plano |

---

## 4. Resumen de diferencias Debian vs Red Hat en este capítulo

| Aspecto | Debian | Red Hat |
|---|---|---|
| Ubicación de `vsftpd.conf` | `/etc/vsftpd.conf` (puede variar) | `/etc/vsftpd/vsftpd.conf` (puede variar) |
| Directorio FTP por defecto | `/srv/ftp/` | `/var/ftp/` |
| Instalación de Pure-FTPd | Repositorio estándar (`apt-get install pure-ftpd`) | Requiere repositorio EPEL |
| Arranque de servicios (Samba, NFS, FTP) | `service nombre start` / `systemctl` según versión | `systemctl start nombre` |
| Resto del capítulo (`smb.conf`, utilidades Samba, `/etc/exports`, utilidades NFS, directivas vsftpd) | Igual | Igual |

---

*Documento generado a partir del Capítulo 10 ("Sharing Files") del libro LPIC-2: Linux Professional Institute Certification Study Guide (Bresnahan & Blum).*
