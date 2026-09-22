# LPIC-2 · Capítulo 2: Maintaining the System
### Equivalencias en Windows Server 2025

---

## 1. Comunicación con los usuarios del sistema

### 1.1 Mensajería "fluida" (en tiempo real)

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| `write`, `wall` | **`msg.exe`**: envía un mensaje emergente a un usuario, sesión o a todos (`*`) en un servidor de Escritorio remoto (Remote Desktop Session Host) |
| `mesg y`/`n` | No existe un control equivalente por terminal; `msg.exe` requiere el permiso especial "Message" en la sesión de destino |
| `who -T` | `query user` o `qwinsta` (lista las sesiones activas en el servidor, útil antes de enviar un mensaje) |
| `notify-send` (notificación de escritorio) | No hay equivalente nativo de línea de comandos tan directo; en un entorno gráfico se usan notificaciones de la **Bandeja del sistema/Centro de actividades**, o el módulo de PowerShell no oficial `BurntToast` para notificaciones tipo "toast" |
| `/sbin/shutdown` con aviso | `shutdown.exe /r /t segundos /c "mensaje"` (el parámetro `/c` añade el comentario que verán los usuarios antes del reinicio/apagado) |

**Sintaxis de `msg.exe`:**
```
msg {usuario | nombre_sesión | ID_sesión | @fichero | *} [/server:servidor] [/time:segundos] [/v] [/w] [mensaje]
```

| Opción | Función |
|---|---|
| `*` | Envía el mensaje a todas las sesiones activas del servidor |
| `@fichero` | Lee la lista de destinatarios desde un fichero de texto |
| `/server:nombre` | Servidor remoto al que dirigir el mensaje |
| `/time:segundos` | Tiempo de espera para que el destinatario confirme la lectura |
| `/w` | Espera a que el usuario cierre el mensaje antes de devolver el control |

Nota importante: `msg.exe` solo está disponible en las ediciones Pro/Enterprise/Server (no en ediciones Home), y solo funciona sobre sesiones de **Remote Desktop Session Host**; no es un equivalente universal de `wall` para todas las sesiones interactivas locales.

### 1.2 Mensajería "estática" (mensajes de bienvenida / login)

| Fichero/concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| `/etc/issue` (mensaje antes del login local) | Directiva de directiva de grupo/seguridad **"Inicio de sesión interactivo: Título del mensaje para los usuarios que intentan iniciar sesión"** y **"...: Texto del mensaje..."**, en Directiva de grupo local (`gpedit.msc` → Configuración del equipo → Configuración de Windows → Configuración de seguridad → Directivas locales → Opciones de seguridad) |
| `/etc/issue.net` (login remoto) | Mismo mecanismo de directiva de seguridad, ya que en Windows el cuadro de inicio de sesión es el mismo tanto en local como en Escritorio remoto (RDP) |
| `/etc/motd` (mensaje tras el login) | No existe un equivalente exacto de "mensaje del día" tras autenticarse; lo más parecido es un script de inicio de sesión (**logon script**, vía GPO: Configuración de usuario → Windows Settings → Scripts → Logon) que muestre un mensaje, o el banner de aviso legal anterior al login (que en la práctica cumple la misma función disuasoria) |

**Herramientas para configurar el banner de aviso legal:**

| Herramienta | Función |
|---|---|
| `gpedit.msc` (Directiva de grupo local) | Configuración gráfica del banner, en un servidor independiente |
| `Group Policy Management` (`gpmc.msc`) | Configuración centralizada del banner para todo un dominio Active Directory (GPO aplicada a una OU) |
| Registro directamente | `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` → valores `LegalNoticeCaption` y `LegalNoticeText` |
| PowerShell | `Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name LegalNoticeText -Value "Aviso..."` |

---

## 2. Copias de seguridad (backup)

### 2.1 Herramienta principal: Windows Server Backup

Windows Server Backup es la utilidad integrada equivalente conceptual al conjunto `tar`/`dump`/soluciones tipo Amanda-Bacula del mundo Linux. Se basa en **VSS (Volume Shadow Copy Service)**, el equivalente funcional a los snapshots (ver Capítulo 2, tipo "snapshot").

| Elemento | Equivalente Linux |
|---|---|
| VSS (Volume Shadow Copy Service) | Mecanismo de snapshot, similar en concepto a LVM snapshot o Btrfs snapshot |
| Backup completo con Windows Server Backup | Backup `full` de `tar`/`dump` |
| Backup incremental automático (Windows Server Backup solo guarda cambios tras el primero) | Backup `incremental` |

### 2.2 Utilidad de línea de comandos: `wbadmin`

Requiere privilegios de administrador o pertenecer al grupo **Backup Operators**, y ejecutarse desde una consola elevada.

| Comando | Función |
|---|---|
| `wbadmin start backup -backupTarget:E: -include:C:,D: -allCritical -quiet` | Backup completo de los volúmenes indicados a la unidad E: |
| `wbadmin start backup -backupTarget:D: -systemstate` | Incluye el **estado del sistema** (Registro, Active Directory si aplica, etc.), similar a un backup de configuración crítica en Linux |
| `wbadmin start systemstatebackup -backupTarget:F:` | Backup exclusivo del estado del sistema |
| `wbadmin enable backup -addtarget:E: -schedule:21:00 -include:C:` | Programa un backup diario automático (equivalente a una tarea `cron`/`systemd timer` que ejecuta `tar`) |
| `wbadmin get versions` | Lista las copias de seguridad disponibles (equivalente a `tar -t` para ver contenido, pero a nivel de catálogo de backups) |
| `wbadmin start recovery -version:fecha -itemtype:File -items:ruta -recoveryTarget:ruta` | Restaura ficheros concretos desde una copia |
| `wbadmin start sysrecovery` | Recuperación completa del sistema (equivalente a restaurar un backup `full` sobre un sistema nuevo) |

Ubicación típica del backup en disco: carpeta `WindowsImageBackup` en la unidad de destino.

### 2.3 Herramientas de copia de archivos (equivalente a `rsync`/`cpio`)

| Herramienta Linux | Equivalente en Windows |
|---|---|
| `rsync` | **`robocopy`** (Robust File Copy), incluido de serie en Windows Server desde hace años; soporta copia incremental (`/MIR` para espejo), reintentos, multi-hilo (`/MT:n`), preservación de ACLs y atributos (`/COPYALL`) |
| `rsync -av origen destino` | `robocopy origen destino /E /COPYALL /R:3 /W:5` |
| Sincronización tipo espejo (`rsync --delete`) | `robocopy origen destino /MIR` (¡cuidado!: borra en destino lo que no exista en origen, igual que `--delete`) |
| `tar` (crear/extraer archivos) | **`tar.exe`** (en realidad `bsdtar`, basado en libarchive), incluido de forma nativa en Windows desde 2018 (Windows 10 Build 17063 en adelante, y por tanto en Windows Server 2025); soporta `.tar`, `.tar.gz`, `.zip` y varios formatos más |
| `tar -zcvf archivo.tar.gz carpeta` | `tar -a -c -f archivo.tar.gz carpeta` (la opción `-a` detecta la compresión por la extensión) |
| `tar -zxvf archivo.tar.gz` | `tar -x -f archivo.tar.gz` |
| `dd` (copia a bajo nivel) | No hay equivalente directo de propósito general; para clonado de disco se usa **DiskPart** (particiones) o herramientas de imagen como **`wbadmin`**/**DISM** para capturar imágenes `.wim`/`.vhdx` |
| `mt` (control de unidades de cinta) | Las cintas magnéticas están en desuso en Windows moderno; Windows Server Backup y soluciones de terceros (ver más abajo) gestionan la cinta si el hardware está presente vía LTFS/gestores propios del fabricante |

### 2.4 Soluciones de backup empresariales (equivalente a Amanda/Bacula/BackupPC)

| Solución | Comparable a |
|---|---|
| **Microsoft System Center Data Protection Manager (DPM)** | Solución de backup centralizada con consola de gestión, similar en alcance a Bacula/Bareos |
| **Azure Backup** / **Azure Backup Server** | Backup en la nube, sin equivalente directo en el libro pero es la evolución natural del backup remoto |
| Soluciones de terceros (Veeam, Commvault, Veritas NetBackup) | Ampliamente usadas en entornos Windows Server como sustituto de Windows Server Backup para necesidades avanzadas, equivalente en función a Amanda/Bacula/Bareos |

---

## 3. Instalación de programas desde código fuente

En Windows Server no existe un flujo equivalente a `./configure && make && make install`, porque el software para Windows se distribuye habitualmente ya compilado (instaladores `.msi`/`.exe`, o paquetes `winget`/`Chocolatey`). Cuando sí es necesario compilar software desde código fuente (típicamente desarrollo o herramientas open-source portadas a Windows), el equivalente conceptual es:

| Paso Linux | Equivalente en Windows |
|---|---|
| Instalar herramientas de compilación (`build-essential`/"Development Tools") | Instalar **Visual Studio Build Tools** (compilador MSVC, enlazador, SDK de Windows) o el conjunto de herramientas de **MinGW-w64**/**Cygwin** si se busca un entorno más similar a GCC |
| `./configure` | **`cmake`** (generador de proyectos multiplataforma, muy extendido también en Windows) o los scripts de configuración propios del proyecto |
| `make` | **`msbuild`** (compilador de proyectos de Visual Studio) o **`nmake`** (make clásico de Microsoft) |
| `make install` | No hay un paso equivalente estandarizado; normalmente el propio sistema de compilación copia los binarios a una carpeta de salida, o se empaqueta en un instalador |
| Gestor de paquetes para dependencias (`apt`/`yum`) | **`winget`** (gestor de paquetes oficial de Microsoft, incluido en Windows Server 2025) o **Chocolatey**/**vcpkg** (este último específico para dependencias de compilación en C/C++) |

**Comandos de `winget` (equivalente más cercano a `apt-get`/`yum` para instalar software ya compilado):**

| Comando | Función |
|---|---|
| `winget search nombre` | Busca un paquete |
| `winget install nombre` | Instala un paquete |
| `winget upgrade --all` | Actualiza todo el software instalado vía winget |
| `winget list` | Lista el software instalado gestionado por winget |

---

## 4. Medición y gestión del uso de recursos

### 4.1 Herramientas gráficas

| Herramienta Linux | Equivalente en Windows Server 2025 |
|---|---|
| `top`/`htop` | **Administrador de tareas** (Task Manager, `taskmgr.exe`), pestañas "Procesos" y "Rendimiento" |
| Vista detallada de recursos por proceso (CPU, disco, red, memoria en tiempo real) | **Monitor de recursos** (Resource Monitor, `resmon.exe`) |
| `sar` (recolector histórico) | **Monitor de rendimiento** (Performance Monitor, `perfmon.exe`), con **Conjuntos de recopiladores de datos** (Data Collector Sets) para registrar contadores a lo largo del tiempo, igual que hace `sar`/`sadc` con `/var/log/sa/` |
| Planificación de capacidad con gráficas | Perfmon + informes, o **Windows Admin Center** con sus paneles de rendimiento |

### 4.2 Herramientas de línea de comandos / PowerShell

| Comando Linux | Equivalente en Windows |
|---|---|
| `free` (memoria) | `Get-CimInstance Win32_OperatingSystem \| Select FreePhysicalMemory,TotalVisibleMemorySize` o `systeminfo` |
| `vmstat` | `Get-Counter '\Memory\*'` o `typeperf "\Memory\Available MBytes"` |
| `uptime` | `(Get-CimInstance Win32_OperatingSystem).LastBootUpTime` o `net statistics server` |
| `mpstat`/uso de CPU | `Get-Counter '\Processor(_Total)\% Processor Time'` |
| `iostat` (I/O de disco) | `Get-Counter '\PhysicalDisk(*)\*'` o `typeperf "\PhysicalDisk(_Total)\Disk Reads/sec"` |
| `ps`/`pstree` | `Get-Process`, o el clásico `tasklist` (cmd.exe) |
| `pmap` (mapa de memoria de un proceso) | `Get-Process -Id PID \| Select *` (menos detallado); para análisis avanzado, **VMMap** (Sysinternals) |
| `lsof` (ficheros/conexiones abiertas por proceso) | `Get-NetTCPConnection -OwningProcess PID` (conexiones) y **Process Explorer** (Sysinternals) para ficheros abiertos |
| `ip -s link` / `netstat` | `Get-NetAdapterStatistics`, `netstat -an` (sigue existiendo en Windows), o `Get-NetTCPConnection`/`Get-NetUDPEndpoint` |
| `ss` | No hay equivalente exacto; `Get-NetTCPConnection` en PowerShell cubre un uso similar |
| `watch comando` | No existe de forma nativa en `cmd.exe`; en PowerShell: `while ($true) { comando; Start-Sleep -Seconds 5; Clear-Host }`, o el cmdlet más directo (en versiones recientes) no tiene equivalente 1:1, por lo que se recurre a bucles como el anterior |
| `tcpdump` | **`netsh trace start`**/`netsh trace stop` (captura nativa), o **Wireshark**/**Microsoft Network Monitor** para análisis gráfico |

### 4.3 `perfmon` y los Conjuntos de recopiladores de datos (equivalente a `sar`)

| Elemento | Función |
|---|---|
| `perfmon.exe` | Consola gráfica de monitorización con **contadores de rendimiento** (Performance Counters), organizados por objeto (Processor, Memory, PhysicalDisk, Network Interface, etc.) |
| Data Collector Sets | Conjuntos programables de contadores que se registran periódicamente a fichero (`.blg`), igual que `sadc` guarda datos en `/var/log/sa/` para que `sar` los consulte después |
| `logman` | Herramienta de línea de comandos para crear y gestionar Data Collector Sets sin abrir la consola gráfica |
| `typeperf` | Herramienta de línea de comandos que muestra contadores de rendimiento en tiempo real o los registra a un fichero CSV — el más parecido en espíritu a ejecutar `sar intervalo cuenta` |
| `Get-Counter` (PowerShell) | Cmdlet moderno equivalente a `typeperf`, permite consultar y registrar contadores desde scripts |

Ejemplo con `typeperf` (equivalente a `sar 2 20`):
```
typeperf "\Processor(_Total)\% Processor Time" -si 2 -sc 20
```

### 4.4 Planificación de capacidad y monitorización continua

| Solución Linux | Equivalente en Windows Server |
|---|---|
| Nagios, Icinga | **System Center Operations Manager (SCOM)** — monitorización de infraestructura a gran escala con alertas |
| Cacti, MRTG, collectd | **Performance Monitor** + Data Collector Sets, o **Windows Admin Center** para paneles visuales integrados |
| RRDTool (base de datos circular) | No hay un equivalente nativo directo; los ficheros `.blg` de Performance Monitor cumplen una función parecida de almacenamiento histórico de contadores |

### 4.5 Resolución de problemas de recursos

| Recurso | Herramientas en Windows |
|---|---|
| Memoria | Administrador de tareas, Monitor de recursos, `Get-Counter '\Memory\*'` |
| Procesos | Administrador de tareas, `Get-Process`, **Process Explorer** (Sysinternals) para árbol de procesos y detalle de hilos |
| CPU | Administrador de tareas (pestaña Rendimiento), `Get-CimInstance Win32_Processor` |
| I/O de disco | Monitor de recursos (pestaña Disco), `Get-Counter '\PhysicalDisk(*)\*'` |
| Red | Monitor de recursos (pestaña Red), `Get-NetAdapterStatistics`, Wireshark para captura profunda |

---

## 5. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 2) | Equivalente en Windows Server 2025 |
|---|---|
| `write`, `wall` | `msg.exe` |
| `/etc/issue`, `/etc/motd` | Banner de aviso legal (GPO/Registro: `LegalNoticeCaption`/`LegalNoticeText`) |
| `tar`, `rsync` | `tar.exe` (bsdtar nativo), `robocopy` |
| Amanda, Bacula, BackupPC | Windows Server Backup / `wbadmin`, System Center DPM, o soluciones de terceros (Veeam, etc.) |
| `./configure && make && make install` | `cmake`/`msbuild`/`nmake` con Visual Studio Build Tools; `winget` para software ya compilado |
| `top`, `htop` | Administrador de tareas |
| `sar`, `vmstat`, `iostat` | Monitor de rendimiento (`perfmon`), `typeperf`, `Get-Counter` |
| `lsof`, `pmap` | Process Explorer / VMMap (Sysinternals) |
| `tcpdump` | `netsh trace`, Wireshark |
| Nagios, Cacti | System Center Operations Manager (SCOM) |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 2 ("Maintaining the System") del libro LPIC-2. Fuentes: conocimiento general de administración de Windows Server, verificado con documentación oficial de Microsoft Learn (`wbadmin`, `robocopy`, `msg`) y fuentes técnicas sobre la disponibilidad nativa de `tar.exe`/`curl.exe` en Windows.*
