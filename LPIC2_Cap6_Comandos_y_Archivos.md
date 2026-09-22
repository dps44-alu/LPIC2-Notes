# LPIC-2 · Capítulo 6: Navigating Network Services
### Recopilación de comandos y archivos de configuración

Objetivos del examen cubiertos: 205.1 (Basic network configuration), 205.2 (Advanced Network Configuration), 205.3 (Troubleshooting network issues)

---

## 1. Conceptos básicos de red

Para que un sistema Linux funcione en red necesita cinco datos:

1. Dirección del host (IP)
2. Dirección de red (netmask)
3. Router por defecto (gateway)
4. Nombre de host (hostname)
5. Dirección de un servidor DNS

### 1.1 IPv6 (resumen)

- Direcciones de 128 bits, en hexadecimal, agrupadas en 8 bloques de 4 dígitos separados por `:` (ej. `fed1:0000:0000:08d3:1319:8a2e:0370:7334`).
- Se pueden comprimir los bloques a cero seguidos con `::` (solo una vez por dirección).
- **Link local**: se autoasigna con el prefijo `fe80::` más la parte derivada de la MAC; permite comunicación automática en la red local sin configuración.
- **Global**: funciona igual que IPv4, con red y host únicos asignados.

### 1.2 DHCP (Dynamic Host Configuration Protocol)

Evita configurar IPs a mano; el cliente pide dirección temporal y el servidor DHCP le asigna IP, máscara, gateway y DNS.

| Programa cliente DHCP | Notas |
|---|---|
| `dhcpcd` | El más popular actualmente |
| `dhclient` | Alternativa habitual |
| `pump` | Menos usado hoy en día |

---

## 2. Ficheros de configuración de red por distribución (Tabla 6.2)

| Distribución | Ubicación |
|---|---|
| **Debian** | Fichero `/etc/network/interfaces` |
| **Red Hat** | Directorio `/etc/sysconfig/network-scripts/` |
| OpenSUSE | Fichero `/etc/sysconfig/network` |

### 2.1 Debian: `/etc/network/interfaces`

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
dns-nameservers 192.168.0.103 192.168.0.1
```

| Directiva | Función |
|---|---|
| `auto eth0` | Activa la interfaz al arrancar |
| `iface eth0 inet dhcp` | Configuración dinámica (DHCP) |
| `iface eth0 inet static` | Configuración estática (requiere `address`, `netmask`, `gateway` adicionales) |
| `iface eth0 inet6 auto` | Dirección IPv6 automática (link local) |
| `dns-nameservers` | Servidores DNS a usar |

Fichero complementario para DHCP: `/etc/dhcp/dhclient.conf`, con la directiva `prepend domain-name-servers IP;` para forzar un servidor DNS concreto.

### 2.2 Red Hat: `/etc/sysconfig/network-scripts/`

**Fichero por interfaz** (`ifcfg-eth0`):
```
DEVICE="eth0"
NM_CONTROLLED="no"
ONBOOT=yes
TYPE=Ethernet
BOOTPROTO=static
NAME="System eth0"
IPADDR=192.168.1.77
NETMASK=255.255.255.0
IPV6INIT=yes
IPV6ADDR=2003:aef0::23d1::0a10:00a1/64
```

**Fichero general** (`/etc/sysconfig/network`):
```
NETWORKING=yes
HOSTNAME=mysystem
GATEWAY=192.168.1.254
IPV6FORWARDING=yes
IPV6_AUTOCONF=no
IPV6_DEFAULTGW=2003:aef0::23d1::0a10:0001
IPV6_DEFAULTDEV=eth0
```

Diferencia clave: en Red Hat, la IP/máscara van en `ifcfg-eth0` (por interfaz) y el hostname/gateway van en el fichero `network` (general); en Debian todo suele ir junto en `/etc/network/interfaces`.

### 2.3 Nombre de host y resolución DNS (comunes a ambas familias)

| Fichero | Función |
|---|---|
| `/etc/hostname` (o `/etc/HOSTNAME` en algunas distros) | Nombre de host del sistema |
| `/etc/resolv.conf` | Configuración del resolver DNS: `domain`, `search`, `nameserver` (se pueden poner varias líneas `nameserver`) |
| `/etc/hosts` | Resolución manual de nombres, consultada **antes** que el DNS |

---

## 3. Configuración por interfaz gráfica: Network Manager

- Se ejecuta automáticamente al arrancar y aparece como icono en el área de notificación.
- Icono de dos flechas = conexión cableada; icono de señal de radio = conexión inalámbrica.
- Permite configuración manual (IP fija) o por DHCP, y actualiza automáticamente los ficheros de configuración correspondientes.

---

## 4. Configuración por línea de comandos

### 4.1 Herramientas "clásicas": `ifconfig`, `iwconfig`, `route`

| Comando | Función |
|---|---|
| `ifconfig` (sin parámetros) | Lista las interfaces y su configuración actual |
| `ifconfig eth0 up IP netmask MASCARA` | Asigna IP y máscara, y activa la interfaz |
| `ifconfig eth0 down` | Desactiva la interfaz (o la deja asignada pero inactiva) |
| `ifup` / `ifdown` | Atajos para activar/desactivar una interfaz ya configurada |
| `iwlist interfaz scan` | Lista los puntos de acceso Wi-Fi detectados (ej. `iwlist wlan0 scan`) |
| `iwconfig interfaz essid "RED" key s:clave` | Configura el SSID y la clave de cifrado de una red Wi-Fi |
| `route` (sin parámetros) | Muestra la tabla de rutas actual |
| `route add default gw IP` | Define el router/gateway por defecto |
| `route add\|del destino gw gateway` | Añade o elimina una ruta concreta |

### 4.2 Herramientas modernas: `ip` e `iw`

| Comando | Función |
|---|---|
| `ip addr show` | Muestra IP, máscara y estado de las interfaces (sustituye a `ifconfig`) |
| `ip route show` | Muestra la tabla de rutas (sustituye a `route`) |
| `iw` | Equivalente moderno de `iwconfig`/`iwlist` para redes inalámbricas |

Recomendación del libro: si `ifconfig`/`iwconfig`/`route` no están disponibles en la distribución, usar `ip` e `iw`.

---

## 5. Comprobación y solución de problemas de red

### 5.1 Comprobación básica de conectividad

| Comando | Función |
|---|---|
| `ping host` / `ping6 host` | Envía paquetes ICMP para comprobar conectividad; `%eth0` tras una IPv6 link-local indica la interfaz de salida |
| `traceroute host` / `traceroute6 host` | Muestra los saltos (routers) por los que pasa el paquete hasta el destino, con el tiempo de cada salto |
| `nc` (netcat) | Simula un servidor (`nc -l puerto`) o un cliente (`nc IP puerto`) para probar transferencias de datos reales |

### 5.2 Resolución de nombres

| Comando | Función |
|---|---|
| `host nombre` | Consulta al DNS la IP asociada a un nombre (o el nombre asociado a una IP) |
| `dig nombre` | Muestra el detalle completo de la consulta DNS (todos los registros, tiempos, servidor consultado) |

### 5.3 Comprobación de conexiones abiertas

| Comando | Función |
|---|---|
| `lsof` | Lista ficheros abiertos; como en Linux las conexiones de red son ficheros, muestra también las sesiones de red activas |
| `netstat` | Estadísticas de puertos en escucha y conexiones activas (considerado obsoleto, sustituido por `ip` y `ss`) |
| `ss` | Alternativa moderna a `netstat`, muestra estadísticas de sockets |
| `arp` | Muestra la tabla ARP (IP ↔ dirección MAC) |

### 5.4 Escaneo de red

| Comando | Función |
|---|---|
| `nmap objetivo` | Escanea qué puertos están abiertos en un host remoto |

Advertencia del libro: usar `nmap` sin permiso del administrador de la red puede tener consecuencias legales/disciplinarias; muchas organizaciones lo prohíben expresamente.

### 5.5 Captura de tráfico

| Comando | Función |
|---|---|
| `tcpdump` | Pone la tarjeta en modo promiscuo y captura/muestra el tráfico de red que ve la interfaz |

### 5.6 Revisar los mensajes de arranque de la tarjeta de red

| Comando/archivo | Función |
|---|---|
| `dmesg` | Puede mostrar si el módulo de la tarjeta de red se cargó correctamente (ver Capítulo 1 y 3) |
| `/var/log/syslog` o `/var/log/messages` | Alternativa si el buffer de arranque ya se vació |

---

## 6. Control de acceso a servicios de red: TCP Wrappers

| Elemento | Detalle |
|---|---|
| Qué es | Programa que actúa de intermediario ante conexiones a aplicaciones de red, comparando la IP de origen contra listas |
| Requisito | La aplicación debe estar enlazada con `libwrap` (comprobar con `ldd /ruta/al/binario \| grep libwrap`) |
| `/etc/hosts.allow` | Lista blanca: direcciones permitidas |
| `/etc/hosts.deny` | Lista negra: direcciones bloqueadas |
| Orden de comprobación | 1º se consulta `hosts.allow` (si coincide, se permite y no se sigue mirando); 2º si no está ahí, se consulta `hosts.deny` (si coincide, se deniega; si tampoco está, se permite por defecto) |
| Formato de cada línea | `servicio: lista-de-clientes` (nombres de host, IPs o comodines, separados por comas) |
| Buena práctica | Dejar `hosts.deny` muy restrictivo (ej. `servicio: ALL`) y añadir solo los clientes necesarios en `hosts.allow` |
| Aplicar cambios | No requiere reiniciar el servicio; los cambios se aplican al momento |

---

## 7. Resumen de diferencias Debian vs Red Hat en este capítulo

| Aspecto | Debian | Red Hat |
|---|---|---|
| Fichero(s) de configuración de red | `/etc/network/interfaces` (todo junto) | `/etc/sysconfig/network-scripts/ifcfg-eth0` (IP/máscara) + `/etc/sysconfig/network` (hostname/gateway) |
| DHCP: fichero de opciones adicionales | `/etc/dhcp/dhclient.conf` | Puede variar según distribución |
| Resto del capítulo (`ip`, `ifconfig`, `route`, herramientas de troubleshooting, TCP Wrappers, `/etc/resolv.conf`, `/etc/hosts`) | Igual | Igual |

---

*Documento generado a partir del Capítulo 6 ("Navigating Network Services") del libro LPIC-2: Linux Professional Institute Certification Study Guide (Bresnahan & Blum).*
