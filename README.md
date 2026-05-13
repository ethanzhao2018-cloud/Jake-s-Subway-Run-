<!DOCTYPE html>
<html lang="en">
<head>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Jake: Tight Squeeze</title>
    <style>
        body { margin: 0; overflow: hidden; background: #050510; touch-action: none; font-family: 'Arial Black', sans-serif; }
        #game-ui { position: absolute; top: 15px; left: 15px; color: gold; font-size: 24px; pointer-events: none; z-index: 5; text-shadow: 2px 2px 0 #000; }
        
        .ctrl-btn {
            position: absolute; bottom: 35px; width: 95px; height: 95px;
            border-radius: 50%; color: white; display: flex; 
            align-items: center; justify-content: center; font-size: 16px;
            z-index: 20; transition: transform 0.1s;
            -webkit-user-select: none; border: 3px solid rgba(255,255,255,0.4);
            backdrop-filter: blur(5px);
        }
        #jump-zone { left: 30px; background: linear-gradient(135deg, #00f2ff, #0066ff); box-shadow: 0 0 20px rgba(0, 242, 255, 0.6); }
        #slide-zone { right: 30px; background: linear-gradient(135deg, #ff007f, #ff00ff); box-shadow: 0 0 20px rgba(255, 0, 127, 0.6); }

        #main-menu { position: absolute; inset: 0; background: rgba(0,0,0,0.92); display: flex; flex-direction: column; align-items: center; justify-content: center; color: white; z-index: 30; }
        .shop-container { background: #1a1a1a; border: 2px solid #333; padding: 15px; border-radius: 20px; width: 300px; max-height: 35vh; overflow-y: auto; margin-bottom: 15px; }
        .shop-item { background: #222; margin-bottom: 10px; padding: 12px; border-radius: 12px; display: flex; justify-content: space-between; align-items: center; border: 1px solid #444; font-size: 14px; }
        
        .custom-section { display: flex; flex-direction: column; align-items: center; gap: 10px; margin-bottom: 20px; background: #222; padding: 15px; border-radius: 15px; width: 280px; border: 1px solid #444; }
        #hoodie-color { width: 50px; height: 30px; border: none; cursor: pointer; vertical-align: middle; }
        .speed-selector { display: flex; gap: 8px; margin-top: 5px; }
        .speed-btn { background: #333; border: 2px solid #555; color: white; padding: 8px 12px; border-radius: 10px; cursor: pointer; font-size: 11px; }
        .speed-btn.active { border-color: #00f2ff; background: #004466; }

        .buy-btn { background: #2ecc71; border: none; color: white; padding: 8px 12px; border-radius: 6px; font-weight: bold; }
        .buy-btn.owned { background: #3498db; }
        #start-btn { background: #ff3e3e; border: none; padding: 12px 70px; font-size: 24px; border-radius: 50px; color: #fff; font-weight: bold; cursor: pointer; box-shadow: 0 0 20px rgba(255,62,62,0.4); }
    </style>
</head>
<body>

    <div id="game-ui">COINS: <span id="coin-ui">0</span></div>

    <div id="jump-zone" class="ctrl-btn">JUMP</div>
    <div id="slide-zone" class="ctrl-btn">SLIDE</div>

    <div id="main-menu">
        <h1 style="color:#00f2ff; margin-bottom: 10px; text-shadow: 0 0 15px #00f2ff;">SUBWAY RUN</h1>
        <p style="color:gold;">BANK: <span id="total-coins">0</span></p>
        
        <div class="shop-container" id="shop-list"></div>

        <div class="custom-section">
            <span style="font-size: 13px;">HOODIE: <input type="color" id="hoodie-color" oninput="changeColor(this.value)"></span>
            <div class="speed-selector">
                <button class="speed-btn active" onclick="setSpeed(8, this)">CHILL</button>
                <button class="speed-btn" onclick="setSpeed(14, this)">PRO</button>
                <button class="speed-btn" onclick="setSpeed(22, this)">GOD</button>
            </div>
        </div>

        <button id="start-btn" onclick="startGame()">RUN</button>
    </div>

    <canvas id="canvas"></canvas>

    <script>
        const canvas = document.getElementById("canvas");
        const ctx = canvas.getContext("2d");
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;

        let save = JSON.parse(localStorage.getItem('jakeV27')) || { total: 0, dj: false, mag: false, shield: false, board: false, color: '#ff4400' };
        function saveG() { localStorage.setItem('jakeV27', JSON.stringify(save)); }

        let gameActive = false, sessionCoins = 0, frame = 0, baseSpeed = 8, speed = 8, objects = [], bullets = [];
        let activePower = null;

        const jake = { x: 100, y: 0, w: 40, h: 70, dy: 0, jump: 26, gravity: 1.8, ground: canvas.height - 150, isGrounded: false, isCrouching: false, jumps: 0 };

        function changeColor(val) { save.color = val; saveG(); }
        function setSpeed(val, btn) { baseSpeed = val; document.querySelectorAll('.speed-btn').forEach(b => b.classList.remove('active')); btn.classList.add('active'); }

        function drawBackground() {
            let grd = ctx.createLinearGradient(0, 0, 0, canvas.height);
            grd.addColorStop(0, "#050510"); grd.addColorStop(1, "#151530");
            ctx.fillStyle = grd; ctx.fillRect(0, 0, canvas.width, canvas.height);
            ctx.fillStyle = "white";
            for(let i=0; i<15; i++) { ctx.globalAlpha = 0.2; ctx.fillRect((i*250 - frame*1.5)%canvas.width, 50 + (i*80)%300, 3, 3); }
            ctx.globalAlpha = 1;
        }

        function drawTrack() {
            ctx.fillStyle = "#111"; ctx.fillRect(0, jake.ground, canvas.width, 150);
            ctx.fillStyle = "#333"; 
            for(let i=0; i<15; i++) { ctx.fillRect((i*140 - (frame*speed))%(canvas.width+140)-140, jake.ground+20, 50, 12); }
            ctx.fillStyle = "#555"; ctx.fillRect(0, jake.ground+5, canvas.width, 5); 
        }

        function drawFace(x, y, type) {
            ctx.fillStyle = "white";
            ctx.fillRect(x + (type==='jake'?15:10), y + 5, 4, 4);
            ctx.fillRect(x + (type==='jake'?22:18), y + 5, 4, 4);
            ctx.fillStyle = "black";
            ctx.fillRect(x + (type==='jake'?17:11), y + 6, 2, 2);
            ctx.fillRect(x + (type==='jake'?24:19), y + 6, 2, 2);
            ctx.strokeStyle = "black"; ctx.lineWidth = 1;
            ctx.beginPath();
            if(type === 'jake') { ctx.arc(x+20, y+12, 4, 0, Math.PI); } 
            else { ctx.moveTo(x+12, y+15); ctx.lineTo(x+22, y+15); } 
            ctx.stroke();
        }

        function drawJake() {
            ctx.save(); ctx.translate(jake.x, jake.y);
            if (jake.isCrouching) {
                if(save.board) { ctx.fillStyle = "#00f2ff"; ctx.fillRect(-5, 28, 55, 6); ctx.shadowBlur=10; ctx.shadowColor="#00f2ff"; }
                ctx.fillStyle = save.color; ctx.fillRect(5, 5, 40, 25);
                ctx.shadowBlur=0;
                ctx.fillStyle = "#ffdbac"; ctx.fillRect(35, 5, 18, 18);
                drawFace(35, 5, 'jake');
            } else {
                ctx.fillStyle = save.color; ctx.fillRect(0, 5, 40, 40); 
                ctx.fillStyle = "#ffdbac"; ctx.fillRect(8, -20, 25, 25); 
                drawFace(8, -20, 'jake');
                ctx.fillStyle = "#222"; ctx.fillRect(4, -24, 34, 8); 
                ctx.fillStyle = "#2980b9"; ctx.fillRect(5, 40, 14, 30); ctx.fillRect(21, 40, 14, 30);
            }
            ctx.restore();
        }

        function spawn() {
            if(frame % 45 === 0) objects.push({x: canvas.width, y: jake.ground - 150, w: 30, h: 30, type:'coin'});
            if(frame % Math.floor(1100/speed) === 0) {
                const r = Math.random();
                if(r < 0.3) objects.push({x: canvas.width, y: jake.ground-70, w: 45, h: 70, type:'police', shotReady: true});
                else if(r < 0.5) objects.push({x: canvas.width, y: jake.ground-65, w: 120, h: 65, type:'train'}); 
                else if(r < 0.75) objects.push({x: canvas.width, y: jake.ground-100, w: 120, h: 45, type:'pipe'}); // EVEN LOWER PIPE
                else objects.push({x: canvas.width, y: jake.ground-75, w: 40, h: 75, type:'barrier'}); 
            }
        }

        function update() {
            if(!gameActive) return;
            frame++;
            document.getElementById('coin-ui').innerText = sessionCoins;
            jake.dy += jake.gravity; jake.y += jake.dy;
            jake.h = jake.isCrouching ? 35 : 70;
            if(jake.y >= jake.ground - jake.h) { jake.y = jake.ground - jake.h; jake.dy = 0; jake.isGrounded = true; jake.jumps = 0; }
            
            bullets.forEach((b, i) => { b.x -= speed + 12; if(jake.x < b.x + 10 && jake.x + jake.w > b.x && jake.y < b.y + 5 && jake.y + jake.h > b.y) endGame(); });

            objects.forEach((o, i) => {
                o.x -= speed;
                if(o.type === 'police' && o.shotReady && o.x < canvas.width - 300) { bullets.push({x: o.x, y: o.y + 25}); o.shotReady = false; }
                if(o.type === 'coin' && save.mag) { o.x += (jake.x - o.x) * 0.25; o.y += (jake.y - o.y) * 0.25; }
                
                // Collision Hitbox adjustment
                if(jake.x + 5 < o.x + o.w - 5 && jake.x + jake.w - 5 > o.x + 5 && jake.y + 5 < o.y + o.h - 5 && jake.y + jake.h - 5 > o.y + 5) {
                    if(o.type === 'coin') { sessionCoins++; objects.splice(i, 1); }
                    else { if(activePower === 'shield') { activePower = null; objects.splice(i, 1); } else endGame(); }
                }
                if(o.x < -400) objects.splice(i, 1);
            });
            speed += 0.001; spawn();
        }

        function draw() {
            ctx.clearRect(0,0,canvas.width, canvas.height);
            drawBackground(); drawTrack();
            bullets.forEach(b => { ctx.fillStyle = "yellow"; ctx.fillRect(b.x, b.y, 15, 6); });
            objects.forEach(o => {
                if(o.type === 'coin') { 
                    ctx.save(); ctx.translate(o.x + 15, o.y + 15);
                    let spin = Math.sin(frame * 0.15);
                    ctx.scale(spin, 1);
                    ctx.fillStyle = "gold"; ctx.beginPath(); ctx.arc(0, 0, 15, 0, Math.PI*2); ctx.fill();
                    ctx.fillStyle = "black"; ctx.font = "bold 16px Arial"; ctx.textAlign="center"; ctx.textBaseline="middle";
                    ctx.fillText("$", 0, 0); ctx.restore();
                } else if(o.type === 'police') { 
                    ctx.fillStyle = "#1e4dff"; ctx.fillRect(o.x, o.y, 45, 40); 
                    ctx.fillStyle="#111"; ctx.fillRect(o.x+5, o.y+40, 35, 30);
                    ctx.fillStyle="#ffdbac"; ctx.fillRect(o.x+10, o.y-15, 24, 24);
                    drawFace(o.x+10, o.y-15, 'police');
                } else if(o.type === 'train') { 
                    ctx.fillStyle = "#2c3e50"; ctx.fillRect(o.x, o.y, o.w, o.h);
                    ctx.fillStyle = "#f1c40f"; ctx.fillRect(o.x+10, o.y+5, 25, 10);
                } else if(o.type === 'barrier') { 
                    ctx.fillStyle = "#7f8c8d"; ctx.fillRect(o.x+15, o.y, 8, o.h);
                    for(let j=0; j<3; j++) { ctx.fillStyle = j%2===0?"red":"white"; ctx.fillRect(o.x, o.y + (j*25), 40, 12); }
                } else if(o.type === 'pipe') { 
                    // Main Pipe Body
                    ctx.fillStyle = "#333"; ctx.fillRect(o.x, o.y, o.w, o.h); 
                    // Danger Stripes
                    ctx.fillStyle = "#ffcc00"; 
                    for(let i=0; i<3; i++) ctx.fillRect(o.x + (i*45), o.y, 20, o.h);
                    ctx.fillStyle = "white"; ctx.font = "bold 14px Arial"; ctx.textAlign="center";
                    ctx.fillText("LOW!", o.x+o.w/2, o.y+28);
                }
            });
            drawJake();
            if(activePower === 'shield') { ctx.strokeStyle = "#00f2ff"; ctx.lineWidth=4; ctx.strokeRect(jake.x-8, jake.y-15, jake.w+16, jake.h+25); }
        }

        function buy(item, cost) { if (!save[item] && save.total >= cost) { save.total -= cost; save[item] = true; saveG(); updateUI(); } }
        function updateUI() {
            document.getElementById('total-coins').innerText = save.total;
            document.getElementById('hoodie-color').value = save.color;
            const items = [{id:'dj', n:'Double Jump', c:50}, {id:'mag', n:'Auto-Magnet', c:150}, {id:'shield', n:'Start Shield', c:250}, {id:'board', n:'Hoverboard', c:500}];
            document.getElementById('shop-list').innerHTML = items.map(it => `<div class="shop-item"><span>${it.n}</span><button class="buy-btn ${save[it.id]?'owned':''}" onclick="buy('${it.id}', ${it.c})">${save[it.id]?'OWNED':it.c}</button></div>`).join('');
        }

        function startGame() { document.getElementById('main-menu').style.display='none'; gameActive = true; sessionCoins = 0; objects = []; bullets = []; speed = baseSpeed; frame = 0; activePower = save.shield ? 'shield' : null; loop(); }
        function endGame() { gameActive = false; save.total += sessionCoins; saveG(); updateUI(); document.getElementById('main-menu').style.display='flex'; }
        function loop() { if(gameActive) { update(); draw(); requestAnimationFrame(loop); } }
        
        document.getElementById('jump-zone').addEventListener('touchstart', (e) => { e.preventDefault(); if(jake.jumps < (save.dj?2:1)) { jake.dy = -jake.jump; jake.jumps++; jake.isCrouching = false; } });
        document.getElementById('slide-zone').addEventListener('touchstart', (e) => { e.preventDefault(); jake.isCrouching = true; });
        document.getElementById('slide-zone').addEventListener('touchend', () => jake.isCrouching = false);
        updateUI();
    </script>
</body>
</html>
