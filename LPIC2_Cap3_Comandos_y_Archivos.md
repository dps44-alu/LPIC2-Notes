# LPIC-2 · Capítulo 3: Mastering the Kernel
### Recopilación de comandos y archivos de configuración

Objetivos del examen cubiertos: 201.1 (Kernel Components), 201.2 (Compiling a Linux kernel), 201.3 (Kernel runtime management and troubleshooting)

---

## 1. Partes del sistema Linux y del kernel

Un sistema Linux completo tiene cuatro partes: el kernel, las utilidades GNU, un entorno de escritorio gráfico y el software de aplicaciones.

### 1.1 Ficheros binarios del kernel (Tabla 3.2)

| Nombre de archivo | Descripción |
|---|---|
| `bzImage` | Binario del kernel comprimido con gzip (el más grande y el más usado) |
| `zImage` | Binario del kernel comprimido con gzip (versión más pequeña) |
| `vmlinux` | Binario sin comprimir, no se usa normalmente para arrancar |
| `vmlinuz` | Nombre genérico del binario comprimido (muchas distros renombran `bzImage` a este nombre) |
| `kernel` | Nombre genérico de un binario sin comprimir |

Ubicación habitual: carpeta `/boot`, con el número de versión añadido al nombre (ej. `vmlinuz-4.3.3`), lo que permite mantener varias versiones instaladas a la vez.

### 1.2 Otras partes del kernel

| Elemento | Descripción | Ubicación típica |
|---|---|---|
| Módulos del kernel | Controladores de dispositivo enlazables en tiempo de ejecución (extensión `.ko`) | `/lib/modules/` |
| Código fuente del kernel | Necesario para recompilar o personalizar el kernel | `/usr/src/linux` (Debian) · `/usr/src/kernels` (Red Hat) |
| Parches del kernel (patches) | Actualizaciones incrementales de versión (ej. 4.3.0 → 4.3.1) | Se aplican con el comando `patch` |
| Cabeceras del kernel (headers) | Ficheros `.h`, necesarios para compilar solo módulos (no hace falta el código fuente completo) | `/usr/src/linux` (Debian) · `/usr/src/kernels` (Red Hat) |
| Documentación del kernel | Explicación de cada parte del código fuente | `/usr/src/linux/Documentation` |

### 1.3 Módulos del kernel

Los módulos evitan tener que compilar todos los controladores dentro del kernel (lo que lo haría enorme). Se distribuyen como código fuente (a compilar) o como ficheros binarios `.ko` ya listos para enlazar.

Nota: algunos fabricantes distribuyen solo el `.ko` sin el código fuente para proteger su propiedad intelectual.

### 1.4 Versionado del kernel (resumen histórico)

| Serie | Formato | Notas |
|---|---|---|
| 0.x | `0.xx` | Primeras versiones de prueba (1991-1992) |
| 1.x | `1.x.y` | `x` impar = desarrollo, `x` par = producción (1994-1995) |
| 2.x | `2.x.y` | Mismo esquema impar/par hasta la versión 2.4 (1996-2001) |
| 2.6.x | `2.6.x.y` | Todas las versiones "2.6" son de producción; el desarrollo se marca con sufijo `-rc` (2003-2011) |
| 3.x | `3.x.y[.z]` | Mismo esquema que 2.6, con un cuarto dígito opcional `.z` para parches urgentes de seguridad (2011-2015) |
| 4.x y posteriores | Igual que la serie 3.x | — |

---

## 2. Compilación de un nuevo kernel

### 2.1 Obtener el código fuente

| Fuente | Detalles |
|---|---|
| www.kernel.org | Repositorio oficial; versiones `stable` (última de producción), `longterm` (versiones anteriores mantenidas) y `mainline` (última de desarrollo) |
| Repositorio de la distribución | Más seguro, ya probado en ese entorno concreto, pero puede ir por detrás de la última versión |

Descarga y extracción:
```
tar xvf linux-version.tar.xz
```

Ubicación recomendada: `/usr/src/linux-version/`, con un enlace a `/usr/src/linux`:
```
ln /usr/src/linux-4.3.3 /usr/src/linux
```
Nota: en distribuciones Red Hat se usa `/usr/src/kernels` en lugar de `/usr/src/linux`.

### 2.2 Crear el fichero de configuración

El fichero de configuración es `/usr/src/linux/.config`. Se genera/actualiza mediante distintos "targets" del comando `make`:

| Target | Función |
|---|---|
| `make config` | Preguntas de texto, una por cada opción (lento, poco práctico) |
| `make defconfig` | Configuración por defecto según el sistema detectado |
| `make oldconfig` | Actualiza una configuración existente solo con las opciones nuevas |
| `make menuconfig` | Menú en modo texto |
| `make xconfig` | Menú gráfico (Qt, GNOME/KDE) |
| `make gconfig` | Menú gráfico (librerías GTK/GNOME) |
| `make mrproper` | Borra toda la configuración anterior y los ficheros objeto compilados (empezar de cero) |
| `make clean` | Elimina ficheros objeto de una compilación anterior |

Prerrequisitos según distribución (para usar los menús gráficos, por ejemplo):

| Distribución | Comando |
|---|---|
| Red Hat (RPM) | `sudo yum groupinstall "Development Tools"`, `sudo yum install qt-devel` |
| Debian (DEB) | `sudo apt-get install pkg-config g++`, `sudo apt-get install libqt4-dev` |

### 2.3 Compilar el kernel

| Comando | Función |
|---|---|
| `make clean` | Elimina ficheros objeto de compilaciones anteriores |
| `make` | Genera un binario del kernel sin comprimir |
| `make bzImage` | Genera un binario del kernel comprimido (el formato más habitual) |

Durante la compilación se ven líneas `CC` (ficheros objeto creándose) y `LD` (enlazado del ejecutable final). El binario resultante queda en `/usr/src/linux/arch/x86/boot/bzImage`.

### 2.4 Instalar el kernel compilado

| Paso | Comando |
|---|---|
| Copiar el binario a `/boot` con su versión | `cp /usr/src/linux/arch/x86/boot/bzImage /boot/vmlinuz-4.3.3` |
| Copiar el `System.map` (útil para depuración) | `cp /usr/src/linux/System.map /boot/System.map-4.3.3` |
| Alternativa automática (copia binario y System.map) | `make install` |

Después hay que actualizar el bootloader: en GRUB Legacy, añadir manualmente una entrada en `menu.lst`/`grub.conf`; en GRUB2, ejecutar `update-grub` (ver Capítulo 1).

### 2.5 Compilar e instalar los módulos

| Comando | Función |
|---|---|
| `make modules` | Compila los ficheros objeto de los módulos |
| `make modules_install` | Instala los módulos en `/lib/modules/kernel-version/` |

### 2.6 Crear el disco RAM inicial (initrd / initramfs)

Necesario cuando el propio kernel necesita módulos (ej. para RAID o un sistema de ficheros concreto) antes de poder leer el disco real.

| Distribución | Comando | Sintaxis |
|---|---|---|
| **Red Hat** | `mkinitrd` | `mkinitrd outputfile version` (ej. `mkinitrd /boot/initrd.img-4.3.3 4.3.3`) |
| **Debian** | `mkinitramfs` | `mkinitramfs -o outputfile version` (ej. `mkinitramfs -o /boot/initramfs-4.3.3.img 4.3.3`) |

Opciones destacadas de `mkinitrd` (Tabla 3.3): `--force` (sobrescribir), `--omit-lvmmodules`, `--omit-raidmodules`, `--omit-scsimodules`, `--verbose`.

Opciones destacadas de `mkinitramfs` (Tabla 3.4): `-o outputfile`, `-c` (comprimir), `-r root`, `-k` (conservar directorio temporal).

Estos ficheros se guardan normalmente en `/boot`, junto al binario del kernel.

### 2.7 Parches del kernel

- Un parche (patch) contiene solo los cambios entre una versión y la siguiente incremental.
- Se aplica con el comando `patch` sobre el código fuente ya existente.
- Para desinstalar un parche: `patch -R`.
- Para aplicar varios parches sucesivos hay que hacerlo en orden (desde la versión base).

---

## 3. Consulta y modificación de parámetros del kernel en caliente

### 3.1 El sistema de ficheros `/proc`

`/proc` es un sistema de ficheros virtual (dinámico) que expone información y estadísticas del kernel en tiempo real.

| Elemento | Función |
|---|---|
| `/proc/sys/kernel/` | Parámetros del kernel modificables en caliente |
| `/proc/sys/kernel/version` | Ejemplo: versión del kernel (`cat /proc/sys/kernel/version`) |
| `/proc/cpuinfo` | Información de la(s) CPU |
| `/proc/dma`, `/proc/ioports` | Información de canales DMA y puertos de E/S |

### 3.2 Comando `sysctl`

| Comando | Función |
|---|---|
| `sysctl nombre.parametro` | Muestra el valor actual de un parámetro |
| `sysctl -w nombre.parametro=valor` | Cambia un parámetro en caliente |
| `/etc/sysctl.conf` | Fichero con parámetros a aplicar automáticamente al ejecutar `sysctl` |
| `/etc/sysctl.d/` | Carpeta con varios ficheros de parámetros, para organizarlos por aplicación |

Muchos parámetros son booleanos (0 = desactivado, 1 = activado), aunque no todos (ej. `scsi_logging_level`, mejor gestionado con la utilidad `scsi_logging_level` del paquete `sg3-utils`/`sg3_utils`).

---

## 4. Gestión de módulos en tiempo de ejecución

| Comando | Función |
|---|---|
| `lsmod` | Lista los módulos cargados actualmente, su tamaño y qué otros módulos dependen de ellos |
| `modinfo nombre_modulo` | Muestra información detallada de un módulo concreto |
| `insmod ruta_al_modulo.ko` | Inserta un módulo en el kernel (requiere conocer el nombre exacto del fichero) |
| `modprobe nombre_modulo` | Inserta un módulo por su nombre (más cómodo que `insmod`, ya busca el fichero) |
| `modprobe -r nombre_modulo` | Elimina un módulo (equivale a usar `rmmod`) |
| `rmmod nombre_modulo` | Elimina un módulo por su nombre |

**Opciones destacadas de `modprobe` (Tabla 3.5):**

| Opción | Función |
|---|---|
| `-a` | Inserta todos los módulos indicados |
| `-b` | Aplica la lista negra definida en la configuración |
| `-c` | Muestra la configuración actual |
| `-d` | Indica el directorio raíz para instalar módulos (por defecto `/`) |
| `-f` | Fuerza la instalación aunque haya problemas de versión |
| `-n` | Simulación (dry run), no instala realmente |
| `-r` | Elimina el módulo indicado |
| `-s` | Envía mensajes de error al syslog |
| `-v` | Modo detallado (verbose) |

---

## 5. Detección de hardware

### 5.1 Coldplug vs Hotplug

| Tipo | Descripción | Ejemplos |
|---|---|---|
| Coldplug | Solo se puede conectar/desconectar con el sistema apagado | Memoria RAM, tarjetas PCI, discos internos |
| Hotplug | Se puede conectar/desconectar con el sistema encendido | USB, red, monitores |

### 5.2 El sistema `udev`

| Elemento | Función |
|---|---|
| `udevd` | Demonio en segundo plano que escucha los eventos de hardware que envía el kernel |
| `/etc/udev/udev.conf` | Configuración general del propio demonio `udevd` |
| `/etc/udev/rules.d/`, `/lib/udev/rules.d/` | Reglas que indican qué módulo cargar y qué nombre de dispositivo asignar cuando ocurre un evento |
| `udevadm` / `udevmonitor` | Permiten monitorizar en directo los eventos de hardware detectados (útil para depurar) |

### 5.3 Comandos para consultar hardware

| Comando | Función |
|---|---|
| `lspci` | Lista los dispositivos conectados por PCI/PCIe |
| `lsusb` | Lista los dispositivos conectados por USB |

**Opciones de `lsusb` (Tabla 3.7):**

| Opción | Función |
|---|---|
| `-d` | Solo dispositivos de un fabricante (vendor ID) concreto |
| `-D` | Solo el dispositivo con el fichero de dispositivo indicado |
| `-s` | Solo dispositivos de un bus concreto |
| `-t` | Muestra en formato árbol (dependencias) |
| `-v` | Modo detallado (verbose) |
| `-V` | Muestra la versión del programa |

---

## 6. Resolución de problemas del kernel

| Comando/archivo | Función |
|---|---|
| `dmesg` | Muestra los mensajes recientes del kernel guardados en el kernel ring buffer (ver también Capítulo 1) |
| `/var/log/dmesg`, `/var/log/messages`, `/var/log/syslog` | Ficheros donde se guardan los mensajes del kernel una vez que salen del buffer circular |
| `/var/log/boot` | Log de arranque en sistemas **Debian** |
| `/var/log/boot.log` | Log de arranque en sistemas **Red Hat** |

---

## 7. Resumen de diferencias Debian vs Red Hat en este capítulo

| Aspecto | Debian | Red Hat |
|---|---|---|
| Ubicación del código fuente/cabeceras del kernel | `/usr/src/linux` | `/usr/src/kernels` |
| Herramientas de compilación previas | `sudo apt-get install pkg-config g++`, `sudo apt-get install libqt4-dev` | `sudo yum groupinstall "Development Tools"`, `sudo yum install qt-devel` |
| Comando para crear el disco RAM inicial | `mkinitramfs` | `mkinitrd` |
| Fichero de log de arranque | `/var/log/boot` | `/var/log/boot.log` |
| Resto del capítulo (módulos, `/proc`, `sysctl`, `udev`, `lspci`/`lsusb`) | Igual | Igual |

---

*Documento generado a partir del Capítulo 3 ("Mastering the Kernel") del libro LPIC-2: Linux Professional Institute Certification Study Guide (Bresnahan & Blum).*
