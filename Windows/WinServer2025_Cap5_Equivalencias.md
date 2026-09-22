# LPIC-2 · Capítulo 5: Administering Advanced Storage Devices
### Equivalencias en Windows Server 2025

Aviso para este capítulo: Windows unifica en una sola tecnología, **Storage Spaces**, buena parte de lo que en Linux se reparte entre `mdadm` (RAID) y LVM (volúmenes lógicos). Por eso varias secciones de este documento remiten al mismo conjunto de cmdlets.

---

## 1. RAID software

### 1.1 Storage Spaces como equivalente a `mdadm`

Windows Server no usa `mdadm`; el mecanismo de RAID software integrado se llama **Storage Spaces**, y combina en un solo flujo lo que en Linux serían pasos separados de `mdadm` (RAID) + LVM (volúmenes lógicos).

| Concepto Linux (RAID con `mdadm`) | Equivalente en Storage Spaces |
|---|---|
| Nivel RAID 0 (striping, sin redundancia) | Opción de resiliencia **Simple** |
| Nivel RAID 1 / RAID 10 (mirroring) | Opción de resiliencia **Mirror** (con `-NumberOfDataCopies 2` o `3` para triple espejo) |
| Nivel RAID 5 (paridad simple) | Opción de resiliencia **Parity** |
| Nivel RAID 6 (doble paridad) | **Parity** con `-PhysicalDiskRedundancy 2` (paridad doble, protege ante el fallo de 2 discos) |
| Disco de repuesto (`--spare-devices`) | Discos marcados como **hot spare** dentro del Storage Pool |
| `/proc/mdstat` (estado del array) | `Get-VirtualDisk` / `Get-StoragePool` (PowerShell), o el panel "Storage Pools" en el Administrador del servidor |
| `mdadm --detail` | `Get-VirtualDisk -FriendlyName nombre \| Format-List *` |
| `mdadm.conf` | No hay fichero de configuración de texto: la definición del pool/disco virtual se guarda internamente en los metadatos del propio Storage Pool |
| `mdadm --monitor` (alertas ante eventos) | Eventos del **Visor de eventos** (registro relacionado con "Storage Spaces"), o `Get-StorageSubSystem \| Get-StorageHealthReport` en entornos con Storage Spaces Direct |

### 1.2 Flujo de creación con Storage Spaces (equivalente a crear un array con `mdadm -C`)

| Paso | Cmdlet PowerShell |
|---|---|
| 1. Ver discos disponibles para agrupar | `Get-PhysicalDisk -CanPool $True` |
| 2. Crear el Storage Pool (agrupación de discos físicos) | `New-StoragePool -FriendlyName "Pool1" -StorageSubsystemFriendlyName "Windows Storage*" -PhysicalDisks (Get-PhysicalDisk -CanPool $True)` |
| 3. Crear el disco virtual con el nivel de resiliencia deseado | `New-VirtualDisk -StoragePoolFriendlyName "Pool1" -FriendlyName "VDisk1" -ResiliencySettingName Mirror -Size 500GB` |
| 4. Inicializar, particionar y formatear (como con cualquier disco nuevo) | `Get-VirtualDisk -FriendlyName VDisk1 \| Get-Disk \| Initialize-Disk -PassThru \| New-Partition -AssignDriveLetter -UseMaximumSize \| Format-Volume` |

Ejemplo con doble paridad (equivalente a RAID 6, requiere al menos 5 discos físicos):
```powershell
New-VirtualDisk -StoragePoolFriendlyName "Pool1" -FriendlyName "VDisk1" `
  -ResiliencySettingName Parity -NumberOfDataCopies 1 -PhysicalDiskRedundancy 2 -Size 20GB
```

### 1.3 Gestión y mantenimiento del pool

| Cmdlet | Función (equivalente a...) |
|---|---|
| `Add-PhysicalDisk` | Añadir un disco al pool (equivalente a `mdadm --manage --add`) |
| `Resize-VirtualDisk` | Ampliar el disco virtual (equivalente a `mdadm --grow`) |
| `Resize-Partition` | Ampliar la partición/volumen tras ampliar el disco virtual (equivalente a `resize2fs`, ver Capítulo 4) |
| `Optimize-StoragePool` | Reequilibra los discos físicos del pool para optimizar espacio y rendimiento (equivalente conceptual a `btrfs balance`, ver Capítulo 4) |
| `Remove-VirtualDisk` / `Remove-StoragePool` | Elimina el disco virtual / el pool completo |

---

## 2. Ajuste del acceso a dispositivos de almacenamiento

| Comando Linux | Equivalente en Windows Server 2025 |
|---|---|
| `hdparm -I` (parámetros del disco) | `Get-PhysicalDisk \| Format-List *` o `Get-Disk \| Format-List *` |
| `hdparm -W` (write-caching) | `Set-PhysicalDisk` con la propiedad de caché de escritura, o desde el Administrador de discos → Propiedades → Directivas |
| `hdparm -t`/`-T` (test de rendimiento) | No hay un cmdlet nativo de benchmark; se usan herramientas como **DiskSpd** (Microsoft) o **Winsat disk** (`winsat disk -drive C`) |
| `sdparm` (tablas VPD de dispositivos SCSI) | `Get-PhysicalDisk \| Select FriendlyName, SerialNumber, Manufacturer, Model, FirmwareVersion` |
| SMART (ver también Capítulo 4) | `Get-PhysicalDisk \| Get-StorageReliabilityCounter` |

### 2.1 NVMe

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| `/dev/nvme*`, namespaces | Windows reconoce los discos NVMe de forma nativa desde Windows Server 2012 R2/Windows 8.1; se identifican con `Get-PhysicalDisk` mostrando `BusType : NVMe` |
| Comprobación del tipo de bus/medio | `Get-PhysicalDisk \| Select FriendlyName, MediaType, BusType` (`MediaType` distingue HDD/SSD, `BusType` distingue SATA/SAS/NVMe/USB) |

---

## 3. iSCSI

### 3.1 Configurar el servidor destino (target): rol "iSCSI Target Server"

Equivalente a `targetcli` en Linux.

| Paso | Comando/cmdlet PowerShell |
|---|---|
| 1. Instalar el rol | `Install-WindowsFeature FS-iSCSITarget-Server` |
| 2. Crear un disco virtual iSCSI (equivalente al backstore de `targetcli`) | `New-IscsiVirtualDisk -Path "E:\iSCSIVirtualDisks\disco1.vhdx" -Size 100GB` |
| 3. Crear el target (con su IQN implícito) | `New-IscsiServerTarget -TargetName "T1" -InitiatorIds "IQN:iqn.1991-05.com.microsoft:cliente01"` |
| 4. Asociar el disco virtual al target | `Add-IscsiVirtualDiskTargetMapping -TargetName "T1" -Path "E:\iSCSIVirtualDisks\disco1.vhdx"` |
| 5. Ver los targets configurados | `Get-IscsiServerTarget \| Format-List TargetName, LunMappings` |
| 6. (Opcional) Autenticación CHAP | `Set-IscsiServerTarget -TargetName "T1" -ChapUserName "usuario" -ChapSecret "clave" -EnableChap $true` |

### 3.2 Configurar el cliente iniciador (initiator)

Equivalente a `iscsiadm` en Linux.

| Elemento | Equivalente en Windows |
|---|---|
| `iscsiadm` (cliente de línea de comandos) | **`iscsicli`** (línea de comandos clásica) o los cmdlets PowerShell del módulo `iSCSI` |
| Aplicación gráfica de configuración | **iSCSI Initiator** (`iscsicpl.exe`), disponible incluso en ediciones Server Core |
| Servicio necesario | **MSiSCSI** (Microsoft iSCSI Initiator Service): `Start-Service MSiSCSI` / `Set-Service MSiSCSI -StartupType Automatic` |

**Comandos principales (PowerShell):**

| Comando | Función |
|---|---|
| `New-IscsiTargetPortal -TargetPortalAddress IP_del_target` | Descubre los targets disponibles en un servidor (equivalente a `iscsiadm -m discovery`) |
| `Get-IscsiTarget` | Lista los targets descubiertos |
| `Connect-IscsiTarget -NodeAddress IQN -IsPersistent $true` | Inicia sesión (login) contra un target y hace la conexión persistente entre reinicios (equivalente a `iscsiadm -m node -l`) |
| `Get-IscsiSession` | Muestra las sesiones iSCSI activas (equivalente a `iscsiadm -m session`) |
| `Connect-IscsiTarget ... -AuthenticationType ONEWAYCHAP -ChapUsername usuario -ChapSecret clave` | Conexión autenticada con CHAP |

Tras la conexión, el disco iSCSI aparece como un disco más en `Get-Disk` (con `BusType : iSCSI`), listo para inicializar, particionar y formatear igual que cualquier disco local — el mismo comportamiento que en Linux, donde aparece como `/dev/sdX` tras el login.

**Multipath (equivalente a la redundancia de rutas SAN):**

| Elemento Linux | Equivalente en Windows |
|---|---|
| `multipath-tools`/`dm-multipath` | **MPIO** (Multipath I/O), rol/característica: `Install-WindowsFeature MultiPath-IO` + `Enable-MSDSMAutomaticClaim -BusType iSCSI` |

---

## 4. Gestión de volúmenes lógicos (equivalente a LVM)

Windows Server ofrece dos caminos según la necesidad:

| Necesidad | Herramienta equivalente |
|---|---|
| Volúmenes lógicos con RAID software combinado, ampliables en caliente (equivalente completo a LVM + `mdadm`) | **Storage Spaces** (ver sección 1) — es la tecnología recomendada hoy en día |
| Volúmenes simples abarcando varios discos, sin las funciones avanzadas de Storage Spaces (tecnología más antigua) | **Discos dinámicos** (Dynamic Disks) y el **Logical Disk Manager**, gestionables con `diskpart` o el Administrador de discos — considerado heredado (legacy), Microsoft recomienda Storage Spaces para todo despliegue nuevo |

### 4.1 Correspondencia conceptual con los términos de LVM

| Término LVM | Equivalente en Storage Spaces |
|---|---|
| PV (Physical Volume) | Disco físico añadido a un **Storage Pool** (`Add-PhysicalDisk`) |
| VG (Volume Group) | **Storage Pool** (`New-StoragePool`) |
| LV (Logical Volume) | **Disco virtual** (Virtual Disk, `New-VirtualDisk`), sobre el que después se crea el volumen/partición formateada |
| PE / LE (extents físicos/lógicos) | Windows no expone extents de forma directa al administrador; la asignación de espacio dentro del pool se gestiona internamente |
| Snapshot de LV (COW) | Instantáneas de **VSS** (Volume Shadow Copy Service, ver Capítulo 2), o snapshots nativos de **ReFS** (block cloning, ver Capítulo 4) |
| `lvextend`/`vgextend` | `Resize-VirtualDisk` / `Add-PhysicalDisk` + `Resize-Partition` |
| `lvrename` | `Get-VirtualDisk -FriendlyName nombre \| Set-VirtualDisk -NewFriendlyName nuevo_nombre` |
| `lvremove` | `Remove-VirtualDisk` |
| Device Mapper | Windows no expone una capa equivalente visible al administrador; el mapeo entre disco físico, pool y disco virtual lo gestiona internamente el subsistema Storage Spaces (`spaceport.sys`) |

---

## 5. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 5) | Equivalente en Windows Server 2025 |
|---|---|
| `mdadm` (RAID software) | Storage Spaces (`New-StoragePool`, `New-VirtualDisk`) |
| Niveles RAID 0/1/5/6/10 | Resiliencia Simple/Mirror/Parity (con redundancia simple o doble) |
| `/proc/mdstat`, `mdadm --detail` | `Get-VirtualDisk`, `Get-StoragePool` |
| `hdparm`, `sdparm` | `Get-PhysicalDisk`, `Get-Disk`, DiskSpd/Winsat para benchmarks |
| `targetcli` (servidor iSCSI) | Rol **FS-iSCSITarget-Server**, `New-IscsiServerTarget`, `New-IscsiVirtualDisk` |
| `iscsiadm` (cliente iSCSI) | `iscsicli`, `iscsicpl.exe`, cmdlets `*-IscsiTarget` |
| `multipath-tools` | **MPIO** (Multipath I/O) |
| LVM (PV/VG/LV) | Storage Spaces (discos físicos/Storage Pool/disco virtual); alternativa heredada: discos dinámicos |
| Snapshot de LV | VSS, snapshots/block cloning de ReFS |
| Device Mapper | Sin equivalente expuesto al administrador (gestión interna de Storage Spaces) |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 5 ("Administering Advanced Storage Devices") del libro LPIC-2. Fuentes: conocimiento general de administración de Windows Server, verificado con documentación oficial de Microsoft Learn (Storage Spaces, `New-VirtualDisk`, `New-StoragePool`, `New-IscsiServerTarget`, cmdlets del módulo iSCSI para Windows Server 2025).*
