# Laboratorio: Direccionamiento IP, DHCP y Monitoreo SNMP con Zabbix

Implementación de una red LAN con Router y Switch Cisco IOS, servidor DHCP, comunidad SNMP de solo lectura, y un servidor **Zabbix** desplegado en Docker para monitorear ambos dispositivos vía SNMP.
---

## Tabla de contenido

- [Topología](#topología)
- [Requisitos cumplidos](#requisitos-cumplidos)
- [1. Configuración del Router (R1)](#1-configuración-del-router-r1)
- [2. Configuración del Switch (SW)](#2-configuración-del-switch-sw)
- [3. Cliente Windows 10](#3-cliente-windows-10)
- [4. Servidor Zabbix (Docker)](#4-servidor-zabbix-docker)
- [5. Verificación](#5-verificación)
- [Notas y troubleshooting](#notas-y-troubleshooting)

---

## Topología

![Topología de red](docs/topology.svg)

| Dispositivo | Interfaz | IP | Función |
|---|---|---|---|
| R1 | e0/0 | DHCP (WAN) | Salida a `Net` |
| R1 | e0/1 | `10.7.41.1/24` | Gateway LAN + servidor DHCP |
| SW | vlan1 | `10.7.41.2/24` | IP de gestión / SNMP |
| Windows 10 | e0/0 → SW | DHCP | Cliente DHCP + verificación SNMP |
| Docker (Zabbix) | e0/2 → SW | `10.7.41.10` (estática) | Servidor de monitoreo |

**Red:** `10.7.41.0/24`

---

## Requisitos cumplidos

- [x] Direccionamiento IP en Router y Switch
- [x] Servidor DHCP para la red LAN (R1)
- [x] Comunidad SNMP de solo lectura en Router y Switch
- [x] Cliente DHCP funcional en la PC Windows 10
- [x] Verificación de datos SNMP desde el cliente (MIB Browser)
- [x] Servidor Zabbix desplegado en Docker
- [x] Verificación de eventos del Router y el Switch en Zabbix

---

## 1. Configuración del Router (R1)

Archivo completo: [`configs/R1.txt`](configs/R1.txt)

```
interface e0/1
 ip address 10.7.41.1 255.255.255.0
 no shutdown
!
ip dhcp excluded-address 10.7.41.1 10.7.41.10
ip dhcp pool LAN_POOL
 network 10.7.41.0 255.255.255.0
 default-router 10.7.41.1
 dns-server 8.8.8.8
!
snmp-server community P4ssw0rdRO RO
snmp-server location ITLA-Lab
snmp-server contact admin@itla.edu.do
```

| Línea | Propósito |
|---|---|
| `ip address 10.7.41.1 255.255.255.0` | IP del router en la LAN, actúa como *default gateway* |
| `ip dhcp pool LAN_POOL` | R1 reparte IPs automáticamente a los clientes de la LAN |
| `snmp-server community P4ssw0rdRO RO` | Comunidad SNMP de **solo lectura** — permite a Zabbix leer datos sin poder modificar la configuración |
| `snmp-server location` / `contact` | Metadata que Zabbix lee y muestra vía SNMP (`sysLocation`, `sysContact`) |

---

## 2. Configuración del Switch (SW)

Archivo completo: [`configs/SW.txt`](configs/SW.txt)

```
interface vlan1
 ip address 10.7.41.2 255.255.255.0
 no shutdown
!
snmp-server community P4ssw0rdRO RO
```

El switch no requiere DHCP relay: R1 está en el mismo dominio de broadcast (topología L2 pura), así que las solicitudes DHCP de los clientes llegan directo a R1.

---

## 3. Cliente Windows 10

**Configuración DHCP:** ninguna manual — NIC en modo *"Obtener una dirección IP automáticamente"* (default).

**Verificación DHCP:**
```powershell
ipconfig /release
ipconfig /renew
ipconfig /all
```
Resultado esperado: IP dentro de `10.7.41.11–254`, máscara `/24`, gateway `10.7.41.1`.

**Verificación SNMP desde el cliente** — con [iReasoning MIB Browser](https://www.ireasoning.com/mibbrowser.shtml):

1. Address: `10.7.41.1` (o `10.7.41.2` para el switch)
2. Advanced → Version `V2c`, Read Community `P4ssw0rdRO`, Port `161`
3. OID `.1.3.6.1.2.1.1` (rama `system`) → Operations: **Walk** → Go

Resultado esperado:
```
sysDescr.0     = Cisco IOS Software...
sysContact.0   = admin@itla.edu.do
sysName.0      = R1.localdomain
sysLocation.0  = ITLA-Lab
```

---

## 4. Servidor Zabbix (Docker)

Nodo Docker en PNetLab con imagen `kalinet/zabbix:latest` (imagen *all-in-one*: servidor + frontend + agente), interfaz `eth1` conectada a `e0/2` del switch.

**Asignación de IP** (desde la consola del nodo):
```bash
ip addr add 10.7.41.10/24 dev eth1
ip route add default via 10.7.41.1
```

**Configuración de hosts en el frontend** (`http://10.7.41.10`, `Admin`/`zabbix`):

| Campo | R1 | SW |
|---|---|---|
| Host name | `R1` | `SW` |
| Interfaz SNMP | `10.7.41.1:161` | `10.7.41.2:161` |
| Template | `Template Net Cisco IOS SNMPv2` | `Template Net Cisco IOS SNMPv2` |
| Macro `{$SNMP_COMMUNITY}` | `P4ssw0rdRO` | `P4ssw0rdRO` |

---

## 5. Verificación

### 5.1 En los dispositivos IOS

```
show snmp community        ! nombre y tipo de acceso (RO)
show snmp                  ! contador "SNMP packets input" confirma polling activo
show ip dhcp binding        ! IPs entregadas por DHCP
```

### 5.2 En Zabbix

**Disponibilidad SNMP** — `Configuration → Hosts`: ícono SNMP en verde junto a `R1` y `SW`.

**Datos recolectados** — `Monitoring → Latest data`, filtrado por host:

| Host | Ítem | Valor obtenido |
|---|---|---|
| R1 | System location | `ITLA-Lab` |
| R1 | System contact | `admin@itla.edu.do` |
| R1 | System description | `Cisco IOS Software...` |
| SW | System name | `SW` |

Estos valores fueron configurados manualmente por CLI y leídos por Zabbix vía SNMP — confirma la recolección de extremo a extremo.

**Eventos detectados** — `Monitoring → Problems`:

Al ejecutar `shutdown` / `no shutdown` sobre `e0/0` en R1, los contadores SNMP de errores de interfaz (`ifInErrors`/`ifOutErrors`) se resetean y generan actividad temporal, detectada por el trigger `High error rate (>2 for 5m)` tanto en R1 como en SW (interfaz compartida del enlace):

```
06:18:03  R1   Interface Et0/0(WAN-Net): High error rate (>2 for 5m)
06:18:04  SW   Interface Et0/1(): High error rate (>2 for 5m)
```

Esto confirma que Zabbix monitorea el estado de las interfaces en tiempo real, no solo datos estáticos.

---

## Notas y troubleshooting

- **`e0/1` de R1 es el enlace hacia el switch**: bajarla no afecta la comunicación entre Windows10, SW y Docker (todos están en el mismo dominio L2); solo afecta la disponibilidad de R1 mismo hacia Zabbix. Para pruebas de interfaz sin ese efecto colateral, se usó `e0/0` (WAN).
- **Windows no trae cliente SNMP por defecto** (`snmpwalk` no existe en PowerShell) — se usó iReasoning MIB Browser en lugar de Net-SNMP CLI.
- **Imagen `kalinet/zabbix:latest`**: no es la imagen oficial de Zabbix, pero es *all-in-one* (server + web + agente en un contenedor), suficiente para este laboratorio. Credenciales por defecto: `Admin` / `zabbix`.
