# Plan de implementación: Telemetría de robots Unitree (G1 + Go2) hacia Splunk

## 0. Objetivo y arquitectura general

Centralizar en Splunk toda la telemetría de dos robots Unitree (**G1** humanoide y **Go2** cuadrúpedo), incluyendo:

- Logs (`/rosout` y logs de sistema)
- Tópicos ROS2 / mensajes DDS
- Datos de sensores (IMU, lidar, cámaras de profundidad, etc.)
- Estado de motores/actuadores (`/joint_states`, torque, temperatura)
- Video en vivo (mostrado en el dashboard, aunque no indexado dentro de Splunk)

### Arquitectura

```
Unitree G1 (segmento red A) ──DDS/ROS2──┐
                                          ├─► Splunk-collector VM
Unitree Go2 (segmento red B) ──DDS/ROS2──┘      (bridge por robot, JSON vía HEC)
                                                       │
                                                       ▼
                                                    Splunk
                                          (índices, búsquedas, dashboard)

Video: Robot ──RTSP──► NVR / MediaMTX ──HLS/WebRTC──► iframe embebido en el dashboard de Splunk
```

**Por qué esta arquitectura:**
- Los dos robots usan por defecto la misma subred `192.168.123.0/24` para DDS multicast → **deben estar en segmentos de red separados** (ya lo hicieron) para evitar colisión de descubrimiento.
- La VM collector necesita **una interfaz de red por segmento** para poder hablar con cada robot sin mezclar tráfico DDS.
- El video **no se indexa en Splunk** (no es la herramienta correcta para almacenar/streamear binarios pesados); se muestra embebido en el dashboard vía un panel HTML/iframe apuntando a un servidor de streaming (NVR / MediaMTX / go2rtc).

---

## 1. Prerrequisitos

- [ ] VM "Splunk-collector" creada y con Ubuntu Server instalado (ya hecho).
- [ ] VM de Splunk existente, funcionando (indexer + dashboards).
- [ ] Acceso a los portgroups/VLANs de cada segmento de red de los robots desde el host ESXi.
- [ ] ROS2 instalado en ambos robots (ya lo tienen, es de fábrica en Unitree).
- [ ] Credenciales de administrador de Splunk.
- [ ] NVR ya configurado para el video (mencionado que ya existe).

---

## 2. Configuración de red en la VM collector

### 2.1. Agregar una interfaz de red por robot

1. Apagá la VM "Splunk-collector".
2. **Edit settings** → **Add network adapter**.
3. Agregá una segunda NIC virtual, conectada al portgroup/VLAN del segmento de red del **Go2** (la primera NIC ya debería estar en el segmento del **G1**, o viceversa — definir cuál es cuál).
4. Prendé la VM.
5. Dentro de Ubuntu, verificá que aparezcan ambas interfaces:
   ```bash
   ip a
   ```
   Deberías ver algo como `eth0` (segmento G1) y `eth1` (segmento Go2).

6. Configurá cada interfaz con una IP estática dentro de su subred correspondiente, editando netplan:
   ```bash
   sudo nano /etc/netplan/00-installer-config.yaml
   ```
   Ejemplo:
   ```yaml
   network:
     version: 2
     ethernets:
       eth0:
         addresses: [192.168.123.50/24]   # segmento G1
       eth1:
         addresses: [192.168.124.50/24]   # segmento Go2 (ajustar según corresponda)
   ```
   Aplicar con:
   ```bash
   sudo netplan apply
   ```

7. Verificar conectividad hacia cada robot:
   ```bash
   ping <ip-del-G1>
   ping <ip-del-Go2>
   ```

### 2.2. Confirmar aislamiento de multicast

- [ ] Confirmar con el equipo de red que ambos segmentos son dominios de broadcast/multicast **separados** (VLANs distintas, sin bridging entre ellas a nivel de multicast).
- [ ] Si en algún punto se necesita que el tráfico multicast cruce una frontera L3 (por ejemplo, si en el futuro la VM no tiene una pata física en cada segmento), va a hacer falta multicast routing o cambiar a un modo de discovery por unicast (ver sección 3.3).

---

## 3. Instalación de ROS2 y el bridge en la VM collector

### 3.1. Instalar ROS2 (misma distro que corren los robots)

```bash
# Verificar qué distro de ROS2 corren los robots (ej: Humble, Iron, Jazzy)
# En el robot: printenv ROS_DISTRO

sudo apt update && sudo apt install -y curl gnupg lsb-release
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(source /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list

sudo apt update
sudo apt install -y ros-<distro>-desktop python3-rosdep2 python3-colcon-common-extensions
```

### 3.2. Instalar dependencias Python para el bridge

```bash
pip install --break-system-packages requests rosidl-runtime-py
```

### 3.3. Configurar el DDS por interfaz de red

Cada bridge debe escuchar únicamente en la interfaz correspondiente a su robot. Esto se logra con variables de entorno del middleware DDS (ejemplo con Cyclone DDS, el más simple de configurar por interfaz):

```bash
sudo apt install -y ros-<distro>-rmw-cyclonedds-cpp
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

Crear un archivo de configuración por robot, por ejemplo `cyclonedds_g1.xml`:
```xml
<CycloneDDS>
  <Domain>
    <General>
      <NetworkInterfaceAddress>eth0</NetworkInterfaceAddress>
    </General>
  </Domain>
</CycloneDDS>
```

Y `cyclonedds_go2.xml` con `eth1`. Cada proceso bridge se lanza con su propia variable de entorno apuntando al XML correspondiente:
```bash
export CYCLONEDDS_URI=file:///home/usuario/cyclonedds_g1.xml
```

> Si en el futuro los robots dejan de estar en la misma LAN física que la VM (conexión remota real), evaluar migrar a **discovery server** (unicast, sin depender de multicast) en vez de descubrimiento multicast tradicional.

### 3.4. Script del bridge (uno por robot)

Guardar como `bridge_node.py` (parametrizado por variable de entorno `ROBOT_NAME`):

```python
import os
import rclpy
from rclpy.node import Node
from rosidl_runtime_py.utilities import get_message
from rosidl_runtime_py.convert import message_to_ordereddict
import requests
import json

ROBOT_NAME = os.environ.get("ROBOT_NAME", "unknown")
HEC_URL = os.environ.get("HEC_URL")
HEC_TOKEN = os.environ.get("HEC_TOKEN")
HEC_INDEX = os.environ.get("HEC_INDEX", "robot_data")

class BridgeNode(Node):
    def __init__(self):
        super().__init__(f'bridge_{ROBOT_NAME}')
        self.session = requests.Session()
        self.session.headers.update({"Authorization": f"Splunk {HEC_TOKEN}"})
        self.subscribed = set()
        self.timer = self.create_timer(5.0, self.discover_topics)
        self.discover_topics()

    def discover_topics(self):
        for name, types in self.get_topic_names_and_types():
            if name in self.subscribed:
                continue
            try:
                msg_type = get_message(types[0])
                self.create_subscription(
                    msg_type, name,
                    lambda msg, n=name: self.callback(n, msg), 10
                )
                self.subscribed.add(name)
                self.get_logger().info(f"Suscrito a {name}")
            except Exception as e:
                self.get_logger().warn(f"No se pudo suscribir a {name}: {e}")

    def callback(self, topic_name, msg):
        try:
            data = message_to_ordereddict(msg)
        except Exception:
            data = {"raw": str(msg)}

        event = {
            "robot": ROBOT_NAME,
            "topic": topic_name,
            "data": data,
        }
        payload = {
            "event": event,
            "sourcetype": "ros2_telemetry",
            "index": HEC_INDEX,
        }
        try:
            self.session.post(HEC_URL, json=payload, verify=False, timeout=3)
        except requests.exceptions.RequestException as e:
            self.get_logger().warn(f"Error enviando a HEC (topic={topic_name}): {e}")

def main():
    rclpy.init()
    node = BridgeNode()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == "__main__":
    main()
```

### 3.5. Servicios systemd (uno por robot, para que corran siempre)

`/etc/systemd/system/bridge-g1.service`:
```ini
[Unit]
Description=Bridge ROS2 -> Splunk HEC (G1)
After=network.target

[Service]
Environment=ROBOT_NAME=g1
Environment=RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
Environment=CYCLONEDDS_URI=file:///home/usuario/cyclonedds_g1.xml
Environment=HEC_URL=https://<ip-splunk>:8088/services/collector/event
Environment=HEC_TOKEN=<token-g1-o-compartido>
Environment=HEC_INDEX=robot_data
ExecStart=/usr/bin/python3 /home/usuario/bridge_node.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Crear un segundo archivo `bridge-go2.service` análogo, cambiando `ROBOT_NAME=go2`, la interfaz DDS y el XML correspondiente.

Activar ambos:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now bridge-g1.service
sudo systemctl enable --now bridge-go2.service
sudo systemctl status bridge-g1.service
sudo systemctl status bridge-go2.service
```

---

## 4. Configuración de Splunk

### 4.1. Crear el índice

1. **Settings** → **Indexes** → **New Index**.
2. Nombre: `robot_data`.
3. Configurar retención según el volumen esperado (los tópicos de alta frecuencia como IMU o joint_states pueden generar mucho volumen; monitorear los primeros días y ajustar).

### 4.2. Habilitar HTTP Event Collector (HEC)

1. **Settings** → **Data Inputs** → **HTTP Event Collector**.
2. **Global Settings** → habilitar HEC, definir puerto (default `8088`), habilitar SSL (recomendado).
3. **New Token**:
   - Nombre: `robot-telemetry`.
   - Índice por defecto: `robot_data`.
   - Sourcetype por defecto: `ros2_telemetry`.
4. Guardar el token generado (se usa en las variables `HEC_TOKEN` de los servicios systemd).

### 4.3. Seguridad del endpoint HEC

- [ ] Restringir el acceso al puerto 8088 solo desde la IP de la VM collector (firewall/reglas de red).
- [ ] Usar certificado válido para HTTPS (o al menos documentar que se usa autofirmado solo en entorno de pruebas).
- [ ] Si en el futuro los robots se conectan de forma remota/itinerante directamente (sin pasar por la VM collector local), evaluar exponer HEC detrás de un reverse proxy con autenticación adicional o VPN.

### 4.4. Verificar que llegan datos

```spl
index="robot_data"
```
Confirmar que aparecen eventos de ambos robots (`robot=g1`, `robot=go2`) y de distintos tópicos.

### 4.5. Logs específicos (`/rosout`)

El tópico `/rosout` ya se captura automáticamente por el bridge (se suscribe a todos los tópicos). Para separarlo visualmente en Splunk, se puede usar un sourcetype distinto en el bridge cuando `topic_name == "/rosout"`, o simplemente filtrar en las búsquedas:
```spl
index="robot_data" topic="/rosout"
```

---

## 5. Video en vivo

### 5.1. Pipeline de video (separado de Splunk)

1. Confirmar que el NVR ya existente puede exponer los streams como **HLS** o **WebRTC** (formatos reproducibles en navegador). Si no, evaluar agregar **MediaMTX** (liviano, gratuito) como capa de conversión RTSP → HLS/WebRTC.
2. Verificar que la URL del stream sea accesible desde la red donde se abre el dashboard de Splunk.

### 5.2. Embeber el video en el dashboard

1. Crear el dashboard en Splunk usando **Dashboard Studio** (no el clásico "Simple XML", que tiene más limitaciones para HTML embebido).
2. Agregar un panel de tipo **"Source code" / HTML** (o un panel custom con visualización HTML) con un `<iframe>` apuntando a la URL del stream HLS/WebRTC.
3. Ubicar este panel junto a los paneles de telemetría (batería, motores, etc.) para tener todo en una sola vista.

### 5.3. Metadata del video en Splunk (opcional pero recomendado)

Aunque el video en sí no se indexa, se puede mandar como eventos normales vía HEC:
- Inicio/fin de grabación.
- Eventos detectados (si hay analítica de video).
- Timestamps clave para correlacionar con otros datos de telemetría.

---

## 6. Dashboard en Splunk

### 6.1. Paneles sugeridos

- [ ] Batería (G1 y Go2) a lo largo del tiempo.
- [ ] Estado de motores/joints (torque, temperatura, posición).
- [ ] Mapa/posición (si hay datos de odometría o GPS).
- [ ] Tabla de logs/errores recientes (`/rosout` con nivel `ERROR` o `WARN`).
- [ ] Panel de video en vivo (embebido, ver sección 5.2).
- [ ] Contador de tópicos activos / última vez visto cada tópico (para detectar caídas de sensores).

### 6.2. Búsquedas base (SPL) de ejemplo

```spl
index="robot_data" robot="g1" topic="/joint_states"
| timechart avg(data.velocity) by topic
```

```spl
index="robot_data" topic="/rosout" data.level>=40
| table _time robot data.name data.msg
```

---

## 7. Checklist de implementación (orden sugerido)

1. [ ] Configurar segunda NIC en la VM collector y verificar conectividad a ambos robots.
2. [ ] Instalar ROS2 (misma distro que los robots) en la VM collector.
3. [ ] Crear índice `robot_data` y token HEC en Splunk.
4. [ ] Probar el envío manual de un evento de prueba vía `curl` al HEC para confirmar conectividad antes de meter ROS2 en el medio.
5. [ ] Configurar Cyclone DDS por interfaz (uno por robot) y confirmar que `ros2 topic list` funciona correctamente para cada uno por separado.
6. [ ] Desplegar el script `bridge_node.py` y los dos servicios systemd.
7. [ ] Verificar en Splunk que llegan eventos de ambos robots.
8. [ ] Configurar HEC de forma segura (firewall, SSL).
9. [ ] Conectar el pipeline de video (NVR/MediaMTX) y confirmar reproducción en navegador.
10. [ ] Armar el dashboard en Dashboard Studio, combinando telemetría + video.
11. [ ] Ajustar retención del índice y monitorear volumen de datos durante la primera semana.

---

## 8. Cosas a definir todavía / decisiones pendientes

- [ ] Confirmar distro de ROS2 exacta que corren G1 y Go2.
- [ ] Confirmar rango de IP/VLAN de cada segmento de red.
- [ ] Confirmar si el NVR actual ya expone HLS/WebRTC o si hace falta agregar MediaMTX.
- [ ] Definir política de retención de datos en Splunk (cuántos días/GB).
- [ ] Definir si se usa un token HEC por robot o uno compartido (recomendado: uno compartido con el campo `robot` para diferenciar, más simple de mantener).
