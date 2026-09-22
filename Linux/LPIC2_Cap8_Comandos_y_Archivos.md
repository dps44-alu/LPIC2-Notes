# LPIC-2 · Capítulo 8: Directing DNS
### Recopilación de comandos y archivos de configuración

Objetivos del examen cubiertos: 207.1 (Basic DNS server configuration), 207.2 (Create and maintain DNS zones), 207.3 (Securing a DNS server)

---

## 1. Instalación de BIND

| Distribución | Paquetes | Servicio/demonio | Usuario del proceso |
|---|---|---|---|
| **Red Hat** | `bind`, `bind-utils` | `named` | `named` |
| **Debian/Ubuntu** | `bind9`, `bind9utils` | `bind9` | `bind` |

Comprobaciones típicas:
```
systemctl status named | grep -i active      # Red Hat
service bind9 status                          # Debian/Ubuntu
ps -ef | grep ^named                           # Red Hat (usuario named)
ps -ef | grep ^bind                            # Debian (usuario bind)
```

---

## 2. Fichero principal: `named.conf`

| Distribución | Ubicación |
|---|---|
| Red Hat | `/etc/named.conf` |
| Debian/Ubuntu | `/etc/bind/named.conf` |

El fichero se organiza en bloques (clauses): comentarios, `options` (opciones globales), `logging`, y las directivas `zone`. Normalmente incluye otros ficheros con `include`:

```
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.default-zones";
```

**Opciones globales destacadas** (dentro de `options { ... }`):

| Opción | Función |
|---|---|
| `listen-on port 53 { ... }` | Interfaz(es)/puerto donde escucha |
| `directory` | Directorio base de los ficheros de zona |
| `allow-query` | Quién puede consultar el servidor |
| `recursion yes\|no` | Si el servidor responde con recursividad (necesario para caché) |
| `dnssec-enable` / `dnssec-validation` | Activa DNSSEC |

Comprobar la sintaxis:
```
named-checkconf /etc/named.conf
```
(sin salida = todo correcto; si hay error, indica la línea y el problema — típicamente un `;` olvidado).

---

## 3. Arrancar, parar y recargar BIND

| Acción | Red Hat | Debian/Ubuntu |
|---|---|---|
| Arrancar | `systemctl start named` | `service bind9 start` (o `systemctl start bind9`) |
| Parar | `systemctl stop named` | `service bind9 stop` |

También se puede usar la utilidad `rndc` (ver sección de troubleshooting).

---

## 4. Servidor DNS solo caché (caching-only / resolver)

Pasos para configurarlo en `named.conf`:

1. Crear una lista de control de acceso: `acl nombre { IP1; IP2; ... };`
2. `allow-query { nombre_acl; };` (o lista de IPs directamente)
3. Asegurarse de que `recursion yes;` (si estaba en `no`, cambiarlo — sin recursividad no hay caché posible)
4. Ajustar `listen-on` si hace falta
5. Comprobar sintaxis con `named-checkconf`
6. Abrir el puerto 53 en el firewall si aplica
7. Reiniciar/recargar BIND

En los clientes que usarán este servidor de caché, apuntar `/etc/resolv.conf` (`nameserver IP`) a su dirección. Cuidado: en distribuciones modernas, `/etc/resolv.conf` se regenera al arrancar, así que hay que fijar el servidor DNS en el fichero de configuración de red persistente (ver Capítulo 6) o en `/etc/dhcp/dhclient.conf` con `prepend domain-name-servers IP;`.

---

## 5. Registro de eventos (logging)

Se configura dentro de la sección `logging { ... }` de `named.conf`, definiendo **canales** (channel) y asociándolos a **categorías** de mensajes.

**Categorías más comunes:** `client`, `config`, `database`, `default`, `delegation-only`, `dispatch`, `dnssec`, `general` (catch-all), `lame-servers`, `network`, `notify`, `queries`, `query-errors`, `resolver`, `security`, `update`, `xfer-in`, `xfer-out`.

Tras modificar el logging, hay que recargar o reiniciar BIND para que se aplique.

---

## 6. Zonas DNS

Una **zona** define sobre qué porción del espacio de nombres tiene autoridad un servidor. Se compone de:

- **Fichero de configuración de zona** (bloque `zone` en `named.conf` o en un fichero incluido)
- **Base de datos de zona** (zone file/database): el fichero de texto con los registros de recursos

### 6.1 Tipos de zona (directiva `type`)

| Tipo | Función |
|---|---|
| `master` | Servidor primario de la zona |
| `slave` | Servidor secundario (recibe copia vía transferencia de zona) |
| `forward` | Reenvía las consultas a otros servidores (`forwarders { IP; ... };`); con `forward only` solo reenvía, con `forward first` primero reenvía y si no hay respuesta lo intenta él mismo |
| `hint` | Define la zona raíz (lista de root servers actuales) |
| `redirect` | Responde a consultas NXDOMAIN |
| `stub` | Como slave, pero solo transfiere los registros NS (no todo el resto) |
| `static-stub` | Como stub, pero los registros se configuran a mano en vez de transferirse |
| `delegation-only` | Fuerza el estado "solo delegación" de una zona raíz |

Clase de la zona (`IN`, `CH`, `HS`) — normalmente siempre `IN` (Internet).

Ejemplo de zona raíz (hint):
```
zone "." IN {
    type hint;
    file "named.ca";
};
```

Ejemplo de zona forward:
```
zone "forward.example.com" IN {
    type forward;
    forwarders { 192.168.64.106; 192.168.64.107; };
};
```

Control de transferencias/actualizaciones dentro de una zona: `allow-update { none; };`, `allow-transfer { ... };`, `allow-notify { ... };`.

### 6.2 Ubicación habitual de las bases de datos de zona

| Distribución | Directorio |
|---|---|
| Red Hat | `/var/named/` |
| Debian/Ubuntu | `/etc/bind/` |

### 6.3 Estructura de una base de datos de zona (zone file)

**Directivas:**

| Directiva | Función |
|---|---|
| `$TTL segundos` | Tiempo de vida por defecto en caché para los registros que no definan el suyo propio |
| `$ORIGIN dominio.` | Dominio de referencia para los registros relativos (el `@`) |

**Tipos de registro de recurso (Resource Records — Tabla 8.2):**

| Tipo | Descripción |
|---|---|
| `A` | Dirección IPv4 |
| `AAAA` | Dirección IPv6 |
| `CNAME` | Alias hacia otro nombre de host (nunca debe apuntar a otro CNAME ni ser destino de SOA/MX) |
| `MX` | Servidor(es) de correo, con valor de preferencia |
| `NS` | Servidor de nombres autoritativo de la zona |
| `PTR` | Puntero usado en resolución inversa |
| `SOA` | Registro de inicio de autoridad (debe haber solo uno por zona, y ser el primero) |
| `TXT` | Texto libre |

**Campos del registro SOA:**
```
@ IN SOA serv1.example.com. hostmaster.example.com. (
    0       ; serial   (se incrementa en cada cambio)
    2H      ; refresh  (cada cuánto revisa el secundario)
    30M     ; retry    (reintento si falla la revisión)
    2W      ; expire   (máximo sin refresco antes de dejar de ser autoritativo)
    7D      ; minimum  (TTL, tiempo antes de vaciar la caché)
)
```
Valores de tiempo: número solo = segundos; `M` minutos, `H` horas, `D` días, `W` semanas.

**Ejemplo de zona directa (fragmento):**
```
$TTL 604800
$ORIGIN example.com.
@   IN SOA serv1.example.com. hostmaster.example.com. ( ... )
@   IN NS  serv1.example.com.
serv1 IN A 192.168.64.110
@   IN MX 0 maila.example.com.
maila IN A 192.168.64.112
LPIC2 IN A 192.168.64.120
www IN CNAME LPIC2
```

**Zona inversa (reverse zone):** mapea IP → FQDN. Usa `$ORIGIN` con la IP invertida seguida de `in-addr.arpa.`, y solo registros `PTR`:
```
120 IN PTR LPIC2.example.com.
```

### 6.4 Comprobar una zona

```
named-checkzone example.com /var/named/example.com.zone
```
Respuesta `OK` = sintaxis correcta.

### 6.5 Delegación de zona

Delegar significa ceder la autoridad de una subzona a otro(s) servidor(es). Se hace añadiendo un registro `NS` para la subzona más un **registro glue** (`A`) con la IP del servidor de esa subzona, para evitar bloqueos en la resolución.

---

## 7. Herramientas de diagnóstico (troubleshooting)

| Herramienta | Uso |
|---|---|
| `host nombre` | Resolución básica; `host -t MX servidor` para ver servidores de correo |
| `dig nombre` | Consulta muy flexible y detallada; por defecto usa los servidores de `/etc/resolv.conf` |
| `dig @servidor nombre` | Dirige la consulta a un servidor DNS concreto |
| `dig +trace nombre` | Muestra el recorrido completo: root → TLD → servidor autoritativo |
| `dig MX +short dominio` | Consulta rápida de servidores de correo |
| `nslookup nombre [servidor]` | Modo no interactivo; sin argumentos entra en modo interactivo (`server X`, `exit`) |
| `nslookup -query=ns dominio` | Busca los servidores autoritativos |

### 7.1 `rndc` (Remote Name Daemon Control)

| Comando | Función |
|---|---|
| `rndc status` | Estado del demonio (CPUs, zonas cargadas, clientes, etc.) |
| `rndc reload` | Recarga configuración y zonas |
| `rndc reload zone` | Recarga una única zona |
| `rndc reconfig` | Recarga configuración y solo las zonas nuevas |
| `rndc stop` | Guarda actualizaciones pendientes y para el servidor |
| `rndc halt` | Para el servidor sin guardar actualizaciones pendientes |
| `rndc flush` | Vacía toda la caché |
| `rndc flushname nombre` | Vacía la caché de un nombre concreto |
| `rndc querylog on\|off` | Activa/desactiva el log de consultas |

Importante: `rndc` puede **parar** BIND pero **no puede arrancarlo ni reiniciarlo** (`rndc restart` no está implementado) — para eso hay que usar `systemctl`/`service`.

---

## 8. Seguridad de BIND

Buenas prácticas recogidas en el libro:

- Mantener BIND actualizado (no hay calendario fijo: actualizar en cuanto salga una versión nueva por las amenazas constantes).
- Ejecutar el demonio `named`/`bind9` como usuario no root (ya es así por defecto: usuario `named` o `bind`).
- Ocultar la información de versión de BIND a clientes externos.
- No ofrecer otros servicios importantes (Apache, CUPS...) en el mismo servidor DNS.
- Usar vistas (`view`) distintas para clientes internos y externos.
- Desactivar las actualizaciones dinámicas si no se necesitan (`allow-update { none; };`).
- Desactivar globalmente las transferencias de zona (`allow-transfer {"none";};` en `options`) y activarlas solo donde haga falta, a nivel de zona concreta (una directiva de zona sobreescribe la opción global).
- Separar servidores que cumplen varias funciones DNS distintas.
- Usar DNSSEC, proteger las transacciones con TSIG, y considerar DANE.

### 8.1 Chroot jail

Aísla el proceso BIND en un directorio raíz "falso" para que, aunque sea comprometido, no pueda acceder al resto del sistema.

| Elemento | Detalle |
|---|---|
| Directorio raíz típico | `/chroot/named/` o `/chroot/bind/` |
| Paquete que automatiza el proceso (Red Hat) | `bind-chroot` |
| Script de configuración | `/usr/libexec/setup-named-chroot.sh /var/named/chroot on` |
| Servicio tras activarlo | `named-chroot` (en vez de `named`) |
| Nueva ubicación de los ficheros | `/var/named/chroot/var/named/` |

### 8.2 DNSSEC y TSIG

| Concepto | Descripción |
|---|---|
| ZSK (Zone Signing Key) | Clave usada para firmar digitalmente los registros de una zona |
| KSK (Key Signing Key) | Clave usada para firmar la ZSK |
| Chain of trust | Cadena en la que una zona superior firma la clave de una zona inferior |
| DNSKEY | Registro donde se guarda la KSK |
| TSIG | Firma las transacciones (transferencias de zona, actualizaciones dinámicas) para autenticarlas |
| `dnssec-keygen` | Genera claves TSIG o DNSSEC |
| `dnssec-signzone` | Firma manualmente una zona (si no se usa firma automática vía `auto-dnssec`/`inline-signing`) |
| `dnssec-dsfromkey` | Genera el registro para la zona padre en la cadena de confianza |

---

## 9. Resumen de diferencias Debian vs Red Hat en este capítulo

| Aspecto | Debian/Ubuntu | Red Hat |
|---|---|---|
| Paquetes | `bind9`, `bind9utils` | `bind`, `bind-utils` |
| Servicio | `bind9` | `named` |
| Usuario del proceso | `bind` | `named` |
| Fichero principal | `/etc/bind/named.conf` | `/etc/named.conf` |
| Directorio de zonas | `/etc/bind/` | `/var/named/` |
| Resto del capítulo (registros de recursos, `rndc`, `dig`/`host`/`nslookup`, DNSSEC/TSIG, chroot) | Igual | Igual |

---

*Documento generado a partir del Capítulo 8 ("Directing DNS") del libro LPIC-2: Linux Professional Institute Certification Study Guide (Bresnahan & Blum).*
