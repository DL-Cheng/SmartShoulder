# Smart Shoulder Exercise Poster

A wall-poster shoulder trainer for older adults, aimed at preventing frozen
shoulder (50肩). Four copper-tape touch pads on a paper poster + an ESP32 +
4 LEDs + 1 buzzer, controlled from a phone web app over MQTT (EMQX).

```
shoulder-poster/
├── esp32/
│   └── SmartShoulderPoster.ino     Arduino sketch (ESP32, Arduino IDE 1.8.9)
├── web/
│   ├── index.html                  The whole web app (mode/level/goal + live view)
│   ├── manifest.json                PWA manifest (installable on phone)
│   ├── sw.js                        Service worker (offline app shell)
│   └── icons/icon-192.png, icon-512.png
└── README.md
```

## 1. Hardware wiring

| Part                  | ESP32 pin | Notes                                   |
|------------------------|-----------|------------------------------------------|
| Touch Pad 1 (Upper-Left)  | GPIO4  (T0) | copper tape, wire soldered to pad        |
| Touch Pad 2 (Upper-Right) | GPIO15 (T3) |                                            |
| Touch Pad 3 (Lower-Left)  | GPIO13 (T4) |                                            |
| Touch Pad 4 (Lower-Right) | GPIO14 (T6) |                                            |
| LED 1 (paired w/ Pad 1)   | GPIO16 | through a 220–330 Ω resistor to GND      |
| LED 2                     | GPIO17 | same                                      |
| LED 3                     | GPIO18 | same                                      |
| LED 4                     | GPIO19 | same                                      |
| Buzzer (+)                | GPIO21 | passive/piezo buzzer, driven by ledc PWM |

Arrange the 4 copper-tape pads (with their matching LED right next to each,
e.g. glued behind translucent paper) at four reach positions on the poster —
upper-left, upper-right, lower-left, lower-right — spaced to encourage a
real overhead/side reach. The poster hangs from a curtain rod, picture rail,
or 3M hook, so its height can be adjusted for different users.

## 2. Arduino firmware

1. Arduino IDE 1.8.9, board: **ESP32 Dev Module**. In Boards Manager,
   install (or update) **"esp32 by Espressif Systems" to a 3.x release** —
   the sketch uses the newer `ledcAttach(pin, freq, resolution)` LEDC API,
   which addresses the buzzer directly by GPIO pin instead of by channel
   number (the old `ledcSetup()` / `ledcAttachPin()` pair no longer exists
   on 3.x). The IDE itself can stay at version 1.8.9.
2. Library Manager → install:
   - **PubSubClient** (Nick O'Leary)
   - **ArduinoJson** (Benoit Blanchon, v6.x)
3. Open `esp32/SmartShoulderPoster.ino`, edit the **USER CONFIG** block at
   the top: WiFi SSID/password, MQTT broker host, and `DEVICE_ID`.
   `DEVICE_ID` must exactly match the "Device ID" field in the web app.
4. Upload. Open the Serial Monitor at 115200 baud — it prints touch
   calibration baselines at boot. If a pad doesn't register reliably,
   either physically increase the copper-tape area or hand-tune
   `touchThreshold[]` in the sketch.
5. By default the sketch uses the free public broker `broker.emqx.io`
   (port 1883, no auth) — good for testing, but it's a shared/public
   broker with no privacy. For real use, create a free **EMQX Cloud
   Serverless** deployment, and put its host + (if you enabled auth)
   username/password into the sketch and into the web app's Advanced
   settings.

## 3. Web app

`web/index.html` is a self-contained page (no build step) that:
- lets you choose **mode** (Stretch Guide / Simon Says) and **difficulty**
  (Level 1 = 3.0 s, Level 2 = 2.0 s, Level 3 = 1.0 s reaction window),
- shows **today's suggested goal**, a random number between 10–60 reps,
  freshly picked once per day (tap the refresh icon to re-roll manually),
- mirrors the poster's LEDs live via the "Reach Compass" diagram,
- shows a **results screen** at the end of every session: mode, reps
  completed vs. goal, and an encouraging message.

### Deploying to GitHub Pages

1. Push the `web/` folder's contents to the root of a GitHub repo (or to
   a `/docs` folder, or a `gh-pages` branch — your choice).
2. Repo → **Settings → Pages** → set the source to that folder/branch.
3. Visit the published URL. On first load, open **Advanced: device
   connection** and set:
   - **MQTT broker (WebSocket URL)** — for the public test broker use
     `wss://broker.emqx.io:8084/mqtt`. GitHub Pages is served over HTTPS,
     so the broker must be `wss://` (secure WebSocket), not `ws://`.
   - **Device ID** — must match `DEVICE_ID` in the Arduino sketch.
   - Username/Password — only needed if your EMQX Cloud deployment
     requires auth.
4. Tap **Reconnect**. The status dot in the header turns green once the
   ESP32 is online and the app is subscribed.

### Installing it like a phone app

- **Android/Chrome**: an "Install App" button appears automatically
  (powered by the `beforeinstallprompt` event) once the site is visited
  over HTTPS with a valid manifest — tap it.
- **iPhone/Safari**: tap the Share icon → "Add to Home Screen" (iOS does
  not support the automatic install prompt).

## 4. MQTT contract

Topic prefix: `sspv1/<DEVICE_ID>/...`

| Topic      | Direction        | Payload (JSON)                                                                 |
|------------|-------------------|----------------------------------------------------------------------------------|
| `.../cmd`    | web → ESP32     | `{"action":"config","mode":"STRETCH\|SIMON","level":1-3,"target":<int>}`<br>`{"action":"start"}`<br>`{"action":"stop"}` |
| `.../state`  | ESP32 → web     | `{"mode":"STRETCH","level":2,"running":true,"activeLed":1,"count":7,"target":35}` |
| `.../result` | ESP32 → web     | `{"mode":"STRETCH","count":7,"target":35,"completed":false}` (sent once, when a session ends) |
| `.../status` | ESP32 → web (retained, LWT) | `"online"` / `"offline"` |

## 5. How each mode works

**Stretch Guide** — LEDs light one at a time in a random order (never the
same position twice in a row), covering Upper-Left, Upper-Right, Lower-Left,
and Lower-Right. The buzzer plays a distinct note for whichever LED is lit.
The user reaches up and touches the matching copper pad before the
difficulty timer runs out. A miss is never penalized — a gentle two-beep
reminder plays and the poster simply moves on to the next (random) position,
keeping the exercise stress-free for older adults.

**Simon Says** — classic growing-memory game. The poster plays back a
sequence of light+tone flashes (one new step added each round), then waits
for the user to reproduce it by touching pads in the same order. A wrong
pad, or running out of time, ends the session. Every correct tap counts as
one completed rep (not just a fully-repeated round), so progress toward the
daily goal keeps climbing throughout each round, not only at the end of it.

In both modes, reaching the day's goal ends the session with a small
celebratory jingle from the buzzer; stopping manually (or a Simon miss)
ends it with a softer chime. Either way, the **result appears on the web
app** — mode played, reps completed, and an encouraging message chosen to
match how the session went.
