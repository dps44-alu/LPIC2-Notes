# LPIC-2 · Capítulo 2: Maintaining the System
### Recopilación de comandos y archivos de configuración

Objetivos del examen cubiertos: 200.1 (Measure and troubleshoot resource usage), 200.2 (Predict future resource needs), 206.1 (Make and install programs from source), 206.2 (Backup operations), 206.3 (Notify users on system-related issues)

---

## 1. Comunicación con los usuarios del sistema

### 1.1 Mensajería "fluida" (en tiempo real)

| Comando/archivo | Función |
|---|---|
| `write usuario terminal_id` | Envía un mensaje privado a un usuario concreto conectado a una terminal |
| `wall mensaje` | Envía un mensaje a todos los usuarios conectados que tengan permiso de escritura en su terminal |
| `mesg` | Muestra si tu terminal acepta mensajes (`is y` / `is n`) |
| `mesg y` / `mesg n` | Activa/desactiva la recepción de mensajes (concede o revoca el acceso de escritura a tu terminal) |
| `who -T` | Muestra qué usuarios tienen el acceso de escritura activado (`+`), desactivado (`-`) o si están en modo gráfico (`?`) |
| `notify-send "Título" "Mensaje"` | Envía una notificación gráfica de escritorio (GUI); requiere el paquete `libnotify-bin` si no está instalado |
| `/sbin/shutdown [opciones] tiempo [mensaje]` | Apaga, reinicia o cambia de runlevel el sistema, y puede enviar un mensaje de aviso |

Opciones útiles de `shutdown`:

| Opción | Función |
|---|---|
| `-k` | Envía el aviso y bloquea nuevos logins, pero NO apaga el sistema |
| `-c` | Cancela un shutdown programado |
| `-H` | Detiene el sistema (halt) |
| `-P` | Apaga el sistema (power off) |
| `-r` | Reinicia el sistema |
| `--no-wall` | No envía el mensaje `wall` a los usuarios (solo el que ejecuta el comando lo recibe) |

Nota: `systemctl` (visto en el Capítulo 1) también envía un mensaje `wall` automáticamente con los comandos `emergency`, `halt`, `kexec`, `power-off`, `reboot` o `rescue`. Se puede evitar con `--no-wall`.

### 1.2 Mensajería "estática" (mensajes de bienvenida / login)

| Archivo | Función |
|---|---|
| `/etc/issue` | Mensaje mostrado antes del login en terminales locales (tty) |
| `/etc/issue.net` | Igual que el anterior, pero para conexiones remotas (por defecto solo Telnet; para usarlo con OpenSSH hay que activar la línea `Banner /etc/issue.net` en `/etc/ssh/sshd_config`) |
| `/etc/motd` | Mensaje del día (Message Of The Day): se muestra justo después de iniciar sesión, antes del prompt |

Para usuarios en entorno gráfico, el mensaje de bienvenida se configura desde el gestor de pantalla (GDM, LightDM), no desde estos archivos.

No hay diferencias relevantes entre Debian y Red Hat en esta sección: los archivos y comandos son los mismos en ambas familias.

---

## 2. Copias de seguridad (backup)

### 2.1 Conceptos previos

- **Backup**: copia de los datos para salvaguardar el original.
- **Archivo/archivado (archive)**: copia (o los propios datos) movidos a almacenamiento de largo plazo, normalmente en soporte más barato y en otra ubicación física.

### 2.2 Tipos de backup

| Tipo | Qué copia | Ventaja | Inconveniente |
|---|---|---|---|
| Full (completo) | Todos los datos, sin mirar la fecha de modificación | Restauración más rápida | Tarda más en crearse y ocupa más espacio |
| Incremental | Solo lo modificado/añadido desde la última copia (de cualquier tipo) | Rápido de crear, poco espacio | Restauración lenta (hay que aplicar el full + todos los incrementales en orden) |
| Differential | Solo lo modificado/añadido desde el último full | Término medio entre full e incremental | Más espacio que el incremental, pero restauración más rápida (solo full + último differential) |
| Snapshot | Full inicial + tabla de punteros; luego copias incrementales que actualizan la tabla | Permite volver a cualquier punto en el tiempo, poco espacio | Requiere soporte de snapshots (copy-on-write o split-mirror) |

### 2.3 Medios de backup (pros y contras)

| Medio | Características |
|---|---|
| Cinta magnética | Barata, buena para archivado a largo plazo, lenta |
| Disco óptico | Portátil, capacidad limitada |
| HDD | Barato, más rápido que la cinta; riesgo si el backup está en el mismo lugar que el original |
| SSD | El más rápido, pero también el más caro |
| Nube (Amazon S3, Google Cloud Storage, etc.) | Elimina la necesidad de rotar soportes; ojo con límites de datos y si ofrecen cifrado en tránsito |

### 2.4 Herramienta `tar`

`tar` (Tape ARchiver) crea un **archivo tar** (archive file); si además se comprime, se llama **tarball**.

**Opciones para crear backups (Tabla 2.2):**

| Opción larga | Opción corta | Función |
|---|---|---|
| `--create` | `-c` | Crea un archivo tar (completo o incremental) |
| `--update` | `-u` | Añade al archivo tar solo los ficheros modificados desde su creación (no vale con cinta) |
| `--listed-incremental=file` | `-g file` | Crea un backup incremental o completo usando metadatos de `file` (archivo `.snar`) |
| `--level=#` | — | Fuerza un nivel de backup concreto |
| `--gzip` | `-z` | Comprime con gzip |
| `--bzip2` | `-j` | Comprime con bzip2 |
| `--xz` | `-J` | Comprime con xz |

Ejemplo de backup completo comprimido:
```
tar -Jcvf Project42.tar.xz *.dat
```

Ejemplo de backup incremental usando un archivo snapshot (`.snar`):
```
tar -g Archive1.snar -Jcvf Project42_Incremental.tar.xz *.dat
```
El primer backup con `-g` es nivel 0 (completo); los siguientes son nivel 1, 2, etc. Se puede forzar un nuevo full con `--level=0`.

**Opciones para verificar/ver backups (Tabla 2.3):**

| Opción larga | Opción corta | Función |
|---|---|---|
| `--compare` / `--diff` | `-d` | Compara el contenido del archivo tar con los ficheros externos |
| `--list` | `-t` | Lista el contenido del archivo tar/tarball |
| `--verbose` | `-v` | Muestra los ficheros según se procesan |
| `--verify` | `-W` | Verifica el archivo justo al crearlo (no se puede usar con compresión) |

**Opciones para restaurar backups (Tabla 2.4):**

| Opción larga | Opción corta | Función |
|---|---|---|
| `--extract` / `--get` | `-x` | Extrae ficheros de un archivo tar |
| `--gunzip` | `-z` | Descomprime un tarball gzip |
| `--bunzip2` | `-j` | Descomprime un tarball bzip2 |
| `--unxz` | `-J` | Descomprime un tarball xz |

### 2.5 Herramienta `mt` (control de cintas magnéticas)

Se usa junto a `tar` cuando el destino del backup es una cinta magnética.

| Elemento | Valor |
|---|---|
| Paquete si no está instalado | `mt-st` |
| Sintaxis general | `mt [-f dispositivo] operación [count] [argumentos]` |
| `/dev/st0`, `/dev/ht1` | Dispositivos de cinta con **rebobinado automático** (SCSI y PATA respectivamente) |
| `/dev/nst0`, `/dev/nht1` | Dispositivos de cinta **sin** rebobinado automático |

**Operaciones principales (Tabla 2.5):**

| Operación | Función |
|---|---|
| `status` | Muestra el estado de la cinta |
| `load` | Carga la cinta |
| `erase` | Borra todo el contenido |
| `fsf count` | Avanza `count` ficheros |
| `bsf count` | Retrocede `count` ficheros |
| `tell` | Muestra la posición actual del cabezal |
| `eod` | Va al final de los datos |
| `rewind` | Rebobina la cinta |
| `eject` / `offline` | Rebobina y expulsa la cinta |

Ejemplo: `mt -f /dev/st0 status`

Para hacer el backup en sí se sigue usando `tar`, apuntando al dispositivo de cinta:
```
tar -Jcvf /dev/st0 /home/chris/Project
```

Nota: para cambiadores de cintas SCSI con varias unidades se usa el comando `mtx`, no `mt`.

### 2.6 Herramienta `rsync`

| Uso | Comando |
|---|---|
| Backup local | `rsync -av origen destino` |
| Backup remoto cifrado (vía OpenSSH) | `rsync -av origen usuario@servidor:/ruta` |
| Backup remoto SIN cifrar (demonio rsync) | `rsync -av origen rsync://servidor:/ruta` |

La opción `-a` (archive) equivale a `-rlptgoD`. `-v` aumenta el detalle de salida, `-h` hace la salida más legible, `--progress` muestra el progreso.

### 2.7 Herramienta `dd`

Copia a bajo nivel (bit a bit), muy usada en informática forense.

| Comando | Función |
|---|---|
| `dd of=dispositivo-salida if=dispositivo-entrada` | Copia bit a bit de un disco o partición a otro |
| `dd of=/dev/sdc if=/dev/zero count=10` | Pone a cero (borra) el disco `/dev/sdc` |

Advertencia: no usar `dd` para hacer backup o restaurar un disco que está montado en ese momento; puede corromper los datos. Tampoco es adecuado para backups incrementales diarios.

### 2.8 Otras utilidades de línea de comandos para backup

`cpio`, `dump`/`restore`, `star` (tar con soporte SELinux), además de `tar`, `rsync` y `dd` ya vistos.

### 2.9 Soluciones de backup completas (GUI/web)

| Solución | Descripción |
|---|---|
| Amanda (Amanda Network Backup) | Usa `dump` y `tar` en su núcleo; edición Community gratuita y Enterprise con GUI (Zmanda Management Console) |
| Bacula | Open source (AGPL v3); componentes Director, Console, File, Storage y Monitor; interfaces web, GUI y texto |
| Bareos | Fork de Bacula, casi clon, con Director también disponible para Windows |
| Duplicity | Línea de comandos o GUI (deja-dup, preinstalado en Ubuntu); permite cifrar con GPG |
| BackupPC | Interfaz web, usa `rsync` y `tar` en su núcleo; soporta Linux, Unix, Mac OS X y Windows |

---

## 3. Instalación de programas desde código fuente

Pasos generales del proceso (todos requieren privilegios de superusuario):

| Paso | Comando/acción |
|---|---|
| 0. Instalar herramientas de compilación | **Red Hat**: `yum groupinstall "Development Tools"` · **Debian**: `sudo apt-get install build-essential` |
| 0.1 Actualizar el sistema antes de instalar | **Red Hat**: `yum update` · **Debian**: `sudo apt-get update && sudo apt-get dist-upgrade` |
| 1. Obtener el código fuente | Normalmente en formato tarball (`.tar.gz`, `.tar.bz2`, `.tar.xz`) |
| 2. Descomprimir/extraer | `tar -zxvf programa-version.tar.gz` |
| 3. Preparar la compilación | `./configure` (revisa dependencias y compiladores, genera/actualiza el `Makefile` a partir de `Makefile.in`) |
| 4. Compilar | `make` (usa el `Makefile` para generar los binarios) |
| 5. Instalar | `make install` (o `sudo make install`): copia los binarios y ficheros de soporte a su ubicación definitiva |

Ficheros de ayuda que suelen incluirse en el tarball: `README`, `INSTALL` (instrucciones adicionales), `COPYING` (licencia), `RELEASE-NOTES`, `NEWS` (cambios de la versión).

Otros detalles:
- `./configure --help` muestra las opciones disponibles del script.
- Es buena práctica redirigir la salida de `configure`, `make` y `make install` con `tee` a un log.
- Para aplicar parches de seguridad: se genera un fichero con `diff` entre el código original y el modificado, y se aplica con el comando `patch`, para luego volver a compilar.
- Ubicación recomendada para guardar el código fuente ya instalado: `/usr/src/`.

Diferencia Debian vs Red Hat en esta sección: solo en los comandos de gestión de paquetes usados como paso previo (`yum` vs `apt-get`); el proceso `configure` / `make` / `make install` en sí es idéntico en ambas familias.

---

## 4. Medición y gestión del uso de recursos

### 4.1 Elementos clave a monitorizar

Uptime del sistema, uso y carga de CPU, uso de memoria y swap, I/O de disco, I/O de red, rendimiento de firewall/router, ancho de banda de red.

### 4.2 Herramientas de línea de comandos (Tabla 2.7, resumen)

| Comando | Qué monitoriza | Tipo de salida |
|---|---|---|
| `free` | Memoria física y swap | Estática |
| `vmstat` | Memoria virtual (swap) | Estática/dinámica |
| `top` / `htop` | CPU, memoria, procesos, uptime | Dinámica |
| `uptime` | Tiempo encendido, carga media, usuarios | Estática |
| `mpstat` | Estadísticas de CPU (multiprocesador) | Estática/dinámica |
| `sar` | CPU, memoria, red, I/O de dispositivo (recolector histórico) | Estática/dinámica |
| `iostat` | Carga de I/O por dispositivo | Estática/dinámica |
| `iotop` | I/O por proceso/hilo | Dinámica |
| `ps` / `pstree` / `w` | Procesos, estados, consumo de CPU | Estática |
| `pmap` | Mapa de memoria de un proceso (PID) | Estática |
| `lsof` | Ficheros abiertos y conexiones de red por proceso | Estática |
| `ip -s link` / `ip route` | Estadísticas de red y rutas (sustituye a `netstat`) | Estática |
| `netstat` | Red y rutas (obsoleto, usar `ip`) | Estática |
| `ss` | Estadísticas de sockets (más info que `netstat`) | Estática |
| `iftop` / `iptraf` / `ntop` | Tráfico de red | Dinámica |
| `mtr` | Rutas de red hacia una URL | Dinámica |
| `tcpdump` | Analizador/sniffer de paquetes | Dinámica |

Comando `watch`: convierte cualquier herramienta de salida estática en dinámica, repitiéndola cada 2 segundos por defecto. Ejemplo: `watch -n 5 iostat` (cada 5 segundos).

### 4.3 La herramienta `sar` en detalle

- Paquete: `sysstat` (si no viene instalado).
- El demonio `sadc` guarda los datos en `/var/log/sa/`.
- Sin opciones, `sar` muestra el uso de CPU del día actual en intervalos de 10 minutos.
- Sintaxis con intervalos: `sar intervalo cuenta` (ej. `sar 2 20` muestra datos de CPU 20 veces, cada 2 segundos).

### 4.4 Planificación de capacidad (capacity planning)

Pasos:
1. Entender las necesidades actuales de los usuarios.
2. Monitorizar el uso actual de recursos.
3. Recoger las necesidades y planes futuros.
4. Hacer predicciones y tomar decisiones con esos datos.

**Soluciones completas de monitorización (recolector + presentación):**

| Solución | Descripción |
|---|---|
| RRDTool (Round-Robin Database Tool) | Base de datos circular; base de otras herramientas como Cacti, MRTG y Nagios |
| Cacti | Presentación (gráficos) sobre RRDTool; almacena en MySQL, frontend en PHP |
| collectd | Demonio recolector escrito en C; configuración por plugins en `collectd.conf` (`/etc/` o `/etc/collectd/`) |
| MRTG (Multi Router Traffic Grapher) | Grafica tráfico de red, escrito en Perl, genera páginas HTML |
| Nagios (Core / XI) | Monitoriza sistemas, dispositivos de red y servicios; interfaz web; envía alertas por email/SMS; Core es gratuito |
| Icinga | Fork de Nagios con interfaz distinta |

### 4.5 Resolución de problemas de recursos

| Recurso | Comandos útiles | Notas |
|---|---|---|
| Memoria | `free`, `sar`, `vmstat` | La memoria se divide en páginas de 4 KB; si un proceso inactivo necesita liberar RAM, sus páginas se copian a la partición swap (swapping) |
| Procesos | `ps`, `pmap`, `pstree` | Un proceso bloqueado por I/O aparece en estado `D` (uninterruptible sleep); columna `b` de `vmstat` cuenta procesos en ese estado |
| CPU | `/proc/cpuinfo`, `lscpu`, `uptime`, `top`, `sar`, `mpstat` | Revisar núcleos, hyper-threading, caché, tiempo de espera, carga media |
| I/O de dispositivo | `iostat`, `iotop`, `lsof`, `sar` | Importante conocer el hardware subyacente (NAS, iSCSI, SAN, LVM, tipo de filesystem) |
| Red | `iftop`, `ip`, `iptraf`, `ntop`, `sar`, `lsof`, `tcpdump`, `ping`, `traceroute`, `ifconfig` | Para redes grandes, mejor usar soluciones completas como MRTG |

No hay diferencias relevantes entre Debian y Red Hat en esta sección: las herramientas de monitorización y sus comandos son los mismos en ambas familias.

---

## 5. Resumen de diferencias Debian vs Red Hat en este capítulo

| Aspecto | Debian | Red Hat |
|---|---|---|
| Instalar herramientas de compilación | `sudo apt-get install build-essential` | `yum groupinstall "Development Tools"` |
| Actualizar el sistema antes de compilar | `sudo apt-get update && sudo apt-get dist-upgrade` | `yum update` |
| Resto del capítulo (mensajería, backup, monitorización) | Igual | Igual |

---

*Documento generado a partir del Capítulo 2 ("Maintaining the System") del libro LPIC-2: Linux Professional Institute Certification Study Guide (Bresnahan & Blum).*
