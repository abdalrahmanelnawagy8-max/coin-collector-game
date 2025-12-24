
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Simple Coin Collector Game</title>
    <style>
        canvas {
            border: 1px solid #000;
            display: block;
            margin: 20px auto;
            background-color: #222;
        }
        body {
            font-family: Arial, sans-serif;
            text-align: center;
        }
        button {
            width: 50px;
            height: 50px;
            font-size: 20px;
            margin: 5px;
        }
    </style>
</head>
<body>
    <h1>Simple Coin Collector</h1>
    <p>Use arrow keys to move the player (blue square) and collect yellow coins. Avoid red spikes! Score: <span id="score">0</span> | Level: <span id="level">1</span></p>
    <canvas id="gameCanvas" width="800" height="600"></canvas>
    <div id="controls" style="display: none; margin: 20px auto; width: 200px;">
        <button id="up">↑</button><br>
        <button id="left">←</button>
        <button id="down">↓</button>
        <button id="right">→</button>
    </div>
    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const scoreElement = document.getElementById('score');
        const levelElement = document.getElementById('level');

        let player = { x: 50, y: 50, size: 20, speed: 5 };
        let coins = [];
        let score = 0;
        let effects = []; // For score animations
        let level = 1;
        let spikes = [];
        let gameOver = false;
        let animationFrame = 0;
        let isMoving = false;
        let boss = null;
        let bossHealth = 10;

        // Generate initial coins
        for (let i = 0; i < 5; i++) {
            coins.push({
                x: Math.random() * (canvas.width - 20),
                y: Math.random() * (canvas.height - 20),
                size: 15
            });
        }

        // Generate initial spikes
        for (let i = 0; i < 3; i++) {
            spikes.push({
                x: Math.random() * (canvas.width - 20),
                y: Math.random() * (canvas.height - 20),
                size: 20
            });
        }

        const keys = {};
        document.addEventListener('keydown', (e) => keys[e.key] = true);
        document.addEventListener('keyup', (e) => keys[e.key] = false);

        // Mobile controls
        if ('ontouchstart' in window) {
            document.getElementById('controls').style.display = 'block';
            const upBtn = document.getElementById('up');
            const downBtn = document.getElementById('down');
            const leftBtn = document.getElementById('left');
            const rightBtn = document.getElementById('right');
            [upBtn, downBtn, leftBtn, rightBtn].forEach(btn => {
                btn.onmousedown = () => keys[btn.id === 'up' ? 'ArrowUp' : btn.id === 'down' ? 'ArrowDown' : btn.id === 'left' ? 'ArrowLeft' : 'ArrowRight'] = true;
                btn.onmouseup = () => keys[btn.id === 'up' ? 'ArrowUp' : btn.id === 'down' ? 'ArrowDown' : btn.id === 'left' ? 'ArrowLeft' : 'ArrowRight'] = false;
                btn.ontouchstart = (e) => { e.preventDefault(); keys[btn.id === 'up' ? 'ArrowUp' : btn.id === 'down' ? 'ArrowDown' : btn.id === 'left' ? 'ArrowLeft' : 'ArrowRight'] = true; };
                btn.ontouchend = () => keys[btn.id === 'up' ? 'ArrowUp' : btn.id === 'down' ? 'ArrowDown' : btn.id === 'left' ? 'ArrowLeft' : 'ArrowRight'] = false;
            });
        }

        function update() {
            isMoving = keys['ArrowLeft'] || keys['ArrowRight'] || keys['ArrowUp'] || keys['ArrowDown'];
            // Move player
            if (keys['ArrowLeft']) player.x -= player.speed;
            if (keys['ArrowRight']) player.x += player.speed;
            if (keys['ArrowUp']) player.y -= player.speed;
            if (keys['ArrowDown']) player.y += player.speed;

            // Wrap around screen edges
            if (player.x < 0) player.x = canvas.width - player.size;
            if (player.x > canvas.width - player.size) player.x = 0;
            if (player.y < 0) player.y = canvas.height - player.size;
            if (player.y > canvas.height - player.size) player.y = 0;

            // Check spike collisions
            spikes.forEach(spike => {
                if (Math.abs(player.x - spike.x) < player.size && Math.abs(player.y - spike.y) < player.size) {
                    gameOver = true;
                }
            });

            // Move and check boss
            if (boss) {
                boss.x += boss.vx;
                boss.y += boss.vy;
                if (boss.x <= 0 || boss.x + boss.size >= canvas.width) boss.vx = -boss.vx;
                if (boss.y <= 0 || boss.y + boss.size >= canvas.height) boss.vy = -boss.vy;
                if (Math.abs(player.x - boss.x) < player.size && Math.abs(player.y - boss.y) < player.size) {
                    gameOver = true;
                }
            }

            // Check coin collisions
            coins = coins.filter(coin => {
                if (Math.abs(player.x - coin.x) < player.size && Math.abs(player.y - coin.y) < player.size) {
                    score++;
                    scoreElement.textContent = score;
                    effects.push({x: coin.x + coin.size/2, y: coin.y, text: '+1', alpha: 1, vy: -2});
                    if (boss) {
                        bossHealth--;
                        if (bossHealth <= 0) {
                            boss = null;
                            effects.push({x: canvas.width/2, y: canvas.height/2, text: 'Boss Defeated!', alpha: 1, vy: -1});
                        }
                    }
                    // Level up every 10 points
                    if (score % 10 === 0) {
                        level++;
                        levelElement.textContent = level;
                        player.speed += 1;
                        // Add more spikes
                        for (let i = 0; i < 2; i++) {
                            spikes.push({
                                x: Math.random() * (canvas.width - 20),
                                y: Math.random() * (canvas.height - 20),
                                size: 20
                            });
                        }
                        // Boss battle every 5 levels
                        if (level % 5 === 0) {
                            boss = {
                                x: canvas.width / 2,
                                y: canvas.height / 2,
                                size: 40,
                                vx: 2,
                                vy: 2
                            };
                            bossHealth = 10;
                        }
                    }
                    return false; // Remove coin
                }
                return true;
            });

            // Add new coin if all collected
            if (coins.length === 0) {
                for (let i = 0; i < 5; i++) {
                    coins.push({
                        x: Math.random() * (canvas.width - 20),
                        y: Math.random() * (canvas.height - 20),
                        size: 15
                    });
                }
            }

            // Update effects
            effects.forEach(effect => {
                effect.y += effect.vy;
                effect.alpha -= 0.02;
            });
            effects = effects.filter(e => e.alpha > 0);

            animationFrame++;
        }

        function draw() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // Draw player (stickman)
            ctx.strokeStyle = 'blue';
            ctx.lineWidth = 2;
            let px = player.x + player.size / 2;
            let py = player.y;
            let legOffset = 0;
            if (isMoving) {
                legOffset = (animationFrame % 8 < 4) ? -1 : 1;
            }
            // Head
            ctx.beginPath();
            ctx.arc(px, py + 5, 4, 0, Math.PI * 2);
            ctx.stroke();
            // Body
            ctx.beginPath();
            ctx.moveTo(px, py + 9);
            ctx.lineTo(px, py + 18);
            ctx.stroke();
            // Arms
            ctx.beginPath();
            ctx.moveTo(px, py + 12);
            ctx.lineTo(px - 5, py + 15);
            ctx.moveTo(px, py + 12);
            ctx.lineTo(px + 5, py + 15);
            ctx.stroke();
            // Legs
            ctx.beginPath();
            ctx.moveTo(px, py + 18);
            ctx.lineTo(px - 3 + legOffset, py + 25 + legOffset);
            ctx.moveTo(px, py + 18);
            ctx.lineTo(px + 3 - legOffset, py + 25 - legOffset);
            ctx.stroke();

            // Draw coins
            coins.forEach(coin => {
                let gradient = ctx.createRadialGradient(coin.x + coin.size / 2, coin.y + coin.size / 2, 0, coin.x + coin.size / 2, coin.y + coin.size / 2, coin.size / 2);
                gradient.addColorStop(0, 'gold');
                gradient.addColorStop(1, 'darkgoldenrod');
                ctx.fillStyle = gradient;
                ctx.beginPath();
                ctx.arc(coin.x + coin.size / 2, coin.y + coin.size / 2, coin.size / 2, 0, Math.PI * 2);
                ctx.fill();
            });

            // Draw spikes
            ctx.fillStyle = 'red';
            spikes.forEach(spike => {
                ctx.beginPath();
                ctx.moveTo(spike.x, spike.y + spike.size);
                ctx.lineTo(spike.x + spike.size / 2, spike.y);
                ctx.lineTo(spike.x + spike.size, spike.y + spike.size);
                ctx.closePath();
                ctx.fill();
            });

            // Draw boss
            if (boss) {
                ctx.fillStyle = 'darkred';
                ctx.beginPath();
                ctx.moveTo(boss.x, boss.y + boss.size);
                ctx.lineTo(boss.x + boss.size / 2, boss.y);
                ctx.lineTo(boss.x + boss.size, boss.y + boss.size);
                ctx.closePath();
                ctx.fill();
                // Eyes
                ctx.fillStyle = 'yellow';
                ctx.beginPath();
                ctx.arc(boss.x + boss.size / 3, boss.y + boss.size / 3, 5, 0, Math.PI * 2);
                ctx.fill();
                ctx.arc(boss.x + 2 * boss.size / 3, boss.y + boss.size / 3, 5, 0, Math.PI * 2);
                ctx.fill();
                // Pupils
                ctx.fillStyle = 'black';
                ctx.beginPath();
                ctx.arc(boss.x + boss.size / 3, boss.y + boss.size / 3, 2, 0, Math.PI * 2);
                ctx.fill();
                ctx.arc(boss.x + 2 * boss.size / 3, boss.y + boss.size / 3, 2, 0, Math.PI * 2);
                ctx.fill();
            }

            // Draw effects
            effects.forEach(effect => {
                ctx.globalAlpha = effect.alpha;
                ctx.fillStyle = 'white';
                ctx.font = '20px Arial';
                ctx.fillText(effect.text, effect.x, effect.y);
            });
            ctx.globalAlpha = 1; // Reset alpha
        }

        function gameLoop() {
            if (!gameOver) {
                update();
            }
            draw();
            if (gameOver) {
                ctx.fillStyle = 'white';
                ctx.font = '40px Arial';
                ctx.fillText('Game Over!', canvas.width / 2 - 100, canvas.height / 2);
                ctx.font = '20px Arial';
                ctx.fillText('Refresh to play again', canvas.width / 2 - 80, canvas.height / 2 + 50);
                return; // Stop loop
            }
            requestAnimationFrame(gameLoop);
        }

        gameLoop();
    </script>
</body>
</html>
