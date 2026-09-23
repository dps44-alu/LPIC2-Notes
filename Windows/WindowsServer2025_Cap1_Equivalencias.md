# LPIC-2 · Capítulo 1: Starting a System
### Equivalencias en Windows Server 2025

Este documento recoge, para cada concepto visto en el Capítulo 1 del libro LPIC-2 (arranque, bootloaders, inicialización del sistema y recuperación), cuál es su equivalente en **Windows Server 2025**. Windows funciona de forma muy distinta a Linux: no hay archivos de texto para configurar el arranque ni scripts de inicio, sino una base de datos de arranque (**BCD**), el **Registro** y un gestor de servicios (**SCM**). Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entenderlo todo: en Windows **no hay GRUB, ni SysV init, ni systemd**.
- El arranque lo hace el **Windows Boot Manager** (`bootmgfw.efi`), que lee su configuración de la base de datos **BCD** (se edita con el comando `bcdedit`, nunca a mano).
- La inicialización la hace el propio kernel y el **Administrador de control de servicios** (Service Control Manager, proceso `services.exe`), que arranca los servicios según su configuración guardada en el Registro.
- No hay runlevels: hay **arranque normal**, **Modo seguro** (varias variantes) y el **Entorno de recuperación de Windows** (WinRE).

Casi todo se puede hacer de dos formas: con herramientas clásicas de la línea de comandos (`cmd.exe`: `bcdedit`, `sc.exe`, `shutdown`...) o con **PowerShell** (cmdlets tipo `Get-Service`). Windows Server 2025 trae Windows PowerShell 5.1 de serie; PowerShell 7 (`pwsh.exe`) se instala aparte. Todos los comandos de este documento se ejecutan en una consola **como Administrador**.

---

## 1. Arranque del sistema (firmware)

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| BIOS | Soportado (modo heredado), con disco MBR. Poco habitual hoy en servidores nuevos |
| UEFI | Opción recomendada y habitual, con disco GPT. Necesario para Secure Boot, TPM 2.0 moderno y funciones de seguridad basadas en virtualización |
| POST | Igual, es una fase del firmware común a cualquier sistema operativo |

Fases del arranque en Windows (equivalente a "firmware → GRUB → kernel → init" en Linux):

| Fase | Qué es | Archivo |
|---|---|---|
| Firmware | UEFI o BIOS, hace el POST y busca el gestor de arranque | — |
| Gestor de arranque | **Windows Boot Manager**: lee la BCD y muestra el menú si hay varias entradas | UEFI: `\EFI\Microsoft\Boot\bootmgfw.efi` (en la partición EFI). BIOS: `\bootmgr` (en la partición "Reservado para el sistema") |
| Cargador del sistema | **Windows Boot Loader**: carga el kernel, la HAL, el registro `SYSTEM` y los controladores de arranque | `C:\Windows\System32\winload.efi` (UEFI) o `winload.exe` (BIOS) |
| Kernel | Núcleo del sistema + capa de abstracción de hardware | `C:\Windows\System32\ntoskrnl.exe` y `hal.dll` |
| Primer proceso en modo usuario | **Session Manager**: prepara el sistema y lanza los siguientes procesos | `C:\Windows\System32\smss.exe` |
| Inicialización | `wininit.exe` arranca `services.exe` (gestor de servicios) y `lsass.exe` (seguridad y autenticación) | `C:\Windows\System32\` |
| Inicio de sesión | `winlogon.exe` muestra la pantalla de inicio de sesión | `C:\Windows\System32\winlogon.exe` |

Equivalencias de procesos básicos:

| Linux | Windows Server 2025 |
|---|---|
| Kernel (PID 0 / hilos del kernel) | Proceso `System Idle Process` (PID 0) y `System` (PID 4) |
| `init` / `systemd` (PID 1) | No hay un proceso único equivalente. `smss.exe` es el primer proceso en modo usuario, y `services.exe` hace la función de gestor de servicios |
| `getty` / `login` | `winlogon.exe` + `LogonUI.exe` |

Para ver los procesos y los servicios que contiene cada uno: `tasklist /svc` o `Get-Process`.

---

## 2. Comandos para ver el proceso de arranque

En Windows **no existe `dmesg`**. Los mensajes del arranque se guardan en los **registros de eventos** (Event Logs), que son archivos binarios `.evtx`.

| Comando / Archivo | Qué hace |
|---|---|
| `eventvwr.msc` | Visor de eventos (herramienta gráfica). Registro **Sistema** = equivalente a los mensajes del kernel y del arranque |
| `Get-WinEvent -LogName System -MaxEvents 50` | Muestra los 50 eventos más recientes del registro Sistema (PowerShell) |
| `wevtutil qe System /c:20 /rd:true /f:text` | Igual desde `cmd`: 20 eventos más recientes en formato texto |
| `C:\Windows\System32\winevt\Logs\` | Carpeta donde se guardan los registros (`System.evtx`, `Application.evtx`...). Equivale a `/var/log` |
| `C:\Windows\ntbtlog.txt` | Registro de arranque: lista de controladores cargados y no cargados. Solo se crea si se activa (`bcdedit /set {current} bootlog yes` o opción "Habilitar el registro de arranque") |
| `(Get-CimInstance Win32_OperatingSystem).LastBootUpTime` | Fecha y hora del último arranque |
| `systeminfo` | Información general, incluida la hora del último arranque |

Eventos útiles del registro **Sistema** para estudiar arranques y apagados:

| ID | Origen | Significado |
|---|---|---|
| 12 | Kernel-General | El sistema operativo ha arrancado (hora exacta de inicio) |
| 13 | Kernel-General | El sistema operativo se está apagando |
| 6005 / 6006 | EventLog | El servicio de registro de eventos se inició / se detuvo (marca de arranque / apagado limpio) |
| 6008 | EventLog | El apagado anterior fue **inesperado** |
| 41 | Kernel-Power | El sistema se reinició sin apagarse correctamente (cuelgue, corte de luz...) |
| 1074 | User32 | Un usuario o programa pidió apagar o reiniciar (incluye el motivo) |
| 7000, 7009, 7023, 7031, 7034 | Service Control Manager | Errores de servicios: no arrancó, tiempo de espera agotado, se detuvo inesperadamente... |

Ejemplo, ver los últimos arranques:
```
Get-WinEvent -FilterHashtable @{LogName='System'; Id=12} -MaxEvents 5
```

Ver más detalle durante el arranque (equivalente a un arranque "verbose" o a quitar `quiet` en GRUB):

| Opción | Función |
|---|---|
| `bcdedit /set {current} sos on` | Muestra en pantalla los controladores según se cargan |
| `bcdedit /set {current} bootlog yes` | Crea el archivo `ntbtlog.txt` |
| Directiva "Mostrar mensajes de estado muy detallados" | Muestra qué está haciendo Windows durante el inicio, apagado e inicio de sesión (en `gpedit.msc` → Configuración del equipo → Plantillas administrativas → Sistema) |
| `msconfig` → pestaña **Arranque** | Casillas "Registro de arranque" e "Información de arranque del SO" (solo con escritorio, no en Server Core) |

---

## 3. Bootloaders

### 3.1 LILO, GRUB Legacy y GRUB2 → Windows Boot Manager y la BCD

Windows no usa LILO ni GRUB. Su gestor propio es el **Windows Boot Manager**, y su configuración se guarda en la **BCD** (Boot Configuration Data), una base de datos binaria con el mismo formato que una rama del Registro.

Un paralelismo histórico: en Windows XP / Server 2003 y anteriores se usaba `NTLDR` con un archivo de texto `boot.ini` (sería el equivalente a LILO / GRUB Legacy). Desde Windows Vista / Server 2008 se usa la BCD (equivalente a GRUB2).

| Elemento GRUB2 (Linux) | Equivalente en Windows Server 2025 |
|---|---|
| `/boot/grub/` | Partición EFI: `\EFI\Microsoft\Boot\` (UEFI) o partición "Reservado para el sistema": `\Boot\` (BIOS) |
| `/boot/grub/grub.cfg` | Archivo `BCD` dentro de esa carpeta. **No se edita a mano**: se usa `bcdedit` |
| `/etc/default/grub` | No hay archivo: los valores globales (tiempo de espera, entrada por defecto) están en la entrada `{bootmgr}` de la BCD |
| `/etc/grub.d/40_custom` | Entradas nuevas creadas con `bcdedit /copy` o `bcdedit /create` |
| `grub-mkconfig` / `update-grub` | No hace falta: `bcdedit` modifica la BCD directamente y el cambio se aplica en el siguiente arranque |
| `grub-install` | `bcdboot` (ver apartado 3.2) |
| Teclas `e` (editar) y `c` (consola) | No existe edición ni consola en el menú de arranque. Se usan las "Opciones avanzadas" (ver apartado 7.1) |
| `menuentry` | Entrada de tipo "Windows Boot Loader" (cargador de sistema operativo) |

**Identificadores especiales de la BCD** (nombres fijos que se usan en lugar de largos GUID):

| Identificador | Qué representa |
|---|---|
| `{bootmgr}` | El propio Windows Boot Manager (menú, tiempo de espera, entrada por defecto) |
| `{current}` | La entrada del sistema que está arrancado ahora mismo |
| `{default}` | La entrada que arranca por defecto |
| `{fwbootmgr}` | El gestor de arranque del firmware UEFI (las entradas de la NVRAM) |
| `{memdiag}` | Diagnóstico de memoria de Windows |
| `{ramdiskoptions}` | Opciones para arrancar una imagen en memoria (WinPE, WinRE) |

**Comandos principales de `bcdedit`:**

| Comando | Función | Equivalente Linux aproximado |
|---|---|---|
| `bcdedit` o `bcdedit /enum` | Muestra las entradas activas | `cat /boot/grub/grub.cfg` |
| `bcdedit /enum all` | Muestra todas las entradas, incluidas las ocultas | — |
| `bcdedit /v` | Muestra los GUID completos en vez de los nombres cortos | — |
| `bcdedit /timeout 10` | Espera 10 segundos en el menú | `GRUB_TIMEOUT=10` |
| `bcdedit /default {id}` | Elige la entrada que arranca por defecto | `GRUB_DEFAULT` |
| `bcdedit /displayorder {id1} {id2}` | Cambia el orden de las entradas del menú | — |
| `bcdedit /copy {current} /d "Descripción"` | Duplica una entrada con otro nombre (devuelve el GUID de la nueva) | Añadir una entrada en `40_custom` |
| `bcdedit /set {id} description "Nombre"` | Cambia el nombre que se ve en el menú | Cambiar el título de un `menuentry` |
| `bcdedit /set {id} opción valor` | Añade o cambia una opción de una entrada | Añadir parámetros a la línea `linux` |
| `bcdedit /deletevalue {id} opción` | Quita una opción | Borrar un parámetro |
| `bcdedit /delete {id}` | Borra una entrada | Borrar un `menuentry` |
| `bcdedit /bootsequence {id}` | Arranca esa entrada **solo en el próximo reinicio** | `grub-reboot` |
| `bcdedit /export C:\copia_bcd` | Copia de seguridad de la BCD | Copiar `grub.cfg` |
| `bcdedit /import C:\copia_bcd` | Restaura la copia | — |
| `bcdedit /store archivo ...` | Trabaja con un archivo BCD distinto del activo | — |
| `bcdedit /set {bootmgr} displaybootmenu yes` | Muestra siempre el menú aunque solo haya una entrada | `GRUB_TIMEOUT_STYLE=menu` |
| `bcdedit /set {default} bootmenupolicy legacy` | Menú clásico en modo texto, con la tecla `F8` activa | — |
| `bcdedit /set {default} bootmenupolicy standard` | Menú gráfico (valor por defecto) | — |

Importante: en **PowerShell** las llaves `{ }` tienen un significado especial, así que hay que poner los identificadores entre comillas: `bcdedit /set '{current}' bootlog yes`. En `cmd` no hace falta.

También existen cmdlets de PowerShell para la BCD (`Get-BcdStore`, `Get-BcdEntry`...), pero `bcdedit` sigue siendo la herramienta de referencia y la que aparece en toda la documentación.

**Opciones que se pasan al kernel** (equivalente a añadir parámetros a la línea `linux` de GRUB). Se aplican con `bcdedit /set {current} opción valor`:

| Opción | Función |
|---|---|
| `safeboot minimal` / `safeboot network` | Arranca en Modo seguro (ver apartado 7.1) |
| `sos on` | Muestra los controladores al cargarse (arranque detallado) |
| `bootlog yes` | Crea `ntbtlog.txt` |
| `quietboot on` | Oculta la animación de arranque |
| `numproc 2` | Usa solo 2 procesadores (pruebas) |
| `truncatememory 0x100000000` | Ignora la memoria por encima de esa dirección (aquí, 4 GB) |
| `debug on` (+ `bcdedit /dbgsettings`) | Activa la depuración del kernel |
| `testsigning on` | Permite controladores firmados solo para pruebas (requiere Secure Boot desactivado) |
| `hypervisorlaunchtype off` / `auto` | Desactiva o activa el hipervisor (Hyper-V) al arrancar |
| `bootstatuspolicy ignoreallfailures` | No lanza la reparación automática tras un arranque fallido |
| `recoveryenabled no` | Desactiva el paso automático a WinRE |

**Consola por puerto serie** (equivalente a `console=ttyS0` en Linux, útil en servidores sin pantalla):

| Comando | Función |
|---|---|
| `bcdedit /ems {current} on` | Activa los Servicios de administración de emergencia (EMS) para esa entrada |
| `bcdedit /emssettings EMSPORT:1 EMSBAUDRATE:115200` | Puerto COM1 a 115200 baudios |
| `bcdedit /bootems {bootmgr} on` | El propio menú de arranque también se ve por el puerto serie |

Con EMS activo se accede a la **SAC** (Special Administration Console), una consola mínima para reiniciar, ver procesos o abrir un `cmd` sin red ni pantalla.

### 3.2 Archivos de arranque y cómo instalar o reparar el gestor (equivalente a `grub-install`)

**Distribución típica de un disco de sistema UEFI/GPT** (la crea el instalador):

| Partición | Formato | Contenido |
|---|---|---|
| Partición del sistema EFI (ESP) | FAT32, ~100 MB | `\EFI\Microsoft\Boot\bootmgfw.efi`, `\EFI\Microsoft\Boot\BCD` y `\EFI\Boot\bootx64.efi` (ruta de arranque por defecto) |
| MSR (reservada de Microsoft) | Sin formato, 16 MB | Uso interno de Windows. No tiene letra |
| Windows | **NTFS** (el sistema no puede arrancar desde ReFS) | `C:\Windows\...` |
| Recuperación | NTFS | `\Recovery\WindowsRE\Winre.wim` (imagen del entorno de recuperación) |

En BIOS/MBR la ESP se sustituye por la partición "Reservado para el sistema" (NTFS), que contiene `\bootmgr` y `\Boot\BCD`.

La ESP no tiene letra de unidad. Para verla: `mountvol S: /s` (solo en UEFI) o con `diskpart` (`select disk 0` → `list partition` → `select partition 1` → `assign letter=S`).

| Situación | Comando en Windows Server 2025 |
|---|---|
| Recrear los archivos de arranque y la BCD (UEFI) | `bcdboot C:\Windows /s S: /f UEFI` (S: es la letra asignada a la ESP) |
| Lo mismo en BIOS | `bcdboot C:\Windows /s S: /f BIOS` |
| Crear archivos para ambos modos | `bcdboot C:\Windows /s S: /f ALL` |
| Escribir el código de arranque en el MBR (BIOS) | `bootrec /fixmbr` |
| Escribir el sector de arranque de la partición (BIOS) | `bootrec /fixboot` (en UEFI suele dar "Acceso denegado"; se usa `bcdboot`) |
| Buscar instalaciones de Windows no presentes en la BCD | `bootrec /scanos` |
| Reconstruir la BCD completa | `bootrec /rebuildbcd` |
| Actualizar el código de arranque de todas las particiones | `bootsect /nt60 SYS /mbr` |

`bootrec` y `bootsect` se usan desde el entorno de recuperación (WinRE) o desde el medio de instalación, no con el sistema en marcha. `bcdboot` funciona en ambos casos.

### 3.3 Gestión de entradas UEFI (equivalente a `efibootmgr`)

Windows no tiene `efibootmgr`, pero `bcdedit` puede leer y cambiar las entradas de la NVRAM del firmware a través de `{fwbootmgr}`:

| Comando | Función | Equivalente `efibootmgr` |
|---|---|---|
| `bcdedit /enum firmware` | Lista las entradas de arranque UEFI | `efibootmgr -v` |
| `bcdedit /set {fwbootmgr} displayorder {id} /addfirst` | Pone esa entrada la primera en el orden de arranque | `efibootmgr -o` |
| `bcdedit /set {fwbootmgr} bootsequence {id}` | Arranca desde esa entrada solo la próxima vez | `efibootmgr -n` |
| `bcdedit /delete {id}` | Borra una entrada del firmware | `efibootmgr -B` |
| `shutdown /r /fw /t 0` | Reinicia y entra directamente en la configuración del firmware UEFI | `systemctl reboot --firmware-setup` |

### 3.4 Bootloaders alternativos

| Bootloader del libro | Situación en Windows Server 2025 |
|---|---|
| systemd-boot | No aplica. Windows solo usa su Windows Boot Manager. Otros gestores (GRUB, rEFInd) pueden lanzar `bootmgfw.efi` en arranque dual |
| U-Boot | No aplica: Windows Server 2025 se instala en equipos x64 con firmware UEFI o BIOS estándar |
| SYSLINUX (arranque desde USB FAT) | Un USB de instalación de Windows usa FAT32 con los archivos UEFI de Microsoft. Se prepara con `diskpart` (`clean`, `create partition primary`, `format fs=fat32 quick`, `active`) y copiando el contenido de la ISO. Si `sources\install.wim` pasa de 4 GB (límite de FAT32), se divide con `DISM /Split-Image /ImageFile:install.wim /SWMFile:install.swm /FileSize:3800` |
| EXTLINUX | No aplica |
| ISOLINUX (CD/DVD) | Para crear una ISO arrancable se usa `oscdimg`, del **Windows ADK**. Archivos de arranque: `etfsboot.com` (BIOS) y `efisys.bin` (UEFI). Ejemplo: `oscdimg -m -o -u2 -udfver102 -bootdata:2#p0,e,betfsboot.com#pEF,e,befisys.bin C:\origen C:\salida.iso` |
| Imagen de rescate personalizada | **WinPE** (entorno de preinstalación de Windows), del Windows ADK + complemento WinPE: `copype amd64 C:\WinPE_amd64`, después `MakeWinPEMedia /ISO C:\WinPE_amd64 C:\WinPE.iso` o `MakeWinPEMedia /UFD C:\WinPE_amd64 F:` (USB) |
| PXELINUX (arranque por red) | Rol **Windows Deployment Services (WDS)**. Ver tabla siguiente |
| MEMDISK (imagen en memoria) | WinPE y WinRE ya arrancan cargando su imagen `.wim` completa en un disco RAM (unidad `X:`), usando las opciones `{ramdiskoptions}` de la BCD y el archivo `boot.sdi`. Otra función parecida: **arranque nativo desde VHDX** (un disco virtual como disco de sistema): se monta el VHDX y se ejecuta `bcdboot V:\Windows` |

**WDS (equivalente a PXELINUX + TFTP):**

| Elemento | Valor |
|---|---|
| Instalación del rol | `Install-WindowsFeature WDS -IncludeManagementTools` |
| Inicializar el servidor | `wdsutil /Initialize-Server /RemInst:"D:\RemoteInstall"` |
| Carpeta raíz (equivale a `/tftpboot`) | `D:\RemoteInstall` |
| Programas de arranque de red | `RemoteInstall\Boot\x64\`: `wdsnbp.com` y `pxeboot.n12` (BIOS), `wdsmgfw.efi` y `bootmgfw.efi` (UEFI) |
| Añadir una imagen de arranque | `wdsutil /Add-Image /ImageFile:"C:\WinPE\boot.wim" /ImageType:Boot` |
| Configuración por equipo (equivale a un archivo por MAC en `pxelinux.cfg/`) | Dispositivos preconfigurados: `wdsutil /Add-Device /Device:SRV01 /ID:00-15-5D-01-02-03` |
| Consola gráfica | `wdsmgmt.msc` |
| DHCP | Si el DHCP no está en el mismo servidor, se configuran las opciones 66 (servidor) y 67 (archivo de arranque) |

Situación actual de WDS: Microsoft lo considera parcialmente obsoleto. Ya **no permite instalar Windows Server 2025 de principio a fin** usando el `boot.wim` del medio de instalación, y la instalación desatendida con `unattend.xml` a través de WDS viene **desactivada por defecto** desde las actualizaciones de abril de 2026 por un problema de seguridad. Sigue sirviendo como servidor PXE que arranca una imagen **WinPE personalizada**; para despliegues nuevos Microsoft recomienda WinPE propio o Configuration Manager.

### 3.5 Secure Boot (UEFI)

| Opción del libro para Linux | Situación en Windows Server 2025 |
|---|---|
| Desactivar Secure Boot | Posible pero no recomendado: Windows viene firmado por Microsoft y arranca sin problemas con Secure Boot activado |
| Usar tu propia clave de firma | Posible (claves propias en el firmware), pero muy poco habitual en Windows |
| Shim firmado por el proveedor | No hace falta: el `bootmgfw.efi` ya está firmado con las claves de Microsoft que traen los equipos |

| Comando | Función |
|---|---|
| `Confirm-SecureBootUEFI` | Devuelve `True` si Secure Boot está activo (da error en equipos BIOS) |
| `msinfo32` | "Información del sistema": muestra el modo de BIOS (UEFI/Heredado) y el "Estado de arranque seguro" |
| `Get-Tpm` / `tpm.msc` | Estado del chip TPM (usado por BitLocker y el arranque medido) |

Aviso práctico: Microsoft está sustituyendo los certificados de Secure Boot de 2011, que empezaron a caducar en junio de 2026, por los nuevos de 2023. Se instalan mediante Windows Update; conviene mantener el servidor actualizado.

Aviso sobre **BitLocker**: si el disco de sistema está cifrado, cambiar la BCD o el firmware puede hacer que en el siguiente arranque pida la clave de recuperación. Antes de hacer cambios: `Suspend-BitLocker -MountPoint C: -RebootCount 1` (o `manage-bde -protectors -disable C:`). Ver estado: `manage-bde -status`.

---

## 4. Inicialización del sistema: el Administrador de control de servicios (equivalente a SysV init)

Windows **no usa SysV**. Diferencias clave:
- **No hay runlevels numerados**. Hay arranque normal, Modo seguro y entorno de recuperación.
- **No hay `/etc/inittab`** ni carpetas `rcN.d` con enlaces `S`/`K`.
- Cada servicio tiene un **tipo de inicio** (Automático, Manual, Deshabilitado...) guardado en el Registro.
- El orden de arranque lo calcula el sistema a partir de las **dependencias** de cada servicio y de los **grupos** de carga.

### 4.1 Equivalencia de runlevels

| Runlevel Linux | Equivalente en Windows Server 2025 |
|---|---|
| 0 (apagar) | `shutdown /s /t 0` o `Stop-Computer` |
| 1 (mono-usuario) | **Modo seguro** (solo controladores y servicios mínimos) o el **entorno de recuperación (WinRE)** |
| 2, 3, 4, 5 (multiusuario) | Arranque normal. Es el único modo de trabajo habitual |
| 6 (reiniciar) | `shutdown /r /t 0` o `Restart-Computer` |

Diferencia "texto / gráfico" (runlevel 3 frente a 5 en Red Hat): en Windows Server se elige **al instalar**:
- **Server Core**: sin escritorio, se administra por consola (`sconfig`, PowerShell) o de forma remota. Parecido a un runlevel 3.
- **Server con Experiencia de escritorio**: con interfaz gráfica completa. Parecido a un runlevel 5.

Desde Windows Server 2016 **no se puede cambiar** de uno a otro después de instalar; hay que reinstalar.

Para saber en qué modo ha arrancado el sistema (equivalente al comando `runlevel`):
```
(Get-CimInstance Win32_ComputerSystem).BootupState
```
Devuelve `Normal boot`, `Fail-safe boot` (Modo seguro) o `Fail-safe with network boot` (Modo seguro con funciones de red).

### 4.2 Dónde se guarda la configuración (equivalente a los archivos y carpetas de SysV)

| Ubicación | Función | Equivalente Linux |
|---|---|---|
| `HKLM\SYSTEM\CurrentControlSet\Services\` | Una clave por cada servicio y controlador: programa, tipo de inicio, dependencias, cuenta... | `/etc/init.d/` + enlaces de runlevels |
| Valor `Start` dentro de cada servicio | Tipo de inicio (ver tabla siguiente) | Enlaces `S`/`K` |
| Valor `DependOnService` | Servicios que deben arrancar antes | Dependencias LSB / `After=` |
| `HKLM\SYSTEM\CurrentControlSet\Control\ServiceGroupOrder` | Orden de carga de los grupos de servicios y controladores | Números de orden en los enlaces `S20...` |
| `HKLM\SYSTEM\CurrentControlSet\Control\SafeBoot\Minimal` y `\Network` | Controladores y servicios que se cargan en Modo seguro | Scripts del runlevel 1 |
| `C:\Windows\System32\config\SYSTEM` | Archivo físico donde se guarda esa parte del Registro | — |
| Tareas programadas con desencadenador "Al iniciar el sistema" | Ejecutar un programa o script al arrancar, antes de iniciar sesión | `/etc/rc.local` |
| Scripts de inicio de directiva de grupo | `gpedit.msc` → Configuración del equipo → Configuración de Windows → Scripts (inicio o apagado). Se guardan en `C:\Windows\System32\GroupPolicy\Machine\Scripts\Startup\` | `/etc/rc.local` |
| `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp` | Programas que se abren al **iniciar sesión** cualquier usuario (no al arrancar el sistema) | Autostart del escritorio |

**Tipos de inicio de un servicio** (valor `Start` del Registro):

| Valor | Nombre | Significado |
|---|---|---|
| 0 | Boot | Controlador cargado por el cargador de arranque (`winload`) |
| 1 | System | Controlador cargado durante la inicialización del kernel |
| 2 | Automatic | Arranca siempre con el sistema. Si además `DelayedAutostart = 1`, es **Automático (inicio retrasado)**: arranca un poco después, para no ralentizar el inicio |
| 3 | Manual | Solo arranca si alguien o algo lo pide |
| 4 | Disabled | No puede arrancar |

Además, un servicio puede tener **desencadenadores** (trigger-start): arrancar solo cuando pasa algo (se conecta un dispositivo, llega tráfico a un puerto, se une a un dominio...). Es parecido a la activación por socket o por ruta de systemd. Se consultan con `sc.exe qtriggerinfo nombre`.

### 4.3 Comandos para gestionar servicios (equivalente a `chkconfig` / `update-rc.d` / `systemctl`)

**Con PowerShell:**

| Comando | Función | Equivalente Linux |
|---|---|---|
| `Get-Service` | Lista todos los servicios y su estado | `systemctl list-units --type=service` |
| `Get-Service sshd \| Select-Object *` | Todos los datos de un servicio, incluido `StartType` | `systemctl status sshd` |
| `Start-Service sshd` | Arranca un servicio | `systemctl start` |
| `Stop-Service sshd` | Para un servicio | `systemctl stop` |
| `Restart-Service sshd` | Reinicia | `systemctl restart` |
| `Suspend-Service` / `Resume-Service` | Pausa / reanuda (si el servicio lo admite) | — |
| `Set-Service sshd -StartupType Automatic` | Arranca siempre con el sistema | `systemctl enable` / `chkconfig on` / `update-rc.d enable` |
| `Set-Service sshd -StartupType Manual` | Solo arranque manual | — |
| `Set-Service sshd -StartupType Disabled` | Lo deshabilita | `systemctl disable` / `systemctl mask` |
| `Get-Service sshd -RequiredServices` | Servicios de los que depende | `systemctl list-dependencies` |
| `Get-Service sshd -DependentServices` | Servicios que dependen de él | `systemctl list-dependencies --reverse` |
| `New-Service -Name MiServicio -BinaryPathName "C:\ruta\prog.exe" -StartupType Automatic` | Crea un servicio nuevo | Crear un archivo `.service` |
| `Get-CimInstance Win32_Service \| Select Name, StartMode, State` | Tabla con tipo de inicio y estado de todos | `systemctl list-unit-files` |

**Con `sc.exe`** (herramienta clásica, más completa):

| Comando | Función |
|---|---|
| `sc.exe query` | Lista los servicios en marcha |
| `sc.exe queryex sshd` | Estado de un servicio, con su PID |
| `sc.exe qc sshd` | Configuración: programa, tipo de inicio, dependencias, cuenta |
| `sc.exe start sshd` / `sc.exe stop sshd` | Arranca / para |
| `sc.exe config sshd start= auto` | Tipo de inicio automático (otros valores: `delayed-auto`, `demand` = manual, `disabled`) |
| `sc.exe config sshd depend= Tcpip/Dnscache` | Define dependencias (separadas por `/`) |
| `sc.exe create MiServicio binPath= "C:\ruta\prog.exe" start= auto` | Crea un servicio |
| `sc.exe delete MiServicio` | Borra un servicio |
| `sc.exe failure sshd reset= 86400 actions= restart/60000/restart/60000//` | Qué hacer si el servicio falla: aquí, reiniciarlo al minuto (dos veces). Equivale a `Restart=` de systemd |
| `sc.exe qfailure sshd` | Muestra la configuración de fallos |

Detalles importantes de `sc.exe`:
- En PowerShell hay que escribir `sc.exe`, porque `sc` a secas es un alias de otro comando (`Set-Content`).
- La sintaxis exige un **espacio después del `=`**: `start= auto` (correcto), `start=auto` (error).
- Para el inicio retrasado se usa `sc.exe config nombre start= delayed-auto`, porque `Set-Service` de PowerShell 5.1 no tiene esa opción (PowerShell 7 sí: `AutomaticDelayedStart`).

**Herramientas gráficas:** `services.msc` (consola de Servicios), el Administrador del servidor y Windows Admin Center. En Server Core se trabaja con PowerShell o de forma remota.

Un servicio de Windows debe ser un programa preparado para ello. Para lanzar al arrancar un programa o script normal (equivalente a `/etc/rc.local`), lo habitual es una **tarea programada**:
```
schtasks /create /tn "InicioPersonal" /tr "C:\scripts\inicio.cmd" /sc onstart /ru SYSTEM
```

No hace falta un equivalente a `systemctl daemon-reload`: los cambios hechos con `sc.exe` o `Set-Service` se aplican al momento.

### 4.4 Comandos para gestionar el estado del sistema completo

| Comando Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `runlevel` | `(Get-CimInstance Win32_ComputerSystem).BootupState` | Indica si el arranque fue normal o en Modo seguro |
| `init 0` / `telinit 0` | `shutdown /s /t 0` | Apaga de inmediato |
| `init 6` / `telinit 6` | `shutdown /r /t 0` | Reinicia de inmediato |
| `init 1` (mono-usuario) | `bcdedit /set {current} safeboot minimal` + reiniciar | Próximos arranques en Modo seguro (ver 7.1) |
| `shutdown -h now` / `poweroff` | `shutdown /s /t 0` o `Stop-Computer` | Apaga |
| `shutdown -r now` / `reboot` | `shutdown /r /t 0` o `Restart-Computer` | Reinicia |
| `shutdown -r +15 "mensaje"` | `shutdown /r /t 900 /c "mensaje"` | Reinicia en 15 minutos (900 segundos) avisando a los usuarios conectados |
| `shutdown -c` | `shutdown /a` | Cancela un apagado o reinicio programado |
| `halt` | `shutdown /p` | Apaga al momento, sin cuenta atrás |
| — | `shutdown /r /o /t 0` | Reinicia y abre las **opciones avanzadas de arranque** (WinRE) |
| — | `shutdown /r /fw /t 0` | Reinicia y entra en el firmware UEFI |

Opciones útiles de `shutdown.exe`:

| Opción | Función |
|---|---|
| `/s` | Apagar |
| `/r` | Reiniciar |
| `/p` | Apagar ya, sin espera ni aviso |
| `/t N` | Esperar N segundos |
| `/c "texto"` | Mensaje que se muestra a los usuarios |
| `/f` | Cerrar los programas a la fuerza, sin preguntar |
| `/a` | Cancelar un apagado programado |
| `/d p:4:1` | Motivo del apagado (queda en el evento 1074). `p` = planeado |
| `/m \\SERVIDOR` | Actuar sobre un equipo remoto |
| `/o` | Junto con `/r`, ir al menú de opciones avanzadas |

PowerShell:

| Comando | Función |
|---|---|
| `Restart-Computer -Force` | Reinicia aunque haya usuarios conectados |
| `Restart-Computer -ComputerName SRV01 -Wait -For PowerShell` | Reinicia un servidor remoto y espera a que vuelva a estar disponible |
| `Stop-Computer -Force` | Apaga |

Avisar a los usuarios antes de un corte (equivale a los avisos de `shutdown` y a `wall`):
- `quser` (o `query user`): lista los usuarios con sesión abierta (local y Escritorio remoto).
- `msg * "El servidor se reiniciará en 15 minutos"`: envía un mensaje a todas las sesiones.

En **Server Core**, la utilidad de texto `sconfig` también tiene opciones para reiniciar y apagar el servidor.

---

## 5. systemd

**Windows Server 2025 no usa systemd.** Todo lo del apartado 5 del libro se sustituye por el Administrador de control de servicios del apartado 4.

| Concepto systemd | Equivalente en Windows Server 2025 |
|---|---|
| Unit `.service` | Servicio (clave en `HKLM\SYSTEM\CurrentControlSet\Services\nombre`) |
| Target | No existe; solo arranque normal y Modo seguro |
| `default.target` | Siempre arranque normal, salvo que la BCD indique `safeboot` |
| `/lib/systemd/system/` y `/etc/systemd/system/` | El Registro (clave `Services`) |
| `systemctl` | `Get-Service`, `Start-Service`, `Set-Service`... o `sc.exe` |
| `systemctl list-units` | `Get-Service` |
| `systemctl isolate` | No hay equivalente directo |
| `systemctl daemon-reload` | No hace falta |
| `After=`, `Requires=` | Dependencias (`DependOnService`, `sc.exe config ... depend=`) |
| `Restart=on-failure` | Acciones de recuperación (`sc.exe failure` o pestaña **Recuperación** en `services.msc`) |
| `User=` | Cuenta del servicio: `LocalSystem`, `LocalService`, `NetworkService` o una cuenta de dominio / gMSA |
| Unidades `.socket` / `.path` | Servicios con desencadenadores (trigger-start) |
| Unidades `.timer` | Tareas programadas (`taskschd.msc`, `schtasks`, `Register-ScheduledTask`) |
| `journalctl` | Visor de eventos / `Get-WinEvent` |

---

## 6. Upstart

No existe en Windows. No aplica.

---

## 7. Recuperación del sistema

### 7.1 Modo seguro y entorno de recuperación (equivalente al modo mono-usuario)

Windows tiene dos herramientas que juntas cubren lo que en Linux hace el modo mono-usuario:
- **Modo seguro** (Safe Mode): arranca el Windows instalado con los controladores y servicios mínimos.
- **WinRE** (Windows Recovery Environment): un sistema pequeño que arranca desde su propia imagen en memoria (unidad `X:`), dejando libre el disco del sistema para repararlo. Incluye un **Símbolo del sistema**.

**Variantes del Modo seguro:**

| Variante | Qué carga | Comando `bcdedit` |
|---|---|---|
| Modo seguro | Solo lo mínimo, sin red | `bcdedit /set {current} safeboot minimal` |
| Modo seguro con funciones de red | Lo mínimo + red | `bcdedit /set {current} safeboot network` |
| Modo seguro con símbolo del sistema | Lo mínimo, y abre `cmd` en lugar del escritorio | `bcdedit /set {current} safeboot minimal` + `bcdedit /set {current} safebootalternateshell yes` |
| Modo de restauración de servicios de directorio (DSRM) | Solo en controladores de dominio: arranca sin Active Directory para repararlo | `bcdedit /set {current} safeboot dsrepair` |

Después se reinicia. Para **volver al arranque normal**:
```
bcdedit /deletevalue {current} safeboot
bcdedit /deletevalue {current} safebootalternateshell
```
(el segundo solo si se usó). Si no se quita, el servidor arrancará siempre en Modo seguro.

Truco: crear una entrada fija de Modo seguro en el menú de arranque (parecido a tener una entrada "recovery mode" en GRUB):
```
bcdedit /copy {current} /d "Windows Server 2025 - Modo seguro"
bcdedit /set {GUID-devuelto} safeboot minimal
```

**Formas de llegar al menú de opciones avanzadas / WinRE:**

1. Desde el sistema en marcha: `shutdown /r /o /t 0`.
2. Con escritorio: mantener `Mayús` pulsada al hacer clic en **Reiniciar**.
3. `reagentc /boottore` + reiniciar: el siguiente arranque va directamente a WinRE.
4. Automáticamente, tras **dos arranques fallidos seguidos** (Reparación automática).
5. Desde el **medio de instalación** (ISO o USB): tras elegir el idioma, pulsar **Reparar el equipo**.
6. Tecla `F8` al arrancar: **solo funciona** si antes se activó el menú clásico con `bcdedit /set {default} bootmenupolicy legacy`.

Dentro de WinRE, en **Solucionar problemas**, aparecen opciones como Símbolo del sistema, Recuperación de imagen del sistema, Configuración de inicio y Configuración del firmware UEFI (según el equipo). **Configuración de inicio** ofrece, entre otras: habilitar el registro de arranque, vídeo de baja resolución, las tres variantes del Modo seguro, deshabilitar la comprobación de firmas de controladores y deshabilitar el reinicio automático tras un error.

**Comando `reagentc`** (gestiona WinRE):

| Comando | Función |
|---|---|
| `reagentc /info` | Muestra si WinRE está activo y dónde está su imagen |
| `reagentc /enable` / `reagentc /disable` | Activa / desactiva WinRE |
| `reagentc /boottore` | El próximo arranque entra en WinRE |

Pasos típicos dentro del **Símbolo del sistema de WinRE**:

| Comando | Función |
|---|---|
| `diskpart` → `list volume` | Ver qué letra tiene cada volumen (en WinRE el disco de Windows puede no ser `C:`) |
| `chkdsk C: /f` | Revisar y reparar el sistema de archivos |
| `bcdboot C:\Windows /s S: /f UEFI` | Reparar el arranque |
| `sfc /scannow /offbootdir=C:\ /offwindir=C:\Windows` | Revisar los archivos protegidos de Windows sin arrancarlo |
| `DISM /Image:C:\ /Cleanup-Image /RevertPendingActions` | Deshacer una actualización a medio instalar que impide arrancar |
| `exit` | Salir y volver al menú de WinRE |

**Proteger el acceso** (equivalente a poner contraseña a GRUB o marcar la consola como `insecure` en FreeBSD):
- El Símbolo del sistema del WinRE **instalado** pide usuario y contraseña de un administrador local.
- Pero si alguien arranca desde un USB o ISO de instalación, obtiene un `cmd` **sin contraseña** y con acceso a los discos. Para evitarlo: cifrar con **BitLocker**, poner contraseña al firmware UEFI y desactivar el arranque desde USB/red si no se usa.
- En controladores de dominio, la contraseña del modo DSRM se cambia con `ntdsutil` → `set dsrm password` → `reset password on server null` → `q` → `q`.

Nota histórica: la opción "Última configuración buena conocida" de versiones antiguas ya no existe; su papel lo cubren WinRE y la Reparación automática.

### 7.2 Selección de kernels anteriores

En Windows **no se guardan varios kernels** para elegir en el menú. El kernel (`ntoskrnl.exe`) se actualiza dentro de las **actualizaciones acumulativas mensuales**. Para volver atrás se desinstala la actualización.

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| Ver kernels instalados | `Get-HotFix` (actualizaciones instaladas) o `DISM /Online /Get-Packages` |
| Volver al kernel anterior | Desinstalar la última actualización acumulativa |
| Hacerlo sin poder arrancar | Desde WinRE, con DISM sobre la instalación sin arrancar (`/Image:C:\`) |
| Volver a un controlador (módulo) anterior | Administrador de dispositivos → Propiedades → **Revertir al controlador anterior**, o `pnputil /delete-driver oemNN.inf /uninstall` (ver la lista con `pnputil /enum-drivers`) |

| Comando | Función |
|---|---|
| `wusa /uninstall /kb:5012345` | Desinstala una actualización por su número KB (no sirve para los paquetes combinados recientes) |
| `DISM /Online /Get-Packages` | Lista los paquetes instalados con su nombre exacto |
| `DISM /Online /Remove-Package /PackageName:Package_for_RollupFix~...` | Desinstala la actualización acumulativa (método válido para los paquetes combinados) |
| `DISM /Image:C:\ /Remove-Package /PackageName:...` | Lo mismo desde WinRE, sin arrancar el sistema |

**Copias completas del sistema** (lo más parecido a los Boot Environments de FreeBSD): **Copias de seguridad de Windows Server**.

| Comando | Función |
|---|---|
| `Install-WindowsFeature Windows-Server-Backup` | Instala la característica |
| `wbadmin start backup -backupTarget:E: -allCritical -quiet` | Copia todo lo necesario para recuperar el sistema completo (recuperación "bare metal") |
| `wbadmin start systemstatebackup -backupTarget:E:` | Copia solo el estado del sistema (registro, arranque, AD si es controlador de dominio...) |
| `wbadmin get versions` | Lista las copias disponibles |

Para restaurar una copia completa: WinRE → Solucionar problemas → **Recuperación de imagen del sistema**. Si el servidor es una máquina virtual de Hyper-V, además se pueden usar **puntos de control** (`Checkpoint-VM`) antes de actualizar.

### 7.3 Fallo del disco raíz (root drive failure)

| Comando Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `fsck /dev/sdaX` | `chkdsk C:` | Revisa el sistema de archivos NTFS (sin `/f` solo informa) |
| `fsck -y /dev/sdaX` | `chkdsk C: /f` | Revisa y repara. En el disco del sistema no puede hacerlo en marcha y lo programa para el siguiente arranque |
| — | `chkdsk C: /r` | Además busca sectores defectuosos (incluye `/f`) |
| — | `chkdsk C: /scan` | Revisión en línea, sin desmontar el volumen |
| — | `chkdsk C: /spotfix` | Repara muy rápido solo los errores ya detectados |
| — | `chkdsk D: /f /x` | Fuerza el desmontaje del volumen antes de reparar |
| (PowerShell) | `Repair-Volume -DriveLetter C -Scan` / `-SpotFix` / `-OfflineScanAndFix` | Lo mismo que `chkdsk` con cmdlets |
| Revisión automática al arrancar | `autochk.exe` (lo lanza Windows si el volumen está marcado como "sucio") | Equivale a la revisión de `fsck` al montar |
| — | `fsutil dirty query C:` | Indica si el volumen está marcado para revisión |
| — | `chkntfs /x D:` | Excluye un volumen de la revisión automática al arrancar |
| (ext4 frente a ZFS) | **ReFS** | Sistema de archivos que se autorrepara; no se repara con `chkdsk`. Para casos graves existe `refsutil` |
| `mount /dev/sdaX /media` | `diskpart` → `select volume N` → `assign letter=M` | Da una letra (o una carpeta) a un volumen para acceder a él |
| `umount /dev/sdaX` | `mountvol M: /p` | Desmonta el volumen y quita su letra |

Los discos se identifican por número (Disco 0, Disco 1...) y los volúmenes por letra. Para verlos: `Get-Disk`, `Get-Partition`, `Get-Volume` o `diskpart` → `list disk` / `list volume`.

**Reparar archivos del sistema dañados** (sin equivalente directo en el capítulo, pero muy usado en Windows):

| Comando | Función |
|---|---|
| `sfc /scannow` | Revisa y repara los archivos protegidos de Windows |
| `DISM /Online /Cleanup-Image /ScanHealth` | Comprueba si el almacén de componentes está dañado |
| `DISM /Online /Cleanup-Image /RestoreHealth` | Lo repara (descarga lo necesario de Windows Update, o de un origen indicado con `/Source`) |

**Disco de rescate (rescue disk):**
- El propio **medio de instalación** de Windows Server 2025 (ISO o USB) → **Reparar el equipo** → Solucionar problemas → Símbolo del sistema. Durante la instalación también se abre una consola con `Mayús+F10`.
- El **WinRE** instalado en la partición de recuperación.
- Una **WinPE** personalizada creada con el Windows ADK (apartado 3.4), a la que se pueden añadir herramientas propias.

---

## 8. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 1) | Equivalente en Windows Server 2025 |
|---|---|
| GRUB2 / `grub.cfg` | Windows Boot Manager / almacén **BCD** |
| `/etc/default/grub` | Entrada `{bootmgr}` de la BCD (`bcdedit /timeout`, `/default`) |
| `grub-mkconfig`, `update-grub` | No hace falta (`bcdedit` cambia la BCD directamente) |
| `grub-install` | `bcdboot` (y `bootrec` / `bootsect` desde WinRE) |
| `efibootmgr` | `bcdedit /enum firmware` y `{fwbootmgr}` |
| Parámetros del kernel en GRUB | `bcdedit /set {current} opción valor` |
| ISOLINUX / PXELINUX | `oscdimg` + WinPE / WDS |
| `dmesg`, logs en `/var/log` | Visor de eventos, `Get-WinEvent`, `C:\Windows\System32\winevt\Logs\`, `ntbtlog.txt` |
| SysV init / systemd | Administrador de control de servicios (`services.exe`) |
| `/etc/inittab`, `/etc/init.d/` | Registro: `HKLM\SYSTEM\CurrentControlSet\Services` |
| Runlevels 0-6 / targets | Arranque normal, Modo seguro, WinRE |
| Runlevel 3 frente a 5 | Server Core frente a Experiencia de escritorio (se elige al instalar) |
| `chkconfig`, `update-rc.d`, `systemctl enable` | `Set-Service -StartupType` o `sc.exe config ... start=` |
| `systemctl start/stop/status` | `Start-Service` / `Stop-Service` / `Get-Service` (o `sc.exe`) |
| `/etc/rc.local` | Tarea programada "al iniciar" o script de inicio de directiva de grupo |
| `runlevel` | `(Get-CimInstance Win32_ComputerSystem).BootupState` |
| `shutdown -h now` / `poweroff` | `shutdown /s /t 0` / `Stop-Computer` |
| `reboot`, `shutdown -r` | `shutdown /r /t 0` / `Restart-Computer` |
| `shutdown -c` | `shutdown /a` |
| Añadir `single` en GRUB | Modo seguro (`bcdedit /set {current} safeboot minimal` o Configuración de inicio) |
| Kernel anterior en el menú | Desinstalar la actualización (`DISM /Remove-Package`) o restaurar copia (`wbadmin`) |
| `fsck` | `chkdsk` / `Repair-Volume` (NTFS); ReFS se autorrepara |
| Live-CD de rescate | Medio de instalación ("Reparar el equipo"), WinRE o WinPE |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 1 ("Starting a System") del libro LPIC-2. Fuentes: documentación oficial de Microsoft Learn (Windows Server, BCDEdit, BCDBoot, Windows RE, Windows PE, sc.exe, shutdown, chkdsk, DISM, Windows Deployment Services) y guía de Microsoft sobre el endurecimiento de WDS (CVE-2026-0386).*
