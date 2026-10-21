# jogo-camilo
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Neon Breakout</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Orbitron:wght@400;700;900&display=swap');

        body {
            margin: 0;
            padding: 0;
            background-color: #050510;
            color: #fff;
            font-family: 'Orbitron', sans-serif;
            overflow: hidden; /* Evita scroll na tela durante o jogo */
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            background-image: 
                radial-gradient(circle at 50% 50%, #1a0b2e 0%, #050510 100%);
        }

        #game-wrapper {
            position: relative;
            width: 100%;
            max-width: 800px; /* Resolução interna do jogo */
            aspect-ratio: 4 / 3;
            box-shadow: 0 0 20px rgba(0, 255, 255, 0.2), 
                        inset 0 0 20px rgba(255, 0, 255, 0.2);
            border: 2px solid #333;
            border-radius: 10px;
            overflow: hidden;
            background: rgba(0, 0, 0, 0.6);
        }

        canvas {
            display: block;
            width: 100%;
            height: 100%;
        }

        /* UI Overlays */
        #ui-layer {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            background: rgba(5, 5, 16, 0.85);
            backdrop-filter: blur(4px);
            z-index: 10;
            transition: opacity 0.3s ease;
        }

        .hidden {
            opacity: 0;
            pointer-events: none;
        }

        h1 {
            font-size: 3rem;
            text-transform: uppercase;
            letter-spacing: 5px;
            margin-bottom: 10px;
            color: #fff;
            text-shadow: 0 0 10px #0ff, 0 0 20px #0ff, 0 0 40px #0ff;
            text-align: center;
        }

        p.subtitle {
            font-size: 1rem;
            color: #aaa;
            margin-bottom: 30px;
            text-align: center;
        }

        .neon-btn {
            background: transparent;
            color: #f0f;
            border: 2px solid #f0f;
            padding: 15px 40px;
            font-size: 1.5rem;
            font-family: 'Orbitron', sans-serif;
            text-transform: uppercase;
            cursor: pointer;
            border-radius: 5px;
            box-shadow: 0 0 10px #f0f, inset 0 0 10px #f0f;
            text-shadow: 0 0 5px #f0f;
            transition: all 0.2s ease;
        }

        .neon-btn:hover {
            background: #f0f;
            color: #000;
            box-shadow: 0 0 20px #f0f, inset 0 0 20px #f0f;
        }

        #hud {
            position: absolute;
            top: 15px;
            left: 20px;
            right: 20px;
            display: flex;
            justify-content: space-between;
            font-size: 1.2rem;
            color: #0ff;
            text-shadow: 0 0 5px #0ff;
            pointer-events: none;
            z-index: 5;
        }

        /* Efeito visual de grade ao fundo do canvas */
        #grid-bg {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background-size: 40px 40px;
            background-image: linear-gradient(to right, rgba(255, 255, 255, 0.05) 1px, transparent 1px),
                              linear-gradient(to bottom, rgba(255, 255, 255, 0.05) 1px, transparent 1px);
            z-index: 1;
            pointer-events: none;
        }
        
        #gameCanvas {
            position: relative;
            z-index: 2;
        }
    </style>
</head>
<body>

    <div id="game-wrapper">
        <div id="grid-bg"></div>
        
        <div id="hud">
            <div>SCORE: <span id="scoreVal">0</span></div>
            <div>LIVES: <span id="livesVal">3</span></div>
        </div>

        <div id="ui-layer">
            <h1 id="game-title">Neon Breakout</h1>
            <p class="subtitle" id="game-message">Controle a barra com o Mouse, Toque ou Setas Direcionais.</p>
            <button class="neon-btn" id="start-btn">Iniciar Jogo</button>
        </div>

        <!-- Canvas com resolução interna fixa para manter a proporção e física consistentes -->
        <canvas id="gameCanvas" width="800" height="600"></canvas>
    </div>

    <script>
        const canvas = document.getElementById("gameCanvas");
        const ctx = canvas.getContext("2d");
        
        // Elementos de UI
        const uiLayer = document.getElementById("ui-layer");
        const startBtn = document.getElementById("start-btn");
        const gameTitle = document.getElementById("game-title");
        const gameMessage = document.getElementById("game-message");
        const scoreUI = document.getElementById("scoreVal");
        const livesUI = document.getElementById("livesVal");

        // Variáveis de Estado do Jogo
        let animationId;
        let isPlaying = false;
        let score = 0;
        let lives = 3;

        // Cores Neon
        const colors = [
            '#FF0055', // Rosa Choque
            '#00FFFF', // Ciano
            '#B400FF', // Roxo
            '#FFFF00', // Amarelo
            '#00FF00'  // Verde
        ];

        // Objetos do Jogo
        const ball = {
            x: canvas.width / 2,
            y: canvas.height - 40,
            dx: 5,
            dy: -5,
            radius: 8,
            color: '#ffffff',
            glow: '#00FFFF'
        };

        const paddle = {
            height: 15,
            width: 120,
            x: (canvas.width - 120) / 2,
            y: canvas.height - 20,
            color: '#ffffff',
            glow: '#FF00FF',
            speed: 8,
            dx: 0
        };

        // Configuração dos Tijolos
        const brickConfig = {
            rowCount: 6,
            columnCount: 8,
            width: 80,
            height: 25,
            padding: 15,
            offsetTop: 60,
            offsetLeft: 25
        };

        let bricks = [];

        // Controles
        let rightPressed = false;
        let leftPressed = false;

        // Inicializa a matriz de tijolos
        function initBricks() {
            bricks = [];
            for (let c = 0; c < brickConfig.columnCount; c++) {
                bricks[c] = [];
                for (let r = 0; r < brickConfig.rowCount; r++) {
                    // Atribui uma cor neon baseada na linha
                    const neonColor = colors[r % colors.length];
                    bricks[c][r] = { x: 0, y: 0, status: 1, color: neonColor };
                }
            }
        }

        // Funções de Desenho
        function drawBall() {
            ctx.beginPath();
            ctx.arc(ball.x, ball.y, ball.radius, 0, Math.PI * 2);
            ctx.fillStyle = ball.color;
            ctx.shadowBlur = 15;
            ctx.shadowColor = ball.glow;
            ctx.fill();
            ctx.closePath();
            // Reset shadow
            ctx.shadowBlur = 0;
        }

        function drawPaddle() {
            ctx.beginPath();
            // Barra arredondada
            ctx.roundRect(paddle.x, paddle.y, paddle.width, paddle.height, 8);
            ctx.fillStyle = paddle.color;
            ctx.shadowBlur = 20;
            ctx.shadowColor = paddle.glow;
            ctx.fill();
            ctx.closePath();
            ctx.shadowBlur = 0;
        }

        function drawBricks() {
            for (let c = 0; c < brickConfig.columnCount; c++) {
                for (let r = 0; r < brickConfig.rowCount; r++) {
                    if (bricks[c][r].status === 1) {
                        const brickX = (c * (brickConfig.width + brickConfig.padding)) + brickConfig.offsetLeft;
                        const brickY = (r * (brickConfig.height + brickConfig.padding)) + brickConfig.offsetTop;
                        
                        bricks[c][r].x = brickX;
                        bricks[c][r].y = brickY;
                        
                        ctx.beginPath();
                        ctx.roundRect(brickX, brickY, brickConfig.width, brickConfig.height, 4);
                        // Efeito Neon para os tijolos
                        ctx.strokeStyle = bricks[c][r].color;
                        ctx.lineWidth = 2;
                        ctx.shadowBlur = 10;
                        ctx.shadowColor = bricks[c][r].color;
                        ctx.stroke();
                        
                        // Preenchimento com opacidade
                        ctx.fillStyle = bricks[c][r].color + '40'; // Adiciona transparência ao hex
                        ctx.fill();
                        
                        ctx.closePath();
                        ctx.shadowBlur = 0;
                    }
                }
            }
        }

        // Lógica de Colisão
        function collisionDetection() {
            for (let c = 0; c < brickConfig.columnCount; c++) {
                for (let r = 0; r < brickConfig.rowCount; r++) {
                    const b = bricks[c][r];
                    if (b.status === 1) {
                        // Verifica se a bola está dentro da área do tijolo
                        if (ball.x > b.x && ball.x < b.x + brickConfig.width && 
                            ball.y > b.y && ball.y < b.y + brickConfig.height) {
                            
                            ball.dy = -ball.dy; // Rebate a bola
                            b.status = 0; // "Quebra" o tijolo
                            score += 10;
                            scoreUI.innerText = score;
                            
                            // Efeito visual na bola ao quebrar tijolo
                            ball.glow = b.color;
                            
                            // Condição de Vitória
                            if (score === brickConfig.rowCount * brickConfig.columnCount * 10) {
                                gameOver(true);
                            }
                        }
                    }
                }
            }
        }

        // Loop principal do Jogo
        function draw() {
            if (!isPlaying) return;

            // Limpa o frame anterior (usando um fill para deixar rastro suave, estilo cyberpunk)
            ctx.fillStyle = 'rgba(0, 0, 0, 0.3)';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            drawBricks();
            drawBall();
            drawPaddle();
            collisionDetection();

            // Colisão da bola com as paredes laterais
            if (ball.x + ball.dx > canvas.width - ball.radius || ball.x + ball.dx < ball.radius) {
                ball.dx = -ball.dx;
            }
            // Colisão da bola com o teto
            if (ball.y + ball.dy < ball.radius) {
                ball.dy = -ball.dy;
            } 
            // Colisão da bola com o chão (abaixo da barra)
            else if (ball.y + ball.dy > canvas.height - ball.radius) {
                // Se bater na barra
                if (ball.x > paddle.x && ball.x < paddle.x + paddle.width) {
                    ball.dy = -ball.dy;
                    // Adiciona um pouco de controle direcional dependendo de onde bateu na barra
                    let hitPoint = ball.x - (paddle.x + paddle.width / 2);
                    ball.dx = hitPoint * 0.15;
                } else {
                    // Perdeu vida
                    lives--;
                    livesUI.innerText = lives;
                    if (!lives) {
                        gameOver(false);
                        return; // Para a execução do draw atual
                    } else {
                        resetBall();
                    }
                }
            }

            // Movimento da barra via teclado
            if (rightPressed && paddle.x < canvas.width - paddle.width) {
                paddle.x += paddle.speed;
            } else if (leftPressed && paddle.x > 0) {
                paddle.x -= paddle.speed;
            }

            // Move a bola
            ball.x += ball.dx;
            ball.y += ball.dy;

            animationId = requestAnimationFrame(draw);
        }

        function resetBall() {
            ball.x = canvas.width / 2;
            ball.y = canvas.height - 40;
            ball.dx = 5 * (Math.random() > 0.5 ? 1 : -1); // Direção inicial aleatória
            ball.dy = -5;
            paddle.x = (canvas.width - paddle.width) / 2;
        }

        function gameOver(isWin) {
            isPlaying = false;
            cancelAnimationFrame(animationId);
            
            uiLayer.classList.remove("hidden");
            
            if (isWin) {
                gameTitle.innerText = "SYSTEM HACKED";
                gameTitle.style.color = "#0f0";
                gameTitle.style.textShadow = "0 0 10px #0f0, 0 0 20px #0f0";
                gameMessage.innerText = `Você venceu! Pontuação Final: ${score}`;
                startBtn.innerText = "Jogar Novamente";
                startBtn.style.borderColor = "#0f0";
                startBtn.style.color = "#0f0";
                startBtn.style.boxShadow = "0 0 10px #0f0, inset 0 0 10px #0f0";
            } else {
                gameTitle.innerText = "GAME OVER";
                gameTitle.style.color = "#f00";
                gameTitle.style.textShadow = "0 0 10px #f00, 0 0 20px #f00";
                gameMessage.innerText = `Sistema bloqueado. Pontuação: ${score}`;
                startBtn.innerText = "Tentar Novamente";
                startBtn.style.borderColor = "#f00";
                startBtn.style.color = "#f00";
                startBtn.style.boxShadow = "0 0 10px #f00, inset 0 0 10px #f00";
            }
        }

        function startGame() {
            score = 0;
            lives = 3;
            scoreUI.innerText = score;
            livesUI.innerText = lives;
            initBricks();
            resetBall();
            uiLayer.classList.add("hidden");
            
            // Limpa totalmente a tela antes de iniciar
            ctx.fillStyle = '#000';
            ctx.fillRect(0, 0, canvas.width, canvas.height);
            
            isPlaying = true;
            draw();
        }

        // --- Event Listeners para Controles ---

        // Iniciar botão
        startBtn.addEventListener("click", startGame);

        // Teclado
        document.addEventListener("keydown", (e) => {
            if (e.key === "Right" || e.key === "ArrowRight") rightPressed = true;
            else if (e.key === "Left" || e.key === "ArrowLeft") leftPressed = true;
        });

        document.addEventListener("keyup", (e) => {
            if (e.key === "Right" || e.key === "ArrowRight") rightPressed = false;
            else if (e.key === "Left" || e.key === "ArrowLeft") leftPressed = false;
        });

        // Mouse e Touch (Responsividade)
        function handleMove(clientX) {
            // Pega a posição relativa ao tamanho atual do canvas na tela
            const rect = canvas.getBoundingClientRect();
            // Calcula a proporção caso a tela seja menor que 800px (tamanho do CSS)
            const scaleX = canvas.width / rect.width; 
            
            let relativeX = (clientX - rect.left) * scaleX;
            if (relativeX > 0 && relativeX < canvas.width) {
                // Centraliza a barra no ponteiro
                paddle.x = relativeX - paddle.width / 2;
                
                // Impede que a barra saia da tela
                if (paddle.x < 0) paddle.x = 0;
                if (paddle.x + paddle.width > canvas.width) paddle.x = canvas.width - paddle.width;
            }
        }

        canvas.addEventListener("mousemove", (e) => handleMove(e.clientX));
        
        canvas.addEventListener("touchmove", (e) => {
            e.preventDefault(); // Evita o scroll da página ao deslizar no canvas
            handleMove(e.touches[0].clientX);
        }, { passive: false });

    </script>
</body>
</html>
