<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'

type Platform = { x: number; y: number; w: number; h: number }
type Coin = { x: number; y: number; taken: boolean }
type Spike = { x: number; y: number; w: number; h: number }
type Level = { name: string; subtitle: string; width: number; platforms: Platform[]; coins: Coin[]; spikes: Spike[]; goal: { x: number; y: number } }

const levels: Level[] = [
  {
    name: 'First Steps', subtitle: 'Learn the basics', width: 1900,
    platforms: [
      {x:0,y:440,w:340,h:80},{x:410,y:390,w:180,h:24},{x:660,y:330,w:170,h:24},
      {x:900,y:390,w:180,h:24},{x:1140,y:330,w:190,h:24},{x:1400,y:385,w:180,h:24},{x:1640,y:440,w:260,h:80}
    ],
    coins: [{x:245,y:390,taken:false},{x:480,y:345,taken:false},{x:725,y:285,taken:false},{x:965,y:345,taken:false},{x:1215,y:285,taken:false},{x:1470,y:340,taken:false}],
    spikes: [{x:280,y:414,w:36,h:26},{x:1000,y:364,w:36,h:26},{x:1515,y:359,w:36,h:26}],
    goal: {x:1800,y:370}
  },
  {
    name: 'Neon Run', subtitle: 'Mind the gaps', width: 2200,
    platforms: [
      {x:0,y:440,w:260,h:80},{x:340,y:380,w:140,h:24},{x:550,y:320,w:140,h:24},
      {x:770,y:365,w:125,h:24},{x:980,y:295,w:150,h:24},{x:1215,y:350,w:140,h:24},
      {x:1435,y:285,w:150,h:24},{x:1660,y:350,w:145,h:24},{x:1880,y:410,w:320,h:80}
    ],
    coins: [{x:190,y:390,taken:false},{x:395,y:335,taken:false},{x:605,y:275,taken:false},{x:815,y:320,taken:false},{x:1035,y:250,taken:false},{x:1260,y:305,taken:false},{x:1485,y:240,taken:false},{x:1710,y:305,taken:false}],
    spikes: [{x:195,y:414,w:35,h:26},{x:815,y:339,w:36,h:26},{x:1290,y:324,w:36,h:26},{x:1720,y:324,w:36,h:26}],
    goal: {x:2070,y:340}
  },
  {
    name: 'Moon Temple', subtitle: 'The final leap', width: 2450,
    platforms: [
      {x:0,y:440,w:250,h:80},{x:320,y:360,w:120,h:24},{x:510,y:300,w:120,h:24},
      {x:700,y:355,w:115,h:24},{x:890,y:270,w:130,h:24},{x:1100,y:330,w:120,h:24},
      {x:1295,y:260,w:125,h:24},{x:1500,y:325,w:120,h:24},{x:1710,y:250,w:130,h:24},
      {x:1925,y:330,w:130,h:24},{x:2140,y:420,w:310,h:70}
    ],
    coins: [{x:365,y:315,taken:false},{x:550,y:255,taken:false},{x:735,y:310,taken:false},{x:935,y:225,taken:false},{x:1140,y:285,taken:false},{x:1335,y:215,taken:false},{x:1540,y:280,taken:false},{x:1755,y:205,taken:false},{x:1970,y:285,taken:false}],
    spikes: [{x:175,y:414,w:36,h:26},{x:735,y:329,w:36,h:26},{x:1135,y:304,w:36,h:26},{x:1540,y:299,w:36,h:26},{x:1970,y:304,w:36,h:26}],
    goal: {x:2320,y:350}
  }
]

const canvas = ref<HTMLCanvasElement | null>(null)
const currentLevel = ref(0)
const playing = ref(false)
const finished = ref(false)
const paused = ref(false)
const score = ref(0)
const best = ref(Number(localStorage.getItem('moonhop-best') || 0))
const collected = ref(0)
const checkpointLabel = ref('START')
const soundOn = ref(false)
const level = computed(() => levels[currentLevel.value])
const totalCoins = computed(() => level.value.coins.length)
const progress = computed(() => Math.min(100, Math.round((player.x / level.value.width) * 100)))

const player = { x: 70, y: 370, w: 26, h: 34, vx: 0, vy: 0, grounded: false, spawnX: 70, spawnY: 370, face: 1 }
const camera = { x: 0 }
const keys = new Set<string>()
let raf = 0
let ctx: CanvasRenderingContext2D | null = null
let lastTime = 0
let elapsed = 0
let jumpQueued = false
let checkpointX = 70
const WORLD_H = 520
const GRAVITY = 1450
const SPEED = 280
const JUMP = 570

function resetLevel() {
  player.x = 70; player.y = 370; player.vx = 0; player.vy = 0
  player.spawnX = 70; player.spawnY = 370; player.grounded = false; player.face = 1
  camera.x = 0; score.value = 0; collected.value = 0; checkpointX = 70
  checkpointLabel.value = 'START'; finished.value = false; paused.value = false
  level.value.coins.forEach(c => c.taken = false)
}
function startGame() { resetLevel(); playing.value = true; lastTime = 0; }
function selectLevel(index: number) { currentLevel.value = index; startGame() }
function goMenu() { playing.value = false; finished.value = false; paused.value = false; }
function jump() { if (!playing.value) { startGame(); return }; if (paused.value) return; jumpQueued = true }
function setKey(key: string, down: boolean) {
  const k = key.toLowerCase()
  if (down) keys.add(k); else keys.delete(k)
  if (down && (k === ' ' || k === 'arrowup' || k === 'w')) { jumpQueued = true }
  if (down && k === 'escape') paused.value = !paused.value
  if (down && k === 'r' && playing.value) resetLevel()
}
function onKeyDown(e: KeyboardEvent) {
  if (['ArrowLeft','ArrowRight','ArrowUp',' ','w','a','d','W','A','D'].includes(e.key)) e.preventDefault()
  setKey(e.key, true)
}
function onKeyUp(e: KeyboardEvent) { setKey(e.key, false) }
function rectsOverlap(a: {x:number;y:number;w:number;h:number}, b: {x:number;y:number;w:number;h:number}) {
  return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + b.h && a.y + a.h > b.y
}
function update(dt: number) {
  if (!playing.value || paused.value || finished.value) return
  const l = level.value
  const left = keys.has('arrowleft') || keys.has('a')
  const right = keys.has('arrowright') || keys.has('d')
  player.vx = (right ? 1 : 0) - (left ? 1 : 0)
  if (player.vx) player.face = Math.sign(player.vx)
  player.vx *= SPEED
  if (jumpQueued && player.grounded) { player.vy = -JUMP; player.grounded = false }
  jumpQueued = false
  const oldY = player.y
  player.x += player.vx * dt
  player.x = Math.max(0, Math.min(l.width - player.w, player.x))
  player.vy += GRAVITY * dt
  player.y += player.vy * dt
  player.grounded = false
  for (const p of l.platforms) {
    const horizontal = player.x + player.w > p.x && player.x < p.x + p.w
    if (horizontal && oldY + player.h <= p.y && player.y + player.h >= p.y && player.vy >= 0) {
      player.y = p.y - player.h; player.vy = 0; player.grounded = true
    }
    if (horizontal && oldY >= p.y + p.h && player.y <= p.y + p.h && player.vy < 0) {
      player.y = p.y + p.h; player.vy = 0
    }
  }
  for (const s of l.spikes) {
    if (rectsOverlap(player, {x:s.x+5,y:s.y+7,w:s.w-10,h:s.h-7})) {
      player.x = checkpointX; player.y = 360; player.vx = 0; player.vy = 0
    }
  }
  if (player.y > WORLD_H + 120) { player.x = checkpointX; player.y = 350; player.vx = 0; player.vy = 0 }
  for (const c of l.coins) {
    if (!c.taken && rectsOverlap(player, {x:c.x-10,y:c.y-10,w:20,h:20})) {
      c.taken = true; collected.value++; score.value += 10
      if (soundOn.value) beep()
    }
  }
  if (player.x > l.width * .53 && checkpointX < l.width * .5) {
    checkpointX = player.x; player.spawnX = checkpointX; checkpointLabel.value = 'CHECKPOINT'
  }
  if (rectsOverlap(player, {x:l.goal.x-14,y:l.goal.y-70,w:44,h:90})) {
    finished.value = true
    score.value += 50
    if (score.value > best.value) { best.value = score.value; localStorage.setItem('moonhop-best', String(best.value)) }
  }
  camera.x += ((player.x - 250) - camera.x) * Math.min(1, dt * 5)
  camera.x = Math.max(0, Math.min(l.width - 900, camera.x))
}
function rounded(x:number,y:number,w:number,h:number,r:number) {
  if (!ctx) return
  ctx.beginPath(); ctx.roundRect(x,y,w,h,r)
}
function draw() {
  if (!ctx || !canvas.value) return
  const c = ctx, W = 900, H = 520
  const l = level.value
  c.clearRect(0,0,W,H)
  const sky = c.createLinearGradient(0,0,0,H)
  sky.addColorStop(0,'#10162f'); sky.addColorStop(1,'#171a37')
  c.fillStyle = sky; c.fillRect(0,0,W,H)
  // stars
  for (let i=0;i<75;i++) {
    const sx = ((i*137 + 23) % W)
    const sy = ((i*71 + 17) % 300)
    c.globalAlpha = .25 + ((i%4)*.16); c.fillStyle = '#d9e4ff'; c.fillRect(sx,sy,2,2)
  }
  c.globalAlpha = 1
  // distant moon
  c.fillStyle = '#d8dcff'; c.globalAlpha = .12; c.beginPath(); c.arc(735,100,48,0,Math.PI*2); c.fill(); c.globalAlpha = 1
  c.save(); c.translate(-camera.x,0)
  // background hills
  for (let i=0;i<Math.ceil(l.width/300)+2;i++) {
    c.fillStyle = i%2 ? '#20254a' : '#25264d'
    c.beginPath(); c.moveTo(i*300,450); c.lineTo(i*300+150,300+(i%3)*25); c.lineTo(i*300+300,450); c.closePath(); c.fill()
  }
  // platforms
  for (const p of l.platforms) {
    c.fillStyle = '#252a4e'; rounded(p.x,p.y,p.w,p.h,7); c.fill()
    c.fillStyle = '#7c6dff'; rounded(p.x,p.y,p.w,7,4); c.fill()
    c.fillStyle = 'rgba(255,255,255,.06)'
    for (let x=p.x+12;x<p.x+p.w-8;x+=24) c.fillRect(x,p.y+15,9,3)
  }
  // spikes
  for (const s of l.spikes) {
    c.fillStyle = '#ff668e'; c.beginPath()
    for (let x=s.x;x<s.x+s.w;x+=12) { c.moveTo(x,s.y+s.h); c.lineTo(x+6,s.y+2); c.lineTo(x+12,s.y+s.h) }
    c.fill()
  }
  // coins
  for (const coin of l.coins) if (!coin.taken) {
    const pulse = Math.sin(elapsed*5+coin.x)*2
    c.fillStyle = '#ffd66b'; c.beginPath(); c.ellipse(coin.x,coin.y,7+pulse*.25,10,0,0,Math.PI*2); c.fill()
    c.strokeStyle = '#fff0ad'; c.lineWidth = 2; c.stroke()
    c.fillStyle = '#fff2b3'; c.fillRect(coin.x-1,coin.y-5,2,10)
  }
  // checkpoint flag
  c.fillStyle = '#c4c9ff'; c.fillRect(checkpointX+8,300,3,130)
  c.fillStyle = '#8b7bff'; c.beginPath(); c.moveTo(checkpointX+11,302); c.lineTo(checkpointX+48,314); c.lineTo(checkpointX+11,328); c.fill()
  // goal portal
  const g=l.goal
  c.globalAlpha=.25; c.fillStyle='#71f7d4'; c.beginPath(); c.ellipse(g.x,g.y-25,28+Math.sin(elapsed*4)*3,45,0,0,Math.PI*2); c.fill(); c.globalAlpha=1
  c.strokeStyle='#71f7d4'; c.lineWidth=4; c.beginPath(); c.ellipse(g.x,g.y-25,18,35,0,0,Math.PI*2); c.stroke()
  c.fillStyle='#b8fff0'; c.fillRect(g.x-2,g.y-58,4,66)
  // player shadow
  c.globalAlpha=.25; c.fillStyle='#000'; c.beginPath(); c.ellipse(player.x+13,player.y+player.h+3,13,4,0,0,Math.PI*2); c.fill(); c.globalAlpha=1
  // player body
  c.fillStyle='#a9a0ff'; rounded(player.x,player.y+7,player.w,player.h-7,6); c.fill()
  c.fillStyle='#d9d5ff'; rounded(player.x+3,player.y,player.w-6,14,5); c.fill()
  c.fillStyle='#171a37'
  const eyeX = player.face >= 0 ? player.x+17 : player.x+7
  c.fillRect(eyeX,player.y+5,3,4)
  c.fillStyle='#7c6dff'; c.fillRect(player.x+4,player.y+player.h-4,7,5); c.fillRect(player.x+16,player.y+player.h-4,7,5)
  c.restore()
  // ground haze
  const haze=c.createLinearGradient(0,400,0,H); haze.addColorStop(0,'rgba(124,109,255,0)'); haze.addColorStop(1,'rgba(124,109,255,.08)'); c.fillStyle=haze;c.fillRect(0,400,W,H-400)
}
function frame(time: number) {
  if (!lastTime) lastTime = time
  const dt = Math.min((time-lastTime)/1000,.032); lastTime = time; elapsed += dt
  update(dt); draw(); raf = requestAnimationFrame(frame)
}
function resizeCanvas() {
  if (!canvas.value) return
  const rect = canvas.value.getBoundingClientRect()
  const dpr = Math.min(window.devicePixelRatio || 1, 2)
  canvas.value.width = Math.round(rect.width*dpr)
  canvas.value.height = Math.round(rect.height*dpr)
  ctx = canvas.value.getContext('2d')
  if (ctx) ctx.setTransform(dpr,0,0,dpr,0,0)
  // Render in a consistent logical coordinate system; scale world to responsive canvas.
  if (ctx) ctx.scale(rect.width/900, rect.height/520)
}
function beep() {
  try {
    const AudioContextClass = window.AudioContext
    const audio = new AudioContextClass()
    const osc = audio.createOscillator(); const gain = audio.createGain()
    osc.connect(gain); gain.connect(audio.destination); osc.frequency.value=760
    gain.gain.setValueAtTime(.035,audio.currentTime); gain.gain.exponentialRampToValueAtTime(.001,audio.currentTime+.08)
    osc.start(); osc.stop(audio.currentTime+.08); osc.onended=()=>audio.close()
  } catch {}
}
function touchKey(key: string, down: boolean) { setKey(key,down) }

watch(currentLevel, resetLevel)
onMounted(() => {
  resizeCanvas()
  window.addEventListener('resize', resizeCanvas)
  window.addEventListener('keydown', onKeyDown)
  window.addEventListener('keyup', onKeyUp)
  raf = requestAnimationFrame(frame)
})
onBeforeUnmount(() => {
  cancelAnimationFrame(raf)
  window.removeEventListener('resize', resizeCanvas)
  window.removeEventListener('keydown', onKeyDown)
  window.removeEventListener('keyup', onKeyUp)
})
</script>

<template>
  <main class="app-shell">
    <header class="topbar">
      <a class="brand" href="#" @click.prevent="goMenu"><span class="brand-mark">✦</span> moonhop<span class="brand-dot">.</span></a>
      <div class="top-meta"><span class="status-dot"></span> INDIE PLATFORMER <span class="meta-divider">/</span> BUILD 01</div>
      <button class="sound-toggle" @click="soundOn = !soundOn" :aria-label="soundOn ? 'Matikan suara' : 'Nyalakan suara'">{{ soundOn ? '♫ SOUND ON' : '♪ SOUND OFF' }}</button>
    </header>

    <section v-if="!playing" class="hero">
      <div class="hero-copy">
        <div class="eyebrow"><span class="eyebrow-line"></span> YOUR NEXT LITTLE ADVENTURE</div>
        <h1>One small hop.<br /><span>One giant</span> adventure.</h1>
        <p class="hero-desc">Jelajahi dunia malam, kumpulkan cahaya, dan temukan jalan menuju portal. Siap untuk melompat?</p>
        <button class="primary-btn" @click="startGame">PLAY GAME <span>↗</span></button>
        <div class="hero-stats"><div><strong>03</strong><small>LEVELS</small></div><i></i><div><strong>∞</strong><small>JUMPS</small></div><i></i><div><strong>1</strong><small>HERO</small></div></div>
      </div>
      <div class="hero-art">
        <div class="orbit orbit-one"></div><div class="orbit orbit-two"></div>
        <div class="hero-moon"></div><div class="hero-platform platform-a"></div><div class="hero-platform platform-b"></div><div class="hero-platform platform-c"></div>
        <div class="hero-coin coin-a">✦</div><div class="hero-coin coin-b">✦</div>
        <div class="hero-character"><div class="char-face"><i></i><i></i></div><div class="char-feet"><i></i><i></i></div></div>
        <div class="art-label label-top">FIG. 001 <span>— THE HOPPER</span></div><div class="art-label label-bottom">SOMEWHERE IN THE MOONLIGHT <span>✧</span></div>
      </div>
      <section class="level-picker">
        <div class="section-heading"><div><span class="mini-kicker">CHOOSE YOUR PATH</span><h2>Pick a level<span>.</span></h2></div><span class="level-count">01 — 03</span></div>
        <div class="level-grid">
          <button v-for="(item, index) in levels" :key="item.name" class="level-card" @click="selectLevel(index)">
            <div class="level-thumb" :class="'thumb-'+index"><span class="thumb-number">0{{ index+1 }}</span><span class="thumb-moon"></span><span class="thumb-platform tp1"></span><span class="thumb-platform tp2"></span><span class="thumb-player"></span><span class="thumb-coin">✦</span></div>
            <div class="level-info"><div><h3>{{ item.name }}</h3><p>{{ item.subtitle }}</p></div><span class="level-arrow">↗</span></div>
          </button>
        </div>
      </section>
    </section>

    <section v-else class="game-page">
      <div class="game-heading"><div><div class="eyebrow"><span class="eyebrow-line"></span> NOW PLAYING</div><h1>{{ level.name }}<span>.</span></h1></div><button class="ghost-btn" @click="goMenu">← BACK TO MENU</button></div>
      <div class="game-hud"><div class="hud-item"><span class="hud-icon">✦</span><div><small>SCORE</small><strong>{{ String(score).padStart(3,'0') }}</strong></div></div><div class="hud-item"><span class="hud-icon coin-icon">●</span><div><small>COINS</small><strong>{{ String(collected).padStart(2,'0') }} <em>/ {{ totalCoins }}</em></strong></div></div><div class="hud-item checkpoint-hud"><span class="hud-icon">⚑</span><div><small>CHECKPOINT</small><strong>{{ checkpointLabel }}</strong></div></div><div class="hud-actions"><button @click="paused=!paused" class="hud-button">{{ paused ? '▶ RESUME' : 'Ⅱ PAUSE' }}</button><button @click="resetLevel" class="hud-button">↻ RESET</button></div></div>
      <div class="canvas-wrap">
        <canvas ref="canvas" aria-label="Area permainan platformer"></canvas>
        <div class="progress-track"><div class="progress-fill" :style="{width: progress+'%'}"></div></div>
        <div v-if="paused && !finished" class="game-overlay"><div class="overlay-card"><span class="overlay-icon">Ⅱ</span><h2>Take a breath.</h2><p>Petualanganmu dijeda.</p><button class="primary-btn" @click="paused=false">CONTINUE <span>↗</span></button></div></div>
        <div v-if="finished" class="game-overlay"><div class="overlay-card"><span class="overlay-icon">✦</span><div class="eyebrow centered">LEVEL COMPLETE</div><h2>You made it!</h2><p>Kamu berhasil mencapai portal dan mengumpulkan {{ collected }} koin.</p><div class="result-score">SCORE <strong>{{ score }}</strong></div><div class="overlay-actions"><button class="ghost-btn" @click="goMenu">MENU</button><button v-if="currentLevel<levels.length-1" class="primary-btn" @click="selectLevel(currentLevel+1)">NEXT LEVEL <span>↗</span></button><button v-else class="primary-btn" @click="startGame">PLAY AGAIN <span>↗</span></button></div></div></div>
      </div>
      <div class="game-footer"><span><kbd>←</kbd> <kbd>→</kbd> MOVE <span class="footer-sep">·</span> <kbd>SPACE</kbd> JUMP</span><span>PROGRESS <strong>{{ progress }}%</strong></span></div>
      <div class="mobile-controls"><button @pointerdown.prevent="touchKey('a',true)" @pointerup.prevent="touchKey('a',false)" @pointerleave="touchKey('a',false)">←</button><button @pointerdown.prevent="touchKey('d',true)" @pointerup.prevent="touchKey('d',false)" @pointerleave="touchKey('d',false)">→</button><button class="jump-btn" @pointerdown.prevent="jump">JUMP ↑</button></div>
      <div class="below-levels"><div><span class="mini-kicker">CHANGE LEVEL</span><h2>Choose another path<span>.</span></h2></div><div class="inline-levels"><button v-for="(item,index) in levels" :key="item.name" @click="selectLevel(index)" :class="{active: currentLevel===index}">0{{index+1}} <span>{{item.name}}</span> ↗</button></div></div>
    </section>
    <footer class="site-footer"><span>MOONHOP © 2026</span><span>MADE FOR THE LOVE OF LITTLE ADVENTURES <b>✦</b></span><span>VUE 3 + CANVAS</span></footer>
  </main>
</template>