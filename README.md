#include <WiFi.h>
#include <WebServer.h>
#include <WebSocketsServer.h>
// WiFi AP Settings
const char* ssid = "AirDrums_Band";
const char* password = "12345678";
WebServer server(80);
WebSocketsServer webSocket = WebSocketsServer(81);
// Pin Definitions
#define TRIG 23
#define LED 2
int echoPins[4] = {22, 21, 19, 4};
String drums[4] = {"kick", "snare", "hihat", "tom"};
// --- WEB UI (HTML/CSS/JS) ---
const char webpage[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
<meta name="viewport" content="width=device-width, initial-scale=1">
<style>
 body { margin:0; background:#0f172a; color:white; font-family:sans-serif; text-align:center; overflow:hidden; }
 header { background: #1e293b; padding: 15px 0; border-bottom: 2px solid #334155; }
 h2 { margin: 0 0 10px 0; font-size: 22px; color: #38bdf8; }
 .tabs { display: flex; justify-content: center; gap: 5px; padding: 0 10px; }
 .tab-btn {
 flex: 1; padding: 10px 5px; font-size: 13px; border: none; background: #334155;
 color: #94a3b8; cursor: pointer; border-radius: 8px 8px 0 0; font-weight: bold; transition: 0.2s;
 }
 .tab-btn.active-tab { background: #38bdf8; color: #0f172a; }
 #overlay {
 position: fixed; top: 0; left: 0; width: 100%; height: 100%;
 background: rgba(15, 23, 42, 0.9); display: flex; align-items: center; justify-content: center; z-index: 10;
 }
 #startBtn {
 padding: 25px 50px; font-size: 24px; border: none; border-radius: 50px;
 background: #22c55e; color: white; cursor: pointer; font-weight: bold;
 box-shadow: 0 4px 20px rgba(34, 197, 94, 0.5);
 }
 .grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 15px; padding: 20px; height: calc(100vh - 180px); }
 .pad {
 border-radius: 20px; display: flex; flex-direction: column; align-items: center;
 justify-content: center; background: #1e293b; font-size: 20px; font-weight: bold;
 transition: 0.1s; border: 2px solid #334155;
 }
 .pad span { font-size: 12px; color: #94a3b8; margin-top: 5px; }
 .p1-hit { background: #ec4899 !important; border-color: white; transform: scale(0.95); }
 .p2-hit { background: #a855f7 !important; border-color: white; transform: scale(0.95); }
 .p3-hit { background: #3b82f6 !important; border-color: white; transform: scale(0.95); }
 .p4-hit { background: #eab308 !important; border-color: white; transform: scale(0.95); }
</style>
</head>
<body>
<div id="overlay"><button id="startBtn" onclick="initBand()"> START BAND</button></div>
<header>
 <h2> Air Band Simulator</h2>
 <div class="tabs">
 <button class="tab-btn active-tab" onclick="changeSet(0)"> Drums</button>
 <button class="tab-btn" onclick="changeSet(1)"> EDM</button>
 <button class="tab-btn" onclick="changeSet(2)"> Rock</button>
 <button class="tab-btn" onclick="changeSet(3)"> Synth</button>
 </div>
</header>
<div class="grid">
 <div id="pad0" class="pad"><div id="n0">Kick</div><span id="sub0">Acoustic Low</span></div>
 <div id="pad1" class="pad"><div id="n1">Snare</div><span id="sub1">Sharp Crack</span></div>
 <div id="pad2" class="pad"><div id="n2">HiHat</div><span id="sub2">Closed Tick</span></div>
 <div id="pad3" class="pad"><div id="n3">Tom</div><span id="sub3">Deep Floor</span></div>
</div>
<script>
let ws; let audioCtx; let currentSet = 0;
const sets = [
 { p: ["Kick", "Snare", "HiHat", "Floor Tom"] },
 { p: ["808 Sub", "Clap", "Trap Hat", "Laser Tom"] },
 { p: ["Riff A", "Riff B", "Bass Drop", "Power Chord"] },
 { p: ["Chords C", "Chords G", "Chords Am", "Chords F"] }
];
function changeSet(idx) {
 currentSet = idx;
 const btns = document.querySelectorAll('.tab-btn');
 btns.forEach((b, i) => i === idx ? b.classList.add('active-tab') : b.classList.remove('active-tab'));
 for(let i=0; i<4; i++) { document.getElementById(`n${i}`).innerText = sets[idx].p[i]; }
}
function initBand() {
 audioCtx = new (window.AudioContext || window.webkitAudioContext)();
 connectWS();
 document.getElementById("overlay").style.display = "none";
}
function connectWS() {
 ws = new WebSocket(`ws://${window.location.hostname}:81/`);
 ws.onmessage = (e) => {
 let msg = e.data.trim();
 let idx = -1;
 if (msg === "kick") idx = 0; else if (msg === "snare") idx = 1;
 else if (msg === "hihat") idx = 2; else if (msg === "tom") idx = 3;
 if (idx !== -1) { playBandSound(idx); triggerVisual(idx); }
 };
 ws.onclose = () => setTimeout(connectWS, 2000);
}
function triggerVisual(idx) {
 const el = document.getElementById(`pad${idx}`);
 el.classList.add(`p${idx+1}-hit`);
 setTimeout(() => el.classList.remove(`p${idx+1}-hit`), 120);
}
function playBandSound(idx) {
 if (!audioCtx) return;
 if (audioCtx.state === 'suspended') audioCtx.resume();
 if (currentSet === 0) playAcoustic(idx); else if (currentSet === 1) playEDM(idx);
 else if (currentSet === 2) playRock(idx); else if (currentSet === 3) playSynth(idx);
}
function playAcoustic(i) {
 if (i === 0) { // Kick
 let o = audioCtx.createOscillator(); let g = audioCtx.createGain();
 o.frequency.setValueAtTime(160, audioCtx.currentTime);
 o.frequency.exponentialRampToValueAtTime(45, audioCtx.currentTime + 0.15);
 g.gain.setValueAtTime(2.2, audioCtx.currentTime);
 g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 0.4);
 o.connect(g); g.connect(audioCtx.destination); o.start(); o.stop(audioCtx.currentTime + 0.4);
 } else if (i === 1) { // Snare
 let n = audioCtx.createBufferSource(); let b = audioCtx.createBuffer(1, audioCtx.sampleRate * 0.3, audioCtx.sampleRate);
 let d = b.getChannelData(0); for (let j = 0; j < d.length; j++) d[j] = Math.random() * 2 - 1;
 n.buffer = b; let g = audioCtx.createGain(); g.gain.setValueAtTime(1.8, audioCtx.currentTime);
 g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 0.25);
 n.connect(g); g.connect(audioCtx.destination); n.start();
 } else if (i === 2) { // Hat
 let o = audioCtx.createOscillator(); let g = audioCtx.createGain(); o.type = 'square';
 o.frequency.setValueAtTime(8000, audioCtx.currentTime); g.gain.setValueAtTime(0.7, audioCtx.currentTime);
 g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 0.08);
 o.connect(g); g.connect(audioCtx.destination); o.start(); o.stop(audioCtx.currentTime + 0.08);
 } else if (i === 3) { // Tom
 let o = audioCtx.createOscillator(); let g = audioCtx.createGain();
 o.frequency.setValueAtTime(180, audioCtx.currentTime);
 o.frequency.exponentialRampToValueAtTime(70, audioCtx.currentTime + 0.2);
 g.gain.setValueAtTime(1.9, audioCtx.currentTime);
 g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 0.5);
 o.connect(g); g.connect(audioCtx.destination); o.start(); o.stop(audioCtx.currentTime + 0.5);
 }
}
function playEDM(i) {
 if (i === 0) { // 808 Boom
 let o = audioCtx.createOscillator(); let g = audioCtx.createGain();
 o.frequency.setValueAtTime(120, audioCtx.currentTime);
 o.frequency.exponentialRampToValueAtTime(35, audioCtx.currentTime + 0.3);
 g.gain.setValueAtTime(2.5, audioCtx.currentTime);
 g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 1.0);
 o.connect(g); g.connect(audioCtx.destination); o.start(); o.stop(audioCtx.currentTime + 1.0);
 } else if (i === 3) { // Laser
 let o = audioCtx.createOscillator(); let g = audioCtx.createGain();
 o.frequency.setValueAtTime(1200, audioCtx.currentTime);
 o.frequency.exponentialRampToValueAtTime(50, audioCtx.currentTime + 0.2);
 g.gain.setValueAtTime(1.2, audioCtx.currentTime);
 g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 0.2);
 o.connect(g); g.connect(audioCtx.destination); o.start(); o.stop(audioCtx.currentTime + 0.2);
 } else { playAcoustic(i); }
}
function playRock(i) {
 let o = audioCtx.createOscillator(); let g = audioCtx.createGain(); o.type = 'sawtooth';
 let f = [146.8, 196.0, 73.4, 110.0]; o.frequency.setValueAtTime(f[i], audioCtx.currentTime);
 g.gain.setValueAtTime(1.5, audioCtx.currentTime); g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 0.6);
 o.connect(g); g.connect(audioCtx.destination); o.start(); o.stop(audioCtx.currentTime + 0.6);
}
function playSynth(i) {
 let o = audioCtx.createOscillator(); let g = audioCtx.createGain(); o.type = 'triangle';
 let c = [261.6, 392.0, 440.0, 349.2]; o.frequency.setValueAtTime(c[i], audioCtx.currentTime);
 g.gain.setValueAtTime(1.5, audioCtx.currentTime); g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 1.2);
 o.connect(g); g.connect(audioCtx.destination); o.start(); o.stop(audioCtx.currentTime + 1.2);
}
</script>
</body>
</html>
)rawliteral";
// --- ARDUINO CODE ---
void setup() {
 Serial.begin(115200);
 pinMode(TRIG, OUTPUT);
 pinMode(LED, OUTPUT);
 digitalWrite(LED, LOW);
 for (int i = 0; i < 4; i++) {
 pinMode(echoPins[i], INPUT);
 }
 WiFi.softAP(ssid, password);
 Serial.print("Access Point Started. Connect to: ");
 Serial.println(ssid);
 Serial.print("IP Address: ");
 Serial.println(WiFi.softAPIP());
 server.on("/", []() {
 server.send_P(200, "text/html", webpage);
 });
 server.begin();
 webSocket.begin();
 Serial.println("Air Band System Ready!");
}
long getDistance(int echoPin) {
 digitalWrite(TRIG, LOW);
 delayMicroseconds(2);
 digitalWrite(TRIG, HIGH);
 delayMicroseconds(10);
 digitalWrite(TRIG, LOW);
 long duration = pulseIn(echoPin, HIGH, 20000); // 20ms timeout
 if (duration == 0) return 999;
 return duration * 0.034 / 2;
}
void loop() {
 server.handleClient();
 webSocket.loop();
 for (int i = 0; i < 4; i++) {
 long d = getDistance(echoPins[i]);
 // Triggers sound if hand is within 2cm to 12cm
 if (d > 2 && d < 12) {
 digitalWrite(LED, HIGH);
 webSocket.broadcastTXT(drums[i]);

 // Debounce: prevent multiple triggers for one hand wave
 delay(120);
 digitalWrite(LED, LOW);
 }
 }
 delay(10); // Small loop stability delay
}
