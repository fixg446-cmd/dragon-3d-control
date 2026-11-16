
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Dragon AI Hand Control</title>
<style>
*{box-sizing:border-box}
body{margin:0;overflow:hidden;background:#050b19;color:white;font-family:Arial}
#scene{position:fixed;inset:0}
#ui{position:fixed;top:15px;width:100%;text-align:center;z-index:3;pointer-events:none}
h2{color:#66eaff;text-shadow:0 0 15px #00bfff;margin:5px}
#status{font-size:13px;margin-top:8px}
#video{position:fixed;bottom:12px;right:12px;width:110px;border:2px solid #36d9ff;border-radius:12px;transform:scaleX(-1);z-index:4}
#start{position:fixed;top:50%;left:50%;transform:translate(-50%,-50%);z-index:8;background:#00bce8;color:#00101b;border:0;border-radius:12px;padding:17px 25px;font-weight:bold;font-size:16px}
#help{position:fixed;bottom:12px;left:12px;font-size:12px;line-height:1.8;background:#101b30cc;padding:10px;border-radius:10px}
</style>
</head>
<body>
<div id="scene"></div>
<div id="ui">
<h2>🐉 DRAGON AI CONTROL</h2>
<div id="status">Press START and show your hand</div>
</div>
<button id="start">START DRAGON CONTROL</button>
<video id="video" autoplay playsinline muted></video>
<div id="help">
🖐 Open hand: Move dragon<br>
☝️ Index up: Fly higher<br>
✊ Fist: Stop<br>
🤏 Pinch: Fire breath
</div>

<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script type="module">
import * as THREE from
'https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js';

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x050b19);
scene.fog = new THREE.Fog(0x050b19,15,45);

const camera = new THREE.PerspectiveCamera(55,innerWidth/innerHeight,.1,100);
camera.position.set(0,1,13);

const renderer = new THREE.WebGLRenderer({antialias:true});
renderer.setSize(innerWidth,innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio,2));
renderer.shadowMap.enabled=true;
document.getElementById('scene').appendChild(renderer.domElement);

scene.add(new THREE.AmbientLight(0x8099bb,2));
const light=new THREE.DirectionalLight(0xffffff,3);
light.position.set(4,8,8);
scene.add(light);

const dragon=new THREE.Group();
scene.add(dragon);

const green=new THREE.MeshStandardMaterial({
color:0x087d68,roughness:.45,metalness:.15
});
const dark=new THREE.MeshStandardMaterial({color:0x073c36});
const gold=new THREE.MeshStandardMaterial({
color:0xffc43d,metalness:.5,roughness:.3
});
const eyeMat=new THREE.MeshBasicMaterial({color:0xff2929});

function sphere(parent,x,y,z,sx,sy,sz,mat){
const m=new THREE.Mesh(new THREE.SphereGeometry(1,24,16),mat);
m.position.set(x,y,z);m.scale.set(sx,sy,sz);
parent.add(m);return m;
}
function cone(parent,x,y,z,r,h,mat,rx=0){
const m=new THREE.Mesh(new THREE.ConeGeometry(r,h,16),mat);
m.position.set(x,y,z);m.rotation.x=rx;
parent.add(m);return m;
}

// Body and neck
sphere(dragon,0,0,0,1.45,.75,.65,green);
sphere(dragon,1.05,.55,0,.48,.65,.45,green);
sphere(dragon,1.45,.95,0,.62,.48,.5,green);
sphere(dragon,1.9,.78,.05,.55,.27,.38,green);
sphere(dragon,1.55,1.12,.43,.11,.12,.07,eyeMat);
sphere(dragon,1.55,1.12,-.43,.11,.12,.07,eyeMat);

// Horns
cone(dragon,1.15,1.55,.3,.15,.65,gold,-.5);
cone(dragon,1.15,1.55,-.3,.15,.65,gold,-.5);

// Legs
for(const x of [-.85,.75]){
for(const z of [-.35,.35]){
sphere(dragon,x,-.72,z,.25,.48,.23,dark);
sphere(dragon,x+.1,-1.05,z,.3,.14,.27,green);
}}

// Tail
let tail=[];
for(let i=0;i<7;i++){
const t=sphere(dragon,-1.25-i*.31,-.05-i*.07,0,
.43-i*.045,.24-i*.022,.25-i*.025,green);
tail.push(t);
}
cone(dragon,-3.5,-.55,0,.28,.8,gold,Math.PI/2);

// Wings
function makeWing(side){
const pivot=new THREE.Group();
pivot.position.set(-.15,.4,side*.35);
dragon.add(pivot);

const shape=new THREE.Shape();
shape.moveTo(0,0);
shape.lineTo(-.35,1.8);
shape.lineTo(-1.35,2.5);
shape.lineTo(-1.05,1.2);
shape.lineTo(-2.1,1.75);
shape.lineTo(-1.65,.5);
shape.lineTo(-.8,-.3);
shape.close();

const mesh=new THREE.Mesh(
new THREE.ShapeGeometry(shape),
new THREE.MeshStandardMaterial({
color:0x087e75,side:THREE.DoubleSide,
transparent:true,opacity:.9
})
);
mesh.position.z=side*.15;
pivot.add(mesh);
return pivot;
}
const wings=[makeWing(1),makeWing(-1)];

// Fire breath
const fireMat=new THREE.MeshBasicMaterial({
color:0xff5500,transparent:true,opacity:.9
});
const fire=new THREE.Mesh(new THREE.ConeGeometry(.45,2,16),fireMat);
fire.position.set(2.8,.75,0);
fire.rotation.z=-Math.PI/2;
dragon.add(fire);
fire.visible=false;

// Background stars
for(let i=0;i<180;i++){
const star=new THREE.Mesh(
new THREE.SphereGeometry(.025,6,6),
new THREE.MeshBasicMaterial({color:0x9edfff})
);
star.position.set(
(Math.random()-.5)*35,
(Math.random()-.5)*20,
-5-Math.random()*15
);
scene.add(star);
}

// Control state
let targetX=0,targetY=0;
let active=false,flying=false,fireOn=false;
let handFound=false;
const status=document.getElementById('status');

const video=document.getElementById('video');

const hands=new Hands({
locateFile:(file)=>`https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
});
hands.setOptions({
maxNumHands:1,
modelComplexity:1,
minDetectionConfidence:.65,
minTrackingConfidence:.65
});

hands.onResults(results=>{
if(!active)return;

if(results.multiHandLandmarks?.length){
handFound=true;
const p=results.multiHandLandmarks[0];

// Hand position
const x=1-p[9].x;
const y=p[9].y;

const indexUp=p[8].y<p[6].y;
const middleDown=p[12].y>p[10].y;
const ringDown=p[16].y>p[14].y;
const pinkyDown=p[20].y>p[18].y;

const fist=middleDown&&ringDown&&pinkyDown&&
p[8].y>p[6].y;

const pinch=Math.hypot(
p[4].x-p[8].x,p[4].y-p[8].y
)<.055;

if(fist){
status.textContent='✊ DRAGON STOPPED';
flying=false;fireOn=false;
}else{
targetX=(x-.5)*9;
targetY=(.5-y)*5;
flying=indexUp;
fireOn=pinch;
status.textContent=pinch?'🔥 FIRE BREATH!':
indexUp?'🐉 DRAGON FLYING':'🖐 DRAGON MOVING';
}
}else{
handFound=false;
flying=false;
fireOn=false;
status.textContent='Show your hand to camera';
}
});

document.getElementById('start').onclick=async()=>{
try{
const stream=await navigator.mediaDevices.getUserMedia({
video:{facingMode:'user',width:640,height:480}
});
video.srcObject=stream;
await video.play();
active=true;
document.getElementById('start').style.display='none';
status.textContent='Camera ready! Show your hand';

async function track(){
if(active&&video.readyState>=2){
await hands.send({image:video});
}
if(active)requestAnimationFrame(track);
}
track();
}catch(e){
status.textContent='Camera permission required: '+e.message;
}
};

let clock=new THREE.Clock();

function animate(){
requestAnimationFrame(animate);
const t=clock.getElapsedTime();

dragon.position.x+=(targetX-dragon.position.x)*.06;
dragon.position.y+=(targetY-dragon.position.y+(flying?1.3:0))*.06;

dragon.rotation.z+=(Math.sin(t*1.5)*.035-dragon.rotation.z)*.03;
dragon.rotation.y+=(Math.sin(t)*.08-dragon.rotation.y)*.03;

wings[0].rotation.y=Math.sin(t*8)*.6;
wings[1].rotation.y=-Math.sin(t*8)*.6;

dragon.position.y+=Math.sin(t*2)*.008;
fire.visible=fireOn;
fire.scale.setScalar(.8+Math.random()*.4);

renderer.render(scene,camera);
}
animate();

addEventListener('resize',()=>{
camera.aspect=innerWidth/innerHeight;
camera.updateProjectionMatrix();
renderer.setSize(innerWidth,innerHeight);
});
</script>
</body>
</html>
