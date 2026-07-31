// main.js — 3D Medieval Village with 32 enemies (8 squads x 4) — Gujarati comments

let scene, camera, renderer;
let player = { pos: new THREE.Vector3(0, 1.6, 0), yaw: 0, pitch: 0 };
const fireballs = [];
const ENEMY_COUNT = 32;
const squads = [];
const enemies = new Array(ENEMY_COUNT).fill(null);
let score = 0;
const clock = new THREE.Clock();

// Init
init();
animate();

function init() {
  // Scene & camera
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x9bd9a8); // light green sky tone

  camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.copy(player.pos);

  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.setSize(window.innerWidth, window.innerHeight);
  document.body.appendChild(renderer.domElement);

  // Lights
  const hemi = new THREE.HemisphereLight(0xffffff, 0x444444, 0.7);
  scene.add(hemi);
  const dir = new THREE.DirectionalLight(0xfff0d6, 0.6);
  dir.position.set(10, 30, 10);
  scene.add(dir);

  // Ground
  const groundMat = new THREE.MeshStandardMaterial({ color: 0x5aa65a });
  const ground = new THREE.Mesh(new THREE.PlaneGeometry(400, 400), groundMat);
  ground.rotation.x = -Math.PI / 2;
  ground.receiveShadow = true;
  scene.add(ground);

  // Fog for depth
  scene.fog = new THREE.FogExp2(0x9bd9a8, 0.002);

  // Build village: castle, houses (15), windmills (2), many trees
  createCastle(new THREE.Vector3(0, 0, -6));
  placeHouses(15);
  createWindmill(new THREE.Vector3(-20, 0, 8));
  createWindmill(new THREE.Vector3(18, 0, 12));
  placeTrees(80);

  // Squad spawn points around circle
  createSquads(8); // 8 squads x 4 = 32 total

  // UI & events
  window.addEventListener('resize', onWindowResize);
  window.addEventListener('mousemove', onMouseMove);
  window.addEventListener('mousedown', onMouseDown);
  window.addEventListener('keydown', onKeyDown);
  window.addEventListener('keyup', onKeyUp);

  // small ground props
  placeProps();

  // initial spawn
  for (let s = 0; s < squads.length; s++) {
    for (let m = 0; m < squads[s].members.length; m++) {
      spawnEnemyAt(s, m);
    }
  }

  updateUI();
}

// ---------- Procedural village elements ----------

function createCastle(pos) {
  // base keep
  const baseGeo = new THREE.BoxGeometry(8, 4, 6);
  const baseMat = new THREE.MeshStandardMaterial({ color: 0x9e9e9e });
  const base = new THREE.Mesh(baseGeo, baseMat);
  base.position.copy(pos).add(new THREE.Vector3(0, 2, 0));
  scene.add(base);

  // towers (4)
  const tGeo = new THREE.CylinderGeometry(0.9, 0.9, 6, 8);
  const tMat = new THREE.MeshStandardMaterial({ color: 0x8a8a8a });
  const offsets = [[-4,3,-3],[4,3,-3],[-4,3,3],[4,3,3]];
  offsets.forEach(o => {
    const t = new THREE.Mesh(tGeo, tMat);
    t.position.copy(pos).add(new THREE.Vector3(o[0], o[1], o[2]));
    scene.add(t);
    const cone = new THREE.Mesh(new THREE.ConeGeometry(1.2, 1.2, 8), new THREE.MeshStandardMaterial({ color: 0x29429f }));
    cone.position.set(t.position.x, t.position.y + 3.1, t.position.z);
    scene.add(cone);
  });
}

function createHouse(pos, scale = 1, color = 0xd8a66a) {
  const body = new THREE.Mesh(new THREE.BoxGeometry(1.4*scale, 1*scale, 1.2*scale), new THREE.MeshStandardMaterial({ color }));
  body.position.copy(pos).add(new THREE.Vector3(0, 0.5*scale, 0));
  scene.add(body);

  const roof = new THREE.Mesh(new THREE.ConeGeometry(0.9*scale, 0.8*scale, 4), new THREE.MeshStandardMaterial({ color: 0x8b2f2f }));
  roof.position.copy(pos).add(new THREE.Vector3(0, 0.5*scale + 0.6*scale, 0));
  roof.rotation.y = Math.PI / 4;
  scene.add(roof);
}

function placeHouses(count) {
  // arrange houses in village cluster around center area
  const ringRadius = 10;
  for (let i = 0; i < count; i++) {
    const a = (i / count) * Math.PI * 2 + (Math.random()-0.5)*0.4;
    const r = ringRadius * (0.6 + Math.random()*0.6);
    const x = Math.cos(a) * r + (Math.random()-0.5)*3;
    const z = Math.sin(a) * r + (Math.random()-0.5)*3;
    const s = 0.8 + Math.random()*0.8;
    const col = 0xd8a66a + Math.floor(Math.random()*0x202020);
    createHouse(new THREE.Vector3(x, 0, z), s, col);
  }
}

function createWindmill(pos) {
  const base = new THREE.Mesh(new THREE.CylinderGeometry(1.2, 1.2, 5, 8), new THREE.MeshStandardMaterial({ color: 0xd9d1c6 }));
  base.position.copy(pos).add(new THREE.Vector3(0, 2.5, 0));
  scene.add(base);

  const top = new THREE.Mesh(new THREE.ConeGeometry(1.4, 0.8, 8), new THREE.MeshStandardMaterial({ color: 0x7b5a3d }));
  top.position.copy(pos).add(new THREE.Vector3(0, 5.1, 0));
  scene.add(top);

  // blades group
  const blades = new THREE.Group();
  for (let i = 0; i < 4; i++) {
    const b = new THREE.BoxGeometry(0.2, 6, 0.6);
    const bm = new THREE.Mesh(b, new THREE.MeshStandardMaterial({ color: 0xffffff }));
    bm.position.set(0, 3, 0);
    const m = new THREE.Mesh(b, new THREE.MeshStandardMaterial({ color: 0xffffff }));
    blades.add(bm);
    bm.rotation.z = i * Math.PI / 2;
  }
  blades.position.copy(pos).add(new THREE.Vector3(0, 4, 0));
  blades.userData = { spin: 0.8 + Math.random()*0.6 };
  scene.add(blades);

  // store for animation
  windmills.push(blades);
}

const windmills = [];

function placeTrees(count) {
  for (let i = 0; i < count; i++) {
    const angle = Math.random() * Math.PI * 2;
    const r = 18 + Math.random() * 60; // many trees around village and far
    const x = Math.cos(angle) * r + (Math.random()-0.5)*6;
    const z = Math.sin(angle) * r + (Math.random()-0.5)*6;
    const h = 1.2 + Math.random()*2.2;
    createTree(new THREE.Vector3(x, 0, z), h);
  }
}

function createTree(pos, height = 2) {
  const trunk = new THREE.Mesh(new THREE.CylinderGeometry(0.15, 0.15, height*0.5, 6), new THREE.MeshStandardMaterial({ color: 0x6b3b2a }));
  trunk.position.copy(pos).add(new THREE.Vector3(0, height*0.25, 0));
  scene.add(trunk);

  const foliage = new THREE.Mesh(new THREE.ConeGeometry(0.9, height, 8), new THREE.MeshStandardMaterial({ color: 0x166b2b }));
  foliage.position.copy(pos).add(new THREE.Vector3(0, height*0.65, 0));
  scene.add(foliage);
}

function placeProps() {
  // small barrels / crates near houses
  for (let i = 0; i < 12; i++) {
    const x = (Math.random()-0.5)*18;
    const z = (Math.random()-0.5)*18;
    const crate = new THREE.Mesh(new THREE.BoxGeometry(0.6,0.6,0.6), new THREE.MeshStandardMaterial({ color: 0x8a5a3a }));
    crate.position.set(x, 0.3, z);
    scene.add(crate);
  }
}

// ---------- Squads & Enemies ----------

function createSquads(numSquads) {
  const radius = 35;
  for (let s = 0; s < numSquads; s++) {
    const angle = (s / numSquads) * Math.PI * 2;
    const spawn = new THREE.Vector3(Math.cos(angle)*radius, 0.5, Math.sin(angle)*radius);
    const members = [];
    for (let m = 0; m < 4; m++) {
      const idx = s*4 + m;
      members.push({ index: idx, alive: false, mesh: null });
    }
    squads.push({ id: s, spawnPoint: spawn, members });
  }
}

// Glow texture for sprites
function createGlowTexture(size = 128, hex = '#66ccff') {
  const canvas = document.createElement('canvas');
  canvas.width = canvas.height = size;
  const ctx = canvas.getContext('2d');
  const grad = ctx.createRadialGradient(size/2, size/2, 0, size/2, size/2, size/2);
  grad.addColorStop(0, hex);
  grad.addColorStop(0.3, hex);
  grad.addColorStop(0.6, 'rgba(0,0,0,0.15)');
  grad.addColorStop(1, 'rgba(0,0,0,0)');
  ctx.fillStyle = grad;
  ctx.fillRect(0,0,size,size);
  return new THREE.CanvasTexture(canvas);
}
const glowTexture = createGlowTexture(128, '#66aaff');

// base geometry/material for enemies
const enemyGeo = new THREE.BoxGeometry(1, 1.2, 1);
const enemyMatBase = new THREE.MeshStandardMaterial({ color: 0x4477ff });

// create enemy mesh helper
function createEnemyMesh(color = 0xff6666) {
  const mat = enemyMatBase.clone();
  mat.color = new THREE.Color(color);
  mat.emissive = new THREE.Color(color).multiplyScalar(0.2);
  mat.transparent = true;
  mat.opacity = 1.0;

  const mesh = new THREE.Mesh(enemyGeo, mat);
  mesh.castShadow = false;

  // glow sprite
  const spriteMat = new THREE.SpriteMaterial({ map: glowTexture, color: color, blending: THREE.AdditiveBlending, transparent: true, opacity: 0.9, depthWrite: false });
  const sprite = new THREE.Sprite(spriteMat);
  sprite.scale.set(2.6, 2.6, 1);
  sprite.position.set(0, 0.6, 0);
  mesh.add(sprite);

  return mesh;
}

// spawn one enemy into given squad/member
function spawnEnemyAt(squadId, memberIdx) {
  const squad = squads[squadId];
  const slot = squad.members[memberIdx];
  const idx = slot.index;

  // remove old if exist
  if (enemies[idx]) {
    scene.remove(enemies[idx]);
    enemies[idx] = null;
  }

  // position offset
  const base = squad.spawnPoint.clone();
  const offset = new THREE.Vector3((Math.random()-0.5)*3.5, 0, (Math.random()-0.5)*3.5);
  const pos = base.add(offset);
  const mesh = createEnemyMesh(0xff8844 + Math.floor(Math.random()*0x004400));
  mesh.position.copy(pos);
  mesh.userData = {
    squadId,
    memberIndex: memberIdx,
    hp: 1,
    alive: true,
    respawning: false,
  };
  mesh.scale.setScalar(0.2);
  mesh.material.opacity = 0.0;
  scene.add(mesh);
  enemies[idx] = mesh;
  slot.alive = true;
  slot.mesh = mesh;

  // fade-in/scale-in
  const duration = 600; // ms
  const start = performance.now();
  function tick() {
    const t = performance.now() - start;
    const k = Math.min(1, t / duration);
    const s = THREE.MathUtils.lerp(0.2, 1.0, k);
    mesh.scale.setScalar(s);
    mesh.material.opacity = k;
    if (k < 1) requestAnimationFrame(tick);
  }
  requestAnimationFrame(tick);
}

// ---------- Controls & shooting ----------

const keys = {};
function onKeyDown(e) { keys[e.key.toLowerCase()] = true; if (e.key === 'r' || e.key === 'R') restart(); }
function onKeyUp(e) { keys[e.key.toLowerCase()] = false; }

let pointerDown = false;
function onMouseMove(e) {
  if (pointerDown || (e.buttons === 1)) {
    const sens = 0.002;
    player.yaw -= (e.movementX || 0) * sens;
    player.pitch -= (e.movementY || 0) * sens;
    player.pitch = Math.max(-Math.PI/2 + 0.1, Math.min(Math.PI/2 - 0.1, player.pitch));
  }
}
function onMouseDown(e) {
  pointerDown = true;
  if (e.button === 0) shootFireball();
}
window.addEventListener('mouseup', () => pointerDown = false);

function shootFireball() {
  const speed = 50;
  const dir = new THREE.Vector3(
    -Math.sin(player.yaw) * Math.cos(player.pitch),
    Math.sin(player.pitch),
    -Math.cos(player.yaw) * Math.cos(player.pitch)
  ).normalize();

  const fbGeo = new THREE.SphereGeometry(0.14, 8, 8);
  const fbMat = new THREE.MeshStandardMaterial({ color: 0xff6a00, emissive: 0xff3d00, emissiveIntensity: 1 });
  const fb = new THREE.Mesh(fbGeo, fbMat);
  fb.position.copy(camera.position).add(dir.clone().multiplyScalar(1.0));
  fb.userData = { vel: dir.multiplyScalar(speed), life: 2.5 };
  scene.add(fb);
  fireballs.push(fb);
}

// ---------- Updates: player, fireballs, enemies ----------

function updatePlayer(dt) {
  const dir = new THREE.Vector3();
  if (keys['w']) dir.z -= 1;
  if (keys['s']) dir.z += 1;
  if (keys['a']) dir.x -= 1;
  if (keys['d']) dir.x += 1;
  dir.normalize();

  const sin = Math.sin(player.yaw), cos = Math.cos(player.yaw);
  const dx = dir.x * cos - dir.z * sin;
  const dz = dir.x * sin + dir.z * cos;
  const speed = 6;
  player.pos.x += dx * speed * dt;
  player.pos.z += dz * speed * dt;

  camera.position.copy(player.pos);
  camera.position.y = player.pos.y;

  camera.rotation.order = "YXZ";
  camera.rotation.y = player.yaw;
  camera.rotation.x = player.pitch;
}

function updateFireballs(dt) {
  for (let i = fireballs.length - 1; i >= 0; i--) {
    const fb = fireballs[i];
    fb.position.add(fb.userData.vel.clone().multiplyScalar(dt));
    fb.userData.life -= dt;
    if (fb.userData.life <= 0) {
      scene.remove(fb);
      fireballs.splice(i, 1);
      continue;
    }
  }
}

function updateEnemies(dt) {
  // animate windmills
  windmills.forEach(w => { w.rotation.z += (w.userData.spin || 1.0) * dt; });

  for (let idx = 0; idx < enemies.length; idx++) {
    const mesh = enemies[idx];
    if (!mesh) continue;
    const ud = mesh.userData;
    const squad = squads[ud.squadId];
    const mIndex = ud.memberIndex;

    // target selection: leader -> player, others -> leader offset
    let targetPos = new THREE.Vector3();
    const leader = squad.members[0].mesh;
    if (mIndex === 0 || !leader) {
      targetPos.copy(player.pos);
    } else {
      const angle = (mIndex / 4) * Math.PI * 2;
      const radius = 1.6 + (mIndex * 0.25);
      const off = new THREE.Vector3(Math.cos(angle)*radius, 0, Math.sin(angle)*radius);
      targetPos.copy(leader.position).add(off);
    }

    // move toward target
    const toTarget = targetPos.clone().sub(mesh.position);
    const dist = toTarget.length();
    if (dist > 0.01) {
      toTarget.normalize();
      const baseSpeed = (mIndex === 0) ? 2.3 : 2.0;
      mesh.position.add(toTarget.multiplyScalar(baseSpeed * dt));
    }

    // face movement/player
    const lookDir = player.pos.clone().sub(mesh.position);
    if (lookDir.lengthSq() > 0.001) mesh.rotation.y = Math.atan2(lookDir.x, lookDir.z);

    // collision with player
    const toPlayer = mesh.position.clone().sub(player.pos);
    if (toPlayer.length() < 1.6) {
      // reached player -> penalize, explode, respawn
      score = Math.max(0, score - 5);
      updateUI();
      spawnExplosion(mesh.position);
      scene.remove(mesh);
      enemies[idx] = null;
      squad.members[mIndex].alive = false;
      squad.members[mIndex].mesh = null;
      setTimeout(() => spawnEnemyAt(ud.squadId, ud.memberIndex), 3000);
      continue;
    }

    // collision with fireballs
    for (let j = fireballs.length - 1; j >= 0; j--) {
      const fb = fireballs[j];
      if (!fb) continue;
      const d = fb.position.distanceTo(mesh.position);
      if (d < 1.0) {
        // hit
        spawnExplosion(mesh.position);
        scene.remove(mesh);
        enemies[idx] = null;
        squad.members[mIndex].alive = false;
        squad.members[mIndex].mesh = null;
        // remove fireball
        scene.remove(fb);
        fireballs.splice(j, 1);
        score += 10;
        updateUI();
        setTimeout(() => spawnEnemyAt(ud.squadId, ud.memberIndex), 3000);
        break;
      }
    }
  }
}

// ---------- Effects ----------

function spawnExplosion(pos) {
  const geo = new THREE.SphereGeometry(0.4, 8, 8);
  const mat = new THREE.MeshBasicMaterial({ color: 0xff8a33, transparent: true, opacity: 0.95 });
  const ex = new THREE.Mesh(geo, mat);
  ex.position.copy(pos);
  scene.add(ex);

  const start = performance.now();
  const duration = 450;
  (function tick() {
    const t = performance.now() - start;
    const k = Math.min(1, t / duration);
    ex.scale.setScalar(1 + k*3);
    mat.opacity = Math.max(0, 1 - k);
    if (k < 1) requestAnimationFrame(tick); else scene.remove(ex);
  })();
}

// ---------- UI and main loop ----------

function updateUI() {
  const scoreEl = document.getElementById('score');
  if (scoreEl) scoreEl.innerText = 'Score: ' + score;
}

function animate() {
  requestAnimationFrame(animate);
  const dt = Math.max(0.001, clock.getDelta());

  updatePlayer(dt);
  updateFireballs(dt);
  updateEnemies(dt);

  renderer.render(scene, camera);
}

function onWindowResize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
}

function restart() {
  // clear fireballs
  for (let fb of fireballs) if (fb) scene.remove(fb);
  fireballs.length = 0;

  // remove enemies
  for (let i = 0; i < enemies.length; i++) {
    if (enemies[i]) { scene.remove(enemies[i]); enemies[i] = null; }
  }
  // reset squads
  squads.forEach(sq => { sq.members.forEach(m => { m.alive = false; m.mesh = null; }); });

  score = 0;
  updateUI();

  setTimeout(() => {
    for (let s = 0; s < squads.length; s++) for (let m = 0; m < squads[s].members.length; m++) spawnEnemyAt(s, m);
  }, 200);
}

// final: ensure UI shown
updateUI();

// End of main.js
