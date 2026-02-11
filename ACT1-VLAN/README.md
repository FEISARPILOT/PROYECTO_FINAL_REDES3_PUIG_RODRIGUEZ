# GNS3 Lab: VLANs + Router-on-a-Stick + DHCP + NAT (Internet) + ACLs + Seguridad L2

Este repositorio contiene **dos ficheros finales de configuración** listos para cargar en un laboratorio de GNS3. Con ellos dejamos montado un escenario típico de campus pequeño: dos VLANs para dos clientes, un router haciendo *router-on-a-stick*, asignación de IPs por DHCP, salida a Internet mediante NAT/PAT y control de tráfico con ACLs. Además, en el switch añadimos medidas de seguridad de capa 2 (DHCP Snooping y Port-Security) y configuramos la consola para que los logs se vean con claridad.

## 1. Ficheros incluidos y utilidad de cada uno

### 1.1. `IOU1_startup-config.cfg` (Switch L2)
Este fichero configura el switch IOU1 para que actúe como conmutador de acceso con segmentación por VLAN y seguridad L2.

Lo más importante que deja hecho es:
- Crea **VLAN 10** (Cliente A) y **VLAN 11** (Cliente B).
- Configura **Ethernet0/0** como **trunk 802.1Q** hacia el router (enlace “router-on-a-stick”).
- Configura **Ethernet0/1** como puerto de acceso en **VLAN 10** y **Ethernet0/2** como acceso en **VLAN 11**, con `spanning-tree portfast`.
- Activa **DHCP Snooping** para evitar servidores DHCP “rogue”, marcando el trunk hacia el router como **trusted** y desactivando Option 82 para evitar incompatibilidades en laboratorio.
- Activa **Port-Security con sticky MAC** en los puertos de acceso para que una MAC “válida” quede fijada y cualquier otra sea detectada como violación.
- Activa `logging synchronous` en consola para que los mensajes (por ejemplo de Port-Security) se impriman ordenados y sea fácil capturarlos.

Nota importante: este `startup-config` ya incluye líneas de sticky MAC “fijadas” (por ejemplo `switchport port-security mac-address sticky 0c36.ae33.0000`). Si tus clientes tienen otra MAC, el puerto puede entrar en violación. En el apartado “Ajustes rápidos” explicamos cómo limpiar o adaptar esto.

### 1.2. `R1_i1_startup-config.cfg` (Router)
Este fichero configura el router R1 como gateway de ambas VLANs y salida a Internet.

Lo más importante que deja hecho es:
- Configura **FastEthernet0/0** hacia el **NAT de GNS3** con **IP por DHCP** y marcado como `ip nat outside`.
- Configura **FastEthernet1/0** como enlace físico trunk sin IP y crea dos subinterfaces:
  - **Fa1/0.10**: gateway de VLAN 10 con `encapsulation dot1Q 10` e IP **192.168.10.1/24**.
  - **Fa1/0.11**: gateway de VLAN 11 con `encapsulation dot1Q 11` e IP **192.168.11.1/24**.
- Levanta **DHCP** en el router con una **pool por VLAN**, entregando gateway y DNS.
- Instala la **ruta por defecto** aprendida desde la WAN (`ip route 0.0.0.0 0.0.0.0 dhcp`).
- Configura **NAT/PAT (overload)** para que las redes internas salgan a Internet usando la IP de la WAN.
- Aplica **ACLs inbound** por VLAN para:
  - Permitir DHCP (cliente 68 → servidor 67).
  - Bloquear comunicación entre VLAN 10 y VLAN 11 (segmentación).
  - Permitir el resto del tráfico hacia Internet.

## 2. Topología esperada en GNS3

### 2.1. Conexiones
- **NAT1 (GNS3)** ↔ **R1 Fa0/0**
- **R1 Fa1/0** ↔ **IOU1 Et0/0** (trunk)
- **IOU1 Et0/1** ↔ **Cliente A (Debian)**
- **IOU1 Et0/2** ↔ **Cliente B (Debian)**

### 2.2. Direccionamiento
- VLAN 10: `192.168.10.0/24` (gateway `192.168.10.1`)
- VLAN 11: `192.168.11.0/24` (gateway `192.168.11.1`)
- WAN R1: IP por DHCP del NAT de GNS3 (típicamente `192.168.122.0/24`)

## 3. Cómo cargar las configuraciones

### 3.1. Opción recomendada: cargar como `startup-config`
En GNS3, en cada dispositivo, carga el fichero correspondiente como configuración de arranque:
- IOU1: `IOU1_startup-config.cfg`
- R1: `R1_i1_startup-config.cfg`

Después reinicia el dispositivo o haz `write memory` si lo aplicaste manualmente.

### 3.2. Opción manual: pegar por consola
Si pegas por consola, hazlo por bloques (GNS3 puede recortar portapapeles). El orden importa: primero IOU1, luego R1, y al final los clientes.

## 4. Verificación rápida del escenario

### 4.1. Verificación en IOU1 (switch)
Comandos recomendados:
- `show vlan brief`
- `show interfaces trunk`
- `show ip dhcp snooping`
- `show port-security interface ethernet0/1`
- `show port-security interface ethernet0/2`

### 4.2. Verificación en R1 (router)
Comandos recomendados:
- `show ip interface brief`
- `show ip route`
- `show ip dhcp pool`
- `show ip dhcp binding`
- `show ip nat translations`
- `show ip nat statistics`
- `show access-lists`

### 4.3. Verificación en Debian (Clientes)
En cada cliente (ajusta el nombre de interfaz si no es `ens4`):
- DHCP: `sudo dhclient -r ens4 && sudo dhclient -v ens4`
- IP y rutas: `ip a`, `ip route`
- Gateway: `ping -c 3 192.168.10.1` (Cliente A) / `ping -c 3 192.168.11.1` (Cliente B)
- Internet por IP: `ping -c 3 8.8.8.8`
- DNS: `getent hosts google.com` y `ping -c 3 google.com`

## 5. Ajustes rápidos y problemas típicos

### 5.1. Port-Security: puertos en err-disable tras cambiar MAC o cambiar de cliente
Si provocamos una violación, el puerto puede quedar en **Secure-shutdown**. Se recupera así:
- En IOU1:
  - `conf t`
  - `interface e0/1`
  - `shutdown`
  - `no shutdown`
  - `end`

Si el problema es que el switch se quedó con una sticky MAC antigua y queremos que reaprenda:
- En IOU1 (sobre el puerto afectado):
  - `conf t`
  - `interface e0/1`
  - `no switchport port-security mac-address sticky`
  - `switchport port-security mac-address sticky`
  - `shutdown`
  - `no shutdown`
  - `end`

### 5.2. DHCP Snooping: sintaxis de VLAN
En algunas imágenes IOU la sintaxis funciona mejor como rango:
- Correcto: `ip dhcp snooping vlan 10-11`

### 5.3. Trunk encapsulation dot1q
En algunos switches el comando `switchport trunk encapsulation dot1q` puede no existir (dot1q ya es el default). Si da error, se elimina esa línea y se mantiene `switchport mode trunk`.

### 5.4. Mensajes de “duplex mismatch”
Si aparece aviso de CDP por dúplex, en este lab se forzó `duplex full` en los extremos del enlace para estabilizarlo.

## 6. Seguridad y credenciales
Los ficheros incluyen `enable secret` y un usuario local `admin` con privilegio 15. Si vas a publicar el repo en abierto, lo normal es reemplazar secretos por valores de laboratorio o cambiarlos tras importar el escenario.

## 7. Objetivo didáctico del repositorio
La idea es que podamos levantar un entorno completo y reproducible en pocos minutos, y luego validar de forma técnica cada componente:
segmentación L2 (VLANs), trunk 802.1Q, routing por subinterfaces, DHCP por VLAN, salida a Internet con NAT/PAT, filtrado con ACLs y seguridad L2 con DHCP Snooping y Port-Security, incluyendo evidencia por logs en consola cuando ocurre una violación.
