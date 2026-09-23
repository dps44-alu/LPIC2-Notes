# LPIC-2 · Capítulo 6: Navigating Network Services
### Equivalencias en Windows Server 2025

Este documento recoge, para cada concepto visto en el Capítulo 6 del libro LPIC-2 (configuración básica y avanzada de red, resolución de problemas y control de acceso), cuál es su equivalente en **Windows Server 2025**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo: en Windows **no hay archivos de texto de configuración de red** como `/etc/network/interfaces` o `ifcfg-eth0`. La configuración se guarda en el **Registro** y se cambia con:
- **PowerShell** (módulos `NetAdapter`, `NetTCPIP` y `DnsClient`): la forma recomendada. Es el equivalente al comando `ip`.
- **`netsh`**: herramienta clásica de texto, todavía muy usada.
- **`ipconfig`** y **`route`**: herramientas clásicas para consultar (parecidas a `ifconfig` y `route`).
- **`sconfig`** (menú de texto, útil en Server Core) y las herramientas gráficas.

Todos los comandos que cambian la configuración se ejecutan en una consola **como Administrador**.

---

## 1. Conceptos básicos de red

Los cinco datos que necesita un equipo para funcionar en red son **los mismos** que en Linux: IP, máscara (en Windows se suele indicar como **longitud de prefijo**, por ejemplo `24` = `255.255.255.0`), puerta de enlace predeterminada, nombre del equipo y servidor DNS.

**Nombres de las interfaces:** no hay `eth0` ni `enp0s3`. Cada tarjeta tiene:

| Identificador | Ejemplo | Uso |
|---|---|---|
| **Alias** (nombre visible) | `Ethernet`, `Ethernet 2` | Se usa en casi todos los comandos (`-InterfaceAlias`). Se puede cambiar: `Rename-NetAdapter -Name "Ethernet" -NewName "LAN"` |
| **Índice** | `12` | Número interno de la interfaz (`-InterfaceIndex`). También aparece en las direcciones IPv6 de enlace local |
| **Descripción** | `Intel(R) Ethernet Controller X710` | Nombre del hardware |

Se ven todos con `Get-NetAdapter`.

### 1.1 IPv6

Los conceptos del libro son los mismos. Particularidades de Windows:
- IPv6 viene **activado por defecto** y Windows lo **prefiere** frente a IPv4 cuando ambos están disponibles.
- Las direcciones de enlace local (`fe80::`) llevan detrás el **índice** de la interfaz, no su nombre: `fe80::1c2d:3e4f:5a6b:7c8d%12` (en Linux sería `%eth0`).
- Microsoft **no recomienda desactivar IPv6**, porque algunos componentes de Windows lo usan internamente. Si se quiere que Windows prefiera IPv4, se hace con el valor de Registro `DisabledComponents = 0x20` en `HKLM\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters` (requiere reiniciar).

### 1.2 DHCP

El cliente DHCP viene **integrado** en Windows como servicio (**Cliente DHCP**, nombre `Dhcp`). No hay que elegir entre programas como `dhcpcd` o `dhclient`.

| Comando Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `dhclient -r eth0` | `ipconfig /release` (o `ipconfig /release "Ethernet"`) | Libera la dirección obtenida por DHCP |
| `dhclient eth0` | `ipconfig /renew` | Pide una dirección nueva al servidor DHCP |
| `dhclient -6` | `ipconfig /release6` / `ipconfig /renew6` | Lo mismo en IPv6 |

Si el equipo está configurado por DHCP y no encuentra servidor, Windows se asigna sola una dirección **APIPA** del rango `169.254.x.x` (autoconfiguración). Ver una dirección de ese rango suele indicar un problema con el DHCP.

---

## 2. Configuración de red (equivalente a la Tabla 6.2)

| Distribución Linux | Ubicación | Equivalente en Windows Server 2025 |
|---|---|---|
| Debian | `/etc/network/interfaces` | Registro: `HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\{GUID}` (IPv4) y `...\Tcpip6\...` (IPv6). **No se edita a mano** |
| Red Hat | `/etc/sysconfig/network-scripts/` | Igual |

Cada interfaz tiene en esa clave valores como `EnableDHCP`, `IPAddress`, `SubnetMask`, `DefaultGateway` y `NameServer`, que son muy parecidos a las variables de un `ifcfg-eth0`, pero se cambian siempre con las herramientas.

### 2.1 Configurar una interfaz (equivalente a `/etc/network/interfaces` e `ifcfg-eth0`)

**Dirección estática** (equivalente a `iface eth0 inet static` o `BOOTPROTO=static`):
```
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.1.77 -PrefixLength 24 -DefaultGateway 192.168.1.254
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.0.103,192.168.0.1
```

**Dirección por DHCP** (equivalente a `iface eth0 inet dhcp` o `BOOTPROTO=dhcp`):
```
Set-NetIPInterface -InterfaceAlias "Ethernet" -Dhcp Enabled
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ResetServerAddresses
```
Si la interfaz tenía antes una IP fija, hay que borrar también la dirección y la puerta de enlace antiguas: `Remove-NetIPAddress -InterfaceAlias "Ethernet" -Confirm:$false` y `Remove-NetRoute -InterfaceAlias "Ethernet" -DestinationPrefix 0.0.0.0/0 -Confirm:$false`.

**Dirección IPv6 estática** (equivalente a `IPV6ADDR=` e `IPV6_DEFAULTGW=`):
```
New-NetIPAddress -InterfaceAlias "Ethernet" -AddressFamily IPv6 -IPAddress 2001:db8:10::a1 -PrefixLength 64 -DefaultGateway 2001:db8:10::1
```

**Lo mismo con `netsh`** (herramienta clásica):

| Comando | Función |
|---|---|
| `netsh interface ipv4 set address name="Ethernet" static 192.168.1.77 255.255.255.0 192.168.1.254` | IP, máscara y puerta de enlace fijas |
| `netsh interface ipv4 set address name="Ethernet" source=dhcp` | Pasar a DHCP |
| `netsh interface ipv4 set dnsservers name="Ethernet" static 192.168.0.103 primary` | Primer servidor DNS |
| `netsh interface ipv4 add dnsservers name="Ethernet" 192.168.0.1 index=2` | Segundo servidor DNS |
| `netsh interface ipv4 set dnsservers name="Ethernet" source=dhcp` | DNS por DHCP |
| `netsh interface ipv6 add address "Ethernet" 2001:db8:10::a1/64` | Dirección IPv6 |

**Equivalencia de las directivas del libro:**

| Directiva Linux | Equivalente en Windows Server 2025 |
|---|---|
| `auto eth0` / `ONBOOT=yes` | Las interfaces están activas por defecto. Activar / desactivar: `Enable-NetAdapter` / `Disable-NetAdapter` |
| `iface eth0 inet6 auto` / `IPV6_AUTOCONF=yes` | Activado por defecto. Desactivar la autoconfiguración: `Set-NetIPInterface -InterfaceAlias "Ethernet" -AddressFamily IPv6 -RouterDiscovery Disabled` |
| `dns-nameservers` | `Set-DnsClientServerAddress` |
| `prepend domain-name-servers` en `dhclient.conf` (forzar un DNS aunque se use DHCP) | Configurar el DNS a mano con `Set-DnsClientServerAddress`: los DNS fijos tienen prioridad sobre los que da el DHCP, aunque la IP siga siendo por DHCP |
| `IPV6FORWARDING=yes` | `Set-NetIPInterface -AddressFamily IPv6 -Forwarding Enabled` (Capítulo 3) |
| `NM_CONTROLLED="no"` | No aplica |

**Un "archivo de configuración" con `netsh`:** aunque Windows no usa archivos de configuración de red, `netsh` puede **exportar** toda la configuración a un archivo de texto y volver a aplicarla. Es útil como copia o para repetir la configuración en otro servidor:

| Comando | Función |
|---|---|
| `netsh interface dump > C:\red.txt` | Guarda la configuración de las interfaces como una lista de comandos `netsh` |
| `netsh exec C:\red.txt` | Vuelve a aplicarla |

**En Server Core:** `sconfig` → opción **Configuración de red** permite elegir la tarjeta y poner IP fija o DHCP y los servidores DNS con menús.

### 2.2 Nombre de host y resolución DNS

| Archivo Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `/etc/hostname` | `Rename-Computer -NewName SRV01 -Restart` (o `sconfig` → opción de nombre del equipo). Consultar: `hostname` o `$env:COMPUTERNAME` | Nombre del equipo. Cambiarlo **requiere reiniciar** |
| `nameserver` en `/etc/resolv.conf` | `Set-DnsClientServerAddress` (por interfaz). Consultar: `Get-DnsClientServerAddress` | Servidores DNS |
| `search` en `/etc/resolv.conf` | `Set-DnsClientGlobalSetting -SuffixSearchList @("empresa.local","sucursal.empresa.local")` | Lista de sufijos que se añaden a los nombres cortos |
| `domain` en `/etc/resolv.conf` | Sufijo DNS principal del equipo (se asigna al unirse a un dominio de Active Directory) y sufijo por conexión: `Set-DnsClient -InterfaceAlias "Ethernet" -ConnectionSpecificSuffix empresa.local` | Dominio del equipo |
| `/etc/hosts` | **`C:\Windows\System32\drivers\etc\hosts`** (mismo formato que en Linux: `IP nombre`) | Resolución manual, consultada antes que el DNS |
| `/etc/services`, `/etc/protocols`, `/etc/networks` | Mismos archivos en `C:\Windows\System32\drivers\etc\` | Nombres de puertos, protocolos y redes |

Ver toda la configuración de una vez (equivalente a mirar `/etc/resolv.conf` y la IP juntas): `ipconfig /all` o `Get-NetIPConfiguration -Detailed`.

**Caché DNS del cliente:** a diferencia de muchas distribuciones Linux, Windows **siempre guarda en caché** las respuestas DNS (servicio **Cliente DNS**, `Dnscache`). Las entradas del archivo `hosts` también se cargan en esa caché.

| Comando | Función |
|---|---|
| `ipconfig /displaydns` o `Get-DnsClientCache` | Muestra la caché |
| `ipconfig /flushdns` o `Clear-DnsClientCache` | Vacía la caché (útil tras cambiar un registro DNS o el archivo `hosts`) |
| `ipconfig /registerdns` | Vuelve a registrar el nombre del equipo en el DNS (en dominios con DNS dinámico) |

Windows Server 2025 también puede usar **DNS cifrado** (DNS sobre HTTPS, DoH) como cliente si el servidor DNS lo admite (`Get-DnsClientDohServerAddress` / `Add-DnsClientDohServerAddress`).

---

## 3. Configuración por interfaz gráfica (equivalente a Network Manager)

| Herramienta | Cómo se abre | Disponible en |
|---|---|---|
| **Conexiones de red** | `ncpa.cpl` → clic derecho en la tarjeta → Propiedades → "Protocolo de Internet versión 4 (TCP/IPv4)" | Experiencia de escritorio |
| **Administrador del servidor** | Servidor local → clic en el enlace de la tarjeta (junto a "Ethernet") | Experiencia de escritorio |
| **Configuración** | Configuración → Red e Internet | Experiencia de escritorio |
| **Windows Admin Center** | Herramienta **Redes** | Administración remota desde el navegador |
| **`sconfig`** | Menú de texto | Server Core y Experiencia de escritorio |

**Perfil de red (importante en Windows):** cada conexión tiene una **categoría** que decide qué reglas del firewall se aplican (apartado 6): **Dominio** (red de la empresa, detectada automáticamente al estar unido a un dominio), **Privada** o **Pública** (la más restrictiva, y la que se pone por defecto a una red desconocida).

| Comando | Función |
|---|---|
| `Get-NetConnectionProfile` | Muestra la categoría de cada conexión |
| `Set-NetConnectionProfile -InterfaceAlias "Ethernet" -NetworkCategory Private` | La cambia a Privada (no se puede poner "Dominio" a mano) |

---

## 4. Configuración por línea de comandos

### 4.1 Herramientas "clásicas" (`ifconfig`, `iwconfig`, `route`)

| Comando Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `ifconfig` | `ipconfig` / `ipconfig /all` | Muestra la configuración (`/all` añade MAC, DHCP, DNS...). `ipconfig` **solo consulta**, no cambia nada |
| `ifconfig eth0 IP netmask MÁSCARA up` | `netsh interface ipv4 set address name="Ethernet" static IP MÁSCARA` | Asigna IP y máscara |
| `ifconfig eth0 down` / `up` | `netsh interface set interface "Ethernet" admin=disabled` / `admin=enabled` | Desactiva / activa la interfaz |
| `ifdown` / `ifup` | Igual que la fila anterior | — |
| `iwlist wlan0 scan` | `netsh wlan show networks` | Redes Wi-Fi visibles |
| `iwconfig wlan0 essid "RED" key ...` | `netsh wlan add profile filename="perfil.xml"` + `netsh wlan connect name="RED"` | Conectarse a una red Wi-Fi |
| `route` | `route print` | Tabla de rutas |
| `route add default gw IP` | `route -p add 0.0.0.0 mask 0.0.0.0 192.168.1.254` | Puerta de enlace predeterminada |
| `route add destino gw gateway` | `route -p add 10.0.0.0 mask 255.0.0.0 192.168.1.1` | Añade una ruta. **`-p` la hace permanente**; sin `-p` se pierde al reiniciar |
| `route del destino` | `route delete 10.0.0.0` | Borra una ruta |
| `arp -a` | `arp -a` (igual) | Tabla ARP |

Las rutas permanentes creadas con `route -p` se guardan en el Registro, en `HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\PersistentRoutes`.

**Wi-Fi en un servidor:** poco habitual. En Windows Server la función inalámbrica no viene activada; hay que instalar la característica **Servicio WLAN** (`Install-WindowsFeature Wireless-Networking`) para que funcionen los comandos `netsh wlan`.

### 4.2 Herramientas "modernas": el equivalente de `ip` e `iw`

El equivalente de `ip` son los cmdlets de PowerShell de los módulos **NetAdapter** y **NetTCPIP**:

| Comando `ip` | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `ip link show` | `Get-NetAdapter` | Tarjetas con su estado, velocidad y MAC |
| `ip link set eth0 up` / `down` | `Enable-NetAdapter -Name "Ethernet"` / `Disable-NetAdapter -Name "Ethernet"` | Activa / desactiva |
| — | `Restart-NetAdapter -Name "Ethernet"` | Desactiva y vuelve a activar (como `ifdown` + `ifup`) |
| `ip link set eth0 name lan` | `Rename-NetAdapter -Name "Ethernet" -NewName "LAN"` | Cambia el nombre |
| `ip link set eth0 mtu 9000` | `Set-NetIPInterface -InterfaceAlias "Ethernet" -NlMtuBytes 9000` (y activar las tramas grandes en el controlador: `Set-NetAdapterAdvancedProperty -Name "Ethernet" -RegistryKeyword "*JumboPacket" -RegistryValue 9014`) | Cambia la MTU |
| `ip -s link` | `Get-NetAdapterStatistics` | Estadísticas de tráfico y errores |
| `ip addr show` | `Get-NetIPAddress` o `Get-NetIPConfiguration` | Direcciones IP |
| `ip addr add 192.168.1.77/24 dev eth0` | `New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.1.77 -PrefixLength 24` | Añade una dirección |
| `ip addr del ...` | `Remove-NetIPAddress -IPAddress 192.168.1.77` | Borra una dirección |
| `ip route show` | `Get-NetRoute` (solo IPv4: `Get-NetRoute -AddressFamily IPv4`) | Tabla de rutas |
| `ip route add 10.0.0.0/8 via 192.168.1.1` | `New-NetRoute -DestinationPrefix 10.0.0.0/8 -InterfaceAlias "Ethernet" -NextHop 192.168.1.1` | Añade una ruta |
| `ip route del ...` | `Remove-NetRoute -DestinationPrefix 10.0.0.0/8` | Borra una ruta |
| `ip neigh` | `Get-NetNeighbor` | Tabla de vecinos (ARP en IPv4, NDP en IPv6) |
| `iw` | `netsh wlan` | Redes inalámbricas |

**Cambios permanentes o temporales:** en Linux, los comandos `ip` son temporales y lo permanente va en los archivos de configuración. En Windows, los cmdlets escriben por defecto **en los dos sitios a la vez**: en la configuración activa (*ActiveStore*) y en la guardada (*PersistentStore*). Para un cambio **solo hasta el próximo reinicio**, como con `ip`, se añade `-PolicyStore ActiveStore`:
```
New-NetRoute -DestinationPrefix 10.0.0.0/8 -InterfaceAlias "Ethernet" -NextHop 192.168.1.1 -PolicyStore ActiveStore
```

### 4.3 Configuración avanzada (objetivo 205.2)

Funciones de red avanzadas que en Linux se configuran con `bonding`, VLAN o `brctl`, y su equivalente:

| Función en Linux | Equivalente en Windows Server 2025 | Ejemplo |
|---|---|---|
| Agregación de tarjetas (*bonding*) | **Formación de equipos NIC** (NIC Teaming / LBFO) en servidores físicos; en hosts Hyper-V se recomienda **SET** (Switch Embedded Teaming) | `New-NetLbfoTeam -Name Equipo1 -TeamMembers "Ethernet","Ethernet 2" -TeamingMode SwitchIndependent` |
| VLAN (`ip link add link eth0 name eth0.10 type vlan id 10`) | Propiedad VLAN del controlador, o una interfaz de equipo con VLAN | `Set-NetAdapter -Name "Ethernet" -VlanID 10` o `Add-NetLbfoTeamNic -Team Equipo1 -VlanID 10` |
| Puente (*bridge*) para máquinas virtuales | Conmutador virtual de Hyper-V | `New-VMSwitch -Name Externo -NetAdapterName "Ethernet"` |
| Reenvío de paquetes (router) | `Set-NetIPInterface -Forwarding Enabled` o el rol **Enrutamiento y acceso remoto** (RRAS): `Install-WindowsFeature Routing -IncludeManagementTools` | — |
| NAT (`iptables -t nat MASQUERADE`) | RRAS con NAT, o `New-NetNat` (pensado para contenedores y máquinas virtuales) | `New-NetNat -Name RedNAT -InternalIPInterfaceAddressPrefix 172.16.0.0/24` |

**Reparar la pila de red** (cuando la configuración de red está dañada y nada funciona):

| Comando | Función |
|---|---|
| `netsh int ip reset` | Restablece la configuración de TCP/IP a sus valores por defecto (requiere reiniciar) |
| `netsh winsock reset` | Restablece el catálogo de Winsock (la capa de sockets de Windows) |

---

## 5. Comprobación y solución de problemas de red

### 5.1 Comprobación básica de conectividad

| Comando Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `ping host` | `ping host` o `Test-Connection host` | Comprueba la conectividad. **En Windows `ping` envía solo 4 paquetes** y termina; `ping -t` sigue hasta pulsar `Ctrl+C` (como el `ping` de Linux) |
| — | `ping -n 10 host` / `ping -l 1400 -f host` | 10 paquetes / paquete de 1400 bytes sin fragmentar (útil para probar la MTU) |
| `ping6 host` | `ping -6 host` | Ping en IPv6. Con una dirección de enlace local: `ping fe80::1%12` (índice de la interfaz) |
| `traceroute host` | `tracert host` o `Test-NetConnection host -TraceRoute` | Saltos hasta el destino |
| `traceroute6 host` | `tracert -6 host` | Lo mismo en IPv6 |
| `mtr host` | `pathping host` | Ruta + estadísticas de pérdida en cada salto (Capítulo 2) |
| `nc IP puerto` (cliente) | `Test-NetConnection IP -Port 443` | Comprueba si un puerto TCP responde (`TcpTestSucceeded : True`) |
| `nc -l puerto` (servidor) | No hay un comando integrado | Para simular un servidor se puede usar `ncat` (incluido en Nmap para Windows) o un pequeño script de PowerShell |

Ejemplo de servidor de prueba con PowerShell (escucha en el puerto 5000 hasta que llega una conexión):
```
$l = [System.Net.Sockets.TcpListener]::new([ipaddress]::Any, 5000); $l.Start(); $c = $l.AcceptTcpClient(); "Conexión desde $($c.Client.RemoteEndPoint)"; $l.Stop()
```

Recuerda que el **firewall de Windows** bloquea por defecto casi todo el tráfico entrante, **incluido el ping** en muchos perfiles. Si un servidor no responde al ping, puede ser por el firewall y no por un fallo de red (apartado 6).

### 5.2 Resolución de nombres

| Comando Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `host nombre` | `nslookup nombre` | Consulta sencilla (también a la inversa: `nslookup 192.168.1.10`) |
| `host -t mx dominio` | `nslookup -type=mx dominio` | Registros de un tipo concreto |
| `dig nombre` | `Resolve-DnsName nombre` | Consulta detallada: tipo, TTL, sección de la respuesta... |
| `dig @servidor nombre MX` | `Resolve-DnsName nombre -Type MX -Server 192.168.0.103` | Consulta a un servidor concreto |
| — | `Resolve-DnsName nombre -DnsOnly -NoHostsFile` | Solo DNS, sin usar el archivo `hosts` ni la caché |

`nslookup` también tiene un modo interactivo (escribir `nslookup` y después las consultas), igual que en Linux.

### 5.3 Comprobación de conexiones abiertas

| Comando Linux | Equivalente en Windows Server 2025 | Función |
|---|---|---|
| `lsof -i` | `Get-NetTCPConnection \| Select LocalAddress, LocalPort, RemoteAddress, RemotePort, State, OwningProcess` | Conexiones con el proceso que las usa |
| `netstat -tulpn` | `netstat -ano` (el PID en la última columna) o `netstat -abno` (añade el nombre del programa) | Puertos en escucha y conexiones. En Windows `netstat` **no está obsoleto** |
| `ss -tln` | `Get-NetTCPConnection -State Listen` | Puertos TCP en escucha |
| `ss -uln` | `Get-NetUDPEndpoint` | Puertos UDP abiertos |
| `netstat -s` | `netstat -s` (igual) | Estadísticas por protocolo |
| `arp -a` / `ip neigh` | `arp -a` / `Get-NetNeighbor` | Tabla IP ↔ MAC |
| — | **TCPView** (Sysinternals) | Conexiones en tiempo real, en ventana gráfica |

Truco: ver qué programa usa un puerto concreto:
```
Get-Process -Id (Get-NetTCPConnection -LocalPort 443 -State Listen).OwningProcess
```

### 5.4 Escaneo de red

Windows no incluye un escáner de puertos. **Nmap tiene versión oficial para Windows** (se instala junto con el controlador de captura Npcap). Para comprobar un único puerto se usa `Test-NetConnection IP -Port N`.

La advertencia del libro se aplica igual: escanear redes sin permiso puede tener consecuencias legales o disciplinarias.

### 5.5 Captura de tráfico

| Linux | Equivalente en Windows Server 2025 |
|---|---|
| `tcpdump` | **`pktmon`** (integrado, Capítulo 2): `pktmon start --capture`, `pktmon stop`, `pktmon etl2pcap` |
| — | `netsh trace start capture=yes tracefile=C:\captura.etl` / `netsh trace stop` (captura clásica de Windows) |
| Wireshark | **Wireshark para Windows** (con Npcap) para capturar y analizar; también abre los archivos convertidos de `pktmon` |

### 5.6 Revisar los mensajes de la tarjeta de red

| Linux | Equivalente en Windows Server 2025 |
|---|---|
| `dmesg` (¿se cargó el controlador?) | `Get-NetAdapter \| Select Name, InterfaceDescription, Status, LinkSpeed, DriverVersion, DriverProvider` y el Administrador de dispositivos (Capítulo 3) |
| `/var/log/syslog`, `/var/log/messages` | Visor de eventos → registro **Sistema** (eventos del controlador de la tarjeta) |
| — | Registros específicos: `Microsoft-Windows-Dhcp-Client/Admin` (problemas de DHCP), `Microsoft-Windows-NetworkProfile/Operational` (cambios de red y perfil), `Microsoft-Windows-DNS-Client/Operational` (consultas DNS, desactivado por defecto) |

---

## 6. Control de acceso a servicios de red: el equivalente de TCP Wrappers

Windows **no tiene TCP Wrappers**. Su función (permitir o bloquear el acceso a un servicio según la IP de origen) la hace el **Firewall de Windows Defender con seguridad avanzada**, que está **activado por defecto** y bloquea todo el tráfico entrante que no tenga una regla que lo permita. El firewall se trata a fondo en el Capítulo 12; aquí se ve solo su uso como sustituto de TCP Wrappers.

| Concepto TCP Wrappers | Equivalente en Windows Server 2025 |
|---|---|
| `/etc/hosts.allow` (lista blanca) | Reglas de entrada con acción **Permitir** (`-Action Allow`) y las IP de origen permitidas (`-RemoteAddress`) |
| `/etc/hosts.deny` (lista negra) | Reglas con acción **Bloquear** (`-Action Block`) |
| Buena práctica: `hosts.deny` con `ALL` | Ya es el comportamiento por defecto: **todo lo entrante está bloqueado** salvo lo permitido |
| Orden: primero `hosts.allow`, después `hosts.deny` | **Distinto**: si una conexión coincide con una regla de bloqueo y con una de permitir, **gana el bloqueo**. No importa el orden de las reglas |
| Formato `servicio: clientes` | Regla con programa o puerto (`-Program`, `-LocalPort`) + direcciones de origen (`-RemoteAddress`) |
| Requisito: el programa enlazado con `libwrap` | No hay requisito: el firewall actúa sobre **cualquier** programa |
| Los cambios se aplican al momento | Igual |

**Ejemplo** (equivalente a permitir SSH solo desde la red local):
```
New-NetFirewallRule -DisplayName "SSH desde la LAN" -Direction Inbound -Protocol TCP -LocalPort 22 -RemoteAddress 192.168.1.0/24 -Action Allow
```

**Comandos principales:**

| Comando | Función |
|---|---|
| `Get-NetFirewallProfile \| Select Name, Enabled, DefaultInboundAction` | Estado del firewall en cada perfil (Dominio, Privado, Público) |
| `Get-NetFirewallRule -Enabled True -Direction Inbound` | Reglas de entrada activas |
| `Get-NetFirewallRule -DisplayName "SSH desde la LAN" \| Get-NetFirewallAddressFilter` | Direcciones permitidas en una regla |
| `Set-NetFirewallRule -DisplayName "..." -RemoteAddress 192.168.1.0/24,10.0.0.5` | Cambia las direcciones permitidas |
| `Disable-NetFirewallRule` / `Remove-NetFirewallRule` | Desactiva / borra una regla |
| `netsh advfirewall firewall add rule name="SSH LAN" dir=in action=allow protocol=TCP localport=22 remoteip=192.168.1.0/24` | Lo mismo con `netsh` |
| `wf.msc` | Consola gráfica del firewall |

Registro de conexiones (para ver qué se bloquea, útil al depurar): `Set-NetFirewallProfile -Profile Domain,Private,Public -LogBlocked True`. El archivo se guarda en `C:\Windows\System32\LogFiles\Firewall\pfirewall.log`.

Además, igual que en Linux algunos programas tienen su propio control de acceso, en Windows algunos servicios también lo tienen: por ejemplo, OpenSSH (`AllowUsers` / `Match Address` en `C:\ProgramData\ssh\sshd_config`) o IIS (característica "Restricciones de dominio y direcciones IP").

---

## 7. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 6) | Equivalente en Windows Server 2025 |
|---|---|
| `eth0`, `enp0s3` | Alias (`Ethernet`, `Ethernet 2`) e índice de interfaz |
| `/etc/network/interfaces`, `ifcfg-eth0` | Registro (`Tcpip\Parameters\Interfaces`), gestionado con PowerShell o `netsh` |
| `iface eth0 inet static` + `address`/`netmask`/`gateway` | `New-NetIPAddress -IPAddress -PrefixLength -DefaultGateway` |
| `iface eth0 inet dhcp` | `Set-NetIPInterface -Dhcp Enabled` |
| `dhclient` / `dhcpcd` | Servicio Cliente DHCP; `ipconfig /release` y `/renew` |
| `/etc/hostname` | `Rename-Computer` (requiere reiniciar) |
| `/etc/resolv.conf` | `Set-DnsClientServerAddress`, `Set-DnsClientGlobalSetting -SuffixSearchList` |
| `/etc/hosts` | `C:\Windows\System32\drivers\etc\hosts` |
| NetworkManager | `ncpa.cpl`, Administrador del servidor, Windows Admin Center, `sconfig` |
| `ifconfig` | `ipconfig` (consultar) / `netsh interface ipv4 set address` (cambiar) |
| `ifup` / `ifdown` | `Enable-NetAdapter` / `Disable-NetAdapter` |
| `ip link` / `ip addr` / `ip route` / `ip neigh` | `Get-NetAdapter` / `Get-NetIPAddress` / `Get-NetRoute` / `Get-NetNeighbor` |
| `route add` | `route -p add` o `New-NetRoute` |
| `iwlist` / `iwconfig` / `iw` | `netsh wlan` (con la característica Servicio WLAN) |
| Bonding / VLAN / bridge | NIC Teaming o SET / `-VlanID` / conmutador virtual de Hyper-V |
| `ping` / `ping6` | `ping` (4 paquetes; `-t` continuo) / `ping -6`, `Test-Connection` |
| `traceroute` | `tracert`, `Test-NetConnection -TraceRoute` |
| `nc` | `Test-NetConnection -Port` (cliente); `ncat` de Nmap |
| `host` / `dig` | `nslookup` / `Resolve-DnsName` |
| `lsof -i`, `ss`, `netstat` | `Get-NetTCPConnection`, `Get-NetUDPEndpoint`, `netstat -ano` |
| `arp` | `arp -a`, `Get-NetNeighbor` |
| `nmap` | Nmap para Windows (no integrado) |
| `tcpdump` | `pktmon`, `netsh trace`, Wireshark |
| `/etc/hosts.allow` + `/etc/hosts.deny` | Firewall de Windows Defender (`New-NetFirewallRule -RemoteAddress`); el bloqueo gana sobre el permiso |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 6 ("Navigating Network Services") del libro LPIC-2. Fuentes: documentación oficial de Microsoft Learn (módulos NetAdapter, NetTCPIP, DnsClient y NetSecurity de PowerShell, netsh, ipconfig, route, nslookup, Resolve-DnsName, Test-NetConnection, pktmon, NIC Teaming y Switch Embedded Teaming, perfiles de red y Firewall de Windows Defender con seguridad avanzada).*
