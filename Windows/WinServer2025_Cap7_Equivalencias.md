# LPIC-2 · Capítulo 7: Organizing Email Services
### Equivalencias en Windows Server 2025

Aviso para este capítulo: Windows Server no incluye, de serie, un servidor de correo completo (a diferencia de DNS, DHCP o archivos, que sí son roles nativos). El producto de Microsoft para esta función es **Exchange Server** (la versión más reciente para entornos on-premises es Exchange Server 2019/SE, instalable sobre Windows Server 2025 según la propia guía de referencia de este documento). Por eso este capítulo usa Exchange Server como equivalente principal, y menciona alternativas más ligeras donde aplica.

---

## 1. Arquitectura del correo

| Concepto Linux | Equivalente en Windows Server 2025 (Exchange) |
|---|---|
| MTA (sendmail/Postfix) | **Servicio de transporte de Exchange** (Transport service), que gestiona el envío/recepción y el enrutado de mensajes |
| MDA (procmail) | **Servicio de transporte de buzón** (Mailbox Transport service) + reglas de transporte, que entregan el correo en la base de datos de buzones |
| MUA (Thunderbird, Evolution) | **Outlook**, **Outlook Web App (OWA)**, o cualquier cliente POP3/IMAP4 estándar apuntando al servidor Exchange |
| Tipos de buzón (mbox, maildir) | **Base de datos de buzones** (Mailbox Database, `.edb`), un almacén estructurado tipo base de datos (motor ESE/JET) en vez de ficheros de texto planos por usuario |

A diferencia del modelo Unix (programas independientes que colaboran), Exchange Server integra MTA, MDA y el almacén de buzones en un único producto con varios servicios internos, todos administrables desde la misma consola y con los mismos cmdlets de PowerShell.

---

## 2. SMTP

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| Puerto TCP 25 | Igual: TCP 25 para SMTP |
| `telnet localhost 25` (comprobación manual) | Funciona igual en Windows si el cliente Telnet está instalado (`Install-WindowsFeature Telnet-Client` o vía "Características opcionales") |
| Comandos ESMTP (`STARTTLS`, `ETRN`) | Mismos comandos del protocolo, ya que Exchange implementa el estándar SMTP/ESMTP |

---

## 3. Sendmail / Postfix → Exchange Server (Transport y conectores)

| Concepto Linux | Equivalente en Exchange Server |
|---|---|
| `sendmail.cf` / `main.cf` (configuración del MTA) | No hay un fichero de texto único: la configuración vive en **Active Directory** y se gestiona con cmdlets de PowerShell (módulo de administración de Exchange) o desde el **Exchange Admin Center (EAC)**, la consola web de administración |
| `mailertable`, `virtusertable`, `access` (tablas de enrutado/control) | **Conectores de envío y recepción** (Send Connectors / Receive Connectors), y **reglas de transporte** (Transport Rules) |
| `relayhost`, `relay_domains` (Postfix) | `New-SendConnector` (define a qué servidor/dominio se reenvía el correo saliente) |
| Recibir correo de un dominio concreto | `New-ReceiveConnector` |
| `aliases`, `newaliases` | **Grupos de distribución** y **alias de destinatario** en Exchange, gestionados con `New-DistributionGroup`/`Set-Mailbox -EmailAddresses` |
| `mailq` (consultar la cola de correo) | `Get-Queue` (PowerShell de Exchange) |
| Reintentar el envío de mensajes en cola | `Retry-Queue` |
| Ver el rastro de un mensaje concreto | `Get-MessageTrackingLog` (equivalente muy superior a revisar el log plano de Postfix/sendmail, con búsqueda estructurada) |

**Comandos principales del lado servidor (equivalente a la administración de Postfix):**

| Cmdlet | Función |
|---|---|
| `Get-TransportService` | Muestra los servidores con el servicio de transporte |
| `Get-Queue` | Lista los mensajes en cola de envío |
| `Retry-Queue -Identity nombre` | Fuerza un nuevo intento de entrega de una cola |
| `Suspend-Queue` / `Resume-Queue` | Pausa/reanuda una cola concreta |
| `New-SendConnector -Name "Internet" -AddressSpaces "*" -SmartHosts IP_relay` | Configura el envío de correo saliente a través de un relay (equivalente a `relayhost` de Postfix) |
| `New-ReceiveConnector` | Define un conector de entrada (equivalente a `smtpd` en Postfix escuchando en un puerto) |

---

## 4. Entrega local avanzada: procmail → Reglas de transporte y reglas de bandeja de entrada

| Concepto Linux | Equivalente en Windows Server 2025 |
|---|---|
| `/etc/procmailrc` (reglas globales del sistema) | **Reglas de transporte** (Transport Rules / Mail Flow Rules), aplicadas a nivel de organización por el administrador, gestionadas con `New-TransportRule`/`Set-TransportRule` desde PowerShell o el EAC |
| `$HOME/.procmailrc` (reglas propias del usuario) | **Reglas de bandeja de entrada** (Inbox Rules), configurables por cada usuario desde Outlook/OWA, o por el administrador con `New-InboxRule` |
| Condiciones sobre cabeceras/cuerpo del mensaje | Condiciones equivalentes en `New-TransportRule`: `-SubjectContainsWords`, `-From`, `-SentTo`, `-HeaderContainsMessageHeader`, etc. |
| Acciones (guardar, reenviar, descartar) | Acciones equivalentes: `-RedirectMessageTo`, `-DeleteMessage`, `-BlindCopyTo`, `-PrependSubject`, `-ApplyClassification` |

**Ejemplo (equivalente a una receta de procmail que reenvía correo según el asunto):**
```powershell
New-TransportRule -Name "Reenviar avisos de facturación" `
  -SubjectContainsWords "factura" `
  -RedirectMessageTo "contabilidad@empresa.com"
```

---

## 5. Filtrado con Sieve → Reglas de transporte / Reglas de Outlook

Windows/Exchange no implementa el estándar Sieve (RFC 5228); su función la cubren dos mecanismos complementarios:

| Concepto Sieve | Equivalente en Exchange |
|---|---|
| Reglas a nivel de servidor, aplicadas por el MDA | **Transport Rules** (a nivel de organización, gestionadas por el administrador) |
| Reglas a nivel de buzón individual | **Inbox Rules** de Outlook/OWA (configurables por el propio usuario, similar en alcance a un script Sieve personal) |
| `fileinto` (mover a una carpeta) | Acción "Mover el mensaje a la carpeta..." en una regla de Outlook, o `-MoveToFolder` en `New-InboxRule` |
| `redirect` | Acción "Reenviar a" / `-ForwardTo` |
| `discard` | Acción "Eliminar" / `-DeleteMessage` |
| `if`/`allof`/`anyof` (condiciones) | Condiciones combinables gráficamente en el asistente de reglas de Outlook, o con múltiples parámetros de condición en `New-InboxRule`/`New-TransportRule` |

---

## 6. Acceso remoto a buzones: POP3 e IMAP

Exchange Server incluye servicios POP3 e IMAP4 propios, **desactivados por defecto** (el protocolo nativo y recomendado para Outlook es MAPI/Exchange ActiveSync, no POP3/IMAP).

| Concepto Linux | Equivalente en Windows Server 2025 (Exchange) |
|---|---|
| Puerto POP3 (110) | Igual, servicio **Microsoft Exchange POP3** |
| Puerto IMAP (143) | Igual, servicio **Microsoft Exchange IMAP4** |
| Activar/configurar el servicio POP3 | `Set-PopSettings` (PowerShell de Exchange), y arrancar el servicio: `Start-Service MSExchangePOP3` |
| Activar/configurar el servicio IMAP4 | `Set-ImapSettings`, y `Start-Service MSExchangeIMAP4` |
| Servidor Courier / Dovecot (servidores POP3/IMAP independientes en Linux) | En Windows no hace falta un servidor aparte: los servicios POP3/IMAP4 forman parte del propio Exchange Server, sobre el mismo almacén de buzones |

**Ejemplo de configuración (equivalente a los ajustes de `ADDRESS`/`PORT`/`MAXDAEMONS` de Courier):**
```powershell
Set-PopSettings -ExternalConnectionSettings "mail.empresa.com:995:SSL","mail.empresa.com:110:TLS" -X509CertificateName mail.empresa.com
Set-ImapSettings -ExternalConnectionSettings "mail.empresa.com:993:SSL","mail.empresa.com:143:TLS" -X509CertificateName mail.empresa.com
```

---

## 7. Alternativas más ligeras (equivalente a "no usar Exchange")

Para escenarios sencillos que no requieran todo el peso de Exchange Server (equivalente a montar un Postfix/Dovecot básico en vez de una solución empresarial), existen opciones de terceros para Windows:

| Solución | Comparable a |
|---|---|
| **hMailServer** (gratuito, código abierto) | Postfix + Dovecot combinados en un solo instalador ligero para Windows, con SMTP, POP3 e IMAP |
| **IIS SMTP Server** (característica heredada de Windows) | Un relay SMTP muy básico, sin buzones reales; comparable a un Postfix mínimo usado solo como relay, no como servidor de buzones completo |

---

## 8. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 7) | Equivalente en Windows Server 2025 |
|---|---|
| sendmail / Postfix (MTA) | Servicio de transporte de **Exchange Server** |
| `mailertable`/`virtusertable`/`access` | Conectores de envío/recepción, reglas de transporte |
| `mailq` | `Get-Queue` |
| `newaliases`/`aliases` | Grupos de distribución, alias de destinatario |
| procmail (`.procmailrc`) | Reglas de transporte (admin) / Reglas de bandeja de entrada (usuario) |
| Sieve | Transport Rules / Inbox Rules de Outlook |
| Buzón tipo mbox/maildir | Base de datos de buzones (`.edb`) |
| Courier / Dovecot (POP3-IMAP) | Servicios POP3/IMAP4 integrados en Exchange (`Set-PopSettings`, `Set-ImapSettings`) |
| Alternativa ligera (Postfix+Dovecot básico) | **hMailServer** |

---

*Documento elaborado como contrapartida en Windows Server 2025 al Capítulo 7 ("Organizing Email Services") del libro LPIC-2. Fuentes: conocimiento general de administración de Windows Server y Exchange Server, verificado con documentación oficial de Microsoft Learn (`New-TransportRule`, `Set-PopSettings`, `Set-ImapSettings` para Exchange Server 2019/SE).*
