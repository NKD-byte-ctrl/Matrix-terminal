!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<meta name="theme-color" content="#000000">
<title>Matrix Terminal</title>
<style>
  * { box-sizing: border-box; margin: 0; padding: 0; }
  html, body { height: 100%; background: #000; overflow: hidden; }
  body { font-family: "Courier New", monospace; color: #00ff41; }

  /* Falling-character canvas sits behind everything */
  #rain { position: fixed; inset: 0; z-index: 0; }

  /* Terminal window */
  #term {
    position: fixed; z-index: 1;
    top: 4vh; bottom: 4vh; left: 50%; transform: translateX(-50%);
    width: min(94vw, 760px);
    display: flex; flex-direction: column;
    background: rgba(0, 10, 0, 0.82);
    border: 1px solid #00ff41;
    border-radius: 8px;
    box-shadow: 0 0 20px rgba(0,255,65,.5), inset 0 0 25px rgba(0,255,65,.08);
    backdrop-filter: blur(2px);
  }
  #bar {
    padding: 8px 12px; font-size: 13px;
    border-bottom: 1px solid #00ff41;
    display: flex; gap: 6px; align-items: center;
  }
  .dot { width: 10px; height: 10px; border-radius: 50%; background: #00ff41; opacity: .6; }
  #bar span { margin-left: 8px; opacity: .8; }

  #out {
    flex: 1; overflow-y: auto; padding: 12px;
    font-size: 15px; line-height: 1.45;
    white-space: pre-wrap; word-break: break-word;
    text-shadow: 0 0 6px rgba(0,255,65,.7);
    -webkit-overflow-scrolling: touch;
  }
  #out::-webkit-scrollbar { width: 6px; }
  #out::-webkit-scrollbar-thumb { background: #00ff41; }
  .err { color: #ff4d4d; text-shadow: 0 0 6px rgba(255,77,77,.7); }
  .dim { opacity: .65; }

  /* Input line with blinking block cursor */
  #line {
    display: flex; align-items: center; gap: 8px;
    padding: 10px 12px; border-top: 1px solid rgba(0,255,65,.4);
    font-size: 16px; /* 16px stops Android/iOS zooming on focus */
  }
  #prompt { white-space: nowrap; text-shadow: 0 0 6px #00ff41; }
  #wrap { position: relative; flex: 1; min-width: 0; }
  #shown { white-space: pre; overflow: hidden; text-shadow: 0 0 6px #00ff41; }
  #cursor {
    display: inline-block; width: 0.6em; height: 1.1em;
    background: #00ff41; vertical-align: text-bottom;
    animation: blink 1s steps(1) infinite;
    box-shadow: 0 0 8px #00ff41;
  }
  @keyframes blink { 50% { opacity: 0; } }
  /* Real input is invisible but receives the keyboard */
  #cmd {
    position: absolute; inset: 0; width: 100%; height: 100%;
    opacity: 0; border: 0; background: transparent; color: transparent;
    font: inherit; font-size: 16px;
  }
</style>
</head>
<body>

<canvas id="rain"></canvas>

<div id="term">
  <div id="bar"><i class="dot"></i><i class="dot"></i><i class="dot"></i><span>neo@matrix: ~</span></div>
  <div id="out" aria-live="polite"></div>
  <div id="line">
    <span id="prompt">neo@matrix:~$</span>
    <div id="wrap">
      <div id="shown"><span id="text"></span><span id="cursor"></span></div>
      <input id="cmd" type="text" autocomplete="off" autocapitalize="off"
             autocorrect="off" spellcheck="false" enterkeyhint="send" aria-label="Terminal input">
    </div>
  </div>
</div>

<script>
/* ---------- 1. MATRIX RAIN ---------- */
const canvas = document.getElementById("rain");
const ctx = canvas.getContext("2d");
const CHARS = "アイウエオカキクケコサシスセソタチツテトナニヌネノ0123456789ABCDEFZ$#@%&*<>";
const SIZE = 16;
let drops = [];
let color = "#00ff41";   // current rain color
let speed = 50;          // milliseconds per frame (lower = faster)
let timer;

function resize() {
  canvas.width = window.innerWidth;
  canvas.height = window.innerHeight;
  const cols = Math.floor(canvas.width / SIZE);
  drops = Array.from({ length: cols }, () => Math.random() * -50);
}

function drawRain() {
  // Translucent black layer makes old characters fade into trails
  ctx.fillStyle = "rgba(0, 0, 0, 0.08)";
  ctx.fillRect(0, 0, canvas.width, canvas.height);
  ctx.fillStyle = color;
  ctx.font = SIZE + "px monospace";
  for (let i = 0; i < drops.length; i++) {
    const ch = CHARS[Math.floor(Math.random() * CHARS.length)];
    ctx.fillText(ch, i * SIZE, drops[i] * SIZE);
    if (drops[i] * SIZE > canvas.height && Math.random() > 0.975) drops[i] = 0;
    drops[i]++;
  }
}

function startRain() {
  clearInterval(timer);
  timer = setInterval(drawRain, speed);
}

window.addEventListener("resize", resize);
resize();
startRain();

/* ---------- 2. TERMINAL ---------- */
const out = document.getElementById("out");
const cmd = document.getElementById("cmd");
const text = document.getElementById("text");
let busy = false;          // true while text is being "typed"
const history = [];
let hIndex = 0;

function scrollDown() { out.scrollTop = out.scrollHeight; }

// Print a line instantly
function print(msg, cls) {
  const div = document.createElement("div");
  if (cls) div.className = cls;
  div.textContent = msg;
  out.appendChild(div);
  scrollDown();
}

// Print a line with a typing effect
function typeLine(msg, cls) {
  return new Promise(resolve => {
    const div = document.createElement("div");
    if (cls) div.className = cls;
    out.appendChild(div);
    let i = 0;
    const t = setInterval(() => {
      div.textContent += msg[i++];
      scrollDown();
      if (i >= msg.length) { clearInterval(t); resolve(); }
    }, 18);
  });
}

const sleep = ms => new Promise(r => setTimeout(r, ms));

/* ---------- 3. COMMANDS ---------- */
const modes = [
  { name: "green", color: "#00ff41", speed: 50 },
  { name: "cyan",  color: "#00e5ff", speed: 35 },
  { name: "red",   color: "#ff2a2a", speed: 25 }
];
let modeIndex = 0;

const commands = {
  help() {
    print("Available commands:");
    print("  help     - show this list");
    print("  about    - about this terminal");
    print("  date     - show current date and time");
    print("  clear    - clear the screen");
    print("  matrix   - change the rain (green > cyan > red)");
    print("  whoami   - who are you?");
    print("  hack     - run a fake hacking sequence");
    print("  echo hi  - print any text back");
  },
  about() {
    print("MATRIX TERMINAL v1.0");
    print("Built with plain HTML, CSS and JavaScript.");
    print("No libraries. No servers. Just one file.");
    print("Wake up, Neo...", "dim");
  },
  date() { print(new Date().toString()); },
  clear() { out.innerHTML = ""; },
  matrix() {
    modeIndex = (modeIndex + 1) % modes.length;
    color = modes[modeIndex].color;
    speed = modes[modeIndex].speed;
    startRain();
    print("Rain mode: " + modes[modeIndex].name + " | speed: " + speed + "ms");
  },
  whoami() { print("neo - the one."); },
  echo(args) { print(args.join(" ")); },
  async hack() {
    const steps = [
      "Connecting to mainframe...",
      "Bypassing firewall [####------] 40%",
      "Cracking encryption [#######---] 70%",
      "Injecting payload [##########] 100%",
      "ACCESS GRANTED"
    ];
    for (const s of steps) { await typeLine(s); await sleep(350); }
    print("(just kidding - this is all fake!)", "dim");
  }
};

async function run(raw) {
  const input = raw.trim();
  print("neo@matrix:~$ " + input, "dim");
  if (!input) return;
  history.push(input);
  hIndex = history.length;
  const [name, ...args] = input.split(/\s+/);
  const fn = commands[name.toLowerCase()];
  if (fn) {
    busy = true;
    await fn(args);
    busy = false;
  } else {
    print("command not found: " + name + " (type 'help')", "err");
  }
  scrollDown();
}

/* ---------- 4. INPUT HANDLING ---------- */
cmd.addEventListener("input", () => { text.textContent = cmd.value; });

cmd.addEventListener("keydown", e => {
  if (e.key === "Enter") {
    if (busy) return;
    const v = cmd.value;
    cmd.value = ""; text.textContent = "";
    run(v);
  } else if (e.key === "ArrowUp" && history.length) {
    e.preventDefault();
    hIndex = Math.max(0, hIndex - 1);
    cmd.value = history[hIndex]; text.textContent = cmd.value;
  } else if (e.key === "ArrowDown") {
    e.preventDefault();
    hIndex = Math.min(history.length, hIndex + 1);
    cmd.value = history[hIndex] || ""; text.textContent = cmd.value;
  }
});

// Tap anywhere on the terminal to bring up the keyboard
document.getElementById("term").addEventListener("click", () => cmd.focus());

/* ---------- 5. BOOT SEQUENCE ---------- */
(async function boot() {
  busy = true;
  await typeLine("Initializing Matrix Terminal...");
  await sleep(300);
  await typeLine("Connection established.");
  await sleep(300);
  await typeLine("Type 'help' to see commands. Tap the screen to type.", "dim");
  busy = false;
  cmd.focus();
})();
</script>
</body>
</html># Matrix-terminal
