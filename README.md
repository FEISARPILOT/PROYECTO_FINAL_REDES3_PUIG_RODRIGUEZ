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
