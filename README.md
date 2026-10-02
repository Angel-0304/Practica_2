# Practica_2
Infraestructura 2

# Infraestructura 2 — VPN Site-to-Site entre Cisco y FortiGate

[Ver video demostrativo](https://itlaedudo-my.sharepoint.com/:f:/g/personal/20242356_itla_edu_do/IgDN_aKp_D9ATL6FPikZjtAoATCooH1UvsXBNc7rNldfzxk?e=N5tFAm)

## Objetivos

- Comunicar al usuario con el servidor a través del enlace VPN.
- Comprobar que la comunicación solo fluye si la VPN está activa.
- Configurar red, NAT y VPN en el FortiGate (GUI) y en el router Cisco (CLI).

## Cambios respecto a la Infraestructura 1

Se clonó el laboratorio anterior y el FG-USER se reemplazó por un router Cisco (R-USER) con las mismas IPs, así que el switch, el PC y el servidor no necesitaron cambios de red. Para el router se usó la imagen L3-ADVENTERPRISEK9, porque la de capa 2 no soporta IPsec.

![Topología lógica](diagramas/topologia-logica.png)

| Equipo | Plataforma | Función |
|---|---|---|
| ISP | Cisco IOL | IPs públicas |
| R-USER | Cisco IOL L3-ADVENTERPRISEK9 15.4-2T | VLAN 10 (router-on-a-stick), DHCP, NAT y VPN |
| FG-Server | FortiGate VM64-KVM v6.4.0 | LAN /28, NAT y VPN |
| SW-Usuarios | Cisco IOL L2 | Trunk y acceso en VLAN 10 |
| PC-Usuario | Docker pnetlab/ubuntu_sv | Pruebas |
| Web-Server | Ubuntu Server 20.04 | Apache con HTTPS |

## Puntos clave del Cisco

**NAT con exclusión de la VPN.** Si el tráfico hacia el servidor se traduce, su origen deja de coincidir con la ACL de la VPN y nunca se cifra:

```
ip access-list extended NAT-LAN
 deny   ip 10.23.56.0 0.0.0.127 10.23.56.128 0.0.0.15
 permit ip 10.23.56.0 0.0.0.127 any
```

**Crypto map** con transform-set `esp-des esp-md5-hmac`, PFS grupo 14 y la ACL `VPN-TRAFICO` como selector, aplicado a Ethernet0/0.

## Coincidencia de parámetros

| Parámetro | FG-Server | R-USER |
|---|---|---|
| IKE / modo | IKEv1, main | IKEv1, main |
| Phase 1 | des-md5 | des / md5 |
| Grupo DH | 14, 5 | 14 |
| Vida Phase 1 | 86400 s | 86400 s |
| Phase 2 | des-md5 | esp-des esp-md5-hmac |
| PFS | grupos 14 y 5 | group14 |
| Vida Phase 2 | 43200 s | 43200 s |
| Selectores | 10.23.56.128/28 ↔ 10.23.56.0/25 | 10.23.56.0/25 ↔ 10.23.56.128/28 |

En el FortiGate se usó el asistente **Site to Site — Cisco**.

## Verificación

```
R-USER# show crypto isakmp sa
dst          src          state     conn-id status
20.24.23.2   20.24.56.2   QM_IDLE   1001    ACTIVE
```

```
root@PC-Usuario:/home# traceroute -n 10.23.56.130
 1  10.23.56.1
 2  * * *
 3  10.23.56.130
```

El salto 2 (FG-Server) no responde porque su mensaje ICMP sale con una IP fuera de los selectores y el Cisco lo descarta. Por eso también hay más paquetes encapsulados que desencapsulados en `show crypto ipsec sa`.

Con la interfaz del túnel deshabilitada en el FortiGate, las pruebas fallan: el NAT excluye ese tráfico, el ISP no conoce las redes privadas y la ruta blackhole descarta el regreso.

## Problemas encontrados

| Problema | Solución |
|---|---|
| Al clonar, todos los equipos arrancaron sin configuración | Reaplicar desde `running-configs/` |
| La PSK del Cisco quedó con el texto de ejemplo | `no crypto isakmp key` y volver a agregarla |
| Netplan y certificado vacíos en el servidor | Apagado con Stop en PNETLab; se agregó `sync` y `shutdown -h now` al script |
| El PC salía por la red de Docker | `ip route replace default via 10.23.56.1 dev eth1` |

## Archivos

- `running-configs/` — R-USER.txt, FG-Server.conf, ISP.txt, SW-Usuarios.txt

## Running-configs

<details>
<summary><b>ISP</b></summary>

```
ISP#show running-config
Building configuration...

Current configuration : 1304 bytes
!
version 15.4
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname ISP
!
boot-start-marker
boot-end-marker
!
no aaa new-model
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
no ip domain lookup
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
redundancy
!
interface Loopback0
 description Simula Internet
 ip address 8.8.8.8 255.255.255.255
!
interface Ethernet0/0
 description Enlace hacia R-USER e0/0
 ip address 20.24.23.1 255.255.255.252
!
interface Ethernet0/1
 description Enlace hacia FG-Server port1
 ip address 20.24.56.1 255.255.255.252
!
interface Ethernet0/2
 no ip address
 shutdown
!
interface Ethernet0/3
 no ip address
 shutdown
!
interface Ethernet1/0
 no ip address
 shutdown
!
interface Ethernet1/1
 no ip address
 shutdown
!
interface Ethernet1/2
 no ip address
 shutdown
!
interface Ethernet1/3
 no ip address
 shutdown
!
ip forward-protocol nd
!
no ip http server
no ip http secure-server
!
control-plane
!
banner motd ^CISP - Infraestructura 2 - Matricula 2024-2356^C
!
line con 0
 exec-timeout 0 0
 logging synchronous
line aux 0
line vty 0 4
 login
 transport input none
!
end
```

</details>

<details>
<summary><b>SW-Usuarios</b></summary>

```
SW-Usuarios#show running-config
Building configuration...

Current configuration : 1037 bytes
!
version 15.2
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
service compress-config
!
hostname SW-Usuarios
!
boot-start-marker
boot-end-marker
!
no aaa new-model
!
no ip domain-lookup
ip cef
no ipv6 cef
!
spanning-tree mode rapid-pvst
spanning-tree extend system-id
!
vlan internal allocation policy ascending
!
interface Ethernet0/0
 description Trunk hacia R-USER e0/1
 switchport trunk allowed vlan 10
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface Ethernet0/1
 description Access hacia PC-Usuario
 switchport access vlan 10
 switchport mode access
 spanning-tree portfast edge
!
interface Ethernet0/2
 shutdown
!
interface Ethernet0/3
 shutdown
!
ip forward-protocol nd
!
no ip http server
no ip http secure-server
!
control-plane
!
line con 0
 exec-timeout 0 0
 logging synchronous
line aux 0
line vty 0 4
 login
!
end
```

</details>

<details>
<summary><b>R-USER</b></summary>

```
R-USER#show running-config
Building configuration...

Current configuration : 2143 bytes
!
version 15.4
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname R-USER
!
boot-start-marker
boot-end-marker
!
no aaa new-model
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
ip dhcp excluded-address 10.23.56.1 10.23.56.9
ip dhcp excluded-address 10.23.56.101 10.23.56.127
!
ip dhcp pool VLAN10-USUARIOS
 network 10.23.56.0 255.255.255.128
 default-router 10.23.56.1
 dns-server 8.8.8.8
!
no ip domain lookup
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
redundancy
!
crypto isakmp policy 10
 hash md5
 authentication pre-share
 group 14
crypto isakmp key <PRE-SHARED-KEY> address 20.24.56.2
!
crypto ipsec transform-set TS-FORTIGATE esp-des esp-md5-hmac
 mode tunnel
!
crypto map CMAP-FORTIGATE 10 ipsec-isakmp
 set peer 20.24.56.2
 set security-association lifetime seconds 43200
 set transform-set TS-FORTIGATE
 set pfs group14
 match address VPN-TRAFICO
!
interface Ethernet0/0
 description WAN hacia ISP
 ip address 20.24.23.2 255.255.255.252
 ip nat outside
 ip virtual-reassembly in
 crypto map CMAP-FORTIGATE
!
interface Ethernet0/1
 description Trunk hacia SW-Usuarios
 no ip address
!
interface Ethernet0/1.10
 description VLAN 10 - Usuarios
 encapsulation dot1Q 10
 ip address 10.23.56.1 255.255.255.128
 ip nat inside
 ip virtual-reassembly in
!
interface Ethernet0/2
 no ip address
 shutdown
!
interface Ethernet0/3
 no ip address
 shutdown
!
ip forward-protocol nd
!
no ip http server
no ip http secure-server
ip nat inside source list NAT-LAN interface Ethernet0/0 overload
ip route 0.0.0.0 0.0.0.0 20.24.23.1
!
ip access-list extended NAT-LAN
 deny   ip 10.23.56.0 0.0.0.127 10.23.56.128 0.0.0.15
 permit ip 10.23.56.0 0.0.0.127 any
ip access-list extended VPN-TRAFICO
 permit ip 10.23.56.0 0.0.0.127 10.23.56.128 0.0.0.15
!
control-plane
!
banner motd ^CR-USER - Infraestructura 2 - Matricula 2024-2356^C
!
line con 0
 exec-timeout 0 0
 logging synchronous
line aux 0
line vty 0 4
 login
 transport input none
!
end
```

> En la política ISAKMP no aparecen `encryption des` ni `lifetime 86400` porque son los valores por defecto de IOS, y el running-config no muestra los valores por defecto.

</details>

<details>
<summary><b>FG-Server</b></summary>

> Extracto de la salida de `show` con las secciones configuradas en el laboratorio. Se omitieron los objetos y perfiles que FortiOS trae por defecto (servicios, certificados, perfiles de seguridad, etc.) y las líneas `uuid`.

```
FG-Server # show
#config-version=FGVMK6-6.4.0-FW-build1579-200330:opmode=1:vdom=0:user=admin
config system global
    set alias "FortiGate-VM64-KVM"
    set hostname "FG-Server"
    set timezone 04
end
config system interface
    edit "port1"
        set vdom "root"
        set ip 20.24.56.2 255.255.255.252
        set allowaccess ping
        set type physical
        set alias "WAN"
        set lldp-reception enable
        set role wan
        set snmp-index 1
    next
    edit "port2"
        set vdom "root"
        set ip 10.23.56.129 255.255.255.240
        set allowaccess ping
        set type physical
        set device-identification enable
        set lldp-transmission enable
        set role lan
        set snmp-index 2
    next
    edit "port3"
        set vdom "root"
        set mode dhcp
        set allowaccess ping https ssh http
        set type physical
        set snmp-index 3
        set defaultgw disable
    next
    edit "VPN-A-CISCO"
        set vdom "root"
        set type tunnel
        set snmp-index 7
        set interface "port1"
    next
end
config firewall address
    edit "VPN-A-CISCO_local_subnet_1"
        set allow-routing enable
        set subnet 10.23.56.128 255.255.255.240
    next
    edit "VPN-A-CISCO_remote_subnet_1"
        set allow-routing enable
        set subnet 10.23.56.0 255.255.255.128
    next
end
config firewall addrgrp
    edit "VPN-A-CISCO_local"
        set member "VPN-A-CISCO_local_subnet_1"
        set comment "VPN: VPN-A-CISCO (Created by VPN wizard)"
        set allow-routing enable
    next
    edit "VPN-A-CISCO_remote"
        set member "VPN-A-CISCO_remote_subnet_1"
        set comment "VPN: VPN-A-CISCO (Created by VPN wizard)"
        set allow-routing enable
    next
end
config vpn ipsec phase1-interface
    edit "VPN-A-CISCO"
        set interface "port1"
        set peertype any
        set net-device disable
        set proposal des-md5 des-sha1
        set comments "VPN: VPN-A-CISCO (Created by VPN wizard)"
        set wizard-type static-fortigate
        set remote-gw 20.24.23.2
        set psksecret <PRE-SHARED-KEY>
    next
end
config vpn ipsec phase2-interface
    edit "VPN-A-CISCO"
        set phase1name "VPN-A-CISCO"
        set proposal des-md5 des-sha1
        set comments "VPN: VPN-A-CISCO (Created by VPN wizard)"
        set src-addr-type name
        set dst-addr-type name
        set src-name "VPN-A-CISCO_local"
        set dst-name "VPN-A-CISCO_remote"
    next
end
config firewall policy
    edit 1
        set name "LAN-a-Internet"
        set srcintf "port2"
        set dstintf "port1"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
    next
    edit 2
        set name "vpn_VPN-A-CISCO_local"
        set srcintf "port2"
        set dstintf "VPN-A-CISCO"
        set srcaddr "VPN-A-CISCO_local"
        set dstaddr "VPN-A-CISCO_remote"
        set action accept
        set schedule "always"
        set service "ALL"
        set comments "VPN: VPN-A-CISCO (Created by VPN wizard)"
    next
    edit 3
        set name "vpn_VPN-A-CISCO_remote"
        set srcintf "VPN-A-CISCO"
        set dstintf "port2"
        set srcaddr "VPN-A-CISCO_remote"
        set dstaddr "VPN-A-CISCO_local"
        set action accept
        set schedule "always"
        set service "ALL"
        set comments "VPN: VPN-A-CISCO (Created by VPN wizard)"
    next
end
config router static
    edit 1
        set gateway 20.24.56.1
        set device "port1"
    next
    edit 2
        set device "VPN-A-CISCO"
        set comment "VPN: VPN-A-CISCO (Created by VPN wizard)"
        set dstaddr "VPN-A-CISCO_remote"
    next
    edit 3
        set distance 254
        set comment "VPN: VPN-A-CISCO (Created by VPN wizard)"
        set blackhole enable
        set dstaddr "VPN-A-CISCO_remote"
    next
end
```

> En la Phase 1 y Phase 2 no aparecen `ike-version 1`, `mode main`, `dhgrp 14 5`, `keylife 86400`, `pfs enable` ni `keylifeseconds 43200` porque son los valores por defecto de FortiOS 6.4.

</details>

- `documentacion/` — informe completo en PDF
