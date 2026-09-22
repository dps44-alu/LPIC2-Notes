# LPIC-2 · Capítulo 7: Organizing Email Services
### Recopilación de comandos y archivos de configuración

Objetivos del examen cubiertos: 211.1 (Using e-mail servers), 211.2 (Managing E-Mail Delivery), 211.3 (Managing Mailbox Access)

---

## 1. Arquitectura del correo en Linux

Linux sigue el modelo Unix: en vez de un único programa monolítico, el correo se reparte en varios programas especializados que colaboran entre sí.

| Componente | Función |
|---|---|
| **MTA** (Mail Transfer Agent) | Envía y recibe mensajes; decide si un mensaje es local o debe transferirse a otro servidor. Ejemplos: sendmail, Postfix, Exim, qmail |
| **MDA** (Mail Delivery Agent) | Recibe los mensajes locales del MTA y decide cómo entregarlos (buzón, carpeta, reenvío...). Ejemplo: procmail |
| **MUA** (Mail User Agent) | Programa con el que el usuario lee y escribe correo. Ejemplos de texto: `binmail`; gráficos: Evolution, Thunderbird, KMail |

La frontera entre estas tres funciones no siempre es nítida: algunos paquetes combinan MTA+MDA, otros MDA+MUA.

### 1.1 Tipos de buzón de usuario

| Tipo | Descripción |
|---|---|
| `mbox` (`/var/spool/mail`) | Un único fichero por usuario con todos sus mensajes |
| `$HOME/mail` | Variante del mbox, en el directorio personal del usuario |
| `maildir` | Un directorio por usuario, con **un fichero independiente por mensaje** (no necesita fichero de bloqueo/lock) |

---

## 2. SMTP (Simple Mail Transfer Protocol)

| Elemento | Detalle |
|---|---|
| Puerto | TCP 25 |
| Comprobación manual | `telnet localhost 25` — si hay un servidor SMTP, responde con un código de 3 dígitos + texto descriptivo |
| Comandos ESMTP destacados | `ETRN` (intercambia roles cliente/servidor para transferir mensajes nuevos), `STARTTLS` (negocia sesión cifrada) |

---

## 3. Sendmail

### 3.1 Componentes (Tabla 7.5)

| Fichero/Programa | Función |
|---|---|
| `sendmail` | Ejecutable principal |
| `sendmail.cf` | Fichero de configuración principal (por defecto `/etc/mail/sendmail.cf`) |
| `sendmail.cw` | Lista de dominios para los que el servidor recibe correo |
| `sendmail.ct` | Lista de usuarios de confianza |
| `aliases` | Direcciones locales válidas que redirigen a otro usuario, fichero o programa |
| `newaliases` | Regenera la base de datos de alias a partir del fichero de texto |
| `mailq` | Consulta la cola de correo pendiente |
| `mqueue` | Directorio donde se guardan los mensajes pendientes de entrega |
| `mailertable` | Sobrescribe el enrutado para dominios concretos |
| `domaintable` | Mapea nombres de dominio antiguos a nuevos |
| `virtusertable` | Mapea usuarios/dominios a direcciones alternativas |
| `relaydomains` | Hosts autorizados a usar el servidor como relay |
| `access` | Permite o deniega mensajes de dominios concretos |

### 3.2 El fichero `sendmail.cf`: líneas de configuración (Tabla 7.6)

| Letra inicial | Define |
|---|---|
| `C` | Clases de texto |
| `D` | Macros |
| `F` | Ficheros con clases de texto |
| `H` | Cabeceras y acciones |
| `K` | Bases de datos de búsqueda |
| `M` | Mailers (métodos de transporte) |
| `O` | Opciones de sendmail |
| `P` | Valores de precedencia |
| `R` | Reglas para analizar direcciones |
| `S` | Grupos de reglas |

Editar `sendmail.cf` directamente es muy complejo; por eso se usa el preprocesador `m4`.

### 3.3 Generar `sendmail.cf` con `m4`

| Macro (Tabla 7.7) | Función |
|---|---|
| `define` | Define valores de opciones concretas |
| `divert(n)` | Controla el buffer de m4 (`-1` borra, `0` inicia uno nuevo) |
| `DOMAIN` | Dominio(s) que usará el MTA |
| `FEATURE` | Activa una característica concreta (ej. `access_db`, `virtusertable`, `local_procmail`) |
| `MAILER` | Método de transporte (ej. `smtp`, `procmail`) |
| `MASQUERADE_AS` | Nombre de host alternativo con el que responde sendmail |
| `OSTYPE` | Sistema operativo, para incluir macros específicas |

Sintaxis de comillas de `m4`: comilla de apertura `` ` `` y de cierre `'`. Cada línea termina en `dnl`.

Generar el fichero final:
```
m4 < myserver.mc > /etc/mail/sendmail.cf
```

### 3.4 Ejecutar sendmail

```
sendmail -bd -q5m
```
`-bd` = ejecutar como demonio en segundo plano; `-q5m` = revisar la cola de salida cada 5 minutos.

---

## 4. Postfix

### 4.1 Ficheros de configuración (Tabla 7.10)

Ubicación por defecto: `/etc/postfix/`

| Fichero | Función |
|---|---|
| `install.cf` | Parámetros usados durante la instalación |
| `main.cf` | Parámetros de procesamiento de mensajes |
| `master.cf` | Controla el arranque/parada de los procesos núcleo de Postfix |

Se puede modificar la configuración con Postfix en marcha; para recargarla sin bajar el servidor:
```
kill -HUP <PID del proceso master>
```

### 4.2 Parámetros comunes de `main.cf` (Tabla 7.11)

| Parámetro | Función |
|---|---|
| `myhostname` | Nombre de host del servidor (por defecto, `gethostname()`) |
| `mydomain` | Dominio local (por defecto, la parte de dominio de `myhostname`) |
| `myorigin` | Dominio/host usado en los mensajes salientes |
| `mydestination` | Dominio(s) para los que Postfix acepta correo |
| `relay_domains` | Dominios permitidos para reenviar correo |
| `relayhost` | Servidor SMTP intermedio para todo el correo saliente |
| `virtual_alias_domains` | Fichero que mapea alias de dominio a cuentas de usuario |

### 4.3 Programas núcleo de Postfix (Tabla 7.8)

| Programa | Función |
|---|---|
| `master` | Controla el arranque del resto de procesos de Postfix (debe estar siempre activo) |
| `pickup` | Detecta mensajes nuevos en la cola `maildrop` y los pasa a `cleanup` |
| `cleanup` | Procesa las cabeceras entrantes y las coloca en la cola de entrada |
| `qmgr` | Gestor central de colas: decide dónde y cómo se entrega cada mensaje |
| `smtp` | Envía mensajes a servidores externos vía SMTP |
| `smtpd` | Recibe mensajes de servidores externos vía SMTP |
| `local` | Entrega mensajes a usuarios locales |
| `bounce` | Registra y devuelve mensajes rebotados |
| `flush` | Gestiona mensajes pendientes de recogida por un servidor remoto |
| `trivial-rewrite` | Normaliza direcciones de cabecera y resuelve hosts remotos |
| `showq` | Informa del estado de la cola |

### 4.4 Utilidades de Postfix (Tabla 7.9)

| Utilidad | Función |
|---|---|
| `mailq` | Consulta la cola de correo |
| `postfix` | Arranca, para o recarga el sistema Postfix |
| `postalias` | Crea/consulta la base de datos de alias |
| `postcat` | Muestra el contenido de ficheros de la cola |
| `postconf` | Consulta/modifica parámetros de `main.cf` |
| `postmap` | Crea/consulta una tabla de búsqueda (lookup table) |
| `postsuper` | Mantenimiento de los directorios de cola |
| `postlog` | Registra un mensaje en el log del sistema con formato Postfix |

### 4.5 Comandos de emulación de sendmail

Postfix sustituye `/usr/sbin/sendmail` por su propia versión compatible:

| Comando | Función |
|---|---|
| `sendmail` | Imita al ejecutable sendmail, pero redirige a Postfix |
| `mailq` | Lista los mensajes en la cola de salida |
| `newaliases` | Regenera la base de datos binaria de alias a partir de `/etc/aliases` |

### 4.6 Tablas de búsqueda (lookup tables) (Tabla 7.12)

| Tabla | Función |
|---|---|
| `access` | Acepta/rechaza hosts SMTP remotos (seguridad) |
| `alias` | Mapea destinatarios alternativos a buzones locales |
| `canonical` | Mapea nombres de buzón alternativos a los reales (en cabeceras) |
| `relocated` | Mapea un buzón antiguo a uno nuevo |
| `transport` | Mapea dominios a métodos de entrega |
| `virtual` | Mapea destinatarios/dominios a buzones locales |

Cada tabla es un fichero de texto; hay que compilarlo a base de datos binaria con `postmap fichero`.

### 4.7 Directorio de mensajes en proceso

`/var/spool/postfix` — donde Postfix guarda los mensajes mientras los procesa (distinto de `/var/spool/mail`, que es el buzón final estilo mbox).

Log habitual de Postfix: `/var/log/maillog` (según distribución).

---

## 5. Entrega local avanzada: procmail

### 5.1 Activar procmail

| MTA | Cómo activarlo |
|---|---|
| Sendmail | Añadir `` MAILER(`procmail')dnl `` al fichero `.mc` |
| Postfix | Añadir a `main.cf`: `mailbox_command = /usr/bin/procmail -m /etc/procmailrc` |

| Fichero | Alcance |
|---|---|
| `/etc/procmailrc` | Reglas ("recetas") aplicadas a **todo el correo entrante** del sistema |
| `$HOME/.procmailrc` | Reglas propias de cada usuario (fichero oculto) |

### 5.2 Formato de una receta (recipe)

```
:0 [flags] [:fichero_de_bloqueo]
* condición
acción
```

**Flags más usadas (Tabla 7.13):** `H` (mira la cabecera, por defecto), `B` (mira el cuerpo), `c` (hace una copia), `D` (distingue mayúsculas/minúsculas), `A`/`a` (encadena con la receta anterior si se cumplió), `E`/`e` (encadena si la anterior NO se cumplió), `w`/`W` (espera al programa externo y comprueba el código de salida).

**Condiciones especiales (Tabla 7.14):** `!` (invierte), `$` (evalúa con sustitución de shell), `?` (usa el código de salida de un programa), `<`/`>` (compara el tamaño del mensaje), `\` (escapa caracteres especiales).

**Caracteres de la línea de acción (Tabla 7.15):**

| Carácter | Acción |
|---|---|
| `!` | Reenvía el mensaje a las direcciones indicadas |
| `\|` | Ejecuta el programa indicado |
| `{` `}` | Abre/cierra un bloque de recetas anidadas |
| texto (ruta) | Guarda el mensaje en el buzón/carpeta indicada |

Ejemplo — borra cualquier mensaje cuyo asunto contenga "work":
```
:0
* ^Subject.*work
/dev/null
```

---

## 6. Filtrado con Sieve

Lenguaje de programación específico para filtrar correo (RFC 5228), usado por MDAs como Dovecot y Cyrus.

### 6.1 Tipos de comando

| Tipo | Comandos |
|---|---|
| **Acción** | `keep` (guarda en el buzón por defecto), `fileinto` (guarda en una ubicación concreta), `redirect` (reenvía), `discard` (descarta en silencio); extensión opcional: `reject` (rechaza y avisa al remitente) |
| **Control** | `if` (comprobación condicional), `require` (carga extensiones externas), `stop` (termina el script) |
| **Test** (usados dentro de `if`) | `address`, `allof` (AND), `anyof` (OR), `envelope`, `exists`, `false`, `header`, `not`, `size`, `true` |

### 6.2 Ejemplo de script

```
require ["fileinto"];
if header :is "Sender" "guitar-list.org"
 {
 fileinto "guitars";
 }
else
 {
 fileinto "saved";
 }
```

---

## 7. Acceso remoto a buzones: POP3 e IMAP

### 7.1 POP3

| Elemento | Detalle |
|---|---|
| Puerto | TCP 110 |
| Comprobación manual | `telnet localhost 110` |
| Autenticación | `USER`/`PASS` (texto plano), `APOP` (contraseña con hash MD5), `AUTH` (negocia método seguro) |

**Comandos del cliente (Tabla 7.3):** `STAT`, `LIST`, `RETR`, `DELE`, `UIDL` (identificador único por mensaje), `TOP`, `NOOP`, `RSET`, `QUIT`.

Limitación de POP3: normalmente descarga y borra los mensajes del servidor, por lo que el correo queda "atado" a la máquina donde se descargó.

### 7.2 IMAP

| Elemento | Detalle |
|---|---|
| Puerto | TCP 143 |
| Comprobación manual | `telnet localhost 143` |
| Ventaja sobre POP3 | Los mensajes permanecen en el servidor; se puede acceder desde varios sitios sin fragmentar el correo; soporta múltiples buzones por usuario |
| Autenticación | `LOGIN` (texto plano) o `AUTHENTICATE` (negociación cifrada) |
| Cada comando del cliente | Va precedido de un identificador único (ej. `a001 LOGOUT`) |

### 7.3 Servidor Courier

| Elemento | Detalle |
|---|---|
| Ubicación de configuración | `/etc/courier/` |
| `authdaemonrc` | Configuración de autenticación de usuarios |
| `imapd`, `pop3d` | Configuración específica de cada servicio |

**Ajustes destacados (Tabla 7.16):**

| Ajuste | Función |
|---|---|
| `ADDRESS` | IP donde escuchar (0 = todas las interfaces) |
| `MAILDIRPATH` | Directorio donde se guardan los mensajes |
| `MAXDAEMONS` | Máximo de conexiones de clientes simultáneas |
| `MAXPERIP` | Máximo de conexiones por cliente |
| `PORT` | Puerto TCP de escucha |

Courier puede autenticar contra la base de usuarios estándar de Linux, o contra MySQL/LDAP.

### 7.4 Servidor Dovecot

| Elemento | Detalle |
|---|---|
| Fichero de configuración | `/etc/dovecot.conf`, o `/etc/dovecot/` con varios ficheros según distribución |
| `mail_location` | Ubicación de los buzones |
| `listen` | Puerto(s) de escucha |
| `mechanisms` | Métodos de autenticación soportados |
| `doveadm` | Herramienta de administración (gestión de buzones, cuotas, sincronización, búsqueda, estadísticas...) |

---

## 8. Resumen de diferencias Debian vs Red Hat en este capítulo

| Aspecto | Debian | Red Hat |
|---|---|---|
| Instalar procmail/Courier/etc. | `apt-get install ...` | `yum install ...` |
| Resto del capítulo (sendmail, Postfix, procmail, Sieve, Courier, Dovecot) | Igual | Igual |

Este capítulo apenas presenta diferencias entre familias de distribuciones: la configuración del correo (sendmail, Postfix, procmail, Sieve, Courier, Dovecot) es prácticamente idéntica en Debian y Red Hat, salvo el gestor de paquetes usado para instalar el software.

---

*Documento generado a partir del Capítulo 7 ("Organizing Email Services") del libro LPIC-2: Linux Professional Institute Certification Study Guide (Bresnahan & Blum).*
