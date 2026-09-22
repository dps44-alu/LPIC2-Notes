# LPIC-2 · Capítulo 3: Mastering the Kernel
### Equivalencias en FreeBSD 15

Este documento recoge, para cada concepto visto en el Capítulo 3 del libro LPIC-2 (componentes del kernel, compilación, parámetros en caliente, módulos y detección de hardware), cuál es su equivalente en **FreeBSD 15**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo: en Linux el kernel es un proyecto separado de las herramientas que lo acompañan. En FreeBSD, **kernel y sistema base se desarrollan juntos, en el mismo repositorio y con la misma versión**. Por eso el código fuente del kernel está dentro del código del sistema completo (`/usr/src`), se compila con el mismo sistema de `make` y no hace falta tocar el bootloader ni crear un initrd al instalar un kernel nuevo.

---

## 1. Partes del sistema FreeBSD y del kernel

Un sistema FreeBSD tiene dos grandes partes:
- **Sistema base**: kernel + herramientas de usuario (shell, `ls`, `ifconfig`, compilador...), todo mantenido por el proyecto FreeBSD. Es el equivalente a "kernel + utilidades GNU" de Linux.
- **Software de terceros**: entorno gráfico y aplicaciones, instalados con `pkg` o Ports en `/usr/local`.

### 1.1 Ficheros binarios del kernel (equivalente a la Tabla 3.2)

| Linux | FreeBSD 15 | Descripción |
|---|---|---|
| `bzImage` / `vmlinuz` | `/boot/kernel/kernel` | El binario del kernel (formato ELF). FreeBSD no usa nombres como `vmlinuz` ni `bzImage` |
| `vmlinux` (sin comprimir) | `/boot/kernel/kernel` | El kernel de FreeBSD normalmente se guarda sin comprimir |
| `vmlinuz-4.3.3` (versión en el nombre) | Carpetas distintas: `/boot/kernel/`, `/boot/kernel.old/`, `/boot/MIKERNEL/` | FreeBSD no pone la versión en el nombre del archivo: cada kernel vive en **su propia carpeta**, junto con sus módulos |
| `System.map` | `/usr/lib/debug/boot/kernel/kernel.debug` | Símbolos de depuración del kernel (se usan para analizar fallos) |

### 1.2 Otras partes del kernel

| Elemento | Linux | FreeBSD 15 |
|---|---|---|
| Módulos del kernel (`.ko`) | `/lib/modules/versión/` | `/boot/kernel/` (en la **misma carpeta** que el kernel) |
| Módulos de terceros (paquetes) | `/lib/modules/.../extra` | `/boot/modules/` (ej. el controlador gráfico `drm-kmod`) |
| Código fuente del kernel | `/usr/src/linux` o `/usr/src/kernels` | `/usr/src/sys/` (dentro del código de todo el sistema, `/usr/src`) |
| Archivos de configuración del kernel | `/usr/src/linux/.config` | `/usr/src/sys/amd64/conf/` (ej. `GENERIC`) |
| Parches | Comando `patch` | `freebsd-update` (parches binarios) o `patch`/`git` sobre `/usr/src` |
| Cabeceras (headers) | `/usr/src/linux` (paquete aparte) | `/usr/include/` (ya instaladas con el sistema base; las del kernel en `/usr/include/sys/`) |
| Documentación | `/usr/src/linux/Documentation` | Páginas de manual: **sección 4** (controladores, ej. `man 4 em`) y **sección 9** (programación del kernel) |

### 1.3 Módulos del kernel

El concepto es el mismo que en Linux: controladores que se cargan en tiempo de ejecución, con extensión `.ko`, para no tener que meterlo todo dentro del kernel. El kernel por defecto (`GENERIC`) ya incluye los controladores más comunes, y el resto se cargan como módulos.

Igual que en Linux, algunos fabricantes distribuyen solo el módulo binario (ej. el controlador de NVIDIA, disponible como paquete).

### 1.4 Versionado de FreeBSD

En FreeBSD no se versiona el kernel por separado: **la versión es la del sistema completo**.

| Rama / tipo | Ejemplo | Significado |
|---|---|---|
| RELEASE | `15.1-RELEASE` | Versión estable publicada oficialmente (para producción) |
| RELEASE con parches | `15.1-RELEASE-p3` | La misma versión con parches de seguridad aplicados (`-pN`) |
| STABLE | `15.1-STABLE` | Rama de mantenimiento: recibe correcciones entre versiones, sin cambios bruscos |
| CURRENT | `16.0-CURRENT` | Rama de desarrollo (equivale a `mainline`/`-rc` en Linux). No usar en producción |

Número mayor (`15`) = versión principal, con cambios importantes. Número menor (`.1`) = versión de mantenimiento de esa serie.

Ramas en el repositorio de código:

| Rama git | Contenido |
|---|---|
| `main` | CURRENT (desarrollo) |
| `stable/15` | 15-STABLE |
| `releng/15.1` | 15.1-RELEASE y sus parches de seguridad |

**Comandos para ver la versión:**

| Comando | Función |
|---|---|
| `freebsd-version -k` | Versión del kernel **instalado** |
| `freebsd-version -r` | Versión del kernel **en ejecución** |
| `freebsd-version -u` | Versión del sistema base (userland) |
| `uname -a` / `uname -r` | Igual que en Linux |

Es normal que kernel y userland tengan distinto número de parche (`-p`): un parche de seguridad puede afectar solo a uno de los dos.

---

## 2. Compilación de un nuevo kernel

### 2.1 Obtener el código fuente

| Fuente | Detalles |
|---|---|
| Durante la instalación | El instalador permite marcar el componente **src**, que deja el código en `/usr/src` |
| Repositorio oficial (git) | `git clone -b releng/15.1 --depth 1 https://git.FreeBSD.org/src.git /usr/src` (requiere `pkg install git`) |
| Actualizar el código ya descargado | `git -C /usr/src pull` |

A diferencia de Linux, no se descarga un tarball del kernel aparte: se descarga el código de **todo el sistema**, y el kernel está en `/usr/src/sys/`. Importante: la rama descargada debe coincidir con la versión del sistema (ej. `releng/15.1` para un 15.1-RELEASE).

### 2.2 Crear el fichero de configuración

FreeBSD **no tiene `make menuconfig`** ni menús gráficos. La configuración del kernel es un **archivo de texto** que se edita a mano.

| Linux | FreeBSD 15 |
|---|---|
| `/usr/src/linux/.config` | `/usr/src/sys/amd64/conf/NOMBRE` (un archivo por cada kernel; `amd64` es la arquitectura) |
| `make defconfig` | Usar el archivo `GENERIC` (el kernel que trae el sistema) |
| `make oldconfig` | No hace falta: si el archivo propio incluye `GENERIC`, hereda automáticamente sus cambios |
| `make menuconfig` / `xconfig` / `gconfig` | No existen. Se edita el archivo con un editor de texto (`ee`, `vi`) |
| `make mrproper` / `make clean` | No hace falta: `make buildkernel` limpia automáticamente la compilación anterior. Para empezar totalmente de cero se puede borrar la carpeta de compilación en `/usr/obj` |

Forma recomendada de crear un kernel propio: un archivo corto que **incluye `GENERIC`** y solo indica los cambios.

```
cd /usr/src/sys/amd64/conf
ee MIKERNEL
```

Contenido de ejemplo:
```
include GENERIC
ident   MIKERNEL

nodevice    bluetooth
options     IPSEC_SUPPORT
device      pf
```

| Palabra clave | Función |
|---|---|
| `include GENERIC` | Parte de la configuración por defecto |
| `ident NOMBRE` | Nombre del kernel (aparece en `uname -a`) |
| `device nombre` | Incluye un controlador dentro del kernel |
| `nodevice nombre` | Quita un controlador que venía en `GENERIC` |
| `options NOMBRE` | Activa una opción del kernel |
| `nooptions NOMBRE` | Quita una opción que venía en `GENERIC` |
| `makeoptions` | Opciones de compilación (ej. `DEBUG=-g`) |

Otros archivos útiles:
- `/usr/src/sys/amd64/conf/NOTES` y `/usr/src/sys/conf/NOTES`: lista comentada de **todas** las opciones y dispositivos posibles.
- `/etc/make.conf` y `/etc/src.conf`: opciones generales de compilación. Por ejemplo, `KERNCONF=MIKERNEL` en `/etc/make.conf` evita tener que escribirlo en cada comando.

**Ver la configuración del kernel en ejecución:** `sysctl -n kern.conftxt` (equivale a consultar el `.config` del kernel actual).

No hacen falta paquetes previos: el compilador y las herramientas ya vienen en el sistema base.

### 2.3 Compilar el kernel

Se ejecuta siempre desde `/usr/src` (no desde `/usr/src/sys`):

| Comando | Función |
|---|---|
| `cd /usr/src` | Carpeta desde la que se compila |
| `make buildkernel KERNCONF=MIKERNEL` | Compila el kernel **y todos sus módulos** |
| `make -j4 buildkernel KERNCONF=MIKERNEL` | Igual, usando 4 procesos en paralelo (más rápido) |
| `make -j$(sysctl -n hw.ncpu) buildkernel KERNCONF=MIKERNEL` | Usa tantos procesos como núcleos tenga la CPU |

Si no se indica `KERNCONF`, se compila `GENERIC`. El resultado queda en `/usr/obj/usr/src/amd64.amd64/sys/MIKERNEL/`.

Para compilar más rápido, se puede limitar qué módulos se compilan con `MODULES_OVERRIDE="zfs linux64"` en `/etc/make.conf`.

### 2.4 Instalar el kernel compilado

| Comando | Función |
|---|---|
| `make installkernel KERNCONF=MIKERNEL` | Mueve el kernel actual a `/boot/kernel.old/` e instala el nuevo en `/boot/kernel/` |
| `make kernel KERNCONF=MIKERNEL` | Hace `buildkernel` + `installkernel` de una vez |
| `make installkernel KERNCONF=MIKERNEL INSTKERNNAME=prueba` | Instala el kernel en `/boot/prueba/` sin tocar el actual (para probarlo) |
| `shutdown -r now` | Reiniciar para usar el nuevo kernel |

Diferencias clave con Linux:
- **No hay que actualizar el bootloader**: el `loader` siempre arranca `/boot/kernel/kernel` (Capítulo 1).
- El kernel anterior se conserva automáticamente en `/boot/kernel.old/`. Si el nuevo falla, se elige en el menú del loader (opción `5`) o con `boot kernel.old`.
- Para probar un kernel una sola vez: instalarlo con `INSTKERNNAME=prueba` y ejecutar `nextboot -k prueba`.

### 2.5 Compilar e instalar los módulos

| Linux | FreeBSD 15 |
|---|---|
| `make modules` | Incluido en `make buildkernel` |
| `make modules_install` | Incluido en `make installkernel` (los módulos van a la misma carpeta que el kernel) |
| Módulos de terceros | Se instalan con `pkg` o Ports y van a `/boot/modules/` |

Cuidado: los módulos de terceros deben coincidir con la versión del kernel. Tras actualizar el sistema a una versión nueva, hay que actualizarlos también (`pkg upgrade`).

### 2.6 Disco RAM inicial (initrd / initramfs)

**FreeBSD no necesita initrd ni initramfs.** En Linux sirven para cargar los módulos necesarios para leer el disco (RAID, un sistema de archivos concreto, cifrado) antes de montar la raíz. En FreeBSD ese trabajo lo hace directamente el **`loader`**: puede leer UFS y ZFS por sí mismo y carga los módulos antes de arrancar el kernel.

| Necesidad en Linux | Solución en FreeBSD 15 (en `/boot/loader.conf`) |
|---|---|
| `mkinitrd` / `mkinitramfs` con módulos de RAID | `geom_mirror_load="YES"` (RAID1 por software), `geom_stripe_load="YES"` (RAID0) |
| Raíz en ZFS | `zfs_load="YES"` |
| Raíz cifrada | `geom_eli_load="YES"` |
| Otro módulo necesario al arrancar | `nombre_load="YES"` |

El caso más parecido a un initrd es la opción **`mfsroot`**: una imagen de sistema de archivos que el loader carga en memoria y usa como raíz (se usa en sistemas de instalación o de rescate, no en el uso normal).

### 2.7 Parches del kernel

| Método | Comandos | Cuándo usarlo |
|---|---|---|
| Parches binarios | `freebsd-update fetch` + `freebsd-update install` | Método habitual. **Solo actualiza el kernel `GENERIC`**: si se usa un kernel propio, hay que recompilarlo |
| Actualizar el código con git | `git -C /usr/src pull` y volver a compilar (`buildkernel` + `installkernel`) | Con kernel propio |
| Parche de un aviso de seguridad | `cd /usr/src && patch < archivo.patch` y volver a compilar | Aplicar un parche concreto publicado en un aviso de seguridad |
| Deshacer un parche | `patch -R < archivo.patch` | Igual que en Linux |

Con pkgbase (sistema base instalado como paquetes, opción nueva en FreeBSD 15), el kernel se actualiza con `pkg upgrade`.

---

## 3. Consulta y modificación de parámetros del kernel en caliente

### 3.1 El sistema de ficheros `/proc`

En FreeBSD **`/proc` no se monta por defecto** y no se usa para configurar el kernel. Toda esa información se consulta con `sysctl`.

| Linux (`/proc`) | FreeBSD 15 |
|---|---|
| `/proc/sys/kernel/` | Árbol de variables de `sysctl` (ej. `kern.*`) |
| `cat /proc/sys/kernel/version` | `sysctl kern.version` |
| `/proc/cpuinfo` | `sysctl hw.model hw.ncpu` o `grep CPU /var/run/dmesg.boot` |
| `/proc/dma`, `/proc/ioports`, `/proc/interrupts` | `devinfo -u` (recursos usados: puertos, memoria, IRQ, DMA) y `vmstat -i` (interrupciones) |
| `/proc` (información de procesos) | `procstat`, `ps` |

Si algún programa necesita `/proc`, se puede montar: `mount -t procfs proc /proc` (o `linprocfs` en `/compat/linux/proc` para programas de Linux).

### 3.2 Comando `sysctl`

El comando existe con el mismo nombre y la misma idea, con pequeñas diferencias de sintaxis:

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `sysctl nombre.parametro` | `sysctl nombre.parametro` | Muestra el valor |
| `sysctl -w nombre=valor` | `sysctl nombre=valor` (**sin `-w`**) | Cambia el valor en caliente |
| `sysctl -a` | `sysctl -a` | Muestra todos los parámetros |
| — | `sysctl -d nombre` | Muestra la **descripción** del parámetro (muy útil) |
| — | `sysctl -W` | Lista solo los parámetros modificables |
| — | `sysctl -T` | Lista solo los parámetros que se fijan al arrancar (tunables) |
| `/etc/sysctl.conf` | `/etc/sysctl.conf` (igual) y `/etc/sysctl.conf.local` | Parámetros aplicados en cada arranque |
| `/etc/sysctl.d/` | `/etc/sysctl.kld.d/` | Parámetros que dependen de un módulo: se aplican al cargar ese módulo |
| `sysctl -p` (recargar) | `service sysctl restart` | Vuelve a aplicar `/etc/sysctl.conf` |

Ejemplo:
```
sysctl net.inet.ip.forwarding=1
echo 'net.inet.ip.forwarding=1' >> /etc/sysctl.conf
```

**Tipos de parámetros en FreeBSD:**
- **Parámetros normales**: se cambian en caliente con `sysctl` y se guardan en `/etc/sysctl.conf`.
- **Tunables** (solo lectura en caliente): se tienen que fijar **antes de arrancar el kernel**, en `/boot/loader.conf` (ej. `kern.maxfiles`). Si se intenta cambiarlos con `sysctl`, da error. El comando `kenv` muestra las variables que el loader pasó al kernel.

---

## 4. Gestión de módulos en tiempo de ejecución

Los comandos cambian de nombre: en FreeBSD empiezan por **`kld`** (Kernel LoaDable).

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `lsmod` | `kldstat` | Lista los módulos cargados |
| `lsmod` (con detalle) | `kldstat -v` | Muestra también qué submódulos contiene cada archivo `.ko` |
| `modinfo modulo` | `kldstat -v -n modulo` o `man 4 modulo` | Información de un módulo (FreeBSD no tiene `modinfo`; la documentación del controlador está en su página de manual) |
| `insmod /ruta/modulo.ko` | `kldload /ruta/modulo.ko` | Carga un módulo desde una ruta concreta |
| `modprobe modulo` | `kldload modulo` | Carga un módulo por su nombre (lo busca solo y carga sus dependencias) |
| `rmmod modulo` / `modprobe -r` | `kldunload modulo` | Descarga un módulo |
| `depmod` | `kldxref /boot/kernel` | Regenera el índice de módulos (`linker.hints`); normalmente se hace solo |
| — | `kldconfig -r` | Muestra las carpetas donde se buscan los módulos |

**Opciones de `kldload` y `kldunload` (equivalente a la Tabla 3.5):**

| Opción `modprobe` | Equivalente FreeBSD | Función |
|---|---|---|
| `-v` | `kldload -v` | Modo detallado |
| — | `kldload -n` | No da error si el módulo ya está cargado |
| — | `kldload -q` | Modo silencioso |
| `-f` | `kldunload -f` | Fuerza la descarga de un módulo |
| `-r` | `kldunload` | Descarga el módulo |
| `-b` (lista negra) | `module_blacklist="modulo1 modulo2"` en `/boot/loader.conf` | Impide que se carguen ciertos módulos |
| `-n` (simulación) | No existe | — |
| `-c` (ver configuración) | `kldconfig -r`, `cat /boot/loader.conf` | — |

**Cargar módulos automáticamente al arrancar** (equivalente a `/etc/modules` o `/etc/modules-load.d/` de Linux):

| Método | Ejemplo | Cuándo se carga |
|---|---|---|
| `/boot/loader.conf` | `if_bridge_load="YES"` | Muy pronto, lo carga el loader antes que el kernel. Necesario para módulos imprescindibles para arrancar (discos, ZFS) |
| `/etc/rc.conf` | `sysrc kld_list+="if_bridge"` | Más tarde, lo cargan los scripts rc. Es el método **recomendado** para lo demás (hace el arranque más rápido) |

---

## 5. Detección de hardware

### 5.1 Coldplug vs Hotplug

Los conceptos son exactamente los mismos. FreeBSD soporta hotplug de USB, red, discos SATA/SAS/NVMe (si la controladora lo permite), etc.

### 5.2 El equivalente a `udev`: `devd` y `devfs`

En FreeBSD el trabajo de `udev` se reparte entre dos piezas:
- **`devfs`**: sistema de archivos de `/dev`. El **kernel** crea y borra los archivos de dispositivo automáticamente (no hace falta un demonio para esto).
- **`devd`**: demonio que escucha los eventos de hardware y ejecuta acciones (cargar un módulo, configurar la red al conectar un cable, etc.).

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `udevd` | `devd` | Demonio que recibe los eventos de hardware del kernel |
| `/etc/udev/udev.conf` | `/etc/devd.conf` | Configuración principal del demonio y reglas base |
| `/etc/udev/rules.d/` | `/etc/devd/` y `/usr/local/etc/devd/` | Reglas propias (archivos `.conf`) |
| `/lib/udev/rules.d/` | `/etc/devd/` (reglas del sistema base) | Reglas que trae el sistema |
| Permisos y enlaces de `/dev` | `/etc/devfs.conf` (al arrancar) y `/etc/devfs.rules` (reglas de permisos) | Cambiar propietario, permisos o crear enlaces de dispositivos |
| `udevadm monitor` | `cat /var/run/devd.pipe` | Ver en directo los eventos de hardware |
| — | `devd -d` | Ejecuta `devd` en primer plano mostrando lo que hace (depuración; parar antes el servicio) |
| Carga automática de controladores | `devmatch` | Detecta el hardware y carga el módulo que corresponda (activo por defecto) |

Ejemplo de regla de `devd` (ejecutar un script al conectar un dispositivo USB):
```
notify 100 {
    match "system"   "USB";
    match "type"     "ATTACH";
    action "/usr/local/bin/aviso_usb.sh";
};
```

**Nombres de dispositivo:** en FreeBSD el nombre lo pone el propio controlador más un número (`em0`, `ada0`, `da1`) y no se cambia con reglas como en `udev`. Para tener nombres fijos:
- Interfaces de red: renombrar en `/etc/rc.conf`, ej. `ifconfig_em0_name="lan0"`.
- Discos: usar **etiquetas** en lugar del nombre del disco (`/dev/gpt/etiqueta`, `/dev/diskid/...`), que no cambian aunque se muevan los discos.

### 5.3 Comandos para consultar hardware

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `lspci` | `pciconf -lv` | Lista los dispositivos PCI/PCIe con fabricante y modelo |
| `lsusb` | `usbconfig` (o `usbconfig list`) | Lista los dispositivos USB |
| — | `devinfo -rv` | Árbol de todos los dispositivos detectados y su controlador |
| — | `camcontrol devlist` | Lista los discos y unidades SATA/SAS/SCSI/USB |
| — | `nvmecontrol devlist` | Lista los discos NVMe |
| — | `geom disk list` | Información detallada de todos los discos |
| `dmidecode` | `dmidecode` (paquete) o `kenv \| grep smbios` | Información de la placa base y la BIOS |

`lsusb` y `lspci` de Linux también se pueden instalar como paquetes (`usbutils` y `pciutils`), pero lo normal es usar las herramientas propias.

**Opciones de `pciconf`:**

| Opción | Función |
|---|---|
| `-l` | Lista los dispositivos PCI |
| `-v` | Añade el nombre del fabricante y del dispositivo |
| `-c` | Muestra las capacidades del dispositivo |
| `-b` | Muestra las zonas de memoria y puertos que usa |

**Opciones de `usbconfig` (equivalente a la Tabla 3.7):**

| Opción `lsusb` | Equivalente `usbconfig` | Función |
|---|---|---|
| (sin opciones) | `usbconfig list` | Lista los dispositivos USB |
| `-s bus:dispositivo` | `usbconfig -d ugen0.2 ...` | Actúa sobre un dispositivo concreto (`ugen` bus.dirección) |
| `-s bus` | `usbconfig -u 0 ...` | Solo los dispositivos de un bus |
| `-v` | `usbconfig -d ugen0.2 dump_device_desc` | Información detallada del dispositivo |
| `-t` (árbol) | `usbconfig show_ifdrv` | Muestra qué controlador usa cada interfaz del dispositivo |
| `-d fabricante` | No hay filtro directo (usar `usbconfig list \| grep`) | — |

---

## 6. Resolución de problemas del kernel

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `dmesg` | `dmesg` (igual) | Mensajes recientes del kernel |
| `/var/log/dmesg` | `/var/run/dmesg.boot` | Mensajes del kernel del último arranque |
| `/var/log/messages`, `/var/log/syslog` | `/var/log/messages` | Log general del sistema |
| `/var/log/boot` (Debian), `/var/log/boot.log` (Red Hat) | No existe por defecto | Se pueden guardar los mensajes de consola en `/var/log/console.log` activando la línea correspondiente en `/etc/syslog.conf` |

**Cuando el kernel falla (kernel panic)**, FreeBSD puede guardar una copia de la memoria para analizarla:

| Elemento | Función |
|---|---|
| `dumpdev="AUTO"` en `/etc/rc.conf` | Guarda el volcado de memoria en la swap cuando hay un panic (activo por defecto) |
| `savecore` | Al siguiente arranque copia el volcado a **`/var/crash/`** (lo hace automáticamente) |
| `crashinfo` | Genera un informe de texto (`/var/crash/core.txt.N`) con el resumen del fallo |
| `kgdb` | Depurador para analizar el volcado a fondo (paquete `gdb`) |
| DDB | Depurador interno del kernel: se puede entrar al arrancar con `boot -d` |
| DTrace | Herramienta del sistema base para rastrear en detalle qué hace el kernel en ejecución |

Para más detalle durante el arranque, usar el modo verbose (`boot -v` o `boot_verbose="YES"` en `/boot/loader.conf`, Capítulo 1).

---

## 7. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 3) | Equivalente en FreeBSD 15 |
|---|---|
| `/boot/vmlinuz-versión` | `/boot/kernel/kernel` (cada kernel en su carpeta) |
| `/lib/modules/versión/` | `/boot/kernel/` (sistema) y `/boot/modules/` (terceros) |
| `/usr/src/linux` | `/usr/src/sys` (dentro del código completo del sistema) |
| kernel.org / versiones del kernel | Versión del sistema completo: RELEASE, STABLE, CURRENT |
| `uname -r` | `uname -r` y `freebsd-version -kru` |
| `.config` | Archivo de texto en `/usr/src/sys/amd64/conf/` (basado en `GENERIC`) |
| `make menuconfig` | Editar el archivo a mano (`include GENERIC` + cambios) |
| `make bzImage` + `make modules` | `make buildkernel KERNCONF=NOMBRE` |
| `make install` + `make modules_install` | `make installkernel KERNCONF=NOMBRE` |
| Actualizar GRUB tras instalar | No hace falta |
| `mkinitrd` / `mkinitramfs` | No hace falta: el loader carga los módulos (`nombre_load="YES"`) |
| Parches con `patch` | `freebsd-update` (kernel GENERIC) o `git`/`patch` + recompilar |
| `/proc/sys/`, `/proc/cpuinfo` | `sysctl` |
| `sysctl -w nombre=valor` | `sysctl nombre=valor` |
| `/etc/sysctl.conf` | `/etc/sysctl.conf` (y tunables en `/boot/loader.conf`) |
| `lsmod` | `kldstat` |
| `insmod` / `modprobe` | `kldload` |
| `rmmod` / `modprobe -r` | `kldunload` |
| `modinfo` | `kldstat -v -n` / `man 4 nombre` |
| `/etc/modules` | `kld_list` en `/etc/rc.conf` o `nombre_load` en `/boot/loader.conf` |
| `udev` / `udevd` | `devd` + `devfs` (+ `devmatch`) |
| `/etc/udev/rules.d/` | `/etc/devd/`, `/usr/local/etc/devd/`, `/etc/devfs.rules` |
| `udevadm monitor` | `cat /var/run/devd.pipe` |
| `lspci` | `pciconf -lv` |
| `lsusb` | `usbconfig` |
| `/var/log/boot.log` | `/var/run/dmesg.boot` y `/var/log/messages` |

---

*Documento elaborado como contrapartida en FreeBSD 15 al Capítulo 3 ("Mastering the Kernel") del libro LPIC-2. Fuentes: FreeBSD Handbook (capítulos "Configuring the FreeBSD Kernel", "Configuration and Tuning" y "Updating and Upgrading FreeBSD") y páginas de manual de FreeBSD: config(8), build(7), kldload(8), kldstat(8), kldunload(8), kldxref(8), sysctl(8), sysctl.conf(5), loader.conf(5), devd(8), devd.conf(5), devfs.conf(5), devmatch(8), pciconf(8), usbconfig(8), freebsd-version(1), savecore(8), crashinfo(8).*
