# LPIC-2 · Capítulo 2: Maintaining the System
### Equivalencias en Windows Server 2025

Este documento recoge, para cada concepto visto en el Capítulo 2 del libro LPIC-2 (comunicación con los usuarios, copias de seguridad, compilación desde código fuente y monitorización de recursos), cuál es su equivalente en **Windows Server 2025**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo: en Windows los usuarios no trabajan en "terminales" sino en **sesiones** (la de la consola física y las de **Escritorio remoto**), así que los mensajes se envían a sesiones. Las copias de seguridad se basan en las **instantáneas de volumen (VSS)**, que permiten copiar incluso archivos abiertos. El software casi nunca se compila: se instala ya compilado (MSI, EXE, `winget`) o se añade como **rol o característica** del sistema. Y la monitorización gira alrededor de los **contadores de rendimiento**, que se consultan con herramientas gráficas (`perfmon`, `resmon`, Administrador de tareas) o con PowerShell (`Get-Counter`).

Todos los comandos se ejecutan en una consola **como Administrador**. Recordatorio: Windows Server 2025 trae Windows PowerShell 5.1; algunas herramientas gráficas (`resmon`, `perfmon` completo, `winget`) solo están en la instalación con **Experiencia de escritorio**, no en Server Core.

---

## 1. Comunicación con los usuarios del sistema

### 1.1 Mensajería "fluida" (en tiempo real)

Para saber a quién se envía un mensaje, primero hay que ver las sesiones abiertas:

| Comando | Función |
|---|---|
| `quser` (o `query user`) | Lista los usuarios con sesión abierta: nombre, ID de sesión, estado (Activo / Desconectado) y hora de inicio |
| `qwinsta` (o `query session`) | Lista todas las sesiones, incluidas las que no tienen usuario |
| `logoff 3` | Cierra la sesión con ID 3 |
| `tsdiscon 3` | Desconecta la sesión 3 sin cerrarla (los programas siguen abiertos) |

| Comando Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `write usuario terminal` | `msg usuario "texto"` o `msg 3 "texto"` (por ID de sesión) | Mensaje privado a un usuario conectado. Aparece como una ventana emergente |
| `wall mensaje` | `msg * "texto"` | Mensaje a todas las sesiones |
| — | `msg * /server:SRV01 "texto"` | Mensaje a las sesiones de otro servidor |
| — | `msg * /time:120 "texto"` | La ventana se cierra sola a los 120 segundos |
| `mesg y` / `mesg n` | No existe | Un usuario no puede bloquear los mensajes de `msg` |
| `who -T` | `quser` | No hay permisos de escritura por terminal; solo se ve quién está conectado |
| `notify-send "Título" "Mensaje"` | `msg` (ventana emergente). No hay un comando integrado para notificaciones tipo "globo" | Aviso gráfico en el escritorio |

Para que `msg` funcione contra **otro** servidor (`/server:`), en el servidor de destino debe existir el valor de registro `AllowRemoteRPC = 1` en `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server`.

En una granja de **Servicios de Escritorio remoto** (RDS) se puede usar también `Send-RDUserMessage` (módulo `RemoteDesktop`), que envía el mensaje a través del agente de conexión.

**`shutdown` con aviso** (visto en el Capítulo 1): `shutdown /r /t 900 /c "Reinicio por mantenimiento en 15 minutos"`. Los usuarios conectados ven el mensaje y la cuenta atrás.

| Opción Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `-k` (avisar y bloquear nuevos logins, sin apagar) | `msg * "aviso"` + `change logon /disable` | Avisar y bloquear nuevos inicios de sesión de Escritorio remoto, sin apagar |
| `-c` (cancelar) | `shutdown /a` | Cancela el apagado programado |
| `-H` (halt) | No hay "halt" separado; `shutdown /s` apaga | — |
| `-P` (power off) | `shutdown /s` (o `/p` para hacerlo al instante) | Apaga |
| `-r` | `shutdown /r` | Reinicia |
| `--no-wall` | No existe | — |

**Comando `change logon`** (control de los inicios de sesión de Escritorio remoto; equivale a la parte "bloquear logins" de `shutdown -k` y al archivo `/etc/nologin`):

| Comando | Función |
|---|---|
| `change logon /query` | Muestra el estado actual |
| `change logon /disable` | Impide nuevos inicios de sesión remotos (la consola física sigue funcionando) |
| `change logon /drain` | Impide nuevos inicios de sesión, pero deja reconectar a los que ya tenían una sesión |
| `change logon /drainuntilrestart` | Igual que `/drain` hasta el próximo reinicio |
| `change logon /enable` | Vuelve a permitir los inicios de sesión |

### 1.2 Mensajería "estática" (mensajes de bienvenida / login)

| Archivo Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `/etc/issue` | **Aviso legal de inicio de sesión** (directiva de seguridad) | Mensaje que se muestra **antes** de iniciar sesión, en la consola y por Escritorio remoto. El usuario debe pulsar Aceptar |
| `/etc/issue.net` + `Banner` en `sshd_config` | `Banner` en `C:\ProgramData\ssh\sshd_config` | Mensaje antes del login por SSH |
| `/etc/motd` | No existe. Se imita con el **perfil de PowerShell** o con un **script de inicio de sesión** | Mensaje tras iniciar sesión |

**Aviso legal** (equivalente a `/etc/issue`). Se configura en `secpol.msc` → Directivas locales → Opciones de seguridad:
- "Inicio de sesión interactivo: título del mensaje para los usuarios que intentan iniciar sesión".
- "Inicio de sesión interactivo: texto del mensaje para los usuarios que intentan iniciar sesión".

Se guarda en el Registro, en `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`, valores `LegalNoticeCaption` (título) y `LegalNoticeText` (texto). En un dominio se configura igual, pero desde una directiva de grupo (`gpmc.msc`).

**SSH**: Windows Server 2025 incluye el servidor OpenSSH. Se activa con `Start-Service sshd` y `Set-Service sshd -StartupType Automatic` (o desde el Administrador del servidor). Su configuración está en `C:\ProgramData\ssh\sshd_config`, y el aviso se añade con una línea como:
```
Banner C:\ProgramData\ssh\banner.txt
```
Después: `Restart-Service sshd`.

**Mensaje tras iniciar sesión** (sustituto de `/etc/motd`):
- En consolas de PowerShell (también por SSH si la shell es PowerShell): añadir un `Write-Host "texto"` al perfil común de todos los usuarios, `$PSHOME\Profile.ps1` (normalmente `C:\Windows\System32\WindowsPowerShell\v1.0\Profile.ps1`).
- En el escritorio: un script de inicio de sesión por directiva de grupo (Configuración de usuario → Configuración de Windows → Scripts → Inicio de sesión) que muestre el mensaje.

---

## 2. Copias de seguridad (backup)

### 2.1 y 2.2 Conceptos y tipos de backup

Los conceptos del libro (completo, incremental, diferencial, snapshot) son los mismos. Lo propio de Windows es:

**El atributo "Archivo" (A).** Cada archivo tiene un atributo que Windows activa cuando el archivo se modifica. Las herramientas clásicas lo usan para saber qué copiar:

| Tipo de backup | Qué hace con el atributo A |
|---|---|
| Completo | Copia todo y **desactiva** el atributo en todos los archivos |
| Incremental | Copia solo los que tienen el atributo y lo **desactiva** (la próxima vez solo se copia lo nuevo) |
| Diferencial | Copia solo los que tienen el atributo pero **no lo desactiva** (cada vez se copia todo lo cambiado desde el último completo) |

| Comando | Función |
|---|---|
| `attrib archivo` | Muestra los atributos (la `A` indica "pendiente de copia") |
| `attrib -a archivo` / `attrib +a archivo` | Quita / pone el atributo |
| `robocopy origen destino /M` | Copia solo los archivos con atributo A y lo quita (**incremental**) |
| `robocopy origen destino /A` | Copia solo los archivos con atributo A sin quitarlo (**diferencial**) |

**Instantáneas: VSS (Servicio de instantáneas de volumen).** Es el equivalente de los snapshots del libro y la base de casi todas las copias en Windows: congela una imagen del volumen en un instante, para poder copiarla aunque los archivos estén abiertos. Las aplicaciones compatibles (SQL Server, Active Directory, Hyper-V...) tienen un **escritor VSS** que deja sus datos en un estado coherente antes de la instantánea.

| Comando | Función |
|---|---|
| `vssadmin list shadows` | Lista las instantáneas existentes |
| `vssadmin create shadow /for=D:` | Crea una instantánea del volumen D: |
| `vssadmin delete shadows /for=D: /oldest` | Borra la instantánea más antigua |
| `vssadmin list writers` | Lista los escritores VSS y su estado (útil cuando una copia falla) |
| `vssadmin list shadowstorage` | Espacio reservado para instantáneas en cada volumen |
| `vssadmin resize shadowstorage /for=D: /on=D: /maxsize=20%` | Cambia ese espacio |
| `diskshadow` | Herramienta interactiva o por scripts para instantáneas avanzadas (exclusiva de Windows Server) |

**Instantáneas de carpetas compartidas** ("Versiones anteriores"): se activan por volumen (Propiedades del disco → pestaña **Instantáneas**). Programan instantáneas automáticas y permiten a los usuarios recuperar ellos mismos una versión anterior de un archivo desde la pestaña "Versiones anteriores". No sustituyen a una copia de seguridad, porque están en el mismo disco.

### 2.3 Medios de backup

| Medio | Situación en Windows Server 2025 |
|---|---|
| Cinta magnética | **Copias de seguridad de Windows Server no admite cintas**. Hace falta software de terceros (Veeam, Veritas Backup Exec, Commvault...) o System Center DPM |
| Disco óptico | No admitido por Copias de seguridad de Windows Server |
| HDD / SSD externo o interno dedicado | Opción recomendada para Copias de seguridad de Windows Server. El disco se formatea y se reserva para las copias; guarda varias versiones automáticamente |
| Carpeta compartida de red (`\\servidor\recurso`) | Admitido, pero **solo guarda la última copia** (cada copia sustituye a la anterior) |
| Volumen normal | Admitido, con peor rendimiento |
| Nube | **Azure Backup** (agente MARS instalado en el servidor) o Microsoft Azure Backup Server (MABS) |

### 2.4 Herramienta `tar`

Windows Server 2025 incluye **`tar.exe`** (`C:\Windows\System32\tar.exe`). Es la versión **bsdtar** (la misma familia que la de FreeBSD), no GNU tar, así que las opciones básicas son iguales pero faltan algunas avanzadas. Además, PowerShell tiene cmdlets propios para archivos **ZIP**.

**Crear archivos (equivalente a la Tabla 2.2):**

| Opción / acción Linux | Equivalente en Windows Server 2025 |
|---|---|
| `tar -cvf copia.tar carpeta` | `tar -cvf copia.tar carpeta` (igual) |
| `-z` (gzip) | `tar -czvf copia.tar.gz carpeta` (igual) |
| `-j` / `-J` (bzip2 / xz) | Depende de las librerías incluidas en `tar.exe`; se comprueba con `tar --version` |
| Crear un ZIP | `tar -a -cf copia.zip carpeta` (`-a` elige el formato por la extensión) o `Compress-Archive -Path C:\Datos -DestinationPath C:\Copias\datos.zip` |
| `-u` (añadir solo lo modificado) | `Compress-Archive -Path C:\Datos -DestinationPath datos.zip -Update` |
| `-g archivo.snar` (incremental) | **No existe** en bsdtar. Para incrementales se usa `robocopy /M` (apartado 2.1) o Copias de seguridad de Windows Server |
| `--level=0` | No existe |

**Ver y verificar (equivalente a la Tabla 2.3):**

| Opción Linux | Equivalente en Windows Server 2025 |
|---|---|
| `-t` (listar) | `tar -tvf copia.tar` (también lista archivos `.zip`) |
| `-v` (detalle) | `-v` (igual) |
| `-d` / `--diff` (comparar con los originales) | No existe en bsdtar. Alternativa: extraer a una carpeta temporal y comparar con `robocopy origen temporal /L /E` (solo lista las diferencias, no copia nada) |
| `-W` (verificar al crear) | No existe. Alternativa: calcular huellas con `Get-FileHash archivo -Algorithm SHA256` antes y después |

**Restaurar (equivalente a la Tabla 2.4):**

| Opción Linux | Equivalente en Windows Server 2025 |
|---|---|
| `tar -xvf copia.tar` | `tar -xvf copia.tar` (igual; detecta la compresión sola) |
| `tar -xvf copia.tar -C destino` | `tar -xvf copia.tar -C destino` (igual) |
| Extraer un ZIP | `tar -xf copia.zip` o `Expand-Archive -Path datos.zip -DestinationPath C:\Restaurado` |

Importante: `tar` y los ZIP **no guardan los permisos NTFS** (listas de control de acceso, ACL). Para copiar datos manteniendo los permisos se usa `robocopy` con `/COPYALL` (apartado 2.6) o Copias de seguridad de Windows Server.

### 2.5 Herramienta `mt` (control de cintas magnéticas)

**No hay equivalente en Windows Server 2025.** No existe un comando `mt`, y la antigua herramienta de copias con soporte de cinta (NTBackup) desapareció hace muchas versiones.

| Concepto Linux | Situación en Windows Server 2025 |
|---|---|
| `/dev/st0`, `/dev/nst0` | La unidad de cinta aparece como `\\.\Tape0`, pero solo la usan programas de copia de terceros |
| Ver la unidad de cinta | Administrador de dispositivos (`devmgmt.msc`) o `Get-PnpDevice -Class TapeDrive` |
| Cambiadores de cintas (`mtx`) | Aparecen como dispositivos de clase `MediumChanger`; se controlan desde el software de copias |
| `mt status`, `rewind`, `eject`... | Funciones del software de copias de terceros o de System Center DPM |

### 2.6 Herramienta `rsync` → `robocopy`

Windows no incluye `rsync`. Su equivalente integrado es **`robocopy`** ("Robust File Copy"), que copia y sincroniza carpetas, reintenta ante errores, conserva permisos y puede trabajar con varios hilos.

| Uso | Comando |
|---|---|
| Copia local de una carpeta con todo su contenido | `robocopy C:\Datos E:\Copia /E` |
| Sincronizar (el destino queda igual que el origen, **borrando** lo que sobra) | `robocopy C:\Datos E:\Copia /MIR` |
| Copia a otro servidor por la red (SMB) | `robocopy C:\Datos \\SRV02\Copias\Datos /MIR` |
| Copia conservando permisos, propietario y auditoría | `robocopy C:\Datos E:\Copia /MIR /COPYALL /DCOPY:DAT` |
| Probar sin copiar nada (qué haría) | `robocopy C:\Datos E:\Copia /MIR /L` |

**Opciones principales de `robocopy`:**

| Opción | Función | Equivalente `rsync` aproximado |
|---|---|---|
| `/E` | Copia subcarpetas, incluidas las vacías | `-r` |
| `/MIR` | Espejo: como `/E` + borra en el destino lo que ya no está en el origen | `-a --delete` |
| `/COPYALL` | Copia datos, atributos, fechas, permisos NTFS, propietario y auditoría | `-a` (parte de permisos y propietario) |
| `/DCOPY:DAT` | Conserva datos, atributos y fechas de las carpetas | `-t` en carpetas |
| `/Z` | Modo reiniciable: si se corta la red, continúa donde iba | `--partial` |
| `/B` | Modo copia de seguridad: copia aunque no se tenga permiso de lectura | — |
| `/MT:16` | Usa 16 hilos a la vez (más rápido con muchos archivos) | — |
| `/R:2 /W:5` | 2 reintentos, esperando 5 segundos (por defecto son 1 millón de reintentos de 30 s) | — |
| `/XO` | No sobrescribe archivos más nuevos en el destino | `-u` |
| `/XD carpeta` / `/XF *.tmp` | Excluye carpetas / archivos | `--exclude` |
| `/L` | Solo lista, no copia | `-n` (`--dry-run`) |
| `/LOG:C:\robocopy.log /TEE` | Guarda un registro y lo muestra también en pantalla | `--log-file` |
| `/NP` | No muestra el porcentaje de progreso (registros más limpios) | Sin `--progress` |
| `/MOT:10` | Queda vigilando y repite la copia cada 10 minutos si hay cambios | — |

Los **códigos de salida** de `robocopy` son especiales: del 0 al 7 significan que todo fue bien (con o sin cambios); **8 o más** significa error. Hay que tenerlo en cuenta en los scripts.

**Cifrado en las copias remotas** (en Linux: `rsync` por SSH frente al demonio `rsync://` sin cifrar):
- `robocopy` hacia `\\servidor\recurso` usa **SMB**. Con SMB 3 se puede exigir cifrado en el recurso compartido: `Set-SmbShare -Name Copias -EncryptData $true`.
- Windows Server 2025 incluye **SMB sobre QUIC** en todas sus ediciones: SMB cifrado con TLS 1.3 a través del puerto UDP 443, pensado para redes no confiables.
- Por SSH se pueden copiar archivos con `scp` y `sftp`, que vienen con el cliente OpenSSH de Windows.

### 2.7 Herramienta `dd`

**Windows no incluye `dd`.** Sus usos se cubren con otras herramientas:

| Uso de `dd` en Linux | Equivalente en Windows Server 2025 |
|---|---|
| Copia exacta de un disco o volumen | Copias de seguridad de Windows Server (copia por bloques del volumen, ver 2.8), o **Disk2vhd** (Sysinternals, gratuita) que convierte un disco en un archivo `.vhdx` incluso con el sistema en marcha |
| Imagen de un volumen para restaurar en otro equipo | `DISM /Capture-Image /ImageFile:E:\imagen.wim /CaptureDir:D:\ /Name:"Datos"` y después `DISM /Apply-Image /ImageFile:E:\imagen.wim /Index:1 /ApplyDir:D:\` (copia por archivos, no por bits) |
| Poner a cero un disco completo (`if=/dev/zero`) | `diskpart` → `select disk 2` → `clean all` (escribe ceros en todo el disco) |
| Poner a cero un volumen al formatearlo | `format E: /FS:NTFS /P:1` (una pasada de ceros) |
| Borrar de forma segura el espacio libre | `cipher /w:E:\` |
| Informática forense (copia bit a bit) | Herramientas de terceros (por ejemplo, FTK Imager) |

Igual que avisa el libro con `dd`: una copia a bajo nivel de un volumen **en uso** puede quedar incoherente. En Windows se evita porque las herramientas de copia usan VSS.

### 2.8 Otras utilidades de línea de comandos para backup

**Copias de seguridad de Windows Server** (Windows Server Backup): la herramienta integrada. Se instala con:
```
Install-WindowsFeature Windows-Server-Backup
```
Consola gráfica: `wbadmin.msc`. Comando: `wbadmin`.

| Comando | Función |
|---|---|
| `wbadmin get disks` | Lista los discos y su identificador (para elegir el disco de destino) |
| `wbadmin enable backup -addtarget:{ID-disco} -schedule:21:00 -include:C:,D: -allCritical` | Programa una copia diaria a las 21:00 en un disco dedicado |
| `wbadmin start backup -backupTarget:E: -include:D:\Datos -quiet` | Hace una copia en este momento |
| `wbadmin start backup -backupTarget:E: -allCritical -quiet` | Copia de todo lo necesario para recuperar el sistema completo |
| `wbadmin start systemstatebackup -backupTarget:E:` | Copia del estado del sistema (Registro, arranque, Active Directory...) |
| `wbadmin get versions` | Lista las copias disponibles |
| `wbadmin get items -version:09/23/2026-21:00` | Muestra qué contiene una copia |
| `wbadmin start recovery -version:09/23/2026-21:00 -itemType:File -items:D:\Datos -recoveryTarget:D:\Restaurado -recursive` | Restaura archivos |
| `wbadmin delete backup -keepVersions:10` | Borra las copias antiguas y deja las 10 últimas |
| `wbadmin get status` | Progreso de la copia en curso |

Las copias de Windows Server Backup son **por bloques** y usan VSS. En un disco dedicado, cada copia se comporta como incremental (solo guarda los bloques cambiados), pero cualquier versión se puede restaurar como si fuera completa.

También hay cmdlets de PowerShell (`New-WBPolicy`, `Add-WBVolume`, `Add-WBFileSpec`, `New-WBBackupTarget`, `Set-WBSchedule`, `Set-WBPolicy`, `Start-WBBackup`, `Get-WBSummary`).

**Otras herramientas:**

| Herramienta | Función | Equivalente Linux aproximado |
|---|---|---|
| `xcopy` | Copia de archivos antigua (sigue existiendo; hoy se recomienda `robocopy`) | `cp -r` |
| `esentutl /y archivo /vss /d destino` | Copia un archivo que está bloqueado por otro programa, usando una instantánea | — |
| `reg export HKLM\SOFTWARE\Empresa copia.reg` / `reg save` | Copia de una rama del Registro | Copiar archivos de `/etc` |
| `Export-VM` | Exporta una máquina virtual de Hyper-V completa | — |
| `ntdsutil` → `ifm` | Copia de la base de datos de Active Directory | — |
| `DISM /Capture-Image` | Imagen de una carpeta o volumen en formato WIM | `tar` / `cpio` |

### 2.9 Soluciones de backup completas

| Solución | Descripción |
|---|---|
| **System Center Data Protection Manager (DPM)** | Solución de Microsoft para empresas: copias centralizadas de servidores, SQL Server, Exchange, Hyper-V..., con soporte de cintas |
| **Azure Backup** (agente MARS) y **Microsoft Azure Backup Server (MABS)** | Copias a la nube de Microsoft |
| Veeam, Veritas Backup Exec, Commvault, Acronis | Soluciones comerciales de terceros muy extendidas |
| Bacula / Bareos | Tienen cliente (File Daemon) para Windows, así que el servidor Windows puede formar parte de una infraestructura de copias Linux |
| BackupPC | Puede copiar equipos Windows a través de SMB sin instalar nada en ellos |
| Duplicati, restic | Programas libres con versión para Windows, con cifrado y copia a la nube |

---

## 3. Instalación de programas desde código fuente

En Windows lo normal **no es compilar**: los programas se distribuyen ya compilados. Por eso primero se ve cómo se instala software en Windows Server 2025 y después cómo se compilaría si hiciera falta.

### 3.1 Instalar software (equivalente a `apt-get` / `yum`)

| Acción | Comando en Windows Server 2025 |
|---|---|
| Ver los **roles y características** disponibles e instalados | `Get-WindowsFeature` |
| Instalar un rol o característica | `Install-WindowsFeature Web-Server -IncludeManagementTools` |
| Desinstalar | `Uninstall-WindowsFeature Web-Server` |
| Características a petición (FoD) | `Get-WindowsCapability -Online` / `Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0` |
| Instalar un programa `.msi` sin preguntas y con registro | `msiexec /i programa.msi /qn /l*v C:\logs\instalacion.log` |
| Desinstalar un `.msi` | `msiexec /x programa.msi /qn` |
| Gestor de paquetes **winget** (solo con Experiencia de escritorio) | `winget search 7zip`, `winget install 7zip.7zip`, `winget list`, `winget upgrade --all`, `winget uninstall 7zip.7zip` |
| Módulos de PowerShell (galería PowerShell) | `Find-Module`, `Install-Module NombreModulo` |
| Ver programas instalados | `appwiz.cpl` (Programas y características) o la clave `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall` |

**Actualizar el sistema** (equivalente a `apt-get update && apt-get dist-upgrade` / `yum update`):
- En Server Core o Desktop: `sconfig` → opción de instalar actualizaciones.
- Con escritorio: Configuración → Windows Update.
- PowerShell (módulo `WindowsUpdateProvider`): `Start-WUScan` para buscar, `Install-WUUpdates` para instalar, `Get-WUIsPendingReboot` para saber si falta reiniciar.
- En empresas: WSUS (sigue disponible pero Microsoft ya no lo desarrolla), Azure Update Manager o Configuration Manager.

### 3.2 Compilar desde código fuente

Si hace falta compilar, el equivalente al paquete `build-essential` / "Development Tools" son las **Build Tools de Visual Studio** (compilador de Microsoft `cl.exe`, enlazador `link.exe`, `nmake`, `msbuild`) y **CMake**:
```
winget install Microsoft.VisualStudio.2022.BuildTools
winget install Kitware.CMake
```
Tras instalarlas se trabaja desde el **"Developer PowerShell for VS"** (o se ejecuta `vcvars64.bat`), que prepara las variables de entorno del compilador.

| Paso del libro (Linux) | Equivalente en Windows Server 2025 |
|---|---|
| 0. Herramientas de compilación | Build Tools de Visual Studio + CMake (o Git para Windows) |
| 1. Obtener el código fuente | Tarball (`.tar.gz`), ZIP o `git clone` |
| 2. Extraer | `tar -xzvf programa.tar.gz` o `Expand-Archive programa.zip` |
| 3. `./configure` | `cmake -S . -B build` (revisa dependencias y genera los archivos de compilación) |
| 4. `make` | `cmake --build build --config Release` (o `nmake` / `msbuild` si el proyecto los usa) |
| 5. `make install` | `cmake --install build` (instala normalmente en `C:\Program Files\NombrePrograma`) |
| `./configure --help` | `cmake -S . -B build -LH` (lista las opciones del proyecto) |
| Guardar la salida con `tee` | `cmake --build build 2>&1 \| Tee-Object -FilePath compilacion.log` |
| Dependencias (librerías de desarrollo) | **vcpkg** (gestor de librerías C/C++ de Microsoft) |
| `diff` y `patch` | `git diff` y `git apply` (con Git para Windows). Windows incluye `fc` y `Compare-Object` para comparar archivos, pero no `patch` |
| Código fuente en `/usr/src/` | No hay una carpeta estándar; es habitual `C:\src\` |
| Añadir el programa al PATH | `[Environment]::SetEnvironmentVariable('Path', $env:Path + ';C:\Program Files\Programa\bin', 'Machine')` (evitar `setx`, que corta las rutas largas) |

Los archivos de ayuda del libro (`README`, `INSTALL`, `COPYING`...) siguen siendo igual de útiles; en proyectos para Windows suele haber además instrucciones específicas para CMake o Visual Studio.

**Programas pensados para Linux** (que usan `./configure` y `make` de verdad):
- **MSYS2** o **Cygwin**: entornos con `gcc`, `make`, `bash` y autotools para compilar en Windows.
- **WSL** (Subsistema de Windows para Linux): ejecuta una distribución Linux real dentro de Windows (`wsl --install`), donde se compila como en el libro. El resultado es un programa Linux, no un programa de Windows.

---

## 4. Medición y gestión del uso de recursos

### 4.1 Elementos clave a monitorizar

Los mismos del libro. En Windows todos se miden con **contadores de rendimiento**, que tienen nombres del tipo `\Objeto(Instancia)\Contador`. Los más importantes:

| Recurso | Contador | Qué indica |
|---|---|---|
| CPU | `\Processor(_Total)\% Processor Time` | Uso total de CPU |
| Carga (equivalente a la carga media) | `\System\Processor Queue Length` | Hilos esperando CPU. Si es alto de forma continua, falta CPU |
| Memoria | `\Memory\Available MBytes` | Memoria libre |
| Memoria comprometida | `\Memory\% Committed Bytes In Use` | Memoria reservada frente al máximo (RAM + archivo de paginación) |
| Paginación (swap) | `\Memory\Pages/sec` y `\Paging File(_Total)\% Usage` | Actividad y uso del archivo de paginación |
| Disco | `\PhysicalDisk(*)\Avg. Disk sec/Read` y `Avg. Disk sec/Write` | Latencia del disco (tiempo por operación) |
| Disco | `\PhysicalDisk(*)\Current Disk Queue Length` | Operaciones esperando al disco |
| Red | `\Network Interface(*)\Bytes Total/sec` | Tráfico por tarjeta |

Para ver los nombres disponibles: `Get-Counter -ListSet *` o `typeperf -q`. Ojo: en un Windows en español, las herramientas gráficas muestran los contadores traducidos, pero en PowerShell los nombres en inglés funcionan siempre.

### 4.2 Herramientas de línea de comandos (equivalente a la Tabla 2.7)

| Comando Linux | Equivalente en Windows Server 2025 | Qué monitoriza |
|---|---|---|
| `uptime` | `(Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime` | Tiempo encendido. No hay "carga media": se usa el contador `Processor Queue Length` |
| `free` | `Get-CimInstance Win32_OperatingSystem \| Select TotalVisibleMemorySize, FreePhysicalMemory` o `systeminfo` | Memoria física y virtual (valores en KB) |
| `swapon -s` | `Get-CimInstance Win32_PageFileUsage` | Archivo de paginación (`C:\pagefile.sys`): tamaño y uso |
| `vmstat 2 10` | `Get-Counter '\Memory\Pages/sec','\Processor(_Total)\% Processor Time' -SampleInterval 2 -MaxSamples 10` | Memoria y CPU a intervalos |
| `top` / `htop` | **Administrador de tareas** (`taskmgr`, funciona también en Server Core) o `Get-Process \| Sort-Object CPU -Descending \| Select-Object -First 15` | Procesos, CPU, memoria |
| — | **Monitor de recursos** (`resmon`, solo con escritorio) | CPU, memoria, disco y red por proceso, en tiempo real |
| `mpstat` | `Get-Counter '\Processor(*)\% Processor Time'` | Uso de cada procesador |
| `sar` | **Monitor de rendimiento** (`perfmon`) y `logman` (ver 4.3) | Recolector histórico |
| `sar 2 20` | `typeperf "\Processor(_Total)\% Processor Time" -si 2 -sc 20` | CPU 20 veces cada 2 segundos |
| `iostat` | `Get-Counter '\PhysicalDisk(*)\Disk Transfers/sec','\PhysicalDisk(*)\Avg. Disk sec/Transfer'` | Carga de E/S por disco |
| `iotop` | `resmon` → pestaña **Disco**, o `Get-Counter '\Process(*)\IO Data Bytes/sec'` | E/S por proceso |
| `ps` | `tasklist` / `Get-Process` | Lista de procesos |
| `ps` con usuario | `tasklist /v` o `Get-Process -IncludeUserName` | Procesos con su usuario |
| `pstree` | `Get-CimInstance Win32_Process \| Select ProcessId, ParentProcessId, Name` o Process Explorer (Sysinternals) | Procesos y su proceso padre |
| `w` | `quser` | Quién está conectado |
| `kill` | `Stop-Process -Id 1234` o `taskkill /PID 1234 /F` | Terminar un proceso |
| `pmap PID` | `Get-Process -Id 1234 \| Select WorkingSet64, PrivateMemorySize64, VirtualMemorySize64` y `(Get-Process -Id 1234).Modules` | Memoria y librerías (DLL) de un proceso. Mapa completo: **VMMap** (Sysinternals) |
| `lsof` (archivos) | `resmon` → CPU → "Identificadores asociados", o **Handle** (Sysinternals: `handle.exe nombre`) | Qué proceso tiene abierto un archivo |
| `lsof` (archivos abiertos por la red) | `Get-SmbOpenFile` | Archivos compartidos que tienen abiertos los clientes |
| `lsof -i` (conexiones) | `Get-NetTCPConnection -OwningProcess 1234` | Conexiones de un proceso |
| `ip -s link` | `Get-NetAdapterStatistics` / `netstat -e` | Estadísticas de las tarjetas de red |
| `ip addr` / `ip route` | `Get-NetIPAddress` / `Get-NetRoute` (o `ipconfig /all` y `route print`) | Direcciones y rutas |
| `netstat` | `netstat -ano` (sigue existiendo y no está obsoleto en Windows) | Conexiones con su PID (`-b` muestra el programa) |
| `ss` | `Get-NetTCPConnection -State Listen` / `Get-NetUDPEndpoint` | Sockets TCP y UDP |
| `iftop` / `iptraf` / `ntop` | `resmon` → pestaña **Red**, Administrador de tareas, o `Get-Counter '\Network Interface(*)\Bytes Total/sec' -Continuous` | Tráfico de red |
| `mtr` | `pathping servidor` | Ruta hacia un destino con estadísticas de pérdida en cada salto |
| `traceroute` | `tracert` o `Test-NetConnection servidor -TraceRoute` | Ruta hacia un destino |
| — | `Test-NetConnection servidor -Port 443` | Comprueba si un puerto TCP responde |
| `tcpdump` | **`pktmon`** (integrado) o `netsh trace` | Captura de paquetes |
| `watch -n 5 comando` | Bucle de PowerShell: `while ($true) { Clear-Host; comando; Start-Sleep 5 }`, o la opción `-Continuous` de `Get-Counter` | Repetir un comando cada N segundos |

**Captura de paquetes con `pktmon`** (equivalente a `tcpdump`):

| Comando | Función |
|---|---|
| `pktmon filter add -p 443` | Captura solo el tráfico del puerto 443 |
| `pktmon start --capture --pkt-size 0` | Empieza a capturar paquetes completos (se guardan en `PktMon.etl`) |
| `pktmon stop` | Termina la captura |
| `pktmon etl2pcap PktMon.etl -o captura.pcapng` | Convierte la captura a formato `pcapng`, que se abre con Wireshark |
| `pktmon filter remove` | Borra los filtros |

**Herramientas Sysinternals**: colección gratuita de Microsoft (Process Explorer, Process Monitor, Handle, VMMap, TCPView, Autoruns...). Muy usadas para diagnóstico; se descargan de Microsoft Learn o se instalan con `winget install Microsoft.Sysinternals.Suite`.

Nota: `wmic` está obsoleto. En Windows Server 2025 ya no viene instalado de serie (se puede añadir como característica a petición). Se usa en su lugar `Get-CimInstance`, como en las tablas anteriores.

### 4.3 El equivalente de `sar`: Monitor de rendimiento y `logman`

| Concepto `sar` (Linux) | Equivalente en Windows Server 2025 |
|---|---|
| Paquete `sysstat` | Integrado en el sistema |
| Demonio `sadc` (recolector) | **Conjuntos de recopiladores de datos** del Monitor de rendimiento, creados con `perfmon` o con `logman` |
| `/var/log/sa/` | `C:\PerfLogs\` (archivos `.blg`) |
| `sar` (ver datos guardados) | Abrir el `.blg` en `perfmon`, o convertirlo con `relog` |
| `sar 2 20` (tiempo real) | `typeperf` o `Get-Counter` |

**Comando `logman`** (crea y controla los recopiladores):

| Comando | Función |
|---|---|
| `logman create counter Base -c "\Processor(_Total)\% Processor Time" "\Memory\Available MBytes" "\PhysicalDisk(_Total)\Avg. Disk sec/Transfer" -si 00:10:00 -o C:\PerfLogs\Base` | Crea un recopilador que guarda esos contadores cada 10 minutos (como `sar` por defecto) |
| `logman start Base` / `logman stop Base` | Arranca / para la recogida |
| `logman query` | Lista los recopiladores y su estado |
| `logman delete Base` | Borra el recopilador |
| `relog C:\PerfLogs\Base_000001.blg -f csv -o base.csv` | Convierte los datos a CSV (para Excel u otras herramientas) |

`perfmon` incluye además el conjunto predefinido **"System Performance"** (Rendimiento del sistema), que recoge datos durante un minuto y genera un **informe** con los problemas detectados. Se lanza desde `perfmon` → Conjuntos de recopiladores de datos → Sistema. Para un informe rápido del estado general: `perfmon /report`.

### 4.4 Planificación de capacidad (capacity planning)

Los cuatro pasos del libro son los mismos. Herramientas propias de Windows Server 2025:

**System Insights**: característica de Windows Server que analiza localmente los datos de rendimiento y **predice** el uso futuro de recursos con modelos de aprendizaje automático. Es la herramienta más directa para el objetivo 200.2 del examen.

| Comando | Función |
|---|---|
| `Install-WindowsFeature System-Insights -IncludeManagementTools` | Instala la característica |
| `Get-InsightsCapability` | Lista las predicciones disponibles: uso de CPU, de red, de almacenamiento total y de cada volumen |
| `Invoke-InsightsCapability -Name "CPU capacity forecasting"` | Ejecuta una predicción ahora |
| `Get-InsightsCapabilityResult -Name "CPU capacity forecasting"` | Muestra el resultado (si se prevé superar la capacidad y cuándo) |
| `Set-InsightsCapabilitySchedule` | Programa cada cuánto se ejecuta |

También se puede ver de forma gráfica en **Windows Admin Center**.

**Soluciones completas de monitorización** (equivalentes a las del libro):

| Solución | Descripción |
|---|---|
| **System Center Operations Manager (SCOM)** | Solución de Microsoft para monitorizar servidores, aplicaciones y red, con alertas e informes históricos |
| **Azure Monitor** (con Azure Arc) | Monitorización en la nube de Microsoft, también para servidores locales |
| **Windows Admin Center** | Consola web gratuita: rendimiento en tiempo real, System Insights, gestión del servidor |
| Nagios / Icinga | Monitorizan Windows mediante un agente (NSClient++ para Nagios; Icinga for Windows) |
| Zabbix | Tiene agente oficial para Windows |
| Cacti / MRTG | Leen datos por SNMP; en Windows se suelen combinar con uno de los agentes anteriores |
| Telegraf + InfluxDB / Grafana | Alternativa moderna: Telegraf lee los contadores de Windows y los envía a una base de datos para graficarlos (papel parecido a collectd + RRDTool) |

### 4.5 Resolución de problemas de recursos

| Recurso | Herramientas útiles | Notas |
|---|---|---|
| Memoria | Administrador de tareas, `resmon` (pestaña Memoria), contadores `Available MBytes`, `Pages/sec`, `% Committed Bytes In Use` | La memoria también se gestiona en páginas de 4 KB. Cuando falta RAM, Windows mueve páginas al archivo de paginación `C:\pagefile.sys` (equivalente a la swap). Un `Pages/sec` alto de forma continua junto con poca memoria libre indica falta de RAM |
| Procesos | `Get-Process`, `tasklist /svc`, Process Explorer, VMMap | No hay un estado `D` como en Linux; un proceso bloqueado por disco se detecta por la cola de disco y por la latencia alta |
| CPU | `Get-CimInstance Win32_Processor \| Select Name, NumberOfCores, NumberOfLogicalProcessors, L2CacheSize, L3CacheSize`, `msinfo32`, Administrador de tareas, contadores `% Processor Time` y `Processor Queue Length` | Equivalente a `/proc/cpuinfo` y `lscpu`. Si `NumberOfLogicalProcessors` es el doble de `NumberOfCores`, hay hyper-threading |
| E/S de disco | `resmon` (pestaña Disco), contadores de `PhysicalDisk`, `Get-PhysicalDisk`, `Get-StorageReliabilityCounter` | Igual que dice el libro: hay que conocer el hardware (SAN, iSCSI, Espacios de almacenamiento, NTFS o ReFS). Una latencia por encima de unos 20 ms de forma continua suele indicar un disco saturado |
| Red | `resmon` (pestaña Red), `Get-NetAdapterStatistics`, `netstat -ano`, `pktmon`, `ping`, `tracert`, `pathping`, `Test-NetConnection` | Para redes grandes, mejor usar una solución completa (SCOM, Zabbix...) |

---

## 5. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 2) | Equivalente en Windows Server 2025 |
|---|---|
| `who -T`, `w` | `quser` / `qwinsta` |
| `write` | `msg usuario` o `msg IDsesión` |
| `wall` | `msg *` |
| `mesg y/n` | No existe |
| `shutdown -k` / `/etc/nologin` | `change logon /disable` o `/drain` |
| `shutdown -c` | `shutdown /a` |
| `/etc/issue` | Aviso legal de inicio de sesión (`LegalNoticeCaption` / `LegalNoticeText`) |
| `Banner` en `sshd_config` | `Banner` en `C:\ProgramData\ssh\sshd_config` |
| `/etc/motd` | Perfil de PowerShell o script de inicio de sesión |
| Snapshots | VSS (`vssadmin`, `diskshadow`) e instantáneas de carpetas compartidas |
| `tar` | `tar.exe` (bsdtar) y `Compress-Archive` / `Expand-Archive` |
| `tar -g` (incremental) | `robocopy /M` o Copias de seguridad de Windows Server |
| `mt` / `mtx` | No existen; software de copias de terceros o System Center DPM |
| `rsync` | `robocopy` |
| `dd` | Windows Server Backup, Disk2vhd, `DISM /Capture-Image`, `diskpart clean all` |
| Amanda, Bacula... | Windows Server Backup (`wbadmin`), System Center DPM, Azure Backup |
| `build-essential` / "Development Tools" | Build Tools de Visual Studio + CMake |
| `./configure && make && make install` | `cmake -S . -B build` + `cmake --build build` + `cmake --install build` |
| `apt-get` / `yum` | `Install-WindowsFeature`, `winget`, `msiexec` |
| `apt-get upgrade` / `yum update` | Windows Update (`sconfig`, `Install-WUUpdates`) |
| `free`, `/proc/meminfo` | `Get-CimInstance Win32_OperatingSystem`, `systeminfo` |
| `/proc/cpuinfo`, `lscpu` | `Get-CimInstance Win32_Processor`, `msinfo32` |
| `top` / `htop` | Administrador de tareas, `resmon`, `Get-Process` |
| `vmstat`, `mpstat`, `iostat` | `Get-Counter` / `typeperf` |
| `sar` + `sadc` | `perfmon` + `logman` (datos en `C:\PerfLogs`) |
| `ps`, `pstree`, `kill` | `Get-Process`, `tasklist`, `Win32_Process`, `Stop-Process` / `taskkill` |
| `pmap` | VMMap (Sysinternals) |
| `lsof` | `handle.exe`, `resmon`, `Get-SmbOpenFile` |
| `ip`, `ss`, `netstat` | `Get-NetIPAddress`, `Get-NetTCPConnection`, `netstat -ano` |
| `mtr` | `pathping` |
| `tcpdump` | `pktmon` / `netsh trace` |
| `watch` | Bucle de PowerShell o `Get-Counter -Continuous` |
| Planificación de capacidad | System Insights, SCOM, Azure Monitor |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 2 ("Maintaining the System") del libro LPIC-2. Fuentes: documentación oficial de Microsoft Learn (msg, query user, change logon, OpenSSH para Windows, Copias de seguridad de Windows Server y wbadmin, vssadmin, diskshadow, robocopy, tar, DISM, Install-WindowsFeature, winget, contadores de rendimiento, logman, typeperf, relog, pktmon, System Insights) y la lista oficial de características quitadas o en desuso de Windows Server.*
