# LPIC-2 · Capítulo 2: Maintaining the System
### Equivalencias en FreeBSD 15

Este documento recoge, para cada concepto visto en el Capítulo 2 del libro LPIC-2 (comunicación con los usuarios, copias de seguridad, compilación desde código fuente y monitorización de recursos), cuál es su equivalente en **FreeBSD 15**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo: FreeBSD distingue entre el **sistema base** (kernel + herramientas propias, que vienen siempre instaladas) y el **software de terceros** (se instala con `pkg` o desde los **Ports**, y va siempre a `/usr/local`). Muchas herramientas del libro existen en FreeBSD, pero algunas vienen en el sistema base con opciones distintas (versiones BSD, no GNU) y otras hay que instalarlas como paquete.

---

## 1. Comunicación con los usuarios del sistema

### 1.1 Mensajería "fluida" (en tiempo real)

| Comando Linux | Equivalente en FreeBSD 15 | Función |
|---|---|---|
| `write usuario terminal` | `write usuario terminal` (igual) | Mensaje privado a un usuario conectado |
| `wall mensaje` | `wall mensaje` (igual) | Mensaje a todos los usuarios conectados |
| `mesg`, `mesg y`, `mesg n` | `mesg`, `mesg y`, `mesg n` (igual) | Ver o cambiar si tu terminal acepta mensajes |
| `who -T` | `who -T` (igual) | Muestra `+` (acepta mensajes), `-` (no acepta) o `?` |
| `notify-send "Título" "Mensaje"` | `notify-send` (paquete `libnotify`: `pkg install libnotify`) | Notificación gráfica en el escritorio |
| `shutdown [opciones] tiempo [mensaje]` | `shutdown [opciones] tiempo [mensaje]` | Apaga o reinicia avisando a los usuarios |

Además, FreeBSD incluye `talk` (conversación en pantalla partida entre dos usuarios), heredado de los Unix clásicos.

**Opciones de `shutdown`: aquí hay diferencias importantes con Linux:**

| Opción Linux | Opción FreeBSD 15 | Función |
|---|---|---|
| `-k` | `-k` | Avisa y bloquea nuevos logins (salvo root), pero **no** apaga (igual idea) |
| `-c` (cancelar) | No existe. Se cancela matando el proceso: `pkill shutdown` | Cancelar un apagado programado |
| — | `-c` | **Cuidado**: en FreeBSD `-c` significa *power cycle* (apagar y volver a encender), no cancelar |
| `-H` | `-h` | Detiene el sistema (halt) sin cortar la corriente |
| `-P` | `-p` | Apaga el equipo (power off) |
| `-r` | `-r` | Reinicia |
| `--no-wall` | No existe | FreeBSD siempre avisa a los usuarios |
| — | `-o` | Ejecuta directamente `halt` o `reboot` sin pasar por `init` |

Ejemplo: `shutdown -r +15 "Reinicio por mantenimiento"`.

Durante un apagado programado, `shutdown` crea el archivo `/var/run/nologin`, que impide nuevos inicios de sesión (igual idea que `/etc/nologin` en Linux).

### 1.2 Mensajería "estática" (mensajes de bienvenida / login)

| Archivo Linux | Equivalente en FreeBSD 15 | Función |
|---|---|---|
| `/etc/issue` | `/etc/gettytab` (campo `im=` mensaje inicial, o `if=/etc/issue` para mostrar un archivo) | Mensaje antes del login en las consolas locales |
| `/etc/issue.net` + `Banner` en `sshd_config` | `Banner /ruta/archivo` en `/etc/ssh/sshd_config` (igual que en Linux) | Mensaje antes del login por SSH |
| `/etc/motd` | **`/etc/motd.template`** | Mensaje del día tras iniciar sesión |

Detalle importante sobre el MOTD en FreeBSD: el mensaje que ven los usuarios está en `/var/run/motd`, y se **genera en cada arranque** a partir de `/etc/motd.template` (se le añade la versión del sistema). Por eso:
- Se edita `/etc/motd.template`, no el archivo final.
- Para aplicar el cambio sin reiniciar: `service motd restart`.
- Para que no se añada la versión del sistema: `sysrc update_motd="NO"`.
- Un usuario puede dejar de ver el MOTD creando el archivo `~/.hushlogin` en su carpeta personal.

Para usuarios en entorno gráfico, igual que en Linux, el mensaje se configura en el gestor de sesión (SDDM, LightDM, etc.).

---

## 2. Copias de seguridad (backup)

### 2.1 y 2.2 Conceptos y tipos de backup

Los conceptos del libro (backup, archivado, completo, incremental, diferencial y snapshot) son **exactamente los mismos** en FreeBSD. La diferencia es que FreeBSD ofrece snapshots de forma nativa en sus dos sistemas de archivos:

| Sistema de archivos | Snapshots | Herramienta |
|---|---|---|
| ZFS (el recomendado y por defecto en el instalador) | Sí, instantáneos y sin coste de espacio inicial | `zfs snapshot`, `zfs send`, `zfs receive` |
| UFS (sistema clásico de FreeBSD) | Sí, pero más limitados | `mksnap_ffs`, o `dump -L` (hace el snapshot automáticamente) |

### 2.3 Medios de backup

Iguales que en Linux (cinta, disco óptico, HDD, SSD, nube). Solo cambian los nombres de dispositivo:

| Dispositivo | Linux | FreeBSD 15 |
|---|---|---|
| Disco SATA | `/dev/sda` | `/dev/ada0` |
| Disco USB / SCSI | `/dev/sdb` | `/dev/da0` |
| Disco NVMe | `/dev/nvme0n1` | `/dev/nda0` |
| Cinta con rebobinado automático | `/dev/st0` | `/dev/sa0` |
| Cinta sin rebobinado | `/dev/nst0` | `/dev/nsa0` |
| Cinta que se expulsa al cerrar | — | `/dev/esa0` |
| Cambiador de cintas | `/dev/sgX` | `/dev/ch0` |

### 2.4 Herramienta `tar`

Diferencia clave: el `tar` de FreeBSD es **bsdtar** (basado en la librería libarchive), no GNU tar. Las opciones básicas son iguales, pero **no tiene** las opciones de backup incremental de GNU tar (`-g`, `--listed-incremental`, `--level`) ni `--compare`/`--verify`.

Si se necesita exactamente el comportamiento del libro, se puede instalar GNU tar: `pkg install gtar` (se ejecuta como `gtar`, con las mismas opciones que en Linux).

**Opciones para crear backups (equivalente a la Tabla 2.2):**

| Opción GNU tar (Linux) | Opción bsdtar (FreeBSD 15) | Función |
|---|---|---|
| `-c` | `-c` | Crea un archivo tar |
| `-u` | `-u` | Añade solo los ficheros más nuevos que los que ya hay en el archivo |
| — | `-r` | Añade ficheros al final de un archivo existente |
| `-g file` (incremental) | No existe. Se usa `--newer-mtime-than archivo_referencia` o `--newer-mtime "fecha"` | Copiar solo lo modificado desde una fecha o desde otro archivo |
| `--level=#` | No existe | — |
| `-z` | `-z` | Comprime con gzip |
| `-j` | `-j` | Comprime con bzip2 |
| `-J` | `-J` | Comprime con xz |
| — | `--zstd` | Comprime con zstd (rápido y con buena compresión) |
| — | `-a` | Elige la compresión automáticamente según la extensión (ej. `.tar.xz`) |

Ejemplo de backup completo comprimido (igual que en el libro):
```
tar -Jcvf Project42.tar.xz *.dat
```

Ejemplo de backup incremental con bsdtar (solo lo modificado después de crear el backup completo):
```
tar -Jcvf Project42_Incremental.tar.xz --newer-mtime-than Project42.tar.xz *.dat
```
Si se compara siempre con el **último incremental**, se obtiene un backup incremental; si se compara siempre con el **backup completo**, se obtiene un backup diferencial.

**Opciones para verificar/ver backups (equivalente a la Tabla 2.3):**

| Opción GNU tar | Opción bsdtar | Función |
|---|---|---|
| `-d` / `--compare` | No existe (usar `gtar -d`) | Comparar el archivo con los ficheros del disco |
| `-t` | `-t` | Lista el contenido |
| `-v` | `-v` | Muestra los ficheros según se procesan |
| `-W` / `--verify` | No existe | Verificar al crear |

**Opciones para restaurar backups (equivalente a la Tabla 2.4):**

| Opción GNU tar | Opción bsdtar | Función |
|---|---|---|
| `-x` | `-x` | Extrae ficheros |
| `-z`, `-j`, `-J` | No hacen falta al extraer: bsdtar **detecta la compresión automáticamente** | Descomprimir |
| — | `-C carpeta` | Extrae dentro de la carpeta indicada (también existe en GNU tar) |

### 2.5 Herramienta `mt` (control de cintas magnéticas)

En FreeBSD `mt` viene en el sistema base (no hay que instalar ningún paquete).

| Elemento | Linux | FreeBSD 15 |
|---|---|---|
| Paquete | `mt-st` | Incluido en el sistema base |
| Sintaxis | `mt [-f dispositivo] operación [count]` | Igual |
| Dispositivo por defecto | `/dev/st0` | `/dev/nsa0` (se puede cambiar con la variable `TAPE`) |

**Operaciones principales (equivalente a la Tabla 2.5):**

| Operación Linux | Operación FreeBSD 15 | Función |
|---|---|---|
| `status` | `status` | Estado de la cinta |
| `load` | `load` | Carga la cinta (si la unidad lo soporta) |
| `erase` | `erase` | Borra la cinta |
| `fsf count` | `fsf count` | Avanza `count` ficheros |
| `bsf count` | `bsf count` | Retrocede `count` ficheros |
| `tell` | `rdspos` | Muestra la posición actual |
| `eod` | `eod` (o `eom`) | Va al final de los datos |
| `rewind` | `rewind` | Rebobina |
| `eject` / `offline` | `offline` (o `rewoffl`) | Rebobina y expulsa |
| — | `weof count` | Escribe marcas de fin de fichero |

Ejemplos:
```
mt -f /dev/sa0 status
tar -Jcvf /dev/sa0 /home/chris/Project
```

**Cambiadores de cintas** (el `mtx` del libro): FreeBSD incluye en el sistema base el comando **`chio`** (dispositivo `/dev/ch0`). Ejemplo: `chio status`, `chio move slot 1 drive 0`. También se puede instalar `mtx` como paquete.

### 2.6 Herramienta `rsync`

`rsync` **no viene en el sistema base**; se instala con `pkg install rsync`. Una vez instalado, su uso es **idéntico** al de Linux:

| Uso | Comando |
|---|---|
| Backup local | `rsync -av origen destino` |
| Backup remoto cifrado (vía OpenSSH) | `rsync -av origen usuario@servidor:/ruta` |
| Backup remoto sin cifrar (demonio rsync) | `rsync -av origen rsync://servidor/modulo` |

Para usar FreeBSD como servidor rsync: configuración en `/usr/local/etc/rsync/rsyncd.conf` y activar con `sysrc rsyncd_enable="YES"` + `service rsyncd start`.

### 2.7 Herramienta `dd`

Igual que en Linux, con pequeñas diferencias:

| Comando | Función |
|---|---|
| `dd if=/dev/ada0 of=/dev/ada1 bs=1m` | Copia bit a bit de un disco a otro (en FreeBSD el tamaño se escribe en minúscula: `1m`, `4k`) |
| `dd if=/dev/zero of=/dev/da1 bs=1m count=10` | Pone a cero el principio del disco `/dev/da1` |
| `dd ... status=progress` | Muestra el progreso mientras copia |
| `Ctrl+T` durante la copia | Truco propio de FreeBSD: muestra el estado actual de `dd` (y de muchos otros comandos) sin interrumpirlo |

La misma advertencia del libro: no usar `dd` sobre un disco montado.

### 2.8 Otras utilidades de línea de comandos para backup

| Utilidad del libro | Situación en FreeBSD 15 |
|---|---|
| `cpio` | Incluido en el sistema base (versión bsdcpio) |
| `dump` / `restore` | Incluidos en el sistema base y muy usados, **solo para UFS** |
| `star` | Disponible como paquete; poco útil en FreeBSD porque no hay SELinux |
| — | `pax`: incluido en el sistema base; formato estándar POSIX, sustituto de tar y cpio |

**`dump` y `restore` (UFS):**

| Comando | Función |
|---|---|
| `dump -0Lauf /backup/root.dump /` | Backup completo (nivel 0) de `/`. `-L` hace un snapshot para copiar el sistema en marcha de forma segura, `-a` ajusta el tamaño automáticamente, `-u` anota la fecha en `/etc/dumpdates` |
| `dump -1Lauf /backup/root1.dump /` | Backup incremental de nivel 1 (lo cambiado desde el nivel 0) |
| `restore -rf /backup/root.dump` | Restaura el backup completo en la carpeta actual |
| `restore -if /backup/root.dump` | Restauración interactiva (se eligen los ficheros) |
| `restore -tf /backup/root.dump` | Lista el contenido del backup |

Los niveles de `dump` (0 a 9) son la forma clásica Unix de hacer backups completos e incrementales.

**Snapshots y backups con ZFS** (la forma más habitual en FreeBSD moderno):

| Comando | Función |
|---|---|
| `zfs snapshot zroot/home@lunes` | Crea un snapshot instantáneo del dataset `zroot/home` |
| `zfs snapshot -r zroot@lunes` | Snapshot de todo el pool, incluyendo los datasets hijos |
| `zfs list -t snapshot` | Lista los snapshots |
| `zfs rollback zroot/home@lunes` | Vuelve el dataset al estado del snapshot |
| `zfs destroy zroot/home@lunes` | Borra un snapshot |
| `zfs send zroot/home@lunes > /backup/home.zfs` | Guarda un snapshot completo en un archivo |
| `zfs send -i @lunes zroot/home@martes > /backup/home_inc.zfs` | Backup **incremental** (solo los cambios entre dos snapshots) |
| `zfs send zroot/home@lunes \| ssh servidor zfs receive tank/copia` | Copia el snapshot a otro equipo por SSH |
| `zfs receive tank/copia < /backup/home.zfs` | Restaura desde un archivo |

Los ficheros de un snapshot también se pueden consultar sin restaurar nada, en la carpeta oculta `.zfs/snapshot/` del dataset (ej. `/home/.zfs/snapshot/lunes/`).

**Snapshots en UFS:** `mksnap_ffs /var/.snap/copia` crea un snapshot del sistema de archivos que contiene esa carpeta; se puede montar con `mdconfig` + `mount` para recuperar ficheros.

### 2.9 Soluciones de backup completas

| Solución del libro | Situación en FreeBSD 15 |
|---|---|
| Amanda | Disponible como paquete (servidor y cliente) |
| Bacula | Disponible como paquete (servidor y cliente) |
| Bareos | Disponible como paquete |
| Duplicity | Disponible como paquete |
| BackupPC | Disponible como paquete |

Otras muy usadas en FreeBSD: `restic` y `borgbackup` (backups cifrados con deduplicación), y `sanoid`/`syncoid` o `zrepl` (automatizan snapshots ZFS y su envío a otro servidor).

Para buscar el nombre exacto de cualquier paquete: `pkg search nombre` (ej. `pkg search bacula`).

---

## 3. Instalación de programas desde código fuente

### 3.1 Compilación manual (el método del libro)

El proceso `./configure` → `make` → `make install` funciona igual, pero hay diferencias importantes:

| Paso | Linux | FreeBSD 15 |
|---|---|---|
| 0. Herramientas de compilación | `apt-get install build-essential` / `yum groupinstall "Development Tools"` | El compilador (**clang/LLVM**) y `make` ya vienen en el sistema base. Si el sistema se instaló en modo mínimo con pkgbase y faltan, se instalan desde el repositorio `FreeBSD-base` con `pkg` |
| 0.1 Actualizar el sistema | `apt-get update && apt-get dist-upgrade` / `yum update` | Sistema base: `freebsd-update fetch install` (o `pkg upgrade` si se usa pkgbase). Paquetes: `pkg update && pkg upgrade` |
| 1-2. Obtener y extraer el código | `tar -zxvf programa.tar.gz` | Igual (`tar -xvf programa.tar.gz`, bsdtar detecta la compresión) |
| 3. Preparar | `./configure` | `./configure` (igual) |
| 4. Compilar | `make` | `make` **o `gmake`** (ver nota) |
| 5. Instalar | `make install` | `make install` / `gmake install` |

**Nota importante sobre `make`:** el `make` de FreeBSD es **BSD make**, no GNU make. Muchos programas pensados para Linux usan `Makefile` con sintaxis GNU y fallan con BSD make. En ese caso:
```
pkg install gmake
gmake
gmake install
```
Otras herramientas que a menudo hay que instalar: `pkg install autoconf automake libtool pkgconf`.

Otros detalles:
- Los programas compilados a mano deben instalarse en **`/usr/local`** (es el prefijo por defecto de `./configure`, así que normalmente no hay que cambiar nada). En FreeBSD todo el software ajeno al sistema base va ahí.
- `diff` y `patch` están en el sistema base y funcionan igual que en Linux.
- `/usr/src/` en FreeBSD está reservada para el **código fuente del propio sistema operativo** (se descarga con `git clone https://git.FreeBSD.org/src.git /usr/src`), no para programas de terceros.

### 3.2 Los Ports: el método recomendado en FreeBSD para compilar

La **Colección de Ports** es un árbol de carpetas (en `/usr/ports`) con "recetas" para compilar miles de programas: descarga el código, aplica los parches necesarios para FreeBSD, resuelve dependencias, compila e instala, y además registra el programa en la base de datos de `pkg` (se puede desinstalar limpiamente, cosa que no ocurre con `make install` manual).

| Comando | Función |
|---|---|
| `git clone https://git.FreeBSD.org/ports.git /usr/ports` | Descarga el árbol de ports (requiere `pkg install git`) |
| `git -C /usr/ports pull` | Actualiza el árbol de ports |
| `cd /usr/ports/www/nginx` | Entra en la carpeta de un port (categoría/nombre) |
| `make config` | Menú para elegir opciones de compilación (como `./configure --with-...`) |
| `make install clean` | Compila, instala y limpia los ficheros temporales |
| `make deinstall` | Desinstala el programa |
| `make reinstall` | Reinstala tras cambiar opciones |
| `pkg info` | Lista todo lo instalado, venga de paquetes o de ports |

Estructura de un port: `Makefile` (la receta), `distinfo` (sumas de verificación del código descargado), `pkg-descr` (descripción) y la carpeta `files/` (parches, igual que los que se aplican con `patch` en el libro).

Herramientas relacionadas: `portmaster` (actualiza ports instalados) y `poudriere` (compila paquetes propios en jails limpios, usado en servidores).

No es buena idea mezclar sin control paquetes binarios (`pkg install`) y ports compilados del mismo programa, porque pueden chocar sus versiones.

---

## 4. Medición y gestión del uso de recursos

### 4.1 Elementos clave a monitorizar

Los mismos que en el libro (uptime, CPU, memoria y swap, I/O de disco y de red, ancho de banda). La diferencia principal es que FreeBSD **no monta `/proc` por defecto**: la información del sistema se consulta con **`sysctl`**.

| Comando | Función |
|---|---|
| `sysctl -a` | Muestra todos los parámetros del kernel |
| `sysctl hw.model hw.ncpu` | Modelo de CPU y número de núcleos |
| `sysctl hw.physmem` | Memoria física en bytes |
| `sysctl vm.loadavg` | Carga media |

### 4.2 Herramientas de línea de comandos (equivalente a la Tabla 2.7)

| Comando Linux | Equivalente en FreeBSD 15 | Notas |
|---|---|---|
| `free` | `top` (cabecera), `vmstat`, `swapinfo -h` | No existe `free`. `swapinfo -h` muestra el uso de swap. Paquete opcional: `freecolor` |
| `vmstat` | `vmstat` (base) | Columnas algo distintas. `vmstat -w 2` repite cada 2 segundos |
| `top` / `htop` | `top` (base) / `htop` (paquete) | El `top` de FreeBSD tiene modos extra (ver abajo) |
| `uptime` | `uptime` (igual) | |
| `mpstat` | `top -P` o `systat -vmstat` | `top -P` muestra el uso de cada CPU por separado |
| `sar` | No existe (el paquete `sysstat` es solo para Linux) | Alternativas: `systat` (en tiempo real) y soluciones como collectd o Zabbix para históricos |
| `iostat` | `iostat` (base) | `iostat -x -w 2` para detalle extendido cada 2 segundos |
| — | `gstat` | Propio de FreeBSD: actividad de cada disco en tiempo real (muy usado) |
| — | `zpool iostat -v 2` | Actividad de disco de un pool ZFS |
| `iotop` | `top -m io` | El `top` de FreeBSD muestra I/O por proceso con `-m io` (ordenar con `-o total`) |
| `ps` | `ps` (versión BSD) | `ps aux` funciona igual |
| `pstree` | `ps -d` o `pstree` (paquete) | `ps -auxd` muestra los procesos en forma de árbol |
| `w` | `w` (igual) | |
| `pmap PID` | `procstat -v PID` | Mapa de memoria de un proceso |
| `lsof` | `fstat`, `procstat -f PID`, `sockstat` | `lsof` también existe como paquete |
| `ip -s link` | `netstat -i` / `netstat -I em0 -w 1` | Estadísticas de interfaz (el segundo, en tiempo real) |
| `ip route` | `netstat -rn` / `route -n get default` | Tabla de rutas |
| `netstat` | `netstat` (base) | En FreeBSD **no está obsoleto**: es la herramienta estándar |
| `ss` | `sockstat` | `sockstat -4 -l` muestra los puertos IPv4 a la escucha |
| `iftop` | `iftop` (paquete) o `systat -ifstat` | |
| `iptraf` | No existe (solo Linux) | Alternativas: `systat -ifstat`, `trafshow` o `bwm-ng` (paquetes) |
| `ntop` | `ntopng` (paquete) | |
| `mtr` | `mtr` (paquete `mtr-nox11`) | `traceroute` y `ping` vienen en el sistema base |
| `tcpdump` | `tcpdump` (base) | Igual |

**Comando `watch`: cuidado.** En FreeBSD, `watch` es otro programa distinto (sirve para espiar la sesión de otra terminal). El equivalente al `watch` de Linux es:
- Paquete `cmdwatch`: `cmdwatch -n 5 iostat`.
- O usar el intervalo que traen muchos comandos: `iostat -w 5`, `vmstat -w 5`, `netstat -w 5`.

**Opciones útiles del `top` de FreeBSD:**

| Comando | Función |
|---|---|
| `top -P` | Uso de cada CPU por separado |
| `top -S` | Muestra también los procesos del sistema (kernel) |
| `top -H` | Muestra los hilos (threads) |
| `top -m io -o total` | Modo I/O: qué procesos leen y escriben más en disco |

**`systat`** (sistema base): panel en tiempo real con varias vistas, equivalente a varias herramientas del libro juntas.

| Comando | Qué muestra |
|---|---|
| `systat -vmstat` | Resumen general: CPU, memoria, discos, interrupciones |
| `systat -iostat` | I/O de disco |
| `systat -ifstat` | Tráfico de red por interfaz |
| `systat -netstat` | Conexiones de red |
| `systat -swap` | Uso de swap |
| `systat -zarc` | Uso de la caché ARC de ZFS |

### 4.3 La herramienta `sar` en detalle

No existe en FreeBSD. Lo más parecido para tener **datos históricos**:
- Los informes automáticos de **`periodic`**: cada día, semana y mes FreeBSD ejecuta scripts de mantenimiento que generan un resumen del estado (uso de disco, estado de ZFS, logins, etc.) y lo envían por correo a root. Configuración en `/etc/periodic.conf` (valores por defecto en `/etc/defaults/periodic.conf`; scripts en `/etc/periodic/daily/`, `weekly/` y `monthly/`).
- Una solución de monitorización como `collectd`, `munin` o `zabbix` (apartado 4.4).

Para análisis muy detallado del rendimiento, FreeBSD incluye **DTrace** en el sistema base (herramienta avanzada de trazado del kernel).

### 4.4 Planificación de capacidad (capacity planning)

Los pasos del libro son los mismos. Todas las soluciones que cita existen como paquetes:

| Solución | Situación en FreeBSD 15 |
|---|---|
| RRDTool | Paquete `rrdtool` |
| Cacti | Paquete `cacti` |
| collectd | Paquete `collectd5`; configuración en `/usr/local/etc/collectd.conf` |
| MRTG | Paquete `mrtg` |
| Nagios | Paquete `nagios4` |
| Icinga | Paquete `icinga2` |

Recordatorio: en FreeBSD todos los programas de terceros guardan su configuración en **`/usr/local/etc/`**, no en `/etc/` (que queda solo para el sistema base). Se activan como cualquier servicio: `sysrc collectd_enable="YES"` + `service collectd start`.

Otras soluciones habituales en FreeBSD: `zabbix` (agente y servidor), `munin` y Prometheus con `node_exporter`.

### 4.5 Resolución de problemas de recursos

| Recurso | Comandos útiles en FreeBSD 15 | Notas |
|---|---|---|
| Memoria | `top`, `vmstat`, `swapinfo -h`, `sysctl vm.stats` | `top` divide la memoria en: **Active** (en uso), **Inact** (inactiva, reutilizable), **Laundry** (pendiente de pasar a swap), **Wired** (bloqueada por el kernel, no puede ir a swap) y **Free** (libre). Con ZFS, la caché **ARC** ocupa bastante memoria Wired: es normal, se libera si otros programas la necesitan |
| Procesos | `ps`, `procstat`, `top` | El estado `D` (esperando a disco) existe igual en `ps`. Columna `b` de `vmstat`, igual que en Linux |
| CPU | `sysctl hw.model hw.ncpu`, `grep CPU /var/run/dmesg.boot`, `uptime`, `top -P` | No hay `/proc/cpuinfo` ni `lscpu` |
| I/O de disco | `iostat`, `gstat`, `top -m io`, `zpool iostat`, `fstat` | Igual que en el libro, conviene conocer el hardware y el sistema de archivos (UFS o ZFS) |
| Red | `netstat`, `sockstat`, `systat -ifstat`, `tcpdump`, `ping`, `traceroute`, `ifconfig` | En FreeBSD `ifconfig` **no está obsoleto**: es la herramienta principal de red (no existe el comando `ip`) |

---

## 5. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 2) | Equivalente en FreeBSD 15 |
|---|---|
| `write`, `wall`, `mesg`, `who -T` | Iguales |
| `shutdown -c` (cancelar) | `pkill shutdown` (en FreeBSD `-c` es apagar y encender) |
| `shutdown -H` / `-P` | `shutdown -h` / `-p` |
| `/etc/issue` | `/etc/gettytab` |
| `/etc/motd` | `/etc/motd.template` (genera `/var/run/motd`) |
| GNU `tar -g` (incremental) | bsdtar `--newer-mtime-than`, o `gtar` instalado como paquete |
| `mt` con `/dev/st0` | `mt` con `/dev/sa0` (sistema base) |
| `mtx` | `chio` (sistema base) |
| `rsync` | `rsync` (paquete) |
| `dd` | `dd` (igual; `Ctrl+T` muestra el progreso) |
| `dump` / `restore` | `dump` / `restore` (sistema base, para UFS) |
| Snapshots | `zfs snapshot` / `zfs send` / `zfs receive` (ZFS) y `mksnap_ffs` (UFS) |
| `build-essential` / "Development Tools" | Ya incluido en el sistema base (clang + BSD make); `gmake` como paquete |
| Compilar desde código fuente | Manual (`./configure && gmake`) o, mejor, **Ports** (`make install clean`) |
| `apt-get upgrade` / `yum update` | `freebsd-update fetch install` + `pkg upgrade` |
| `/proc`, `free`, `lscpu` | `sysctl`, `top`, `swapinfo` |
| `sar` | `systat` (tiempo real) + `periodic` y soluciones de monitorización |
| `iotop` | `top -m io` |
| `pmap` | `procstat -v` |
| `lsof` / `ss` | `fstat`, `procstat -f` / `sockstat` |
| `ip` | `ifconfig`, `netstat`, `route` |
| `watch` | `cmdwatch` (paquete) o la opción `-w` de cada comando |
| Configuración en `/etc/` | Sistema base en `/etc/`, programas de terceros en `/usr/local/etc/` |

---

*Documento elaborado como contrapartida en FreeBSD 15 al Capítulo 2 ("Maintaining the System") del libro LPIC-2. Fuentes: FreeBSD Handbook (capítulos de Paquetes y Ports, Almacenamiento/Backups, ZFS y Configuración) y páginas de manual de FreeBSD: shutdown(8), wall(1), gettytab(5), motd(5), tar(1) (bsdtar), mt(1), chio(1), dd(1), dump(8), restore(8), zfs(8), top(1), systat(1), gstat(8), procstat(1), sockstat(1), periodic(8).*
