# Plan de conectividad y telemetría — Robots Unitree

**Versión:** 2026-08-19 (mediciones de Fase 0 incorporadas)
**Alcance:** IR1101, red de robots, transporte DDS, pipeline a Splunk
**Documento base:** `RED-Y-DDS.md`

---

## 0. Principio rector

> **DDS corto y local, HTTP largo y ruteado.**

Ninguna configuración de red hace viajar DDS entre VLANs. El proceso que lee DDS
vive en el segmento L2 del robot; lo que cruza la red es HTTPS, RTSP o WebRTC.

Todo lo que sigue se apoya en esto. Si en algún momento un paso parece requerir
que DDS atraviese el IR1101, el paso está mal planteado.

---

## 1. Estado actual

### Funcionando

| Elemento | Estado |
|---|---|
| VPN IKEv2 IR1101 → Meraki MX HQ | Operativa |
| NAT `192.168.123.18` → `10.1.254.18` | Operativa (ping desde HQ OK) |
| Contenedor ThousandEyes (IOx) | RUNNING, preservado |
| Pipeline de video JPEG | Funcionando por cable |

### Bloqueado

| Elemento | Causa |
|---|---|
| Ping desde HQ a `10.1.254.161` | El **controlador de bajo nivel `.161`** no tiene default gateway (su tabla de rutas sigue **sin verificar**). No confundir con el Jetson `.18`, que **sí lo tiene** — medido 2026-08-19 |
| Ingesta a Splunk | Puerto 8088 (HEC) cerrado |
| Lectura DDS del G1 | PC1 sin SSH, binding no modificable |
| CURWB como transporte | Sin validar con cable desenchufado |

### Desconocido

- VLAN ID real del segmento de robots (≠ 123 necesariamente)
- Si esa VLAN llega trunkeada a `vmnic3`
- Tabla de rutas del controlador de bajo nivel `.161` (el Jetson `.18` ya está medido)

---

## 2. Restricciones que no se negocian

1. **Los dos robots son `192.168.123.161`** en bajo nivel. Imposible ponerlos en
   el mismo segmento L2. Requiere VLAN por robot.
2. **El `.161` no es modificable** en ninguno de los dos robots. Firmware cerrado
   en el Go2; sin SSH en PC1 del G1.
3. **NAT no transporta DDS.** Los locators viajan en el payload RTPS y no existe
   ALG de RTPS en ningún vendor.
4. **El contenedor ThousandEyes del IR1101 es intocable**, incluida la línea 10
   de la ACL `NAT`.
5. **`rt/lowstate` no se relaya.** Usar `rt/lf/lowstate`: mismos datos, 20 Hz.
   Medido el 2026-08-19 (ver `CENSO-GO2.md`):
   - **Go2:** `rt/lowstate` = 500 Hz, 1,18 KB/msg, **593 KB/s** (51 GB/día crudos).
     `rt/lf/lowstate` = 20 Hz, mismo tamaño de mensaje, **23,8 KB/s**.
   - **G1** (medido antes): `rt/lowstate` = 1041 Hz, **2,2 MB/s**; `rt/lf/lowstate` = 42 KB/s.

   Ni la versión `lf/` entra reenviada tal cual: expandida a JSON son ~6-10 GB/día
   contra un presupuesto de 500 MB/día. Hay que **extraer campos**, no reenviar
   mensajes (ver `PLAN.md` §5.1 y §6).

---

## 3. Arquitectura objetivo

```
  VLAN robot A (VRF ROBOT-GO2)      VLAN robot B (VRF ROBOT-G1)
 ┌────────────────────────────┐    ┌────────────────────────────┐
 │  Go2                       │    │  G1                        │
 │   .161  bajo nivel  ──┐    │    │   .161  bajo nivel  ──┐    │
 │   .18   Jetson  ◀─────┘DDS │    │   .164  PC2     ◀─────┘DDS │
 │          │                 │    │          │                 │
 │          │ colector local  │    │          │ colector local  │
 └──────────┼─────────────────┘    └──────────┼─────────────────┘
            │ HTTPS                           │ HTTPS
            └──────────────┬──────────────────┘
                           ▼
                    ┌─────────────┐
                    │   IR1101    │  NAT VRF-aware + VPN IKEv2
                    │ 10.1.254.0/24│
                    └──────┬──────┘
                           │  Starlink Mini (bypass, CGNAT)
                           ▼
                    ┌─────────────┐
                    │  Meraki MX  │  HQ  192.168.0.0/16
                    └──────┬──────┘
                           ▼
                    Splunk  192.168.20.200:8088
```

**Mapeo de direcciones tras NAT:**

| Recurso | IP nativa | Visible desde HQ |
|---|---|---|
| Go2 — Jetson (**alto nivel**, SSH) | `192.168.123.18` | `10.1.254.18` |
| Go2 — controlador (**bajo nivel**, publica el DDS) | `192.168.123.161` | `10.1.254.161` |
| G1 — PC2 / Jetson (**alto nivel**, SSH) | `192.168.123.164` | `10.1.254.64` |
| G1 — PC1 (**bajo nivel**, publica el DDS, **sin SSH**) | `192.168.123.161` | `10.1.254.61` |

> Cuidado con la nomenclatura: el **`.161` es el BAJO nivel** en los dos robots (es el que
> publica los tópicos de estado). El alto nivel es `.18` en el Go2 y `.164` en el G1. Ver
> §2.1.

---

## 4. Fases

### Fase 0 — Mediciones que cierran decisiones

Barata y desbloquea todo lo demás. **No avanzar sin completarla.**

- [x] **Tabla de rutas del Jetson `.18`** — ✅ **medido 2026-08-19**:
      `default via 192.168.123.1 dev eth0`, internet OK (8.8.8.8 en 2,86 ms), DNS OK,
      NTP activo y sincronizado. Ping al `.161` en **0,25 ms** (mismo L2 → el DDS local
      funciona). Ping a Splunk `192.168.20.200` en **0,74 ms**, tcp/8000 abierto,
      **tcp/8088 cerrado**.
      → **El Jetson tiene los dos lados resueltos: ve el DDS y alcanza Splunk.**
- [ ] **Tabla de rutas del controlador de bajo nivel `.161`** — sigue pendiente (es otra
      máquina, y es la que decide si un lector *ruteado* sería posible).

- [ ] **VLAN ID real del segmento de robots.**
      ```bash
      show interface FastEthernet0/0/1 switchport
      show vlan brief
      ```

- [ ] **¿La VLAN de robots llega trunkeada a `vmnic3`?** Confirmar en ESXi.

- [x] **Inventario del Jetson** — ✅ **hecho 2026-08-19** (detalle en `PLAN.md` §2.3):
      aarch64 / Ubuntu 20.04.5, **g++ 9.4.0 + cmake + make + git**, 4 cores,
      15,4 GB RAM (14,2 libres), **469 GB disco (429 libres)**, load average **0,00**,
      ROS2 Foxy y ROS1 Noetic instalados, servicio `unitree-upgrade` corriendo.
      **Ya corre un agente de terceros**: contenedor `go2-jetson-01` con
      `thousandeyes/enterprise-agent` → hay precedente y patrón (Docker) para desplegar
      software en el robot.

**Criterio de salida:** las cuatro respuestas documentadas.
**Estado 2026-08-19: 2 de 4 cerradas** (Jetson e inventario). Faltan las dos de VLAN, que
necesitan CLI del IR1101 y de ESXi — ninguna bloquea al colector.

---

### Fase 1 — Habilitar el HEC

Bloquea el pipeline completo y no depende de la red.

- [ ] Habilitar HTTP Event Collector en Splunk `192.168.20.200`
- [ ] Crear token dedicado por robot (`go2-01`, `g1-01`)
- [ ] Crear índices separados por fuente de datos
- [ ] Abrir tcp/8088 en el firewall de HQ hacia `10.1.254.0/24`
- [ ] Validar desde el Jetson:
      ```bash
      curl -k https://192.168.20.200:8088/services/collector/health
      ```

**Criterio de salida:** el health check responde desde el Jetson a través de la VPN.

---

### Fase 2 — Cerrar la reachability del `.161`

- [ ] Aplicar twice-NAT en el IR1101:
      ```
      ip nat pool POOL-MASK-HQ 192.168.123.200 192.168.123.220 prefix-length 24
      !
      ip access-list extended ACL-MASK-HQ
       permit ip 192.168.0.0 0.0.255.255 any
      !
      ip nat outside source list ACL-MASK-HQ pool POOL-MASK-HQ add-route
      ```
- [ ] Verificar la doble traducción:
      ```
      show ip nat translations
      ```
      Esperado: Inside global `10.1.254.161`, Inside local `192.168.123.161`,
      Outside local `192.168.123.200`, Outside global `<IP de HQ>`
- [ ] Ping desde HQ a `10.1.254.161`

**Alternativa si el twice-NAT resulta frágil:** no exponer el `.161` y operarlo
desde el Jetson por SSH. Es como el SDK de Unitree espera trabajar de todos modos.

**Criterio de salida:** ping desde HQ, o decisión documentada de no exponerlo.

---

### Fase 3 — Colector de telemetría en el Jetson

El proceso corre **dentro** del segmento del robot. Nada de DDS cruza el IR1101.

- [ ] Suscribir a `rt/lf/lowstate` (20 Hz), no a `rt/lowstate`
- [ ] QoS del lector en **BEST_EFFORT** (empareja con escritores RELIABLE y no se
      espiraliza en NACKs ante pérdida)
- [ ] `CYCLONEDDS_URI` con `<Interfaces>` fijando `eth0` explícitamente —
      `ChannelFactory::Init(0, "eth0")` no alcanza por sí solo
- [ ] Decimación adicional a 1–5 Hz para lo que va a Splunk
- [ ] POST batcheado al HEC con reintento y cola en disco
- [ ] Servicio systemd con `Restart=always`

**Criterio de salida:** eventos del Go2 visibles en Splunk de forma sostenida
durante 1 hora.

---

### Fase 4 — Validar CURWB con el cable desenchufado

Determina si el G1 es leíble sin acceso a PC1.

- [ ] **Desenchufar físicamente** el cable del robot. Toda medición con el cable
      conectado es inválida: `192.168.123.0/24` está directamente conectada por
      `eth0` y el camino inalámbrico nunca se ejercita.
- [ ] Confirmar bridge L2 transparente (el multicast tiene que sobrevivir)
- [ ] ```bash
      ros2 daemon stop
      ros2 topic list --no-daemon | wc -l
      ```
      12 tópicos = no ve al robot. ~121 = funcionando.
- [ ] Medir latencia y pérdida en handoff entre APs

**Criterio de salida:** conteo de tópicos con el cable desenchufado, documentado.

---

### Fase 5 — Segundo robot: VRF en el IR1101

Solo después de que el Go2 esté estable end-to-end.

- [ ] Crear VRFs y VLAN por robot (ver §5 del config)
- [ ] NAT VRF-aware con `match-in-vrf`
- [ ] Sub-interfaces taggeadas en la VM colectora sobre `TRUNK ITINERANTE` (4095)
- [ ] CycloneDDS bindeado **por interfaz**, un dominio por robot
- [ ] Reservas DHCP para las MAC de los robots (evita peers rancios)

**Criterio de salida:** los dos robots gestionables desde HQ sin conflicto de IP.

---

## 5. Config de referencia — VRF en el IR1101

```
vrf definition ROBOT-GO2
 rd 65000:1
 address-family ipv4
 exit-address-family
!
vrf definition ROBOT-G1
 rd 65000:2
 address-family ipv4
 exit-address-family
!
interface Vlan123
 vrf forwarding ROBOT-GO2
 ip address 192.168.123.1 255.255.255.0
 ip nat inside
!
interface Vlan124
 vrf forwarding ROBOT-G1
 ip address 192.168.123.1 255.255.255.0
 ip nat inside
!
ip nat inside source static 192.168.123.18  10.1.254.18  vrf ROBOT-GO2 match-in-vrf
ip nat inside source static 192.168.123.161 10.1.254.161 vrf ROBOT-GO2 match-in-vrf
ip nat inside source static 192.168.123.164 10.1.254.64  vrf ROBOT-G1  match-in-vrf
ip nat inside source static 192.168.123.161 10.1.254.61  vrf ROBOT-G1  match-in-vrf
!
ip route vrf ROBOT-GO2 0.0.0.0 0.0.0.0 GigabitEthernet0/0/0 global
ip route vrf ROBOT-G1  0.0.0.0 0.0.0.0 GigabitEthernet0/0/0 global
```

`match-in-vrf` es obligatorio: sin él, con dos statics para la misma IP local el
comportamiento es indeterminado.

---

## 6. Opcional — eth1 en el Jetson

Solo si un consumidor remoto necesita **ROS 2 nativo** (RViz, nodo de navegación).
Para Splunk es una vuelta innecesaria: el colector HTTPS de la Fase 3 hace lo
mismo con una pieza menos.

Requiere dongle USB3-GbE (RTL8153). El puerto del dock **no** es una NIC
independiente: es otro puerto del switch interno de `192.168.123.0/24`.

- [ ] Fijar nombre de interfaz por udev (el dongle hace que NetworkManager
      renombre interfaces e invalida cualquier test)
- [ ] `CYCLONEDDS_URI` con dos bloques `<Domain>`: dominio 0 en `eth0` con
      multicast, dominio 42 en `eth1` con unicast y peers explícitos
- [ ] `domain_bridge` relayando solo tópicos decimados, QoS `best_effort`
- [ ] Verificar locators: en el lado remoto deben aparecer `10.1.254.x` y
      **cero** `192.168.123.x`

---

## 7. Trampas conocidas

| Trampa | Mitigación |
|---|---|
| `ros2 topic list` con daemon viejo | `ros2 daemon stop` o `--no-daemon` |
| Test de CURWB con el cable conectado | Desenchufar físicamente |
| Ping a IP WiFi del robot con cable conectado | `ping -I wlan0` **desde el robot** |
| Escaneo UDP 7400-7500 | Rate limiting de ICMP hace parecer abiertos los cerrados |
| Peers DDS rancios | Reservas DHCP por MAC |
| Dongle USB-C renombra interfaces | Regla udev persistente |
| `dds.env` es root | Editar con sudo |
| "VLAN 20" en docs viejos | Hoy VLAN 20 = servers. Robots WiFi = VLAN 51 |
| MTU 1500 en vSwitch + tagging del guest | +4 bytes; sospechoso #1 si fallan muestras grandes |

---

## 8. Riesgos

| Riesgo | Impacto | Mitigación |
|---|---|---|
| CURWB no valida sin cable | Alto — sin camino para el G1 | Fase 4 temprano; fallback a colector onboard |
| Twice-NAT inestable | Medio | Fallback: no exponer `.161`, operar vía Jetson |
| Starlink CGNAT + túnel caído | Alto | IP SLA keepalive ya configurado; MX no puede iniciar |
| Saturación por relay de tópicos | Medio | Solo `lf/`, decimación, best_effort |
| Firmware Unitree pisa cambios | Medio | No modificar el stack de control; todo en el Jetson |

---

## 9. Documentos relacionados

| Doc | Contenido |
|---|---|
| `RED-Y-DDS.md` | Fundamento técnico de por qué DDS no rutea |
| `IR1101-GO2-01-FULL.cfg` | Running-config conciliada del IR1101 — ⚠️ **referenciado pero no presente en esta máquina** |
| `SplunkCode/PLAN.md` | Plan de telemetría — **autoridad sobre el colector y el contrato de datos** (la Fase 3 de acá no lo repite: ver §6-§7 de ese doc) |
| `SplunkCode/CENSO-GO2.md` | Mediciones reales de tópicos del Go2 (2026-08-19) |
| `robot-nvr-bridge/docs/ARQUITECTURA.md` | Pipeline de video funcionando |
| `robot-nvr-bridge/docs/DOS-ROBOTS.md` | **Tabla de IPs desactualizada** — corregir |

---

## 10. Próximo paso

**Fase 0.** Cuatro mediciones, unos 30 minutos con el robot encendido. Cierran
decisiones de arquitectura que hoy están abiertas y evitan trabajo sobre
supuestos.
