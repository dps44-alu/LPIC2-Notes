# LPIC-2 · Capítulo 5: Administering Advanced Storage Devices
### Recopilación de comandos y archivos de configuración

Objetivos del examen cubiertos: 204.1 (Configuring RAID), 204.2 (Adjusting Storage Device Access), 204.3 (Logical Volume Manager)

---

## 1. RAID (Redundant Array of Independent Disks)

### 1.1 Niveles de RAID

| Nivel | Nombre alternativo | Mínimo de discos | Tolerancia a fallos | Notas |
|---|---|---|---|---|
| RAID 0 | Disk striping | 2 | No | Reparte los datos entre discos; máxima velocidad, sin protección |
| RAID 1 | Disk mirroring | 2 (en múltiplos de 2) | Sí | Copia idéntica en cada disco; caro (necesitas el doble de capacidad) |
| RAID 2 | — | — | — | En desuso, corrección de errores ya gestionada por los propios discos |
| RAID 3 | — | 3 | Parcial | Forma de RAID 0 con disco de paridad dedicado; sin regeneración si falla el disco de paridad |
| RAID 4 | — | 3 | Parcial | Como RAID 3 pero trabaja por bloques en vez de bytes (más rápido); mismo problema con el disco de paridad |
| RAID 5 | Disk striping with parity | 3 | Sí | Paridad repartida entre discos; si falla un segundo disco durante la reconstrucción, se pierde el array |
| RAID 6 | Disk striping with double parity | 4 | Sí | Paridad duplicada en dos discos distintos; soporta el fallo de 2 discos; escritura algo más lenta |
| RAID 10 | Disk mirroring and striping (RAID 1+0) | 4 | Sí | Combina RAID 1 y RAID 0; reconstrucción muy rápida, pero caro |

Nota: RAID 2, 3 y 4 están prácticamente en desuso hoy en día; el examen se centra en 0, 1, 4, 5, 6 y 10.

### 1.2 Implementación de RAID software: `mdadm`

El controlador del kernel usado es **md** (Multiple Devices), que soporta RAID 0, 1, 10, 4, 5 y 6.

**Comprobaciones previas:**

| Comando | Función |
|---|---|
| `uname -r` | Comprueba la versión del kernel (2.6+ soporta todos los niveles) |
| `ls /proc/mdstat` | Confirma que el sistema tiene soporte RAID |
| `modprobe raid6` (o `raid0`, `raid1`, etc.) | Intenta cargar el módulo del nivel RAID deseado |
| `cat /proc/mdstat` | Muestra la línea `Personalities` con los niveles RAID soportados actualmente |
| `dpkg -s mdadm` / `rpm -qa \| grep mdadm` | Comprueba si `mdadm` está instalado (Debian / Red Hat) |

**Preparar los discos:** particionar con `fdisk` o `parted`; código de tipo de partición recomendado `fd` (0xFD00) con GPT, o `da` (0xDA) con MBR y superblock md v1.0+.

### 1.3 Modos del comando `mdadm` (Tabla 5.1)

| Modo (opción larga) | Opción corta | Función |
|---|---|---|
| `--assemble` | `-A` | Ensambla en un array activo unos discos ya creados anteriormente |
| `--autodetect` | — | Pide al kernel activar los arrays auto-detectados |
| `--build` | `-B` | Construye un array a partir de discos sin superblock |
| `--create` | `-C` | Crea un array nuevo (añade superblocks y los ensambla) |
| `--follow` / `--monitor` | `-F` | Monitoriza los dispositivos md y actúa ante eventos (inútil en RAID 0) |
| `--grow` | `-G` | Amplía, reduce o remodela un array |
| `--incremental` | `-I` | Añade un disco al array, activándolo si con eso ya es posible |
| `--manage` | — | Gestiona miembros del array (ej. añadir un disco de repuesto) |
| `--misc` | — | Operaciones varias (ej. borrar el superblock de un disco) |

### 1.4 Crear un array

```
mdadm -C /dev/md0 -l 6 -n 4 /dev/sdf1 /dev/sdg1 /dev/sdh1 /dev/sdi1
```
(`-C`/`--create`, `-l`/`--level` nivel RAID, `-n`/`--raid-devices` número de discos activos)

Otras opciones útiles de creación: `--chunk`/`-c` (tamaño de fragmento en KiB, por defecto 512), `--spare-devices`/`-x` (discos de repuesto), `--force`/`-f`.

Ayuda contextual por modo: `mdadm --create --help`, `mdadm --grow --help`, etc.

### 1.5 Comprobar y consultar un array

| Comando | Función |
|---|---|
| `cat /proc/mdstat` | Estado actual de todos los arrays RAID activos |
| `watch cat /proc/mdstat` | Monitorización en directo, refrescando cada pocos segundos |
| `mdadm --misc --detail /dev/md0` | Detalle completo de un array |
| `mdadm --misc --examine /dev/sdX1` | Examina si una partición concreta pertenece a un array |

### 1.6 Guardar la configuración: `mdadm.conf`

| Elemento | Detalle |
|---|---|
| Ubicación | `/etc/mdadm.conf` o `/etc/mdadm/mdadm.conf`, según distribución (comprobar con `man mdadm.conf`) |
| No se crea automáticamente | Hay que generarlo a mano tras crear el array |
| Comando recomendado | `mdadm --verbose --detail --scan /dev/md0 >> /etc/mdadm.conf` |
| Palabra clave de cada entrada | `ARRAY` |

### 1.7 Gestión del array

| Tarea | Comando |
|---|---|
| Formatear y montar (como cualquier partición) | `mkfs -t ext4 /dev/md0` seguido de `mount -t ext4 /dev/md0 punto` |
| Añadir un disco de repuesto | `mdadm --manage /dev/md0 --add /dev/sdX1` |
| Ampliar/remodelar (tamaño, forma, nivel) | `mdadm --grow ...` (operación larga y delicada; usar `--backup-file` en otro disco) |
| Parar el array (antes de eliminarlo) | `mdadm --manage /dev/md0 --stop` (o similar en modo manage) |

### 1.8 Monitorización: `mdadm --monitor`

Sintaxis: `mdadm --monitor opciones dispositivos` (equivalente a `--follow`)

**Opciones destacadas:**

| Opción | Función |
|---|---|
| `--mail=` / `-m` | Dirección de correo para alertas |
| `--program=` / `-p` | Programa a ejecutar ante un evento |
| `--syslog` / `-y` | Envía las alertas al syslog |
| `--delay=` / `-d` | Segundos entre comprobaciones (por defecto 60) |
| `--scan` / `-s` | Busca dirección de correo/programa en `mdadm.conf` (palabras clave `MAILADDR` y `PROGRAM`) |
| `--daemonise` / `-f` | Ejecuta en segundo plano como demonio |
| `--oneshot` / `-1` | Comprueba una vez y sale |

**Eventos que puede reportar (Tabla 5.2):** `DeviceDisappeared`, `RebuildStarted`, `RebuildNN`, `RebuildFinished`, `Fail`, `FailSpare`, `SpareActive`, `NewArray`, `DegradedArray`, `MoveSpare`, `SparesMissing`, `TestMessage`.

Nota: RAID 0 nunca se monitoriza (sin tolerancia a fallos, no aplica).

---

## 2. Ajuste del acceso a dispositivos de almacenamiento

### 2.1 `hdparm` (discos PATA/SATA)

| Comando | Función |
|---|---|
| `hdparm -I dispositivo` | Muestra todos los parámetros del disco |
| `hdparm -W dispositivo` | Consulta/activa el write-caching |
| `hdparm -t dispositivo` | Test de rendimiento de lectura (con el disco inactivo) |
| `hdparm -T dispositivo` | Test de rendimiento de la caché |

### 2.2 `sdparm` (dispositivos SCSI)

Consulta las tablas VPD (Vital Product Data) de un dispositivo SCSI: número de serie, número de parte, etc. También permite controlar comportamiento (ej. parar el giro del disco, ajustar el write-back cache).

### 2.3 `sysctl` y parámetros del kernel relacionados con almacenamiento

| Comando | Función |
|---|---|
| `sysctl -a` | Lista todos los parámetros modificables del kernel |
| `/proc/sys/` | Carpeta con los ficheros equivalentes a cada parámetro |

(Ver también Capítulo 3 para el uso general de `sysctl`.)

### 2.4 NVMe (Non-Volatile Memory Express)

| Elemento | Detalle |
|---|---|
| Qué es | Estándar de interfaz y conjunto de comandos para SSD conectados por PCI Express |
| Soporte en el kernel | Desde la versión 3.3 |
| Ficheros de dispositivo | `/dev/nvme*` |
| Estructura | Usa "namespaces" como capa adicional sobre la que luego van las particiones |
| Ejemplo de nomenclatura | `/dev/nvme2n4p1` = 3ª unidad NVMe, namespace 4, partición 1 |

### 2.5 SMART

Ver Capítulo 4 (`smartctl`, `smartd`) — aplica igualmente a este capítulo de almacenamiento avanzado.

### 2.6 Identificadores de dispositivos SCSI/iSCSI

| Elemento | Detalle |
|---|---|
| LUN (Logical Unit Number) | Identifica un dispositivo SCSI lógico concreto en el sistema destino |
| WWID / WWN (World Wide Identifier / World Wide Name) | Identificador hexadecimal único grabado por el fabricante; no cambia aunque se añadan discos nuevos; se ve con `ls -l /dev/disk/by-id` |
| `scsi_id` | Genera un identificador único para un dispositivo SCSI |
| IQN (iSCSI Qualified Name) | Dirección única que identifica tanto al servidor destino (target) como al disco iSCSI que ofrece |

---

## 3. iSCSI

### 3.1 Configurar el servidor destino (target): `targetcli`

| Paso | Comando (dentro de `targetcli`) |
|---|---|
| Activar el subsistema al arranque | `systemctl enable target` |
| Entrar en la utilidad | `targetcli` |
| Crear un backstore (apunta a un disco/partición/fichero) | `cd /backstores/block` → `create iscsidisk1 dev=/dev/sde` |
| Crear el IQN del target | `cd /iscsi` → `create iqn.2016-02.com.example.server07:iscsidisk1` |
| Ver el resultado | `ls` |

Requiere el paquete `targetcli`. En distros antiguas se usaba `tgtd`/`tgtadm` en su lugar.

### 3.2 Configurar el cliente iniciador (initiator): `iscsiadm`

| Elemento | Detalle |
|---|---|
| Paquete | `iscsi-initiator-utils` (Red Hat) / `open-iscsi` (Debian) |
| Demonio necesario | `iscsid` (configurado en `/etc/iscsi/iscsid.conf`) |
| Fichero con el nombre del iniciador | `/etc/iscsi/initiatorname.iscsi` |
| Base de datos de descubrimiento | `/var/lib/iscsi/send_targets` (IPs) y `/var/lib/iscsi/nodes` (IQNs) |

**Comandos principales:**

| Comando | Función |
|---|---|
| `iscsiadm -m discovery -t st -p IP_target` | Descubre los discos iSCSI disponibles en el servidor destino |
| `iscsiadm -m node -T IQN -p IP_target -l` | Inicia sesión (login) contra un target concreto |
| `iscsiadm -m session -P3` | Muestra información detallada de la sesión activa |
| `systemctl enable iscsid` | Activa el demonio `iscsid` al arranque |

Tras el login, el disco iSCSI aparece como un dispositivo SCSI normal (ej. `/dev/sdj`), usable con `lsblk`, `mkfs`, `mount`, etc., igual que un disco local. Debe añadirse a `/etc/fstab` del iniciador para montarlo de forma persistente.

---

## 4. Gestión de volúmenes lógicos (LVM)

### 4.1 Conceptos

| Elemento | Comando de creación | Descripción |
|---|---|---|
| PV (Physical Volume) | `pvcreate` | Marca una partición o disco entero para uso por LVM |
| VG (Volume Group) | `vgcreate` | Agrupa uno o más PV en un "pool" de almacenamiento |
| LV (Logical Volume) | `lvcreate` | Fragmento de almacenamiento extraído de un VG; se formatea y monta como una partición normal |
| PE (Physical Extent) | — | Bloque más pequeño asignable en un PV (4 MiB por defecto; ajustable con `-s`/`--physicalextentsize` al crear el VG) |
| LE (Logical Extent) | — | Bloque de un LV, mapeado a los PE del VG |

Relación: una partición (PV) pertenece a un único VG; un VG puede tener varios LV; un LV pertenece a un único VG.

### 4.2 Crear la estructura completa

| Paso | Comando |
|---|---|
| 1. Crear los PV | `pvcreate /dev/sdj1` (repetir para cada partición) |
| 2. Ver PVs/VGs existentes antes de nombrar uno nuevo | `vgdisplay` |
| 3. Crear el VG | `vgcreate vg00 /dev/sdj1 /dev/sdk1 /dev/sdl1 /dev/sdm1` |
| 4. Crear el LV | `lvcreate -L 2g vg00` (o `-n nombre` para darle nombre) |

### 4.3 Interfaz interactiva `lvm`

Se entra escribiendo `lvm` (paquete `lvm2`); dentro se dispone de todos los comandos `pv*`, `vg*`, `lv*` también accesibles directamente desde la shell normal. Comando `help` dentro de la utilidad para listarlos todos; `quit` para salir.

### 4.4 Comandos de consulta

| Comando | Función |
|---|---|
| `pvdisplay`, `pvs`, `pvscan` | Información sobre los PV |
| `vgdisplay`, `vgs`, `vgscan` | Información sobre los VG |
| `lvdisplay`, `lvs`, `lvscan` | Información sobre los LV |
| `lvdisplay --maps` | Muestra qué extents físicos (PE) respaldan a cada extent lógico (LE) |

### 4.5 Ampliar un LV (crecimiento en caliente)

| Paso | Comando |
|---|---|
| 1. Añadir un PV nuevo al VG (si hace falta más espacio) | `vgextend vg00 /dev/sdn1` |
| 2. Ampliar el LV | `lvextend -L 4g /dev/vg00/lvol0` |
| 3. Ampliar el sistema de archivos dentro del LV | `resize2fs` (ext) o herramienta equivalente del FS (ver Capítulo 4) |

Reducir un LV: `lvreduce` (¡puede destruir datos, usar con precaución!).

### 4.6 Renombrar y eliminar

| Tarea | Comando |
|---|---|
| Renombrar un LV | `lvrename nombre_antiguo nombre_nuevo` |
| Eliminar un LV (tras desmontarlo) | `lvremove nombre_lv` |
| Backup de la metadata de un VG | `vgcfgbackup` |
| Restaurar la metadata de un VG | `vgcfgrestore` |

### 4.7 Snapshots de LV (LVM snapshot)

- Es un snapshot **copy-on-write (COW)**: al crearse, solo se copian metadatos (es instantáneo); los datos originales solo se copian al área del snapshot en el momento en que se modifican en el LV original.
- Un snapshot **no sustituye a un backup**, pero sirve para hacer copias en caliente sin interrumpir el servicio, o para probar cambios sobre datos de producción sin afectar al LV original.
- Son legibles y escribibles, y se pueden montar igual que cualquier LV.

### 4.8 El Device Mapper

| Elemento | Detalle |
|---|---|
| Qué es | Driver del kernel que mapea bloques de almacenamiento físico a bloques virtuales; es la base tanto de LVM como de RAID |
| `dmsetup info` | Lista los dispositivos mapeados |
| `dmsetup info /dev/vg00/lvol0` | Información de un LV concreto |
| Nombre alternativo de un LV | `/dev/mapper/VG-LV` (ej. `/dev/mapper/vg00-lvol0`) |

Buena práctica: usar los nombres `/dev/mapper/...` en `/etc/fstab` para el montaje persistente de LVs, ya que algunas distribuciones lo requieren para invocar LVM correctamente al arrancar.

---

## 5. Resumen de diferencias Debian vs Red Hat en este capítulo

| Aspecto | Debian | Red Hat |
|---|---|---|
| Comprobar si `mdadm` está instalado | `dpkg -s mdadm` | `rpm -qa \| grep mdadm` |
| Ubicación de `mdadm.conf` | Puede variar (comprobar con `man mdadm.conf`) | Puede variar (comprobar con `man mdadm.conf`) |
| Paquete del iniciador iSCSI | `open-iscsi` | `iscsi-initiator-utils` |
| Resto del capítulo (niveles RAID, `mdadm`, LVM, `hdparm`, `targetcli`) | Igual | Igual |

---

*Documento generado a partir del Capítulo 5 ("Administering Advanced Storage Devices") del libro LPIC-2: Linux Professional Institute Certification Study Guide (Bresnahan & Blum).*
