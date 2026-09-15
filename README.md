Markdown
## GNS3 - VLANs y DHCP Relay
Lab GNS3 con Router-on-a-Stick...

![topologia](TopologiaGns3.png)

Lab GNS3 con Router-on-a-Stick. 2 VLANs sobre trunk dot1q: VLAN 100 (192.168.1.0/24) para servidores con DHCP en 192.168.1.10 y VLAN 200 (192.168.2.0/24) para clientes. Usa ip helper-address para dar DHCP entre VLANs.

## CONFIGURACION-ROUTERS

## Router R3 - Router-on-a-Stick
Este es el que hace toda la chamba.

Encendimos la interfaz física Gig0/0 con no shutdown
Creamos 2 subinterfaces:
G0/0.100 -> VLAN 100 Servidores
G0/0.200 -> VLAN 200 Clientes
A cada una le pusimos encapsulation dot1Q [vlan]
Le asignamos IP:
192.168.1.1 /24 para la VLAN 100
192.168.2.1 /24 para la VLAN 200
Lo más importante: En la subinterfaz de clientes (200) le metimos el ip helper-address 192.168.1.10 para que reenvíe las peticiones DHCP al servidor que está en la otra VLAN.
Con eso el router rutea entre VLANs y hace de intermediario DHCP.
## Configuración Realizada
### Router R3 (Router-on-a-Stick)

| Subinterfaz | VLAN | IP | Función |
|---|---|---|---|
| G2/0 | - | no IP | Trunk físico hacia el switch |
| G2/0.100 | 100 | 192.168.1.1/24 | Gateway de Servidores |
| G2/0.200 | 200 | 192.168.2.1/24 | Gateway de Clientes + DHCP Relay |

```RT-03
conf t
 hostname RT-03
 interface Se1/1
  ip address 10.0.2.2 255.255.255.252
  clock rate 64000
  no shutdown
  exit
 interface Gig2/0.100
  encapsulation dot1Q 100
  ip address 192.168.1.1 255.255.255.0
  no shutdown
  exit
 interface Gig2/0.200
  encapsulation dot1Q 200
  ip address 192.168.2.1 255.255.255.0
  ip helper-address 192.168.1.10
  no shutdown
  exit
 ip route 192.168.3.0 255.255.255.0 10.0.2.1
 exit
 copy running-config startup-config
```
## Router R1 - El de las VPS
Este es el que da internet a tus VPCs de GNS3.

Encendimos la interfaz serial Se1/0 con no shutdown para conectarnos al R2 del medio.

Configuramos la interfaz Gig2/0 como la puerta de enlace de tus VPS.

Le asignamos IP:
192.168.3.1 /24 para toda la red de VPS 192.168.3.0

Lo más importante: En la Gig2/0 le metimos el ip helper-address 192.168.1.10 para que todas las peticiones DHCP de tus VPCs se vayan hasta el servidor que está en la VLAN 100 del otro lado.

Con eso R1 les da salida a las VPS y pide las IPs al servidor central.
``` RT-01
conf term
hostname RT-01
interface Ser1/0
ip address 10.0.1.1 255.255.255.252
clock rate 64000
no shutdown
exit
interface Gig2/0
ip address 192.168.3.1 255.255.255.0
ip helper-address 192.168.1.10
no shutdown
exit
ip route 192.168.1.0 255.255.255.0 10.0.1.2
ip route 192.168.2.0 255.255.255.0 10.0.1.2
copy running-config startup-config
```
## Router R2 - El del Centro
Este es el que hace toda la conexión, no da IPs a nadie.

Encendimos las dos seriales Se1/0 y Se1/1 con no shutdown

A cada una le pusimos IP:
10.0.1.2 /30 para hablar con R1
10.0.2.1 /30 para hablar con R3

Este no lleva subinterfaces ni helper-address porque no tiene LANs.

Lo más importante: Le metimos las 3 rutas estáticas para que R1 pueda llegar a las VLANs y R3 pueda llegar a las VPS. Si no, no se hablan.
```RT-02
conf term
hostname RT-02
interface Se1/0
ip address 10.0.1.2 255.255.255.252
clock rate 64000
no shutdown
exit
interface Se1/1
ip address 10.0.2.1 255.255.255.252
clock rate 64000
no shutdown
exit
ip route 192.168.3.0 255.255.255.0 10.0.1.1
ip route 192.168.1.0 255.255.255.0 10.0.2.2
ip route 192.168.2.0 255.255.255.0 10.0.2.2
exit
copy running-config startup-config
```
## Tabla de Enrutamiento

Router	Red que Enruta	Por Donde Sale	Hacia Donde Va
R1 - VPS	192.168.3.0/24	Gig2/0	Red local VPS
	192.168.1.0/24	Se1/0 -> 10.0.1.2	VLAN 100 DataCenter
	192.168.2.0/24	Se1/0 -> 10.0.1.2	VLAN 200 Docker
R2 - Centro	10.0.1.0/30	Se1/0	Enlace R1-R2
	10.0.2.0/30	Se1/1	Enlace R2-R3
	192.168.3.0/24	Se1/0 -> 10.0.1.1	VPS
	192.168.1.0/24	Se1/1 -> 10.0.2.2	VLAN 100
	192.168.2.0/24	Se1/1 -> 10.0.2.2	VLAN 200
R3 - VLANs	192.168.1.0/24	Gig2/0.100	VLAN 100 Servidores
	192.168.2.0/24	Gig2/0.200	VLAN 200 Clientes
	192.168.3.0/24	Se1/1 -> 10.0.2.1	VPS
