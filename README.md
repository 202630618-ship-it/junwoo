<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>할로우 나이트 미니 웹게임</title>
    <style>
        body {
            margin: 0;
            background-color: #111;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            color: white;
            font-family: Arial, sans-serif;
            overflow: hidden;
        }
        #gameContainer {
            position: relative;
        }
        canvas {
            border: 4px solid #333;
            background: linear-gradient(to bottom, #1a1a2e, #16161d);
            box-shadow: 0 0 20px rgba(0,0,0,0.8);
        }
        #ui {
            position: absolute;
            top: 10px;
            left: 10px;
            font-size: 16px;
            text-shadow: 2px 2px 4px #000;
        }
        #bossUi {
            position: absolute;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            width: 400px;
            display: none;
        }
        .hp-bar {
            width: 100%;
            height: 15px;
            background-color: #333;
            border: 2px solid #fff;
        }
        .hp-fill {
            width: 100%;
            height: 100%;
            background-color: #e63946;
            transition: width 0.1s;
        }
        #bossName {
            text-align: center;
            margin-bottom: 5px;
            font-weight: bold;
            font-size: 14px;
            letter-spacing: 2px;
        }
        #controls {
            position: absolute;
            bottom: -40px;
            left: 0;
            width: 100%;
            text-align: center;
            font-size: 12px;
            color: #888;
        }
    </style>
</head>
<body>

<div id="gameContainer">
    <div id="ui">
        <div>가면 (HP): <span id="playerHp">★★★★★</span></div>
        <div>잡몹 처치: <span id="score">0</span></div>
    </div>
    
    <div id="bossUi">
        <div id="bossName">호넷 (HORNET)</div>
        <div class="hp-bar"><div id="bossHpFill" class="hp-fill"></div></div>
    </div>

    <canvas id="gameCanvas" width="800" height="450"></canvas>
    
    <div id="controls">
        이동: 방향키(←, →) | 점프: Z | 칼날 공격: X
    </div>
</div>

<script>
const canvas = document.getElementById("gameCanvas");
const ctx = canvas.getContext("2d");

// 게임 상태
let score = 0;
let gameOver = false;
let gameWon = false;

// 키 입력 상태
const keys = { Left: false, Right: false, z: false, x: false };

// 플레이어 설정
const player = {
    x: 100, y: 300, width: 30, height: 45,
    vx: 0, vy: 0, speed: 4, jumpForce: 11,
    grounded: false, hp: 5, maxHp: 5,
    direction: "right",
    isAttacking: false, attackTimer: 0, attackCooldown: 0,
    isInvincible: false, invincibleTimer: 0
};

// 칼날 공격 히트박스
const attackBox = { x: 0, y: 0, width: 50, height: 40 };

// 잡몹 (기어다니는 크롤러)
const enemies = [
    { x: 300, y: 365, width: 30, height: 20, speed: 1.5, direction: 1, hp: 1 },
    { x: 600, y: 365, width: 30, height: 20, speed: -1.5, direction: -1, hp: 1 }
];

// 보스 (호넷)
const hornet = {
    x: 650, y: 300, width: 35, height: 55,
    vx: 0, vy: 0, hp: 15, maxHp: 15,
    state: "idle", stateTimer: 60,
    direction: -1, speed: 5, active: false
};

// 지형 (바닥)
const groundY = 385;

// 키 이벤트 리스너
window.addEventListener("keydown", (e) => {
    if (e.key === "ArrowLeft") keys.Left = true;
    if (e.key === "ArrowRight") keys.Right = true;
    if (e.key.toLowerCase() === "z") keys.z = true;
    if (e.key.toLowerCase() === "x") handleAttack();
});

window.addEventListener("keyup", (e) => {
    if (e.key === "ArrowLeft") keys.Left = false;
    if (e.key === "ArrowRight") keys.Right = false;
    if (e.key.toLowerCase() === "z") keys.z = false;
});

function handleAttack() {
    if (player.attackCooldown <= 0 && !player.isAttacking) {
        player.isAttacking = true;
        player.attackTimer = 10; // 공격 지속 프레임
        player.attackCooldown = 25; // 다음 공격까지 딜레이
    }
}

// 충돌 체크 함수
function checkCollision(rect1, rect2) {
    return rect1.x < rect2.x + rect2.width &&
           rect1.x + rect1.width > rect2.x &&
           rect1.y < rect2.y + rect2.height &&
           rect1.y + rect1.height > rect2.y;
}

// 업데이트 루프
function update() {
    if (gameOver || gameWon) return;

    // --- 플레이어 이동 및 중력 ---
    if (keys.Left) {
        player.vx = -player.speed;
        player.direction = "left";
    } else if (keys.Right) {
        player.vx = player.speed;
        player.direction = "right";
    } else {
        player.vx = 0;
    }

    // 점프
    if (keys.z && player.grounded) {
        player.vy = -player.jumpForce;
        player.grounded = false;
    }

    // 중력 적용
    player.vy += 0.5;
    player.x += player.vx;
    player.y += player.vy;

    // 화면 벽 제한
    if (player.x < 0) player.x = 0;
    if (player.x > canvas.width - player.width) player.x = canvas.width - player.width;

    // 바닥 충돌
    if (player.y + player.height >= groundY) {
        player.y = groundY - player.height;
        player.vy = 0;
        player.grounded = true;
    }

    // 타이머 감소
    if (player.attackCooldown > 0) player.attackCooldown--;
    if (player.invincibleTimer > 0) player.invincibleTimer--;
    if (player.invincibleTimer === 0) player.isInvincible = false;

    // --- 공격 히트박스 위치 설정 ---
    if (player.isAttacking) {
        player.attackTimer--;
        if (player.direction === "right") {
            attackBox.x = player.x + player.width;
        } else {
            attackBox.x = player.x - attackBox.width;
        }
        attackBox.y = player.y + 5;

        if (player.attackTimer <= 0) {
            player.isAttacking = false;
        }
    }

    // --- 잡몹 업데이트 ---
    enemies.forEach((enemy, index) => {
        enemy.x += enemy.speed;
        // 벽 반사
        if (enemy.x < 0 || enemy.x > canvas.width - enemy.width) {
            enemy.speed *= -1;
        }

        // 플레이어 공격 범위 충돌
        if (player.isAttacking && checkCollision(attackBox, enemy)) {
            enemies.splice(index, 1);
            score++;
            document.getElementById("score").innerText = score;
            // 피격 리코일 효과 (플레이어 살짝 밀려남)
            player.vx = player.direction === "right" ? -3 : 3;
        }

        // 플레이어 몸체 충돌
        if (checkCollision(player, enemy) && !player.isInvincible) {
            playerHit();
        }
    });

    // 일정 점수(2마리 처치) 시 보스 등장
    if (score >= 2 && !hornet.active) {
        hornet.active = true;
        document.getElementById("bossUi").style.display = "block";
    }

    // --- 보스 (호넷) AI 업데이트 ---
    if (hornet.active) {
        hornet.stateTimer--;

        // 방향 전환
        hornet.direction = hornet.x > player.x ? -1 : 1;

        if (hornet.stateTimer <= 0) {
            // 무작위 패턴 전환
            const r = Math.random();
            if (r < 0.4) {
                hornet.state = "dash"; // 돌진 공격
                hornet.vx = hornet.direction * hornet.speed * 2;
                hornet.stateTimer = 30;
            } else if (r < 0.8) {
                hornet.state = "jump"; // 공중 도약
                hornet.vy = -12;
                hornet.vx = hornet.direction * hornet.speed;
                hornet.stateTimer = 50;
            } else {
                hornet.state = "idle";
                hornet.vx = 0;
                hornet.stateTimer = 40;
            }
        }

        // 중력 및 이동 적용
        if (hornet.state === "jump") {
            hornet.vy += 0.4;
        }
        hornet.x += hornet.vx;
        hornet.y += hornet.vy;

        // 보스 바닥 제한
        if (hornet.y + hornet.height >= groundY) {
            hornet.y = groundY - hornet.height;
            hornet.vy = 0;
            if (hornet.state === "jump") hornet.vx = 0;
        }
        if (hornet.x < 0) hornet.x = 0;
        if (hornet.x > canvas.width - hornet.width) hornet.x = canvas.width - hornet.width;

        // 보스가 플레이어 공격에 맞음
        if (player.isAttacking && checkCollision(attackBox, hornet) && player.attackTimer === 9) {
            hornet.hp--;
            document.getElementById("bossHpFill").style.width = (hornet.hp / hornet.maxHp) * 100 + "%";
            if (hornet.hp <= 0) {
                gameWon = true;
            }
        }

        // 보스가 플레이어를 타격
        if (checkCollision(player, hornet) && !player.isInvincible) {
            playerHit();
        }
    }
}

function playerHit() {
    player.hp--;
    player.isInvincible = true;
    player.invincibleTimer = 60; // 1초 무적
    
    let hpStr = "";
    for(let i=0; i<player.maxHp; i++) {
        hpStr += i < player.hp ? "★" : "☆";
    }
    document.getElementById("playerHp").innerText = hpStr;

    if (player.hp <= 0) {
        gameOver = true;
    }
}

// 그리기 루프
function draw() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // 바닥 그리기
    ctx.fillStyle = "#22252a";
    ctx.fillRect(0, groundY, canvas.width, canvas.height - groundY);
    ctx.strokeStyle = "#444";
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(0, groundY);
    ctx.lineTo(canvas.width, groundY);
    ctx.stroke();

    // 잡몹 그리기
    ctx.fillStyle = "#8d99ae";
    enemies.forEach(enemy => {
        ctx.fillRect(enemy.x, enemy.y, enemy.width, enemy.height);
    });

    // 보스 (호넷) 그리기
    if (hornet.active) {
        ctx.fillStyle = "#d90429"; // 호넷의 붉은 망토 색상
        ctx.fillRect(hornet.x, hornet.y, hornet.width, hornet.height);
        
        // 호넷의 바늘(칼날) 시각 효과 (돌진 시)
        if (hornet.state === "dash") {
            ctx.strokeStyle = "#edf2f4";
            ctx.lineWidth = 3;
            ctx.beginPath();
            ctx.moveTo(hornet.x + hornet.width/2, hornet.y + hornet.height/2);
            ctx.lineTo(hornet.x + (hornet.direction * 60), hornet.y + hornet.height/2);
            ctx.stroke();
        }
    }

    // 플레이어 그리기 (무적 상태일 때 깜빡임)
    if (!player.isInvincible || Math.floor(player.invincibleTimer / 4) % 2 === 0) {
