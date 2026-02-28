<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Traffic Hero: Professional Edition</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body { margin: 0; background: #1a1a1a; overflow: hidden; font-family: 'Segoe UI', sans-serif; }
        #gameCanvas { background: #333; display: block; margin: 0 auto; border-left: 8px dashed #555; border-right: 8px dashed #555; }
        .light { width: 24px; height: 24px; border-radius: 50%; border: 2px solid #222; margin: 2px; transition: 0.3s; }
    </style>
</head>
<body class="flex items-center justify-center min-h-screen">

    <div class="relative shadow-2xl border-8 border-slate-800 rounded-lg overflow-hidden">
        <canvas id="gameCanvas" width="400" height="600"></canvas>
        
        <div class="absolute top-4 left-4 right-4 flex justify-between items-start pointer-events-none">
            <div class="bg-black/70 p-3 rounded-lg border border-white/20 text-white">
                <p class="text-[10px] uppercase tracking-widest opacity-70">Safety Score</p>
                <span id="score" class="text-2xl font-black text-yellow-400">0</span>
            </div>
            
            <div class="bg-gray-900 p-2 rounded-xl border-2 border-gray-700 flex flex-col items-center">
                <div id="redL" class="light bg-red-950"></div>
                <div id="yellowL" class="light bg-yellow-950"></div>
                <div id="greenL" class="light bg-green-950"></div>
            </div>
        </div>

        <div id="startScreen" class="absolute inset-0 bg-slate-900/95 flex flex-col items-center justify-center p-8 text-center">
            <h1 class="text-white text-4xl font-black mb-4 tracking-tighter italic">DRIVE<span class="text-red-500">SAFE</span> 1.0</h1>
            <div class="text-gray-400 text-sm mb-8 space-y-2 text-left">
                <p>🚦 <span class="text-white">Red Light:</span> Stop before the line.</p>
                <p>🚶 <span class="text-white">Zebra:</span> Let the pedestrian finish crossing.</p>
                <p>⚠️ <span class="text-white">Penalty:</span> Mistake = Mandatory Safety Quiz.</p>
            </div>
            <button onclick="initGame()" class="bg-red-600 hover:bg-red-500 text-white px-12 py-4 rounded-full font-bold transition-all shadow-lg shadow-red-500/30 active:scale-95">START ENGINE</button>
        </div>

        <div id="quizScreen" class="hidden absolute inset-0 bg-black/95 flex flex-col items-center justify-center p-6">
            <div class="animate-bounce text-red-500 font-bold mb-2 tracking-widest uppercase">⚠️ TRAFFIC VIOLATION!</div>
            <h2 id="qText" class="text-white text-lg font-medium mb-6 text-center italic"></h2>
            <div id="qOptions" class="w-full space-y-3"></div>
        </div>
    </div>

<script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    
    let active = false;
    let score = 0;
    let speed = 0;
    let roadOffset = 0;
    let signal = 'GREEN';
    let player = { x: 175, y: 480, w: 50, h: 90 };
    let hazard = { y: -200, pedX: -30, active: false, type: 'NONE' };

    const questions = [
        { q: "What is the primary purpose of a Zebra Crossing?", a: ["Decorating the road", "Giving pedestrians right of way", "A place to park"], c: 1 },
        { q: "Your speed is 40km/h and the light turns Yellow. You should...", a: ["Speed up to beat it", "Brake safely if possible", "Close your eyes"], c: 1 },
        { q: "When can you move on a Red light?", a: ["If no cars are coming", "Never", "If you are in a hurry"], c: 1 }
    ];

    const keys = {};
    window.addEventListener('keydown', e => keys[e.code] = true);
    window.addEventListener('keyup', e => keys[e.code] = false);

    function initGame() {
        document.getElementById('startScreen').classList.add('hidden');
        active = true;
        score = 0;
        speed = 0;
        cycleLights();
        loop();
    }

    function cycleLights() {
        if (!active) return;
        const times = { 'GREEN': 6000, 'YELLOW': 2000, 'RED': 5000 };
        const order = ['GREEN', 'YELLOW', 'RED'];
        let currentIndex = order.indexOf(signal);
        signal = order[(currentIndex + 1) % order.length];
        
        // Update Visuals
        document.getElementById('redL').style.backgroundColor = signal === 'RED' ? '#ff0000' : '#300';
        document.getElementById('redL').style.boxShadow = signal === 'RED' ? '0 0 20px #ff0000' : 'none';
        document.getElementById('yellowL').style.backgroundColor = signal === 'YELLOW' ? '#ffdd00' : '#330';
        document.getElementById('greenL').style.backgroundColor = signal === 'GREEN' ? '#00ff00' : '#030';
        
        setTimeout(cycleLights, times[signal]);
    }

    function triggerQuiz(msg) {
        active = false;
        const quiz = questions[Math.floor(Math.random()*questions.length)];
        document.getElementById('quizScreen').classList.remove('hidden');
        document.getElementById('qText').innerText = msg + "\n\n" + quiz.q;
        
        const area = document.getElementById('qOptions');
        area.innerHTML = '';
        quiz.a.forEach((opt, i) => {
            const b = document.createElement('button');
            b.className = "w-full bg-slate-800 text-white p-4 rounded-lg hover:bg-red-600 transition text-left border border-white/10";
            b.innerText = `${i+1}. ${opt}`;
            b.onclick = () => {
                if(i === quiz.c) {
                    document.getElementById('quizScreen').classList.add('hidden');
                    hazard.active = false;
                    hazard.y = -200;
                    active = true;
                    loop();
                } else {
                    location.reload(); // Hard reset on wrong answer
                }
            };
            area.appendChild(b);
        });
    }

    function drawRoad() {
        // Road markings
        ctx.strokeStyle = "rgba(255,255,255,0.2)";
        ctx.setLineDash([20, 20]);
        ctx.beginPath();
        ctx.moveTo(200, 0); ctx.lineTo(200, 600);
        ctx.stroke();
        ctx.setLineDash([]);
    }

    function loop() {
        if (!active) return;
        ctx.fillStyle = "#333";
        ctx.fillRect(0, 0, 400, 600);
        
        drawRoad();

        // Physics
        if (keys['ArrowUp']) speed = Math.min(speed + 0.15, 6);
        else speed = Math.max(speed - 0.2, 0);

        // Movement
        if (keys['ArrowLeft'] && player.x > 20) player.x -= 4;
        if (keys['ArrowRight'] && player.x < 330) player.x += 4;

        // Rule: Red Light Penalty
        if (signal === 'RED' && speed > 0.2) {
            triggerQuiz("You failed to stop for the RED light!");
            return;
        }

        // Zebra Logic
        if (!hazard.active && Math.random() < 0.005) {
            hazard.active = true;
            hazard.y = -100;
            hazard.pedX = -20;
        }

        if (hazard.active) {
            hazard.y += speed;
            hazard.pedX += 1.5;

            // Draw Zebra
            for(let i=0; i<10; i++) {
                ctx.fillStyle = "white";
                ctx.fillRect(i*45, hazard.y, 30, 80);
            }
            
            // Draw Pedestrian (Simple Stickman)
            ctx.fillStyle = "#ffcc00"; // Shirt
            ctx.fillRect(hazard.pedX, hazard.y + 20, 15, 25);
            ctx.fillStyle = "#ffdbac"; // Head
            ctx.beginPath();
            ctx.arc(hazard.pedX + 7, hazard.y + 15, 7, 0, Math.PI*2);
            ctx.fill();

            // Collision / Proximity Check
            let carFront = player.y;
            if (speed > 0.5 && carFront < hazard.y + 80 && carFront + 20 > hazard.y) {
                triggerQuiz("Stop completely before the Zebra crossing while people are walking!");
                return;
            }

            if (hazard.y > 600) {
                hazard.active = false;
                score += 100;
                document.getElementById('score').innerText = score;
            }
        }

        // Draw Player (Car)
        ctx.fillStyle = "#1e40af"; // Body
        ctx.roundRect(player.x, player.y, player.w, player.h, 10);
        ctx.fill();
        ctx.fillStyle = "#93c5fd"; // Windshield
        ctx.fillRect(player.x + 5, player.y + 15, 40, 20);
        // Brake Lights
        ctx.fillStyle = speed === 0 ? "#ff0000" : "#500";
        ctx.fillRect(player.x + 5, player.y + 80, 10, 5);
        ctx.fillRect(player.x + 35, player.y + 80, 10, 5);

        requestAnimationFrame(loop);
    }

    // Helper for rounded rects
    CanvasRenderingContext2D.prototype.roundRect = function (x, y, w, h, r) {
      if (w < 2 * r) r = w / 2;
      if (h < 2 * r) r = h / 2;
      this.beginPath();
      this.moveTo(x + r, y);
      this.arcTo(x + w, y, x + w, y + h, r);
      this.arcTo(x + w, y + h, x, y + h, r);
      this.arcTo(x, y + h, x, y, r);
      this.arcTo(x, y, x + w, y, r);
      this.closePath();
      return this;
    }
</script>
</body>
</html>
