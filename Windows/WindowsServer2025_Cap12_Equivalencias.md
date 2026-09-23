# LPIC-2 · Capítulo 12: Setting Up System Security
### Equivalencias en Windows Server 2025

Este documento recoge, para cada concepto visto en el Capítulo 12 del libro LPIC-2 (router y NAT, cortafuegos, OpenSSH, OpenVPN, auditoría y detección de intrusiones), cuál es su equivalente en **Windows Server 2025**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo: **`iptables` no existe en Windows**. Su papel se reparte entre varias piezas:
- **Firewall de Windows Defender con seguridad avanzada**: filtra el tráfico que entra y sale del propio servidor (lo que en `iptables` son las cadenas `INPUT` y `OUTPUT`). Viene **activado por defecto** y bloquea todo lo entrante que no esté permitido. Ya se vio como sustituto de TCP Wrappers en el Capítulo 6.
- **Enrutamiento y acceso remoto (RRAS)**: rol que convierte el servidor en router, hace NAT y ofrece VPN (lo que en Linux hacen la cadena `FORWARD`, la tabla `NAT` y OpenVPN).
- Todo se apoya en la **Plataforma de filtrado de Windows** (WFP), el equivalente a Netfilter en el kernel.

Además, Windows Server 2025 incluye herramientas de seguridad propias que no aparecen en el libro: **Microsoft Defender Antivirus**, **Credential Guard**, **Windows LAPS** y las líneas base de seguridad con **OSConfig** (apartado 6.3).

Todos los comandos se ejecutan en una consola **como Administrador**.

---

## 1. Direcciones privadas y NAT

### 1.1 y 1.2 Direcciones privadas IPv4 e IPv6 link-local

**Iguales** que en el libro: son estándares de Internet. Solo recordar que en Windows las direcciones `fe80::` llevan detrás el **índice** de la interfaz (`fe80::1%12`) en lugar de su nombre (Capítulo 6).

### 1.3 NAT y router

| Tarea en Linux | Equivalente en Windows Server 2025 |
|---|---|
| Activar el reenvío (`net.ipv4.ip_forward=1`) | `Set-NetIPInterface -Forwarding Enabled` (Capítulo 3) o, de forma completa, el rol RRAS |
| Router con NAT (`iptables -t nat ... MASQUERADE`) | Rol **Enrutamiento y acceso remoto** con NAT: `Install-WindowsFeature Routing -IncludeManagementTools`, y después se configura el NAT con la consola `rrasmgmt.msc` (asistente "Configurar y habilitar Enrutamiento y acceso remoto" → NAT) |
| NAT sencillo para máquinas virtuales y contenedores | **WinNAT**: `New-NetNat -Name RedNAT -InternalIPInterfaceAddressPrefix 172.16.0.0/24` |
| Redirección de puertos (`iptables -t nat ... DNAT`) | Con WinNAT: `Add-NetNatStaticMapping -NatName RedNAT -Protocol TCP -ExternalIPAddress 0.0.0.0 -ExternalPort 8080 -InternalIPAddress 172.16.0.10 -InternalPort 80`. En RRAS: pestaña **Servicios y puertos** de la interfaz pública |
| Redirección de un puerto TCP a otro equipo (sin NAT) | `netsh interface portproxy add v4tov4 listenport=8080 connectaddress=172.16.0.10 connectport=80`. Ver: `netsh interface portproxy show all` |

La advertencia del libro sobre la redirección de puertos se aplica igual: expone un servicio interno a Internet.

---

## 2. Cortafuegos (equivalente a `iptables`)

### 2.0 El Firewall de Windows Defender

| Elemento | Detalle |
|---|---|
| Consola gráfica | `wf.msc` (Firewall de Windows Defender con seguridad avanzada) |
| Módulo de PowerShell | **NetSecurity** (`Get-NetFirewallRule`, `New-NetFirewallRule`...) |
| Herramienta clásica | `netsh advfirewall` |
| Servicio | **Firewall de Windows Defender** (`mpssvc`); no se debe detener |
| **Perfiles** | **Dominio**, **Privado** y **Público**: cada red tiene uno (Capítulo 6) y cada regla indica en qué perfiles se aplica |
| Configuración por defecto | Entrante: **bloquear** (salvo reglas que permiten); saliente: **permitir** |

### 2.1 y 2.2 Cadenas y tablas de `iptables`

| Cadena / tabla de `iptables` | Equivalente en Windows Server 2025 |
|---|---|
| `INPUT` | **Reglas de entrada** (`-Direction Inbound`) |
| `OUTPUT` | **Reglas de salida** (`-Direction Outbound`) |
| `FORWARD` | El Firewall de Windows **no filtra el tráfico que se reenvía** entre redes. Para eso: filtros de paquetes de RRAS (en cada interfaz, desde `rrasmgmt.msc`) o un cortafuegos de red dedicado |
| `PREROUTING` / `POSTROUTING` | NAT de RRAS o WinNAT (apartado 1.3) |
| Tabla `FILTER` | Reglas del firewall |
| Tabla `NAT` | RRAS / WinNAT |
| Tabla `MANGLE` (modificar paquetes) | Lo más parecido son las **directivas de QoS**, que marcan los paquetes con un valor DSCP: `New-NetQosPolicy -Name "VoIP" -IPProtocolMatchCondition UDP -IPDstPortStartMatchCondition 5060 -IPDstPortEndMatchCondition 5060 -DSCPAction 46` |
| — | **Reglas de seguridad de conexión**: exigir IPsec (autenticación o cifrado) entre equipos. No tienen equivalente en `iptables` |

### 2.3 Opciones básicas de `iptables` y sus equivalentes (Tabla 12.1)

| Opción de `iptables` | Equivalente en Windows Server 2025 |
|---|---|
| `-A` (añadir regla) | `New-NetFirewallRule` |
| `-D` (borrar regla) | `Remove-NetFirewallRule -DisplayName "..."` |
| `-F` (vaciar) | `netsh advfirewall reset` (vuelve a la configuración y las reglas **por defecto**, no deja el firewall vacío) |
| `-I` (insertar en una posición) | No existe: **las reglas no tienen orden**. Si una conexión coincide con una regla de bloqueo y otra de permitir, **gana el bloqueo** |
| `-L` (listar) | `Get-NetFirewallRule -Enabled True` o `netsh advfirewall firewall show rule name=all` |
| `-P` (política por defecto) | `Set-NetFirewallProfile -Profile Domain,Private,Public -DefaultInboundAction Block -DefaultOutboundAction Allow` |
| `-R` (sustituir) | `Set-NetFirewallRule -DisplayName "..." -RemoteAddress ...` |
| `-S` (detalle) | `Show-NetFirewallRule` o `Get-NetFirewallRule -DisplayName "..." \| Get-NetFirewallPortFilter` (y `Get-NetFirewallAddressFilter`, `Get-NetFirewallApplicationFilter`) |
| `-t tabla` | No aplica |
| — | `Enable-NetFirewallRule` / `Disable-NetFirewallRule` (activar o desactivar una regla sin borrarla) |

Windows trae **muchas reglas predefinidas**, agrupadas por función (por ejemplo, "Compartir archivos e impresoras"). Se activan en bloque: `Enable-NetFirewallRule -DisplayGroup "Escritorio remoto"` (el nombre del grupo depende del idioma del sistema; también se puede usar `-Group` con el nombre interno).

### 2.4 Políticas y acciones

| Acción de `iptables` | Equivalente en Windows Server 2025 |
|---|---|
| `ACCEPT` | `-Action Allow` |
| `DROP` | `-Action Block` (el Firewall de Windows **descarta en silencio**) |
| `REJECT` (con aviso al origen) | No existe: el bloqueo siempre es silencioso |
| `LOG` | Registro del firewall por perfil: `Set-NetFirewallProfile -Profile Domain,Private,Public -LogBlocked True -LogAllowed False`. Archivo: `C:\Windows\System32\LogFiles\Firewall\pfirewall.log` |

Registro más detallado: la auditoría de la Plataforma de filtrado de Windows, que anota cada conexión bloqueada en el registro de **Seguridad** (evento **5157**):
```
auditpol /set /subcategory:"Conexión de Plataforma de filtrado" /failure:enable
```
(En un sistema en inglés, la subcategoría se llama "Filtering Platform Connection".)

**Ejemplo del libro** (bloquear todo el tráfico saliente):
```
Set-NetFirewallProfile -Profile Domain,Private,Public -DefaultOutboundAction Block
```
Cuidado: después hay que crear reglas de salida para todo lo que el servidor necesite (DNS, Windows Update, Active Directory...).

### 2.5 Opciones de una regla (equivalente a la Tabla 12.2)

| Opción de `iptables` | Parámetro de `New-NetFirewallRule` |
|---|---|
| `-s dirección` (origen) | Regla de entrada: `-RemoteAddress`. Regla de salida: `-LocalAddress` |
| `-d dirección` (destino) | Regla de entrada: `-LocalAddress`. Regla de salida: `-RemoteAddress` |
| `-i` / `-o` (interfaz) | `-InterfaceAlias "Ethernet"` o `-InterfaceType Wired` |
| `-j` (acción) | `-Action Allow` / `Block` |
| `-p` (protocolo) | `-Protocol TCP`, `UDP`, `ICMPv4`, `ICMPv6` |
| `--dport` (puerto de destino) | Regla de entrada: `-LocalPort`. Regla de salida: `-RemotePort` |
| `--sport` (puerto de origen) | Regla de entrada: `-RemotePort`. Regla de salida: `-LocalPort` |
| `-g` (saltar a otra cadena) | No existe |
| — | `-Program "C:\ruta\programa.exe"` o `-Service nombre`: la regla se aplica solo a un programa o servicio (no existe en `iptables`) |
| — | `-Profile Domain,Private` (en qué perfiles se aplica) |

En Windows la regla se escribe desde el punto de vista del servidor: "local" es el propio servidor y "remoto" es el otro equipo, en ambas direcciones.

**Ejemplos del libro:**

| `iptables` | Windows Server 2025 |
|---|---|
| `iptables -A INPUT -s 10.0.1.25 -j REJECT` | `New-NetFirewallRule -DisplayName "Bloquear 10.0.1.25" -Direction Inbound -RemoteAddress 10.0.1.25 -Action Block` |
| `iptables -A OUTPUT -p tcp --dport 1234 -j DROP` | `New-NetFirewallRule -DisplayName "Bloquear salida 1234" -Direction Outbound -Protocol TCP -RemotePort 1234 -Action Block` |

Otro ejemplo habitual: permitir el ping (viene bloqueado por defecto):
```
Enable-NetFirewallRule -Name "FPS-ICMP4-ERQ-In"
```

### 2.6 Persistencia y configuración completa

| Linux | Windows Server 2025 |
|---|---|
| Las reglas se pierden al reiniciar | **Las reglas son permanentes**: se guardan en el Registro y se aplican solas al arrancar |
| `iptables-save > reglas.txt` | `netsh advfirewall export "C:\reglas.wfw"` |
| `iptables-restore < reglas.txt` | `netsh advfirewall import "C:\reglas.wfw"` |
| Script con reglas al arrancar | No hace falta. Para aplicar las mismas reglas a muchos servidores: **Directivas de grupo** (Configuración del equipo → Configuración de Windows → Configuración de seguridad → Firewall de Windows Defender con seguridad avanzada) |
| `ip6tables` (IPv6 aparte) | Las mismas reglas valen para **IPv4 e IPv6** |
| Ver qué se está aplicando realmente | `Get-NetFirewallRule -PolicyStore ActiveStore` (reglas activas, sumando las locales y las de directivas de grupo) y `netsh wfp show state` (estado completo de WFP, muy técnico) |

---

## 3. OpenSSH

### 3.1 Componentes

Windows Server 2025 incluye **OpenSSH** (el mismo programa del libro, adaptado a Windows):

| Elemento | Linux | Windows Server 2025 |
|---|---|---|
| Servidor | `sshd` | Servicio `sshd` (viene incluido; se activa desde el Administrador del servidor o con `Set-Service sshd -StartupType Automatic` + `Start-Service sshd`) |
| Cliente | `ssh` | `ssh.exe` (en `C:\Windows\System32\OpenSSH\`, disponible siempre) |
| Configuración del servidor | `/etc/ssh/sshd_config` | `C:\ProgramData\ssh\sshd_config` |
| Claves del servidor | `/etc/ssh/ssh_host_*` | `C:\ProgramData\ssh\ssh_host_*_key` |
| Registro | `/var/log/auth.log` o `/var/log/secure` | Visor de eventos → Registros de aplicaciones y servicios → **OpenSSH** → Operational |
| Regla del firewall | Manual | Se crea sola (`OpenSSH-Server-In-TCP`, puerto 22) |
| Reiniciar tras cambios | `systemctl restart sshd` | `Restart-Service sshd` |

**Shell por defecto:** al conectarse por SSH, Windows abre `cmd.exe`. Para que abra PowerShell:
```
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -PropertyType String -Force
```

### 3.2 Fichero de configuración del servidor (Tabla 12.3)

| Opción | Situación en Windows Server 2025 |
|---|---|
| `Protocol` | Obsoleta: solo existe la versión 2 del protocolo |
| `PasswordAuthentication` | Igual |
| `PubkeyAuthentication` | Igual |
| `AllowUsers` / `DenyUsers` | Igual. Los usuarios del dominio se escriben en minúsculas con el dominio: `AllowUsers empresa\juan` |
| `AllowGroups` / `DenyGroups` | Igual, con grupos de Windows (`AllowGroups "empresa\administradores ssh"`) |
| `PermitRootLogin` | No aplica (no hay usuario `root`). Para limitar a los administradores se usa `DenyGroups administradores` o un bloque `Match Group administradores` |
| `X11Forwarding` | No aplica en Windows |
| `AllowTcpForwarding` | Igual |

**Particularidad importante de Windows:** el `sshd_config` de Windows trae al final un bloque para los miembros del grupo **Administradores**: sus claves públicas **no** se leen de su carpeta personal, sino de un archivo común:
```
Match Group administrators
       AuthorizedKeysFile __PROGRAMDATA__/ssh/administrators_authorized_keys
```

### 3.3 y 3.4 Uso del cliente y claves

| Tarea | Linux | Windows Server 2025 |
|---|---|---|
| Conectarse | `ssh usuario@servidor` | Igual. Con un usuario del dominio: `ssh empresa\juan@srv01` o `ssh juan@empresa.local@srv01` |
| Generar claves | `ssh-keygen -t rsa` | `ssh-keygen -t ed25519` (recomendado). Se guardan en `C:\Users\<usuario>\.ssh\` |
| Clave pública en el servidor (usuario normal) | `~/.ssh/authorized_keys` | `C:\Users\<usuario>\.ssh\authorized_keys` |
| Clave pública en el servidor (administradores) | — | `C:\ProgramData\ssh\administrators_authorized_keys`, con permisos solo para Administradores y SYSTEM: `icacls C:\ProgramData\ssh\administrators_authorized_keys /inheritance:r /grant "*S-1-5-32-544:F" /grant "SYSTEM:F"` |
| Añadir la clave (`cat id_rsa.pub >> authorized_keys`) | Igual con PowerShell: `Get-Content id_ed25519.pub \| Add-Content C:\Users\juan\.ssh\authorized_keys` (no existe `ssh-copy-id`) |
| Agente de claves | `ssh-agent` + `ssh-add` | Servicio `ssh-agent` (desactivado por defecto): `Set-Service ssh-agent -StartupType Automatic`, `Start-Service ssh-agent` y después `ssh-add` |
| Copiar archivos | `scp`, `sftp` | Iguales (incluidos) |

**Alternativas propias de Windows para la administración remota** (sin equivalente en el libro):

| Herramienta | Descripción |
|---|---|
| **Comunicación remota de PowerShell** (WinRM) | Consola remota nativa de Windows: `Enter-PSSession -ComputerName SRV01` o `Invoke-Command -ComputerName SRV01 -ScriptBlock { Get-Service }`. Activada por defecto en Windows Server (puertos 5985 HTTP y 5986 HTTPS) |
| **Escritorio remoto** (RDP) | Acceso gráfico (puerto 3389); se activa con `sconfig` o en las propiedades del sistema |
| **Windows Admin Center** | Administración por navegador web |

---

## 4. OpenVPN

### 4.1 Concepto

El concepto es el mismo. La VPN integrada en Windows Server es el rol **Acceso remoto** (componente **VPN** de RRAS), que admite:

| Protocolo | Situación en Windows Server 2025 |
|---|---|
| **IKEv2** (IPsec) | Recomendado, para usuarios remotos y para unir dos sedes |
| **SSTP** | VPN sobre TLS en el puerto 443 (atraviesa casi cualquier firewall, parecido a OpenVPN en TCP 443) |
| L2TP/IPsec y PPTP | Antiguos y **desactivados por defecto** en Windows Server 2025 (PPTP es inseguro) |
| OpenVPN, WireGuard | No integrados, pero existen versiones oficiales para Windows (apartado 4.6) |

### 4.2 Instalación

| Linux | Windows Server 2025 |
|---|---|
| `apt-get install openvpn` / `yum install openvpn` | `Install-WindowsFeature DirectAccess-VPN -IncludeManagementTools` |
| — | `Install-RemoteAccess -VpnType Vpn` (VPN para usuarios) o `Install-RemoteAccess -VpnType VpnS2S` (VPN entre sedes) |
| Herramienta de administración | `rrasmgmt.msc` (Enrutamiento y acceso remoto) y el módulo **RemoteAccess** de PowerShell |

### 4.3 Ficheros de configuración

**No hay `server.conf` ni `client.conf`**: la configuración se guarda en el Registro y se gestiona con la consola o con cmdlets.

| Opción de OpenVPN (Tabla 12.4) | Equivalente en la VPN de Windows Server 2025 |
|---|---|
| `config` | No aplica |
| `dev tun` | Windows crea una interfaz virtual para cada conexión VPN (se ve con `Get-NetAdapter` e `ipconfig`) |
| `ifconfig` (IP de los extremos) | Grupo de direcciones que el servidor reparte a los clientes (propiedades del servidor en `rrasmgmt.msc` → IPv4 → grupo de direcciones estático, o por DHCP). Entre sedes: subredes de cada interfaz (`-IPv4Subnet`) |
| `secret` (clave estática) | **Clave precompartida** (PSK) de IKEv2 |
| `nobind` | No aplica |

**VPN entre dos sedes** (equivalente al túnel punto a punto del libro), con IKEv2 y clave precompartida:
```
Add-VpnS2SInterface -Name "Sede2" -Destination 203.0.113.10 -Protocol IKEv2 -AuthenticationMethod PSKOnly -SharedSecret "clave-secreta-larga" -IPv4Subnet "10.2.0.0/24:100"
Connect-VpnS2SInterface -Name "Sede2"
Get-VpnS2SInterface
```
(`10.2.0.0/24:100` = red de la otra sede y su métrica.)

**Cliente VPN** (equivalente a `client.conf`), en un equipo Windows:
```
Add-VpnConnection -Name "Empresa" -ServerAddress vpn.empresa.com -TunnelType Ikev2 -AuthenticationMethod MachineCertificate
rasdial "Empresa"
```
Ver las conexiones configuradas: `Get-VpnConnection`. Desconectar: `rasdial "Empresa" /disconnect`.

### 4.4 Métodos de cifrado

| Método del libro | Equivalente en Windows Server 2025 |
|---|---|
| Clave estática (`openvpn --genkey --secret`) | **Clave precompartida** (PSK) de IKEv2, adecuada para VPN entre sedes. No se recomienda para usuarios |
| Clave pública con CA (scripts `build-ca`, `build-key-server`...) | **Certificados** emitidos por una CA de **Servicios de certificados de Active Directory** (Capítulo 9), que puede repartirlos automáticamente a los equipos del dominio (inscripción automática) |
| Usuario y contraseña | Autenticación de usuarios de Active Directory, normalmente a través del rol **Servidor de directivas de redes** (NPS, un servidor RADIUS): `Install-WindowsFeature NPAS -IncludeManagementTools` |

### 4.5 Arranque manual y comprobación

| Linux | Windows Server 2025 |
|---|---|
| `openvpn server.conf` | El servicio **Enrutamiento y acceso remoto** (`RemoteAccess`) arranca solo; `Restart-Service RemoteAccess` |
| `openvpn client.conf` | `rasdial "Empresa"` |
| `ifconfig` (ver `tun0`) | `Get-NetAdapter` / `ipconfig` (aparece la interfaz de la conexión VPN) |
| Ver clientes conectados | `Get-RemoteAccessConnectionStatistics` |
| Estado del servidor | `Get-RemoteAccess` |

### 4.6 OpenVPN y otras VPN en Windows

| Programa | Descripción |
|---|---|
| **OpenVPN** para Windows | Versión oficial para Windows del mismo programa del libro. Los archivos de configuración (`.ovpn`, mismas directivas) van en `C:\Program Files\OpenVPN\config\` (cliente) o `config-auto\` (conexiones que arrancan como servicio). Usa su propio controlador de red virtual |
| **WireGuard** para Windows | VPN moderna y sencilla, con versión oficial para Windows |
| **Azure VPN Gateway** | VPN gestionada en la nube de Microsoft, para unir la red local con Azure |

---

## 5. Escaneo de puertos y herramientas de auditoría

| Herramienta del libro | Equivalente en Windows Server 2025 |
|---|---|
| `telnet host puerto` | `Test-NetConnection host -Port 25` (integrado). El cliente Telnet existe, pero hay que instalarlo: `Install-WindowsFeature Telnet-Client` |
| `nc` (netcat) | No integrado: `Test-NetConnection` para el lado cliente; `ncat` (incluido con Nmap para Windows) para cliente y servidor (Capítulo 6) |
| `nmap` | **Nmap para Windows** (versión oficial, con el controlador de captura Npcap) |
| OpenVAS | No tiene versión para Windows (se usa en una máquina Linux o como dispositivo virtual). En el ámbito de Microsoft: **Microsoft Defender Vulnerability Management** (parte de Microsoft Defender para punto de conexión) |

**Auditoría de la configuración de seguridad** (propio de Windows, sin equivalente en el libro):

| Herramienta | Función |
|---|---|
| **OSConfig** (módulo `Microsoft.OSConfig`) | Aplica y **comprueba** la línea base de seguridad de Microsoft para Windows Server 2025, y corrige automáticamente los ajustes que se desvían. Ver apartado 6.3 |
| **Microsoft Security Compliance Toolkit** | Líneas base de seguridad descargables y herramientas para comparar configuraciones (Policy Analyzer) y aplicarlas (LGPO) |
| **Analizador de procedimientos recomendados** | Revisa la configuración de cada rol: `Invoke-BpaModel` / `Get-BpaResult` (Capítulo 8) |
| `auditpol /get /category:*` | Muestra qué eventos de seguridad se están auditando |

---

## 6. Sistemas de detección de intrusiones (IDS)

### 6.1 fail2ban y alternativas (IDS de host)

Windows **no incluye un equivalente de fail2ban** que bloquee automáticamente las IP de los atacantes. Lo que sí incluye son medidas que atacan el mismo problema (los intentos repetidos de contraseña):

| Medida | Descripción |
|---|---|
| **Directiva de bloqueo de cuentas** | Bloquea la **cuenta** (no la IP) tras varios intentos fallidos (Capítulo 11): `net accounts /lockoutthreshold:5 /lockoutduration:15` |
| **Limitador de autenticación de SMB** | Retrasa cada intento fallido de contraseña por SMB (Capítulo 10) |
| Registro de intentos fallidos | Registro de **Seguridad**: evento **4625** (inicio de sesión fallido), **4624** (correcto) y **4740** (cuenta bloqueada) |

Ver los últimos intentos fallidos (equivalente a revisar `/var/log/auth.log`):
```
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 20
```

| fail2ban | Equivalente en Windows Server 2025 |
|---|---|
| Programa fail2ban | No existe. Alternativa libre para Windows: **IPBan**, que lee los eventos 4625 y bloquea las IP con reglas del firewall de Windows |
| `/etc/fail2ban/jail.conf` | Configuración de IPBan (archivo XML propio) |
| Solución hecha a mano | **Tarea programada** que se lanza con el evento 4625 y ejecuta un script que cuenta los fallos por IP y crea una regla `New-NetFirewallRule -Action Block -RemoteAddress IP` |

**Herramientas de detección en el propio servidor** (HIDS, sin equivalente directo en el libro):

| Herramienta | Descripción |
|---|---|
| **Microsoft Defender Antivirus** | Integrado y activo por defecto. Estado: `Get-MpComputerStatus`; actualizar firmas: `Update-MpSignature`; análisis: `Start-MpScan -ScanType QuickScan`; amenazas detectadas: `Get-MpThreatDetection` |
| **Sysmon** (Sysinternals) | Servicio gratuito de Microsoft que registra con mucho detalle la actividad del sistema (procesos creados, conexiones de red, cambios en archivos...) en el Visor de eventos. Muy usado como base para detectar intrusiones: `sysmon64 -accepteula -i configuracion.xml` |
| **Microsoft Defender para punto de conexión** | Detección y respuesta avanzada (EDR), de pago, gestionada desde la nube |

### 6.2 Snort y alternativas (NIDS)

Windows no incluye un sistema de detección de intrusiones de red.

| Opción | Descripción |
|---|---|
| **Snort 3** / **Suricata** | Los dos tienen versión para Windows (usan el controlador de captura Npcap). La configuración (`HOME_NET`, `EXTERNAL_NET`, reglas `->` / `<>`) es la misma del libro |
| **Microsoft Defender for Identity** | Servicio de Microsoft (de pago) que analiza el tráfico de los **controladores de dominio** para detectar ataques contra Active Directory |
| Puerto espejo (SPAN) | En el switch físico. Para máquinas virtuales de Hyper-V, el conmutador virtual tiene **creación de reflejo del puerto**: `Set-VMNetworkAdapter -VMName VM1 -PortMirroring Source` (la máquina observada) y `Set-VMNetworkAdapter -VMName IDS -PortMirroring Destination` (la máquina con el IDS) |

### 6.3 Protección del propio servidor (propio de Windows Server 2025)

Funciones de seguridad integradas que no tienen equivalente en el libro, pero que forman parte de cualquier servidor Windows bien configurado:

| Función | Descripción | Comandos |
|---|---|---|
| **Líneas base con OSConfig** | Aplica más de 300 ajustes de seguridad recomendados por Microsoft según el papel del servidor, y vigila que no cambien (control de desviaciones) | `Install-Module -Name Microsoft.OSConfig -Scope AllUsers -Repository PSGallery -Force` y después `Set-OSConfigDesiredConfiguration -Scenario SecurityBaseline/WindowsServer/2025/MemberServer -Default` (o `.../WorkgroupMember`, `.../DomainController`). Comprobar el cumplimiento: `Get-OSConfigDesiredConfiguration -Scenario SecurityBaseline/WindowsServer/2025/MemberServer` |
| **Credential Guard** | Protege las credenciales guardadas en memoria usando la virtualización (evita el robo de contraseñas con herramientas como Mimikatz) | Estado: `msinfo32` → Resumen del sistema → "Servicios de seguridad basada en virtualización en ejecución" |
| **Windows LAPS** | Cambia automáticamente la contraseña del administrador local de cada servidor y la guarda en Active Directory (evita que todos los servidores tengan la misma) | `Get-LapsADPassword -Identity SRV01 -AsPlainText` (ver la contraseña actual) |
| **Control de aplicaciones** (App Control for Business) | Solo permite ejecutar programas autorizados | Directivas de App Control (antes WDAC) |
| **BitLocker** | Cifrado de volúmenes (Capítulo 4) | `manage-bde -status` |

Nota: en versiones anteriores del módulo OSConfig, los nombres de los escenarios eran del tipo `SecurityBaseline/WS2025/MemberServer`; la documentación actual de Microsoft usa `SecurityBaseline/WindowsServer/2025/MemberServer`.

---

## 7. Recursos de seguridad

Los recursos del libro, actualizados (afecta también a Linux):

| Recurso del libro | Situación actual |
|---|---|
| **US-CERT** | Integrado en la agencia **CISA** de EE.UU. La base de datos **NVD** la mantiene el NIST |
| **SANS Institute** | Sigue activo (incluye el SANS Internet Storm Center) |
| **Bugtraq** | **Cerrada** en 2021. Alternativas de divulgación completa: las listas `oss-security` y `Full Disclosure` |

**Recursos propios de Microsoft:**

| Recurso | Descripción |
|---|---|
| **Guía de actualizaciones de seguridad** del MSRC (Microsoft Security Response Center) | Lista oficial de vulnerabilidades de los productos de Microsoft (con su CVE), gravedad y actualización que las corrige |
| **Patch Tuesday** | Publicación mensual de las actualizaciones de seguridad (segundo martes de cada mes, Capítulo 3) |
| **Blog de Microsoft Security Baselines** | Anuncio de las nuevas líneas base de seguridad recomendadas para cada versión de Windows |
| **Windows release health** | Estado de las actualizaciones de Windows y problemas conocidos |

---

## 8. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 12) | Equivalente en Windows Server 2025 |
|---|---|
| `net.ipv4.ip_forward=1` | `Set-NetIPInterface -Forwarding Enabled` o rol RRAS |
| `iptables -t nat ... MASQUERADE` | NAT de RRAS o `New-NetNat` |
| Redirección de puertos (DNAT) | `Add-NetNatStaticMapping` o `netsh interface portproxy` |
| `iptables` | Firewall de Windows Defender (`wf.msc`, módulo NetSecurity, `netsh advfirewall`) |
| Netfilter | Plataforma de filtrado de Windows (WFP) |
| `ip6tables` | Las mismas reglas (IPv4 e IPv6 juntas) |
| Cadenas `INPUT` / `OUTPUT` | Reglas de entrada / salida |
| Cadena `FORWARD` | Filtros de paquetes de RRAS (el firewall de Windows no filtra el tráfico reenviado) |
| Tabla `MANGLE` | Directivas de QoS (`New-NetQosPolicy`) |
| `ACCEPT` / `DROP` / `REJECT` / `LOG` | `Allow` / `Block` / (no existe) / registro del perfil (`pfirewall.log`) y evento 5157 |
| Orden de las reglas | Sin orden: el bloqueo gana |
| `iptables -A` / `-D` / `-L` / `-P` | `New-` / `Remove-` / `Get-NetFirewallRule` / `Set-NetFirewallProfile` |
| `iptables -F` | `netsh advfirewall reset` (vuelve a los valores por defecto) |
| `iptables-save` / `iptables-restore` | `netsh advfirewall export` / `import` (las reglas ya son permanentes) |
| `sshd`, `/etc/ssh/sshd_config` | Servicio `sshd`, `C:\ProgramData\ssh\sshd_config` |
| `~/.ssh/authorized_keys` | `C:\Users\<usuario>\.ssh\authorized_keys` y, para administradores, `administrators_authorized_keys` |
| `PermitRootLogin` | `DenyGroups` / `Match Group administradores` |
| `ssh-keygen -t rsa` | `ssh-keygen -t ed25519` |
| OpenVPN (servidor) | Rol Acceso remoto: VPN con IKEv2 o SSTP (o OpenVPN para Windows) |
| `server.conf` / `client.conf` | `rrasmgmt.msc`, `Add-VpnS2SInterface` / `Add-VpnConnection` |
| `secret` (clave estática) | Clave precompartida (PSK) |
| `build-ca`, `build-key`... | Certificados de AD CS |
| `telnet host puerto` / `nc` | `Test-NetConnection -Port` / `ncat` |
| `nmap` | Nmap para Windows |
| OpenVAS | Defender Vulnerability Management, OSConfig, Security Compliance Toolkit |
| fail2ban | Bloqueo de cuentas, limitador de SMB, IPBan o tarea programada con el evento 4625 |
| `/var/log/auth.log` | Registro de Seguridad (eventos 4624, 4625, 4740) |
| Snort | Snort 3 o Suricata para Windows; Defender for Identity |
| Puerto espejo | SPAN del switch; reflejo de puerto de Hyper-V (`-PortMirroring`) |
| Bugtraq / US-CERT | `oss-security` / CISA y NVD + MSRC |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 12 ("Setting Up System Security") del libro LPIC-2. Fuentes: documentación oficial de Microsoft Learn (Firewall de Windows Defender y módulo NetSecurity, Plataforma de filtrado de Windows, Enrutamiento y acceso remoto y VPN, WinNAT, OpenSSH para Windows, comunicación remota de PowerShell, Microsoft Defender Antivirus, Sysmon, Credential Guard, Windows LAPS, OSConfig y líneas base de seguridad de Windows Server 2025), además de la documentación de OpenVPN, Nmap, Snort, Suricata e IPBan para Windows.*

---

## Nota final

Con este capítulo se completa la adaptación a Windows Server 2025 de los 12 capítulos del libro LPIC-2. Junto con las recopilaciones de Linux (Debian 13 / Rocky Linux 10) y las equivalencias de FreeBSD 15, tienes ahora un conjunto de documentos con la misma estructura, que permiten comparar capítulo a capítulo cómo se hace cada tarea en los tres sistemas.
