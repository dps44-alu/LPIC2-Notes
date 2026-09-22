# LPIC-2 · Capítulo 6: Navigating Network Services
### Equivalencias en FreeBSD 15

Este documento recoge, para cada concepto visto en el Capítulo 6 del libro LPIC-2 (configuración básica y avanzada de red, resolución de problemas y control de acceso), cuál es su equivalente en **FreeBSD 15**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo: en FreeBSD toda la configuración de red permanente va en **un solo archivo, `/etc/rc.conf`**, igual que los servicios (Capítulo 1). Y la herramienta principal es **`ifconfig`**, que en FreeBSD **no está obsoleta**: es la herramienta oficial y actual. No existe el comando `ip` ni NetworkManager.

---

## 1. Conceptos básicos de red

Los cinco datos que necesita un equipo para funcionar en red son **los mismos** que en Linux: IP, máscara, puerta de enlace (gateway), nombre de host y servidor DNS.

**Nombres de las interfaces:** en FreeBSD el nombre de la tarjeta de red depende del **controlador** que usa, más un número:

| Ejemplo | Tipo de tarjeta |
|---|---|
| `em0`, `igb0`, `ix0` | Tarjetas Intel |
| `re0` | Tarjetas Realtek |
| `vtnet0` | Tarjeta virtual (KVM, bhyve, Proxmox) |
| `lo0` | Interfaz de loopback (`lo` en Linux) |
| `wlan0` | Interfaz Wi-Fi (se crea encima de la tarjeta física, ver apartado 4.1) |

Para ver los nombres: `ifconfig -l`.

### 1.1 IPv6

Los conceptos son **idénticos** (128 bits, `::`, direcciones link-local `fe80::` y globales). Detalles de FreeBSD:
- Las direcciones link-local se escriben con la interfaz tras `%`, igual que en Linux: `fe80::1%em0`.
- Para que las interfaces tengan IPv6 activo: `ipv6_activate_all_interfaces="YES"` en `/etc/rc.conf`.
- Para recibir la configuración automática del router (SLAAC): `ifconfig_em0_ipv6="inet6 accept_rtadv"` y `rtsold_enable="YES"`.

### 1.2 DHCP

| Linux | FreeBSD 15 | Notas |
|---|---|---|
| `dhclient` | **`dhclient`** (sistema base) | El cliente DHCP por defecto en FreeBSD. Se activa desde `/etc/rc.conf` (apartado 2) |
| `dhcpcd` | `dhcpcd` (paquete: `pkg install dhcpcd`) | Alternativa, útil sobre todo para DHCPv6 |
| `pump` | No existe | — |
| — | `rtsold` (sistema base) | Configuración automática IPv6 por anuncios del router |

Comando manual: `dhclient em0` (pide una IP por DHCP para la interfaz `em0`).

---

## 2. Ficheros de configuración de red (equivalente a la Tabla 6.2)

| Distribución | Ubicación |
|---|---|
| Debian | `/etc/network/interfaces` |
| Red Hat | `/etc/sysconfig/network-scripts/` |
| **FreeBSD 15** | **`/etc/rc.conf`** (todo en el mismo archivo: IP, gateway, hostname, rutas, Wi-Fi) |

### 2.1 Configuración en `/etc/rc.conf`

**Ejemplo con DHCP** (equivalente al ejemplo de Debian del libro):
```
hostname="mysystem.example.com"
ifconfig_em0="DHCP"
ifconfig_em0_ipv6="inet6 accept_rtadv"
```

**Ejemplo con IP fija** (equivalente al ejemplo de Red Hat del libro):
```
hostname="mysystem.example.com"
ifconfig_em0="inet 192.168.1.77 netmask 255.255.255.0"
defaultrouter="192.168.1.254"
ifconfig_em0_ipv6="inet6 2001:db8::10 prefixlen 64"
ipv6_defaultrouter="2001:db8::1"
```

También se puede escribir la máscara en formato CIDR: `ifconfig_em0="inet 192.168.1.77/24"`.

| Variable de `rc.conf` | Equivalente Debian / Red Hat | Función |
|---|---|---|
| `hostname="..."` | `/etc/hostname` / `HOSTNAME=` | Nombre del equipo |
| `ifconfig_em0="DHCP"` | `iface eth0 inet dhcp` / `BOOTPROTO=dhcp` | Configuración por DHCP |
| `ifconfig_em0="SYNCDHCP"` | — | DHCP, pero **esperando** a tener IP antes de seguir arrancando (útil si otros servicios la necesitan) |
| `ifconfig_em0="inet IP netmask MÁSCARA"` | `iface eth0 inet static` + `address` / `IPADDR=` + `NETMASK=` | IP fija |
| `defaultrouter="IP"` | `gateway` / `GATEWAY=` | Puerta de enlace IPv4 |
| `ifconfig_em0_ipv6="inet6 IP prefixlen 64"` | `IPV6ADDR=` | IPv6 fija |
| `ifconfig_em0_ipv6="inet6 accept_rtadv"` | `iface eth0 inet6 auto` / `IPV6_AUTOCONF=yes` | IPv6 automática |
| `ipv6_defaultrouter="IP"` | `IPV6_DEFAULTGW=` | Puerta de enlace IPv6 |
| `ifconfig_em0_alias0="inet 192.168.1.78/32"` | `eth0:0` / `IPADDR2=` | Segunda IP en la misma tarjeta |
| `gateway_enable="YES"` | `net.ipv4.ip_forward=1` | Reenviar paquetes entre interfaces (actuar como router) |
| `ipv6_gateway_enable="YES"` | `IPV6FORWARDING=yes` | Igual, para IPv6 |
| `static_routes="red1"` + `route_red1="-net 10.0.0.0/8 192.168.1.1"` | `route add` en scripts / `route-eth0` | Rutas estáticas permanentes |
| `ifconfig_em0_name="lan0"` | Reglas de `udev` | Cambiar el nombre de la interfaz |
| No se escribe nada | `auto eth0` / `ONBOOT=yes` | En FreeBSD, toda interfaz con línea `ifconfig_` se activa al arrancar |

Se pueden añadir estas líneas con `sysrc` en lugar de editar el archivo (ej. `sysrc defaultrouter="192.168.1.254"`).

**Aplicar los cambios sin reiniciar:**

| Comando | Función |
|---|---|
| `service netif restart` | Vuelve a configurar todas las interfaces |
| `service netif restart em0` | Solo la interfaz `em0` |
| `service routing restart` | Vuelve a aplicar las rutas y el gateway (hacerlo siempre después de `netif restart`) |

**Opciones de DHCP** (equivalente a `/etc/dhcp/dhclient.conf`): el archivo es **`/etc/dhclient.conf`**, con la misma sintaxis que en Debian:
```
interface "em0" {
    prepend domain-name-servers 192.168.0.103;
}
```

### 2.2 Nombre de host y resolución DNS

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `/etc/hostname` | `hostname="..."` en `/etc/rc.conf` | Nombre del equipo permanente |
| `hostname nuevo` / `hostnamectl` | `hostname nuevo` | Cambia el nombre hasta el siguiente reinicio |
| `/etc/resolv.conf` | `/etc/resolv.conf` (igual: `domain`, `search`, `nameserver`) | Servidores DNS |
| — | `/etc/resolvconf.conf` + comando `resolvconf` | Gestiona `resolv.conf` automáticamente (lo usan `dhclient` y las VPN). Si se quiere escribir `resolv.conf` a mano sin que se sobrescriba, poner `resolvconf=NO` en `/etc/resolvconf.conf` |
| `/etc/hosts` | `/etc/hosts` (igual) | Resolución manual de nombres |
| `/etc/nsswitch.conf` | `/etc/nsswitch.conf` (igual) | Orden de consulta (línea `hosts: files dns`: primero `/etc/hosts`, luego DNS) |

FreeBSD incluye además **`local-unbound`**, un servidor DNS de caché local en el sistema base, que se puede activar con `sysrc local_unbound_enable="YES"`.

---

## 3. Configuración por interfaz gráfica: Network Manager

**NetworkManager no existe en FreeBSD.** Alternativas:

| Herramienta | Tipo | Función |
|---|---|---|
| `bsdconfig networking` | Menús en modo texto (sistema base) | Configura hostname, interfaces, gateway y DNS; escribe los cambios en `/etc/rc.conf` |
| `bsdinstall netconfig` | Menús en modo texto (sistema base) | El asistente de red del instalador, reutilizable después |
| `networkmgr` | Gráfico (paquete) | Icono en la barra del escritorio para redes cableadas y Wi-Fi, parecido a NetworkManager |

---

## 4. Configuración por línea de comandos

### 4.1 Herramientas "clásicas" (`ifconfig`, `iwconfig`, `route`)

En FreeBSD las herramientas clásicas son las **actuales y oficiales**. `ifconfig` gestiona también el Wi-Fi (no hay `iwconfig` ni `iw`).

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `ifconfig` | `ifconfig` | Lista las interfaces y su configuración |
| `ifconfig eth0 up IP netmask MÁSCARA` | `ifconfig em0 inet 192.168.1.77/24 up` | Asigna IP y activa la interfaz (temporal, hasta reiniciar) |
| — | `ifconfig em0 inet 192.168.1.78/32 alias` | Añade una segunda IP |
| — | `ifconfig em0 inet 192.168.1.78 -alias` | Quita una IP concreta |
| `ifconfig eth0 down` | `ifconfig em0 down` | Desactiva la interfaz |
| `ifup` / `ifdown` | `service netif start em0` / `service netif stop em0` | Activa o desactiva una interfaz con la configuración de `rc.conf` |
| `ifconfig -a` | `ifconfig -a` | Todas las interfaces |
| — | `ifconfig -l` | Solo los nombres de las interfaces |
| — | `ifconfig em0 inet6 2001:db8::10/64` | Asigna una IPv6 |

**Wi-Fi** (equivalente a `iwlist` / `iwconfig`): en FreeBSD la tarjeta física (ej. `iwlwifi0`, `iwm0`, `rtw880`) no se usa directamente, sino que se crea encima una interfaz **`wlan0`**.

| Linux | FreeBSD 15 | Función |
|---|---|---|
| — | `sysctl net.wlan.devices` | Muestra las tarjetas Wi-Fi detectadas |
| — | `ifconfig wlan0 create wlandev iwlwifi0` | Crea la interfaz `wlan0` sobre la tarjeta física |
| `iwlist wlan0 scan` | `ifconfig wlan0 up scan` | Busca redes Wi-Fi |
| — | `ifconfig wlan0 list scan` | Muestra el resultado de la última búsqueda |
| `iwconfig wlan0 essid "RED"` | `ifconfig wlan0 ssid "RED"` | Conecta a una red abierta |
| `iwconfig ... key` (WEP) | `wpa_supplicant` (sistema base) | Redes protegidas (WPA2/WPA3): ver abajo |
| — | `wpa_cli` | Consultar y controlar `wpa_supplicant` en marcha |

**Configuración permanente de Wi-Fi con WPA:**

En `/etc/rc.conf`:
```
wlans_iwlwifi0="wlan0"
ifconfig_wlan0="WPA SYNCDHCP"
```
En `/etc/wpa_supplicant.conf`:
```
network={
    ssid="MiRed"
    psk="clave_de_la_red"
}
```
Aplicar: `service netif restart wlan0`.

**Rutas** (equivalente a `route`):

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `route` (sin parámetros) | `netstat -rn` | Muestra la tabla de rutas (con `route` sin parámetros no se ve nada en FreeBSD) |
| `route add default gw IP` | `route add default 192.168.1.254` | Define el gateway (temporal). **Sin la palabra `gw`** |
| `route add -net red gw IP` | `route add -net 10.0.0.0/8 192.168.1.1` | Añade una ruta |
| `route del ...` | `route delete -net 10.0.0.0/8` | Elimina una ruta |
| — | `route -n get default` | Muestra qué gateway e interfaz se usan para una dirección |
| — | `route -6 add default 2001:db8::1` | Gateway IPv6 |
| — | `route flush` | Borra todas las rutas (cuidado en conexiones remotas) |

Los cambios hechos con `ifconfig` y `route` se pierden al reiniciar; para hacerlos permanentes, se escriben en `/etc/rc.conf` (apartado 2.1).

### 4.2 Herramientas "modernas" de Linux: `ip` e `iw`

**No existen en FreeBSD.** Su equivalencia:

| Linux | FreeBSD 15 |
|---|---|
| `ip addr show` | `ifconfig` |
| `ip link set eth0 up` | `ifconfig em0 up` |
| `ip route show` | `netstat -rn` |
| `ip route add` | `route add` |
| `ip neigh` | `arp -a` (IPv4) / `ndp -a` (IPv6) |
| `ip -s link` | `netstat -i` / `netstat -I em0 -w 1` |
| `iw` | `ifconfig wlan0 ...` + `wpa_supplicant` |

---

## 5. Comprobación y solución de problemas de red

### 5.1 Comprobación básica de conectividad

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `ping host` | `ping host` | Igual. En FreeBSD, `ping` sin opciones sigue hasta que se pulsa `Ctrl+C`; con `-c 4` envía solo 4 paquetes |
| `ping6 host` | `ping -6 host` | IPv6 (el mismo comando `ping` sirve para las dos versiones) |
| `ping6 fe80::1%eth0` | `ping -6 fe80::1%em0` | Link-local indicando la interfaz |
| `traceroute host` | `traceroute host` (igual, sistema base) | Saltos hasta el destino |
| `traceroute6 host` | `traceroute6 host` (igual, sistema base) | Saltos en IPv6 |
| `nc` | `nc` (sistema base) | Igual: `nc -l 5000` (servidor) y `nc IP 5000` (cliente) |

### 5.2 Resolución de nombres

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `host nombre` | `host nombre` (sistema base) | Consulta rápida al DNS |
| `dig nombre` | `drill nombre` (sistema base) | Consulta detallada (misma información que `dig`, formato muy parecido) |
| `dig` | `dig` (paquete `bind-tools`) | Si se prefiere exactamente la misma herramienta |
| `nslookup` | `nslookup` (paquete `bind-tools`) | — |
| `getent hosts nombre` | `getent hosts nombre` | Resuelve el nombre como lo hace el sistema (mirando también `/etc/hosts`) |

### 5.3 Comprobación de conexiones abiertas

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `lsof -i` | `sockstat` (sistema base) o `lsof` (paquete) | Qué programa tiene abierta cada conexión o puerto |
| — | `sockstat -4 -l` | Puertos IPv4 a la escucha, con usuario, programa y PID |
| — | `sockstat -c` | Solo conexiones establecidas |
| `netstat` | `netstat` (sistema base, **no obsoleto**) | Conexiones y estadísticas |
| — | `netstat -an` | Todas las conexiones y puertos, sin resolver nombres |
| — | `netstat -s` | Estadísticas por protocolo (errores, paquetes perdidos...) |
| `ss` | `sockstat` | Estadísticas de sockets |
| `arp` | `arp -a` | Tabla ARP (IP ↔ MAC) |
| `ip -6 neigh` | `ndp -a` | Tabla de vecinos IPv6 (el "ARP" de IPv6) |

### 5.4 Escaneo de red

| Linux | FreeBSD 15 |
|---|---|
| `nmap objetivo` | `nmap objetivo` (paquete: `pkg install nmap`) |

La misma advertencia del libro: no escanear redes sin permiso.

### 5.5 Captura de tráfico

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `tcpdump` | `tcpdump` (sistema base, igual) | Captura el tráfico. Ej.: `tcpdump -i em0 port 80` |
| — | `tshark` / `wireshark` (paquetes) | Analizadores más completos |

### 5.6 Revisar los mensajes de arranque de la tarjeta de red

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `dmesg` | `dmesg \| grep em0` | Mensajes del controlador de red |
| `/var/log/syslog`, `/var/log/messages` | `/var/run/dmesg.boot` y `/var/log/messages` | Mensajes del arranque y del sistema |
| — | `pciconf -lv \| grep -B3 network` | Comprueba que la tarjeta se detecta y qué controlador usa |
| — | `ifconfig em0` (línea `status:`) | `active` = cable conectado; `no carrier` = sin cable o sin enlace |

---

## 6. Control de acceso a servicios de red: TCP Wrappers

FreeBSD **mantiene TCP Wrappers en el sistema base**, pero con dos diferencias importantes respecto al libro:

**1. Se usa un solo archivo, `/etc/hosts.allow`**, con una tercera columna que dice si se permite o se deniega. **No se usa `/etc/hosts.deny`.**

| Linux | FreeBSD 15 |
|---|---|
| `/etc/hosts.allow` (lista blanca) | `/etc/hosts.allow` con reglas `: allow` |
| `/etc/hosts.deny` (lista negra) | `/etc/hosts.allow` con reglas `: deny` |
| Formato: `servicio: clientes` | Formato: `servicio : clientes : allow` o `servicio : clientes : deny` |
| Orden: primero `hosts.allow`, luego `hosts.deny` | Las reglas se leen **de arriba abajo** y se aplica **la primera que coincida** |

Ejemplo de `/etc/hosts.allow` (equivalente a la buena práctica del libro: denegar todo salvo lo necesario):
```
# Permitir el servicio ftpd solo desde la red local
ftpd : 192.168.1.0/255.255.255.0 : allow
ftpd : ALL : deny

# Permitir el resto de servicios a todos
ALL : ALL : allow
```

Igual que en Linux, los cambios se aplican al momento, sin reiniciar el servicio.

**2. Qué servicios lo usan:**

| Servicio | ¿Usa TCP Wrappers? |
|---|---|
| Servicios lanzados por **`inetd`** | **Sí**. Viene activado por defecto con las opciones `-wW` (`inetd_flags="-wW -C 60"` en `/etc/defaults/rc.conf`) |
| Otros programas enlazados con `libwrap` | Sí (comprobar con `ldd /ruta/al/binario \| grep libwrap`, igual que en Linux) |
| **`sshd`** | **No se debe contar con ello**: OpenSSH eliminó el soporte de TCP Wrappers hace años. Para limitar el acceso a SSH se usan las opciones `AllowUsers` / `AllowGroups` / `Match Address` de `/etc/ssh/sshd_config` o un cortafuegos |

Para un control de acceso de red más completo, en FreeBSD se usan los cortafuegos del sistema base (**`pf`** o **`ipfw`**), que se tratan en el Capítulo 12.

---

## 7. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 6) | Equivalente en FreeBSD 15 |
|---|---|
| `eth0`, `enp0s3` | Nombre según el controlador: `em0`, `re0`, `vtnet0`... |
| `/etc/network/interfaces` (Debian) | `/etc/rc.conf` |
| `/etc/sysconfig/network-scripts/` + `/etc/sysconfig/network` (Red Hat) | `/etc/rc.conf` |
| `iface eth0 inet dhcp` | `ifconfig_em0="DHCP"` |
| `iface eth0 inet static` + `address`/`netmask` | `ifconfig_em0="inet IP netmask MÁSCARA"` |
| `gateway` / `GATEWAY=` | `defaultrouter="IP"` |
| `/etc/hostname` | `hostname="..."` en `/etc/rc.conf` |
| `/etc/dhcp/dhclient.conf` | `/etc/dhclient.conf` |
| `/etc/resolv.conf`, `/etc/hosts` | Iguales (+ `resolvconf` y `local-unbound`) |
| NetworkManager | `bsdconfig networking` o `networkmgr` (paquete) |
| `ip addr` / `ifconfig` | `ifconfig` (herramienta oficial) |
| `ifup` / `ifdown` | `service netif start/stop em0` |
| `systemctl restart networking` | `service netif restart` + `service routing restart` |
| `ip route` / `route` | `netstat -rn` / `route add default IP` (sin `gw`) |
| `iwlist` / `iwconfig` / `iw` | `ifconfig wlan0 ...` + `wpa_supplicant` |
| `ping6` | `ping -6` |
| `dig` | `drill` (sistema base) o `dig` (paquete `bind-tools`) |
| `ss` / `lsof -i` | `sockstat` |
| `netstat` | `netstat` (no obsoleto) |
| `ip neigh` / `arp` | `arp -a` / `ndp -a` |
| `nmap` | `nmap` (paquete) |
| `tcpdump` | `tcpdump` (igual) |
| `/etc/hosts.allow` + `/etc/hosts.deny` | Solo `/etc/hosts.allow` con `: allow` / `: deny` |

---

*Documento elaborado como contrapartida en FreeBSD 15 al Capítulo 6 ("Navigating Network Services") del libro LPIC-2. Fuentes: FreeBSD Handbook (capítulos "Network" / "Advanced Networking" y "Wireless Networking") y páginas de manual de FreeBSD: rc.conf(5), ifconfig(8), route(8), netstat(1), sockstat(1), dhclient(8), dhclient.conf(5), resolv.conf(5), resolvconf(8), wpa_supplicant(8), wpa_supplicant.conf(5), ping(8), drill(1), arp(8), ndp(8), hosts_access(5), hosts_options(5), inetd(8).*
