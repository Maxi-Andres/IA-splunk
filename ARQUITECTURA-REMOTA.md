# Operar el robot desde cualquier red

Cómo funciona el diseño, y por qué es así. Documento de arquitectura: el objetivo final, el
estado real hoy, y lo que falta para cada capacidad.

Fundamento técnico: `RED-Y-DDS.md`. Pasos concretos: `IMPLEMENTACION.md`.

---

## 0. El objetivo

Que el robot esté **donde sea** — el campo con Starlink, otra oficina, LTE — y que desde HQ
se pueda: **ver su telemetría, ver el video, y mandarle comandos**, sin depender de que
ninguna computadora esté en la misma red que el robot.

---

## 1. La regla, que está medida y no supuesta

> **DDS corto y local. HTTP largo y ruteado.**

DDS no cruza fronteras de subred con estos robots. Medido el 2026-08-19 sobre el Go2:

| Desde dónde | Ping al robot | Tópicos DDS visibles |
|---|---|---|
| Su propia subred (`192.168.123.0/24`) | 0,09 ms | **122** |
| Otra subred, ruteada | ✅ 1,32 ms | **2** |
| Otra subred + peers unicast por IP | ✅ | **3** |

La capa 3 funciona perfecto y el DDS no ve nada. El robot anuncia únicamente locators
`192.168.123.x` y su controlador de bajo nivel no tiene ruta fuera de esa subred, así que
**no hay peer, ruta ni NAT que lo arregle.**

De ahí sale la única conclusión posible:

> **La única máquina que va a estar L2-adyacente al DDS del robot, esté el robot donde esté,
> es el robot mismo. Entonces todo lo que toca DDS corre adentro del robot.**

---

## 2. Las dos computadoras del robot

Importante no confundirlas, porque hacen cosas distintas:

| | Go2 (perro) | G1 (humanoide) | Rol |
|---|---|---|---|
| **Bajo nivel** | `192.168.123.161` | `192.168.123.161` | Control de motores. **Publica los tópicos DDS de estado.** Sin SSH útil |
| **Alto nivel** | `192.168.123.18` (Jetson) | `192.168.123.164` (PC2) | Linux completo, con SSH. **Acá corren nuestros agentes** |

Los dos robots comparten el `.161`, así que **nunca pueden estar en el mismo segmento L2**
(conflicto de IP, no solo colisión de tópicos). Con los agentes adentro del robot eso deja de
importar: cada `.161` queda privado dentro de su propio robot.

El alto nivel del Go2, medido: aarch64, Ubuntu 20.04, 4 cores, 15 GB RAM, **429 GB libres**,
g++ 9.4 + cmake, internet propio con DNS, NTP sincronizado, y **load average 0,00**. Sobra.

---

## 3. Todo sale del robot hacia afuera, nunca al revés

Esta es la decisión que hace que "desde cualquier red" funcione de verdad:

```
        ROBOT (cualquier red, detrás de NAT/CGNAT)          HQ
   ┌──────────────────────────────────────────┐      ┌──────────────────┐
   │  .161 bajo nivel ──DDS──┐                │      │                  │
   │  (nunca sale del robot) │                │      │   Splunk         │
   │                         ▼                │      │   mediamtx       │
   │                   agentes (Jetson .18)   │      │   cola de cmds   │
   └─────────────────────────┬────────────────┘      └────────▲─────────┘
                             │                                │
                             │  telemetría  ──HTTPS push───────┤
                             │  video       ──RTSP/SRT push───┤
                             │  comandos    ──HTTPS pull ──────┘
                             │
                        (VPN IR1101 → Meraki MX, ya operativa)
```

**El robot no acepta ni una conexión entrante.** Manda telemetría, empuja video, y **va a
buscar** los comandos. Consecuencias:

- Atraviesa NAT y CGNAT sin configurar nada, sin puertos abiertos, sin IP fija.
- No hay endpoint en el robot que alguien pueda atacar desde la red.
- Los que miran no se conectan al robot: se conectan a **HQ**, donde el robot ya dejó todo.

Eso último es la clave de "ver el video desde cualquier IP": no alcanzás al robot, alcanzás
el mediamtx de HQ. Es un servicio interno normal.

---

## 4. Estado por capacidad

### 4.1. Telemetría a Splunk — ✅ **funcionando**

Agente nativo en C++ en el Jetson (`robot-splunk-bridge`), servicio systemd. Lee
`rt/lf/lowstate` y `rt/lf/sportmodestate`, extrae campos elegidos a mano, y postea al HEC de
Splunk. **40 MB/día**, con cap propio de bytes y spool en disco para los cortes de enlace.

Consumo medido: **1,2% CPU / 9 MB** el lector, **0,4% / 16 MB** el shipper, contra un techo
de 25% / 256 MB. Invisible para el robot.

Verificado con la PC del escritorio en otra VLAN sin ver un solo tópico DDS: los datos
llegaron igual.

### 4.2. Video — **lo próximo**

Hoy la captura corre en la PC del escritorio y lee DDS, así que **exige que el robot esté en
la misma red**. Hay que mover la mitad de arriba al robot:

| Pieza | Hoy | Va a quedar |
|---|---|---|
| `go2_jpeg_stream` (captura por DDS) | PC del escritorio | **Jetson del robot** |
| `ffmpeg` (encode a H.264) | PC del escritorio | **Jetson del robot** |
| `mediamtx` (servidor RTSP/HLS/WebRTC) | PC del escritorio | HQ — recibe el push |
| `Frigate` (NVR: graba, timeline) | PC del escritorio | HQ — no cambia nada |

`run.sh` de `robot-nvr-bridge` ya tiene el destino parametrizado (`RTSP=rtsp://host:8554/robot`);
falta hacer opcional el mediamtx local, que hoy arranca siempre.

Dos oportunidades del lado del robot: el Jetson tiene **encoder por hardware**, y el Go2
genera **H.264 nativo** en `rt/frontvideostream`. El fracaso conocido de ese tópico fue
leyéndolo **desde afuera** del robot; leerlo localmente es un test distinto y, si anda, se
ahorra la recodificación completa.

⚠️ **Escala:** la telemetría son 40 MB/día; el video 1080p son **2-4 Mbps ≈ 20-40 GB/día**.
Por un enlace satelital entra, pero es otro orden de magnitud y probablemente convenga
**on-demand** en vez de continuo. Decisión a tomar antes de construirlo.

### 4.3. Comandos — **el último, y necesita diseño de seguridad**

Es el único que **escribe** en DDS, así que también tiene que correr en el robot. Pero rompe
la propiedad que hace seguro al agente de telemetría: el de hoy es **read-only** y no puede
mover el robot ni con un bug. Un relay de comandos sí puede.

**El patrón correcto es pull, no un endpoint.** El agente del robot consulta una cola en HQ
(long-poll HTTPS saliente) y ejecuta lo que encuentra. Comparado con exponer un endpoint HTTP
en el robot:

- No hay puerto entrante en el robot. Nada que escanear ni atacar desde la red.
- Funciona detrás de CGNAT sin trucos.
- La autenticación vive en HQ, donde ya hay infraestructura, no en el robot.

Lo que igual hace falta: **lista blanca de comandos** (no un pasamanos genérico a DDS),
**dead-man switch** (si se corta el enlace, el robot para), y un **log de auditoría** de qué
se mandó y quién. Nada de eso es difícil, pero merece su propio diseño y no ser una extensión
del agente de telemetría.

---

## 5. Cómo llega el tráfico

Ya está resuelto y operativo (detalle en `PLAN-CONECTIVIDAD-ROBOTS.md`):

| Pieza | Estado |
|---|---|
| VPN IKEv2 del IR1101 al Meraki MX de HQ | **Operativa** |
| NAT del Jetson: `192.168.123.18` → `10.1.254.18` | **Operativa** |
| Uplink de campo: Starlink Mini (bypass de CGNAT, keepalive por IP SLA) | Configurado |

Aclaración sobre "desde cualquier IP": el **robot** puede estar en cualquier red, porque todo
lo suyo es saliente. Los que **miran u operan** llegan a los servicios de HQ (Splunk,
mediamtx, la cola de comandos) — o sea que tienen que estar en la red corporativa o en su VPN.
No es "cualquiera desde internet", y está bien que no lo sea.

---

## 6. Lo que este diseño NO resuelve

**ROS2 nativo remoto.** Si alguna vez hace falta que una herramienta que habla DDS (RViz, un
nodo de navegación) vea al robot desde HQ, extraer campos no alcanza: hacen falta los tópicos
de verdad. Para eso están `zenoh-bridge-dds` y el DDS Router de eProsima, que puentean DDS por
WAN sin extender L2. Igual corren **adentro del robot** — la regla de §1 no cambia, solo cambia
qué proceso se pone ahí.

**Los dos robots en la misma LAN local.** Sigue siendo imposible por el conflicto del `.161`.
Este diseño lo **esquiva** (cada uno reporta por su cuenta), no lo arregla. Si alguna vez hay
que tener los dos en la LAN leyéndolos desde afuera, hace falta una VLAN por robot.

---

## 7. Pendientes

- [ ] Mover la captura de video al Jetson (§4.2) y decidir continuo vs on-demand.
- [ ] Diseñar el relay de comandos con su modelo de seguridad (§4.3).
- [ ] Migrar el bridge de cámara de AI-VL a consumir el stream de HQ en vez de DDS.
- [ ] **Cambiar la password del Jetson** (hoy `123`) — ya hay un token de Splunk ahí adentro.
- [ ] Repetir el despliegue en el G1 (PC2, `.164`), que usa `unitree_hg` en vez de `unitree_go`.
