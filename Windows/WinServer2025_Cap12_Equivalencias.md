# LPIC-2 · Capítulo 12: Setting Up System Security
### Equivalencias en Windows Server 2025

Con este documento se completa la contrapartida en Windows Server 2025 de los 12 capítulos del libro LPIC-2.

---

## 1. Direcciones privadas y NAT

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| Rangos de IP privadas (10.x, 172.16-31.x, 192.168.x) | Idénticos: son estándares IANA, no dependen del sistema operativo |
| IPv6 link local (`fe80::`) | Igual, mismo estándar |
| NAT (traducción de direcciones) | **`New-NetNat`** (PowerShell, NAT ligero integrado en el sistema desde Windows Server 2016) o el rol completo **RRAS** (Routing and Remote Access) configurado como NAT, más apropiado para escenarios de router con varias interfaces |

**Ejemplo con `New-NetNat` (equivalente ligero a una regla MASQUERADE de `iptables`):**
```powershell
New-NetNat -Name "NAT-Interno" -InternalIPInterfaceAddressPrefix "192.168.1.0/24"
```

---

## 2. Firewall: iptables → Firewall de Windows Defender

Ya se introdujo en el Capítulo 6 el uso básico de `*-NetFirewallRule`; aquí se profundiza en la correspondencia con el modelo de `iptables` (cadenas, tablas, políticas).

| Concepto Linux (`iptables`) | Equivalente en Windows Server 2025 |
|---|---|
| Cadenas `INPUT`/`OUTPUT`/`FORWARD` | Dirección de la regla: `-Direction Inbound`/`Outbound` (Windows no distingue una cadena `FORWARD` separada: el reenvío entre interfaces se gestiona con NAT/enrutamiento, no con reglas de firewall por cadena) |
| Tablas `filter`/`nat`/`mangle` | No hay tablas separadas: todas las reglas conviven en el mismo motor (WFP, Windows Filtering Platform), diferenciadas por tipo de acción y ámbito |
| **Perfiles de red** (sin equivalente exacto en `iptables`, propio de Windows) | Windows aplica reglas distintas según el **perfil de red activo**: Dominio, Privado, Público (`Get-NetFirewallProfile`) — una capa adicional que `iptables` no tiene de forma nativa |
| `-P cadena ACCEPT/DROP` (política por defecto) | `Set-NetFirewallProfile -DefaultInboundAction Block/Allow -DefaultOutboundAction Block/Allow` |
| `-A cadena -s IP -j DROP` | `New-NetFirewallRule -Direction Inbound -RemoteAddress IP -Action Block` |
| `-A cadena -p tcp --dport puerto -j ACCEPT` | `New-NetFirewallRule -Direction Inbound -Protocol TCP -LocalPort puerto -Action Allow` |
| `iptables -L` | `Get-NetFirewallRule` |
| `iptables-save`/`iptables-restore` (persistencia de reglas) | No hace falta: las reglas de Windows Firewall **ya son persistentes** entre reinicios por defecto, sin necesidad de un paso de guardado/restauración aparte |
| `ip6tables` (reglas IPv6 aparte) | No aplica: en Windows, una misma regla de `*-NetFirewallRule` puede cubrir IPv4 e IPv6 según el `-RemoteAddress` especificado |

---

## 3. OpenSSH

Novedad importante en Windows Server 2025: **OpenSSH Server viene instalado por defecto** (en versiones anteriores había que instalarlo como característica opcional), lo que acerca mucho la administración remota de Windows al modelo habitual de Linux.

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| Paquete `openssh-server` | **Ya instalado por defecto**; solo hay que habilitar el servicio: `Start-Service sshd` + `Set-Service -Name sshd -StartupType Automatic` |
| `sshd`/`ssh` | Los mismos binarios: `sshd.exe`/`ssh.exe`, implementación real de OpenSSH (Win32-OpenSSH), no una reimplementación distinta |
| `/etc/ssh/sshd_config` | **`%ProgramData%\ssh\sshd_config`** — mismo formato de fichero, mismas directivas (`PasswordAuthentication`, `PubkeyAuthentication`, `AllowUsers`, `PermitRootLogin`...) |
| `/etc/ssh/ssh_config` (cliente) | `%ProgramData%\ssh\ssh_config` |
| Grupo para restringir el acceso SSH (equivalente a `AllowGroups` combinado con un grupo del sistema) | Grupo local **OpenSSH Users**, creado automáticamente; se gestiona con `Add-LocalGroupMember -Group "OpenSSH Users" -Member "usuario"` |
| `~/.ssh/authorized_keys` | Misma ruta y formato: `C:\Users\usuario\.ssh\authorized_keys` (para administradores, hay una ubicación especial compartida: `%ProgramData%\ssh\administrators_authorized_keys`) |
| `ssh-keygen` | El mismo comando, disponible de forma nativa |
| Regla de firewall para el puerto 22 | Se crea automáticamente al habilitar el rol (`OpenSSH-Server-In-TCP`); comprobar con `Get-NetFirewallRule -Name OpenSSH-Server-In-TCP` |

**Habilitar el servicio (equivalente a `systemctl enable --now sshd`):**
```powershell
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

Este es uno de los pocos capítulos donde Windows no solo tiene un "equivalente" sino que ejecuta **literalmente el mismo software** que en Linux (Win32-OpenSSH es un puerto oficial de OpenSSH), por lo que la mayor parte del contenido del libro sobre `sshd_config` aplica prácticamente sin cambios.

---

## 4. VPN: OpenVPN → RRAS (Routing and Remote Access)

| Concepto Linux (OpenVPN) | Equivalente en Windows Server 2025 |
|---|---|
| Paquete `openvpn` | Rol **Remote Access** con el servicio **RRAS**: `Install-WindowsFeature RemoteAccess, DirectAccess-VPN, Routing -IncludeManagementTools` |
| `server.conf`/`client.conf` | Configuración vía el asistente de RRAS o los cmdlets del módulo `RemoteAccess`, sin fichero de texto único |
| Protocolos soportados | RRAS acepta **SSTP** e **IKEv2** por defecto en Windows Server 2025; a partir de esta versión, **PPTP y L2TP ya no se aceptan por defecto** en instalaciones nuevas (se pueden reactivar si hace falta compatibilidad con clientes antiguos) — cambio de seguridad relevante respecto a versiones anteriores |
| Cifrado con clave estática o de clave pública (certificados) | RRAS usa certificados (para SSTP/IKEv2) o autenticación integrada con Active Directory/RADIUS, en vez de un fichero de clave compartida como en el modo estático de OpenVPN |
| Arrancar el servidor | `Install-RemoteAccess -VpnType VPN` (instala y configura el servicio VPN de acceso remoto) |
| Interfaz de túnel resultante (`tun0`) | Adaptador de red virtual **WAN Miniport** que aparece en el sistema tras aceptar conexiones |
| **OpenVPN en sí** (alternativa sin usar RRAS) | El propio proyecto OpenVPN también tiene versión nativa para Windows, instalable y configurable con los mismos `server.conf`/`client.conf` del libro, sin cambios — válido si se prefiere mantener exactamente la misma solución que en Linux |

---

## 5. Escaneo de puertos y herramientas de auditoría

Ver también el Capítulo 6 para el detalle de `Test-NetConnection`, `pktmon`, etc.

| Herramienta Linux | Equivalente en Windows Server 2025 |
|---|---|
| `telnet host puerto` | `Test-NetConnection -ComputerName host -Port puerto`, o el cliente Telnet clásico si se instala (`Install-WindowsFeature Telnet-Client`) |
| `nc` (netcat) | No incluido de serie; `Test-NetConnection` cubre el caso de comprobar un puerto |
| `nmap` | Nmap para Windows (mismo proyecto, portado oficialmente) |
| OpenVAS | No hay equivalente nativo de Microsoft; en entornos Windows se usan soluciones comerciales de escaneo de vulnerabilidades (Nessus, Qualys, Microsoft Defender Vulnerability Management) |

---

## 6. Sistemas de detección de intrusiones (IDS)

Windows no tiene una herramienta equivalente exacta a fail2ban o Snort instalada por defecto, pero cubre funciones similares con otras piezas del sistema:

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| fail2ban (bloqueo tras intentos fallidos de login, IDS de host) | **Directiva de bloqueo de cuenta** (Account Lockout Policy, en `secpol.msc` o GPO): bloquea la *cuenta* (no la IP) tras un número de intentos fallidos — mecanismo distinto pero con el mismo objetivo de frenar ataques de fuerza bruta. Para bloquear también por IP de origen, hace falta una solución adicional (ej. reglas dinámicas con **Microsoft Defender for Identity**, o soluciones de terceros) |
| Snort (NIDS, análisis de tráfico de red) | **Microsoft Defender for Identity** (análisis de tráfico y comportamiento en entornos con Active Directory, detecta ataques como pass-the-hash, reconnaissance, etc.) para el ámbito de directorio; para inspección de red más genérica, Windows no trae un NIDS integrado — se recurre a soluciones de terceros o a los propios registros de auditoría avanzada de Windows |
| Logs consultados por fail2ban (`/var/log/auth.log`) | **Visor de eventos**, registro de Seguridad, especialmente los eventos 4625 (inicio de sesión fallido) y 4740 (cuenta bloqueada) |
| Antivirus/EDR (no cubierto explícitamente en el libro, pero relevante en Windows) | **Microsoft Defender Antivirus** y, en el nivel empresarial, **Microsoft Defender for Endpoint**, integrados de serie en Windows Server |

**Consultar intentos fallidos de inicio de sesión (equivalente a revisar `/var/log/auth.log` antes de que actúe fail2ban):**
```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 50
```

---

## 7. Recursos de seguridad

Los recursos generales del libro (US-CERT, SANS, Bugtraq) son universales y aplican igual para vulnerabilidades de Windows; Microsoft añade su propio canal específico:

| Recurso | Descripción |
|---|---|
| **MSRC** (Microsoft Security Response Center) | Equivalente específico de Microsoft a US-CERT: publica avisos de seguridad, boletines mensuales (Patch Tuesday) y coordina la divulgación responsable de vulnerabilidades de productos Microsoft |
| US-CERT / CISA, SANS, Bugtraq (SecurityFocus) | Siguen siendo relevantes igualmente para entornos Windows, ya que cubren vulnerabilidades de cualquier plataforma, no solo Linux |

---

## 8. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 12) | Equivalente en Windows Server 2025 |
|---|---|
| NAT / direcciones privadas | `New-NetNat`, rol RRAS; mismos rangos de IP privadas |
| `iptables` (cadenas, tablas, reglas) | Firewall de Windows Defender (`*-NetFirewallRule`), con perfiles de red como capa adicional |
| OpenSSH | **Instalado por defecto** en Windows Server 2025; mismo software (Win32-OpenSSH), mismo `sshd_config` |
| OpenVPN | RRAS (SSTP/IKEv2 por defecto en 2025) como alternativa nativa; OpenVPN también disponible sin cambios |
| `nmap` | Nmap para Windows; `Test-NetConnection` para comprobaciones puntuales |
| fail2ban | Directiva de bloqueo de cuenta + Visor de eventos (evento 4625) |
| Snort | Microsoft Defender for Identity (ámbito AD); sin NIDS genérico nativo |
| US-CERT/SANS/Bugtraq | Igual de válidos + **MSRC** como canal específico de Microsoft |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 12 ("Setting Up System Security") del libro LPIC-2. Fuentes: conocimiento general de administración de Windows Server, verificado con documentación oficial de Microsoft Learn (`New-NetFirewallRule`, instalación nativa de OpenSSH en Windows Server 2025, `Install-RemoteAccess` y el cambio de protocolos VPN por defecto en RRAS para Windows Server 2025).*

---

## Nota final

Con este capítulo se completa la contrapartida en Windows Server 2025 de los 12 capítulos del libro LPIC-2. Un patrón general se repite a lo largo de toda la serie: unas veces Windows tiene un equivalente casi 1:1 (DNS, DHCP, SMB, OpenSSH), otras veces cubre la misma necesidad con una arquitectura distinta (PAM, kernel, runlevels), y en algunos casos no hay equivalente nativo y hace falta recurrir a software de terceros (Squid, Snort, fail2ban). Tienes ahora ambas series de documentos —la de Linux (LPIC-2) y la de Windows Server 2025— organizadas capítulo a capítulo con el mismo formato, listas para comparar ambos mundos en paralelo.
