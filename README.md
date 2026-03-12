<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Luffy: Skypiea Endless Run</title>
    <style>
        body { margin: 0; overflow: hidden; background: #87CEEB; font-family: 'Courier New', Courier, monospace; }
        canvas { display: block; margin: 0 auto; background: linear-gradient(#87CEEB, #E0F7FA); }
        #ui { position: absolute; top: 20px; left: 20px; color: #FFF; text-shadow: 2px 2px #000; pointer-events: none; }
        #msg { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); color: gold; font-size: 30px; text-align: center; display: none; }
    </style>
</head>
<body>

    <div id="ui">
        <h1>SHANDORA GOLD: <span id="score">0</span></h1>
        <h3>HAKI METER: <span id="haki">0</span>%</h3>
        <p>Use Arrows: Left/Right to turn, Up to Jump, Down to Slide</p>
    </div>

    <div id="msg" id="deathScreen">
        GAME OVER<br>ENEL CAUGHT YOU!<br>
        <button onclick="location.reload()" style="padding: 10px 20px; font-size: 20px; cursor: pointer;">RESTART</button>
    </div>

    <canvas id="gameCanvas"></canvas>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const scoreEl = document.getElementById('score');
const hakiEl = document.getElementById('haki');
const deathScreen = document.getElementById('msg');

canvas.width = 600;
canvas.height = 800;

// Game Constants
const LANES = [100, 300, 500];
let gameSpeed = 7;
let score = 0;
let haki = 0;
let frames = 0;
let isGameOver = false;

// Player Object
const luffy = {
    lane: 1, // Start in Middle
    x: LANES[1],
    y: 700,
    width: 50,
    height: 80,
    isJumping: false,
    isSliding: false,
    jumpY: 0,
    hakiActive: false,
    
    draw() {
        ctx.save();
        ctx.translate(this.x, this.y + this.jumpY);

        // Haki Aura
        if (this.hakiActive) {
            ctx.shadowBlur = 20;
            ctx.shadowColor = "purple";
            ctx.strokeStyle = "purple";
            ctx.lineWidth = 5;
            ctx.strokeRect(-this.width/2, -this.height, this.width, this.height);
        }

        // Draw Luffy (Simplified Red Vest / Blue Shorts)
        ctx.fillStyle = "#FF0000"; // Red Vest
        let drawHeight = this.isSliding ? this.height/2 : this.height;
        let drawY = this.isSliding ? -this.height/2 : -this.height;
        ctx.fillRect(-this.width/2, drawY, this.width, drawHeight);
        
        // Head / Straw Hat
        ctx.fillStyle = "#F5DEB3";
        ctx.fillRect(-15, drawY - 30, 30, 30);
        ctx.fillStyle = "yellow"; // Straw Hat
        ctx.fillRect(-25, drawY - 35, 50, 10);
        
        ctx.restore();
    },

    update() {
        // Smooth lane movement
        let targetX = LANES[this.lane];
        this.x += (targetX - this.x) * 0.2;

        // Jump Logic
        if (this.isJumping) {
            this.jumpY -= 8;
            if (this.jumpY < -150) this.isJumping = false;
        } else if (this.jumpY < 0) {
            this.jumpY += 8;
        }
    }
};

// Obstacles & Gold
let obstacles = [];
class Obstacle {
    constructor(type) {
        this.lane = Math.floor(Math.random() * 3);
        this.x = LANES[this.lane];
        this.y = -100;
        this.type = type; // 'bolt', 'log', 'gold'
        this.width = 60;
        this.height = 60;
    }

    draw() {
        ctx.save();
        if (this.type === 'bolt') {
            ctx.fillStyle = "cyan"; // Enel's Lightning
            ctx.beginPath();
            ctx.moveTo(this.x, this.y);
            ctx.lineTo(this.x-20, this.y+30);
            ctx.lineTo(this.x, this.y+30);
            ctx.lineTo(this.x-10, this.y+60);
            ctx.fill();
        } else if (this.type === 'gold') {
            ctx.fillStyle = "gold";
            ctx.beginPath();
            ctx.arc(this.x, this.y, 15, 0, Math.PI*2);
            ctx.fill();
        } else {
            ctx.fillStyle = "#8B4513"; // Fallen Log
            ctx.fillRect(this.x - 30, this.y, 60, 30);
        }
        ctx.restore();
    }

    update() {
        this.y += gameSpeed;
    }
}

// Control Logic
window.addEventListener('keydown', (e) => {
    if (isGameOver) return;
    if (e.key === "ArrowLeft" && luffy.lane > 0) luffy.lane--;
    if (e.key === "ArrowRight" && luffy.lane < 2) luffy.lane++;
    if (e.key === "ArrowUp" && !luffy.isJumping) luffy.isJumping = true;
    if (e.key === "ArrowDown") {
        luffy.isSliding = true;
        setTimeout(() => luffy.isSliding = false, 500);
    }
    if (e.key === " " && haki >= 100) {
        activateHaki();
    }
});

function activateHaki() {
    haki = 0;
    luffy.hakiActive = true;
    gameSpeed += 5;
    setTimeout(() => {
        luffy.hakiActive = false;
        gameSpeed -= 5;
    }, 5000);
}

function spawnObstacles() {
    if (frames % 60 === 0) {
        let r = Math.random();
        if (r < 0.7) obstacles.push(new Obstacle('gold'));
        else if (r < 0.85) obstacles.push(new Obstacle('bolt'));
        else obstacles.push(new Obstacle('log'));
    }
}

function checkCollision(obj) {
    let dx = Math.abs(luffy.x - obj.x);
    let dy = Math.abs((luffy.y + luffy.jumpY) - obj.y);

    if (dx < 40 && dy < 50) {
        if (obj.type === 'gold') {
            score += 10;
            if (haki < 100) haki += 5;
            return true; 
        } else {
            if (luffy.hakiActive) return true; // Invincible
            if (obj.type === 'bolt' && luffy.isSliding) return false; // Can't slide under lightning
            if (obj.type === 'log' && luffy.isJumping) return false; // Can jump over logs
            
            isGameOver = true;
            deathScreen.style.display = 'block';
        }
    }
    return false;
}

function animate() {
    if (isGameOver) return;
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    
    // Draw Background Lanes
    ctx.fillStyle = "rgba(255, 255, 255, 0.3)";
    LANES.forEach(lx => ctx.fillRect(lx-50, 0, 100, canvas.height));

    frames++;
    gameSpeed += 0.001; // Increase difficulty
    
    luffy.update();
    luffy.draw();

    spawnObstacles();

    obstacles.forEach((obs, index) => {
        obs.update();
        obs.draw();
        if (checkCollision(obs) || obs.y > 900) {
            obstacles.splice(index, 1);
        }
    });

    scoreEl.innerText = score;
    hakiEl.innerText = haki;

    requestAnimationFrame(animate);
}

animate();
</script>
</body>
</html>
