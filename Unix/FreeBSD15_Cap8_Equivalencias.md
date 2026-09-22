# LPIC-2 · Capítulo 8: Directing DNS
### Equivalencias en FreeBSD 15

Este documento recoge, para cada concepto visto en el Capítulo 8 del libro LPIC-2 (servidor DNS BIND, zonas, diagnóstico y seguridad), cuál es su equivalente en **FreeBSD 15**. Cuando no hay traducción exacta, se explica la diferencia.

Idea general para entender este capítulo: **BIND es el mismo programa en FreeBSD que en Linux**, así que la sintaxis de `named.conf`, los archivos de zona, `rndc` y DNSSEC son idénticos. Lo que cambia es:
- BIND **no viene en el sistema base** (se quitó hace años): se instala como paquete, y por eso sus archivos están en **`/usr/local/etc/namedb/`**.
- FreeBSD trae en el sistema base **`local-unbound`**, un servidor DNS solo caché, y las herramientas de consulta **`host`** y **`drill`**.

---

## 1. Instalación de BIND

| Elemento | Red Hat | Debian | FreeBSD 15 |
|---|---|---|---|
| Paquete del servidor | `bind` | `bind9` | `bind920` (versión estable actual; también existe `bind918`, versión de soporte extendido) |
| Herramientas de consulta | `bind-utils` | `bind9utils` / `dnsutils` | `bind-tools` (`dig`, `nslookup`, `host`) |
| Servicio | `named` | `bind9` | `named` |
| Usuario del proceso | `named` | `bind` | `bind` |

Para ver las versiones disponibles: `pkg search bind9`.

**Instalar y comprobar:**
```
pkg install bind920
sysrc named_enable="YES"
service named start
service named status
ps -aux | grep named
```

El paquete ya incluye `named-checkconf`, `named-checkzone`, `rndc` y las herramientas de DNSSEC.

---

## 2. Fichero principal: `named.conf`

| Distribución | Ubicación |
|---|---|
| Red Hat | `/etc/named.conf` |
| Debian | `/etc/bind/named.conf` |
| **FreeBSD 15** | **`/usr/local/etc/namedb/named.conf`** |

**Estructura de la carpeta `/usr/local/etc/namedb/`** que crea el paquete:

| Archivo / carpeta | Función |
|---|---|
| `named.conf` | Configuración principal (viene muy comentada, con ejemplos) |
| `named.root` | Lista de servidores raíz (zona `hint`) |
| `primary/` | Zonas de las que este servidor es **primario** (`empty.db`, `localhost-forward.db`, `localhost-reverse.db` y las propias) |
| `secondary/` | Copias de zonas recibidas de otro servidor (secundario) |
| `dynamic/` | Zonas con actualizaciones dinámicas |
| `working/` | Directorio de trabajo de `named` (lo indica la opción `directory`) |
| `rndc.key` | Clave para usar `rndc` (se genera automáticamente al arrancar por primera vez) |

La sintaxis es **idéntica** a la del libro (bloques `options`, `logging`, `zone`, `acl`, `include`...). Las opciones globales también: `listen-on`, `directory`, `allow-query`, `recursion`, `dnssec-validation`.

Diferencias de versión que afectan al libro (BIND 9.18 / 9.20):
- `dnssec-enable` ya no existe (DNSSEC viene siempre activado); solo se usa `dnssec-validation auto;`.
- Los tipos de zona `master` y `slave` se llaman ahora **`primary`** y **`secondary`** (los nombres antiguos siguen funcionando).
- Algunas opciones del libro, como `auto-dnssec` o el tipo `delegation-only`, están obsoletas o eliminadas en las versiones actuales.

**Comprobar la sintaxis** (igual que en Linux):
```
named-checkconf /usr/local/etc/namedb/named.conf
```
Sin salida = todo correcto.

---

## 3. Arrancar, parar y recargar BIND

| Acción | Red Hat | Debian | FreeBSD 15 |
|---|---|---|---|
| Activar al arranque | `systemctl enable named` | `systemctl enable bind9` | `sysrc named_enable="YES"` |
| Arrancar | `systemctl start named` | `service bind9 start` | `service named start` |
| Parar | `systemctl stop named` | `service bind9 stop` | `service named stop` |
| Reiniciar | `systemctl restart named` | `service bind9 restart` | `service named restart` |
| Recargar | `systemctl reload named` | `service bind9 reload` | `service named reload` o `rndc reload` |

---

## 4. Servidor DNS solo caché (caching-only / resolver)

### 4.1 Con BIND

Los pasos del libro son **exactamente los mismos**, editando `/usr/local/etc/namedb/named.conf`:

1. Crear la lista de acceso: `acl redlocal { 192.168.1.0/24; localhost; };`
2. `allow-query { redlocal; };`
3. `recursion yes;`
4. Ajustar `listen-on` (por defecto el `named.conf` de FreeBSD escucha **solo en `127.0.0.1`**: hay que añadir la IP de la red)
5. Comprobar con `named-checkconf`
6. Abrir el puerto 53 en el cortafuegos (`pf` o `ipfw`, Capítulo 12)
7. `service named restart`

### 4.2 Con `local-unbound` (propio de FreeBSD, sin equivalente en el libro)

Para una caché DNS **solo para el propio equipo**, FreeBSD trae `local-unbound` en el sistema base, sin instalar nada:

| Elemento | Detalle |
|---|---|
| Activar | `sysrc local_unbound_enable="YES"` + `service local_unbound start` |
| Configuración | `/var/unbound/unbound.conf` (se genera automáticamente con `local-unbound-setup`) |
| Qué hace | Escucha en `127.0.0.1`, guarda en caché las respuestas y valida DNSSEC. Cambia `/etc/resolv.conf` para que el equipo lo use |
| Control | `local-unbound-control` (ej. `local-unbound-control flush_zone example.com`) |

Si se quiere un servidor de caché con Unbound **para toda la red**, se instala el paquete `unbound` (configuración en `/usr/local/etc/unbound/unbound.conf`).

### 4.3 Configurar los clientes

Igual que en el libro: `nameserver IP` en `/etc/resolv.conf`. En FreeBSD, `resolv.conf` puede regenerarse con DHCP o con `resolvconf` (Capítulo 6). Para fijar el servidor DNS de forma permanente:
- Con DHCP: `prepend domain-name-servers IP;` en **`/etc/dhclient.conf`**.
- Con `resolvconf`: `name_servers="IP"` en `/etc/resolvconf.conf`.

---

## 5. Registro de eventos (logging)

La sección `logging { ... }`, los canales y las categorías (`client`, `config`, `default`, `general`, `queries`, `security`, `xfer-in`, `xfer-out`...) son **idénticos**.

Diferencias de FreeBSD:
- Si no se configura nada, BIND envía sus mensajes a `syslog`, y aparecen en **`/var/log/messages`**.
- Si se usa un canal a fichero, la ruta debe estar dentro de una carpeta en la que el usuario `bind` pueda escribir (por ejemplo `/var/log/named/`, creándola con `mkdir /var/log/named && chown bind /var/log/named`). Si BIND está en chroot (apartado 8.1), la ruta es relativa a la carpeta del chroot.

---

## 6. Zonas DNS

### 6.1 Tipos de zona

Los mismos que en el libro, con los nombres actuales:

| Tipo en el libro | Nombre actual (BIND 9.18/9.20) | Función |
|---|---|---|
| `master` | `primary` (también acepta `master`) | Servidor primario |
| `slave` | `secondary` (también acepta `slave`) | Servidor secundario |
| `forward` | `forward` | Reenvía las consultas |
| `hint` | `hint` | Zona raíz |
| `redirect` | `redirect` | Respuestas NXDOMAIN |
| `stub` / `static-stub` | `stub` / `static-stub` | Igual |
| `delegation-only` | Eliminado en las versiones actuales | — |

Ejemplo de zona raíz en FreeBSD (ya viene así en el `named.conf` del paquete):
```
zone "." { type hint; file "/usr/local/etc/namedb/named.root"; };
```

Ejemplo de zona primaria propia:
```
zone "example.com" {
    type primary;
    file "/usr/local/etc/namedb/primary/example.com.db";
    allow-update { none; };
    allow-transfer { 192.168.64.111; };
};
```

Ejemplo de zona secundaria:
```
zone "example.com" {
    type secondary;
    file "/usr/local/etc/namedb/secondary/example.com.db";
    primaries { 192.168.64.110; };
};
```

### 6.2 Ubicación de las bases de datos de zona

| Distribución | Directorio |
|---|---|
| Red Hat | `/var/named/` |
| Debian | `/etc/bind/` |
| **FreeBSD 15** | **`/usr/local/etc/namedb/primary/`** (primarias), **`secondary/`** (secundarias) y **`dynamic/`** (dinámicas) |

Importante: las carpetas `secondary/` y `dynamic/` deben pertenecer al usuario `bind`, porque `named` escribe en ellas. El paquete ya las crea así.

### 6.3 Estructura de una base de datos de zona

**Exactamente igual** que en el libro, porque el formato de zona es un estándar: directivas `$TTL` y `$ORIGIN`, registros `A`, `AAAA`, `CNAME`, `MX`, `NS`, `PTR`, `SOA`, `TXT`, los campos del SOA y las unidades de tiempo (`M`, `H`, `D`, `W`). Los ejemplos del libro (zona directa y zona inversa `in-addr.arpa.`) funcionan sin cambios.

### 6.4 Comprobar una zona

Igual que en Linux, cambiando la ruta:
```
named-checkzone example.com /usr/local/etc/namedb/primary/example.com.db
```
Respuesta `OK` = sintaxis correcta.

### 6.5 Delegación de zona

Igual que en el libro: registro `NS` para la subzona más su registro glue (`A`).

---

## 7. Herramientas de diagnóstico

| Herramienta del libro | FreeBSD 15 | Notas |
|---|---|---|
| `host nombre` | `host nombre` (sistema base) | `host -t MX dominio` funciona igual |
| `dig nombre` | **`drill nombre`** (sistema base) o `dig` (paquete `bind-tools` o incluido con BIND) | `drill` da la misma información que `dig` con un formato casi igual |
| `dig @servidor nombre` | `drill @servidor nombre` | Consultar a un servidor concreto |
| `dig +trace nombre` | `drill -T nombre` | Recorrido desde la raíz hasta el servidor autoritativo |
| `dig MX +short dominio` | `drill MX dominio` | (`drill` no tiene `+short`) |
| — | `drill -D nombre` | Muestra también los registros DNSSEC |
| — | `drill -x 192.168.64.120` | Consulta inversa (`dig -x`) |
| `nslookup` | `nslookup` (paquete `bind-tools`) | No viene en el sistema base |

### 7.1 `rndc`

**Idéntico** al del libro (`rndc status`, `reload`, `reload zona`, `reconfig`, `stop`, `halt`, `flush`, `flushname`, `querylog on|off`). Detalles de FreeBSD:
- La clave de `rndc` está en `/usr/local/etc/namedb/rndc.key` y se crea automáticamente la primera vez que arranca `named` (manualmente: `rndc-confgen -a`).
- Igual que dice el libro, `rndc` no puede arrancar BIND: para eso se usa `service named start`.

Para `local-unbound`, el equivalente de `rndc` es `local-unbound-control` (ej. `local-unbound-control stats`, `local-unbound-control flush nombre`).

---

## 8. Seguridad de BIND

Las buenas prácticas del libro se aplican **igual**. En FreeBSD:
- **Mantenerlo actualizado**: `pkg upgrade bind920`. La herramienta `pkg audit -F` avisa si hay fallos de seguridad conocidos en los paquetes instalados.
- **Usuario no root**: el script de arranque ya lanza `named` con el usuario `bind` (`named_uid="bind"` en `/etc/defaults/rc.conf`).
- **Ocultar la versión**: `version "no disponible";` dentro de `options`, igual que en Linux.
- **Vistas, `allow-update`, `allow-transfer`, DNSSEC, TSIG**: sintaxis idéntica.
- **Separar servicios**: en FreeBSD es muy habitual meter BIND dentro de una **jail** (entorno aislado propio de FreeBSD), lo que va más allá del chroot.

### 8.1 Chroot jail

En FreeBSD el chroot de BIND no necesita un paquete aparte como `bind-chroot`: lo hace el propio script de arranque con dos líneas de `/etc/rc.conf`:

| Variable de `rc.conf` | Función |
|---|---|
| `named_chrootdir="/var/named"` | Carpeta que será la raíz de BIND. Si está vacía (valor por defecto), no se usa chroot |
| `named_chroot_autoupdate="YES"` | Crea y actualiza automáticamente la estructura de carpetas dentro del chroot (valor por defecto) |

| Linux (Red Hat) | FreeBSD 15 |
|---|---|
| Paquete `bind-chroot` | No hace falta |
| `/usr/libexec/setup-named-chroot.sh ... on` | `sysrc named_chrootdir="/var/named"` |
| Servicio `named-chroot` | El mismo servicio `named` |
| Archivos en `/var/named/chroot/var/named/` | Archivos dentro de `/var/named/usr/local/etc/namedb/` (el script los prepara) |

Tras cambiarlo: `service named restart`.

**Alternativa propia de FreeBSD: las jails.** Una jail es un entorno aislado con su propia IP, usuarios y procesos, mucho más completo que un chroot. Es la forma habitual en FreeBSD de separar servicios como BIND del resto del sistema. Se configuran en `/etc/jail.conf` (se tratarán con la seguridad en el Capítulo 12).

### 8.2 DNSSEC y TSIG

Los conceptos (ZSK, KSK, cadena de confianza, DNSKEY, TSIG) y las herramientas (`dnssec-keygen`, `dnssec-signzone`, `dnssec-dsfromkey`) son **idénticos**; vienen con el paquete de BIND.

Diferencia de versión: en BIND 9.18/9.20 la forma recomendada de firmar zonas es la opción **`dnssec-policy`**, que genera y renueva las claves automáticamente, en lugar de `auto-dnssec` (eliminada en 9.20):
```
zone "example.com" {
    type primary;
    file "/usr/local/etc/namedb/dynamic/example.com.db";
    dnssec-policy default;
    inline-signing yes;
};
```

Para generar una clave TSIG en las versiones actuales se usa `tsig-keygen nombre_clave` (sustituye al uso de `dnssec-keygen` para TSIG que aparece en el libro).

### 8.3 Otros servidores DNS disponibles

| Programa | Uso | Paquete |
|---|---|---|
| Unbound | Solo caché / resolver | `local-unbound` (sistema base) o `unbound` |
| NSD | Solo autoritativo (sin caché) | `nsd` |
| Knot DNS | Solo autoritativo | `knot3` |
| PowerDNS | Autoritativo con base de datos | `powerdns` |

Separar el servidor autoritativo (NSD, Knot) del de caché (Unbound) sigue la recomendación del libro de no mezclar funciones DNS en el mismo servidor.

---

## 9. Resumen de correspondencias clave del capítulo

| Concepto en Linux (LPIC-2 Cap. 8) | Equivalente en FreeBSD 15 |
|---|---|
| Paquete `bind` / `bind9` | Paquete `bind920` (o `bind918`) |
| `bind-utils` / `dnsutils` | `bind-tools` (o `host` y `drill` del sistema base) |
| Servicio `named` / `bind9` | Servicio `named` (`named_enable="YES"`) |
| Usuario `named` / `bind` | Usuario `bind` |
| `/etc/named.conf` / `/etc/bind/named.conf` | `/usr/local/etc/namedb/named.conf` |
| `/var/named/` / `/etc/bind/` | `/usr/local/etc/namedb/primary/`, `secondary/`, `dynamic/` |
| `type master` / `slave` | `type primary` / `secondary` (los antiguos siguen valiendo) |
| `named-checkconf` / `named-checkzone` | Iguales |
| `rndc` | Igual (clave en `/usr/local/etc/namedb/rndc.key`) |
| `dig` | `drill` (sistema base) o `dig` (paquete) |
| `dig +trace` | `drill -T` |
| `nslookup` | Paquete `bind-tools` |
| Servidor solo caché | BIND o `local-unbound` (sistema base) |
| `/etc/dhcp/dhclient.conf` | `/etc/dhclient.conf` |
| `bind-chroot` + `named-chroot` | `named_chrootdir="/var/named"` en `/etc/rc.conf` |
| Aislamiento avanzado | Jails de FreeBSD |
| `auto-dnssec` | `dnssec-policy` |
| `dnssec-keygen` para TSIG | `tsig-keygen` |

---

*Documento elaborado como contrapartida en FreeBSD 15 al Capítulo 8 ("Directing DNS") del libro LPIC-2. Fuentes: FreeBSD Handbook (sección "Domain Name System (DNS)" del capítulo "Network Servers") y páginas de manual de FreeBSD: rc.conf(5), local-unbound(8), local-unbound-setup(8), drill(1), host(1), resolv.conf(5), dhclient.conf(5), además de la documentación de BIND 9 incluida en el paquete.*
