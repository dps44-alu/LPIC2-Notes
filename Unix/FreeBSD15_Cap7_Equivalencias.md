# LPIC-2 · Capítulo 7: Organizing Email Services
### Equivalencias en FreeBSD 15

Este documento recoge, para cada concepto visto en el Capítulo 7 del libro LPIC-2 (servidores de correo, entrega local, filtrado y acceso a buzones), cuál es su equivalente en **FreeBSD 15**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo: los programas de correo del libro (sendmail, Postfix, procmail, Dovecot, Courier) son los **mismos** en FreeBSD y se configuran casi igual. Lo que cambia es:
- **Qué viene de serie**: desde FreeBSD 14, el MTA por defecto del sistema base es **dma** (DragonFly Mail Agent), un programa pequeño que solo entrega correo local y reenvía el saliente a otro servidor. Sendmail sigue incluido en FreeBSD 15, pero desactivado y con su eliminación del sistema base ya anunciada.
- **Dónde están los archivos**: todo lo que se instala con `pkg` guarda su configuración en **`/usr/local/etc/`** (no en `/etc/`).
- **Cómo se elige el MTA**: con el programa **`mailwrapper`** y su archivo **`/etc/mail/mailer.conf`** (apartado 1.2).

---

## 1. Arquitectura del correo en FreeBSD

El modelo es el mismo que en el libro (MTA, MDA, MUA como programas separados):

| Componente | En el sistema base de FreeBSD 15 | Disponibles como paquete |
|---|---|---|
| **MTA** | `dma` (activo por defecto) y `sendmail` (incluido, pero desactivado) | Postfix, Exim, OpenSMTPD, sendmail (versión del paquete) |
| **MDA** | La entrega local la hace el propio MTA | procmail, maildrop, Dovecot LDA (con Sieve) |
| **MUA** | `mail` / `mailx` (modo texto) | mutt, alpine, Thunderbird, Evolution, KMail |

### 1.1 Tipos de buzón de usuario

| Tipo | Linux | FreeBSD 15 |
|---|---|---|
| `mbox` | `/var/spool/mail/usuario` | **`/var/mail/usuario`** |
| `$HOME/mail` | Igual | Igual (lo decide el MDA o el cliente) |
| `maildir` | `$HOME/Maildir/` | `$HOME/Maildir/` (igual; lo configuran Postfix, Dovecot, etc.) |

Detalle útil: en FreeBSD los informes diarios de `periodic` y las salidas de `cron` llegan por correo local a root, así que es habitual leerlos con `mail` desde la consola.

### 1.2 Elegir el MTA: `mailwrapper` y `/etc/mail/mailer.conf`

Esta pieza es propia de FreeBSD y no aparece en el libro. Los comandos `sendmail`, `mailq` y `newaliases` del sistema son en realidad **enlaces a `mailwrapper`**, que mira el archivo **`/etc/mail/mailer.conf`** para saber qué MTA ejecutar de verdad. Así cualquier programa que llame a `sendmail` funciona, sea cual sea el MTA instalado.

Contenido por defecto (usa dma):
```
sendmail    /usr/libexec/dma
mailq       /usr/libexec/dma
newaliases  /usr/libexec/dma
```

| Para usar... | Qué hacer con `mailer.conf` |
|---|---|
| dma (por defecto) | Nada |
| Sendmail del sistema base | Copiar el ejemplo: `cp /usr/share/examples/sendmail/mailer.conf /etc/mail/mailer.conf` |
| Postfix | `install -m 0644 /usr/local/share/postfix/mailer.conf.postfix /etc/mail/mailer.conf` (el paquete ofrece hacerlo al instalarse) |

---

## 2. SMTP

| Elemento | Linux | FreeBSD 15 |
|---|---|---|
| Puerto | TCP 25 | TCP 25 (igual) |
| Comprobación manual | `telnet localhost 25` | `nc localhost 25` (sistema base) o `telnet localhost 25` si el cliente telnet está instalado |
| Comandos ESMTP (`ETRN`, `STARTTLS`) | Iguales | Iguales (son parte del protocolo) |

Importante: **dma no escucha en el puerto 25**. Solo entrega el correo generado en la propia máquina. Para recibir correo de otros servidores hace falta Sendmail, Postfix u otro MTA completo.

---

## 3. Sendmail

### 3.0 dma, el MTA por defecto (sin equivalente en el libro)

Antes de ver Sendmail, conviene conocer dma, porque es lo que trae FreeBSD 15 funcionando de serie.

| Elemento | Detalle |
|---|---|
| Programa | `/usr/libexec/dma` |
| Qué hace | Entrega el correo local (a `/var/mail/`) y envía el correo externo, directamente o a través de un servidor intermedio (smarthost) |
| Qué no hace | No escucha en el puerto 25: no sirve como servidor de correo para recibir |
| Configuración | `/etc/dma/dma.conf` |
| Usuario y contraseña del smarthost | `/etc/dma/auth.conf` (formato `usuario\|servidor:contraseña`) |
| Cola | `/var/spool/dma/` |
| Procesar la cola | `dma -q` |
| Servicio | No es un demonio, no hay que activarlo en `rc.conf` |

**Parámetros más usados de `/etc/dma/dma.conf`:**

| Parámetro | Equivalente aproximado | Función |
|---|---|---|
| `SMARTHOST servidor` | `relayhost` (Postfix) / `SMART_HOST` (Sendmail) | Envía todo el correo saliente a través de ese servidor |
| `PORT 587` | — | Puerto del smarthost |
| `AUTHPATH /etc/dma/auth.conf` | — | Archivo con las credenciales |
| `SECURETRANSFER` + `STARTTLS` | — | Conexión cifrada con el smarthost |
| `MASQUERADE usuario@dominio` | `MASQUERADE_AS` | Cambia el remitente de los mensajes |
| `NULLCLIENT` | — | No entrega nada en local: lo envía todo al smarthost |

### 3.1 Componentes de Sendmail (equivalente a la Tabla 7.5)

Sendmail sigue en el sistema base de FreeBSD 15. Sus archivos están en **`/etc/mail/`**, como en Linux, pero algunos cambian de nombre:

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `sendmail` | `/usr/libexec/sendmail/sendmail` | Ejecutable principal |
| `/etc/mail/sendmail.cf` | `/etc/mail/sendmail.cf` | Configuración principal (generada, no se edita a mano) |
| `sendmail.cw` | **`/etc/mail/local-host-names`** | Dominios para los que el servidor recibe correo |
| `sendmail.ct` | `/etc/mail/trusted-users` | Usuarios de confianza |
| `aliases` | **`/etc/mail/aliases`** (`/etc/aliases` es un enlace a este archivo) | Alias de correo |
| `newaliases` | `newaliases` | Regenera la base de datos de alias |
| `mailq` | `mailq` | Muestra la cola |
| `mqueue` | `/var/spool/mqueue/` (y `/var/spool/clientmqueue/` para el correo enviado por usuarios locales) | Mensajes pendientes |
| `mailertable` | `/etc/mail/mailertable` | Enrutado por dominio |
| `domaintable` | `/etc/mail/domaintable` | Mapeo de dominios antiguos a nuevos |
| `virtusertable` | `/etc/mail/virtusertable` | Direcciones virtuales |
| `relaydomains` | **`/etc/mail/relay-domains`** | Hosts autorizados a usar el servidor como relay |
| `access` | `/etc/mail/access` | Permitir o denegar por dominio o IP |

### 3.2 El fichero `sendmail.cf` (Tabla 7.6)

El formato (líneas que empiezan por `C`, `D`, `F`, `H`, `K`, `M`, `O`, `P`, `R`, `S`) es **exactamente el mismo**, porque es el mismo programa. Y la recomendación es la misma: no editarlo a mano.

### 3.3 Generar `sendmail.cf` con `m4`

Las macros `m4` (Tabla 7.7: `define`, `divert`, `DOMAIN`, `FEATURE`, `MAILER`, `MASQUERADE_AS`, `OSTYPE`) son **idénticas**. La diferencia es que FreeBSD trae un **`Makefile` en `/etc/mail/`** que hace todo el proceso:

| Archivo | Función |
|---|---|
| `/etc/mail/freebsd.mc` | Plantilla de ejemplo del servidor (incluye `OSTYPE(freebsd6)`) |
| `/etc/mail/freebsd.submit.mc` | Plantilla del envío local (el que usan los usuarios) |
| `/etc/mail/NOMBRE_DEL_EQUIPO.mc` | Plantilla propia. Se crea copiando `freebsd.mc` y se edita esta |

| Comando (desde `/etc/mail`) | Función | Equivalente Linux |
|---|---|---|
| `make` | Genera el `.cf` a partir del `.mc` y compila las tablas (`access`, `virtusertable`...) | `m4 < myserver.mc > sendmail.cf` + `makemap` |
| `make install` | Copia el `.cf` generado a `/etc/mail/sendmail.cf` | Copia manual |
| `make restart` | Reinicia Sendmail para aplicar los cambios | `systemctl restart sendmail` |
| `make aliases` | Regenera la base de datos de alias | `newaliases` |

Se puede seguir usando `m4` directamente, igual que en el libro, si se prefiere.

### 3.4 Ejecutar Sendmail

En lugar de lanzar `sendmail -bd -q5m` a mano, se usa `/etc/rc.conf`:

| Variable de `rc.conf` | Función |
|---|---|
| `sendmail_enable="YES"` | Arranca Sendmail escuchando en el puerto 25 (equivale a `-bd`) |
| `sendmail_enable="NO"` | No escucha en la red, pero se mantiene el procesado de la cola local |
| `sendmail_enable="NONE"` | Desactiva Sendmail por completo (el valor que se usa con dma o Postfix) |
| `sendmail_flags="-L sm-mta -bd -q30m"` | Opciones del demonio (valor por defecto; aquí se cambia el intervalo de la cola) |

Además, hay que apuntar `mailer.conf` a Sendmail (apartado 1.2). Luego: `service sendmail start`.

Log: **`/var/log/maillog`**.

---

## 4. Postfix

Postfix **no viene en el sistema base**; se instala como paquete. Su funcionamiento es idéntico al de Linux.

### 4.0 Instalación y activación

```
pkg install postfix
sysrc postfix_enable="YES"
sysrc sendmail_enable="NONE"
install -m 0644 /usr/local/share/postfix/mailer.conf.postfix /etc/mail/mailer.conf
service postfix start
```

Conviene desactivar también las tareas diarias pensadas para Sendmail, en `/etc/periodic.conf`:
```
daily_clean_hoststat_enable="NO"
daily_status_mail_rejects_enable="NO"
daily_status_include_submit_mailq="NO"
daily_submit_queuerun="NO"
```

### 4.1 Ficheros de configuración (equivalente a la Tabla 7.10)

| Linux | FreeBSD 15 | Función |
|---|---|---|
| `/etc/postfix/` | **`/usr/local/etc/postfix/`** | Carpeta de configuración |
| `main.cf` | `/usr/local/etc/postfix/main.cf` | Parámetros de procesamiento (igual) |
| `master.cf` | `/usr/local/etc/postfix/master.cf` | Procesos de Postfix (igual) |
| `install.cf` | No existe en las versiones actuales | — |

Recargar la configuración sin parar el servidor (equivalente a `kill -HUP` al proceso `master`):
```
postfix reload
```
o `service postfix reload`.

### 4.2 Parámetros de `main.cf` (Tabla 7.11)

**Idénticos** a los del libro: `myhostname`, `mydomain`, `myorigin`, `mydestination`, `relay_domains`, `relayhost`, `virtual_alias_domains`. Se consultan y cambian igual con `postconf`:

| Comando | Función |
|---|---|
| `postconf -n` | Muestra solo los parámetros cambiados respecto al valor por defecto |
| `postconf myhostname` | Muestra un parámetro |
| `postconf -e "myhostname = mail.example.com"` | Cambia un parámetro en `main.cf` |

Detalle de FreeBSD: el paquete configura los alias con `alias_maps = hash:/etc/mail/aliases`, es decir, usa el mismo archivo de alias del sistema.

### 4.3 Programas núcleo de Postfix (Tabla 7.8)

Los mismos (`master`, `pickup`, `cleanup`, `qmgr`, `smtp`, `smtpd`, `local`, `bounce`, `flush`, `trivial-rewrite`, `showq`). En FreeBSD están en **`/usr/local/libexec/postfix/`**.

### 4.4 Utilidades de Postfix (Tabla 7.9)

Las mismas (`mailq`, `postfix`, `postalias`, `postcat`, `postconf`, `postmap`, `postsuper`, `postlog`), en **`/usr/local/sbin/`**.

### 4.5 Comandos de emulación de sendmail

Iguales (`sendmail`, `mailq`, `newaliases`), pero en FreeBSD pasan por `mailwrapper`: solo llaman a Postfix si `/etc/mail/mailer.conf` apunta a él (apartado 1.2).

### 4.6 Tablas de búsqueda (Tabla 7.12)

Las mismas (`access`, `alias`, `canonical`, `relocated`, `transport`, `virtual`), guardadas normalmente en `/usr/local/etc/postfix/`, y se compilan igual con `postmap archivo`.

### 4.7 Directorios y logs

| Elemento | Linux | FreeBSD 15 |
|---|---|---|
| Cola de Postfix | `/var/spool/postfix` | `/var/spool/postfix` (igual) |
| Buzones mbox | `/var/spool/mail` | `/var/mail` |
| Log | `/var/log/maillog` (según distribución) | `/var/log/maillog` |

### 4.8 Otros MTA disponibles

| MTA | Paquete | Configuración |
|---|---|---|
| Exim | `exim` | `/usr/local/etc/exim/configure` |
| OpenSMTPD | `opensmtpd` | `/usr/local/etc/mail/smtpd.conf` |
| Sendmail (versión del paquete) | `sendmail` | Para quien necesite funciones que no trae la versión del sistema base |

---

## 5. Entrega local avanzada: procmail

procmail está disponible como paquete (`pkg install procmail`). Funciona igual, cambiando las rutas:

| Elemento | Linux | FreeBSD 15 |
|---|---|---|
| Programa | `/usr/bin/procmail` | `/usr/local/bin/procmail` |
| Reglas para todo el sistema | `/etc/procmailrc` | `/usr/local/etc/procmailrc` |
| Reglas de cada usuario | `$HOME/.procmailrc` | `$HOME/.procmailrc` (igual) |

### 5.1 Activar procmail

| MTA | Cómo activarlo en FreeBSD 15 |
|---|---|
| Sendmail | En el `.mc`: `` FEATURE(`local_procmail', `/usr/local/bin/procmail')dnl `` y `` MAILER(`procmail')dnl ``, y luego `make install restart` en `/etc/mail` |
| Postfix | En `main.cf`: `mailbox_command = /usr/local/bin/procmail -a "$EXTENSION"` |

### 5.2 Formato de las recetas

**Exactamente igual** que en el libro (flags de la Tabla 7.13, condiciones de la Tabla 7.14 y acciones de la Tabla 7.15), porque es el mismo programa. El ejemplo del libro funciona sin cambios:
```
:0
* ^Subject.*work
/dev/null
```

Nota: procmail ya no se desarrolla activamente. En instalaciones nuevas es más habitual filtrar con **Sieve** en Dovecot (apartado 6) o con `maildrop` (paquete).

---

## 6. Filtrado con Sieve

El lenguaje Sieve es un estándar (RFC 5228), así que **los comandos y el ejemplo del libro son idénticos** en FreeBSD. Lo que cambia es cómo se instala:

| Elemento | FreeBSD 15 |
|---|---|
| Paquete para Dovecot | `dovecot-pigeonhole` (añade Sieve y ManageSieve a Dovecot) |
| Configuración | `/usr/local/etc/dovecot/conf.d/90-sieve.conf` |
| Script de cada usuario | Normalmente `~/.dovecot.sieve` (o el que se indique en la configuración) |
| Protocolo ManageSieve | Puerto TCP 4190, para que los clientes de correo suban sus filtros |

Para que los filtros Sieve se apliquen, el correo debe entregarlo Dovecot (su LDA o LMTP). Con Postfix, por ejemplo: `mailbox_transport = lmtp:unix:private/dovecot-lmtp` en `main.cf`.

---

## 7. Acceso remoto a buzones: POP3 e IMAP

### 7.1 y 7.2 POP3 e IMAP

Los protocolos, puertos y comandos son **los mismos** (POP3: puerto 110, comandos `USER`, `PASS`, `STAT`, `LIST`, `RETR`, `DELE`...; IMAP: puerto 143, comandos con identificador tipo `a001 LOGIN`). Versiones cifradas: POP3S en el 995 e IMAPS en el 993.

Comprobación manual en FreeBSD: `nc localhost 110` o `nc localhost 143` (o `telnet` si está instalado). Para probar las versiones cifradas: `openssl s_client -connect localhost:993`.

FreeBSD no trae ningún servidor POP3/IMAP en el sistema base: se instala Dovecot o Courier como paquete.

### 7.3 Servidor Courier

| Elemento | Linux | FreeBSD 15 |
|---|---|---|
| Paquete | `courier-imap` | `courier-imap` (necesita también `courier-authlib`) |
| Carpeta de configuración | `/etc/courier/` | **`/usr/local/etc/courier-imap/`** |
| `authdaemonrc` | `/etc/courier/authdaemonrc` | `/usr/local/etc/authlib/authdaemonrc` |
| `imapd`, `pop3d` | `/etc/courier/` | `/usr/local/etc/courier-imap/imapd` y `pop3d` |
| Activar | `systemctl enable ...` | `sysrc courier_authdaemond_enable="YES"`, `sysrc courier_imap_imapd_enable="YES"`, `sysrc courier_imap_pop3d_enable="YES"` |

Los ajustes de la Tabla 7.16 (`ADDRESS`, `MAILDIRPATH`, `MAXDAEMONS`, `MAXPERIP`, `PORT`) son **los mismos**. Courier solo trabaja con buzones en formato **maildir**.

### 7.4 Servidor Dovecot

| Elemento | Linux | FreeBSD 15 |
|---|---|---|
| Instalar | `apt-get` / `yum install dovecot` | `pkg install dovecot` |
| Archivo principal | `/etc/dovecot/dovecot.conf` | **`/usr/local/etc/dovecot/dovecot.conf`** |
| Archivos por temas | `/etc/dovecot/conf.d/` | `/usr/local/etc/dovecot/conf.d/` |
| Configuración de ejemplo | Instalada directamente | El paquete trae ejemplos en `/usr/local/etc/dovecot/example-config/`, que se copian a la carpeta de configuración para empezar |
| Activar | `systemctl enable dovecot` | `sysrc dovecot_enable="YES"` + `service dovecot start` |
| `doveadm` | Igual | Igual |
| Ver la configuración efectiva | `doveconf -n` | `doveconf -n` (igual) |

Los parámetros del libro (`mail_location`, `listen`, `auth_mechanisms`) son los mismos. Ejemplo para buzones mbox de FreeBSD:
```
mail_location = mbox:~/mail:INBOX=/var/mail/%u
```
o, para maildir:
```
mail_location = maildir:~/Maildir
```

Atención: Dovecot 2.4 cambió bastantes nombres de parámetros respecto a la 2.3 (por ejemplo, `mail_location` se sustituye por `mail_driver` y `mail_path`). Conviene comprobar qué versión instala el paquete con `pkg info dovecot` y usar los archivos de ejemplo que la acompañan.

---

## 8. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 7) | Equivalente en FreeBSD 15 |
|---|---|
| MTA por defecto de la distribución | `dma` (sistema base, solo local y reenvío) |
| Elegir el MTA (`alternatives`) | `mailwrapper` + `/etc/mail/mailer.conf` |
| `/var/spool/mail` | `/var/mail` |
| `telnet localhost 25` | `nc localhost 25` |
| Sendmail | Sendmail del sistema base (desactivado por defecto; eliminación anunciada) o paquete `sendmail` |
| `sendmail.cw` / `relaydomains` | `/etc/mail/local-host-names` / `/etc/mail/relay-domains` |
| `/etc/aliases` | `/etc/mail/aliases` |
| `m4 < x.mc > sendmail.cf` | `cd /etc/mail && make install restart` |
| `sendmail -bd -q5m` | `sendmail_enable="YES"` y `sendmail_flags` en `/etc/rc.conf` |
| `/etc/postfix/` | `/usr/local/etc/postfix/` |
| `kill -HUP` al master de Postfix | `postfix reload` / `service postfix reload` |
| `/usr/bin/procmail`, `/etc/procmailrc` | `/usr/local/bin/procmail`, `/usr/local/etc/procmailrc` |
| Sieve (Dovecot) | Paquete `dovecot-pigeonhole` |
| `/etc/courier/` | `/usr/local/etc/courier-imap/` y `/usr/local/etc/authlib/` |
| `/etc/dovecot/` | `/usr/local/etc/dovecot/` |
| `/var/log/maillog` | `/var/log/maillog` (igual) |
| `systemctl enable servicio` | `sysrc servicio_enable="YES"` |

---

*Documento elaborado como contrapartida en FreeBSD 15 al Capítulo 7 ("Organizing Email Services") del libro LPIC-2. Fuentes: FreeBSD Handbook (capítulo "Electronic Mail") y páginas de manual de FreeBSD: dma(8), mailwrapper(8), mailer.conf(5), sendmail(8), aliases(5), rc.conf(5), periodic.conf(5), además de la documentación de los paquetes de Postfix, procmail, Dovecot y Courier.*
