<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Rhythm Jumper</title>
    <style>
        body, html {
            margin: 0;
            padding: 0;
            overflow: hidden; /* Prevents scrollbars */
            background-color: #000; /* High-contrast background for the page itself */
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
        }
        canvas {
            display: block;
            /* The canvas itself will have its background drawn in JS,
               but a fallback or initial color can be set here if desired.
               #111 is used in JS, so this is mostly for structure. */
            /* background-color: #111; */
        }
    </style>
</head>
<body>
    <canvas id="gameCanvas"></canvas>

    <script>
        // Get the canvas element and its context
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');

        // --- Game Variables ---
        let screenWidth;
        let screenHeight;
        let gameRunning = false;
        let animationFrameId; // To control the game loop

        // Player properties
        let player = {
            x: 50,
            y: 0, // Will be set dynamically
            width: 30,
            height: 30,
            color: '#00FF00', // Bright green for high contrast
            velocityY: 0,
            gravity: 0.6,     // Slightly increased gravity for a snappier feel
            jumpStrength: -12, // Adjusted jump strength
            isJumping: false
        };

        // Obstacle properties
        let obstacles = [];
        let obstacleWidth = 40;
        let obstacleMinHeightRatio = 0.1; // e.g. 10% of screen height
        let obstacleMaxHeightRatio = 0.4; // e.g. 40% of screen height
        let obstacleGapRatio = 0.25;      // e.g. 25% of screen height for the gap
        let obstacleColor = '#FF0000'; // Bright red
        let obstacleSpeed = 3;
        const initialObstacleSpeed = 3; // To reset speed

        // Environment & Game Mechanics
        let groundHeightRatio = 0.05; // 5% of screen height
        let actualGroundHeight; // in pixels, calculated in resizeCanvas

        let gameSpeedIncreaseInterval = 5000; // Increase speed every 5 seconds (in ms)
        let lastSpeedIncreaseTime = 0;
        let speedIncrement = 0.2;

        // Scoring
        let score = 0;
        let highScore = localStorage.getItem('rhythmJumperHighScore') || 0;

        // Beat/Pulse for obstacle spawning
        let beatInterval = 1800; // ms, approx 33 BPM if 1 beat = 1 obstacle. Adjust for rhythm.
        let lastBeatTime = 0; // Time of the last obstacle spawn

        // --- Utility Functions ---
        function getRandom(min, max) {
            return Math.random() * (max - min) + min;
        }

        // --- Canvas & Display ---
        function resizeCanvas() {
            screenWidth = window.innerWidth;
            screenHeight = window.innerHeight;
            canvas.width = screenWidth;
            canvas.height = screenHeight;

            actualGroundHeight = screenHeight * groundHeightRatio;

            // Update player size/position based on screen size if desired
            // For now, player size is fixed, but y position needs adjustment
            player.width = 30; // Or scale with screenWidth/Height
            player.height = 30;


            // Update obstacle dynamic properties based on new screen size
            // obstacleGap = screenHeight * obstacleGapRatio; // Recalculate gap if needed

            // Set player's initial y position
            if (!gameRunning) { // Only set initial position if game hasn't started or during reset
                player.y = screenHeight - player.height - actualGroundHeight;
            } else {
                // If game is running and screen resizes, adjust player Y to prevent falling through floor
                if (player.y + player.height > screenHeight - actualGroundHeight) {
                    player.y = screenHeight - player.height - actualGroundHeight;
                    player.isJumping = false; // May need to reset jump state
                    player.velocityY = 0;
                }
            }
            // Redraw elements if game is running and screen resizes (or on initial load)
            if (gameRunning) {
                draw();
            } else {
                // If game is not running, show the appropriate message (start or game over)
                if (score > 0 || (localStorage.getItem('rhythmJumperPlayedOnce') === 'true')) { // Show game over if a game was played
                    drawGameOverMessage();
                } else {
                    drawStartMessage();
                }
            }
        }

        // --- Game Control ---
        function handleTap(event) {
            if (event) event.preventDefault(); // Prevent default touch behavior

            if (!gameRunning) {
                startGame();
            } else if (!player.isJumping) {
                player.velocityY = player.jumpStrength;
                player.isJumping = true;
            }
        }

        // Event listeners
        canvas.addEventListener('mousedown', handleTap);
        canvas.addEventListener('touchstart', handleTap, { passive: false });
        document.addEventListener('keydown', function(e) { // Allow spacebar to jump too
            if (e.code === 'Space') {
                e.preventDefault(); // Prevent space from scrolling the page
                handleTap(null);
            }
        });


        // --- Drawing ---
        function drawPlayer() {
            ctx.fillStyle = player.color;
            ctx.fillRect(player.x, player.y, player.width, player.height);
        }

        function drawObstacles() {
            ctx.fillStyle = obstacleColor;
            obstacles.forEach(obstacle => {
                // Top part of the obstacle
                ctx.fillRect(obstacle.x, 0, obstacle.width, obstacle.topHeight);
                // Bottom part of the obstacle
                ctx.fillRect(obstacle.x, screenHeight - obstacle.bottomHeight - actualGroundHeight, obstacle.width, obstacle.bottomHeight);
            });
        }

        function drawScore() {
            ctx.fillStyle = '#FFFFFF';
            ctx.font = `${Math.max(18, screenWidth * 0.03)}px Arial`; // Responsive font size
            ctx.textAlign = 'left';
            ctx.fillText(`Score: ${score}`, 20, Math.max(30, screenHeight * 0.05));
            ctx.textAlign = 'right';
            ctx.fillText(`High Score: ${highScore}`, screenWidth - 20, Math.max(30, screenHeight * 0.05));
        }

        function drawMessages(mainText, subText1, subText2, subText3) {
            const overlayAlpha = gameRunning ? 0 : 0.7; // No overlay if game is somehow running with message
            ctx.fillStyle = `rgba(0, 0, 0, ${overlayAlpha})`;
            ctx.fillRect(0, 0, screenWidth, screenHeight);

            ctx.fillStyle = '#FFFFFF';
            ctx.textAlign = 'center';

            const mainFontSize = Math.max(28, screenWidth * 0.07);
            const subFontSize = Math.max(18, screenWidth * 0.04);
            const detailFontSize = Math.max(16, screenWidth * 0.03);

            ctx.font = `${mainFontSize}px Arial`;
            ctx.fillText(mainText, screenWidth / 2, screenHeight / 2 - mainFontSize * 1.2);

            if (subText1) {
                ctx.font = `${subFontSize}px Arial`;
                ctx.fillText(subText1, screenWidth / 2, screenHeight / 2);
            }
            if (subText2) {
                ctx.font = `${detailFontSize}px Arial`;
                ctx.fillText(subText2, screenWidth / 2, screenHeight / 2 + subFontSize * 1.5);
            }
            if (subText3) {
                ctx.font = `${detailFontSize}px Arial`;
                ctx.fillText(subText3, screenWidth / 2, screenHeight / 2 + subFontSize * 1.5 + detailFontSize * 1.5);
            }
        }

        function drawGameOverMessage() {
            drawMessages('Game Over', `Your Score: ${score}`, `High Score: ${highScore}`, 'Tap or Space to Restart');
        }

        function drawStartMessage() {
            let highScoreText = highScore > 0 ? `Current High Score: ${highScore}` : '';
            drawMessages('Rhythm Jumper', 'Tap or Press Space to Start', highScoreText, '');
            // Clear any residual score from a potential previous game over state if starting fresh
            if(score > 0 && !gameRunning) score = 0;
        }


        function draw() {
            // Clear the canvas
            ctx.fillStyle = '#111'; // Main background color
            ctx.fillRect(0, 0, screenWidth, screenHeight);

            // Draw ground line
            ctx.fillStyle = '#333';
            ctx.fillRect(0, screenHeight - actualGroundHeight, screenWidth, actualGroundHeight);

            drawPlayer();
            drawObstacles();
            drawScore();

            // Message handling is now done by resizeCanvas or specific game state functions
            // to prevent drawing them over a running game.
        }

        // --- Game Logic Updates ---
        function updatePlayer() {
            player.y += player.velocityY;
            player.velocityY += player.gravity;

            // Ground collision
            if (player.y + player.height > screenHeight - actualGroundHeight) {
                player.y = screenHeight - player.height - actualGroundHeight;
                player.velocityY = 0;
                player.isJumping = false;
            }

            // Prevent player from going above the screen (optional)
            if (player.y < 0) {
                player.y = 0;
                player.velocityY = 0; // Stop upward momentum if hitting ceiling
            }
        }

        function spawnObstacle() {
            const obstacleGap = screenHeight * obstacleGapRatio;
            const minH = screenHeight * obstacleMinHeightRatio;
            const maxH = screenHeight * obstacleMaxHeightRatio;

            // The total height available for the two parts of the obstacle, excluding ground
            const availableHeightForParts = screenHeight - actualGroundHeight - obstacleGap;

            let topObstacleHeight = getRandom(minH, Math.min(maxH, availableHeightForParts - minH));
            let bottomObstacleHeight = availableHeightForParts - topObstacleHeight;

            // Ensure bottom obstacle isn't too small if top is large
            if (bottomObstacleHeight < minH && availableHeightForParts > minH + minH ) { // check if possible to meet minH for both
                 bottomObstacleHeight = minH;
                 topObstacleHeight = availableHeightForParts - bottomObstacleHeight;
            }
             // Ensure top obstacle isn't too small
            if (topObstacleHeight < minH && availableHeightForParts > minH + minH) {
                topObstacleHeight = minH;
                bottomObstacleHeight = availableHeightForParts - topObstacleHeight;
            }


            obstacles.push({
                x: screenWidth,
                width: obstacleWidth, // Could also be responsive: screenWidth * 0.05
                topHeight: topObstacleHeight,
                bottomHeight: bottomObstacleHeight,
                passed: false // To track for scoring
            });
        }

        function updateObstacles(currentTime) {
            // Spawn new obstacles based on beat/pulse
            if (gameRunning && currentTime - lastBeatTime > beatInterval) {
                spawnObstacle();
                lastBeatTime = currentTime;
            }

            for (let i = obstacles.length - 1; i >= 0; i--) {
                let obs = obstacles[i];
                obs.x -= obstacleSpeed;

                // Scoring: if obstacle passed player's x position and hasn't been scored yet
                if (!obs.passed && obs.x + obs.width < player.x) {
                    score++;
                    obs.passed = true;
                }

                // Remove obstacles that are off-screen
                if (obs.x + obs.width < 0) {
                    obstacles.splice(i, 1);
                }
            }
        }

        function checkCollisions() {
            for (let obs of obstacles) {
                // Check collision with top part of obstacle
                if (player.x < obs.x + obs.width &&
                    player.x + player.width > obs.x &&
                    player.y < obs.topHeight) {
                    return true; // Collision
                }
                // Check collision with bottom part of obstacle
                // The Y for the top-left corner of the bottom obstacle is screenHeight - obs.bottomHeight - actualGroundHeight
                if (player.x < obs.x + obs.width &&
                    player.x + player.width > obs.x &&
                    player.y + player.height > screenHeight - obs.bottomHeight - actualGroundHeight) {
                    return true; // Collision
                }
            }
            return false; // No collision
        }

        function updateGameSpeed(currentTime) {
            if (gameRunning && currentTime - lastSpeedIncreaseTime > gameSpeedIncreaseInterval) {
                obstacleSpeed += speedIncrement;
                lastSpeedIncreaseTime = currentTime;
                // console.log("Game speed increased to:", obstacleSpeed.toFixed(2));
            }
        }

        // --- Game Loop ---
        function gameLoop(currentTime) {
            if (!gameRunning) { // Should be caught by gameOver or startGame's cancelAnimationFrame
                return;
            }

            updatePlayer();
            updateObstacles(currentTime);
            updateGameSpeed(currentTime);

            if (checkCollisions()) {
                gameOver();
                return; // Important to stop further processing in this frame
            }

            draw(); // Draw all game elements

            animationFrameId = requestAnimationFrame(gameLoop);
        }

        // --- Game State Management ---
        function startGame() {
            if (gameRunning) return; // Prevent multiple starts

            gameRunning = true;
            localStorage.setItem('rhythmJumperPlayedOnce', 'true');
            score = 0;
            obstacleSpeed = initialObstacleSpeed;
            obstacles = [];

            // Reset player position and state
            player.y = screenHeight - player.height - actualGroundHeight;
            player.velocityY = 0;
            player.isJumping = false;

            lastBeatTime = performance.now();
            lastSpeedIncreaseTime = performance.now();

            if (animationFrameId) {
                cancelAnimationFrame(animationFrameId);
            }
            animationFrameId = requestAnimationFrame(gameLoop);
        }

        function gameOver() {
            gameRunning = false;
            if (animationFrameId) {
                cancelAnimationFrame(animationFrameId);
            }

            if (score > highScore) {
                highScore = score;
                localStorage.setItem('rhythmJumperHighScore', highScore);
            }
            drawGameOverMessage(); // Explicitly draw game over message
            // console.log("Game Over! Score:", score, "High Score:", highScore);
        }

        // --- Initialization ---
        function init() {
            window.addEventListener('resize', resizeCanvas); // Adjust canvas on window resize
            resizeCanvas(); // Initial resize and draw start message
            // drawStartMessage() is called inside resizeCanvas if !gameRunning
        }

        // Start the initialization process
        init();
    </script>
</body>
</html>
