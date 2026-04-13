
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Aero-Acoustics Lab</title>
    <style>
        :root { --bg: #0a0a12; --panel: #161625; --accent: #00f2ff; --text: #e0e0ff; --source: #ff007b; }
        body, html { margin: 0; padding: 0; overflow: hidden; background: var(--bg); color: var(--text); font-family: 'Segoe UI', sans-serif; }
        canvas { display: block; }
        #controls { position: absolute; top: 20px; left: 20px; width: 280px; background: var(--panel); padding: 20px; border-radius: 12px; border: 1px solid rgba(255,255,255,0.1); backdrop-filter: blur(10px); }
        h2 { margin: 0 0 15px 0; font-size: 1.2rem; color: var(--accent); text-transform: uppercase; }
        .control-group { margin-bottom: 15px; }
        label { display: block; margin-bottom: 5px; font-size: 0.85rem; opacity: 0.8; }
        input[type="range"] { width: 100%; accent-color: var(--accent); }
        .btn-row { display: flex; gap: 8px; margin-top: 10px; }
        button { flex: 1; padding: 10px; border: none; border-radius: 5px; background: var(--accent); color: var(--bg); font-weight: bold; cursor: pointer; }
        button:hover { opacity: 0.8; }
        #audioBtn { background: #ff007b; color: white; }
        .stats { margin-top: 15px; font-family: monospace; background: rgba(0,0,0,0.3); padding: 10px; border-radius: 4px; font-size: 0.8rem; }
    </style>
</head>
<body>

<div id="controls">
    <h2>Aero-Acoustics</h2>
    <div class="control-group">
        <label>Source Speed ($v_s$): <span id="vSrcVal">50</span></label>
        <input type="range" id="vSource" min="0" max="200" value="50">
    </div>
    <div class="control-group">
        <label>Wave Speed ($v_w$): <span id="vWaveVal">100</span></label>
        <input type="range" id="vWave" min="40" max="200" value="100">
    </div>
    <div class="btn-row">
        <button id="audioBtn">Enable Audio</button>
        <button id="pauseBtn">Pause</button>
    </div>
    <div class="stats">
        Mach Number: <span id="machNum">0.50</span><br>
        Status: <span id="status">Subsonic</span>
    </div>
</div>

<canvas id="simCanvas"></canvas>

<script>
    const canvas = document.getElementById('simCanvas');
    const ctx = canvas.getContext('2d');
    
    let waves = [];
    let sourceX = 100;
    let isPaused = false;
    let lastWaveTime = 0;

    // Audio State
    let audioCtx = null;
    let oscillator = null;
    let gainNode = null;

    const inputs = {
        vs: document.getElementById('vSource'),
        vw: document.getElementById('vWave'),
        audioBtn: document.getElementById('audioBtn'),
        pauseBtn: document.getElementById('pauseBtn')
    };

    function initAudio() {
        if (!audioCtx) {
            audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            oscillator = audioCtx.createOscillator();
            gainNode = audioCtx.createGain();
            oscillator.type = 'sawtooth';
            oscillator.connect(gainNode);
            gainNode.connect(audioCtx.destination);
            oscillator.start();
            gainNode.gain.value = 0.05;
            inputs.audioBtn.innerText = "Mute Audio";
        } else {
            audioCtx.state === 'running' ? audioCtx.suspend() : audioCtx.resume();
            inputs.audioBtn.innerText = audioCtx.state === 'running' ? "Mute Audio" : "Unmute";
        }
    }

    function resize() {
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;
    }

    window.addEventListener('resize', resize);
    resize();

    class Wave {
        constructor(x, y, v) {
            this.x = x; this.y = y; this.radius = 0; this.v = v; this.alpha = 0.6;
        }
        update(dt) { this.radius += this.v * dt; this.alpha -= 0.2 * dt; }
        draw() {
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
            ctx.strokeStyle = `rgba(0, 242, 255, ${this.alpha})`;
            ctx.stroke();
        }
    }

    function animate(timestamp) {
        if (!isPaused) {
            ctx.fillStyle = '#0a0a12';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            const vs = parseFloat(inputs.vs.value);
            const vw = parseFloat(inputs.vw.value);
            const mach = vs / vw;
            const centerY = canvas.height / 2;

            document.getElementById('vSrcVal').innerText = vs;
            document.getElementById('vWaveVal').innerText = vw;
            document.getElementById('machNum').innerText = mach.toFixed(2);
            
            // Audio Doppler logic
            if (audioCtx && audioCtx.state === 'running') {
                const observerX = canvas.width / 2;
                const dist = observerX - sourceX;
                const baseFreq = 220;
                // Simple Doppler formula approximation for audio
                let factor = vw / (vw + (dist > 0 ? -vs : vs));
                oscillator.frequency.setTargetAtTime(baseFreq * factor, audioCtx.currentTime, 0.1);
            }

            // Move Source
            sourceX += vs * 0.1;
            if (sourceX > canvas.width + 100) sourceX = -100;

            // Emit waves
            if (timestamp - lastWaveTime > 200) {
                waves.push(new Wave(sourceX, centerY, vw));
                lastWaveTime = timestamp;
            }

            // Draw Mach Cone (Shockwave)
            if (mach >= 1) {
                const angle = Math.asin(1 / mach);
                const len = 1000;
                ctx.beginPath();
                ctx.moveTo(sourceX, centerY);
                ctx.lineTo(sourceX - Math.cos(angle) * len, centerY - Math.sin(angle) * len);
                ctx.moveTo(sourceX, centerY);
                ctx.lineTo(sourceX - Math.cos(angle) * len, centerY + Math.sin(angle) * len);
                ctx.strokeStyle = 'rgba(255, 0, 123, 0.5)';
                ctx.lineWidth = 3;
                ctx.stroke();
                document.getElementById('status').innerText = "SUPERSONIC";
                document.getElementById('status').style.color = "#ff007b";
            } else {
                document.getElementById('status').innerText = "SUBSONIC";
                document.getElementById('status').style.color = "#00f2ff";
            }

            waves.forEach((w, i) => {
                w.update(0.1);
                w.draw();
                if (w.alpha <= 0) waves.splice(i, 1);
            });

            // Source point
            ctx.fillStyle = '#ff007b';
            ctx.beginPath();
            ctx.arc(sourceX, centerY, 6, 0, Math.PI * 2);
            ctx.fill();
        }
        requestAnimationFrame(animate);
    }

    inputs.audioBtn.onclick = initAudio;
    inputs.pauseBtn.onclick = () => isPaused = !isPaused;
    requestAnimationFrame(animate);
</script>
</body>
</html>

