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

