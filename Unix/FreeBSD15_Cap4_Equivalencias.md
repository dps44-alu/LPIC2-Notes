# LPIC-2 · Capítulo 4: Managing the Filesystem
### Equivalencias en FreeBSD 15

Este documento recoge, para cada concepto visto en el Capítulo 4 del libro LPIC-2 (tipos de sistemas de archivos, formateo, montaje, mantenimiento, swap, AutoFS, medios ópticos y cifrado), cuál es su equivalente en **FreeBSD 15**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo: FreeBSD tiene **dos sistemas de archivos nativos**:
- **UFS** (Unix File System): el clásico de FreeBSD. Sencillo y ligero. Su papel es parecido al de ext4 en Linux.
- **ZFS**: sistema de archivos y gestor de volúmenes en uno, con snapshots, compresión, RAID y comprobación de integridad. Su papel es parecido al de Btrfs en Linux, pero más maduro. **Es la opción por defecto del instalador.**

Además, los discos se gestionan con **GEOM**, el sistema de FreeBSD que hace de capa entre el disco y el sistema de archivos (particiones, etiquetas, cifrado, RAID). Muchas herramientas de este capítulo son parte de GEOM (`gpart`, `glabel`, `geli`).

---

## 1. Conceptos básicos de sistemas de archivos

Los conceptos del libro (partición, volumen, formateo, inodo, enlace duro) son **los mismos** en FreeBSD. UFS usa inodos igual que ext4. ZFS también identifica cada fichero con un número de objeto, que `ls -i` muestra como número de inodo.

**Herramientas de particionado:**

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `fdisk`, `gdisk`, `parted` | **`gpart`** | Crear, ver, modificar y borrar tablas de particiones (GPT y MBR) |
| — | `bsdinstall partedit` | Editor de particiones con menús (el mismo que usa el instalador) |

**Comandos básicos de `gpart`:**

| Comando | Función |
|---|---|
| `gpart show` | Muestra todos los discos y sus particiones |
| `gpart show -l` | Igual, mostrando las **etiquetas** de las particiones |
| `gpart create -s gpt ada1` | Crea una tabla de particiones GPT en el disco `ada1` |
| `gpart add -t freebsd-ufs -a 1m -s 20g -l datos ada1` | Crea una partición UFS de 20 GB, alineada a 1 MB, con la etiqueta `datos` |
| `gpart add -t freebsd-zfs -a 1m ada1` | Crea una partición para ZFS con el resto del disco |
| `gpart add -t freebsd-swap -s 4g ada1` | Crea una partición de swap |
| `gpart delete -i 2 ada1` | Borra la partición número 2 |
| `gpart destroy -F ada1` | Borra la tabla de particiones completa |
| `gpart resize -i 1 ada1` | Amplía la partición 1 hasta ocupar el espacio libre |
| `gpart modify -i 1 -l nueva ada1` | Cambia la etiqueta de la partición 1 |

Tipos de partición más usados: `freebsd-ufs`, `freebsd-zfs`, `freebsd-swap`, `freebsd-boot` (arranque BIOS), `efi` (partición EFI), `ms-basic-data` (FAT/NTFS), `linux-data`.

Nombres de las particiones: `/dev/ada1p1` es la partición 1 del disco `ada1` en GPT. En discos MBR antiguos se usan "slices": `/dev/ada1s1`, y dentro de ellos particiones BSD: `/dev/ada1s1a`.

---

## 2. Tipos de sistemas de archivos

### 2.1 Nativos de FreeBSD (equivalente a la Tabla 4.1)

| Nombre | Tamaño máx. de fichero y de FS | Integridad | Notas |
|---|---|---|---|
| **UFS2** | Enormes (en la práctica, sin límite para el hardware actual) | **Soft Updates** (SU) y **Soft Updates + Journaling** (SU+J) | El sistema clásico de FreeBSD. Equivale en uso a ext4 |
| **ZFS** | 16 EiB por fichero, 256 ZiB por pool | **COW** (copy-on-write) + sumas de verificación de todos los datos | RAID, snapshots, compresión, cifrado y gestión de volúmenes integrados. Equivale a Btrfs |

**Soft Updates en UFS** (en lugar del journaling clásico): UFS ordena las escrituras en disco de forma que, tras un corte de luz, el sistema de archivos solo puede quedar con algo de espacio "perdido", nunca con datos incoherentes. Por eso puede revisarse con `fsck` **en segundo plano** con el sistema ya arrancado. Con **SU+J** se añade un pequeño journal para que esa revisión tras un fallo sea casi inmediata. El instalador usa SU+J por defecto en UFS.

### 2.2 No nativos (equivalente a la Tabla 4.2)

| Sistema de archivos | Soporte en FreeBSD 15 | Cómo se usa |
|---|---|---|
| FAT / FAT32 (`vfat`) | Lectura y escritura, en el sistema base | `mount -t msdosfs` |
| exFAT | Lectura y escritura, con paquete | `pkg install fusefs-exfat` |
| NTFS | Lectura y escritura, con paquete (mediante FUSE) | `pkg install fusefs-ntfs` y luego `ntfs-3g` |
| ext2 / ext3 / ext4 | Soporte en el sistema base (módulo `ext2fs`). Lectura y escritura, pero **sin usar el journal** de ext3/ext4 | `mount -t ext2fs` |
| XFS | Sin soporte nativo | Solo mediante paquetes FUSE experimentales (ej. `fusefs-lkl`) |
| Btrfs | Sin soporte nativo | Igual que XFS |
| ZFS | **Nativo** (ver 2.1) | — |
| ISO9660 / UDF | Nativo | `mount -t cd9660` / `mount -t udf` (ver apartado 11) |
| NFS | Nativo | `mount -t nfs` (Capítulo 10) |

Para usar sistemas FUSE hay que cargar el módulo: `kldload fusefs` (o `sysrc kld_list+="fusefs"` para que se cargue siempre).

### 2.3 Consultar los sistemas de archivos soportados

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `cat /proc/filesystems` | `lsvfs` | Lista los tipos de sistema de archivos que el kernel tiene cargados ahora mismo |
| — | `ls /boot/kernel/ \| grep fs` | Muestra los módulos de sistemas de archivos que se pueden cargar (`ext2fs.ko`, `fusefs.ko`, `udf.ko`...) |

---

## 3. Creación de un sistema de archivos (formateo)

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `mkfs -t ext4 /dev/sdb1` | `newfs /dev/ada1p1` | Crea un sistema de archivos **UFS** |
| — | `newfs -U /dev/ada1p1` | UFS con Soft Updates activados |
| — | `newfs -j /dev/ada1p1` | UFS con Soft Updates + Journaling (SU+J) |
| — | `newfs -L datos /dev/ada1p1` | UFS con etiqueta `datos` (accesible como `/dev/ufs/datos`) |
| — | `newfs -t /dev/ada1p1` | Activa TRIM (para discos SSD) |
| `mkfs.vfat /dev/sdb1` | `newfs_msdos -F 32 /dev/da0p1` | Crea un sistema FAT32 |
| `mkfs.ext4` | `mkfs.ext4` / `mke2fs` (paquete `e2fsprogs`) | Crear ext2/3/4 desde FreeBSD |
| `mkfs.btrfs` | `zpool create` + `zfs create` | En ZFS no hay "formateo" (ver abajo) |

**En ZFS no se formatea**: se crea un **pool** con uno o varios discos y dentro se crean **datasets** (parecido a subvolúmenes de Btrfs), que se montan solos.

| Comando | Función |
|---|---|
| `zpool create tank /dev/ada1p1` | Crea un pool llamado `tank` y lo monta en `/tank` |
| `zfs create tank/datos` | Crea un dataset, montado automáticamente en `/tank/datos` |
| `zfs create -o mountpoint=/srv/web tank/web` | Crea un dataset con un punto de montaje concreto |

**Comprobar el tipo tras formatear:**

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `blkid` | `fstyp /dev/ada1p1` | Indica qué sistema de archivos tiene una partición |
| — | `fstyp -l /dev/ada1p1` | Igual, mostrando también la etiqueta |
| `parted -l` | `gpart show` | Muestra particiones y su tipo |
| `lsblk -f` | `gpart show -l`, `glabel status` | Particiones con sus etiquetas (existe también `lsblk` como paquete) |

---

## 4. Montaje del sistema de archivos

### 4.1 Montaje temporal: `mount`

Sintaxis básica: `mount -t tipo dispositivo punto_de_montaje` (igual que en Linux). Los tipos cambian de nombre: `ufs`, `msdosfs`, `ext2fs`, `cd9660`, `udf`, `nfs`, `tmpfs`, `nullfs`.

Ejemplo: `mount -t ufs /dev/ada1p1 /mnt`

**Opciones de `mount` (equivalente a la Tabla 4.3):**

| Opción Linux | Opción FreeBSD 15 | Función |
|---|---|---|
| `-a` | `-a` | Monta todo lo que hay en `/etc/fstab` |
| `-F` (montar en paralelo) | No existe | En FreeBSD `-F archivo` sirve para usar **otro archivo fstab** |
| `-f` (simular) | `-d` | Simula el montaje sin montar (en FreeBSD `-f` significa **forzar**) |
| `-L etiqueta` | No existe. Se monta usando el dispositivo de la etiqueta: `/dev/gpt/datos` o `/dev/ufs/datos` | Montar por etiqueta |
| `-U uuid` | No existe. Se usa `/dev/gptid/UUID` | Montar por UUID |
| `-n` (sin `/etc/mtab`) | No aplica: FreeBSD no tiene `/etc/mtab`, la lista de montajes la guarda el kernel | — |
| `-o opciones` | `-o opciones` | Opciones adicionales |
| `-r` | `-r` | Solo lectura |
| `-w` | `-w` | Lectura/escritura |
| `-t tipo` | `-t tipo` | Tipo de sistema de archivos |
| `-v` | `-v` | Modo detallado |
| `-o remount` | `-u` | Cambia las opciones de algo ya montado (ej. `mount -u -o rw /`) |
| — | `-p` | Muestra los montajes actuales en formato `fstab` |

Cada tipo tiene además su programa propio, que es lo que `mount -t` llama por debajo: `mount_msdosfs`, `mount_cd9660`, `mount_nfs`, `mount_nullfs`, `mount_tmpfs`, etc.

Desmontar: `umount dispositivo` o `umount punto_de_montaje` (igual). `umount -f` fuerza el desmontaje; `umount -a` desmonta todo lo de `fstab`.

### 4.2 Montaje persistente: `/etc/fstab`

Mismo archivo y **mismos 6 campos** que en Linux:

```
dispositivo       punto_de_montaje  tipo    opciones        dump  fsck
/dev/gpt/rootfs   /                 ufs     rw              1     1
/dev/gpt/datos    /datos            ufs     rw              2     2
/dev/gpt/swap     none              swap    sw              0     0
tmpfs             /tmp              tmpfs   rw,mode=1777    0     0
/dev/cd0          /cdrom            cd9660  ro,noauto       0     0
```

Diferencias con Linux:
- No se usa `UUID=` ni `LABEL=`: se escribe directamente la ruta de la etiqueta (`/dev/gpt/datos`, `/dev/ufs/datos`) o del identificador (`/dev/gptid/...`).
- Las líneas de swap ponen `none` en el punto de montaje.
- Opciones propias útiles: `late` (montar después de arrancar la red, útil para NFS), `noauto` (no montar al arrancar), `failok` (si falla el montaje, el arranque sigue en vez de ir a modo mono-usuario).
- **Los datasets ZFS no van en `/etc/fstab`**: se montan solos según su propiedad `mountpoint`, siempre que esté activado `zfs_enable="YES"` en `/etc/rc.conf`.

`mount -a` funciona igual que en Linux (monta lo pendiente y sirve para comprobar el archivo).

### 4.3 Unidades de montaje de systemd

No existen en FreeBSD (no hay systemd). Se usa siempre `/etc/fstab` para UFS y otros, y las **propiedades de ZFS** para los datasets:

| Comando | Función |
|---|---|
| `zfs set mountpoint=/srv/datos tank/datos` | Cambia dónde se monta un dataset (se aplica al momento) |
| `zfs set canmount=off tank/datos` | Hace que el dataset no se monte |
| `zfs mount tank/datos` / `zfs unmount tank/datos` | Monta o desmonta un dataset a mano |
| `zfs mount -a` | Monta todos los datasets |

### 4.4 Ver los sistemas de archivos montados

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `mount` | `mount` | Lista los sistemas montados |
| `cat /proc/mounts` / `/etc/mtab` | `mount -p` | Lista en formato `fstab` |
| `mountpoint directorio` | `df directorio` | Indica en qué sistema de archivos está ese directorio (no existe `mountpoint`) |
| `findmnt` | `mount`, `df -h`, `zfs list` | No existe una vista en árbol; `zfs list` muestra los datasets y dónde se montan |
| — | `df -hT` | Espacio usado y tipo de cada sistema de archivos montado |
| `lsblk -f` | `gpart show -l`, `glabel status` | Dispositivos con sus etiquetas |
| `e2label` | `tunefs -L etiqueta` (UFS) o `gpart modify -l` (etiqueta de la partición) | Ver o cambiar etiquetas |
| `findfs LABEL=etiqueta` | `glabel status` | Muestra a qué dispositivo corresponde cada etiqueta |

**Etiquetas en FreeBSD (muy usadas en lugar de los UUID):**

| Tipo de etiqueta | Ruta | Cómo se crea |
|---|---|---|
| Etiqueta de partición GPT | `/dev/gpt/nombre` | `gpart add -l nombre` o `gpart modify -l nombre` |
| Etiqueta del sistema UFS | `/dev/ufs/nombre` | `newfs -L nombre` o `tunefs -L nombre` |
| Etiqueta genérica GEOM | `/dev/label/nombre` | `glabel label nombre /dev/ada1p1` |
| Identificador único de partición GPT | `/dev/gptid/UUID` | Automática |

---

## 5. UUID de los sistemas de archivos

En FreeBSD lo habitual es usar **etiquetas GPT** en lugar de UUID, porque son igual de estables y más fáciles de leer. Aun así, los UUID existen:

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `blkid` | `gpart list ada1` (campo `rawuuid`) | UUID de cada partición GPT |
| — | `ls /dev/gptid/` | Particiones accesibles por UUID |
| — | `zpool get guid tank` | Identificador único de un pool ZFS |
| `uuidgen` | `uuidgen` (igual, sistema base) | Genera un UUID |
| `tune2fs -U` | No hay equivalente en UFS | — |

El instalador de FreeBSD suele **desactivar** las rutas `/dev/gptid/` y `/dev/diskid/` para que no aparezcan nombres duplicados; lo hace con estas líneas en `/boot/loader.conf`:
```
kern.geom.label.gptid.enable="0"
kern.geom.label.disk_ident.enable="0"
```
Si se necesitan, basta con borrarlas (o ponerlas a `1`) y reiniciar.

---

## 6. Sistemas de archivos temporales/virtuales

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `/proc` | `procfs`, **no montado por defecto** | La información del kernel se consulta con `sysctl` (Capítulo 3) |
| `tmpfs` | `tmpfs` | Sistema de archivos en memoria: `mount -t tmpfs tmpfs /tmp` |
| — | `tmpmfs="YES"` en `/etc/rc.conf` | Monta `/tmp` en memoria automáticamente al arrancar |
| — | `clear_tmp_enable="YES"` en `/etc/rc.conf` | Borra el contenido de `/tmp` en cada arranque |
| `/dev` (udev) | `devfs` | `/dev` virtual gestionado por el kernel |
| — | `fdescfs` | Descriptores de archivo en `/dev/fd` (lo necesitan algunos programas, como bash) |
| `mount --bind` | `mount -t nullfs /origen /destino` | Montar una carpeta en otro sitio (muy usado con jails) |
| Disco RAM (`/dev/ram`) | `mdconfig -a -t swap -s 1g` | Crea un disco en memoria (`/dev/md0`) sobre el que se puede usar `newfs` |

Igual que en Linux, estos sistemas no ocupan espacio en disco y su contenido se pierde al apagar.

---

## 7. El sistema de archivos ZFS (equivalente a la sección de Btrfs)

Btrfs no existe en FreeBSD. Su equivalente directo es **ZFS**, que ofrece lo mismo (COW, RAID integrado, snapshots, compresión, scrub) y más. ZFS usa dos comandos: **`zpool`** (gestiona los discos y el pool) y **`zfs`** (gestiona los datasets dentro del pool).

**Consulta de información:**

| Btrfs (Linux) | ZFS (FreeBSD 15) | Función |
|---|---|---|
| `btrfs filesystem show` | `zpool status` | Estado del pool y de cada disco |
| `btrfs filesystem df` | `zpool list` y `zfs list` | Espacio total del pool / espacio de cada dataset |
| `btrfs property get` | `zfs get all tank/datos` | Muestra las propiedades de un dataset |

**Utilidades de ajuste (equivalente a la Tabla 4.8):**

| Btrfs | ZFS | Función |
|---|---|---|
| `btrfs balance` | No existe (ZFS reparte los datos nuevos automáticamente) | — |
| `btrfs-convert` | No existe | No se puede convertir UFS a ZFS; hay que copiar los datos |
| `btrfstune` / `btrfs property set` | `zfs set propiedad=valor dataset` | Cambia propiedades (ej. `zfs set compression=zstd tank/datos`) |
| — | `zfs set quota=10G tank/datos` | Limita el espacio de un dataset |
| — | `zpool add tank /dev/ada2p1` | Añade un disco al pool para ampliarlo |

**Utilidades de comprobación/reparación (equivalente a la Tabla 4.11):**

| Btrfs | ZFS | Función |
|---|---|---|
| `btrfs scrub` | `zpool scrub tank` | Lee todo el pool comprobando las sumas de verificación y repara lo que pueda (con redundancia) |
| `btrfs check` | `zpool status -v tank` | Muestra errores y qué ficheros están dañados |
| `btrfs rescue` | `zpool import -F tank` | Intenta recuperar un pool dañado volviendo a un estado consistente anterior |
| `btrfs restore` | `zpool import -o readonly=on tank` | Importa el pool en solo lectura para rescatar ficheros |
| — | `zpool clear tank` | Borra los contadores de errores tras solucionar el problema |

ZFS no tiene `fsck`: la integridad se comprueba con `scrub`. FreeBSD puede lanzar un scrub periódico automáticamente con `daily_scrub_zfs_enable="YES"` en `/etc/periodic.conf`.

**RAID integrado en ZFS** (Btrfs: 0, 1 y 10):

| Tipo | Comando de creación | Equivalente RAID |
|---|---|---|
| Stripe | `zpool create tank ada1 ada2` | RAID 0 |
| Mirror | `zpool create tank mirror ada1 ada2` | RAID 1 |
| Varios mirror | `zpool create tank mirror ada1 ada2 mirror ada3 ada4` | RAID 10 |
| RAID-Z1 / Z2 / Z3 | `zpool create tank raidz1 ada1 ada2 ada3` | Parecido a RAID 5 / 6 / "triple paridad" |

A diferencia de Btrfs, en ZFS los niveles con paridad (RAID-Z) sí son estables y recomendados. Más detalle en el Capítulo 5.

---

## 8. Mantenimiento y ajuste de sistemas de archivos

### 8.1 UFS (equivalente a las herramientas de ext2/ext3/ext4, Tabla 4.6)

| Utilidad ext (Linux) | Utilidad UFS (FreeBSD 15) | Función |
|---|---|---|
| `debugfs` | `fsdb` | Herramienta interactiva para modificar metadatos (solo para expertos) |
| `e2label` | `tunefs -L etiqueta` | Cambia la etiqueta |
| `resize2fs` | `growfs` | **Amplía** el sistema de archivos (se puede hacer montado). UFS **no se puede reducir** |
| `tune2fs` | `tunefs` | Ajusta atributos |

**Opciones de `tunefs`** (se usan con el sistema desmontado o montado en solo lectura):

| Opción | Función |
|---|---|
| `tunefs -p /dev/ada1p1` | Muestra la configuración actual |
| `tunefs -L datos /dev/ada1p1` | Cambia la etiqueta |
| `tunefs -n enable /dev/ada1p1` | Activa Soft Updates |
| `tunefs -j enable /dev/ada1p1` | Activa Soft Updates + Journaling |
| `tunefs -t enable /dev/ada1p1` | Activa TRIM |
| `tunefs -a enable /dev/ada1p1` | Activa ACLs (listas de control de acceso) |

Ampliar una partición UFS (tras añadir espacio al disco, por ejemplo en una máquina virtual):
```
gpart recover ada0
gpart resize -i 2 ada0
growfs /
```

### 8.2 XFS

No existe en FreeBSD. Las herramientas del libro (`xfs_admin`, `xfs_fsr`, `xfs_growfs`) no tienen equivalente. Para ampliar un sistema, se usa `growfs` (UFS) o se añaden discos al pool (ZFS).

### 8.3 Comprobación y reparación

**UFS (equivalente a la Tabla 4.9):**

| Utilidad ext (Linux) | Utilidad UFS (FreeBSD 15) | Función |
|---|---|---|
| `fsck.ext4` | `fsck_ufs` (o `fsck -t ufs`) | Comprueba y repara; los ficheros recuperados van a `lost+found`, igual que en Linux |
| `fsck -y` | `fsck -y` | Responde "sí" a todo |
| — | `fsck -p` | Modo "preen": repara solo los problemas sencillos, sin preguntar (el que se usa al arrancar) |
| — | `fsck -B` | Revisión en segundo plano de un sistema montado (gracias a Soft Updates) |
| `debugfs` (rescatar datos) | `fsdb` | Acceso a bajo nivel |
| `dumpe2fs` | `dumpfs` | Muestra toda la información del sistema de archivos |
| `tune2fs -l` | `tunefs -p` | Muestra los atributos |

Igual que en Linux, no se debe reparar con `fsck` un sistema montado en lectura/escritura (salvo el modo `-B`, pensado para ello).

**ZFS:** no usa `fsck` (ver apartado 7: `zpool scrub`, `zpool status`).

**Herramientas de XFS (Tabla 4.10):** no existen en FreeBSD. Las funciones equivalentes serían:

| Utilidad XFS | Equivalente en FreeBSD 15 |
|---|---|
| `xfs_repair` / `xfs_check` | `fsck_ufs` (UFS) o `zpool scrub` (ZFS) |
| `xfsdump` / `xfsrestore` | `dump` / `restore` (UFS) o `zfs send` / `zfs receive` (ZFS), ver Capítulo 2 |
| `xfs_info` | `dumpfs` / `tunefs -p` (UFS) o `zfs get all` (ZFS) |

### 8.4 Monitorización SMART

El paquete es el mismo que en Linux, **`smartmontools`**, y el comando `smartctl` funciona igual. Cambian las rutas y los nombres de disco:

| Elemento | FreeBSD 15 |
|---|---|
| Instalar | `pkg install smartmontools` |
| `smartctl -i /dev/ada0` | Información básica (igual que en Linux, con nombre de disco de FreeBSD) |
| `smartctl -s on /dev/ada0` | Activa SMART |
| `smartctl -H /dev/ada0` | Resumen de salud |
| `smartctl -a /dev/ada0` | Toda la información |
| `smartctl -t short /dev/ada0` | Lanza una autoprueba |
| `smartctl -l error /dev/ada0` | Registro de errores |
| `smartctl -a /dev/nvme0` | Discos NVMe |
| Demonio `smartd` | Se activa con `sysrc smartd_enable="YES"` y `service smartd start` |
| Configuración de `smartd` | **`/usr/local/etc/smartd.conf`** (el paquete trae un ejemplo en `smartd.conf.sample`) |
| `DEVICESCAN` | Igual que en Linux |
| Informe diario | `daily_status_smart_devices="/dev/ada0 /dev/ada1"` en `/etc/periodic.conf` (añade el estado SMART al informe diario que recibe root) |

Herramientas del sistema base relacionadas: `camcontrol identify ada0` (información del disco SATA) y `nvmecontrol logpage -p 2 nvme0` (estado de salud de un disco NVMe).

---

## 9. Espacio de intercambio (swap)

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `mkswap dispositivo` | No hace falta. Basta con que la partición sea de tipo `freebsd-swap` (`gpart add -t freebsd-swap`) | Preparar la swap |
| `swapon dispositivo` | `swapon /dev/ada0p3` | Activa la swap |
| `swapon -a` | `swapon -a` | Activa toda la swap de `/etc/fstab` |
| `swapon -s` / `cat /proc/swaps` | `swapinfo -h` o `swapctl -l` | Muestra el estado de la swap |
| `swapoff dispositivo` | `swapoff /dev/ada0p3` | Desactiva la swap |
| `swapon -p prioridad` | No existe | FreeBSD reparte la carga entre todas las áreas de swap por igual |
| `free -m` | `swapinfo -m` / `top` | Uso de swap |

**Swap en un fichero:**
```
dd if=/dev/zero of=/usr/swap0 bs=1m count=2048
chmod 0600 /usr/swap0
```
Línea en `/etc/fstab`:
```
md99    none    swap    sw,file=/usr/swap0,late    0    0
```
Activarla sin reiniciar: `swapon -aL`.

**Swap cifrada** (ventaja propia de FreeBSD): basta con añadir `.eli` al dispositivo en `/etc/fstab`. En cada arranque se cifra con una clave aleatoria nueva, sin tener que configurar nada más:
```
/dev/ada0p3.eli    none    swap    sw    0    0
```

---

## 10. AutoFS (montaje automático de recursos de red)

FreeBSD tiene su **propio AutoFS en el sistema base** (no es el mismo programa que en Linux, pero usa el mismo formato de mapas, compatible con Solaris/Linux).

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `/etc/default/autofs` / `/etc/sysconfig/autofs` | `/etc/rc.conf` (`autofs_enable="YES"`) | Activar el servicio |
| `/etc/auto.master` | **`/etc/auto_master`** | Mapa maestro |
| `/etc/auto.direct` | `/etc/auto_direct` (o el nombre que se quiera) | Mapa directo |
| `/etc/auto.directorio` | `/etc/auto_directorio` | Mapa indirecto |
| — | `/etc/autofs/` | Mapas especiales que trae el sistema (ver abajo) |
| `TimeOutIdleSec` | Opción `-t segundos` de `autounmountd` | Tiempo de inactividad antes de desmontar (por defecto 600 segundos) |

Piezas del sistema: `automountd` (monta al acceder), `autounmountd` (desmonta tras el tiempo de espera) y el comando `automount` (lee los mapas).

**Activar AutoFS:**
```
sysrc autofs_enable="YES"
service automount start
service automountd start
service autounmountd start
```

**Ejemplo de mapa indirecto** — en `/etc/auto_master`:
```
/datos    auto_datos
```
Y en `/etc/auto_datos`:
```
docs    -fstype=nfs,rw    servidor:/export/docs
```
Al acceder a `/datos/docs`, se monta automáticamente el recurso NFS.

**Ejemplo de mapa directo** — en `/etc/auto_master`:
```
/-    auto_direct
```
Y en `/etc/auto_direct`:
```
/srv/copias    -fstype=nfs    servidor:/export/copias
```

**Mapas especiales incluidos:**

| Línea en `auto_master` | Función |
|---|---|
| `/net -hosts -nobrowse,nosuid,intr` | Acceso automático a los recursos NFS de cualquier servidor en `/net/servidor/` |
| `/media -media -nosuid` | Monta automáticamente los medios extraíbles (USB, CD) en `/media` |

Tras editar los mapas: `automount` (o `automount -v` para ver el detalle). Igual que en Linux, lo gestionado por AutoFS no debe estar en `/etc/fstab`.

---

## 11. Medios ópticos: ISO9660 y UDF

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `mkisofs` / `genisoimage` | **`makefs -t cd9660`** (sistema base) o `mkisofs` / `xorriso` (paquetes `cdrtools` y `xorriso`) | Crear una imagen ISO |
| `cdrecord` | `cdrecord` (paquete `cdrtools`) | Grabar una ISO en un disco óptico |
| `file imagen.iso` | `file imagen.iso` (igual) | Comprobar si es arrancable |
| `mount -t iso9660 /dev/sr0 /mnt` | `mount -t cd9660 /dev/cd0 /mnt` | Montar un CD/DVD |
| `mount -t udf` | `mount -t udf /dev/cd0 /mnt` | Montar un disco UDF |
| `mount -o loop imagen.iso /mnt` | `mdconfig -a -t vnode -f imagen.iso` (crea `/dev/md0`) y luego `mount -t cd9660 /dev/md0 /mnt` | Montar una imagen ISO |
| — | `mdconfig -d -u 0` | Elimina el dispositivo `md0` tras desmontar |

**Ejemplos de `makefs`:**
```
makefs -t cd9660 -o rockridge -o label=DATOS datos.iso /carpeta
makefs -t cd9660 -o rockridge -o bootimage='i386;/carpeta/boot/cdboot' -o no-emul-boot arranque.iso /carpeta
```

| Opción de `makefs` | Extensión ISO9660 |
|---|---|
| `-o rockridge` | Rock Ridge (metadatos Unix) |
| `-o bootimage=...` + `-o no-emul-boot` | El Torito (disco arrancable) |
| — | Joliet no está soportado por `makefs`; para Joliet usar `mkisofs -J` o `xorriso` |

Nombres de dispositivo: `/dev/cd0` (primer lector óptico, el `/dev/sr0` de Linux).

---

## 12. Sistemas de archivos cifrados

| Linux | FreeBSD 15 | Características |
|---|---|---|
| **LUKS** (`cryptsetup`) | **GELI** (`geli`, sistema base) | Cifrado de disco o partición completa, con metadatos en el propio dispositivo y **dos ranuras de clave** (se puede cambiar la contraseña sin volver a cifrar). Es el método recomendado para UFS |
| `dm-crypt` básico (sin metadatos) | `geli onetime` | Cifrado con clave aleatoria de un solo uso, sin metadatos (se usa para swap y datos temporales) |
| — | **Cifrado nativo de ZFS** | Cifra datasets concretos dentro de un pool |
| `eCryptfs` (fichero a fichero, en capa) | `pefs` (paquete `pefs-kmod`) | Cifrado en capa sobre otro sistema de archivos. Poco habitual; lo normal es GELI o ZFS |

No se pueden abrir volúmenes LUKS de forma nativa en FreeBSD.

**Comandos de GELI:**

| Comando | Función |
|---|---|
| `geli init -s 4096 -l 256 /dev/ada1p1` | Prepara la partición para cifrado (pide la contraseña) |
| `geli attach /dev/ada1p1` | Abre el dispositivo cifrado; aparece como `/dev/ada1p1.eli` |
| `newfs /dev/ada1p1.eli` | Formatea el dispositivo cifrado (solo la primera vez) |
| `mount /dev/ada1p1.eli /privado` | Monta el sistema cifrado |
| `geli detach /dev/ada1p1` | Cierra el dispositivo cifrado (tras desmontar) |
| `geli setkey -n 1 /dev/ada1p1` | Pone o cambia la clave en la ranura 1 |
| `geli list` / `geli status` | Muestra los dispositivos cifrados abiertos |
| `geli backup /dev/ada1p1 /root/ada1p1.eli.bak` | Copia de seguridad de los metadatos (imprescindible: sin ellos no se pueden recuperar los datos) |
| `geli restore /root/ada1p1.eli.bak /dev/ada1p1` | Restaura los metadatos |

Abrir el disco automáticamente al arrancar (pide la contraseña en la consola): `geli_devices="ada1p1"` en `/etc/rc.conf` y la línea normal con `/dev/ada1p1.eli` en `/etc/fstab`. Para la partición raíz se usa `geom_eli_load="YES"` en `/boot/loader.conf` (el instalador lo configura solo si se elige cifrar el disco).

**Cifrado nativo de ZFS:**

| Comando | Función |
|---|---|
| `zfs create -o encryption=on -o keyformat=passphrase tank/secreto` | Crea un dataset cifrado con contraseña |
| `zfs unload-key tank/secreto` | Cierra el dataset (tras desmontarlo con `zfs unmount`) |
| `zfs load-key tank/secreto` + `zfs mount tank/secreto` | Pide la contraseña y lo vuelve a montar |
| `zfs change-key tank/secreto` | Cambia la contraseña |
| `zfs get encryption,keystatus tank/secreto` | Muestra el estado del cifrado |

---

## 13. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 4) | Equivalente en FreeBSD 15 |
|---|---|
| `fdisk`, `gdisk`, `parted` | `gpart` |
| ext4 | UFS2 (con Soft Updates / SU+J) |
| Btrfs | ZFS |
| XFS | Sin equivalente nativo |
| vfat / NTFS / ext4 | `msdosfs` / `fusefs-ntfs` / `ext2fs` |
| `/proc/filesystems` | `lsvfs` |
| `mkfs.ext4` | `newfs` |
| `mkfs.vfat` | `newfs_msdos` |
| `mkfs.btrfs` | `zpool create` + `zfs create` |
| `blkid` | `fstyp`, `gpart list`, `glabel status` |
| `mount -o remount` | `mount -u` |
| `mount -f` (simular) | `mount -d` |
| `mount -L` / `mount -U` | `/dev/gpt/etiqueta` / `/dev/gptid/UUID` |
| `/etc/fstab` | `/etc/fstab` (igual, sin `UUID=`/`LABEL=`); ZFS no lo usa |
| Unidades `.mount` de systemd | Propiedad `mountpoint` de ZFS |
| `findmnt`, `lsblk` | `mount`, `df -hT`, `zfs list`, `gpart show -l` |
| `e2label` / `tune2fs` | `tunefs` |
| `resize2fs` | `growfs` (solo ampliar) |
| `debugfs` / `dumpe2fs` | `fsdb` / `dumpfs` |
| `fsck.ext4` | `fsck_ufs` (UFS) / `zpool scrub` (ZFS) |
| `btrfs scrub` / `btrfs filesystem show` | `zpool scrub` / `zpool status` |
| `xfsdump` / `xfsrestore` | `dump` / `restore` o `zfs send` / `zfs receive` |
| `smartctl`, `/etc/smartd.conf` | `smartctl`, `/usr/local/etc/smartd.conf` |
| `mkswap` + `swapon` | Tipo `freebsd-swap` + `swapon` |
| `swapon -s` / `free` | `swapinfo -h` |
| AutoFS, `/etc/auto.master` | AutoFS del sistema base, `/etc/auto_master` |
| `mkisofs` / `genisoimage` | `makefs -t cd9660` |
| `mount -o loop` | `mdconfig` + `mount` |
| LUKS / `cryptsetup` | GELI / `geli` y cifrado nativo de ZFS |
| eCryptfs | `pefs` (paquete) |

---

*Documento elaborado como contrapartida en FreeBSD 15 al Capítulo 4 ("Managing the Filesystem") del libro LPIC-2. Fuentes: FreeBSD Handbook (capítulos "Storage", "GEOM", "The Z File System (ZFS)" y "Other File Systems") y páginas de manual de FreeBSD: gpart(8), newfs(8), tunefs(8), growfs(8), fsck(8), fsck_ffs(8), dumpfs(8), fsdb(8), mount(8), fstab(5), lsvfs(1), fstyp(8), glabel(8), zpool(8), zfs(8), swapon(8), swapinfo(8), mdconfig(8), makefs(8), autofs(5), auto_master(5), automount(8), geli(8).*
