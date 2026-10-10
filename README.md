# Práctica #3 — Infraestructura #2: FortiGate, VPN IPsec y Jump Server con RDP RemoteApp

- **Asignatura:** Seguridad de Redes
- **Estudiante:** Luis Roble
- **Matrícula:** 2025-2168

Video: <https://youtu.be/XXXXXXXXXXX>

---

## 1. Objetivo

En esta práctica monté un escenario con un **FortiGate** (toda la configuración y la demostración las hice por GUI), un **equipo de red Cisco**, un **ISP** con IPs públicas y dos servidores: un **Web Server** (sistema de caja) y un **Jump Server**, cada uno en su propia LAN `/29`.

Lo que quise lograr:

- Una **VPN IPsec Remote-Site** entre el cliente y el servidor que **solo me da acceso al Jump Server**.
- Que el **Jump Server** solo llegue a estos servicios del Web Server: **HTTPS, RDP y SSH**.
- Dos usuarios en la **VLAN 10** (`/25`) con DHCP y permisos distintos:
  - **Sin privilegios:** solo tiene publicado el servicio **Web** en el Jump Server, y le puse políticas explícitas que le restringen el acceso SSH a los servidores.
  - **Con privilegios:** tiene publicados **Web, PuTTY y RDP** en el Jump Server.
- El servicio **RDP RemoteApp** y el **RDP RemoteApp Web Client** configurados en el Jump Server.

---

## 2. Topología

![Topología](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/Topologia.png)

### Diagrama lógico

```mermaid
flowchart LR
    INET["Internet / NAT"]
    ISP["ISP<br/>IPs públicas"]
    FG["FortiGate<br/>(GUI)"]
    CISCO["Equipo Cisco<br/>(gateway servidores)"]
    SWU["SW-USERS<br/>VLAN 10"]
    SWV["SW-VPN"]
    UNP["Usuario sin privilegios<br/>VLAN 10 / DHCP"]
    UP["Usuario con privilegios<br/>VLAN 10 / DHCP"]
    VPNC["Cliente VPN<br/>IPsec Remote-Site"]
    JUMP["Jump Server<br/>/29<br/>RDP RemoteApp + Web Client"]
    WEB["Web Server<br/>/29<br/>HTTPS / RDP / SSH"]

    INET --- ISP
    ISP --- FG
    ISP --- CISCO
    FG ---|"trunk 802.1Q VLAN 10"| SWU
    SWU --- UNP
    SWU --- UP
    FG --- SWV
    SWV --- VPNC
    CISCO --- JUMP
    CISCO --- WEB
    VPNC -. "VPN: solo Jump Server" .-> JUMP
    UNP -. "solo Web" .-> JUMP
    UP -. "Web / PuTTY / RDP" .-> JUMP
    JUMP -. "HTTPS / RDP / SSH" .-> WEB
```

---

## 3. Direccionamiento

| Equipo | Interfaz | Dirección |
| --- | --- | --- |
| ISP | hacia FortiGate | `[IP pública]/[máscara]` |
| ISP | hacia Cisco | `[IP pública]/[máscara]` |
| FortiGate | WAN | `[IP pública]/[máscara]` |
| FortiGate | VLAN 10 (usuarios) | `[IP]/25` |
| FortiGate | VPN / LAN | `[IP]/[máscara]` |
| Equipo Cisco | WAN | `[IP pública]/[máscara]` |
| Equipo Cisco | LAN Web Server | `[IP]/29` |
| Equipo Cisco | LAN Jump Server | `[IP]/29` |
| Web Server | NIC | `[IP]/29` |
| Jump Server | NIC | `[IP]/29` |
| Usuarios VLAN 10 | DHCP | `[rango]/25` |
| Clientes VPN | pool | `[rango]` |

---

## 4. Herramientas

- [GNS3 / EVE-NG]
- FortiGate VM [versión], configurado por GUI
- Cisco IOS [versión] (equipo de red e ISP) y switches
- Windows Server para el Jump Server (RDP RemoteApp + Web Client)
- Ubuntu para el Web Server y los usuarios
- PuTTY, cliente RDP y cliente VPN

---

## 5. Configuración

### 5.1 ISP

El ISP entrega las IPs públicas y tiene rutas hacia las redes de la práctica.

![ISP - show ip route e interface brief](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/ISP-%20SH%20IP%20ROUTE%20Y%20INTERFACE%20BRIEF.png)

Config completa: [`Configs/ISP-2168.txt`](Configs/ISP-2168.txt)

### 5.2 Switch de usuarios (VLAN 10)

Creé la VLAN 10 para los usuarios y dejé un trunk 802.1Q hacia el FortiGate.

![show vlan brief](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/SW-%20SH%20VLAN%20BRIEF.png)

![interface trunk](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/SW-%20INT%20TRUNK.png)

Config completa: [`Configs/SW-USERS.txt`](Configs/SW-USERS.txt)

### 5.3 Switch de VPN

![Switch VPN](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/SW%20-%20VPN.png)

Config completa: [`Configs/SW- VPN.txt`](Configs/SW-%20VPN.txt)

### 5.4 Equipo de red Cisco

Conecta las LAN de los servidores (Web Server y Jump Server, cada uno con su `/29`) con el ISP.

Config completa: [`Configs/CISCO-2168.txt`](Configs/CISCO-2168.txt)

### 5.5 FortiGate (todo por GUI)

**Interfaces y redes**

![FortiGate - interfaces](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/FG%20-%20NW%20INTERFACE.png)

**Objetos (addresses)**

![FortiGate - addresses](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/FG%20-%20address.png)

**Rutas estáticas**

![FortiGate - rutas estáticas](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/fg%20-%20static%20r.png)

**Políticas de firewall**

![FortiGate - políticas](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/fg%20-%20policy.png)

Resumen de la lógica de políticas que apliqué:

| Política | Origen → Destino | Servicio | Acción |
| --- | --- | --- | --- |
| VPN → Jump Server | VPN-REMOTE → Jump Server | [servicios] | ACCEPT |
| Jump → Web Server | Jump Server → Web Server | HTTPS, RDP, SSH | ACCEPT |
| Usuario sin privilegios → Jump | [IP] → Jump Server | Web | ACCEPT |
| Usuario con privilegios → Jump | [IP] → Jump Server | Web, PuTTY (SSH), RDP | ACCEPT |
| Deny SSH usuario sin privilegios | [IP] → Servidores | SSH | **DENY** (explícita) |

El orden importa: las reglas DENY explícitas las puse por encima de las ACCEPT generales.

**Jump Server → Web Server (HTTPS, RDP, SSH)**

![Jump Server a Web Server](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/FG%20-%20JM%20TO%20WEB%20CONF.png)

Config completa: [`Configs/FortiGate.conf`](Configs/FortiGate.conf) (sin claves ni contraseñas).

### 5.6 VPN IPsec Remote-Site

![VPN](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/vpn%20.png)

- Autenticación: [PSK / XAUTH / certificado]
- Fase 1: [versión IKE, modo y propuestas]
- Fase 2: [propuestas]
- Pool de clientes: `[rango]`
- Acceso permitido: **únicamente el Jump Server**

### 5.7 Servidores

**Web Server (sistema de caja)**, en su propia LAN `/29`.

![Página web del servidor](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/pagina%20web%20del%20sv.png)

**Jump Server**, en su propia LAN `/29`, con RDP RemoteApp y RDP RemoteApp Web Client.

![Web server desde XRDP](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/web%20server%20desde%20xrdp.png)

### 5.8 Usuarios (VLAN 10 con DHCP)

- **Sin privilegios:** solo ve publicado el servicio Web del Jump Server, y tiene políticas explícitas que le bloquean el SSH hacia los servidores.
- **Con privilegios:** tiene publicados Web, PuTTY y RDP en el Jump Server.

---

## 6. Validación

### 6.1 Usuario sin privilegios

Solo tiene acceso al servicio Web del Jump Server.

![Jump sin privilegios](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/jmplab%20sin%20privilegios.png)

Prueba de puertos desde este usuario:

![Puertos usuario sin privilegios](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/user%20sin%20priv%20puertos%20.png)

### 6.2 Usuario con privilegios

Tiene publicados Web, PuTTY y RDP en el Jump Server.

![Jump con privilegios](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/jumplab%20-%20con%20privilegios.png)

![Puertos usuario con privilegios](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/ubu%20user%20priv%20puertos.png)

### 6.3 SSH desde Windows al servidor Ubuntu

![SSH Windows a Ubuntu](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/ssh%20wdw%20sv%20a%20ubuntu%20.png)

### 6.4 Pruebas de puertos (true / false)

Verifiqué qué puertos responden y cuáles no, incluyendo el 80.

![Puertos en true y false](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/puertos%20en%20true%20y%20false%2080.png)

### 6.5 Logs y reportes del FortiGate

Evidencia del tráfico permitido y bloqueado.

![Log y report](https://github.com/sebas533/PRACTICA-3-INF-2/raw/main/IMAGENES/Log%20y%20report.png)

### 6.6 Tabla de pruebas

| # | Prueba | Resultado esperado |
| --- | --- | --- |
| 1 | VPN → Jump Server | Permitido |
| 2 | VPN → Web Server directo | Bloqueado |
| 3 | Jump → Web Server por HTTPS / RDP / SSH | Permitido |
| 4 | Jump → Web Server por otro puerto | Bloqueado |
| 5 | Usuario sin privilegios → Web del Jump | Permitido |
| 6 | Usuario sin privilegios → SSH a servidores | Bloqueado |
| 7 | Usuario con privilegios → Web, PuTTY y RDP en el Jump | Permitido |
| 8 | DHCP en VLAN 10 | IP asignada |

---

## 7. Configuraciones

Carpeta [`Configs/`](Configs):

| Archivo | Equipo |
| --- | --- |
| `CISCO-2168.txt` | Equipo de red Cisco |
| `FortiGate.conf` | FortiGate |
| `ISP-2168.txt` | ISP |
| `SW- VPN.txt` | Switch de VPN |
| `SW-USERS.txt` | Switch de usuarios (VLAN 10) |

---

## 8. Resultado

- Configuré el FortiGate completamente por GUI.
- La **VPN solo me da acceso al Jump Server**.
- El **Jump Server solo alcanza HTTPS, RDP y SSH** del Web Server.
- El **usuario sin privilegios** solo usa el servicio Web del Jump Server y no puede hacer SSH a los servidores.
- El **usuario con privilegios** accede a Web, PuTTY y RDP en el Jump Server.
- RDP RemoteApp y RemoteApp Web Client quedaron funcionando.

---

## Estructura del repositorio

```
PRACTICA-3-INF-2/
├── README.md
├── Configs/
│   ├── CISCO-2168.txt
│   ├── FortiGate.conf
│   ├── ISP-2168.txt
│   ├── SW- VPN.txt
│   └── SW-USERS.txt
└── IMAGENES/
    └── (capturas de topología, FortiGate, switches, VPN, servidores y pruebas)
```
