# Red y DDS: por qué leer datos de estos robots es complicado

Documento de referencia. Explica **por qué no se pueden "pasar los tópicos DDS" de un lado a
otro de la red**, por qué los dos robots no pueden convivir en el mismo segmento, y qué está
verificado y qué no.

Aplica a cualquier cosa que lea datos del robot: el bridge de telemetría a Splunk
(`PLAN.md`), el pipeline de video (`robot-nvr-bridge`) y AI-VL.

Última actualización: 2026-08-19 (se agregó §8.2-§8.4: el caso de robots en
cualquier red).

---

## 0. Resumen en cinco hechos

1. **DDS no tiene servidor central.** No existe "la IP del tópico" para apuntarle. Los
   procesos se hablan directo entre sí, así que lo único que importa es si se pueden
   **encontrar** y **alcanzar** mutuamente.
2. **Los dos robots tienen la misma IP de bajo nivel: `192.168.123.161`.** No es solo que los
   tópicos se mezclan — es un **conflicto de IP**. Los dos en el mismo segmento L2 es imposible.
3. **El robot solo anuncia direcciones `192.168.123.x`** (3022 locators medidos, cero en otra
   subred). Le hables desde donde le hables, te va a contestar "buscame en la 123".
4. **La consecuencia de diseño:** nunca se pasan tópicos DDS por la red. El proceso que lee DDS
   se pone **al lado del robot**, extrae los datos, y lo que cruza la red es **HTTP/RTSP**, que
   rutea sin drama. **DDS corto y local, HTTP largo y ruteado.**
5. **Y si el robot puede estar en cualquier red** (campo, Starlink, LTE), "al lado del robot"
   significa **adentro del robot**: es la única máquina que va a estar L2-adyacente a su propio
   DDS, esté donde esté. Ver §8.2 — esto también hace desaparecer el problema del punto 2.

---

## 1. DDS no es como MQTT ni como un syslog

Esto es la raíz de toda la confusión, así que va primero.

En MQTT, HTTP o syslog apuntás a **la IP de un servidor**. El servidor está en el medio, y
rutear hasta él es todo lo que hace falta. Si el servidor es alcanzable, funciona.

DDS no tiene nada de eso. Es **peer-to-peer**: cada proceso habla **directo** con cada otro
proceso, sin intermediario. `rt/lowstate` **no es un lugar** — es un nombre que dos procesos
usan para reconocer que están hablando de lo mismo. No hay una IP del tópico, no hay un broker
al que apuntar, no hay un caño que puedas redirigir.

Por eso la pregunta "¿cómo paso los tópicos a la otra red?" no tiene respuesta: no hay tópicos
para pasar. Hay procesos que tienen que **encontrarse entre sí primero**.

---

## 2. Cómo se encuentran los procesos (discovery)

Cuando un proceso DDS arranca, grita a la red: *"existo, me llamo tal, y me podés encontrar en
estas direcciones"*. Ese grito se llama **SPDP** y por defecto va por **multicast**
(`239.255.0.1:7400` en CycloneDDS).

**El multicast es link-local:** sale con TTL=1 y los routers **no lo reenvían** salvo que los
configures a propósito (PIM/IGMP routing). O sea: el grito se muere en el primer router. Es la
razón por la que dos máquinas en VLANs distintas no se ven en DDS aunque se pinguen perfecto.

**Eso tiene arreglo.** Se le puede decir a DDS: *"no cuentes con el multicast, mandá tu
presentación directo a esta lista de IPs"* — son los **peers unicast**, y el unicast rutea
perfecto. Ya está implementado en `unitree_ros2/dds.env`:

```
CYCLONEDDS_IFACE=enp4s0          # a qué interfaz se bindea nuestro DDS
ROBOT_DDS_PEERS=192.168.51.115   # a quién le habla directo, sin multicast
```

`unitree_ros2/setup.sh` arma el `CYCLONEDDS_URI` con eso, y **deja `239.255.0.1` primero en la
lista de peers** para no desactivar el multicast local sin querer.

> **Entonces "DDS solo funciona en L2" no es cierto como ley general.** Se puede rutear. El
> problema es otro, y está en la sección que sigue.

---

## 3. El problema real: los locators

Encontrarse es una **presentación mutua**. Aunque mi grito llegue al robot por unicast, el
robot me contesta con *su* presentación, y ahí viene la parte crítica: **las direcciones donde
se lo puede contactar**. Se llaman **locators**, y salen de las interfaces a las que su proceso
DDS se bindeó.

Medido el 2026-08-03 con tracing de discovery de CycloneDDS: **3022 locators anunciados, todos
`192.168.123.x`, cero en cualquier otra subred.** Los dos onboard del G1 bindean DDS a `eth0`
únicamente.

```
Lector en otra VLAN                                     Robot
  192.168.20.50                                    192.168.123.161
       │                                                  │
       │──── "existo, buscame en 192.168.20.50" ─────────▶│  unicast, rutea OK
       │                                                  │
       │◀─── "existo, buscame en 192.168.123.161" ────────│  ¿puede volver la respuesta?
       │                                                  │
       │◀════════ datos, solo si AMBAS direcciones ══════▶│
                      son alcanzables
```

### Los tres requisitos para leer DDS desde otra red

| # | Requisito | Estado |
|---|---|---|
| 1 | Discovery que no dependa de multicast | ✅ Resuelto con peers unicast |
| 2 | Que el lector pueda mandar paquetes a `192.168.123.x` | 🟡 Plausible, hay ruta |
| 3 | **Que el robot pueda contestarte a tu subred** | ❌ Acá está el problema |

**El punto 3 es el que falla, y por una razón prosaica:** los robots Unitree vienen con IP
estática de fábrica en `eth0`, y es muy probable que **no tengan default gateway configurado**.
Si el robot no tiene ruta de salida de la `123`, no puede contestarle a nadie que esté afuera.
No es un firewall ni una config de DDS — es que su tabla de rutas no tiene por dónde mandar la
respuesta.

**Por eso tener presencia L2 es la apuesta segura:** si estás en el mismo segmento, el robot no
necesita rutear nada. Te contesta directo por ARP, sin gateway, sin depender de nadie.

> ⚠️ **A verificar (2 minutos con el robot prendido):** la tabla de rutas del robot
> (`ip route` por SSH al alto nivel) y una prueba de discovery unicast desde otra VLAN. Hasta
> no hacerlo, "no se puede leer DDS desde otra VLAN" es una **presunción razonable, no un hecho
> probado** — ver §6.

---

## 4. El conflicto de IP: los dos robots son `.161`

| Robot | Bajo nivel (donde vive el DDS) | Alto nivel (SSH) |
|---|---|---|
| **Go2** (perro) | `192.168.123.161` | `192.168.123.18` |
| **G1** (humanoide) | `192.168.123.161` | `192.168.123.164` |

Esto **resuelve una contradicción** entre documentos viejos: `robot-nvr-bridge/docs/DOS-ROBOTS.md`
dice "Go2 = `.161`, G1 = `.164`", y una medición posterior encontró al G1 Pro en `.161`. Las dos
cosas eran ciertas **sobre interfaces distintas**: `.161` es el bajo nivel de **los dos** robots,
y solo difiere el alto nivel. (Ese doc habría que corregirlo.)

### Qué rompe esto

- **Los dos robots en el mismo segmento L2 es imposible**, no "desprolijo". Hay conflicto de IP
  antes de que DDS entre en el tema.
- **Cualquier medición hecha con los dos robots conectados a la vez es sospechosa
  retroactivamente.** Incluidas las de `DOS-ROBOTS.md` (los "Publisher count: 2").
- **Mata la "Opción A" de los docs viejos** (un dominio DDS por robot): cambiar el dominio DDS
  **no arregla una IP duplicada**. Era la opción recomendada y ya no sirve sola.
- **Deja una sola salida limpia: una VLAN por robot.** La IP `.161` se puede repetir sin
  problema si está en segmentos L2 distintos. Y el lector necesita **una interfaz taggeada por
  VLAN**, con CycloneDDS bindeado por interfaz — que es la "Opción B" de
  `AI-VL-ecosystem/docs/SEPARAR_ROBOTS_MULTIPLES.md`.
- **Un lector ruteado no puede resolver esto nunca**: no hay tabla de rutas que alcance dos
  hosts distintos con la misma IP. Sin NAT (que es peor), la presencia L2 por VLAN es
  obligatoria para el escenario de dos robots. **Esta conclusión no depende de ningún test.**

> **Salvo que el lector esté adentro del robot** (§8.2): ahí cada `.161` queda privado y el
> conflicto se neutraliza. Todo lo de arriba aplica a los consumidores que leen DDS **desde
> afuera** del robot — hoy el pipeline de video y AI-VL.

---

## 5. Estado real de la red hoy

```
   Robots                     Esta PC (ia-pc)              Server ESXi
                              Ubuntu 26.04                 vSwitch0, MTU 1500
 ┌──────────┐                ┌───────────────┐            ┌────────────────────────┐
 │ Go2 .161 │────┐           │ enp4s0        │            │ VLAN20-VMs (VLAN 20)   │
 └──────────┘    ├─ 123/24 ──│ .123.99       │            │  └ Splunk .20.200      │
 ┌──────────┐    │           │ wlp3s0 (down) │            │ TRUNK ITINERANTE (4095)│
 │ G1  .161 │────┘           └───────┬───────┘            │  ├ C9800 CURWB 2       │
 └──────────┘                        │                    │  └ Splunk-collector    │
                              gw .123.1                   │ VM Network (VLAN 0)    │
                                     └── 0.55 ms ────────▶│ Management (VLAN 20)   │
                                        a .20.200         │  └ vmk0 .20.3          │
                                                          │ uplinks: vmnic3 (1G,   │
                                                          │   activo) + vmnic0     │
                                                          └────────────────────────┘
```

| Dato | Valor | Verificado |
|---|---|---|
| Red de los robots | `192.168.123.0/24` — **creada a propósito** para esta PC + los robots, no es la LAN de la oficina | 2026-08-18 (usuario) |
| Esta PC | `192.168.123.99` en `enp4s0`, Ubuntu 26.04 | 2026-08-18 |
| Gateway | `192.168.123.1`, rutea a otras VLANs | 2026-08-18 |
| Camino a Splunk | vía gw, **0.55 ms**, tcp/8000 y 8089 abiertos, **8088 cerrado** (HEC apagado) | 2026-08-18 |
| Portgroup de trunk | `TRUNK ITINERANTE`, **VLAN ID 4095 = VGT** (pasan todas las VLANs, el tagging lo hace el guest) | 2026-08-18 |
| VLAN ID del segmento de robots | **desconocido** — el número de subred (123) no es necesariamente el ID de VLAN | ❌ |
| ¿La VLAN de robots llega trunkeada a `vmnic3`? | **desconocido** | ❌ |
| Tópicos con el robot conectado | 121 por cable / 12 sin robot (o sea: 12 = "no ve nada") | 2026-08-03 |
| `rt/lowstate` | **1041 Hz, 2.2 MB/s** | 2026-08-03 |
| `rt/lf/lowstate` | 20 Hz, 42 KB/s — mismos datos, **52x más liviano** | 2026-08-03 |

Config del vSwitch a tener en cuenta: promiscuous mode, forged transmits y MAC changes están en
**No** (correcto, VGT no los necesita) y el **MTU es 1500** — con tagging del guest se suman
4 bytes, riesgo bajo para muestras chicas pero sospechoso #1 si alguna vez fallan las grandes.

---

## 6. Qué se probó, qué pasó, y qué fue falso positivo

Esta sección existe porque **varias pruebas dieron resultados engañosos** y se perdió tiempo.

### El fracaso del G1 por WiFi (2026-08-03) — y su causa real

Se intentó leer DDS del G1 con el robot en WiFi (VLAN 51) y el server por cable. **Falló**:
con el cable físicamente desconectado, `ros2 topic list` devolvía 12 tópicos (o sea, ninguno
del robot).

**Pero la causa no fue "L3 no funciona".** Le estábamos mandando el discovery a la **IP de WiFi
del robot** (`192.168.51.x`), donde **no había ningún DDS escuchando** — su DDS estaba bindeado
solo a `eth0`. Le hablábamos a la dirección equivocada. Es un escenario **distinto** de "hablarle
a `123.161`, que sí es donde el DDS escucha".

Conclusión honesta: ese test **no prueba** que un lector ruteado no pueda leer DDS. Prueba que no
se puede hablarle a una interfaz donde el DDS no escucha. Lo primero sigue **sin probar**.

### Falsos positivos conocidos (trampas)

| Prueba | Por qué engaña |
|---|---|
| `ping` a la IP de WiFi del robot **con el cable conectado** | `192.168.123.0/24` está directamente conectada por `eth0` (métrica 100), así que las respuestas salen por el cable y el camino inalámbrico **nunca se ejercita**. Forzalo con `ping -I wlan0` **desde el robot**. |
| Cualquier test de CURWB con el cable conectado | Con el cable todo funciona, sin importar si el CURWB anda. La lectura de 121 tópicos del 2026-08-06 es **inválida** por esto. **Solo vale con el cable físicamente desenchufado.** |
| `ping 8.8.8.8` desde el robot | No prueba nada sobre DDS: el kernel rutea eso bien sin importar a qué interfaz se bindeó el proceso DDS. |
| Escanear puertos UDP 7400-7500 | El rate limiting de ICMP-unreachable de Linux hace que los puertos cerrados parezcan abiertos. |
| `ros2 topic list` con el daemon viejo corriendo | Un daemon rancio **ignora silenciosamente** el `CYCLONEDDS_URI` nuevo. Hacer `ros2 daemon stop` o usar `--no-daemon`. |
| Un cliente WiFi que no llega ni a su propio gateway | No es firewall: es un **lease DHCP viejo** de una VLAN que ya no rutea. Pasó cuando el SSID "ROBOTS ONLY" se movió de VLAN 20 a VLAN 51. `nmcli connection down/up` para re-pedir lease. |

### Trampas de configuración

- **El SDK no recibe nada si `CYCLONEDDS_URI` no fija la interfaz del robot.**
  `ChannelFactory::Init(0, "enp4s0")` **no alcanza** por sí solo. Fue el detalle clave para que
  funcionara el pipeline de video.
- **`dds.env` es root** (lo escribe el executor por `POST /dds`) — editarlo con sudo.
- **Un peer rancio** (`192.168.20.5`) sobrevivió a su lease DHCP y apuntaba a la nada. Poner
  **reservas DHCP** para las MAC de los robots.
- **Enchufar un adaptador USB-C ethernet al Jetson** hace que NetworkManager renombre las
  interfaces, y el tráfico se va por el switch de la oficina en vez del bus interno del robot —
  invalida cualquier test.
- **Ojo con "VLAN 20" en docs viejos:** ahí decía que la WiFi de robots estaba en VLAN 20. Se
  movió a **VLAN 51** (~2026-08-03), y hoy **VLAN 20 es la de los servers** (ahí vive Splunk).

### El acceso que no tenemos

**PC1 del G1 no tiene SSH en ningún puerto** (22, 2222, 8022, 23: todos rechazados). Así que su
binding de DDS **no se puede cambiar**. Solo PC2 (el Jetson, `.164`) tiene SSH. Por eso se eligió
el camino CURWB: un bridge L2 transparente extiende el segmento `123` por aire, el multicast
sigue funcionando y **no hay que cambiar nada en el robot ni en el server**. Sigue **sin validar**
(hay que probarlo con el cable desenchufado).

---

## 7. QoS: la otra complicación de DDS

Aparte de encontrarse, dos procesos DDS tienen que tener QoS **compatibles** o **no se
emparejan**, aunque se vean perfecto en la red.

- Un lector **RELIABLE** no matchea a un escritor **BEST_EFFORT**. Al revés sí: un lector
  BEST_EFFORT **puede** leer de un escritor RELIABLE.
- El escritor de `/frontvideostream` ofrece **RELIABLE + KEEP_LAST(1)**. Por cable va bien; por
  WiFi, una muestra de 130 KB se fragmenta en ~90 datagramas UDP y RELIABLE convierte cualquier
  pérdida en tormentas de NACK/retransmisión.
- **Regla práctica:** para sensores, pedir **BEST_EFFORT** en el lector. Matchea con todo y no
  se espiraliza si el enlace pierde paquetes.

Relacionado: hay un caso conocido de que **los mensajes grandes no se reensamblan** en esta
combinación. El H.264 nativo del `rt/frontvideostream` **empareja con el escritor pero nunca
entrega un cuadro completo** por el SDK nativo, y llega corrupto por el puente ROS2. Por eso el
pipeline de video usa JPEG (`GetImageSample`, mensajes chicos y confiables) y no H.264 nativo.

---

## 8. Qué significa todo esto para el diseño

### 8.1. La regla

**No se pasan tópicos DDS por la red. Nunca.** El proceso que lee DDS se pone donde el DDS es
fácil — mismo segmento L2 que el robot — extrae ahí los datos que importan, y lo que cruza la red
es un protocolo ruteable:

| Consumidor | Lee DDS en | Cruza la red como |
|---|---|---|
| Telemetría → Splunk | mismo segmento que el robot | **HTTPS** (HEC), rutea a cualquier VLAN |
| Video → NVR / dashboard | mismo segmento que el robot | **RTSP / HLS / WebRTC**, rutea |
| AI-VL (comandos, cámara) | mismo segmento que el robot | HTTP/WS al frontend |

**DDS corto y local, HTTP largo y ruteado.** Por eso la única pregunta que importa al elegir
dónde corre un proceso es *"¿puede ver DDS?"*, y todo lo demás se resuelve solo.

### 8.2. Y si el robot puede estar en cualquier red

Requisito planteado el 2026-08-19: los robots tienen que poder operar **desde cualquier lado** —
el campo con Starlink, LTE, otra oficina. O sea que **no se puede asumir que compartan subred con
nada**, y todas las soluciones de presencia L2 remota (VLAN por robot, sub-interfaces taggeadas,
trunk hacia el ESXi, CURWB) dejan de alcanzar: sirven para un robot **local**, no para uno
itinerante.

La observación que resuelve el caso general:

> **La única máquina que va a estar L2-adyacente al DDS del robot, esté el robot donde esté, es
> el robot mismo.**

Entonces el lector va **adentro del robot**. Cada robot tiene una computadora de alto nivel con
acceso a su propio bus DDS (Go2: Jetson en `.123.18`; G1: PC2 en `.123.164`), y el SDK de Unitree
trae **libs aarch64 precompiladas** (`lib/aarch64/`, `thirdparty/lib/aarch64/`), así que
compilar ahí es un trámite. Lo que sale del robot es **HTTPS saliente**.

Lo que se cae solo con esa decisión:

- **El requisito de misma subred desaparece.** El robot reporta desde cualquier red con internet.
- **El conflicto de `.161` (§4) se neutraliza**: la IP queda privada adentro de cada robot y nunca
  se expone. N robots con la misma IP, sin pisarse.
- **NAT deja de importar**: solo hay conexiones salientes, ningún puerto entrante.
- **El ancho de banda pasa a ser trivial**: campos curados son KB/s, no los 2,2 MB/s de
  `rt/lowstate`.

### 8.3. El túnel es para el HTTPS, no para el DDS

La tentación es tunelizar la red del robot (ZeroTier con bridging L2, WireGuard + bridge) para que
un lector remoto crea que está en la misma LAN. **Es la peor opción de todas:**

- DDS sobre WAN se degrada muy mal: con RTT alto, jitter y pérdida, la QoS RELIABLE se convierte
  en tormentas de retransmisión.
- Manda los tópicos **completos** por el enlace.
- **Es imposible para dos robots**: dos túneles presentando `192.168.123.161` cada uno = conflicto
  de IP del lado del server. No hay configuración que lo arregle.

Sí hace falta un túnel, pero para otra cosa: para que el robot alcance el HEC de Splunk, que vive
en una IP privada. Ese túnel lleva **HTTPS con reintentos**, que es exactamente el tráfico que
tolera un enlace satelital.

### 8.4. Cuándo sí sirve un DDS Router

Si algún día hace falta **comandar** un robot remoto, o que AI-VL lo vea como si fuera local,
extraer campos ya no alcanza: hacen falta los tópicos de verdad. Para eso están
`zenoh-bridge-dds` y el DDS Router de eProsima, hechos para puentear DDS por WAN sin extender L2.
Igual corren **adentro del robot** — el punto de §8.2 no cambia, solo cambia qué proceso se pone
ahí.

## 9. Pendientes

**Del caso itinerante (§8.2) — es el que manda ahora:**

- [ ] Inventariar el Jetson de alto nivel del **Go2** (`192.168.123.18`): SSH, arquitectura,
      espacio, versión de OS, salida a internet propia, NTP.
- [ ] Confirmar **permiso para instalar un servicio en el robot**. Si no lo hay, se cae la
      arquitectura y hay que volver a la presencia L2 con robots locales.
- [ ] Definir cómo alcanza el robot el HEC de Splunk desde afuera (VPN vs endpoint publicado).
- [ ] Verificar que el reloj del robot esté sincronizado (si no, los timestamps mienten).
- [ ] Validar **con el cable externo desenchufado** — el único test que vale (§6).

**Del caso local (sigue abierto para video y AI-VL, que leen DDS desde afuera del robot):**

- [ ] **Tabla de rutas del robot** — ¿tiene default gateway? Define si un lector ruteado es
      posible para un robot solo. (2 min con el robot prendido)
- [ ] **VLAN ID** del segmento de robots (≠ 123 necesariamente) y si llega trunkeada a `vmnic3`.
- [ ] Cómo se separan los dos robots si están los dos en la LAN local (el conflicto de `.161`
      sigue vivo ahí).
- [ ] Validar el CURWB con el cable desenchufado.
- [ ] Corregir la tabla de IPs de `robot-nvr-bridge/docs/DOS-ROBOTS.md` y revisar sus conclusiones
      a la luz del conflicto de `.161`.

## 10. Documentos relacionados

| Doc | Qué tiene |
|---|---|
| `SplunkCode/PLAN.md` | El plan de telemetría a Splunk que se apoya en este documento |
| `robot-nvr-bridge/docs/ARQUITECTURA.md` | El pipeline de video que ya funciona, y el detalle del `CYCLONEDDS_URI` |
| `robot-nvr-bridge/docs/DOS-ROBOTS.md` | Investigación de separar los dos robots — **tabla de IPs desactualizada**, y su "Opción A" quedó invalidada por §4 |
| `AI-VL-ecosystem/docs/SEPARAR_ROBOTS_MULTIPLES.md` | La misma discusión del lado de los comandos; su "Opción B" (por interfaz) es el camino que sobrevive |
| `unitree_ros2/dds.env.example` | Los dos knobs del transporte DDS, documentados |
