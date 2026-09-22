# LPIC-2 · Capítulo 12: Setting Up System Security
### Recopilación de comandos y archivos de configuración

Objetivos del examen cubiertos: 212.1 (Configuring a router), 212.3 (Secure shell - SSH), 212.4 (Security tasks), 212.5 (OpenVPN)

---

## 1. Direcciones privadas y NAT

### 1.1 Rangos de IP privadas (IPv4, según IANA)

| Rango |
|---|
| 10.0.0.0 – 10.255.255.255 |
| 172.16.0.0 – 172.31.255.255 |
| 192.168.0.0 – 192.168.255.255 |

Estas direcciones no son enrutables desde fuera de la red local, lo que protege frente a atacantes externos.

### 1.2 IPv6: direcciones link local

Empiezan siempre por `fe80`, con la parte de host derivada normalmente de la MAC, para garantizar unicidad sin configuración manual.

### 1.3 NAT (Network Address Translation)

Traduce las IPs privadas de la red local a una única IP pública para salir a Internet. El servidor NAT mantiene una tabla dinámica de conexiones activas para saber a qué cliente interno reenviar cada paquete entrante. Los clientes internos pueden iniciar conexiones salientes, pero los hosts externos no pueden iniciar conexiones entrantes directamente (salvo mediante **redirección de puertos / port forwarding**, que expone un servicio interno concreto — usar con precaución).

---

## 2. Firewalls con `iptables`

### 2.1 Cadenas (chains) del kernel

| Cadena | Función |
|---|---|
| `PREROUTING` | Procesa paquetes antes de la decisión de enrutado |
| `INPUT` | Paquetes destinados al propio sistema local |
| `FORWARD` | Paquetes que se reenvían a otro sistema |
| `POSTROUTING` | Paquetes salientes hacia otros sistemas, tras el filtro FORWARD |
| `OUTPUT` | Paquetes que salen del propio sistema local |

### 2.2 Tablas (tables)

| Tabla | Función |
|---|---|
| `FILTER` | Reglas para permitir/bloquear paquetes |
| `MANGLE` | Reglas para modificar características de los paquetes |
| `NAT` | Reglas para cambiar direcciones de los paquetes |

### 2.3 Opciones básicas de `iptables` (Tabla 12.1)

| Opción | Función |
|---|---|
| `-A cadena regla` | Añade una regla al final de la cadena |
| `-D cadena regla` | Elimina una regla |
| `-F [cadena]` | Vacía todas las reglas de una cadena (o de todas si no se especifica) |
| `-I cadena índice regla` | Inserta una regla en una posición concreta |
| `-L [cadena]` | Lista las reglas |
| `-P cadena destino` | Define la política por defecto de la cadena |
| `-R cadena índice regla` | Sustituye una regla en una posición |
| `-S [cadena]` | Lista las reglas en detalle |
| `-t tabla` | Especifica la tabla a la que aplica la regla |

### 2.4 Políticas por defecto

| Política | Efecto |
|---|---|
| `ACCEPT` | Pasa el paquete a la siguiente cadena |
| `DROP` | Descarta el paquete sin avisar |
| `LOG` | Registra el paquete y lo pasa a la siguiente cadena |
| `REJECT` | Descarta el paquete y envía un aviso de rechazo al origen |

Ejemplo: bloquear todo el tráfico saliente:
```
sudo iptables -t filter -P OUTPUT DROP
```

### 2.5 Opciones de una regla (Tabla 12.2)

| Opción | Función |
|---|---|
| `-d dirección` | Dirección de destino |
| `-g cadena` | Salta a otra cadena |
| `-i nombre` | Interfaz de entrada |
| `-j destino` | Acción a tomar (`ACCEPT`, `DROP`, `LOG`, `REJECT`) |
| `-o nombre` | Interfaz de salida |
| `-p protocolo` | Protocolo (tcp, udp, icmp) |
| `-s dirección` | Dirección de origen |
| `--sport` / `--dport` | Puerto de origen/destino |

Ejemplo — bloquear todo el tráfico entrante de una IP:
```
sudo iptables -A INPUT -s 10.0.1.25 -j REJECT
```

Ejemplo — bloquear un puerto TCP concreto de salida:
```
sudo iptables -A OUTPUT -p tcp --dport 1234 -j DROP
```

### 2.6 Persistencia de las reglas

Las reglas de `iptables` **no sobreviven a un reinicio**. Hay que guardarlas y restaurarlas:
```
sudo iptables-save > myrules.txt
sudo iptables-restore < myrules.txt
```
(Alternativa: un script con los comandos `iptables` individuales, ejecutado al arrancar el sistema.)

Nota: `iptables` gestiona solo IPv4; para IPv6 se usa el comando equivalente `ip6tables`.

---

## 3. OpenSSH

### 3.1 Componentes

| Programa | Función |
|---|---|
| `sshd` | Servidor SSH, escucha conexiones entrantes |
| `ssh` | Cliente SSH, se conecta a un servidor remoto |

### 3.2 Fichero de configuración del servidor: `/etc/ssh/sshd_config`

Formato: `opción valor` (la mayoría admiten `yes`/`no`).

**Opciones comunes (Tabla 12.3):**

| Opción | Función |
|---|---|
| `Protocol` | Nivel de protocolo de cifrado soportado (2 es el preferido y más seguro) |
| `PasswordAuthentication` | Permite autenticación por contraseña de texto |
| `PubkeyAuthentication` | Permite autenticación por certificado/clave |
| `AllowUsers` | Lista (separada por espacios) de usuarios permitidos |
| `DenyUsers` | Lista de usuarios bloqueados |
| `PermitRootLogin` | Permite el login directo del usuario root |
| `X11Forwarding` | Permite tunelizar aplicaciones gráficas X11 |
| `AllowTcpForwarding` | El servidor acepta protocolos tunelizados |

Tras cualquier cambio hay que reiniciar el servicio `sshd` para que surta efecto. Para X11 forwarding también hay que activar `ForwardX11 yes` en el `ssh_config` del cliente.

### 3.3 Uso del cliente

```
ssh usuario@servidor
ssh servidor          # usa el mismo usuario que en el cliente
```

### 3.4 Autenticación por clave pública/privada

```
ssh-keygen -q -t rsa -f ~/.ssh/id_rsa -C '' -N ''
```
Genera `id_rsa` (clave privada) e `id_rsa.pub` (clave pública, con cifrado RSA).

Copiar la clave pública al servidor:
```
cat id_rsa.pub >> ~/.ssh/authorized_keys
```
(`>>` añade sin sobrescribir; si se accede desde varios clientes, cada uno genera su par de claves y añade su pública al mismo `authorized_keys`).

Al conectar, `ssh` compara la clave privada local con la pública del servidor; si coinciden, no pide contraseña.

---

## 4. OpenVPN

### 4.1 Concepto

Crea un **túnel cifrado punto a punto** entre dos redes separadas por una red pública, funcionando como una interfaz de red más (`tun0`). Aunque la conexión es entre iguales, OpenVPN llama a un extremo "servidor" y al otro "cliente" (en despliegues multipunto, un sitio es el servidor y el resto clientes).

### 4.2 Instalación

```
sudo apt-get install openvpn     # Debian
yum install openvpn              # Red Hat
```

### 4.3 Ficheros de configuración

| Fichero | Uso |
|---|---|
| `/etc/openvpn/server.conf` | Lado servidor |
| `/etc/openvpn/client.conf` | Lado cliente |

(También se pueden pasar opciones por línea de comandos, precedidas de `--`.)

**Opciones destacadas (Tabla 12.4):**

| Opción | Función |
|---|---|
| `config` | Fichero(s) de configuración adicionales |
| `dev` | Nombre del dispositivo virtual del túnel |
| `nobind` | Crea el túnel sin dirección/puerto local fijo |
| `ifconfig` | IPs de los dos extremos del túnel |
| `secret` | Fichero de clave estática de cifrado |

### 4.4 Métodos de cifrado

| Método | Descripción |
|---|---|
| Clave estática | Servidor y cliente comparten el mismo fichero de clave: `openvpn --genkey --secret secret.key` |
| Clave pública | Cada extremo genera su par de claves, firmadas por una CA (vía OpenSSL, ver Capítulo 9). Scripts de ayuda: `vars`, `build-ca`, `build-key-server`, `build-key`, `build-dh` |

Ejemplo mínimo de `server.conf` (clave estática):
```
dev tun
ifconfig 192.168.1.10 10.0.1.1
keepalive 10 60
ping-timer-rem
persist-tun
persist-key
secret secret.key
```

### 4.5 Arranque

```
sudo openvpn server.conf     # en el servidor
sudo openvpn client.conf     # en el cliente
```
Al conectar con éxito se crea el dispositivo `tun0`, comprobable con `ifconfig`.

---

## 5. Escaneo de puertos y herramientas de auditoría

| Herramienta | Función |
|---|---|
| `telnet host puerto` | Comprueba un puerto TCP individual |
| `nc` (netcat) | Servidor/cliente de prueba (`nc -l puerto` servidor, `nc IP puerto` cliente); útil para probar reglas de firewall |
| `nmap` | Escanea rangos de puertos TCP/UDP |
| OpenVAS | Frontend web para varios escáneres de vulnerabilidades; usa **NVT** (Network Vulnerability Tests) para simular ataques reales contra el sistema escaneado |

---

## 6. Sistemas de detección de intrusiones (IDS)

Un IDS compara eventos del sistema o de la red contra reglas predefinidas de comportamiento malicioso, y puede avisar al administrador (a diferencia de un escáner de puertos, que no detecta ataques en curso).

### 6.1 fail2ban (IDS de host)

| Elemento | Detalle |
|---|---|
| Qué monitoriza | Logs del sistema (`/var/log/auth.log`, `/var/log/pwdfail`) y de aplicaciones (ej. `/var/log/apache/error.log`) |
| Qué detecta | Intentos de login fallidos repetidos desde el mismo host |
| Acción | Bloquea (ban) la IP del atacante mediante reglas de firewall |
| Fichero de configuración | `/etc/fail2ban/jail.conf` |
| Limitación | Puede generar falsos positivos y bloquear clientes legítimos (aunque el bloqueo puede configurarse para expirar) |

### 6.2 Snort (NIDS — Network Intrusion Detection System)

A diferencia de fail2ban, Snort no monitoriza el propio servidor sino el **tráfico de red**, analizándolo en tiempo real. Su eficacia depende de dónde se coloque en la red: conectado a un switch normal, solo verá el tráfico destinado a él, así que suele requerir un puerto espejo (mirror/SPAN) para ver todo el tráfico relevante.

| Elemento | Detalle |
|---|---|
| `HOME_NET` | Variable de configuración que define las direcciones locales a monitorizar |
| `EXTERNAL_NET` | Variable que define las direcciones remotas |
| Formato de dirección en reglas | `origen -> destino` (flujo dirigido), `origen <- destino`, o `origen <> destino` (bidireccional) |
| Modos de ejecución | Sniffer (vuelca los paquetes en pantalla), logging (los guarda en fichero), NIDS (analiza y genera alertas de eventos) |

---

## 7. Recursos de seguridad

| Recurso | Descripción |
|---|---|
| **US-CERT** | Operado por el Departamento de Seguridad Nacional de EE.UU.; publica avisos actualizados sobre vulnerabilidades y métodos de ataque; mantiene la **National Vulnerability Database (NVD)** |
| **SANS Institute** | No publica informes en tiempo real, pero ofrece papers de investigación que ayudan a entender el funcionamiento interno de las vulnerabilidades |
| **Bugtraq** (SecurityFocus, patrocinado por Symantec) | Lista de correo de "divulgación completa" (full disclosure): no solo reporta vulnerabilidades conocidas, sino cómo los atacantes las están explotando |

---

## 8. Resumen de diferencias Debian vs Red Hat en este capítulo

| Aspecto | Debian | Red Hat |
|---|---|---|
| Instalación de OpenVPN | `sudo apt-get install openvpn` | `yum install openvpn` |
| Resto del capítulo (`iptables`, OpenSSH, fail2ban, Snort, recursos de seguridad) | Igual | Igual |

Este último capítulo es, junto con el 2 y el 11, de los que menos diferencias reales presenta entre familias de distribuciones: casi todo el contenido (`iptables`, OpenSSH, OpenVPN, fail2ban, Snort) funciona de forma idéntica en Debian y Red Hat.

---

*Documento generado a partir del Capítulo 12 ("Setting Up System Security") del libro LPIC-2: Linux Professional Institute Certification Study Guide (Bresnahan & Blum).*

---

## Nota final

Con este capítulo se completa la recopilación de los 12 capítulos del libro. Tienes ahora un conjunto de documentos, uno por capítulo, con los comandos, archivos de configuración y diferencias Debian/Red Hat organizados de forma consistente, listos para repasar de cara al examen LPIC-2.
