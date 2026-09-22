# LPIC-2 · Capítulo 1: Starting a System
### Recopilación de comandos y archivos de configuración

Objetivos del examen cubiertos: 202.1 (Customizing SysV-init system startup), 202.2 (System Recovery), 202.3 (Alternate Bootloaders)

---

## 1. Arranque del sistema (firmware)

| Elemento | Tipo | Descripción |
|---|---|---|
| BIOS | Firmware antiguo | Solo puede leer un sector del disco, por eso necesita un bootloader en dos fases |
| UEFI | Firmware moderno | Sustituye al BIOS, permite arrancar directamente ficheros más grandes, usa extensión `.efi` |
| POST (Power-On Self Test) | Proceso | Primera comprobación de hardware que hace el firmware |

No hay diferencia entre Debian y Red Hat en esta parte: es un tema de firmware, no de distribución.

---

## 2. Comandos para ver el proceso de arranque

| Comando | Qué hace |
|---|---|
| `dmesg` | Muestra los mensajes más recientes del arranque, guardados en el kernel ring buffer (buffer circular en memoria) |

Atajo de teclado: `Esc` o `Ctrl+Alt+F1` para ver los mensajes de arranque si la distro los oculta.

Los ficheros de log de arranque suelen guardarse en `/var/log`.

---

## 3. Bootloaders

### 3.1 LILO (Linux Loader) — el más antiguo, en desuso

| Elemento | Valor |
|---|---|
| Archivo de configuración | `/etc/lilo.conf` |
| Limitación | No funciona con UEFI |

### 3.2 GRUB Legacy

| Elemento | Valor |
|---|---|
| Carpeta de configuración | `/boot/grub` |
| Archivo de configuración (Debian) | `menu.lst` |
| Archivo de configuración (Red Hat: CentOS/Fedora) | `grub.conf` |
| Comando para instalar en el MBR | `grub-install '(hd0)'` |
| Comando para instalar en una partición (formato Linux) | `grub-install /dev/sda1` |
| Comando para instalar en una partición (formato GRUB) | `grub-install 'hd(0,0)'` |
| Comando principal en el fichero | `title`, `kernel`, `initrd`, `root (hdX,Y)` |
| Comandos globales | `color`, `default`, `fallback`, `hiddenmenu`, `splashimage`, `timeout` |
| Teclas en el menú de arranque | `e` editar, `b` arrancar con cambios, `c` shell interactiva |

### 3.3 GRUB2 — el más usado actualmente (Debian y Red Hat)

| Elemento | Valor |
|---|---|
| Carpeta | `/boot/grub` |
| Archivo final (no se edita a mano) | `/boot/grub/grub.cfg` |
| Archivo de configuración global | `/etc/default/grub` (variables tipo `GRUB_TIMEOUT`) |
| Carpeta de scripts que generan grub.cfg | `/etc/grub.d/` (ej. `40_custom` para entradas personalizadas) |
| Comando para regenerar grub.cfg | `grub-mkconfig > /boot/grub/grub.cfg` (o con `-o`) |
| Comando equivalente simplificado (Debian/Ubuntu) | `update-grub` |
| Palabra clave en vez de `title` | `menuentry "Nombre" { ... }` |
| Palabra clave en vez de `root (hdX,Y)` | `set root=(hdX,Y)` — ojo: en GRUB2 las particiones empiezan en 1, no en 0 |
| Palabra clave en vez de `kernel` | `linux /ruta/al/kernel` |
| Teclas en el menú | flechas para moverse, `e` editar, `c` shell de comandos |
| Truco para ver el menú oculto (Ubuntu) | mantener pulsada `Shift` al arrancar |

Diferencia clave Debian vs Red Hat con GRUB2: ambos usan la misma estructura de archivos y comandos; la diferencia práctica es que Debian/Ubuntu ofrece el atajo `update-grub`, mientras que en Red Hat se suele ejecutar `grub-mkconfig` directamente (o `grub2-mkconfig` según versión).

### 3.4 Bootloaders alternativos

| Bootloader | Uso | Archivos clave |
|---|---|---|
| systemd-boot | Sistemas con systemd, arranca imágenes EFI | — |
| U-Boot | Arranca desde cualquier tipo de disco/imagen | — |
| SYSLINUX | Sistemas con FAT (ej. USB) | — |
| EXTLINUX | Sistemas ext2/ext3/ext4/btrfs | — |
| ISOLINUX | LiveCD/LiveDVD | `isolinux.bin` (programa), `isolinux.cfg` (configuración), `isolinuxhpfx.bin` (para USB, generado con `xorriso`) |
| PXELINUX | Arranque por red (PXE) | Servidor TFTP: `/tftpboot/pxelinux.0` y carpeta `/tftpboot/pxelinux.cfg/` (un archivo por MAC) |
| MEMDISK | Arrancar sistemas DOS antiguos desde Syslinux | — |

### 3.5 Secure Boot (UEFI)

Tres formas de convivir con Secure Boot en Linux:
- Desactivar Secure Boot en el firmware UEFI.
- Comprar/usar tu propia clave de firma digital.
- Usar un shim firmado por el proveedor de la distro.

---

## 4. Inicialización del sistema: SysV init

### 4.1 Runlevels

| Runlevel | Significado general |
|---|---|
| 0 | Apagar el sistema |
| 1 | Modo mono-usuario (mantenimiento) |
| 2 | **Debian**: modo multiusuario completo (gráfico) |
| 3 | **Red Hat**: modo multiusuario en texto |
| 4 | Sin definir |
| 5 | **Red Hat**: modo multiusuario gráfico |
| 6 | Reiniciar el sistema |

Diferencia importante: Red Hat usa el runlevel 3 (texto) y 5 (gráfico) como niveles multiusuario habituales; Debian usa el runlevel 2 para todo el modo multiusuario, tanto gráfico como texto.

### 4.2 Archivos y carpetas de SysV

| Archivo/Carpeta | Función |
|---|---|
| `/etc/inittab` | Define el runlevel por defecto (línea `id:N:initdefault:`) y qué procesos arrancan en cada runlevel |
| `/etc/init.d/` | Contiene los scripts de arranque/parada de cada servicio |
| `/etc/rc.d/` (o `/etc/init.d/rcX.d`, `/etc/rcX.d`) | Carpetas por runlevel con enlaces a los scripts; `X` es el número de runlevel |
| Script `/etc/init.d/rc` o `/etc/rc.d/rc` | Ejecuta todos los scripts correspondientes al runlevel indicado |

Formato de línea en `/etc/inittab`:
```
id:runlevels:action:process
```

Valores posibles de `action`: `boot`, `bootwait`, `initdefault`, `kbrequest`, `once`, `powerfail`, `powerwait`, `respawn`, `sysinit`, `wait`.

Convención de nombres de los scripts en las carpetas de runlevel:
- Empiezan por `S` → arrancan el programa (Start).
- Empiezan por `K` → paran el programa (Kill).
- Llevan un número → define el orden de ejecución.

### 4.3 Comandos para gestionar runlevels de programas individuales

| Comando | Distribución típica | Función |
|---|---|---|
| `chkconfig` | **Red Hat** | Lista y modifica en qué runlevels arranca un programa |
| `chkconfig --list network` | Red Hat | Muestra en qué runlevels arranca el servicio `network` |
| `chkconfig --levels 12345 network on` | Red Hat | Activa el servicio `network` en los runlevels 1 a 5 |
| `update-rc.d` | **Debian** | Equivalente a `chkconfig` para cambiar los runlevels de un programa |

### 4.4 Comandos para gestionar el runlevel del sistema completo

| Comando | Función |
|---|---|
| `runlevel` | Muestra el runlevel anterior y el actual (ej. `N 2`; `N` = sin runlevel previo) |
| `init N` | Cambia inmediatamente al runlevel N (brusco, no avisa a otros usuarios) |
| `telinit N` | Igual que `init`, cambia el runlevel actual |
| `shutdown` | Cambia de forma controlada al runlevel 1 (o apaga, según opciones) |
| `halt` | Para el sistema (runlevel 0) |
| `poweroff` | Apaga el sistema (runlevel 0) |
| `reboot` | Reinicia el sistema (runlevel 6) |

Estos comandos (`shutdown`, `halt`, `poweroff`, `reboot`) permiten avisar a otros usuarios y programar el cambio (ej. `+15` para dentro de 15 minutos).

---

## 5. Inicialización del sistema: systemd (Red Hat moderno, y también Debian moderno)

Nota: el libro presenta systemd como propio de Fedora/CentOS/Red Hat en esa época, pero hoy en día Debian 13 y Rocky Linux 10 usan ambos systemd como sistema de init por defecto.

### 5.1 Conceptos

- **Unit**: un servicio o acción del sistema (nombre + tipo + archivo de configuración).
- Tipos de unit: `automount`, `device`, `mount`, `path`, `service`, `snapshot`, `socket`, `target`.
- **Target**: agrupa varias units; es el equivalente moderno al runlevel.
- Compatibilidad con runlevels clásicos: `runlevel0.target` a `runlevel6.target`.

### 5.2 Archivos y carpetas

| Archivo/Carpeta | Función |
|---|---|
| `/lib/systemd/system/` | Archivos de configuración de las units (ej. `sshd.service`) |
| `/etc/systemd/system/default.target` | Define el target por defecto; normalmente es un enlace simbólico a un target en `/lib/systemd/system/` |
| `/etc/systemd/system/` | Units personalizadas o enlaces creados con `enable` |

Ejemplo de estructura de un archivo `.service`: secciones `[Unit]`, `[Service]`, `[Install]` (con claves como `Description`, `After`, `ExecStart`, `Restart`, `WantedBy`).

Ejemplo de estructura de un archivo `.target`: secciones `[Unit]` con `Requires`, `After`, `Conflicts`, `Wants`.

### 5.3 Comando `systemctl`

| Comando | Función |
|---|---|
| `systemctl list-units` | Lista las units cargadas y su estado |
| `systemctl start nombre` | Arranca una unit |
| `systemctl stop nombre` | Para una unit |
| `systemctl restart nombre` | Reinicia una unit |
| `systemctl reload nombre` | Recarga la configuración de una unit sin pararla |
| `systemctl status nombre` | Muestra el estado de una unit (admite también un PID) |
| `systemctl enable nombre` | Hace que la unit arranque en el próximo inicio del sistema |
| `systemctl disable nombre` | Evita que la unit arranque en el próximo inicio |
| `systemctl isolate nombre` | Arranca esa unit y para todas las demás (cambio de target) |
| `systemctl default` | Cambia al target por defecto |
| `systemctl daemon-reload` | Recarga la configuración de systemd tras crear/modificar un archivo de unit |

No hay diferencia real de comandos entre Debian y Red Hat en systemd: la sintaxis de `systemctl` es idéntica en ambas familias.

---

## 6. Upstart (histórico, usado antiguamente en Ubuntu)

| Elemento | Valor |
|---|---|
| Carpeta de configuración | `/etc/init/` |
| Comando para arrancar un servicio | `start nombre` (ej. `sudo start bluetooth`) |
| Comando para parar un servicio | `stop nombre` (ej. `sudo stop bluetooth`) |

Upstart está prácticamente en desuso hoy en día (sustituido por systemd), se incluye aquí solo como referencia histórica del libro.

---

## 7. Recuperación del sistema

### 7.1 Modo mono-usuario (single-user mode)

Pasos generales:
1. En el menú de GRUB, seleccionar la entrada y pulsar `e` para editar.
2. Buscar la línea `linux` o `linux16`.
3. Añadir la palabra `single` al final de esa línea.
4. `Ctrl+X` (o `F10`) para arrancar con el cambio.
5. El sistema arranca en runlevel 1, con acceso solo para root.

Comando para comprobar el runlevel una vez dentro: `runlevel`

### 7.2 Selección de kernels anteriores

Buena práctica: mantener el kernel anterior instalado y una entrada en el menú de GRUB que apunte a él, por si el nuevo kernel falla al arrancar. La mayoría de distribuciones lo hacen automáticamente.

### 7.3 Fallo del disco raíz (root drive failure)

| Comando | Función |
|---|---|
| `fsck /dev/sdaX` | Revisa y repara errores del sistema de archivos en la partición indicada (`fsck` es en realidad un envoltorio para comandos específicos según el tipo de sistema de archivos: ext2, ext3, ext4, etc.) |
| `fsck -y /dev/sdaX` | Igual que arriba, pero responde "sí" automáticamente a todas las preguntas de reparación |
| `mount /dev/sdaX /media` | Monta la partición reparada para poder revisarla |
| `umount /dev/sdaX` | Desmonta la partición antes de reiniciar |

Procedimiento típico: arrancar con un disco de rescate (rescue disk) que carga un Linux mínimo en memoria, dejando los discos del sistema libres para poder repararlos con `fsck`.

---

## 8. Resumen de diferencias Debian vs Red Hat en este capítulo

| Aspecto | Debian | Red Hat |
|---|---|---|
| Runlevel multiusuario por defecto | 2 (gráfico y texto a la vez) | 3 (texto) y 5 (gráfico) |
| Comando para gestionar runlevels de un servicio | `update-rc.d` | `chkconfig` |
| Archivo GRUB Legacy | `menu.lst` | `grub.conf` |
| Atajo para regenerar GRUB2 | `update-grub` | `grub-mkconfig` (manual, sin atajo específico en el libro) |
| systemd | Igual (`systemctl`, mismas rutas) | Igual (`systemctl`, mismas rutas) |

---

*Documento generado a partir del Capítulo 1 ("Starting a System") del libro LPIC-2: Linux Professional Institute Certification Study Guide (Bresnahan & Blum).*
