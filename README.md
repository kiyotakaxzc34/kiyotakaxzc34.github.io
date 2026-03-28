<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Pixel Dungeon - 16:9 Aspect Ratio</title>
    <style>
        body {
            margin: 0;
            background-color: #0d0d12;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            width: 100vw;
            overflow: hidden;
            touch-action: none;
        }
        #canvas-container {
            position: relative;
            /* Обеспечиваем контейнер 16:9 */
            width: 95vw;
            aspect-ratio: 16 / 9;
            max-height: 90vh;
            display: flex;
            align-items: center;
            justify-content: center;
        }
        canvas {
            image-rendering: pixelated;
            border: 4px solid #1a1a24;
            background: #15151b;
            width: 100%;
            height: 100%;
            box-shadow: 0 10px 40px rgba(0,0,0,0.8);
        }
        #joystick-zone {
            position: fixed;
            bottom: 30px;
            left: 30px;
            width: 100px;
            height: 100px;
            background: rgba(255, 255, 255, 0.03);
            border-radius: 50%;
            border: 2px solid rgba(255, 255, 255, 0.1);
            display: flex;
            align-items: center;
            justify-content: center;
            z-index: 10;
        }
        #joystick-knob {
            width: 40px;
            height: 40px;
            background: rgba(255, 255, 255, 0.2);
            border-radius: 50%;
            position: absolute;
            pointer-events: none;
        }
    </style>
</head>
<body>

    <div id="canvas-container">
        <canvas id="gameCanvas"></canvas>
    </div>

    <div id="joystick-zone">
        <div id="joystick-knob"></div>
    </div>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

const TILE_SIZE = 16;
const SCALE = 3;
const SZ = TILE_SIZE * SCALE;

// Логический размер карты (весь уровень)
const MAP_COLS = 60; 
const MAP_ROWS = 45;

// Видимая область в формате 16:9 (примерно 16 на 9 тайлов)
const VIEW_WIDTH_TILES = 16; 
const VIEW_HEIGHT_TILES = 9;

// Устанавливаем внутреннее разрешение канваса
canvas.width = VIEW_WIDTH_TILES * SZ;
canvas.height = VIEW_HEIGHT_TILES * SZ;

let map = [];
let rooms = [];

function generateSoulKnightMap() {
    map = Array.from({length: MAP_ROWS}, () => new Array(MAP_COLS).fill(1));
    rooms = [];

    const minSize = 7;
    const maxSize = 12;
    const count = 10;

    for (let i = 0; i < count; i++) {
        let w = Math.floor(Math.random() * (maxSize - minSize)) + minSize;
        let h = Math.floor(Math.random() * (maxSize - minSize)) + minSize;
        let x = Math.floor(Math.random() * (MAP_COLS - w - 2)) + 1;
        let y = Math.floor(Math.random() * (MAP_ROWS - h - 2)) + 1;

        let newRoom = { x, y, w, h, cx: Math.floor(x + w / 2), cy: Math.floor(y + h / 2) };
        let intersects = rooms.some(r => !(newRoom.x + newRoom.w + 2 < r.x || newRoom.x > r.x + r.w + 2 || newRoom.y + newRoom.h + 2 < r.y || newRoom.y > r.h + r.y + 2));

        if (!intersects) {
            for (let ry = newRoom.y; ry < newRoom.y + newRoom.h; ry++) {
                for (let rx = newRoom.x; rx < newRoom.x + newRoom.w; rx++) {
                    map[ry][rx] = 0;
                }
            }
            if (rooms.length > 0) {
                let prev = rooms[rooms.length - 1];
                createCorridors(prev.cx, prev.cy, newRoom.cx, newRoom.cy);
            }
            rooms.push(newRoom);
        }
    }
}

function createCorridors(x1, y1, x2, y2) {
    let x = x1;
    let y = y1;
    while (x !== x2) {
        map[y][x] = 0;
        if(y+1 < MAP_ROWS) map[y+1][x] = 0; // ширина 2
        x += x > x2 ? -1 : 1;
    }
    while (y !== y2) {
        map[y][x] = 0;
        if(x+1 < MAP_COLS) map[y][x+1] = 0; // ширина 2
        y += y > y2 ? -1 : 1;
    }
}

const player = {
    x: 0, y: 0,
    vx: 0, vy: 0,
    speed: 0.16,
    flip: false,
    
    spawn() {
        if(rooms.length > 0) {
            this.x = rooms[0].cx;
            this.y = rooms[0].cy;
            this.vx = this.x;
            this.vy = this.y;
        }
    },
    
    update(ix, iy) {
        let nx = this.x + ix * this.speed;
        let ny = this.y + iy * this.speed;
        if (!this.checkWall(nx, this.y)) this.x = nx;
        if (!this.checkWall(this.x, ny)) this.y = ny;
        if (ix < 0) this.flip = true;
        if (ix > 0) this.flip = false;
        this.vx += (this.x - this.vx) * 0.2;
        this.vy += (this.y - this.vy) * 0.2;
    },

    checkWall(nx, ny) {
        const r = 0.35;
        const pts = [[nx-r, ny-r], [nx+r, ny-r], [nx-r, ny+r], [nx+r, ny+r]];
        return pts.some(p => map[Math.floor(p[1])][Math.floor(p[0])] === 1);
    },

    draw(t, camX, camY) {
        const bounce = Math.sin(t / 120) * 1.5;
        const dx = (this.vx - camX) * SZ;
        const dy = (this.vy - camY) * SZ;

        ctx.save();
        ctx.translate(dx, dy);
        if (this.flip) ctx.scale(-1, 1);

        // Shadow
        ctx.fillStyle = "rgba(0,0,0,0.3)";
        ctx.beginPath(); ctx.ellipse(0, 5*SCALE, 6*SCALE, 3*SCALE, 0, 0, 7); ctx.fill();

        // Knight Body
        ctx.fillStyle = "#3d3d4e";
        ctx.fillRect(-5*SCALE, (-4+bounce)*SCALE, 10*SCALE, 9*SCALE);
        // Helmet
        ctx.fillStyle = "#bdc3c7";
        ctx.fillRect(-6*SCALE, (-11+bounce)*SCALE, 12*SCALE, 8*SCALE);
        // Eyes
        ctx.fillStyle = "#3498db";
        ctx.fillRect(this.flip ? -5*SCALE : 1*SCALE, (-8+bounce)*SCALE, 4*SCALE, 2*SCALE);

        ctx.restore();
    }
};

const camera = {
    x: 0, y: 0,
    update(tx, ty) {
        let targetCamX = tx - VIEW_WIDTH_TILES / 2;
        let targetCamY = ty - VIEW_HEIGHT_TILES / 2;
        this.x = Math.max(0, Math.min(targetCamX, MAP_COLS - VIEW_WIDTH_TILES));
        this.y = Math.max(0, Math.min(targetCamY, MAP_ROWS - VIEW_HEIGHT_TILES));
    }
};

function drawTile(x, y, type) {
    if (type === 1) {
        // Wall
        ctx.fillStyle = "#2c2c36";
        ctx.fillRect(x, y, SZ, SZ);
        ctx.fillStyle = "#40404f"; ctx.fillRect(x, y, SZ, 4);
        ctx.fillStyle = "#1a1a24"; ctx.fillRect(x, y + SZ - 4, SZ, 4);
    } else {
        // Floor
        ctx.fillStyle = "#1a1a24";
        ctx.fillRect(x, y, SZ, SZ);
        ctx.strokeStyle = "#22222e";
        ctx.strokeRect(x, y, SZ, SZ);
    }
}

// Joystick logic
const zone = document.getElementById('joystick-zone');
const knob = document.getElementById('joystick-knob');
let joy = { x: 0, y: 0, active: false };

function moveJoy(e) {
    if (!joy.active) return;
    const r = zone.getBoundingClientRect();
    const cx = r.left + r.width/2, cy = r.top + r.height/2;
    const ex = e.touches ? e.touches[0].clientX : e.clientX;
    const ey = e.touches ? e.touches[0].clientY : e.clientY;
    let dx = ex - cx, dy = ey - cy;
    const d = Math.sqrt(dx*dx + dy*dy);
    const m = 35;
    if (d > m) { dx *= m/d; dy *= m/d; }
    knob.style.transform = `translate(${dx}px, ${dy}px)`;
    joy.x = dx / m; joy.y = dy / m;
}

zone.addEventListener('pointerdown', e => { joy.active = true; moveJoy(e); });
window.addEventListener('pointermove', moveJoy);
window.addEventListener('pointerup', () => { joy.active = false; joy.x = 0; joy.y = 0; knob.style.transform = 'translate(0,0)'; });

generateSoulKnightMap();
player.spawn();

function loop(t) {
    camera.update(player.vx, player.vy);
    ctx.clearRect(0,0, canvas.width, canvas.height);

    const sC = Math.floor(camera.x), eC = Math.min(MAP_COLS, sC + VIEW_WIDTH_TILES + 1);
    const sR = Math.floor(camera.y), eR = Math.min(MAP_ROWS, sR + VIEW_HEIGHT_TILES + 1);

    for(let r=sR; r<eR; r++) {
        for(let c=sC; c<eC; c++) {
            drawTile((c - camera.x) * SZ, (r - camera.y) * SZ, map[r][c]);
        }
    }

    player.update(joy.active ? joy.x : 0, joy.active ? joy.y : 0);
    player.draw(t, camera.x, camera.y);
    requestAnimationFrame(loop);
}
requestAnimationFrame(loop);
</script>
</body>
</html>

            const hashBuffer = await crypto.subtle.digest('SHA-256', utf8);
            return Array.from(new Uint8Array(hashBuffer)).map(b => b.toString(16).padStart(2, '0')).join('');
        }

        window.allAchievements = [
            { id: 'start', t: 15, n: 'НОВИЧОК' },
            { id: 'pro', t: 50, n: 'ПРОФИ' },
            { id: 'legend', t: 100, n: 'ЛЕГЕНДА' }
        ];

        window.handleAuth = async () => {
            const nameInput = document.getElementById('nick');
            const name = nameInput.value.trim().toUpperCase();
            const pass = document.getElementById('pass').value.trim();
            const err = document.getElementById('auth-err');
            
            const englishRegex = /^[A-Z0-9]+$/;
            if (!englishRegex.test(name)) { err.innerText = "NICK: ENG ONLY!"; return; }
            if (name.length < 3) { err.innerText = "NICK MIN 3"; return; }
            if (pass.length < 8) { err.innerText = "PASS MIN 8"; return; }

            err.innerText = "LOADING...";
            const hp = await hash(pass);
            try {
                const ref = doc(db, 'artifacts', appId, 'public', 'data', 'users', name);
                const snap = await getDoc(ref);
                if (snap.exists()) {
                    if (snap.data().pass === hp) {
                        localStorage.setItem('f_nick', name);
                        window.initPlayerSession(name);
                    } else { err.innerText = "WRONG PASSWORD!"; }
                } else {
                    await setDoc(ref, { score: 0, ach: [], pass: hp });
                    localStorage.setItem('f_nick', name);
                    window.initPlayerSession(name);
                }
            } catch(e) { err.innerText = "ERROR"; }
        };

        window.initPlayerSession = async (n) => {
            try {
                const snap = await getDoc(doc(db, 'artifacts', appId, 'public', 'data', 'users', n));
                if (snap.exists()) {
                    const d = snap.data();
                    localStorage.setItem('f_ach', JSON.stringify(d.ach || []));
                    localStorage.setItem('f_best', d.score || 0);
                    document.getElementById('p-tag').innerText = `PLAYER: ${n}`;
                    document.getElementById('m-best').innerText = `BEST: ${d.score}`;
                }
            } finally {
                document.getElementById('auth-menu').style.display = 'none';
                document.getElementById('main-menu').style.display = 'block';
            }
        };

        window.showLeaderboard = async () => {
            document.getElementById('main-menu').style.display = 'none';
            document.getElementById('lb-menu').style.display = 'block';
            const list = document.getElementById('lb-list');
            list.innerHTML = "<div style='font-size:7px'>LOADING...</div>";
            try {
                const q = query(collection(db, 'artifacts', appId, 'public', 'data', 'users'), orderBy("score", "desc"), limit(5));
                const snap = await getDocs(q);
                list.innerHTML = "";
                let i = 1;
                snap.forEach((d) => {
                    list.innerHTML += `<div class="row"><span>${i}. ${d.id}</span><span>${d.data().score}</span></div>`;
                    i++;
                });
            } catch(e) { list.innerHTML = "ERROR"; }
        };

        window.syncScoreSilent = (s) => {
            const n = localStorage.getItem('f_nick');
            if (n) updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'users', n), { score: s }).catch(()=>{});
        };

        signInAnonymously(auth).then(() => {
            const n = localStorage.getItem('f_nick');
            if (n) {
                window.initPlayerSession(n);
            } else {
                document.getElementById('auth-menu').style.display = 'block';
            }
        });
    </script>

    <style>
        * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
        body { margin: 0; background: #000; font-family: 'Press Start 2P', cursive; overflow: hidden; touch-action: none; height: 100vh; display: flex; justify-content: center; align-items: center; }
        #game { position: relative; width: 320px; height: 480px; background: #70c5ce; overflow: hidden; }
        canvas { display: block; width: 100%; height: 100%; image-rendering: pixelated; }
        .overlay { position: absolute; inset: 0; display: flex; flex-direction: column; justify-content: center; align-items: center; pointer-events: none; z-index: 10; padding: 20px; }
        .card { background: #fff; border: 4px solid #000; padding: 20px; text-align: center; pointer-events: auto; display: none; width: 280px; box-shadow: 6px 6px 0 rgba(0,0,0,0.3); }
        #score-ui { position: absolute; top: 30px; width: 100%; text-align: center; font-size: 24px; color: #fff; text-shadow: 3px 3px 0 #000; z-index: 5; display: none; }
        input { border: 3px solid #000; padding: 10px; font-family: 'Press Start 2P'; font-size: 8px; width: 100%; margin-bottom: 10px; outline: none; }
        button { background: #e67e22; border: 3px solid #000; padding: 12px; color: #fff; cursor: pointer; font-family: 'Press Start 2P'; font-size: 8px; width: 100%; margin: 5px 0; }
        .row { display: flex; justify-content: space-between; font-size: 7px; padding: 8px 0; border-bottom: 2px solid #eee; }
    </style>
</head>
<body>

<div id="game">
    <canvas id="cvs" width="160" height="240"></canvas>
    <div id="score-ui">0</div>
    
    <div class="overlay">
        <div id="auth-menu" class="card">
            <h2 style="font-size:10px;">LOGIN</h2>
            <input type="text" id="nick" placeholder="NICKNAME" maxlength="10">
            <input type="password" id="pass" placeholder="PASSWORD" maxlength="16">
            <p id="auth-err" style="color:red; font-size:6px; min-height:10px;"></p>
            <button onclick="handleAuth()">START</button>
        </div>

        <div id="main-menu" class="card">
            <h1 style="font-size:12px; color:#f39c12;">FLAPPY PIXEL</h1>
            <p id="p-tag" style="font-size:7px; color:#27ae60;"></p>
            <p id="m-best" style="font-size:7px; color:#7f8c8d; margin-bottom:10px;">BEST: 0</p>
            <button onclick="startGame()">PLAY</button>
            <button onclick="showLeaderboard()" style="background:#2980b9;">TOP 5</button>
            <button onclick="localStorage.clear(); location.reload();" style="background:#95a5a6; font-size:6px;">EXIT</button>
        </div>

        <div id="lb-menu" class="card">
            <h2 style="font-size:10px;">TOP 5</h2>
            <div id="lb-list" style="margin-bottom:10px;"></div>
            <button onclick="location.reload()">BACK</button>
        </div>
    </div>
</div>

<script>
    const cvs = document.getElementById('cvs');
    const ctx = cvs.getContext('2d');
    const scoreUI = document.getElementById('score-ui');

    let bird, pipes, score, active = false;
    let spd = 1.3, gap = 80;

    class Bird {
        constructor() { this.x = 40; this.y = 100; this.v = 0; }
        update() { 
            this.v += 0.23; this.y += this.v; 
            if (this.y > 220 || this.y < -20) die(); 
        }
        draw() {
            ctx.save();
            ctx.translate(this.x, this.y);
            ctx.rotate(Math.min(Math.PI/2, Math.max(-0.4, this.v * 0.1)));
            ctx.fillStyle = "#f1c40f"; ctx.strokeStyle = "#000"; ctx.lineWidth = 1;
            ctx.fillRect(-6, -4, 12, 8); ctx.strokeRect(-6, -4, 12, 8);
            ctx.fillStyle = "white"; ctx.fillRect(2, -3, 2, 2);
            ctx.restore();
        }
        jump() { this.v = -3.5; }
    }

    class Pipe {
        constructor(x) {
            this.x = x; this.h = Math.floor(Math.random() * 80) + 40; this.p = false;
        }
        draw() {
            ctx.fillStyle = "#2ecc71"; ctx.strokeStyle = "#000";
            ctx.fillRect(this.x, 0, 24, this.h); ctx.strokeRect(this.x, 0, 24, this.h);
            ctx.fillRect(this.x, this.h + gap, 24, 240); ctx.strokeRect(this.x, this.h + gap, 24, 240);
        }
    }

    function startGame() {
        active = true; score = 0; pipes = [new Pipe(200)]; bird = new Bird();
        document.getElementById('main-menu').style.display = 'none';
        scoreUI.style.display = 'block'; scoreUI.innerText = '0';
        loop();
    }

    function die() {
        active = false;
        const best = parseInt(localStorage.getItem('f_best') || "0");
        if (score > best) {
            localStorage.setItem('f_best', score);
            window.syncScoreSilent(score);
        }
        setTimeout(() => location.reload(), 200);
    }

    function loop() {
        if (!active) return;
        ctx.clearRect(0,0,160,240);
        bird.update();
        if (pipes[pipes.length-1].x < 80) pipes.push(new Pipe(160));
        for (let i = pipes.length-1; i>=0; i--) {
            pipes[i].x -= spd; pipes[i].draw();
            if (bird.x+4 > pipes[i].x && bird.x-4 < pipes[i].x+24) {
                if (bird.y-3 < pipes[i].h || bird.y+3 > pipes[i].h + gap) die();
            }
            if (!pipes[i].p && bird.x > pipes[i].x+24) {
                score++; pipes[i].p = true; scoreUI.innerText = score;
            }
            if (pipes[i].x < -30) pipes.splice(i,1);
        }
        bird.draw();
        ctx.fillStyle = "#ded895"; ctx.fillRect(0, 226, 160, 14);
        requestAnimationFrame(loop);
    }

    // Фоновая отрисовка пока меню открыто
    function preview() {
        if (!active) {
            ctx.clearRect(0,0,160,240);
            ctx.fillStyle = "#ded895"; ctx.fillRect(0, 226, 160, 14);
            requestAnimationFrame(preview);
        }
    }
    preview();

    const j = (e) => { if(active) bird.jump(); e?.preventDefault(); };
    cvs.addEventListener('touchstart', j); cvs.addEventListener('mousedown', j);
    window.addEventListener('keydown', e => { if(e.code==='Space') j(); });
</script>
</body>
</html>

