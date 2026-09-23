# LPIC-2 · Capítulo 5: Administering Advanced Storage Devices
### Equivalencias en Windows Server 2025

Este documento recoge, para cada concepto visto en el Capítulo 5 del libro LPIC-2 (RAID por software, ajuste de discos, NVMe, iSCSI y gestión de volúmenes lógicos), cuál es su equivalente en **Windows Server 2025**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo: en Linux, RAID (`mdadm`) y LVM son dos herramientas separadas. En Windows Server, las dos funciones las cubre una sola tecnología: **Espacios de almacenamiento** (Storage Spaces). Funciona en tres capas, muy parecidas a las de LVM:
- **Discos físicos** (equivalente a los PV de LVM).
- **Grupo de almacenamiento** o *pool* (equivalente a un VG): une varios discos en un único "depósito" de espacio.
- **Discos virtuales** (equivalente a los LV): se sacan del grupo, y al crearlos se elige su **resistencia** (sin protección, espejo o paridad), que es el equivalente a elegir el nivel de RAID. Después se particionan y formatean como un disco normal (Capítulo 4).

Existen además otras dos opciones:
- **Discos dinámicos**: el sistema antiguo de Windows para RAID y volúmenes que ocupan varios discos. Microsoft los considera **obsoletos** y recomienda Espacios de almacenamiento, pero siguen existiendo (se resumen en el apartado 4.9).
- **Espacios de almacenamiento directos** (Storage Spaces Direct, S2D): la misma idea, pero uniendo los discos locales de **varios servidores** de un clúster (solo en la edición Datacenter). Es parecido a Ceph o GlusterFS en Linux.

Todo se gestiona con el módulo **Storage** de PowerShell, desde el **Administrador del servidor** (Servicios de archivos y almacenamiento → Grupos de almacenamiento) o desde **Windows Admin Center**. Los comandos se ejecutan en una consola **como Administrador**.

---

## 1. RAID

### 1.1 Niveles de RAID

| Nivel | Equivalente en Espacios de almacenamiento | Mínimo de discos | Tolerancia a fallos | Equivalente en discos dinámicos (obsoleto) |
|---|---|---|---|---|
| RAID 0 | **Simple** (`-ResiliencySettingName Simple`) | 1 | No | Volumen seccionado (*striped*) |
| RAID 1 | **Reflejo doble** (`Mirror`, `-PhysicalDiskRedundancy 1`) | 2 | 1 disco | Volumen reflejado (*mirrored*) |
| — | **Reflejo triple** (`Mirror`, `-PhysicalDiskRedundancy 2`) | 5 | 2 discos | — |
| RAID 2, 3, 4 | No existen | — | — | — |
| RAID 5 | **Paridad simple** (`Parity`, `-PhysicalDiskRedundancy 1`) | 3 | 1 disco | Volumen RAID-5 |
| RAID 6 | **Paridad doble** (`Parity`, `-PhysicalDiskRedundancy 2`) | 7 | 2 discos | — |
| RAID 10 | **Reflejo** con varias columnas: Espacios de almacenamiento reparte los datos entre varios pares de espejo automáticamente cuando hay discos suficientes | 4 | 1 disco por copia | — |

Una diferencia importante con RAID clásico: Espacios de almacenamiento no reserva discos enteros para cada copia, sino que reparte **fragmentos** de datos entre todos los discos del grupo. Por eso se pueden mezclar discos de distinto tamaño y crear varios discos virtuales con distinta resistencia en el mismo grupo.

### 1.2 Implementación de RAID software (equivalente a `mdadm`)

Espacios de almacenamiento viene **integrado** en Windows Server 2025; no hay que instalar nada. Su controlador del kernel es `spaceport.sys` (el equivalente al controlador `md`).

**Comprobaciones previas:**

| Comando Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `cat /proc/mdstat` (soporte) | `Get-StorageSubSystem` | Muestra el subsistema de almacenamiento ("Windows Storage on NOMBRE") |
| `modprobe raid6` | No hace falta | Todos los niveles están siempre disponibles |
| `dpkg -s mdadm` / `rpm -q mdadm` | No hace falta | Integrado en el sistema |
| — | `Get-PhysicalDisk -CanPool $true` | Lista los discos que se pueden añadir a un grupo |
| — | `Get-StoragePool -IsPrimordial $true` | El grupo "primordial": contiene automáticamente todos los discos disponibles sin usar |

**Preparar los discos:** no hace falta particionarlos ni poner códigos de tipo de partición. Al contrario, los discos deben estar **vacíos**: sin particiones y sin ser el disco del sistema. Si un disco tiene datos o restos de otro grupo:

| Comando | Función |
|---|---|
| `Clear-Disk -Number 2 -RemoveData -RemoveOEM` | Borra todas las particiones del disco 2 |
| `Reset-PhysicalDisk -FriendlyName "PhysicalDisk2"` | Borra los metadatos de Espacios de almacenamiento que queden en el disco (equivale a `mdadm --zero-superblock`) |

Si `CanPool` es `False`, la columna `CannotPoolReason` indica el motivo (tiene particiones, es el disco de arranque, no tiene espacio suficiente...).

### 1.3 Modos de `mdadm` y sus equivalentes (equivalente a la Tabla 5.1)

| Modo `mdadm` | Equivalente en Windows Server 2025 |
|---|---|
| `--create` | `New-StoragePool` (crear el grupo) + `New-VirtualDisk` o `New-Volume` (crear el "array" con su nivel) |
| `--assemble` | Automático: los metadatos están en los propios discos y Windows reconoce el grupo al arrancar. Si se mueven los discos a otro servidor, el grupo puede aparecer como **solo lectura**: `Set-StoragePool -FriendlyName Pool1 -IsReadOnly $false` |
| `--autodetect` | Automático |
| `--build` | No existe |
| `--follow` / `--monitor` | Estado con `Get-VirtualDisk` y `Get-PhysicalDisk`, y alertas por el Visor de eventos (apartado 1.8) |
| `--grow` | `Add-PhysicalDisk` (más discos) + `Resize-VirtualDisk` (más tamaño). **No se puede cambiar el nivel** de un disco virtual ya creado |
| `--incremental` | `Add-PhysicalDisk` |
| `--manage` | `Add-PhysicalDisk`, `Remove-PhysicalDisk`, `Set-PhysicalDisk -Usage Retired`, `Repair-VirtualDisk` |
| `--misc` | `Reset-PhysicalDisk` (borrar metadatos), `Get-*` (consultas) |

### 1.4 Crear un array

Equivalente al ejemplo del libro (`mdadm -C /dev/md0 -l 6 -n 4 ...`), en dos pasos:

**1. Crear el grupo** con todos los discos disponibles:
```
New-StoragePool -FriendlyName Pool1 -StorageSubSystemFriendlyName "Windows Storage*" -PhysicalDisks (Get-PhysicalDisk -CanPool $true)
```

**2. Crear el disco virtual** con el nivel deseado. Forma rápida, con `New-Volume`, que crea el disco virtual, lo inicializa, lo particiona, lo formatea y le da letra en un solo paso:
```
New-Volume -StoragePoolFriendlyName Pool1 -FriendlyName Datos -ResiliencySettingName Parity -PhysicalDiskRedundancy 1 -Size 500GB -FileSystem ReFS -DriveLetter E
```

Forma detallada (paso a paso, como en Linux: crear el array y después formatearlo):
```
New-VirtualDisk -StoragePoolFriendlyName Pool1 -FriendlyName Datos -ResiliencySettingName Mirror -Size 500GB -ProvisioningType Fixed
Get-VirtualDisk -FriendlyName Datos | Get-Disk | Initialize-Disk -PassThru | New-Partition -AssignDriveLetter -UseMaximumSize | Format-Volume -FileSystem ReFS
```

**Opciones de creación** (equivalentes a las de `mdadm`):

| Opción `mdadm` | Equivalente en `New-VirtualDisk` | Función |
|---|---|---|
| `-l` / `--level` | `-ResiliencySettingName Simple \| Mirror \| Parity` + `-PhysicalDiskRedundancy 1 \| 2` | Nivel de RAID |
| `-n` / `--raid-devices` | `-NumberOfColumns` | Número de discos entre los que se reparte cada escritura |
| `-c` / `--chunk` | `-Interleave` | Tamaño de cada fragmento (256 KB por defecto) |
| `-x` / `--spare-devices` | `Add-PhysicalDisk ... -Usage HotSpare` (apartado 1.7) | Discos de repuesto |
| — | `-Size 500GB` o `-UseMaximumSize` | Tamaño del disco virtual |
| — | `-ProvisioningType Fixed \| Thin` | Reservar todo el espacio desde el principio (`Fixed`) o ir ocupándolo según se usa (`Thin`, "aprovisionamiento fino") |

Ayuda de cualquier cmdlet: `Get-Help New-VirtualDisk -Detailed` o `Get-Help New-VirtualDisk -Examples`.

### 1.5 Comprobar y consultar un array

| Comando Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `cat /proc/mdstat` | `Get-VirtualDisk \| Select FriendlyName, ResiliencySettingName, HealthStatus, OperationalStatus, Size` | Estado de todos los discos virtuales |
| `watch cat /proc/mdstat` (ver reconstrucción) | `Get-StorageJob` | Trabajos en curso (reparación, reequilibrado) con su porcentaje completado |
| `mdadm --detail /dev/md0` | `Get-VirtualDisk -FriendlyName Datos \| Format-List *` | Todos los datos de un disco virtual |
| — | `Get-VirtualDisk -FriendlyName Datos \| Get-PhysicalDisk` | Discos físicos que usa ese disco virtual |
| `mdadm --examine /dev/sdX1` | `Get-PhysicalDisk \| Select FriendlyName, CanPool, Usage, HealthStatus, OperationalStatus` | Estado de cada disco y si pertenece a un grupo (`CanPool = False` y `Usage = Auto-Select` indican que ya está en uso) |
| — | `Get-StoragePool -FriendlyName Pool1 \| Get-PhysicalDisk` | Discos de un grupo |

**Estados habituales:**

| Propiedad | Valores | Significado |
|---|---|---|
| `HealthStatus` | `Healthy` / `Warning` / `Unhealthy` | Sano / en peligro (sin redundancia completa) / sin acceso a los datos |
| `OperationalStatus` (disco virtual) | `OK` / `Degraded` / `InService` / `Detached` | Correcto / falta algún disco / reparándose / desconectado |
| `OperationalStatus` (disco físico) | `OK` / `Lost Communication` / `Predictive Failure` | Correcto / no responde / SMART predice un fallo |

### 1.6 Guardar la configuración (equivalente a `mdadm.conf`)

**No hace falta.** Toda la configuración del grupo y de los discos virtuales se guarda como **metadatos en cada uno de los discos del grupo**. Windows los lee al arrancar y reconstruye todo solo. No existe un archivo de configuración que haya que mantener.

### 1.7 Gestión del array

| Tarea | Comando en Windows Server 2025 |
|---|---|
| Formatear y montar | Como cualquier disco (Capítulo 4): `Initialize-Disk`, `New-Partition`, `Format-Volume`. O directamente con `New-Volume` |
| Añadir un disco al grupo | `Add-PhysicalDisk -StoragePoolFriendlyName Pool1 -PhysicalDisks (Get-PhysicalDisk -FriendlyName PhysicalDisk5)` |
| Añadir un disco de repuesto | Lo mismo con `-Usage HotSpare` al final |
| Marcar un disco averiado para sustituirlo | `Set-PhysicalDisk -FriendlyName PhysicalDisk3 -Usage Retired` |
| Reconstruir los datos en los discos que quedan | `Repair-VirtualDisk -FriendlyName Datos` (seguir el progreso con `Get-StorageJob`) |
| Quitar el disco averiado del grupo | `Remove-PhysicalDisk -StoragePoolFriendlyName Pool1 -PhysicalDisks (Get-PhysicalDisk -FriendlyName PhysicalDisk3)` |
| Repartir los datos tras añadir discos | `Optimize-StoragePool -FriendlyName Pool1` |
| Ampliar un disco virtual | `Resize-VirtualDisk -FriendlyName Datos -Size 1TB` y después ampliar la partición con `Resize-Partition` (Capítulo 4) |
| Parar el array (equivalente a `--stop`) | `Disconnect-VirtualDisk -FriendlyName Datos` (se vuelve a activar con `Connect-VirtualDisk`) |
| Eliminar | `Remove-VirtualDisk -FriendlyName Datos` y, si ya no se usa, `Remove-StoragePool -FriendlyName Pool1` |

Consejo de Microsoft: en lugar de dejar discos de repuesto parados, es mejor dejar **espacio sin asignar** en el grupo (el equivalente a la capacidad de un disco o dos). Así, cuando falla un disco, la reparación usa ese espacio libre repartido entre todos los discos, y es más rápida.

Por defecto, cuando un disco falla, Windows **no repara solo** en un servidor independiente hasta que se sustituye o se retira el disco. Ese comportamiento se ajusta con `Set-StoragePool -RetireMissingPhysicalDisks Always`.

### 1.8 Monitorización (equivalente a `mdadm --monitor`)

| Función de `mdadm --monitor` | Equivalente en Windows Server 2025 |
|---|---|
| Vigilar los arrays | Registro de eventos **`Microsoft-Windows-StorageSpaces-Driver/Operational`** (Visor de eventos → Registros de aplicaciones y servicios → Microsoft → Windows → StorageSpaces-Driver) |
| `--syslog` | Automático: los eventos siempre quedan en ese registro |
| `--program=` (ejecutar algo ante un evento) | **Programador de tareas** con desencadenador "Al producirse un evento" (o, desde el Visor de eventos, clic derecho en el evento → **Adjuntar tarea a este evento**) |
| `--mail=` | No hay envío de correo integrado (el servidor SMTP de Windows se quitó en Windows Server 2025). Se envía desde un script lanzado por la tarea anterior, o con herramientas de monitorización |
| `--oneshot` | `Get-VirtualDisk` / `Get-PhysicalDisk` (una consulta) |
| `--daemonise` | No hace falta: Windows vigila los discos siempre |
| Monitorización centralizada | Windows Admin Center, System Center Operations Manager, Azure Monitor (Capítulo 2) |

**Eventos de `mdadm` (Tabla 5.2) y cómo se ven en Windows:**

| Evento `mdadm` | Equivalente en Windows Server 2025 |
|---|---|
| `DegradedArray` | Disco virtual con `OperationalStatus = Degraded` y `HealthStatus = Warning` |
| `Fail`, `DeviceDisappeared` | Disco físico con `OperationalStatus = Lost Communication` o `HealthStatus = Unhealthy` |
| `RebuildStarted`, `RebuildNN`, `RebuildFinished` | Trabajo de reparación visible con `Get-StorageJob` (porcentaje y estado) |
| `SpareActive` | Un disco con `Usage = HotSpare` pasa a usarse |
| `NewArray` | Nuevo disco virtual creado (evento en el registro de StorageSpaces) |

En clústeres con **Espacios de almacenamiento directos**, además existe el **Servicio de mantenimiento** (Health Service), que detecta problemas y los muestra con `Get-HealthFault`.

---

## 2. Ajuste del acceso a dispositivos de almacenamiento

### 2.1 Equivalente a `hdparm` (discos SATA)

| Comando Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `hdparm -I` | `Get-PhysicalDisk -FriendlyName PhysicalDisk1 \| Format-List *` y `Get-Disk -Number 1 \| Format-List *` | Todos los datos del disco (modelo, bus, tamaño de sector, firmware...) |
| — | `Get-StorageFirmwareInformation -FriendlyName PhysicalDisk1` | Versión del firmware (y `Update-StorageFirmware` para actualizarlo) |
| `hdparm -W` (caché de escritura) | Ver: `Get-PhysicalDisk \| Get-StorageAdvancedProperty` (`IsDeviceCacheEnabled`). Cambiar: Administrador de dispositivos → propiedades del disco → pestaña **Directivas** → "Habilitar caché de escritura en el dispositivo" | Consultar o activar la caché de escritura del disco |
| `hdparm -t` / `-T` (rendimiento) | **DiskSpd** (herramienta gratuita de Microsoft, se descarga de su repositorio en GitHub). Ejemplo: `diskspd -c1G -d30 -r -w0 -t4 -o8 -b64K -Sh -L E:\prueba.dat` (30 s de lectura aleatoria en bloques de 64 KB) | Prueba de rendimiento |

### 2.2 Equivalente a `sdparm` (dispositivos SCSI)

Los datos de las tablas VPD (número de serie, modelo, identificador único) se consultan con los mismos cmdlets:

| Comando | Función |
|---|---|
| `Get-PhysicalDisk \| Select FriendlyName, Model, SerialNumber, FirmwareVersion, BusType, MediaType` | Datos de identificación de cada disco |
| `Get-Disk \| Select Number, UniqueId, UniqueIdFormat, SerialNumber, Location` | Identificador único (WWN, EUI-64...) y ubicación física |
| `powercfg /change disk-timeout-ac 0` | Tiempo tras el que se paran los discos inactivos (`0` = nunca). En servidores conviene que no se paren |

### 2.3 `sysctl` y parámetros del kernel relacionados con almacenamiento

Igual que en el Capítulo 3, el equivalente son valores del **Registro** y algunos cmdlets:

| Parámetro | Dónde | Función |
|---|---|---|
| `TimeOutValue` | `HKLM\SYSTEM\CurrentControlSet\Services\disk` | Segundos que Windows espera a un disco antes de dar error (equivale a `/sys/block/sdX/device/timeout`). Se suele aumentar con discos SAN o iSCSI |
| Configuración de rutas múltiples | `Get-MPIOSetting` / `Set-MPIOSetting` | Tiempos y reintentos de MPIO (apartado 3.2) |
| Comportamiento de NTFS y caché | `fsutil behavior` | Capítulo 3 |

### 2.4 NVMe

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| Soporte en el kernel | Controlador integrado **`stornvme.sys`** (StorNVMe) |
| `/dev/nvme*` | No hay archivos de dispositivo: cada unidad NVMe aparece como un disco más (Disco 0, Disco 1...). Se distingue por su tipo de bus: `Get-PhysicalDisk \| Where-Object BusType -eq NVMe` |
| Namespaces | Cada namespace aparece como un **disco independiente** |
| `/dev/nvme2n4p1` | Disco N, partición M (no hay una nomenclatura que indique el namespace) |
| `nvme smart-log` | `Get-PhysicalDisk \| Where BusType -eq NVMe \| Get-StorageReliabilityCounter` (temperatura, desgaste, errores) |
| `nvme fw-download` | `Update-StorageFirmware` |

**NVMe nativo (novedad de 2025):** históricamente Windows trataba los discos NVMe como si fueran SCSI, traduciendo las órdenes. Microsoft ha publicado para Windows Server 2025 una nueva pila de almacenamiento **NVMe nativa**, sin esa traducción, que da más rendimiento y usa menos CPU. Viene incluida en las actualizaciones acumulativas, pero **desactivada por defecto**: se activa con un valor de Registro o una directiva, según indica el anuncio oficial de Microsoft. Conviene probarla antes de usarla en producción.

**NVMe sobre red (NVMe-oF):** en Windows Server 2025 no hay un iniciador NVMe-oF integrado; Microsoft lo está probando en las versiones de prueba (Insider) de la próxima versión de Windows Server. Para discos en red se sigue usando iSCSI o Fibre Channel.

### 2.5 SMART

Ver Capítulo 4, apartado 8.4 (`Get-PhysicalDisk`, `Get-StorageReliabilityCounter`).

### 2.6 Identificadores de dispositivos SCSI/iSCSI

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| LUN | `Get-Disk \| Select Number, FriendlyName, Location` (la ubicación incluye el número de LUN) o `diskpart` → `select disk 2` → `detail disk` (línea "Id. de LUN") |
| WWID / WWN (`/dev/disk/by-id`) | `Get-Disk \| Select Number, UniqueId, UniqueIdFormat`. El formato indica el tipo de identificador (por ejemplo, `FCPH Name` = WWN, `EUI64`) |
| WWN de la tarjeta Fibre Channel del servidor | `Get-InitiatorPort \| Where ConnectionType -eq 'Fibre Channel' \| Select NodeAddress, PortAddress` |
| `scsi_id` | `Get-Disk \| Select UniqueId` |
| IQN del iniciador | `(Get-InitiatorPort \| Where ConnectionType -eq 'iSCSI').NodeAddress`. Formato por defecto: `iqn.1991-05.com.microsoft:nombre-del-servidor` |
| IQN del destino | Lo asigna el servidor de destino iSCSI (apartado 3.1) |

---

## 3. iSCSI

Windows Server 2025 incluye las dos partes: el **servidor de destino iSCSI** (rol que hay que instalar) y el **iniciador iSCSI** (integrado).

### 3.1 Configurar el servidor destino (target): Servidor de destino iSCSI

Diferencia importante con `targetcli`: en Windows el almacenamiento que se ofrece por iSCSI son siempre **discos virtuales en archivos `.vhdx`**, no discos o particiones físicas directamente.

| Paso en Linux (`targetcli`) | Equivalente en Windows Server 2025 |
|---|---|
| Instalar el paquete `targetcli` | `Install-WindowsFeature FS-iSCSITarget-Server -IncludeManagementTools` |
| `systemctl enable target` | El servicio (`WinTarget`) se configura como automático al instalar el rol |
| Crear un *backstore* | `New-IscsiVirtualDisk -Path E:\iSCSI\disco1.vhdx -SizeBytes 100GB` (añadir `-UseFixed` para reservar todo el espacio desde el principio) |
| Crear el IQN del destino | `New-IscsiServerTarget -TargetName Target1 -InitiatorIds "IQN:iqn.1991-05.com.microsoft:srv02.empresa.local"`. El IQN del destino se genera solo: `iqn.1991-05.com.microsoft:nombreservidor-target1-target` |
| Asignar el *backstore* al destino (LUN) | `Add-IscsiVirtualDiskTargetMapping -TargetName Target1 -Path E:\iSCSI\disco1.vhdx` |
| Lista de control de acceso (ACL) | Parámetro `-InitiatorIds` (por IQN, por dirección IP con `IPAddress:192.168.1.20`, o por nombre DNS con `DNSName:srv02.empresa.local`). Se cambia con `Set-IscsiServerTarget` |
| Autenticación CHAP | `Set-IscsiServerTarget -TargetName Target1 -EnableChap $true -Chap (Get-Credential)` |
| `ls` (ver el resultado) | `Get-IscsiServerTarget` y `Get-IscsiVirtualDisk` |
| Herramienta gráfica | Administrador del servidor → Servicios de archivos y almacenamiento → **iSCSI** |

El rol abre automáticamente en el firewall el puerto **TCP 3260** (el puerto estándar de iSCSI).

### 3.2 Configurar el cliente iniciador (initiator)

El iniciador está **integrado** en Windows (servicio **Iniciador iSCSI de Microsoft**, `MSiSCSI`). No hay que instalar nada, solo arrancar el servicio:
```
Set-Service MSiSCSI -StartupType Automatic
Start-Service MSiSCSI
```

| Elemento Linux | Equivalente en Windows Server 2025 |
|---|---|
| Paquete `open-iscsi` / `iscsi-initiator-utils` | Integrado |
| Demonio `iscsid` | Servicio `MSiSCSI` |
| `/etc/iscsi/iscsid.conf` | No hay archivo: la configuración se guarda en el Registro y se cambia con cmdlets |
| `/etc/iscsi/initiatorname.iscsi` | Nombre del iniciador: se ve con `Get-InitiatorPort` y se cambia con `Set-InitiatorPort -NodeAddress "nombre-actual" -NewNodeAddress "iqn.2026-09.local.empresa:srv02"` |
| `/var/lib/iscsi/` (base de datos) | Registro de Windows |
| Herramienta gráfica | `iscsicpl` (Propiedades del iniciador iSCSI) |

**Comandos principales:**

| Comando Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `iscsiadm -m discovery -t st -p IP` | `New-IscsiTargetPortal -TargetPortalAddress 192.168.1.10` y después `Get-IscsiTarget` | Descubre los destinos del servidor |
| `iscsiadm -m node -T IQN -p IP -l` | `Connect-IscsiTarget -NodeAddress "iqn.1991-05.com.microsoft:srv01-target1-target" -IsPersistent $true` | Inicia sesión. `-IsPersistent $true` hace que se reconecte en cada arranque |
| `iscsiadm -m session -P3` | `Get-IscsiSession` y `Get-IscsiConnection` | Información de las sesiones activas |
| `iscsiadm -m node -u` | `Disconnect-IscsiTarget -NodeAddress "..."` | Cierra la sesión |
| — | `Remove-IscsiTargetPortal -TargetPortalAddress 192.168.1.10` | Olvida el servidor de destino |

También existe la herramienta clásica de texto `iscsicli` (por ejemplo, `iscsicli QAddTargetPortal 192.168.1.10`, `iscsicli ListTargets`, `iscsicli QLoginTarget iqn...`).

**Después del inicio de sesión:** el disco iSCSI aparece como un **disco nuevo**. Por la directiva de Windows Server para discos compartidos (Capítulo 4), suele aparecer **sin conexión**, así que hay que ponerlo en línea, inicializarlo y formatearlo:
```
Get-Disk | Where-Object BusType -eq iSCSI
Set-Disk -Number 3 -IsOffline $false
Initialize-Disk -Number 3 -PartitionStyle GPT
New-Partition -DiskNumber 3 -UseMaximumSize -AssignDriveLetter | Format-Volume -FileSystem NTFS
```
No hace falta nada parecido a `/etc/fstab`: con la sesión persistente, Windows vuelve a conectar el disco y le da la misma letra en cada arranque. Si algún servicio necesita que el volumen iSCSI esté disponible **antes** de arrancar, se usa la pestaña **Volúmenes y dispositivos** → "Configurar automáticamente" de `iscsicpl` (o `iscsicli BindPersistentVolumes`).

**Rutas múltiples (MPIO)**, equivalente a `dm-multipath` en Linux (varias conexiones de red al mismo disco, por rendimiento y redundancia):

| Comando | Función |
|---|---|
| `Install-WindowsFeature Multipath-IO` | Instala la característica (requiere reiniciar) |
| `Enable-MSDSMAutomaticClaim -BusType iSCSI` | Hace que MPIO gestione los discos iSCSI automáticamente |
| `Connect-IscsiTarget -NodeAddress "..." -IsMultipathEnabled $true -InitiatorPortalAddress IP_local -TargetPortalAddress IP_destino` | Conecta cada ruta (una vez por cada tarjeta de red) |
| `mpclaim -s -d` | Muestra los discos gestionados por MPIO y su directiva de reparto de carga |

---

## 4. Gestión de volúmenes lógicos (equivalente a LVM)

El equivalente de LVM en Windows Server 2025 es de nuevo **Espacios de almacenamiento**. Un disco virtual con resistencia **Simple** es exactamente un volumen lógico de LVM sin RAID; con **Mirror** o **Parity**, es un volumen lógico con RAID integrado.

### 4.1 Conceptos

| Elemento LVM | Equivalente en Espacios de almacenamiento | Descripción |
|---|---|---|
| PV (Physical Volume) | **Disco físico** del grupo | Un disco completo (no una partición) añadido a un grupo |
| VG (Volume Group) | **Grupo de almacenamiento** (*storage pool*) | Conjunto de discos que forman un único depósito de espacio |
| LV (Logical Volume) | **Disco virtual** (*virtual disk*, también llamado "espacio") | Porción del grupo, que Windows ve como un disco normal; dentro se crean particiones y volúmenes |
| PE (Physical Extent) | **Bloque de asignación** (*slab*), de 256 MB en Espacios de almacenamiento | Unidad mínima en que se reparte el espacio de los discos |
| LE (Logical Extent) | No se usa este concepto | — |

Relación: igual que en LVM. Un disco físico pertenece a un único grupo; un grupo puede tener varios discos virtuales; un disco virtual pertenece a un único grupo.

Diferencia: en LVM, el LV se formatea directamente. En Windows, el disco virtual se comporta como un **disco** entero, así que dentro hay que crear una partición y formatearla (o usar `New-Volume`, que lo hace todo).

### 4.2 Crear la estructura completa

| Paso LVM | Equivalente en Windows Server 2025 |
|---|---|
| 1. `pvcreate /dev/sdj1` | No hace falta: basta con que los discos estén vacíos (`Get-PhysicalDisk -CanPool $true`) |
| 2. `vgdisplay` (ver lo que existe) | `Get-StoragePool` |
| 3. `vgcreate vg00 /dev/sdj1 ...` | `New-StoragePool -FriendlyName vg00 -StorageSubSystemFriendlyName "Windows Storage*" -PhysicalDisks (Get-PhysicalDisk -CanPool $true)` |
| 4. `lvcreate -L 2g -n lvol0 vg00` | `New-VirtualDisk -StoragePoolFriendlyName vg00 -FriendlyName lvol0 -ResiliencySettingName Simple -Size 2GB` |
| 5. `mkfs` + `mount` | `Get-VirtualDisk lvol0 \| Get-Disk \| Initialize-Disk -PassThru \| New-Partition -AssignDriveLetter -UseMaximumSize \| Format-Volume -FileSystem NTFS` |
| Pasos 4 y 5 juntos | `New-Volume -StoragePoolFriendlyName vg00 -FriendlyName lvol0 -ResiliencySettingName Simple -Size 2GB -FileSystem NTFS -DriveLetter F` |

**Aprovisionamiento fino** (equivalente a los *thin volumes* de LVM): con `-ProvisioningType Thin`, el disco virtual puede ser más grande que el espacio real del grupo y solo ocupa lo que se va escribiendo. Hay que vigilar que el grupo no se llene.

### 4.3 Interfaz interactiva `lvm`

No hay una shell propia de Espacios de almacenamiento. Las alternativas son:
- **PowerShell**: todos los cmdlets del módulo Storage. Para verlos: `Get-Command -Module Storage`.
- **`diskpart`**: shell interactiva de discos (útil para discos básicos y dinámicos, pero no crea grupos de almacenamiento).
- **Administrador del servidor** y **Windows Admin Center**: interfaz gráfica.

### 4.4 Comandos de consulta

| Comando LVM | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `pvdisplay`, `pvs` | `Get-PhysicalDisk` / `Get-StoragePool -FriendlyName vg00 \| Get-PhysicalDisk` | Discos físicos (de todos o de un grupo) |
| `vgdisplay`, `vgs` | `Get-StoragePool -IsPrimordial $false \| Select FriendlyName, Size, AllocatedSize, HealthStatus` | Grupos de almacenamiento, tamaño y espacio usado |
| `lvdisplay`, `lvs` | `Get-VirtualDisk` | Discos virtuales |
| `pvscan`, `vgscan`, `lvscan` | `Update-StorageProviderCache -DiscoveryLevel Full` | Vuelve a detectar discos, grupos y discos virtuales |
| `lvdisplay --maps` | `Get-VirtualDisk -FriendlyName lvol0 \| Get-PhysicalDisk` o `Get-PhysicalExtent -VirtualDisk (Get-VirtualDisk lvol0)` | Qué discos físicos (y qué bloques) respaldan a cada disco virtual |

### 4.5 Ampliar un volumen (crecimiento en caliente)

| Paso LVM | Equivalente en Windows Server 2025 |
|---|---|
| 1. `vgextend vg00 /dev/sdn1` | `Add-PhysicalDisk -StoragePoolFriendlyName vg00 -PhysicalDisks (Get-PhysicalDisk -CanPool $true)` |
| 2. `lvextend -L 4g /dev/vg00/lvol0` | `Resize-VirtualDisk -FriendlyName lvol0 -Size 4GB` |
| 3. `resize2fs` | `Resize-Partition -DriveLetter F -Size (Get-PartitionSupportedSize -DriveLetter F).SizeMax` (Capítulo 4) |

Todo se hace **en caliente**, sin desmontar el volumen.

Reducir (`lvreduce`): **los discos virtuales de Espacios de almacenamiento no se pueden reducir**. Solo se puede reducir la partición de dentro (en NTFS), pero el espacio no vuelve al grupo. Para recuperarlo hay que crear un disco virtual más pequeño y mover los datos.

### 4.6 Renombrar y eliminar

| Comando LVM | Equivalente en Windows Server 2025 |
|---|---|
| `lvrename` | `Set-VirtualDisk -FriendlyName lvol0 -NewFriendlyName datos` |
| `vgrename` | `Set-StoragePool -FriendlyName vg00 -NewFriendlyName Pool1` |
| `lvremove` | `Remove-VirtualDisk -FriendlyName lvol0` (se pierde todo su contenido) |
| `vgremove` | `Remove-StoragePool -FriendlyName vg00` (sin discos virtuales dentro) |
| `vgcfgbackup` / `vgcfgrestore` | No hacen falta: los metadatos se guardan **replicados en todos los discos** del grupo |

### 4.7 Snapshots

Los discos virtuales de Espacios de almacenamiento **no tienen instantáneas propias** como los snapshots de LVM. Las alternativas son:

| Tecnología | Descripción | Diferencia con los snapshots de LVM |
|---|---|---|
| **VSS** (Capítulo 2) | Instantáneas de volumen con copia al escribir (como LVM: al crearla solo se guardan metadatos y los bloques se copian cuando cambian) | Normalmente son de **solo lectura** |
| `diskshadow` | Permite crear una instantánea y mostrarla con una letra para leerla o copiarla (`add volume E: alias Copia` → `create` → `expose %Copia% X:`) | — |
| Clonación de bloques de ReFS | Copias instantáneas de archivos grandes (por ejemplo, discos de máquinas virtuales) | Por archivo, no por volumen |
| Puntos de control de Hyper-V | Instantáneas de una máquina virtual completa (`Checkpoint-VM`) | Solo para máquinas virtuales |

Igual que dice el libro de los snapshots de LVM: **ninguna de estas instantáneas sustituye a una copia de seguridad**, porque están en los mismos discos.

### 4.8 El Device Mapper → la pila de almacenamiento de Windows

En Linux, el Device Mapper es la base de LVM y RAID. En Windows ese papel lo reparten varios controladores de la **pila de almacenamiento**:

| Componente | Función | Parecido en Linux |
|---|---|---|
| `partmgr.sys` (Administrador de particiones) | Gestiona las particiones de los discos | Tabla de particiones del kernel |
| `volmgr.sys` (Administrador de volúmenes) | Crea los volúmenes sobre las particiones | Device Mapper |
| `spaceport.sys` | Espacios de almacenamiento (grupos y discos virtuales) | `md` + LVM |
| `volmgrx.sys` | Discos dinámicos (obsoletos) | `md` |
| `storport.sys` + miniport (`stornvme.sys`, `storahci.sys`...) | Comunicación con el hardware | Controladores SCSI/NVMe |

| Comando Linux | Equivalente en Windows Server 2025 |
|---|---|
| `dmsetup info` | `Get-VirtualDisk` y `Get-Volume` |
| `dmsetup info /dev/vg00/lvol0` | `Get-VirtualDisk -FriendlyName lvol0 \| Format-List *` |
| Nombres `/dev/mapper/VG-LV` | Nombre del disco virtual (`FriendlyName`) y ruta del volumen (`\\?\Volume{GUID}\`, Capítulo 4) |

Todos estos cmdlets usan por debajo la **API de administración de almacenamiento** de Windows, basada en CIM (espacio de nombres `root\Microsoft\Windows\Storage`), lo que permite gestionar igual discos locales, Espacios de almacenamiento y cabinas SAN compatibles.

### 4.9 Discos dinámicos (sistema antiguo, obsoleto)

Se incluyen como referencia porque aún existen en Windows Server 2025 y pueden encontrarse en servidores antiguos. Microsoft **no recomienda usarlos** para nada nuevo.

| Tipo de volumen dinámico | Equivalente Linux | Comando en `diskpart` |
|---|---|---|
| Simple | Partición normal | `create volume simple size=10240 disk=1` |
| Distribuido (*spanned*) | LV lineal que ocupa varios discos | `extend size=10240 disk=2` (sobre un volumen simple) |
| Seccionado (*striped*) | RAID 0 | `create volume stripe disk=1,2` |
| Reflejado (*mirrored*) | RAID 1 | `create volume mirror disk=1,2`, o `add disk=2` para reflejar un volumen existente |
| RAID-5 | RAID 5 | `create volume raid disk=1,2,3` |

Otros comandos: `convert dynamic` (convertir un disco básico en dinámico), `break disk=2` (romper un espejo), `repair disk=4` (reconstruir un RAID-5 con un disco nuevo).

---

## 5. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 5) | Equivalente en Windows Server 2025 |
|---|---|
| `mdadm` + LVM | Espacios de almacenamiento (módulo Storage de PowerShell) |
| Controlador `md` / Device Mapper | `spaceport.sys` / pila de almacenamiento (`volmgr.sys`, `partmgr.sys`) |
| RAID 0 / 1 / 5 / 6 / 10 | Simple / Reflejo / Paridad simple / Paridad doble / Reflejo con varias columnas |
| `mdadm --create` | `New-StoragePool` + `New-VirtualDisk` o `New-Volume` |
| `mdadm --assemble` | Automático |
| `/proc/mdstat` | `Get-VirtualDisk`, `Get-StorageJob` |
| `mdadm --detail` / `--examine` | `Get-VirtualDisk \| Format-List *` / `Get-PhysicalDisk` |
| `mdadm.conf` | No hace falta (metadatos en los discos) |
| `mdadm --add` (repuesto) | `Add-PhysicalDisk -Usage HotSpare` |
| `mdadm --fail` + `--remove` | `Set-PhysicalDisk -Usage Retired` + `Repair-VirtualDisk` + `Remove-PhysicalDisk` |
| `mdadm --grow` | `Add-PhysicalDisk` + `Resize-VirtualDisk` + `Optimize-StoragePool` |
| `mdadm --monitor` | Registro `StorageSpaces-Driver/Operational` + tareas programadas por evento |
| `mdadm --zero-superblock` | `Reset-PhysicalDisk` |
| `hdparm -I` | `Get-PhysicalDisk \| Format-List *` |
| `hdparm -W` | `Get-StorageAdvancedProperty` / Administrador de dispositivos |
| `hdparm -t` | DiskSpd |
| `sdparm` | `Get-PhysicalDisk`, `Get-Disk` |
| `/dev/nvme*` | Discos con `BusType = NVMe` (controlador `stornvme.sys`) |
| WWN / `/dev/disk/by-id` | `Get-Disk \| Select UniqueId` |
| `targetcli` | Rol Servidor de destino iSCSI (`New-IscsiVirtualDisk`, `New-IscsiServerTarget`) |
| `iscsiadm`, `iscsid` | Iniciador integrado (`MSiSCSI`, `New-IscsiTargetPortal`, `Connect-IscsiTarget`) |
| `initiatorname.iscsi` | `Get-InitiatorPort` / `Set-InitiatorPort` |
| `dm-multipath` | MPIO (`Multipath-IO`, `mpclaim`) |
| PV / VG / LV | Disco físico / Grupo de almacenamiento / Disco virtual |
| `pvcreate` + `vgcreate` | `New-StoragePool` |
| `lvcreate` | `New-VirtualDisk` / `New-Volume` |
| `pvs`, `vgs`, `lvs` | `Get-PhysicalDisk`, `Get-StoragePool`, `Get-VirtualDisk` |
| `vgextend` + `lvextend` + `resize2fs` | `Add-PhysicalDisk` + `Resize-VirtualDisk` + `Resize-Partition` |
| `lvreduce` | No es posible |
| `lvrename` / `lvremove` | `Set-VirtualDisk -NewFriendlyName` / `Remove-VirtualDisk` |
| Snapshots de LVM | VSS, `diskshadow`, puntos de control de Hyper-V |
| `dmsetup` | `Get-VirtualDisk`, `Get-Volume` |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 5 ("Administering Advanced Storage Devices") del libro LPIC-2. Fuentes: documentación oficial de Microsoft Learn (Espacios de almacenamiento y Espacios de almacenamiento directos, módulo Storage de PowerShell, discos dinámicos, Servidor de destino iSCSI e iniciador iSCSI, MPIO, DiskSpd) y anuncio oficial de Microsoft sobre la compatibilidad con NVMe nativo en Windows Server 2025.*
