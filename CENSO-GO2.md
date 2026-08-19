# Censo de tópicos DDS — Go2

Medido el **2026-08-19** con el Go2 por cable en `192.168.123.0/24`, quieto (de pie, sin
caminar), desde el devcontainer Humble con `ros2 topic bw`. **122 tópicos** visibles.

> Ojo: medido con el robot **quieto**. Los tópicos de movimiento (`sportmodestate`) pueden
> variar caminando; los de estado no deberían.

## Tópicos de estado relevantes

| Tópico | Tasa | Tamaño msg | Ancho de banda | Crudo/día | ¿Se usa? |
|---|---:|---:|---:|---:|---|
| `/lowstate` | **500 Hz** | 1,18 KB | 593 KB/s | **51,2 GB** | ❌ demasiado |
| `/lf/lowstate` | **20 Hz** | 1,18 KB | 23,8 KB/s | 2,06 GB | ✅ **fuente principal** |
| `/sportmodestate` | ~298 Hz | 0,52 KB | 155 KB/s | 13,4 GB | ❌ demasiado |
| `/lf/sportmodestate` | **20 Hz** | 0,52 KB | 10,4 KB/s | 895 MB | ✅ **fuente de pose** |
| `/utlidar/imu` | ~253 Hz | 0,32 KB | 80,9 KB/s | 6,99 GB | ❌ el IMU ya viene en lowstate |
| `/utlidar/lidar_state` | ~5 Hz | 108 B | 552 B/s | 47,7 MB | ✅ salud del lidar |
| `/lf/battery_alarm` | ~1,1 Hz | 140 B | 157 B/s | 13,6 MB | ✅ eventos de batería |
| `/multiplestate` | ~1,1 Hz | 84 B | 95 B/s | 8,2 MB | 🟡 a evaluar |

No medidos a propósito (van en la denylist igual): `/frontvideostream`, `/utlidar/cloud`,
`/utlidar/cloud_deskewed`, `/utlidar/voxel_map`, `/uslam/*`.

## Hallazgos

**1. El Go2 es mucho más liviano que el G1.** La proyección del plan usaba la medición del G1
(1041 Hz, 2,2 MB/s). El Go2 real: **500 Hz y 1,18 KB por mensaje**. Sigue siendo imposible
reenviarlo (51 GB/día crudos ≈ **100x** el presupuesto, y en JSON expandido mucho más), pero el
mensaje chico es buena noticia para el evento curado.

**2. La familia `/lf/*` existe en el Go2 y es la fuente correcta.** `/lf/lowstate` trae **los
mismos datos a 20 Hz** — 25x menos tráfico que `/lowstate`, idéntico contenido (mismo tamaño de
mensaje: 1,18 KB). El agente debe suscribirse a **`/lf/lowstate`, no a `/lowstate`**: menos carga
DDS en el robot, menos CPU en el agente, mismo dato.

**3. Ni la versión liviana entra reenviada tal cual.** `/lf/lowstate` son 2,06 GB/día crudos; en
JSON expandido son ~6-10 GB/día = **12-20x el presupuesto**. Confirma la Decisión 1 del plan con
números del Go2: hay que extraer campos, no reenviar mensajes.

**4. Hay un tópico dedicado de alarma de batería** (`/lf/battery_alarm`, 1,1 Hz, 140 B) — fuente
directa para `robot:event` sin tener que inferir el umbral nosotros.

**5. `/utlidar/imu` no hace falta:** el IMU ya viene dentro de `LowState`, y ese tópico solo
agrega 7 GB/día de lo mismo.

## Detalles de operación

- **`ros2 topic bw`/`hz` NO aceptan `--no-daemon`** — con esa flag devuelven vacío en silencio.
  (Con `list` sí funciona, y ahí es necesaria para no leer un daemon rancio.)
- Al terminar, `ros2 topic bw` imprime `failed to initialize wait set: ... rcl_shutdown()` — es
  el `timeout` matando el proceso, no un error real.
