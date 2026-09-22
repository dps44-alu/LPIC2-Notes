# LPIC-2 · Capítulo 6: Navigating Network Services
### Equivalencias en Windows Server 2025

---

## 1. Conceptos básicos de red

Los cinco datos necesarios para funcionar en red (IP, máscara, gateway, hostname, DNS) son exactamente los mismos conceptos en Windows; solo cambia dónde y cómo se configuran.

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| IPv6 (formato, link local `fe80::`, direcciones globales) | Idéntico: Windows implementa el mismo estándar IPv6, con autoconfiguración de direcciones link local igual que en Linux |
| DHCP (cliente) | Windows trae su propio cliente DHCP integrado en la pila TCP/IP, sin necesidad de un programa externo como `dhclient`/`dhcpcd` |

---

## 2. Configuración de red: no hay ficheros de texto equivalentes

Diferencia clave con Linux: Windows **no guarda la configuración de red en ficheros de texto** editables (como `/etc/network/interfaces` o `ifcfg-eth0`). Toda la configuración de una interfaz (IP, máscara, gateway, DNS) se almacena internamente en el **Registro de Windows**, y se modifica mediante herramientas gráficas, `netsh` o cmdlets de PowerShell — nunca editando un fichero directamente.

| Elemento Linux | Equivalente conceptual en Windows |
|---|---|
| `/etc/network/interfaces` (Debian) | Propiedades de la interfaz de red, accesibles desde **Panel de control → Centro de redes → Cambiar configuración del adaptador**, o vía PowerShell/`netsh` |
| `/etc/sysconfig/network-scripts/ifcfg-eth0` (Red Hat) | Igual: sin fichero de texto, configuración vía `netsh`/PowerShell almacenada en el Registro |
| `/etc/dhcp/dhclient.conf` | No hay fichero de opciones de cliente DHCP editable de forma directa; el comportamiento del cliente DHCP se ajusta con `Set-NetIPInterface -Dhcp Enabled/Disabled` |

### 2.1 Configuración de IP estática

| Herramienta | Comando/ejemplo |
|---|---|
| `netsh` (clásico) | `netsh interface ipv4 set address name="Ethernet" static 192.168.1.77 255.255.255.0 192.168.1.1` |
| PowerShell (moderno, recomendado) | `New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.1.77 -PrefixLength 24 -DefaultGateway 192.168.1.1` |
| Consultar la IP asignada | `Get-NetIPAddress` (equivalente a `ip addr show`) |
| Modificar una IP ya asignada | `Set-NetIPAddress` |

### 2.2 Configuración de DHCP en el cliente

```powershell
Set-NetIPInterface -InterfaceAlias "Ethernet" -Dhcp Enabled
```

### 2.3 Servidores DNS

| Herramienta | Comando/ejemplo |
|---|---|
| `netsh` | `netsh interface ipv4 set dnsservers name="Ethernet" source=static address=192.168.1.10` |
| PowerShell (equivalente a editar `/etc/resolv.conf`) | `Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("192.168.1.10","192.168.1.11")` |
| Volver a obtener el DNS por DHCP | `Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ResetServerAddresses` |
| Consultar el DNS configurado | `Get-DnsClientServerAddress` |

### 2.4 Nombre de host

| Comando Linux | Equivalente en Windows |
|---|---|
| `hostname`, `/etc/hostname` | `hostname` (solo consulta) o **`Rename-Computer -NewName nuevo_nombre -Restart`** (PowerShell) para cambiarlo, ya que en Windows requiere reinicio |

### 2.5 Resolución de nombres local: `hosts`

Esta es una de las pocas piezas de este capítulo que **sí existe como fichero de texto editable**, y con el mismo formato que en Linux:

| Elemento | Windows Server 2025 |
|---|---|
| Fichero | `C:\Windows\System32\drivers\etc\hosts` |
| Formato | Idéntico a `/etc/hosts`: `IP nombre_host [alias]` por línea |
| Orden de resolución | También se consulta antes que el DNS, igual que en Linux |

---

## 3. Configuración gráfica: Panel de control / Configuración

Windows ofrece un equivalente directo a Network Manager: el panel **Configuración → Red e Internet** (o el clásico **Centro de redes y recursos compartidos**), donde cada adaptador se configura con un asistente gráfico similar en espíritu, aunque sin el concepto de "perfiles de conexión guardados" tan explícito como en Network Manager.

---

## 4. Herramientas de línea de comandos / PowerShell

| Comando Linux (clásico) | Comando Linux (moderno) | Equivalente en Windows |
|---|---|---|
| `ifconfig` | `ip addr show` | `ipconfig` (clásico) o `Get-NetIPAddress`/`Get-NetAdapter` (PowerShell) |
| `route` | `ip route show` | `route print` (clásico) o `Get-NetRoute` (PowerShell) |
| `route add default gw IP` | `ip route add default via IP` | `route add 0.0.0.0 mask 0.0.0.0 IP` o `New-NetRoute -DestinationPrefix "0.0.0.0/0" -NextHop IP` |
| `iwconfig`/`iwlist` (Wi-Fi) | `iw` | `netsh wlan show networks`, `netsh wlan connect`, o `Get-NetAdapter`/`Get-NetConnectionProfile` |
| `ifup`/`ifdown` | — | `Enable-NetAdapter`/`Disable-NetAdapter` (PowerShell), o `netsh interface set interface "Ethernet" admin=enabled/disabled` |

---

## 5. Comprobación y solución de problemas de red

| Comando Linux | Equivalente en Windows Server 2025 |
|---|---|
| `ping` | `ping` (sintaxis muy similar; en Windows, `-t` para ping continuo en vez de ser el comportamiento por defecto) |
| `traceroute` | **`tracert`** (nombre distinto, mismo propósito) |
| `nc` (netcat) | No viene incluido de serie; alternativa moderna: `Test-NetConnection -ComputerName host -Port puerto`, que combina ping + comprobación de puerto TCP en un solo cmdlet |
| `host`/`dig` | **`nslookup`** (clásico, interactivo o no) o **`Resolve-DnsName`** (PowerShell, más moderno y con salida estructurada) |
| `lsof` (conexiones de red por proceso) | `Get-NetTCPConnection -OwningProcess PID`, o `netstat -ano` (la columna final muestra el PID) |
| `netstat` | Sigue existiendo tal cual (`netstat -an`, `netstat -ano`), y también su equivalente moderno `Get-NetTCPConnection`/`Get-NetUDPEndpoint` |
| `ss` | No hay equivalente exacto; `Get-NetTCPConnection` cubre un uso similar en PowerShell |
| `arp` | `arp -a` (existe igual en Windows) o `Get-NetNeighbor` (PowerShell) |
| `nmap` (escaneo de puertos) | No incluido de serie; **`Test-NetConnection -Port`** sirve para comprobar puertos concretos uno a uno; para escaneo de rangos completos se recurre a **Nmap para Windows** (mismo proyecto, portado) u otras herramientas de terceros |
| `tcpdump` (captura de tráfico) | **`pktmon`** (Packet Monitor, herramienta nativa desde Windows Server 2019/2022), o **`netsh trace start`/`netsh trace stop`**; para análisis gráfico, **Wireshark**/**Microsoft Network Monitor** |

**`Test-NetConnection` — el cmdlet más versátil de esta sección (combina `ping` + comprobación de puerto):**
```powershell
Test-NetConnection -ComputerName servidor -Port 443 -InformationLevel Detailed
```
Devuelve, entre otros datos: resultado de la resolución DNS, si el ping tuvo éxito, y si el puerto TCP indicado respondió (`TcpTestSucceeded`).

**Captura de tráfico con `pktmon` (equivalente nativo más cercano a `tcpdump`):**
```
pktmon start --etw -p 0 -l real-time
pktmon stop
```

---

## 6. Control de acceso a servicios de red: Windows Firewall (equivalente a TCP Wrappers)

| Concepto Linux (TCP Wrappers) | Equivalente en Windows Server 2025 |
|---|---|
| `/etc/hosts.allow` / `/etc/hosts.deny` | **Firewall de Windows Defender** (Windows Defender Firewall), con reglas de entrada/salida gestionadas gráficamente (`wf.msc`), con `netsh advfirewall`, o con los cmdlets PowerShell `*-NetFirewallRule` |
| Regla por servicio (`servicio: IPs`) | Regla de firewall por puerto/programa/perfil de red, mucho más granular que TCP Wrappers |
| Aplicación inmediata sin reiniciar el servicio | Igual: las reglas de Windows Firewall se aplican al momento |

**Comandos PowerShell principales:**

| Comando | Función |
|---|---|
| `Get-NetFirewallRule` | Lista las reglas de firewall existentes |
| `New-NetFirewallRule -DisplayName "nombre" -Direction Inbound -Protocol TCP -LocalPort 443 -Action Allow` | Crea una regla de entrada permitiendo un puerto (equivalente a una línea en `hosts.allow`) |
| `New-NetFirewallRule -DisplayName "nombre" -Direction Inbound -Action Block -RemoteAddress 10.0.0.5` | Bloquea el tráfico de una IP concreta (equivalente a una línea en `hosts.deny`) |
| `Set-NetFirewallRule` | Modifica una regla existente |
| `Remove-NetFirewallRule` | Elimina una regla |
| `Get-NetFirewallProfile` | Consulta el estado de los tres perfiles de red (Dominio, Privado, Público) |
| `netsh advfirewall show currentprofile` | Equivalente clásico (no PowerShell) para ver el perfil de firewall activo |

A diferencia de TCP Wrappers (que solo protege aplicaciones enlazadas con `libwrap`), el Firewall de Windows filtra a nivel de **todo el sistema operativo**, sin depender de que cada aplicación lo soporte explícitamente — es conceptualmente más parecido a `iptables`/`nftables` (ver Capítulo 12) que al TCP Wrappers del libro.

---

## 7. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 6) | Equivalente en Windows Server 2025 |
|---|---|
| `/etc/network/interfaces`, `ifcfg-eth0` | Sin fichero equivalente; configuración vía `netsh`/PowerShell, guardada en el Registro |
| `/etc/resolv.conf` | `Set-DnsClientServerAddress` / `Get-DnsClientServerAddress` |
| `/etc/hosts` | `C:\Windows\System32\drivers\etc\hosts` (mismo formato) |
| `ifconfig`, `ip addr` | `ipconfig`, `Get-NetIPAddress` |
| `route` | `route print`, `Get-NetRoute` |
| `ping` | `ping` (igual) |
| `traceroute` | `tracert` |
| `host`, `dig` | `nslookup`, `Resolve-DnsName` |
| `netstat`, `ss` | `netstat`, `Get-NetTCPConnection` |
| `nmap` | `Test-NetConnection` (puerto a puerto); Nmap portado para escaneo completo |
| `tcpdump` | `pktmon`, `netsh trace`, Wireshark |
| TCP Wrappers (`hosts.allow`/`hosts.deny`) | Firewall de Windows Defender (`*-NetFirewallRule`) |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 6 ("Navigating Network Services") del libro LPIC-2. Fuentes: conocimiento general de administración de Windows Server, verificado con documentación oficial de Microsoft Learn (`Set-DnsClientServerAddress`, `Test-NetConnection`, `New-NetFirewallRule` para Windows Server 2025, y `pktmon`).*
