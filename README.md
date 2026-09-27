# Smart Delivery Tracking System — Full Setup Guide

> Core rule: **the Raspberry Pi thinks and decides. Node-RED only displays, notifies, and sends commands.**

```
 OwnTracks ──► HiveMQ Cloud ──► Raspberry Pi 4 (brain) ──MQTT──► Node-RED ──► Dashboard / Telegram / Email / Worldmap
                                  │ PN532 · State machine                          │
                                  │ Geofence · Persistence · LEDs/Buzzer           │ commands (Confirm / Issue)
                                  ◄────────────────────────────────────────────────┘
```

## 0) Files

```
smart_delivery/
├── config.py            ← settings (UIDs, destination, GPIO pins)
├── .env.example         ← template for secrets (MQTT host/user/password)
├── requirements.txt
├── pn532_test.py         ← tests the PN532 alone (just prints the UID)
├── nfc_reader.py         ← card reading
├── indicators.py         ← LEDs + Buzzer
├── state_manager.py      ← state machine + geofence + validation + JSON persistence
├── mqtt_client.py        ← connection to HiveMQ Cloud
├── main.py               ← runs everything
└── node-red/smart_delivery_flow.json   ← the ready-to-import Node-RED flow
```

---

## 1) Notes and assumptions you should know before starting

1. **The PN532 is on I2C** (as you specified). Address `0x24`.
2. **GPIO pins (chosen by me, change them in `config.py` if different):** Green LED = GPIO17 (pin 11), Red LED = GPIO27 (pin 13), Buzzer = GPIO22 (pin 15). Assumed the buzzer is **active-high** (`BUZZER_ACTIVE_HIGH = True`).
3. **Transition from PICKED_UP to IN_TRANSIT:** the spec didn't define exactly what triggers it. I chose to make it happen **on the first OwnTracks location update received after the courier card is scanned**.
4. **The customer card** triggers the same success signal (green LED + one short beep) — the spec didn't specify a signal for it. If you don't want that, tell me and I'll remove it (a small edit in `main.py` — nothing else changes, the signal is driven purely by `handle_card`'s result).
5. **Telegram is sent directly via the Bot API** (through an `http request` node) instead of the Telegram palette node. Reason: if your bot is already running elsewhere, Telegram doesn't allow more than one polling connection per token, and this approach avoids that conflict. The `chat_id` must have already sent `/start` to the bot.
6. **Email:** assumed Gmail SMTP (with an App Password), and a single address (`MANAGER_EMAIL`) receives both the delivered and issue emails. If you'd rather the delivered email go to the customer, let me know.
7. **Time:** the Pi prints timestamps in local time with the UTC offset (e.g. `+03:00`). Set the timezone to `Africa/Cairo`.
8. **OwnTracks messages are sent retained**, so after every reconnect the last one arrives again. The Pi ignores any location with the same or an older timestamp than the last one it stored (this is also part of the duplicate protection).
9. **Issue reports** can be recorded in any state (even after DELIVERED) and never change the shipment state.
10. **Extra events** sent by the Pi are for display only (no notifications attached): `in_transit`, `nfc_rejected`, `confirm_delivery_rejected`, `issue_rejected`.
11. **Node-RED keeps the event log and the last issue in memory only** (lost on restart). The permanent record lives in `shipment_state.json` on the Pi.
12. **Email:** the node only confirms the request was *handed to SMTP*, not that the email actually arrived.
13. **Secrets:** the HiveMQ credentials and email credentials are stored **encrypted** by Node-RED. The Telegram token is stored in the tab's Environment Variable and is saved **unencrypted** in `flows.json` — so don't Export/Share the flow once you've filled it in.
14. **To restart testing from scratch:** `python main.py --reset` (deletes `shipment_state.json`).
15. **Run only one copy of `main.py`** at a time (the same MQTT client id will kick the other one out).

---

## 2) Part 1: Raspberry Pi

### 2.1 System setup

```bash
sudo apt update
sudo apt install -y python3-venv python3-dev build-essential i2c-tools unzip
sudo timedatectl set-timezone Africa/Cairo
sudo raspi-config        # Interface Options → I2C → Yes → Finish  (if not already enabled)
sudo reboot
```

After rebooting, confirm the PN532 is visible:

```bash
i2cdetect -y 1
```
**Expected:** `24` appears in the `20:` row.

### 2.2 Wiring

| PN532 | Raspberry Pi |
|---|---|
| VCC | 3.3V (pin 1) |
| GND | GND (pin 6) |
| SDA | GPIO2 (pin 3) |
| SCL | GPIO3 (pin 5) |

The PN532's mode switches must be set to I2C (on the red module: SEL0 = ON, SEL1 = OFF).

| Component | Wiring |
|---|---|
| Green LED | GPIO17 (pin 11) ← 330Ω resistor ← LED (long leg +) ← GND (pin 9) |
| Red LED | GPIO27 (pin 13) ← 330Ω resistor ← LED ← GND (pin 14) |
| Buzzer (active) | GPIO22 (pin 15) ← buzzer (+), buzzer (−) ← GND (pin 20) |

### 2.3 Copying the project and creating the venv

From your own machine (where you downloaded the zip):
```bash
scp smart_delivery.zip pi@<PI-IP>:~/
```
(find the IP with `hostname -I` on the Pi; change the username if it isn't `pi`.)

On the Pi:
```bash
mkdir -p ~/smart_delivery
cd ~/smart_delivery
unzip ~/smart_delivery.zip
python3 -m venv venv --system-site-packages
source venv/bin/activate
pip install -r requirements.txt
```
**Expected:** after a couple of minutes you'll see `Successfully installed ...` and `(venv)` at the start of the prompt.
> Every time you open a new terminal: `cd ~/smart_delivery && source venv/bin/activate`

### 2.4 Testing the PN532 (Test 2)

```bash
python pn532_test.py
```
**Expected:**
```
Found PN532 - firmware 1.6
Waiting for a card... (Ctrl+C to stop)
UID: 04A1B2C3    (04 A1 B2 C3)
```
Tap each card on the reader and note its UID. Stop with `Ctrl+C`.

### 2.5 Configuration

1) Open `config.py`:
```bash
nano config.py
```
Change: `COURIER_CARD_UID` and `CUSTOMER_CARD_UID` (paste the UID; spaces and colons don't matter), `DESTINATION_LAT` / `DESTINATION_LON` / `DESTINATION_RADIUS_METERS`, and the GPIO pins if different. (Save with `Ctrl+O`, Enter, then `Ctrl+X`.)

2) Secrets file:
```bash
cp .env.example .env
nano .env
chmod 600 .env
```
Fill in `MQTT_HOST` (from your cluster page in HiveMQ, looks like `xxxx.s1.eu.hivemq.cloud`), `MQTT_USERNAME`, and `MQTT_PASSWORD`. The port stays `8883` (TLS).

> **HiveMQ Cloud:** at https://console.hivemq.cloud, open your cluster ← **Access Management** ← create new credentials with **Publish and Subscribe** permission. Best practice: create three separate users — `pi`, `nodered`, and `owntracks`.

### 2.6 OwnTracks on the courier's phone

OwnTracks ← Preferences ← **Connection**:
- **Mode:** MQTT (on Android it's called Private MQTT)
- **Host:** same host as your HiveMQ cluster, **Port:** `8883`, **TLS:** ON
- **Username / Password:** the `owntracks` user's credentials
- **Device ID:** e.g. `phone` → the topic becomes `owntracks/<username>/phone`

On the main screen, set **Monitoring mode = Move** while testing. The Pi listens on `owntracks/+/+` automatically; if you want it to listen to one specific topic, set `OWNTRACKS_TOPIC=...` in `.env`.

### 2.7 Running it (Test 1)

```bash
python main.py
```
**Expected:**
```
10:15:01 INFO    nfc: PN532 ready (firmware 1.6)
10:15:01 INFO    state: No saved state - new shipment starts in READY_FOR_PICKUP
10:15:01 INFO    main: Running - shipment SH001 is READY_FOR_PICKUP. Waiting for cards...
10:15:02 INFO    mqtt: MQTT connected to xxxx.s1.eu.hivemq.cloud
```
Stop with `Ctrl+C`. If you run it again without `--reset`, it loads the saved state (`Loaded saved state: ...`).

---

## 3) Part 2: Node-RED (click-by-click)

### 3.1 Installation (if not already installed)

```bash
bash <(curl -sL https://raw.githubusercontent.com/node-red/linux-installers/master/deb/update-nodejs-and-nodered)
sudo systemctl enable nodered.service
sudo systemctl start nodered.service
```
Open in a browser: `http://<PI-IP>:1880`
> **Security:** don't expose port 1880 to the internet and don't port-forward it. To protect it with a password on your local network, run `node-red admin init` and follow the prompts (adminAuth).

### 3.2 Installing the missing nodes (Manage Palette)

Repeat these steps **for each of the three:** `@flowfuse/node-red-dashboard`, then `node-red-contrib-web-worldmap`, then `node-red-node-email` (the last one is often already installed; skip it if you see it in the Nodes tab):

1. Click the **☰** icon (top right) ← **Manage palette**.
2. **Install** tab.
3. Type the package name in the search box.
4. Click **install** next to the right result ← **Install** in the confirmation dialog.
5. Wait for *"Node added to palette"* ← **Close**.

### 3.3 Importing the flow

1. Copy `node-red/smart_delivery_flow.json` to your machine (unzip it there).
2. In the Node-RED editor: **☰ ← Import**.
3. Click **select a file to import** and choose `smart_delivery_flow.json`.
4. Under *Import to* choose **new flow** ← click **Import**.
5. A tab called **Smart Delivery** will appear (if you see a "Node types missing" message, a package from 3.2 wasn't installed).

### 3.4 Settings after import (before Deploy)

**a) HiveMQ connection**
1. Double-click the **Pi events** node (top left).
2. Click the pencil icon ✏️ next to *Server*.
3. **Connection** tab: under *Server* type your host. *Port* = `8883`. (Leave *Enable secure (SSL/TLS) connection* checked.)
4. **Security** tab: *Username* and *Password* for the `nodered` user.
5. **Update** then **Done**.

**b) Tab environment variables** (all your Node-RED-side settings live here, nothing is hardcoded in the code)
1. Double-click the **Smart Delivery** tab name (in the tab bar).
2. Scroll to **Environment Variables** and set:
   - `SHIPMENT_ID` = `SH001` (must match `SHIPMENT_ID` in `config.py`)
   - `TELEGRAM_BOT_TOKEN` = your bot's token
   - `TELEGRAM_CHAT_ID` = the chat ID (send `/start` to the bot first, then message `@userinfobot` to get your ID — that's the private-chat chat_id)
   - `MANAGER_EMAIL` = the manager's email
3. **Done**.

**c) Email**
1. Double-click the **Send email** node.
2. *Server* = `smtp.gmail.com`, *Port* = `465`, *Use secure connection* checked.
3. *Userid* = your Gmail address, *Password* = an **App Password** (created from your Google account after enabling 2-Step Verification).
4. Leave the *To* field empty (the address comes from `MANAGER_EMAIL`).
5. **Done**.

**d) Deploy:** click the red **Deploy** button (top right).
**Expected:** a green *connected* dot appears under each MQTT node.

### 3.5 Opening the Dashboard and the Worldmap

- Dashboard: `http://<PI-IP>:1880/dashboard/delivery`
- Worldmap: `http://<PI-IP>:1880/worldmap`
- The layout is also reachable from the sidebar ← dropdown ← **Dashboard 2.0**.

### 3.6 Building the Dashboard by hand (optional — to understand / customize)

The imported flow already has all of this ready. If you want to build something similar from scratch:
1. **ui-base:** drag any widget (e.g. `ui-text`) onto the canvas ← double-click ← click ✏️ next to *Group* ← click ✏️ next to *Page* ← click ✏️ next to *UI* (this is the ui-base, Path = `/dashboard`) ← Add.
2. **ui-page:** Name = `Delivery`, Path = `/delivery` ← Add.
3. **ui-group:** Name = e.g. `Shipment`, Page = `Delivery`, Width = `6` ← Add.
4. **Widgets:** for each widget, pick its Group and set the Label.

### 3.7 Every node's details (Name / Type / Purpose / Input / Output / Connects To)

**Section 1 — Events (Telegram / Email / Log)**

| Node Name | Node Type | Purpose | Input | Output | Connects To |
|---|---|---|---|---|---|
| Pi events | mqtt in | Receives the Pi's events. Topic `delivery/+/events`, QoS 1, Output: parsed JSON | MQTT | `msg.payload` = event JSON | Drop duplicate events |
| Drop duplicate events | function | Prevents the same (event + timestamp) from being processed twice (extra, simple protection) | event JSON | same message, or nothing | Route by event, Add to event log, All events (debug) |
| All events (debug) | debug | Shows every event in the Debug sidebar for testing | event JSON | — | — |
| Add to event log | function | Keeps the last 20 events and turns them into one text block | event JSON | `msg.payload` = multi-line text | Event / trip log |
| Event / trip log | ui-template (group: Event / Trip Log) | Displays the log on the Dashboard | text | — | — |
| Route by event | switch | Routes by `msg.payload.event`. Output1 `ready_for_pickup`, Output2 `delivered`, Output3 `issue_reported`, Output4 `picked_up`/`customer_verified`/`nfc_rejected` | event JSON | 4 outputs | (1) Build Telegram request, (2) Build 'delivered' email, (3) Format issue info + Build 'issue' email, (4) Format last NFC event |
| Build Telegram request | function | Builds a `sendMessage` request with the text "الشحنة جاهزة للاستلام" | event | `msg.url` + `msg.payload` | Telegram sendMessage |
| Telegram sendMessage | http request (POST, JSON) | Sends to the Telegram Bot API | prepared request | Telegram's reply | Telegram API reply |
| Telegram API reply | debug | Shows Telegram's reply (`ok: true` = Telegram accepted the request) | HTTP reply | — | — |
| Build 'delivered' email | function | Builds the delivery-success email | `delivered` event | `msg.to` / `msg.topic` / `msg.payload` | Send email |
| Build 'issue' email | function | Builds the issue email (ID, type, details, time, last location + its timestamp) | `issue_reported` event | `msg.to` / `msg.topic` / `msg.payload` | Send email |
| Send email | e-mail | Sends via SMTP | prepared email | — | — |
| Format issue info | function | "Type / Details / Time" text | `issue_reported` event | `msg.payload` text | Issue information |
| Issue information | ui-text (group: Issue Report) | Shows the latest issue | text | — | — |
| Format last NFC event | function | Text describing the last NFC event | NFC event | `msg.payload` text | Last NFC event |
| Last NFC event | ui-text (group: Shipment) | Shows the last NFC event | text | — | — |

**Section 2 — State**

| Node Name | Node Type | Purpose | Input | Output | Connects To |
|---|---|---|---|---|---|
| Pi state | mqtt in | Topic `delivery/+/state`, QoS 1, JSON | MQTT | state JSON | Split state for widgets |
| Split state for widgets | function (4 outputs) | Formatting only: Output1 shipment ID, Output2 the state, Output3 Yes/No for the card, Output4 `{enabled: true/false}` for the button (true only when state = CUSTOMER_VERIFIED) | state JSON | 4 outputs | (1) Shipment ID, (2) Current Status, (3) Customer card verified, (4) Confirm Delivery |
| Shipment ID | ui-text (Shipment) | Shows the shipment ID | text | — | — |
| Current Status | ui-text (Shipment) | Shows the current state | text | — | — |
| Customer card verified | ui-text (Shipment) | Yes / No | text | — | — |
| Disable Confirm at start | inject (fires once, on deploy) | Starts the button disabled | — | `msg.enabled = false` | Confirm Delivery |
| Confirm Delivery | ui-button (group: Delivery Action) | The confirm button. Never changes any state itself | click + `msg.enabled` | `msg.payload = "confirm"` | Build confirm_delivery command |

**Section 3 — Location**

| Node Name | Node Type | Purpose | Input | Output | Connects To |
|---|---|---|---|---|---|
| Pi location | mqtt in | Topic `delivery/+/location`, QoS 1, JSON | MQTT | location JSON | Split location for widgets |
| Split location for widgets | function (5 outputs) | Formatting only (no geofence logic): lat, lon, time, OwnTracks info text, map message | location JSON | 5 outputs | Last known latitude, Last known longitude, Last location update time, OwnTracks information, Worldmap (courier) |
| Last known latitude / Last known longitude | ui-text ×2 (Location) | Coordinates of the last location | text | — | — |
| Last location update time | ui-text (Location) | Time of the last update | text | — | — |
| OwnTracks information | ui-text (Location) | Location source + time of last fix | text | — | — |
| Worldmap (courier) | worldmap | Plots the courier on the map (`{name, lat, lon}`) | `msg.payload` | — | — |
| Worldmap link | ui-template (Location) | A link that opens `/worldmap` | — | — | — |

**Section 4 — Commands**

| Node Name | Node Type | Purpose | Input | Output | Connects To |
|---|---|---|---|---|---|
| Build confirm_delivery command | function (2 outputs) | Builds `{"command":"confirm_delivery","shipment_id":...,"timestamp":...}` | button click | Output1 MQTT command (`msg.topic = delivery/<id>/commands`), Output2 status text | (1) Commands to Pi, (2) Command status |
| Issue Type | ui-dropdown (Issue Report) | The four exact issue types; doesn't send anything by itself | user selection | `msg.topic = "issue_type"` + the value | Remember issue type/details |
| Issue Details | ui-text-input (Issue Report) | Optional details | user typing | `msg.topic = "issue_details"` + the text | Remember issue type/details |
| Remember issue type/details | function (no output) | Stores both values in flow context only | from the two above | — | — |
| Report Issue | ui-button (Issue Report) | The report button | click | `msg.payload = "report"` | Build issue_report command |
| Build issue_report command | function (2 outputs) | Builds `{"command":"issue_report","shipment_id","issue_type","details","timestamp"}`; if no type is selected, returns a warning message instead | button click | Output1 MQTT command, Output2 status text | (1) Commands to Pi, (2) Command status |
| Commands to Pi | mqtt out | Publishes to `msg.topic`, QoS 1, retain = false | command JSON | — | — |
| Command status | ui-text (Delivery Action) | States that the command was sent and is awaiting the Pi | text | — | — |

Plus 4 `comment` nodes that divide the canvas into sections, and the config nodes: **HiveMQ Cloud** (mqtt-broker) and **HiveMQ TLS** (tls-config), **Smart Delivery** (ui-base) and **Default Theme** (ui-theme), **Delivery** (ui-page), and five groups: Shipment / Location (OwnTracks) / Delivery Action / Issue Report / Event / Trip Log.

**The Dashboard displays:** Shipment ID, Current Status, last known lat/lon, last location update time, OwnTracks info, last NFC event, event/trip log, issue information, customer card verification, a Confirm Delivery button (disabled until the Pi reports CUSTOMER_VERIFIED), Issue Type, Issue Details, and a Report Issue button.

---

## 4) Test sequence (9 tests)

Prepare: `python main.py --reset`, Node-RED deployed, and the Dashboard open.

| # | Do | Expected |
|---|---|---|
| 1 | Start the Pi and Node-RED | Pi log: `MQTT connected`, and the Dashboard shows `READY_FOR_PICKUP`. If you stop `main.py` and restart it (without `--reset`) it says `Loaded saved state` |
| 2 | `python pn532_test.py` | The UID appears with no errors |
| 3 | Tap the **courier card** | `PICKED_UP`, green LED, one short beep, `picked_up` event in the log, Dashboard updated |
| 4 | OwnTracks **outside** the geofence (press *Report location* in OwnTracks) | Location updates on the Dashboard and Worldmap; state stays `IN_TRANSIT` |
| 5 | Enter the geofence | State becomes `NEAR_DESTINATION`, **exactly one** `ready_for_pickup` event, **exactly one** Telegram message. Send another location update inside the geofence ← no second Telegram message |
| 6 | Tap the **customer card** | `customer_verified = Yes`, state `CUSTOMER_VERIFIED` (not DELIVERED), no email sent, Confirm Delivery button enabled |
| 7 | Press **Confirm Delivery** | The Pi validates ← `DELIVERED` ← `shipment_state.json` updated ← `delivered` event ← success email sent. Press it again ← no second delivery, no second email |
| 8 | Tap an unknown card, or tap the courier card in the wrong state | Red LED + two beeps + no state change (the log records `nfc_rejected`) |
| 9 | Pick an Issue Type, type Details, press **Report Issue** | The Pi records the issue ← `issue_reported` event ← email to the manager (with ID/type/details/time/last location + its timestamp) ← the state **does not change** (never marked Failed) |

**Tips for testing the geofence without moving around:** for Test 4, set `DESTINATION_LAT/LON` far from your current location. Then change them to your actual current coordinates (or increase `DESTINATION_RADIUS_METERS`), restart `main.py` (**without** `--reset` — the `IN_TRANSIT` state gets reloaded from the file), and press *Report location* in OwnTracks for Test 5.

---

## 5) MQTT summary

| Topic | Direction | Retained | Content |
|---|---|---|---|
| `delivery/SH001/events` | From Pi to Node-RED | No | `{event, shipment_id, timestamp, ...}` |
| `delivery/SH001/state` | From Pi to Node-RED | Yes | `{shipment_id, state, customer_verified, timestamp}` |
| `delivery/SH001/location` | From Pi to Node-RED | Yes | `{shipment_id, latitude, longitude, timestamp}` |
| `delivery/SH001/commands` | From Node-RED to Pi | No | `{command, shipment_id, timestamp, ...}` |
| `owntracks/<user>/<device>` | From OwnTracks to Pi | (OwnTracks sends it retained) | OwnTracks JSON |

Event names: `picked_up`, `in_transit`, `ready_for_pickup`, `customer_verified`, `delivered`, `issue_reported`, `nfc_rejected` (+ `card`, `reason`), `confirm_delivery_rejected` (+ `reason`), `issue_rejected` (+ `reason`).

`issue_reported` additionally includes: `issue_type`, `details`, `latitude`, `longitude`, `last_location_timestamp` (or `null` if there's no location yet).

---

## 6) Troubleshooting

- **`i2cdetect` doesn't show `24`:** re-check the wiring, VCC on 3.3V, and the PN532's mode switches (I2C).
- **Repeated `RuntimeError`s or timeouts from the PN532:** the Pi's I2C sometimes isn't compatible with the PN532 at the default speed. Open `sudo nano /boot/firmware/config.txt` and add the line `dtparam=i2c_arm_baudrate=10000`, then `sudo reboot`.
- **`MQTT connection refused`:** the host/username/password in `.env` are wrong, or the credentials don't have Publish & Subscribe permission.
- **No locations arriving:** open the Web Client in HiveMQ, subscribe to `owntracks/#`, and check the actual topic name; if it's different, set it as `OWNTRACKS_TOPIC`.
- **Dashboard is empty:** confirm the MQTT nodes are green, and that `main.py` is running (state and location are retained, so they appear as soon as Node-RED connects).
- **Telegram isn't arriving:** open the Debug sidebar and check **Telegram API reply**; if `ok: false` you'll see the reason (usually `chat not found` = you haven't sent `/start` to the bot, or the chat_id is wrong).
- **Email isn't sending:** check the status under the **Send email** node; for Gmail you need an App Password.
- **Buzzer is silent or always on:** flip `BUZZER_ACTIVE_HIGH` in `config.py`.
- **Confirm Delivery button seems wrongly enabled:** not a safety issue — the Pi still rejects the command if the state isn't `CUSTOMER_VERIFIED` (you'll see `confirm_delivery_rejected` in the log).

---

## 7) Security and realism

- **Never expose the Node-RED editor to the internet.** Use it from the local network only.
- HiveMQ credentials live in `.env` (not in the code), and `.env` is listed in `.gitignore`.
- The Dashboard never displays any credentials.
- The NFC UID is acceptable for this educational prototype, but it is **not strong production-grade authentication** (it can be cloned).
