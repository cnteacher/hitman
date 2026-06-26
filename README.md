<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
<title>棒打小人打水漂</title>
<style>
*{margin:0;padding:0;box-sizing:border-box;-webkit-tap-highlight-color:transparent;}
html,body{width:100vw;height:100vh;overflow:hidden;background:#000;display:flex;justify-content:center;align-items:center;}
#gameWrap{position:relative;width:100vw;height:100vh;max-width:100vh;max-height:100vw;transform-origin:center;}
#gameCanvas{display:block;width:100%;height:100%;}
.ui{position:absolute;color:#fff;font-family:system-ui,sans-serif;text-shadow:0 0 4px #000;}
.scoreBox{top:12px;left:12px;font-size:18px;line-height:1.6;}
.curDis{font-size:24px;font-weight:bold;color:#fff;}
.maxDis{color:#ccc;font-size:16px;}
.resetBtn{top:12px;right:12px;width:44px;height:44px;border-radius:50%;background:rgba(255,255,255,0.2);border:2px solid #fff;color:#fff;font-size:20px;cursor:pointer;display:flex;align-items:center;justify-content:center;}
.muteBtn{top:64px;right:12px;width:44px;height:44px;border-radius:50%;background:rgba(255,255,255,0.2);border:2px solid #fff;color:#fff;font-size:16px;cursor:pointer;display:flex;align-items:center;justify-content:center;}
.tipBox{position:absolute;top:40%;left:50%;transform:translate(-50%,-50%);background:rgba(0,0,0,0.6);padding:16px 24px;border-radius:12px;color:#fff;font-size:16px;white-space:nowrap;display:none;}
.recordTip{position:absolute;top:50%;left:50%;transform:translate(-50%,-50%);background:gold;color:#000;padding:20px 30px;border-radius:12px;font-size:22px;font-weight:bold;display:none;box-shadow:0 0 12px gold;}
</style>
</head>
<body>
<div id="gameWrap">
    <canvas id="gameCanvas"></canvas>
    <div class="ui scoreBox">
        <div class="curDis">当前：0.00 m</div>
        <div class="maxDis">最高纪录：0.00 m</div>
    </div>
    <button class="ui resetBtn" id="resetBtn">↻</button>
    <button class="ui muteBtn" id="muteBtn">🔊</button>
    <div class="tipBox" id="tipBox">点击屏幕击打小人，飞得越远分数越高</div>
    <div class="recordTip" id="recordTip">恭喜刷新最远纪录！</div>
</div>

<script>
// ====================== 全局可调参数（拓展预留接口）======================
const CONFIG = {
    basePower: 18,          // 击打基础力度
    gravity: 0.32,          // 重力系数
    airDrag: 0.008,         // 空气阻力
    groundFriction: 0.035,  // 地面滑行摩擦力
    bounceDamp: 0.62,       // 弹跳衰减系数（打水漂多次弹跳）
    springMinBoost: 0.15,   // 弹簧最小增益
    springMaxBoost: 0.30,   // 弹簧最大增益
    springMaxCount: 3,      // 单局最多弹簧数量
    springSpawnRate: 0.35,  // 每2秒生成概率
    armSwingTime: 200,      // 挥棒动画时长ms
    hitShockMs: 100,        // 击打震动
    landShockMs: 150,       // 落地震动
    springShockMs: 80,      // 弹簧震动
    recordShock: [100,50,100],// 破纪录震动
    fpsMinLimit: 20         // 最低帧率阈值，低于则降特效
};

// 画布与DOM
const wrap = document.getElementById('gameWrap');
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const tipBox = document.getElementById('tipBox');
const recordTip = document.getElementById('recordTip');
const resetBtn = document.getElementById('resetBtn');
const muteBtn = document.getElementById('muteBtn');
const curDisDom = document.querySelector('.curDis');
const maxDisDom = document.querySelector('.maxDis');

// 全局状态
let W, H, groundY;
let gameState = 'ready'; // ready/hit/fly/slide/end
let mute = false;
let lastTime = 0;
let frameCount = 0;
let fpsTime = 0;
let fps = 60;
let showTip = true;
let tipTimer = null;

// 小人数据
const player = {
    x: 0, y: 0, vx: 0, vy: 0,
    w: 60, h: 80,
    rot: 0, rotSpeed: 0,
    isAir: true, slideAnim: 0, bounceCount: 0
};
// 手臂球棒
const batArm = {
    baseX: 0, baseY: 0, swingAngle: 0, maxAngle: 75, swinging: false, swingStart: 0
};
// 弹簧数组
let springs = [];
// 粒子特效
let particles = [];
// 计分
let originX = 0;
let currentDist = 0;
let maxRecord = parseFloat(localStorage.getItem('hitManMax') || '0.00');

// 音频简易音效（Web Audio）
let audioCtx = null;
function initAudio(){
    if(audioCtx) return;
    audioCtx = new (window.AudioContext || window.webkitAudioContext)();
}
function playSound(freq, dur, vol=0.3){
    if(mute || !audioCtx) return;
    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    osc.connect(gain);
    gain.connect(audioCtx.destination);
    osc.frequency.value = freq;
    gain.gain.setValueAtTime(vol, audioCtx.currentTime);
    gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + dur);
    osc.start();
    osc.stop(audioCtx.currentTime + dur);
}
function vibrate(msArr){
    if(!navigator.vibrate || mute) return;
    navigator.vibrate(msArr);
}

// 屏幕适配 强制横屏16:9
function resizeCanvas(){
    const ratio = 16 / 9;
    let vw = window.innerWidth;
    let vh = window.innerHeight;
    if(vw / vh > ratio){
        wrap.style.height = '100vh';
        wrap.style.width = (vh * ratio) + 'px';
    }else{
        wrap.style.width = '100vw';
        wrap.style.height = (vw / ratio) + 'px';
    }
    const rect = wrap.getBoundingClientRect();
    canvas.width = rect.width;
    canvas.height = rect.height;
    W = canvas.width;
    H = canvas.height;
    groundY = H * 0.72;
    resetGame();
}

// 重置整局游戏
function resetGame(){
    gameState = 'ready';
    springs = [];
    particles = [];
    // 小人初始位置 画布85%宽度地面
    originX = W * 0.85;
    player.x = originX;
    player.y = groundY - player.h;
    player.vx = 0; player.vy = 0;
    player.rot = 0; player.rotSpeed = 0;
    player.isAir = true;
    player.bounceCount = 0;
    // 球棒复位
    batArm.baseX = player.x + player.w;
    batArm.baseY = player.y + player.h * 0.4;
    batArm.swingAngle = 0;
    batArm.swinging = false;
    // 计分重置
    currentDist = 0;
    updateScoreUI();
    // 提示弹窗
    showTip = true;
    tipBox.style.display = 'block';
    clearTimeout(tipTimer);
    tipTimer = setTimeout(()=>{tipBox.style.display='none';},3000);
}

// 击打逻辑
function hitMan(){
    if(gameState !== 'ready') return;
    initAudio();
    gameState = 'hit';
    vibrate(CONFIG.hitShockMs);
    playSound(320,0.15,0.4);
    // 挥棒动画
    batArm.swinging = true;
    batArm.swingStart = performance.now();
    // 发射初速度 左上方30°
    const rad = Math.PI * 30 / 180;
    player.vx = -CONFIG.basePower * Math.cos(rad);
    player.vy = -CONFIG.basePower * Math.sin(rad);
    setTimeout(()=>{
        gameState = 'fly';
        spawnSpringLoop();
    }, CONFIG.armSwingTime);
}

// 定时生成弹簧
function spawnSpringLoop(){
    if(gameState === 'end') return;
    setTimeout(()=>{
        if(gameState === 'end') return;
        // 概率生成
        if(Math.random() < CONFIG.springSpawnRate && springs.length < CONFIG.springMaxCount){
            const sx = W * 0.08 + Math.random() * (originX - W * 0.12);
            springs.push({
                x: sx, y: groundY - 10, w:50, h:40, used:false
            });
        }
        spawnSpringLoop();
    }, 2000);
}

// 生成尘土粒子
function spawnDust(x,y){
    const count = fps < CONFIG.fpsMinLimit ? 4 : 10;
    for(let i=0;i<count;i++){
        particles.push({
            x:x, y:y,
            vx: (Math.random()-0.5)*3,
            vy: -Math.random()*2,
            life: 30,
            size: 2+Math.random()*4
        });
    }
}

// 更新物理每一帧
function updatePhysics(dt){
    if(gameState === 'ready' || gameState === 'end') return;
    const p = player;
    // 空中逻辑
    if(p.isAir){
        // 重力
        p.vy += CONFIG.gravity;
        // 空气阻力
        p.vx *= (1 - CONFIG.airDrag);
        // 旋转
        p.rotSpeed += p.vy * 0.002;
        p.rot += p.rotSpeed * dt;
        // 落地判定
        if(p.y + p.h >= groundY){
            vibrate(CONFIG.landShockMs);
            playSound(120,0.2,0.3);
            spawnDust(p.x + p.w/2, groundY);
            p.y = groundY - p.h;
            // 打水漂多次弹跳
            if(Math.abs(p.vy) > 2.5){
                p.vy = -p.vy * CONFIG.bounceDamp;
                p.bounceCount++;
            }else{
                // 弹跳力度不足，进入滑行
                p.isAir = false;
                p.vy = 0;
                gameState = 'slide';
            }
        }
        // 左边界碰撞
        if(p.x <= 0){
            p.x = 0;
            p.vy += 2;
            p.vx *= 0.4;
        }
    }else{
        // 地面滑行
        p.vx *= (1 - CONFIG.groundFriction);
        p.slideAnim += dt * 0.008;
        // 速度归零结算
        if(Math.abs(p.vx) < 0.05){
            p.vx = 0;
            settleGame();
        }
    }
    // 更新位置
    p.x += p.vx;
    p.y += p.vy;
    // 实时距离
    currentDist = Math.abs(originX - p.x) / 10;
    updateScoreUI();

    // 弹簧碰撞检测
    for(let s of springs){
        if(s.used) continue;
        const hitX = (p.x + p.w) > s.x && p.x < s.x + s.w;
        const hitY = (p.y + p.h) > s.y && p.y < s.y + s.h;
        if(hitX && hitY){
            s.used = true;
            vibrate(CONFIG.springShockMs);
            playSound(480,0.2,0.35);
            const boost = 1 + CONFIG.springMinBoost + Math.random()*(CONFIG.springMaxBoost - CONFIG.springMinBoost);
            if(p.isAir){
                p.vx *= boost;
                p.vy = -Math.abs(p.vy) * boost * 0.8;
            }else{
                p.vx *= boost;
            }
            spawnDust(s.x + s.w/2, s.y);
        }
    }

    // 更新粒子
    for(let i=particles.length-1;i>=0;i--){
        const pt = particles[i];
        pt.x += pt.vx;
        pt.y += pt.vy;
        pt.vy += 0.1;
        pt.life--;
        if(pt.life <=0) particles.splice(i,1);
    }

    // 挥棒动画更新
    if(batArm.swinging){
        const pass = performance.now() - batArm.swingStart;
        const rate = Math.min(1, pass / CONFIG.armSwingTime);
        batArm.swingAngle = batArm.maxAngle * (1 - Math.cos(rate * Math.PI));
        if(rate >= 1) batArm.swinging = false;
    }
}

// 结算对局
function settleGame(){
    gameState = 'end';
    const finalDis = currentDist.toFixed(2);
    if(parseFloat(finalDis) > maxRecord){
        maxRecord = parseFloat(finalDis);
        localStorage.setItem('hitManMax', maxRecord.toFixed(2));
        // 破纪录提示
        vibrate(CONFIG.recordShock);
        playSound(600,0.4,0.4);
        recordTip.style.display = 'block';
        setTimeout(()=>recordTip.style.display='none',2000);
    }
    updateScoreUI();
}

// 更新分数UI
function updateScoreUI(){
    curDisDom.innerText = `当前：${currentDist.toFixed(2)} m`;
    maxDisDom.innerText = `最高纪录：${maxRecord.toFixed(2)} m`;
}

// 渲染画面
function render(){
    // 渐变背景 天空浅绿→地面深绿
    const skyGrad = ctx.createLinearGradient(0,0,0,groundY);
    skyGrad.addColorStop(0,'#b8e8b8');
    skyGrad.addColorStop(1,'#7ccd7c');
    ctx.fillStyle = skyGrad;
    ctx.fillRect(0,0,W,groundY);
    // 地面层
    const groundGrad = ctx.createLinearGradient(0,groundY,0,H);
    groundGrad.addColorStop(0,'#58a858');
    groundGrad.addColorStop(1,'#3d7d3d');
    ctx.fillStyle = groundGrad;
    ctx.fillRect(0,groundY,W,H-groundY);

    // 绘制埋地弹簧（与地面齐平，仅顶部露出）
    for(let s of springs){
        ctx.save();
        ctx.translate(s.x + s.w/2, s.y + s.h);
        if(s.used) ctx.fillStyle = '#999';
        else ctx.fillStyle = '#4fc14f';
        // 弹簧主体埋地下
        ctx.fillRect(-s.w/2, 5, s.w, s.h - 10);
        // 顶部露出弹簧头
        ctx.fillRect(-s.w/2, 0, s.w, 10);
        ctx.restore();
    }

    // 绘制小人
    const p = player;
    ctx.save();
    ctx.translate(p.x + p.w/2, p.y + p.h/2);
    if(p.isAir) ctx.rotate(p.rot);
    // 身体
    ctx.fillStyle = '#ff8866';
    ctx.fillRect(-p.w/2, -p.h/2, p.w, p.h);
    // 头部
    ctx.beginPath();
    ctx.arc(0, -p.h/2 + 12, 18, 0, Math.PI*2);
    ctx.fillStyle = '#ffbc99';
    ctx.fill();
    ctx.restore();

    // 绘制手臂+棒球棒
    ctx.save();
    ctx.translate(batArm.baseX, batArm.baseY);
    ctx.rotate(-batArm.swingAngle * Math.PI / 180);
    // 手臂
    ctx.fillStyle = '#ffbc99';
    ctx.fillRect(0,-6,36,12);
    // 球棒 长120
    ctx.fillStyle = '#8b5a2b';
    ctx.fillRect(36,-5,120,10);
    ctx.fillStyle = '#fff';
    ctx.fillRect(146,-7,14,14);
    ctx.restore();

    // 尘土粒子
    for(let pt of particles){
        ctx.globalAlpha = pt.life / 30;
        ctx.fillStyle = '#e6d9b8';
        ctx.fillRect(pt.x, pt.y, pt.size, pt.size);
    }
    ctx.globalAlpha = 1;
}

// 主游戏循环
function gameLoop(timestamp){
    const dt = timestamp - lastTime;
    lastTime = timestamp;
    // 帧率统计
    frameCount++;
    fpsTime += dt;
    if(fpsTime > 1000){
        fps = frameCount;
        frameCount = 0;
        fpsTime = 0;
    }
    updatePhysics(dt);
    render();
    requestAnimationFrame(gameLoop);
}

// 事件绑定
canvas.addEventListener('touchstart',(e)=>{
    e.preventDefault();
    hitMan();
});
canvas.addEventListener('mousedown',()=>hitMan());
resetBtn.addEventListener('click',resetGame);
muteBtn.addEventListener('click',()=>{
    mute = !mute;
    muteBtn.innerText = mute ? '🔇' : '🔊';
});
// 长按3秒清空最高纪录
let longPressTimer = null;
maxDisDom.addEventListener('mousedown',()=>{
    longPressTimer = setTimeout(()=>{
        if(confirm('确认清空历史最高纪录？')){
            maxRecord = 0;
            localStorage.setItem('hitManMax','0.00');
            updateScoreUI();
        }
    },3000);
});
maxDisDom.addEventListener('mouseup',()=>clearTimeout(longPressTimer));
maxDisDom.addEventListener('mouseleave',()=>clearTimeout(longPressTimer));

// 窗口适配
window.addEventListener('resize',resizeCanvas);
window.addEventListener('orientationchange',resizeCanvas);
resizeCanvas();
requestAnimationFrame(gameLoop);
</script>
</body>
</html>
