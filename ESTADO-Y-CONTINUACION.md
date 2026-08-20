# Estado del proyecto y cómo continuar

Documento de traspaso, escrito el **2026-08-20** al límite del contexto de la conversación
anterior. Lo que sigue es lo que está medido, lo que está roto, y lo que sigue.

Lectura previa recomendada, en este orden:
`RED-Y-DDS.md` (por qué el diseño es así) → `ARQUITECTURA-REMOTA.md` (la arquitectura) →
`IPS-Y-DONDE-CAMBIARLAS.md` (dónde está cada cosa) → este documento.

---

## 0. El problema abierto, en una línea

**El video ya es fluido (~12 fps de captura, 5-10 fps percibidos) pero se congela ~1 s cada
~4 s.** Todo lo demás funciona. Diagnóstico y experimentos en **§4.4** — hay una hipótesis
concreta y un switch para probarla en un paso.

---

## 1. Qué es esto

Tres agentes corren **adentro del robot** (en su Jetson de alto nivel) y mandan todo hacia
afuera. Nada entra al robot: ni un puerto abierto, ni IP fija.

| Agente | Qué hace | Servicio systemd |
|---|---|---|
| Telemetría | lee `rt/lf/lowstate` por DDS → HTTP al HEC de Splunk | `robot-splunk-bridge` |
| Video | lee la cámara por DDS → MJPEG por HTTP + H.264 por RTMP | `robot-video` |
| Comandos | recibe HTTP → publica DDS localmente | `robot-command-relay` |

**Los tres están `enabled`**, así que prender el robot alcanza.

**Por qué adentro del robot:** DDS no cruza fronteras de subred con estos robots. Medido:
122 tópicos desde su propia subred, **2** desde otra, **3** incluso con peers unicast por IP.
Detalle en `RED-Y-DDS.md`. Eso es lo que permite que el robot esté en cualquier red.

---

## 2. Direcciones y accesos

| Qué | Dónde |
|---|---|
| Robot — bajo nivel (publica el DDS) | `192.168.123.161` — **el G1 usa la misma**, ver `RED-Y-DDS.md` §4 |
| Robot — Jetson (SSH, corren los agentes) | `192.168.123.18`, NATeado por el IR1101 a **`10.1.254.18`** |
| Esta PC | `192.168.20.99` (cambió varias veces; `enp4s0`, perfil NetworkManager "Profile 1") |
| Splunk | `192.168.20.200` — `:8000` UI, `:8088` HEC |
| SSH al robot | `unitree` / password `123` ⚠️ **cambiarla, hay tokens adentro** |
| Splunk | índice `go2-robot-data`, token `Go2-01` |

Repos en el robot: `~/robot-splunk-bridge`, `~/robot-nvr-bridge`, `~/unitree_sdk2` (upstream).
**Los dos primeros compilan C++ y los dos necesitan `./build.sh` tras un `git pull`.**

---

## 3. Qué funciona

| Capacidad | Estado |
|---|---|
| Telemetría → Splunk | ✅ **40 MB/día medidos** (8% del presupuesto), spool en disco, cap de bytes |
| Dashboard de Splunk | ✅ paneles + video embebido con `<img>` sobre el MJPEG |
| Comandos remotos | ✅ relay con lista blanca, clamp de velocidad, dead-man de 1,5 s, auditoría |
| Switch DDS ↔ WAN | ✅ en la UI: menú Camera (fuente) y Net (transporte), por robot |
| Página "Robot" en AI-VL | ✅ muestra lo que el robot tiene fijo y cómo cambiarlo |
| Video | ⚠️ **fluido pero se congela ~1 s cada ~4 s** — ver §4.4 |

Contrato de datos de la telemetría en `PLAN.md` §6. Mediciones de tópicos en `CENSO-GO2.md`.

---

## 4. El problema del video: las tres restricciones medidas

Las tres son reales y se apilan. **No perseguir una sin tener en cuenta las otras.**

### 4.1. El enlace al robot: ~353 kB/s ⚠️ **verificar primero**

Medido de dos formas que coincidieron: `dd` por SSH dio **353 kB/s**, y el MJPEG nativo daba
235 KB/s. O sea ~2,8 Mbps.

| Qué | Necesita |
|---|---|
| MJPEG nativo 1080p @ 14 fps | **3,0 MB/s** — 8,5x más de lo que hay |
| MJPEG 640px @ 14 fps | 0,5 MB/s |
| H.264 del NVR @ 2 Mbps | 0,25 MB/s — **70% del enlace él solo** |

**Los dos caminos de video comparten ese enlace y lo saturan.** Esto explica el síntoma mejor
que cualquier bug.

⚠️ **PENDIENTE Y PRIORITARIO:** no se sabe **qué tramo** limita a 2,8 Mbps (¿la radio del
IR1101? ¿LTE? ¿el NAT?). Confirmar con `iperf3` y averiguar el camino físico. Si el enlace es
realmente así, es la restricción que domina **todo** el diseño y hay que planificar alrededor.
Si es un problema corregible, se arregla el video sin tocar código.

### 4.2. La cámara da 14 fps, no 3

Medido en el robot con el capturador solo: **14,0 fps a 221 KB/frame** con `MAXFPS=0`
(y con el servicio corriendo en paralelo compitiendo, así que el techo es mayor).
La cámara **nunca** fue el límite — eso fue una conclusión equivocada durante la sesión.

### 4.3. El reescalado con cv2 cuesta ~400 ms/frame

Techo del vivo ≈ **2,4 fps**, sin importar el ancho de banda. El reescalado corre en Python
con cv2 en el Jetson.

**El arreglo claro:** hacerlo por **hardware**. En el robot están instalados `nvjpegdec`,
`nvvidconv` y `nvjpegenc` (verificado). Sería ~10x más rápido. Requiere sacar el resize de
Python y meterlo en una rama de GStreamer, con el problema a resolver de cómo entregarle esos
JPEG chicos al servidor HTTP (fifo, `tcpserversink` + shim, o similar).

---

### 4.4. ⚠️ EL SÍNTOMA ABIERTO: se congela ~1 s cada ~4 s

Reportado el 2026-08-20 después de los arreglos: el video se ve fluido y de pronto se detiene
un momento, vuelve, y repite con un período de **unos 4 segundos**.

**Hipótesis principal, y encaja con el período:** las ráfagas de keyframe del H.264 saturan el
enlace y dejan sin ancho de banda al MJPEG, que lo comparte. `IDR_FRAMES=15` con `NVR_FPS=5`
es **un keyframe cada 3 s**, y un keyframe es mucho más grande que el resto de los frames.

**Experimentos, en orden de costo (todos son `nano robot/video.env` + restart, sin compilar):**

1. **`NVR_ENABLE=0`** — apaga la rama de grabación por completo. Si el congelamiento
   desaparece, está confirmado que es la grabación compitiendo por el enlace. Es la prueba
   decisiva y cuesta un minuto.
2. **`IDR_FRAMES=150`** (un keyframe cada 30 s a 5 fps). Si el período del congelamiento
   cambia o se alarga, son los keyframes.
3. **`BITRATE=300000`**. Si mejora, es contención de ancho de banda en general.
4. Si se confirma que son las ráfagas, el arreglo prolijo es **CBR en el encoder**:
   `nvv4l2h264enc` tiene `control-rate` y `peak-bitrate`. En CBR las ráfagas se aplanan en vez
   de competir. Hoy el pipeline no fija `control-rate`.

**Otras hipótesis, menos probables:** contención del GIL entre el hilo del reescalado y el
`pump()`; colapso de la ventana de congestión de TCP en un enlace saturado; Frigate
reconectándose al RTMP periódicamente.

**Lo que ya NO es:** no es la cámara (da 12-14 fps), no es CPU (load 0,3 de 4 cores), no es el
tee estrangulando (§4.5), y no es el descarte caótico de frames (§4.5).

---

### 4.5. Lecciones de los arreglos que ya se hicieron

Dos errores que costaron horas y que conviene no repetir:

**El reescalado corría dentro del loop de captura.** Un decode+resize+encode de 1080p por
frame bloqueaba al lector: la captura caía de 14 a 2,3 fps. Ahora corre en un hilo aparte que
procesa solo el frame más nuevo.

**El NVR podía estrangular la captura.** `rtmpsink` se bloquea cuando el uplink está lleno,
GStreamer deja de leer su stdin, el pipe se llena y un `write()` bloqueante propagaba el
atasco hasta la cámara. Ahora la rama del NVR tiene cola acotada y **cadencia fija**.

**Y el error que cometí al arreglar eso:** descarté "el frame más viejo cuando la cola se
llena". Eso mantiene la captura libre pero le entrega al **encoder** huecos irregulares, y con
`do-timestamp=true` los timestamps quedan erráticos: Frigate pasó a decir *"no frames have
been received"*. La distinción que faltaba:

> **Un decoder tolera frames perdidos. Un encoder necesita cadencia** — prefiere menos fps
> antes que fps irregulares.

Las dos ramas quieren cosas **opuestas**: el vivo quiere el frame más nuevo y no le importa
perder intermedios; la grabación quiere intervalo regular. Aplicarles la misma política rompe
una de las dos. Hoy el vivo toma "el más nuevo" y la grabación toma "uno cada 1/NVR_FPS".

---

## 5. La cadena de video, como está hoy

## 6. Lo que ya se descartó (no volver a intentar)

| Idea | Por qué no |
|---|---|
| **SRT del robot a mediamtx** | El robot trae **libsrt 1.4.0** (Ubuntu 20.04) y la implementación SRT propia de mediamtx **rechaza su handshake**. Bisecado: el mismo pipeline conecta bien contra un listener de libsrt, así que la incompatibilidad es libsrt↔mediamtx. `PROTO=srt` sigue cableado para el día que se actualice libsrt |
| **Leer RTSP de mediamtx con OpenCV** | 2 fps: decodificar 1080p H.264 en Python es CPU-bound. Peor que el MJPEG |
| **El vivo a través de Frigate** | Frigate es un NVR y bufferea a propósito: ~7 s de latencia. El vivo no debe pasar por la cadena de grabación |
| **`<iframe>` en el dashboard de Splunk** | Simple XML de Splunk 9 **elimina el tag**; no es CSP ni dominios de confianza, no hay setting. Se resolvió con un `<img>` sobre el MJPEG |
| **Lector DDS ruteado (desde otra subred)** | Medido tres veces: 122 → 2 → 3 tópicos. Ni con peers unicast |
| **Túnel L2 para traer la red del robot** | Imposible con dos robots: los dos son `.161`, conflicto de IP del lado del server |

---

## 7. Lo próximo, en orden

0. **Aislar el congelamiento periódico con `NVR_ENABLE=0`** (§4.4). Un minuto, y decide todo
   lo que sigue.
1. **Confirmar el ancho de banda del enlace con `iperf3`** y averiguar qué tramo lo limita
   (§4.1). Es lo más barato y puede resolver el video sin tocar código.
2. **Ajustar la config para el enlace real.** Si sigue en 2,8 Mbps: `MJPEG_WIDTH=640` y bajar
   `BITRATE` del NVR a ~600 kbps. Los dos caminos no entran a 2 Mbps.
3. **Reescalado por hardware** (§4.3) si el vivo sigue lento. Es la mejora estructural.
4. **Probar `rt/frontvideostream` LOCALMENTE en el robot.** El Go2 publica H.264 nativo a
   30 fps por su propio encoder. Ese tópico está documentado como fallido, pero **el fracaso
   fue siempre leyéndolo desde AFUERA del robot** — nunca se probó adentro, que es un
   escenario distinto (bus interno, sin fragmentación por red). Si funciona: 30 fps ya
   comprimido, sin polling del `videohub` y sin re-encodear. Hay un `go2_h264_stream` en el
   repo que quedó de aquel intento. **Es la mejora de mayor techo.**
5. **El `videorate` que falta en el pipeline del robot.** El pipeline de esta PC necesitaba
   `-vsync cfr -r 15` en ffmpeg porque los timestamps de los JPEG llegan irregulares; el
   equivalente en GStreamer nunca se aplicó. Sospechoso de los saltos del NVR ("3 fps bien,
   un salto, 3 fps bien"). Puede haber desaparecido solo al destrabar el tee — verificar
   antes de trabajar en esto.
6. Cambiar la password del Jetson.
7. Replicar todo en el **G1** (`unitree_hg` en vez de `unitree_go`; su PC2 es `.164`).
8. Primer `move` real supervisado por el relay. Ojo: **el robot tiene que estar parado** —
   echado (`mode: 0`, `body_height` 0.089) su servicio de sport devuelve **-1**.

---

## 8. Trampas que costaron tiempo (no repetirlas)

| Trampa | Detalle |
|---|---|
| `read(n)` en un pipe | **Bloquea hasta juntar los n bytes.** Con frames de 220 KB agregaba un segundo de latencia. Usar `read1()` |
| Resize dentro del loop de captura | Estranguló la captura de 14 a 2,3 fps. El trabajo caro va en otro hilo, procesando solo el frame más nuevo |
| `StartLimit*` en `[Service]` | Van en `[Unit]`. systemd 245 los **ignora** con un warning y la protección queda desactivada |
| `Environment=` vs `EnvironmentFile=` | Se aplican **en orden** y gana el último. El archivo va **después** o se ignora |
| `.env` con `setdefault` + re-exec | Un cambio por API se escribía bien y **nunca aplicaba**: el valor viejo seguía en el entorno heredado. Hay que sobreescribir `os.environ` también |
| `SportClient` como objeto global | **Segfault antes de `main()`**: su constructor necesita `ChannelFactory::Init()` primero. Usar `unique_ptr` creado en `main` |
| Nombres de binario de 16 caracteres | El kernel trunca `comm` a 15: `ps -C telemetry_reader` y `pgrep` sin `-f` lo reportan muerto estando vivo |
| `timechart avg('campo.con.puntos')` | Devuelve vacío en silencio. Hacer `eval` a un nombre limpio primero |
| Índice de Splunk equivocado | `{"text":"Incorrect index","code":7}` **incluso para `main`**. "Success sin index + Incorrect con cualquiera" = nombre mal o no permitido en el token |
| `ros2 topic hz`/`bw` con `--no-daemon` | Devuelven vacío en silencio. Con `list` sí hay que usarla |
| Placeholders `<ASI>` en comandos de shell | bash interpreta `<` como redirección. Costó dos veces |
| Probar CURWB con el cable conectado | Todo funciona igual, la prueba no prueba nada. Desenchufar físicamente |

---

## 9. Repos y quién commitea

| Repo | Remoto |
|---|---|
| `~/Desktop/robot-splunk-bridge` | `github.com/Maxi-Andres/IA-splunk-bridge` |
| `~/Desktop/robot-nvr-bridge` | `github.com/Maxi-Andres/AI-ROBOT-NVR` |
| `~/Desktop/AI-VL-ecosystem/*` | tres repos separados |
| `~/Desktop/unitree_ros2` | el fork de Unitree con el executor y el bridge de cámara |

**El usuario commitea y pushea todo a mano.** Nunca correr `git commit` ni `git push`.
El flujo es: editar acá → él pushea → en el robot `git pull && ./build.sh` → reiniciar el
servicio.

Y el código y los comentarios **en inglés** (convención del ecosistema AI-VL); estos docs de
planificación en castellano.
