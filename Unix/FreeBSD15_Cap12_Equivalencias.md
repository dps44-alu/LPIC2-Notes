# LPIC-2 · Capítulo 12: Setting Up System Security
### Equivalencias en FreeBSD 15

Este documento recoge, para cada concepto visto en el Capítulo 12 del libro LPIC-2 (router y NAT, cortafuegos, OpenSSH, OpenVPN, auditoría y detección de intrusiones), cuál es su equivalente en **FreeBSD 15**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo: **`iptables` no existe en FreeBSD**. En su lugar, FreeBSD trae en el sistema base **dos cortafuegos completos**:
- **`pf`** (Packet Filter): viene de OpenBSD, con reglas escritas en un archivo de texto muy legible. Es el más usado.
- **`ipfw`**: el cortafuegos propio de FreeBSD, con reglas numeradas, más parecido en su manejo a `iptables`.

Existe un tercero, IPFilter (`ipf`), más antiguo y en desuso. Además, FreeBSD aporta herramientas de seguridad propias que no aparecen en el libro: `blocklistd`, `pkg audit`, `freebsd-update IDS`, los informes de seguridad diarios y las jails.

---

## 1. Direcciones privadas y NAT

### 1.1 y 1.2 Direcciones privadas IPv4 e IPv6 link-local

**Iguales** que en el libro (son estándares): `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` y direcciones `fe80::` para IPv6.

### 1.3 NAT y router

El concepto es el mismo. Para que FreeBSD funcione como router, primero hay que activar el reenvío de paquetes entre interfaces:

| Linux | FreeBSD 15 |
|---|---|
| `net.ipv4.ip_forward=1` en `/etc/sysctl.conf` | `gateway_enable="YES"` en `/etc/rc.conf` (o en caliente: `sysctl net.inet.ip.forwarding=1`) |
| `net.ipv6.conf.all.forwarding=1` | `ipv6_gateway_enable="YES"` en `/etc/rc.conf` |

El NAT y la redirección de puertos (port forwarding) se hacen con `pf` o con `ipfw` (apartado 2).

---

## 2. Cortafuegos (equivalente a `iptables`)

### 2.0 Cuál elegir

| Cortafuegos | Configuración | Orden de las reglas | Recomendado para |
|---|---|---|---|
| **`pf`** | Archivo `/etc/pf.conf` | Gana la **última** regla que coincida (salvo que tenga `quick`) | La mayoría de casos: servidores, routers, NAT |
| **`ipfw`** | Script de comandos (ej. `/etc/ipfw.rules`) o tipos predefinidos | Gana la **primera** regla que coincida (por número) | Quien prefiera un manejo parecido a `iptables`, o necesite control de ancho de banda (`dummynet`) |

Importante: se debe usar **solo uno** de los dos a la vez.

### 2.1 y 2.2 Cadenas y tablas de `iptables`

`pf` e `ipfw` no usan cadenas ni tablas como `iptables`. Su equivalencia aproximada:

| Concepto `iptables` | En `pf` | En `ipfw` |
|---|---|---|
| Cadena `INPUT` | Reglas con `in` (hacia una interfaz) y destino el propio equipo | Reglas con `in` y `to me` |
| Cadena `OUTPUT` | Reglas con `out` y origen el propio equipo | Reglas con `out` y `from me` |
| Cadena `FORWARD` | Reglas `in` en una interfaz + `out` en otra | Reglas con `via` interfaz |
| `PREROUTING` (redirección) | `rdr` | `fwd` o `ipfw nat` |
| `POSTROUTING` (NAT de salida) | `nat` | `ipfw nat` (NAT dentro del kernel) |
| Tabla `FILTER` | Reglas `pass` / `block` | Reglas `allow` / `deny` |
| Tabla `NAT` | Reglas `nat` / `rdr` / `binat` | `ipfw nat` |
| Tabla `MANGLE` | `scrub` (normalizar paquetes) | Opciones de las reglas |

**Tablas de direcciones** (no confundir con las tablas de `iptables`): tanto `pf` como `ipfw` permiten crear listas de IPs con nombre (ej. `<bloqueados>` en `pf`) que se usan en las reglas y se modifican en caliente.

### 2.3 Opciones básicas de `iptables` y sus equivalentes (Tabla 12.1)

| `iptables` | `pf` (`pfctl`) | `ipfw` |
|---|---|---|
| (activar el cortafuegos) | `pfctl -e` | `ipfw enable firewall` |
| (desactivarlo) | `pfctl -d` | `ipfw disable firewall` |
| `-A cadena regla` (añadir) | Escribir la regla en `/etc/pf.conf` y recargar: `pfctl -f /etc/pf.conf` | `ipfw add regla` (o `ipfw add NÚMERO regla`) |
| `-D cadena regla` (borrar) | Quitarla de `/etc/pf.conf` y recargar | `ipfw delete NÚMERO` |
| `-F` (vaciar) | `pfctl -F rules` (o `pfctl -F all` para vaciar también NAT, estados y tablas) | `ipfw -q flush` |
| `-I cadena índice regla` (insertar) | Colocar la regla en su sitio dentro del archivo | `ipfw add NÚMERO regla` (el número marca la posición) |
| `-L` (listar) | `pfctl -s rules` (reglas), `pfctl -s nat` (NAT), `pfctl -s states` (conexiones activas) | `ipfw list` (o `ipfw -a list` con contadores) |
| `-P cadena destino` (política) | Una regla general al principio del archivo: `block all` (o `pass all`) | La última regla (65535) es por defecto `deny ip from any to any` |
| `-R` (sustituir) | Editar el archivo y recargar | Borrar y volver a añadir con el mismo número |
| `-S` (detalle) | `pfctl -s all` / `pfctl -vv -s rules` | `ipfw -a list` |
| `-t tabla` | No aplica | No aplica |
| — | `pfctl -nf /etc/pf.conf` (comprueba la sintaxis sin aplicar nada) | — |
| — | `pfctl -t bloqueados -T add 10.0.1.25` (añade una IP a una tabla) | `ipfw table bloqueados add 10.0.1.25` |

### 2.4 Políticas y acciones

| `iptables` | `pf` | `ipfw` | Efecto |
|---|---|---|---|
| `ACCEPT` | `pass` | `allow` | Deja pasar el paquete |
| `DROP` | `block drop` (o `block` a secas) | `deny` | Descarta sin avisar |
| `REJECT` | `block return` | `reset` (TCP) / `unreach port` | Descarta y avisa al origen |
| `LOG` | Palabra `log` dentro de la regla | Palabra `log` dentro de la regla | Registra el paquete |

Ejemplo del libro — bloquear todo el tráfico saliente:

| `iptables` | `pf` (en `/etc/pf.conf`) | `ipfw` |
|---|---|---|
| `iptables -t filter -P OUTPUT DROP` | `block out all` | `ipfw add 100 deny ip from me to any out` |

### 2.5 Opciones de una regla (equivalente a la Tabla 12.2)

| `iptables` | `pf` | `ipfw` |
|---|---|---|
| `-s dirección` | `from dirección` | `from dirección` |
| `-d dirección` | `to dirección` | `to dirección` |
| `-i interfaz` | `in on interfaz` | `in via interfaz` (o `recv interfaz`) |
| `-o interfaz` | `out on interfaz` | `out via interfaz` (o `xmit interfaz`) |
| `-p protocolo` | `proto tcp` / `udp` / `icmp` | `tcp` / `udp` / `icmp` (tras la acción) |
| `--sport` / `--dport` | `from ... port N` / `to ... port N` | Número de puerto tras la dirección: `from any 1234 to any 80` |
| `-j destino` | Acción al principio de la regla (`pass`, `block`) | Acción tras el número de regla (`allow`, `deny`) |
| `-g cadena` | `anchor` (grupo de reglas) | `skipto NÚMERO` |
| `-m state --state ESTABLISHED` | `keep state` (en `pf` las reglas `pass` ya guardan el estado por defecto) | `keep-state` / `setup` |

**Ejemplos del libro traducidos:**

Bloquear todo el tráfico entrante de una IP:

| `iptables` | `pf` | `ipfw` |
|---|---|---|
| `iptables -A INPUT -s 10.0.1.25 -j REJECT` | `block return in quick from 10.0.1.25` | `ipfw add 100 unreach host ip from 10.0.1.25 to me in` |

Bloquear un puerto TCP de salida:

| `iptables` | `pf` | `ipfw` |
|---|---|---|
| `iptables -A OUTPUT -p tcp --dport 1234 -j DROP` | `block drop out proto tcp to port 1234` | `ipfw add 200 deny tcp from me to any 1234 out` |

### 2.6 Configuración completa y persistencia

A diferencia de `iptables`, **las reglas no se pierden al reiniciar**: se escriben en un archivo que se carga en cada arranque. No hace falta `iptables-save` ni `iptables-restore`. Además, **las mismas reglas valen para IPv4 e IPv6** (no hay un `ip6tables` aparte); si se quiere limitar a una versión se usa `inet` o `inet6` (en `pf`) o `ip4` / `ip6` (en `ipfw`).

**Con `pf`** — en `/etc/rc.conf`:
```
pf_enable="YES"
pf_rules="/etc/pf.conf"
pflog_enable="YES"
```

Ejemplo de `/etc/pf.conf` para un router con NAT (orden obligatorio: macros, tablas, opciones, `scrub`, NAT/redirecciones y, al final, filtrado):
```
# Macros (variables)
ext_if = "em0"
int_if = "em1"
int_net = "192.168.1.0/24"

# Tablas
table <bloqueados> persist

# Opciones
set skip on lo0

# Normalización de paquetes
scrub in all

# NAT de salida y redirección de puertos (port forwarding)
nat on $ext_if from $int_net to any -> ($ext_if)
rdr on $ext_if proto tcp from any to ($ext_if) port 8080 -> 192.168.1.10 port 80

# Filtrado
block log all
block drop in quick from <bloqueados>
pass out all
pass in on $int_if from $int_net
pass in on $ext_if proto tcp to ($ext_if) port 22
pass in on $ext_if proto tcp to 192.168.1.10 port 80
```

| Comando | Función |
|---|---|
| `service pf start` / `service pf reload` | Activa el cortafuegos / recarga `/etc/pf.conf` |
| `pfctl -nf /etc/pf.conf` | Comprueba la sintaxis (siempre antes de recargar, sobre todo en conexiones remotas) |
| `pfctl -s info` | Estadísticas generales |
| `tcpdump -n -e -ttt -i pflog0` | Muestra en directo los paquetes registrados con `log` (el log se guarda en `/var/log/pflog`) |

**Con `ipfw`** — en `/etc/rc.conf`, usando un tipo predefinido:
```
firewall_enable="YES"
firewall_type="workstation"
firewall_myservices="22/tcp 80/tcp 443/tcp"
firewall_allowservices="any"
firewall_logging="YES"
```
Tipos predefinidos: `open` (permite todo), `client` / `workstation` (protege un equipo), `simple` (router sencillo), `closed` (bloquea todo salvo `lo0`). O bien un script propio: `firewall_script="/etc/ipfw.rules"`.

Ejemplo de `/etc/ipfw.rules`:
```
#!/bin/sh
ipfw -q flush
cmd="ipfw -q add"
$cmd 00100 allow all from any to any via lo0
$cmd 00200 check-state
$cmd 00300 allow tcp from me to any out setup keep-state
$cmd 00400 allow tcp from any to me 22 in setup keep-state
$cmd 00500 deny log all from any to any
```

Con `ipfw`, los paquetes registrados van a `/var/log/security`.

---

## 3. OpenSSH

### 3.1 Componentes

**Los mismos** (`sshd`, `ssh`), incluidos en el **sistema base** de FreeBSD.

| Elemento | FreeBSD 15 |
|---|---|
| Activar el servidor | `sysrc sshd_enable="YES"` (el instalador ofrece activarlo) |
| Arrancar / reiniciar / recargar | `service sshd start` / `restart` / `reload` |
| Log de accesos | **`/var/log/auth.log`** |

### 3.2 Fichero de configuración del servidor

Mismo archivo que en Linux: **`/etc/ssh/sshd_config`**. Las opciones de la Tabla 12.3 son **las mismas**, con estas notas:

| Opción | Nota |
|---|---|
| `Protocol` | Ya **no existe** en las versiones actuales de OpenSSH (solo se usa el protocolo 2). Afecta también a Linux |
| `PermitRootLogin` | En FreeBSD el valor por defecto es **`no`** |
| `PasswordAuthentication` | En FreeBSD el valor por defecto es `no`, pero las contraseñas se siguen aceptando a través de PAM con **`KbdInteractiveAuthentication`**. Para prohibir de verdad las contraseñas hay que poner **las dos** a `no` |
| `AllowUsers`, `DenyUsers`, `PubkeyAuthentication`, `X11Forwarding`, `AllowTcpForwarding` | Iguales |
| `UseBlocklist` | Propia de FreeBSD: avisa a `blocklistd` de los intentos fallidos (apartado 6) |

Comprobar la configuración antes de reiniciar: `sshd -t`. Aplicar: `service sshd reload`.

La configuración del cliente está en `/etc/ssh/ssh_config` (igual que en Linux).

### 3.3 y 3.4 Uso del cliente y claves

**Igual** que en el libro. Hoy se recomienda el tipo de clave **Ed25519** en lugar de RSA:
```
ssh-keygen -t ed25519
```
Genera `~/.ssh/id_ed25519` (privada) e `id_ed25519.pub` (pública).

Copiar la clave pública al servidor (el método del libro sigue sirviendo):
```
cat ~/.ssh/id_ed25519.pub | ssh usuario@servidor 'cat >> ~/.ssh/authorized_keys'
```

---

## 4. OpenVPN

### 4.1 Concepto

El mismo que en el libro.

### 4.2 Instalación

| Linux | FreeBSD 15 |
|---|---|
| `apt-get install openvpn` / `yum install openvpn` | `pkg install openvpn` |

El dispositivo del túnel (`tun0`) lo proporciona el módulo del kernel `if_tun`, que se carga automáticamente.

### 4.3 Ficheros de configuración

| Linux | FreeBSD 15 |
|---|---|
| `/etc/openvpn/server.conf` | **`/usr/local/etc/openvpn/openvpn.conf`** (nombre que usa por defecto el script de arranque) |
| `/etc/openvpn/client.conf` | El mismo archivo en el cliente (o varios, ver abajo) |

**Activar como servicio** (en `/etc/rc.conf`):
```
openvpn_enable="YES"
openvpn_configfile="/usr/local/etc/openvpn/openvpn.conf"
```
Arrancar: `service openvpn start`.

Para tener **varios túneles** a la vez, se crea un enlace del script de arranque con otro nombre (ej. `ln -s /usr/local/etc/rc.d/openvpn /usr/local/etc/rc.d/openvpn_oficina`) y se configura con `openvpn_oficina_enable` y `openvpn_oficina_configfile`.

Las opciones de la Tabla 12.4 (`config`, `dev`, `nobind`, `ifconfig`, `secret`) son **las mismas**, y el ejemplo de `server.conf` del libro funciona igual cambiando la ruta. Recordatorio (apartado 1.3): para que los paquetes pasen de la VPN a la red local hace falta `gateway_enable="YES"`.

### 4.4 Métodos de cifrado

Notas de versión (afectan también a Linux):
- **Clave estática**: el modo `secret` del libro está **marcado como obsoleto** en OpenVPN 2.6 y en las versiones más recientes se ha retirado o requiere opciones especiales. En las versiones actuales la clave se genera con `openvpn --genkey secret secret.key`. Solo debe usarse para pruebas.
- **Clave pública (certificados)**: es el método recomendado. Los scripts del libro (`vars`, `build-ca`, `build-key-server`, `build-key`, `build-dh`) son de la versión antigua de **easy-rsa**. En FreeBSD se instala con `pkg install easy-rsa`, y los comandos actuales son:

| easy-rsa antiguo (libro) | easy-rsa 3 (actual) |
|---|---|
| `source vars` | `easyrsa init-pki` |
| `build-ca` | `easyrsa build-ca` |
| `build-key-server server` | `easyrsa build-server-full server nopass` |
| `build-key cliente` | `easyrsa build-client-full cliente nopass` |
| `build-dh` | `easyrsa gen-dh` |

### 4.5 Arranque manual y comprobación

Igual que en el libro: `openvpn /usr/local/etc/openvpn/openvpn.conf`, y comprobar el túnel con `ifconfig tun0`.

### 4.6 Otras VPN en FreeBSD 15

| VPN | Detalle |
|---|---|
| **WireGuard** | El módulo del kernel `if_wg` viene en el sistema base; las herramientas de configuración (`wg`, `wg-quick`) se instalan con el paquete `wireguard-tools` |
| **IPsec** | Soporte en el kernel del sistema base (herramienta `setkey`); para gestionar las conexiones se suele usar el paquete `strongswan` |

---

## 5. Escaneo de puertos y herramientas de auditoría

| Herramienta del libro | FreeBSD 15 |
|---|---|
| `telnet host puerto` | `nc -zv host puerto` (sistema base; comprueba si un puerto está abierto) |
| `nc` | `nc` (sistema base, igual) |
| `nmap` | `nmap` (paquete) |
| OpenVAS | No hay un paquete sencillo en FreeBSD. Lo habitual es ejecutarlo (Greenbone) en otra máquina o en contenedor, y escanear desde allí el servidor FreeBSD |

**Herramientas de auditoría propias de FreeBSD** (sin equivalente directo en el libro):

| Herramienta | Función |
|---|---|
| **`pkg audit -F`** | Descarga la base de datos de vulnerabilidades de FreeBSD (VuXML) y avisa de qué **paquetes instalados** tienen fallos de seguridad conocidos. Se ejecuta automáticamente cada día |
| **`freebsd-update IDS`** | Compara todos los archivos del **sistema base** con sus valores oficiales y avisa de cualquier archivo modificado (detecta manipulaciones) |
| **Informes de seguridad diarios** (`periodic security`) | Cada noche root recibe por correo: cambios en programas con SUID, intentos de login fallidos, paquetes bloqueados por el cortafuegos, cambios en los discos montados... Scripts en `/etc/periodic/security/`, configuración en `/etc/periodic.conf` |
| **Opciones de endurecimiento** | El instalador (`bsdinstall`) ofrece un menú de "System Hardening" (ocultar procesos de otros usuarios, borrar `/tmp` al arrancar, desactivar `sendmail`...). Se pueden cambiar después con `sysctl` (`security.bsd.*`) |
| **`securelevel`** | Nivel de seguridad del kernel: a partir del nivel 1 impide, por ejemplo, cargar módulos o modificar archivos protegidos, incluso siendo root (`kern_securelevel_enable="YES"` y `kern_securelevel="1"` en `/etc/rc.conf`) |
| **Jails** | Entornos aislados (propia IP, usuarios y procesos) para separar servicios. Configuración en `/etc/jail.conf` o `/etc/jail.conf.d/`, y se gestionan con `jail`, `jls` y `jexec` |

---

## 6. Sistemas de detección de intrusiones (IDS)

### 6.1 fail2ban y alternativas (IDS de host)

| Herramienta | FreeBSD 15 | Notas |
|---|---|---|
| **`blocklistd`** | **Sistema base** | Equivalente propio de FreeBSD a fail2ban. Los propios servicios (`sshd` con `UseBlocklist yes`, `ftpd`, etc.) le avisan de cada intento fallido y él bloquea la IP en `pf` o `ipfw`. En FreeBSD 15 se ha renombrado desde `blacklistd` (los nombres antiguos siguen funcionando con un aviso) |
| fail2ban | Paquete (buscar con `pkg search fail2ban`) | Funciona igual que en Linux, leyendo los logs |
| sshguard | Paquete `sshguard` | Alternativa ligera muy usada en FreeBSD |

**`blocklistd`:**

| Elemento | Detalle |
|---|---|
| Activar | `sysrc blocklistd_enable="YES"` + `service blocklistd start` |
| Configuración | `/etc/blocklistd.conf` (cuántos fallos se permiten y cuánto dura el bloqueo) |
| Integración con `pf` | Añadir en `/etc/pf.conf`: `anchor "blocklistd/*" in on $ext_if` |
| Integración con `sshd` | `UseBlocklist yes` en `/etc/ssh/sshd_config` |
| Ver las IPs bloqueadas | `blocklistctl dump -b` |

**fail2ban en FreeBSD:**

| Elemento | Linux | FreeBSD 15 |
|---|---|---|
| Configuración | `/etc/fail2ban/jail.conf` | **`/usr/local/etc/fail2ban/jail.conf`** (los cambios propios se ponen en `jail.local`) |
| Logs que vigila | `/var/log/auth.log`, logs de Apache... | `/var/log/auth.log` (SSH), `/var/log/httpd-error.log` (Apache)... |
| Cómo bloquea | `iptables` | `pf` o `ipfw` (se indica con `banaction = pf` o `ipfw`) |
| Activar | `systemctl enable fail2ban` | `sysrc fail2ban_enable="YES"` + `service fail2ban start` |

La limitación de los falsos positivos que menciona el libro es igual en todas estas herramientas.

### 6.2 Snort y alternativas (NIDS)

| Herramienta | FreeBSD 15 |
|---|---|
| Snort | Paquete `snort3`. Configuración en `/usr/local/etc/snort/`. En Snort 3 la configuración principal es un archivo Lua (`snort.lua`), pero las variables `HOME_NET` y `EXTERNAL_NET` y el formato de las reglas (`->`, `<>`) son los mismos del libro |
| Suricata | Paquete `suricata` (configuración en `/usr/local/etc/suricata/suricata.yaml`). Alternativa muy usada y compatible con las reglas de Snort |

**Puerto espejo (SPAN) con FreeBSD:** el libro indica que un NIDS necesita ver todo el tráfico. Si el equipo FreeBSD hace de puente (bridge), puede copiar todo el tráfico a otra interfaz sin necesidad de un switch con puerto espejo:
```
ifconfig bridge0 create
ifconfig bridge0 addm em0 addm em1 span em2 up
```
Todo lo que pasa entre `em0` y `em1` se copia a `em2`, donde escucha el NIDS.

---

## 7. Recursos de seguridad

Los recursos del libro, actualizados (afecta también a Linux):

| Recurso del libro | Situación actual |
|---|---|
| **US-CERT** | Integrado en la agencia **CISA** de EE.UU. La base de datos **NVD** la mantiene el NIST |
| **SANS Institute** | Sigue activo (incluye el SANS Internet Storm Center) |
| **Bugtraq** | **Cerrada** en 2021. Alternativas de divulgación completa: la lista `oss-security` y `Full Disclosure` |

**Recursos propios de FreeBSD:**

| Recurso | Descripción |
|---|---|
| **FreeBSD Security Advisories** (avisos `FreeBSD-SA-AA:NN`) | Avisos oficiales de fallos de seguridad del sistema base, con la solución (normalmente aplicar `freebsd-update fetch install`). Se publican en la web de seguridad de FreeBSD |
| **Errata Notices** (`FreeBSD-EN-AA:NN`) | Avisos de fallos importantes que no son de seguridad |
| Lista **freebsd-security-notifications** | Lista de correo que envía los avisos anteriores en cuanto se publican |
| **VuXML** | Base de datos de vulnerabilidades de los paquetes de terceros; es la que usa `pkg audit` |

---

## 8. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 12) | Equivalente en FreeBSD 15 |
|---|---|
| `net.ipv4.ip_forward=1` | `gateway_enable="YES"` |
| `iptables` | `pf` (`pfctl`, `/etc/pf.conf`) o `ipfw` |
| `ip6tables` | Las mismas reglas de `pf` / `ipfw` (IPv4 e IPv6 juntas) |
| Cadenas `INPUT` / `OUTPUT` / `FORWARD` | Reglas `in` / `out` en cada interfaz |
| Tabla `NAT` (`MASQUERADE`, `DNAT`) | `nat` / `rdr` en `pf`, o `ipfw nat` |
| `ACCEPT` / `DROP` / `REJECT` / `LOG` | `pass` / `block drop` / `block return` / `log` (en `pf`) |
| `iptables -L` | `pfctl -s rules` / `ipfw list` |
| `iptables -F` | `pfctl -F rules` / `ipfw -q flush` |
| `iptables-save` / `iptables-restore` | No hace falta: reglas en archivo, cargadas al arrancar |
| `sshd`, `/etc/ssh/sshd_config` | Iguales (sistema base) |
| `systemctl restart sshd` | `service sshd restart` |
| `PasswordAuthentication no` | `PasswordAuthentication no` **y** `KbdInteractiveAuthentication no` |
| `ssh-keygen -t rsa` | `ssh-keygen -t ed25519` (recomendado) |
| `/etc/openvpn/server.conf` | `/usr/local/etc/openvpn/openvpn.conf` |
| `build-ca`, `build-key`... (easy-rsa 2) | `easyrsa build-ca`, `easyrsa build-client-full`... (easy-rsa 3) |
| `telnet host puerto` | `nc -zv host puerto` |
| OpenVAS | `pkg audit`, `freebsd-update IDS` e informes `periodic security` (y OpenVAS en otra máquina) |
| fail2ban | `blocklistd` (sistema base), fail2ban o sshguard (paquetes) |
| `/etc/fail2ban/jail.conf` | `/usr/local/etc/fail2ban/jail.conf` (y `jail.local`) |
| Snort | `snort3` o `suricata` (paquetes) |
| Puerto espejo en el switch | `ifconfig bridge0 ... span em2` |
| Bugtraq / US-CERT | `oss-security` / CISA y NVD + FreeBSD Security Advisories |
| Chroot / aislamiento | Jails |

---

*Documento elaborado como contrapartida en FreeBSD 15 al Capítulo 12 ("Setting Up System Security") del libro LPIC-2. Fuentes: FreeBSD Handbook (capítulos "Firewalls", "Security" y "Jails"), notas de las versiones FreeBSD 15.0 y 15.1 (cambio de `blacklistd` a `blocklistd`) y páginas de manual de FreeBSD: pf.conf(5), pfctl(8), ipfw(8), rc.conf(5), sshd_config(5), blocklistd(8), blocklistd.conf(5), pkg-audit(8), freebsd-update(8), periodic.conf(5), security(7), jail(8), if_bridge(4), además de la documentación de OpenVPN, easy-rsa, fail2ban y Snort incluida en sus paquetes.*

---

## Nota final

Con este capítulo se completa la adaptación a FreeBSD 15 de los 12 capítulos del libro LPIC-2. Junto con las recopilaciones de Linux (Debian 13 / Rocky Linux 10) y las equivalencias de Windows Server 2025, tienes ahora un conjunto de documentos con la misma estructura, que permiten comparar capítulo a capítulo cómo se hace cada tarea en los tres sistemas.
