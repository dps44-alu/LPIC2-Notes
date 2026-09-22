# LPIC-2 · Capítulo 11: Managing Network Clients
### Equivalencias en FreeBSD 15

Este documento recoge, para cada concepto visto en el Capítulo 11 del libro LPIC-2 (servidor DHCP, PAM y OpenLDAP), cuál es su equivalente en **FreeBSD 15**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo:
- **DHCP y OpenLDAP** son los mismos programas que en Linux, instalados como paquetes (configuración en `/usr/local/etc/`).
- **PAM** existe en FreeBSD y usa el mismo formato de reglas, pero la implementación es **OpenPAM** (no Linux-PAM) y **los módulos disponibles son distintos**.
- Las contraseñas de los usuarios no están en `/etc/shadow`, sino en **`/etc/master.passwd`** (ver apartado 2.4).

---

## 1. DHCP

### 1.1 Conceptos

Los mismos que en el libro (asignación estática, dinámica o mixta; peticiones broadcast del cliente al arrancar).

### 1.2 Instalación del servidor

Aviso importante (afecta también a Linux): **ISC dejó de mantener el servidor ISC DHCP en 2022**, y recomienda sustituirlo por su nuevo servidor **Kea**. En FreeBSD 15 hay dos opciones:

| Servidor | Paquete | Configuración | Notas |
|---|---|---|---|
| ISC DHCP (el del libro) | `isc-dhcp44-server` | **`/usr/local/etc/dhcpd.conf`** | Misma sintaxis que el libro. Sin mantenimiento por parte de ISC |
| Kea (recomendado por ISC) | `kea` | `/usr/local/etc/kea/kea-dhcp4.conf` | Configuración en formato JSON |

FreeBSD no incluye ningún servidor DHCP en el sistema base (solo el cliente).

**Activar ISC DHCP** (en `/etc/rc.conf`):
```
dhcpd_enable="YES"
dhcpd_ifaces="em1"
```
`dhcpd_ifaces` indica en qué interfaces debe escuchar (en Debian esto se configura en `/etc/default/isc-dhcp-server`). Arrancar: `service isc-dhcpd start`.

Comprobar la sintaxis antes de arrancar: `dhcpd -t -cf /usr/local/etc/dhcpd.conf`.

### 1.3 a 1.6 Opciones globales, subredes, hosts fijos y BOOTP

La sintaxis de `dhcpd.conf` es **idéntica** a la del libro, porque es el mismo programa: opciones globales, `subnet ... netmask`, `range`, `shared-network`, `host`, `hardware ethernet`, `fixed-address`, `group`, `allow booting`, `allow bootp`, `filename`, `next-server`.

Corrección del ejemplo del libro (afecta también a Linux): la opción del router se escribe **`option routers`** (en plural), no `option router`.

```
option domain-name-servers 10.0.0.10, 10.0.0.11;

subnet 10.1.0.0 netmask 255.255.0.0 {
    option routers 10.1.0.1;
    option broadcast-address 10.1.255.255;
    range 10.1.0.10 10.1.0.200;
}

host shadrach {
    hardware ethernet 00:01:02:FE:DC:BA;
    fixed-address 10.1.0.5;
}
```

Para el arranque por red (PXE/BOOTP) en FreeBSD, `filename` apunta al cargador de FreeBSD (`pxeboot` o `loader.efi`, Capítulo 1), servido por TFTP.

**Equivalencia con Kea** (para quien quiera usar el servidor recomendado):

| ISC DHCP (`dhcpd.conf`) | Kea (`kea-dhcp4.conf`, JSON) |
|---|---|
| `subnet 10.1.0.0 netmask 255.255.0.0 { ... }` | `"subnet4": [ { "subnet": "10.1.0.0/16", ... } ]` |
| `range 10.1.0.10 10.1.0.200;` | `"pools": [ { "pool": "10.1.0.10 - 10.1.0.200" } ]` |
| `option routers 10.1.0.1;` | `"option-data": [ { "name": "routers", "data": "10.1.0.1" } ]` |
| `host { hardware ethernet ...; fixed-address ...; }` | `"reservations": [ { "hw-address": "...", "ip-address": "..." } ]` |
| Activar | `sysrc kea_enable="YES"` + `service kea start` |

### 1.7 Ficheros de estado y utilidades

| Elemento | Linux | FreeBSD 15 |
|---|---|---|
| Registro de asignaciones (ISC DHCP) | `/var/lib/dhcp/dhcpd.leases` | **`/var/db/dhcpd/dhcpd.leases`** |
| Registro de asignaciones (Kea) | `/var/lib/kea/` | `/var/db/kea/` (archivo `kea-leases4.csv`) |
| Tabla ARP | `arp` | `arp -a` |
| Mensajes del servidor | `/var/log/messages` o `syslog` | `/var/log/messages` |

### 1.8 Clientes DHCP

| Cliente | FreeBSD 15 |
|---|---|
| `dhclient` | **Sistema base**, cliente por defecto. Se activa con `ifconfig_em0="DHCP"` en `/etc/rc.conf` (Capítulo 6) |
| `dhcpcd` | Paquete `dhcpcd` |
| `pump` | No existe |

---

## 2. PAM

### 2.1 Qué resuelve

Lo mismo que en Linux: una forma común de que los programas (login, `su`, `sshd`...) deleguen la autenticación. FreeBSD usa **OpenPAM**, una implementación distinta pero compatible en formato con Linux-PAM.

### 2.2 Métodos de configuración

| Método | Linux | FreeBSD 15 |
|---|---|---|
| Ficheros por servicio (sistema base) | `/etc/pam.d/` | **`/etc/pam.d/`** (igual) |
| Ficheros por servicio (programas de paquetes) | `/etc/pam.d/` | **`/usr/local/etc/pam.d/`** |
| Fichero único | `/etc/pam.conf` | `/etc/pam.conf` (soportado, pero no se usa por defecto) |

Archivos habituales en `/etc/pam.d/` de FreeBSD: `system` (reglas comunes que incluyen casi todos los demás), `login`, `sshd`, `su`, `passwd`, `other` (reglas para servicios sin archivo propio), `xdm`, `cron`.

### 2.3 Estructura de una línea PAM

**Idéntica** al libro: `tipo control módulo argumentos` (con el campo `servicio` delante solo en `/etc/pam.conf`).

- **Tipos**: `auth`, `account`, `password`, `session` (iguales).
- **Controles**: `required`, `requisite`, `sufficient`, `optional` (iguales), más dos propios de OpenPAM:

| Control | Función |
|---|---|
| `binding` | Como `sufficient`, pero si falla, el fallo es definitivo (no se prueban más reglas con éxito) |
| `include` | Incluye todas las reglas de otro archivo. Ej.: `auth include system` |

Ejemplo real de `/etc/pam.d/system` en FreeBSD:
```
auth        required    pam_unix.so     no_warn try_first_pass nullok
account     required    pam_login_access.so
account     required    pam_unix.so
session     required    pam_lastlog.so  no_fail
password    required    pam_unix.so     no_warn try_first_pass
```

Los módulos del sistema base están en **`/usr/lib/`** (ej. `/usr/lib/pam_unix.so`); los de paquetes, en `/usr/local/lib/`.

### 2.4 Módulos de autenticación (equivalente a la Tabla 11.2)

| Módulo Linux | FreeBSD 15 | Método |
|---|---|---|
| `pam_unix.so` | `pam_unix.so` (sistema base) | Usuarios locales. En FreeBSD lee **`/etc/master.passwd`** (el equivalente a `/etc/shadow`) |
| `pam_krb5.so` | `pam_krb5.so` (sistema base) | Kerberos 5 |
| `pam_ldap.so` | `pam_ldap.so` (paquetes `pam_ldap` o `nss-pam-ldapd`) | LDAP |
| `pam_nis.so` | No hace falta: `pam_unix.so` usa NIS a través de `/etc/nsswitch.conf` | NIS |
| `pam_sss.so` | Paquete `sssd` (soporte más limitado que en Linux) | Directorios de red (AD, LDAP) |
| `pam_userdb.so` | No existe | — |
| — | `pam_radius.so`, `pam_tacplus.so` (sistema base) | Servidores RADIUS y TACACS+ |

**Sobre `/etc/master.passwd`:** FreeBSD guarda usuarios y contraseñas cifradas en `/etc/master.passwd` (solo lo lee root) y genera a partir de él `/etc/passwd` (sin contraseñas) y unas bases de datos rápidas. Por eso **no se debe editar a mano**: se usa `vipw` (lo edita y regenera todo) o los comandos `pw`, `adduser` y `passwd`. Si se modifica por otros medios, hay que ejecutar `pwd_mkdb -p /etc/master.passwd`.

### 2.5 Otros módulos (equivalente a la Tabla 11.3)

| Módulo Linux | Equivalente en FreeBSD 15 | Función |
|---|---|---|
| `pam_access.so` | `pam_login_access.so` + archivo **`/etc/login.access`** | Permitir o denegar el login según usuario y origen |
| `pam_chroot.so` | `pam_chroot.so` (igual) | Login dentro de un entorno enjaulado |
| `pam_console.so` | `pam_securetty.so` (y la palabra `secure` de `/etc/ttys`) | Controla desde qué terminales puede entrar root |
| `pam_cracklib.so` | **`pam_passwdqc.so`** | Comprueba la fortaleza de las contraseñas |
| `pam_deny.so` | `pam_deny.so` (igual) | Deniega siempre |
| — | `pam_permit.so` | Permite siempre |
| `pam_env.so` | No como módulo: las variables de entorno se definen en **`/etc/login.conf`** (clases de login) | Variables de entorno |
| `pam_lastlog.so` | `pam_lastlog.so` (igual) | Última conexión |
| `pam_limits.so` | No como módulo: los límites se definen en **`/etc/login.conf`** | Límites de recursos (ficheros abiertos, CPU, memoria...) |
| `pam_listfile.so` | `pam_ftpusers.so` (para FTP) / `pam_group.so` (por grupo) | Permitir o denegar según una lista |
| — | `pam_nologin.so` | Impide el login de usuarios normales si existe `/var/run/nologin` |
| — | `pam_rootok.so` | Permite el acceso sin contraseña si ya se es root |
| — | `pam_exec.so` | Ejecuta un programa externo |

**`/etc/login.conf`** (propio de FreeBSD): define **clases de login** con límites de recursos, variables de entorno, formato de contraseña, etc. Cada usuario pertenece a una clase (por defecto `default`). Tras editarlo, hay que regenerar su base de datos:
```
cap_mkdb /etc/login.conf
```
Ejemplo de límites en una clase:
```
default:\
    :openfiles=1024:\
    :maxproc=256:\
    :setenv=EDITOR=vi:\
    ...
```

### 2.6 SSSD

Existe como paquete (`sssd`), pero en FreeBSD es menos habitual que en Linux. Para autenticar contra LDAP o Active Directory, lo más común en FreeBSD es:
- **LDAP**: `nss-pam-ldapd` (demonio `nslcd`, configuración en `/usr/local/etc/nslcd.conf`) más `pam_ldap.so`, y añadir `ldap` en `/etc/nsswitch.conf` (ej. `passwd: files ldap`).
- **Active Directory**: Samba con `winbindd` (Capítulo 10), usando `pam_winbind.so`.

---

## 3. OpenLDAP

### 3.1 Conceptos

Los mismos que en el libro (árbol, objeto, atributo, object class, DN, schema): son parte del estándar LDAP.

### 3.2 Instalación y configuración del servidor

| Elemento | Linux | FreeBSD 15 |
|---|---|---|
| Paquete servidor | `slapd` / `openldap-servers` | **`openldap26-server`** (OpenLDAP 2.6) |
| Paquete cliente | `ldap-utils` / `openldap-clients` | `openldap26-client` (se instala también con el servidor) |
| `slapd.conf` | `/etc/openldap/slapd.conf` o `/etc/ldap/slapd.conf` | **`/usr/local/etc/openldap/slapd.conf`** (el paquete trae un ejemplo `slapd.conf.sample`) |
| `slapd-config` (`cn=config`) | `/etc/openldap/slapd.d/` o `/etc/ldap/slapd.d/` | **`/usr/local/etc/openldap/slapd.d/`** (el paquete trae un ejemplo `slapd.ldif.sample`) |
| Schemas | `/etc/openldap/schema/` | `/usr/local/etc/openldap/schema/` |
| Configuración de los clientes | `/etc/openldap/ldap.conf` | `/usr/local/etc/openldap/ldap.conf` |
| Base de datos | `/var/lib/ldap/` | `/var/db/openldap-data/` |

**Activar el servidor** (en `/etc/rc.conf`):
```
slapd_enable="YES"
slapd_flags='-h "ldapi://%2fvar%2frun%2fopenldap%2fldapi/ ldap://0.0.0.0/"'
slapd_sockets="/var/run/openldap/ldapi"
```
Para usar el método moderno `slapd-config` en lugar de `slapd.conf`, se añade:
```
slapd_cn_config="YES"
```
Arrancar: `service slapd start`.

**Directivas de `slapd.conf`**: **idénticas** a las del libro (`suffix`, `rootdn`, `rootpw`). Para `rootpw` cifrada se usa igual `slappasswd`. Nota de versión: en OpenLDAP 2.6 el tipo de base de datos recomendado es `database mdb`.

### 3.3 Utilidades del lado servidor

**Las mismas** (`slapd`, `slapadd`, `slapcat`, `slapindex`, `slappasswd`), en `/usr/local/sbin/`. La advertencia del libro también se aplica: `slapadd` requiere que el servicio esté parado (`service slapd stop`).

El ejemplo LDIF del libro y su carga con `slapadd -l ldif.txt` funcionan igual.

Nota de versión (afecta también a Linux): **`slurpd` ya no existe** desde OpenLDAP 2.4. La replicación se hace ahora con **syncrepl**, configurado dentro del propio `slapd`.

### 3.4 Utilidades del lado cliente

**Las mismas** (`ldapadd`, `ldapdelete`, `ldapmodify`, `ldappasswd`, `ldapsearch`), en `/usr/local/bin/`.

Nota de versión sobre `ldapsearch` (afecta también a Linux): en OpenLDAP 2.6 la opción **`-h host` ya no existe**; hay que usar siempre **`-H uri`**:
```
ldapsearch -x -H ldap://servidor -b "dc=ispnet1,dc=net" "(cn=rblum)"
```
El resto de opciones de la Tabla 11.7 (`-b`, `-D`, `-x`, `-Z`, `-y`, `-k`) funcionan igual.

---

## 4. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 11) | Equivalente en FreeBSD 15 |
|---|---|
| Paquete `isc-dhcp-server` / `dhcp` | `isc-dhcp44-server` (o `kea`, recomendado) |
| `/etc/dhcp/dhcpd.conf` | `/usr/local/etc/dhcpd.conf` |
| `/etc/default/isc-dhcp-server` (interfaces) | `dhcpd_ifaces="em1"` en `/etc/rc.conf` |
| `/var/lib/dhcp/dhcpd.leases` | `/var/db/dhcpd/dhcpd.leases` |
| `option router` (errata del libro) | `option routers` |
| `dhclient` | `dhclient` (sistema base) |
| Linux-PAM | OpenPAM |
| `/etc/pam.d/` | `/etc/pam.d/` (sistema) y `/usr/local/etc/pam.d/` (paquetes) |
| `/etc/shadow` | `/etc/master.passwd` (se edita con `vipw`) |
| `pam_access.so` | `pam_login_access.so` + `/etc/login.access` |
| `pam_cracklib.so` | `pam_passwdqc.so` |
| `pam_limits.so` / `pam_env.so` | `/etc/login.conf` (+ `cap_mkdb`) |
| `pam_console.so` | `pam_securetty.so` + `/etc/ttys` |
| `pam_nis.so` | `pam_unix.so` + `/etc/nsswitch.conf` |
| `pam_sss.so` / SSSD | `nss-pam-ldapd` o Samba `winbindd` (SSSD disponible como paquete) |
| Paquete `slapd` / `openldap-servers` | `openldap26-server` |
| `/etc/openldap/` o `/etc/ldap/` | `/usr/local/etc/openldap/` |
| `/var/lib/ldap/` | `/var/db/openldap-data/` |
| `systemctl start slapd` | `service slapd start` (`slapd_enable="YES"`) |
| `slapd-config` | `slapd_cn_config="YES"` + `/usr/local/etc/openldap/slapd.d/` |
| `slurpd` | Obsoleto: syncrepl |
| `ldapsearch -h` | `ldapsearch -H ldap://...` |

---

*Documento elaborado como contrapartida en FreeBSD 15 al Capítulo 11 ("Managing Network Clients") del libro LPIC-2. Fuentes: FreeBSD Handbook (secciones "Dynamic Host Configuration Protocol (DHCP)" y "Lightweight Directory Access Protocol (LDAP)" del capítulo "Network Servers", y capítulo "Security") y páginas de manual de FreeBSD: pam(3), pam.conf(5), pam_unix(8), pam_login_access(8), pam_passwdqc(8), login.conf(5), login.access(5), master.passwd(5), vipw(8), pwd_mkdb(8), cap_mkdb(1), nsswitch.conf(5), rc.conf(5), además de la documentación de ISC DHCP, Kea y OpenLDAP incluida en sus paquetes.*
