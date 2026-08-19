# Guía de implementación — telemetría del Go2 a Splunk

Pensada para ejecutarla **vos**, en orden, entendiendo qué prueba cada paso.

**Regla que ordena todo: nada toca el robot hasta la Etapa D.** Las tres primeras etapas
corren en esta PC y en Splunk. Si algo falla, falla acá, no en el robot.

Estado a 2026-08-19: **Etapa B ya validada** (ver abajo). El bloqueo real es la Etapa A.

---

## Etapa A — Habilitar el HEC en Splunk  ⬅️ **el único bloqueo real**

Es lo único que no puedo hacer yo: necesita admin en Splunk. Todo lo demás está probado.

**A1. Habilitar el HEC**
En Splunk (`https://192.168.20.200:8000`): **Settings → Data Inputs → HTTP Event
Collector → Global Settings** → `All Tokens: Enabled`, `Enable SSL: yes`, puerto `8088`.

**A2. Crear el índice**
**Settings → Indexes → New Index** → nombre `go2-robot-data`, retención 2 días
(`frozenTimePeriodInSecs = 172800`).

**A3. Crear un token por robot**
**Settings → Data Inputs → HTTP Event Collector → New Token** → nombre `go2-01`,
índice por defecto `go2-robot-data`. Guardá el token.

> **Un token por robot, no uno compartido.** El token va a vivir dentro del robot, y la
> password SSH del Jetson es `123`. Si un robot se pierde o se compromete, querés poder
> revocar solo el suyo.

**A4. Verificar que el puerto abrió** — desde esta PC:
```bash
timeout 3 bash -c '</dev/tcp/192.168.20.200/8088' && echo ABIERTO || echo cerrado
```

**A5. Prueba de humo con `curl`** — antes de meter DDS en el medio:
```bash
curl -k https://192.168.20.200:8088/services/collector/event \
  -H "Authorization: Splunk <TOKEN>" \
  -d '{"event":{"hola":"mundo"},"sourcetype":"robot:test","index":"go2-robot-data"}'
```
Esperado: `{"text":"Success","code":0}`. Después, en Splunk: `index=go2-robot-data`.

**Criterio de salida:** el `curl` responde `Success` y el evento aparece en la búsqueda.

---

## Etapa B — El colector de prueba, desde esta PC  ✅ **ya validado**

Un script de Python que corre en el devcontainer que ya tenés andando. **No instala nada,
no toca el robot, y no necesita compilar.** Es descartable: sirve para probar el camino de
datos y para tener el dashboard andando antes del 25.

Archivo: `~/Desktop/robot-splunk-bridge/poc/telemetry_poc.py`

**B1. Copiarlo al contenedor** (va a `/tmp`, no ensucia ningún repo):
```bash
cd ~/Desktop/robot-splunk-bridge/poc
docker cp telemetry_poc.py unitree_ros2_devcontainer-devcontainer-humble-1:/tmp/
```

**B2. Correrlo en seco** — imprime los eventos y **no manda nada**:
```bash
docker exec -it unitree_ros2_devcontainer-devcontainer-humble-1 bash -lc \
  'source /workspace/setup.sh; cd /tmp; python3 telemetry_poc.py --dry-run'
```
✅ **Ya probado el 2026-08-19 contra el robot real.** Sale telemetría de verdad: batería
al 97%, 32,89 V, temperaturas de los 12 motores, IMU, posición.

**B3. Mandarlo a Splunk de verdad** (después de la Etapa A):
```bash
docker exec -it unitree_ros2_devcontainer-devcontainer-humble-1 bash -lc \
  'source /workspace/setup.sh; cd /tmp;
   HEC_URL=https://192.168.20.200:8088/services/collector/event \
   HEC_TOKEN=<TOKEN> python3 telemetry_poc.py'
```

**B4. Verificar en Splunk:**
```
index=go2-robot-data | stats count by sourcetype
```
Esperado: `robot:vitals`, `robot:motors`, `robot:pose`, `robot:health`.

**Volumen medido** (tamaños reales de JSON, no estimaciones): **40,0 MB/día = 8% de los
500 MB**. Detalle: vitals 378 B, motors 755 B, pose 246 B, health 237 B.

**Qué mirar cuando corra:**
- `error_code: 1001` en `robot:pose` — el robot lo reporta hoy, estando echado. Confirmar
  si es "no está en modo sport" o algo real.
- `imu.temp: 79` mientras los motores están a 26-35 °C.

**Criterio de salida:** los cuatro sourcetypes en Splunk, sostenido 1 hora.

---

## Etapa C — Probar que el DDS NO cruza de red

Es la prueba que justifica poner el agente adentro del robot. La idea: **desde otra subred,
el ping al robot funciona y el DDS no.** Eso es todo el argumento del diseño, demostrado.

### ⚠️ Antes de hacerlo, sabé el costo

Mover esta PC de subred **corta el video**: `go2_jpeg_stream` corre acá y lee DDS, así que
deja de capturar. mediamtx y Frigate siguen prendidos pero sin frames nuevos. Se recupera
solo al volver la PC a la 123 (el supervisor reintenta), pero vas a tener un hueco en la
grabación.

### Opción C1 — sin riesgo, sin tocar esta PC (recomendada)

Hacé la prueba **desde la VM `Splunk-collector`**, que está en el portgroup
`TRUNK ITINERANTE` y podés dejar en VLAN 20. Mismo resultado, cero downtime de video.

### Opción C2 — moviendo esta PC

**C0. Anotá la configuración actual antes de tocar nada:**
```bash
ip -brief addr ; ip route
# hoy: enp4s0 = 192.168.123.99/24, default via 192.168.123.1
```

**C1. Cambiá `enp4s0`** a otra subred (otra VLAN o una IP estática de otro rango),
asegurándote de conservar **ruta** hacia `192.168.123.0/24` por el gateway.

**C2. Confirmá que la capa 3 sigue viva** — esto es clave, tiene que funcionar:
```bash
ping -c2 192.168.123.161      # DEBE responder (rutea por el gateway)
```

**C3. Probá el DDS** — esto es lo que tiene que fallar:
```bash
docker exec -it unitree_ros2_devcontainer-devcontainer-humble-1 bash -lc \
  'source /workspace/setup.sh; ros2 daemon stop; ros2 topic list --no-daemon | wc -l'
```
- **122** = ve al robot (no esperado desde otra subred)
- **12** = no lo ve. Son solo los tópicos locales. **Esto confirma la tesis.**

**C4. Probá con peers unicast** — y acá está lo interesante:
```bash
sudo sed -i 's/^ROBOT_DDS_PEERS=.*/ROBOT_DDS_PEERS=192.168.123.161/' \
  ~/Desktop/unitree_ros2/dds.env
# repetir C3
```
Si con peers unicast **tampoco** aparecen los tópicos, queda probado que el problema no es
el multicast sino que **el controlador `.161` no puede contestar hacia afuera de su subred**
(sin default gateway). Eso cierra el último pendiente abierto de `RED-Y-DDS.md`.

**C5. Volvé todo a como estaba** (C0) y verificá que el video volvió:
```bash
~/Desktop/robot-nvr-bridge/status.sh
```

**Criterio de salida:** el conteo de tópicos desde la otra subred, documentado, con y sin
peers unicast.

---

## Etapa D — Recién acá, el robot

**No empezar hasta que la Etapa B esté andando sostenida.** El agente de producción es un
binario nativo en C++ (el SDK trae libs aarch64 precompiladas, y el Jetson ya tiene
g++ 9.4.0 + cmake), así que **no hace falta ROS2 en el robot**.

Orden, y el motivo de cada paso:

**D1. Compilar en el Jetson, sin instalar nada.** Copiar el fuente y el SDK, compilar ahí.
Si algo falta, te enterás antes de tocar servicios.

**D2. Correrlo a mano, en primer plano.** Con `--dry-run` primero, después contra el HEC.
Lo ves funcionar, lo cortás con Ctrl-C. Sin systemd, sin nada persistente.

**D3. Dejarlo corriendo una hora a mano** y mirar el consumo (`top`) y el dashboard.

**D4. Recién entonces, el servicio systemd** — con `Restart=always` y límites de recursos:
```ini
MemoryMax=256M
CPUQuota=25%
Nice=10
```
El agente es **read-only**: se suscribe y nada más, no publica en ningún tópico de comando.

**D5. Probar el corte de enlace** a propósito y confirmar que el spool en disco drena sin
perder ni duplicar datos (hay 429 GB libres en el Jetson).

**Criterio de salida:** el agente corriendo como servicio en el robot, sobreviviendo a un
corte de red y a un reinicio.

---

## Lo que YA está hecho

| | |
|---|---|
| Censo de tópicos del Go2 | `CENSO-GO2.md` — 122 tópicos, tasas y tamaños reales |
| Inventario del Jetson | `PLAN.md` §2.3 — viable, con toolchain y 429 GB libres |
| Camino Jetson → Splunk | 0,74 ms, verificado. Solo falta el 8088 |
| Camino Jetson → DDS (`.161`) | 0,25 ms, mismo L2 |
| VPN del robot a HQ | Ya operativa (IR1101 → Meraki MX) |
| El colector de prueba | Escrito y **validado en seco contra el robot real** |
| Volumen real | **40,0 MB/día medidos** = 8% del presupuesto |

## Lo que falta, en orden

1. **Habilitar el HEC** (Etapa A) — bloquea todo lo demás.
2. Correr el PoC contra Splunk (B3) y armar el dashboard.
3. La prueba de la otra subred (Etapa C) — cuando quieras, no bloquea.
4. El agente nativo en el robot (Etapa D) — después del 25 está bien.
5. Cambiar la password del Jetson antes de dejar un token ahí.
