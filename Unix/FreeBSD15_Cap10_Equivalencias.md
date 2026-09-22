# LPIC-2 · Capítulo 10: Sharing Files
### Equivalencias en FreeBSD 15

Este documento recoge, para cada concepto visto en el Capítulo 10 del libro LPIC-2 (Samba, NFS y FTP), cuál es su equivalente en **FreeBSD 15**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo:
- **Samba** es el mismo programa que en Linux (mismas directivas y utilidades), pero se instala como paquete y su archivo se llama **`smb4.conf`**.
- **NFS** viene en el **sistema base** de FreeBSD, pero el archivo **`/etc/exports` tiene una sintaxis distinta** a la de Linux y no existe `exportfs`. Es la parte de este capítulo que más cambia.
- **FTP**: FreeBSD 15 ha **quitado su servidor FTP del sistema base**. vsftpd y Pure-FTPd se instalan como paquetes.

---

## 1. Samba

### 1.1 Instalación y ubicación de ficheros

| Elemento | Linux | FreeBSD 15 |
|---|---|---|
| Paquete | `samba` | `sambaXXX` según la versión (ej. `samba420`). Buscar la más reciente con `pkg search samba4` |
| Fichero principal | `/etc/samba/smb.conf` | **`/usr/local/etc/smb4.conf`** (fíjate en el `4` del nombre) |
| Logs | `/var/log/samba/` | **`/var/log/samba4/`** |
| Bases de datos de usuarios | `/var/lib/samba/` | **`/var/db/samba4/private/`** (ej. `passdb.tdb`) |

El paquete **no crea** `smb4.conf`: hay que escribirlo desde cero (o partir de los ejemplos del libro, que valen sin cambios).

La recomendación del libro sobre la base de usuarios (`tdbsam` para entornos pequeños, `ldapsam` para grandes, `smbpasswd` en desuso) es la misma.

### 1.2 Daemons de Samba

Los mismos: `smbd`, `nmbd` y `winbindd`. En FreeBSD se arrancan juntos con un único servicio:

| Variable de `/etc/rc.conf` | Función |
|---|---|
| `samba_server_enable="YES"` | Activa Samba (arranca `smbd`, `nmbd` y `winbindd` según lo necesario) |
| `nmbd_enable="NO"` | Desactiva solo `nmbd` (si no hay clientes antiguos que usen NetBIOS) |
| `winbindd_enable="YES"` | Activa `winbindd` (para unirse a un dominio Windows) |

| Linux | FreeBSD 15 |
|---|---|
| `systemctl start smb` / `smbd` | `service samba_server start` |
| `systemctl enable smb` | `sysrc samba_server_enable="YES"` |
| `systemctl status smb` | `service samba_server status` |

### 1.3 a 1.5 Estructura de `smb4.conf` y directivas

**Exactamente igual** que `smb.conf` en Linux: secciones `[global]`, `[nombre_share]`, `[netlogon]`, `[printers]`, `[profiles]`; directivas globales de la Tabla 10.4 (`workgroup`, `server string`, `interfaces`, `hosts allow`, `log file`, `max log size`, `security`, `passdb backend`, `log level`) y de recurso de la Tabla 10.6 (`comment`, `browseable`, `valid users`, `path`, `guest ok`, `writable`, `write list`...).

Solo cambian las rutas:
```
[global]
    workgroup = GRUPO
    server string = Samba %v en FreeBSD
    security = user
    passdb backend = tdbsam
    log file = /var/log/samba4/log.%m
    max log size = 1000

[compartido]
    comment = Carpeta compartida
    path = /tank/compartido
    valid users = @personal
    writable = yes
```

Notas de versión (afectan también a Linux):
- `security = share` ya no existe en las versiones actuales de Samba; solo se usa `security = user` (o `ads` / `domain` en dominios).
- Samba 4.11 y posteriores desactivan por defecto el protocolo antiguo **SMB1**.

**Detalle propio de FreeBSD con ZFS:** si el recurso está en un dataset ZFS y se quieren usar los permisos de Windows (ACL), se añade al recurso:
```
vfs objects = zfsacl
nfs4:mode = special
```

**Impresoras** (sección `[printers]`): funciona igual, pero FreeBSD no trae CUPS en el sistema base; se instala con `pkg install cups`, y la carpeta de la cola (ej. `/var/spool/samba`) hay que crearla a mano con permisos `1777`.

### 1.6 Pasos para configurar un recurso compartido

| Paso del libro | FreeBSD 15 |
|---|---|
| 1. Crear el directorio (en `/srv/`) | FreeBSD no tiene `/srv` por defecto. Se puede crear, o mejor usar un dataset ZFS: `zfs create -o mountpoint=/srv/compartido tank/compartido` |
| 2. Modificar la configuración | Editar `/usr/local/etc/smb4.conf` |
| 3. Comprobar la sintaxis | `testparm` (igual) |
| 4. Crear el usuario local | **`pw useradd usuario -m`** + `passwd usuario` (en FreeBSD se usa `pw` o `adduser` en lugar de `useradd`) |
| 5. Añadirlo a Samba | `smbpasswd -a usuario` o `pdbedit -a usuario` (igual) |
| 6. Comprobar la cuenta | `pdbedit -L -v` (igual) |
| 7. Arrancar el demonio | `service samba_server start` |
| 8. Comprobar los recursos | `smbclient -L //localhost -U usuario` (igual) |
| 9. Arranque automático | `sysrc samba_server_enable="YES"` |
| 10. Cortafuegos | Abrir los puertos en `pf` o `ipfw` (Capítulo 12) |

### 1.7 Utilidades de administración (Tabla 10.3)

**Las mismas** (`net`, `nmblookup`, `pdbedit`, `rpcclient`, `smbcacls`, `smbclient`, `smbcontrol`, `smbpasswd`, `smbstatus`, `smbtar`, `testparm`, `wbinfo`, `samba-tool`), instaladas con el paquete en `/usr/local/bin/` y `/usr/local/sbin/`.

La excepción es **`mount.cifs`**, que no existe en FreeBSD (ver apartado 1.8).

### 1.8 Montar un recurso Samba en un cliente FreeBSD

Esta es la parte que más cambia:

| Opción | Detalle |
|---|---|
| `mount_smbfs` (sistema base) | Existe, pero **solo habla SMB1**, que los servidores actuales (Windows y Samba) tienen desactivado. Solo sirve con servidores antiguos o configurados expresamente para aceptar SMB1 |
| `smbclient` (paquete Samba) | Acceso tipo FTP a un recurso, sin montarlo. Funciona con cualquier versión de SMB |
| `fusefs-smbnetfs` (paquete) | Monta recursos SMB2/SMB3 mediante FUSE. Es la opción recomendada para montar recursos de servidores modernos |

Uso de `mount_smbfs` (con servidores que acepten SMB1):
```
mount_smbfs -I 192.168.56.102 //usuario@servidor/ssharea /mnt/cshare
```

Equivalente al fichero de credenciales del libro: **`/etc/nsmb.conf`** (para todo el sistema) o `~/.nsmbrc` (por usuario):
```
[SERVIDOR:USUARIO]
password=clave
```
Igual que en Linux, debe pertenecer a root y tener permisos `600`.

Línea en `/etc/fstab`:
```
//usuario@servidor/ssharea    /mnt/cshare    smbfs    rw,-N    0    0
```
(`-N` indica que no pida contraseña y use la de `nsmb.conf`.)

### 1.9 Puertos de Samba

**Los mismos** (137-139 NetBIOS, 445 SMB, 389/636 LDAP, 88/464 Kerberos).

Comprobar qué puertos usa el demonio (equivalente a `ss -utlpn`):
```
sockstat -4 -l | grep smbd
```

### 1.10 Solución de problemas

Igual que en el libro (`log level`, `smbclient -L`, `testparm`, `pdbedit -L`, `wbinfo -u`/`-g`, `nmblookup -A IP`, `ping`, `traceroute`). Los logs están en `/var/log/samba4/`.

---

## 2. NFS

NFS viene **completo en el sistema base** de FreeBSD (servidor y cliente). No hay que instalar nada.

### 2.0 Activar el servidor y el cliente

**Servidor** (en `/etc/rc.conf`):
```
rpcbind_enable="YES"
nfs_server_enable="YES"
mountd_enable="YES"
```

Para NFSv4, además:
```
nfsv4_server_enable="YES"
nfsuserd_enable="YES"
```

Arrancar: `service nfsd start` (arranca también `rpcbind` y `mountd`).

**Cliente**: `sysrc nfs_client_enable="YES"`. Para bloqueo de ficheros con NFSv3, también `rpc_lockd_enable="YES"` y `rpc_statd_enable="YES"`.

### 2.1 El fichero `/etc/exports`

El archivo tiene el mismo nombre, pero **la sintaxis es completamente distinta** a la de Linux:

| Linux | FreeBSD 15 |
|---|---|
| `ruta cliente(opciones)` | `ruta -opciones cliente` |
| Opciones entre paréntesis, pegadas al cliente | Opciones con guion, **antes** de los clientes |
| Red: `192.168.56.0/24(ro)` | Red: `-network 192.168.56.0/24` |

**Ejemplos equivalentes a los del libro:**

Linux:
```
/srv/nfs_share_perm 192.168.56.101(rw,no_root_squash)
/srv/nfs_share_perm 192.168.56.*(ro)
```

FreeBSD 15:
```
/srv/nfs_share_perm  -maproot=root  192.168.56.101
/srv/nfs_share_perm  -ro  -network 192.168.56.0/24
```

Regla importante de FreeBSD: dentro de un mismo sistema de archivos, **cada cliente solo puede aparecer en una línea**. (El aviso del libro sobre el espacio antes del paréntesis no aplica aquí, porque no hay paréntesis.)

**Equivalencia de opciones:**

| Opción Linux | Opción FreeBSD 15 | Función |
|---|---|---|
| `rw` | Por defecto (no se escribe nada) | Lectura/escritura |
| `ro` | `-ro` | Solo lectura |
| `root_squash` | Por defecto | El root del cliente se trata como el usuario `nobody` |
| `no_root_squash` | `-maproot=root` | El root del cliente mantiene sus privilegios |
| `all_squash` | `-mapall=nobody` | Todos los usuarios del cliente se tratan como `nobody` |
| `anonuid` / `anongid` | `-maproot=1000:1000` o `-mapall=1000:1000` | Usuario y grupo a los que se asignan los clientes |
| `sync` / `async` | El servidor trabaja en modo seguro (sync) por defecto | El modo `async` se activa para todo el servidor con `sysctl vfs.nfsd.async=1` (con el mismo riesgo que indica el libro) |
| `fsid=0` / `fsid=root` (NFSv4) | Línea **`V4:`** | Define la raíz de NFSv4 (ej. `V4: /srv -network 192.168.56.0/24`) |
| `subtree_check` | No existe | — |
| — | `-alldirs` | Permite que los clientes monten cualquier subcarpeta (solo si la ruta es la raíz de un sistema de archivos) |
| — | `-sec=krb5` | Autenticación con Kerberos |

**Compartir con ZFS** (propio de FreeBSD): un dataset ZFS se puede compartir sin tocar `/etc/exports`, con una propiedad (se usan las mismas opciones de FreeBSD):
```
zfs set sharenfs="-maproot=root -network 192.168.56.0/24" tank/nfs
```
ZFS guarda estas líneas en `/etc/zfs/exports`, que `mountd` lee automáticamente.

### 2.2 Utilidades NFS (equivalente a la Tabla 10.15)

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `exportfs -r` / `exportfs -ra` | **`service mountd reload`** | Vuelve a leer `/etc/exports` (en FreeBSD no existe `exportfs`) |
| `exportfs -v` | `showmount -e localhost` | Ver lo que se está exportando |
| `exportfs -u` / `-o` / `-i` | No existen | Se edita `/etc/exports` y se recarga |
| `mount.nfs` / `umount.nfs` | `mount -t nfs` / `umount` (o `mount_nfs`) | Montar y desmontar |
| `mountstats` / `nfsiostat` | `nfsstat -m` (opciones de cada montaje) y `nfsstat -c -w 1` (actividad en directo) | Estadísticas por montaje |
| `nfsstat` | `nfsstat` (igual; `-c` cliente, `-s` servidor) | Estadísticas |
| `rpcinfo` | `rpcinfo -p` (igual) | Servicios RPC |
| `showmount` | `showmount -e servidor` / `showmount -a` (igual) | Exportaciones y clientes conectados |
| — | `nfsdumpstate` | Muestra los bloqueos y clientes activos de NFSv4 |

### 2.3 Opciones de montaje (equivalente a la Tabla 10.17)

| Opción Linux | Opción FreeBSD 15 | Función |
|---|---|---|
| `intr` | `intr` | Permite interrumpir si el servidor cae |
| `nfsvers=3` / `nfsvers=4` | `nfsv3` / `nfsv4` (también `vers=3` / `vers=4`) | Versión de NFS |
| `tcp` | `tcp` (valor por defecto) | Usar TCP |
| `udp` | `udp` | Usar UDP (solo NFSv3) |
| `port=número` | `port=número` | Puerto del servidor |
| `nolock` | `nolockd` | Sin bloqueo de ficheros (NFSv3) |
| `noexec` / `nosuid` | `noexec` / `nosuid` (iguales) | Sin ejecutables / sin SUID |
| — | `soft` | Da error en lugar de esperar indefinidamente si el servidor no responde |
| — | `late` (en `/etc/fstab`) | Monta después de que la red esté lista (muy recomendable para NFS) |

Montaje manual:
```
mount -t nfs -o nfsv4 192.168.56.102:/srv/nfs_share_perm /mnt/nfs
```

Entrada en `/etc/fstab` equivalente al ejemplo del libro:
```
192.168.56.102:/srv/nfs_share_perm  /home/usuario/NFSPerm  nfs  rw,nfsv3,tcp,intr,late  0  0
```

Igual que recomienda el libro, para montajes bajo demanda se puede usar el **AutoFS** de FreeBSD (Capítulo 4), incluido su mapa especial `/net`.

### 2.4 Comprobación y solución de problemas

| Linux | FreeBSD 15 |
|---|---|
| `rpcinfo -p` (comprobar `portmapper`, `mountd`, `nfs`) | `rpcinfo -p` (igual) |
| `exportfs -v` / `showmount -e` | `showmount -e` |
| `showmount -a` | `showmount -a` (igual) |
| `ping`, `traceroute`, `nmap` | Iguales |
| `/var/log/messages` | `/var/log/messages` (igual; `mountd` avisa aquí de los errores de sintaxis en `/etc/exports`) |

Mensajes de error típicos y su solución en FreeBSD:

| Error | Solución en FreeBSD 15 |
|---|---|
| `RPC: Program Not Registered` | `service mountd reload` (o comprobar que `nfsd` está arrancado) |
| `no route to host` | Servidor caído o cortafuegos (igual que en Linux) |
| `connection refused` | Comprobar `service rpcbind status` |
| `Permission denied` al montar | Revisar `/etc/exports` y los mensajes de `mountd` en `/var/log/messages` |

---

## 3. FTP

**Cambio importante en FreeBSD 15:** el servidor FTP propio de FreeBSD (`ftpd`), que venía en el sistema base, **se ha eliminado** en esta versión. Quien lo necesite puede instalarlo con el paquete `freebsd-ftpd`. Para las opciones del libro (vsftpd y Pure-FTPd), se usan sus paquetes.

Recomendación general: siempre que sea posible, usar **SFTP** (incluido en `sshd`, sistema base) en lugar de FTP, porque cifra usuarios, contraseñas y datos.

### 3.1 vsftpd

| Elemento | Linux | FreeBSD 15 |
|---|---|---|
| Paquete | `vsftpd` | `vsftpd` |
| Fichero de configuración | `/etc/vsftpd.conf` o `/etc/vsftpd/vsftpd.conf` | **`/usr/local/etc/vsftpd.conf`** |
| Directorio del usuario anónimo | `/var/ftp/` o `/srv/ftp/` | El `HOME` del usuario `ftp` (se suele crear en `/var/ftp`) o el indicado con `anon_root` |
| Activar y arrancar | `systemctl enable --now vsftpd` | `sysrc vsftpd_enable="YES"` + `service vsftpd start` |
| Reiniciar tras cambios | `systemctl restart vsftpd` | `service vsftpd restart` |

En FreeBSD, vsftpd funciona en **modo independiente** (`listen=YES`); el archivo de ejemplo del paquete ya viene así. También necesita la carpeta vacía que indica `secure_chroot_dir` (el paquete usa `/usr/local/share/vsftpd/empty`).

**Las directivas de la Tabla 10.18 son idénticas** (`anonymous_enable`, `local_enable`, `write_enable`, `anon_upload_enable`, `anon_mkdir_write_enable`, `anon_other_write_enable`, `anon_root`, `anon_world_readable_only`, `chown_uploads`, `chroot_local_user`, `chroot_list_enable`, `ftp_username`, `listen`, `listen_ipv6`, `userlist_enable`, `log_ftp_protocol`), y los tres ejemplos del libro (con usuario, anónimo y con subidas anónimas) funcionan sin cambios.

**Usuarios en FreeBSD:**

| Tarea | Linux | FreeBSD 15 |
|---|---|---|
| Crear el usuario del acceso anónimo (si no existe) | `useradd -d /var/ftp -s /sbin/nologin ftp` | `pw useradd ftp -d /var/ftp -s /usr/sbin/nologin` |
| Crear un usuario FTP sin acceso a la shell | `useradd -s /sbin/nologin usuario` | `pw useradd usuario -m -s /usr/sbin/nologin` |
| Shell sin acceso | `/usr/sbin/nologin` o `/sbin/nologin` | **`/usr/sbin/nologin`** |
| Lista de shells válidas | `/etc/shells` | `/etc/shells` (por defecto **no** incluye `nologin`; hay que añadirlo si vsftpd usa PAM, igual que advierte el libro) |

**TCP Wrappers:** vsftpd puede usarlos con la directiva `tcp_wrappers=YES`. En FreeBSD las reglas van en el único archivo `/etc/hosts.allow`, con la sintaxis de `: allow` / `: deny` (Capítulo 6):
```
vsftpd : 192.168.56.0/255.255.255.0 : allow
vsftpd : ALL : deny
```

### 3.2 Pure-FTPd

| Elemento | Linux | FreeBSD 15 |
|---|---|---|
| Instalación | `apt-get install pure-ftpd` / repositorio EPEL en Red Hat | `pkg install pure-ftpd` (repositorio oficial, no hace falta nada extra) |
| Configuración | Opciones de línea de comandos (o archivos sueltos, según distribución) | **`/usr/local/etc/pure-ftpd.conf`** (el paquete trae un ejemplo `pure-ftpd.conf.sample`), donde cada opción se escribe como una línea |
| Activar y arrancar | `systemctl` | `sysrc pureftpd_enable="YES"` + `service pure-ftpd start` |

Las **opciones de línea de comandos de la Tabla 10.19** (`-4`, `-6`, `-A`, `-a gid`, `-B`) son **las mismas**, y siguen pudiendo usarse si se lanza `pure-ftpd` a mano. En el archivo de configuración tienen nombres equivalentes (por ejemplo, `ChrootEveryone yes` equivale a `-A`, e `IPV4Only yes` a `-4`).

### 3.3 Otras opciones de FTP

| Programa | Paquete | Notas |
|---|---|---|
| ftpd clásico de FreeBSD | `freebsd-ftpd` | El que venía en el sistema base hasta FreeBSD 14 (archivos `/etc/ftpusers`, `/etc/ftpchroot`, `/etc/ftpwelcome`) |
| ProFTPD | `proftpd` | Configuración parecida a Apache |
| SFTP | Incluido en `sshd` | Recomendado. Para encerrar usuarios en su carpeta: `ChrootDirectory` + `ForceCommand internal-sftp` en `/etc/ssh/sshd_config` |

---

## 4. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 10) | Equivalente en FreeBSD 15 |
|---|---|
| Paquete `samba` | Paquete `sambaXXX` (ej. `samba420`) |
| `/etc/samba/smb.conf` | `/usr/local/etc/smb4.conf` |
| `/var/log/samba/` | `/var/log/samba4/` |
| `/var/lib/samba/` | `/var/db/samba4/private/` |
| `systemctl start smb` | `service samba_server start` |
| `useradd` | `pw useradd` / `adduser` |
| `mount.cifs` | `mount_smbfs` (solo SMB1) o `fusefs-smbnetfs` (SMB2/3) |
| `credentials=` en `fstab` | `/etc/nsmb.conf` |
| `ss -utlpn` | `sockstat -4 -l` |
| NFS (paquete `nfs-utils` / `nfs-kernel-server`) | Sistema base (`nfs_server_enable="YES"`) |
| `/etc/exports`: `ruta cliente(opciones)` | `/etc/exports`: `ruta -opciones cliente` |
| `no_root_squash` / `all_squash` | `-maproot=root` / `-mapall=nobody` |
| `ro` | `-ro` |
| `fsid=0` (NFSv4) | Línea `V4:` |
| `exportfs -ra` | `service mountd reload` |
| — | `zfs set sharenfs=...` |
| `mountstats` / `nfsiostat` | `nfsstat -m` / `nfsstat -w 1` |
| `nfsvers=4` / `nolock` | `nfsv4` / `nolockd` |
| — | `late` en `/etc/fstab` para NFS |
| `/etc/vsftpd.conf` | `/usr/local/etc/vsftpd.conf` |
| `/sbin/nologin` | `/usr/sbin/nologin` |
| Pure-FTPd con EPEL | `pkg install pure-ftpd` + `/usr/local/etc/pure-ftpd.conf` |
| `/etc/hosts.allow` + `/etc/hosts.deny` | Solo `/etc/hosts.allow` con `: allow` / `: deny` |
| — | `ftpd` del sistema base eliminado en FreeBSD 15 (paquete `freebsd-ftpd`) |

---

*Documento elaborado como contrapartida en FreeBSD 15 al Capítulo 10 ("Sharing Files") del libro LPIC-2. Fuentes: FreeBSD Handbook (secciones "Network File System (NFS)", "File and Print Services for Microsoft Windows Clients (Samba)" y "File Transfer Protocol (FTP)" del capítulo "Network Servers"), notas de la versión FreeBSD 15.0 (eliminación de `ftpd`) y páginas de manual de FreeBSD: exports(5), mountd(8), nfsd(8), mount_nfs(8), nfsstat(1), showmount(8), rpcinfo(8), mount_smbfs(8), nsmb.conf(5), rc.conf(5), pw(8).*
