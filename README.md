<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
<title>Astana Courier 2.5D</title>
<style>
  * {
    box-sizing: border-box;
    user-select: none;
    -webkit-user-select: none;
    -webkit-tap-highlight-color: transparent;
  }

  html, body {
    margin: 0;
    padding: 0;
    overflow: hidden;
    width: 100%;
    height: 100%;
    background: #08111c;
    font-family: Arial, sans-serif;
    touch-action: none;
  }

  canvas {
    display: block;
    width: 100vw;
    height: 100vh;
    background: #42d96b;
    image-rendering: pixelated;
    touch-action: none;
  }

  #mobileControls {
    position: fixed;
    inset: 0;
    z-index: 10;
    pointer-events: none;
  }

  #joyBase {
    position: absolute;
    left: 26px;
    bottom: 32px;
    width: 132px;
    height: 132px;
    border-radius: 50%;
    background: rgba(0,0,0,0.38);
    border: 3px solid rgba(255,255,255,0.35);
    pointer-events: auto;
    touch-action: none;
  }

  #joyKnob {
    position: absolute;
    left: 38px;
    top: 38px;
    width: 56px;
    height: 56px;
    border-radius: 50%;
    background: #00eaff;
    border: 3px solid #fff;
    box-shadow: 0 0 18px #00eaff;
  }

  .btn {
    position: absolute;
    right: 26px;
    width: 82px;
    height: 82px;
    border-radius: 50%;
    border: 3px solid rgba(255,255,255,0.8);
    color: white;
    font-weight: 900;
    font-size: 18px;
    background: rgba(0,0,0,0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    pointer-events: auto;
    touch-action: none;
  }

  #btnE {
    bottom: 36px;
    background: rgba(255, 214, 0, 0.78);
    color: #111;
  }

  #btnSprint {
    bottom: 130px;
    background: rgba(0, 255, 135, 0.6);
  }

  #topHint {
    position: fixed;
    left: 10px;
    top: 10px;
    z-index: 20;
    color: white;
    background: rgba(0,0,0,0.58);
    border: 1px solid rgba(255,255,255,0.25);
    border-radius: 10px;
    padding: 8px 10px;
    font-size: 12px;
    line-height: 1.35;
    max-width: 335px;
  }
</style>
</head>
<body>

<canvas id="game"></canvas>

<div id="topHint">
  <b>Astana Courier 2.5D</b><br>
  Телефон: джойстик + БЕГ + E<br>
  ПК: WASD/стрелки, Shift, E
</div>

<div id="mobileControls">
  <div id="joyBase">
    <div id="joyKnob"></div>
  </div>
  <div id="btnSprint" class="btn">БЕГ</div>
  <div id="btnE" class="btn">E</div>
</div>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

function resize() {
  canvas.width = window.innerWidth;
  canvas.height = window.innerHeight;
}
window.addEventListener("resize", resize);
resize();

document.addEventListener("touchmove", e => e.preventDefault(), { passive: false });

const WORLD_W = 3000;
const WORLD_H = 2100;
const ROAD_W = 96;

const verticalRoads = [
  { x: 260, name: "пр. Тәуелсіздік" },
  { x: 620, name: "ул. Достык" },
  { x: 980, name: "ул. Сығанақ" },
  { x: 1340, name: "пр. Нуржол" },
  { x: 1700, name: "ул. Кунаева" },
  { x: 2060, name: "ул. Сарайшык" },
  { x: 2440, name: "пр. Кабанбай Батыра" },
  { x: 2780, name: "пр. Мәңгілік Ел" }
];

const horizontalRoads = [
  { y: 260, name: "пр. Туран" },
  { y: 560, name: "пр. Кабанбай Батыра" },
  { y: 860, name: "ул. Туркестан" },
  { y: 1160, name: "пр. Мәңгілік Ел" },
  { y: 1460, name: "пр. Тәуелсіздік" },
  { y: 1760, name: "ул. Акмешит" },
  { y: 1980, name: "Южная магистраль" }
];

const keys = {};
let interactPressed = false;
let mobileSprint = false;

window.addEventListener("keydown", e => {
  keys[e.key.toLowerCase()] = true;
  if (e.key.toLowerCase() === "e") interactPressed = true;
});
window.addEventListener("keyup", e => keys[e.key.toLowerCase()] = false);

const joy = { active: false, id: null, x: 0, y: 0, max: 46 };
const joyBase = document.getElementById("joyBase");
const joyKnob = document.getElementById("joyKnob");
const btnE = document.getElementById("btnE");
const btnSprint = document.getElementById("btnSprint");

function setKnob(x, y) {
  joyKnob.style.left = 38 + x + "px";
  joyKnob.style.top = 38 + y + "px";
}

function updateJoy(e) {
  const r = joyBase.getBoundingClientRect();
  const cx = r.left + r.width / 2;
  const cy = r.top + r.height / 2;

  let dx = e.clientX - cx;
  let dy = e.clientY - cy;
  const len = Math.hypot(dx, dy);

  if (len > joy.max) {
    dx = dx / len * joy.max;
    dy = dy / len * joy.max;
  }

  joy.x = dx / joy.max;
  joy.y = dy / joy.max;

  if (Math.abs(joy.x) < 0.12) joy.x = 0;
  if (Math.abs(joy.y) < 0.12) joy.y = 0;

  setKnob(dx, dy);
}

function resetJoy() {
  joy.active = false;
  joy.id = null;
  joy.x = 0;
  joy.y = 0;
  setKnob(0, 0);
}

joyBase.addEventListener("pointerdown", e => {
  joy.active = true;
  joy.id = e.pointerId;
  joyBase.setPointerCapture(e.pointerId);
  updateJoy(e);
});
joyBase.addEventListener("pointermove", e => {
  if (joy.active && e.pointerId === joy.id) updateJoy(e);
});
joyBase.addEventListener("pointerup", resetJoy);
joyBase.addEventListener("pointercancel", resetJoy);

btnE.addEventListener("pointerdown", e => {
  e.preventDefault();
  interactPressed = true;
});

btnSprint.addEventListener("pointerdown", e => {
  e.preventDefault();
  mobileSprint = true;
});
btnSprint.addEventListener("pointerup", e => {
  e.preventDefault();
  mobileSprint = false;
});
btnSprint.addEventListener("pointercancel", () => mobileSprint = false);
btnSprint.addEventListener("pointerleave", () => mobileSprint = false);

function clamp(v, a, b) {
  return Math.max(a, Math.min(b, v));
}

function rand(a, b) {
  return a + Math.random() * (b - a);
}

function pick(arr) {
  return arr[Math.floor(Math.random() * arr.length)];
}

function dist(a, b) {
  return Math.hypot(a.x - b.x, a.y - b.y);
}

function circleRectCollision(cx, cy, r, rect) {
  const nx = clamp(cx, rect.x, rect.x + rect.w);
  const ny = clamp(cy, rect.y, rect.y + rect.d);
  return Math.hypot(cx - nx, cy - ny) < r;
}

function shade(hex, amount) {
  const num = parseInt(hex.replace("#", ""), 16);
  let r = (num >> 16) + amount;
  let g = ((num >> 8) & 255) + amount;
  let b = (num & 255) + amount;

  r = clamp(r, 0, 255);
  g = clamp(g, 0, 255);
  b = clamp(b, 0, 255);

  return `rgb(${r},${g},${b})`;
}

const player = {
  x: 260,
  y: 260,
  r: 15,
  speed: 185,
  sprint: 285,
  hp: 100,
  money: 0,
  level: 1,
  deliveries: 0,
  hasPackage: false,
  hitTimer: 0
};

let camera = { x: 0, y: 0 };
let buildings = [];
let cars = [];
let particles = [];
let quest = null;
let message = "Найди жёлтую точку и нажми E";
let messageTimer = 4;

function showMessage(text, time = 3) {
  message = text;
  messageTimer = time;
}

const places = [
  {
    name: "Байтерек",
    address: "проспект Нуржол, 14",
    x: 1340,
    y: 560,
    type: "landmark",
    landmark: "baiterek"
  },
  {
    name: "Хан Шатыр",
    address: "проспект Туран, 37",
    x: 980,
    y: 260,
    type: "landmark",
    landmark: "tent"
  },
  {
    name: "Astana Opera",
    address: "улица Динмухамеда Кунаева, 1",
    x: 1700,
    y: 560,
    type: "landmark",
    landmark: "opera"
  },
  {
    name: "ТРЦ Keruen",
    address: "проспект Достык, 9",
    x: 1340,
    y: 860,
    type: "mall"
  },
  {
    name: "Abu Dhabi Plaza",
    address: "улица Сығанақ, 16/5",
    x: 980,
    y: 860,
    type: "tower"
  },
  {
    name: "Астана Арена",
    address: "проспект Кабанбай Батыра, 48",
    x: 2060,
    y: 560,
    type: "stadium"
  },
  {
    name: "Nazarbayev University",
    address: "проспект Кабанбай Батыра, 53",
    x: 2440,
    y: 860,
    type: "campus"
  },
  {
    name: "MEGA Silk Way",
    address: "проспект Кабанбай Батыра, 62",
    x: 2440,
    y: 560,
    type: "mall"
  },
  {
    name: "EXPO / Нур Әлем",
    address: "проспект Мәңгілік Ел, 55",
    x: 2780,
    y: 860,
    type: "landmark",
    landmark: "expo"
  },
  {
    name: "Дворец мира и согласия",
    address: "проспект Тәуелсіздік, 57",
    x: 1340,
    y: 1460,
    type: "landmark",
    landmark: "pyramid"
  },
  {
    name: "Мечеть Хазрет Султан",
    address: "проспект Тәуелсіздік, 48",
    x: 1700,
    y: 1460,
    type: "landmark",
    landmark: "mosque"
  },
  {
    name: "Национальный музей РК",
    address: "проспект Тәуелсіздік, 54",
    x: 2060,
    y: 1460,
    type: "museum"
  },
  {
    name: "Ботанический сад",
    address: "район проспекта Кабанбай Батыра",
    x: 2440,
    y: 1160,
    type: "park"
  },
  {
    name: "Вокзал Нурлы Жол",
    address: "проспект Мәңгілік Ел, район вокзала",
    x: 2780,
    y: 1760,
    type: "station"
  },
  {
    name: "Левый берег Hub",
    address: "район проспекта Нуржол",
    x: 620,
    y: 560,
    type: "hub"
  },
  {
    name: "Курьерский склад",
    address: "игровой склад на проспекте Туран",
    x: 260,
    y: 260,
    type: "warehouse"
  },
  {
    name: "Кофейня Достык",
    address: "улица Достык, игровой квартал",
    x: 620,
    y: 860,
    type: "food"
  },
  {
    name: "Маркет Туркестан",
    address: "улица Туркестан, игровой квартал",
    x: 2060,
    y: 1160,
    type: "shop"
  }
];

const bright = [
  "#ff3bd5", "#00d9ff", "#ffe600", "#ff7b00", "#7cff00",
  "#aa66ff", "#ff304f", "#00ff95", "#2f8cff", "#ffffff"
];

function addBuilding(x, y, w, d, z, color, kind = "normal", label = "") {
  buildings.push({ x, y, w, d, z, color, kind, label });
}

function generateCity() {
  buildings = [];

  for (let i = 0; i < verticalRoads.length - 1; i++) {
    for (let j = 0; j < horizontalRoads.length - 1; j++) {
      const x1 = verticalRoads[i].x + ROAD_W / 2 + 18;
      const x2 = verticalRoads[i + 1].x - ROAD_W / 2 - 18;
      const y1 = horizontalRoads[j].y + ROAD_W / 2 + 18;
      const y2 = horizontalRoads[j + 1].y - ROAD_W / 2 - 18;

      if (x2 - x1 < 90 || y2 - y1 < 80) continue;
      if (Math.random() < 0.22) continue;

      const count = Math.random() < 0.55 ? 1 : 2;

      for (let k = 0; k < count; k++) {
        const w = rand(70, Math.min(190, x2 - x1));
        const d = rand(60, Math.min(150, y2 - y1));
        const x = rand(x1, x2 - w);
        const y = rand(y1, y2 - d);
        const z = rand(25, 95);

        addBuilding(x, y, w, d, z, pick(bright), "normal");
      }
    }
  }

  for (const p of places) {
    if (p.landmark === "baiterek") {
      addBuilding(p.x + 42, p.y - 95, 135, 135, 110, "#ffd400", "baiterek", p.name);
    } else if (p.landmark === "tent") {
      addBuilding(p.x + 48, p.y - 105, 230, 145, 95, "#00eaff", "tent", p.name);
    } else if (p.landmark === "pyramid") {
      addBuilding(p.x + 45, p.y - 115, 190, 190, 120, "#c77dff", "pyramid", p.name);
    } else if (p.landmark === "expo") {
      addBuilding(p.x - 240, p.y - 100, 170, 170, 90, "#00b7ff", "expo", p.name);
    } else if (p.landmark === "mosque") {
      addBuilding(p.x + 45, p.y - 110, 210, 170, 80, "#ffffff", "mosque", p.name);
    } else if (p.type === "tower") {
      addBuilding(p.x + 42, p.y - 110, 135, 130, 150, "#2f8cff", "tower", p.name);
    } else if (p.type === "stadium") {
      addBuilding(p.x + 40, p.y - 90, 230, 150, 55, "#7cff00", "stadium", p.name);
    } else if (p.type === "park") {
      // парк рисуется отдельно, здание не нужно
    } else {
      addBuilding(p.x + 45, p.y - 85, 155, 120, rand(45, 90), pick(bright), "named", p.name);
    }
  }
}

function createCars() {
  cars = [];
  const count = 20 + player.level * 5;

  for (let i = 0; i < count; i++) {
    const horizontal = Math.random() < 0.55;

    if (horizontal) {
      const road = pick(horizontalRoads);
      const lane = pick([-24, 24]);
      const dir = Math.random() < 0.5 ? 1 : -1;

      cars.push({
        x: rand(0, WORLD_W),
        y: road.y + lane,
        w: 48,
        d: 25,
        dx: dir,
        dy: 0,
        speed: rand(90, 175) + player.level * 8,
        color: pick(["#ff304f", "#00eaff", "#ffe600", "#7cff00", "#ff7b00", "#ffffff"])
      });
    } else {
      const road = pick(verticalRoads);
      const lane = pick([-24, 24]);
      const dir = Math.random() < 0.5 ? 1 : -1;

      cars.push({
        x: road.x + lane,
        y: rand(0, WORLD_H),
        w: 25,
        d: 48,
        dx: 0,
        dy: dir,
        speed: rand(90, 175) + player.level * 8,
        color: pick(["#ff304f", "#00eaff", "#ffe600", "#7cff00", "#ff7b00", "#ffffff"])
      });
    }
  }
}

const questTypes = [
  {
    name: "Еда",
    reward: 1.0,
    time: 0.82,
    text: "Горячий заказ"
  },
  {
    name: "Документы",
    reward: 1.25,
    time: 1.0,
    text: "Важные документы"
  },
  {
    name: "Посылка",
    reward: 1.1,
    time: 1.1,
    text: "Обычная посылка"
  },
  {
    name: "Срочно",
    reward: 1.55,
    time: 0.7,
    text: "Срочная доставка"
  }
];

function createQuest() {
  let pickup = pick(places);
  let drop = pick(places);

  while (drop === pickup || dist(pickup, drop) < 600) {
    drop = pick(places);
  }

  const type = pick(questTypes);
  const d = dist(pickup, drop);
  const maxTime = clamp((32 + d / 85 - player.level * 1.4) * type.time, 22, 105);
  const reward = Math.floor((55 + d / 13 + player.level * 18) * type.reward);

  quest = {
    pickup,
    drop,
    type,
    reward,
    timeLeft: maxTime,
    maxTime
  };

  player.hasPackage = false;
  showMessage(`${type.text}: ${pickup.name} → ${drop.name}`, 4);
}

function canMoveTo(x, y) {
  if (x < player.r || y < player.r || x > WORLD_W - player.r || y > WORLD_H - player.r) {
    return false;
  }

  for (const b of buildings) {
    if (circleRectCollision(x, y, player.r, b)) return false;
  }

  return true;
}

function movePlayer(dt) {
  let dx = 0;
  let dy = 0;

  if (keys["w"] || keys["arrowup"]) dy -= 1;
  if (keys["s"] || keys["arrowdown"]) dy += 1;
  if (keys["a"] || keys["arrowleft"]) dx -= 1;
  if (keys["d"] || keys["arrowright"]) dx += 1;

  if (joy.x !== 0 || joy.y !== 0) {
    dx = joy.x;
    dy = joy.y;
  }

  const len = Math.hypot(dx, dy);
  if (len > 1) {
    dx /= len;
    dy /= len;
  }

  const sprinting = keys["shift"] || mobileSprint;
  const speed = sprinting ? player.sprint : player.speed;

  const nx = player.x + dx * speed * dt;
  const ny = player.y + dy * speed * dt;

  if (canMoveTo(nx, player.y)) player.x = nx;
  if (canMoveTo(player.x, ny)) player.y = ny;

  if (len > 0.1 && Math.random() < 0.32) {
    particles.push({
      x: player.x - dx * 12,
      y: player.y - dy * 12,
      vx: rand(-20, 20),
      vy: rand(-20, 20),
      life: 0.25,
      color: sprinting ? "#ffe600" : "#ffffff"
    });
  }
}

function updateCars(dt) {
  for (const c of cars) {
    c.x += c.dx * c.speed * dt;
    c.y += c.dy * c.speed * dt;

    if (c.x < -80) c.x = WORLD_W + 80;
    if (c.x > WORLD_W + 80) c.x = -80;
    if (c.y < -80) c.y = WORLD_H + 80;
    if (c.y > WORLD_H + 80) c.y = -80;

    const rect = {
      x: c.x - c.w / 2,
      y: c.y - c.d / 2,
      w: c.w,
      d: c.d
    };

    if (player.hitTimer <= 0 && circleRectCollision(player.x, player.y, player.r, rect)) {
      player.hp -= 16;
      player.hitTimer = 1.0;
      showMessage("Машина сбила курьера! Осторожнее на дорогах", 2.5);

      for (let i = 0; i < 14; i++) {
        particles.push({
          x: player.x,
          y: player.y,
          vx: rand(-180, 180),
          vy: rand(-180, 180),
          life: 0.65,
          color: "#ff304f"
        });
      }

      if (player.hp <= 0) {
        player.hp = 100;
        player.money = Math.max(0, player.money - 50);
        player.x = 260;
        player.y = 260;
        player.hasPackage = false;
        createQuest();
        showMessage("Ты потерял заказ. Штраф 50₸. Респавн на складе", 4);
      }
    }
  }
}

function updateQuest(dt) {
  if (!quest) return;

  if (player.hasPackage) {
    quest.timeLeft -= dt;

    if (quest.timeLeft <= 0) {
      player.hasPackage = false;
      player.money = Math.max(0, player.money - 25);
      createQuest();
      showMessage("Время вышло! Штраф 25₸", 3);
      return;
    }
  }

  const target = player.hasPackage ? quest.drop : quest.pickup;
  const near = Math.hypot(player.x - target.x, player.y - target.y) < 62;

  if (near && interactPressed) {
    if (!player.hasPackage) {
      player.hasPackage = true;
      showMessage(`Взял: ${quest.type.name}. Адрес: ${quest.drop.address}`, 4);
    } else {
      const bonus = Math.floor(Math.max(0, quest.timeLeft) * 2);
      const total = quest.reward + bonus;

      player.money += total;
      player.deliveries++;
      player.hasPackage = false;

      for (let i = 0; i < 36; i++) {
        particles.push({
          x: player.x,
          y: player.y,
          vx: rand(-230, 230),
          vy: rand(-230, 230),
          life: 0.9,
          color: pick(["#ffe600", "#00ff95", "#00eaff", "#ff3bd5"])
        });
      }

      showMessage(`Доставлено в ${target.name}! +${total}₸`, 3.5);

      if (player.deliveries % 3 === 0) {
        player.level++;
        player.hp = Math.min(100, player.hp + 25);
        createCars();
        showMessage(`Уровень ${player.level}! Больше трафика и выше награды`, 4);
      }

      createQuest();
    }
  }

  interactPressed = false;
}

function updateParticles(dt) {
  for (const p of particles) {
    p.life -= dt;
    p.x += (p.vx || 0) * dt;
    p.y += (p.vy || 0) * dt;
  }

  particles = particles.filter(p => p.life > 0);
}

function update(dt) {
  movePlayer(dt);
  updateCars(dt);
  updateQuest(dt);
  updateParticles(dt);

  if (player.hitTimer > 0) player.hitTimer -= dt;
  if (messageTimer > 0) messageTimer -= dt;

  camera.x = clamp(player.x - canvas.width / 2, 0, Math.max(0, WORLD_W - canvas.width));
  camera.y = clamp(player.y - canvas.height / 2, 0, Math.max(0, WORLD_H - canvas.height));
}

function sx(x) { return Math.floor(x - camera.x); }
function sy(y) { return Math.floor(y - camera.y); }

function drawBackground() {
  ctx.fillStyle = "#45d96a";
  ctx.fillRect(sx(0), sy(0), WORLD_W, WORLD_H);

  // Парки
  ctx.fillStyle = "#1fb653";
  ctx.fillRect(sx(90), sy(90), 230, 135);
  ctx.fillRect(sx(2250), sy(1030), 390, 210);
  ctx.fillRect(sx(2050), sy(1540), 310, 180);

  // Река Ишим, игровая версия
  ctx.fillStyle = "#16b8ff";
  ctx.beginPath();
  ctx.moveTo(sx(0), sy(1000));
  ctx.bezierCurveTo(sx(460), sy(910), sx(820), sy(1110), sx(1300), sy(1015));
  ctx.bezierCurveTo(sx(1800), sy(930), sx(2240), sy(1105), sx(WORLD_W), sy(1000));
  ctx.lineTo(sx(WORLD_W), sy(1088));
  ctx.bezierCurveTo(sx(2240), sy(1195), sx(1800), sy(1010), sx(1300), sy(1105));
  ctx.bezierCurveTo(sx(820), sy(1200), sx(460), sy(1000), sx(0), sy(1100));
  ctx.closePath();
  ctx.fill();

  ctx.strokeStyle = "rgba(255,255,255,0.35)";
  ctx.lineWidth = 3;
  ctx.beginPath();
  ctx.moveTo(sx(0), sy(1045));
  ctx.bezierCurveTo(sx(460), sy(955), sx(820), sy(1155), sx(1300), sy(1060));
  ctx.bezierCurveTo(sx(1800), sy(975), sx(2240), sy(1150), sx(WORLD_W), sy(1045));
  ctx.stroke();
}

function drawRoads() {
  ctx.fillStyle = "#38434f";

  for (const r of verticalRoads) {
    ctx.fillRect(sx(r.x - ROAD_W / 2), sy(0), ROAD_W, WORLD_H);
  }

  for (const r of horizontalRoads) {
    ctx.fillRect(sx(0), sy(r.y - ROAD_W / 2), WORLD_W, ROAD_W);
  }

  ctx.strokeStyle = "#f6ff61";
  ctx.lineWidth = 4;
  ctx.setLineDash([25, 25]);

  for (const r of verticalRoads) {
    ctx.beginPath();
    ctx.moveTo(sx(r.x), sy(0));
    ctx.lineTo(sx(r.x), sy(WORLD_H));
    ctx.stroke();
  }

  for (const r of horizontalRoads) {
    ctx.beginPath();
    ctx.moveTo(sx(0), sy(r.y));
    ctx.lineTo(sx(WORLD_W), sy(r.y));
    ctx.stroke();
  }

  ctx.setLineDash([]);

  // Названия улиц
  ctx.font = "bold 13px Arial";
  ctx.textAlign = "center";
  ctx.fillStyle = "rgba(255,255,255,0.82)";

  for (const r of horizontalRoads) {
    if (sy(r.y) > -40 && sy(r.y) < canvas.height + 40) {
      ctx.fillText(r.name, canvas.width / 2, sy(r.y - 32));
    }
  }

  ctx.save();
  ctx.rotate(-Math.PI / 2);
  for (const r of verticalRoads) {
    const x = sx(r.x - 32);
    if (x > -40 && x < canvas.width + 40) {
      ctx.fillText(r.name, -canvas.height / 2, x);
    }
  }
  ctx.restore();
}

function drawBuilding(b) {
  const x = sx(b.x);
  const y = sy(b.y);
  const w = b.w;
  const d = b.d;
  const z = b.z;
  const roofY = y - z;

  ctx.fillStyle = "rgba(0,0,0,0.28)";
  ctx.fillRect(x + z * 0.22, y + z * 0.24, w + 10, d + 10);

  if (b.kind === "baiterek") {
    ctx.fillStyle = "rgba(0,0,0,0.25)";
    ctx.beginPath();
    ctx.ellipse(x + w / 2 + 18, y + d / 2 + 22, 70, 38, 0, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = "#ff9f00";
    ctx.fillRect(x + w / 2 - 10, y - z + 55, 20, z + 70);

    ctx.fillStyle = "#ffd400";
    ctx.beginPath();
    ctx.arc(x + w / 2, y - z + 50, 62, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = "#ffffff";
    ctx.beginPath();
    ctx.arc(x + w / 2, y - z + 50, 24, 0, Math.PI * 2);
    ctx.fill();
  } else if (b.kind === "tent") {
    ctx.fillStyle = shade(b.color, -50);
    ctx.beginPath();
    ctx.moveTo(x, y + d);
    ctx.lineTo(x + w / 2, roofY);
    ctx.lineTo(x + w, y + d);
    ctx.closePath();
    ctx.fill();

    ctx.strokeStyle = "#ffffff";
    ctx.lineWidth = 4;
    ctx.stroke();

    ctx.fillStyle = "rgba(255,255,255,0.35)";
    ctx.beginPath();
    ctx.moveTo(x + w / 2, roofY);
    ctx.lineTo(x + w / 2, y + d);
    ctx.stroke();
  } else if (b.kind === "pyramid") {
    ctx.fillStyle = shade(b.color, -20);
    ctx.beginPath();
    ctx.moveTo(x + w / 2, roofY);
    ctx.lineTo(x + w, y + d);
    ctx.lineTo(x, y + d);
    ctx.closePath();
    ctx.fill();

    ctx.strokeStyle = "#ffffff";
    ctx.lineWidth = 3;
    ctx.stroke();
  } else if (b.kind === "expo") {
    ctx.fillStyle = "rgba(0,0,0,0.25)";
    ctx.beginPath();
    ctx.ellipse(x + w / 2 + 14, y + d / 2 + 20, 85, 55, 0, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = "#00b7ff";
    ctx.beginPath();
    ctx.arc(x + w / 2, y - z / 2 + d / 2, 78, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = "rgba(255,255,255,0.45)";
    ctx.beginPath();
    ctx.arc(x + w / 2 - 20, y - z / 2 + d / 2 - 25, 24, 0, Math.PI * 2);
    ctx.fill();
  } else if (b.kind === "mosque") {
    ctx.fillStyle = shade(b.color, -55);
    ctx.fillRect(x, roofY + d * 0.2, w, d * 0.8);

    ctx.fillStyle = b.color;
    ctx.beginPath();
    ctx.arc(x + w / 2, roofY + d * 0.38, 58, Math.PI, 0);
    ctx.fill();

    ctx.fillStyle = "#00e0c6";
    ctx.beginPath();
    ctx.arc(x + w / 2, roofY + d * 0.38, 38, Math.PI, 0);
    ctx.fill();

    ctx.fillStyle = "#ffffff";
    ctx.fillRect(x + 18, roofY - 25, 18, d + 50);
    ctx.fillRect(x + w - 36, roofY - 25, 18, d + 50);
  } else {
    // Передняя стена
    ctx.fillStyle = shade(b.color, -60);
    ctx.beginPath();
    ctx.moveTo(x, roofY + d);
    ctx.lineTo(x + w, roofY + d);
    ctx.lineTo(x + w, y + d);
    ctx.lineTo(x, y + d);
    ctx.closePath();
    ctx.fill();

    // Правая стена
    ctx.fillStyle = shade(b.color, -35);
    ctx.beginPath();
    ctx.moveTo(x + w, roofY);
    ctx.lineTo(x + w, roofY + d);
    ctx.lineTo(x + w, y + d);
    ctx.lineTo(x + w, y);
    ctx.closePath();
    ctx.fill();

    // Крыша
    ctx.fillStyle = b.color;
    ctx.fillRect(x, roofY, w, d);

    ctx.fillStyle = "rgba(255,255,255,0.28)";
    for (let wx = 10; wx < w - 10; wx += 28) {
      for (let wy = 10; wy < d - 10; wy += 28) {
        ctx.fillRect(x + wx, roofY + wy, 8, 8);
      }
    }
  }

  if (b.label && Math.abs(x) < canvas.width + 250 && Math.abs(y) < canvas.height + 250) {
    ctx.fillStyle = "rgba(0,0,0,0.62)";
    ctx.fillRect(x, roofY - 26, Math.min(170, b.label.length * 8 + 14), 20);
    ctx.fillStyle = "#fff";
    ctx.font = "bold 12px Arial";
    ctx.textAlign = "left";
    ctx.fillText(b.label.slice(0, 20), x + 6, roofY - 11);
  }
}

function drawBuildings() {
  const sorted = [...buildings].sort((a, b) => (a.y + a.d) - (b.y + b.d));
  for (const b of sorted) drawBuilding(b);
}

function drawCars() {
  for (const c of cars) {
    const x = sx(c.x - c.w / 2);
    const y = sy(c.y - c.d / 2);

    ctx.fillStyle = "rgba(0,0,0,0.35)";
    ctx.fillRect(x + 5, y + 6, c.w, c.d);

    ctx.fillStyle = c.color;
    ctx.fillRect(x, y, c.w, c.d);

    ctx.fillStyle = "#111";
    if (c.w > c.d) {
      ctx.fillRect(x + 10, y + 5, 11, c.d - 10);
      ctx.fillRect(x + c.w - 22, y + 5, 11, c.d - 10);
    } else {
      ctx.fillRect(x + 5, y + 10, c.w - 10, 11);
      ctx.fillRect(x + 5, y + c.d - 22, c.w - 10, 11);
    }
  }
}

function drawQuestMarker() {
  if (!quest) return;

  const target = player.hasPackage ? quest.drop : quest.pickup;
  const color = player.hasPackage ? "#00ff75" : "#ffe600";
  const pulse = 8 + Math.sin(performance.now() / 120) * 5;

  ctx.lineWidth = 7;
  ctx.strokeStyle = "#000";
  ctx.beginPath();
  ctx.arc(sx(target.x), sy(target.y), 39 + pulse, 0, Math.PI * 2);
  ctx.stroke();

  ctx.lineWidth = 5;
  ctx.strokeStyle = color;
  ctx.beginPath();
  ctx.arc(sx(target.x), sy(target.y), 39 + pulse, 0, Math.PI * 2);
  ctx.stroke();

  ctx.fillStyle = color;
  ctx.beginPath();
  ctx.arc(sx(target.x), sy(target.y), 13, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = "#000";
  ctx.font = "bold 14px Arial";
  ctx.textAlign = "center";
  ctx.fillText(player.hasPackage ? "СДАТЬ" : "ВЗЯТЬ", sx(target.x), sy(target.y) - 47);

  const angle = Math.atan2(target.y - player.y, target.x - player.x);
  const ax = canvas.width / 2 + Math.cos(angle) * 84;
  const ay = canvas.height / 2 + Math.sin(angle) * 84;

  ctx.save();
  ctx.translate(ax, ay);
  ctx.rotate(angle);
  ctx.fillStyle = color;
  ctx.beginPath();
  ctx.moveTo(23, 0);
  ctx.lineTo(-13, -11);
  ctx.lineTo(-7, 0);
  ctx.lineTo(-13, 11);
  ctx.closePath();
  ctx.fill();
  ctx.restore();
}

function drawPlayer() {
  const x = sx(player.x);
  const y = sy(player.y);

  ctx.fillStyle = "rgba(0,0,0,0.38)";
  ctx.beginPath();
  ctx.ellipse(x + 5, y + 10, 19, 10, 0, 0, Math.PI * 2);
  ctx.fill();

  if (player.hitTimer > 0 && Math.floor(performance.now() / 80) % 2 === 0) {
    ctx.globalAlpha = 0.45;
  }

  ctx.fillStyle = "#161616";
  ctx.beginPath();
  ctx.arc(x, y, player.r + 4, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = "#00ffcc";
  ctx.beginPath();
  ctx.arc(x, y, player.r, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = "#ffeb3b";
  ctx.fillRect(x - 7, y - 23, 14, 9);

  if (player.hasPackage) {
    ctx.fillStyle = "#ff8c00";
    ctx.fillRect(x - 21, y - 5, 12, 15);
    ctx.strokeStyle = "#fff";
    ctx.strokeRect(x - 21, y - 5, 12, 15);
  }

  ctx.globalAlpha = 1;
}

function drawParticles() {
  for (const p of particles) {
    ctx.globalAlpha = clamp(p.life, 0, 1);
    ctx.fillStyle = p.color;
    ctx.fillRect(sx(p.x), sy(p.y), 5, 5);
  }
  ctx.globalAlpha = 1;
}

function drawUI() {
  const panelW = Math.min(390, canvas.width - 24);

  ctx.fillStyle = "rgba(0,0,0,0.66)";
  ctx.fillRect(12, 74, panelW, 138);

  ctx.strokeStyle = "rgba(255,255,255,0.35)";
  ctx.lineWidth = 2;
  ctx.strokeRect(12, 74, panelW, 138);

  ctx.textAlign = "left";
  ctx.fillStyle = "#fff";
  ctx.font = "bold 18px Arial";
  ctx.fillText("ASTANA COURIER 2.5D", 24, 101);

  ctx.font = "14px Arial";
  ctx.fillText("Уровень: " + player.level, 24, 126);
  ctx.fillText("Доставки: " + player.deliveries, 125, 126);
  ctx.fillText("Деньги: " + player.money + "₸", 245, 126);

  ctx.fillStyle = "#222";
  ctx.fillRect(24, 140, 175, 15);
  ctx.fillStyle = player.hp > 35 ? "#00ff75" : "#ff304f";
  ctx.fillRect(24, 140, 175 * player.hp / 100, 15);
  ctx.strokeStyle = "#fff";
  ctx.strokeRect(24, 140, 175, 15);

  ctx.fillStyle = "#fff";
  ctx.fillText("HP " + Math.floor(player.hp), 210, 153);

  if (quest) {
    const target = player.hasPackage ? quest.drop : quest.pickup;

    ctx.font = "13px Arial";
    ctx.fillStyle = "#fff";
    ctx.fillText((player.hasPackage ? "Куда: " : "Забрать: ") + target.name, 24, 178);

    ctx.fillStyle = "#d7f7ff";
    ctx.fillText(target.address.slice(0, 45), 24, 197);

    if (player.hasPackage) {
      const t = Math.ceil(quest.timeLeft);
      ctx.fillStyle = t < 10 ? "#ff304f" : "#ffe600";
      ctx.font = "bold 15px Arial";
      ctx.fillText("Время: " + t, panelW - 96, 153);
    }
  }

  if (messageTimer > 0) {
    const w = Math.min(canvas.width - 24, 680);
    ctx.fillStyle = "rgba(0,0,0,0.68)";
    ctx.fillRect(canvas.width / 2 - w / 2, 226, w, 44);
    ctx.strokeStyle = "rgba(255,255,255,0.35)";
    ctx.strokeRect(canvas.width / 2 - w / 2, 226, w, 44);

    ctx.fillStyle = "#fff";
    ctx.font = "bold 15px Arial";
    ctx.textAlign = "center";
    ctx.fillText(message.slice(0, 70), canvas.width / 2, 253);
  }

  drawMiniMap();
}

function drawMiniMap() {
  if (canvas.width < 470) return;

  const mw = 165;
  const mh = 116;
  const mx = canvas.width - mw - 12;
  const my = 74;

  ctx.fillStyle = "rgba(0,0,0,0.64)";
  ctx.fillRect(mx, my, mw, mh);
  ctx.strokeStyle = "rgba(255,255,255,0.42)";
  ctx.strokeRect(mx, my, mw, mh);

  const sxm = mw / WORLD_W;
  const sym = mh / WORLD_H;

  ctx.fillStyle = "#555";
  for (const r of verticalRoads) {
    ctx.fillRect(mx + (r.x - ROAD_W / 2) * sxm, my, ROAD_W * sxm, mh);
  }
  for (const r of horizontalRoads) {
    ctx.fillRect(mx, my + (r.y - ROAD_W / 2) * sym, mw, ROAD_W * sym);
  }

  if (quest) {
    const target = player.hasPackage ? quest.drop : quest.pickup;
    ctx.fillStyle = player.hasPackage ? "#00ff75" : "#ffe600";
    ctx.beginPath();
    ctx.arc(mx + target.x * sxm, my + target.y * sym, 5, 0, Math.PI * 2);
    ctx.fill();
  }

  ctx.fillStyle = "#00eaff";
  ctx.beginPath();
  ctx.arc(mx + player.x * sxm, my + player.y * sym, 5, 0, Math.PI * 2);
  ctx.fill();
}

function drawBorder() {
  ctx.strokeStyle = "#ffffff";
  ctx.lineWidth = 8;
  ctx.strokeRect(sx(0), sy(0), WORLD_W, WORLD_H);
}

function draw() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  drawBackground();
  drawRoads();
  drawBuildings();
  drawCars();
  drawQuestMarker();
  drawParticles();
  drawPlayer();
  drawBorder();
  drawUI();
}

function init() {
  generateCity();
  createCars();
  createQuest();
}

let last = performance.now();

function loop(now) {
  const dt = Math.min(0.033, (now - last) / 1000);
  last = now;

  update(dt);
  draw();

  requestAnimationFrame(loop);
}

init();
requestAnimationFrame(loop);
</script>
</body>
</html>
