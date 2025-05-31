<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Evidence Lockdown: Rhythm Scan</title>
    <style>
        body, html {
            margin: 0;
            padding: 0;
            overflow: hidden;
            background-color: #0A0A0A; /* Dark background for the page */
            color: #E0E0E0;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }
        canvas {
            display: block;
            background-color: #181818; /* Slightly lighter dark grey for canvas */
            box-shadow: 0 0 15px rgba(0, 255, 255, 0.3); /* Cyan glow for a techy feel */
        }
    </style>
</head>
<body>
    <canvas id="gameCanvas"></canvas>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');

        // --- Game Configuration ---
        let screenWidth, screenHeight;
        let gameRunning = false;
        let animationFrameId;

        const BEATS_PER_MINUTE = 90; // Rhythm of the game
        const BEAT_INTERVAL = 60000 / BEATS_PER_MINUTE; // Milliseconds per beat
        let lastBeatTime = 0;
        let beatCount = 0; // Total beats elapsed in current game

        // Scanner Configuration
        const SCANNER_COLOR = 'rgba(0, 255, 255, 0.7)'; // Cyan, semi-transparent
        const SCANNER_WIDTH = 8;
        let scannerX = 0;
        const SCANNER_SPEED_PIXELS_PER_MS = 0.15; // Adjust for desired sweep speed
        let scannerDirection = 1; // 1 for right, -1 for left

        // Evidence & Distractor Configuration
        let items = []; // Will hold evidence and distractors
        const ITEM_RADIUS = 15;
        const EVIDENCE_COLOR = 'rgba(0, 255, 0, 0.8)'; // Green
        const DISTRACTOR_COLOR = 'rgba(255, 165, 0, 0.8)'; // Orange
        const ITEM_SPAWN_CHANCE_PER_BEAT = 0.6; // 60% chance to spawn an item on a beat
        const DISTRACTOR_SPAWN_CHANCE = 0.3; // 30% of spawned items are distractors
        const ITEM_ACTIVE_DURATION_MS = BEAT_INTERVAL * 3.5; // How long an item stays tappable
        const PERFECT_TAP_WINDOW_MS = BEAT_INTERVAL / 3; // Time window for a perfect tap

        // Scoring
        let score = 0;
        let highScore = localStorage.getItem('evidenceLockdownHighScore') || 0;
        let combo = 0;
        const SCORE_PER_EVIDENCE = 100;
        const COMBO_MULTIPLIER_BONUS = 10; // Extra points per combo level
        const PENALTY_FOR_DISTRACTOR = -150;
        const PENALTY_FOR_MISS = -50; // If an evidence item expires untouched

        // Feedback
        let tapFeedback = []; // {x, y, text, color, time}

        // Game State
        let lives = 3; // Player lives
        const INITIAL_LIVES = 3;

        // --- Utility Functions ---
        function getRandom(min, max) { return Math.random() * (max - min) + min; }
        function lerp(start, end, amt) { return (1 - amt) * start + amt * end; }

        // --- Canvas & Display ---
        function resizeCanvas() {
            screenWidth = window.innerWidth;
            screenHeight = window.innerHeight;
            canvas.width = screenWidth;
            canvas.height = screenHeight;
            scannerX = SCANNER_WIDTH / 2; // Start scanner at the left
            draw(); // Redraw, especially for start/game over messages
        }

        // --- Input Handling ---
        function handleTap(event) {
            if (event) event.preventDefault();

            if (!gameRunning) {
                startGame();
                return;
            }

            const tapTime = performance.now();
            let itemHit = false;

            for (let i = items.length - 1; i >= 0; i--) {
                const item = items[i];
                const distanceToScanner = Math.abs(item.x - scannerX);

                // Check if tap is near the scanner line and an item is close to the line
                if (distanceToScanner < item.radius + SCANNER_WIDTH && Math.abs(tapTime - item.spawnTime - (ITEM_ACTIVE_DURATION_MS / 2)) < PERFECT_TAP_WINDOW_MS * 2) {
                    // More precise check if scanner is actually overlapping item vertically when line is there
                    if (scannerX >= item.x - item.radius && scannerX <= item.x + item.radius &&
                        tapTime >= item.spawnTime && tapTime <= item.spawnTime + ITEM_ACTIVE_DURATION_MS) {

                        if (item.type === 'evidence') {
                            score += SCORE_PER_EVIDENCE + (combo * COMBO_MULTIPLIER_BONUS);
                            combo++;
                            addTapFeedback(item.x, item.y, `+${SCORE_PER_EVIDENCE + ((combo-1) * COMBO_MULTIPLIER_BONUS)}`, EVIDENCE_COLOR, tapTime);
                            // Optional: bonus for perfect timing
                            const timingAccuracy = Math.abs(tapTime - (item.spawnTime + BEAT_INTERVAL)); // Assuming items should be tapped ON a beat after spawn
                            if(timingAccuracy < PERFECT_TAP_WINDOW_MS / 2) {
                                score += 50; // Perfect timing bonus
                                addTapFeedback(item.x, item.y - 20, `Perfect! +50`, SCANNER_COLOR, tapTime);
                            }

                        } else { // Distractor
                            score += PENALTY_FOR_DISTRACTOR;
                            combo = 0; // Reset combo
                            lives--;
                            addTapFeedback(item.x, item.y, `${PENALTY_FOR_DISTRACTOR}`, DISTRACTOR_COLOR, tapTime);
                        }
                        items.splice(i, 1); // Remove item
                        itemHit = true;
                        break; // Process one item per tap
                    }
                }
            }

            if (!itemHit) {
                // Optional: Penalty for tapping nothing, or just do nothing
                // combo = 0;
                // addTapFeedback(scannerX, screenHeight / 2, 'Miss!', '#FFAAAA', tapTime);
            }

            if (lives <= 0) {
                gameOver();
            }
        }

        function addTapFeedback(x, y, text, color, time) {
            tapFeedback.push({ x, y, text, color, spawnTime: time, alpha: 1 });
        }

        // --- Item Spawning & Management ---
        function spawnItem(currentTime) {
            if (Math.random() > ITEM_SPAWN_CHANCE_PER_BEAT) return;

            const type = Math.random() < DISTRACTOR_SPAWN_CHANCE ? 'distractor' : 'evidence';
            const x = getRandom(ITEM_RADIUS * 2, screenWidth - ITEM_RADIUS * 2);
            const y = getRandom(ITEM_RADIUS * 2, screenHeight - ITEM_RADIUS * 2);

            // Avoid spawning too close to existing items (simple check)
            for (const existingItem of items) {
                const dist = Math.hypot(existingItem.x - x, existingItem.y - y);
                if (dist < ITEM_RADIUS * 4) return; // Too close, skip spawn this time
            }

            items.push({
                x, y,
                radius: ITEM_RADIUS,
                type,
                color: type === 'evidence' ? EVIDENCE_COLOR : DISTRACTOR_COLOR,
                spawnTime: currentTime,
                alpha: 0 // Start invisible, fade in
            });
        }

        function updateItems(currentTime, deltaTime) {
            for (let i = items.length - 1; i >= 0; i--) {
                const item = items[i];

                // Fade in
                if (item.alpha < 1) {
                    item.alpha += (deltaTime / (BEAT_INTERVAL / 2))); // Fade in over half a beat
                    if (item.alpha > 1) item.alpha = 1;
                }

                // Check for expiration
                if (currentTime > item.spawnTime + ITEM_ACTIVE_DURATION_MS) {
                    if (item.type === 'evidence') {
                        score += PENALTY_FOR_MISS;
                        combo = 0;
                        lives--;
                        addTapFeedback(item.x, item.y, 'Missed!', DISTRACTOR_COLOR, currentTime);
                    }
                    items.splice(i, 1);
                }
            }
            if (lives <= 0 && gameRunning) {
                 gameOver();
            }
        }

        // --- Scanner Logic ---
        function updateScanner(deltaTime) {
            scannerX += SCANNER_SPEED_PIXELS_PER_MS * deltaTime * scannerDirection;

            if (scannerX - SCANNER_WIDTH / 2 > screenWidth) {
                scannerX = screenWidth - SCANNER_WIDTH / 2;
                scannerDirection = -1;
            } else if (scannerX + SCANNER_WIDTH / 2 < 0) {
                scannerX = SCANNER_WIDTH / 2;
                scannerDirection = 1;
            }
        }

        // --- Drawing Functions ---
        function drawBackground() {
            // Could add a subtle grid or "scanline" effect here later
            ctx.fillStyle = '#181818';
            ctx.fillRect(0, 0, screenWidth, screenHeight);
        }

        function drawScanner() {
            ctx.fillStyle = SCANNER_COLOR;
            ctx.fillRect(scannerX - SCANNER_WIDTH / 2, 0, SCANNER_WIDTH, screenHeight);

            // Pulse effect for scanner based on beat
            const timeSinceLastBeat = performance.now() - lastBeatTime;
            const pulseProgress = Math.min(1, timeSinceLastBeat / BEAT_INTERVAL);
            const pulseAlpha = Math.sin(pulseProgress * Math.PI) * 0.3; // Sine wave for smooth pulse

            ctx.fillStyle = `rgba(0, 255, 255, ${pulseAlpha})`;
            ctx.fillRect(scannerX - SCANNER_WIDTH * 2, 0, SCANNER_WIDTH * 4, screenHeight);
        }

        function drawItems(currentTime) {
            items.forEach(item => {
                const age = currentTime - item.spawnTime;
                const pulsate = Math.sin(age / (BEAT_INTERVAL/2) * Math.PI) * 0.2 + 0.8; // 0.8 to 1.0 scale
                const currentRadius = item.radius * pulsate * item.alpha;

                ctx.beginPath();
                ctx.arc(item.x, item.y, currentRadius, 0, Math.PI * 2);
                ctx.fillStyle = item.color.replace(/, [0-9\.]+\)/, `, ${item.alpha * (item.type === 'evidence' ? 0.8 : 0.6)})`); // Adjust base alpha
                ctx.fill();

                // Outline to indicate "active" or "perfect tap" window (conceptual)
                const timeToPerfect = item.spawnTime + BEAT_INTERVAL - currentTime; // Example: perfect tap is on the beat after spawn
                if(item.type === 'evidence' && timeToPerfect > 0 && timeToPerfect < PERFECT_TAP_WINDOW_MS * 1.5) {
                    ctx.strokeStyle = `rgba(255, 255, 0, ${0.5 * item.alpha * (1 - timeToPerfect / (PERFECT_TAP_WINDOW_MS * 1.5))})`; // Fading yellow
                    ctx.lineWidth = 3;
                    ctx.stroke();
                }
            });
        }

        function drawUI() {
            ctx.fillStyle = '#E0E0E0';
            const fontSize = Math.max(18, screenWidth * 0.025);
            ctx.font = `${fontSize}px 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif`;
            ctx.textAlign = 'left';
            ctx.fillText(`Score: ${score}`, 20, fontSize * 1.5);
            ctx.fillText(`Lives: ${lives}`, 20, fontSize * 3);

            ctx.textAlign = 'right';
            ctx.fillText(`High Score: ${highScore}`, screenWidth - 20, fontSize * 1.5);
            if (combo > 1) {
                ctx.fillStyle = SCANNER_COLOR;
                ctx.fillText(`Combo: x${combo}`, screenWidth - 20, fontSize * 3);
            }
        }

        function drawTapFeedback(currentTime) {
            for (let i = tapFeedback.length - 1; i >= 0; i--) {
                const fb = tapFeedback[i];
                const age = currentTime - fb.spawnTime;
                const fadeDuration = 800; // 0.8 seconds

                if (age > fadeDuration) {
                    tapFeedback.splice(i, 1);
                    continue;
                }

                fb.alpha = 1 - (age / fadeDuration);
                const yPos = fb.y - (age / 20); // Text floats up

                ctx.font = `bold ${Math.max(16, screenWidth*0.02)}px Arial`;
                ctx.fillStyle = fb.color.replace(/, [0-9\.]+\)/, `, ${fb.alpha})`); // Use feedback's own alpha
                ctx.textAlign = 'center';
                ctx.fillText(fb.text, fb.x, yPos);
            }
        }

        function drawMessages(mainText, subText1, subText2 = "") {
            ctx.fillStyle = 'rgba(10, 10, 10, 0.75)'; // Dark overlay
            ctx.fillRect(0, 0, screenWidth, screenHeight);

            ctx.fillStyle = '#E0E0E0';
            ctx.textAlign = 'center';

            const mainFontSize = Math.max(30, screenWidth * 0.06);
            const subFontSize = Math.max(18, screenWidth * 0.035);

            ctx.font = `bold ${mainFontSize}px 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif`;
            ctx.fillText(mainText, screenWidth / 2, screenHeight / 2 - mainFontSize * 0.8);

            ctx.font = `${subFontSize}px 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif`;
            ctx.fillText(subText1, screenWidth / 2, screenHeight / 2 + subFontSize);
            if (subText2) {
                ctx.fillText(subText2, screenWidth / 2, screenHeight / 2 + subFontSize * 2.5);
            }
        }

        // --- Game Loop & State ---
        let lastTime = 0;
        function gameLoop(currentTime) {
            if (!gameRunning) {
                if (lives <= 0) drawMessages("Game Over!", `Final Score: ${score}`, "Tap to Restart");
                return;
            }
            animationFrameId = requestAnimationFrame(gameLoop);

            const deltaTime = currentTime - lastTime;
            lastTime = currentTime;

            // Beat tracking
            if (currentTime - lastBeatTime >= BEAT_INTERVAL) {
                lastBeatTime = currentTime - ((currentTime - lastBeatTime) % BEAT_INTERVAL); // Align to beat grid
                beatCount++;
                spawnItem(currentTime); // Spawn items on the beat
                // Potentially increase difficulty on certain beat counts
                if (beatCount > 0 && beatCount % 20 === 0) { // Every 20 beats
                    // SCANNER_SPEED_PIXELS_PER_MS *= 1.05; // Example: increase speed
                    // ITEM_SPAWN_CHANCE_PER_BEAT = Math.min(0.9, ITEM_SPAWN_CHANCE_PER_BEAT + 0.05);
                }
            }

            updateScanner(deltaTime);
            updateItems(currentTime, deltaTime);
            // checkCollisions() is integrated into handleTap for this game type

            draw(); // Draw all game elements
            drawTapFeedback(currentTime);
        }

        function draw() {
            ctx.clearRect(0, 0, screenWidth, screenHeight); // Ensure clean slate
            drawBackground();
            drawScanner();
            drawItems(performance.now()); // Use current time for item animations
            drawUI();
        }

        function startGame() {
            score = 0;
            combo = 0;
            lives = INITIAL_LIVES;
            items = [];
            tapFeedback = [];
            beatCount = 0;
            // Reset scanner speed and item spawn chance if they change during game
            // SCANNER_SPEED_PIXELS_PER_MS = 0.15;
            // ITEM_SPAWN_CHANCE_PER_BEAT = 0.6;


            scannerX = SCANNER_WIDTH / 2;
            scannerDirection = 1;

            lastBeatTime = performance.now();
            lastTime = performance.now();
            gameRunning = true;

            if (animationFrameId) cancelAnimationFrame(animationFrameId);
            gameLoop(performance.now());
        }

        function gameOver() {
            gameRunning = false;
            if (animationFrameId) cancelAnimationFrame(animationFrameId);

            if (score > highScore) {
                highScore = score;
                localStorage.setItem('evidenceLockdownHighScore', highScore);
            }
            // Game over message is drawn in the gameLoop's !gameRunning check or by resizeCanvas
            draw(); // Final draw of game state
            drawMessages("Game Over!", `Final Score: ${score}`, "Tap to Restart");
        }

        // --- Initialization ---
        function init() {
            resizeCanvas(); // Set initial size and draw start message
            window.addEventListener('resize', resizeCanvas);
            canvas.addEventListener('mousedown', handleTap);
            canvas.addEventListener('touchstart', handleTap, { passive: false });

            // Initial "Tap to Start" message
            drawMessages("Evidence Lockdown", "Tap Screen to Start Scan", `High Score: ${highScore}`);
        }

        init();
    </script>
</body>
</html>
