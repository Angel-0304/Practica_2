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
- `scripts/setup-webserver.sh`
- `documentacion/` — informe completo en PDF
