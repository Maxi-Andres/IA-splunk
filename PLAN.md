# Plan de telemetría de robots Unitree → Splunk

Revisión del plan original (`Telemetria-Splunk.md`) contra el estado real del stack.
Fundamento técnico de red y DDS: **`RED-Y-DDS.md`**. Mediciones: **`CENSO-GO2.md`**.
Red, VPN y transporte: **`PLAN-CONECTIVIDAD-ROBOTS.md`** (es la autoridad sobre esa capa; este
documento es la autoridad sobre el colector y el contrato de datos).

- Creado: 2026-08-18
- **Revisado 2026-08-19: cambio de premisa mayor — ver §0.**

---

## 0. Las dos decisiones que definen el plan

**Decisión 1 — extraer campos, no reenviar tópicos.** El diseño no puede ser *"reenviar
tópicos DDS a Splunk"*, tiene que ser *"extraer campos específicos a tasa controlada"*. Motivo
aritmético en §5.1: incluso el tópico más liviano, reenviado tal cual, es 20-35x el presupuesto
de licencia.

**Decisión 2 (2026-08-19) — el agente corre ADENTRO del robot.** Requisito nuevo del usuario:
los robots tienen que poder estar **en cualquier red** — en el campo con Starlink, con LTE, en
otra oficina — así que **no se puede asumir que compartan subred con nada**. Eso invalida todo
el diseño anterior de "un collector con presencia L2 en la red del robot" (VM en el trunk,
VLAN por robot, sub-interfaces taggeadas).

La observación que lo resuelve:

> **La única máquina que va a estar L2-adyacente al DDS del robot, esté el robot donde esté, es
> el robot mismo.**

Entonces el lector va adentro del robot, y lo que sale es **HTTPS saliente**. Ver §2.

---

## 1. Objetivo y alcance

Ver en un dashboard de Splunk, en una sola vista: **video en vivo, batería, temperaturas,
estado de motores, errores/faults y posición** de los robots Unitree.

- **Ahora:** un solo robot, el **Go2 (perro)**.
- **Después:** los dos (Go2 + G1), y con la premisa nueva, **N robots en N redes distintas**.
  El diseño es multi-robot desde el día uno (campo `robot` en cada evento, config por robot).
- **Requisito de movilidad:** el robot tiene que poder reportar desde cualquier red con salida
  a internet, sin configuración de red del lado del robot y sin puertos entrantes.
- **Fuera de alcance:** indexar video o nubes de puntos en Splunk. El video se muestra
  embebido; no consume licencia.

---

## 2. La decisión de arquitectura: el agente va en el robot

### 2.1. Opciones evaluadas

**Opción A — Agente adentro del robot, HTTPS saliente** ✅ **elegida**

| Pros | Contras |
|---|---|
| **Funciona desde cualquier red**: Starlink, LTE, WiFi ajeno. Es el único que cumple el requisito | Hay que desplegar y mantener software **en el robot** |
| Atraviesa NAT sin configurar nada: solo conexiones **salientes** | Hay que tener cuidado de no desestabilizar el robot (agente read-only, con límite de recursos) |
| **El conflicto de `.161` desaparece**: la IP queda privada dentro de cada robot, nunca se expone. N robots, todos `.161`, sin pisarse | El robot necesita poder alcanzar el HEC de Splunk (§8) |
| Ancho de banda trivial: ~45 MB/día curados. Ideal para un enlace satelital | Depende del reloj del robot (§7.4) |
| **No hace falta trunk, ni VLAN por robot, ni sub-interfaces, ni VM collector.** Todo eso existía para darle presencia L2 a un lector remoto — problema que se elimina | |
| El SDK trae **libs aarch64 precompiladas** → compilar en el Jetson es un trámite | |
| Ya está medido que PC2 del G1 lee el DDS de PC1 por el bus interno **con el cable externo desenchufado** (0,16 ms) | |

**Opción B — Túnel L2 (extender la red del robot por internet)** ❌

Traer la `192.168.123.0/24` del robot hasta el server por un túnel (ZeroTier con bridging L2,
WireGuard + bridge, etc.), para que el lector remoto siga creyendo que está en la misma red.

| Pros | Contras |
|---|---|
| No se toca nada en el robot ni en el server | **DDS sobre WAN se degrada muy mal**: RTT alto, jitter y pérdida, con QoS RELIABLE = tormentas de retransmisión |
| Conceptualmente simple | **Manda los tópicos completos por el enlace**: los 2,2 MB/s de `rt/lowstate` cruzarían el Starlink |
| | **Imposible para dos robots**: dos túneles presentando `192.168.123.161` cada uno = conflicto de IP en el server. Sin salida |
| | El discovery multicast sobre túnel es frágil |

**Opción C — DDS Router / bridge Zenoh (`zenoh-bridge-dds`, eProsima DDS Router)** 🟡 archivada

Herramientas hechas exactamente para puentear DDS por WAN sin extender L2.

| Pros | Contras |
|---|---|
| Preserva la semántica de tópicos: si algún día hace falta **comandar** el robot en el campo o que AI-VL lo vea, es la herramienta correcta | Igual **hay que correr un proceso en el robot** — mismo costo de despliegue que la Opción A |
| Bidireccional | Reenvía tópicos completos → mucho más ancho de banda que campos curados |
| Maduro y usado en producción | Una pieza más que mantener, para un problema que la Opción A ya resuelve |

**Veredicto:** para telemetría a Splunk es sobredimensionado. **Queda anotada como el camino
correcto si más adelante se quiere teleoperación o AI-VL contra un robot remoto** — esa
conversación es distinta y esta es la respuesta.

**Opción D — VM collector con presencia L2 en la red del robot** ❌ descartada por el requisito

Era la recomendación de la revisión anterior (VM `Splunk-collector` en el portgroup
`TRUNK ITINERANTE`, VLAN 4095/VGT, con una sub-interfaz taggeada por robot). Técnicamente
correcta, pero **solo sirve si el robot está en una VLAN que llega al server** — exactamente lo
que el requisito nuevo prohíbe asumir. Se descarta como arquitectura.

> Lo que sí sobrevive de esa investigación: si algún robot **está** en la LAN local, el mismo
> agente puede correr fuera del robot sin cambiar una línea de código (todo es configuración).
> Y para el trial, el agente va a correr en esta PC justamente así (§11).

### 2.2. Arquitectura resultante

```
        ┌─────────── ROBOT (en cualquier red) ────────────┐
        │                                                  │
        │  bajo nivel .123.161 ──DDS──▶ agente             │        ┌──────────┐
        │  (bus interno, nunca sale)    (Jetson .123.18)   │        │  Splunk  │
        │                                     │            │        │  HEC     │
        └─────────────────────────────────────┼────────────┘        └────▲─────┘
                                              │                          │
                                              └── HTTPS saliente ────────┘
                                             (Starlink / LTE / WiFi, vía §8)

  DDS: corto, local, adentro del robot.      HTTPS: largo, ruteado, atraviesa NAT.
```

**DDS corto y local, HTTPS largo y ruteado.** Es la misma tesis de `RED-Y-DDS.md` §8, llevada
a su conclusión: si el DDS tiene que ser local, entonces el lector va donde está el DDS.

### 2.3. Dónde corre, por robot

| Robot | Host del agente | Acceso | Estado |
|---|---|---|---|
| **Go2** | Jetson `192.168.123.18` | SSH `unitree` | ✅ **Inventariado 2026-08-19 — viable** (abajo) |
| **G1** | PC2 / Jetson `192.168.123.164` (aarch64, Ubuntu 20.04) | SSH | Lectura de DDS de PC1 por bus interno **ya validada** |

### Inventario del Jetson del Go2 (verificado 2026-08-19)

| Aspecto | Valor | Implicancia |
|---|---|---|
| Arquitectura / OS | **aarch64**, Ubuntu 20.04.5, kernel 5.10.104-tegra | El SDK trae lib aarch64 precompilada → compila directo |
| Toolchain | **g++ 9.4.0, cmake, make, git** | No hay que instalar nada para compilar |
| Recursos | 4 cores, **15,4 GB RAM (14,2 libres)**, **469 GB disco (429 libres)** | Sobra. El spool puede ser generoso |
| Carga | **load average 0,00** — idle | El agente no compite con nada |
| Ruteo | `default via 192.168.123.1 dev eth0` | **Tiene default gateway** → puede salir de la 123 |
| Internet | ping 8.8.8.8 en **2,86 ms**, DNS resolviendo | **Puede alcanzar Splunk** (y un futuro endpoint remoto) |
| Reloj | NTP **activo y sincronizado**; TZ Asia/Shanghai (fábrica) | Correcto en UTC. El agente debe emitir **epoch**, y la TZ es irrelevante |
| ROS2 | Foxy (y ROS1 Noetic) instalados | No se usan: el SDK nativo evita depender de ellos |
| **Software de terceros ya corriendo** | contenedor `go2-jetson-01` = **ThousandEyes enterprise-agent (Cisco)** | **Hay precedente de agentes de terceros en el robot, desplegados como Docker** |

Ese último punto es el más importante: **el riesgo de "no nos dejan instalar software en el robot"
ya está resuelto en la práctica**, y además muestra el patrón aceptado (contenedor Docker).

Nota sobre el G1: **PC1 (`.161`) no tiene SSH en ningún puerto**, así que ahí no se puede
desplegar nada — pero no hace falta: PC2 lee el DDS de PC1 por el bus interno. Y con el camino
del SDK nativo, el ROS2 Foxy pelado de PC2 deja de ser un problema (§7.1).

---

## 3. Inventario: qué tenemos y qué no

### 3.1. Ya funciona, no hay que hacerlo

| Pieza | Dónde | Estado |
|---|---|---|
| Captura de video del robot → RTSP/HLS/WebRTC | `robot-nvr-bridge` (mediamtx + ffmpeg) | Funcionando, con systemd y auto-reinicio |
| NVR con grabación continua y timeline | `robot-nvr-bridge/frigate` | Funcionando, 3 días de retención |
| Patrón de compilación contra el SDK | `robot-nvr-bridge/build.sh` | Probado en Ubuntu 26.04 |
| **Libs del SDK para aarch64** | `unitree_sdk2/lib/aarch64/` + `thirdparty/lib/aarch64/` | Precompiladas; el `CMakeLists.txt` elige por arquitectura |
| Tipos de mensaje de **los dos** robots | `unitree_sdk2/include/unitree/idl/{go2,hg}/` | `LowState_`, `IMUState_`, `BmsState_`, `MotorState_`, `SportModeState_` |
| Transporte DDS parametrizado | `unitree_ros2/setup.sh` + `dds.env` | Referencia de config |
| Camino de esta PC a Splunk | verificado 2026-08-18 | vía `192.168.123.1`, **0.55 ms**, 8000 y 8089 abiertos |

### 3.2. Falta, y hay que construirlo

| Falta | Comentario |
|---|---|
| **El agente de telemetría** | No existe nada. Hoy **nada del stack lee `/lowstate`**. Código nuevo de verdad. |
| **HEC habilitado en Splunk** | Puerto 8088 **cerrado** — verificado. Primera acción del lado de Splunk. |
| Index + token HEC | No creados. |
| Abrir tcp/8088 hacia `10.1.254.0/24` en el firewall de HQ | El transporte ya existe (VPN del IR1101, §8); solo falta el puerto |


| Dashboard | No existe. |

### 3.3. Cosas descartadas (y por qué)

| Descartado | Motivo |
|---|---|
| Instalar ROS2 en una VM nueva | Ubuntu 26.04 no tiene Humble, y `ros-humble-desktop` pelado **no trae los msgs de Unitree**. Se usa el SDK nativo (§7.1) |
| Agente que se suscribe a **todos** los tópicos | Rompe el presupuesto por 20-1700x y mandaría el video a Splunk |
| Un POST HEC por mensaje, síncrono, en el callback | Bloquea el receptor DDS y pierde muestras |
| Indexar `/rosout` | Los servicios del robot son *bare DDS apps*, no nodos ROS2 → no loguean ahí |
| Iframe en Dashboard Studio | No tiene panel HTML; eso es Simple XML (§10) |
| **VM collector con presencia L2 / VLAN por robot / trunk** | Incompatible con el requisito de movilidad (§2.1 Opción D) |
| **Túnel L2 para traer la red del robot** | §2.1 Opción B — imposible para dos robots por el conflicto de `.161` |
| `timechart avg(data.velocity)` sobre `/joint_states` | Es un array de N joints, no un escalar. Hay que aplanar por joint |

---

## 4. El conflicto de IP: neutralizado, no resuelto

Los dos robots usan `192.168.123.161` para el bajo nivel (Go2 alto nivel `.18`, G1 alto nivel
`.164`). Detalle completo en `RED-Y-DDS.md` §4.

**Con el agente adentro del robot, esto deja de ser un problema para la telemetría:** cada
`.161` queda privado dentro de su robot y nunca se expone. N robots pueden tener la misma IP
sin pisarse.

**Pero sigue siendo un problema para todo lo demás.** Si los dos robots se conectan a la vez a
la misma LAN local, sigue habiendo conflicto de IP para el pipeline de video y para AI-VL, que
leen DDS desde afuera del robot. Ese tema sigue abierto en `robot-nvr-bridge/docs/DOS-ROBOTS.md`.
Este plan lo **esquiva**, no lo arregla.

---

## 5. Restricciones duras

### 5.1. Presupuesto de licencia

**500 MB/día** hasta el **25 de agosto** (trial de Splunk Enterprise), **compartido con otra
persona** que está armando otro dashboard — así que el disponible real es menor y desconocido.
Splunk licencia **bytes crudos ingestados**: el tamaño del JSON *es* el consumo.

500 MB/día = **6 KB/s sostenidos**.

| Enfoque | Volumen/día | vs. presupuesto |
|---|---|---|
| `/lowstate` del **Go2** (medido: 500 Hz, 1,18 KB, 593 KB/s) tal cual, en crudo | **51 GB** | **100x** |
| `/lf/lowstate` del Go2 (20 Hz, 23,8 KB/s) tal cual, en crudo | 2,06 GB | **4x** |
| `/lf/lowstate` expandido a JSON (lo que Splunk cobra) | ~6-10 GB | **12-20x** |
| *(referencia G1, medido antes: `/lowstate` a 1041 Hz / 2,2 MB/s)* | *~190 GB* | *~380x* |
| Campos curados a 3 s (§6) | **~45 MB** | **9%** |

El número clave es el tercero: **ni el tópico liviano entra, una vez expandido a JSON.** No hay
downsample de tópicos que salve el enfoque de reenvío; hay que extraer campos. (Todo medido sobre
el Go2 real — ver `CENSO-GO2.md`.)

**Mitigación:** el agente lleva su **propio contador de bytes diario** con techo configurable
(arranca en **150 MB/día**). Al llegar, deja de enviar y lo loguea. Así no existe el escenario
de comernos la licencia compartida.

> La **retención no reduce el consumo de licencia**: la licencia cuenta ingesta, no
> almacenamiento.

### 5.2. Deadline

El trial vence el **25/08**. Prioridad: que el dato llegue y se vea; la prolijidad de infra
después. Y con los dos robots apagados, el despliegue en el robot **no se puede hacer todavía**
— de ahí el orden de §11.

### 5.3. DDS no cruza routers

Discovery multicast link-local + el robot anuncia **solo locators `192.168.123.x`** (3022
medidos, cero en otra subred) + probablemente sin default gateway. Detalle en `RED-Y-DDS.md`
§2-§3. **Esta restricción es la que motiva toda la Decisión 2.**

---

## 6. Contrato de datos

Todo al índice `go2-robot-data`, con `robot=<nombre>` y **`time` del reloj del evento** (no de
recepción — sin eso no hay correlación con el video, y con buffering en el campo los eventos
pueden llegar minutos tarde).

| Sourcetype | Origen DDS | Cadencia | Tamaño est. | Día est. |
|---|---|---|---|---|
| `robot:vitals` | **`/lf/lowstate`** → `bms_state`, `imu_state`, `power_v/a`, temps, `bit_flag` | 3 s | **378 B** | **10,4 MB** |
| `robot:motors` | **`/lf/lowstate`** → `motor_state[]` (12 del Go2): `q`, `tau_est`, `temperature`, `lost` | 3 s | **755 B** | **20,8 MB** |
| `robot:pose` | **`/lf/sportmodestate`** → `position`, `velocity`, `yaw_speed`, `body_height`, `gait_type`, `mode` | 3 s | **246 B** | **6,8 MB** |
| `robot:health` | derivado: Hz por tópico, `dds_alive`, bytes enviados vs cap, **backlog del spool** | 10 s | **237 B** | **2,0 MB** |
| `robot:event` | **`/lf/battery_alarm`** + cambio de `mode`/`gait_type`, `error_code` ≠ 0, temp > umbral, robot caído/vuelto, enlace caído/vuelto | por evento | ~300 B | ~1 MB |
| `robot:event` | discretos (ver abajo) | por evento | ~250 B | ~1 MB |
| **Total** | | | | **40,0 MB — 8% del presupuesto** |

> **Medido, no estimado** (2026-08-19): tamaños reales del JSON del colector de prueba
> corriendo contra el Go2. Procedimiento en `IMPLEMENTACION.md` Etapa B.

Notas de diseño:

- **`robot:event` no se downsamplea.** Ahí está el valor: un pico de temperatura de 200 ms se ve
  igual aunque la base vaya a 3 s.
- **`robot:motors` va aplanado por joint** (`motors.FL_hip.temp`, no un array), si no no se puede
  graficar. Es el error del `avg(data.velocity)` del plan original.
- **`robot:health` da el panel de "sensor caído"** sin indexar los datos del sensor: mide la tasa
  de los tópicos pesados y reporta el número. Con el agente en el campo, además reporta **estado
  del enlace y cuánto backlog tiene el spool** — telemetría del propio agente.
- **Denylist explícita de tópicos binarios** (`rt/frontvideostream`, `rt/api/videohub/response`,
  point clouds, `rt/utlidar/*`): jamás se serializan.
- **El G1 usa `unitree_hg::LowState_`, el Go2 `unitree_go::LowState_`.** Mapeo por tipo de robot,
  nombres de salida normalizados para que el dashboard sea el mismo.
- **Suscribir `/lf/*`, NO los tópicos de alta frecuencia.** Medido en el Go2 el 2026-08-19:
  `/lf/lowstate` trae **los mismos datos a 20 Hz** que `/lowstate` a 500 Hz, con el mismo tamaño
  de mensaje (1,18 KB) — **25x menos tráfico DDS, dato idéntico**. Censo completo en
  `CENSO-GO2.md`.
- **`round()` sobre un `float32` de numpy no redondea**: devuelve un numpy float que serializa
  como `-0.014299999922513962` en vez de `-0.0143`. Hay que hacer `round(float(x), 4)`. Detectado
  midiendo — inflaba el JSON sin necesidad.

---

## 7. El agente

### 7.1. Por qué SDK nativo y no ROS2

| | SDK nativo (C++) | ROS2 (rclpy) |
|---|---|---|
| Corre en Ubuntu 26.04 (esta PC) | Sí, probado en `robot-nvr-bridge` | No — solo en Docker (7,67 GB) |
| **Corre en el Jetson del robot** | **Sí: libs aarch64 precompiladas en el repo** | PC2 tiene Foxy pelado, sin CycloneDDS como RMW ni los msgs de Unitree |
| Dependencias a instalar en el robot | El SDK y nada más | ROS2 + CycloneDDS 0.10.x + 3 paquetes de msgs compilados |
| Tipos de los dos robots | `idl/go2/` y `idl/hg/`, ya en el repo | Hay que buildear `unitree_go` + `unitree_hg` + `unitree_api` |
| Introspección genérica de mensajes | No tiene | Sí (`message_to_ordereddict`) |

La última fila parece una ventaja de ROS2 pero **no nos sirve**: §6 elige campos a mano. Y con
la Decisión 2, la fila que decide es la segunda: **meter ROS2 en el robot es invasivo y frágil;
el SDK nativo es un binario estático.**

### 7.2. Componentes

```
telemetry_reader (C++, en el robot)          hec_shipper (en el robot)
├─ CycloneDDS bindeado a la interfaz          ├─ lee NDJSON de stdin
│  interna del robot                          ├─ cola en memoria + SPOOL EN DISCO acotado
├─ suscribe rt/lowstate, rt/sportmodestate    ├─ batching: N eventos o T ms por POST
├─ QoS BEST_EFFORT (no RELIABLE)              ├─ campo `time` del evento, no de envío
├─ decima a la cadencia configurada           ├─ contador de bytes diario + cap
├─ extrae SOLO los campos del contrato        ├─ reintento con backoff, sin flood de logs
└─ emite una línea JSON por evento            └─ POST a /services/collector/event
```

Decisiones y por qué:

- **QoS BEST_EFFORT en el lector.** Compatible con escritores RELIABLE (esa dirección matchea) y
  evita tormentas de NACK. Con RELIABLE + depth 10 además no matchea escritores BEST_EFFORT.
- **Decimar en el lector, no en el shipper.** Lo que no se serializa no cuesta nada.
- **Spool en disco acotado (nuevo, por el requisito de movilidad).** En el campo el enlace se
  corta. El agente escribe a un ring buffer en disco (arranca en 50 MB) y drena cuando vuelve la
  conectividad. Como cada evento lleva su `time`, los eventos atrasados **caen en el timestamp
  correcto** en Splunk. Sin el spool, un corte de enlace = agujero permanente.
- **Cola en memoria que descarta lo viejo** entre el reader y el spool: el agente nunca puede
  frenar la recepción DDS ni crecer sin límite.
- **Todo por env vars** (`ROBOT_NAME`, `ROBOT_TYPE`, `DDS_IFACE`, `HEC_URL`, `HEC_TOKEN`,
  `RATE_*`, `DAILY_BYTE_CAP`, `SPOOL_MB`): es lo que permite correrlo en el robot, en esta PC o
  en una VM **sin cambiar código**.
- **systemd con `Restart=always`**, igual que `robot-nvr.service`, que ya demostró recuperarse
  solo cuando el robot se cae y vuelve.

### 7.3. Reglas de convivencia con el robot

El agente corre en una computadora que hace mover un robot. No negociable:

- **Read-only:** suscribe y nada más. No publica en **ningún** tópico de comando. Sin excepción.
- **Límites de recursos** por systemd (`MemoryMax`, `CPUQuota`) para que no pueda competir con el
  control.
- **Nice bajo** y `Restart=always` pero con `StartLimitBurst` para no entrar en loop de crasheo.
- Escribe **solo** en su directorio de spool, con tamaño acotado. Nunca puede llenar el disco del
  robot.
- Se instala **sin tocar nada existente** del robot: su propio directorio, su propio unit.

### 7.4. El reloj

Si el reloj del robot está corrido, los eventos caen con timestamp equivocado y el dashboard
miente. **Verificado 2026-08-19 en el Jetson del Go2: NTP activo y sincronizado, hora correcta en
UTC** — la zona horaria es Asia/Shanghai (default de fábrica), lo que **no importa** siempre que el
agente emita **epoch** y no una hora local formateada. Igual el agente reporta su offset en
`robot:health`, para que un robot con el reloj corrido se detecte en el dashboard en vez de mentir
en silencio.

### 7.5. Dónde vive el código

Repo nuevo `robot-splunk-bridge`, hermano de los otros en `~/Desktop`, con dos targets de build
(x86_64 para pruebas locales, aarch64 para el robot). **Código y comentarios en inglés**
(convención del ecosistema); los docs de planificación en castellano.

---

## 8. El camino del robot a Splunk — ya resuelto

Cuando escribí este plan lo puse como la decisión abierta principal. **No lo es: ya existe.**
Está documentado en `PLAN-CONECTIVIDAD-ROBOTS.md`, que es la autoridad sobre la parte de red:

| Pieza | Estado |
|---|---|
| **VPN IKEv2 IR1101 → Meraki MX (HQ)** | **Operativa** |
| NAT del Jetson: `192.168.123.18` → `10.1.254.18` | **Operativa**, ping desde HQ OK |
| Uplink de campo | **Starlink Mini** (bypass de CGNAT, IP SLA keepalive configurado) |
| Contenedor ThousandEyes (IOx) en el IR1101 | RUNNING, **intocable** |

Y es exactamente la forma correcta según la tesis del diseño: **el túnel lleva HTTPS, no DDS.**
El IR1101 le da al robot un segmento L2 propio que viaja con él, con salida VPN — así que el
agente publica al HEC desde cualquier lado sin que DDS cruce nunca el router.

**Verificado hoy (2026-08-19) desde el Jetson del Go2, por cable:**

| Prueba | Resultado |
|---|---|
| Jetson → Splunk `192.168.20.200` | **0,74 ms** |
| Jetson → `192.168.20.200:8000` | abierto |
| Jetson → `192.168.20.200:8088` (HEC) | **cerrado** ← único bloqueo real |
| Jetson → bajo nivel `192.168.123.161` | **0,25 ms** (mismo L2 → el DDS va a funcionar) |

O sea: **el agente tiene los dos lados resueltos.** Ve el DDS del bajo nivel y alcanza Splunk.
Lo único que falta del lado de la red es abrir tcp/8088 hacia `10.1.254.0/24` en el firewall de
HQ, junto con habilitar el HEC (§9).

## 9. Splunk

1. **Habilitar HEC** — hoy el 8088 está cerrado. Settings → Data Inputs → HTTP Event Collector →
   Global Settings → habilitar, SSL on.
2. **Index `go2-robot-data`**, retención 2 días (`frozenTimePeriodInSecs = 172800`).
3. **Un índice y un token POR ROBOT** — es lo que se construyó (2026-08-19): token `Go2-01` con
   índice `go2-robot-data`. Difiere de lo que yo había recomendado (un índice compartido con el
   campo `robot`), y está bien: da **retención y cuota independientes por robot** y permite revocar
   un token sin afectar a los demás. El costo es que las búsquedas entre robots necesitan comodín
   (`index=*robot-data`). El campo `robot` se sigue mandando igual, así que ninguna búsqueda se
   rompe.
4. **Confirmar qué index/sourcetype usa la otra persona** para no pisarle nada.
5. **Prueba con `curl` antes de meter DDS en el medio.** Valida red, token e index sin ninguna
   variable de robot encima. El paso 4 del checklist original está bien puesto.
6. Restringir el 8088 al origen que corresponda (rango de la VPN, no "cualquiera").
7. **Con el agente en el robot, el token viaja en el robot.** Si un robot se pierde o se
   compromete, hay que poder revocarlo — resuelto por el punto 3 (token por robot). ⚠️ Y la
   password SSH del Jetson es `123`: cambiarla antes de dejar el token ahí.

### 9.1. Estado verificado (2026-08-19)

| Prueba | Resultado |
|---|---|
| `GET /services/collector/health` | `HEC is healthy` |
| Token `Go2-01` | válido |
| POST a `index=go2-robot-data` | **Success** |
| PoC contra el HEC | **36 eventos, 15.944 B, 0 errores** |

**Trampa que costó dos intentos:** si el índice que mandás no está en la lista permitida del
token, Splunk responde `{"text":"Incorrect index","code":7}` — **incluso para `main`**. Y si
omitís el campo `index`, usa el default del token y da `Success`. Así que "Success sin index +
Incorrect index con cualquier nombre" significa **nombre de índice equivocado o no permitido**,
no un problema de token ni de red.

---

## 10. Video

**Hoy ya funciona y cuesta 0 MB de licencia**, con el robot en la LAN local: mediamtx expone HLS
`:8888` y WebRTC `:8889` desde `robot-nvr-bridge`, y el panel es un `<iframe>` a
`http://192.168.123.99:8889/robot`.

Dos avisos de Splunk:

- **Simple XML con panel `<html>`, no Dashboard Studio** (que no tiene panel HTML). Al revés de
  lo que decía el plan original.
- Puede hacer falta tocar `web.conf` por CSP / `X-Frame-Options`.

**El caso del campo es harina de otro costal** y queda fuera de este plan: hoy la cadena
JPEG→H.264→RTSP corre **en esta PC**, leyendo el video por DDS desde la LAN. Con el robot remoto
habría que **encodear en el robot y empujar hacia afuera** (RTSP push / SRT / WebRTC hacia
mediamtx). Dos notas para cuando se encare:

- El Jetson del robot tiene encoder por hardware, y el **Go2 ya genera H.264 nativo**
  (`rt/frontvideostream`). El fracaso conocido de leer ese tópico fue **desde afuera del robot**;
  leerlo **localmente en el Jetson** es un test distinto y podría dar H.264 sin recodificar. Vale
  la pena probarlo, no es una promesa.
- El ancho de banda del video (Mbps) no tiene nada que ver con el de la telemetría (KB/s). Un
  enlace satelital aguanta 1080p, pero con latencia y jitter — hay que bajar expectativas de
  "vivo".

---

## 11. Plan de ejecución

**Los dos robots están apagados**, así que el despliegue en el robot no se puede empezar. El
orden aprovecha eso: se construye y se prueba local, y el despliegue al robot es el último paso
— y **no cambia ni una línea de código**, solo variables de entorno.

### Hasta el 25/08 (con el Go2 por cable en la LAN)

| # | Paso | Necesita | Bloquea a |
|---|---|---|---|
| 0 | Habilitar HEC + index + token + `curl` de prueba | Acceso admin a Splunk | todo |
| 1 | ~~**Censo de tópicos del Go2**~~ ✅ **hecho 2026-08-19** → `CENSO-GO2.md` | — | — |
| 2 | `telemetry_reader` (C++): DDS → NDJSON curado | paso 1 | 3 |
| 3 | `hec_shipper`: cola, spool, batching, `time`, cap de bytes | paso 0 | 4 |
| 4 | Correr el agente **en esta PC** contra el robot por cable, verificar los 5 sourcetypes en Splunk | pasos 2, 3 | 5 |
| 5 | Dashboard Simple XML: vitals + motores + errores + posición + iframe de video | paso 4 | — |
| 6 | Medir consumo real 24 h y ajustar cadencias | paso 4 | — |

### Después: llevarlo al robot

| # | Paso |
|---|---|
| 7 | ~~Inventariar el Jetson del Go2~~ ✅ **hecho 2026-08-19** — viable (ver §2.3) |
| 8 | Compilar el agente para aarch64 y desplegarlo en el Jetson con su unit de systemd y sus límites (§7.3) |
| 9 | Validar **con el cable externo desenchufado** — es el único test que vale (ver los falsos positivos en `RED-Y-DDS.md` §6) |
| 10 | Resolver el camino a Splunk desde afuera (§8): VPN en el robot |
| 11 | Prueba de campo real: cortar el enlace a propósito y confirmar que el spool drena sin perder datos ni duplicar |
| 12 | Repetir para el G1 en PC2 (`.164`) |
| 13 | Alertas (ojo: si el trial cae a Splunk Free, **se pierde alerting**) |

---

## 12. Decisiones abiertas

- [x] ~~Camino del robot al HEC desde afuera~~ — **resuelto**: VPN IKEv2 del IR1101 al Meraki MX,
      ya operativa (§8). Falta abrir tcp/8088 hacia `10.1.254.0/24` en el firewall de HQ.
- [ ] ¿Un token HEC por robot (revocable) o uno compartido? Recomiendo por robot (§9.7).
- [x] ~~Acceso al Jetson del Go2~~ — SSH `unitree`, inventariado, y ya corre un agente de terceros.
- [ ] **Cambiar la password del Jetson** (hoy es `123`) antes de dejar un token de Splunk ahí.
- [ ] Presupuesto post-25/08: ¿se compra más volumen? ¿Y qué queda — si cae a Free, se pierden
      alerting y autenticación, y el alerting es justo lo que querríamos para temperatura crítica.
- [ ] Cuánto del presupuesto consume la otra persona (define el cap real del agente).
- [ ] Certificado de Splunk: propio o autofirmado (define si el shipper valida TLS).
- [ ] Video en el campo: fuera de alcance de este plan, decidir si se encara (§10).
- [ ] Separación de los dos robots en la LAN local: este plan lo **esquiva**, pero sigue abierto
      para video y AI-VL (§4).

---

## 13. Riesgos

| Riesgo | Impacto | Mitigación |
|---|---|---|
| **El agente desestabiliza el robot** | Grave: es la computadora que lo hace caminar | §7.3: read-only, límites por systemd, spool acotado, instalación aislada |
| Nos comemos la licencia compartida | Rompe el dashboard del otro usuario | Cap de bytes diario **en el agente**, no solo confiar en las cadencias |
| El enlace del campo se corta | Agujeros en la telemetría | Spool en disco + `time` por evento → los datos llegan tarde pero **al timestamp correcto** |
| El reloj del robot está corrido | El dashboard miente y no se puede correlacionar | ✅ Verificado 2026-08-19: NTP activo y sincronizado en el Jetson del Go2. Igual el agente emite **epoch** y reporta offset en `robot:health` |
| El token HEC viaja en el robot | Un robot perdido = credencial expuesta | Token por robot, revocable (§9.7) |
| El robot se cae/vuelve seguido (documentado como intermitente) | Huecos | `Restart=always` + `robot:event` de caída/retorno: el hueco se vuelve un dato |
| Sobrecalentamiento del G1 (~104 °C, se apaga solo) | Aplica cuando entre el G1 | Es justamente una de las métricas a monitorear |
| El trial vence con el dashboard a medio hacer | Hay que rehacer la demo | Orden de §11: dato en Splunk primero, robot después |
| ~~No nos dejan instalar software en el robot~~ | ~~Se cae la arquitectura entera~~ | ✅ **Resuelto 2026-08-19**: ya corre un ThousandEyes agent (Cisco) en el Jetson del Go2, como contenedor Docker. Hay precedente y patrón |
| **La pass de SSH del robot es `123`** | Un token HEC guardado ahí queda muy expuesto | Token **por robot**, revocable (§9.7); permisos restrictivos en el archivo; plantear el cambio de credencial al dueño del robot |
