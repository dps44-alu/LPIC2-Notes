# LPIC-2 · Capítulo 11: Managing Network Clients
### Recopilación de comandos y archivos de configuración

Objetivos del examen cubiertos: 210.1 (DHCP configuration), 210.2 (PAM authentication), 210.3 (LDAP client usage), 210.4 (Configuring an OpenLDAP server)

---

## 1. DHCP

### 1.1 Conceptos

Métodos de asignación de IP: **estática** (manual), **dinámica** (DHCP) o mixta. DHCP (Dynamic Host Configuration Protocol) usa un servidor central que escucha peticiones broadcast de los clientes al arrancar y les asigna una IP única.

### 1.2 Instalación del servidor (ISC DHCPd)

| Distribución | Paquete |
|---|---|
| Debian | `isc-dhcp-server` (`sudo apt-get install isc-dhcp-server`; distros antiguas: `dhcp3-server`) |
| Red Hat | `dhcp` (`yum install dhcp`) |

Fichero de configuración (igual en ambas familias): `/etc/dhcp/dhcpd.conf`.

### 1.3 Opciones globales

Se definen al principio del fichero y aplican a todos los clientes:
```
option domain-name-servers 10.0.0.10 10.0.0.11;
option smtp-server 10.0.0.100;
option pop-server 10.0.0.100;
option nntp-server 10.0.0.101;
option time-servers 10.0.0.150;
```

### 1.4 Definición de subredes (`subnet`)

```
subnet 10.1.0.0 netmask 255.255.0.0 {
    option router 10.1.0.1;
    option broadcast-address 10.1.255.255;
    range 10.1.0.10 10.1.0.200;
}
```

| Directiva | Función |
|---|---|
| `subnet ... netmask ...` | Define la subred y su máscara |
| `option router` | Router/gateway por defecto para esa subred |
| `option broadcast-address` | Dirección de broadcast local |
| `range` | Rango (pool) de IPs a asignar dinámicamente |

Varias subredes con opciones comunes se pueden agrupar con `shared-net nombre { ... }`.

### 1.5 Direcciones IP fijas por dispositivo (hosts estáticos)

```
host shadrach {
    hardware ethernet 00:01:02:FE:DC:BA;
    fixed-address 10.1.0.5;
    option router 10.1.0.1;
    option broadcast-address 10.1.255.255;
    netmask 255.255.0.0;
    option host-name "shadrach";
}
```

| Directiva | Función |
|---|---|
| `host nombre { ... }` | Agrupa la configuración de un dispositivo concreto |
| `hardware ethernet` | Dirección MAC del dispositivo |
| `fixed-address` | IP fija que siempre se le asignará |
| `option host-name` | Nombre de host asignado |

Para agrupar varios hosts estáticos que comparten router/broadcast/máscara: `group { ... host { } host { } }`.

### 1.6 Soporte BOOTP

Para dispositivos sin disco (o routers/switches que cargan su SO por red):
```
allow booting;
allow bootp;
```
Y en cada `host`:
```
filename "/mybootfile.img";
server-name "mainhost";
next-server "backuphost";
```

### 1.7 Ficheros de estado y utilidades

| Elemento | Función |
|---|---|
| `/var/lib/dhcp/dhcpd.leases` | Registra las IPs actualmente asignadas (leases) y su duración |
| `arp` | Alternativa para ver IPs y sus MACs en la red |

### 1.8 Clientes DHCP

| Cliente | Notas |
|---|---|
| `dhclient` | El más usado en Debian y Red Hat; se instala/activa automáticamente al detectar tarjeta de red |
| `dhcpcd` | Alternativa |
| `pump` | Alternativa, menos usado |

---

## 2. PAM (Pluggable Authentication Modules)

### 2.1 Qué resuelve

PAM ofrece una API común para que cualquier aplicación delegue la autenticación sin tener que implementar cada método (fichero local, Kerberos, NIS, LDAP...) por su cuenta.

### 2.2 Dos métodos de configuración

| Método | Descripción |
|---|---|
| Fichero único | `/etc/pam.conf`, con una línea por regla: `servicio tipo control módulo argumentos` |
| Ficheros por aplicación | `/etc/pam.d/`, un fichero por servicio (el nombre del fichero es el nombre del servicio, y se omite ese campo en cada línea) |

### 2.3 Estructura de una línea PAM

`servicio tipo control módulo argumentos`

**Tipos (`type`):**

| Tipo | Función |
|---|---|
| `account` | Verificación de la cuenta |
| `auth` | Autenticación |
| `password` | Gestión de contraseñas |
| `session` | Servicios externos (logging, montaje de directorio, etc.) |

**Valores de control (`control`):**

| Control | Comportamiento si falla |
|---|---|
| `requisite` | Termina la aplicación inmediatamente |
| `required` | Devuelve fallo pero sigue comprobando el resto de reglas |
| `sufficient` | Si tiene éxito, detiene el proceso con éxito (si falla, sigue) |
| `optional` | No es determinante salvo que sea la única regla definida |

Ejemplo:
```
login auth required pam_unix.so
```

### 2.4 Módulos de autenticación (Tabla 11.2)

| Módulo | Método |
|---|---|
| `pam_unix.so` | `/etc/passwd` y `/etc/shadow` estándar |
| `pam_krb5.so` | Kerberos 5 |
| `pam_ldap.so` | Servidor LDAP |
| `pam_nis.so` | Servidor NIS |
| `pam_sss.so` | System Security Services Daemon (SSSD), para directorios de red (AD, LDAP) |
| `pam_userdb.so` | Fichero de base de datos `.db` |

### 2.5 Otros módulos de librería (Tabla 11.3, selección)

| Módulo | Función |
|---|---|
| `pam_access.so` | Login anónimo (ej. FTP público) |
| `pam_chroot.so` | Crea un entorno de login enjaulado |
| `pam_console.so` | Entorno de login por consola |
| `pam_cracklib.so` | Comprueba la fortaleza de la contraseña |
| `pam_deny.so` | Prohíbe el login (usado como default) |
| `pam_env.so` | Define/borra variables de entorno |
| `pam_lastlog.so` | Muestra la última vez que se conectó la cuenta |
| `pam_limits.so` | Aplica límites de recursos (ficheros abiertos, CPU...) |
| `pam_listfile.so` | Permite/deniega según una lista en fichero |

### 2.6 SSSD (System Security Services Daemon)

Permite autenticar contra directorios de red (Active Directory, OpenLDAP) usando las mismas credenciales en varios sistemas Linux/Windows; se integra con PAM a través de `pam_sss.so`.

---

## 3. OpenLDAP

### 3.1 Conceptos del directorio LDAP

| Elemento | Descripción |
|---|---|
| Árbol LDAP (LDAP tree / DIT) | Estructura jerárquica donde se almacenan los objetos |
| Objeto | Elemento del directorio (usuario, grupo, etc.) |
| Atributo | Valor de información asociado a un objeto |
| Object class | Plantilla que define el conjunto de atributos aplicable a un objeto |
| Distinguished Name (DN) | Nombre único que identifica la posición y el nombre de un objeto en el árbol |
| Schema | Define la estructura de la base de datos: qué objetos existen y cómo se relacionan |

### 3.2 Métodos de configuración del servidor

| Método | Descripción |
|---|---|
| `slapd.conf` | Fichero de texto único: `/etc/slapd.conf` (o `/etc/ldap.conf` según distro) |
| `slapd-config` | Método moderno (desde OpenLDAP 2.3.3), usa ficheros LDIF en `/etc/slapd.d/`, con el fichero núcleo `cn=config`; nunca se edita a mano, se gestiona con las utilidades OpenLDAP. Es el método por defecto en distribuciones Debian y Red Hat actuales |

**Directivas clave de `slapd.conf`:**
```
suffix "dc=ispnet1, dc=net"
rootdn "cn=Administrator, dc=ispnet1, dc=net"
rootpw testpasswd
```

| Directiva | Función |
|---|---|
| `suffix` | Raíz del árbol LDAP (normalmente el dominio de la organización) |
| `rootdn` | Cuenta con privilegios totales de administrador sobre la base LDAP |
| `rootpw` | Contraseña del administrador (en texto plano o cifrada con prefijo `{SHA}`, generada con `slappasswd`) |

### 3.3 Utilidades del lado servidor

| Utilidad | Función |
|---|---|
| `slapd` | Programa principal del servidor LDAP, escucha peticiones de clientes |
| `slapadd` | Añade objetos directamente a la base de datos leyendo un fichero LDIF (requiere que `slapd` esté **parado**, ya que bloquea la base) |
| `slapcat` | Exporta la base de datos LDAP a un fichero LDIF |
| `slapindex` | Reindexa la base de datos según un atributo |
| `slappasswd` | Genera una contraseña cifrada a partir de texto plano |
| `slurpd` | Usado para la replicación entre servidores LDAP |

Ejemplo de fichero LDIF para un nuevo usuario:
```
dn: cn=rblum, dc=engineering, dc=ispnet1, dc=net
cn: Rich Blum
givenName: Rich
sn: Blum
telephoneNumber: 312-555-1234
mail: rich@myhost.com
objectClass: inetOrgPerson
objectClass: top
```
Carga del fichero: `slapadd -l ldif.txt`

### 3.4 Utilidades del lado cliente

| Utilidad | Función |
|---|---|
| `ldapadd` | Añade objetos definidos en un fichero LDIF (enlace/link a `ldapmodify`) — a diferencia de `slapadd`, actúa como cliente contra el servidor en marcha, por lo que no requiere pararlo (aunque es más lento para inserciones masivas) |
| `ldapdelete` | Elimina objetos (también enlace a `ldapmodify`) |
| `ldapmodify` | Modifica atributos de objetos existentes |
| `ldappasswd` | Genera un valor cifrado a partir de texto |
| `ldapsearch` | Consulta la base de datos LDAP |

**Opciones destacadas de `ldapsearch` (Tabla 11.7, selección):**

| Opción | Función |
|---|---|
| `-b base` | DN desde el que empezar la búsqueda |
| `-D bind` | DN usado para autenticarse |
| `-h host` / `-H uri` | Servidor LDAP a consultar |
| `-x` | Autenticación simple |
| `-Z` | Inicia la operación con TLS |
| `-y passfile` | Lee la contraseña desde un fichero |
| `-k` / `-K` | Autenticación Kerberos |

---

## 4. Resumen de diferencias Debian vs Red Hat en este capítulo

| Aspecto | Debian | Red Hat |
|---|---|---|
| Paquete del servidor DHCP | `isc-dhcp-server` | `dhcp` |
| Fichero de configuración DHCP | `/etc/dhcp/dhcpd.conf` (igual) | `/etc/dhcp/dhcpd.conf` (igual) |
| Método de configuración OpenLDAP por defecto | `slapd-config` (moderno) | `slapd-config` (moderno) |
| Resto del capítulo (DHCP, PAM, OpenLDAP) | Igual | Igual |

Este capítulo tiene muy pocas diferencias reales entre familias de distribuciones: la principal es el nombre del paquete del servidor DHCP.

---

*Documento generado a partir del Capítulo 11 ("Managing Network Clients") del libro LPIC-2: Linux Professional Institute Certification Study Guide (Bresnahan & Blum).*
