#include <WiFi.h>
#include <WebServer.h>
#include <DHT.h>

// --- Access Point Credentials ---
const char* ssid = "ESP32-SmartHome"; 
const char* password = "Password123"; // Must be at least 8 chars

WebServer server(80);

// --- DHT22 Sensors ---
#define DHTTYPE DHT22
#define DHT_MAIN 4
#define DHT_BED 5
#define DHT_KIT 18

DHT dhtMain(DHT_MAIN, DHTTYPE);
DHT dhtBed(DHT_BED, DHTTYPE);
DHT dhtKit(DHT_KIT, DHTTYPE);

// --- Ultrasonic Sensors (Trig, Echo) ---
#define TRIG_MAIN 12
#define ECHO_MAIN 14
#define TRIG_BED 27
#define ECHO_BED 35  // Input Only
#define TRIG_KIT 25
#define ECHO_KIT 33

// --- Gas Sensor ---
#define MQ2_KIT 34

// --- Indicator LEDs & Buzzer ---
// One white LED per room
#define LED_MAIN 19
#define LED_BED 23
#define LED_KIT 15

// The physical buzzer
#define BUZZER 26

// --- Backend State Variables ---
// Individual light states
bool lightMain = false;
bool lightBed = false;
bool lightKit = false;

bool alertsArmedMain = true;
bool alertsArmedBed = true;
bool alertsArmedKit = true;

bool alertActiveMain = false;
bool alertActiveBed = false;
bool alertActiveKit = false;

// Non-blocking timer variables
unsigned long previousMillis = 0;
const long interval = 2000; 

float t_main = 0, h_main = 0, d_main = 0;
float t_bed = 0, h_bed = 0, d_bed = 0;
float t_kit = 0, h_kit = 0, d_kit = 0;
int gas_kit = 0;

// ==========================================
// FRONTEND: HTML / CSS / JS (Stored in Memory)
// Contains the Web Audio Siren & New Thresholds
// ==========================================
const char index_html[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>ESP32 Smart Home</title>
<style>
  :root {
    --bg: #f6f8fb;
    --fg: #0f172a;
    --muted: #64748b;
    --card: #ffffff;
    --border: #e2e8f0;
    --primary: #0ea5e9;
    --primary-fg: #ffffff;
    --danger: #ef4444;
    --warn: #f59e0b;
    --ok: #10b981;
    --shadow: 0 2px 6px rgba(0,0,0,.06), 0 1px 2px rgba(0,0,0,.04);
    --radius: 14px;
  }
  html.dark {
    --bg: #0b1220;
    --fg: #e2e8f0;
    --muted: #94a3b8;
    --card: #0f172a;
    --border: #1e293b;
    --primary: #38bdf8;
    --primary-fg: #0b1220;
    --danger: #f87171;
    --warn: #fbbf24;
    --ok: #34d399;
    --shadow: 0 2px 6px rgba(0,0,0,.4), 0 1px 2px rgba(0,0,0,.3);
  }
  * { box-sizing: border-box; }
  html, body { margin: 0; padding: 0; }
  body {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
    background: var(--bg);
    color: var(--fg);
    min-height: 100vh;
    -webkit-font-smoothing: antialiased;
    transition: background .25s ease, color .25s ease;
  }
  @keyframes sirenFlash { 
    0% { background-color: rgba(239, 68, 68, 0.2); } 
    50% { background-color: rgba(239, 68, 68, 0.8); } 
    100% { background-color: rgba(239, 68, 68, 0.2); } 
  }
  body.alarm-active {
    animation: sirenFlash 0.8s infinite;
  }
  header {
    position: sticky; top: 0; z-index: 5;
    background: color-mix(in srgb, var(--card) 90%, transparent);
    backdrop-filter: blur(8px);
    border-bottom: 1px solid var(--border);
  }
  .wrap {
    max-width: 1100px; margin: 0 auto; padding: 16px 20px;
    display: flex; align-items: center; justify-content: space-between;
    gap: 12px; flex-wrap: wrap;
  }
  .brand { display: flex; align-items: center; gap: 12px; }
  .logo {
    width: 40px; height: 40px; border-radius: 12px;
    background: color-mix(in srgb, var(--primary) 15%, transparent);
    color: var(--primary);
    display: grid; place-items: center; font-size: 20px;
  }
  h1 { font-size: 18px; margin: 0; }
  .sub { font-size: 12px; color: var(--muted); margin-top: 2px; }
  .badge {
    display: inline-flex; align-items: center; gap: 6px;
    font-size: 12px; font-weight: 500;
    padding: 6px 10px; border-radius: 999px;
    border: 1px solid var(--border); background: var(--card);
  }
  .badge.ok    { color: var(--ok);   border-color: color-mix(in srgb, var(--ok) 40%, var(--border)); background: color-mix(in srgb, var(--ok) 12%, var(--card)); }
  .badge.warn  { color: var(--warn); border-color: color-mix(in srgb, var(--warn) 40%, var(--border)); background: color-mix(in srgb, var(--warn) 12%, var(--card)); }
  .dot { width: 8px; height: 8px; border-radius: 50%; background: currentColor; }

  main { max-width: 1100px; margin: 0 auto; padding: 20px; }
  .summary {
    display: grid; gap: 14px;
    grid-template-columns: repeat(3, minmax(0, 1fr));
    margin-bottom: 18px;
  }
  .grid {
    display: grid; gap: 14px;
    grid-template-columns: repeat(3, minmax(0, 1fr));
  }
  @media (max-width: 820px) { .summary, .grid { grid-template-columns: 1fr; } }

  .card {
    background: var(--card);
    border: 1px solid var(--border);
    border-radius: var(--radius);
    box-shadow: var(--shadow);
    padding: 16px; position: relative; overflow: hidden;
  }
  .card.alert { border-color: color-mix(in srgb, var(--danger) 60%, var(--border)); }
  .card.alert::before {
    content: ""; position: absolute; inset: 0 0 auto 0; height: 3px;
    background: var(--danger); animation: pulse 1.2s ease-in-out infinite;
  }
  @keyframes pulse { 0%,100%{opacity:1} 50%{opacity:.4} }

  .row { display: flex; align-items: center; justify-content: space-between; gap: 10px; }
  .label { font-size: 12px; color: var(--muted); }
  .big { font-size: 28px; font-weight: 600; margin-top: 4px; font-variant-numeric: tabular-nums; }

  .room-head { display: flex; align-items: flex-start; justify-content: space-between; gap: 10px; margin-bottom: 12px; }
  .room-name { font-size: 17px; font-weight: 600; margin: 0; }
  .room-id { font-size: 11px; color: var(--muted); margin-top: 2px; }

  .metrics { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
  .metric {
    border: 1px solid var(--border); border-radius: 10px;
    padding: 10px 12px;
    background: color-mix(in srgb, var(--card) 70%, var(--bg));
  }
  .metric .label { display: flex; align-items: center; gap: 6px; }
  .metric .value { font-size: 22px; font-weight: 600; margin-top: 4px; font-variant-numeric: tabular-nums; }
  .v-warn   { color: var(--warn); }
  .v-danger { color: var(--danger); }

  .alert-row {
    margin-top: 14px;
    display: flex; align-items: center; justify-content: space-between;
    padding: 10px 12px;
    border: 1px solid var(--border); border-radius: 10px;
    background: color-mix(in srgb, var(--card) 70%, var(--bg));
  }
  .alert-row .ttl { font-size: 14px; font-weight: 500; }
  .alert-row .desc { font-size: 12px; color: var(--muted); }

  .switch { position: relative; width: 42px; height: 24px; flex-shrink: 0; }
  .switch input { opacity: 0; width: 0; height: 0; }
  .slider {
    position: absolute; cursor: pointer; inset: 0;
    background: var(--border); border-radius: 999px;
    transition: background .2s ease;
  }
  .slider::before {
    content: ""; position: absolute;
    height: 18px; width: 18px; left: 3px; top: 3px;
    background: #fff; border-radius: 50%;
    transition: transform .2s ease;
  }
  .switch input:checked + .slider { background: var(--primary); }
  .switch input:checked + .slider::before { transform: translateX(18px); }
  .switch input:disabled + .slider { opacity: .6; cursor: not-allowed; }

  .btn {
    display: inline-flex; align-items: center; gap: 6px;
    padding: 8px 14px;
    border-radius: 10px;
    border: 1px solid var(--border);
    background: var(--card); color: var(--fg);
    font-size: 14px; font-weight: 500;
    cursor: pointer;
    transition: background .15s ease;
  }
  .btn:hover { background: color-mix(in srgb, var(--fg) 6%, var(--card)); }
  .btn.primary { background: var(--primary); color: var(--primary-fg); border-color: var(--primary); }

  .pill {
    display: inline-flex; align-items: center; gap: 6px;
    padding: 4px 10px; border-radius: 999px;
    font-size: 11px; font-weight: 500;
    background: color-mix(in srgb, var(--fg) 8%, transparent);
    color: var(--muted);
  }
  .pill.danger { color: var(--danger); background: color-mix(in srgb, var(--danger) 14%, transparent); }

  .lights-card.on { background: color-mix(in srgb, var(--warn) 14%, var(--card)); border-color: color-mix(in srgb, var(--warn) 40%, var(--border)); }

  footer { text-align: center; color: var(--muted); font-size: 12px; padding: 24px 16px; }
  code { font-family: ui-monospace, SFMono-Regular, Menlo, monospace; font-size: 12px; }

  .toasts { position: fixed; bottom: 20px; right: 20px; display: grid; gap: 10px; z-index: 100; }
  .toast {
    background: var(--card);
    border: 1px solid var(--border);
    border-left: 3px solid var(--danger);
    border-radius: 10px;
    padding: 10px 14px;
    box-shadow: var(--shadow);
    min-width: 240px;
    animation: slidein .25s ease-out;
  }
  .toast .t { font-size: 13px; font-weight: 600; }
  .toast .d { font-size: 12px; color: var(--muted); margin-top: 2px; }
  @keyframes slidein { from { transform: translateY(8px); opacity: 0; } to { transform: none; opacity: 1; } }
</style>
</head>
<body>

<header>
  <div class="wrap">
    <div class="brand">
      <div class="logo">⌂</div>
      <div>
        <h1>Smart Home Control</h1>
        <div class="sub">ESP32 · 3 rooms · live monitoring</div>
      </div>
    </div>
    <div id="conn" class="badge warn">
      <span class="dot"></span>
      <span id="conn-text">Connecting…</span>
    </div>
  </div>
</header>

<main>
  <div style="text-align: center; margin-bottom: 20px; font-size: 12px; color: var(--muted);">
    🔊 Click anywhere on the page to unlock the Web Alarm Audio.
  </div>
  <section class="summary">
    <div class="card">
      <div class="row">
        <div>
          <div class="label">Active alerts</div>
          <div id="alert-count" class="big">0</div>
        </div>
        <div class="logo" id="alert-icon" style="background: color-mix(in srgb, var(--ok) 15%, transparent); color: var(--ok);">!</div>
      </div>
    </div>

    <div class="card">
      <div class="row">
        <div>
          <div class="label">Uptime</div>
          <div id="uptime" class="big">—</div>
        </div>
        <div class="logo">⏱</div>
      </div>
    </div>

    <div id="lights-card" class="card lights-card">
      <div class="row">
        <div>
          <div class="label">House lights</div>
          <div id="lights-text" class="big" style="font-size: 18px;">Off</div>
        </div>
        <button id="mode-btn" class="btn primary">💡 Turn All On</button>
      </div>
      <div style="margin-top: 10px; font-size: 12px; color: var(--muted);">
        Master control for all 3 LEDs
      </div>
    </div>
  </section>

  <section id="rooms" class="grid"></section>

  <footer>
    Connect to <code id="ssid">ESP32-SmartHome</code> Wi-Fi to control your house.
  </footer>
</main>

<div id="toasts" class="toasts"></div>

<script>
(() => {
  const API = "/api";
  const POLL_MS = 2000;

  const $ = (id) => document.getElementById(id);
  const roomsEl    = $("rooms");
  const connEl     = $("conn");
  const connText   = $("conn-text");
  const alertCount = $("alert-count");
  const alertIcon  = $("alert-icon");
  const uptimeEl   = $("uptime");
  const lightsCard = $("lights-card");
  const lightsText = $("lights-text");
  const modeBtn    = $("mode-btn");
  const ssidEl     = $("ssid");
  const toastsEl   = $("toasts");

  let lastAlertKey = "";
  let pendingMode = false;
  const pendingAlerts = new Set();
  
  // --- Web Audio API for Siren ---
  let audioCtx = null;
  let webAlarmInterval = null;

  function initAudio() {
    if (!audioCtx) {
      audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    }
    if (audioCtx.state === 'suspended') {
      audioCtx.resume();
    }
  }
  document.body.addEventListener('click', initAudio, { once: true });

  function playSiren() {
    if (!audioCtx) return;
    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    osc.connect(gain);
    gain.connect(audioCtx.destination);
    
    osc.type = 'square';
    osc.frequency.setValueAtTime(800, audioCtx.currentTime); 
    osc.frequency.setValueAtTime(1200, audioCtx.currentTime + 0.3); 
    
    gain.gain.setValueAtTime(0.1, audioCtx.currentTime);
    osc.start();
    gain.gain.exponentialRampToValueAtTime(0.00001, audioCtx.currentTime + 0.6);
    osc.stop(audioCtx.currentTime + 0.7);
  }

  // --- Offline Fallback Demo Data ---
  let demo = {
    rooms: [
      { id: "room-1", name: "Living Room", temperature: 22.0, humidity: 45, distanceCm: 120,                 alertsEnabled: true, alertActive: false, lightOn: false },
      { id: "room-2", name: "Bedroom",     temperature: 23.0, humidity: 48, distanceCm: 90,                  alertsEnabled: true, alertActive: false, lightOn: false },
      { id: "room-3", name: "Kitchen",     temperature: 24.0, humidity: 51, distanceCm: 60,                  alertsEnabled: true, alertActive: false, gasPpm: 120, lightOn: false }
    ],
    deviceConnected: false,
    uptimeSec: 0,
    ssid: "ESP32-SmartHome"
  };
  const jitter = (v, a, mn, mx) => Math.max(mn, Math.min(mx, +(v + (Math.random() - .5) * a).toFixed(1)));
  const tickDemo = () => {
    demo.uptimeSec += 2;
    demo.rooms = demo.rooms.map(r => {
      const t = r.temperature !== undefined ? jitter(r.temperature, .4, 16, 50) : undefined;
      const h = r.humidity    !== undefined ? jitter(r.humidity,   1.2, 20, 90) : undefined;
      const d = r.distanceCm  !== undefined ? jitter(r.distanceCm,  8, 5, 300)  : undefined;
      const g = r.gasPpm      !== undefined ? jitter(r.gasPpm,     12, 30, 600) : undefined;
      
      const danger =
        (g !== undefined && g > 350) ||
        (t !== undefined && t > 47)  ||
        (d !== undefined && d < 15);
      const next = { ...r, temperature: t, humidity: h, distanceCm: d,
                     alertActive: r.alertsEnabled && danger };
      if (g !== undefined) next.gasPpm = g; else delete next.gasPpm;
      return next;
    });
    return { ...demo };
  };

  async function call(path, opts) {
    const ctrl = new AbortController();
    const tm = setTimeout(() => ctrl.abort(), 2500);
    try {
      const res = await fetch(API + path, {
        ...opts,
        signal: ctrl.signal,
        headers: { "content-type": "application/json", ...((opts && opts.headers) || {}) }
      });
      if (!res.ok) throw new Error("HTTP " + res.status);
      return await res.json();
    } finally { clearTimeout(tm); }
  }

  async function fetchState() {
    try {
      const data = await call("/state");
      return { ...data, deviceConnected: true };
    } catch (e) {
      const s = tickDemo();
      return { ...s, deviceConnected: false };
    }
  }

  async function postAlerts(roomId, enabled) {
    pendingAlerts.add(roomId);
    try {
      try {
        const data = await call("/alerts", {
          method: "POST",
          body: JSON.stringify({ roomId, enabled })
        });
        render({ ...data, deviceConnected: true });
      } catch {
        demo.rooms = demo.rooms.map(r =>
          r.id === roomId ? { ...r, alertsEnabled: enabled, alertActive: enabled && r.alertActive } : r
        );
        render({ ...demo, deviceConnected: false });
      }
    } finally {
      pendingAlerts.delete(roomId);
    }
  }

  // Updated to handle individual or "all" lights
  async function postLight(roomId, state) {
    pendingMode = true;
    try {
      try {
        const data = await call("/lights", {
          method: "POST",
          body: JSON.stringify({ roomId, state })
        });
        render({ ...data, deviceConnected: true });
      } catch {
        if (roomId === "all") {
          demo.rooms.forEach(r => r.lightOn = state);
        } else {
          demo.rooms.forEach(r => { if(r.id === roomId) r.lightOn = state; });
        }
        render({ ...demo, deviceConnected: false });
      }
    } finally {
      pendingMode = false;
    }
  }

  function fmtUptime(sec) {
    const m = Math.floor(sec / 60);
    const s = sec % 60;
    return m + "m " + s + "s";
  }

  function metricClass(v, warn, danger) {
    if (v >= danger) return "v-danger";
    if (v >= warn)   return "v-warn";
    return "";
  }

  function toast(title, desc) {
    const t = document.createElement("div");
    t.className = "toast";
    t.innerHTML = '<div class="t"></div><div class="d"></div>';
    t.querySelector(".t").textContent = title;
    t.querySelector(".d").textContent = desc;
    toastsEl.appendChild(t);
    setTimeout(() => t.remove(), 4500);
  }

  function metric(icon, label, value, cls) {
    return '<div class="metric">' +
             '<div class="label">' + icon + ' ' + label + '</div>' +
             '<div class="value ' + cls + '">' + value + '</div>' +
           '</div>';
  }

  function render(state) {
    if(!state) return;
    
    if (state.deviceConnected) {
      connEl.className = "badge ok";
      connText.textContent = "Connected · " + state.ssid;
    } else {
      connEl.className = "badge warn";
      connText.textContent = "Demo mode (not on ESP32 Wi-Fi)";
    }
    ssidEl.textContent = state.ssid;

    const alerting = state.rooms.filter(r => r.alertActive).length;
    alertCount.textContent = alerting;
    
    if (alerting > 0) {
      alertIcon.style.background = "color-mix(in srgb, var(--danger) 15%, transparent)";
      alertIcon.style.color = "var(--danger)";
      document.body.classList.add('alarm-active');
      if (!webAlarmInterval) webAlarmInterval = setInterval(playSiren, 800);
    } else {
      alertIcon.style.background = "color-mix(in srgb, var(--ok) 15%, transparent)";
      alertIcon.style.color = "var(--ok)";
      document.body.classList.remove('alarm-active');
      if (webAlarmInterval) { clearInterval(webAlarmInterval); webAlarmInterval = null; }
    }

    uptimeEl.textContent = fmtUptime(state.uptimeSec);

    // Compute master light state
    const allLightsOn = state.rooms.every(r => r.lightOn);
    const anyLightsOn = state.rooms.some(r => r.lightOn);

    lightsCard.classList.toggle("on", anyLightsOn);
    lightsText.textContent = anyLightsOn ? (allLightsOn ? "All On" : "Some On") : "All Off";
    modeBtn.disabled = pendingMode;
    modeBtn.textContent = anyLightsOn ? "Turn All Off" : "💡 Turn All On";
    modeBtn.onclick = () => { initAudio(); postLight("all", !anyLightsOn); };

    roomsEl.innerHTML = "";
    state.rooms.forEach(r => {
      const card = document.createElement("div");
      card.className = "card" + (r.alertActive ? " alert" : "");

      const tCls = r.temperature !== undefined ? metricClass(r.temperature, 44, 47) : "";
      const dCls = r.distanceCm  !== undefined && r.distanceCm < 15 ? "v-warn" : "";
      const gCls = r.gasPpm      !== undefined ? metricClass(r.gasPpm, 250, 350) : "";
      const pendingAlt = pendingAlerts.has(r.id);

      let metricsHtml = "";
      if (r.temperature !== undefined) metricsHtml += metric("🌡", "Temp", r.temperature.toFixed(1) + "°C", tCls);
      if (r.humidity    !== undefined) metricsHtml += metric("💧", "Humid",    Math.round(r.humidity) + "%",     "");
      if (r.distanceCm  !== undefined) metricsHtml += metric("📏", "Dist",    Math.round(r.distanceCm) + " cm", dCls);
      if (r.gasPpm      !== undefined) metricsHtml += metric("💨", "Gas",         Math.round(r.gasPpm) + " ppm",    gCls);

      card.innerHTML =
        '<div class="room-head">' +
          '<div>' +
            '<h2 class="room-name"></h2>' +
            '<div class="room-id"></div>' +
          '</div>' +
          '<div style="display:flex; gap:6px; flex-wrap:wrap; justify-content:flex-end;">' +
            (r.alertActive ? '<span class="pill danger">⚠ Alert</span>' : '') +
            '<span class="pill">' + (r.alertsEnabled ? '🔔 Armed' : '🔕 Off') + '</span>' +
          '</div>' +
        '</div>' +
        '<div class="metrics">' + metricsHtml + '</div>' +
        '<div class="alert-row">' +
          '<div>' +
            '<div class="ttl">System Alert</div>' +
          '</div>' +
          '<label class="switch">' +
            '<input type="checkbox" class="alert-toggle" ' + (r.alertsEnabled ? 'checked' : '') + (pendingAlt ? ' disabled' : '') + '>' +
            '<span class="slider"></span>' +
          '</label>' +
        '</div>' +
        '<div class="alert-row" style="margin-top: 8px;">' +
          '<div>' +
            '<div class="ttl">Room Light</div>' +
          '</div>' +
          '<label class="switch">' +
            '<input type="checkbox" class="light-toggle" ' + (r.lightOn ? 'checked' : '') + (pendingMode ? ' disabled' : '') + '>' +
            '<span class="slider"></span>' +
          '</label>' +
        '</div>';

      card.querySelector(".room-name").textContent = r.name;
      card.querySelector(".room-id").textContent = r.id;
      
      card.querySelector(".alert-toggle")
          .addEventListener("change", (e) => { initAudio(); postAlerts(r.id, e.target.checked); });
          
      card.querySelector(".light-toggle")
          .addEventListener("change", (e) => { initAudio(); postLight(r.id, e.target.checked); });

      roomsEl.appendChild(card);
    });

    const key = state.rooms.map(r => r.id + ":" + (r.alertActive ? "1" : "0")).join("|");
    if (key !== lastAlertKey) {
      state.rooms.forEach(r => {
        if (r.alertActive && !lastAlertKey.includes(r.id + ":1")) {
          toast("Alert · " + r.name, "A sensor reading crossed a safety threshold.");
        }
      });
      lastAlertKey = key;
    }
  }

  async function loop() {
    const s = await fetchState();
    render(s);
  }
  loop();
  setInterval(loop, POLL_MS);
})();
</script>
</body>
</html>
)rawliteral";
// ==========================================
// END OF FRONTEND
// ==========================================

// --- Helper Functions ---
float readDistance(int trigPin, int echoPin) {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  long duration = pulseIn(echoPin, HIGH, 30000); 
  if (duration == 0) return -1; 
  return duration * 0.034 / 2;
}

// Generate the JSON State for the Website
String generateStateJson() {
  String json = "{";
  json += "\"rooms\": [";
  json += "{\"id\": \"room-1\", \"name\": \"Living Room\", \"temperature\": " + String(t_main) + ", \"humidity\": " + String(h_main) + ", \"distanceCm\": " + String(d_main) + ", \"alertsEnabled\": " + (alertsArmedMain ? "true" : "false") + ", \"alertActive\": " + (alertActiveMain ? "true" : "false") + ", \"lightOn\": " + (lightMain ? "true" : "false") + "},";
  json += "{\"id\": \"room-2\", \"name\": \"Bedroom\", \"temperature\": " + String(t_bed) + ", \"humidity\": " + String(h_bed) + ", \"distanceCm\": " + String(d_bed) + ", \"alertsEnabled\": " + (alertsArmedBed ? "true" : "false") + ", \"alertActive\": " + (alertActiveBed ? "true" : "false") + ", \"lightOn\": " + (lightBed ? "true" : "false") + "},";
  json += "{\"id\": \"room-3\", \"name\": \"Kitchen\", \"temperature\": " + String(t_kit) + ", \"humidity\": " + String(h_kit) + ", \"distanceCm\": " + String(d_kit) + ", \"gasPpm\": " + String(gas_kit) + ", \"alertsEnabled\": " + (alertsArmedKit ? "true" : "false") + ", \"alertActive\": " + (alertActiveKit ? "true" : "false") + ", \"lightOn\": " + (lightKit ? "true" : "false") + "}";
  json += "],";
  json += "\"deviceConnected\": true,";
  json += "\"uptimeSec\": " + String(millis() / 1000) + ",";
  json += "\"ssid\": \"" + String(ssid) + "\"";
  json += "}";
  return json;
}

// --- Web Server Handlers ---
void handleRoot() {
  server.send(200, "text/html", index_html);
}

void handleGetState() {
  server.send(200, "application/json", generateStateJson());
}

void handlePostLights() {
  if (server.hasArg("plain")) {
    String body = server.arg("plain");
    
    // Check if the payload says state is true (turn on)
    bool isTurnOn = (body.indexOf("\"state\":true") > 0 || body.indexOf("\"state\": true") > 0);
    
    // Check which room to target, or "all"
    if (body.indexOf("\"roomId\":\"all\"") > 0 || body.indexOf("\"roomId\": \"all\"") > 0) {
      lightMain = isTurnOn;
      lightBed = isTurnOn;
      lightKit = isTurnOn;
    } else if (body.indexOf("\"roomId\":\"room-1\"") > 0 || body.indexOf("\"roomId\": \"room-1\"") > 0) {
      lightMain = isTurnOn;
    } else if (body.indexOf("\"roomId\":\"room-2\"") > 0 || body.indexOf("\"roomId\": \"room-2\"") > 0) {
      lightBed = isTurnOn;
    } else if (body.indexOf("\"roomId\":\"room-3\"") > 0 || body.indexOf("\"roomId\": \"room-3\"") > 0) {
      lightKit = isTurnOn;
    }
    
    // Update the white LEDs based on the new individual states
    digitalWrite(LED_MAIN, lightMain ? HIGH : LOW);
    digitalWrite(LED_BED, lightBed ? HIGH : LOW);
    digitalWrite(LED_KIT, lightKit ? HIGH : LOW);
  }
  server.send(200, "application/json", generateStateJson());
}

void handlePostAlerts() {
  if (server.hasArg("plain")) {
    String body = server.arg("plain");
    bool isEnabled = (body.indexOf("\"enabled\":true") > 0 || body.indexOf("\"enabled\": true") > 0);
    
    if (body.indexOf("\"roomId\":\"room-1\"") > 0 || body.indexOf("\"roomId\": \"room-1\"") > 0) alertsArmedMain = isEnabled;
    else if (body.indexOf("\"roomId\":\"room-2\"") > 0 || body.indexOf("\"roomId\": \"room-2\"") > 0) alertsArmedBed = isEnabled;
    else if (body.indexOf("\"roomId\":\"room-3\"") > 0 || body.indexOf("\"roomId\": \"room-3\"") > 0) alertsArmedKit = isEnabled;
  }
  server.send(200, "application/json", generateStateJson());
}

void setup() {
  Serial.begin(115200);

  // Init LEDs
  pinMode(LED_MAIN, OUTPUT); digitalWrite(LED_MAIN, LOW);
  pinMode(LED_BED, OUTPUT); digitalWrite(LED_BED, LOW);
  pinMode(LED_KIT, OUTPUT); digitalWrite(LED_KIT, LOW);
  
  pinMode(BUZZER, OUTPUT); digitalWrite(BUZZER, LOW);

  // Init HC-SR04
  pinMode(TRIG_MAIN, OUTPUT); pinMode(ECHO_MAIN, INPUT);
  pinMode(TRIG_BED, OUTPUT); pinMode(ECHO_BED, INPUT);
  pinMode(TRIG_KIT, OUTPUT); pinMode(ECHO_KIT, INPUT);

  // Init DHT
  dhtMain.begin();
  dhtBed.begin();
  dhtKit.begin();

  // Start Access Point
  Serial.println("\nSetting up Access Point...");
  WiFi.softAP(ssid, password);
  
  IPAddress IP = WiFi.softAPIP();
  Serial.print("AP IP address: ");
  Serial.println(IP); 

  // Start API Routes
  server.on("/", HTTP_GET, handleRoot);
  server.on("/api/state", HTTP_GET, handleGetState);
  server.on("/api/lights", HTTP_POST, handlePostLights);
  server.on("/api/alerts", HTTP_POST, handlePostAlerts);

  server.begin();
}

void loop() {
  server.handleClient(); 

  unsigned long currentMillis = millis();
  if (currentMillis - previousMillis >= interval) {
    previousMillis = currentMillis;

    // Read Sensors
    t_main = dhtMain.readTemperature();
    h_main = dhtMain.readHumidity();
    d_main = readDistance(TRIG_MAIN, ECHO_MAIN);

    t_bed = dhtBed.readTemperature();
    h_bed = dhtBed.readHumidity();
    d_bed = readDistance(TRIG_BED, ECHO_BED);

    t_kit = dhtKit.readTemperature();
    h_kit = dhtKit.readHumidity();
    d_kit = readDistance(TRIG_KIT, ECHO_KIT);
    
    gas_kit = analogRead(MQ2_KIT);
    
    // Clean NaN values for JSON
    if(isnan(t_main)) t_main = 0; if(isnan(h_main)) h_main = 0;
    if(isnan(t_bed)) t_bed = 0; if(isnan(h_bed)) h_bed = 0;
    if(isnan(t_kit)) t_kit = 0; if(isnan(h_kit)) h_kit = 0;

    // --- Safety Alert Logic ---
    alertActiveMain = (d_main > 0 && d_main < 15) || (t_main > 47);
    alertActiveBed = (d_bed > 0 && d_bed < 15) || (t_bed > 47);
    alertActiveKit = (d_kit > 0 && d_kit < 15) || (t_kit > 47) || (gas_kit > 350);

    // Global Hardware Buzzer Trigger
    bool shouldBuzz = false;
    if (alertActiveMain && alertsArmedMain) shouldBuzz = true;
    if (alertActiveBed && alertsArmedBed) shouldBuzz = true;
    if (alertActiveKit && alertsArmedKit) shouldBuzz = true;

    if (shouldBuzz) {
      digitalWrite(BUZZER, HIGH);
    } else {
      digitalWrite(BUZZER, LOW);
    }
  }
}