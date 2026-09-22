# LPIC-2 · Capítulo 11: Managing Network Clients
### Equivalencias en Windows Server 2025

---

## 1. DHCP

Windows Server incluye un rol **DHCP Server** nativo, con la misma función y conceptos que el ISC DHCPd del libro, aunque sin fichero de configuración de texto.

| Concepto Linux (`dhcpd.conf`) | Equivalente en Windows Server 2025 |
|---|---|
| Paquete `isc-dhcp-server`/`dhcp` | Rol **DHCP Server**: `Install-WindowsFeature DHCP -IncludeManagementTools` |
| Autorización del servidor (no existe en Linux; peculiaridad de Windows en entornos AD) | `Add-DhcpServerInDC` — en un dominio Active Directory, un servidor DHCP debe **autorizarse** explícitamente antes de poder responder peticiones, medida de seguridad sin equivalente en Linux |
| `subnet ... netmask ... { }` | **Ámbito** (Scope): `Add-DhcpServerv4Scope -Name "Red1" -StartRange IP -EndRange IP -SubnetMask máscara` |
| `option router`, `option domain-name-servers` (opciones globales/por subred) | `Set-DhcpServerv4OptionValue -ScopeId IP_red -Router IP_gw -DnsServer IP_dns -DnsDomain dominio` |
| `range` (rango de asignación dinámica) | Definido directamente en `Add-DhcpServerv4Scope` con `-StartRange`/`-EndRange` |
| Excluir un rango de la asignación dinámica | `Add-DhcpServerv4ExclusionRange -ScopeId IP_red -StartRange IP -EndRange IP` |
| `host nombre { hardware ethernet MAC; fixed-address IP; }` (IP fija por dispositivo) | `Add-DhcpServerv4Reservation -ScopeId IP_red -IPAddress IP -ClientId MAC -Description "texto"` |
| `group { }` (agrupar hosts con opciones comunes) | **Directivas** (Policies) de DHCP, con `Add-DhcpServerv4Policy`, que aplican opciones distintas según condiciones (MAC, nombre de clase de proveedor, etc.) |
| Soporte BOOTP (`allow booting`, `filename`, `next-server`) | Windows DHCP también soporta BOOTP/PXE mediante opciones específicas del ámbito (opción 66/67, `next-server`/`boot filename`), muy usado junto con **Windows Deployment Services (WDS)** |
| `/var/lib/dhcp/dhcpd.leases` | `Get-DhcpServerv4Lease -ScopeId IP_red` (consulta las concesiones activas directamente desde el propio servidor, sin fichero de texto) |
| Clientes DHCP (`dhclient`, `dhcpcd`) | Cliente DHCP integrado en la pila TCP/IP de Windows (ver también Capítulo 6) |

**Flujo completo de creación (equivalente al `dhcpd.conf` de ejemplo del libro):**
```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
Add-DhcpServerInDC -DnsName "dhcp01.empresa.local" -IPAddress 192.168.1.10
Add-DhcpServerv4Scope -Name "Oficina Principal" -StartRange 192.168.1.100 `
  -EndRange 192.168.1.200 -SubnetMask 255.255.255.0 -State Active
Set-DhcpServerv4OptionValue -ScopeId 192.168.1.0 -Router 192.168.1.1 `
  -DnsServer 192.168.1.10,192.168.1.11 -DnsDomain empresa.local
Add-DhcpServerv4Reservation -ScopeId 192.168.1.0 -IPAddress 192.168.1.5 `
  -ClientId "AA-BB-CC-DD-EE-FF" -Description "Impresora planta 4"
```

---

## 2. PAM: sin equivalente directo, modelo de autenticación distinto

Windows **no tiene un sistema modular tipo PAM**. La autenticación está integrada en el propio sistema operativo a través de la **LSA (Local Security Authority)**, con un modelo distinto pero que cubre funciones equivalentes:

| Concepto Linux (PAM) | Equivalente conceptual en Windows Server 2025 |
|---|---|
| `/etc/pam.d/` (módulos por servicio) | No existe una configuración por aplicación; la autenticación pasa siempre por la **LSA** de forma centralizada, con "paquetes de autenticación" internos (Kerberos, NTLM, Negotiate) equivalentes en espíritu a `pam_krb5.so`/`pam_unix.so` |
| Proveedores de autenticación (`pam_unix.so`, `pam_ldap.so`, `pam_krb5.so`) | **Proveedores de credenciales** (Credential Providers): módulos que Windows carga en la pantalla de inicio de sesión para aceptar distintos métodos (contraseña, PIN, biometría con Windows Hello for Business, tarjeta inteligente) |
| `pam_cracklib.so` (política de complejidad de contraseñas) | **Directiva de contraseñas** de Active Directory / directiva de seguridad local (`secpol.msc`, o GPO "Password must meet complexity requirements") |
| `pam_limits.so` (límites de recursos por sesión) | **Cuotas de recursos** gestionadas por otros mecanismos (Directiva de grupo, Resource Manager), no integradas en el flujo de login como en PAM |
| `pam_listfile.so` (permitir/denegar según lista) | Directiva "Iniciar sesión localmente" / "Denegar el inicio de sesión localmente" en Directiva de grupo, que restringe por usuario o grupo qué cuentas pueden autenticarse en una máquina |
| SSSD (autenticación contra LDAP/AD desde Linux) | En sentido inverso, Windows autentica de forma nativa contra **Active Directory** sin necesidad de una capa adicional — es Linux quien necesita SSSD para hablar con AD, no al revés |

**Idea clave a transmitir:** mientras que PAM es una capa *añadida* y configurable módulo a módulo, en Windows la autenticación es una función *integrada* del núcleo del sistema (LSA), y su personalización se hace mediante Directiva de grupo y proveedores de credenciales, no editando ficheros de reglas por servicio.

---

## 3. Cliente LDAP

Windows incluye herramientas de cliente LDAP nativas, ya que Active Directory es en sí un servicio compatible con LDAP.

| Concepto Linux (`ldapsearch`, `ldapadd`...) | Equivalente en Windows Server 2025 |
|---|---|
| `ldapsearch` | **`ldp.exe`** (herramienta gráfica, incluida al instalar el rol AD DS, permite conectar/enlazar/buscar contra cualquier directorio compatible con LDAP), o `dsquery` (línea de comandos, más orientada a Active Directory) |
| `ldapadd`/`ldapmodify`/`ldapdelete` | **`ldifde`** (importa/exporta/modifica objetos usando ficheros LDIF, igual formato que en Linux) |
| `ldappasswd` | `dsmod user -pwd` (cambia contraseñas desde línea de comandos), o `Set-ADAccountPassword` (PowerShell) |
| Opciones de `ldapsearch` (`-b` base, `-D` bind, `-x` simple, `-Z` TLS) | Parámetros equivalentes en `ldp.exe` (cuadros de diálogo "Connect"/"Bind") y en `dsquery`/`Get-ADObject` (`-SearchBase`, `-Credential`, `-Server`) |
| Consulta estructurada de objetos vía PowerShell (no existe un cmdlet nativo así en Linux) | **`Get-ADUser`**, `Get-ADGroup`, `Get-ADComputer`, `Get-ADObject` (módulo `ActiveDirectory` de PowerShell), que devuelven objetos estructurados en vez de texto plano |

**Ejemplo de consulta LDAP (equivalente a `ldapsearch -b "dc=empresa,dc=local" "(cn=rblum)"`):**
```powershell
Get-ADUser -Filter "Name -eq 'rblum'" -SearchBase "DC=empresa,DC=local" -Properties *
```

---

## 4. Servidor de directorio: OpenLDAP → Active Directory Domain Services (AD DS)

Windows no ofrece un rol equivalente a "un servidor OpenLDAP independiente y genérico"; su servicio de directorio es **Active Directory Domain Services (AD DS)**, que además de LDAP incluye Kerberos, DNS integrado y gestión de dominio — mucho más amplio que OpenLDAP, pero que cubre la misma necesidad de fondo (un directorio centralizado de usuarios, grupos y objetos).

| Concepto Linux (OpenLDAP) | Equivalente en Windows Server 2025 (AD DS) |
|---|---|
| DN (Distinguished Name), object class, schema | Mismos conceptos: AD DS es un directorio LDAP y usa DN/objectClass/schema de la misma forma |
| `suffix "dc=empresa,dc=local"` | El **dominio** de Active Directory en sí (ej. `empresa.local`), definido al crear el bosque |
| `rootdn`/`rootpw` (administrador del directorio) | La cuenta de **Administrador de dominio**, con privilegios totales sobre el directorio |
| `slapd.conf` / `slapd-config` | No hay fichero de configuración de texto: la configuración del bosque/dominio se gestiona con los cmdlets del módulo `ADDSDeployment` y se almacena en la propia base de datos del directorio |
| Instalar el servicio | `Install-WindowsFeature AD-Domain-Services -IncludeManagementTools`, seguido de `Install-ADDSForest -DomainName "empresa.local" -InstallDNS` (crea un bosque/dominio nuevo, con DNS integrado) |
| `slapd` (demonio del servidor) | Servicio **Active Directory Domain Services** (NTDS), y **KDC de Kerberos** integrado |
| Base de datos del directorio | `ntds.dit` (equivalente a la base de datos interna de `slapd`, normalmente en `%SystemRoot%\NTDS\`) |
| `slapadd`/`slapcat` (importar/exportar directamente en la base de datos, con el servidor parado) | `ntdsutil` (mantenimiento offline de la base de datos AD, incluyendo copias de seguridad, defragmentación, restauración) |
| `slappasswd` (generar contraseña cifrada) | Las contraseñas de AD se gestionan siempre a través de la API de directorio, nunca manualmente como un hash en un fichero |
| Crear un nuevo objeto (usuario) | `New-ADUser -Name "Rich Blum" -SamAccountName rblum -Path "OU=Empleados,DC=empresa,DC=local"` |
| `New-ADGroup`, `New-ADOrganizationalUnit` | Crear grupos y unidades organizativas (OU), conceptos también presentes en el árbol de OpenLDAP aunque con nombres distintos |

**Ejemplo de creación de un usuario (equivalente al fichero LDIF de ejemplo del libro):**
```powershell
New-ADUser -Name "Rich Blum" -GivenName "Rich" -Surname "Blum" `
  -SamAccountName "rblum" -UserPrincipalName "rblum@empresa.local" `
  -EmailAddress "rich@empresa.local" -Path "OU=Empleados,DC=empresa,DC=local" `
  -Enabled $true -AccountPassword (ConvertTo-SecureString "Clave.Segura1" -AsPlainText -Force)
```

### 4.1 LDAP seguro (LDAPS)

Igual que en OpenLDAP (donde `ldapsearch -Z` inicia TLS), Active Directory también soporta LDAP cifrado:

| Elemento | Windows Server 2025 |
|---|---|
| Puerto LDAP sin cifrar | TCP 389 (igual que en OpenLDAP) |
| Puerto LDAPS (LDAP sobre SSL/TLS) | TCP 636 (igual que en OpenLDAP) |
| Activar LDAPS | Requiere instalar un certificado válido en el controlador de dominio (vía autoridad de certificación interna o comercial) |
| Comprobar la conexión LDAPS | `ldp.exe`, conectando al puerto 636 con SSL activado |

---

## 5. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 11) | Equivalente en Windows Server 2025 |
|---|---|
| Paquete `isc-dhcp-server`/`dhcp` | Rol **DHCP Server** |
| `dhcpd.conf` (subnet/host/group) | `Add-DhcpServerv4Scope`, `Add-DhcpServerv4Reservation`, `Add-DhcpServerv4Policy` |
| `dhcpd.leases` | `Get-DhcpServerv4Lease` |
| PAM (`pam.d`, módulos) | LSA + Proveedores de credenciales + Directiva de grupo (sin fichero de reglas por servicio) |
| `ldapsearch`/`ldapadd`/`ldapmodify` | `ldp.exe`, `dsquery`, `ldifde` |
| OpenLDAP (`slapd`, `slapd.conf`) | **Active Directory Domain Services** (`Install-ADDSForest`) |
| `slapadd`/`slapcat` | `ntdsutil` |
| Crear un usuario vía LDIF | `New-ADUser` |
| Puertos LDAP/LDAPS (389/636) | Mismos puertos, igual concepto |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 11 ("Managing Network Clients") del libro LPIC-2. Fuentes: conocimiento general de administración de Windows Server, verificado con documentación oficial de Microsoft Learn (`Add-DhcpServerv4Scope`, `Add-DhcpServerv4Reservation`, `Install-ADDSForest` para Windows Server 2025, y herramientas `ldp.exe`/`dsquery`).*
