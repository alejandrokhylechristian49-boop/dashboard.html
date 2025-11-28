<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>ESP32 Weather Dashboard</title>
<link href="https://fonts.googleapis.com/css2?family=Roboto:wght@400;700&display=swap" rel="stylesheet">
<style>
  :root {
    --bg-gradient: linear-gradient(135deg, #0f2027, #203a43, #2c5364);
    --card-bg: rgba(255,255,255,0.05);
    --card-shadow: rgba(0, 255, 224, 0.3);
  }

  body {
    margin: 0;
    font-family: 'Roboto', sans-serif;
    background: var(--bg-gradient);
    color: #fff;
    display: flex;
    flex-direction: column;
    align-items: center;
    min-height: 100vh;
    transition: background 1s;
  }

  h1 {
    margin-top: 20px;
    font-size: 2rem;
    text-align: center;
    color: #00ffe0;
    text-shadow: 0 0 10px #00ffe0;
    transition: color 1s, text-shadow 1s;
  }

  .dashboard {
    margin-top: 20px;
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
    gap: 20px;
    width: 90%;
    max-width: 900px;
    transition: all 1s;
  }

  .card {
    background: var(--card-bg);
    padding: 20px;
    border-radius: 12px;
    box-shadow: 0 0 20px var(--card-shadow);
    text-align: center;
    transition: transform 0.2s, box-shadow 0.2s, background 1s;
  }

  .card:hover {
    transform: translateY(-5px);
    box-shadow: 0 0 30px var(--card-shadow);
  }

  .card h2 {
    margin-bottom: 15px;
    font-size: 1.5rem;
    color: #00ffe0;
    transition: color 1s;
  }

  .sensor-value {
    font-size: 2rem;
    font-weight: 700;
    color: #fff;
    transition: color 1s;
  }

  .led-buttons button, .lcd-controls button, .temp-override button {
    margin: 5px;
    padding: 10px 20px;
    border-radius: 8px;
    border: none;
    cursor: pointer;
    font-size: 1rem;
    font-weight: 700;
    transition: 0.2s;
  }

  .led-buttons button { background: #00ffe0; color: #000; }
  .led-buttons button:hover { background: #00b8a9; }

  .lcd-controls input {
    padding: 10px;
    width: 200px;
    border-radius: 8px;
    border: none;
    margin-right: 5px;
  }

  #lcdDisplay {
    font-family: monospace;
    background: rgba(0,0,0,0.3);
    padding: 15px;
    border-radius: 8px;
    min-height: 40px;
    font-size: 1.2rem;
    margin-bottom: 10px;
    text-align: center;
    color: #0f0;
    transition: all 1s;
  }

  footer { margin-top: 30px; font-size: 0.9rem; opacity: 0.7; }

  .temp-override button { background: #ff9900; color: #000; }
  .temp-override button:hover { background: #ffcc66; }

</style>
</head>
<body>

<h1>ESP32 Weather Dashboard</h1>
<div id="status" style="margin-top:5px; font-weight:bold;">Checking connection...</div>


<div class="dashboard">

  <!-- Temperature Card -->
  <div class="card">
    <h2>Temperature</h2>
    <div class="sensor-value" id="temp">-- °C</div>
  </div>

  <!-- Pressure Card -->
  <div class="card">
    <h2>Pressure</h2>
    <div class="sensor-value" id="press">-- hPa</div>
  </div>

  <!-- LED Control Card -->
  <div class="card led-buttons">
    <h2>LED Control</h2>
    <button id="ledOn">Turn ON</button>
    <button id="ledOff">Turn OFF</button>
  </div>

  <!-- LCD Control Card -->
  <div class="card lcd-controls">
    <h2>LCD Display</h2>
    <div id="lcdDisplay">Waiting for messages...</div>
    <input type="text" id="lcdInput" placeholder="Type your message">
    <button id="updateLCD">Update LCD</button>
  </div>

  <!-- Temperature Override Card -->
  <div class="card temp-override">
    <h2>Override Temperature</h2>
    <button id="coldBtn">Cold (≤20°C)</button>
    <button id="hotBtn">Hot (≥37.5°C)</button>
    <button id="returnBtn">Return to Sensor</button>
  </div>

</div>

<footer>Connected via HiveMQ Cloud MQTT</footer>

<script src="https://unpkg.com/mqtt/dist/mqtt.min.js"></script>
<script>

  // Check if ESP32 is offline
setInterval(() => {
  if(Date.now() - lastHeartbeat > heartbeatTimeout){
    statusEl.innerText = "ESP32 Offline ❌";
    statusEl.style.color = "red";
  }
}, 1000);

  // ----- MQTT Config -----
  const MQTT_HOST = "wss://6baaab30108541b99203414955385496.s1.eu.hivemq.cloud:8884/mqtt";
  const MQTT_USER = "esp32user";
  const MQTT_PASS = "Esp32user";

  const TOPIC_TEMP = "sensors/weather/temperature";
  const TOPIC_PRESS = "sensors/weather/pressure";
  const TOPIC_LED  = "control/led";
  const TOPIC_LCD  = "control/lcd";

  const client = mqtt.connect(MQTT_HOST, {username: MQTT_USER, password: MQTT_PASS});

  // ----- DOM Elements -----
  const tempEl = document.getElementById("temp");
  const pressEl = document.getElementById("press");
  const lcdEl  = document.getElementById("lcdDisplay");
  const lcdInput = document.getElementById("lcdInput");
  const updateBtn = document.getElementById("updateLCD");

  const coldBtn = document.getElementById("coldBtn");
  const hotBtn = document.getElementById("hotBtn");
  const returnBtn = document.getElementById("returnBtn");

  // ----- LED Buttons -----
  document.getElementById("ledOn").onclick = () => client.publish(TOPIC_LED, "ON");
  document.getElementById("ledOff").onclick = () => client.publish(TOPIC_LED, "OFF");

  // ----- LCD Button -----
  updateBtn.onclick = () => {
    if(lcdInput.value.length > 0){
      client.publish(TOPIC_LCD, lcdInput.value);
      lcdInput.value = "";
    }
  }

  // ----- Temperature Override -----
  let override = null; // "cold" or "hot" or null

coldBtn.onclick = () => client.publish("control/temperature_override", "20.0");
hotBtn.onclick = () => client.publish("control/temperature_override", "40.0");
returnBtn.onclick = () => client.publish("control/temperature_override", "sensor")

  // ----- MQTT Events -----
  client.on('connect', () => {
    console.log("Connected to MQTT broker");
    client.subscribe([TOPIC_TEMP, TOPIC_PRESS, TOPIC_LCD]);
  });

  client.on('message', (topic, message) => {
    const msg = message.toString();
    if(topic === TOPIC_TEMP){
      let tempVal;
      try {
        const d = JSON.parse(msg);
        tempVal = d.value;
      } catch(e){ return; }

      if(override === null){
        tempEl.innerText = `${tempVal.toFixed(1)} °C`;
        updateDesign(tempVal);
      }
    }
    else if(topic === TOPIC_PRESS){
      try { const d = JSON.parse(msg); pressEl.innerText = `${d.value.toFixed(1)} hPa`; } catch(e) {}
    }
    else if(topic === TOPIC_LCD){
      lcdEl.innerText = msg;
    }
  });

  const TOPIC_HEARTBEAT = "sensors/weather/heartbeat";
let lastHeartbeat = Date.now();
const heartbeatTimeout = 7000; // 7 seconds without heartbeat → offline
const statusEl = document.getElementById("status");

// Subscribe to heartbeat topic
client.on('connect', () => {
  console.log("Connected to MQTT broker");
  client.subscribe([TOPIC_TEMP, TOPIC_PRESS, TOPIC_LCD, TOPIC_HEARTBEAT]);
});

// Inside client.on('message', ...)
client.on('message', (topic, message) => {
  const msg = message.toString();

  // --- Heartbeat ---
  if(topic === TOPIC_HEARTBEAT){
      lastHeartbeat = Date.now();
      statusEl.innerText = "ESP32 Online ✅";
      statusEl.style.color = "lime";
  }

  // --- Temperature ---
  else if(topic === TOPIC_TEMP){
    let tempVal;
    try {
      const d = JSON.parse(msg);
      tempVal = d.value;
    } catch(e){ return; }

    if(override === null){
      tempEl.innerText = `${tempVal.toFixed(1)} °C`;
      updateDesign(tempVal);
    }
  }
  // --- Pressure ---
  else if(topic === TOPIC_PRESS){
    try { const d = JSON.parse(msg); pressEl.innerText = `${d.value.toFixed(1)} hPa`; } catch(e) {}
  }
  // --- LCD ---
  else if(topic === TOPIC_LCD){
    lcdEl.innerText = msg;
  }
});

  // ----- Design Update Function -----
  function updateDesign(temp){
    if(temp <= 20){
      // Frozen design
      document.body.style.background = "linear-gradient(135deg, #74ebd5, #ACB6E5)";
      document.querySelectorAll(".card").forEach(c => {
        c.style.background = "rgba(173,216,230,0.2)";
        c.style.boxShadow = "0 0 20px #00ffff";
      });
      document.querySelectorAll(".card h2").forEach(h => h.style.color="#00ffff");
      lcdEl.style.color = "#0ff";
    } else if(temp >= 37.5){
      // Hot / desert design
      document.body.style.background = "linear-gradient(135deg, #FF512F, #F09819)";
      document.querySelectorAll(".card").forEach(c => {
        c.style.background = "rgba(255,165,0,0.2)";
        c.style.boxShadow = "0 0 20px #ff8c00";
      });
      document.querySelectorAll(".card h2").forEach(h => h.style.color="#ff4500");
      lcdEl.style.color = "#ff0";
    } else {
      // Normal design
      document.body.style.background = "linear-gradient(135deg, #0f2027, #203a43, #2c5364)";
      document.querySelectorAll(".card").forEach(c => {
        c.style.background = "rgba(255,255,255,0.05)";
        c.style.boxShadow = "0 0 20px rgba(0, 255, 224, 0.3)";
      });
      document.querySelectorAll(".card h2").forEach(h => h.style.color="#00ffe0");
      lcdEl.style.color = "#0f0";
    }
  }

</script>
</body>
</html>
