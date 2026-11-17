# Clash-Gatouz0
Gatouz0 his favourite game
<!DOCTYPE html>
<html lang="de">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,user-scalable=no">
<title>Arena Clash</title>
<style>
  html,body{margin:0;height:100%;background:#1b1b2f;display:flex;align-items:center;justify-content:center;overflow:hidden}
  canvas{background:#000;touch-action:none;display:block;border-radius:6px}
</style>
</head>
<body>
<canvas id="c" width="480" height="860"></canvas>
<script>
// ====================================================================
//  ARENA CLASH – ein kleines Echtzeit-Kartenspiel (Eigenentwicklung)
//  Karte antippen/ziehen -> auf deiner (unteren) Arena-Hälfte platzieren.
//  Tasten 1-4 wählen ebenfalls eine Karte.
// ====================================================================
const cv = document.getElementById('c'), ctx = cv.getContext('2d');
const W = 480, H = 860, AH = 720;      // Canvas-Größe / Höhe der Arena
const RT = 340, RB = 380;               // Fluss oben / unten
const BR = [120, 360], BW = 64;         // Brücken (Mitte-x) und Breite
 
function fit() {
  const s = Math.min(innerWidth / W, innerHeight / H);
  cv.style.width = W * s + 'px';
  cv.style.height = H * s + 'px';
}
addEventListener('resize', fit); fit();
 
// ---------------- Karten ----------------
const CARDS = {
  knight:   { name: 'Krieger',  cost: 3, icon: '⚔️', color: '#c0392b', units: 1, hp: 700,  dmg: 75,  speed: 55, range: 6,   cd: 1.1, radius: 12, targets: 'any' },
  archers:  { name: 'Schützen', cost: 3, icon: '🏹', color: '#27ae60', units: 2, hp: 190,  dmg: 35,  speed: 58, range: 120, cd: 0.9, radius: 9,  targets: 'any', proj: true },
  giant:    { name: 'Koloss',   cost: 5, icon: '🗿', color: '#8e6b3f', units: 1, hp: 2100, dmg: 110, speed: 36, range: 6,   cd: 1.5, radius: 18, targets: 'building' },
  goblins:  { name: 'Kobolde',  cost: 2, icon: '👺', color: '#16a085', units: 3, hp: 110,  dmg: 45,  speed: 90, range: 6,   cd: 0.8, radius: 8,  targets: 'any' },
  wizard:   { name: 'Zauberer', cost: 5, icon: '🧙', color: '#8e44ad', units: 1, hp: 430,  dmg: 110, speed: 52, range: 110, cd: 1.4, radius: 11, targets: 'any', proj: true, splash: 38 },
  fireball: { name: 'Feuerball',cost: 4, icon: '🔥', color: '#e67e22', spell: true, radius: 66, dmg: 230, towerDmg: 95 }
};
const AGGRO = 130;
 
// ---------------- Hilfsfunktionen ----------------
const clamp = (v, a, b) => Math.max(a, Math.min(b, v));
const rnd = (a, b) => a + Math.random() * (b - a);
function shuffle(a) { a = a.slice(); for (let i = a.length - 1; i > 0; i--) { const j = Math.floor(Math.random() * (i + 1)); [a[i], a[j]] = [a[j], a[i]]; } return a; }
const nearestBridge = x => BR.reduce((b, c) => Math.abs(c - x) < Math.abs(b - x) ? c : b);
const sideOf = y => y < RT ? 0 : (y > RB ? 1 : 2);
 
// ---------------- Spielzustand ----------------
let G, mouse = { x: 0, y: 0 }, dragging = false, flashT = 0;
 
function mkTower(team, type, x, y) {
  const king = type === 'king';
  G.towers.push({ team, type, x, y, hp: king ? 2400 : 1400, max: king ? 2400 : 1400,
    r: king ? 30 : 24, cd: 0, dead: false, active: !king, range: 150, dmg: king ? 70 : 55, tcd: 0.9 });
}
 
function newGame() {
  G = { units: [], towers: [], projs: [], fx: [], time: 180, over: false, msg: '',
        elixir: 5, eElixir: 5, queue: shuffle(Object.keys(CARDS)), sel: null, eTimer: 4 };
  mkTower(1, 'princess', 110, 150); mkTower(1, 'princess', 370, 150); mkTower(1, 'king', 240, 66);
  mkTower(0, 'princess', 110, 570); mkTower(0, 'princess', 370, 570); mkTower(0, 'king', 240, 654);
}
 
function crowns(team) { // Kronen, die 'team' durch zerstörte Gegnertürme hat
  let c = 0;
  for (const t of G.towers) if (t.team !== team && t.dead) c += t.type === 'king' ? 3 : 1;
  return Math.min(3, c);
}
 
function spawnUnit(key, team, x, y) {
  const d = CARDS[key];
  for (let i = 0; i < d.units; i++) {
    G.units.push({ key, def: d, team, x: x + (i - (d.units - 1) / 2) * 20, y: y + rnd(-6, 6),
      hp: d.hp, max: d.hp, r: d.radius, cd: rnd(0, 0.4), target: null, rt: 0 });
  }
}
 
function castSpell(key, team, x, y) {
  G.fx.push({ k: 'fire', x, y, t: 0.45, max: 0.45, team, key });
}
 
// ---------------- Schaden ----------------
function damage(e, amt) {
  if (e.dead || e.hp <= 0) return;
  e.hp -= amt;
  if (e.type) { // Turm
    if (e.type === 'king') e.active = true;
    if (e.hp <= 0) {
      e.hp = 0; e.dead = true;
      G.fx.push({ k: 'puff', x: e.x, y: e.y, t: 0.7, max: 0.7, r: 50, color: '#ffcc66' });
      if (e.type === 'princess') { const k = G.towers.find(t => t.team === e.team && t.type === 'king'); if (k) k.active = true; }
      if (e.type === 'king') endGame(e.team === 1 ? 'win' : 'lose');
    }
  } else if (e.hp <= 0) {
    G.fx.push({ k: 'puff', x: e.x, y: e.y, t: 0.35, max: 0.35, r: e.r + 8, color: '#ffffff' });
  }
}
 
function endGame(res) {
  if (G.over) return;
  G.over = true;
  G.msg = res === 'win' ? 'SIEG! 🎉' : res === 'lose' ? 'NIEDERLAGE' : 'UNENTSCHIEDEN';
  G.res = res;
}
 
// ---------------- Zielsuche & Wegfindung ----------------
function towerEligible(t, team) {
  if (t.dead || t.team === team) return false;
  if (t.type === 'king') return G.towers.some(o => o.team === t.team && o.type === 'princess' && o.dead);
  return true;
}
 
function findTarget(u) {
  let best = null, bd = 1e9;
  if (u.def.targets !== 'building') {
    for (const o of G.units) {
      if (o.team === u.team || o.hp <= 0) continue;
      const d = Math.hypot(o.x - u.x, o.y - u.y);
      if (d < AGGRO && d < bd) { bd = d; best = o; }
    }
    if (best) return best;
  }
  bd = 1e9;
  for (const t of G.towers) {
    if (!towerEligible(t, u.team)) continue;
    const d = Math.hypot(t.x - u.x, t.y - u.y);
    if (d < bd) { bd = d; best = t; }
  }
  return best;
}
 
function wayPoint(u, tg) {
  const su = sideOf(u.y), st = sideOf(tg.y);
  if (su === st || st === 2) return tg;
  if (su === 2) { // auf der Brücke: geradeaus weiter
    const bx = nearestBridge(u.x);
    return { x: bx, y: st === 0 ? RT - 16 : RB + 16 };
  }
  const bx = BR.reduce((b, c) => (Math.abs(u.x - c) + Math.abs(c - tg.x)) < (Math.abs(u.x - b) + Math.abs(b - tg.x)) ? c : b);
  if (Math.abs(u.x - bx) > BW / 2 - 8) return { x: bx, y: su === 0 ? RT - 14 : RB + 14 };
  return { x: bx, y: su === 0 ? RB + 16 : RT - 16 };
}
 
function attack(u, tg) {
  const d = u.def;
  if (d.proj) {
    G.projs.push({ x: u.x, y: u.y, tg, dmg: d.dmg, team: u.team, splash: d.splash || 0, speed: 300, color: d.color });
  } else damage(tg, d.dmg);
}
 
// ---------------- Update ----------------
function update(dt) {
  if (G.over) { updateFx(dt); return; }
  G.time -= dt;
  const rate = (G.time < 60 ? 2 : 1) / 2.8;
  G.elixir = Math.min(10, G.elixir + dt * rate);
  G.eElixir = Math.min(10, G.eElixir + dt * rate);
  if (G.time <= 0) {
    G.time = 0;
    const a = crowns(0), b = crowns(1);
    endGame(a > b ? 'win' : a < b ? 'lose' : 'draw');
    return;
  }
 
  // Gegner-KI
  G.eTimer -= dt;
  if (G.eTimer <= 0) { enemyPlay(); G.eTimer = rnd(2, 4.5); }
 
  // Einheiten
  for (const u of G.units) {
    if (u.hp <= 0) continue;
    u.cd -= dt; u.rt -= dt;
    let tg = u.target;
    if (!tg || tg.hp <= 0 || tg.dead || u.rt <= 0) { tg = u.target = findTarget(u); u.rt = 0.3; }
    if (!tg) continue;
    const d = Math.hypot(tg.x - u.x, tg.y - u.y);
    if (d - tg.r - u.r <= u.def.range) {
      if (u.cd <= 0) { attack(u, tg); u.cd = u.def.cd; }
    } else {
      const wp = wayPoint(u, tg);
      const dx = wp.x - u.x, dy = wp.y - u.y, l = Math.hypot(dx, dy) || 1;
      u.x += dx / l * u.def.speed * dt;
      u.y += dy / l * u.def.speed * dt;
    }
  }
 
  // Einheiten voneinander trennen
  const U = G.units;
  for (let i = 0; i < U.length; i++) for (let j = i + 1; j < U.length; j++) {
    const a = U[i], b = U[j];
    if (a.hp <= 0 || b.hp <= 0) continue;
    let dx = b.x - a.x, dy = b.y - a.y;
    const d = Math.hypot(dx, dy) || 0.01, m = a.r + b.r;
    if (d < m) {
      const p = (m - d) * 0.5; dx /= d; dy /= d;
      a.x -= dx * p * 0.5; a.y -= dy * p * 0.5;
      b.x += dx * p * 0.5; b.y += dy * p * 0.5;
    }
  }
  // Fluss nur über Brücken
  for (const u of U) {
    u.x = clamp(u.x, 10, W - 10);
    if (u.y > RT && u.y < RB) {
      const bx = nearestBridge(u.x);
      u.x = clamp(u.x, bx - BW / 2 + 4, bx + BW / 2 - 4);
    }
    u.y = clamp(u.y, 10, AH - 10);
  }
 
  // Türme
  for (const t of G.towers) {
    if (t.dead || !t.active) continue;
    t.cd -= dt;
    if (t.cd > 0) continue;
    let best = null, bd = 1e9;
    for (const u of G.units) {
      if (u.team === t.team || u.hp <= 0) continue;
      const d = Math.hypot(u.x - t.x, u.y - t.y);
      if (d - u.r <= t.range && d < bd) { bd = d; best = u; }
    }
    if (best) {
      G.projs.push({ x: t.x, y: t.y, tg: best, dmg: t.dmg, team: t.team, splash: 0, speed: 340, color: t.team === 0 ? '#5dade2' : '#ec7063' });
      t.cd = t.tcd;
    }
  }
 
  // Geschosse
  for (const p of G.projs) {
    const tg = p.tg;
    if (!tg || tg.hp <= 0 || tg.dead) { p.done = true; continue; }
    const dx = tg.x - p.x, dy = tg.y - p.y, d = Math.hypot(dx, dy);
    const step = p.speed * dt;
    if (d <= step + tg.r * 0.5) {
      if (p.splash) {
        for (const o of G.units) {
          if (o.team !== p.team && o.hp > 0 && Math.hypot(o.x - tg.x, o.y - tg.y) <= p.splash + o.r) damage(o, p.dmg);
        }
        G.fx.push({ k: 'puff', x: tg.x, y: tg.y, t: 0.3, max: 0.3, r: p.splash, color: '#c39bd3' });
      } else damage(tg, p.dmg);
      p.done = true;
    } else { p.x += dx / d * step; p.y += dy / d * step; }
  }
  G.projs = G.projs.filter(p => !p.done);
  G.units = G.units.filter(u => u.hp > 0);
  updateFx(dt);
}
 
function updateFx(dt) {
  for (const f of G.fx) {
    f.t -= dt;
    if (f.k === 'fire' && f.t <= 0 && !f.done) {
      f.done = true;
      const d = CARDS[f.key];
      for (const u of G.units) if (u.team !== f.team && u.hp > 0 && Math.hypot(u.x - f.x, u.y - f.y) <= d.radius + u.r) damage(u, d.dmg);
      for (const t of G.towers) if (t.team !== f.team && !t.dead && Math.hypot(t.x - f.x, t.y - f.y) <= d.radius + t.r) damage(t, d.towerDmg);
      G.fx.push({ k: 'boom', x: f.x, y: f.y, t: 0.5, max: 0.5, r: d.radius });
    }
  }
  G.fx = G.fx.filter(f => f.t > 0);
}
 
// ---------------- Gegner-KI ----------------
function enemyPlay() {
  let keys = Object.keys(CARDS).filter(k => CARDS[k].cost <= G.eElixir);
  if (!keys.length) return;
  // manchmal etwas Elixier sparen
  if (G.eElixir < 6 && Math.random() < 0.3) return;
  let k = keys[Math.floor(Math.random() * keys.length)];
  const c = CARDS[k];
  if (c.spell) {
    let best = null, bn = 0;
    for (const u of G.units) {
      if (u.team !== 0) continue;
      const n = G.units.filter(o => o.team === 0 && Math.hypot(o.x - u.x, o.y - u.y) < c.radius).length;
      if (n > bn) { bn = n; best = u; }
    }
    if (best && bn >= 2) { G.eElixir -= c.cost; castSpell(k, 1, best.x, best.y); return; }
    keys = keys.filter(x => !CARDS[x].spell);
    if (!keys.length) return;
    k = keys[Math.floor(Math.random() * keys.length)];
  }
  const cost = CARDS[k].cost;
  // Verteidigung: in die Nähe von Bedrohungen setzen
  const threats = G.units.filter(u => u.team === 0 && u.y < RT + 80);
  let x, y;
  if (threats.length) {
    const t = threats.reduce((a, b) => a.y < b.y ? a : b);
    x = clamp(t.x + rnd(-30, 30), 30, W - 30); y = rnd(180, 270);
  } else {
    x = Math.random() < 0.5 ? rnd(80, 160) : rnd(320, 400); y = rnd(200, 300);
  }
  G.eElixir -= cost;
  spawnUnit(k, 1, x, y);
}
 
// ---------------- Eingabe ----------------
function cardRect(i) { return { x: 66 + i * 102, y: 752, w: 98, h: 102 }; }
function pos(e) {
  const r = cv.getBoundingClientRect();
  return { x: (e.clientX - r.left) * W / r.width, y: (e.clientY - r.top) * H / r.height };
}
 
function validDeploy(key, x, y) {
  if (CARDS[key].spell) return y < AH && y > 0;
  return y > RB + 14 && y < AH - 14 && x > 14 && x < W - 14;
}
 
function tryDeploy(x, y) {
  const key = G.sel; if (!key) return false;
  const c = CARDS[key];
  if (G.elixir < c.cost || !validDeploy(key, x, y)) { flashT = 0.25; return false; }
  G.elixir -= c.cost;
  if (c.spell) castSpell(key, 0, x, y); else spawnUnit(key, 0, x, y);
  const i = G.queue.indexOf(key);
  G.queue.splice(i, 1); G.queue.push(key);
  G.sel = null;
  return true;
}
 
cv.addEventListener('pointerdown', e => {
  const p = pos(e); mouse = p;
  if (G.over) { newGame(); return; }
  if (p.y >= AH) {
    for (let i = 0; i < 4; i++) {
      const r = cardRect(i);
      if (p.x >= r.x && p.x <= r.x + r.w && p.y >= r.y && p.y <= r.y + r.h) {
        const key = G.queue[i];
        G.sel = G.sel === key ? null : key;
        dragging = !!G.sel;
        return;
      }
    }
  } else if (G.sel) tryDeploy(p.x, p.y);
});
cv.addEventListener('pointermove', e => { mouse = pos(e); });
addEventListener('pointerup', e => {
  const p = pos(e);
  if (dragging && G.sel && p.y < AH) tryDeploy(p.x, p.y);
  dragging = false;
});
addEventListener('keydown', e => {
  const n = parseInt(e.key);
  if (n >= 1 && n <= 4 && !G.over) { const k = G.queue[n - 1]; G.sel = G.sel === k ? null : k; }
  if (e.key === 'Escape') G.sel = null;
});
 
// ---------------- Zeichnen ----------------
function bar(x, y, w, f, col) {
  ctx.fillStyle = 'rgba(0,0,0,.55)'; ctx.fillRect(x - w / 2 - 1, y - 1, w + 2, 6);
  ctx.fillStyle = col; ctx.fillRect(x - w / 2, y, w * clamp(f, 0, 1), 4);
}
function icon(txt, x, y, size) {
  ctx.font = size + 'px serif'; ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
  ctx.fillText(txt, x, y);
}
function text(t, x, y, size, col, align, weight) {
  ctx.font = (weight || 'bold') + ' ' + size + 'px sans-serif';
  ctx.fillStyle = col; ctx.textAlign = align || 'center'; ctx.textBaseline = 'middle';
  ctx.fillText(t, x, y);
}
 
function draw() {
  ctx.clearRect(0, 0, W, H);
  // Gras
  for (let i = 0; i < AH / 40; i++) { ctx.fillStyle = i % 2 ? '#5aa845' : '#63b24e'; ctx.fillRect(0, i * 40, W, 40); }
  // Fluss
  ctx.fillStyle = '#3b8fd6'; ctx.fillRect(0, RT, W, RB - RT);
  ctx.fillStyle = 'rgba(255,255,255,.18)';
  for (let x = 0; x < W; x += 60) ctx.fillRect(x + ((performance.now() / 40) % 60), RT + 10, 26, 3);
  // Brücken
  for (const b of BR) {
    ctx.fillStyle = '#9a6b3a'; ctx.fillRect(b - BW / 2, RT - 6, BW, RB - RT + 12);
    ctx.strokeStyle = '#6e4a24'; ctx.lineWidth = 2;
    for (let y = RT - 4; y < RB + 6; y += 9) { ctx.beginPath(); ctx.moveTo(b - BW / 2, y); ctx.lineTo(b + BW / 2, y); ctx.stroke(); }
  }
  // Einsatzzone
  if (G.sel && !CARDS[G.sel].spell) { ctx.fillStyle = 'rgba(255,255,255,.14)'; ctx.fillRect(0, RB, W, AH - RB); }
 
  // Türme
  for (const t of G.towers) {
    const col = t.team === 0 ? '#2e86de' : '#e74c3c';
    if (t.dead) {
      ctx.fillStyle = '#7f8c8d'; ctx.beginPath(); ctx.arc(t.x, t.y, t.r * 0.7, 0, 7); ctx.fill();
      icon('💥', t.x, t.y, 20); continue;
    }
    ctx.fillStyle = col; ctx.beginPath(); ctx.arc(t.x, t.y, t.r, 0, 7); ctx.fill();
    ctx.lineWidth = 4; ctx.strokeStyle = t.type === 'king' ? '#f1c40f' : '#ecf0f1'; ctx.stroke();
    icon(t.type === 'king' ? '👑' : '🏰', t.x, t.y, t.type === 'king' ? 28 : 24);
    bar(t.x, t.y + t.r + 6, 46, t.hp / t.max, col);
    text(Math.ceil(t.hp), t.x, t.y + t.r + 20, 11, '#fff');
  }
 
  // Einheiten
  for (const u of [...G.units].sort((a, b) => a.y - b.y)) {
    ctx.fillStyle = 'rgba(0,0,0,.25)'; ctx.beginPath(); ctx.ellipse(u.x, u.y + u.r * 0.8, u.r, u.r * 0.5, 0, 0, 7); ctx.fill();
    ctx.fillStyle = u.def.color; ctx.beginPath(); ctx.arc(u.x, u.y, u.r, 0, 7); ctx.fill();
    ctx.lineWidth = 3; ctx.strokeStyle = u.team === 0 ? '#3498db' : '#e74c3c'; ctx.stroke();
    icon(u.def.icon, u.x, u.y + 1, u.r * 1.4);
    if (u.hp < u.max) bar(u.x, u.y - u.r - 9, Math.max(22, u.r * 2), u.hp / u.max, u.team === 0 ? '#3498db' : '#e74c3c');
  }
 
  // Geschosse
  for (const p of G.projs) { ctx.fillStyle = p.color; ctx.beginPath(); ctx.arc(p.x, p.y, p.splash ? 6 : 4, 0, 7); ctx.fill(); }
 
  // Effekte
  for (const f of G.fx) {
    const k = 1 - f.t / f.max;
    if (f.k === 'puff') {
      ctx.globalAlpha = 1 - k; ctx.fillStyle = f.color; ctx.beginPath(); ctx.arc(f.x, f.y, f.r * (0.4 + k), 0, 7); ctx.fill(); ctx.globalAlpha = 1;
    } else if (f.k === 'fire') { // fallender Feuerball
      const d = CARDS[f.key];
      ctx.strokeStyle = 'rgba(230,126,34,.6)'; ctx.lineWidth = 2; ctx.setLineDash([6, 6]);
      ctx.beginPath(); ctx.arc(f.x, f.y, d.radius, 0, 7); ctx.stroke(); ctx.setLineDash([]);
      icon('🔥', f.x, f.y - (1 - k) * 160, 30);
    } else if (f.k === 'boom') {
      ctx.globalAlpha = 1 - k; ctx.fillStyle = '#f39c12'; ctx.beginPath(); ctx.arc(f.x, f.y, f.r * (0.5 + k * 0.6), 0, 7); ctx.fill();
      ctx.fillStyle = '#f7dc6f'; ctx.beginPath(); ctx.arc(f.x, f.y, f.r * 0.5 * (1 - k), 0, 7); ctx.fill(); ctx.globalAlpha = 1;
    }
  }
 
  // Vorschau
  if (G.sel && mouse.y < AH && !G.over) {
    const c = CARDS[G.sel], ok = validDeploy(G.sel, mouse.x, mouse.y) && G.elixir >= c.cost;
    ctx.globalAlpha = 0.75;
    if (c.spell) {
      ctx.fillStyle = ok ? 'rgba(230,126,34,.35)' : 'rgba(231,76,60,.35)';
      ctx.beginPath(); ctx.arc(mouse.x, mouse.y, c.radius, 0, 7); ctx.fill();
    } else {
      ctx.fillStyle = ok ? c.color : '#e74c3c'; ctx.beginPath(); ctx.arc(mouse.x, mouse.y, c.radius, 0, 7); ctx.fill();
    }
    icon(c.icon, mouse.x, mouse.y, c.spell ? 28 : c.radius * 1.4);
    ctx.globalAlpha = 1;
  }
 
  drawHud();
  if (G.over) drawOver();
}
 
function drawHud() {
  // Zeit & Kronen
  const m = Math.floor(G.time / 60), s = Math.floor(G.time % 60);
  ctx.fillStyle = 'rgba(0,0,0,.55)'; ctx.fillRect(6, 6, 118, 54);
  text(m + ':' + String(s).padStart(2, '0') + (G.time < 60 ? '  2x' : ''), 65, 20, 18, G.time < 60 ? '#f9e79f' : '#fff');
  text('Du 👑' + crowns(0), 36, 46, 14, '#85c1e9');
  text('Gegner 👑' + crowns(1), 94, 46, 14, '#f1948a');
 
  // Panel
  ctx.fillStyle = '#2c2c4a'; ctx.fillRect(0, AH, W, H - AH);
  // Elixier
  const ex = 66, ey = 728, ew = 404, eh = 16;
  ctx.fillStyle = '#1a1a2e'; ctx.fillRect(ex, ey, ew, eh);
  const g = ctx.createLinearGradient(ex, 0, ex + ew, 0); g.addColorStop(0, '#b24bf3'); g.addColorStop(1, '#8e24aa');
  ctx.fillStyle = g; ctx.fillRect(ex, ey, ew * G.elixir / 10, eh);
  ctx.strokeStyle = 'rgba(255,255,255,.25)'; ctx.lineWidth = 1;
  for (let i = 1; i < 10; i++) { ctx.beginPath(); ctx.moveTo(ex + ew * i / 10, ey); ctx.lineTo(ex + ew * i / 10, ey + eh); ctx.stroke(); }
  text('💧' + Math.floor(G.elixir), 30, 738, 20, '#e8c6ff');
 
  // Nächste Karte
  const nk = CARDS[G.queue[4]];
  text('Nächste', 33, 764, 10, '#aaa', 'center', 'normal');
  ctx.fillStyle = nk.color; ctx.globalAlpha = 0.8; ctx.fillRect(8, 774, 50, 60); ctx.globalAlpha = 1;
  icon(nk.icon, 33, 800, 24); text(nk.cost, 33, 824, 13, '#fff');
 
  // Handkarten
  for (let i = 0; i < 4; i++) {
    const key = G.queue[i], c = CARDS[key], r = cardRect(i), sel = G.sel === key, can = G.elixir >= c.cost;
    const oy = sel ? -10 : 0;
    ctx.fillStyle = c.color; ctx.globalAlpha = can ? 1 : 0.45;
    ctx.fillRect(r.x, r.y + oy, r.w, r.h);
    ctx.lineWidth = sel ? 4 : 2; ctx.strokeStyle = sel ? (flashT > 0 && !can ? '#e74c3c' : '#fff') : 'rgba(0,0,0,.4)';
    ctx.strokeRect(r.x, r.y + oy, r.w, r.h);
    icon(c.icon, r.x + r.w / 2, r.y + oy + 42, 38);
    text(c.name, r.x + r.w / 2, r.y + oy + 84, 14, '#fff');
    ctx.globalAlpha = 1;
    ctx.fillStyle = '#9b27af'; ctx.beginPath(); ctx.arc(r.x + 14, r.y + oy + 14, 12, 0, 7); ctx.fill();
    ctx.strokeStyle = '#fff'; ctx.lineWidth = 2; ctx.stroke();
    text(c.cost, r.x + 14, r.y + oy + 15, 14, '#fff');
    text(i + 1, r.x + r.w - 10, r.y + oy + 10, 11, 'rgba(255,255,255,.6)');
  }
}
 
function drawOver() {
  ctx.fillStyle = 'rgba(0,0,0,.65)'; ctx.fillRect(0, 0, W, H);
  const col = G.res === 'win' ? '#f1c40f' : G.res === 'lose' ? '#e74c3c' : '#ecf0f1';
  text(G.msg, W / 2, 330, 46, col);
  text('Kronen  Du ' + crowns(0) + ' : ' + crowns(1) + ' Gegner', W / 2, 395, 20, '#fff');
  text('Tippen / Klicken für Neustart', W / 2, 450, 18, '#bbb', 'center', 'normal');
}
 
// ---------------- Hauptschleife ----------------
let last = performance.now();
function loop(now) {
  const dt = Math.min(0.05, (now - last) / 1000); last = now;
  flashT = Math.max(0, flashT - dt);
  update(dt);
  draw();
  requestAnimationFrame(loop);
}
newGame();
requestAnimationFrame(loop);
</script>
</body>
</html>
