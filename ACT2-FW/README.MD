
# GNS3 Lab: pfSense como FW/NAT + Router-on-a-Stick (VLAN10/11) + Rutas estáticas + Hardening + DNS Enforcement

Este repositorio documenta y deja reproducible un escenario en **GNS3** donde un **router Cisco (R1)** realiza *router-on-a-stick* para dos VLANs (10 y 11), mientras que **pfSense** actúa como **firewall perimetral**, **gateway de Internet** y **punto central de NAT**, además de implementar controles de acceso (GUI/SSH), registro básico de eventos y medidas para **evitar el DNS Hijacking** (forzando el uso de DNS corporativo).

El laboratorio se divide en dos partes principales:

**ACT 2.1**: Migración del NAT a pfSense y enrutamiento correcto entre VLANs e Internet.  
**ACT 2.2**: Endurecimiento (hardening) + monitoreo + “DNS Enforcement” (evitar bypass de DNS).

---

## 0. Dispositivos e imágenes utilizadas

El escenario está construido con los siguientes nodos/plantillas:

- **pfSense CE 2.7.2** (Firewall)
- **Cisco 7200 (R1)** con IOS `c7200-adventerprisek9-mz.124-24.T5.image`
- **Cisco IOU L2 (IOU1)** con `i86bi-linux-l2-adventerprisek9-15.2d.bin`
- **2x Debian 12.6** (Cliente A y Cliente B)

---

## 1. Topología y direccionamiento

### 1.1. Conexiones físicas / lógicas (GNS3)

- **NAT1 (GNS3)** ↔ **pfSense em0 (WAN)**
- **pfSense em1 (LAN / tránsito)** ↔ **R1 Fa0/0**
- **R1 Fa1/0** ↔ **IOU1 e0/0** (trunk 802.1Q)
- **IOU1 e0/1** ↔ **Cliente A (VLAN 10)**
- **IOU1 e0/2** ↔ **Cliente B (VLAN 11)**
- **pfSense em2 (OPT1 / gestión)** ↔ **Cloud1 / red de administración (si aplica)**

### 1.2. Direccionamiento del laboratorio

- **Tránsito pfSense–R1 (LAN pfSense)**: `10.0.0.0/30`
  - pfSense (LAN/em1): `10.0.0.1/30`
  - R1 (Fa0/0): `10.0.0.2/30`
- **VLAN 10**: `192.168.10.0/24` (GW: `192.168.10.1`)
- **VLAN 11**: `192.168.11.0/24` (GW: `192.168.11.1`)
- **WAN pfSense**: IP por DHCP desde NAT1 (GNS3)
- **OPT1 pfSense** (gestión): `192.168.56.0/24` (según tu red de administración)

---

## 2. ACT 2.1 — Centralizar NAT en pfSense y dar salida a Internet a VLAN10/VLAN11

### 2.1. Objetivo técnico

En esta actividad, el router R1 deja de hacer NAT para evitar “doble NAT” y posibles interferencias.  
A partir de este punto, **pfSense será el único encargado del NAT**, mientras R1 se limita a rutear entre VLANs y el enlace de tránsito hacia pfSense.

El resultado esperado es:

- Los clientes de **VLAN 10** y **VLAN 11** navegan a Internet.
- R1 enruta hacia pfSense por defecto.
- pfSense conoce cómo volver a las VLANs mediante rutas estáticas hacia R1.

### 2.2. Quitar el NAT de R1 (script `1_R1_Remove_NAT`)

En R1 se ejecuta el script **`1_R1_Remove_NAT`** para eliminar la configuración de NAT/PAT previa (típicamente `ip nat inside`, `ip nat outside` y reglas `ip nat inside source ... overload`).  
Esto es imprescindible porque, si NAT permanece activo en R1, se generan escenarios de doble traducción y el troubleshooting se vuelve ambiguo (no queda claro qué dispositivo está traduciendo y dónde se rompe el retorno).

Tras ejecutar el script, se valida el estado de R1 con:

- `show ip interface brief` para confirmar interfaces/subinterfaces levantadas.
- `show ip route` para confirmar que la **ruta por defecto** apunta a **pfSense (10.0.0.1)** y que las redes `192.168.10.0/24` y `192.168.11.0/24` siguen siendo “connected” por sus subinterfaces.

### 2.3. Configurar pfSense para enrutar hacia las VLANs (Gateway + Static Routes)

Como pfSense no está conectado directamente a `192.168.10.0/24` ni a `192.168.11.0/24`, necesita saber que esas redes están “detrás” de R1.

Para ello:

1) Se crea un **Gateway en pfSense** sobre la interfaz **LAN (tránsito)** con destino `10.0.0.2` (R1).  
2) Se crean **rutas estáticas**:
   - `192.168.10.0/24` → gateway `10.0.0.2`
   - `192.168.11.0/24` → gateway `10.0.0.2`

Con esto, el tráfico de retorno desde Internet (que llega a pfSense) sabe volver hacia cada VLAN a través de R1.

### 2.4. Permitir navegación: reglas y NAT en pfSense

Una vez el enrutamiento es correcto, pfSense debe:

- Permitir el tráfico desde las VLANs (ruteadas) hacia fuera mediante reglas en la interfaz LAN (tránsito).
- Aplicar NAT de salida en WAN para que los clientes puedan comunicarse con Internet.

En este punto, se valida desde los clientes que:
- Se obtiene IP por DHCP (entregada por R1).
- Hay conectividad L3 hasta Internet (`ping 8.8.8.8`).
- Hay resolución DNS si el DNS está bien definido.

---

## 3. ACT 2.2 — Hardening + Monitoreo + DNS Enforcement (evitar DNS Hijacking)

### 3.1. Objetivo técnico

En esta actividad se mejora la postura de seguridad y control:

- Acceso a la **GUI (HTTPS)** de pfSense sin desactivar firewall, limitando por interfaz/red.
- Acceso **SSH** a pfSense restringido a un único host autorizado.
- Desactivación controlada de la regla automática **Anti-lockout**, sustituyéndola por reglas explícitas.
- Uso de logs/estados/monitores para evidenciar el comportamiento del firewall.
- Implementación de medidas para impedir que un cliente “se salte” el DNS corporativo (DNS Hijacking / DNS bypass).

---

## 4. DNS Enforcement — Evitar que el cliente use DNS no corporativo

### 4.1. Concepto: “DNS Hijacking” y “DNS bypass” en entornos corporativos

Un cliente puede intentar usar un DNS externo (por ejemplo `8.8.8.8`) en lugar del DNS corporativo (`10.0.0.1`, que en este lab es pfSense).  
Aunque eso “funcione”, rompe políticas corporativas típicas: filtrado DNS, split-DNS, trazabilidad, bloqueo de dominios, etc.

La solución consiste en dos capas:

1) **Forzar por NAT (Port Forward/Redirect)** que cualquier petición DNS (TCP/UDP 53) hacia “cualquier destino” sea redirigida a pfSense.  
2) **Controlar en DNS Resolver (Access Lists)** qué redes pueden consultar al resolver, garantizando que únicamente VLAN10/VLAN11 usen el DNS corporativo y de forma controlada.

### 4.2. Alias “VLANs_USERS” para agrupar redes

Para facilitar reglas limpias y mantenibles, se crea un **Alias** llamado `VLANs_USERS` que agrupa:

- `192.168.10.0/24` (VLAN 10)
- `192.168.11.0/24` (VLAN 11)

Este alias se reutiliza en reglas de firewall y en NAT, evitando repetir redes en múltiples sitios.

### 4.3. NAT Port Forward para redirigir DNS a pfSense (anti-bypass)

Se crea una regla de **NAT > Port Forward** con estos criterios (la idea, no el valor literal exacto):

- **Interfaz**: LAN (tránsito, porque ahí “entra” el tráfico ruteado desde VLAN10/11 hacia pfSense)
- **Protocolo**: TCP/UDP
- **Source**: `VLANs_USERS`
- **Destination**: any
- **Destination Port**: DNS (53)
- **Redirect target IP**: “This Firewall” / LAN address (pfSense)
- **Redirect target port**: 53
- **Filter rule association**: crear regla asociada

Justificación: aunque el usuario configure manualmente `8.8.8.8`, el firewall reescribe el destino y obliga a que la consulta acabe en el DNS corporativo (pfSense).  
A efectos prácticos, el cliente “cree” que consulta a 8.8.8.8, pero realmente está consultando al resolver corporativo.

### 4.4. DNS Resolver: Access Lists por VLAN

En **Services > DNS Resolver > Access Lists** se crean listas “allow” específicas:

- `VLAN 10` → allow → red `192.168.10.0/24`
- `VLAN 11` → allow → red `192.168.11.0/24`

Justificación: el resolver no queda “abierto” a cualquier red por defecto.  
Solo las redes definidas pueden consultar DNS, y queda alineado con el diseño: VLAN10 y VLAN11 son las únicas redes de usuarios.

---

## 5. Evidencias y verificación (lo que debe comprobarse)

### 5.1. Evidencia de que el cliente intenta usar DNS externo, pero no es quien resuelve realmente

Desde un cliente Debian se prueba resolución contra:

- DNS externo: `host google.com 8.8.8.8`
- DNS corporativo: `host google.com 10.0.0.1`

Si el control está bien aplicado, aunque el comando muestre que “se usa” `8.8.8.8`, la infraestructura fuerza la resolución mediante pfSense.

En pfSense se valida en **Diagnostics > States** que aparecen estados donde se observa el intento del cliente de ir a `8.8.8.8:53`, mientras que las reglas/redirect garantizan que el flujo real se gestiona conforme a política.

### 5.2. Reglas del Firewall (LAN) y justificación de cada una (en base a su comentario)

En la tabla de reglas de la interfaz LAN, se mantienen reglas explícitas con descripciones claras. La lógica típica detrás de esas reglas es:

- Permitir DNS solo hacia pfSense (This Firewall:53) para centralizar resolución y trazabilidad.
- Permitir SSH únicamente desde el host administrador autorizado (por ejemplo `192.168.10.21`) hacia pfSense.
- Permitir ICMP para diagnóstico (ping) desde redes de usuarios y/o desde R1 según necesidad operativa.
- Mantener el redirect DNS (NAT) como capa anti-bypass.

El criterio general es que cada regla exista porque cubre una necesidad operativa concreta (navegación, diagnóstico, administración) y, al mismo tiempo, reduzca superficie de ataque (no abrir SSH o DNS de forma indiscriminada).

---

## 6. Troubleshooting rápido

Si algo falla, estas comprobaciones suelen localizar el problema:

- En R1: `show ip route` y verificar default route `0.0.0.0/0` hacia `10.0.0.1`.
- En pfSense: verificar que existen rutas estáticas a `192.168.10.0/24` y `192.168.11.0/24` vía `10.0.0.2`.
- En pfSense: revisar reglas LAN y NAT Port Forward DNS.
- En clientes: confirmar gateway correcto (192.168.10.1 / 192.168.11.1) y pruebas `ping 8.8.8.8` + resolución DNS.

---

## 7. Nota de seguridad

Este repo está pensado para laboratorio. En un entorno real se recomienda:

- Cambiar credenciales por defecto.
- Limitar gestión (GUI/SSH) a una red de administración dedicada.
- Activar logging selectivo (no todo) y exportar logs a syslog central si procede.
- Usar DNS-over-TLS/HTTPS si el diseño corporativo lo requiere.

