
```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<title>网页射击游戏</title>
<style>
body, html { margin:0; padding:0; overflow:hidden; background:#111; font-family:sans-serif; }
canvas { display:block; background:#222; margin:0 auto; }
#ui {
    position:absolute; top:10px; left:50%; transform:translateX(-50%);
    color:#fff; font-size:18px; text-align:center; z-index:10;
}
#startScreen, #gameOverScreen {
    position:absolute; top:0; left:0; width:100%; height:100%;
    background:rgba(0,0,0,0.85); color:#fff; display:flex;
    flex-direction:column; justify-content:center; align-items:center;
    font-size:24px; z-index:20;
}
button { font-size:20px; padding:10px 20px; margin:5px; cursor:pointer; }
#pauseButton { position:absolute; top:10px; right:10px; z-index:10; }
</style>
</head>
<body>
<canvas id="gameCanvas" width="800" height="600"></canvas>
<div id="ui">
    分数: <span id="score">0</span> |
    连击: <span id="combo">0</span> |
    血量: <span id="hp">100</span> |
    时间: <span id="time">60</span>
</div>
<button id="pauseButton">暂停</button>
<div id="startScreen">
    <h1>网页射击游戏</h1>
    <button id="normalMode">普通闯关模式</button>
    <button id="timeMode">限时挑战模式</button>
</div>
<div id="gameOverScreen" style="display:none;">
    <h1>游戏结束</h1>
    <p>本局得分: <span id="finalScore">0</span></p>
    <p>历史最高分: <span id="highScore">0</span></p>
    <h3>排行榜</h3>
    <ol id="rankingList"></ol>
    <button id="restartBtn">重新开始</button>
</div>
<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const scoreEl = document.getElementById('score');
const comboEl = document.getElementById('combo');
const hpEl = document.getElementById('hp');
const timeEl = document.getElementById('time');
const startScreen = document.getElementById('startScreen');
const gameOverScreen = document.getElementById('gameOverScreen');
const finalScoreEl = document.getElementById('finalScore');
const highScoreEl = document.getElementById('highScore');
const rankingListEl = document.getElementById('rankingList');
const pauseButton = document.getElementById('pauseButton');

let gameMode = 'normal'; // normal | time
let running = false;
let paused = false;

// 玩家
const player = {x:400, y:300, size:20, speed:4, hp:100};
let keys = {};
let mouse = {x:0,y:0,down:false};

// 子弹
let bullets = [];
let bulletCooldown = 0;
let bulletLimit = 10;
let ammo = bulletLimit;

// 敌人
let enemies = [];
const enemyTypes = [
    {type:'normal', color:'green', hp:1, speed:1, score:10},
    {type:'elite', color:'orange', hp:2, speed:2, score:30},
    {type:'boss', color:'red', hp:5, speed:1, score:100}
];

// 游戏数据
let score = 0;
let combo = 0;
let timeLeft = 60;
let spawnTimer = 0;
let rankData = JSON.parse(localStorage.getItem('rankData')||'[]');

function startGame(mode){
    gameMode = mode;
    running = true;
    paused = false;
    player.hp = 100;
    player.x = canvas.width/2;
    player.y = canvas.height/2;
    bullets = [];
    enemies = [];
    score = 0;
    combo = 0;
    timeLeft = (mode==='time')?60:999;
    startScreen.style.display='none';
    gameOverScreen.style.display='none';
    requestAnimationFrame(gameLoop);
}

function endGame(){
    running = false;
    finalScoreEl.textContent = score;
    let highScore = Math.max(score, localStorage.getItem('highScore')||0);
    localStorage.setItem('highScore', highScore);
    highScoreEl.textContent = highScore;
    rankData.push(score);
    rankData.sort((a,b)=>b-a);
    if(rankData.length>10) rankData = rankData.slice(0,10);
    localStorage.setItem('rankData', JSON.stringify(rankData));
    rankingListEl.innerHTML = rankData.map(s=>`<li>${s}</li>`).join('');
    gameOverScreen.style.display='flex';
}

function spawnEnemy(){
    let type = enemyTypes[Math.floor(Math.random()*enemyTypes.length)];
    let edge = Math.floor(Math.random()*4);
    let x=0,y=0;
    if(edge===0){ x=0; y=Math.random()*canvas.height; }
    if(edge===1){ x=canvas.width; y=Math.random()*canvas.height; }
    if(edge===2){ x=Math.random()*canvas.width; y=0; }
    if(edge===3){ x=Math.random()*canvas.width; y=canvas.height; }
    enemies.push({x,y,size:20, ...type});
}

function update(){
    if(paused) return;
    // 玩家移动
    if(keys['w']) player.y -= player.speed;
    if(keys['s']) player.y += player.speed;
    if(keys['a']) player.x -= player.speed;
    if(keys['d']) player.x += player.speed;
    player.x = Math.max(0,Math.min(canvas.width,player.x));
    player.y = Math.max(0,Math.min(canvas.height,player.y));

    // 子弹发射
    if(mouse.down && bulletCooldown<=0 && ammo>0){
        let angle = Math.atan2(mouse.y-player.y, mouse.x-player.x);
        bullets.push({x:player.x, y:player.y, dx:Math.cos(angle)*8, dy:Math.sin(angle)*8});
        bulletCooldown = 10;
        ammo--;
    }
    if(bulletCooldown>0) bulletCooldown--;

    // 子弹移动
    bullets.forEach((b,i)=>{
        b.x += b.dx;
        b.y += b.dy;
        if(b.x<0 || b.x>canvas.width || b.y<0 || b.y>canvas.height){
            bullets.splice(i,1);
        }
    });

    // 敌人刷新
    spawnTimer++;
    if(spawnTimer>50){
        spawnEnemy();
        spawnTimer=0;
    }

    // 敌人移动
    enemies.forEach(e=>{
        let dx = player.x - e.x;
        let dy = player.y - e.y;
        let dist = Math.sqrt(dx*dx+dy*dy);
        e.x += dx/dist * e.speed;
        e.y += dy/dist * e.speed;
    });

    // 碰撞检测
    enemies.forEach((e,ei)=>{
        bullets.forEach((b,bi)=>{
            let dx = e.x - b.x;
            let dy = e.y - b.y;
            if(Math.sqrt(dx*dx+dy*dy) < e.size){
                e.hp--;
                bullets.splice(bi,1);
                if(e.hp<=0){
                    score += e.score;
                    combo++;
                    enemies.splice(ei,1);
                }
            }
        });
        let dx = e.x - player.x;
        let dy = e.y - player.y;
        if(Math.sqrt(dx*dx+dy*dy)<e.size+player.size){
            player.hp -= 1;
            combo = 0;
            if(player.hp<=0) endGame();
        }
    });

    // 自动回血子弹
    if(ammo<bulletLimit && frameCount%60===0) ammo++;

    // 时间倒计时
    if(gameMode==='time'){
        if(frameCount%60===0){
            timeLeft--;
            if(timeLeft<=0) endGame();
        }
    }

    // UI更新
    scoreEl.textContent = score;
    comboEl.textContent = combo;
    hpEl.textContent = player.hp;
    timeEl.textContent = timeLeft;
}

function draw(){
    ctx.clearRect(0,0,canvas.width,canvas.height);

    // 玩家
    ctx.fillStyle='blue';
    ctx.beginPath();
    ctx.arc(player.x,player.y,player.size,0,Math.PI*2);
    ctx.fill();

    // 子弹
    ctx.fillStyle='yellow';
    bullets.forEach(b=>{
        ctx.beginPath
