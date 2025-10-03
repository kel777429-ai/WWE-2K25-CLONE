# WWE-2K25-CLONE 
<!doctype html>

<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Wrestling Game — Inspired by WWE 2K25 (Clone)</title>
  <style>
    html,body{height:100%;margin:0;background:#111;color:#eee;font-family:Inter,Segoe UI,Arial}
    #game{display:flex;flex-direction:column;height:100vh}
    canvas{background:linear-gradient(#222,#111);display:block;margin:0 auto;border:4px solid #222;box-shadow:0 10px 30px rgba(0,0,0,.7)}
    #ui{display:flex;justify-content:space-between;padding:10px 20px;align-items:center}
    .panel{background:rgba(255,255,255,0.03);padding:8px 12px;border-radius:8px;min-width:120px}
    .meter{height:8px;background:rgba(255,255,255,0.08);border-radius:6px;overflow:hidden}
    .meter > i{display:block;height:100%;background:linear-gradient(90deg,#0f0,#8f8)}
    #log{max-height:110px;overflow:auto;padding:8px;font-size:13px}
    button{background:#222;border:1px solid rgba(255,255,255,0.06);color:#fff;padding:6px 10px;border-radius:6px}
    #controls{display:flex;gap:8px}
    .help{font-size:13px;color:#bbb}
  </style>
</head>
<body>
  <div id="game">
    <div id="ui">
      <div class="panel">
        <div><strong id="playerName">PLAYER</strong></div>
        <div class="help">Stamina</div>
        <div class="meter" style="width:160px;margin:6px 0"><i id="playerStam" style="width:100%"></i></div>
        <div class="help">Finisher</div>
        <div class="meter" style="width:160px;margin:6px 0"><i id="playerFin" style="width:0%"></i></div>
      </div><div style="text-align:center">
    <div class="panel"> <strong id="matchState">Entrance</strong> </div>
    <div style="height:6px"></div>
    <div class="panel">Time: <span id="matchTimer">0:00</span></div>
  </div>

  <div class="panel" style="text-align:right">
    <div><strong id="opponentName">CPU</strong></div>
    <div class="help">Stamina</div>
    <div class="meter" style="width:160px;margin:6px 0"><i id="cpuStam" style="width:100%"></i></div>
    <div class="help">Finisher</div>
    <div class="meter" style="width:160px;margin:6px 0"><i id="cpuFin" style="width:0%"></i></div>
  </div>
</div>

<canvas id="arena" width="1200" height="600"></canvas>

<div id="ui" style="align-items:flex-start">
  <div class="panel" style="flex:1">
    <div id="log">Welcome — press Enter to start. Controls: Arrow keys to move, A = Attack, S = Grapple, D = Dodge, F = Finisher, R = Taunt.</div>
  </div>
  <div class="panel" style="width:320px">
    <div style="display:flex;justify-content:space-between;align-items:center">
      <div class="help">Round</div>
      <div><strong id="roundText">1</strong></div>
    </div>
    <div style="height:8px"></div>
    <div id="controls">
      <button id="btnStart">Start</button>
      <button id="btnReset">Reset</button>
      <button id="btnToggleAI">Toggle AI</button>
    </div>
  </div>
</div>

  </div><script>
/*
  Wrestling Game (single-file) - Simplified but feature-rich
  - Two wrestlers (player + cpu)
  - Movement, attacks, grapples, reversals, finishers, pins
  - Simple AI with aggression and reversal timing
  - Entrances, crowd meter, announcer log, stamina/finisher meters
  - No copyrighted assets; placeholders used.
*/

const canvas = document.getElementById('arena');
const ctx = canvas.getContext('2d');
const W = canvas.width, H = canvas.height;

let game = null;

class Wrestler {
  constructor(name,x,team='blue',isPlayer=false){
    this.name = name; this.x = x; this.y = H-180; this.w = 80; this.h = 160;
    this.vx = 0; this.facing = (isPlayer?1:-1); this.team = team;
    this.color = (team==='blue'?'#2F8EF7':'#F72F4B');
    this.isPlayer = isPlayer;
    this.health = 100; // not used directly; stamina used
    this.stamina = 100;
    this.finisher = 0;
    this.anim = 'idle'; this.animTimer = 0;
    this.locked = false; // locked during moves
    this.onGround = true;
    this.grappleTarget = null;
    this.inPin = false;
    this.pinTimer = 0;
    this.controls = {};
    this.reversalWindow = 0; // frames where reversal is possible
    this.combo = 0;
  }
  rect(){ return {x:this.x - this.w/2, y:this.y - this.h, w:this.w, h:this.h}; }
}

class Game {
  constructor(){
    this.player = new Wrestler('PLAYER', W*0.3, 'blue', true);
    this.cpu = new Wrestler('CPU', W*0.7, 'red', false);
    window.player = this.player; window.cpu = this.cpu;

    this.player.controls = {left:false,right:false,attack:false,grapple:false,dodge:false,finisher:false,taunt:false};
    this.cpu.controls = {left:false,right:false,attack:false,grapple:false,dodge:false,finisher:false,taunt:false};

    this.state = 'entrance';
    this.timer = 180; // seconds
    this.tick = 0;
    this.logs = [];
    this.crowd = 50; // 0-100
    this.aiOn = true;
    this.round = 1;
    this.addLog('Entrance: Wrestlers make their way to the ring.');

    this.setupBindings();
  }
  addLog(txt){
    const log = document.getElementById('log');
    this.logs.unshift((new Date()).toLocaleTimeString()+ ' — ' + txt);
    log.innerHTML = this.logs.slice(0,30).join('<br>');
  }
  setupBindings(){
    document.addEventListener('keydown', e=>{
      if(e.repeat) return;
      if(this.state === 'entrance' && e.key==='Enter'){ this.startMatch(); }
      if(e.key==='ArrowLeft'){ this.player.controls.left=true; }
      if(e.key==='ArrowRight'){ this.player.controls.right=true; }
      if(e.key.toLowerCase()==='a'){ this.player.controls.attack=true; }
      if(e.key.toLowerCase()==='s'){ this.player.controls.grapple=true; }
      if(e.key.toLowerCase()==='d'){ this.player.controls.dodge=true; }
      if(e.key.toLowerCase()==='f'){ this.player.controls.finisher=true; }
      if(e.key.toLowerCase()==='r'){ this.player.controls.taunt=true; }
    });
    document.addEventListener('keyup', e=>{
      if(e.key==='ArrowLeft'){ this.player.controls.left=false; }
      if(e.key==='ArrowRight'){ this.player.controls.right=false; }
      if(e.key.toLowerCase()==='a'){ this.player.controls.attack=false; }
      if(e.key.toLowerCase()==='s'){ this.player.controls.grapple=false; }
      if(e.key.toLowerCase()==='d'){ this.player.controls.dodge=false; }
      if(e.key.toLowerCase()==='f'){ this.player.controls.finisher=false; }
      if(e.key.toLowerCase()==='r'){ this.player.controls.taunt=false; }
    });

    document.getElementById('btnStart').addEventListener('click', ()=>this.startMatch());
    document.getElementById('btnReset').addEventListener('click', ()=>this.reset());
    document.getElementById('btnToggleAI').addEventListener('click', ()=>{this.aiOn=!this.aiOn;this.addLog('AI: '+(this.aiOn?'ON':'OFF'))});
  }
  startMatch(){
    if(this.state !== 'battle'){
      this.state = 'battle'; this.addLog('Match started!');
      this.round = 1; document.getElementById('roundText').innerText = this.round;
      document.getElementById('matchState').innerText = 'Battle';
    }
  }
  reset(){
    this.player = new Wrestler('PLAYER', W*0.3, 'blue', true);
    this.cpu = new Wrestler('CPU', W*0.7, 'red', false);
    this.state = 'entrance'; this.timer = 180; this.addLog('Match reset.');
    document.getElementById('matchState').innerText = 'Entrance';
  }
  update(dt){
    this.tick++;
    if(this.state === 'entrance'){
      // simple entrance animation
      this.player.x += 1; this.cpu.x -= 1;
      if(this.tick>120) { document.getElementById('matchState').innerText='Ready'; }
      return;
    }
    if(this.state === 'battle'){
      this.timer = Math.max(0, this.timer - dt);
      this.updateControls(this.player, dt, true);
      if(this.aiOn) this.aiBehaviour(this.cpu,dt);
      else this.updateControls(this.cpu, dt, false);

      this.resolveMovement(this.player,dt);
      this.resolveMovement(this.cpu,dt);

      this.handleInteractions(dt);

      // update ui meters
      document.getElementById('playerStam').style.width = Math.max(0, this.player.stamina)+'%';
      document.getElementById('cpuStam').style.width = Math.max(0, this.cpu.stamina)+'%';
      document.getElementById('playerFin').style.width = Math.min(100,this.player.finisher)+'%';
      document.getElementById('cpuFin').style.width = Math.min(100,this.cpu.finisher)+'%';
      document.getElementById('matchTimer').innerText = this.formatTime(this.timer);

      if(this.player.inPin || this.cpu.inPin){ this.handlePins(dt); }

      if(this.timer<=0){ this.endMatch('Time Limit Draw'); }
    }
  }
  formatTime(sec){
    const s = Math.floor(sec); const m = Math.floor(s/60); const r = s%60; return `${m}:${r.toString().padStart(2,'0')}`;
  }
  updateControls(wrestler,dt,isPlayer){
    // apply player inputs to velocity
    if(wrestler.locked) return;
    const speed = 220 * (dt);
    if(isPlayer){
      if(wrestler.controls.left) { wrestler.vx = -speed; wrestler.facing = -1; }
      else if(wrestler.controls.right){ wrestler.vx = speed; wrestler.facing = 1; }
      else wrestler.vx = 0;

      if(wrestler.controls.attack){ this.tryAttack(wrestler); }
      if(wrestler.controls.grapple){ this.tryGrapple(wrestler); }
      if(wrestler.controls.dodge){ this.tryDodge(wrestler); }
      if(wrestler.controls.finisher){ this.tryFinisher(wrestler); }
      if(wrestler.controls.taunt){ this.doTaunt(wrestler); }
    }
  }
  aiBehaviour(ai,dt){
    const p = this.player;
    // simple AI: approach, attack, attempt grapple if close
    const dist = Math.abs(ai.x - p.x);
    // random decision every 30 ticks
    if(this.tick % 30 === 0){
      const r = Math.random();
      if(dist > 160) { ai.controls.left = (ai.x>p.x); ai.controls.right = (ai.x<p.x); }
      else { ai.controls.left = ai.controls.right = false; }
      ai.controls.attack = (r < 0.5 && dist < 220);
      ai.controls.grapple = (r < 0.22 && dist < 120);
      ai.controls.dodge = (r < 0.08 && dist < 200);
      ai.controls.finisher = (ai.finisher>85 && r<0.3);
    }
    // use updateControls to process
    this.updateControls(ai,dt,false);
  }
  tryAttack(w){
    if(w.locked) return;
    w.anim='attack'; w.locked=true; w.animTimer=20; w.stamina=Math.max(0,w.stamina-6);
    // set reversal window for opponent
    const target = (w===this.player?this.cpu:this.player);
    target.reversalWindow = 14; // frames
    this.addLog(`${w.name} throws a strike.`);
  }
  tryGrapple(w){
    if(w.locked) return;
    // must be close
    const target = (w===this.player?this.cpu:this.player);
    if(Math.abs(w.x - target.x) < 120){
      if(target.reversalWindow>0 && Math.random()<0.65){ // opponent reverses
        this.addLog(`${target.name} reverses the grapple!`);
        target.locked=true; target.anim='grappleReverse'; target.animTimer=26; target.stamina=Math.max(0,target.stamina-8);
        target.finisher = Math.min(100,target.finisher + 6);
      } else {
        w.locked=true; w.anim='grapple'; w.animTimer=34; w.grappleTarget=target; w.stamina=Math.max(0,w.stamina-10);
        this.addLog(`${w.name} hits a grapple on ${target.name}.`);
      }
    }
  }
  tryDodge(w){
    if(w.locked) return; w.locked=true; w.anim='dodge'; w.animTimer=16; w.stamina=Math.max(0,w.stamina-4);
    this.addLog(`${w.name} attempts to dodge.`);
  }
  tryFinisher(w){
    if(w.locked) return;
    if(w.finisher < 100) { this.addLog(`${w.name} tries a finisher but doesn't have full meter.`); return; }
    const target = (w===this.player?this.cpu:this.player);
    if(Math.abs(w.x - target.x) > 140){ this.addLog('Finisher missed (too far).'); return; }
    // target can reverse with good timing
    if(target.reversalWindow>0 && Math.random()<0.6){ this.addLog(`${target.name} narrowly escapes the finisher!`); target.reversalWindow=0; return; }
    w.locked=true; w.anim='finisher'; w.animTimer=80; w.stamina=Math.max(0,w.stamina-20); w.finisher=0;
    this.addLog(`${w.name} hits their FINISHER!`);
    // big damage leads to pin attempt
    target.inPin=true; target.pinTimer=0; this.addLog(`${w.name} goes for pin.`);
  }
  doTaunt(w){ if(w.locked) return; w.finisher=Math.min(100,w.finisher+6); this.crowd = Math.min(100,this.crowd+6); this.addLog(`${w.name} taunts the crowd!`); }

  resolveMovement(w,dt){
    // simple bounds
    w.x += w.vx;
    w.x = Math.max(80, Math.min(W-80, w.x));
    if(w.locked){
      w.animTimer--;
      if(w.animTimer<=0){ w.locked=false; w.anim='idle'; w.grappleTarget=null; }
    }
    // natural stamina regen if idle
    if(!w.locked && Math.abs(w.vx)<0.1){ w.stamina = Math.min(100, w.stamina + 0.4); }
    // moving drains
    if(Math.abs(w.vx)>0.1) w.stamina = Math.max(0,w.stamina - 0.25);

    // reversal window decays
    if(w.reversalWindow>0) w.reversalWindow = Math.max(0, w.reversalWindow - 1);

    // finisher gain over time/attacks
    if(this.tick % 40 === 0) w.finisher = Math.min(100, w.finisher + 1);
  }

  handleInteractions(dt){
    // detect collisions when attack finishes
    [this.player,this.cpu].forEach(w => {
      if(w.anim === 'attack' && w.animTimer === 0){
        const target = (w===this.player?this.cpu:this.player);
        if(Math.abs(w.x - target.x) < 140){
          // check target dodge
          if(target.anim === 'dodge' && target.animTimer>0){ this.addLog(`${target.name} dodges the strike.`); }
          else {
            // damage
            this.addLog(`${w.name} connects with a strike on ${target.name}.`);
            target.stamina = Math.max(0,target.stamina - 18);
            target.finisher = Math.min(100,target.finisher + 8);
            this.crowd = Math.min(100,this.crowd + 3);
            // possible pin if low stamina
            if(target.stamina < 15){ target.inPin=true; target.pinTimer=0; this.addLog(`${w.name} attempts a pin!`); }
          }
        }
      }
      if(w.anim === 'grapple' && w.animTimer === 0 && w.grappleTarget){
        const t = w.grappleTarget;
        if(Math.abs(w.x - t.x) < 160){
          this.addLog(`${w.name} performs a suplex-like move on ${t.name}.`);
          t.stamina = Math.max(0,t.stamina - 26);
          w.finisher = Math.min(100,w.finisher + 10);
          this.crowd = Math.min(100,this.crowd + 6);
          if(t.stamina < 12){ t.inPin=true; t.pinTimer=0; this.addLog(`${w.name} goes for a pin!`); }
        }
      }
    });
  }

  handlePins(dt){
    // whichever is pinned loses if ref counts to 3
    const pinned = this.player.inPin ? this.player : (this.cpu.inPin ? this.cpu : null);
    const attacker = pinned === this.player ? this.cpu : this.player;
    if(!pinned) return;
    pinned.pinTimer += 1*dt*60; // frames in seconds
    // acceptance: press reversal quickly
    if(pinned === this.player && this.player.controls.dodge && pinned.pinTimer < 40){ // player escapes
      pinned.inPin = false; pinned.pinTimer = 0; this.addLog(`${pinned.name} kicks out!`); return;
    }
    if(pinned.pinTimer > 120){ // ref counts to 3
      this.addLog(`${attacker.name} wins by pinfall!`);
      this.endMatch(`${attacker.name} by pinfall`);
    }
  }

  endMatch(reason){
    this.addLog('Match ended: ' + reason);
    this.state = 'ended'; document.getElementById('matchState').innerText = 'Ended';
  }
}

function drawRing(){
  // ring mat
  ctx.fillStyle = '#1f1f1f'; ctx.fillRect(120,80,W-240,H-160);
  // ropes
  const ropesY = [150, 230, 310];
  ropesY.forEach((y,idx)=>{
    ctx.strokeStyle = idx===1? '#f2c94c' : '#e0e0e0'; ctx.lineWidth = 6; ctx.beginPath(); ctx.moveTo(140,y); ctx.lineTo(W-140,y); ctx.stroke();
  });
  // turnbuckles
  [[140,150],[140,310],[W-140,150],[W-140,310]].forEach(([x,y])=>{
    ctx.fillStyle='#333'; ctx.beginPath(); ctx.arc(x,y,22,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#222'; ctx.fillRect(x-12,y-6,24,12);
  });
}

function render(){
  ctx.clearRect(0,0,W,H);
  drawRing();
  // crowd background
  const g = game;
  // draw crowd top band
  const gradient = ctx.createLinearGradient(0,0,0,80); gradient.addColorStop(0,'#000'); gradient.addColorStop(1,'rgba(0,0,0,0)'); ctx.fillStyle = gradient; ctx.fillRect(0,0,W,80);

  // draw wrestlers
  [g.player,g.cpu].forEach(w=>{
    // shadow
    ctx.fillStyle = 'rgba(0,0,0,0.45)'; ctx.fillRect(w.x - w.w/2 + 6, w.y - 6, w.w, 8);
    // body
    ctx.fillStyle = w.color; ctx.fillRect(w.x - w.w/2, w.y - w.h, w.w, w.h);
    // face
    ctx.fillStyle = '#fff'; ctx.fillRect(w.x - 18, w.y - w.h + 12, 36, 28);
    // name
    ctx.fillStyle = '#fff'; ctx.font = '12px Arial'; ctx.textAlign='center'; ctx.fillText(w.name, w.x, w.y - w.h - 10);
    // small status
    if(w.locked){ ctx.fillStyle='rgba(0,0,0,0.5)'; ctx.fillRect(w.x - w.w/2, w.y - w.h - 26, w.w, 18); ctx.fillStyle='#fff'; ctx.fillText(w.anim, w.x, w.y - w.h - 14); }
  });

  // crowd meter
  ctx.fillStyle='#222'; ctx.fillRect(20,H-70,200,40);
  ctx.fillStyle='#fff'; ctx.font='14px Arial'; ctx.fillText('Crowd', 30, H-48);
  ctx.fillStyle='#333'; ctx.fillRect(20+60, H-62, 120, 18);
  ctx.fillStyle='#fff'; ctx.fillRect(20+60, H-62, 120*(g.crowd/100), 18);

  // match state & logs overlay
  ctx.fillStyle='rgba(0,0,0,0.6)'; ctx.fillRect(W-360,H-90,340,74);
  ctx.fillStyle='#fff'; ctx.font='12px Arial'; ctx.textAlign='left';
  ctx.fillText('State: '+document.getElementById('matchState').innerText, W-350, H-68);
  ctx.fillText('Timer: '+document.getElementById('matchTimer').innerText, W-350, H-50);
  ctx.fillText('Round: '+document.getElementById('roundText').innerText, W-350, H-32);
}

function gameLoop(ts){
  if(!game) return;
  const dt = 1/60; // fixed
  game.update(dt);
  render();
  if(game.state !== 'ended') requestAnimationFrame(gameLoop);
}

// initialize
function init(){
  game = new Game();
  // show names
  document.getElementById('playerName').innerText = game.player.name;
  document.getElementById('opponentName').innerText = game.cpu.name;
  requestAnimationFrame(gameLoop);
}

init();
</script></body>
</html>
