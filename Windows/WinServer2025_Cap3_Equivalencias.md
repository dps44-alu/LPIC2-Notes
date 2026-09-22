# LPIC-2 · Capítulo 3: Mastering the Kernel
### Equivalencias en Windows Server 2025

Aviso importante para este capítulo: el modelo de Windows es fundamentalmente distinto al de Linux en este punto. El **núcleo (kernel) de Windows NT no se recompila** ni se distribuye como código fuente al público: es un binario cerrado (`ntoskrnl.exe`) que Microsoft actualiza mediante Windows Update. Por tanto, gran parte del Capítulo 3 (obtener código fuente del kernel, `make menuconfig`, compilar un `bzImage`, crear un `initrd`...) **no tiene equivalente real** en Windows Server. Lo que sí existe, y se recoge en este documento, es el equivalente funcional de cada pieza: gestión de controladores (módulos), parámetros del sistema en caliente, detección de hardware y resolución de problemas del núcleo.

---

## 1. Partes del sistema y del "kernel" de Windows

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| Binario del kernel (`bzImage`, `vmlinuz`) | `%WINDIR%\System32\ntoskrnl.exe` (núcleo) y `%WINDIR%\System32\hal.dll` (capa de abstracción de hardware) — binarios cerrados, sin código fuente público |
| Módulos del kernel (`.ko`) | **Controladores de dispositivo** (`.sys`), cargados dinámicamente por el subsistema Plug and Play |
| Código fuente del kernel (`/usr/src/linux`) | No disponible públicamente (Windows es de código cerrado); Microsoft ofrece el **Windows Driver Kit (WDK)** y cabeceras del SDK para *desarrollar* controladores, pero no el código del núcleo en sí |
| Documentación del kernel | Documentación técnica en **Microsoft Learn** (sección Windows Drivers / Kernel-Mode Driver Architecture) |
| Parches del kernel (`patch`) | **Windows Update** / **Cumulative Updates**, aplicados como paquetes `.msu`/`.cab`, nunca como parches de código fuente |
| Versionado del kernel (2.x, 3.x, 4.x...) | Versionado por **build de Windows** (ej. Windows Server 2025 = build 26100.x), consultable con `winver` o `[System.Environment]::OSVersion` en PowerShell |

---

## 2. "Compilación" de un nuevo kernel: no aplica

No existe un proceso equivalente a `make menuconfig` / `make bzImage` / `make modules_install`. Lo más parecido conceptualmente es:

| Concepto Linux | Equivalente aproximado en Windows |
|---|---|
| Elegir opciones del kernel a compilar (soporte de hardware, sistemas de archivos, etc.) | Elegir la **edición de Windows Server 2025** (Standard/Datacenter, con o sin Desktop Experience) e instalar **roles y características** (`Install-WindowsFeature` / Server Manager) — el kernel es el mismo, lo que cambia es qué componentes están activos |
| Compilar el kernel con opciones concretas | No aplica: se instala una imagen ya compilada por Microsoft (`.wim`) |
| Personalizar una imagen antes de desplegarla (lo más cercano a "reconfigurar antes de instalar") | **DISM** (Deployment Image Servicing and Management): permite montar una imagen `.wim`, añadir/quitar controladores, paquetes de actualización o características, y volver a capturarla |
| `make install` (instalar el nuevo kernel) | Aplicar una actualización acumulativa vía Windows Update, o desplegar una imagen personalizada con DISM/MDT/SCCM |

**Comandos DISM relacionados (lo más parecido a "personalizar y reempaquetar el kernel"):**

| Comando | Función |
|---|---|
| `DISM /Mount-Image /ImageFile:install.wim /Index:1 /MountDir:C:\mount` | Monta una imagen de Windows para editarla sin arrancarla |
| `DISM /Image:C:\mount /Add-Package /PackagePath:actualizacion.cab` | Añade una actualización/paquete a la imagen montada |
| `DISM /Image:C:\mount /Add-Driver /Driver:C:\drivers /Recurse` | Añade controladores a la imagen (equivalente a incluir módulos en el `initrd`) |
| `DISM /Unmount-Image /MountDir:C:\mount /Commit` | Guarda los cambios y cierra la imagen |
| `DISM /Online /Get-Drivers` | Lista los controladores instalados en el sistema en marcha |

### 2.1 El "disco RAM inicial" en Windows: no hay equivalente directo

`initrd`/`initramfs` no tiene equivalente en Windows, porque el arranque de Windows carga los controladores de arranque crítico (disco, controlador de almacenamiento) directamente desde el propio proceso de arranque gestionado por `winload.exe`, sin necesidad de un sistema de ficheros temporal previo. El concepto más cercano es el **entorno WinPE** (Windows Preinstallation Environment), un sistema mínimo arrancable usado para instalar o reparar Windows — comparable en espíritu a un live-CD de rescate, no exactamente al `initrd`.

---

## 3. Consulta y modificación de parámetros del sistema en caliente

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| `/proc` (sistema de ficheros virtual con info del kernel) | No existe un sistema de ficheros virtual equivalente; la información análoga se obtiene vía **WMI/CIM** (`Get-CimInstance`), el **Registro de Windows**, o contadores de rendimiento (`Get-Counter`, ver Capítulo 2) |
| `/proc/cpuinfo` | `Get-CimInstance Win32_Processor` o `systeminfo` |
| `sysctl` (consultar/modificar parámetros en caliente) | No hay un comando único equivalente; los "parámetros del sistema" en Windows se modifican mayoritariamente a través del **Registro** (`HKLM\SYSTEM\CurrentControlSet\Control\...`), con `reg.exe` o los cmdlets `Get-ItemProperty`/`Set-ItemProperty` de PowerShell |
| `/etc/sysctl.conf` (parámetros persistentes) | El propio Registro ya es persistente; no requiere un fichero adicional. Para aplicar configuraciones de forma centralizada y repetible, se usa **Directiva de grupo (GPO)** |
| Aplicar cambios sin reiniciar (como muchos parámetros de `sysctl`) | Depende del parámetro: algunas claves del Registro se aplican en caliente, otras requieren reiniciar el servicio afectado o el propio sistema — no hay una garantía uniforme como con `sysctl -w` |

**Ejemplo de consulta/modificación vía Registro (PowerShell):**
```powershell
Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management' -Name LargeSystemCache
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management' -Name LargeSystemCache -Value 1
```

---

## 4. Gestión de controladores (módulos) en tiempo de ejecución

| Comando Linux | Equivalente en Windows Server 2025 |
|---|---|
| `lsmod` (módulos cargados) | `Get-CimInstance Win32_SystemDriver` (PowerShell), o el **Administrador de dispositivos** (`devmgmt.msc`) de forma gráfica |
| `modinfo` (info de un módulo) | `pnputil /enum-drivers` (lista los paquetes de controladores de terceros en el almacén), o `Get-WindowsDriver -Online` (con DISM/PowerShell) para más detalle |
| `insmod` (insertar un módulo por fichero) | `pnputil /add-driver archivo.inf /install` (añade e instala un paquete de controlador concreto) |
| `modprobe nombre` (insertar por nombre, resolviendo dependencias) | El **Administrador de dispositivos**, o `pnputil /scan-devices` (fuerza un nuevo escaneo Plug and Play que detecta e instala automáticamente los controladores necesarios) |
| `modprobe -r` / `rmmod` (eliminar un módulo) | `pnputil /delete-driver oem#.inf /uninstall` |
| `depmod` (regenerar dependencias de módulos) | No aplica: Windows resuelve las dependencias de controladores internamente sin un paso equivalente |
| `/etc/modprobe.d/` (configuración de carga de módulos) | No hay un fichero equivalente; el comportamiento de cada controlador se configura mediante sus propias propiedades en el Administrador de dispositivos o claves específicas del Registro |

### 4.1 `pnputil`: la herramienta principal de gestión de controladores

`pnputil.exe` es la herramienta de línea de comandos más parecida en función al conjunto `lsmod`/`insmod`/`modprobe`/`rmmod` de Linux, y viene incluida de serie desde Windows Vista.

| Comando | Función |
|---|---|
| `pnputil /enum-drivers` | Lista todos los paquetes de controladores de terceros en el almacén de controladores |
| `pnputil /enum-devices` | Lista los dispositivos reconocidos por el sistema |
| `pnputil /add-driver archivo.inf /install` | Añade e instala un controlador (admite `*.inf` y `/subdirs` para instalar varios a la vez) |
| `pnputil /delete-driver oem#.inf /uninstall` | Elimina un controlador, desinstalándolo de los dispositivos que lo usan |
| `pnputil /export-driver oem#.inf carpeta_destino` | Exporta un controlador del almacén a una carpeta (útil para backup o reutilización) |
| `pnputil /disable-device <ID>` / `/enable-device <ID>` | Deshabilita/habilita un dispositivo concreto |
| `pnputil /scan-devices` | Fuerza un nuevo escaneo Plug and Play |

---

## 5. Detección de hardware

### 5.1 Coldplug vs Hotplug

El concepto es idéntico en Windows: hardware que solo puede conectarse/desconectarse con el sistema apagado (RAM, tarjetas PCIe internas) frente a hardware conectable en caliente (USB, red, algunos discos SATA/NVMe con hot-swap).

### 5.2 El sistema Plug and Play (equivalente a `udev`)

| Elemento `udev` | Equivalente en Windows |
|---|---|
| `udevd` (demonio que escucha eventos de hardware) | Subsistema **Plug and Play (PnP)**, integrado en el propio núcleo y en el servicio **PlugPlay** (`services.msc`) |
| `/etc/udev/rules.d/` (reglas de qué módulo cargar y qué nombre asignar) | El **almacén de controladores** (Driver Store, `%WINDIR%\System32\DriverStore\`) y la información de los ficheros `.inf`, que definen qué controlador corresponde a cada ID de hardware |
| `udevadm monitor` (monitorizar eventos de hardware en directo) | **Visor de eventos** (`eventvwr.msc`), registro "Microsoft-Windows-Kernel-PnP/Configuration", o el Administrador de dispositivos con la opción "Mostrar dispositivos ocultos" |

### 5.3 Comandos para consultar hardware

| Comando Linux | Equivalente en Windows Server 2025 |
|---|---|
| `lspci` (dispositivos PCI/PCIe) | `Get-CimInstance Win32_PnPEntity` filtrando por bus PCI, o de forma gráfica el **Administrador de dispositivos** (`devmgmt.msc`); también `pnputil /enum-devices /class PCI` |
| `lsusb` (dispositivos USB) | `Get-PnpDevice -Class USB` (PowerShell), o el Administrador de dispositivos, categoría "Controladoras de bus serie universal (USB)" |
| Información general de hardware | `systeminfo`, `Get-ComputerInfo` (PowerShell), o **msinfo32.exe** (Información del sistema, gráfico) |
| Comprobar el estado/errores de un dispositivo | `Get-PnpDevice \| Where-Object Status -ne 'OK'` — lista dispositivos con problemas, similar a revisar mensajes de error de `dmesg` relacionados con hardware |

---

## 6. Resolución de problemas del kernel

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| `dmesg` (mensajes recientes del kernel) | **Visor de eventos** (`eventvwr.msc`), especialmente los registros **Sistema** y **Setup**; también `Get-WinEvent -LogName System` |
| `/var/log/messages`, `/var/log/syslog` | Registro de eventos de Windows (Event Log), consultable con `Get-WinEvent`/`Get-EventLog` |
| Pantallazo de error grave del kernel (kernel panic) | **BSOD (Blue Screen of Death)**; el volcado de memoria se guarda en `%WINDIR%\MEMORY.DMP` (o minivolcados en `%WINDIR%\Minidump\`), analizable con **WinDbg** |
| Depuración avanzada del núcleo | **Driver Verifier** (`verifier.exe`), herramienta integrada que fuerza condiciones de estrés sobre los controladores para detectar comportamientos incorrectos, algo sin equivalente directo en el libro pero muy usada en troubleshooting de "kernel" en Windows |
| Historial de fiabilidad del sistema (cuándo empezaron a fallar cosas) | **Monitor de confiabilidad** (Reliability Monitor, `perfmon /rel`), que muestra una línea temporal de errores de hardware, aplicaciones y Windows |

---

## 7. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 3) | Equivalente en Windows Server 2025 |
|---|---|
| Kernel compilable (`bzImage`, `make menuconfig`) | No aplica: núcleo cerrado y precompilado (`ntoskrnl.exe`); lo más cercano es personalizar imágenes con **DISM** |
| Módulos del kernel (`.ko`) | Controladores de dispositivo (`.sys`) |
| `lsmod`, `insmod`, `modprobe`, `rmmod` | `pnputil` (`/enum-drivers`, `/add-driver`, `/delete-driver`), Administrador de dispositivos |
| `initrd`/`initramfs` | No aplica; lo más cercano conceptualmente es **WinPE** |
| `/proc`, `sysctl` | WMI/CIM (`Get-CimInstance`), Registro de Windows |
| `udev` | Subsistema Plug and Play, servicio **PlugPlay**, Driver Store |
| `lspci`, `lsusb` | `Get-PnpDevice`, Administrador de dispositivos |
| `dmesg` | Visor de eventos (`eventvwr.msc`, `Get-WinEvent`) |
| Kernel panic | BSOD + volcado de memoria (`MEMORY.DMP`), analizable con WinDbg |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 3 ("Mastering the Kernel") del libro LPIC-2. Fuentes: conocimiento general de administración de Windows Server, verificado con documentación oficial de Microsoft Learn (sintaxis de `pnputil`). Dado que el modelo de kernel de Windows es cerrado y no compilable por el usuario, varios conceptos de este capítulo no tienen equivalente directo; se ha indicado explícitamente en cada caso.*
