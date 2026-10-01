# Lab — Molding simulation, Mosquitto, and Node-RED

The simulation and Node-RED are both MQTT clients. Mosquitto is the only broker between them. Do not add a WebSocket listener in Node-RED, and do not use the nodes **WS moulage in** or **WS moulage out**.

Do the steps in order.

## Architecture

```
Simulation (browser)  -- MQTT -->  Mosquitto  <-- MQTT --  Node-RED  -->  dashboard /ui
         ws://127.0.0.1:9001              tcp://127.0.0.1:1883
```

Node-RED connects to Mosquitto on port **1883** (MQTT).

The page is a browser, and a browser cannot open a raw MQTT socket on 1883. Mosquitto therefore has a second MQTT listener on port **9001** for browser clients. The address you type in the page is `ws://127.0.0.1:9001`. That address is Mosquitto, not Node-RED.

## Topics


| Who publishes | Topic            | Payload                                                                                                                         |
| ------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Page          | `moulage/piece`  | `id`, `timestamp`, `status`, `temp`, `packed`, `totalCount`, `conformeCount`, `defectTotal`, `defectRate`                       |
| Page          | `moulage/status` | `status`, `temp`, `speed`, `defectStreak`, `bottleneck`, `challengeActive`                                                      |
| Node-RED      | `moulage/cmd`    | `{ "action": "START" }`, `STOP`, `RESET`, or `{ "action": "SET_SPEED", "value": 60 }`, `{ "action": "SET_TEMP", "value": 185 }` |


`status` is one of `En Marche`, `Surchauffe`, `Arrêt Sécurité`.

## What you need

- Node.js and Node-RED, with `node-red-dashboard` installed
- `moulage-nodered-flow-v2.json` imported once, so the dashboard nodes already exist
- [Mosquitto for Windows](https://mosquitto.org/download/)
- A browser with internet access (the page loads Three.js and MQTT.js)

Node-RED already includes MQTT nodes. Do not install another MQTT palette.

## 1. Start Mosquitto

`mosquitto.conf` in this folder must contain only this:

```conf
listener 1883
protocol mqtt

listener 9001
protocol websockets

allow_anonymous true
```

Port 1883 is MQTT for Node-RED and for `mosquitto_pub` / `mosquitto_sub`. Port 9001 is MQTT for the page.

If the Windows service **Mosquitto Broker** is running, stop it first (`Stop-Service mosquitto` in an administrator PowerShell). Otherwise port 1883 is already taken.

Leave this terminal open:

```powershell
cd "C:\Users\pc\Desktop\M2 BDIO\IOT\Simulation-de-moulage-automobile-V2-dashboard-Node-RED"
mosquitto -v -c mosquitto.confmosquitto -v -c mosquitto.confmosquitto -v -c mosquitto.conf
```

If the command is not found:

```powershell
& "C:\Program Files\Mosquitto\mosquitto.exe" -v -c mosquitto.conf
```

Both listeners must start with no error after them.

## 2. Check the broker alone

Terminal A, leave it open:

```powershell
mosquitto_sub -h 127.0.0.1 -p 1883 -t "moulage/#" -v
```

Terminal B:

```powershell
mosquitto_pub -h 127.0.0.1 -p 1883 -t "moulage/cmd" -m "{\"action\":\"START\"}"
```

Terminal A must print:

```text
moulage/cmd {"action":"START"}
```

If it does not, stop here and fix Mosquitto. Node-RED cannot create that path for you.

## 3. Subscribe Node-RED to the broker

Open [http://127.0.0.1:1880](http://127.0.0.1:1880).

Delete these nodes if they are on the flow. They are not part of this lab:

- **WS moulage in**
- **Normaliser le message**
- **WS moulage out**

Keep **Type d'événement**, **Traiter pièce**, **Traiter statut**, **Commande JSON**, and the dashboard nodes.

1. Drag **mqtt in**. Double-click it. Next to **Server**, **Add new mqtt-broker**, then the pencil.
2. **Server**: `127.0.0.1`. **Port**: `1883`. No username. **Update**, then **Add**.
3. On **mqtt in**:
  - **Topic**: `moulage/#`
  - **QoS**: `0`
  - **Output**: a parsed JSON object
  - **Name**: `MQTT moulage in`
4. Drag a **function** node named `Topic vers type` and paste:

```javascript
const topic = msg.topic;
if (topic === "moulage/piece") msg.type = "moulage:piece";
else if (topic === "moulage/status") msg.type = "moulage:status";
else return null;

if (typeof msg.payload === "string") {
    try { msg.payload = JSON.parse(msg.payload); }
    catch (e) { return null; }
}
if (!msg.payload || typeof msg.payload !== "object") return null;
return msg;
```

1. Wire **MQTT moulage in** → **Topic vers type** → **Type d'événement**.

The switch still compares `msg.type` with `moulage:piece` and `moulage:status`. The function is what fills `msg.type` from the MQTT topic. Do not change **Traiter pièce** or **Traiter statut**.

## 4. Publish commands from the dashboard

1. Drag **mqtt out**. Use the same broker, `127.0.0.1` port `1883`.
2. **Topic**: `moulage/cmd`. **QoS**: `0`. **Name**: `MQTT moulage out`.
3. Wire **Commande JSON** → **MQTT moulage out**. Nothing else may be wired to the output of **Commande JSON**.

**Commande JSON** already builds `{ action, value }`. Leave it.

**Deploy**.

On the dashboard [http://127.0.0.1:1880/ui](http://127.0.0.1:1880/ui), tab **Moulage V2**, click **Démarrer**. Terminal A must show:

```text
moulage/cmd {"action":"START","payload":{"action":"START"}}
```

The line does not start yet. The page is not subscribed to `moulage/cmd`.

## 5. Open the page through a local server

From this folder:

```powershell
npx --yes serve -l 5500
```

Open [http://127.0.0.1:5500/SimulationMoulageAuto_V2.html](http://127.0.0.1:5500/SimulationMoulageAuto_V2.html). The 3D scene must load. Do not open the file by double-click.

## 6. Make the page an MQTT client

Edit `SimulationMoulageAuto_V2.html`. Save, then reload with Ctrl+F5.

### 6.1 Library

Immediately before `<script type="module">`, add:

```html
<script src="https://unpkg.com/mqtt@5.10.3/dist/mqtt.min.js"></script>
```

In the module, call `window.mqtt`. The bare name `mqtt` is not visible there.

### 6.2 Broker address

Replace the default address with Mosquitto’s browser listener:

```javascript
const DEFAULT_BROKER = 'ws://127.0.0.1:9001';
```

Set the text input that currently contains `ws://127.0.0.1:1880/ws/moulage` to `ws://127.0.0.1:9001`.

Change the section title **WebSocket Node-RED** to **MQTT Mosquitto**.

### 6.3 Publish

In the object that sends events, replace `forward` with:

```javascript
forward(type, payload) {
  if (!this.client || !this.client.connected) return;
  const topic = type === 'moulage:piece' ? 'moulage/piece' : 'moulage/status';
  try { this.client.publish(topic, JSON.stringify(payload)); } catch (err) { /* fermeture */ }
},
```

Keep `handle` as it is. It already understands `START`, `STOP`, `RESET`, `SET_SPEED`, and `SET_TEMP`.

### 6.4 Connect and subscribe

Replace `window.connectWebSocket` with:

```javascript
window.connectMqtt = function (url) {
  const target = (url && String(url).trim()) || (el.wsUrl.value || '').trim() || DEFAULT_BROKER;
  window.disconnectMqtt();
  el.wsUrl.value = target;
  setWsMode('connecting');
  if (!window.mqtt) {
    setWsMode('error');
    return;
  }
  const client = window.mqtt.connect(target, { reconnectPeriod: 2000 });
  Mqtt.client = client;
  client.on('connect', function () {
    if (Mqtt.client !== client) return;
    client.subscribe('moulage/cmd');
    setWsMode('open');
    publishStatus(true);
  });
  client.on('message', function (topic, buf) {
    if (topic !== 'moulage/cmd') return;
    Mqtt.handle(buf.toString());
  });
  client.on('error', function () {
    if (Mqtt.client === client) setWsMode('error');
  });
  client.on('close', function () {
    if (Mqtt.client === client) {
      Mqtt.client = null;
      setWsMode('closed');
    }
  });
};
```

Replace `window.disconnectWebSocket` with:

```javascript
window.disconnectMqtt = function () {
  const client = Mqtt.client;
  Mqtt.client = null;
  if (!client) {
    setWsMode('closed');
    return;
  }
  try { client.end(true); } catch (err) { /* déjà fermé */ }
  setWsMode('closed');
};
```

Rename the object `Ws` to `Mqtt` everywhere it is still used (`forward`, `handle`, and these two functions). `handle` stays inside that object.

### 6.5 Button

Replace the click listener of **Connecter** with:

```javascript
el.btnWs.addEventListener('click', () => {
  const open = Mqtt.client && (Mqtt.client.connected || Mqtt.client.reconnecting);
  if (open) window.disconnectMqtt();
  else window.connectMqtt(el.wsUrl.value.trim());
});
```

Reload [http://127.0.0.1:5500/SimulationMoulageAuto_V2.html](http://127.0.0.1:5500/SimulationMoulageAuto_V2.html).

## 7. Check MQTT end to end

`mosquitto_sub` from step 2 stays open.

1. On the page, address `ws://127.0.0.1:9001`, click **Connecter**. The label becomes **Connecté**. Terminal A prints `moulage/status`.
2. Click **Démarrer** on the page. After the first inspection, Terminal A prints `moulage/piece`. The dashboard counters move.
3. Click **Arrêter** on the dashboard. The page stops. Terminal A shows `moulage/cmd` with `"action":"STOP"`.
4. Move the dashboard speed slider. The page slider **Vitesse du convoyeur** follows (`SET_SPEED`).
5. Move the dashboard temperature slider. The page follows (`SET_TEMP`). Above 220 °C the status topic says `Surchauffe`.
6. Lower the temperature under 220 °C, then click **Réinitialiser Sécurité** on the dashboard (`RESET`).



## If a message does not arrive


| What you see                                               | What to check                                                                                                                                       |
| ---------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `mosquitto_pub` does not show up in `mosquitto_sub`        | Mosquitto is not the process listening on 1883. Read the broker terminal.                                                                           |
| Page stays on **Connexion…** or **Erreur de connexion**    | Nothing is listening on 9001, the MQTT script failed to load, or the file was opened with a double-click. In the console, `window.mqtt` must exist. |
| `moulage/cmd` is printed but the line does not start       | The page is not connected, or it did not subscribe to `moulage/cmd`.                                                                                |
| `moulage/piece` is printed but the dashboard stays at zero | **MQTT moulage in** is not deployed, or `msg.type` is not `moulage:piece` / `moulage:status`.                                                       |
| The same command runs twice                                | **Commande JSON** still has a second wire. Only **MQTT moulage out** may be connected there.                                                        |
| The dashboard never shows **En Marche**                    | `msg.type` is not exactly `moulage:status`.                                                                                                         |


