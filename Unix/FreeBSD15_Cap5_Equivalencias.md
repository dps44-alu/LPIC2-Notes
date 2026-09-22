# LPIC-2 · Capítulo 5: Administering Advanced Storage Devices
### Equivalencias en FreeBSD 15

Este documento recoge, para cada concepto visto en el Capítulo 5 del libro LPIC-2 (RAID por software, ajuste de discos, NVMe, iSCSI y gestión de volúmenes lógicos), cuál es su equivalente en **FreeBSD 15**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo: en Linux, RAID (`mdadm`) y LVM se apoyan en el **Device Mapper** y el driver **md**. En FreeBSD todo esto lo hace **GEOM**, un sistema de "capas" que se apilan unas sobre otras: un disco puede pasar por una capa de espejo, luego por una de cifrado y luego tener particiones. Cada tipo de capa es una **clase GEOM** con su propio comando (`gmirror`, `gstripe`, `geli`...).

Además, **ZFS** (Capítulo 4) reúne en una sola herramienta lo que en Linux hacen `mdadm` + LVM + el sistema de archivos. En un FreeBSD moderno, lo habitual es usar **ZFS** tanto para RAID como para "volúmenes lógicos", y dejar las clases GEOM para casos concretos (por ejemplo, un espejo con UFS).

---

## 1. RAID

### 1.1 Niveles de RAID

Los niveles son **los mismos conceptos** que en el libro. Esta es la forma de conseguir cada uno en FreeBSD 15:

| Nivel | Con clases GEOM | Con ZFS (recomendado) |
|---|---|---|
| RAID 0 (striping) | `gstripe` | Pool con varios discos sin redundancia |
| RAID 1 (mirroring) | `gmirror` | `mirror` |
| RAID 3 | `graid3` (FreeBSD sí lo mantiene) | — |
| RAID 5 | `graid` (solo con formatos de RAID de placa base) | `raidz1` |
| RAID 6 | — | `raidz2` |
| RAID 10 | `gmirror` + `gstripe` apilados | Varios `mirror` en el mismo pool |
| Triple paridad | — | `raidz3` (soporta el fallo de 3 discos) |
| Unir discos sin RAID (JBOD / lineal) | `gconcat` | — |

Nota sobre `graid`: sirve para usar el "RAID de la BIOS" (fake RAID de Intel, AMD, etc.) que trae la placa base. Para RAID 5 y 6 por software, lo recomendado en FreeBSD es **ZFS con RAID-Z**, que además evita el problema del "write hole" (datos corruptos si se corta la luz a mitad de escritura).

### 1.2 Implementación de RAID software (equivalente a `mdadm`)

**Comprobaciones previas:**

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `uname -r` | `freebsd-version -k` | Versión del kernel (todas las clases GEOM y ZFS vienen en el sistema base) |
| `cat /proc/mdstat` | `geom -t` | Muestra el árbol de capas GEOM activas |
| `modprobe raid1` | `kldload geom_mirror` | Carga el módulo del tipo de RAID |
| — | `kldload geom_stripe`, `geom_raid3`, `geom_concat`, `geom_raid` | Módulos de las otras clases |
| — | `geom_mirror_load="YES"` en `/boot/loader.conf` | Carga el módulo en cada arranque (necesario para que el RAID esté listo antes de montar) |
| Instalar `mdadm` | No hace falta | Todo viene en el sistema base |

La forma más cómoda de cargar un módulo GEOM a mano es usar el propio comando: `gmirror load` (equivale a `kldload geom_mirror`).

**Preparar los discos:** con `gpart` (Capítulo 4). A diferencia de Linux, **no hace falta un tipo de partición especial** (como `fd` en Linux): las clases GEOM pueden usar discos completos o particiones de cualquier tipo.

### 1.3 Modos de `mdadm` y sus equivalentes (equivalente a la Tabla 5.1)

Tomando `gmirror` como ejemplo (las demás clases usan los mismos verbos):

| Modo `mdadm` | Equivalente en FreeBSD 15 | Función |
|---|---|---|
| `--create` | `gmirror label` | Crea el RAID y guarda sus metadatos en el **último sector** de cada disco |
| `--build` (sin superblock) | `gstripe create` / `gconcat create` | Crea el RAID **sin metadatos** (no se recuerda al reiniciar) |
| `--assemble` / `--autodetect` | Automático | GEOM detecta los metadatos de los discos y monta el RAID solo al arrancar |
| `--monitor` | `gmirror status`, `devd` y `periodic` | Ver apartado 1.8 |
| `--grow` | `gmirror insert` (añadir disco) / `zpool attach`, `zpool add` (ZFS) | Ampliar el RAID |
| `--manage --add` | `gmirror insert gm0 /dev/ada3` | Añadir un disco |
| `--manage --remove` | `gmirror remove gm0 ada3` | Quitar un disco |
| `--misc --zero-superblock` | `gmirror clear ada1` | Borra los metadatos de un disco |
| — | `gmirror destroy gm0` | Elimina el RAID completo |
| — | `gmirror stop gm0` | Para el RAID (sin borrar los metadatos) |

### 1.4 Crear un array

**Con clases GEOM:**

| Tipo | Comando | Dispositivo resultante |
|---|---|---|
| RAID 1 | `gmirror label -v -b round-robin gm0 /dev/ada1 /dev/ada2` | `/dev/mirror/gm0` |
| RAID 0 | `gstripe label -v -s 131072 st0 /dev/ada1 /dev/ada2` | `/dev/stripe/st0` |
| RAID 3 | `graid3 label -v gr0 ada1 ada2 ada3` | `/dev/raid3/gr0` |
| JBOD | `gconcat label -v gc0 ada1 ada2` | `/dev/concat/gc0` |
| RAID de placa base | `graid label Intel r0 RAID1 ada1 ada2` | `/dev/raid/r0` |

Opciones útiles:

| Opción | Equivalente `mdadm` | Función |
|---|---|---|
| `-v` | `--verbose` | Modo detallado |
| `-s tamaño` (gstripe) | `--chunk` | Tamaño de fragmento (en bytes) |
| `-b algoritmo` (gmirror) | — | Cómo se reparten las lecturas entre los discos: `load` (por defecto, al disco menos ocupado), `round-robin` (alternando), `prefer` (siempre al de más prioridad), `split` (dividiendo cada petición) |
| No hay número de discos (`-n`) | `--raid-devices` | El número lo marcan los discos que se escriben en el comando |
| No hay nivel (`-l`) | `--level` | El nivel lo marca el comando usado (`gmirror`, `gstripe`...) |

En `graid3` el número de discos debe ser 3, 5, 9, 17... y el último disco guarda la paridad.

**Con ZFS** (el equivalente directo a `mdadm -C ... -l 6 -n 4` del libro):
```
zpool create tank raidz2 ada1 ada2 ada3 ada4
```

| Comando | Equivalente |
|---|---|
| `zpool create tank mirror ada1 ada2` | RAID 1 |
| `zpool create tank raidz1 ada1 ada2 ada3` | RAID 5 |
| `zpool create tank raidz2 ada1 ada2 ada3 ada4` | RAID 6 |
| `zpool create tank mirror ada1 ada2 mirror ada3 ada4` | RAID 10 |
| `zpool create tank raidz1 ada1 ada2 ada3 spare ada4` | RAID 5 con un disco de repuesto (`-x` en `mdadm`) |

### 1.5 Comprobar y consultar un array

| Linux | FreeBSD 15 (GEOM) | FreeBSD 15 (ZFS) | Función |
|---|---|---|---|
| `cat /proc/mdstat` | `gmirror status` (o `gstripe status`, `graid3 status`...) | `zpool status` | Estado de los RAID |
| `watch cat /proc/mdstat` | `cmdwatch gmirror status` (paquete `cmdwatch`) | `zpool status 5` (repite cada 5 segundos) | Seguimiento en directo |
| `mdadm --detail /dev/md0` | `gmirror list gm0` | `zpool status -v tank` | Detalle completo |
| `mdadm --examine /dev/sdX1` | `gmirror dump ada1` | `zdb -l /dev/ada1` | Ver los metadatos RAID de un disco concreto |

Estados habituales de `gmirror status`: `COMPLETE` (todo correcto), `DEGRADED` (falta o falla un disco), `SYNCHRONIZING` (reconstruyendo, con porcentaje). En ZFS: `ONLINE`, `DEGRADED`, `FAULTED`, `UNAVAIL`, `RESILVERING`.

### 1.6 Guardar la configuración (equivalente a `mdadm.conf`)

**No hace falta ningún archivo de configuración.** Tanto las clases GEOM como ZFS guardan la información del RAID **dentro de los propios discos** y se detectan solas al arrancar. Solo hay que asegurarse de que el módulo se carga al inicio:

| Tipo de RAID | Línea necesaria |
|---|---|
| gmirror | `geom_mirror_load="YES"` en `/boot/loader.conf` |
| gstripe | `geom_stripe_load="YES"` en `/boot/loader.conf` |
| graid3 | `geom_raid3_load="YES"` en `/boot/loader.conf` |
| gconcat | `geom_concat_load="YES"` en `/boot/loader.conf` |
| ZFS | `zfs_enable="YES"` en `/etc/rc.conf` (y `zfs_load="YES"` en `/boot/loader.conf` si la raíz está en ZFS) |

Los pools ZFS importados se recuerdan en el archivo `/etc/zfs/zpool.cache`, que se gestiona solo.

### 1.7 Gestión del array

| Tarea | GEOM (gmirror) | ZFS |
|---|---|---|
| Formatear y montar | `newfs -U /dev/mirror/gm0` y `mount /dev/mirror/gm0 /datos` | No hace falta formatear: `zfs create tank/datos` |
| Sustituir un disco averiado | `gmirror forget gm0` (olvida el disco que ya no está) + `gmirror insert gm0 /dev/ada3` | `zpool replace tank ada2 ada5` |
| Añadir un disco de repuesto | `gmirror` no tiene repuestos: se añade como disco activo más | `zpool add tank spare ada5` |
| Forzar una reconstrucción | `gmirror rebuild gm0 ada2` | Automática al sustituir (resilver) |
| Convertir un disco solo en espejo | `gmirror insert` | `zpool attach tank ada1 ada2` |
| Quitar un disco de un espejo | `gmirror remove gm0 ada2` | `zpool detach tank ada2` |
| Ampliar la capacidad | Sustituir los discos por otros mayores y luego `gmirror resize` | Añadir otro grupo de discos: `zpool add tank mirror ada5 ada6`, o ampliar un RAID-Z con un disco más: `zpool attach tank raidz1-0 ada6` |
| Poner un disco fuera de servicio | `gmirror deactivate gm0 ada2` | `zpool offline tank ada2` |
| Parar / eliminar | `gmirror stop gm0` / `gmirror destroy gm0` | `zpool export tank` / `zpool destroy tank` |

En ZFS, la propiedad `zpool set autoreplace=on tank` hace que un disco nuevo colocado en la misma ranura sustituya automáticamente al averiado.

### 1.8 Monitorización (equivalente a `mdadm --monitor`)

| Método | Cómo se configura | Función |
|---|---|---|
| Informe diario por correo (`periodic`) | En `/etc/periodic.conf`: `daily_status_gmirror_enable="YES"`, `daily_status_graid3_enable="YES"`, `daily_status_gstripe_enable="YES"`, `daily_status_gconcat_enable="YES"`, `daily_status_zfs_enable="YES"` | Añade el estado de los RAID al informe diario que recibe **root** por correo |
| Eventos en tiempo real (`devd`) | Reglas en `/etc/devd/` o `/usr/local/etc/devd/` | Ejecuta una acción (enviar correo, un script) cuando un disco falla o se desconecta |
| Mensajes del kernel | `/var/log/messages` | Los cambios de estado del RAID se registran ahí (equivale a `--syslog`) |
| `zfsd` (solo ZFS) | `sysrc zfsd_enable="YES"` | Demonio que activa automáticamente los discos de repuesto (`spare`) cuando falla un disco |

Equivalencia de opciones de `--monitor`:

| Opción `mdadm` | Equivalente FreeBSD 15 |
|---|---|
| `--mail=` | El correo de `periodic` va a root. Para que llegue a otra dirección, redirigir root en `/etc/mail/aliases` y ejecutar `newaliases` |
| `--program=` | Una regla de `devd` con `action "/ruta/script"` |
| `--syslog` | Automático (el kernel lo registra) |
| `--daemonise` | `devd` y `zfsd` ya son demonios |
| `--oneshot` | Ejecutar `gmirror status` o `zpool status -x` (este último solo dice si hay algún problema) |

Eventos: los nombres cambian, pero cubren lo mismo que la Tabla 5.2 (disco desconectado, disco con fallo, reconstrucción iniciada/terminada, array degradado).

Igual que en el libro, un RAID 0 no se monitoriza porque no tiene tolerancia a fallos.

---

## 2. Ajuste del acceso a dispositivos de almacenamiento

### 2.1 Equivalente a `hdparm` (discos SATA)

En FreeBSD los discos SATA, SAS, SCSI y USB pasan todos por el subsistema **CAM**, y la herramienta principal es **`camcontrol`**.

| Linux (`hdparm`) | FreeBSD 15 | Función |
|---|---|---|
| `hdparm -I /dev/sda` | `camcontrol identify ada0` | Muestra todos los parámetros del disco |
| `hdparm -W /dev/sda` | `sysctl kern.cam.ada.0.write_cache` | Consulta la caché de escritura (`1` activa, `0` desactivada, `-1` la que traiga el disco) |
| `hdparm -W1 /dev/sda` | `kern.cam.ada.0.write_cache=1` en `/boot/loader.conf` | Activa la caché de escritura |
| `hdparm -t /dev/sda` | `diskinfo -t /dev/ada0` | Prueba de velocidad de lectura del disco |
| `hdparm -T /dev/sda` | No existe | — |
| — | `diskinfo -v /dev/ada0` | Tamaño, sectores, número de serie y otros datos del disco |
| — | `diskinfo -i /dev/ada0` | Prueba de operaciones por segundo (IOPS) |
| `hdparm -S` (reposo) | `camcontrol standby ada0` / `camcontrol idle ada0 -t 600` | Pone el disco en reposo o programa el reposo tras 600 segundos |
| `hdparm -B` (ahorro de energía) | `camcontrol apm ada0 -l 128` | Nivel de gestión de energía del disco |

### 2.2 Equivalente a `sdparm` (dispositivos SCSI)

| Linux (`sdparm`) | FreeBSD 15 | Función |
|---|---|---|
| Consultar datos del dispositivo (VPD) | `camcontrol inquiry da0` | Fabricante, modelo, número de serie |
| — | `camcontrol inquiry da0 -S` | Solo el número de serie |
| Ver/cambiar el comportamiento (páginas de modo) | `camcontrol modepage da0 -m 8` | Muestra la página de caché; con `-e` se edita |
| Parar el giro del disco | `camcontrol stop da0` (`camcontrol start da0` para arrancarlo) | Parar o arrancar el motor |
| `sdparm` / `sg3_utils` | Disponibles también como paquetes | Si se prefieren las herramientas de Linux |

### 2.3 `sysctl` y parámetros del kernel relacionados con almacenamiento

Igual que en el Capítulo 3: `sysctl -a` lista todo y `sysctl -d` explica cada parámetro. Ramas útiles para almacenamiento:

| Rama | Contenido |
|---|---|
| `kern.cam.*` | Parámetros de los discos (reintentos, tiempos de espera, caché) |
| `kern.geom.*` | Parámetros de GEOM (ej. `kern.geom.mirror.*`) |
| `vfs.*` | Parámetros de los sistemas de archivos |
| `vfs.zfs.*` | Parámetros de ZFS (ej. tamaño máximo de la caché ARC) |

No existe `/proc/sys/`: todo se hace con `sysctl`.

### 2.4 NVMe

FreeBSD tiene soporte NVMe completo en el sistema base. Los nombres de dispositivo cambian:

| Elemento | Linux | FreeBSD 15 |
|---|---|---|
| Controladora | `/dev/nvme0` | `/dev/nvme0` |
| Namespace | `/dev/nvme0n1` | `/dev/nvme0ns1` |
| Disco que se usa para particionar y montar | `/dev/nvme0n1` | **`/dev/nda0`** |
| Partición | `/dev/nvme0n1p1` | `/dev/nda0p1` |

Los namespaces son el mismo concepto que en el libro. En FreeBSD cada namespace aparece como un disco `ndaX`, que es el que se usa con `gpart`, `newfs` o `zpool`. (En versiones antiguas el disco se llamaba `nvdX`.)

**Comando `nvmecontrol`:**

| Comando | Función |
|---|---|
| `nvmecontrol devlist` | Lista las controladoras NVMe, sus namespaces y tamaños |
| `nvmecontrol identify nvme0` | Información de la controladora (modelo, firmware, capacidades) |
| `nvmecontrol identify nvme0ns1` | Información de un namespace |
| `nvmecontrol logpage -p 2 nvme0` | Estado de salud (el equivalente a SMART en NVMe) |
| `nvmecontrol ns` | Gestión de namespaces (crear, borrar, asociar) |
| `nvmecontrol format nvme0ns1` | Formateo a bajo nivel (borra todo) |

### 2.5 SMART

Ver Capítulo 4 (`smartctl`, `smartd`, configuración en `/usr/local/etc/smartd.conf`).

### 2.6 Identificadores de dispositivos SCSI/iSCSI

| Concepto | Linux | FreeBSD 15 |
|---|---|---|
| LUN | Igual concepto | Igual concepto. Se ve con `camcontrol devlist -v` (formato bus:target:lun) |
| WWID / WWN | `ls -l /dev/disk/by-id` | `geom disk list` (campo **`lunid`** = WWN, campo `ident` = número de serie) |
| Nombre fijo basado en el número de serie | `/dev/disk/by-id/...` | `/dev/diskid/DISK-numero_serie` (si no está desactivado en `/boot/loader.conf`, ver Capítulo 4) |
| `scsi_id` | Genera un identificador único | `camcontrol inquiry da0 -S` (número de serie) |
| IQN | Igual concepto | Igual concepto y mismo formato (`iqn.año-mes.dominio.invertido:nombre`) |

---

## 3. iSCSI

FreeBSD tiene **iSCSI propio en el sistema base**, tanto servidor (target) como cliente (initiator). No usa `targetcli` ni `open-iscsi`.

| Función | Linux | FreeBSD 15 |
|---|---|---|
| Servidor (target) | `targetcli` + servicio `target` | Demonio **`ctld`** + archivo **`/etc/ctl.conf`** |
| Cliente (initiator) | `iscsiadm` + `iscsid` | Demonio **`iscsid`** + comando **`iscsictl`** + archivo **`/etc/iscsi.conf`** |

### 3.1 Configurar el servidor destino (target): `ctld`

En lugar de un menú interactivo como `targetcli`, se escribe un archivo de configuración.

| Paso | Comando / archivo |
|---|---|
| 1. Crear el disco que se va a compartir (opcional, con ZFS) | `zfs create -V 20G tank/iscsi1` (crea `/dev/zvol/tank/iscsi1`) |
| 2. Escribir la configuración | `/etc/ctl.conf` (ejemplo abajo) |
| 3. Proteger el archivo | `chmod 600 /etc/ctl.conf` (`ctld` no arranca si otros usuarios pueden leerlo) |
| 4. Activar al arranque | `sysrc ctld_enable="YES"` |
| 5. Arrancar | `service ctld start` |
| 6. Aplicar cambios posteriores | `service ctld reload` |

**Ejemplo de `/etc/ctl.conf`:**
```
portal-group pg0 {
    discovery-auth-group no-authentication
    listen 0.0.0.0
}

target iqn.2026-09.com.example.server07:iscsidisk1 {
    auth-group no-authentication
    portal-group pg0

    lun 0 {
        path /dev/zvol/tank/iscsi1
    }
}
```

| Bloque / palabra clave | Equivalente en `targetcli` | Función |
|---|---|---|
| `portal-group` | Portal (`/iscsi/.../tpg1/portals`) | IP y puerto en los que escucha el servidor (puerto 3260 por defecto) |
| `target iqn...` | `cd /iscsi` → `create iqn...` | Define el IQN del target |
| `lun 0 { path ... }` | Backstore + LUN | Disco, partición, zvol o fichero que se comparte |
| `auth-group` | ACL / autenticación | Quién puede conectarse (`no-authentication` o usuario y contraseña CHAP) |

Para usar contraseña se define un grupo de autenticación:
```
auth-group ag0 {
    chap usuario1 contraseña_larga_de_12_o_mas
}
```
y dentro del target se pone `auth-group ag0`.

**Comando `ctladm`** (consulta del servidor en funcionamiento):

| Comando | Función |
|---|---|
| `ctladm lunlist` | Lista los LUN que se están ofreciendo |
| `ctladm devlist -v` | Detalle de los dispositivos compartidos |
| `ctladm portlist` | Puertos y targets activos |
| `ctladm islist` | Sesiones iSCSI de clientes conectados |

### 3.2 Configurar el cliente iniciador (initiator): `iscsictl`

| Elemento Linux | Equivalente FreeBSD 15 |
|---|---|
| Paquete `open-iscsi` / `iscsi-initiator-utils` | No hace falta (sistema base) |
| Demonio `iscsid` + `/etc/iscsi/iscsid.conf` | Demonio `iscsid`: `sysrc iscsid_enable="YES"` + `service iscsid start` |
| `/etc/iscsi/initiatorname.iscsi` | Opción `initiatorname` en `/etc/iscsi.conf`. Si no se indica, se usa `iqn.1994-09.org.freebsd:nombre_del_equipo` |
| Base de datos en `/var/lib/iscsi/` | No existe: las conexiones permanentes se escriben en `/etc/iscsi.conf` |

**Comandos principales:**

| Linux (`iscsiadm`) | FreeBSD 15 (`iscsictl`) | Función |
|---|---|---|
| `iscsiadm -m discovery -t st -p IP` | `iscsictl -Ad -p IP` | Descubre y conecta a los targets de un servidor |
| `iscsiadm -m node -T IQN -p IP -l` | `iscsictl -A -p IP -t IQN` | Inicia sesión en un target concreto |
| `iscsiadm -m session -P3` | `iscsictl -L` (o `iscsictl -Lv` con detalle) | Muestra las sesiones activas |
| `iscsiadm -m node -u` (logout) | `iscsictl -R -t IQN` | Cierra la sesión con un target |
| — | `iscsictl -Aa` | Conecta todas las sesiones definidas en `/etc/iscsi.conf` |

**Conexión permanente** — en `/etc/iscsi.conf`:
```
disco1 {
    TargetAddress = 192.168.1.10
    TargetName    = iqn.2026-09.com.example.server07:iscsidisk1
}
```
Y en `/etc/rc.conf`:
```
iscsictl_enable="YES"
iscsictl_flags="-Aa"
```

Igual que en Linux, tras conectar el disco aparece como un disco SCSI normal (ej. `/dev/da1`), y se usa con `gpart`, `newfs`, `mount` o `zpool`. En `/etc/fstab` conviene añadir la opción **`late`** para que se monte después de que la red y el iSCSI estén listos:
```
/dev/da1p1    /iscsi    ufs    rw,late    2    2
```

---

## 4. Gestión de volúmenes lógicos (equivalente a LVM)

FreeBSD **no tiene LVM**. Su equivalente es **ZFS**, que agrupa discos en un pool y reparte el espacio en datasets o en volúmenes de bloque. (Existe también `gvinum`, un gestor de volúmenes antiguo, hoy en desuso.)

### 4.1 Conceptos

| LVM (Linux) | ZFS (FreeBSD 15) | Descripción |
|---|---|---|
| PV (Physical Volume) | **vdev** / disco del pool | Los discos o particiones que forman el pool. No hay que "prepararlos" antes (no existe `pvcreate`) |
| VG (Volume Group) | **Pool** (`zpool`) | El conjunto de almacenamiento del que sale el espacio |
| LV (Logical Volume) | **Dataset** (sistema de archivos) o **zvol** (volumen de bloque) | Un dataset ya es un sistema de archivos listo para usar. Un **zvol** es un disco virtual que hay que formatear, como un LV |
| PE / LE (extents) | Bloques de tamaño variable | ZFS reparte el espacio solo; el tamaño de bloque se ajusta con las propiedades `recordsize` (datasets) y `volblocksize` (zvols) |

Relación: igual que en LVM, un disco pertenece a un único pool, y un pool puede tener muchos datasets y zvols. La gran diferencia es que los **datasets no tienen un tamaño fijo**: todos comparten el espacio libre del pool, y se limitan con cuotas si hace falta.

### 4.2 Crear la estructura completa

| Paso LVM | Equivalente ZFS | Comando |
|---|---|---|
| 1. `pvcreate` | No hace falta | — |
| 2. `vgdisplay` (ver lo existente) | `zpool list` | Muestra los pools que ya hay |
| 3. `vgcreate vg00 discos...` | Crear el pool | `zpool create vg00 ada1 ada2 ada3 ada4` |
| 4. `lvcreate -L 2g -n lvol0 vg00` | Crear un zvol (volumen de bloque) | `zfs create -V 2G vg00/lvol0` |
| 4b. (sin equivalente en LVM) | Crear un dataset (sistema de archivos directo) | `zfs create vg00/datos` |

Un zvol aparece como `/dev/zvol/vg00/lvol0` y se usa como cualquier disco: `newfs /dev/zvol/vg00/lvol0` y `mount`. Se usa sobre todo para discos de máquinas virtuales (`bhyve`), iSCSI o swap. Para guardar ficheros normales lo habitual es usar un **dataset**, que no necesita formateo ni tamaño previo.

### 4.3 Interfaz interactiva `lvm`

No existe un equivalente interactivo. Todo se hace con los comandos `zpool` y `zfs`. La ayuda se consulta con `zpool help`, `zfs help` o las páginas de manual (ej. `man zfs-create`).

### 4.4 Comandos de consulta

| LVM | ZFS | Función |
|---|---|---|
| `pvdisplay`, `pvs`, `pvscan` | `zpool status` | Discos que forman cada pool y su estado |
| `vgdisplay`, `vgs` | `zpool list` | Tamaño, espacio usado y libre de cada pool |
| `vgscan` | `zpool import` | Busca pools en los discos que no estén importados |
| `lvdisplay`, `lvs`, `lvscan` | `zfs list` | Datasets y zvols con su espacio |
| — | `zfs list -t volume` | Solo los zvols |
| — | `zfs get all vg00/lvol0` | Todas las propiedades de un dataset o zvol |
| `lvdisplay --maps` | `zpool iostat -v` | Cómo se reparte la actividad entre los discos del pool |

### 4.5 Ampliar un volumen (crecimiento en caliente)

| Paso LVM | Equivalente ZFS |
|---|---|
| 1. `vgextend vg00 /dev/sdn1` | `zpool add vg00 ada5` (añade espacio al pool) |
| 2. `lvextend -L 4g /dev/vg00/lvol0` | **zvol**: `zfs set volsize=4G vg00/lvol0` · **dataset**: no hace falta (usa todo el espacio libre del pool) o se sube su cuota: `zfs set quota=4G vg00/datos` |
| 3. `resize2fs` | Si el zvol tiene UFS: `growfs /dev/zvol/vg00/lvol0`. Un dataset no necesita nada |

Cuidado con `zpool add`: si el pool tiene redundancia (mirror, raidz), hay que añadir un grupo con la **misma redundancia** (ej. `zpool add vg00 mirror ada5 ada6`). Añadir un disco suelto a un pool redundante deja ese disco sin protección.

**Reducir** (equivalente a `lvreduce`): en un dataset basta con bajar la cuota (`zfs set quota=1G vg00/datos`), sin riesgo. Reducir `volsize` de un zvol con datos **los destruye**, igual que `lvreduce`. Quitar discos de un pool solo es posible en algunos casos (`zpool remove`).

### 4.6 Renombrar y eliminar

| LVM | ZFS | Función |
|---|---|---|
| `lvrename antiguo nuevo` | `zfs rename vg00/antiguo vg00/nuevo` | Renombra un dataset o zvol |
| `lvremove` | `zfs destroy vg00/lvol0` | Elimina un dataset o zvol (y `zfs destroy -r` para incluir sus snapshots) |
| `vgcfgbackup` | No hace falta: los metadatos están repetidos en todos los discos del pool | — |
| `vgcfgrestore` | `zpool import` | Vuelve a leer el pool desde los discos |
| — | `zpool export vg00` | Desconecta el pool (para mover los discos a otro equipo, donde se usa `zpool import`) |
| — | `zpool history vg00` | Historial de todos los comandos ejecutados sobre el pool |

### 4.7 Snapshots

Los snapshots de ZFS son también **copy-on-write**, como los de LVM, pero más cómodos: no hay que reservar espacio para ellos al crearlos y se pueden tener miles.

| LVM | ZFS | Función |
|---|---|---|
| `lvcreate -s -L 1g -n snap /dev/vg00/lvol0` | `zfs snapshot vg00/datos@snap` | Crea un snapshot (instantáneo) |
| Snapshot escribible y montable | `zfs clone vg00/datos@snap vg00/prueba` | Crea una copia **escribible** a partir del snapshot (los snapshots de ZFS son de solo lectura) |
| Montar el snapshot para leer | Carpeta `.zfs/snapshot/snap/` dentro del dataset | Acceso directo a los ficheros del snapshot |
| `lvconvert --merge` (volver atrás) | `zfs rollback vg00/datos@snap` | Devuelve el dataset al estado del snapshot |
| `lvremove` del snapshot | `zfs destroy vg00/datos@snap` | Borra el snapshot |

Igual que dice el libro, **un snapshot no sustituye a un backup** (está en los mismos discos). Para copias de verdad se usa `zfs send` hacia otro equipo (Capítulo 2).

Con UFS también se pueden hacer snapshots (`mksnap_ffs`, Capítulo 2), aunque son más limitados.

### 4.8 El Device Mapper → GEOM

| Linux | FreeBSD 15 | Función |
|---|---|---|
| Device Mapper (base de LVM y RAID) | **GEOM** (base de gmirror, gstripe, geli, gpart...) | Capa del kernel que transforma los dispositivos de bloque |
| `dmsetup info` | `geom -t` | Muestra el árbol de capas GEOM (qué va encima de qué) |
| `dmsetup info /dev/vg00/lvol0` | `geom mirror list gm0` / `geom disk list ada0` | Información de un dispositivo concreto |
| — | `gstat` | Actividad de cada capa GEOM en tiempo real |
| `/dev/mapper/vg00-lvol0` | `/dev/zvol/vg00/lvol0` (ZFS), `/dev/mirror/gm0`, `/dev/stripe/st0`, `/dev/ada1p1.eli`... | Nombre del dispositivo resultante, para usar en `/etc/fstab` |

Ejemplo de `/etc/fstab` con dispositivos de capas GEOM:
```
/dev/mirror/gm0p1          /datos    ufs    rw    2    2
/dev/zvol/vg00/lvol0       /vol0     ufs    rw    2    2
```

---

## 5. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 5) | Equivalente en FreeBSD 15 |
|---|---|
| Driver `md` / `mdadm` | Clases GEOM (`gmirror`, `gstripe`, `graid3`, `gconcat`, `graid`) y ZFS |
| RAID 0 / 1 / 10 | `gstripe` / `gmirror` / apilados, o ZFS (stripe / `mirror` / varios `mirror`) |
| RAID 5 / 6 | ZFS `raidz1` / `raidz2` |
| `mdadm --create` | `gmirror label` / `zpool create` |
| `/proc/mdstat` | `gmirror status` / `zpool status` |
| `mdadm --detail` / `--examine` | `gmirror list` / `gmirror dump` / `zdb -l` |
| `/etc/mdadm.conf` | No hace falta (metadatos en los discos) + `geom_*_load="YES"` |
| `mdadm --manage --add` | `gmirror insert` / `zpool replace`, `zpool add spare` |
| `mdadm --monitor` | `periodic` (informe diario), `devd`, `zfsd` |
| `hdparm -I` / `-t` | `camcontrol identify` / `diskinfo -t` |
| `sdparm` | `camcontrol inquiry` / `camcontrol modepage` |
| `/dev/nvme0n1p1` | `/dev/nda0p1` (namespace: `/dev/nvme0ns1`) |
| `nvme-cli` | `nvmecontrol` |
| `/dev/disk/by-id` (WWN) | `geom disk list` (`lunid`) y `/dev/diskid/` |
| `targetcli` | `ctld` + `/etc/ctl.conf` (consulta con `ctladm`) |
| `iscsiadm` + `open-iscsi` | `iscsid` + `iscsictl` + `/etc/iscsi.conf` |
| LVM: PV / VG / LV | ZFS: disco (vdev) / pool / dataset o zvol |
| `pvcreate` | No hace falta |
| `vgcreate` | `zpool create` |
| `lvcreate` | `zfs create -V` (zvol) o `zfs create` (dataset) |
| `pvs` / `vgs` / `lvs` | `zpool status` / `zpool list` / `zfs list` |
| `vgextend` + `lvextend` | `zpool add` + `zfs set volsize` / `quota` |
| `lvrename` / `lvremove` | `zfs rename` / `zfs destroy` |
| `vgcfgbackup` / `vgcfgrestore` | No hace falta / `zpool import` |
| Snapshot de LVM | `zfs snapshot` (+ `zfs clone` para escribir) |
| Device Mapper / `dmsetup` | GEOM / `geom -t`, `gstat` |
| `/dev/mapper/VG-LV` | `/dev/zvol/pool/volumen` |

---

*Documento elaborado como contrapartida en FreeBSD 15 al Capítulo 5 ("Administering Advanced Storage Devices") del libro LPIC-2. Fuentes: FreeBSD Handbook (capítulos "GEOM: Modular Disk Transformation Framework", "The Z File System (ZFS)" y la sección "iSCSI Initiator and Target Configuration") y páginas de manual de FreeBSD: geom(8), gmirror(8), gstripe(8), graid3(8), gconcat(8), graid(8), zpool(8), zfs(8), zfsd(8), camcontrol(8), diskinfo(8), nvmecontrol(8), ctld(8), ctl.conf(5), ctladm(8), iscsid(8), iscsictl(8), iscsi.conf(5), periodic.conf(5).*
