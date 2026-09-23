# LPIC-2 · Capítulo 4: Managing the Filesystem
### Equivalencias en Windows Server 2025

Este documento recoge, para cada concepto visto en el Capítulo 4 del libro LPIC-2 (tipos de sistemas de archivos, formateo, montaje, mantenimiento, swap, AutoFS, medios ópticos y cifrado), cuál es su equivalente en **Windows Server 2025**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo: Windows Server 2025 tiene **dos sistemas de archivos principales**:
- **NTFS**: el clásico de Windows, con permisos, cuotas, compresión y cifrado. Es obligatorio para el disco del sistema. Su papel es parecido al de ext4 en Linux.
- **ReFS** (Resilient File System): pensado para grandes volúmenes de datos y virtualización. Comprueba la integridad de los datos, se autorrepara y, junto con **Espacios de almacenamiento** (Capítulo 5), ofrece RAID. Su papel es parecido al de Btrfs o ZFS.

Además, la forma de "montar" es distinta: en Windows los volúmenes se asignan a una **letra de unidad** (`D:`) o a una **carpeta vacía** de un volumen NTFS, y el sistema los monta **automáticamente** al arrancar. No existe un archivo `/etc/fstab`.

La gestión de discos se hace con tres herramientas: la consola gráfica **Administración de discos** (`diskmgmt.msc`), la herramienta de texto **`diskpart`** y los cmdlets del módulo **Storage** de PowerShell (`Get-Disk`, `New-Partition`, `Format-Volume`...). Todos los comandos se ejecutan en una consola **como Administrador**.

---

## 1. Conceptos básicos de sistemas de archivos

Los conceptos del libro (partición, volumen, formateo) son **los mismos**. Lo que cambia es el equivalente del **inodo**:

| Concepto Linux | Equivalente en NTFS |
|---|---|
| Inodo | **Registro de la MFT** (Master File Table, "tabla maestra de archivos"). Cada archivo tiene al menos un registro con sus metadatos: tamaño, fechas, permisos (a través de un identificador de seguridad) y dónde están sus datos |
| Número de inodo | **Número de referencia del archivo** (File ID). Se ve con `fsutil file queryfileid C:\ruta\archivo` |
| El nombre no está en el inodo | En NTFS **el nombre sí se guarda dentro del registro de la MFT**, además de en el índice de la carpeta |
| Enlace duro (hard link) | Igual: varios nombres para el mismo registro de la MFT. Se crea con `mklink /H nuevo existente` o `New-Item -ItemType HardLink` |
| Enlace simbólico | `mklink archivo destino`, `mklink /D carpeta destino` o `New-Item -ItemType SymbolicLink` |
| — | **Unión** (junction): enlace de carpeta a carpeta local, más antiguo. `mklink /J carpeta destino` |

Ver los enlaces duros de un archivo: `fsutil hardlink list C:\ruta\archivo`.

**Herramientas de particionado:**

| Linux | Windows Server 2025 | Función |
|---|---|---|
| `fdisk`, `gdisk`, `parted` | `diskpart` | Herramienta de texto interactiva para discos, particiones y volúmenes |
| — | Módulo Storage de PowerShell | Cmdlets para hacer lo mismo desde scripts |
| — | `diskmgmt.msc` | Administración de discos (gráfica, solo con escritorio o en remoto) |

**Comandos básicos** (ejemplo con un disco nuevo, el número 1):

| `diskpart` | PowerShell | Función |
|---|---|---|
| `list disk` | `Get-Disk` | Lista los discos |
| `select disk 1` → `online disk` | `Set-Disk -Number 1 -IsOffline $false` | Pone el disco en línea (ver nota más abajo) |
| `convert gpt` | `Initialize-Disk -Number 1 -PartitionStyle GPT` | Crea una tabla de particiones GPT en un disco vacío |
| `create partition primary size=20480` | `New-Partition -DiskNumber 1 -Size 20GB` | Crea una partición de 20 GB |
| `create partition primary` | `New-Partition -DiskNumber 1 -UseMaximumSize` | Crea una partición con todo el espacio libre |
| `list partition` | `Get-Partition -DiskNumber 1` | Lista las particiones del disco |
| `delete partition` | `Remove-Partition -DiskNumber 1 -PartitionNumber 2` | Borra una partición |
| `clean` | `Clear-Disk -Number 1 -RemoveData` | Borra la tabla de particiones completa |
| `extend` | `Resize-Partition` | Amplía una partición (apartado 8.1) |

Detalle de Windows Server: por defecto, los discos nuevos conectados a buses compartidos (SAN, iSCSI, Fibre Channel) aparecen **sin conexión** (offline), para evitar que dos servidores escriban a la vez en el mismo disco. Se ve con `diskpart` → `san` y se cambia con `Set-Disk -IsOffline $false`.

Nombres de los discos: no hay `/dev/sdb`. Los discos se identifican por **número** (Disco 0, Disco 1...) y las particiones por número dentro de cada disco. Internamente un disco es `\\.\PhysicalDrive1`.

---

## 2. Tipos de sistemas de archivos

### 2.1 Nativos de Windows (equivalente a la Tabla 4.1)

| Nombre | Tamaño máx. de fichero | Tamaño máx. de volumen | Integridad | Notas |
|---|---|---|---|---|
| **NTFS** | 8 PB (en la práctica) | 8 PB con clústeres de 2 MB (16 TB con el tamaño de clúster por defecto de 4 KB) | Journaling (registro de transacciones de metadatos) | Obligatorio para el disco del sistema. Admite permisos, cuotas, compresión, cifrado EFS, enlaces y puntos de montaje |
| **ReFS** | 35 PB | 35 PB | Sumas de comprobación de metadatos (y opcionalmente de datos) + escritura en lugar nuevo (parecido a COW) | Autorreparación con Espacios de almacenamiento, clonación de bloques. **No sirve para arrancar**, ni admite compresión de NTFS, cifrado EFS o cuotas de disco NTFS |
| **FAT32** | 4 GB | 2 TB (las herramientas de Windows solo formatean hasta 32 GB) | Sin journaling | Memorias USB antiguas y la partición EFI |
| **exFAT** | Prácticamente ilimitado para su uso | Muy grande | Sin journaling | Memorias USB y tarjetas con archivos de más de 4 GB |
| **UDF** / **CDFS** | — | — | — | Discos ópticos (apartado 11) |

**Novedad de Windows Server 2025 en ReFS**: **deduplicación y compresión** propias de ReFS (distintas de la deduplicación clásica de NTFS). Ver apartado 7.

### 2.2 No nativos (equivalente a la Tabla 4.2)

| Sistema | Situación en Windows Server 2025 |
|---|---|
| ext2/3/4, XFS, Btrfs, ZFS | **No soportados de forma nativa.** Solo con software de terceros, o accediendo desde una distribución Linux en WSL (`wsl --mount \\.\PhysicalDrive2 --partition 1`) |
| NFS (red) | Cliente NFS integrado como característica: `Install-WindowsFeature NFS-Client` |
| SMB (red) | Nativo: es el sistema de archivos en red propio de Windows (Capítulo 10) |

### 2.3 Consultar los sistemas de archivos soportados

No hay un equivalente directo de `/proc/filesystems`. Cada sistema de archivos es un controlador del kernel, así que se puede ver cuáles están cargados:

| Comando | Función |
|---|---|
| `Get-CimInstance Win32_SystemDriver \| Where-Object Name -in 'Ntfs','ReFS','fastfat','exfat','cdfs','udfs' \| Select Name, State` | Estado de los controladores de sistemas de archivos |
| `format /?` | Muestra los sistemas de archivos que se pueden usar al formatear |
| `Get-Volume` | Muestra el sistema de archivos de cada volumen (columna `FileSystemType`) |

---

## 3. Creación de un sistema de archivos (formateo)

| Linux | Windows Server 2025 | Función |
|---|---|---|
| `mkfs -t ext4 /dev/sdb1` | `format E: /FS:NTFS /Q` | Formatea el volumen E: en NTFS (`/Q` = rápido) |
| `mkfs.ext4 -L datos /dev/sdb1` | `format E: /FS:NTFS /V:Datos /Q` | Igual, con etiqueta |
| — | `format E: /FS:ReFS /Q` | Formatea en ReFS |
| — | `format E: /FS:NTFS /A:64K /Q` | Tamaño de clúster (unidad de asignación) de 64 KB |
| `mkfs.vfat` | `format E: /FS:FAT32 /Q` o `/FS:exFAT` | FAT32 / exFAT |
| (PowerShell) | `Format-Volume -DriveLetter E -FileSystem NTFS -NewFileSystemLabel Datos -AllocationUnitSize 65536` | Lo mismo con PowerShell |
| (`diskpart`) | `select volume 3` → `format fs=ntfs label=Datos quick` | Lo mismo con `diskpart` |

Ejemplo completo en una sola línea de PowerShell (inicializar un disco vacío, crear la partición, darle letra y formatearla):
```
Get-Disk -Number 1 | Initialize-Disk -PartitionStyle GPT -PassThru | New-Partition -UseMaximumSize -AssignDriveLetter | Format-Volume -FileSystem NTFS -NewFileSystemLabel Datos
```

Tamaño de clúster: 4 KB por defecto. Para volúmenes con archivos grandes (máquinas virtuales, bases de datos, copias de seguridad) se recomienda 64 KB.

**Comprobar el tipo tras formatear** (equivalente a `blkid` y `parted -l`):

| Comando | Función |
|---|---|
| `Get-Volume` | Letra, etiqueta, sistema de archivos, tamaño y espacio libre de cada volumen |
| `Get-Partition` | Particiones de todos los discos, con su tipo y letra |
| `diskpart` → `list volume` | Lo mismo en `diskpart` |
| `fsutil fsinfo volumeinfo E:` | Datos del volumen: nombre, número de serie, sistema de archivos y funciones que admite |
| `fsutil fsinfo ntfsinfo E:` / `fsutil fsinfo refsinfo E:` | Detalles internos de NTFS / ReFS (versión, tamaño de clúster...) |

---

## 4. Montaje del sistema de archivos

### 4.1 Montaje temporal: el equivalente de `mount`

Windows **monta automáticamente** todos los volúmenes que encuentra en los discos en línea (función *automount*). "Montar" en Windows es, en la práctica, **dar a un volumen una letra o una carpeta** por la que acceder a él.

| Linux | Windows Server 2025 | Función |
|---|---|---|
| `mount /dev/sdb1 /mnt/datos` | `Add-PartitionAccessPath -DiskNumber 1 -PartitionNumber 2 -AccessPath "C:\Montajes\Datos"` | Monta el volumen en una carpeta vacía de un volumen NTFS (la carpeta debe existir) |
| — | `Set-Partition -DiskNumber 1 -PartitionNumber 2 -NewDriveLetter E` | Le asigna la letra E: |
| (`diskpart`) | `select volume 3` → `assign letter=E` o `assign mount=C:\Montajes\Datos` | Lo mismo con `diskpart` |
| (`mountvol`) | `mountvol C:\Montajes\Datos \\?\Volume{GUID}\` | Lo mismo indicando el identificador del volumen |
| `umount /mnt/datos` | `Remove-PartitionAccessPath -DiskNumber 1 -PartitionNumber 2 -AccessPath "C:\Montajes\Datos"` | Quita el punto de montaje |
| — | `mountvol E: /D` | Quita la letra E: |
| — | `mountvol E: /P` | Quita la letra y **desmonta** el volumen (no se vuelve a montar hasta asignarle una letra o carpeta) |
| — | `fsutil volume dismount E:` | Desmonta el volumen (cerrando lo que esté abierto) sin quitarle la letra |
| `mount -o loop imagen.iso /mnt` | `Mount-DiskImage -ImagePath C:\ISO\imagen.iso` | Monta una imagen ISO, VHD o VHDX (le da una letra automáticamente) |
| — | `Dismount-DiskImage -ImagePath C:\ISO\imagen.iso` | Desmonta la imagen |

**Opciones de `mount` (Tabla 4.3) y su equivalente:**

| Opción `mount` | Equivalente en Windows Server 2025 |
|---|---|
| `-a` (montar todo `/etc/fstab`) | Automático al arrancar. Activar / desactivar el automontaje: `mountvol /E` / `mountvol /N` (o `diskpart` → `automount enable` / `disable`) |
| `-f` (simular) | No existe |
| `-L etiqueta` / `-U uuid` | Los volúmenes se identifican por su **GUID de volumen** (`\\?\Volume{...}\`), que se puede usar con `mountvol` (apartado 5) |
| `-n` (sin `/etc/mtab`) | No aplica |
| `-o opciones` | No hay opciones de montaje. Los comportamientos se ajustan con `fsutil behavior` (Capítulo 3) o como propiedades del volumen |
| `-r` (solo lectura) | `Set-Disk -Number 1 -IsReadOnly $true` (disco completo) o `diskpart` → `attributes volume set readonly` (volumen) |
| `-w` (lectura/escritura) | `Set-Disk -Number 1 -IsReadOnly $false` o `attributes volume clear readonly` |
| `-t tipo` | No hace falta: Windows detecta el sistema de archivos solo |
| `-v` | No aplica |

**Recursos de red** (en Linux se montan con `mount -t nfs` o `-t cifs` y se ponen en `/etc/fstab`):

| Comando | Función |
|---|---|
| `net use Z: \\SRV02\Datos /persistent:yes` | Conecta una carpeta compartida SMB como unidad Z: y la recuerda en los siguientes inicios de sesión |
| `New-SmbMapping -LocalPath Z: -RemotePath \\SRV02\Datos -Persistent $true` | Lo mismo con PowerShell |
| `net use Z: /delete` / `Remove-SmbMapping -LocalPath Z:` | Desconecta |
| `mount -o anon \\SRV-NFS\exportacion Z:` | Monta un recurso **NFS** (con el cliente NFS instalado; aquí `mount` es un comando de Windows, distinto del de Linux) |

Muy importante: en Windows **no hace falta montar un recurso de red para usarlo**. Se puede acceder directamente con su ruta UNC (`\\SRV02\Datos\carpeta\archivo`) en el explorador, en `cmd` o en PowerShell.

### 4.2 Montaje persistente: el equivalente de `/etc/fstab`

**No existe un archivo `/etc/fstab`.** La persistencia funciona así:

| Qué | Dónde se guarda |
|---|---|
| Letras de unidad de los volúmenes locales | Registro, en `HKLM\SYSTEM\MountedDevices` (Windows recuerda qué letra tenía cada volumen) |
| Carpetas de montaje | Dentro de la propia carpeta del volumen NTFS (un "punto de reanálisis" que apunta al volumen) |
| Unidades de red | Por usuario, con `/persistent:yes` (se guarda en `HKCU\Network`) o, en un dominio, con **Directivas de grupo** (Preferencias → Asignaciones de unidades) |

| Campo de `/etc/fstab` | Equivalente en Windows Server 2025 |
|---|---|
| 1. Identificación del dispositivo | GUID del volumen (automático) |
| 2. Punto de montaje | Letra o carpeta asignada |
| 3. Tipo | Detectado automáticamente |
| 4. Opciones | No existen |
| 5. `dump` | No aplica (copias con Windows Server Backup, Capítulo 2) |
| 6. Orden de `fsck` | `autochk` revisa al arrancar los volúmenes marcados como "sucios". Excluir un volumen: `chkntfs /x E:` (Capítulo 1) |

### 4.3 Unidades de montaje de systemd

No existen. Si hace falta montar algo con condiciones especiales al arrancar, se usa una **tarea programada** con desencadenador "Al iniciar el sistema" que ejecute `Add-PartitionAccessPath`, `Mount-DiskImage` o `net use` (Capítulo 1).

### 4.4 Ver los sistemas de archivos montados

| Comando Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `mount`, `findmnt`, `cat /proc/mounts` | `mountvol` (sin parámetros) | Lista los GUID de todos los volúmenes y sus letras y carpetas de montaje |
| `lsblk -f` | `Get-Partition \| Select DiskNumber, PartitionNumber, DriveLetter, AccessPaths` | Particiones con todas sus rutas de acceso |
| `df -hT` | `Get-Volume` o `Get-PSDrive -PSProvider FileSystem` | Volúmenes con tamaño, espacio libre y sistema de archivos |
| — | `fsutil fsinfo drives` | Letras de unidad en uso |
| — | `net use` / `Get-SmbMapping` | Unidades de red conectadas |
| `mountpoint directorio` | `fsutil reparsepoint query C:\Montajes\Datos` | Indica si la carpeta es un punto de montaje (o otro tipo de enlace) |
| `e2label dispositivo etiqueta` | `label E: Datos` o `Set-Volume -DriveLetter E -NewFileSystemLabel Datos` | Ver o cambiar la etiqueta |
| `findfs LABEL=etiqueta` | `Get-Volume -FileSystemLabel Datos` | Busca un volumen por etiqueta |

---

## 5. UUID de los sistemas de archivos

Windows usa varios identificadores únicos, cada uno con su papel:

| Identificador | Qué identifica | Cómo verlo |
|---|---|---|
| **GUID de volumen** | Cada volumen. Es el equivalente más directo al UUID de Linux: `\\?\Volume{0b9f...}\` | `mountvol` o `Get-Volume \| Select DriveLetter, UniqueId` |
| **GUID de partición** | Cada partición GPT | `Get-Partition \| Select DriveLetter, Guid` |
| **GUID / firma de disco** | Cada disco | `Get-Disk \| Select Number, Guid, Signature` o `diskpart` → `uniqueid disk` |
| **Número de serie del volumen** | Número corto que se asigna al formatear | `vol E:` o `fsutil fsinfo volumeinfo E:` |

| Comando Linux | Equivalente en Windows Server 2025 |
|---|---|
| `blkid` | `Get-Volume \| Select DriveLetter, FileSystemLabel, FileSystemType, UniqueId` |
| `uuidgen` | `New-Guid` |
| `tune2fs -U nuevo-uuid` | Para el disco: `diskpart` → `select disk 1` → `uniqueid disk id={nuevo-GUID}`. No hay una herramienta integrada para cambiar el número de serie del volumen |

Caso práctico: si se **clona un disco**, el clon tiene el mismo identificador que el original. Windows no permite dos discos con el mismo identificador en línea a la vez y deja el segundo **sin conexión**; se soluciona cambiándole el identificador con `uniqueid disk`.

---

## 6. Sistemas de archivos temporales/virtuales

Windows **no tiene** `/proc` ni sistemas de archivos virtuales en memoria como `tmpfs`.

| Concepto Linux | Situación en Windows Server 2025 |
|---|---|
| `/proc` | No existe (Capítulo 3). Lo más parecido es la rama `HKLM\HARDWARE` del Registro, que se crea en memoria en cada arranque |
| `tmpfs` (disco en memoria) | No hay disco RAM integrado en el sistema instalado. Solo WinPE y WinRE usan uno (la unidad `X:`, Capítulo 1) |
| `/tmp` | Carpetas temporales en disco: `C:\Windows\Temp` (sistema) y `%TEMP%` (cada usuario). No se vacían solas al reiniciar |

Una idea parecida a los sistemas de archivos virtuales son las **unidades de PowerShell** (*PSDrives*): PowerShell permite recorrer como si fueran carpetas cosas que no son archivos, como el Registro (`HKLM:`), las variables de entorno (`Env:`) o los certificados (`Cert:`). Se listan con `Get-PSDrive`. Por ejemplo: `Get-ChildItem Env:` o `Set-Location HKLM:\SOFTWARE`.

---

## 7. El sistema de archivos ReFS (equivalente a la sección de Btrfs)

ReFS cumple en Windows un papel parecido al de Btrfs: pensado para la integridad de los datos y los volúmenes grandes. Las funciones de RAID y reparto de datos entre discos no están en el propio ReFS, sino en **Espacios de almacenamiento** (Capítulo 5), que es sobre lo que ReFS se usa normalmente.

| Comando Btrfs | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `btrfs filesystem show` | `Get-Volume -DriveLetter E` y `fsutil fsinfo refsinfo E:` | Información del volumen ReFS (versión, tamaño de clúster...) |
| `btrfs filesystem df` | `Get-Volume` o `fsutil volume diskfree E:` | Espacio usado y libre |
| `btrfs scrub` | Tarea programada **Data Integrity Scan** (`\Microsoft\Windows\Data Integrity Scan\`), que revisa periódicamente los volúmenes ReFS | Lee los datos y comprueba su integridad |
| `btrfs balance` | `Optimize-StoragePool -FriendlyName Pool1` | Reparte los datos entre todos los discos del grupo de almacenamiento (Capítulo 5) |
| `btrfs-convert` | No hay conversión entre NTFS y ReFS: hay que copiar los datos. Solo existe `convert E: /FS:NTFS` para pasar de FAT a NTFS | Convertir entre sistemas de archivos |
| `btrfs property set` (etiqueta) | `Set-Volume -DriveLetter E -NewFileSystemLabel Datos` | Cambiar propiedades |
| `btrfs check`, `rescue`, `restore` | `refsutil` (ver abajo) | Comprobar y recuperar un volumen dañado |
| Snapshots de Btrfs | Instantáneas VSS (Capítulo 2) y clonación de bloques de ReFS | Copias instantáneas |
| RAID 0, 1, 10 | Espacios de almacenamiento: simple, reflejo, paridad (Capítulo 5) | RAID |

**Flujos de integridad** (sumas de comprobación de los datos, no solo de los metadatos):

| Comando | Función |
|---|---|
| `Get-FileIntegrity -FileName E:\Datos\archivo.vhdx` | Indica si el archivo tiene activada la comprobación de integridad |
| `Set-FileIntegrity -FileName E:\Datos -Enable $true` | La activa (en una carpeta, se aplica a los archivos nuevos que se creen dentro) |

Si un dato no coincide con su suma de comprobación y el volumen está en un espacio con reflejo o paridad, ReFS lo **repara solo** con la otra copia.

**Herramienta `refsutil`** (reparación y análisis de ReFS):

| Comando | Función |
|---|---|
| `refsutil salvage -QA E: C:\Trabajo` | Análisis rápido de un volumen ReFS dañado y recuperación de archivos (hay varias variantes `-QS`, `-FS`, `-FA`...) |
| `refsutil triage E: C:\Trabajo` | Diagnóstico del daño |
| `refsutil leak E:` | Busca y recupera espacio "perdido" |

**Deduplicación y compresión** (reducen el espacio ocupado cuando hay datos repetidos, como en Btrfs):

| Comando | Función |
|---|---|
| `Install-WindowsFeature FS-Data-Deduplication` | Instala la característica |
| `Enable-ReFSDedup -Volume E: -Type DedupAndCompress` | **ReFS (novedad de Windows Server 2025)**: activa deduplicación y compresión (o solo una: `Dedup` / `Compress`) |
| `Get-ReFSDedupStatus -Volume E:` | Estado y espacio ahorrado en ReFS |
| `Enable-DedupVolume -Volume E: -UsageType Default` | **NTFS** (deduplicación clásica): activa la deduplicación en el volumen |
| `Start-DedupJob -Volume E: -Type Optimization` / `Get-DedupStatus` | NTFS: lanza la optimización / muestra el ahorro |

---

## 8. Mantenimiento y ajuste de sistemas de archivos

### 8.1 NTFS (equivalente a las herramientas de ext2/ext3/ext4, Tabla 4.6)

| Utilidad Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `debugfs` | `fsutil` (subcomandos `file`, `reparsepoint`, `usn`, `objectid`...) | Consultar y modificar metadatos |
| `e2label` | `label` o `Set-Volume -NewFileSystemLabel` | Cambiar la etiqueta |
| `resize2fs` | `Resize-Partition` o `diskpart` → `extend` / `shrink` | Ampliar o reducir. **NTFS se puede ampliar y reducir con el volumen montado y en uso** |
| `tune2fs` | `fsutil behavior set ...`, `fsutil 8dot3name`, `chkntfs` | Ajustar atributos y comportamiento |

Ampliar o reducir un volumen:

| Comando | Función |
|---|---|
| `Get-PartitionSupportedSize -DriveLetter E` | Muestra el tamaño mínimo y máximo posible |
| `Resize-Partition -DriveLetter E -Size (Get-PartitionSupportedSize -DriveLetter E).SizeMax` | Amplía E: hasta ocupar todo el espacio libre contiguo |
| `Resize-Partition -DriveLetter E -Size 50GB` | Deja E: en 50 GB (amplía o reduce) |
| `diskpart` → `select volume E` → `extend` / `shrink desired=10240` | Lo mismo con `diskpart` (`shrink` reduce 10 GB) |

**ReFS** se puede ampliar en caliente, pero **no se puede reducir**.

**Desfragmentación y optimización** (equivale a `xfs_fsr`, apartado 8.2):

| Comando | Función |
|---|---|
| `Optimize-Volume -DriveLetter E -Analyze -Verbose` | Analiza la fragmentación |
| `Optimize-Volume -DriveLetter E -Defrag` | Desfragmenta (discos mecánicos) |
| `Optimize-Volume -DriveLetter E -ReTrim` | Envía TRIM al SSD para liberar bloques no usados (equivale a `fstrim` en Linux) |
| `defrag E: /O` | Aplica la optimización adecuada según el tipo de disco |

Windows ejecuta esta optimización automáticamente una vez por semana mediante una tarea programada.

### 8.2 XFS

**XFS no existe en Windows.** Como referencia, las herramientas de la Tabla 4.7 se corresponden con las de NTFS y ReFS ya vistas:

| Utilidad XFS | Equivalente en Windows Server 2025 |
|---|---|
| `xfs_admin` | `Set-Volume`, `label` |
| `xfs_fsr` | `Optimize-Volume` / `defrag` |
| `xfs_growfs` | `Resize-Partition` |
| `xfs_info` | `fsutil fsinfo ntfsinfo` / `refsinfo` |

### 8.3 Comprobación y reparación

**NTFS (equivalente a la Tabla 4.9).** La herramienta principal es `chkdsk` (Capítulo 1):

| Utilidad Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `fsck.ext4` | `chkdsk E: /f` o `Repair-Volume -DriveLetter E -OfflineScanAndFix` | Comprueba y repara |
| — | `chkdsk E: /scan` / `Repair-Volume -DriveLetter E -Scan` | Revisión **en línea**, sin desmontar |
| — | `chkdsk E: /spotfix` / `Repair-Volume -DriveLetter E -SpotFix` | Repara en segundos solo los errores ya detectados |
| Carpeta `lost+found` | Carpeta **`found.000`** en la raíz del volumen, con archivos `FILE0000.CHK`, `FILE0001.CHK`... | Donde `chkdsk` deja los fragmentos de archivos recuperados |
| `dumpe2fs` / `tune2fs -l` | `fsutil fsinfo ntfsinfo E:` | Información interna del sistema de archivos |
| — | `fsutil dirty query E:` | Indica si el volumen está marcado para revisar |
| — | `fsutil repair state E:` | Estado de corrupción detectado por la autorreparación de NTFS |

NTFS tiene **autorreparación en línea**: corrige muchos errores pequeños sin desmontar el volumen. Solo los problemas que no puede arreglar así quedan anotados para que los repare `chkdsk /spotfix` o el siguiente arranque.

**ReFS (equivalente a la Tabla 4.10 de XFS):** no usa `chkdsk`. Se repara solo (apartado 7) y, en casos graves, con `refsutil`.

| Utilidad XFS | Equivalente en Windows Server 2025 |
|---|---|
| `xfs_repair` / `xfs_check` | Autorreparación de ReFS + `refsutil salvage` / `triage` |
| `xfsdump` / `xfsrestore` | `wbadmin start backup` / `wbadmin start recovery` (Capítulo 2) |
| `xfs_metadump` | No hay equivalente |

### 8.4 Monitorización SMART

Windows **no incluye `smartctl` ni `smartd`**, pero consulta la salud de los discos a través de su propio sistema de almacenamiento:

| Comando Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `smartctl -H` | `Get-PhysicalDisk \| Select FriendlyName, MediaType, HealthStatus, OperationalStatus` | Resumen de salud (`Healthy`, `Warning`, `Unhealthy`) |
| `smartctl -a` | `Get-PhysicalDisk \| Get-StorageReliabilityCounter \| Select DeviceId, Temperature, ReadErrorsTotal, WriteErrorsTotal, Wear, PowerOnHours` | Contadores de fiabilidad: temperatura, errores, desgaste del SSD, horas de funcionamiento |
| `smartctl -i` | `Get-PhysicalDisk \| Select FriendlyName, Model, SerialNumber, FirmwareVersion, BusType` | Información básica del disco |
| — | `Get-CimInstance -Namespace root\wmi -ClassName MSStorageDriver_FailurePredictStatus` | Indica si SMART predice un fallo (`PredictFailure = True`) |
| `smartctl -l error` | Visor de eventos → registro Sistema, eventos del origen **disk** (por ejemplo, 7 y 153: errores de lectura o de E/S) | Registro de errores |
| `smartctl -t short/long` | No hay equivalente integrado | Autopruebas del disco |
| `smartd` | En clústeres con Espacios de almacenamiento directos, el **Servicio de mantenimiento** (Health Service) vigila los discos y avisa de fallos. En un servidor normal, se vigila con SCOM, Windows Admin Center o herramientas de terceros | Vigilancia automática |

Para tener exactamente las mismas funciones que en Linux se puede instalar **smartmontools** para Windows (`smartctl` y `smartd` también existen en Windows como software de terceros).

---

## 9. Espacio de intercambio (swap)

Windows no usa particiones de swap: usa un **archivo de paginación**, `pagefile.sys`, en la raíz de uno o varios volúmenes. Por defecto Windows lo gestiona solo (tamaño automático en `C:`).

| Comando Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `swapon -s`, `cat /proc/swaps` | `Get-CimInstance Win32_PageFileUsage \| Select Name, AllocatedBaseSize, CurrentUsage, PeakUsage` | Archivos de paginación en uso (tamaños en MB) |
| `free -m` | `Get-CimInstance Win32_OperatingSystem \| Select TotalVirtualMemorySize, FreeVirtualMemory` | Memoria virtual total y libre (KB) |
| Ver configuración | `Get-CimInstance Win32_PageFileSetting` | Archivos de paginación configurados a mano |
| `mkswap` + `swapon` | Ver ejemplo siguiente | Crear un archivo de paginación |
| `swapoff` | `Get-CimInstance Win32_PageFileSetting -Filter "Name='D:\\pagefile.sys'" \| Remove-CimInstance` | Quitar un archivo de paginación |
| Prioridad (`swapon -p`) | No existe; Windows reparte el uso entre todos los archivos de paginación | — |
| Herramienta gráfica | `sysdm.cpl` → Opciones avanzadas → Rendimiento → Configuración → Opciones avanzadas → **Memoria virtual** | — |

Ejemplo, crear un archivo de paginación de 4 a 8 GB en D: (primero hay que desactivar la gestión automática):
```
Set-CimInstance -Query "SELECT * FROM Win32_ComputerSystem" -Property @{AutomaticManagedPagefile=$false}
New-CimInstance -ClassName Win32_PageFileSetting -Property @{Name='D:\pagefile.sys'; InitialSize=4096; MaximumSize=8192}
```
Los cambios en el archivo de paginación **necesitan reiniciar**.

Notas:
- Conviene mantener un archivo de paginación en `C:` de tamaño suficiente, porque Windows lo usa para guardar el **volcado de memoria** si el sistema falla (Capítulo 3).
- La configuración se guarda en el Registro: valor `PagingFiles` de `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management`.
- Existe también `swapfile.sys`, un pequeño archivo auxiliar que Windows gestiona solo.

---

## 10. AutoFS (montaje automático de recursos de red)

**No hay un equivalente directo de AutoFS**, porque en Windows no hace falta: cualquier carpeta compartida se usa **directamente por su ruta UNC** (`\\servidor\recurso`) y la conexión se abre en el momento de acceder, igual que hace AutoFS. Lo que sí existe es una forma de organizar esas rutas en un espacio de nombres único: **Espacios de nombres DFS** (DFS Namespaces).

| Concepto AutoFS | Equivalente en Windows Server 2025 |
|---|---|
| Montaje bajo demanda | Acceso directo por ruta UNC (sin montar nada) |
| `/etc/auto.master` + mapas | **Espacio de nombres DFS**: una ruta única (`\\empresa.local\Publico`) con "carpetas" que apuntan a recursos de distintos servidores |
| Mapa indirecto (`/etc/auto.directorio`) | Carpetas de un espacio de nombres DFS: `\\empresa.local\Publico\Ventas` → `\\SRV02\Ventas` |
| Mapa directo (`/etc/auto.direct`) | Asignación de unidades por Directiva de grupo (letra fija para una ruta concreta) |
| `TimeOutIdleSec` (desmontar por inactividad) | El servidor SMB cierra las sesiones inactivas: `Set-SmbServerConfiguration -AutoDisconnectTimeout 15` (minutos) |
| Archivo de configuración del servicio | No hay archivo: se configura con `dfsmgmt.msc` o PowerShell |

**Comandos de Espacios de nombres DFS:**

| Comando | Función |
|---|---|
| `Install-WindowsFeature FS-DFS-Namespace -IncludeManagementTools` | Instala el rol |
| `New-DfsnRoot -Path \\empresa.local\Publico -TargetPath \\SRV01\Publico -Type DomainV2` | Crea el espacio de nombres (la carpeta compartida `Publico` debe existir en SRV01) |
| `New-DfsnFolder -Path \\empresa.local\Publico\Ventas -TargetPath \\SRV02\Ventas` | Añade una carpeta que apunta a otro servidor |
| `Get-DfsnFolder -Path \\empresa.local\Publico\*` | Lista las carpetas del espacio de nombres |
| `dfsmgmt.msc` | Consola gráfica de DFS |

Una ventaja sobre AutoFS: una carpeta DFS puede tener **varios destinos** (copias en distintos servidores, sincronizadas con **Replicación DFS**), y los clientes usan automáticamente el más cercano o el que esté disponible.

---

## 11. Medios ópticos: ISO9660 y UDF

| Comando Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `mkisofs` / `genisoimage` | `oscdimg` (del **Windows ADK**, Capítulo 1) | Crear una imagen ISO |
| `cdrecord` | `isoburn /Q E: C:\ISO\imagen.iso` (Grabadora de imágenes de disco de Windows, solo con escritorio) | Grabar una ISO en un disco óptico |
| `mount -o loop imagen.iso` | `Mount-DiskImage -ImagePath C:\ISO\imagen.iso` | Montar una ISO |
| `file imagen.iso` (¿arrancable?) | No hay comando directo. Se monta la ISO y se comprueba si tiene archivos de arranque (`boot\etfsboot.com` para BIOS, `efi\boot\bootx64.efi` para UEFI) | Comprobar si es arrancable |
| Formatear un disco regrabable | `format E: /FS:UDF /R:2.01` | Formatear en UDF |

Ejemplos de `oscdimg`:

| Comando | Función |
|---|---|
| `oscdimg -n -lDATOS C:\Carpeta C:\ISO\datos.iso` | ISO de datos con nombres largos (`-n`) y etiqueta DATOS (`-l`) |
| `oscdimg -j1 -lDATOS C:\Carpeta C:\ISO\datos.iso` | Con nombres Joliet (compatibilidad) |
| `oscdimg -u2 -udfver102 -lDATOS C:\Carpeta C:\ISO\datos.iso` | Solo UDF (para archivos grandes) |
| `oscdimg -m -o -u2 -udfver102 -bootdata:2#p0,e,betfsboot.com#pEF,e,befisys.bin C:\Origen C:\ISO\arranque.iso` | ISO arrancable en BIOS y UEFI (El Torito) |

Las extensiones del libro también existen en Windows: **El Torito** (arranque), **Joliet** (nombres largos, que precisamente creó Microsoft) y **UDF**. **Rock Ridge** (metadatos Unix) no se usa en Windows. Los controladores de lectura son `cdfs.sys` (ISO9660) y `udfs.sys` (UDF).

---

## 12. Sistemas de archivos cifrados

| Tipo en Linux | Equivalente en Windows Server 2025 | Características |
|---|---|---|
| `dm-crypt` / **LUKS** (volumen completo) | **BitLocker** | Cifra volúmenes completos (sistema o datos). Una clave maestra protegida por varios "protectores" |
| **eCryptfs** (archivo a archivo, en capa) | **EFS** (Sistema de cifrado de archivos) | Cifra archivos y carpetas concretos dentro de NTFS, ligado al certificado de cada usuario |

### BitLocker (equivalente a LUKS)

En Windows Server, BitLocker es una **característica** que hay que instalar:
```
Install-WindowsFeature BitLocker -IncludeAllSubFeature -IncludeManagementTools -Restart
```

El parecido con LUKS es directo: LUKS tiene una clave maestra y varias claves de usuario en distintas "ranuras"; BitLocker tiene una clave de cifrado del volumen protegida por uno o varios **protectores**:

| Protector | Descripción |
|---|---|
| TPM | El chip de seguridad del equipo libera la clave si el arranque no ha sido alterado (solo disco del sistema) |
| TPM + PIN | Además pide un PIN al arrancar |
| Contraseña | Para volúmenes de datos |
| Contraseña de recuperación | 48 dígitos; **hay que guardarla en lugar seguro** (o en Active Directory) |
| Clave de inicio / de recuperación en archivo | Archivo en un USB |

| Comando | Función | Equivalente `cryptsetup` |
|---|---|---|
| `manage-bde -status` / `Get-BitLockerVolume` | Estado de cifrado de todos los volúmenes | `cryptsetup status` |
| `Enable-BitLocker -MountPoint E: -EncryptionMethod XtsAes256 -UsedSpaceOnly -RecoveryPasswordProtector` | Cifra E: y crea una contraseña de recuperación | `cryptsetup luksFormat` |
| `Add-BitLockerKeyProtector -MountPoint E: -PasswordProtector` | Añade una contraseña como protector (la pide) | `cryptsetup luksAddKey` |
| `manage-bde -protectors -get E:` | Lista los protectores (y muestra la contraseña de recuperación) | `cryptsetup luksDump` |
| `Remove-BitLockerKeyProtector -MountPoint E: -KeyProtectorId "{...}"` | Quita un protector | `cryptsetup luksRemoveKey` |
| `Unlock-BitLocker -MountPoint E: -Password (Read-Host -AsSecureString)` | Desbloquea un volumen de datos | `cryptsetup luksOpen` |
| `Lock-BitLocker -MountPoint E:` | Bloquea el volumen | `cryptsetup luksClose` |
| `Enable-BitLockerAutoUnlock -MountPoint E:` | Desbloqueo automático al arrancar (requiere el disco del sistema cifrado) | Entrada en `/etc/crypttab` |
| `Backup-BitLockerKeyProtector -MountPoint E: -KeyProtectorId "{...}"` | Guarda la contraseña de recuperación en Active Directory | — |
| `Suspend-BitLocker` / `Resume-BitLocker` | Suspende la protección temporalmente (antes de cambios de firmware o BCD, Capítulo 1) | — |
| `Disable-BitLocker -MountPoint E:` | Descifra el volumen | — |

Windows Server incluye además **BitLocker Network Unlock** (característica `BitLocker-NetworkUnlock`): permite que los servidores con el disco del sistema cifrado con TPM + PIN arranquen sin escribir el PIN cuando están conectados a la red de la empresa.

### EFS (equivalente a eCryptfs)

EFS solo funciona en **NTFS** (no en ReFS) y cifra con un certificado del usuario, así que solo ese usuario (y los **agentes de recuperación**) puede abrir los archivos. No necesita un segundo montaje como eCryptfs: se activa como un atributo del archivo o de la carpeta.

| Comando | Función |
|---|---|
| `cipher /e E:\Privado` | Cifra la carpeta (los archivos nuevos que se creen dentro también se cifran) |
| `cipher /d E:\Privado` | Descifra la carpeta |
| `cipher E:\Privado` | Muestra qué archivos están cifrados (`E`) y cuáles no (`U`) |
| `cipher /c E:\Privado\archivo.txt` | Muestra qué usuarios pueden descifrar un archivo |
| `cipher /x C:\Copia\certificado_efs` | Hace copia del certificado y la clave EFS del usuario (**imprescindible**: sin ella, si se pierde el perfil, se pierden los datos) |
| `cipher /r:agente_recuperacion` | Crea un certificado de agente de recuperación |

Para cifrar volúmenes de datos de servidores, Microsoft recomienda BitLocker. EFS es útil para proteger archivos concretos de unos usuarios frente a otros.

---

## 13. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 4) | Equivalente en Windows Server 2025 |
|---|---|
| `fdisk`, `gdisk`, `parted` | `diskpart`, `Get-Disk` / `Initialize-Disk` / `New-Partition`, `diskmgmt.msc` |
| Inodo | Registro de la MFT (NTFS) |
| `ln` / `ln -s` | `mklink /H` / `mklink` (y `mklink /J`) |
| ext4 | NTFS |
| Btrfs | ReFS + Espacios de almacenamiento |
| XFS | Sin equivalente |
| vfat | FAT32 / exFAT |
| `/proc/filesystems` | Controladores de sistemas de archivos (`Win32_SystemDriver`), `format /?` |
| `mkfs.ext4` | `format /FS:NTFS` o `Format-Volume` |
| `blkid` | `Get-Volume`, `fsutil fsinfo volumeinfo` |
| `mount` / `umount` | Letras y carpetas: `Set-Partition -NewDriveLetter`, `Add-PartitionAccessPath`, `mountvol` / `Remove-PartitionAccessPath`, `mountvol /P` |
| `mount -o loop` | `Mount-DiskImage` |
| `mount -t cifs` / `-t nfs` | `net use` / `New-SmbMapping`; `mount` del cliente NFS de Windows |
| `/etc/fstab` | No existe: automontaje + `HKLM\SYSTEM\MountedDevices` + Directivas de grupo |
| `findmnt`, `lsblk`, `df` | `mountvol`, `Get-Partition`, `Get-Volume` |
| UUID | GUID de volumen (`\\?\Volume{...}\`) |
| `uuidgen` / `tune2fs -U` | `New-Guid` / `diskpart uniqueid disk` |
| `e2label` | `label` / `Set-Volume` |
| `resize2fs` / `xfs_growfs` | `Resize-Partition` (NTFS amplía y reduce en caliente; ReFS solo amplía) |
| `xfs_fsr` / `fstrim` | `Optimize-Volume` / `defrag` |
| `fsck` / `lost+found` | `chkdsk` / `Repair-Volume` / carpeta `found.000` |
| `dumpe2fs` | `fsutil fsinfo ntfsinfo` |
| `btrfs scrub` / `btrfs check` | Data Integrity Scan / `refsutil` |
| `btrfs balance` | `Optimize-StoragePool` |
| `smartctl`, `smartd` | `Get-PhysicalDisk`, `Get-StorageReliabilityCounter` (o smartmontools para Windows) |
| `mkswap` + `swapon` | `pagefile.sys` (`Win32_PageFileSetting`, `sysdm.cpl`) |
| AutoFS | Rutas UNC + Espacios de nombres DFS |
| `mkisofs` / `cdrecord` | `oscdimg` / `isoburn` |
| LUKS / `cryptsetup` | BitLocker / `manage-bde`, `*-BitLocker` |
| eCryptfs | EFS / `cipher` |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 4 ("Managing the Filesystem") del libro LPIC-2. Fuentes: documentación oficial de Microsoft Learn (NTFS, ReFS e integridad de ReFS, deduplicación de ReFS en Windows Server 2025, módulo Storage de PowerShell, diskpart, format, mountvol, fsutil, chkdsk, refsutil, Optimize-Volume, archivos de paginación, Espacios de nombres DFS, oscdimg, BitLocker y cipher).*
