# LPIC-2 · Capítulo 1: Starting a System
### Equivalencias en FreeBSD 15

Este documento recoge, para cada concepto visto en el Capítulo 1 del libro LPIC-2 (arranque, bootloaders, inicialización del sistema y recuperación), cuál es su equivalente en **FreeBSD 15**. FreeBSD es un sistema Unix completo (kernel + sistema base desarrollados juntos), así que muchas ideas son parecidas a Linux, pero las herramientas y rutas cambian. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entenderlo todo: en FreeBSD **no hay GRUB ni systemd**. El arranque lo hace su propio gestor (`loader`) y la inicialización la hace `init` + el sistema de scripts **rc**, configurado casi todo desde un único archivo: `/etc/rc.conf`.

---

## 1. Arranque del sistema (firmware)

| Concepto Linux | Equivalente en FreeBSD 15 |
|---|---|
| BIOS | Soportado. Arranque en varias fases: MBR/PMBR → `boot1`/`gptboot` → `loader` → kernel |
| UEFI | Soportado y habitual. El firmware carga directamente `loader.efi` desde la partición EFI (ESP) |
| POST | Igual, es una fase del firmware común a cualquier sistema operativo |

Fases del arranque en FreeBSD (equivalente a "BIOS → GRUB → kernel → init" en Linux):

| Fase | Qué es | Archivo |
|---|---|---|
| Fase 1 (solo BIOS) | Código mínimo en el primer sector del disco | `/boot/boot0` (MBR con menú) o `/boot/pmbr` (disco GPT) |
| Fase 2 (solo BIOS) | Lee el sistema de archivos y busca el loader | `/boot/gptboot` (UFS) o `/boot/gptzfsboot` (ZFS) |
| Fase 3 | El gestor de arranque de FreeBSD, con menú y línea de comandos | `/boot/loader` (BIOS) o `loader.efi` (UEFI) |
| Kernel | Núcleo del sistema | `/boot/kernel/kernel` |
| init | Primer proceso del sistema (PID 1) | `/sbin/init` |

En UEFI no existen las fases 1 y 2: el firmware lee directamente `loader.efi` desde la ESP.

---

## 2. Comandos para ver el proceso de arranque

| Comando / Archivo | Qué hace |
|---|---|
| `dmesg` | Igual que en Linux: muestra el buffer de mensajes del kernel |
| `dmesg -a` | Muestra todo el buffer, incluidos los mensajes de la consola y de los scripts de arranque |
| `/var/run/dmesg.boot` | Copia guardada de los mensajes del kernel del **último arranque** (útil cuando el buffer ya se ha llenado con otros mensajes) |
| `/var/log/messages` | Log general del sistema (lo gestiona `syslogd`) |
| `/var/log/console.log` | Mensajes de la consola, si se activa la línea correspondiente en `/etc/syslog.conf` |

Trucos en la consola:
- Tecla `Scroll Lock` (Bloq Despl) para desplazarse hacia atrás por los mensajes con las flechas o Re Pág/Av Pág. Se vuelve a pulsar para salir.
- `Alt+F1` a `Alt+F8` (o `Ctrl+Alt+F1...`) para cambiar entre consolas virtuales.
- Arranque "verboso" (más detalle): opción del menú del loader o `boot_verbose="YES"` en `/boot/loader.conf`.

---

## 3. Bootloaders

### 3.1 LILO, GRUB Legacy y GRUB2 → el `loader` de FreeBSD

FreeBSD no usa LILO ni GRUB por defecto. Su gestor propio se llama **loader** y cumple la función de GRUB2: muestra el menú (el famoso menú con el logo "Beastie"), permite elegir kernel, pasar opciones y tiene una línea de comandos propia.

| Elemento GRUB2 (Linux) | Equivalente en FreeBSD 15 |
|---|---|
| `/boot/grub/` | `/boot/` (contiene `loader`, `kernel/`, módulos, scripts) |
| `/boot/grub/grub.cfg` (generado) | No existe un archivo generado. El loader lee directamente sus archivos de configuración |
| `/etc/default/grub` | `/boot/loader.conf` (se edita a mano) |
| Valores por defecto | `/boot/defaults/loader.conf` (**no se edita**; sirve de referencia con todas las variables posibles) |
| `/etc/grub.d/40_custom` | `/boot/loader.conf.local` o archivos dentro de `/boot/loader.conf.d/` |
| `grub-mkconfig` / `update-grub` | No hace falta: los cambios en `loader.conf` se aplican en el siguiente arranque sin regenerar nada |
| Scripts del menú | `/boot/lua/` (el menú del loader está programado en Lua) |
| Teclas `e`, `c` en GRUB | En el menú del loader: número de opción, o `3`/`Esc` para ir a la línea de comandos del loader |

**Variables habituales de `/boot/loader.conf`:**

| Variable | Función |
|---|---|
| `autoboot_delay="5"` | Segundos de espera del menú (equivale a `GRUB_TIMEOUT`); `-1` arranca sin esperar |
| `beastie_disable="YES"` | Oculta el menú gráfico del loader |
| `kernel="kernel"` | Qué kernel cargar (nombre de carpeta dentro de `/boot`) |
| `boot_single="YES"` | Arranca siempre en modo mono-usuario |
| `boot_verbose="YES"` | Arranque con mensajes detallados |
| `nombre_load="YES"` | Carga un módulo del kernel al arrancar (ej. `zfs_load="YES"`) |
| `console="comconsole"` | Usa el puerto serie como consola (servidores sin pantalla) |
| `vfs.root.mountfrom="ufs:/dev/ada0p2"` | Indica qué partición montar como raíz |

**Opciones del menú del loader (FreeBSD 15):**

| Tecla | Opción |
|---|---|
| `1` o `Enter` | Arrancar en modo multiusuario (normal) |
| `2` | Arrancar en modo mono-usuario |
| `3` o `Esc` | Salir a la línea de comandos del loader (`OK`) |
| `4` | Reiniciar |
| `5` | Elegir kernel (ej. `kernel` o `kernel.old`) |
| `6` | Opciones de arranque (single user, verbose, etc.) |
| `8` | Entornos de arranque (Boot Environments), solo si el sistema usa ZFS |

La numeración exacta puede variar un poco según la instalación; el menú siempre muestra el número al lado de cada opción.

**Comandos de la línea de comandos del loader (prompt `OK`)** — equivalente a la shell `c` de GRUB:

| Comando | Función |
|---|---|
| `boot` | Arranca con la configuración actual |
| `boot -s` | Arranca en modo mono-usuario |
| `boot kernel.old` | Arranca el kernel anterior |
| `unload` | Descarga el kernel y los módulos ya cargados |
| `load /boot/kernel.old/kernel` | Carga un kernel concreto |
| `load nombre_modulo` | Carga un módulo del kernel |
| `lsdev` | Lista los dispositivos (discos) que ve el loader |
| `lsmod` | Lista los módulos cargados |
| `ls` | Lista archivos de un dispositivo |
| `show` / `show variable` | Muestra variables del loader |
| `set variable=valor` / `unset variable` | Crea o borra una variable |
| `more archivo` | Muestra un archivo (ej. `more /boot/loader.conf`) |
| `include archivo` | Ejecuta los comandos de un archivo |
| `autoboot 10` | Arranca tras 10 segundos |
| `help` | Ayuda |
| `reboot` | Reinicia |

**Opciones que se pasan al kernel** (equivalente a añadir parámetros a la línea `linux` de GRUB):

| Opción | Función |
|---|---|
| `-s` | Modo mono-usuario |
| `-v` | Mensajes detallados (verbose) |
| `-a` | Pregunta qué dispositivo montar como raíz |
| `-c` | Configuración de dispositivos en el arranque |
| `-d` | Entra en el depurador del kernel (DDB) lo antes posible |
| `-h` | Cambia entre consola interna y consola serie |

### 3.2 Instalar o reparar el bootloader (equivalente a `grub-install`)

| Situación | Comando en FreeBSD 15 |
|---|---|
| Disco GPT + BIOS + UFS | `gpart bootcode -b /boot/pmbr -p /boot/gptboot -i 1 ada0` |
| Disco GPT + BIOS + ZFS | `gpart bootcode -b /boot/pmbr -p /boot/gptzfsboot -i 1 ada0` |
| Disco MBR + BIOS (con menú `boot0`) | `gpart bootcode -b /boot/boot0 ada0` |
| Configurar el menú `boot0` | `boot0cfg -B ada0` (instala) / `boot0cfg -v ada0` (muestra configuración) |
| UEFI | Copiar el loader a la ESP: `cp /boot/loader.efi /boot/efi/EFI/freebsd/loader.efi` y también a `/boot/efi/EFI/BOOT/BOOTX64.EFI` (ruta de arranque por defecto) |

En `-i 1`, el número es el índice de la partición de tipo `freebsd-boot` (en un disco GPT con BIOS). `ada0` es el primer disco SATA; otros nombres de disco: `da0` (USB/SCSI), `nda0` (NVMe), `vtbd0` (disco virtual).

**Punto de montaje de la ESP (partición EFI):** `/boot/efi`.

### 3.3 Gestión de entradas UEFI: `efibootmgr`

Equivalente directo al `efibootmgr` de Linux (mismo nombre, sintaxis muy parecida).

| Comando | Función |
|---|---|
| `efibootmgr -v` | Lista las entradas de arranque UEFI con detalle |
| `efibootmgr -c -a -L FreeBSD -l /boot/efi/EFI/freebsd/loader.efi` | Crea y activa una nueva entrada llamada "FreeBSD" |
| `efibootmgr -o 0001,0000` | Cambia el orden de arranque |
| `efibootmgr -n -b 0001` | Arranca una sola vez (solo el próximo arranque) desde la entrada 0001 |
| `efibootmgr -B -b 0001` | Borra una entrada |

### 3.4 Bootloaders alternativos

| Bootloader del libro | Situación en FreeBSD 15 |
|---|---|
| systemd-boot | No existe (FreeBSD no usa systemd). Otros gestores UEFI, como rEFInd, pueden lanzar `loader.efi` |
| U-Boot | Se usa en placas ARM (ej. Raspberry Pi): U-Boot carga `loader.efi` de FreeBSD. Disponible en los ports `sysutils/u-boot-*` |
| SYSLINUX / EXTLINUX | No se usan para arrancar FreeBSD; existe el port `sysutils/syslinux` pero no es lo habitual |
| ISOLINUX (CD/DVD) | FreeBSD usa su propio arranque de CD: `/boot/cdboot` (BIOS) y `loader.efi` (UEFI). Para crear una ISO se usa `makefs -t cd9660` (del sistema base) o `mkisofs`/`xorriso` desde paquetes |
| PXELINUX (arranque por red) | FreeBSD usa `/boot/pxeboot` (BIOS) o `loader.efi` (UEFI) servidos por TFTP. Servidor TFTP: `tftpd` activado desde `/etc/inetd.conf`, con los archivos normalmente en `/tftpboot`. Se complementa con un servidor DHCP y NFS para el sistema raíz ("diskless") |
| MEMDISK | No hay equivalente directo. FreeBSD tiene discos en memoria con `md(4)` y el comando `mdconfig`, y puede arrancar una raíz en memoria (`mfsroot`) |
| GRUB2 (opcional) | Se puede instalar desde paquetes (`pkg install grub2-efi`), útil en arranque dual con Linux, pero no es lo recomendado |

### 3.5 Secure Boot (UEFI)

| Opción del libro para Linux | Situación en FreeBSD 15 |
|---|---|
| Desactivar Secure Boot | Es la opción habitual: `loader.efi` no viene firmado con las claves de Microsoft que traen la mayoría de equipos |
| Usar tu propia clave de firma | Posible: registrar tus claves en el firmware y firmar `loader.efi` (por ejemplo con `sbsign`, del paquete `sbsigntools`) |
| Shim firmado por el proveedor | No hay un shim oficial firmado para FreeBSD |

---

## 4. Inicialización del sistema: `init` y el sistema rc (equivalente a SysV init)

FreeBSD usa un modelo derivado de BSD, **no SysV**. Diferencias clave:
- **No hay runlevels numerados**. Solo existen dos estados: **mono-usuario** (single-user) y **multiusuario** (multi-user).
- **No hay `/etc/inittab`** ni carpetas `rcN.d` con enlaces `S`/`K`.
- Un servicio se activa o desactiva con una línea en `/etc/rc.conf` (ej. `sshd_enable="YES"`).
- El orden de arranque lo calcula automáticamente `rcorder` leyendo las dependencias escritas dentro de cada script.

### 4.1 Equivalencia de runlevels

| Runlevel Linux | Equivalente en FreeBSD 15 |
|---|---|
| 0 (apagar) | `shutdown -p now` o `poweroff` |
| 1 (mono-usuario) | Modo mono-usuario: `boot -s` en el loader, o `shutdown now` desde el sistema en marcha |
| 2, 3, 4, 5 (multiusuario) | Modo multiusuario (el único modo normal; no hay distinción entre texto y gráfico a nivel de init) |
| 6 (reiniciar) | `shutdown -r now` o `reboot` |

El entorno gráfico no depende de un "nivel": se arranca como un servicio más (ej. un gestor de sesión como `sddm` o `lightdm`, activado en `rc.conf`) o manualmente con `startx`.

### 4.2 Archivos y carpetas del sistema rc

| Archivo/Carpeta | Función | Equivalente Linux |
|---|---|---|
| `/sbin/init` | Primer proceso (PID 1) | `/sbin/init` |
| `/etc/rc` | Script principal que ejecuta `init` al pasar a multiusuario | `/etc/init.d/rc` |
| `/etc/rc.conf` | **Archivo principal**: qué servicios arrancan y su configuración (red, nombre de máquina, etc.) | `/etc/inittab` + enlaces de runlevels + parte de `/etc/default/` |
| `/etc/defaults/rc.conf` | Valores por defecto de todas las variables. **No se edita** | — |
| `/etc/rc.conf.local` y `/etc/rc.conf.d/` | Configuración adicional (opcional) | — |
| `/etc/rc.d/` | Scripts de los servicios del **sistema base** | `/etc/init.d/` |
| `/usr/local/etc/rc.d/` | Scripts de los servicios instalados con **paquetes o ports** | `/etc/init.d/` |
| `/etc/rc.subr` | Funciones comunes que usan todos los scripts rc | `/lib/lsb/init-functions` |
| `/etc/rc.shutdown` | Script que se ejecuta al apagar (para los servicios en orden inverso) | Scripts `K` |
| `/etc/rc.local` | Comandos personalizados al final del arranque (antiguo, pero sigue funcionando) | `/etc/rc.local` |
| `/etc/ttys` | Define las terminales de login y si la consola es segura (ver apartado 7) | Parte de `/etc/inittab` (líneas `getty`) |

**Ejemplo de `/etc/rc.conf`:**
```
hostname="servidor.ejemplo.local"
ifconfig_em0="DHCP"
sshd_enable="YES"
ntpd_enable="YES"
zfs_enable="YES"
```

**Estructura básica de un script rc** (equivalente a un archivo `.service` de systemd o un script de `/etc/init.d/`):
```
#!/bin/sh
# PROVIDE: miservicio
# REQUIRE: NETWORKING
# KEYWORD: shutdown

. /etc/rc.subr

name="miservicio"
rcvar="miservicio_enable"
command="/usr/local/bin/miservicio"

load_rc_config $name
run_rc_command "$1"
```

| Línea | Significado | Equivalente systemd |
|---|---|---|
| `# PROVIDE:` | Nombre que ofrece el script | Nombre de la unit |
| `# REQUIRE:` | Qué debe arrancar antes | `After=` / `Requires=` |
| `# BEFORE:` | Qué debe arrancar después | `Before=` |
| `# KEYWORD: shutdown` | Se ejecuta también al apagar | — |
| `rcvar=` | Variable de `rc.conf` que lo activa | `systemctl enable` |
| `command=` | Programa que se ejecuta | `ExecStart=` |

### 4.3 Comandos para gestionar servicios (equivalente a `chkconfig` / `update-rc.d` / `systemctl`)

**Comando `sysrc`** (edita `/etc/rc.conf` de forma segura, sin abrir el editor):

| Comando | Función |
|---|---|
| `sysrc sshd_enable="YES"` | Activa un servicio para que arranque siempre (equivale a `systemctl enable` / `chkconfig on`) |
| `sysrc sshd_enable="NO"` | Lo desactiva |
| `sysrc -x sshd_enable` | Borra la variable de `rc.conf` |
| `sysrc sshd_enable` | Muestra el valor actual de la variable |
| `sysrc -a` | Muestra todas las variables definidas en `rc.conf` |

**Comando `service`** (equivalente a `systemctl`):

| Comando | Función | Equivalente systemd |
|---|---|---|
| `service sshd start` | Arranca un servicio (debe estar activado en `rc.conf`) | `systemctl start` |
| `service sshd stop` | Para un servicio | `systemctl stop` |
| `service sshd restart` | Reinicia | `systemctl restart` |
| `service sshd reload` | Recarga configuración (si el script lo soporta) | `systemctl reload` |
| `service sshd status` | Muestra si está en marcha | `systemctl status` |
| `service sshd enable` | Activa el servicio en `rc.conf` (igual que `sysrc ..._enable=YES`) | `systemctl enable` |
| `service sshd disable` | Lo desactiva en `rc.conf` | `systemctl disable` |
| `service sshd onestart` | Arranca el servicio **una vez**, aunque no esté activado en `rc.conf` | `systemctl start` de una unit no habilitada |
| `service sshd onestop` | Para un servicio arrancado con `onestart` | — |
| `service sshd rcvar` | Muestra qué variable de `rc.conf` controla el servicio | — |
| `service -e` | Lista los servicios activados | `systemctl list-unit-files --state=enabled` |
| `service -l` | Lista todos los scripts disponibles | `systemctl list-unit-files` |
| `service -r` | Muestra el orden de arranque calculado | `systemd-analyze critical-chain` (aprox.) |
| `rcorder /etc/rc.d/*` | Calcula el orden de arranque a partir de las líneas `REQUIRE`/`PROVIDE` | — |

Importante: si un servicio no tiene `_enable="YES"` en `rc.conf`, `service nombre start` no hace nada y avisa. Por eso existen las variantes `one...` (`onestart`, `onestop`, `onerestart`).

No hace falta un equivalente a `systemctl daemon-reload`: los scripts rc se leen cada vez que se ejecutan.

### 4.4 Comandos para gestionar el estado del sistema completo

| Comando Linux | Equivalente en FreeBSD 15 | Función |
|---|---|---|
| `runlevel` | No existe | No hay runlevels que consultar. `who -b` muestra la hora del último arranque |
| `init N` / `telinit N` | `init 0`, `init 1`, `init 6` (se aceptan por compatibilidad) | `0` apagar, `1` pasar a mono-usuario, `6` reiniciar |
| — | `init q` | Vuelve a leer `/etc/ttys` |
| `shutdown` (paso a runlevel 1) | `shutdown now` | Pasa a modo mono-usuario |
| `shutdown -h now` | `shutdown -p now` | Apaga el equipo (`-p` = power off) |
| — | `shutdown -h now` | Detiene el sistema **sin cortar la corriente** |
| `shutdown -r now` | `shutdown -r now` | Reinicia |
| `shutdown -r +15 "mensaje"` | `shutdown -r +15 "mensaje"` | Reinicia dentro de 15 minutos avisando a los usuarios (igual que en Linux) |
| `shutdown -c` (cancelar) | No existe `-c`: se cancela matando el proceso `shutdown` pendiente (`pkill shutdown`) | Cancela un apagado programado |
| `halt` | `halt` (o `halt -p` para apagar) | Detiene el sistema sin avisar a usuarios |
| `poweroff` | `poweroff` | Apaga |
| `reboot` | `reboot` | Reinicia |

Igual que en Linux, `halt`, `poweroff` y `reboot` actúan de inmediato sin avisar; `shutdown` permite programar y avisar.

**Arrancar una sola vez con otra configuración** (útil tras cambiar el kernel):

| Comando | Función |
|---|---|
| `nextboot -k kernel.old` | El próximo arranque (solo uno) usará `/boot/kernel.old/kernel`; después vuelve al normal |
| `nextboot -o "-s"` | El próximo arranque será en modo mono-usuario |
| `nextboot -D` | Anula lo programado con `nextboot` |

---

## 5. systemd

**FreeBSD 15 no usa systemd.** Todo lo del apartado 5 del libro (units, targets, `systemctl`, `/lib/systemd/system/`) se sustituye por el sistema rc del apartado 4 de este documento.

| Concepto systemd | Equivalente en FreeBSD 15 |
|---|---|
| Unit `.service` | Script rc en `/etc/rc.d/` o `/usr/local/etc/rc.d/` |
| Target | No existe; solo mono-usuario y multiusuario |
| `default.target` | Siempre multiusuario, salvo que se indique `-s` o `boot_single="YES"` |
| `/lib/systemd/system/` | `/etc/rc.d/` |
| `/etc/systemd/system/` | `/usr/local/etc/rc.d/` (servicios de terceros) y `/etc/rc.conf` (activación) |
| `systemctl` | `service` + `sysrc` |
| `systemctl isolate` | No hay equivalente directo |
| `systemctl daemon-reload` | No hace falta |
| `After=`, `Requires=` | Líneas `# REQUIRE:` / `# BEFORE:` del script rc |

---

## 6. Upstart

No existe en FreeBSD. No aplica.

---

## 7. Recuperación del sistema

### 7.1 Modo mono-usuario (single-user mode)

Formas de entrar:
1. En el menú del loader, pulsar `2` (Boot Single User).
2. O pulsar `3`/`Esc` para ir al prompt `OK` y escribir `boot -s`.
3. Desde el sistema en marcha: `shutdown now`.

Al entrar aparece el mensaje:
```
Enter full pathname of shell or RETURN for /bin/sh:
```
Se pulsa `Enter` y se obtiene una shell de root **sin red** y con la raíz montada **en solo lectura**.

Pasos típicos dentro del modo mono-usuario:

| Comando | Función |
|---|---|
| `fsck -p` / `fsck -y` | Revisa los sistemas de archivos UFS antes de escribir en ellos |
| `mount -u /` | Vuelve a montar la raíz en modo lectura/escritura (UFS) |
| `mount -a -t ufs` | Monta el resto de sistemas de archivos UFS de `/etc/fstab` |
| `zfs set readonly=off zroot` | Permite escribir en la raíz si el sistema usa ZFS (`zroot` es el nombre por defecto del pool) |
| `zfs mount -a` | Monta todos los datasets ZFS |
| `passwd root` | Cambia la contraseña de root (recuperar acceso) |
| `exit` | Sale del modo mono-usuario y continúa hacia multiusuario |

**Proteger el modo mono-usuario con contraseña** (equivalente a proteger GRUB): en `/etc/ttys`, cambiar la palabra `secure` por `insecure` en la línea de la consola:
```
console none   unknown  off insecure
```
Así el sistema pedirá la contraseña de root antes de dar la shell.

### 7.2 Selección de kernels anteriores

| Concepto Linux | Equivalente en FreeBSD 15 |
|---|---|
| Mantener el kernel anterior en el menú de GRUB | Al instalar un kernel nuevo, el anterior se guarda automáticamente en `/boot/kernel.old/` |
| Elegir el kernel anterior en el menú | Opción `5` del menú del loader (cambia entre `kernel` y `kernel.old`), o `boot kernel.old` en el prompt `OK` |
| Arrancar el anterior solo una vez | `nextboot -k kernel.old` |

**Boot Environments (solo con ZFS, que es la opción por defecto del instalador)**: es una ventaja propia de FreeBSD. Un entorno de arranque es una copia (instantánea) de todo el sistema. Si una actualización sale mal, se vuelve al estado anterior completo, no solo al kernel.

| Comando | Función |
|---|---|
| `bectl list` | Lista los entornos de arranque |
| `bectl create antes-de-actualizar` | Crea un entorno de arranque (copia del sistema actual) |
| `bectl activate nombre` | Hace que el próximo arranque use ese entorno (de forma permanente) |
| `bectl activate -t nombre` | Lo usa solo en el próximo arranque (temporal) |
| `bectl destroy nombre` | Borra un entorno de arranque |

También se pueden elegir desde la opción `8` del menú del loader.

Si el sistema se actualizó con `freebsd-update`, se puede deshacer la última actualización con `freebsd-update rollback`.

### 7.3 Fallo del disco raíz (root drive failure)

| Comando Linux | Equivalente en FreeBSD 15 | Función |
|---|---|---|
| `fsck /dev/sdaX` | `fsck /dev/ada0p2` | Revisa un sistema de archivos. Igual que en Linux, `fsck` llama al programa específico (`fsck_ufs`, `fsck_msdosfs`...) |
| `fsck -y /dev/sdaX` | `fsck -y /dev/ada0p2` | Responde "sí" a todas las reparaciones |
| — | `fsck_ufs -y /dev/ada0p2` | Llama directamente a la versión para UFS |
| (ext4) | `zpool status` / `zpool scrub zroot` | En ZFS **no existe fsck**: el pool se comprueba y repara con `scrub` |
| `mount /dev/sdaX /media` | `mount /dev/ada0p2 /mnt` | Monta la partición reparada (UFS) |
| — | `zpool import -f -R /mnt zroot` | Importa (monta) un pool ZFS en `/mnt` desde un sistema de rescate |
| `umount /dev/sdaX` | `umount /mnt` (UFS) / `zpool export zroot` (ZFS) | Desmonta antes de reiniciar |

Opciones de `/etc/rc.conf` relacionadas con la revisión automática de discos al arrancar:

| Variable | Función |
|---|---|
| `fsck_y_enable="YES"` | Si `fsck` encuentra errores al arrancar, los repara automáticamente respondiendo "sí" |
| `background_fsck="NO"` | Desactiva la revisión en segundo plano (por defecto FreeBSD revisa UFS en segundo plano tras arrancar) |

Nombres de dispositivo: en Linux `/dev/sda1`; en FreeBSD `/dev/ada0p1` (GPT) o `/dev/ada0s1a` (MBR con particiones BSD). Para verlos: `gpart show` o `geom disk list`.

**Disco de rescate (rescue disk):**
- El propio medio de instalación de FreeBSD (USB o ISO) ofrece la opción **Live System** (o "Shell") en su menú de bienvenida: carga un FreeBSD mínimo desde el que se pueden reparar los discos, igual que el rescue disk del libro.
- Además, FreeBSD incluye siempre la carpeta **`/rescue`**: contiene programas básicos (`sh`, `mount`, `fsck`, `zfs`, `ifconfig`...) compilados de forma estática, que funcionan aunque las librerías del sistema estén dañadas.

---

## 8. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 1) | Equivalente en FreeBSD 15 |
|---|---|
| GRUB2 / `grub.cfg` | `loader` / `/boot/loader.conf` |
| `/etc/default/grub` | `/boot/loader.conf` (valores por defecto en `/boot/defaults/loader.conf`) |
| `grub-mkconfig`, `update-grub` | No hace falta (se edita `loader.conf` y listo) |
| `grub-install` | `gpart bootcode` (BIOS) o copiar `loader.efi` a la ESP (UEFI) |
| `efibootmgr` | `efibootmgr` (igual) |
| ISOLINUX / PXELINUX | `cdboot` / `pxeboot` o `loader.efi` |
| SysV init / systemd | `init` + sistema rc (`/etc/rc`, `/etc/rc.d/`, `/usr/local/etc/rc.d/`) |
| `/etc/inittab` | `/etc/rc.conf` (servicios) y `/etc/ttys` (terminales) |
| Runlevels 0-6 / targets | Solo mono-usuario y multiusuario |
| `chkconfig`, `update-rc.d`, `systemctl enable` | `sysrc nombre_enable="YES"` o `service nombre enable` |
| `systemctl start/stop/status` | `service nombre start/stop/status` |
| `dmesg`, logs en `/var/log` | `dmesg`, `/var/run/dmesg.boot`, `/var/log/messages` |
| `shutdown -h now` / `poweroff` | `shutdown -p now` / `poweroff` |
| `reboot`, `shutdown -r` | `reboot`, `shutdown -r` (igual) |
| `init N` / `telinit N` | `init 0/1/6` (compatibilidad) o `shutdown` |
| Añadir `single` en GRUB | Opción `2` del menú o `boot -s` |
| Kernel anterior en el menú | `/boot/kernel.old` (opción `5` del menú) y Boot Environments con `bectl` |
| `fsck` | `fsck` / `fsck_ufs` (UFS) y `zpool scrub` (ZFS) |
| Live-CD de rescate | Opción "Live System" del medio de instalación y carpeta `/rescue` |

---

*Documento elaborado como contrapartida en FreeBSD 15 al Capítulo 1 ("Starting a System") del libro LPIC-2. Fuentes: FreeBSD Handbook (capítulo "The FreeBSD Booting Process" y capítulos relacionados) y páginas de manual de FreeBSD: loader(8), loader.conf(5), boot(8), init(8), rc(8), rc.conf(5), service(8), sysrc(8), shutdown(8), nextboot(8), efibootmgr(8), gpart(8), bectl(8).*
