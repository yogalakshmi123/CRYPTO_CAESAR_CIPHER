# CRYPTO_CAESAR_CIPHER
```bash
TO RUN THIS CODE in Visual Studio Code:
click ctrl+` it will open the terminal
install dependencies using this command: npm install
start the server-> use this command(npm start)

pakage.json
{
  "name": "caesar-realtime",
  "version": "1.0.0",
  "description": "Real-time Caesar Cipher chat demo with brute-force and frequency analysis (educational).",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "body-parser": "^1.20.2",
    "express": "^4.18.2",
    "socket.io": "^4.7.1"
  },
  "license": "MIT"
}
server.js
/**
 * server.js
 * Node.js + Express + Socket.IO backend for caesar-realtime
 *
 * - Serves static files from ./public
 * - Exposes two REST endpoints:
 *    POST /api/encrypt  -> { text, shift }  => { ciphertext }
 *    POST /api/decrypt  -> { text, shift }  => { plaintext }
 * - Relays ciphertext-only chat messages via Socket.IO.
 *
 * Note: This project is educational. Caesar cipher is not secure.
 */

const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const bodyParser = require('body-parser');
const path = require('path');

const app = express();
const server = http.createServer(app);
const io = new Server(server);

// Middleware
app.use(bodyParser.json());
app.use(express.static(path.join(__dirname, 'public')));

// Utility: Caesar cipher functions (works for A-Z, a-z; preserves non-letters)
function caesarShiftChar(chCode, base, shift) {
  // shift within 0..25 range
  return ((chCode - base + shift + 26) % 26) + base;
}
function caesarEncrypt(text, shift) {
  shift = ((Number(shift) % 26) + 26) % 26;
  let out = '';
  for (let ch of String(text)) {
    const code = ch.charCodeAt(0);
    if (code >= 65 && code <= 90) {
      out += String.fromCharCode(caesarShiftChar(code, 65, shift));
    } else if (code >= 97 && code <= 122) {
      out += String.fromCharCode(caesarShiftChar(code, 97, shift));
    } else {
      out += ch; // preserve spaces, punctuation, numbers, etc.
    }
  }
  return out;
}
function caesarDecrypt(text, shift) {
  return caesarEncrypt(text, -Number(shift));
}

// REST endpoints -----------------------------------------------------------
// Encrypt endpoint: accepts { text, shift } and returns { ciphertext }
app.post('/api/encrypt', (req, res) => {
  try {
    const { text = '', shift = 0 } = req.body;
    const ciphertext = caesarEncrypt(String(text), Number(shift) || 0);
    res.json({ ciphertext });
  } catch (err) {
    res.status(400).json({ error: 'Invalid request' });
  }
});

// Decrypt endpoint: accepts { text, shift } and returns { plaintext }
app.post('/api/decrypt', (req, res) => {
  try {
    const { text = '', shift = 0 } = req.body;
    const plaintext = caesarDecrypt(String(text), Number(shift) || 0);
    res.json({ plaintext });
  } catch (err) {
    res.status(400).json({ error: 'Invalid request' });
  }
});

// Socket.IO real-time chat -----------------------------------------------
// Important: server only relays ciphertext + username (no keys).
io.on('connection', (socket) => {
  console.log('user connected:', socket.id);

  // Expecting payload: { username, ciphertext }
  socket.on('chat-message', (data) => {
    // Basic validation (avoid broadcasting huge payloads)
    const username = String(data.username || 'Anonymous').slice(0, 60);
    const ciphertext = String(data.ciphertext || '').slice(0, 5000);

    // Broadcast ciphertext-only message to all connected clients
    io.emit('chat-message', {
      id: socket.id,
      username,
      ciphertext,
      ts: Date.now()
    });
  });

  socket.on('disconnect', () => {
    console.log('user disconnected:', socket.id);
  });
});

// Start server ------------------------------------------------------------
const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
  console.log(`caesar-realtime server running: http://localhost:${PORT}`);
});

 FRONTEND -index.html,style.css,script.js
index.html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>Caesar-Realtime — Caesar Cipher Chat Demo</title>
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <link rel="stylesheet" href="style.css" />
</head>
<body>
  <div class="app">
    <aside class="panel">
      <h1>Caesar Cipher Chat</h1>

      <label>Username
        <input id="username" placeholder="Your name" />
      </label>

      <label>Shift (key)
        <input id="key" type="number" value="3" min="-1000" max="1000" />
      </label>

      <div class="controls">
        <button id="btnConnect">Connect</button>
        <button id="btnDisconnect" disabled>Disconnect</button>
      </div>

      <hr />

      <h3>Encrypt / Decrypt</h3>
      <textarea id="plain" rows="3" placeholder="Type plaintext here..."></textarea>
      <div class="row">
        <button id="btnEncrypt">Encrypt →</button>
        <button id="btnDecrypt">← Decrypt</button>
      </div>
      <textarea id="cipher" rows="3" placeholder="Ciphertext will appear here"></textarea>

      <h3>Brute Force</h3>
      <button id="btnBrute">Try All Shifts</button>
      <div id="bruteResults" class="brute"></div>

      <h3>Frequency Analysis</h3>
      <button id="btnFreq">Show Frequency</button>
      <div id="freqChart" class="freq"></div>

      <hr />
      <small>Tip: Keep the same <strong>Shift</strong> on both clients to read messages easily. The server only relays ciphertext.</small>
    </aside>

    <main class="chat">
      <div class="chat-header">
        <h2>Live Chat</h2>
        <div id="status">Disconnected</div>
      </div>

      <div id="messages" class="messages"></div>

      <div class="composer">
        <input id="messageInput" placeholder="Type message..." />
        <button id="sendBtn" disabled>Send (Encrypted)</button>
      </div>

      <div class="info">
        <strong>How it works:</strong>
        Messages are encrypted in your browser with the Caesar shift you set and only ciphertext is sent to the server.
        Click "Decrypt locally" on a received message to attempt decryption using your local key.
      </div>
    </main>
  </div>

  <!-- Socket.IO client library (served by server) -->
  <script src="/socket.io/socket.io.js"></script>
  <script src="script.js"></script>
</body>
</html>


style.css
/* style.css - simple responsive layout */
* { box-sizing: border-box; }
body { margin: 0; font-family: Inter, system-ui, Arial, sans-serif; background: #f6f8fb; color: #222; }
.app { display: flex; min-height: 100vh; gap: 18px; padding: 18px; }

.panel {
  width: 340px;
  background: white;
  border-radius: 12px;
  padding: 18px;
  box-shadow: 0 6px 18px rgba(27,31,36,0.06);
  border: 1px solid #e6e9ef;
}
.panel h1 { margin: 0 0 12px 0; font-size: 20px; }

.panel label { display:block; margin-bottom:10px; font-size: 13px; color:#444; }
.panel input[type="text"], .panel input[type="number"], .panel textarea {
  width: 100%; padding: 8px 10px; margin-top:6px; border-radius:8px; border:1px solid #ddd;
  font-size: 14px; resize: vertical;
}

.controls { display:flex; gap:10px; margin-top:6px; }
.controls button { flex:1; padding:8px 10px; border-radius:8px; border:none; cursor:pointer; background:#0b63d7; color:white; }
.controls button[disabled] { opacity:0.5; cursor:not-allowed; }

.row { display:flex; gap:8px; margin-top:8px; }
.row button { flex:1; padding:8px; border-radius:8px; border:none; background:#2f80ed; color:white; cursor:pointer; }

.brute { max-height:140px; overflow:auto; font-family:monospace; background:#fbfcff; padding:8px; border-radius:8px; border:1px solid #eef2ff; margin-top:8px; font-size:13px; }
.freq { display:flex; gap:6px; margin-top:8px; align-items:flex-end; height:100px; padding:6px; background:#fff; border-radius:8px; border: 1px solid #eef2ff; }

.freq .bar { width: 12px; display:flex; flex-direction:column; align-items:center; justify-content:flex-end; font-size:11px; }
.freq .bar div { width: 100%; background:#91c5ff; border-radius:3px 3px 0 0; min-height:2px; }

.chat { flex:1; display:flex; flex-direction:column; gap:12px; }
.chat-header { display:flex; justify-content:space-between; align-items:center; }
.chat-header h2 { margin:0; }
#status { font-size:13px; color:#666; }

.messages { background: #ffffff; border:1px solid #e9eef8; border-radius:12px; padding:12px; flex:1; overflow:auto; }
.message { margin-bottom:10px; padding:8px 12px; border-radius:10px; background:#fbfbff; border:1px solid #eef2ff; }
.message .meta { font-size:12px; color:#666; margin-bottom:6px; }
.message .cipher { font-family:monospace; white-space:pre-wrap; word-break:break-word; }

.composer { display:flex; gap:8px; align-items:center; }
.composer input { flex:1; padding:10px; border-radius:10px; border:1px solid #ddd; }
.composer button { padding:10px 14px; border-radius:10px; border:none; background:#0b63d7; color:white; cursor:pointer; }

.info { font-size:13px; color:#555; margin-top:6px; }
@media (max-width:900px) {
  .app { flex-direction:column; padding:12px; }
  .panel { width:100%; }
}

script.js
/**
 * script.js - client-side logic for caesar-realtime
 *
 * Features implemented on client:
 * - Caesar encrypt/decrypt (supports negative shifts, preserves non-letters)
 * - Send ciphertext-only chat messages via Socket.IO
 * - Brute-force all 26 shifts with clickable results
 * - Frequency analysis bar chart
 * - Local decrypt button per received message (uses your local key input)
 *
 * Important: The server only relays ciphertext + username. Keys remain local to clients.
 */

const socket = io(); // Socket.IO client (connects automatically)

// DOM references
const usernameEl = document.getElementById('username');
const keyEl = document.getElementById('key');
const btnConnect = document.getElementById('btnConnect');
const btnDisconnect = document.getElementById('btnDisconnect');
const statusEl = document.getElementById('status');

const plainEl = document.getElementById('plain');
const cipherEl = document.getElementById('cipher');
const btnEncrypt = document.getElementById('btnEncrypt');
const btnDecrypt = document.getElementById('btnDecrypt');
const btnBrute = document.getElementById('btnBrute');
const bruteResults = document.getElementById('bruteResults');
const btnFreq = document.getElementById('btnFreq');
const freqChart = document.getElementById('freqChart');

const sendBtn = document.getElementById('sendBtn');
const messageInput = document.getElementById('messageInput');
const messagesEl = document.getElementById('messages');

let connected = false;

// ----------------- Caesar cipher helpers -----------------
function caesarShiftChar(chCode, base, shift) {
  return ((chCode - base + shift + 26) % 26) + base;
}
function caesarEncrypt(text, shift) {
  // Normalize shift to 0..25
  shift = ((Number(shift) % 26) + 26) % 26;
  let out = '';
  for (let ch of String(text)) {
    const code = ch.charCodeAt(0);
    if (code >= 65 && code <= 90) { // A-Z
      out += String.fromCharCode(caesarShiftChar(code, 65, shift));
    } else if (code >= 97 && code <= 122) { // a-z
      out += String.fromCharCode(caesarShiftChar(code, 97, shift));
    } else {
      out += ch;
    }
  }
  return out;
}
function caesarDecrypt(text, shift) {
  return caesarEncrypt(text, -Number(shift));
}

// ----------------- UI actions -----------------
btnConnect.onclick = () => {
  connected = true;
  statusEl.textContent = 'Connected (ready)';
  btnConnect.disabled = true;
  btnDisconnect.disabled = false;
  sendBtn.disabled = false;
};
btnDisconnect.onclick = () => {
  connected = false;
  statusEl.textContent = 'Disconnected';
  btnConnect.disabled = false;
  btnDisconnect.disabled = true;
  sendBtn.disabled = true;
};

// Encrypt / Decrypt buttons (local)
btnEncrypt.onclick = () => {
  const pt = plainEl.value || '';
  const k = Number(keyEl.value) || 0;
  cipherEl.value = caesarEncrypt(pt, k);
};
btnDecrypt.onclick = () => {
  const ct = cipherEl.value || '';
  const k = Number(keyEl.value) || 0;
  plainEl.value = caesarDecrypt(ct, k);
};

// Brute force: try all 26 shifts and show clickable results
btnBrute.onclick = () => {
  const ct = cipherEl.value || plainEl.value || '';
  if (!ct) {
    bruteResults.innerText = 'No text to brute force.';
    return;
  }
  bruteResults.innerHTML = '';
  for (let s = 0; s < 26; s++) {
    const attempt = caesarDecrypt(ct, s);
    const el = document.createElement('div');
    el.textContent = `Shift ${s}: ${attempt}`;
    el.style.cursor = 'pointer';
    el.title = 'Click to copy this attempt to plaintext box';
    el.onclick = () => { plainEl.value = attempt; };
    bruteResults.appendChild(el);
  }
};

// Frequency analysis: simple letter count normalized into bars
btnFreq.onclick = () => {
  const ct = cipherEl.value || plainEl.value || '';
  if (!ct) {
    freqChart.innerHTML = '<div style="padding:6px">No text for frequency analysis.</div>';
    return;
  }
  const counts = {};
  for (let ch of ct) {
    const up = ch.toUpperCase();
    if (up >= 'A' && up <= 'Z') {
      counts[up] = (counts[up] || 0) + 1;
    }
  }
  const letters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ'.split('');
  const max = Math.max(...letters.map(l => counts[l] || 0), 1);

  freqChart.innerHTML = '';
  for (let l of letters) {
    const barWrap = document.createElement('div');
    barWrap.className = 'bar';
    const bar = document.createElement('div');
    const height = Math.round(((counts[l] || 0) / max) * 90); // px
    bar.style.height = height + 'px';
    const label = document.createElement('div');
    label.style.fontSize = '10px';
    label.textContent = l;
    barWrap.appendChild(bar);
    barWrap.appendChild(label);
    freqChart.appendChild(barWrap);
  }
};

// ----------------- Chat: send encrypted message -----------------
sendBtn.onclick = () => {
  const text = messageInput.value.trim();
  if (!text || !connected) return;
  const k = Number(keyEl.value) || 0;
  const ciphertext = caesarEncrypt(text, k);

  // Payload: only ciphertext + username. Do NOT include the key.
  const payload = {
    username: usernameEl.value || 'Anonymous',
    ciphertext
  };

  socket.emit('chat-message', payload);

  // local echo (optional)
  messageInput.value = '';
};

// ----------------- Chat: receive messages -----------------
// Server broadcasts: { id, username, ciphertext, ts }
socket.on('chat-message', (data) => {
  const msgEl = document.createElement('div');
  msgEl.className = 'message';

  const meta = document.createElement('div');
  meta.className = 'meta';
  const time = new Date(data.ts || Date.now()).toLocaleTimeString();
  meta.textContent = `${data.username} • ${time}`;
  msgEl.appendChild(meta);

  const ct = document.createElement('div');
  ct.className = 'cipher';
  ct.textContent = data.ciphertext;
  msgEl.appendChild(ct);

  // Decrypt locally button - uses the key currently in the user's input field.
  const btn = document.createElement('button');
  btn.textContent = 'Decrypt locally';
  btn.style.marginTop = '6px';
  btn.onclick = () => {
    const myKey = Number(keyEl.value) || 0;
    const pt = caesarDecrypt(data.ciphertext, myKey);
    // Show in a small modal/alert for simplicity
    alert(`Decrypted with key ${myKey}:\n\n${pt}`);
  };
  msgEl.appendChild(btn);

  messagesEl.appendChild(msgEl);
  messagesEl.scrollTop = messagesEl.scrollHeight;
});

// Optional: show socket connection state
socket.on('connect', () => {
  statusEl.textContent = 'Socket.IO: connected';
});
socket.on('disconnect', () => {
  statusEl.textContent = 'Socket.IO: disconnected';
});

