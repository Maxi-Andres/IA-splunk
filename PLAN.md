# Plan de telemetría de robots Unitree → Splunk

Revisión del plan original (`Telemetria-Splunk.md`) contra el estado real del stack que ya
corre en esta máquina. Fecha: 2026-08-18.

**Cambio conceptual respecto del plan original:** el diseño no puede ser *"reenviar topics
DDS a Splunk"*. Tiene que ser **"extraer campos específicos y mandarlos a tasa controlada"**.
El motivo es aritmético y está en §4.

---

## 1. Objetivo y alcance

Ver en un dashboard de Splunk, en una sola vista: **video en vivo, batería, temperaturas,
estado de motores, errores/faults y posición** de los robots Unitree.

- **Ahora:** un solo robot, el **Go2 (perro)**.
- **Después:** los dos (Go2 + G1). El diseño se hace multi-robot desde el día uno (campo
  `robot` en cada evento, config por robot), pero **no se implementa la parte de dos robots
  hasta que esté definida la topología de red** — ver §5.
- **Fuera de alcance:** indexar video o nubes de puntos en Splunk. El video se muestra
  embebido desde el pipeline que ya existe; no consume licencia.

---

## 2. Arquitectura

```
┌──────────────┐   DDS (CycloneDDS)   ┌────────────────────┐   HTTPS/HEC   ┌──────────┐
│  Go2 (perro) │ ───rt/lowstate──────▶│  telemetry-bridge  │ ─────────────▶│  Splunk  │
│ .123.161     │    rt/sportmodestate │  (SDK nativo C++   │  NDJSON       │ .20.200  │
└──────────────┘    rt/lidarstate     │   + shipper Python)│  batches      └──────────┘
                                      └────────────────────┘                    ▲
┌──────────────┐   DDS (videohub)     ┌────────────────────┐                    │ iframe
│  Go2 cámara  │ ───JPEG─────────────▶│ robot-nvr-bridge   │ ── HLS/WebRTC ─────┘
└──────────────┘                      │ (YA FUNCIONANDO)   │    :8888 / :8889
                                      └────────────────────┘
```

Dos caminos independientes que se cruzan solo en el dashboard, correlacionados por
timestamp. Si se cae uno, el otro sigue.

---

## 3. Inventario: qué tenemos y qué no

### 3.1. Ya funciona, no hay que hacerlo

| Pieza | Dónde | Estado |
|---|---|---|
| Captura de video del robot → RTSP/HLS/WebRTC | `robot-nvr-bridge` (mediamtx + ffmpeg) | Funcionando, con systemd y auto-reinicio |
| NVR con grabación continua y timeline | `robot-nvr-bridge/frigate` | Funcionando, 3 días de retención |
| SDK de Unitree compilado en Ubuntu 26.04 | `unitree_sdk2` + `robot-nvr-bridge/build.sh` | Probado — es el patrón a reusar |
| Tipos de mensaje de **los dos** robots | `unitree_sdk2/include/unitree/idl/{go2,hg}/` | `LowState_`, `IMUState_`, `BmsState_`, `MotorState_`, `SportModeState_` |
| ROS2 Humble + msgs de Unitree (alternativa) | devcontainer `unitree_ros2_devcontainer-...-humble` | Funcionando, pero ver §7.1 |
| Transporte DDS parametrizado (interfaz + peers unicast) | `unitree_ros2/setup.sh` + `dds.env` | Reusable como referencia de config |
| Camino de red de esta PC a Splunk | verificado 2026-08-18 | ruta vía `192.168.123.1`, **0.55 ms**, puertos 8000 y 8089 abiertos |
| Portgroup de trunk en el ESXi | `TRUNK ITINERANTE`, VLAN ID 4095 (VGT) | Ya existe, y `Splunk-collector` ya está atachada |

### 3.2. Falta, y hay que construirlo

| Falta | Comentario |
|---|---|
| **El bridge de telemetría** | No existe nada. Hoy **nada del stack lee `/lowstate`**: el executor solo publica comandos y el bridge de cámara solo lee imágenes. Es código nuevo de verdad. |
| **HEC habilitado en Splunk** | Puerto 8088 **cerrado** — verificado. Es la primera acción del lado de Splunk. |
| Index + token HEC | No creados. |
| Censo de topics del Go2 | Necesita el robot prendido. Define las tasas reales (§4). |
| Dashboard | No existe. |
| Definición de topología de red para dos robots | **Decisión abierta y bloqueante para el segundo robot** (§5.3). |

### 3.3. Cosas del plan original que hay que descartar

| Del plan original | Por qué no |
|---|---|
| Instalar ROS2 en una VM nueva (§3.1) | Ubuntu 26.04 no tiene Humble. Y `ros-humble-desktop` pelado **no trae los msgs de Unitree**, así que `get_message()` falla en todos los topics del robot. Usamos el SDK nativo (§7.1). |
| Bridge que se suscribe a **todos** los topics | Rompe el presupuesto por 20-1700x (§4) y mandaría el video a Splunk. |
| Un POST HEC por mensaje, síncrono, en el callback | Bloquea el receptor DDS y pierde muestras. Va con cola + batching (§7.2). |
| Indexar `/rosout` como fuente de logs (§4.5) | Los servicios del robot son *bare DDS apps* (`_CREATED_BY_BARE_DDS_APP_`), no nodos ROS2 → no loguean en `/rosout`. Los logs del robot salen por otro lado o no salen. |
| Panel de iframe en Dashboard Studio (§5.2) | Dashboard Studio no tiene panel HTML. El `<html>` embebido es de Simple XML (§9). |
| "Los robots ya están en segmentos separados" | Falso hoy. Ver §5.3 — y con el conflicto de IP es peor de lo que parecía. |
| `timechart avg(data.velocity)` sobre `/joint_states` | `velocity` es un array de N joints, no un escalar. Hay que aplanar por joint en el bridge. |

---

## 4. Restricciones duras (esto define el diseño)

### 4.1. Presupuesto de licencia

**500 MB/día**, hasta el **25 de agosto** (trial de Splunk Enterprise). Splunk licencia por
**bytes crudos ingestados**, así que el tamaño del JSON *es* el consumo. Y el presupuesto es
**compartido con otra persona** que está armando otro dashboard en la misma instancia — así
que el número real disponible es menor y desconocido.

La aritmética que mata al diseño original: 500 MB/día = **6 KB/s sostenidos**.

| Enfoque | Volumen/día | vs. presupuesto |
|---|---|---|
| `rt/lowstate` completo (medido: 1041 Hz, 2.2 MB/s) reenviado tal cual | ~860 GB | **1.700x** |
| `rt/lf/lowstate` (20 Hz, 42 KB/s) reenviado tal cual | ~11-18 GB | **20-35x** |
| Campos curados a 3 s (§6) | **~45 MB** | **9%** |

El número clave es el del medio: **incluso el topic "liviano", reenviado tal cual, es 20-35x
el presupuesto.** No hay downsample de topics que salve el enfoque de reenvío; hay que
extraer campos.

**Mitigación adicional:** el bridge lleva su **propio contador de bytes diario** con un techo
configurable (arranca en **150 MB/día**). Al llegar al techo deja de enviar y lo loguea. Así
no existe el escenario de comernos la licencia y romperle el dashboard al otro usuario. Con
45 MB estimados hay 3x de colchón antes de que el cap actúe.

> Aclaración porque se confunde seguido: **la retención no reduce el consumo de licencia.**
> La licencia cuenta lo ingestado, no lo almacenado. Poner 2 días de retención ahorra disco,
> no MB de licencia.

### 4.2. Deadline

El trial vence el **25/08** — 7 días. Eso ordena las prioridades: primero que el dato
**llegue y se vea**, después que la infra quede prolija. Todo lo que sea setup de
infraestructura que se pueda hacer después del 25, se hace después del 25.

### 4.3. Conflicto de IP entre los dos robots

Direcciones reales:

| Robot | Bajo nivel (DDS) | Alto nivel (SSH) |
|---|---|---|
| Go2 (perro) | `192.168.123.161` | `192.168.123.18` |
| G1 (humanoide) | `192.168.123.161` | `192.168.123.164` |

**Los dos robots usan `192.168.123.161`.** Esto es más grave que la colisión de topics DDS
que ya estaba documentada en `robot-nvr-bridge/docs/DOS-ROBOTS.md`: con los dos en el mismo
segmento L2 hay **conflicto de IP**, no solo mezcla de datos. Consecuencias:

- Los dos robots en la misma red **es imposible**, no "desprolijo". Cualquier medición hecha
  con los dos conectados a la vez es sospechosa retroactivamente.
- **Separar por VLAN pasa a ser obligatorio**, no una opción de diseño.
- La "Opción A" (dominio DDS por robot) del doc `DOS-ROBOTS.md` **no alcanza sola**: cambiar
  el dominio DDS no arregla un conflicto de IP.
- El único camino que resuelve las dos cosas de una es **una VLAN por robot** (la "Opción B"),
  porque la IP `.161` puede repetirse sin problema si está en segmentos L2 distintos.

### 4.4. DDS no cruza routers

El discovery de DDS es multicast y el multicast es link-local. Y además el robot **anuncia
únicamente locators `192.168.123.x`** (medidos 3022, cero en otra subred). Por lo tanto:
**quien lea DDS necesita presencia L2 en la red del robot.** Tener ruta no alcanza. Esto ya
lo comprobamos por las malas con el G1 por WiFi.

---

## 5. Dónde va el collector

### 5.1. Las opciones

**Opción 1 — Esta PC (`ia-pc`, `192.168.123.99`, `enp4s0`)**

| Pros | Contras |
|---|---|
| Ya tiene pata L2 en la red del robot | Es una **workstation**: se apaga, se reinicia, alguien la usa |
| SDK ya compilado y probado en Ubuntu 26.04 | Sin backup ni monitoreo de server |
| Camino a Splunk verificado (0.55 ms) | Para dos robots necesitaría una segunda NIC física o VLANs taggeadas hacia ella (hoy solo `enp4s0` activa; `wlp3s0` down) |
| El video ya sale de acá → todo en un lugar | Acopla telemetría y video: un problema afecta a los dos |
| **Setup: cero.** Se puede tener dato en Splunk hoy | |

**Opción 2 — VM `Splunk-collector` en `TRUNK ITINERANTE` (VLAN 4095 / VGT)**

| Pros | Contras |
|---|---|
| **Escala a N robots sin hardware**: una sub-interfaz taggeada por VLAN de robot (`ens160.X`), y CycloneDDS bindeado por interfaz → es exactamente la "Opción B" de `SEPARAR_ROBOTS_MULTIPLES.md` | Requiere que el puerto físico que alimenta `vmnic3` **trunkee la VLAN de los robots** — **a verificar** |
| Es un server: uptime, snapshots, backup | Hay que configurar netplan con VLANs taggeadas (el tagging lo hace el guest, no el vSwitch) |
| Ya está atachada al portgroup correcto — **no hay que crear nada en el vSwitch** | Si es Ubuntu 26.04 como la de Splunk: no hay ROS2 → SDK nativo (que es el camino elegido igual) |
| Misma vSwitch que Splunk: con una sub-interfaz en VLAN 20, el tráfico HEC **no sale del host** | Hay que compilar el SDK ahí (~1 h si sale bien) |
| Desacopla telemetría de video (fallas independientes) | **No puede hacer el video**: mediamtx/Frigate siguen en esta PC. No es problema, pero hay que saberlo |

**Opción 3 — VM en VLAN 20 sin trunk** — ❌ **No funciona.** Es el escenario de §4.4: el
multicast no cruza y el robot anuncia solo `123.x`. Descartada por evidencia, no por opinión.

**Opción 4 — Dentro del devcontainer Humble de esta PC** — sirve para prototipar (ROS2 ya
anda) pero es el peor destino final: depende de que VS Code levante un contenedor de 7,67 GB.

### 5.2. Recomendación

**Empezar en la Opción 1, migrar a la Opción 2 cuando se defina la topología de robots.**

Razones:

1. **El trial vence en 7 días.** Arrancar por la VM gasta el trial en setup de infra en vez de
   en validar que el dato sirve. En esta PC hay dato en Splunk el día 1.
2. **La decisión de topología todavía está abierta**, y es justo la que define cómo se
   configura la VM (qué VLANs taggear). Configurarla antes de esa decisión es trabajo que se
   rehace.
3. **La migración es barata si el bridge se escribe portable**: todo por variables de entorno
   (interfaz DDS, nombre del robot, URL y token de HEC, tasas, cap de bytes). Migrar = compilar
   el SDK en la VM + netplan + copiar el systemd unit. Nada de código cambia.

La Opción 2 **es el destino final correcto** — sobre todo por el punto de escalar a dos robots
con sub-interfaces taggeadas, que resuelve de una el conflicto de IP y la colisión de topics
sin comprar nada.

### 5.3. Decisión abierta que bloquea al segundo robot

Cómo se separan los dos robots. Con el conflicto de IP de §4.3, la respuesta casi forzada es
**una VLAN por robot, y el collector con una sub-interfaz taggeada en cada una**. Falta
confirmar:

- [ ] Qué **VLAN ID** tiene hoy el segmento de los robots (`192.168.123.0/24`). No es
      necesariamente 123 — el número de subred y el ID de VLAN son cosas distintas.
- [ ] Si esa VLAN llega **trunkeada** al puerto físico del ESXi (`vmnic3`).
- [ ] Qué VLAN nueva se usaría para el segundo robot.
- [ ] Cómo entra el G1 (el CURWB bridgea a nivel L2, así que cae en la VLAN del puerto donde
      está el Mesh End — eso define en qué VLAN termina el G1).

Mientras esto no esté definido: **un robot, el Go2, por cable.**

---

## 6. Contrato de datos (el corazón del diseño)

Cuatro sourcetypes, todos al index `robot_data`, todos con `robot=go2` y `time` del reloj del
evento (no de recepción — sin eso no se puede correlacionar con el video).

| Sourcetype | Origen DDS | Cadencia | Tamaño est. | Día est. |
|---|---|---|---|---|
| `robot:vitals` | `rt/lowstate` → `bms_state`, `imu_state`, `power_v/a`, temps, `bit_flag` | 3 s | ~500 B | 14 MB |
| `robot:motors` | `rt/lowstate` → `motor_state[]` (12 del Go2): `q`, `tau_est`, `temperature`, `lost` | 3 s | ~2 KB | 21 MB |
| `robot:pose` | `rt/sportmodestate` → `position`, `velocity`, `yaw_speed`, `body_height`, `gait_type`, `mode` | 3 s | ~250 B | 7 MB |
| `robot:health` | derivado: `last_seen` + Hz medido por topic + estado de lidar | 10 s | ~300 B | 3 MB |
| `robot:event` | discreto: cambio de `mode`/`gait_type`, `error_code` ≠ 0, temp > umbral, robot caído/vuelto | por evento | ~300 B | ~1 MB |
| **Total** | | | | **~45 MB (9%)** |

Notas de diseño:

- **`robot:event` no se downsamplea.** Ahí está el valor real: un pico de temperatura de
  200 ms se ve igual aunque la telemetría base vaya a 3 s.
- **`robot:motors` va aplanado por joint** (`motors.FL_hip.temp`, no un array), si no no se
  puede graficar en Splunk. Es el error del `avg(data.velocity)` del plan original.
- **`robot:health` es el que da el panel de "sensor caído"** sin indexar los datos del sensor:
  mide la tasa de los topics pesados y reporta solo el número.
- **Denylist explícita de topics binarios** (`rt/frontvideostream`, `rt/api/videohub/response`,
  point clouds, `rt/utlidar/*`): jamás se serializan. En el plan original habrían entrado
  como arrays JSON de miles de enteros.
- **El G1 usa `unitree_hg::LowState_`, el Go2 `unitree_go::LowState_`** (distinta cantidad de
  motores y campos). El bridge tiene un mapeo por tipo de robot; los nombres de campo de
  salida se normalizan para que el dashboard sea el mismo.
- Todos los números de esta tabla son **estimaciones proyectadas de mediciones del G1**. Se
  reemplazan por medidos en el paso 1 de §10.

---

## 7. El bridge

### 7.1. Por qué SDK nativo y no ROS2

| | SDK nativo (C++) | ROS2 (rclpy) |
|---|---|---|
| Corre en Ubuntu 26.04 | Sí, ya probado en `robot-nvr-bridge` | No — solo en Docker (imagen de 7,67 GB) |
| Dependencias en el collector | El SDK y nada más | ROS2 + CycloneDDS 0.10.x + 3 paquetes de msgs compilados |
| Tipos de los dos robots | `idl/go2/` y `idl/hg/`, ya en el repo | `unitree_go` + `unitree_hg` + `unitree_api`, hay que buildearlos |
| Introspección genérica de mensajes | No tiene | Sí (`message_to_ordereddict`) |

El último punto parece una ventaja de ROS2 pero **no nos sirve**: el diseño de §6 elige campos
a mano, no serializa mensajes enteros. Justamente lo que ROS2 aporta es lo que hay que evitar.

El patrón está probado: `robot-nvr-bridge/src/go2_jpeg_stream.cpp` ya compila contra
`libunitree_sdk2.a` en esta máquina y lee DDS del robot.

### 7.2. Componentes

Mismo patrón que `robot-nvr-bridge` (C++ captura → stdout → proceso que despacha):

```
telemetry_reader (C++)                      hec_shipper (Python)
├─ CycloneDDS bindeado a la interfaz         ├─ lee NDJSON de stdin
│  del robot (CYCLONEDDS_URI)                ├─ cola acotada (descarta lo viejo, no bloquea)
├─ suscribe rt/lowstate, rt/sportmodestate   ├─ batching: N eventos o T ms por POST
├─ QoS BEST_EFFORT (no RELIABLE)             ├─ campo `time` del evento
├─ decima a la cadencia configurada          ├─ contador de bytes diario + cap
├─ extrae SOLO los campos del contrato       ├─ reintento con backoff, sin flood de logs
└─ emite una línea JSON por evento           └─ POST a /services/collector/event
```

Decisiones y por qué:

- **QoS BEST_EFFORT en el lector.** Es compatible con escritores RELIABLE (esa dirección sí
  matchea) y evita las tormentas de NACK/retransmisión si algún día el enlace es inalámbrico.
  Con RELIABLE + depth 10, además, no matchea escritores BEST_EFFORT.
- **La decimación va en el lector, no en el shipper.** Descartar temprano: lo que no se
  serializa no cuesta nada.
- **Cola acotada que descarta lo viejo.** Si Splunk no responde, el bridge no puede frenar la
  recepción DDS ni crecer sin límite. Telemetría vieja no sirve; se tira.
- **`time` explícito en cada evento.** Sin esto Splunk sella con hora de recepción y se pierde
  la correlación con el video, que es la mitad del valor del dashboard.
- **Todo por env vars** (`ROBOT_NAME`, `ROBOT_TYPE`, `DDS_IFACE`, `HEC_URL`, `HEC_TOKEN`,
  `RATE_*`, `DAILY_BYTE_CAP`): es lo que hace barata la migración de §5.2.
- **systemd con `Restart=always`**, igual que `robot-nvr.service`, que ya demostró recuperarse
  solo cuando el robot se cae y vuelve.

### 7.3. Dónde vive el código

Repo nuevo `robot-splunk-bridge`, hermano de los otros en `~/Desktop`. **El código y sus
comentarios en inglés** (convención del ecosistema); este plan queda en castellano para
acompañar al doc original.

---

## 8. Splunk

1. **Habilitar HEC** — hoy el 8088 está cerrado. Settings → Data Inputs → HTTP Event
   Collector → Global Settings → habilitar, SSL on.
2. **Index `robot_data`** con retención de 2 días (`frozenTimePeriodInSecs = 172800`).
3. **Un token HEC compartido** para los dos robots (el campo `robot` los distingue). Es más
   simple de mantener que uno por robot — eso el plan original lo tiene bien.
4. **Confirmar qué index/sourcetype usa la otra persona** para no pisarle nada.
5. **Prueba con `curl` antes de meter DDS en el medio** — el paso 4 del checklist original
   está bien puesto y hay que respetarlo: valida el camino de red, el token y el index sin
   ninguna variable de robot encima.
6. Restringir el 8088 a la IP del collector.

---

## 9. Video

**Ya está resuelto, cuesta 0 MB de licencia.** mediamtx expone HLS en `:8888` y WebRTC en
`:8889`. El panel es un `<iframe>` a `http://192.168.123.99:8889/robot`.

Dos avisos:

- **Va en un dashboard Simple XML con panel `<html>`, no en Dashboard Studio** (que no tiene
  panel HTML). Al revés de lo que dice el plan original.
- Puede hacer falta tocar `web.conf` por CSP / `X-Frame-Options` para que Splunk permita el
  iframe.
- Quien mire el dashboard tiene que poder alcanzar `192.168.123.99:8889`. Si el dashboard se
  abre desde la VLAN 20, hay que confirmar que esa ruta existe en ese sentido (el sentido
  inverso ya está verificado).

Metadata de video **sí** puede ir a Splunk como eventos normales (inicio/fin de grabación,
detecciones de Frigate) — es texto, cuesta nada, y sirve para correlacionar.

---

## 10. Plan de ejecución hasta el 25/08

Ordenado para que el dato llegue rápido y la infra quede prolija después.

| # | Paso | Necesita | Bloquea a |
|---|---|---|---|
| 0 | Habilitar HEC + index + token + `curl` de prueba | Acceso admin a Splunk | todo |
| 1 | **Censo de topics del Go2**: 60 s de captura, tamaños y Hz reales por topic | **Robot prendido** (~2 min) | fijar las tasas de §6 con números medidos |
| 2 | `telemetry_reader` (C++): DDS → NDJSON curado por stdout | paso 1 | 3 |
| 3 | `hec_shipper` (Python): cola, batching, `time`, cap de bytes | paso 0 | 4 |
| 4 | systemd + verificar en Splunk que llegan los 5 sourcetypes | | 5 |
| 5 | Dashboard Simple XML: vitals + motores + errores + posición + iframe de video | paso 4, §9 | — |
| 6 | Medir consumo real 24 h y ajustar cadencias | paso 4 | — |

Después del 25 / cuando se defina la topología:

| # | Paso |
|---|---|
| 7 | Confirmar VLAN ID de la red de robots y si llega trunkeada a `vmnic3` |
| 8 | Migrar el bridge a la VM `Splunk-collector` (compilar SDK + netplan con VLAN taggeada + systemd) |
| 9 | Definir VLAN del segundo robot y levantar la segunda instancia del bridge |
| 10 | Alertas (ojo: si el trial cae a Splunk Free, **se pierde alerting**) |

---

## 11. Decisiones abiertas

- [ ] **VLAN ID** del segmento de robots, y si llega trunkeada a `vmnic3`. (§5.3)
- [ ] Topología para dos robots — forzada hacia "una VLAN por robot" por el conflicto de IP. (§4.3)
- [ ] Presupuesto de licencia post-25/08: ¿se compra más volumen o el diseño vive en 500 MB
      para siempre? Y **qué queda después del 25** — si cae a Free, se pierden alerting y
      autenticación, y el alerting es justo lo que querríamos para temperatura crítica.
- [ ] Cuánto del presupuesto consume la otra persona (define el cap real del bridge).
- [ ] Acceso: ¿creamos nosotros el index y el token, o lo hace infra?
- [ ] Certificado de Splunk: propio o autofirmado (define si el shipper valida TLS).

---

## 12. Riesgos

| Riesgo | Impacto | Mitigación |
|---|---|---|
| Nos comemos la licencia compartida | Le rompe el dashboard al otro usuario | Cap de bytes diario **en el bridge** (§4.1), no solo confiar en las cadencias |
| El robot se cae/vuelve seguido (ya documentado como intermitente) | Huecos en la telemetría | `Restart=always` + `robot:event` de caída/retorno, que convierte el hueco en un dato |
| Sobrecalentamiento del G1 (~104 °C, se apaga solo) | No aplica al Go2 ahora; sí cuando entre el G1 | Justamente es una de las métricas a monitorear — el dashboard sirve para esto |
| El trial vence con el dashboard a medio hacer | Hay que rehacer la demo | Orden de §10: dato en Splunk primero, prolijidad después |
| La VLAN de robots no llega trunkeada al ESXi | La Opción 2 se cae | El bridge queda en esta PC; el diseño no cambia, solo el host |
| MTU 1500 en el vSwitch con tagging del guest | Muestras DDS grandes podrían fragmentar mal | Riesgo bajo: la telemetría son muestras chicas. Si aparece, es sospechoso #1 |
