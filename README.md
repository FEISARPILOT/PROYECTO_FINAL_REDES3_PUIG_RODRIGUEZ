# PROYECTO_FINAL_REDES3_PUIG_RODRIGUEZ 
Repositorio donde subiremos el Albert Rodriguez y el Joan Puig los ficheros de configuración de los dispositivos usados en cada escenario de las actividades de GNS3.

## Actividad 1 (GNS3): VLANs + Router-on-a-Stick + DHCP + NAT (Internet) + ACLs + Seguridad L2

En esta primera actividad montamos en GNS3 un escenario base de red “realista” usando un switch L2 (IOU1) y un router (R1). El objetivo es construir, a partir de dos clientes, una red segmentada por VLANs y demostrar que cada parte del camino funciona de forma verificable: la conmutación en capa 2, el enrutamiento entre VLANs, la asignación automática de direcciones y la salida a Internet con control de tráfico.

El escenario se apoya en dos ficheros de configuración:

- `IOU1_startup-config.cfg`: crea las VLAN 10 y 11, asigna los puertos de acceso para Cliente A y Cliente B, configura el enlace trunk 802.1Q hacia el router y añade seguridad de capa 2 mediante DHCP Snooping y Port-Security (con sticky MAC), además de ajustar la consola para ver los logs con claridad durante las pruebas.

- `R1_i1_startup-config.cfg`: configura el router en modo router-on-a-stick con subinterfaces dot1Q para cada VLAN, actúa como servidor DHCP para ambas redes, establece la ruta por defecto hacia el NAT de GNS3 y habilita NAT/PAT para dar acceso a Internet. También aplica ACLs para permitir DHCP, permitir salida a Internet y, opcionalmente, bloquear la comunicación directa entre VLAN 10 y VLAN 11.

Como resultado, ambos clientes obtienen IP automáticamente por DHCP, pueden alcanzar su gateway, tienen conectividad a Internet mediante NAT/PAT y el laboratorio permite evidenciar políticas de seguridad (ACLs y Port-Security) con capturas y comandos de verificación.

### Topologia de red
Si queremos acceder al diagrama de la topología inicial, podemos acceder a él a través de esta URL.
 https://app.diagrams.net/#G1E1sBBJKPGhXsnZgbV_wlVQ_m8ko8uLoZ#%7B%22pageId%22%3A%22fRRjgxpdgqEocGk969YJ%22%7D


## Actividad 2 (ACT 2.1 y ACT 2.2): pfSense como firewall perimetral y NAT central

En esta actividad se centraliza el perímetro en **pfSense**:  
- **R1** sigue como **router-on-a-stick** y **gateway** de las VLANs (inter-VLAN).  
- **pfSense** pasa a ser el **firewall perimetral** y el que realiza el **NAT de salida**.

Así se evita el **doble NAT**, se concentran las políticas en un único punto y el diseño se asemeja más a un entorno real.

### Topología (resumen)
- **WAN (pfSense)** ↔ **NAT de GNS3**
- **LAN/Transit (pfSense)** ↔ **R1** (enlace de tránsito)
- **OPT1 (pfSense)**: administración/gestión (según práctica)

---

## ACT 2.1: Quitar NAT de R1 y redirigir la salida a pfSense

**Acción:** ejecutar `1_R1_Remove_NAT` en **R1**.  
**Resultado esperado:** R1 deja de traducir direcciones y su **ruta por defecto** apunta a **pfSense** por el enlace de tránsito.

- Interfaces y tabla de rutas: enlace de tránsito operativo + default route hacia pfSense.

---

## ACT 2.2: Enrutamiento estático y NAT en pfSense para dar Internet a las VLANs

### Pasos clave en pfSense
1. **Crear gateway interno hacia R1** (ej. `GW_R1_TRANSIT`) en la interfaz Transit/LAN.  
   - **[CAPTURA ACT2.2-01 — Insertar aquí]**

2. **Añadir rutas estáticas** hacia las VLANs (p. ej. `192.168.10.0/24` y `192.168.11.0/24`) usando `GW_R1_TRANSIT`.  
   - **[CAPTURA ACT2.2-02 — Insertar aquí]** Ruta VLAN 10  
   - **[CAPTURA ACT2.2-03 — Insertar aquí]** Ruta VLAN 11

3. **Configurar Outbound NAT** para que pfSense traduzca las VLANs hacia la **WAN** (modo automático/híbrido).  
   - **[CAPTURA ACT2.2-04 — Insertar aquí]**

4. **Reglas de firewall** en la interfaz Transit/LAN para permitir que las redes internas salgan a Internet.  
   - **[CAPTURA ACT2.2-05 — Insertar aquí]**

### Verificación (rápida)
Desde hosts en VLAN 10/11: `ping 8.8.8.8` y `traceroute` (confirmar salto por pfSense).

### Topologia de red
Si queremos acceder al diagrama de la topología inicial de cada ejercicio, podemos acceder a él a través de estas URLs.

- `ACT 2.1`: https://drive.google.com/file/d/1EWtqjvkYynzZX3Q9x9eZGoF1wKxCtHWQ/view?usp=sharing

- `ACT 2.2`: https://drive.google.com/file/d/17LD131_-_xeeBpQVkOkg6WbProJHFD3i/view?usp=sharing


