# LPIC-2 · Capítulo 8: Directing DNS
### Equivalencias en Windows Server 2025

A diferencia del correo (Capítulo 7), aquí Windows Server sí ofrece un **rol nativo equivalente a BIND**: el rol **DNS Server**, integrado en el propio sistema operativo desde siempre y muy usado tanto de forma independiente como junto a Active Directory.

---

## 1. Instalación del servidor DNS

| Concepto Linux (BIND) | Equivalente en Windows Server 2025 |
|---|---|
| Paquetes `bind`/`bind9` | Rol **DNS Server** |
| Instalación | `Install-WindowsFeature DNS -IncludeManagementTools` (PowerShell), o desde el Administrador del servidor → Agregar roles y características |
| Demonio | Servicio **DNS Server** (`DNS`), gestionable con `Get-Service DNS`, `Restart-Service DNS` |
| Usuario del proceso | Se ejecuta bajo la cuenta de servicio del sistema (`NT AUTHORITY\NetworkService` o similar), sin un usuario dedicado tipo `named`/`bind` como en Linux |

---

## 2. Configuración del servidor: sin `named.conf`

Diferencia clave: Windows **no tiene un fichero de configuración de texto único** equivalente a `named.conf`. Toda la configuración (opciones globales, zonas, forwarders, ACLs) se guarda en el **Registro** y en la propia base de datos del servidor DNS (o en Active Directory, si las zonas están integradas en el directorio), y se administra mediante:

| Herramienta | Tipo |
|---|---|
| **Administrador de DNS** (`dnsmgmt.msc`) | Consola gráfica (MMC), el equivalente visual más directo a editar `named.conf` a mano |
| **`dnscmd`** | Herramienta de línea de comandos clásica (heredada, pero aún funcional) |
| **Módulo `DnsServer` de PowerShell** | Conjunto moderno de cmdlets (`*-DnsServer*`), la vía recomendada para automatizar cualquier tarea |

**Comprobar la sintaxis / estado general (equivalente a `named-checkconf`):**
```powershell
Get-DnsServerSetting
Get-DnsServer
```
No existe un "comprobador de sintaxis" independiente porque no hay un fichero de texto que pueda tener errores de sintaxis: los cmdlets validan los parámetros al ejecutarse.

---

## 3. Arrancar, parar y recargar

| Acción | Comando |
|---|---|
| Arrancar el servicio | `Start-Service DNS` |
| Parar el servicio | `Stop-Service DNS` |
| Reiniciar | `Restart-Service DNS` |
| Vaciar la caché de resolución del servidor (equivalente a `rndc flush`) | `Clear-DnsServerCache` |
| Vaciar la caché de un nombre concreto (equivalente a `rndc flushname`) | `Clear-DnsServerCache -Name nombre` |

---

## 4. Servidor DNS solo caché (caching-only / resolver)

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| `recursion yes;`, `allow-query` | Windows permite recursividad por defecto; se restringe con `Set-DnsServerRecursion` y listas de control con `Set-DnsServerQueryResolutionPolicy` |
| Configurar reenviadores (`forward`, `forwarders`) | `Add-DnsServerForwarder -IPAddress 8.8.8.8,8.8.4.4` (equivalente directo a `forwarders { IP; ... };`) |
| Instalar el rol sin crear ninguna zona propia | Instalar el rol DNS y solo configurar `Add-DnsServerForwarder`, sin usar `Add-DnsServerPrimaryZone` — el servidor queda funcionando como resolutor con caché, igual que un BIND configurado solo con `forward` |

---

## 5. Registro de eventos (logging)

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| Bloque `logging { }` en `named.conf`, canales/categorías | **Visor de eventos**, registro específico "DNS Server" (`Applications and Services Logs\Microsoft\Windows\DNS-Server`) |
| Registro de consultas (`queries`) | `Set-DnsServerDiagnostics -QueryLogging $true` (activa el registro detallado de consultas) |
| Fichero de log de depuración | `Set-DnsServerDiagnostics -EnableLoggingToFile $true -LogFilePath ruta` |

---

## 6. Zonas DNS

### 6.1 Tipos de zona: correspondencia directa

Windows DNS usa prácticamente la misma terminología que BIND para los tipos de zona, lo que hace esta sección muy directa:

| Tipo en BIND | Equivalente en Windows DNS |
|---|---|
| `master` | **Zona principal** (Primary zone) |
| `slave` | **Zona secundaria** (Secondary zone) |
| `stub` | **Zona de código auxiliar** (Stub zone) — mismo concepto: solo transfiere los registros NS |
| `forward` | **Reenviador condicional** (Conditional Forwarder) — reenvía consultas de un dominio concreto a otros servidores |
| `hint` | **Sugerencias de raíz** (Root Hints), gestionadas con `Get-DnsServerRootHint`/`Add-DnsServerRootHint` |
| Zona integrada en Active Directory (sin equivalente directo en BIND) | **Zona integrada en AD** (Active Directory-integrated zone): replica la zona automáticamente entre todos los controladores de dominio con el rol DNS, sin transferencias de zona explícitas |

### 6.2 Cmdlets para crear zonas

| Cmdlet | Equivalente a... |
|---|---|
| `Add-DnsServerPrimaryZone -Name "empresa.com" -ZoneFile "empresa.com.dns"` | Crear una zona `master` con fichero de zona en texto |
| `Add-DnsServerPrimaryZone -Name "empresa.com" -ReplicationScope "Domain"` | Crear una zona principal integrada en Active Directory (sin equivalente en BIND) |
| `Add-DnsServerSecondaryZone -Name "empresa.com" -MasterServers IP` | Crear una zona `slave` |
| `Add-DnsServerStubZone -Name "empresa.com" -MasterServers IP` | Crear una zona `stub` |
| `Add-DnsServerConditionalForwarderZone -Name "empresa.com" -MasterServers IP` | Crear una zona `forward` para un dominio concreto |
| `Add-DnsServerZoneDelegation` | Delegar una subzona a otro servidor (equivalente al registro NS + glue del libro) |

### 6.3 Ubicación de las bases de datos de zona (cuando no están integradas en AD)

| Elemento | Windows Server 2025 |
|---|---|
| Carpeta de ficheros de zona | `%SystemRoot%\System32\dns\` |
| Formato del fichero | **Compatible con el formato de zona de BIND** (mismo formato de texto con `$TTL`, `$ORIGIN`, y registros `SOA`/`NS`/`A`/`MX`/`CNAME`/`PTR`), lo que permite migrar zonas entre BIND y Windows DNS con cambios mínimos |

### 6.4 Registros de recursos (Resource Records): mismos tipos, cmdlets específicos

| Tipo de registro | Cmdlet para añadirlo |
|---|---|
| `A` | `Add-DnsServerResourceRecordA -ZoneName "empresa.com" -Name "host1" -IPv4Address 192.168.1.10` |
| `AAAA` | `Add-DnsServerResourceRecordAAAA` |
| `CNAME` | `Add-DnsServerResourceRecordCName -ZoneName "empresa.com" -Name "www" -HostNameAlias "host1.empresa.com"` |
| `MX` | `Add-DnsServerResourceRecordMX` |
| `NS` | `Add-DnsServerResourceRecord -NS` |
| `PTR` (zona inversa) | `Add-DnsServerResourceRecordPtr -ZoneName "1.168.192.in-addr.arpa" -Name "10" -PtrDomainName "host1.empresa.com"` |
| `TXT` | `Add-DnsServerResourceRecord -Txt` |
| Consultar registros de una zona | `Get-DnsServerResourceRecord -ZoneName "empresa.com"` (equivalente a leer el fichero de zona) |

**Crear un registro A con su PTR asociado en un solo paso (más cómodo que en BIND, donde hay que editar dos zonas por separado):**
```powershell
Add-DnsServerResourceRecordA -ZoneName "empresa.com" -Name "host1" -IPv4Address 192.168.1.10 -CreatePtr
```

**Crear la zona inversa (equivalente a la zona `in-addr.arpa` del libro):**
```powershell
Add-DnsServerPrimaryZone -NetworkID "192.168.1.0/24" -ZoneFile "1.168.192.in-addr.arpa.dns"
```

### 6.5 Comprobar una zona

No existe un `named-checkzone` independiente; la validación ocurre al añadir o modificar registros con los cmdlets, que rechazan datos incorrectos directamente.

---

## 7. Herramientas de diagnóstico

| Herramienta Linux | Equivalente en Windows Server 2025 |
|---|---|
| `host` | `nslookup` (modo básico) |
| `dig` | **`Resolve-DnsName`** (PowerShell, salida estructurada y más moderna que `nslookup`) |
| `dig @servidor nombre` | `Resolve-DnsName nombre -Server IP` |
| `dig +trace` | No hay un equivalente de un solo comando; se puede reconstruir consultando manualmente desde la raíz con `Resolve-DnsName -Server` apuntando a cada nivel |
| `nslookup -query=ns dominio` | `Resolve-DnsName dominio -Type NS` |
| `rndc reload` / `rndc reload zone` | `Restart-Service DNS` (recarga completa) — Windows no tiene un comando de recarga selectiva de una sola zona tan directo; normalmente los cambios vía cmdlets se aplican sin necesidad de recargar |

**Ejemplos con `Resolve-DnsName` (el sustituto natural de `dig`):**
```powershell
Resolve-DnsName empresa.com -Type MX
Resolve-DnsName empresa.com -Server 8.8.8.8
```

---

## 8. Seguridad del servidor DNS

Las mismas buenas prácticas del libro aplican en Windows, con herramientas equivalentes:

| Buena práctica (BIND) | Equivalente en Windows Server 2025 |
|---|---|
| Mantener BIND actualizado | Windows Update / Windows Server Update Services (WSUS) |
| Ejecutar como usuario no root | El servicio DNS ya corre con privilegios reducidos por defecto (cuenta de servicio del sistema) |
| Ocultar la versión de BIND | No aplica igual: Windows DNS no expone su versión por consulta `CHAOS TXT version.bind` como BIND |
| Vistas (`view`) para clientes internos/externos | **Directivas de zona DNS** (`Add-DnsServerQueryResolutionPolicy`), que permiten dar respuestas distintas según el origen del cliente, similar en espíritu a las vistas de BIND |
| Desactivar actualizaciones dinámicas (`allow-update { none; };`) | `Set-DnsServerPrimaryZone -Name "empresa.com" -DynamicUpdate None` |
| Desactivar transferencias de zona globalmente | `Set-DnsServerPrimaryZone -Name "empresa.com" -SecureSecondaries NoTransfer` (o `TransferToZoneNameServer`/`TransferToSecureServers`, según el nivel de restricción deseado) |
| chroot jail | **No aplica en Windows**: el modelo de aislamiento de procesos de Windows es distinto (no existe un chroot equivalente); el aislamiento se consigue con máquinas virtuales, contenedores Windows, o ejecutando el rol en un servidor dedicado |

### 8.1 DNSSEC y TSIG

Windows DNS Server soporta **DNSSEC de forma nativa** desde Windows Server 2012, con un conjunto de cmdlets propio:

| Concepto Linux (BIND) | Equivalente en Windows Server 2025 |
|---|---|
| `dnssec-keygen` (generar claves ZSK/KSK) | `Add-DnsServerSigningKey` (añade una clave KSK o ZSK a una zona) |
| `dnssec-signzone` (firmar la zona) | `ConvertTo-DnsServerSignedZone -ZoneName "empresa.com"` (firma la zona automáticamente) |
| Rotación de claves | `Enable-DnsServerSigningKeyRollover` / `Disable-DnsServerSigningKeyRollover` |
| `dnssec-dsfromkey` (registro DS para la zona padre) | `Export-DnsServerDnsSecPublicKey` / `Export-DnsServerZoneKeySigningKeyDelegation` |
| Trust anchor | `Add-DnsServerTrustAnchor` |
| TSIG (autenticación de transferencias) | Windows DNS también soporta TSIG, configurable en las propiedades de la zona secundaria/reenviador junto con la clave compartida |

---

## 9. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 8) | Equivalente en Windows Server 2025 |
|---|---|
| Paquete `bind`/`bind9` | Rol **DNS Server** (`Install-WindowsFeature DNS`) |
| `named.conf` | Sin fichero único; Administrador de DNS / `DnsServer` (PowerShell) / `dnscmd` |
| Zona `master`/`slave`/`stub`/`forward`/`hint` | Primary / Secondary / Stub / Conditional Forwarder / Root Hints (misma terminología conceptual) |
| Ficheros de zona (`/var/named`, `/etc/bind`) | `%SystemRoot%\System32\dns\` (formato compatible con BIND) |
| Registros A/AAAA/CNAME/MX/NS/PTR | `Add-DnsServerResourceRecord*` |
| `rndc` | `Restart-Service DNS`, `Clear-DnsServerCache` |
| `dig`, `host`, `nslookup` | `Resolve-DnsName`, `nslookup` (sigue existiendo igual) |
| Vistas (`view`) | Directivas de zona (`*-DnsServerQueryResolutionPolicy`) |
| chroot jail | No aplica; aislamiento vía VM/contenedores |
| DNSSEC (`dnssec-keygen`/`dnssec-signzone`) | `Add-DnsServerSigningKey`, `ConvertTo-DnsServerSignedZone` |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 8 ("Directing DNS") del libro LPIC-2. Fuentes: conocimiento general de administración de Windows Server, verificado con documentación oficial de Microsoft Learn (módulo `DnsServer` de PowerShell, cmdlets de zonas, registros y DNSSEC específicos de Windows Server 2025).*
