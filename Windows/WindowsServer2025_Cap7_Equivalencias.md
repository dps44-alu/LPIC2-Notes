# LPIC-2 · Capítulo 7: Organizing Email Services
### Equivalencias en Windows Server 2025

Este documento recoge, para cada concepto visto en el Capítulo 7 del libro LPIC-2 (servidores de correo, entrega local, filtrado y acceso a buzones), cuál es su equivalente en **Windows Server 2025**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo: **Windows Server 2025 no incluye ningún servidor de correo**. Hasta Windows Server 2022 existía la característica "Servidor SMTP" (un MTA sencillo heredado de IIS 6), pero **en Windows Server 2025 se ha eliminado** y Microsoft no ofrece sustituto dentro del sistema: remite a Exchange Server o a un servidor SMTP de terceros.

Por eso, en Windows el correo se resuelve de una de estas formas:
- **Exchange Server Subscription Edition (Exchange SE)**: el servidor de correo de Microsoft para instalar en tus propios servidores. Es un producto aparte (de pago, con suscripción), compatible con Windows Server 2025. Desde octubre de 2025 es la única versión de Exchange local con soporte.
- **Exchange Online** (Microsoft 365): el mismo servicio, en la nube de Microsoft.
- **Servidores de correo de terceros** para Windows (MailEnable, SmarterMail, MDaemon...).

A diferencia de Linux, Exchange **reúne en un solo producto** el MTA, el MDA, el servidor POP3/IMAP y el acceso web (Outlook en la web). No se configura con archivos de texto, sino con la **Consola de administración de Exchange** (Exchange Management Shell, una consola de PowerShell con cmdlets propios) o con el **Centro de administración de Exchange** (EAC, web). Su configuración se guarda en **Active Directory**, que es obligatorio para instalarlo.

En este documento se usan los comandos de Exchange SE como equivalente principal. Se ejecutan en la **Exchange Management Shell**, como administrador de Exchange.

---

## 1. Arquitectura del correo en Windows Server

| Componente Linux | Equivalente en Exchange Server SE |
|---|---|
| **MTA** (sendmail, Postfix...) | **Servicios de transporte**: Transporte de front-end (recibe y envía por SMTP) y Transporte (colas, enrutamiento, resolución de destinatarios). Opcionalmente, un servidor **Transporte perimetral** (Edge Transport) en la zona desmilitarizada (DMZ) |
| **MDA** (procmail) | **Transporte de buzón** (Mailbox Transport Delivery): entrega los mensajes en la base de datos de buzones. El filtrado se hace con **reglas de transporte** y **reglas de bandeja de entrada** (apartado 5) |
| **MUA** | **Outlook** (escritorio), **Outlook en la web** (OWA, en el navegador), móviles con Exchange ActiveSync, y cualquier cliente POP3/IMAP (apartado 7) |

Exchange tiene dos **roles de servidor**:

| Rol | Función |
|---|---|
| **Buzón** (Mailbox) | Contiene todo: bases de datos de buzones, transporte, acceso de clientes (OWA, ActiveSync, POP3, IMAP) |
| **Transporte perimetral** (Edge Transport) | Opcional. Servidor SMTP expuesto a Internet, fuera del dominio, que filtra el correo antes de pasarlo al servidor de buzones |

### 1.1 Tipos de buzón de usuario

| Tipo Linux | Equivalente en Windows / Exchange |
|---|---|
| `mbox` (un archivo por usuario) | No se usa. Lo más parecido es el archivo **PST** (Outlook), que guarda los mensajes de un usuario en un solo archivo, pero en el equipo del cliente, no en el servidor |
| `maildir` (un archivo por mensaje) | No se usa |
| Buzones del servidor | **Base de datos de buzones** (archivo `.edb`): **muchos buzones en una sola base de datos**, con sus **registros de transacciones** (archivos `.log`) aparte, como en un gestor de bases de datos |

| Comando | Función |
|---|---|
| `Get-MailboxDatabase \| Select Name, Server, EdbFilePath, LogFolderPath` | Bases de datos de buzones y dónde están sus archivos |
| `New-MailboxDatabase -Name DB02 -Server EX01 -EdbFilePath E:\DB02\DB02.edb -LogFolderPath F:\Logs\DB02` | Crea una base de datos nueva |
| `Mount-Database DB02` | La pone en servicio |

### 1.2 Elegir el software de correo (equivalente a elegir el MTA)

| Opción | Cuándo usarla |
|---|---|
| **Exchange Server SE** | Correo completo de empresa en servidores propios (necesita Active Directory y licencia con suscripción) |
| **Exchange Online** (Microsoft 365) | Correo de empresa sin mantener servidores de correo propios |
| Servidores de terceros para Windows | MailEnable, SmarterMail, MDaemon... Más sencillos que Exchange; se configuran cada uno a su manera |
| Postfix / Dovecot | No existen para Windows. Si se necesitan, lo razonable es una **máquina virtual Linux** en Hyper-V (y aplicar entonces el Capítulo 7 del libro tal cual) |

---

## 2. SMTP

| Concepto Linux | Situación en Windows Server 2025 / Exchange |
|---|---|
| Puerto TCP 25 | Igual. Exchange escucha en el 25 (correo entre servidores) y en el **587** (envío desde clientes autenticados) |
| `telnet localhost 25` | Funciona igual, pero el **cliente Telnet no viene instalado**: `Install-WindowsFeature Telnet-Client`. Para comprobar solo si el puerto responde: `Test-NetConnection mail.empresa.com -Port 25` |
| Prueba con cifrado | Windows no incluye `openssl s_client`, pero sí **`curl.exe`**, que habla SMTP, POP3 e IMAP: `curl -v smtp://mail.empresa.com:587 --ssl-reqd` |
| `STARTTLS` | Admitido. Exchange usa el certificado asignado al servicio SMTP: `Enable-ExchangeCertificate -Thumbprint HUELLA -Services SMTP` |
| `ETRN` | Función muy antigua que las versiones actuales de Exchange no admiten |

Los códigos de respuesta de 3 cifras (`220`, `250`, `550`...) son los mismos, porque SMTP es un estándar.

---

## 3. Sendmail

### 3.0 Enviar correo desde un servidor Windows (sin equivalente en el libro)

Muchos servidores no necesitan un servidor de correo completo: solo tienen que **enviar avisos** (alertas, informes, tareas programadas). En Linux eso lo hace el MTA local. En Windows Server 2025, al no haber MTA, el correo se envía **directamente a un servidor de correo que haga de relay** (Exchange, Microsoft 365 o un servidor SMTP de terceros):

```
Send-MailMessage -SmtpServer mail.empresa.com -Port 587 -UseSsl -Credential (Get-Credential) -From alertas@empresa.com -To admin@empresa.com -Subject "Aviso de SRV01" -Body "Disco casi lleno"
```

`Send-MailMessage` sigue funcionando en Windows PowerShell 5.1, pero Microsoft lo considera **obsoleto** (no garantiza conexiones seguras con servidores modernos). Para usos nuevos se recomiendan librerías externas o la API del servicio de correo que se use. La antigua acción "Enviar un correo electrónico" del Programador de tareas también está en desuso.

En Exchange, para que servidores y aplicaciones internas puedan enviar correo **sin autenticarse**, se crea un **conector de recepción de relay** que solo acepte sus IP (apartado 3.1).

### 3.1 Componentes de Sendmail (equivalente a la Tabla 7.5)

| Archivo / programa de Sendmail | Equivalente en Exchange Server SE |
|---|---|
| `sendmail` (ejecutable) | Servicios de transporte: `MSExchangeFrontEndTransport` y `MSExchangeTransport` |
| `sendmail.cf` | No hay archivo: configuración en Active Directory, gestionada con cmdlets (`Get-/Set-TransportConfig`, conectores, dominios aceptados) |
| `sendmail.cw` (dominios locales) | **Dominios aceptados** de tipo autoritativo: `New-AcceptedDomain -Name empresa.com -DomainName empresa.com -DomainType Authoritative`. Consultar: `Get-AcceptedDomain` |
| `sendmail.ct` (usuarios de confianza) | Permisos de los **conectores de recepción** (`-PermissionGroups`) y roles de administración de Exchange |
| `aliases` | **Direcciones adicionales** de un buzón: `Set-Mailbox juan -EmailAddresses @{add="jlopez@empresa.com"}`. Para enviar a varios usuarios: **grupos de distribución** (`New-DistributionGroup`) |
| `newaliases` | No hace falta: los cambios se guardan en Active Directory y se aplican solos |
| `mailq` | `Get-Queue` y `Get-Message` (apartado 4.4) |
| `mqueue` (directorio de la cola) | Base de datos de colas: `%ExchangeInstallPath%TransportRoles\data\Queue\mail.que` |
| `mailertable` (enrutado por dominio) | **Conectores de envío** con un espacio de direcciones por dominio: `New-SendConnector -Name "Socio" -AddressSpaces "socio.com" -SmartHosts mx.socio.com -DNSRoutingEnabled $false` |
| `domaintable` (dominios antiguos a nuevos) | **Directivas de direcciones de correo** (`New-EmailAddressPolicy`), que asignan las direcciones de todos los usuarios según una plantilla |
| `virtusertable` | Varios dominios aceptados + direcciones adicionales en cada buzón |
| `relaydomains` | Dominios aceptados de tipo **InternalRelay** o **ExternalRelay**, y conectores de recepción de relay |
| `access` | **Filtros contra correo no deseado**: listas de IP bloqueadas y permitidas, filtro de remitentes y de destinatarios (apartado 4.6) |

**Conector de recepción de relay** (equivalente a autorizar un equipo en `access` o `relaydomains`):
```
New-ReceiveConnector -Name "Relay aplicaciones" -Server EX01 -TransportRole FrontendTransport -Custom -Bindings 0.0.0.0:25 -RemoteIPRanges 192.168.1.50,192.168.1.51
Set-ReceiveConnector "EX01\Relay aplicaciones" -PermissionGroups AnonymousUsers
Get-ReceiveConnector "EX01\Relay aplicaciones" | Add-ADPermission -User "NT AUTHORITY\ANONYMOUS LOGON" -ExtendedRights "Ms-Exch-SMTP-Accept-Any-Recipient"
```

### 3.2 El fichero `sendmail.cf` (Tabla 7.6)

No hay equivalente: Exchange no tiene un archivo de configuración con reglas de reescritura. Esa lógica está integrada en los servicios de transporte y se ajusta con los cmdlets de este capítulo.

### 3.3 Generar `sendmail.cf` con `m4`

No aplica. Lo más parecido a "generar la configuración con plantillas" es escribir **scripts de PowerShell** con los cmdlets de Exchange y ejecutarlos en cada servidor.

### 3.4 Ejecutar Sendmail

| Linux | Equivalente en Windows Server 2025 / Exchange |
|---|---|
| `sendmail -bd` (demonio) | Los servicios de Exchange arrancan solos con el sistema (tipo de inicio Automático) |
| `-q5m` (revisar la cola cada 5 minutos) | Reintentos automáticos. El intervalo se cambia por servidor: `Set-TransportService EX01 -MessageRetryInterval 00:05:00` |
| Ver los servicios | `Get-Service MSExchange*` |
| Reiniciar el transporte | `Restart-Service MSExchangeTransport` |

---

## 4. Postfix

Postfix no existe para Windows. En este apartado se compara cada parte de Postfix con su equivalente en **Exchange SE**, que es la forma más completa de entender cómo funciona el transporte de correo en Windows.

### 4.1 Ficheros de configuración (equivalente a la Tabla 7.10)

| Archivo de Postfix | Equivalente en Exchange Server SE |
|---|---|
| `install.cf` | Parámetros del programa de instalación (`Setup.exe /Mode:Install /Roles:Mailbox ...`) |
| `main.cf` | Configuración global de transporte (`Get-TransportConfig` / `Set-TransportConfig`) y de cada servidor (`Get-TransportService` / `Set-TransportService`) |
| `master.cf` | Servicios de Windows de Exchange y **conectores de recepción** (qué escucha en cada puerto) |
| `kill -HUP` (recargar) | No hace falta en la mayoría de cambios: Exchange lee de nuevo la configuración de Active Directory de forma periódica. Si un cambio no se aplica, `Restart-Service MSExchangeTransport` |

### 4.2 Parámetros de `main.cf` (Tabla 7.11)

| Parámetro Postfix | Equivalente en Exchange Server SE |
|---|---|
| `myhostname` | Nombre (FQDN) que anuncia cada conector: `Set-ReceiveConnector ... -Fqdn mail.empresa.com` y `Set-SendConnector ... -Fqdn mail.empresa.com` |
| `mydomain` | Dominio aceptado por defecto (`Get-AcceptedDomain`) |
| `myorigin` | **Directiva de direcciones de correo**: decide qué dirección y dominio tienen los usuarios (`Set-EmailAddressPolicy "Default Policy" -EnabledPrimarySMTPAddressTemplate "%g.%s@empresa.com"`) |
| `mydestination` | Dominios aceptados de tipo **Authoritative** |
| `relay_domains` | Dominios aceptados de tipo **InternalRelay** / **ExternalRelay** |
| `relayhost` | **Conector de envío** a Internet con host inteligente: `New-SendConnector -Name Internet -Usage Internet -AddressSpaces "*" -SmartHosts relay.proveedor.com -DNSRoutingEnabled $false` |
| `virtual_alias_domains` | Dominios aceptados + direcciones adicionales en los buzones |
| `message_size_limit` | `Set-TransportConfig -MaxSendSize 25MB -MaxReceiveSize 25MB` |

### 4.3 Programas núcleo de Postfix (Tabla 7.8)

| Programa Postfix | Equivalente en Exchange Server SE |
|---|---|
| `master` | Administrador de servicios de Windows (SCM) + servicio `MSExchangeServiceHost` |
| `smtpd` | Servicio **Transporte de front-end** (`MSExchangeFrontEndTransport`) y sus conectores de recepción |
| `pickup` | **Directorio de recogida** (`%ExchangeInstallPath%TransportRoles\Pickup`): un archivo `.eml` copiado ahí se envía solo. Y el servicio **Envío de transporte de buzón** (`MSExchangeSubmission`), que recoge lo que los usuarios envían desde su buzón |
| `cleanup` y `trivial-rewrite` | **Categorizador** del servicio Transporte: resuelve destinatarios, expande grupos y aplica reescrituras |
| `qmgr` | Colas del servicio **Transporte** (`MSExchangeTransport`) |
| `smtp` | **Conectores de envío** |
| `local` | **Entrega de transporte de buzón** (`MSExchangeDelivery`) |
| `bounce` | Generación de **informes de no entrega** (NDR) |
| `flush` | No aplica |
| `showq` | `Get-Queue` |

### 4.4 Utilidades de Postfix (Tabla 7.9)

| Utilidad Postfix | Equivalente en Exchange Server SE |
|---|---|
| `mailq` | `Get-Queue` (colas y número de mensajes) y `Get-Message -Queue "EX01\Submission"` (mensajes de una cola). Gráfico: **Visor de colas** (Exchange Toolbox) |
| `postfix start/stop/reload` | `Start-Service` / `Stop-Service` / `Restart-Service MSExchangeTransport` |
| `postalias` | `Set-Mailbox -EmailAddresses`, cmdlets de grupos (`New-DistributionGroup`, `Add-DistributionGroupMember`) |
| `postcat` | `Export-Message` (guarda en un archivo un mensaje de la cola, que antes debe estar suspendido) |
| `postconf` | `Get-TransportConfig`, `Get-TransportService`, `Get-ReceiveConnector`, `Get-SendConnector` (y sus `Set-`) |
| `postmap` | No hace falta: no hay tablas que compilar |
| `postsuper` | `Retry-Queue` (reintentar), `Suspend-Message` / `Resume-Message`, `Remove-Message` (borrar de la cola) |
| `postlog` | No aplica |

### 4.5 Comandos de emulación de sendmail

No existen: en Windows no hay un comando `sendmail` compatible. Los programas envían el correo por SMTP a un servidor (apartado 3.0), o dejando un archivo `.eml` en el **directorio de recogida** de Exchange.

### 4.6 Tablas de búsqueda (Tabla 7.12)

| Tabla Postfix | Equivalente en Exchange Server SE |
|---|---|
| `access` | **Agentes contra correo no deseado**: se instalan en el servidor de buzones con el script `Install-AntiSpamAgents.ps1` (carpeta `Scripts` de Exchange). Después: `Add-IPBlockListEntry -IPAddress 203.0.113.5` (bloquear IP), `Add-IPAllowListEntry` (permitir IP), `Set-SenderFilterConfig -BlockedSenders spam@ejemplo.com`, `Set-RecipientFilterConfig` |
| `alias` | Direcciones adicionales del buzón y grupos de distribución |
| `canonical` | **Reescritura de direcciones** (solo en servidores de Transporte perimetral): `New-AddressRewriteEntry -Name "Ventas" -InternalAddress ventas.empresa.local -ExternalAddress empresa.com` |
| `relocated` | Sin equivalente directo. Se usa una regla de transporte que rechace con un mensaje explicativo, o un contacto de correo que reenvíe a la nueva dirección |
| `transport` | Conectores de envío con espacio de direcciones por dominio |
| `virtual` | Dominios aceptados + direcciones de los buzones |

Reenvío de un buzón a otra dirección (otro uso típico de los alias): `Set-Mailbox juan -ForwardingSmtpAddress juan@otro.com -DeliverToMailboxAndForward $true`.

### 4.7 Directorios y logs

| Linux | Equivalente en Exchange Server SE |
|---|---|
| `/var/spool/postfix` | `%ExchangeInstallPath%TransportRoles\data\Queue\` (base de datos de colas `mail.que`) |
| `/var/spool/mail` | Bases de datos de buzones (`.edb`) |
| `/var/log/maillog` | **Registros de seguimiento de mensajes** en `%ExchangeInstallPath%TransportRoles\Logs\MessageTracking\`. Se consultan con `Get-MessageTrackingLog -Sender juan@empresa.com -Start (Get-Date).AddHours(-2)` |
| Log detallado de SMTP | **Registros de protocolo** (desactivados por defecto en cada conector): `Set-ReceiveConnector "EX01\Default Frontend EX01" -ProtocolLoggingLevel Verbose`. Se guardan en `TransportRoles\Logs\FrontEnd\ProtocolLog\` |
| Eventos del sistema | Visor de eventos → registro **Aplicación** (orígenes `MSExchange...`) |

`%ExchangeInstallPath%` suele ser `C:\Program Files\Microsoft\Exchange Server\V15\`.

### 4.8 Otros MTA disponibles

| Programa | Situación en Windows Server 2025 |
|---|---|
| Servidor SMTP de IIS | **Eliminado** en Windows Server 2025 |
| Exim, qmail, sendmail, Postfix | No existen para Windows (solo en una máquina virtual Linux o, para pruebas, en WSL) |
| MailEnable, SmarterMail, MDaemon | Servidores de correo comerciales para Windows (algunos con versión gratuita limitada) |
| hMailServer | Servidor libre para Windows muy usado en el pasado, pero ya sin desarrollo activo; no recomendable para instalaciones nuevas |

---

## 5. Entrega local avanzada: procmail

Exchange no usa procmail. Su función se reparte entre dos tipos de reglas:

| Procmail | Equivalente en Exchange Server SE | Alcance |
|---|---|---|
| `/etc/procmailrc` (para todo el correo) | **Reglas de transporte** (también llamadas reglas de flujo de correo): `New-TransportRule` | Toda la organización. Se aplican al mensaje en tránsito |
| `$HOME/.procmailrc` (por usuario) | **Reglas de bandeja de entrada**: `New-InboxRule` (o las reglas que el usuario crea en Outlook / OWA) | Un buzón. Se aplican al entregar el mensaje |

### 5.1 Activar procmail

No hace falta activar nada: las reglas de transporte y de bandeja de entrada están siempre disponibles. Para consultarlas: `Get-TransportRule` y `Get-InboxRule -Mailbox juan`.

### 5.2 Formato de las recetas

Una regla de Exchange tiene la misma estructura que una receta: **condiciones** (qué mensajes), **acciones** (qué hacer) y **excepciones**. Se escribe como parámetros del cmdlet.

**Ejemplo del libro** (borrar los mensajes cuyo asunto contenga "work"):

| Procmail | Exchange (regla de transporte) |
|---|---|
| `:0` `* ^Subject.*work` `/dev/null` | `New-TransportRule -Name "Borrar work" -SubjectContainsWords "work" -DeleteMessage $true` |

**Equivalencias de flags, condiciones y acciones (Tablas 7.13, 7.14 y 7.15):**

| Procmail | Regla de transporte (`New-TransportRule`) | Regla de bandeja de entrada (`New-InboxRule`) |
|---|---|---|
| Flag `H` (mirar la cabecera) | `-HeaderContainsMessageHeader "X-Nombre" -HeaderContainsWords "texto"`, `-SubjectContainsWords`, `-From` | `-HeaderContainsWords`, `-SubjectContainsWords`, `-FromAddressContainsWords` |
| Flag `B` (mirar el cuerpo) | `-SubjectOrBodyContainsWords` | `-BodyContainsWords` |
| Flag `c` (copia) | `-BlindCopyTo usuario@empresa.com` | `-CopyToFolder` |
| Condición `!` (negar) | Parámetros `-ExceptIf...` (ej. `-ExceptIfFrom`) | Parámetros `-ExceptIf...` |
| Condición `<` / `>` (tamaño) | `-MessageSizeOver 10MB`, `-AttachmentSizeOver 5MB` | `-WithinSizeRangeMinimum` / `-WithinSizeRangeMaximum` |
| Condición `?` (código de salida de un programa) | Sin equivalente (solo con **agentes de transporte** programados en .NET) | Sin equivalente |
| Acción `!` (reenviar) | `-RedirectMessageTo` | `-ForwardTo` o `-RedirectTo` |
| Acción `\|` (ejecutar un programa) | Sin equivalente directo (agentes de transporte en .NET) | Sin equivalente |
| Acción ruta (guardar en una carpeta) | No (las reglas de transporte no conocen las carpetas de cada buzón) | `-MoveToFolder "juan:\Guitarras"` |
| `/dev/null` (borrar) | `-DeleteMessage $true` (borrado silencioso) | `-DeleteMessage $true` (va a Elementos eliminados) |
| Encadenar (`A`, `E`) y `{ }` | Orden con `-Priority` y `-StopRuleProcessing $true` | `-StopProcessingRules $true` |

Otras acciones útiles de las reglas de transporte, sin equivalente en procmail: añadir un aviso legal a todos los mensajes (`-ApplyHtmlDisclaimerText`), añadir una cabecera (`-SetHeaderName` / `-SetHeaderValue`) o rechazar con un mensaje al remitente (`-RejectMessageReasonText`).

---

## 6. Filtrado con Sieve

**Exchange no admite Sieve.** El papel de un script Sieve de usuario lo cumplen las **reglas de bandeja de entrada**.

| Comando Sieve | Equivalente en reglas de bandeja de entrada |
|---|---|
| `keep` | Comportamiento por defecto (el mensaje se queda en la Bandeja de entrada) |
| `fileinto "carpeta"` | `-MoveToFolder "usuario:\carpeta"` |
| `redirect` | `-RedirectTo` |
| `discard` | `-DeleteMessage $true` (o una regla de transporte con `-DeleteMessage` para un borrado silencioso) |
| `reject` | Solo con reglas de transporte: `-RejectMessageReasonText "texto"` |
| `stop` | `-StopProcessingRules $true` |
| `if` / `else` | Una regla con condición + otra regla con la condición contraria (`-ExceptIf...`) |
| `allof` (Y) | Varias condiciones en la misma regla (se tienen que cumplir todas) |
| `anyof` (O) | Varios valores en una misma condición (basta con uno), o varias reglas |
| `require` | No hace falta |

**Ejemplo del libro traducido** (mensajes de `guitar-list.org` a la carpeta "guitars"; el resto, a "saved"):
```
New-InboxRule -Mailbox juan -Name "Lista guitarras" -HeaderContainsWords "guitar-list.org" -MoveToFolder "juan:\guitars" -StopProcessingRules $true
New-InboxRule -Mailbox juan -Name "Resto" -ExceptIfHeaderContainsWords "guitar-list.org" -MoveToFolder "juan:\saved"
```

---

## 7. Acceso remoto a buzones: POP3 e IMAP

### 7.1 y 7.2 POP3 e IMAP

Los protocolos, puertos y comandos (Tabla 7.3, `LOGIN`, `a001 LOGOUT`...) son **estándar**, así que son los mismos. Lo propio de Exchange:
- El cliente habitual de Exchange no usa POP3 ni IMAP: **Outlook** usa MAPI sobre HTTP, y los móviles usan **Exchange ActiveSync**. Por eso **POP3 e IMAP vienen desactivados** por defecto.
- Cada protocolo tiene dos servicios: uno de front-end (atiende a los clientes) y uno de back-end.

| Protocolo | Puertos | Servicios de Windows |
|---|---|---|
| POP3 | 110 (texto / STARTTLS) y 995 (TLS) | `MSExchangePOP3` y `MSExchangePOP3BE` |
| IMAP | 143 (texto / STARTTLS) y 993 (TLS) | `MSExchangeIMAP4` y `MSExchangeIMAP4BE` |

**Activar IMAP** (POP3 es igual, cambiando los nombres):
```
Set-Service MSExchangeIMAP4 -StartupType Automatic
Set-Service MSExchangeIMAP4BE -StartupType Automatic
Start-Service MSExchangeIMAP4, MSExchangeIMAP4BE
```

Comprobación manual: `telnet mail.empresa.com 143` (con el cliente Telnet instalado, apartado 2) o, con TLS, `curl -v "imaps://mail.empresa.com/" -u juan`.

### 7.3 Servidor Courier

No existe para Windows. Sus ajustes (Tabla 7.16) tienen equivalente directo en la configuración de IMAP y POP3 de Exchange:

| Ajuste de Courier | Equivalente en Exchange Server SE |
|---|---|
| `ADDRESS` y `PORT` | `Set-ImapSettings -SSLBindings 0.0.0.0:993 -UnencryptedOrTLSBindings 0.0.0.0:143` |
| `MAILDIRPATH` | Ubicación de la base de datos de buzones (`-EdbFilePath` en `New-MailboxDatabase`) |
| `MAXDAEMONS` | `Set-ImapSettings -MaxConnections 2400` |
| `MAXPERIP` | `Set-ImapSettings -MaxConnectionFromSingleIP 100` |
| — | `Set-ImapSettings -MaxConnectionsPerUser 16` (conexiones por usuario) |
| `authdaemonrc` (autenticación) | Las cuentas son las de **Active Directory**. El tipo de inicio de sesión se elige con `Set-ImapSettings -LoginType SecureLogin` (solo con cifrado) o `PlainTextLogin` |

Para POP3, los mismos parámetros con `Set-PopSettings`. Tras cambiar la configuración hay que reiniciar los servicios del protocolo.

### 7.4 Servidor Dovecot

Dovecot tampoco existe para Windows. Equivalencias:

| Elemento de Dovecot | Equivalente en Exchange Server SE |
|---|---|
| `/etc/dovecot.conf` / `/etc/dovecot/` | `Get-ImapSettings` / `Get-PopSettings` y la configuración en Active Directory |
| `mail_location` | Base de datos de buzones (`Get-MailboxDatabase`) |
| `listen` | `-SSLBindings` / `-UnencryptedOrTLSBindings` |
| `mechanisms` | `-LoginType` |
| Certificado TLS | `Set-ImapSettings -X509CertificateName mail.empresa.com` y `Enable-ExchangeCertificate -Services IMAP,POP` |
| Activar o no el protocolo por usuario | `Set-CASMailbox juan -ImapEnabled $true -PopEnabled $false` |

**El equivalente de `doveadm`** son los cmdlets de buzón de la Exchange Management Shell:

| Tarea con `doveadm` | Equivalente en Exchange Server SE |
|---|---|
| Crear buzones | `Enable-Mailbox juan` (usuario de AD que ya existe) o `New-Mailbox` (crea usuario y buzón) |
| Estadísticas del buzón | `Get-MailboxStatistics juan \| Select DisplayName, ItemCount, TotalItemSize` |
| Cuotas | `Set-Mailbox juan -IssueWarningQuota 9GB -ProhibitSendQuota 10GB -ProhibitSendReceiveQuota 11GB` |
| Exportar un buzón | `New-MailboxExportRequest -Mailbox juan -FilePath \\SRV01\PST\juan.pst` (a un archivo PST; requiere el rol "Mailbox Import Export") |
| Mover buzones entre bases de datos | `New-MoveRequest -Identity juan -TargetDatabase DB02` |
| Buscar en buzones | Búsquedas de cumplimiento (`New-ComplianceSearch`) |

---

## 8. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 7) | Equivalente en Windows Server 2025 |
|---|---|
| MTA incluido en el sistema | **Ninguno**: el Servidor SMTP de IIS se eliminó en Windows Server 2025 |
| sendmail / Postfix | Exchange Server SE (servicios de transporte) o servidores de terceros |
| MDA (procmail) | Transporte de buzón + reglas de transporte y de bandeja de entrada |
| MUA | Outlook, Outlook en la web (OWA), clientes POP3/IMAP |
| `mbox` / `maildir` | Bases de datos de buzones `.edb` (servidor); archivos PST (cliente) |
| Archivos de configuración de texto | Cmdlets de Exchange + configuración en Active Directory |
| Enviar avisos desde el servidor | Relay a Exchange / Microsoft 365 / otro servidor (`Send-MailMessage`, obsoleto) |
| `mydestination` / `sendmail.cw` | Dominios aceptados autoritativos (`New-AcceptedDomain`) |
| `relay_domains` / `relaydomains` | Dominios aceptados de relay + conectores de recepción de relay |
| `relayhost` / `mailertable` / `transport` | Conectores de envío (`New-SendConnector -SmartHosts`) |
| `myorigin` / `domaintable` | Directivas de direcciones de correo |
| `aliases` / `newaliases` | Direcciones adicionales del buzón y grupos de distribución (sin regenerar nada) |
| `access` | Agentes contra correo no deseado (`Add-IPBlockListEntry`, `Set-SenderFilterConfig`) |
| `canonical` | Reescritura de direcciones (Transporte perimetral) |
| `mailq` / `postsuper` / `postcat` | `Get-Queue`, `Get-Message` / `Retry-Queue`, `Remove-Message` / `Export-Message` |
| `postconf` / `main.cf` | `Get-/Set-TransportConfig`, `Get-/Set-TransportService` |
| `postfix reload` | Automático o `Restart-Service MSExchangeTransport` |
| `/var/spool/postfix` | `TransportRoles\data\Queue\mail.que` |
| `/var/log/maillog` | Seguimiento de mensajes (`Get-MessageTrackingLog`) y registros de protocolo |
| `/etc/procmailrc` | Reglas de transporte (`New-TransportRule`) |
| `~/.procmailrc` / Sieve | Reglas de bandeja de entrada (`New-InboxRule`) |
| Courier / Dovecot | Servicios POP3 e IMAP de Exchange (`Set-ImapSettings`, `Set-PopSettings`) |
| `doveadm` | `Get-MailboxStatistics`, `Set-Mailbox`, `New-MailboxExportRequest`, `New-MoveRequest` |
| `telnet host 25/110/143` | Igual (instalando el cliente Telnet), `Test-NetConnection -Port`, `curl.exe` |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 7 ("Organizing Email Services") del libro LPIC-2. Fuentes: lista oficial de Microsoft de características quitadas en Windows Server 2025 (Servidor SMTP), anuncios oficiales de Exchange Server Subscription Edition y matriz de compatibilidad de Exchange Server, y documentación de Microsoft Learn sobre Exchange Server (transporte, conectores, dominios aceptados, colas, seguimiento de mensajes, reglas de transporte, reglas de bandeja de entrada, POP3 e IMAP4).*
