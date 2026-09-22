# LPIC-2 · Capítulo 4: Managing the Filesystem
### Recopilación de comandos y archivos de configuración

Objetivos del examen cubiertos: 203.1 (Operating the Linux filesystem), 203.2 (Maintaining a Linux filesystem), 203.3 (Creating and configuring filesystem options)

---

## 1. Conceptos básicos de sistemas de archivos

- **Partición/volumen**: una partición divide lógicamente un dispositivo de almacenamiento; un volumen agrupa varias particiones (gestión de volúmenes lógicos, ver Capítulo 5).
- **Herramientas de particionado**: `parted`, `fdisk`, `gdisk` (vistas en LPIC-1).
- **Formateo (alto nivel)**: crea las estructuras del sistema de archivos sobre la partición/volumen (metadatos, tabla de inodos, journal, etc.). Se hace con `mkfs`.
- **Inodo**: número único asignado a cada fichero al crearse; guarda sus metadatos (permisos, propietario, punteros a los bloques de datos). El nombre del fichero **no** se guarda en el inodo, sino en la entrada del directorio (un directorio es en sí un fichero que contiene una tabla nombre↔inodo). Excepción: un enlace duro (hard link) comparte el mismo número de inodo que el fichero original.

---

## 2. Tipos de sistemas de archivos

### 2.1 Nativos de Linux (Tabla 4.1)

| Nombre | Tamaño máx. de fichero | Tamaño máx. de FS | Integridad | Notas |
|---|---|---|---|---|
| `ext2` | 2 TiB | 16 TiB | Sin journaling | Uno de los originales, en desuso |
| `ext3` | 2 TiB | 16 TiB | Journaling | Mejora de ext2 con journal |
| `ext4` | 16 TiB | 1 EiB | Journaling | Mejora de ext3, mayor tamaño y rendimiento |
| `reiserFS` | 1 EiB | 16 TiB | Journaling | Anterior a ext3; Reiser4 no se integró en el kernel |
| `btrfs` | 16 EiB | 16 EiB | COW (copy-on-write) | Soporta RAID y gestión de volúmenes integrada, snapshots, compresión |

**Journaling vs COW**: el journaling registra los cambios pendientes en un log (journal) antes de aplicarlos, para poder recuperarlos tras un fallo. COW (usado por Btrfs) nunca sobrescribe datos en el sitio: escribe los cambios en un lugar nuevo y solo después actualiza los punteros.

### 2.2 No nativos (Tabla 4.2)

| Nombre | Tamaño máx. fichero | Tamaño máx. FS | Journaling | Notas |
|---|---|---|---|---|
| `ntfs` | 2 TiB | 256 TiB | Sí | De Microsoft; lectura soportada, escritura puede requerir software adicional |
| `vfat` | 4 GiB | 2 TiB | No | FAT32, típico en memorias USB |
| `XFS` | 8 EiB | 8 EiB | Sí | De Silicon Graphics, alto rendimiento |
| `ZFS` | 16 EiB | 256 ZiB | COW | De Sun/Oracle, comparable a Btrfs |

### 2.3 Consultar los sistemas de archivos soportados

| Archivo/Comando | Función |
|---|---|
| `/proc/filesystems` | Lista los tipos de sistema de archivos soportados por el kernel en ese momento |

---

## 3. Creación de un sistema de archivos (formateo)

| Comando | Función |
|---|---|
| `mkfs -t fstype dispositivo` | Formatea usando el tipo indicado (`fstype`: ext2, ext3, ext4, xfs, btrfs, etc.) |
| `mkfs.fstype dispositivo` | Forma equivalente, más directa (ej. `mkfs.ext4 /dev/sdb1`) |
| `mke2fs dispositivo` | Crea un sistema ext2 (apunta a `mkfs.ext2`); con `-t fstype` también sirve para ext3/ext4 |

Requiere privilegios de superusuario. Comprobar el tipo tras formatear:

| Comando | Función |
|---|---|
| `parted -l` | Lista particiones y su tipo de sistema de archivos |
| `blkid` | Muestra el tipo, UUID y etiqueta de cada dispositivo de bloque |

---

## 4. Montaje del sistema de archivos (attaching)

### 4.1 Montaje temporal: `mount`

Sintaxis básica: `mount -t fstype dispositivo punto_de_montaje`

**Opciones destacadas de `mount` (Tabla 4.3):**

| Opción | Función |
|---|---|
| `-a` | Monta todos los sistemas listados en `/etc/fstab` |
| `-F` | (con `-a`) monta todos a la vez |
| `-f` | Simula el montaje sin montar realmente |
| `-L label` | Monta por etiqueta |
| `-U uuid` | Monta por UUID |
| `-n` | Monta sin registrar en `/etc/mtab` |
| `-o opts` | Opciones adicionales separadas por comas |
| `-r` | Monta solo lectura |
| `-w` | Monta lectura/escritura |
| `-s` | Ignora opciones no soportadas por el sistema de archivos |
| `-t fstype` | Especifica el tipo |
| `-v` | Modo detallado (verbose) |

Desmontar: `umount dispositivo` o `umount punto_de_montaje`

### 4.2 Montaje persistente: `/etc/fstab`

Cada línea tiene 6 campos (Listing 4.1):

```
dispositivo/UUID/label   punto_de_montaje   tipo   opciones   dump   fsck
```

| Campo | Significado |
|---|---|
| 1. Identificación | Nombre de partición, volumen, `UUID=...`, `Label=...`, recurso NFS, o `swap` |
| 2. Punto de montaje | Ruta absoluta (o `swap` para particiones swap) |
| 3. Tipo de sistema de archivos | ext4, xfs, nfs, swap, etc. |
| 4. Opciones de montaje | Igual que las de `mount -o` |
| 5. Backup (dump) | `0` = no incluir con `dump`, `1` = sí |
| 6. Orden de comprobación (fsck) | vacío o `0` = no comprobar, `1` = prioridad máxima (normalmente solo la raíz `/`), `2` = se comprueba después de las de prioridad 1 |

Comando útil: `mount -a` — monta todo lo que esté en `/etc/fstab` y aún no esté montado (además sirve para comprobar la sintaxis del fichero).

Nota: los sistemas de archivos gestionados por AutoFS **no** deben tener entrada en `/etc/fstab`.

### 4.3 Unidades de montaje de systemd (alternativa a `/etc/fstab`)

| Elemento | Detalle |
|---|---|
| Ubicación | `/etc/systemd/system/` |
| Nombre del fichero | Ruta del punto de montaje sin la `/` inicial, con `/` internas sustituidas por `-`, y extensión `.mount` (ej. `/home/temp/` → `home-temp.mount`) |
| Secciones mínimas | `[Unit]`, `[Mount]` (con `What=`, `Where=`, `Type=`, `Options=`), `[Install]` (con `WantedBy=` o `RequiredBy=`) |
| Activar de forma persistente | `systemctl enable nombre.mount` |

Recomendación del libro: usar `/etc/fstab` salvo que se necesite ajustar algo específico vía unidad de montaje; `systemd` sigue gestionando lo definido en `/etc/fstab` igualmente (lo convierte en unidades nativas al arrancar).

### 4.4 Ver los sistemas de archivos montados

| Comando | Función |
|---|---|
| `mountpoint directorio` | Indica si ese directorio es un punto de montaje |
| `mount` (sin opciones) | Lista los sistemas montados (lee de `/etc/mtab`) |
| `cat /proc/mounts` | Igual, pero más actualizado que `/etc/mtab` (y no se ve afectado por `mount -n`) |
| `findmnt` | Muestra los sistemas montados en formato árbol |
| `lsblk -f` | Muestra dispositivos de bloque con su UUID y etiqueta |
| `e2label dispositivo` | Muestra (o cambia) la etiqueta de un sistema ext2/ext3/ext4 |
| `findfs LABEL=etiqueta` / `findfs UUID=uuid` | Devuelve el dispositivo asociado a una etiqueta o UUID |

---

## 5. UUID de los sistemas de archivos

| Comando | Función |
|---|---|
| `blkid` | Muestra el UUID de cada dispositivo |
| `lsblk -f` | Alternativa para ver UUID y etiquetas |
| `uuidgen` | Genera un nuevo UUID |
| `tune2fs -U nuevo-uuid dispositivo` | Asigna un UUID a un sistema ext2/3/4 **sin montar** |

El UUID es el método preferido para identificar particiones en `/etc/fstab`, especialmente con muchas particiones.

---

## 6. Sistemas de archivos temporales/virtuales

| Elemento | Descripción |
|---|---|
| `/proc` | Sistema de archivos virtual en memoria con información del kernel y procesos (ver también Capítulo 3) |
| Sin tamaño en `ls -l` | Los ficheros de `/proc` no ocupan espacio real en disco |
| Se recrean al arrancar | Los datos se pierden al apagar, ya que residen en memoria |

---

## 7. El sistema de archivos Btrfs (particularidades)

| Comando | Función |
|---|---|
| `btrfs filesystem show` | Muestra información del sistema de archivos, incluyendo UUID de los dispositivos |
| `btrfs filesystem df` | Muestra el uso de espacio |

**Utilidades de ajuste (Tabla 4.8):**

| Utilidad | Función |
|---|---|
| `btrfs balance` | Reequilibra los datos en el sistema de archivos |
| `btrfsconvert` | Convierte entre ext2/3/4 y Btrfs (y viceversa) |
| `btrfstune` | Ajusta atributos y activa/desactiva funciones extendidas |
| `btrfs property set` | Establece propiedades (ej. etiqueta) |

**Utilidades de comprobación/reparación (Tabla 4.11):**

| Utilidad | Función |
|---|---|
| `btrfs check` | Comprueba y opcionalmente repara un sistema desmontado |
| `btrfs get property` | Consulta propiedades |
| `btrfs rescue` | Recupera un sistema de archivos dañado |
| `btrfs restore` | Restaura ficheros desde un sistema dañado (la más potente de las tres de reparación) |
| `btrfs scrub` | Lee todo el disco comprobando la consistencia (puede afectar al rendimiento mientras se ejecuta) |

Btrfs soporta RAID integrado en niveles **0, 1 y 10** (no 5 ni 6, según el libro).

---

## 8. Mantenimiento y ajuste de sistemas de archivos

### 8.1 Extendidos: ext2 / ext3 / ext4 (Tabla 4.6)

| Utilidad | Función |
|---|---|
| `debugfs` | Utilidad interactiva para modificar metadatos |
| `e2label` | Cambia la etiqueta |
| `resize2fs` | Amplía o reduce un sistema **desmontado** |
| `tune2fs` | Ajusta atributos, incluyendo UUID y etiqueta |

### 8.2 XFS (Tabla 4.7)

| Utilidad | Función |
|---|---|
| `xfs_admin` | Ajusta atributos (UUID, etiqueta) |
| `xfs_fsr` | Mejora la disposición (layout) de los ficheros |
| `xfs_growfs` | Amplía el tamaño del sistema de archivos |

### 8.3 Comprobación y reparación

**Extendidos (Tabla 4.9):**

| Utilidad | Función |
|---|---|
| `fsck.fstype` | Comprueba/repara (ej. `fsck.ext4`); usa la carpeta `lost+found` para ficheros recuperados |
| `debugfs` | Extrae datos para moverlos a otra ubicación |
| `dumpe2fs` | Muestra información del sistema de archivos |
| `tune2fs -l` | Muestra los atributos del sistema de archivos |

Nota: `fsck.xfs` y `fsck.btrfs` no hacen nada real (son "stubs"), porque XFS y Btrfs tienen sus propias herramientas de reparación.

**XFS (Tabla 4.10):**

| Utilidad | Función |
|---|---|
| `xfs_check` | Comprueba consistencia sin reparar ("dry run"; ya no incluido en muchas distros) |
| `xfsdump` | Copia de seguridad de datos y atributos |
| `xfs_info` | Muestra información (equivale a `xfs_growfs -n`) |
| `xfs_metadump` | Copia los metadatos a un fichero |
| `xfs_repair` | Comprueba y repara (usar `-n` para simular si no existe `xfs_check`) |
| `xfsrestore` | Restaura desde una copia hecha con `xfsdump` |

### 8.4 Monitorización SMART (Self-Monitoring Analysis and Reporting Technology)

| Elemento | Detalle |
|---|---|
| Paquete | `smartmontools` |
| `smartctl -i dispositivo` | Información básica; indica si el dispositivo soporta y tiene SMART habilitado |
| `smartctl -s on dispositivo` | Activa SMART en el dispositivo |
| `smartctl -H dispositivo` | Resumen de salud (PASSED / FAILED) |
| `smartctl -a dispositivo` | Toda la información disponible |
| `smartctl -t short\|long\|selftest dispositivo` | Lanza una autoprueba |
| `smartctl -l error dispositivo` | Muestra el registro de errores del dispositivo |
| `smartd` | Demonio que programa comprobaciones automáticas (cada 30 min por defecto) |
| `/etc/smartd.conf` o `/etc/smartmontools/smartd.conf` | Configuración de `smartd` |
| `DEVICESCAN` | Palabra clave de configuración para analizar automáticamente todos los dispositivos SMART |

Nota: un dispositivo SMART "no es inteligente": no garantiza predecir un fallo, aunque ayuda a detectar señales de alerta.

---

## 9. Espacio de intercambio (swap)

| Comando | Función |
|---|---|
| `mkswap dispositivo` | Prepara una partición (o fichero) como espacio swap |
| `swapon dispositivo` | Activa el espacio swap |
| `swapon -s` | Muestra estadísticas del swap activo |
| `swapoff dispositivo` | Desactiva el espacio swap |
| `swapon -p prioridad dispositivo` | Define la prioridad de uso (0 a 32767; mayor número = mayor prioridad); requiere desactivar primero con `swapoff` |
| `cat /proc/swaps` | Alternativa para ver el estado del swap |
| `free -m` | Muestra memoria y swap en MB |

El swap puede residir en una partición, en un volumen lógico (LVM, ver Capítulo 5) o incluso en un fichero creado con `dd`.

---

## 10. AutoFS (montaje automático de recursos de red)

| Elemento | Detalle |
|---|---|
| Fichero de configuración del servicio | `/etc/sysconfig/autofs` (Red Hat) o `/etc/default/autofs` (Debian) |
| `/etc/auto.master` | Mapa maestro: activa el montaje automático o apunta a otros ficheros de mapas |
| Direct map | Apunta a `/etc/auto.direct`; monta con rutas absolutas |
| Indirect map | Apunta a `/etc/auto.directory` (donde `directory` coincide con el punto de montaje); monta con rutas relativas bajo ese directorio |
| `DirectoryMode` | Permisos para los puntos de montaje creados automáticamente (por defecto `0755`) |
| `TimeOutIdleSec` | Tiempo de inactividad antes de desmontar automáticamente |

---

## 11. Medios ópticos: ISO9660 y UDF

| Comando | Función |
|---|---|
| `mkisofs` | Crea una imagen ISO (sustituido en distros recientes por `genisoimage`) |
| `genisoimage` | Alternativa moderna a `mkisofs` |
| `cdrecord` | Graba una imagen ISO en un disco óptico |
| `file` | Permite comprobar si una imagen ISO es arrancable |

Extensiones del estándar ISO9660: **El Torito** (discos arrancables), **Joliet** (nombres largos, compatibilidad con Windows), **Rock Ridge** (metadatos estilo Unix). UDF (Universal Disk Format) es un estándar aparte, usado sobre todo en DVDs.

---

## 12. Sistemas de archivos cifrados

| Tipo | Herramienta | Características |
|---|---|---|
| `dm-crypt` (básico) | `cryptsetup` | Usa Device Mapper; una sola clave (hash de contraseña sin salt); sin metadatos en el volumen; no recomendado salvo buen conocimiento de cifrado |
| **LUKS** (dm-crypt mejorado) | `cryptsetup` | Clave maestra + varias claves de usuario; guarda metadatos en el volumen; método preferido. En los objetivos del examen puede aparecer como "dm-crypt/LUKS" |
| `eCryptfs` | Comando `mount` (paquete `ecryptfs-utils`) | Sistema en capa (pseudo-filesystem) sobre otro ya existente; cifra fichero a fichero; requiere **dos** montajes: uno del sistema de archivos base y otro para aplicar la capa eCryptfs encima; ejemplo: `mount -t ext4 /dev/sdd1 /home` seguido de `mount -t eCryptfs /home /home` |

---

## 13. Resumen de diferencias Debian vs Red Hat en este capítulo

| Aspecto | Debian | Red Hat |
|---|---|---|
| Fichero de configuración del servicio AutoFS | `/etc/default/autofs` | `/etc/sysconfig/autofs` |
| Fichero de configuración de `smartd` | Puede variar: `/etc/smartmontools/smartd.conf` | Puede variar: `/etc/smartd.conf` |
| Resto del capítulo (`mkfs`, `mount`, `/etc/fstab`, herramientas de ext/XFS/Btrfs, cifrado, swap) | Igual | Igual |

---

*Documento generado a partir del Capítulo 4 ("Managing the Filesystem") del libro LPIC-2: Linux Professional Institute Certification Study Guide (Bresnahan & Blum).*
