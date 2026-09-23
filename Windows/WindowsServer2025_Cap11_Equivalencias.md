# LPIC-2 · Capítulo 11: Managing Network Clients
### Equivalencias en Windows Server 2025

Este documento recoge, para cada concepto visto en el Capítulo 11 del libro LPIC-2 (servidor DHCP, PAM y OpenLDAP), cuál es su equivalente en **Windows Server 2025**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo:
- **DHCP**: Windows Server tiene su propio **rol Servidor DHCP**. Los conceptos del libro (opciones, subredes, rangos, IP fijas) son los mismos, con otros nombres: la `subnet` se llama **ámbito** (*scope*) y el `host` con IP fija se llama **reserva**. No hay archivo `dhcpd.conf`: se configura con PowerShell o con la consola `dhcpmgmt.msc`.
- **PAM**: **no existe en Windows**. La autenticación la hace siempre el **Subsistema de autoridad de seguridad local** (LSA, proceso `lsass.exe`) con sus propios métodos (Kerberos, NTLM...), y lo que en PAM se configura con módulos, en Windows se configura con **directivas de seguridad**.
- **OpenLDAP**: el directorio LDAP de Windows es **Active Directory**. Existen dos versiones: **AD DS** (el directorio completo de un dominio, con Kerberos y DNS) y **AD LDS** (un directorio LDAP "puro", sin dominio, que es lo más parecido a un OpenLDAP independiente).

Todos los comandos se ejecutan en una consola **como Administrador** (en el caso de AD, como administrador del dominio).

---

## 1. DHCP

### 1.1 Conceptos

Los mismos del libro. Vocabulario de Windows:

| Término en el libro (ISC DHCPd) | Término en Windows Server |
|---|---|
| `subnet` | **Ámbito** (*scope*) |
| `range` | Intervalo de direcciones del ámbito |
| `host` con `fixed-address` | **Reserva** |
| `shared-network` | **Superámbito** (*superscope*) |
| Lease | **Concesión** |

### 1.2 Instalación del servidor

| Elemento | Linux | Windows Server 2025 |
|---|---|---|
| Instalación | `apt-get install isc-dhcp-server` / `yum install dhcp` | `Install-WindowsFeature DHCP -IncludeManagementTools` |
| Servicio | `isc-dhcp-server` / `dhcpd` | **Servidor DHCP** (`DHCPServer`) |
| Configuración | `/etc/dhcp/dhcpd.conf` | Base de datos `C:\Windows\System32\dhcp\dhcp.mdb`, gestionada con el módulo **DhcpServer** de PowerShell, `netsh dhcp` o `dhcpmgmt.msc` |
| Elegir la tarjeta de red en la que escucha | `/etc/default/isc-dhcp-server` | `Set-DhcpServerv4Binding -InterfaceAlias "Ethernet" -BindingState $true` |

**Pasos después de instalar** (propios de Windows):

| Comando | Función |
|---|---|
| `Add-DhcpServerSecurityGroup` | Crea los grupos locales "Administradores de DHCP" y "Usuarios de DHCP" |
| `Add-DhcpServerInDC -DnsName srv01.empresa.local -IPAddress 10.0.0.5` | **Autoriza** el servidor en Active Directory. En un dominio, un servidor DHCP de Windows **no reparte direcciones hasta estar autorizado** (protección contra servidores DHCP "piratas") |
| `Get-DhcpServerInDC` | Lista los servidores DHCP autorizados |
| `Restart-Service DHCPServer` | Reinicia el servicio |

### 1.3 a 1.6 Opciones globales, subredes, hosts fijos y BOOTP

**Opciones globales** (equivalente a las líneas `option` al principio de `dhcpd.conf`). En Windows se llaman **opciones de servidor**:

| Línea del libro | Equivalente en Windows Server 2025 |
|---|---|
| `option domain-name-servers 10.0.0.10 10.0.0.11;` | `Set-DhcpServerv4OptionValue -DnsServer 10.0.0.10,10.0.0.11 -DnsDomain empresa.local` |
| `option smtp-server 10.0.0.100;` | `Set-DhcpServerv4OptionValue -OptionId 69 -Value 10.0.0.100` |
| `option pop-server 10.0.0.100;` | `Set-DhcpServerv4OptionValue -OptionId 70 -Value 10.0.0.100` |
| `option nntp-server 10.0.0.101;` | `Set-DhcpServerv4OptionValue -OptionId 71 -Value 10.0.0.101` |
| `option time-servers 10.0.0.150;` | `Set-DhcpServerv4OptionValue -OptionId 4 -Value 10.0.0.150` (opción 42 para servidores NTP) |

Las opciones se identifican por su **número** del estándar DHCP. Ver las opciones que conoce el servidor: `Get-DhcpServerv4OptionDefinition`. Si alguna no está predefinida, se crea con `Add-DhcpServerv4OptionDefinition`.

**Definición de subredes** (equivalente al bloque `subnet` del libro):
```
Add-DhcpServerv4Scope -Name "Red 10.1" -StartRange 10.1.0.10 -EndRange 10.1.0.200 -SubnetMask 255.255.0.0 -LeaseDuration 8.00:00:00
Set-DhcpServerv4OptionValue -ScopeId 10.1.0.0 -Router 10.1.0.1
Set-DhcpServerv4OptionValue -ScopeId 10.1.0.0 -OptionId 28 -Value 10.1.255.255
```
El ámbito se identifica por su dirección de red (`-ScopeId 10.1.0.0`).

| Directiva del libro | Equivalente en Windows Server 2025 |
|---|---|
| `subnet ... netmask ...` | `Add-DhcpServerv4Scope ... -SubnetMask` |
| `range` | `-StartRange` y `-EndRange` |
| `option router` | `Set-DhcpServerv4OptionValue -ScopeId ... -Router` |
| `option broadcast-address` | Opción 28 |
| Duración de la concesión (`default-lease-time`) | `-LeaseDuration días.horas:minutos:segundos` |
| Excluir direcciones del rango | `Add-DhcpServerv4ExclusionRange -ScopeId 10.1.0.0 -StartRange 10.1.0.50 -EndRange 10.1.0.60` |
| `shared-network nombre { ... }` | Superámbito: `Add-DhcpServerv4Superscope -SuperscopeName "Edificio A" -ScopeId 10.1.0.0,10.2.0.0` |

**Direcciones IP fijas por dispositivo** (equivalente al bloque `host shadrach` del libro): **reservas**.
```
Add-DhcpServerv4Reservation -ScopeId 10.1.0.0 -IPAddress 10.1.0.5 -ClientId "00-01-02-FE-DC-BA" -Name "shadrach"
Set-DhcpServerv4OptionValue -ReservedIP 10.1.0.5 -OptionId 12 -Value "shadrach"
```

| Directiva del libro | Equivalente en Windows Server 2025 |
|---|---|
| `host nombre { ... }` | Reserva (`Add-DhcpServerv4Reservation -Name`) |
| `hardware ethernet` | `-ClientId` (la MAC, con guiones) |
| `fixed-address` | `-IPAddress` |
| `option host-name` | Opción 12 de la reserva |
| `option router`, `netmask`... en el `host` | No hace falta repetirlas: la reserva hereda las opciones del ámbito (y se pueden cambiar con `-ReservedIP`) |
| `group { host ... host ... }` | **Directivas de DHCP**: aplican opciones a un grupo de clientes según su MAC, clase de proveedor, nombre, etc. (`Add-DhcpServerv4Policy`) |

Las opciones se aplican en este orden, de menos a más concreto: servidor → ámbito → directiva → reserva.

**BOOTP y arranque por red:**

| Directiva del libro | Equivalente en Windows Server 2025 |
|---|---|
| `allow bootp;` | Tipo de ámbito: `Set-DhcpServerv4Scope -ScopeId 10.1.0.0 -Type Both` (DHCP y BOOTP) |
| `allow booting;` | Activado por defecto |
| `filename` | Opción **67** (nombre del archivo de arranque) |
| `next-server` / `server-name` | Opción **66** (servidor de arranque) |

Estas opciones 66 y 67 son las que se usan, por ejemplo, con WDS (Capítulo 1).

### 1.7 Ficheros de estado y utilidades

| Linux | Windows Server 2025 |
|---|---|
| `/var/lib/dhcp/dhcpd.leases` | Concesiones guardadas en la base de datos. Ver: `Get-DhcpServerv4Lease -ScopeId 10.1.0.0` |
| — | `Get-DhcpServerv4ScopeStatistics` (direcciones libres y usadas de cada ámbito) |
| `arp` | `arp -a` o `Get-NetNeighbor` (Capítulo 6) |
| Registro en `/var/log/syslog` | Registros de auditoría diarios: `C:\Windows\System32\dhcp\DhcpSrvLog-<día>.log`, y el Visor de eventos (**DHCP-Server**) |
| Copia de la configuración (copiar `dhcpd.conf`) | Copia automática cada hora en `C:\Windows\System32\dhcp\backup\`. Exportar e importar todo: `Export-DhcpServer -File C:\dhcp.xml -Leases` / `Import-DhcpServer -File C:\dhcp.xml -BackupPath C:\dhcpbackup` |
| Configuración en texto | `netsh dhcp server dump > dhcp.txt` |

**Funciones adicionales** (sin equivalente en el libro):

| Función | Comando |
|---|---|
| **Conmutación por error** (dos servidores DHCP comparten los ámbitos) | `Add-DhcpServerv4Failover -Name FO1 -PartnerServer srv02.empresa.local -ScopeId 10.1.0.0 -LoadBalancePercent 50 -SharedSecret "secreto"` |
| Registrar a los clientes en el DNS | `Set-DhcpServerv4DnsSetting -DynamicUpdates Always` |
| DHCP para IPv6 | `Add-DhcpServerv6Scope` |
| Agente de retransmisión DHCP (equivale a `dhcrelay`) | Rol Enrutamiento y acceso remoto (RRAS), componente "Agente de retransmisión DHCP" |

### 1.8 Clientes DHCP

El cliente DHCP de Windows es el servicio **Cliente DHCP**, integrado en el sistema (Capítulo 6):

| Linux | Windows Server 2025 |
|---|---|
| `dhclient` / `dhcpcd` / `pump` | Servicio `Dhcp` (siempre presente) |
| `dhclient -r` / `dhclient` | `ipconfig /release` / `ipconfig /renew` |

---

## 2. PAM

### 2.1 Qué resuelve → cómo lo resuelve Windows

PAM permite que cada aplicación use distintos métodos de autenticación sin programarlos. En Windows el problema se resuelve de otra forma: **ninguna aplicación comprueba contraseñas por su cuenta**. Todas piden la autenticación al sistema, y quien la hace es el **LSA** (`lsass.exe`), usando **paquetes de seguridad**:

| Componente de Windows | Función | Parecido en Linux |
|---|---|---|
| **LSA** (`lsass.exe`) | Autentica a los usuarios, aplica las directivas de seguridad y crea el "token" con los permisos del usuario | La pila PAM |
| **Paquetes de seguridad** (SSP) | Métodos de autenticación: **Kerberos**, **NTLM**, **Negotiate** (elige Kerberos y, si no puede, NTLM), Schannel (TLS), CredSSP... | Los módulos `pam_*.so` |
| **SSPI** | Interfaz común que usan las aplicaciones para autenticar | La API de PAM |
| **Proveedores de credenciales** | Formas de iniciar sesión en la pantalla de bienvenida: contraseña, tarjeta inteligente... | Módulos `auth` de `login` |

### 2.2 Métodos de configuración

**No hay `/etc/pam.conf` ni `/etc/pam.d/`**. La configuración equivalente está en:

| Dónde | Qué se configura |
|---|---|
| **Directiva de seguridad local** (`secpol.msc`) | Contraseñas, bloqueo de cuentas, derechos de inicio de sesión, restricciones de NTLM... en un equipo |
| **Directivas de grupo** (`gpmc.msc`) | Lo mismo para todos los equipos de un dominio |
| Registro: `HKLM\SYSTEM\CurrentControlSet\Control\Lsa` | Paquetes de seguridad cargados (valores `Security Packages` y `Authentication Packages`) y opciones de LSA. **No se debe tocar** salvo instrucciones concretas |

Diferencia importante: en PAM cada servicio (`login`, `sshd`, `ftp`...) tiene sus propias reglas. En Windows las reglas son **comunes** para todo el sistema; lo que cambia según el servicio es qué **derecho de inicio de sesión** se necesita (apartado 2.5).

### 2.3 Estructura de una línea PAM

En Windows no hay reglas encadenadas. Cada **tipo** de PAM tiene su equivalente:

| Tipo de PAM | Equivalente en Windows Server 2025 |
|---|---|
| `auth` (autenticación) | Paquetes de seguridad del LSA (Kerberos en un dominio; NTLM para cuentas locales o equipos antiguos) |
| `account` (verificación de la cuenta) | Estado y restricciones de la cuenta: deshabilitada, caducada, bloqueada, horario de inicio de sesión, equipos permitidos (`Set-ADUser juan -LogonWorkstations "PC01,PC02"`, `-AccountExpirationDate`) y **derechos de inicio de sesión** |
| `password` (gestión de contraseñas) | **Directiva de contraseñas** (apartado 2.5) |
| `session` (acciones al iniciar sesión) | Perfil del usuario, **scripts de inicio de sesión** y **directivas de grupo** |

| Valor de control de PAM | Situación en Windows |
|---|---|
| `requisite`, `required`, `sufficient`, `optional` | No existen: no hay una pila configurable. El paquete **Negotiate** decide el método (primero Kerberos, después NTLM), y todas las restricciones de cuenta deben cumplirse siempre |

El ejemplo del libro (`login auth required pam_unix.so`) equivale al comportamiento por defecto de Windows: el inicio de sesión local comprueba la contraseña contra la base de datos de cuentas (SAM o Active Directory).

### 2.4 Módulos de autenticación (equivalente a la Tabla 11.2)

| Módulo PAM | Equivalente en Windows Server 2025 |
|---|---|
| `pam_unix.so` (`/etc/passwd` y `/etc/shadow`) | Cuentas locales en la base de datos **SAM** (`C:\Windows\System32\config\SAM`, no se lee ni se edita directamente), comprobadas por el paquete **NTLM** (MSV1_0). Se gestionan con `New-LocalUser`, `Set-LocalUser`, `net user` o `lusrmgr.msc` |
| `pam_krb5.so` | **Kerberos**, integrado. Es el método por defecto en un dominio de Active Directory |
| `pam_ldap.so` | Unir el equipo a **Active Directory** (`Add-Computer -DomainName empresa.local -Restart`). Para otros directorios LDAP no hay soporte integrado |
| `pam_nis.so` | No existe (los componentes de NIS de Windows se eliminaron hace años) |
| `pam_sss.so` (SSSD) | Unión al dominio (ver apartado 2.6) |
| `pam_userdb.so` | No existe |

### 2.5 Otros módulos (equivalente a la Tabla 11.3)

| Módulo PAM | Equivalente en Windows Server 2025 |
|---|---|
| `pam_access.so` / `pam_listfile.so` (quién puede entrar) | **Derechos de inicio de sesión** (`secpol.msc` → Directivas locales → Asignación de derechos de usuario): "Permitir el inicio de sesión local", "Denegar el inicio de sesión local", "Permitir inicio de sesión a través de Servicios de Escritorio remoto", "Tener acceso a este equipo desde la red" y sus versiones "Denegar" |
| `pam_chroot.so` | No existe (para FTP, aislamiento de usuarios, Capítulo 10) |
| `pam_console.so` | Derecho "Permitir el inicio de sesión local" |
| `pam_cracklib.so` (fortaleza de la contraseña) | **Directiva de contraseñas**: longitud mínima, complejidad, antigüedad e historial. Local: `net accounts /minpwlen:14 /maxpwage:90` o `secpol.msc` → Directivas de cuenta. Dominio: `Set-ADDefaultDomainPasswordPolicy -Identity empresa.local -MinPasswordLength 14 -ComplexityEnabled $true`. Distintas directivas para distintos grupos: `New-ADFineGrainedPasswordPolicy` |
| `pam_deny.so` | Cuenta deshabilitada (`Disable-LocalUser`, `Disable-ADAccount`) o derechos "Denegar..." |
| `pam_env.so` | Variables de entorno por directiva de grupo (Preferencias → Entorno) o scripts de inicio de sesión |
| `pam_lastlog.so` | `net user juan` (línea "Última sesión iniciada") o `Get-ADUser juan -Properties LastLogonDate`. Para mostrarlo al usuario al iniciar sesión: directiva "Mostrar información acerca de inicios de sesión anteriores durante el inicio de sesión de usuario" |
| `pam_limits.so` | No hay límites de recursos por usuario en general. Para el espacio en disco: **cuotas** (`fsutil quota` o las cuotas del rol Administrador de recursos del servidor de archivos, `New-FsrmQuota`) |
| Bloqueo tras intentos fallidos (`pam_faillock`, no aparece en el libro) | **Directiva de bloqueo de cuentas**: `net accounts /lockoutthreshold:5 /lockoutduration:15` o `Set-ADDefaultDomainPasswordPolicy -LockoutThreshold 5`. Ver cuentas bloqueadas: `Search-ADAccount -LockedOut`; desbloquear: `Unlock-ADAccount juan` |

Ver la directiva de contraseñas y bloqueo actual: `net accounts` (local) o `Get-ADDefaultDomainPasswordPolicy` (dominio).

### 2.6 SSSD

SSSD sirve para que un equipo Linux use las cuentas de un directorio de red. En Windows ese papel lo hace directamente la **unión al dominio**:

| Tarea | Comando en Windows Server 2025 |
|---|---|
| Unir el equipo a un dominio | `Add-Computer -DomainName empresa.local -Credential EMPRESA\Administrador -Restart` |
| Comprobar la relación con el dominio | `Test-ComputerSecureChannel` (y `-Repair` para arreglarla) o `nltest /sc_verify:empresa.local` |
| Ver el controlador de dominio que se usa | `nltest /dsgetdc:empresa.local` |
| Ver los grupos del usuario actual | `whoami /groups` |
| Ver los tiques de Kerberos (equivale a `klist` de Linux) | `klist` |

Al revés, para que equipos **Linux** usen las cuentas del Active Directory de un Windows Server, en Linux se usan precisamente SSSD (con `realmd`) o Samba `winbindd`.

---

## 3. OpenLDAP → Active Directory

### 3.1 Conceptos

Los conceptos del libro (árbol o DIT, objeto, atributo, clase de objeto, DN, esquema) son **los mismos**, porque Active Directory es un directorio LDAP. Diferencias de forma:

| Concepto | OpenLDAP (libro) | Active Directory |
|---|---|---|
| DN | `cn=rblum, dc=engineering, dc=ispnet1, dc=net` | `CN=Rich Blum,OU=Ingenieria,DC=empresa,DC=local` (se usan mucho las **unidades organizativas**, `OU`) |
| Clase de usuario | `inetOrgPerson` | `user` (la clase `inetOrgPerson` también existe) |
| Identificador de inicio de sesión | `uid` | `sAMAccountName` (`rblum`) y `userPrincipalName` (`rblum@empresa.local`) |
| Esquema | Archivos en `/etc/openldap/schema/` | Dentro del propio directorio (partición `CN=Schema,CN=Configuration,...`). Se amplía con `ldifde` o con el complemento Esquema de Active Directory |

### 3.2 Instalación y configuración del servidor

**Dos opciones:**

| Opción | Cuándo usarla | Instalación |
|---|---|---|
| **AD DS** (Servicios de dominio de Active Directory) | Directorio de un dominio: usuarios, equipos, directivas de grupo, Kerberos, DNS | `Install-WindowsFeature AD-Domain-Services -IncludeManagementTools` y después `Install-ADDSForest -DomainName empresa.local -InstallDns` (el servidor se convierte en **controlador de dominio**) |
| **AD LDS** (Active Directory Lightweight Directory Services) | Directorio LDAP independiente para aplicaciones, sin dominio (lo más parecido a OpenLDAP) | `Install-WindowsFeature ADLDS -IncludeManagementTools` y después crear una **instancia** con el asistente `C:\Windows\ADAM\adaminstall.exe` |

| Elemento de OpenLDAP | Equivalente en AD DS |
|---|---|
| Paquete `slapd` / `openldap-servers` | Rol `AD-Domain-Services` |
| Servicio `slapd` | **Servicios de dominio de Active Directory** (servicio `NTDS`, dentro de `lsass.exe`) |
| `slapd.conf` / `slapd-config` (`cn=config`) | No hay archivo: la configuración está **dentro del propio directorio**, en la partición `CN=Configuration,DC=empresa,DC=local` (una idea parecida a `cn=config`) |
| `suffix "dc=ispnet1, dc=net"` | Nombre del dominio al crearlo (`-DomainName empresa.local` → `DC=empresa,DC=local`) |
| `rootdn` | Cuenta **Administrador** del dominio y grupo **Admins. del dominio** |
| `rootpw` | Contraseña del Administrador; además se define una contraseña de **modo de restauración** (DSRM) al crear el dominio (`-SafeModeAdministratorPassword`) |
| Base de datos (`/var/lib/ldap/`) | `C:\Windows\NTDS\ntds.dit` (+ registros de transacciones) y la carpeta compartida `C:\Windows\SYSVOL` |
| Puertos 389 / 636 | 389 (LDAP) y 636 (LDAPS); además **3268 / 3269** (Catálogo global, búsquedas en todo el bosque) |

Windows Server 2025 añade un nuevo **nivel funcional** de dominio y bosque ("Windows Server 2025") y refuerza la seguridad de LDAP (por ejemplo, admite **TLS 1.3** en las conexiones LDAPS).

**Herramientas de administración:** Usuarios y equipos de Active Directory (`dsa.msc`), Centro de administración de Active Directory (`dsac.exe`), **Editor ADSI** (`adsiedit.msc`, un editor LDAP genérico) y el módulo **ActiveDirectory** de PowerShell.

### 3.3 Utilidades del lado servidor

| Utilidad de OpenLDAP | Equivalente en Windows Server 2025 |
|---|---|
| `slapd` | Servicio `NTDS` |
| `slapadd -l archivo.ldif` (carga masiva) | `ldifde -i -f archivo.ldf` (LDIF) o `csvde -i -f usuarios.csv` (CSV). **No hace falta parar el servicio**: trabajan contra el directorio en marcha |
| `slapcat` (exportar a LDIF) | `ldifde -f export.ldf -d "DC=empresa,DC=local"` o `csvde -f export.csv`. Para copias de seguridad reales: copia del **estado del sistema** con `wbadmin` (Capítulo 2) |
| `slapindex` | No hay reindexado manual: los índices se definen en el esquema y los mantiene el servicio. Mantenimiento de la base de datos (compactarla, comprobarla): `ntdsutil` |
| `slappasswd` | No hace falta: AD guarda las contraseñas protegidas automáticamente. Se cambian con `Set-ADAccountPassword` |
| `slurpd` (replicación) | **Replicación multimaestro integrada** entre controladores de dominio (todos admiten cambios). Comprobar: `repadmin /replsummary`, `repadmin /showrepl`; forzar: `repadmin /syncall /AdeP` |

**El ejemplo LDIF del libro, adaptado a Active Directory** (`usuario.ldf`):
```
dn: CN=Rich Blum,OU=Ingenieria,DC=empresa,DC=local
changetype: add
objectClass: user
cn: Rich Blum
givenName: Rich
sn: Blum
telephoneNumber: 312-555-1234
mail: rich@empresa.com
sAMAccountName: rblum
userPrincipalName: rblum@empresa.local
```
Carga: `ldifde -i -f usuario.ldf`. Detalles: en el formato de `ldifde` se indica `changetype: add`, y el usuario se crea **deshabilitado y sin contraseña**; se completa con `Set-ADAccountPassword rblum -Reset` y `Enable-ADAccount rblum`.

Lo mismo en una sola orden de PowerShell (la forma habitual en Windows):
```
New-ADUser -Name "Rich Blum" -GivenName Rich -Surname Blum -SamAccountName rblum -UserPrincipalName rblum@empresa.local -OfficePhone "312-555-1234" -EmailAddress rich@empresa.com -Path "OU=Ingenieria,DC=empresa,DC=local" -AccountPassword (Read-Host -AsSecureString) -Enabled $true
```

### 3.4 Utilidades del lado cliente

| Utilidad de OpenLDAP | Equivalente en Windows Server 2025 |
|---|---|
| `ldapadd` | `ldifde -i -f archivo.ldf`, `New-ADUser`, `New-ADGroup`, `New-ADObject` |
| `ldapdelete` | `Remove-ADUser`, `Remove-ADObject` (o `dsrm "DN"`) |
| `ldapmodify` | `ldifde -i` con `changetype: modify`, o `Set-ADUser rblum -OfficePhone "312-555-9999"` / `Set-ADObject -Replace @{atributo="valor"}` |
| `ldappasswd` | `Set-ADAccountPassword rblum -Reset` |
| `ldapsearch` | `Get-ADUser`, `Get-ADGroup`, `Get-ADObject` (con filtros LDAP), `dsquery`, o el cliente gráfico **`ldp.exe`** |

Ejemplo de búsqueda (equivalente a `ldapsearch -b "..." "(filtro)"`):
```
Get-ADObject -LDAPFilter "(&(objectClass=user)(sn=Blum))" -SearchBase "OU=Ingenieria,DC=empresa,DC=local" -Properties mail, telephoneNumber
```

**Opciones de `ldapsearch` (Tabla 11.7) y su equivalente en los cmdlets de AD:**

| Opción de `ldapsearch` | Equivalente |
|---|---|
| `-b base` | `-SearchBase "OU=...,DC=..."` |
| `-D bind` (usuario para conectarse) | `-Credential (Get-Credential)` |
| `-h host` / `-H uri` | `-Server dc01.empresa.local` (para una instancia de AD LDS: `-Server srv01:50000`, con su puerto) |
| `-x` (autenticación simple) | `-AuthType Basic` (por defecto se usa `Negotiate`, es decir, Kerberos) |
| `-Z` (TLS) | En `ldp.exe`: conexión SSL (puerto 636) o StartTLS |
| `-y passfile` | Credencial guardada y cargada con `Import-Clixml` |
| `-k` / `-K` (Kerberos) | Es el comportamiento por defecto |
| Filtro de búsqueda | `-LDAPFilter "(cn=Rich*)"` (sintaxis LDAP estándar) o `-Filter "Surname -eq 'Blum'"` (sintaxis de PowerShell) |

Detalle técnico: los cmdlets del módulo ActiveDirectory no hablan LDAP directamente, sino con los **Servicios web de Active Directory** (puerto 9389) del controlador de dominio. Las herramientas `ldp.exe`, `ldifde`, `csvde` y `dsquery` sí usan LDAP.

---

## 4. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 11) | Equivalente en Windows Server 2025 |
|---|---|
| Paquete `isc-dhcp-server` / `dhcp` | Rol Servidor DHCP (`Install-WindowsFeature DHCP`) |
| `/etc/dhcp/dhcpd.conf` | Base de datos `dhcp.mdb`, módulo DhcpServer, `netsh dhcp`, `dhcpmgmt.msc` |
| — | Autorizar el servidor en AD (`Add-DhcpServerInDC`) |
| `option ...` globales | Opciones de servidor (`Set-DhcpServerv4OptionValue`) |
| `subnet` + `range` | Ámbito (`Add-DhcpServerv4Scope`) |
| `shared-network` | Superámbito |
| `host` + `fixed-address` | Reserva (`Add-DhcpServerv4Reservation`) |
| `group` | Directivas de DHCP |
| `filename` / `next-server` | Opciones 67 / 66 |
| `dhcpd.leases` | `Get-DhcpServerv4Lease` |
| `dhclient` | Servicio Cliente DHCP (`ipconfig /renew`) |
| PAM | LSA + paquetes de seguridad (Kerberos, NTLM, Negotiate) |
| `/etc/pam.conf`, `/etc/pam.d/` | Directiva de seguridad local y directivas de grupo |
| `/etc/shadow` / `pam_unix.so` | Base de datos SAM (cuentas locales) |
| `pam_krb5.so` | Kerberos (integrado) |
| `pam_ldap.so` / `pam_sss.so` / SSSD | Unión al dominio de Active Directory |
| `pam_access.so` / `pam_listfile.so` | Derechos de inicio de sesión |
| `pam_cracklib.so` | Directiva de contraseñas (`net accounts`, `Set-ADDefaultDomainPasswordPolicy`) |
| `pam_lastlog.so` | `net user`, `LastLogonDate` |
| `pam_limits.so` | Sin equivalente general (cuotas de disco) |
| OpenLDAP | AD DS (dominio) o AD LDS (LDAP independiente) |
| `slapd.conf` / `cn=config` | Partición de configuración de AD |
| `suffix` / `rootdn` / `rootpw` | Nombre del dominio / Administrador / contraseña + DSRM |
| `slapadd` / `slapcat` | `ldifde` / `csvde` (sin parar el servicio) |
| `slurpd` | Replicación multimaestro (`repadmin`) |
| `ldapadd` / `ldapmodify` / `ldapdelete` | `ldifde`, `New-ADUser` / `Set-ADUser` / `Remove-ADObject` |
| `ldappasswd` | `Set-ADAccountPassword` |
| `ldapsearch` | `Get-ADObject -LDAPFilter`, `Get-ADUser`, `dsquery`, `ldp.exe` |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 11 ("Managing Network Clients") del libro LPIC-2. Fuentes: documentación oficial de Microsoft Learn (Servidor DHCP y módulo DhcpServer de PowerShell, autorización en Active Directory, conmutación por error de DHCP, arquitectura de autenticación de Windows y LSA, directivas de contraseñas y bloqueo de cuentas, derechos de usuario, AD DS y AD LDS, módulo ActiveDirectory de PowerShell, ldifde, csvde, repadmin y novedades de Active Directory en Windows Server 2025).*
