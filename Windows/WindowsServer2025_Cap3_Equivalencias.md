# LPIC-2 · Capítulo 3: Mastering the Kernel
### Equivalencias en Windows Server 2025

Este documento recoge, para cada concepto visto en el Capítulo 3 del libro LPIC-2 (componentes del kernel, compilación, parámetros en caliente, módulos y detección de hardware), cuál es su equivalente en **Windows Server 2025**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo: el kernel de Windows es **software cerrado**. No se puede descargar su código, ni configurarlo, ni compilarlo. Por eso buena parte del capítulo (apartado 2) no tiene equivalente directo, y lo que en Linux se hace compilando, en Windows se hace de otras formas:
- El kernel se actualiza con **Windows Update** (actualizaciones acumulativas mensuales).
- Los "módulos" del kernel son los **controladores** (drivers, archivos `.sys`), que se instalan con paquetes `.inf` y la herramienta `pnputil`.
- No existe `/proc` ni `sysctl`: los parámetros del kernel están en el **Registro** y se consultan con **PowerShell / CIM** o con herramientas concretas (`bcdedit`, `netsh`, `fsutil`...).
- El papel de `udev` lo hace el **Administrador de Plug and Play** (PnP), integrado en el propio kernel.

Todos los comandos se ejecutan en una consola **como Administrador**.

---

## 1. Partes del sistema Windows y del kernel

Windows divide el sistema en dos "modos" de ejecución:

| Modo | Qué contiene | Equivalente Linux aproximado |
|---|---|---|
| **Modo kernel** | Kernel y ejecutivo (`ntoskrnl.exe`), capa de abstracción de hardware (`hal.dll`), controladores (`.sys`), parte del sistema gráfico (`win32k*.sys`) | Kernel + módulos |
| **Modo usuario** | Servicios, subsistemas, librerías del sistema (`ntdll.dll`, `kernel32.dll`...) y aplicaciones | Utilidades GNU, demonios y aplicaciones |

En servidores con **seguridad basada en virtualización** (VBS, activa en muchos equipos nuevos), por debajo de todo funciona además el hipervisor de Windows, y una parte protegida del kernel (`securekernel.exe`) vigila la integridad del kernel normal.

### 1.1 Ficheros binarios del kernel (equivalente a la Tabla 3.2)

| Linux | Windows Server 2025 | Descripción |
|---|---|---|
| `vmlinuz` / `bzImage` | `C:\Windows\System32\ntoskrnl.exe` | El kernel y el ejecutivo de Windows (gestión de memoria, procesos, E/S, seguridad...) |
| — | `C:\Windows\System32\hal.dll` | Capa de abstracción de hardware: separa el kernel de los detalles de la placa base |
| — | `C:\Windows\System32\win32k.sys`, `win32kbase.sys`, `win32kfull.sys` | Parte del sistema de ventanas y gráficos que se ejecuta en modo kernel |
| — | `C:\Windows\System32\securekernel.exe` | Kernel seguro, solo activo con VBS |
| `System.map` | Archivos de **símbolos** `.pdb` | No vienen instalados: se descargan del servidor de símbolos de Microsoft cuando se depura (apartado 6) |

Históricamente existían varios kernels para elegir según el hardware (`ntoskrnl.exe` para un procesador, `ntkrnlmp.exe` para varios, `ntkrnlpa.exe` para memoria PAE), algo parecido a los distintos binarios de la Tabla 3.2. Hoy en x64 solo existe `ntoskrnl.exe`.

Diferencia importante: en Linux se pueden tener **varias versiones del kernel** en `/boot`. En Windows solo hay **una versión activa**; las versiones anteriores de los archivos del sistema se guardan en el **almacén de componentes** (`C:\Windows\WinSxS`) para poder desinstalar actualizaciones.

### 1.2 Otras partes del kernel

| Elemento Linux | Equivalente en Windows Server 2025 | Ubicación típica |
|---|---|---|
| Módulos del kernel (`.ko`) | **Controladores** (drivers, archivos `.sys`) | `C:\Windows\System32\drivers\` |
| `/lib/modules/versión/` | **Almacén de controladores** (Driver Store): copia de cada paquete de controlador instalado, con su `.inf` (instrucciones de instalación) y su `.cat` (firma digital) | `C:\Windows\System32\DriverStore\FileRepository\` |
| Configuración de módulos | Una clave por controlador en el Registro (tipo, tipo de inicio, grupo...) | `HKLM\SYSTEM\CurrentControlSet\Services\nombre` |
| Código fuente del kernel | **No disponible** (software propietario). Microsoft publica ejemplos de controladores en GitHub (repositorio *Windows-driver-samples*) | — |
| Parches del kernel | **Actualizaciones acumulativas** de Windows Update (apartado 2.7) | — |
| Cabeceras del kernel (headers) | **Windows Driver Kit (WDK)** + Windows SDK: cabeceras (`wdm.h`, `ntddk.h`...), librerías y herramientas para compilar controladores | Se instalan junto a Visual Studio |
| Documentación del kernel | Documentación de controladores de Windows en Microsoft Learn y libros como *Windows Internals* | — |

### 1.3 Módulos del kernel (controladores)

Igual que los módulos de Linux, los controladores evitan meter todo el soporte de hardware dentro del kernel. Tipos principales:

| Tipo | Descripción |
|---|---|
| Controlador en modo kernel (KMDF / WDM) | Se ejecuta dentro del kernel, como un módulo `.ko` |
| Controlador en modo usuario (UMDF) | Se ejecuta fuera del kernel; si falla, no tumba el sistema. Linux no tiene un equivalente tan extendido |
| Controlador de filtro / minifiltro | Se "engancha" encima de otro para vigilar o modificar su trabajo (antivirus, copias de seguridad, cifrado). Se gestionan con `fltmc` |
| Controlador de arranque (boot-start) | Se carga antes que el propio kernel termine de iniciarse (discos, sistema de archivos). Ver apartado 2.6 |

Diferencias clave con Linux:
- En Windows **casi todos los controladores se distribuyen solo en binario** (lo que el libro menciona como caso excepcional de algunos fabricantes, aquí es lo normal).
- En Windows x64 los controladores de modo kernel **deben estar firmados por Microsoft** para cargarse (con Secure Boot activado). Solo se pueden cargar controladores de prueba sin firmar con `bcdedit /set testsigning on` (Capítulo 1), y no es recomendable en producción.
- Con **Integridad de memoria** (HVCI) activada, además se bloquean los controladores incompatibles y los que Microsoft tiene en su lista de controladores vulnerables.

### 1.4 Versionado de Windows

Windows usa la numeración **NT** (el nombre de la familia de kernels de Windows desde 1993). La versión del kernel va unida a la del sistema completo.

| Formato | Ejemplo | Significado |
|---|---|---|
| `Mayor.Menor.Compilación.Revisión` | `10.0.26100.xxxx` | 10.0 = versión NT, 26100 = compilación (build) de Windows Server 2025, `xxxx` = revisión (UBR), que sube con cada actualización mensual |
| Versión comercial | 24H2 | Versión de la que deriva Windows Server 2025 (la misma base que Windows 11 24H2) |

**Historia resumida** (equivalente a la Tabla de versiones del kernel):

| Versión NT | Windows Server | Notas |
|---|---|---|
| 3.1 a 4.0 | Windows NT Server (1993-1996) | Primeras versiones |
| 5.0 | Windows 2000 Server | Aparece Active Directory |
| 5.2 | Windows Server 2003 | — |
| 6.0 / 6.1 | Windows Server 2008 / 2008 R2 | Aparecen la BCD, Server Core e Hyper-V |
| 6.2 / 6.3 | Windows Server 2012 / 2012 R2 | — |
| 10.0 (build 14393) | Windows Server 2016 | Desde aquí la versión NT ya no cambia: solo cambia el número de compilación |
| 10.0 (build 17763) | Windows Server 2019 | — |
| 10.0 (build 20348) | Windows Server 2022 | — |
| 10.0 (build 26100) | **Windows Server 2025** | — |

**Canales de publicación** (parecido a `stable` / `longterm` / `mainline` de kernel.org):

| Canal | Descripción | Parecido en Linux |
|---|---|---|
| LTSC (canal de mantenimiento a largo plazo) | Versión "con año" (2025). 5 años de soporte estándar + 5 de soporte extendido | `longterm` |
| Annual Channel | Versión publicada cada año, con menos soporte, pensada sobre todo para contenedores y Azure Stack HCI | `stable` |
| Windows Insider para Windows Server | Versiones de prueba de la próxima edición | `mainline` / `-rc` |

**Comandos para ver la versión** (equivalente a `uname -r`):

| Comando | Qué muestra |
|---|---|
| `ver` (en `cmd`) | Versión completa, ej. `10.0.26100.xxxx` |
| `winver` | Ventana con la versión (solo con escritorio) |
| `[Environment]::OSVersion.Version` | Versión del sistema en PowerShell |
| `Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' \| Select ProductName, DisplayVersion, CurrentBuild, UBR` | Nombre, versión (24H2), compilación y revisión |
| `(Get-Item C:\Windows\System32\ntoskrnl.exe).VersionInfo.ProductVersion` | Versión del archivo del kernel |
| `Get-ComputerInfo -Property OsName, OsVersion, OsBuildNumber` | Resumen del sistema |

---

## 2. Compilación de un nuevo kernel

**En Windows no se compila el kernel.** La tabla siguiente muestra qué se hace en su lugar en cada paso del libro:

| Paso en Linux | Equivalente en Windows Server 2025 |
|---|---|
| Descargar el código fuente | Descargar **actualizaciones** (Windows Update o el Catálogo de Microsoft Update) |
| Configurar el kernel (`.config`) | Elegir **roles y características** del sistema y ajustar parámetros en el **Registro** y en la **BCD** |
| Compilar (`make bzImage`) | No aplica |
| Instalar el kernel y actualizar GRUB | Instalar la actualización y **reiniciar**: Windows actualiza la BCD solo |
| Compilar e instalar módulos | Instalar **controladores** con `pnputil` |
| Crear el initrd | No hace falta (apartado 2.6) |
| Aplicar parches | Instalar actualizaciones acumulativas |
| Imagen de sistema personalizada | **Mantenimiento de imágenes con DISM** (añadir actualizaciones, controladores y características a una imagen `.wim` antes de instalarla) |

### 2.1 Obtener el código fuente → obtener actualizaciones

| Fuente en Linux | Equivalente en Windows Server 2025 |
|---|---|
| kernel.org | **Catálogo de Microsoft Update** (`catalog.update.microsoft.com`): descarga manual de actualizaciones en archivos `.msu` |
| Repositorio de la distribución | **Windows Update** (directo), o en empresas **WSUS**, Azure Update Manager o Configuration Manager |

Instalar una actualización descargada a mano:

| Comando | Función |
|---|---|
| `wusa C:\Updates\archivo.msu /quiet /norestart` | Instala un paquete `.msu` sin preguntas y sin reiniciar |
| `DISM /Online /Add-Package /PackagePath:C:\Updates\archivo.msu` | Lo mismo con DISM |
| `DISM /Online /Add-Package /PackagePath:C:\Updates\` | Instala todos los paquetes de una carpeta |

Detalle de Windows Server 2025: usa **actualizaciones acumulativas con punto de control** (checkpoint). Algunas actualizaciones necesitan que antes esté instalada una actualización "punto de control" anterior; si se instalan a mano, conviene poner todos los `.msu` necesarios en la misma carpeta e instalarlos con el último comando de la tabla, para que DISM respete el orden.

### 2.2 Crear el fichero de configuración → elegir componentes y parámetros

No existe un `.config` ni `make menuconfig`. Lo más parecido a "decidir qué lleva el sistema":

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| Elegir qué partes incluir en el kernel | Elegir **roles y características**: `Get-WindowsFeature`, `Install-WindowsFeature`, `Uninstall-WindowsFeature` (Capítulo 2) |
| Elegir un sistema mínimo | Instalar **Server Core** en vez de Experiencia de escritorio (Capítulo 1) |
| Parámetros del kernel al arrancar | `bcdedit /set {current} opción valor` (Capítulo 1) |
| Parámetros del kernel en ejecución | Registro y herramientas específicas (apartado 3) |
| `make defconfig` (valores por defecto) | La configuración por defecto de una instalación nueva |

### 2.3 Compilar el kernel

No aplica. El kernel llega ya compilado dentro de las actualizaciones.

### 2.4 Instalar el kernel compilado

Al instalar una actualización acumulativa, Windows sustituye `ntoskrnl.exe` y el resto de archivos al **reiniciar**. No hay que tocar el gestor de arranque.

| Comando / ubicación | Función |
|---|---|
| `Get-WUIsPendingReboot` | Devuelve `True` si hay una actualización esperando un reinicio |
| `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending` | Si esta clave existe, hay un reinicio pendiente por actualizaciones |
| `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired` | Igual, marcado por Windows Update |

**Hotpatch** (novedad de Windows Server 2025): permite instalar algunas actualizaciones de seguridad **en memoria, sin reiniciar**. En Windows Server 2025 Standard y Datacenter requiere conectar el servidor a **Azure Arc** con una suscripción de pago (en las ediciones Azure Edition viene incluido). Cada trimestre hay una actualización "base" que sí necesita reiniciar. Es parecido a `kpatch` / `livepatch` en Linux.

### 2.5 Compilar e instalar los módulos → controladores

**Compilar un controlador** (solo para desarrolladores): Visual Studio + **Windows Driver Kit (WDK)**; se compila con `msbuild` como cualquier proyecto de Visual Studio. El resultado es un `.sys`, un `.inf` y un `.cat`, y hay que firmarlo para que Windows lo cargue.

**Instalar un controlador** (equivalente a `make modules_install`):

| Comando | Función |
|---|---|
| `pnputil /add-driver C:\Drivers\red\driver.inf /install` | Añade el paquete al almacén de controladores y lo instala en los dispositivos compatibles |
| `pnputil /add-driver C:\Drivers\*.inf /subdirs /install` | Lo mismo con todos los `.inf` de una carpeta y sus subcarpetas |
| `pnputil /export-driver * C:\CopiaDrivers` | Copia de todos los controladores de terceros instalados (útil antes de reinstalar) |
| `Export-WindowsDriver -Online -Destination C:\CopiaDrivers` | Lo mismo en PowerShell |

### 2.6 Disco RAM inicial (initrd / initramfs)

**Windows no usa initrd ni initramfs.** En Linux sirven para cargar los módulos necesarios para leer el disco antes de montar la raíz. En Windows ese trabajo lo hace el **cargador de arranque** (`winload.efi`, Capítulo 1): lee el disco con ayuda del firmware y carga en memoria el kernel **y los controladores marcados como de arranque** (tipo de inicio `0 = Boot` en el Registro, Capítulo 1) antes de ceder el control al kernel.

| Necesidad en Linux | Solución en Windows Server 2025 |
|---|---|
| Incluir en el initrd el módulo de la controladora de discos | Que el controlador de la controladora tenga inicio de tipo `Boot` (valor `Start = 0`). Los controladores de almacenamiento se instalan así automáticamente |
| Regenerar el initrd tras cambiar de controladora | Si se cambia la controladora de discos (por ejemplo, al migrar a otro hardware o a una máquina virtual), hay que instalar antes su controlador. Si no, Windows no arranca y muestra el error **`INACCESSIBLE_BOOT_DEVICE`** (equivalente a un kernel que no encuentra la raíz por falta de módulos en el initrd) |
| Añadir módulos a un initrd sin arrancar | Inyectar controladores en una imagen o en un Windows sin arrancar: `DISM /Image:C:\ /Add-Driver /Driver:D:\Drivers\disco.inf` (desde WinRE o WinPE) |
| Módulos durante la instalación | Botón **"Cargar controlador"** del instalador de Windows |

Lo más parecido a un initramfs es **WinPE** (`boot.wim`): un Windows mínimo que se carga entero en memoria y se usa para instalar o reparar el sistema (Capítulo 1).

**Mantenimiento de imágenes con DISM** (preparar una imagen de instalación ya actualizada y con controladores, sin arrancarla):

| Comando | Función |
|---|---|
| `DISM /Mount-Image /ImageFile:C:\ISO\sources\install.wim /Index:1 /MountDir:C:\Montaje` | Monta la imagen en una carpeta |
| `DISM /Image:C:\Montaje /Add-Driver /Driver:C:\Drivers /Recurse` | Añade controladores |
| `DISM /Image:C:\Montaje /Add-Package /PackagePath:C:\Updates\archivo.msu` | Añade actualizaciones |
| `DISM /Unmount-Image /MountDir:C:\Montaje /Commit` | Guarda los cambios y desmonta (`/Discard` los descarta) |

### 2.7 Parches del kernel → actualizaciones

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| Parche incremental (4.3.0 → 4.3.1) | **Actualización acumulativa mensual** (LCU), publicada el segundo martes de cada mes ("Patch Tuesday"). Es acumulativa: incluye todas las correcciones anteriores |
| Aplicar varios parches en orden | No hace falta: basta con la última acumulativa (y, si lo pide, su punto de control) |
| `patch` | `wusa` / `DISM /Add-Package` / Windows Update |
| `patch -R` (deshacer) | `DISM /Online /Remove-Package /PackageName:...` (Capítulo 1, apartado 7.2) |
| Ver parches aplicados | `Get-HotFix` o `DISM /Online /Get-Packages` |

El almacén de componentes (`C:\Windows\WinSxS`) crece con cada actualización porque guarda las versiones anteriores:

| Comando | Función |
|---|---|
| `DISM /Online /Cleanup-Image /AnalyzeComponentStore` | Indica cuánto ocupa y si conviene limpiarlo |
| `DISM /Online /Cleanup-Image /StartComponentCleanup` | Borra versiones antiguas que ya no se necesitan |
| `DISM /Online /Cleanup-Image /StartComponentCleanup /ResetBase` | Borra todas las versiones anteriores. **Después ya no se pueden desinstalar** las actualizaciones instaladas |

---

## 3. Consulta y modificación de parámetros del kernel en caliente

### 3.1 El sistema de ficheros `/proc`

**Windows no tiene `/proc`** ni ningún sistema de archivos virtual parecido. La información que en Linux se lee de `/proc` se obtiene de tres fuentes:
- **WMI / CIM**: base de datos de información del sistema, que se consulta con `Get-CimInstance`.
- **Contadores de rendimiento** (Capítulo 2).
- La rama **`HKLM\HARDWARE`** del Registro, que Windows crea en memoria en cada arranque con el hardware detectado (es lo más parecido a un archivo "virtual" de `/proc`).

| Linux (`/proc`) | Equivalente en Windows Server 2025 |
|---|---|
| `/proc/sys/kernel/` | Registro: `HKLM\SYSTEM\CurrentControlSet\Control\` (apartado 3.2) |
| `cat /proc/sys/kernel/version` | `(Get-CimInstance Win32_OperatingSystem).Version` o `ver` |
| `/proc/cpuinfo` | `Get-CimInstance Win32_Processor \| Format-List *` o `HKLM\HARDWARE\DESCRIPTION\System\CentralProcessor\0` |
| `/proc/meminfo` | `Get-CimInstance Win32_OperatingSystem` (memoria libre) y `Get-CimInstance Win32_PhysicalMemory` (módulos de RAM) |
| `/proc/dma` | `Get-CimInstance Win32_DMAChannel` |
| `/proc/ioports` | `Get-CimInstance Win32_PortResource` |
| `/proc/interrupts` | `Get-CimInstance Win32_IRQResource` |
| `/proc/iomem` | `Get-CimInstance Win32_DeviceMemoryAddress` |
| `/proc/PID/` (procesos) | `Get-Process`, `Get-CimInstance Win32_Process` |
| Todo lo anterior en gráfico | `msinfo32` → **Recursos de hardware** (DMA, E/S, IRQ, memoria, conflictos) |

### 3.2 El equivalente de `sysctl`: el Registro y herramientas específicas

En Windows no hay un comando único como `sysctl`. Los parámetros del kernel y de sus componentes se guardan en el **Registro**, y además algunas áreas tienen su propia herramienta.

**Diferencias importantes con `sysctl`:**
- El Registro es **persistente**: un cambio se conserva al reiniciar (no hace falta un `/etc/sysctl.conf`).
- Muchos parámetros del Registro solo se leen al arrancar, así que **necesitan reiniciar** para aplicarse. Las herramientas específicas (`netsh`, `Set-NetIPInterface`, `fsutil`...) sí aplican muchos cambios en caliente.
- Igual que en Linux, muchos valores son de tipo "activado / desactivado" con `0` y `1` (tipo `REG_DWORD`).

**Comandos para leer y cambiar el Registro:**

| Comando | Función | Equivalente `sysctl` |
|---|---|---|
| `reg query "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management"` | Muestra los valores de una clave | `sysctl vm` |
| `reg query "clave" /v Nombre` | Muestra un valor concreto | `sysctl nombre` |
| `reg add "clave" /v Nombre /t REG_DWORD /d 1 /f` | Crea o cambia un valor | `sysctl -w nombre=1` |
| `reg delete "clave" /v Nombre /f` | Borra un valor (vuelve al comportamiento por defecto) | — |
| `Get-ItemProperty -Path 'HKLM:\SYSTEM\...' -Name Nombre` | Leer (PowerShell) | `sysctl nombre` |
| `Set-ItemProperty -Path 'HKLM:\SYSTEM\...' -Name Nombre -Value 1` | Cambiar (PowerShell) | `sysctl -w` |
| `reg export "clave" copia.reg` / `reg import copia.reg` | Guardar / aplicar un conjunto de valores desde un archivo | `/etc/sysctl.conf` y `sysctl -p` |
| `regedit` | Editor gráfico del Registro | — |

Para aplicar los mismos parámetros en muchos servidores (equivalente a repartir archivos en `/etc/sysctl.d/`), se usan archivos `.reg` o, mejor, **Directivas de grupo** (Preferencias → Registro).

**Claves del Registro más relacionadas con el kernel:**

| Clave (dentro de `HKLM\SYSTEM\CurrentControlSet\`) | Qué controla | Parecido en Linux |
|---|---|---|
| `Control\Session Manager\Memory Management` | Memoria: archivo de paginación (`PagingFiles`), borrarlo al apagar (`ClearPageFileAtShutdown`)... | `vm.*` |
| `Control\Session Manager\kernel` | Opciones generales del kernel | `kernel.*` |
| `Control\PriorityControl` | Reparto de CPU entre procesos en primer y segundo plano (`Win32PrioritySeparation`) | `kernel.sched_*` |
| `Control\FileSystem` | Opciones de NTFS: rutas largas (`LongPathsEnabled`), nombres cortos 8.3... | `fs.*` |
| `Control\CrashControl` | Qué hacer si el kernel falla (apartado 6) | `kernel.panic` |
| `Services\Tcpip\Parameters` | Pila TCP/IP (ej. `IPEnableRouter`) | `net.ipv4.*` |
| `Services\nombre` | Configuración de cada controlador y servicio | Opciones de módulos (`/etc/modprobe.d/`) |

**Herramientas específicas que cambian parámetros en caliente:**

| Ejemplo de `sysctl` en Linux | Equivalente en Windows Server 2025 |
|---|---|
| `sysctl -w net.ipv4.ip_forward=1` | `Set-NetIPInterface -Forwarding Enabled` (en caliente, todas las interfaces) o `IPEnableRouter = 1` en el Registro (requiere reiniciar) |
| `sysctl net.ipv4.ip_local_port_range` | `netsh int ipv4 show dynamicport tcp` / `netsh int ipv4 set dynamicport tcp start=10000 num=50000` |
| Ajustes TCP (`net.ipv4.tcp_*`) | `Get-NetTCPSetting` / `Set-NetTCPSetting`, o `netsh int tcp show global` / `netsh int tcp set global autotuninglevel=normal` |
| Parámetros de IPv4 en general | `Get-NetIPv4Protocol` / `Set-NetIPv4Protocol` |
| Opciones de sistema de archivos (`fs.*`, `noatime`) | `fsutil behavior query disablelastaccess` / `fsutil behavior set disablelastaccess 1` |
| Swap (`vm.swappiness`, tamaño) | Tamaño del archivo de paginación: `sysdm.cpl` → Opciones avanzadas → Rendimiento → Memoria virtual (o la clase CIM `Win32_PageFileSetting`) |
| Energía y frecuencia de CPU | `powercfg /list`, `powercfg /setactive SCHEME_MIN` (plan "Alto rendimiento") |
| Parámetros de arranque del kernel | `bcdedit` (Capítulo 1) |

---

## 4. Gestión de módulos (controladores) en tiempo de ejecución

| Comando Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `lsmod` | `driverquery` | Lista los controladores instalados, su tipo y fecha |
| `lsmod` (solo los cargados) | `Get-CimInstance Win32_SystemDriver \| Where-Object State -eq 'Running' \| Select Name, StartMode, PathName` | Controladores en ejecución |
| — | `sc.exe query type= driver` | Estado de los controladores con `sc.exe` |
| `modinfo nombre` | `driverquery /v`, `pnputil /enum-drivers` o `Get-WindowsDriver -Online` | Información detallada: proveedor, versión, clase, firma |
| — | `(Get-Item C:\Windows\System32\drivers\nombre.sys).VersionInfo` | Versión de un archivo `.sys` concreto |
| `insmod ruta.ko` | `sc.exe create nombre type= kernel start= demand binPath= C:\ruta\nombre.sys` + `sc.exe start nombre` | Cargar un controlador "clásico" (no Plug and Play) indicando su archivo |
| `modprobe nombre` | `pnputil /add-driver archivo.inf /install` | Instalar un controlador Plug and Play; el `.inf` indica qué archivos y dependencias necesita |
| `rmmod` / `modprobe -r` | `sc.exe stop nombre` | Descargar un controlador clásico (solo si el controlador lo permite; muchos no) |
| — | `pnputil /delete-driver oem12.inf /uninstall` | Quitar un paquete de controlador del sistema (el nombre `oemNN.inf` se ve con `pnputil /enum-drivers`) |
| — | `fltmc` / `fltmc load nombre` / `fltmc unload nombre` | Listar, cargar y descargar minifiltros (antivirus, copias...) |

**Opciones de `modprobe` (Tabla 3.5) y su equivalente:**

| Opción `modprobe` | Equivalente en Windows Server 2025 |
|---|---|
| `-a` (varios módulos) | `pnputil /add-driver C:\Drivers\*.inf /subdirs /install` |
| `-b` (lista negra) | Deshabilitar el dispositivo (`Disable-PnpDevice -InstanceId "..."` o `pnputil /disable-device "..."`), poner el controlador como deshabilitado (`sc.exe config nombre start= disabled`), o bloquear su instalación con la directiva "Restricciones de instalación de dispositivos" |
| `-c` (ver configuración) | `pnputil /enum-drivers` y la clave del Registro del controlador |
| `-d` (otro directorio raíz) | Trabajar sobre un Windows sin arrancar: `DISM /Image:D:\ /Add-Driver` o `/Get-Drivers` |
| `-f` (forzar) | `pnputil /delete-driver oem12.inf /uninstall /force`. Para instalar controladores sin firma, solo en modo de pruebas (`testsigning`) |
| `-n` (simulación) | No existe |
| `-r` (quitar) | `pnputil /delete-driver ... /uninstall` o `sc.exe stop` |
| `-s` (errores al syslog) | Windows registra siempre la instalación de controladores en `C:\Windows\INF\setupapi.dev.log` y en el Visor de eventos |
| `-v` (detalle) | El registro `setupapi.dev.log` contiene el detalle completo de cada instalación |

**Comprobar firmas de controladores:**

| Comando | Función |
|---|---|
| `Get-CimInstance Win32_PnPSignedDriver \| Select DeviceName, DriverVersion, IsSigned, Signer` | Lista los controladores de dispositivos con su firma |
| `driverquery /si` | Indica si cada controlador está firmado |
| `sigverif` | Herramienta gráfica de comprobación de firmas |
| Registro `Microsoft-Windows-CodeIntegrity/Operational` (Visor de eventos) | Controladores que Windows ha bloqueado por firma o por incompatibilidad con la integridad de memoria |

---

## 5. Detección de hardware

### 5.1 Coldplug vs Hotplug

Los conceptos son los mismos. En Windows:
- Los dispositivos **hotplug** (USB, discos externos) se detectan solos; antes de desconectarlos se usa **"Quitar hardware de forma segura"**.
- Para que Windows vuelva a buscar cambios de hardware: `pnputil /scan-devices`, o en el Administrador de dispositivos → Acción → **Buscar cambios de hardware**.
- Para discos nuevos en concreto: `Update-HostStorageCache` o `diskpart` → `rescan`.

### 5.2 El equivalente a `udev`: el Administrador de Plug and Play

| Elemento Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| Kernel que envía eventos de hardware | **Administrador de Plug and Play** (parte del kernel) | Detecta el dispositivo, lee su identificador y busca un controlador |
| `udevd` | El mismo Administrador PnP + el servicio **Plug and Play** (`PlugPlay`) en modo usuario | Coordina la instalación y avisa a los programas |
| `/etc/udev/rules.d/`, `/lib/udev/rules.d/` | **Archivos `.inf`** (`C:\Windows\INF\` y el almacén de controladores) | Cada `.inf` indica para qué identificadores de hardware sirve un controlador. Windows elige el que mejor encaja |
| `/etc/udev/udev.conf` | No hay archivo equivalente; algunas opciones se controlan por directiva de grupo (instalación de dispositivos) | — |
| Reglas para nombres fijos de dispositivos | Letras de unidad (guardadas en `HKLM\SYSTEM\MountedDevices`, se cambian con `Set-Partition -DriveLetter E -NewDriveLetter F`) y nombres de red (`Rename-NetAdapter -Name "Ethernet" -NewName "LAN"`) | Nombres estables para discos y tarjetas |
| `/dev/` | No hay archivos de dispositivo visibles. Cada dispositivo tiene una **ruta de instancia**, ej. `PCI\VEN_8086&DEV_15F3&...` | Identificador único de cada dispositivo |
| `udevadm monitor` | Seguir en directo el registro de instalación: `Get-Content C:\Windows\INF\setupapi.dev.log -Wait -Tail 20`, o el registro de eventos `Microsoft-Windows-Kernel-PnP/Configuration` | Ver eventos de hardware en tiempo real |
| `udevadm info` | `Get-PnpDeviceProperty -InstanceId "..."` | Todas las propiedades de un dispositivo |

Eventos útiles del registro `Microsoft-Windows-Kernel-PnP/Configuration`: **400** (dispositivo configurado) y **410** (dispositivo iniciado).

Los **identificadores de hardware** son los mismos números que muestra `lspci -nn` o `lsusb` en Linux, con otro formato:
- PCI: `PCI\VEN_8086&DEV_15F3` (fabricante 8086 = Intel, dispositivo 15F3).
- USB: `USB\VID_046D&PID_C52B` (fabricante 046D = Logitech, producto C52B).

### 5.3 Comandos para consultar hardware

| Comando Linux | Equivalente en Windows Server 2025 |
|---|---|
| `lspci` | `Get-PnpDevice -PresentOnly \| Where-Object InstanceId -like 'PCI\*'` |
| `lsusb` | `Get-PnpDevice -PresentOnly \| Where-Object InstanceId -like 'USB*'` |
| Todos los dispositivos | `Get-PnpDevice -PresentOnly` o `pnputil /enum-devices /connected` |
| Por tipo (red, discos, vídeo...) | `Get-PnpDevice -Class Net` o `pnputil /enum-devices /class Display` |
| Dispositivos con problemas (sin controlador, con error) | `Get-PnpDevice -Status ERROR` o `pnputil /enum-devices /problem` |
| Herramienta gráfica | **Administrador de dispositivos** (`devmgmt.msc`) y `msinfo32` |
| `lshw` / `dmidecode` | `Get-CimInstance Win32_BIOS`, `Win32_BaseBoard`, `Win32_PhysicalMemory`, `Win32_Processor`; `Get-PhysicalDisk`; `Get-NetAdapter`; `Get-ComputerInfo` |

**Opciones de `lsusb` (Tabla 3.7) y su equivalente:**

| Opción `lsusb` | Equivalente en Windows Server 2025 |
|---|---|
| `-d fabricante` | Filtrar por identificador: `Get-PnpDevice -PresentOnly \| Where-Object InstanceId -like '*VID_046D*'` (USB) o `'*VEN_8086*'` (PCI) |
| `-D dispositivo` | `Get-PnpDevice -InstanceId "ruta de instancia"` |
| `-s bus` | Propiedad de ubicación: `Get-PnpDeviceProperty -InstanceId "..." -KeyName DEVPKEY_Device_LocationInfo` |
| `-t` (árbol) | `pnputil /enum-devices /relations` (muestra padre e hijos), o Administrador de dispositivos → Ver → **Dispositivos por conexión** |
| `-v` (detalle) | `Get-PnpDeviceProperty -InstanceId "..."` o `pnputil /enum-devices /drivers` |
| `-V` (versión del programa) | No aplica |

---

## 6. Resolución de problemas del kernel

| Linux | Windows Server 2025 | Función |
|---|---|---|
| `dmesg` | Visor de eventos, registro **Sistema** / `Get-WinEvent -LogName System` (Capítulo 1) | Mensajes del kernel y de los controladores |
| `/var/log/messages`, `/var/log/syslog` | `C:\Windows\System32\winevt\Logs\System.evtx` | Registro general del sistema |
| `/var/log/boot` (Debian), `/var/log/boot.log` (Red Hat) | `C:\Windows\ntbtlog.txt` (si se activa el registro de arranque, Capítulo 1) | Registro del arranque |
| Log de carga de módulos | `C:\Windows\INF\setupapi.dev.log` | Instalación de controladores |

**Cuando el kernel falla** (equivalente a un *kernel panic*): Windows muestra una **pantalla de error** (conocida como "pantalla azul" o *bugcheck*) con un **código de parada** que indica el tipo de fallo, guarda un volcado de memoria y se reinicia.

| Código de parada (ejemplos) | Significado habitual |
|---|---|
| `INACCESSIBLE_BOOT_DEVICE` (0x7B) | No puede leer el disco de arranque (falta el controlador de la controladora; ver 2.6) |
| `DRIVER_IRQL_NOT_LESS_OR_EQUAL` (0xD1) | Un controlador accedió a memoria de forma incorrecta |
| `PAGE_FAULT_IN_NONPAGED_AREA` (0x50) | Acceso a memoria no válida (controlador defectuoso o RAM dañada) |
| `SYSTEM_SERVICE_EXCEPTION` (0x3B) | Error dentro de una llamada del sistema, normalmente por un controlador |

| Elemento | Función | Parecido en Linux |
|---|---|---|
| `C:\Windows\MEMORY.DMP` | Volcado de memoria del kernel tras un fallo | `kdump` / `/var/crash` |
| `C:\Windows\Minidump\*.dmp` | Volcados pequeños, uno por fallo | — |
| `sysdm.cpl` → Opciones avanzadas → **Inicio y recuperación** | Elegir el tipo de volcado y si reiniciar solo | `kernel.panic` |
| `HKLM\SYSTEM\CurrentControlSet\Control\CrashControl`, valor `CrashDumpEnabled` | Tipo de volcado: `0` ninguno, `1` completo, `2` del kernel, `3` pequeño, `7` automático (por defecto) | — |
| Evento **1001** (origen *BugCheck* / *WER-SystemErrorReporting*) en el registro Sistema | Tras reiniciar, indica el código de parada y dónde se guardó el volcado | — |
| Evento **41** (Kernel-Power) | El equipo se reinició sin apagarse bien (Capítulo 1) | — |

**Herramientas de diagnóstico del kernel:**

| Herramienta | Función |
|---|---|
| **WinDbg** (`winget install Microsoft.WinDbg`) | Depurador de Microsoft. Abre el `MEMORY.DMP` o un minidump; el comando `!analyze -v` indica normalmente qué controlador causó el fallo, `lm` lista los módulos cargados y `k` muestra la pila. Descarga los símbolos (`.pdb`) del servidor de Microsoft automáticamente |
| **Comprobador de controladores** (`verifier`) | Somete a los controladores a pruebas extra para detectar al culpable de fallos aleatorios: `verifier /standard /driver nombre.sys`, ver la configuración con `verifier /querysettings`, desactivar con `verifier /reset` (si el sistema no arranca, se hace desde Modo seguro) |
| Depuración del kernel en vivo | `bcdedit /debug on` + `bcdedit /dbgsettings net hostip:IP port:50000` y conectar WinDbg desde otro equipo (Capítulo 1) |
| `msinfo32` → Resumen del sistema | Estado de la seguridad basada en virtualización y de la integridad de memoria |
| Diagnóstico de memoria de Windows (`mdsched`) | Comprueba la RAM en el siguiente reinicio (útil cuando los fallos son aleatorios) |

---

## 7. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 3) | Equivalente en Windows Server 2025 |
|---|---|
| `/boot/vmlinuz-versión` | `C:\Windows\System32\ntoskrnl.exe` (+ `hal.dll`); una sola versión activa |
| `/lib/modules/versión/` | `C:\Windows\System32\drivers\` y el almacén de controladores (`DriverStore`) |
| Módulo `.ko` | Controlador `.sys` + `.inf` + `.cat` (firmado) |
| `/usr/src/linux` | No disponible; WDK para desarrollar controladores |
| Cabeceras del kernel | Windows Driver Kit (WDK) + Windows SDK |
| `uname -r` | `ver`, `[Environment]::OSVersion.Version` (10.0.26100.x) |
| kernel.org: stable / longterm / mainline | Annual Channel / LTSC / Windows Insider |
| `.config`, `make menuconfig` | Roles y características, Registro, BCD |
| `make bzImage`, `make install` | No aplica: actualizaciones de Windows Update + reinicio |
| Actualizar GRUB tras instalar | No hace falta |
| `make modules_install` | `pnputil /add-driver archivo.inf /install` |
| `mkinitrd` / `mkinitramfs` | No hace falta: `winload` carga los controladores de arranque; `DISM /Add-Driver` para inyectarlos sin arrancar |
| `patch` / `patch -R` | Actualizaciones acumulativas (`wusa`, `DISM /Add-Package`) / `DISM /Remove-Package` |
| `kpatch` / `livepatch` | Hotpatch (con Azure Arc) |
| `/proc/cpuinfo`, `/proc/dma`, `/proc/ioports` | `Get-CimInstance Win32_Processor`, `Win32_DMAChannel`, `Win32_PortResource`, `msinfo32` |
| `sysctl` / `/etc/sysctl.conf` | Registro (`reg`, `Set-ItemProperty`) + herramientas específicas (`netsh`, `Set-NetIPInterface`, `fsutil`, `powercfg`) |
| `lsmod` | `driverquery`, `Win32_SystemDriver` |
| `modinfo` | `pnputil /enum-drivers`, `Get-WindowsDriver -Online` |
| `insmod` / `modprobe` | `sc.exe create ... type= kernel` / `pnputil /add-driver` |
| `rmmod` / `modprobe -r` | `sc.exe stop` / `pnputil /delete-driver /uninstall` |
| Lista negra de módulos | `Disable-PnpDevice`, `start= disabled`, directivas de instalación de dispositivos |
| `udev` / `udevd` | Administrador de Plug and Play (+ servicio `PlugPlay`) |
| `/etc/udev/rules.d/` | Archivos `.inf` |
| `udevadm monitor` | `setupapi.dev.log` y el registro `Kernel-PnP/Configuration` |
| `lspci` / `lsusb` | `Get-PnpDevice` filtrando por `PCI\` o `USB`, Administrador de dispositivos |
| `dmesg` | Visor de eventos / `Get-WinEvent` |
| Kernel panic + `kdump` | Pantalla de error (bugcheck) + `MEMORY.DMP` / `Minidump`, analizados con WinDbg |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 3 ("Mastering the Kernel") del libro LPIC-2. Fuentes: documentación oficial de Microsoft Learn (arquitectura de Windows y controladores, PnPUtil, DISM, Driver Store, Windows Driver Kit, WMI/CIM, Registro, netsh, fsutil, Hotpatch para Windows Server 2025, actualizaciones acumulativas con punto de control, volcados de memoria, WinDbg y Comprobador de controladores).*
