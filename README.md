<!DOCTYPE html>
<html lang="fa">
<head>
<meta charset="UTF-8">
<title>📊 Live SNR – DSN Network</title>
<style>
html,body{margin:0;overflow:hidden;background:#00030a;font-family:system-ui}
canvas{display:block}
.hud{
  position:absolute;left:16px;top:16px;
  padding:14px 18px;border-radius:14px;
  background:rgba(255,255,255,0.06);
  ackdrop-filter:blur(10px);
  color:#d9f3ff;font-size:13px;min-width:300px;
}
.chart{
  position:absolute;left:16px;bottom:16px;
  background:rgba(255,255,255,0.06);
  border-radius:14px;
  backdrop-filter:blur(10px);
  padding:10px;
}
.good{color:#9ff0ff}
.mid{color:#ffd29f}
.bad{color:#ff9f9f}
.warn{color:#ffb366}
</style>
</head>
<body>

<canvas id="space"></canvas>
<canvas id="snrChart" class="chart" width="360" height="140"></canvas>
<div class="hud" id="hud"></div>

<script>
const space=document.getElementById("space");
const ctx=space.getContext("2d");
const chart=document.getElementById("snrChart");
const cctx=chart.getContext("2d");

let w,h;
function resize(){w=space.width=innerWidth;h=space.height=innerHeight}
resize();addEventListener("resize",resize);

// Earth & satellite
const earth={x:w/2,y:h/2,r:70,mu:9000};
const sat={x:earth.x,y:earth.y-240,vx:2.25,vy:0};

// DSN stations
const stations=[
  {name:"Goldstone",angle:0.2,color:"#00ffcc"},
  {name:"Madrid",angle:2.2,color:"#66aaff"},
  {name:"Canberra",angle:4.1,color:"#ffaa66"}
];

// Sun & storm
const sun={angle:0};
let storm={active:false,intensity:0,timer:0};

// Radio
const txPower=130;
const maxDist=420;
let snrHistory=[];

// Physics
function gravity(){
  const dx=earth.x-sat.x,dy=earth.y-sat.y;
  const d=Math.hypot(dx,dy);
  const f=earth.mu/(d*d);
  sat.vx+=f*dx/d;
  sat.vy+=f*dy/d;
}

// LOS
function hasLOS(st){
  const dx=sat.x-st.x,dy=sat.y-st.y;
  const d=Math.hypot(dx,dy);
  if(d>maxDist) return false;
  const t=((earth.x-st.x)*dx+(earth.y-st.y)*dy)/(d*d);
  const px=st.x+t*dx,py=st.y+t*dy;
  return Math.hypot(px-earth.x,py-earth.y)>earth.r;
}

// Shadow
function inShadow(){
  const lx=Math.cos(sun.angle),ly=Math.sin(sun.angle);
  const dx=sat.x-earth.x,dy=sat.y-earth.y;
  const proj=dx*lx+dy*ly;
  if(proj<0) return false;
  const perp=Math.abs(-dx*ly+dy*lx);
  return perp<earth.r;
}

// Storm
function updateStorm(){
  if(!storm.active && Math.random()<0.001){
    storm.active=true;
    storm.intensity=1+Math.random()*3;
    storm.timer=300+Math.random()*400;
  }
  if(storm.active){
    storm.timer--;
    storm.intensity*=0.994;
    if(storm.timer<=0||storm.intensity<0.1)storm.active=false;
  }
}

// Noise
function plasmaNoise(){
  let n=0.04+Math.random()*0.05;
  if(storm.active) n+=storm.intensity*0.25;
  return n;
}

// Link
function link(st){
  if(!hasLOS(st)) return null;
  const dx=sat.x-st.x,dy=sat.y-st.y;
  const d=Math.hypot(dx,dy);
  let signal=txPower/(d*d);
  if(inShadow()) signal*=0.25;
  const noise=plasmaNoise();
  const snr=signal/noise;
  return {snr,dist:d};
}

// Chart
function drawChart(){
  cctx.clearRect(0,0,chart.width,chart.height);
  cctx.strokeStyle="#66e0ff";
  cctx.lineWidth=2;
  cctx.beginPath();

  snrHistory.forEach((v,i)=>{
    const x=i*(chart.width/100);
    const y=chart.height - Math.min(v*6,chart.height-10);
    if(i===0) cctx.moveTo(x,y);
    else cctx.lineTo(x,y);
  });
  cctx.stroke();

  cctx.fillStyle="#9ff0ff";
  cctx.fillText("SNR (Live)",10,14);
}

// Update
function update(){
  gravity();
  sat.x+=sat.vx; sat.y+=sat.vy;
  stations.forEach(s=>{
    s.angle+=0.0007;
    s.x=earth.x+Math.cos(s.angle)*earth.r;
    s.y=earth.y+Math.sin(s.angle)*earth.r;
  });
  updateStorm();
  sun.angle+=0.0003;
}

// Draw
function draw(){
  ctx.fillStyle="rgba(0,3,10,0.35)";
  ctx.fillRect(0,0,w,h);

  update();

  // Earth
  ctx.beginPath();
  ctx.arc(earth.x,earth.y,earth.r,0,Math.PI*2);
  ctx.fillStyle="#0b3d91";
  ctx.fill();

  // Satellite
  ctx.beginPath();
  ctx.arc(sat.x,sat.y,4,0,Math.PI*2);
  ctx.fillStyle="#e6f7ff";
  ctx.fill();

  let best=null;
  stations.forEach(st=>{
    ctx.beginPath();
    ctx.arc(st.x,st.y,4,0,Math.PI*2);
    ctx.fillStyle=st.color;
    ctx.fill();
    const q=link(st);
    if(q && (!best || q.snr>best.q.snr)) best={st,q};
  });

  let hud="🛰️ DSN Mission Control<br>";
  if(storm.active) hud+=`🌞 <span class="warn">SOLAR STORM</span><br>`;

  if(best){
    snrHistory.push(best.q.snr);
    if(snrHistory.length>100) snrHistory.shift();
    drawChart();

    let cls=best.q.snr>12?"good":best.q.snr>6?"mid":"bad";
    ctx.strokeStyle=cls==="good"?"rgba(120,220,255,0.9)":cls==="mid"?"rgba(255,200,120,0.7)":"rgba(255,120,120,0.6)";
    ctx.lineWidth=1.8;
    ctx.beginPath();
    ctx.moveTo(best.st.x,best.st.y);
    ctx.lineTo(sat.x,sat.y);
    ctx.stroke();

    hud+=`
📡 Station: <b>${best.st.name}</b><br>
📊 SNR: <span class="${cls}">${best.q.snr.toFixed(1)}</span><br>
🌑 Shadow: ${inShadow()?"YES":"NO"}<br>
🔁 Auto Handover: ON
`;
  }else{
    hud+=`📡 <span class="bad">NO LINK</span>`;
  }

  document.getElementById("hud").innerHTML=hud;
  requestAnimationFrame(draw);
}

draw();
</script>
</body>
</html>
