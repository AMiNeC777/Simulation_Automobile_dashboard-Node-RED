# Lab — Connect the molding simulation to Node-RED with Mosquitto

You will put Mosquitto between the simulation page and Node-RED. Both become clients of the broker. The dashboard, the counters, and the CSV journal stay as they are.

Do the steps in order. Do not point the page at `ws://127.0.0.1:1880/ws/moulage` and expect Mosquitto to answer: that address is Node-RED’s own WebSocket, and the page does not speak MQTT yet.

## Architecture

```
Simulation (browser)  -- ws://127.0.0.1:9001 -->  Mosquitto  <-- 1883 --  Node-RED  -->  dashboard /ui
```

A browser cannot open MQTT on port 1883. Mosquitto must expose a second listener, MQTT over WebSocket, on port 9001. Node-RED keeps using port 1883.

## Topics

| Direction | Topic | Payload |
| --- | --- | --- |
| Page → broker | `moulage/piece` | Piece object: `id`, `timestamp`, `status`, `temp`, `packed`, `totalCount`, `conformeCount`, `defectTotal`, `defectRate` |
| Page → broker | `moulage/status` | `{ status, temp, speed, defectStreak, bottleneck, challengeActive }` |
| Node-RED → broker | `moulage/cmd` | `{ "action": "START" }`, `STOP`, `RESET`, or `{ "action": "SET_SPEED", "value": 60 }`, `{ "action": "SET_TEMP", "value": 185 }` |

`status` on `moulage/status` is still one of `En Marche`, `Surchauffe`, `Arrêt Sécurité`. The dashboard already understands those three strings.

## What you need

- Node.js and Node-RED, with `node-red-dashboard` installed
- The flow `moulage-nodered-flow-v2.json` already imported and deployed
- [Mosquitto for Windows](https://mosquitto.org/download/)
- A browser and a network connection (the page loads Three.js and MQTT.js from the internet)

MQTT nodes are already part of Node-RED. Do not install an extra MQTT palette.

## 1. Configure Mosquitto

Create `mosquitto.conf` in this project folder:

```conf
listener 1883
protocol mqtt

listener 9001
protocol websockets

allow_anonymous true
```

`allow_anonymous true` is acceptable only on your own machine. If port 1883 is already taken by the Windows service **Mosquitto Broker**, stop that service first (Services, or `Stop-Service mosquitto` in an administrator PowerShell), otherwise the command below fails with “address already in use”.

Start the broker in a terminal and leave it open:

```powershell
cd "C:\Users\pc\Desktop\M2 BDIO\IOT\Simulation-de-moulage-automobile-V2-dashboard-Node-RED"
mosquitto -v -c mosquitto.conf
```

If `mosquitto` is not found, call it by its install path:

```powershell
& "C:\Program Files\Mosquitto\mosquitto.exe" -v -c mosquitto.conf
```

`-v` prints every connection. You should see both listeners start, with no error after them.

## 2. Prove the broker before touching the flow

Open two more terminals.

Subscribe to everything under `moulage/`:

```powershell
mosquitto_sub -h 127.0.0.1 -t "moulage/#" -v
```

In the other terminal, publish one command:

```powershell
mosquitto_pub -h 127.0.0.1 -t "moulage/cmd" -m "{\"action\":\"START\"}"
```

The subscriber must print:

```text
moulage/cmd {"action":"START"}
```

The Mosquitto window must show the publish. If either side stays silent, fix the broker before continuing. Node-RED cannot repair a broker that does not deliver messages.

## 3. Point Node-RED at the broker

Start Node-RED (`node-red`) and open [http://127.0.0.1:1880](http://127.0.0.1:1880).

1. Drag an **mqtt in** node onto the flow.
2. Double-click it. Next to **Server**, choose **Add new mqtt-broker**, then the pencil.
3. **Server**: `127.0.0.1`. **Port**: `1883`. Leave the username empty. **Update**, then **Add**.
4. On the mqtt in node:
   - **Topic**: `moulage/#`
   - **QoS**: 0
   - **Output**: a parsed JSON object
   - **Name**: `MQTT moulage in`
5. Drag a **function** node, name it `Topic vers type`, and paste this:

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

6. Wire **MQTT moulage in** → **Topic vers type** → the existing switch **Type d'événement**.
7. Disconnect the wire that goes from **WS moulage in** into **Normaliser le message**. Leave those two nodes on the canvas so you can see what they used to do. They must not stay connected, or the same event can be handled twice.

The switch still expects `msg.type` to be `moulage:piece` or `moulage:status`. The function above is what produces that field from the topic. **Traiter pièce**, **Traiter statut**, and the dashboard nodes stay untouched.

## 4. Publish dashboard commands on `moulage/cmd`

1. Drag an **mqtt out** node.
2. Select the same broker (`127.0.0.1:1883`).
3. **Topic**: `moulage/cmd`. **QoS**: 0. **Name**: `MQTT moulage out`.
4. Disconnect **Commande JSON** from **WS moulage out**.
5. Wire **Commande JSON** → **MQTT moulage out**.

**Commande JSON** already builds `{ action, value }`. Do not rewrite it.

**Deploy**.

With `mosquitto_sub` still running, open the dashboard [http://127.0.0.1:1880/ui](http://127.0.0.1:1880/ui), tab **Moulage V2**, and click **Démarrer**. The subscriber must show:

```text
moulage/cmd {"action":"START","payload":{"action":"START"}}
```

`SET_SPEED` and `SET_TEMP` also carry `"value"`. The page reads `action` and `value` from that object. The extra `payload` field is harmless.

The simulation does not move yet. Nothing is subscribed to `moulage/cmd` except your terminal.

## 5. Serve the page over HTTP

Do not open the HTML file with a double-click. A `file://` page is a poor place to open a WebSocket.

In a new terminal, from this project folder:

```powershell
npx --yes serve -l 5500
```

Open [http://127.0.0.1:5500/SimulationMoulageAuto_V2.html](http://127.0.0.1:5500/SimulationMoulageAuto_V2.html).

The 3D scene must appear. **Connecter** still talks to Node-RED’s WebSocket. The next step changes that.

## 6. Teach the page to speak MQTT

Edit `SimulationMoulageAuto_V2.html`. Reload the page after each save (Ctrl+F5).

### 6.1 Load the MQTT library

Just before `<script type="module">`, add a normal script tag (not `type="module"`):

```html
<script src="https://unpkg.com/mqtt@5.10.3/dist/mqtt.min.js"></script>
```

Inside the module, the client is `window.mqtt`. A module does not see the name `mqtt` by itself.

### 6.2 Change the default address

Find:

```javascript
const DEFAULT_WS = 'ws://127.0.0.1:1880/ws/moulage';
```

Replace it with:

```javascript
const DEFAULT_WS = 'ws://127.0.0.1:9001';
```

In the HTML, change the input `value` of `#ws-url` to `ws://127.0.0.1:9001`. You can rename the heading **WebSocket Node-RED** to **MQTT Mosquitto** so the panel matches what it does.

### 6.3 Publish pieces and status

Replace `Ws.forward` so a connected client publishes one topic per event. `readyState` belongs to WebSocket; an MQTT client uses `.connected`.

```javascript
forward(type, payload) {
  if (!this.client || !this.client.connected) return;
  const topic = type === 'moulage:piece' ? 'moulage/piece' : 'moulage/status';
  try { this.client.publish(topic, JSON.stringify(payload)); } catch (err) { /* fermeture */ }
},
```

Leave `Ws.handle` as it is. It already accepts `START`, `STOP`, `RESET`, `SET_SPEED`, and `SET_TEMP`.

### 6.4 Connect and subscribe

Replace `window.connectWebSocket` with:

```javascript
window.connectWebSocket = function (url) {
  const target = (url && String(url).trim()) || (el.wsUrl.value || '').trim() || DEFAULT_WS;
  window.disconnectWebSocket();
  el.wsUrl.value = target;
  setWsMode('connecting');
  if (!window.mqtt) {
    setWsMode('error');
    return;
  }
  const client = window.mqtt.connect(target, { reconnectPeriod: 2000 });
  Ws.client = client;
  client.on('connect', function () {
    if (Ws.client !== client) return;
    client.subscribe('moulage/cmd');
    setWsMode('open');
    publishStatus(true);
  });
  client.on('message', function (topic, buf) {
    if (topic !== 'moulage/cmd') return;
    Ws.handle(buf.toString());
  });
  client.on('error', function () {
    if (Ws.client === client) setWsMode('error');
  });
  client.on('close', function () {
    if (Ws.client === client) {
      Ws.client = null;
      setWsMode('closed');
    }
  });
};
```

Replace `window.disconnectWebSocket` with:

```javascript
window.disconnectWebSocket = function () {
  const client = Ws.client;
  Ws.client = null;
  if (!client) {
    setWsMode('closed');
    return;
  }
  try { client.end(true); } catch (err) { /* déjà fermé */ }
  setWsMode('closed');
};
```

### 6.5 Fix the Connecter button

Find the click listener on `#btn-ws`. It checks `Ws.socket.readyState`. Replace that test:

```javascript
el.btnWs.addEventListener('click', () => {
  const open = Ws.client && (Ws.client.connected || Ws.client.reconnecting);
  if (open) window.disconnectWebSocket();
  else window.connectWebSocket(el.wsUrl.value.trim());
});
```

Reload [http://127.0.0.1:5500/SimulationMoulageAuto_V2.html](http://127.0.0.1:5500/SimulationMoulageAuto_V2.html).

## 7. Check the whole path

Keep `mosquitto_sub -h 127.0.0.1 -t "moulage/#" -v` open.

1. On the page, leave the address `ws://127.0.0.1:9001` and click **Connecter**. The text must become **Connecté**. Mosquitto’s verbose log must show a WebSocket client.
2. The subscriber must print a `moulage/status` message as soon as the page connects.
3. Click **Démarrer** on the page. The line runs, and `moulage/piece` messages appear after the first inspection. The dashboard counters on [http://127.0.0.1:1880/ui](http://127.0.0.1:1880/ui) must move.
4. Click **Arrêter** on the dashboard. The page must stop. The subscriber shows `moulage/cmd` with `"action":"STOP"`.
5. Move the dashboard speed slider. The page slider **Vitesse du convoyeur** must follow (`SET_SPEED`, 10 to 100 from the dashboard).
6. Move the dashboard temperature slider. The page temperature must follow (`SET_TEMP`, 150 to 250). Above 220 °C the line goes to **Surchauffe** and `moulage/status` reports that word.
7. Click **Réinitialiser Sécurité** on the dashboard after you have lowered the temperature under 220 °C. That is `RESET`.

## If something stays silent

| What you see | What it usually means |
| --- | --- |
| `mosquitto_sub` prints nothing when you `mosquitto_pub` | The broker is not the process you think. Check the verbose window, and that nothing else owns port 1883. |
| Page stays on **Connexion…** or **Erreur de connexion** | Port 9001 is down, the page was opened with `file://`, or the MQTT script did not load. In the browser console, `window.mqtt` must be a function container, not `undefined`. |
| Subscriber shows `moulage/cmd` but the line does not start | The page is not connected, or it never subscribed to `moulage/cmd`. Click **Connecter** again and look for a WebSocket client in the Mosquitto log. |
| Pieces appear in `mosquitto_sub` but the dashboard stays at zero | **MQTT moulage in** is not deployed, the function does not set `msg.type`, or **WS moulage in** is still wired and the switch is not receiving the MQTT function. |
| Dashboard command works twice | **WS moulage out** is still wired as well as **MQTT moulage out**. Disconnect the WebSocket output. |
| Status badge on the dashboard never says **En Marche** | The page published the object, but `msg.type` is not exactly `moulage:status`. The switch compares that string. |
