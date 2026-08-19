# Inventario de IPs: qué es cada una y dónde se configura

Generado el 2026-08-19 grepeando los repos, no de memoria. Regenerable con:

```bash
cd ~/Desktop && grep -rnoE "192\.168\.[0-9]+\.[0-9]+|10\.1\.254\.[0-9]+" \
  --include="*.sh" --include="*.yml" --include="*.yaml" --include="*.service" \
  --include="*.py" --include="*.cpp" --include="*.xml" --include="*.env" \
  robot-nvr-bridge robot-splunk-bridge unitree_ros2 SplunkCode | sort -u
```

---

## 1. Quién es quién

| Qué | IP | Notas |
|---|---|---|
| **Go2 — bajo nivel (publica el DDS)** | `192.168.123.161` | Fijo de fábrica, **no modificable**. Sin SSH útil |
| **Go2 — Jetson (alto nivel, SSH)** | `192.168.123.18` | usuario `unitree`. **Acá corren nuestros agentes** |
| **G1 — PC1 (bajo nivel)** | `192.168.123.161` | ⚠️ **la misma que el Go2**. Sin SSH en ningún puerto |
| **G1 — PC2 (Jetson, SSH)** | `192.168.123.164` | aarch64, Ubuntu 20.04, ROS2 Foxy |
| **Esta PC (workstation)** | `192.168.123.99` ↔ `192.168.20.99` | `enp4s0`, perfil NetworkManager **"Profile 1"** |
| Gateway de la red de robots | `192.168.123.1` | rutea a las otras VLANs |
| Gateway de la VLAN 20 | `192.168.20.1` | |
| **Splunk** | `192.168.20.200` | `:8000` UI · `:8088` HEC · `:8089` API |
| ESXi — management (vmk0) | `192.168.20.3` | |
| Jetson del Go2 visto desde HQ | `10.1.254.18` | NAT del IR1101, operativo |
| Bajo nivel visto desde HQ | `10.1.254.161` | twice-NAT, **pendiente** |
| G1 PC2 visto desde HQ | `10.1.254.64` | |
| WiFi "ROBOTS ONLY" | VLAN **51**, `192.168.51.x` | G1 PC1 `.116`, PC2 `.115` |

> ⚠️ **Los dos robots comparten `192.168.123.161`.** Nunca pueden estar en el mismo segmento
> L2: es conflicto de IP, no solo colisión de tópicos DDS. Ver `RED-Y-DDS.md` §4.
>
> ⚠️ **Docs viejos dicen "VLAN 20" para el WiFi de robots.** Se movió a la **VLAN 51**
> (~2026-08-03). Hoy **VLAN 20 = servidores** (ahí vive Splunk).

---

## 2. Si cambia la IP de ESTA PC — 5 lugares

Es la que más duele porque aparece repartida. Hoy `192.168.20.99`.

| Archivo | Línea | Qué es | Efecto si queda mal |
|---|---|---|---|
| `robot-nvr-bridge/frigate/config/config.yml` | 18 | de dónde Frigate lee el RTSP | **No graba** |
| `robot-nvr-bridge/robot/robot-video.service` | 16 | `PUBLISH_HOST` — a dónde publica el robot | **No llega el video** |
| `robot-nvr-bridge/frigate/docker-compose.yml` | 4-5 | solo comentarios (URLs de la UI) | cosmético |
| `robot-nvr-bridge/frigate/config/backup_config.yaml` | 18 | copia de respaldo, sin uso | ninguno |
| `SplunkCode/dashboard-go2.xml` | 108, 111 | iframe y link del panel de video | **panel negro** |

Y además, fuera de los archivos:

- **La subred determina si hay DDS.** En `192.168.123.x` esta PC ve los 122 tópicos; en
  cualquier otra ve 2. Eso decide si **AI-VL y la captura local de video funcionan**.
- Cambiarla: `sudo nmcli con mod "Profile 1" ipv4.addresses <IP>/24 ipv4.gateway <GW>` y
  `sudo nmcli con up "Profile 1"`. Si además movés el **puerto del switch** de VLAN, hay que
  revertir eso también.

---

## 3. Si cambia la IP de Splunk

Hoy `192.168.20.200`.

| Archivo | Línea | Qué es |
|---|---|---|
| `robot-splunk-bridge/systemd/robot-splunk-bridge.service` | 16 | `HEC_URL` — **el que importa**, vive en el robot |
| `robot-splunk-bridge/poc/telemetry_poc.py` | 18 | solo un ejemplo en el docstring |

Después de cambiarlo: `git push` acá, y en el robot
`git pull && sudo cp systemd/*.service /etc/systemd/system/ && sudo systemctl daemon-reload && sudo systemctl restart robot-splunk-bridge`.

---

## 4. Si cambia la IP del robot

Hoy bajo nivel `192.168.123.161`, Jetson `192.168.123.18`. **De fábrica y no se cambian**, pero
si algún día pasara:

| Archivo | Línea | Qué es |
|---|---|---|
| `robot-nvr-bridge/status.sh` | 6 | `ROBOT_IP`, solo para el chequeo de ping |
| `robot-nvr-bridge/start-all.sh` | 8 | ídem |
| `unitree_ros2/robot_executor/.env` | 14 | IP del robot para AI-VL |
| `unitree_ros2/robot_executor/robot_executor_service.py` | 84 | default en código |

---

## 5. Interfaces de red (no IPs, pero rompen igual)

| Dónde | Variable | Valor | Nota |
|---|---|---|---|
| `robot-nvr-bridge/run.sh` | `NIC` | `enp4s0` | la de **esta PC** hacia el robot |
| `robot-nvr-bridge/robot/run-video.sh` | `NIC` | `eth0` | la del **robot** (bus interno) |
| `robot-splunk-bridge/run.sh` | `DDS_IFACE` | `eth0` | ídem, en el robot |
| `unitree_ros2/dds.env` | `CYCLONEDDS_IFACE` | `enp4s0` | AI-VL, en esta PC. **Es root: editar con sudo** |
| `unitree_ros2/dds.env` | `ROBOT_DDS_PEERS` | `192.168.51.115` | ⚠️ **peer viejo del G1 por WiFi.** Vaciarlo en LAN plana |

**Sin la interfaz correcta no hay DDS.** `ChannelFactory::Init(0, nic)` por sí solo no recibe
nada: hace falta el `CYCLONEDDS_URI` con `<NetworkInterface>`. Es el detalle que costó más de
encontrar en todo el proyecto.

---

## 6. Puertos

| Puerto | Dónde | Qué |
|---|---|---|
| `8554` | esta PC | mediamtx **RTSP** — lo que consume Frigate |
| `8888` | esta PC | mediamtx **HLS** (navegador) |
| `8889` | esta PC | mediamtx **WebRTC** (navegador, mínima latencia) |
| **`1935`** | esta PC | mediamtx **RTMP** — por acá publica el robot |
| `8890` | esta PC | mediamtx SRT — habilitado pero **inservible**: libsrt 1.4.0 del robot es incompatible |
| `5000` / `8971` | esta PC | Frigate UI (sin auth / con auth) |
| `8000` / `8088` / `8089` | Splunk | UI / **HEC** / API |
| `7400-7500` | robot ↔ lector | DDS (UDP). **Solo funciona dentro del mismo L2** |

---

## 7. Credenciales

| Qué | Dónde vive | Nota |
|---|---|---|
| Token HEC de Splunk | `~/.splunk_hec_token` **en el robot** (modo 600) | Fuera del repo: no se pushea ni se pisa con un pull |
| SSH del Jetson | usuario `unitree` | ⚠️ **la password es `123`** — cambiarla, hay un token ahí adentro |
| Splunk | índice `go2-robot-data`, token `Go2-01` | Un índice y un token **por robot**, para poder revocar |
