# LPIC-2 · Capítulo 4: Managing the Filesystem
### Equivalencias en Windows Server 2025

---

## 1. Tipos de sistemas de archivos

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| ext2/ext3/ext4 (con journaling) | **NTFS** — sistema de archivos con journaling, el estándar de Windows desde Windows NT |
| Btrfs (copy-on-write, checksums, snapshots, RAID integrado) | **ReFS** (Resilient File System) — también basado en COW, con checksums de metadatos (y opcionalmente de datos vía "integrity streams"), integración con Storage Spaces para autorreparación, y soporte de volúmenes hasta 35 PB en Windows Server 2025 |
| vfat/FAT32 | **FAT32** (mismo nombre, mismas limitaciones: 4 GiB máx. por fichero) |
| Sistemas no nativos (NTFS en el mundo Linux) | En Windows, el "no nativo" sería al revés: soporte de **ext2/ext3/ext4 de solo lectura** vía WSL2 o herramientas de terceros, ya que Windows no monta sistemas de archivos Linux de forma nativa |
| — (no tiene equivalente exacto en Linux) | **exFAT**: pensado para memorias USB grandes, sin journaling, con menos limitaciones que FAT32 |

### 1.1 Comparativa rápida NTFS vs ReFS (equivalente a la comparación ext4 vs Btrfs del libro)

| Característica | NTFS | ReFS |
|---|---|---|
| Journaling / integridad | Journaling clásico | Checksums de metadatos (y datos opcionalmente), autorreparación con Storage Spaces |
| Tamaño máx. práctico | ~256 TB | Hasta 35 PB (Windows Server 2025) |
| Snapshots / block cloning | Solo vía VSS (a nivel de volumen) | Block cloning nativo, más eficiente para VHDX/Hyper-V |
| Arrancable como volumen de sistema | Sí (estándar) | Desde Windows Server 2025 (novedad, antes solo NTFS) |
| Cuotas de disco, cifrado EFS, compresión | Sí | No (ReFS no soporta EFS ni compresión NTFS clásica) |

---

## 2. Creación de un sistema de archivos (formateo)

| Comando Linux | Equivalente en Windows Server 2025 |
|---|---|
| `mkfs -t fstype dispositivo` / `mkfs.ext4` | **`format.com`** (símbolo del sistema clásico) o el cmdlet PowerShell **`Format-Volume`** |
| Herramienta interactiva de particionado (`fdisk`/`parted`) | **`diskpart`** (línea de comandos interactiva) o los cmdlets `Initialize-Disk`, `New-Partition` de PowerShell |

**Ejemplo con `format.com`:**
```
format E: /FS:NTFS /Q
```

**Ejemplo con PowerShell (flujo completo: inicializar disco, crear partición y formatear, similar a `fdisk`+`mkfs` encadenados):**
```powershell
Get-Disk | Where-Object PartitionStyle -eq 'RAW' |
    Initialize-Disk -PartitionStyle GPT -PassThru |
    New-Partition -AssignDriveLetter -UseMaximumSize |
    Format-Volume -FileSystem NTFS -Confirm:$false
```

| Cmdlet | Función |
|---|---|
| `Initialize-Disk` | Prepara un disco nuevo con un esquema de particiones (GPT/MBR) |
| `New-Partition` | Crea una partición dentro de un disco ya inicializado |
| `Format-Volume` | Formatea el volumen con el sistema de archivos indicado (`-FileSystem NTFS/ReFS/FAT32/exFAT`) |

---

## 3. Montaje del sistema de archivos

Windows monta automáticamente cada volumen al detectarlo, asignándole una **letra de unidad** (C:, D:...) o un **punto de montaje** dentro de una carpeta NTFS existente — no requiere una operación manual equivalente a `mount` en el uso normal.

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| `mount -t fstype dispositivo punto` | Asignación automática de letra de unidad al conectar el disco; para forzarlo manualmente: **`mountvol`** o `Add-PartitionAccessPath` (PowerShell) |
| Punto de montaje dentro de un directorio (en vez de una letra) | Igual concepto en Windows: se puede montar un volumen dentro de una carpeta NTFS vacía en lugar de asignarle letra, con `mountvol ruta GUID_volumen` o desde el Administrador de discos |
| `umount` | `mountvol ruta /D` (elimina el punto de montaje), o `Remove-PartitionAccessPath` |
| `/etc/fstab` (montaje persistente) | **No existe un fichero equivalente**: las asignaciones de letra/punto de montaje quedan grabadas automáticamente en el Registro (`HKLM\SYSTEM\MountedDevices`) y persisten solas entre reinicios |
| `mount -a` (montar todo lo listado) | `mountvol /E` (rehabilita el montaje automático de nuevos volúmenes básicos, si se había desactivado con `mountvol /N`) |

**Sintaxis de `mountvol` (Tabla comparativa):**

| Opción | Función |
|---|---|
| `mountvol` (sin opciones) | Lista los volúmenes y sus puntos de montaje actuales |
| `mountvol ruta nombre_volumen` | Crea un punto de montaje en una carpeta NTFS existente |
| `mountvol ruta /D` | Elimina un punto de montaje |
| `mountvol /N` | Desactiva el montaje automático de nuevos volúmenes básicos |
| `mountvol /E` | Reactiva el montaje automático |
| `mountvol /R` | Elimina puntos de montaje huérfanos (de volúmenes que ya no existen) |

**Montar imágenes ISO/VHD (sin equivalente exacto en el libro, pero muy usado en Windows):**
```powershell
Mount-DiskImage -ImagePath "C:\ISO\instalacion.iso"
```

---

## 4. Identificación de volúmenes (equivalente a UUID)

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| `blkid` (muestra UUID/etiqueta) | `Get-Volume` (PowerShell), que muestra el `UniqueId` (ruta GUID del volumen) y la etiqueta (`FileSystemLabel`) |
| `uuidgen` | No aplica igual: el GUID del volumen lo asigna el propio sistema al formatear, no se genera manualmente |
| `tune2fs -U` (cambiar UUID) | No es una operación habitual/soportada en NTFS/ReFS; el identificador de volumen es fijo desde el formateo |
| `e2label`/etiqueta de volumen | `label` (comando clásico) o `Set-Volume -NewFileSystemLabel "nombre"` (PowerShell) |
| Formato del UUID en `/etc/fstab` | Formato equivalente en Windows: rutas `\\?\Volume{GUID}\`, usadas internamente por `mountvol` y el Registro |

Ejemplo:
```powershell
Get-Volume | Select-Object DriveLetter, FileSystemLabel, FileSystem, UniqueId
```

---

## 5. Mantenimiento, comprobación y reparación

| Comando Linux | Equivalente en Windows Server 2025 |
|---|---|
| `fsck` (comprobar/reparar) | **`chkdsk`** (clásico) o el cmdlet moderno **`Repair-Volume`** |
| `tune2fs -l` (ver atributos) | `fsutil fsinfo volumeinfo C:` o `Get-Volume` |
| `dumpe2fs` | `fsutil fsinfo ntfsinfo C:` (información detallada del sistema NTFS: tamaño de clúster, MFT, etc.) |
| `resize2fs` (redimensionar) | `Resize-Partition` (PowerShell) o desde el Administrador de discos |
| `xfs_repair`, `btrfs check` | `Repair-Volume -DriveLetter C -Scan` (equivalente al chequeo online que hacen XFS/Btrfs sin desmontar) |
| Reequilibrado/optimización (`btrfs balance`, desfragmentación) | **`Optimize-Volume`** (desfragmenta HDD o hace TRIM en SSD según el tipo de disco detectado) |

**Opciones destacadas de `chkdsk`:**

| Opción | Función |
|---|---|
| `chkdsk C:` | Comprobación de solo lectura (informe sin reparar) |
| `chkdsk C: /F` | Repara errores encontrados |
| `chkdsk C: /R` | Localiza sectores dañados y recupera información legible (implica `/F`) |
| `chkdsk C: /X` | Fuerza el desmontaje del volumen antes de comprobar |

**`Repair-Volume` (equivalente moderno, con reparación en caliente similar a XFS/Btrfs):**
```powershell
Repair-Volume -DriveLetter C -Scan
Repair-Volume -DriveLetter C -OfflineScanAndFix
```

---

## 6. Monitorización SMART

| Comando Linux | Equivalente en Windows Server 2025 |
|---|---|
| `smartctl -a dispositivo` | `Get-PhysicalDisk \| Get-StorageReliabilityCounter` (PowerShell), que expone temperatura, horas de encendido, errores de lectura/escritura y ciclos de carga — el equivalente directo a los atributos SMART |
| `smartctl -H` (resumen de salud) | `Get-PhysicalDisk \| Select-Object FriendlyName, HealthStatus, OperationalStatus` |
| `smartctl -t short/long` (autoprueba) | No hay un cmdlet nativo directo para lanzar autopruebas SMART; se recurre a herramientas del fabricante o a `wmic diskdrive get status` para un chequeo básico |
| `smartd` (demonio de comprobación periódica) | No hay un demonio equivalente nativo; el rol de "Windows Server Storage" y **Storage Spaces** llevan su propia monitorización de salud integrada (`Get-StorageSubSystem`, `Get-StorageHealthReport` en entornos con Storage Spaces Direct) |

Ejemplo:
```powershell
Get-PhysicalDisk | Get-StorageReliabilityCounter |
    Select-Object DeviceId, Temperature, PowerOnHours, ReadErrorsTotal, WriteErrorsTotal
```

---

## 7. Espacio de intercambio (swap)

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| Partición/fichero swap (`mkswap`, `swapon`) | **Archivo de paginación** (`pagefile.sys`), gestionado por Windows automáticamente, o configurable de forma manual |
| `swapon -s` / `free` (ver estado) | `Get-CimInstance Win32_PageFileUsage` o Panel de control → Sistema → Configuración avanzada → Rendimiento → Memoria virtual |
| `/etc/fstab` con entrada `swap` | El tamaño/ubicación del `pagefile.sys` se configura en el Registro (`HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management`) o vía la interfaz gráfica; no hay un fichero de configuración de texto equivalente |
| Prioridad entre varios swaps (`swapon -p`) | Windows permite definir un `pagefile.sys` por cada unidad, pero no un sistema de prioridades tan explícito como en Linux |

**Consulta y configuración vía PowerShell:**
```powershell
Get-CimInstance Win32_PageFileUsage
Get-CimInstance Win32_PageFileSetting
```

---

## 8. Recursos compartidos montados automáticamente (equivalente a AutoFS)

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| AutoFS (`/etc/auto.master`, montaje bajo demanda de recursos de red) | **Asignación de unidades de red por Directiva de grupo** (GPO → Configuración de usuario → Preferencias → Configuración de Windows → Asignaciones de unidad), que conecta automáticamente unidades de red al iniciar sesión |
| `net use` (montar un recurso de red manualmente, equivalente a `mount` de un recurso Samba/NFS) | `net use Z: \\servidor\recurso /persistent:yes` |
| Recursos compartidos con espacio de nombres unificado | **DFS Namespaces** (Distributed File System), que agrupa varios recursos compartidos bajo una única ruta lógica, similar en espíritu a un árbol AutoFS centralizado |

---

## 9. Sistemas de archivos cifrados

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| LUKS/dm-crypt (cifrado de volumen completo) | **BitLocker**, gestionado con **`manage-bde`** (línea de comandos) o el Panel de control |
| eCryptfs (cifrado fichero a fichero, en capas) | **EFS** (Encrypting File System), cifrado a nivel de fichero/carpeta individual sobre NTFS (no disponible en ReFS) |

**Comandos destacados de `manage-bde`:**

| Comando | Función |
|---|---|
| `manage-bde -status` | Muestra el estado de cifrado de todos los volúmenes |
| `manage-bde -on C:` | Activa el cifrado BitLocker en la unidad C: |
| `manage-bde -off C:` | Desactiva (descifra) el volumen |
| `manage-bde -lock C:` / `-unlock C:` | Bloquea/desbloquea el acceso a un volumen cifrado |
| `manage-bde -protectors -add C: -RecoveryPassword` | Añade un protector de clave (contraseña de recuperación) |

**EFS (equivalente a eCryptfs, cifrado por fichero/carpeta):**
```
cipher /E carpeta          REM cifra una carpeta con EFS
cipher /D carpeta          REM descifra
```

---

## 10. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 4) | Equivalente en Windows Server 2025 |
|---|---|
| ext4 | NTFS |
| Btrfs (COW, checksums, snapshots) | ReFS |
| `mkfs` | `format.com`, `Format-Volume` |
| `mount` / `/etc/fstab` | Montaje automático por letra de unidad; `mountvol` para puntos de montaje manuales (sin fichero de configuración equivalente) |
| `blkid` / UUID | `Get-Volume` / `UniqueId` (ruta GUID del volumen) |
| `fsck` | `chkdsk`, `Repair-Volume` |
| `smartctl`/`smartd` | `Get-PhysicalDisk \| Get-StorageReliabilityCounter` |
| `mkswap`/`swapon` | `pagefile.sys` (archivo de paginación) |
| AutoFS | Asignación de unidades por GPO, `net use`, DFS Namespaces |
| LUKS/dm-crypt | BitLocker (`manage-bde`) |
| eCryptfs | EFS (`cipher.exe`) |
| `mkisofs`/`cdrecord` | `Mount-DiskImage` (montar ISO), `oscdimg.exe` (crear ISO, incluido en el ADK de Windows) |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 4 ("Managing the Filesystem") del libro LPIC-2. Fuentes: conocimiento general de administración de Windows Server, verificado con documentación oficial de Microsoft Learn (ReFS en Windows Server 2025, `Format-Volume`, `mountvol`, `manage-bde`, `Get-StorageReliabilityCounter`).*
