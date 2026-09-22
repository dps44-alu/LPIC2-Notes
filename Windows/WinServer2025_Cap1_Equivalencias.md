# LPIC-2 · Capítulo 1: Starting a System
### Equivalencias en Windows Server 2025

Este documento recoge, para cada concepto visto en el Capítulo 1 del libro LPIC-2 (arranque, bootloaders, inicialización del sistema y recuperación), cuál es su equivalente o la herramienta más parecida en **Windows Server 2025**. No todos los conceptos de Linux tienen una traducción exacta; en esos casos se indica la diferencia conceptual.

---

## 1. Arranque del sistema (firmware)

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| BIOS / UEFI | Igual: Windows Server 2025 arranca desde firmware **BIOS (modo Legacy/CSM)** o, lo habitual hoy, **UEFI** con Secure Boot |
| POST | Igual, es una fase del firmware común a cualquier sistema operativo |

No hay diferencias relevantes en esta fase: el firmware no depende del sistema operativo que se vaya a cargar.

---

## 2. El gestor de arranque: Windows Boot Manager

En Linux esta función la cumple GRUB/GRUB2/LILO. En Windows la cumple el **Windows Boot Manager** (`bootmgr` en BIOS/MBR, o `\EFI\Microsoft\Boot\bootmgfw.efi` en UEFI/GPT), apoyado en un almacén de configuración llamado **BCD (Boot Configuration Data)**, que sustituye al antiguo `boot.ini` de versiones anteriores a Windows Vista.

| Elemento GRUB2 | Elemento equivalente en Windows |
|---|---|
| `/boot/grub/grub.cfg` | Almacén BCD: `\Boot\BCD` (BIOS/MBR) o `\EFI\Microsoft\Boot\BCD` (UEFI/GPT) |
| `/etc/default/grub` | No existe fichero de texto editable a mano; el BCD es una base de datos binaria que solo se modifica con herramientas |
| `grub-mkconfig` / `update-grub` | No aplica igual — el BCD se modifica directamente con `bcdedit`, no se "regenera" desde plantillas |
| Menú de arranque (`menuentry`) | Menú del Windows Boot Manager, configurable con `bcdedit` |

### 2.1 Herramienta `bcdedit`

Comando de línea de comandos (requiere privilegios de administrador) para gestionar el almacén BCD.

| Comando | Función |
|---|---|
| `bcdedit /enum` | Lista las entradas de arranque actuales |
| `bcdedit /enum all` | Lista todas las entradas, incluidas las del gestor de arranque y el firmware |
| `bcdedit /export C:\BCD_Backup.bcd` | Exporta (copia de seguridad) del almacén BCD completo |
| `bcdedit /import archivo.bcd` | Restaura un almacén BCD desde una copia exportada |
| `bcdedit /create` | Crea una nueva entrada de arranque |
| `bcdedit /delete {id}` | Elimina una entrada de arranque |
| `bcdedit /set {id} elemento valor` | Modifica un elemento de una entrada (ej. `description`, `timeout`, `default`) |
| `bcdedit /deletevalue {id} elemento` | Elimina un valor concreto de una entrada |
| `bcdedit /default {id}` | Define la entrada de arranque por defecto |
| `bcdedit /timeout segundos` | Define el tiempo de espera del menú de arranque |
| `bcdedit /set {current} bootstatuspolicy ignoreallfailures` | Evita que el sistema entre en modo de recuperación automática ante fallos de arranque (útil para romper "bucles" de reparación) |

Alternativa recomendada para cambios más seguros/gráficos: **`msconfig.exe`** (System Configuration Utility), pestaña "Arranque" (Boot), que permite marcar arranque seguro, depuración, sin GUI, etc. sin tocar el BCD directamente.

### 2.2 Reparar el gestor de arranque

| Herramienta | Función |
|---|---|
| `bootrec /fixmbr` | Repara el registro de arranque maestro (MBR) |
| `bootrec /fixboot` | Escribe un nuevo sector de arranque en la partición del sistema |
| `bootrec /scanos` | Busca instalaciones de Windows en el disco |
| `bootrec /rebuildbcd` | Reconstruye el almacén BCD a partir de las instalaciones encontradas |
| `bcdboot C:\Windows` | Reinstala los archivos de arranque a partir de una instalación de Windows existente (equivalente aproximado a reinstalar GRUB en el MBR/ESP) |

Estas herramientas se ejecutan normalmente desde el **entorno de recuperación de Windows (WinRE)**, de forma similar a como en Linux se repara GRUB arrancando desde un live-CD/USB de rescate.

### 2.3 Arranque seguro (Secure Boot) y cifrado

| Concepto Linux | Equivalente Windows |
|---|---|
| Secure Boot (UEFI) — desactivarlo, usar clave propia, o shim firmado | Igual concepto: Secure Boot se activa/desactiva desde el firmware UEFI; puede requerir suspenderse temporalmente con `bcdedit` o desde el firmware antes de ciertos cambios de arranque |
| Cifrado de disco (LUKS/dm-crypt, ver Capítulo 4) | **BitLocker**: puede requerir suspenderse antes de tocar el BCD (`manage-bde -protectors -disable C:`) |

---

## 3. Inicialización del sistema: no hay runlevels ni SysV init

Windows **no tiene un concepto directo de runlevels ni de SysV init/systemd**. La arquitectura de Windows separa esto en dos mecanismos distintos:

- El **arranque del sistema operativo en sí** (kernel `ntoskrnl.exe`, controladores) no tiene "niveles" seleccionables como en Linux; siempre arranca hacia el mismo estado base, y las variantes (modo seguro, etc.) se seleccionan como *opciones de arranque*, no como un nivel de ejecución con servicios distintos.
- Los **programas que arrancan con el sistema** se gestionan mediante el **Service Control Manager (SCM)** y las aplicaciones de inicio de sesión (Startup Apps), no mediante scripts por "runlevel".

### 3.1 El Service Control Manager (equivalente a `systemd`/SysV init)

| Concepto Linux | Equivalente en Windows |
|---|---|
| `systemctl` / unit files (`.service`) | Service Control Manager (SCM); cada servicio es una entrada del Registro bajo `HKLM\SYSTEM\CurrentControlSet\Services\` |
| `/etc/init.d/`, `/lib/systemd/system/` | No hay ficheros de texto; la configuración de cada servicio vive en el Registro de Windows |
| `systemctl start/stop/restart nombre` | `sc.exe start/stop nombre` o los cmdlets de PowerShell `Start-Service`, `Stop-Service`, `Restart-Service` |
| `systemctl enable/disable nombre` | `sc.exe config nombre start= auto\|demand\|disabled` o `Set-Service -Name nombre -StartupType Automatic\|Manual\|Disabled` |
| `systemctl status nombre` | `sc.exe query nombre` o `Get-Service -Name nombre` |
| `systemctl list-units` | `Get-Service` (PowerShell) o el panel gráfico **Services** (`services.msc`) |

### 3.2 Herramientas de gestión de servicios

| Herramienta | Tipo | Función |
|---|---|---|
| `services.msc` | Consola gráfica (MMC) | Ver, arrancar, parar, y configurar el tipo de inicio de cada servicio |
| `sc.exe` | Línea de comandos clásica | Crear, consultar, configurar y eliminar servicios |
| `Get-Service`, `Start-Service`, `Stop-Service`, `Restart-Service`, `Set-Service` | PowerShell | Gestión moderna de servicios, con nombres más legibles |
| Administrador de tareas → pestaña "Inicio" (Startup apps) | Gráfico | Gestiona qué aplicaciones (no servicios del sistema) arrancan al iniciar sesión, similar en espíritu a `systemctl enable` pero para programas de usuario, no servicios del sistema |
| `msconfig.exe` | Gráfico | Pestaña "Servicios" para activar/desactivar servicios de arranque, y pestaña "Inicio" (redirige al Administrador de tareas en versiones modernas) |

**Tipos de inicio de un servicio (equivalente aproximado a habilitar/deshabilitar en un runlevel):**

| Tipo de inicio | Equivalente conceptual en systemd |
|---|---|
| `Automatic` (Automático) | `systemctl enable` |
| `Automatic (Delayed Start)` | Similar, pero con arranque retrasado tras el resto de servicios críticos |
| `Manual` | Servicio instalado pero no habilitado (`systemctl disable`, pero arrancable a demanda) |
| `Disabled` | `systemctl mask` (no se puede arrancar ni siquiera a mano hasta reactivarlo) |

---

## 4. Comprobación de mensajes de arranque

| Concepto Linux | Equivalente en Windows |
|---|---|
| `dmesg`, kernel ring buffer | **Visor de eventos** (`eventvwr.msc`), especialmente el registro **Sistema** (System) y **Aplicación** |
| `/var/log/` | Registros de eventos de Windows (Event Log), consultables también por PowerShell |
| Consulta por línea de comandos | `Get-EventLog -LogName System -Newest 50` o el más moderno `Get-WinEvent -LogName System -MaxEvents 50` |
| `wevtutil` | Herramienta de línea de comandos clásica equivalente para consultar/exportar registros de eventos |

---

## 5. Apagado y reinicio del sistema

| Comando Linux | Comando equivalente en Windows |
|---|---|
| `shutdown` | `shutdown.exe` (sintaxis distinta: `/s` apagar, `/r` reiniciar, `/t segundos` retardo, `/f` forzar cierre de aplicaciones) |
| `halt` / `poweroff` | `shutdown /s /t 0` |
| `reboot` | `shutdown /r /t 0` |
| `shutdown` con aviso a usuarios | `shutdown /r /t 60 /c "Mensaje de aviso"` (con `/c` se añade un comentario/mensaje) |
| `init N` / `telinit N` (cambiar runlevel) | No hay equivalente: Windows no tiene "niveles" de ejecución intercambiables en caliente |
| Reinicio hacia opciones avanzadas (equivalente a runlevel de rescate) | `shutdown /r /o /t 0` (reinicia directamente al menú de **opciones de inicio avanzadas** de WinRE) |

PowerShell también ofrece cmdlets equivalentes: `Restart-Computer`, `Stop-Computer`.

---

## 6. Recuperación del sistema

### 6.1 Modo seguro (Safe Mode) — equivalente aproximado al modo mono-usuario

| Concepto Linux | Equivalente en Windows |
|---|---|
| Editar la línea `linux`/`linux16` en GRUB y añadir `single` | Arrancar en **Modo seguro** (Safe Mode) desde el menú de **Opciones de inicio avanzadas** (Advanced Startup Options), o forzarlo con `bcdedit /set {current} safeboot minimal` |
| Runlevel 1 (mono-usuario, solo root) | Modo seguro: carga solo controladores y servicios esenciales, con acceso solo de administrador |
| Variante con red | `bcdedit /set {current} safeboot network` (equivalente a un modo seguro "con red") |
| Quitar el modo tras terminar | `bcdedit /deletevalue {current} safeboot` |

Acceso al menú de recuperación:
- Desde Windows en marcha: **Configuración → Recuperación → Reinicio avanzado**, o `shutdown /r /o /t 0`.
- Si el sistema no arranca: tras varios fallos consecutivos, Windows entra automáticamente en **WinRE** (Windows Recovery Environment).

### 6.2 WinRE (Windows Recovery Environment) — equivalente al entorno de rescate/live-CD

| Concepto Linux | Equivalente en Windows |
|---|---|
| Arrancar desde un disco de rescate (rescue disk) para reparar el sistema | **WinRE**, entorno mínimo incluido en la propia instalación (partición de recuperación) o arrancable desde el medio de instalación/USB |
| `fsck` (comprobar/reparar sistema de archivos) | `chkdsk C: /f /r` (comprueba y repara el sistema de archivos; `/r` localiza sectores dañados) |
| Comprobar la integridad de ficheros del sistema | `sfc /scannow` (System File Checker) |
| Reparar la imagen del sistema (equivalente a reinstalar paquetes corruptos) | `DISM /Online /Cleanup-Image /RestoreHealth` (o `/Offline` apuntando a una instalación montada) |
| Reparar el sistema de ficheros desde fuera (SO offline) | `sfc /scannow /offbootdir=D:\ /offwindir=D:\Windows` (ejecutado desde WinRE contra una instalación "fría") |
| `mount` / `umount` de la partición a reparar | En WinRE, `diskpart` para listar/asignar letras de unidad a las particiones antes de repararlas |

### 6.3 Restauración del sistema

| Concepto Linux | Equivalente en Windows |
|---|---|
| Mantener el kernel anterior instalado por si el nuevo falla | **Puntos de restauración del sistema** (System Restore): revierten cambios de configuración, controladores y del Registro a un estado anterior, sin afectar a los documentos del usuario |
| Comando relacionado | `rstrui.exe` (asistente gráfico) o el cmdlet PowerShell `Restore-Computer` |
| Backups completos del sistema (ver también Capítulo 2) | **Windows Server Backup** (`wbadmin`), que permite recuperación completa del sistema desde WinRE |

---

## 7. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 1) | Equivalente en Windows Server 2025 |
|---|---|
| GRUB2 / `grub.cfg` | Windows Boot Manager / almacén BCD |
| `grub-mkconfig`, `update-grub` | `bcdedit` (edición directa de entradas) |
| SysV init / systemd | Service Control Manager (`services.msc`, `sc.exe`, cmdlets `*-Service`) |
| Runlevels | No existe equivalente directo; lo más parecido son los *tipos de inicio* de cada servicio y el *Modo seguro* como estado reducido |
| `dmesg`, logs de arranque | Visor de eventos (`eventvwr.msc`, `Get-WinEvent`) |
| `shutdown`, `halt`, `reboot`, `init N` | `shutdown.exe`, `Restart-Computer`, `Stop-Computer` |
| Modo mono-usuario / rescate | Modo seguro (Safe Mode) y WinRE |
| `fsck` | `chkdsk` |
| Live-CD de rescate | WinRE (partición de recuperación o medio de instalación) |
| Kernel anterior como respaldo | Puntos de restauración del sistema (System Restore) |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 1 ("Starting a System") del libro LPIC-2. Fuentes: conocimiento general de administración de Windows Server más documentación oficial de Microsoft Learn (bcdedit, BCD) consultada para verificar la vigencia de los comandos en Windows Server 2025.*
