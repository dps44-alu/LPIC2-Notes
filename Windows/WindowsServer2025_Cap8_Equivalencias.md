# LPIC-2 · Capítulo 8: Directing DNS
### Equivalencias en Windows Server 2025

Este documento recoge, para cada concepto visto en el Capítulo 8 del libro LPIC-2 (servidor DNS BIND, zonas, diagnóstico y seguridad), cuál es su equivalente en **Windows Server 2025**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo: Windows Server no usa BIND, sino su propio **Servidor DNS**, que es un **rol** del sistema. Los conceptos del libro (zonas, registros, SOA, transferencias, reenviadores, DNSSEC) son **los mismos**, porque DNS es un estándar. Lo que cambia es:
- **No hay `named.conf`**: la configuración se guarda en el Registro y se gestiona con el módulo **DnsServer** de PowerShell, con la consola gráfica **Administrador de DNS** (`dnsmgmt.msc`) o con la herramienta clásica `dnscmd`.
- Las zonas pueden guardarse en **archivos de texto** (con el mismo formato que BIND) o, lo más habitual, **dentro de Active Directory** ("zonas integradas en AD"), que se replican solas entre controladores de dominio.
- El servidor DNS de Windows está muy ligado a **Active Directory**: al crear un controlador de dominio se instala y configura automáticamente, porque AD depende del DNS para funcionar.

Todos los comandos se ejecutan en una consola **como Administrador**.

---

## 1. Instalación del servidor DNS

| Elemento | Red Hat | Debian | Windows Server 2025 |
|---|---|---|---|
| Instalación | `yum install bind bind-utils` | `apt-get install bind9 bind9utils` | `Install-WindowsFeature DNS -IncludeManagementTools` |
| Servicio | `named` | `bind9` | **Servidor DNS** (nombre del servicio: `DNS`) |
| Proceso | `named` | `named` | `dns.exe` |
| Usuario del proceso | `named` | `bind` | Cuenta **LocalSystem** (no se puede cambiar) |
| Herramientas de consulta (`bind-utils`) | Paquete aparte | Paquete aparte | `nslookup` y `Resolve-DnsName`, integrados en el sistema (Capítulo 6) |

Si el servidor se promociona a **controlador de dominio**, el asistente instala el rol DNS automáticamente.

Comprobaciones típicas:
```
Get-WindowsFeature DNS          # ¿Está instalado el rol?
Get-Service DNS                 # ¿Está en marcha el servicio?
Get-Process dns                 # Proceso del servidor DNS
```

---

## 2. El equivalente de `named.conf`

**No existe un archivo de configuración.** La configuración global se guarda en el Registro (`HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters`) y la de las zonas en el Registro o en Active Directory. Se consulta toda de una vez con:
```
Get-DnsServer
```

| Opción de `named.conf` | Equivalente en Windows Server 2025 |
|---|---|
| `listen-on port 53 { ... }` | `Set-DnsServerSetting -ListeningIPAddress @("192.168.1.10")` (o Administrador de DNS → Propiedades del servidor → pestaña **Interfaces**). Por defecto escucha en todas las IP |
| `directory` | Carpeta fija: `C:\Windows\System32\dns\` |
| `allow-query` | No hay una opción global. Se usan **directivas de resolución de consultas** (apartado 4) o el firewall |
| `recursion yes \| no` | `Set-DnsServerRecursion -Enable $true` / `$false` (consultar con `Get-DnsServerRecursion`) |
| `forwarders` | `Set-DnsServerForwarder -IPAddress 8.8.8.8,1.1.1.1` |
| `dnssec-validation` | Anclajes de confianza (*trust anchors*): `Add-DnsServerTrustAnchor -Root` instala los de la zona raíz para validar respuestas firmadas (apartado 8.2) |
| `include` | No aplica |

Todos los cmdlets del módulo DnsServer admiten `-ComputerName`, así que se puede administrar un servidor DNS remoto desde otro equipo.

**Comprobar la configuración** (equivalente a `named-checkconf`): no hay un comprobador de sintaxis, porque no hay archivo que escribir a mano. Sí hay herramientas de comprobación:

| Comando | Función |
|---|---|
| `Test-DnsServer -IPAddress 127.0.0.1` | Comprueba si el servidor responde |
| `Test-DnsServer -IPAddress 127.0.0.1 -ZoneName empresa.com` | Comprueba si el servidor responde con autoridad para una zona |
| Administrador de DNS → Propiedades del servidor → pestaña **Supervisión** | Pruebas de consulta simple y recursiva |
| `Invoke-BpaModel -ModelId Microsoft/Windows/DNSServer` y después `Get-BpaResult -ModelId Microsoft/Windows/DNSServer` | **Analizador de procedimientos recomendados**: revisa la configuración y avisa de errores habituales |

---

## 3. Arrancar, parar y recargar el servidor DNS

| Acción | Linux | Windows Server 2025 |
|---|---|---|
| Arrancar | `systemctl start named` | `Start-Service DNS` (o `net start dns`) |
| Parar | `systemctl stop named` | `Stop-Service DNS` |
| Reiniciar | `systemctl restart named` | `Restart-Service DNS` |
| Recargar la configuración | `rndc reload` | En general no hace falta: los cambios hechos con los cmdlets o la consola se aplican al momento |

---

## 4. Servidor DNS solo caché (caching-only / resolver)

En Windows, un servidor DNS **sin zonas** ya funciona como servidor de caché: resuelve cualquier nombre usando las **sugerencias de raíz** (*root hints*) y guarda las respuestas en caché.

| Paso del libro | Equivalente en Windows Server 2025 |
|---|---|
| Instalar el servidor | `Install-WindowsFeature DNS -IncludeManagementTools` |
| 1-2. `acl` + `allow-query` (quién puede consultar) | Directivas de resolución de consultas (ver ejemplo abajo) |
| 3. `recursion yes;` | Activada por defecto. Comprobar: `Get-DnsServerRecursion` |
| 4. `listen-on` | `Set-DnsServerSetting -ListeningIPAddress ...` |
| 5. `named-checkconf` | `Test-DnsServer` |
| 6. Abrir el puerto 53 en el firewall | Automático: al instalar el rol se crean y activan las reglas del firewall para DNS |
| 7. Reiniciar el servicio | No hace falta |

**Limitar quién puede usar el servidor** (equivalente a `acl` + `allow-query`): se define la subred de los clientes y se crea una directiva que **ignore** las consultas del resto:
```
Add-DnsServerClientSubnet -Name "LAN" -IPv4Subnet 192.168.0.0/16
Add-DnsServerQueryResolutionPolicy -Name "SoloLAN" -Action IGNORE -ClientSubnet "ne,LAN"
```
(`ne` = "no es igual a": se ignoran las consultas de clientes que **no** estén en la subred LAN.)

**Reenviadores** (equivalente a `forwarders`, `forward only` y `forward first`):

| Comando | Función | Equivalente BIND |
|---|---|---|
| `Set-DnsServerForwarder -IPAddress 8.8.8.8,1.1.1.1 -UseRootHint $true` | Reenvía, y si no hay respuesta resuelve él mismo | `forward first` |
| `Set-DnsServerForwarder -IPAddress 8.8.8.8 -UseRootHint $false` | Solo reenvía | `forward only` |
| `Get-DnsServerForwarder` | Muestra los reenviadores | — |

**Gestionar la caché:**

| Comando | Función | Equivalente BIND |
|---|---|---|
| `Show-DnsServerCache` | Muestra el contenido de la caché | `rndc dumpdb -cache` |
| `Clear-DnsServerCache` | Vacía la caché | `rndc flush` |
| `Get-DnsServerCache` / `Set-DnsServerCache -MaxTTL 1.00:00:00` | Ver / cambiar la configuración de la caché (aquí, TTL máximo de 1 día) | `max-cache-ttl` |

**Configurar los clientes:** igual que en el Capítulo 6: `Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.1.10`, o repartir el servidor DNS por DHCP (opción 6). En Windows no hay un `/etc/resolv.conf` que se regenere, así que el problema que menciona el libro no se da.

---

## 5. Registro de eventos (logging)

El servidor DNS de Windows tiene tres tipos de registro:

| Registro | Contenido | Cómo se usa |
|---|---|---|
| Registro de eventos **Servidor DNS** | Arranque, parada, errores al cargar zonas, problemas de transferencias (equivale a la categoría `general` / `default`) | Visor de eventos → Registros de aplicaciones y servicios → **DNS Server**, o `Get-WinEvent -LogName "DNS Server"` |
| Registro de **auditoría** (`Microsoft-Windows-DNSServer/Audit`) | Cambios de configuración, de zonas y de registros (equivale a `update`, `security`) | Activado por defecto |
| Registro **analítico** (`Microsoft-Windows-DNSServer/Analytical`) | Cada consulta y respuesta, con poco impacto en el rendimiento (equivale a `queries`) | Desactivado por defecto; se activa en el Visor de eventos (Ver → Mostrar registros analíticos y de depuración) |
| **Registro de depuración** (archivo de texto) | Paquetes y consultas con mucho detalle | `Set-DnsServerDiagnostics` (ver abajo). Archivo por defecto: `C:\Windows\System32\dns\dns.log` |

**Registro de depuración** (equivalente a crear canales y categorías en la sección `logging`):
```
Set-DnsServerDiagnostics -Queries $true -Answers $true -Update $true -Notifications $true -EnableLoggingToFile $true -LogFilePath "C:\DNSLogs\dns.log" -MaxMBFileSize 500000000
```

| Categoría de BIND | Parámetro de `Set-DnsServerDiagnostics` |
|---|---|
| `queries` | `-Queries $true` (y `-Answers $true` para las respuestas) |
| `update` | `-Update $true` |
| `notify` | `-Notifications $true` |
| `network` | `-SendPackets`, `-ReceivePackets`, `-UdpPackets`, `-TcpPackets` |
| Filtrar por cliente | `-FilterIPAddressList 192.168.1.50` |
| Todo | `-All $true` (útil para depurar, pero genera muchísimo registro) |

Para desactivarlo: `Set-DnsServerDiagnostics -All $false`. Los cambios se aplican sin reiniciar el servicio.

---

## 6. Zonas DNS

### 6.1 Tipos de zona

Windows tiene los mismos tipos básicos y además la opción de guardar las zonas en Active Directory:

| Tipo en BIND | Equivalente en Windows Server 2025 | Comando |
|---|---|---|
| `master` (primaria) | **Zona principal** en archivo | `Add-DnsServerPrimaryZone -Name empresa.com -ZoneFile empresa.com.dns` |
| — | **Zona principal integrada en Active Directory**: se guarda en AD y todos los controladores de dominio con DNS la tienen como primaria (varios primarios a la vez) | `Add-DnsServerPrimaryZone -Name empresa.com -ReplicationScope Domain` (o `Forest`) |
| `slave` (secundaria) | **Zona secundaria** | `Add-DnsServerSecondaryZone -Name empresa.com -ZoneFile empresa.com.dns -MasterServers 192.168.1.10` |
| `forward` | **Reenviador condicional** (reenvía solo las consultas de ese dominio) | `Add-DnsServerConditionalForwarderZone -Name forward.example.com -MasterServers 192.168.64.106,192.168.64.107` |
| `hint` (zona raíz) | **Sugerencias de raíz**, guardadas en `C:\Windows\System32\dns\cache.dns` (equivale a `named.ca`) | `Get-DnsServerRootHint`, `Add-DnsServerRootHint`, `Import-DnsServerRootHint -NameServer otro-servidor` |
| `stub` | **Zona de código auxiliar** (solo NS, SOA y registros de los servidores de nombres) | `Add-DnsServerStubZone -Name sucursal.com -MasterServers 192.168.70.10 -ZoneFile sucursal.com.dns` |
| `redirect`, `static-stub`, `delegation-only` | No existen | — |
| Clase `IN` | Siempre `IN` | — |

Ver las zonas: `Get-DnsServerZone`. Borrar una zona: `Remove-DnsServerZone -Name empresa.com`.

**Control de actualizaciones, transferencias y notificaciones** (equivalente a `allow-update`, `allow-transfer` y `also-notify`):

| Directiva BIND | Equivalente en Windows Server 2025 |
|---|---|
| `allow-update { none; };` | `Set-DnsServerPrimaryZone -Name empresa.com -DynamicUpdate None` |
| Actualizaciones permitidas | `-DynamicUpdate NonsecureAndSecure` (cualquiera) o **`Secure`** (solo equipos autenticados del dominio; solo en zonas integradas en AD, y es la opción recomendada) |
| `allow-transfer { none; };` | `Set-DnsServerPrimaryZone -Name empresa.com -SecureSecondaries NoTransfer` |
| `allow-transfer { IP; };` | `Set-DnsServerPrimaryZone -Name empresa.com -SecureSecondaries TransferToSecureServers -SecondaryServers 192.168.1.11` |
| Transferir a los servidores NS de la zona | `-SecureSecondaries TransferToZoneNameServer` |
| `also-notify` / `notify` | `-Notify NotifyServers -NotifyServers 192.168.1.11` (o `Notify` para avisar a los servidores NS, o `NoNotify`) |

Forzar una transferencia en un secundario (equivalente a `rndc retransfer`): `Start-DnsServerZoneTransfer -Name empresa.com -FullTransfer`.

### 6.2 Ubicación de las bases de datos de zona

| Tipo de zona | Ubicación |
|---|---|
| Zonas en archivo | `C:\Windows\System32\dns\` (un archivo por zona, ej. `empresa.com.dns`) |
| Copias de seguridad de las zonas | `C:\Windows\System32\dns\backup\` |
| Zonas integradas en AD | Dentro de la base de datos de Active Directory, en las particiones de aplicación **DomainDnsZones** (se replica a los DC del dominio) o **ForestDnsZones** (a los de todo el bosque) |

### 6.3 Estructura de una base de datos de zona

Las **zonas en archivo** usan el **mismo formato estándar que BIND**: registro SOA, registros de recursos, TTL, `@`, nombres con punto final... Windows puede incluso **cargar un archivo de zona de BIND**: se copia en `C:\Windows\System32\dns\` y se crea la zona indicando que ya existe:
```
Add-DnsServerPrimaryZone -Name example.com -ZoneFile example.com.zone -LoadExisting
```

Aun así, lo normal en Windows es **no editar los archivos a mano**, sino crear los registros con cmdlets o con la consola:

| Registro (Tabla 8.2) | Comando en Windows Server 2025 |
|---|---|
| `A` | `Add-DnsServerResourceRecordA -ZoneName empresa.com -Name LPIC2 -IPv4Address 192.168.64.120 -CreatePtr` (`-CreatePtr` crea también el PTR en la zona inversa) |
| `AAAA` | `Add-DnsServerResourceRecordAAAA -ZoneName empresa.com -Name LPIC2 -IPv6Address 2001:db8::120` |
| `CNAME` | `Add-DnsServerResourceRecordCName -ZoneName empresa.com -Name www -HostNameAlias LPIC2.empresa.com` |
| `MX` | `Add-DnsServerResourceRecordMX -ZoneName empresa.com -Name "." -MailExchange maila.empresa.com -Preference 10` (`"."` = la propia zona, el `@` de BIND) |
| `NS` | `Add-DnsServerResourceRecord -ZoneName empresa.com -Name "." -NS -NameServer serv2.empresa.com` |
| `PTR` | `Add-DnsServerResourceRecordPtr -ZoneName 64.168.192.in-addr.arpa -Name 120 -PtrDomainName LPIC2.empresa.com` |
| `TXT` | `Add-DnsServerResourceRecord -ZoneName empresa.com -Name "." -Txt -DescriptiveText "v=spf1 mx -all"` |
| `SRV` (muy usado por Active Directory) | `Add-DnsServerResourceRecord -ZoneName empresa.com -Name _sip._tcp -Srv -DomainName sip.empresa.com -Priority 0 -Weight 0 -Port 5060` |
| `SOA` | Se crea solo al crear la zona. Ver: `Get-DnsServerResourceRecord -ZoneName empresa.com -RRType SOA` |

| Comando | Función |
|---|---|
| `Get-DnsServerResourceRecord -ZoneName empresa.com` | Lista todos los registros de la zona |
| `Get-DnsServerResourceRecord -ZoneName empresa.com -Name www` | Registros de un nombre |
| `Remove-DnsServerResourceRecord -ZoneName empresa.com -RRType A -Name LPIC2 -Force` | Borra un registro |

**Diferencias con el SOA de BIND:**
- **El número de serie lo incrementa Windows solo** con cada cambio: no hay que acordarse de subirlo a mano.
- El TTL por defecto de los registros es de **1 hora** (el "TTL mínimo" del SOA).
- Los campos (refresh, retry, expire, minimum) son los mismos y se cambian en Administrador de DNS → Propiedades de la zona → pestaña **Inicio de autoridad (SOA)**.

**Zona inversa:**
```
Add-DnsServerPrimaryZone -NetworkId "192.168.64.0/24" -ReplicationScope Domain
```
Windows crea el nombre correcto (`64.168.192.in-addr.arpa`) a partir de la red.

**Envejecimiento y borrado** (propio de Windows, sin equivalente en el libro): en redes con actualizaciones dinámicas, los equipos registran solos su nombre. Para que se borren los registros de equipos que ya no existen:
```
Set-DnsServerZoneAging -Name empresa.com -Aging $true
Set-DnsServerScavenging -ScavengingState $true -ScavengingInterval 7.00:00:00
```

### 6.4 Comprobar una zona

No existe un equivalente directo de `named-checkzone`. Opciones:

| Comando | Función |
|---|---|
| `Get-DnsServerZone -Name empresa.com \| Format-List *` | Estado y configuración de la zona |
| `Test-DnsServer -IPAddress 127.0.0.1 -ZoneName empresa.com` | Comprueba que el servidor responde con autoridad para la zona |
| Registro de eventos **DNS Server** | Si un archivo de zona tiene errores, al cargarlo aparecen avisos indicando la línea |
| `Export-DnsServerZone -Name empresa.com -FileName empresa.com.copia` | Exporta la zona a un archivo de texto (en `C:\Windows\System32\dns\`) para revisarla o guardarla |

### 6.5 Delegación de zona

El cmdlet crea de una vez el registro **NS** y el registro **glue** (A) que describe el libro:
```
Add-DnsServerZoneDelegation -Name empresa.com -ChildZoneName sucursal -NameServer ns1.sucursal.empresa.com -IPAddress 192.168.70.10
```
Consultar las delegaciones: `Get-DnsServerZoneDelegation -Name empresa.com`.

---

## 7. Herramientas de diagnóstico

| Linux | Windows Server 2025 | Función |
|---|---|---|
| `host nombre` | `nslookup nombre` | Consulta sencilla |
| `host -t MX dominio` | `nslookup -type=mx dominio` | Registros de un tipo |
| `dig nombre` | `Resolve-DnsName nombre` | Consulta detallada (tipo, TTL, sección) |
| `dig @servidor nombre` | `Resolve-DnsName nombre -Server 192.168.1.10` o `nslookup nombre 192.168.1.10` | Consulta a un servidor concreto |
| `dig MX +short dominio` | `(Resolve-DnsName dominio -Type MX).NameExchange` | Solo los nombres de los servidores de correo |
| `dig +trace nombre` | No hay equivalente directo. Se puede seguir la cadena a mano: `Resolve-DnsName com -Type NS -Server a.root-servers.net`, luego el dominio contra uno de esos servidores, etc. | Recorrido desde la raíz |
| `nslookup -query=ns dominio` | `nslookup -type=ns dominio` o `Resolve-DnsName dominio -Type NS` | Servidores autoritativos |
| `nslookup` interactivo | Igual: `nslookup` → `server 192.168.1.10` → `set type=mx` → `set debug` → `exit` | — |
| — | `nslookup` → `ls -d empresa.com` | Lista la zona completa (solo si el servidor permite la transferencia de zona a este equipo) |

**`dnscmd`**: herramienta clásica de texto para el servidor DNS (se instala con las herramientas de administración del rol). Sigue funcionando (`dnscmd /info`, `dnscmd /enumzones`, `dnscmd /zoneprint empresa.com`, `dnscmd /clearcache`), pero Microsoft recomienda usar los cmdlets de PowerShell, porque `dnscmd` podría desaparecer en versiones futuras.

### 7.1 El equivalente de `rndc`

El papel de `rndc` lo cumplen los cmdlets del módulo **DnsServer** (que, además, funcionan en remoto con `-ComputerName`):

| Comando `rndc` | Equivalente en Windows Server 2025 |
|---|---|
| `rndc status` | `Get-DnsServer` (configuración completa), `Get-DnsServerStatistics` (contadores de consultas, errores...), `Get-DnsServerZone` (zonas cargadas) |
| `rndc reload` | No suele hacer falta. Si se ha editado a mano un archivo de zona: `dnscmd /zonereload empresa.com` |
| `rndc reload zona` | `dnscmd /zonereload empresa.com` (principal) o `Start-DnsServerZoneTransfer -Name empresa.com` (secundaria) |
| `rndc reconfig` | No hace falta: los cambios se aplican al momento |
| `rndc stop` (guardando cambios) | `Sync-DnsServerZone` (escribe en los archivos los cambios pendientes) + `Stop-Service DNS` |
| `rndc halt` | `Stop-Service DNS` |
| `rndc flush` | `Clear-DnsServerCache` |
| `rndc querylog on\|off` | `Set-DnsServerDiagnostics -Queries $true` / `$false`, o activar el registro analítico |

Diferencia con `rndc`: desde PowerShell **sí se puede arrancar y reiniciar** el servidor (`Start-Service DNS`, `Restart-Service DNS`).

---

## 8. Seguridad del servidor DNS

Buenas prácticas del libro y su equivalente:

| Práctica del libro | Equivalente en Windows Server 2025 |
|---|---|
| Mantener BIND actualizado | Windows Update (el servidor DNS forma parte del sistema) |
| Ejecutar como usuario no root | No se puede: el servicio usa la cuenta **LocalSystem**. Por eso es aún más importante el resto de medidas y, en servidores DNS públicos, usar **Server Core** y no unirlos al dominio |
| Ocultar la versión | Por defecto el servidor DNS de Windows no responde a las consultas de versión (ajuste `EnableVersionQuery = 0`) |
| No mezclar con otros servicios | Igual. Excepción habitual y aceptada: los **controladores de dominio** llevan DNS integrado |
| Vistas (`view`) para clientes internos y externos | **Ámbitos de zona** + **directivas DNS** (*split-brain*, ver ejemplo abajo) |
| `allow-update { none; };` | `-DynamicUpdate None`, o `Secure` en zonas integradas en AD |
| `allow-transfer { none; };` global | Las zonas integradas en AD **no permiten transferencias por defecto**; en las demás: `-SecureSecondaries NoTransfer` |
| Separar funciones DNS | Igual: por ejemplo, un servidor autoritativo público y otro de caché interno |
| DNSSEC | Admitido (apartado 8.2) |
| TSIG | **No admitido** (ver 8.2) |
| DANE | Sin soporte específico |

**Otras medidas de seguridad propias del servidor DNS de Windows:**

| Medida | Comando | Función |
|---|---|---|
| Bloqueo de caché | `Set-DnsServerCache -LockingPercent 100` | Impide sobrescribir una respuesta de la caché hasta que caduque su TTL (protege contra el envenenamiento de caché) |
| Grupo de sockets | `Set-DnsServerSetting -SocketPoolSize 10000` | Usa puertos de origen aleatorios para las consultas (más difícil de falsificar) |
| Limitación de respuestas | `Set-DnsServerResponseRateLimiting -Mode Enable` | Limita las respuestas repetidas (frena ataques de amplificación) |
| Lista global de bloqueo | `Get-DnsServerGlobalQueryBlockList` | Nombres que el servidor nunca resuelve (por defecto `wpad` e `isatap`) |
| Directivas de bloqueo | `Add-DnsServerQueryResolutionPolicy -Name "Bloqueo" -Action IGNORE -FQDN "eq,*.malicioso.com"` | Ignora las consultas de dominios concretos |

**Ejemplo de *split-brain*** (equivalente a las vistas de BIND): el mismo nombre responde con una IP interna a los clientes de la LAN y con la pública al resto:
```
Add-DnsServerClientSubnet -Name "LAN" -IPv4Subnet 192.168.0.0/16
Add-DnsServerZoneScope -ZoneName empresa.com -Name "Interno"
Add-DnsServerResourceRecordA -ZoneName empresa.com -ZoneScope "Interno" -Name www -IPv4Address 192.168.64.120
Add-DnsServerQueryResolutionPolicy -Name "VistaInterna" -Action ALLOW -ClientSubnet "eq,LAN" -ZoneScope "Interno,1" -ZoneName empresa.com
```
El registro `www` normal (ámbito por defecto) contiene la IP pública y lo reciben todos los demás clientes.

### 8.1 Chroot jail

**No existe en Windows.** El servicio DNS no se puede encerrar en un directorio. Las medidas equivalentes para limitar los daños si el servidor es atacado son:
- Usar **Server Core** (menos componentes y menos superficie de ataque).
- Dedicar un servidor solo a DNS (sin otros roles), y no unirlo al dominio si es un DNS público.
- Aplicar las medidas de la tabla anterior (bloqueo de caché, limitación de respuestas, directivas).
- Ejecutarlo en una **máquina virtual** aislada, que es lo más parecido en Windows a aislar un servicio en un chroot o una jail.

### 8.2 DNSSEC y TSIG

**DNSSEC:** el servidor DNS de Windows firma zonas y valida respuestas. Los conceptos (ZSK, KSK, cadena de confianza, DNSKEY, DS) son los mismos. Detalle propio de Windows: en zonas integradas en AD, un servidor hace de **maestro de claves** (*Key Master*): genera y renueva las claves de la zona, y el resto de controladores de dominio reciben la zona ya firmada. La renovación automática de claves está incluida.

| Herramienta de BIND | Equivalente en Windows Server 2025 |
|---|---|
| `dnssec-keygen` (claves DNSSEC) | `Add-DnsServerSigningKey -ZoneName empresa.com -Type KeySigningKey -CryptoAlgorithm RsaSha256` (y `-Type ZoneSigningKey` para la ZSK) |
| `dnssec-signzone` | `Invoke-DnsServerZoneSign -ZoneName empresa.com -SignWithDefault` (genera claves con valores por defecto y firma la zona en un solo paso) |
| Quitar la firma | `Invoke-DnsServerZoneUnsign -ZoneName empresa.com` |
| Ver las claves | `Get-DnsServerSigningKey -ZoneName empresa.com` |
| Ver los registros DNSKEY | `Get-DnsServerResourceRecord -ZoneName empresa.com -RRType DNSKEY` |
| `dnssec-dsfromkey` (registro para la zona padre) | Windows genera el archivo `dsset-empresa.com` en `C:\Windows\System32\dns\` con los registros DS que hay que entregar a la zona padre |
| Anclajes de confianza para validar | `Add-DnsServerTrustAnchor -Root` (zona raíz) o `Get-DnsServerTrustAnchor` para verlos |
| Configuración DNSSEC de la zona | `Get-DnsServerDnsSecZoneSetting -ZoneName empresa.com` |

**TSIG:** el servidor DNS de Windows **no admite TSIG** con claves compartidas como BIND. En su lugar:
- Para las **actualizaciones dinámicas**, usa **GSS-TSIG**: autenticación con Kerberos de Active Directory (la opción `-DynamicUpdate Secure`).
- Para las **transferencias de zona**, se restringen por dirección IP (`-SecureSecondaries TransferToSecureServers`) o se usan **zonas integradas en AD**, que se replican de forma autenticada y cifrada dentro de Active Directory sin necesidad de transferencias.

### 8.3 Otros servidores DNS disponibles para Windows

| Programa | Situación |
|---|---|
| BIND | ISC dejó de publicar versiones para Windows (la última rama con soporte para Windows ya no recibe actualizaciones). No recomendable |
| Unbound | Tiene versión para Windows (servidor de caché) |
| Technitium DNS Server | Servidor DNS libre, con interfaz web, que funciona en Windows |

---

## 9. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 8) | Equivalente en Windows Server 2025 |
|---|---|
| Paquete `bind` / `bind9` | Rol **Servidor DNS** (`Install-WindowsFeature DNS`) |
| `bind-utils` / `dnsutils` | `nslookup` y `Resolve-DnsName` (integrados) |
| Servicio `named` / `bind9` | Servicio `DNS` (proceso `dns.exe`) |
| Usuario `named` / `bind` | LocalSystem |
| `named.conf` | Registro + módulo DnsServer (`Get-DnsServer`, `Set-DnsServerSetting`...) |
| `/var/named/` / `/etc/bind/` | `C:\Windows\System32\dns\` o Active Directory |
| `named.ca` (zona `hint`) | `cache.dns` / sugerencias de raíz |
| `listen-on` | `Set-DnsServerSetting -ListeningIPAddress` |
| `recursion` | `Set-DnsServerRecursion` |
| `forwarders`, `forward only/first` | `Set-DnsServerForwarder -UseRootHint $false / $true` |
| `acl` + `allow-query` | Subredes de cliente + directivas de resolución de consultas |
| `type master` / `slave` / `stub` | `Add-DnsServerPrimaryZone` / `SecondaryZone` / `StubZone` |
| `type forward` | `Add-DnsServerConditionalForwarderZone` |
| — | Zonas integradas en Active Directory (`-ReplicationScope`) |
| Registros en el archivo de zona | `Add-DnsServerResourceRecord*` (o archivo `.dns` con el mismo formato) |
| Subir el serial del SOA | Automático |
| `allow-update` | `-DynamicUpdate None / Secure` |
| `allow-transfer` | `-SecureSecondaries` |
| `named-checkconf` / `named-checkzone` | `Test-DnsServer`, Analizador de procedimientos recomendados, registro de eventos |
| `logging` | Registro DNS Server, auditoría, analítico y `Set-DnsServerDiagnostics` |
| `dig` / `host` | `Resolve-DnsName` / `nslookup` |
| `rndc` | Cmdlets del módulo DnsServer (`Get-DnsServerStatistics`, `Clear-DnsServerCache`...) y `dnscmd` |
| Vistas (`view`) | Ámbitos de zona + directivas DNS |
| Chroot | No existe (Server Core, servidor dedicado, máquina virtual) |
| DNSSEC (`dnssec-keygen`, `dnssec-signzone`) | `Add-DnsServerSigningKey`, `Invoke-DnsServerZoneSign` |
| TSIG | No admitido: GSS-TSIG (Kerberos) y zonas integradas en AD |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 8 ("Directing DNS") del libro LPIC-2. Fuentes: documentación oficial de Microsoft Learn (Servidor DNS de Windows Server, módulo DnsServer de PowerShell, zonas integradas en Active Directory, directivas DNS y *split-brain*, registro de diagnóstico y analítico de DNS, DNSSEC en Windows Server, bloqueo de caché y grupo de sockets, limitación de respuestas).*
