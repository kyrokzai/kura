<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
  <title>Astana Courier Mobile</title>
  <style>
    * {
      box-sizing: border-box;
      -webkit-tap-highlight-color: transparent;
      user-select: none;
      -webkit-user-select: none;
    }

    html, body {
      margin: 0;
      padding: 0;
      overflow: hidden;
      width: 100%;
      height: 100%;
      background: #071018;
      font-family: Arial, sans-serif;
      touch-action: none;
    }

    canvas {
      display: block;
      width: 100vw;
      height: 100vh;
      image-rendering: pixelated;
      background: #4ee069;
      touch-action: none;
    }

    #mobileControls {
      position: fixed;
      inset: 0;
      pointer-events: none;
      z-index: 10;
    }

    #joystickBase {
      position: absolute;
      left: 28px;
      bottom: 34px;
      width: 130px;
      height: 130px;
      border-radius: 50%;
      background: rgba(0, 0, 0, 0.35);
      border: 3px solid rgba(255, 255, 255, 0.35);
      pointer-events: auto;
      touch-action: none;
    }

    #joystickKnob {
      position: absolute;
      left: 38px;
      top: 38px;
      width: 54px;
      height: 54px;
      border-radius: 50%;
      background: #00eaff;
      border: 3px solid white;
      box-shadow: 0 0 18px #00eaff;
    }

    .btn {
      position: absolute;
      right: 28px;
      width: 82px;
      height: 82px;
      border-radius: 50%;
      border: 3px solid rgba(255,255,255,0.75);
      color: white;
      font-weight: 900;
      font-size: 18px;
      background: rgba(0,0,0,0.48);
      pointer-events: auto;
      display: flex;
      align-items: center;
      justify-content: center;
      touch-action: none;
      box-shadow: 0 0 18px rgba(255,255,255,0.18);
    }

    #interactBtn {
      bottom: 38px;
      background: rgba(255, 196, 0, 0.7);
      color: #111;
    }

    #sprintBtn {
      bottom: 132px;
      background: rgba(0, 255, 140, 0.55);
    }

    #smallHint {
      position: fixed;
      left: 12px;
      top: 12px;
      padding: 8px 10px;
      background: rgba(0,0,0,0.55);
      color: white;
      border: 1px solid rgba(255,255,255,0.25);
      border-radius: 10px;
      font-size: 12px;
      line-height: 1.35;
      z-index: 20;
      max-width: 310px;
    }

    @media (min-width: 900px) {
      #joystickBase, .btn {
        opacity: 0.55;
      }
    }
  </style>
</head>
<body>

<canvas id="game"></canvas>

<div id="smallHint">
  <b>Astana Courier</b><br>
  Телефон: джойстик + кнопки справа<br>
  ПК: WASD/стрелки, Shift, E
</div>

<div id="mobileControls">
  <div id="joystickBase">
    <div id="joystickKnob"></div>
  </div>
  <div id="sprintBtn" class="btn">БЕГ</div>
  <div id="interactBtn" class="btn">E</div>
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

const WORLD_W = 2400;
const WORLD_H = 1700;
const ROAD_W = 96;

const verticalRoads = [230, 570, 910, 1250, 1590, 1930, 2220];
const horizontalRoads = [210, 500, 790, 1080, 1370, 1570];

const keys = {};
let interactPressed = false;
let mobileSprint = false;

window.addEventListener("keydown", e => {
  keys[e.key.toLowerCase()] = true;
  if (e.key.toLowerCase() === "e") interactPressed = true;
});
window.addEventListener("keyup", e => {
  keys[e.key.toLowerCase()] = false;
});

const joy = {
  active: false,
  pointerId: null,
  x: 0,
  y: 0,
  max: 46
};

const joystickBase = document.getElementById("joystickBase");
const joystickKnob = document.getElementById("joystickKnob");
const sprintBtn = document.getElementById("sprintBtn");
const interactBtn = document.getElementById("interactBtn");

function setKnob(x, y) {
  joystickKnob.style.left = 38 + x + "px";
  joystickKnob.style.top = 38 + y + "px";
}

joystickBase.addEventListener("pointerdown", e => {
  joy.active = true;
  joy.pointerId = e.pointerId;
  joystickBase.setPointerCapture(e.pointerId);
  updateJoystick(e);
});

joystickBase.addEventListener("pointermove", e => {
  if (!joy.active || e.pointerId !== joy.pointerId) return;
  updateJoystick(e);
});

joystickBase.addEventListener("pointerup", resetJoystick);
joystickBase.addEventListener("pointercancel", resetJoystick);

function updateJoystick(e) {
  const rect = joystickBase.getBoundingClientRect();
  const cx = rect.left + rect.width / 2;
  const cy = rect.top + rect.height / 2;

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

function resetJoystick() {
  joy.active = false;
  joy.pointerId = null;
  joy.x = 0;
  joy.y = 0;
  setKnob(0, 0);
}

sprintBtn.addEventListener("pointerdown", e => {
  e.preventDefault();
  mobileSprint = true;
});
sprintBtn.addEventListener("pointerup", e => {
  e.preventDefault();
  mobileSprint = false;
});
sprintBtn.addEventListener("pointercancel", () => mobileSprint = false);
sprintBtn.addEventListener("pointerleave", () => mobileSprint = false);

interactBtn.addEventListener("pointerdown", e => {
  e.preventDefault();
  interactPressed = true;
});

function clamp(v, a, b) {
  return Math.max(a, Math.min(b, v));
}

function rand(min, max) {
  return min + Math.random() * (max - min);
}

function pick(arr) {
  return arr[Math.floor(Math.random() * arr.length)];
}

function distance(a, b) {
  return Math.hypot(a.x - b.x, a.y - b.y);
}

function circleRectCollision(cx, cy, r, rect) {
  const nx = clamp(cx, rect.x, rect.x + rect.w);
  const ny = clamp(cy, rect.y, rect.y + rect.h);
  return Math.hypot(cx - nx, cy - ny) < r;
}

const player = {
  x: 230,
  y: 210,
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

const brightColors = [
  "#ff3bd5", "#00dcff", "#ffe600", "#ff7b00", "#7cff00",
  "#ad60ff", "#ff304f", "#00ff95", "#2f8cff", "#ffffff"
];

function generateBuildings() {
  buildings = [];

  const xBlocks = [
    [20, 170], [290, 520], [630, 860], [970, 1200],
    [1310, 1540], [1650, 1880], [1990, 2160], [2280, 2380]
  ];

  const yBlocks = [
    [20, 160], [270, 450], [560, 740], [850, 1030],
    [1140, 1320], [1430, 1510], [1620, 1680]
  ];

  for (const xb of xBlocks) {
    for (const yb of yBlocks) {
      if (Math.random() < 0.18) continue;

      const w = rand(70, Math.max(80, xb[1] - xb[0] - 30));
      const h = rand(55, Math.max(65, yb[1] - yb[0] - 30));
      const x = xb[0] + rand(8, Math.max(9, xb[1] - xb[0] - w));
      const y = yb[0] + rand(8, Math.max(9, yb[1] - yb[0] - h));

      buildings.push({
        x, y, w, h,
        color: pick(brightColors),
        roof: pick(["line", "dots", "plain"])
      });
    }
  }

  buildings.push({ x: 1110, y: 310, w: 150, h: 150, color: "#ffd400", landmark: "baiterek" });
  buildings.push({ x: 1760, y: 300, w: 240, h: 145, color: "#00eaff", landmark: "tent" });
  buildings.push({ x: 1390, y: 900, w: 190, h: 190, color: "#c77dff", landmark: "pyramid" });
}

const points = [];
for (const x of verticalRoads) {
  for (const y of horizontalRoads) {
    points.push({ x, y, name: "Район " + x + "-" + y });
  }
}

points[0].name = "Старый вокзал";
points[4].name = "Южный склад";
points[10].name = "Кофейня Абая";
points[16].name = "Офис курьеров";
points[22].name = "Байтерек";
points[28].name = "IT Hub";
points[34].name = "Хан Шатыр";
points[39].name = "Набережная";

function createQuest() {
  let pickup = pick(points);
  let drop = pick(points);

  while (drop === pickup || distance(pickup, drop) < 480) {
    drop = pick(points);
  }

  const d = distance(pickup, drop);
  const maxTime = clamp(28 + d / 85 - player.level * 1.5, 24, 90);

  quest = {
    pickup,
    drop,
    reward: Math.floor(45 + d / 14 + player.level * 18),
    timeLeft: maxTime,
    maxTime
  };

  player.hasPackage = false;
  showMessage("Новый заказ: " + pickup.name + " → " + drop.name, 4);
}

function createCars() {
  cars = [];
  const count = 18 + player.level * 5;

  for (let i = 0; i < count; i++) {
    const horizontal = Math.random() < 0.55;

    if (horizontal) {
      const y = pick(horizontalRoads) + pick([-24, 24]);
      const dir = Math.random() < 0.5 ? 1 : -1;

      cars.push({
        x: rand(0, WORLD_W),
        y,
        w: 46,
        h: 25,
        dx: dir,
        dy: 0,
        speed: rand(90, 170) + player.level * 8,
        color: pick(["#ff304f", "#00eaff", "#ffe600", "#7cff00", "#ff7b00", "#ffffff"])
      });
    } else {
      const x = pick(verticalRoads) + pick([-24, 24]);
      const dir = Math.random() < 0.5 ? 1 : -1;

      cars.push({
        x,
        y: rand(0, WORLD_H),
        w: 25,
        h: 46,
        dx: 0,
        dy: dir,
        speed: rand(90, 170) + player.level * 8,
        color: pick(["#ff304f", "#00eaff", "#ffe600", "#7cff00", "#ff7b00", "#ffffff"])
      });
    }
  }
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

  const isSprint = keys["shift"] || mobileSprint;
  const speed = isSprint ? player.sprint : player.speed;

  const nx = player.x + dx * speed * dt;
  const ny = player.y + dy * speed * dt;

  if (canMoveTo(nx, player.y)) player.x = nx;
  if (canMoveTo(player.x, ny)) player.y = ny;

  if (len > 0.1 && Math.random() < 0.35) {
    particles.push({
      x: player.x - dx * 12,
      y: player.y - dy * 12,
      vx: rand(-20, 20),
      vy: rand(-20, 20),
      life: 0.25,
      color: isSprint ? "#ffe600" : "#ffffff"
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
      y: c.y - c.h / 2,
      w: c.w,
      h: c.h
    };

    if (player.hitTimer <= 0 && circleRectCollision(player.x, player.y, player.r, rect)) {
      player.hp -= 16;
      player.hitTimer = 1;
      showMessage("Осторожно! Машина сбила курьера", 2);

      for (let i = 0; i < 14; i++) {
        particles.push({
          x: player.x,
          y: player.y,
          vx: rand(-180, 180),
          vy: rand(-180, 180),
          life: 0.65,
          color: "#ff3030"
        });
      }

      if (player.hp <= 0) {
        player.hp = 100;
        player.money = Math.max(0, player.money - 50);
        player.x = 230;
        player.y = 210;
        player.hasPackage = false;
        createQuest();
        showMessage("Ты потерял заказ. Штраф 50₸. Респавн.", 4);
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
      showMessage("Время вышло! Заказ провален", 3);
      return;
    }
  }

  const target = player.hasPackage ? quest.drop : quest.pickup;
  const near = Math.hypot(player.x - target.x, player.y - target.y) < 58;

  if (near && interactPressed) {
    if (!player.hasPackage) {
      player.hasPackage = true;
      showMessage("Посылка взята. Доставь в: " + quest.drop.name, 3);
    } else {
      const bonus = Math.floor(Math.max(0, quest.timeLeft) * 2);
      const total = quest.reward + bonus;

      player.money += total;
      player.deliveries++;
      player.hasPackage = false;

      for (let i = 0; i < 32; i++) {
        particles.push({
          x: player.x,
          y: player.y,
          vx: rand(-230, 230),
          vy: rand(-230, 230),
          life: 0.9,
          color: pick(["#ffe600", "#00ff95", "#00eaff", "#ff3bd5"])
        });
      }

      showMessage("Доставлено! +" + total + "₸", 3);

      if (player.deliveries % 3 === 0) {
        player.level++;
        player.hp = Math.min(100, player.hp + 25);
        createCars();
        showMessage("Уровень " + player.level + "! Трафик стал сложнее", 4);
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

  camera.x = clamp(player.x - canvas.width / 2, 0, WORLD_W - canvas.width);
  camera.y = clamp(player.y - canvas.height / 2, 0, WORLD_H - canvas.height);
}

function sx(x) { return Math.floor(x - camera.x); }
function sy(y) { return Math.floor(y - camera.y); }

function drawBackground() {
  ctx.fillStyle = "#45db66";
  ctx.fillRect(sx(0), sy(0), WORLD_W, WORLD_H);

  ctx.fillStyle = "#1fb653";
  ctx.fillRect(sx(40), sy(40), 150, 120);
  ctx.fillRect(sx(720), sy(1140), 180, 170);
  ctx.fillRect(sx(2080), sy(1430), 260, 130);

  ctx.fillStyle = "#18b7ff";
  ctx.beginPath();
  ctx.moveTo(sx(0), sy(670));
  ctx.bezierCurveTo(sx(430), sy(610), sx(780), sy(735), sx(1200), sy(690));
  ctx.bezierCurveTo(sx(1680), sy(635), sx(1960), sy(760), sx(WORLD_W), sy(700));
  ctx.lineTo(sx(WORLD_W), sy(770));
  ctx.bezierCurveTo(sx(1960), sy(830), sx(1680), sy(710), sx(1200), sy(775));
  ctx.bezierCurveTo(sx(780), sy(820), sx(430), sy(700), sx(0), sy(780));
  ctx.closePath();
  ctx.fill();
}

function drawRoads() {
  ctx.fillStyle = "#38434f";

  for (const x of verticalRoads) {
    ctx.fillRect(sx(x - ROAD_W / 2), sy(0), ROAD_W, WORLD_H);
  }

  for (const y of horizontalRoads) {
    ctx.fillRect(sx(0), sy(y - ROAD_W / 2), WORLD_W, ROAD_W);
  }

  ctx.strokeStyle = "#f6ff61";
  ctx.lineWidth = 4;
  ctx.setLineDash([24, 24]);

  for (const x of verticalRoads) {
    ctx.beginPath();
    ctx.moveTo(sx(x), sy(0));
    ctx.lineTo(sx(x), sy(WORLD_H));
    ctx.stroke();
  }

  for (const y of horizontalRoads) {
    ctx.beginPath();
    ctx.moveTo(sx(0), sy(y));
    ctx.lineTo(sx(WORLD_W), sy(y));
    ctx.stroke();
  }

  ctx.setLineDash([]);
}

function drawBuildings() {
  for (const b of buildings) {
    const x = sx(b.x);
    const y = sy(b.y);

    ctx.fillStyle = "rgba(0,0,0,0.34)";
    ctx.fillRect(x + 7, y + 8, b.w, b.h);

    if (b.landmark === "baiterek") {
      ctx.fillStyle = "#ffd400";
      ctx.beginPath();
      ctx.arc(x + b.w / 2, y + b.h / 2, 62, 0, Math.PI * 2);
      ctx.fill();

      ctx.fillStyle = "#fff";
      ctx.beginPath();
      ctx.arc(x + b.w / 2, y + b.h / 2, 24, 0, Math.PI * 2);
      ctx.fill();

      ctx.fillStyle = "#ff8500";
      ctx.fillRect(x + b.w / 2 - 9, y + 35, 18, 105);
    } else if (b.landmark === "tent") {
      ctx.fillStyle = b.color;
      ctx.beginPath();
      ctx.moveTo(x + b.w / 2, y);
      ctx.lineTo(x + b.w, y + b.h);
      ctx.lineTo(x, y + b.h);
      ctx.closePath();
      ctx.fill();

      ctx.strokeStyle = "#ffffff";
      ctx.lineWidth = 4;
      ctx.stroke();
    } else if (b.landmark === "pyramid") {
      ctx.fillStyle = b.color;
      ctx.beginPath();
      ctx.moveTo(x + b.w / 2, y);
      ctx.lineTo(x + b.w, y + b.h);
      ctx.lineTo(x, y + b.h);
      ctx.closePath();
      ctx.fill();

      ctx.strokeStyle = "#ffffff";
      ctx.lineWidth = 3;
      ctx.stroke();
    } else {
      ctx.fillStyle = b.color;
      ctx.fillRect(x, y, b.w, b.h);

      ctx.fillStyle = "rgba(255,255,255,0.32)";
      if (b.roof === "dots") {
        for (let px = 12; px < b.w; px += 26) {
          for (let py = 12; py < b.h; py += 26) {
            ctx.fillRect(x + px, y + py, 8, 8);
          }
        }
      } else if (b.roof === "line") {
        for (let px = 0; px < b.w; px += 18) {
          ctx.fillRect(x + px, y, 7, b.h);
        }
      } else {
        ctx.fillRect(x + 8, y + 8, b.w - 16, 8);
      }
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
  ctx.arc(sx(target.x), sy(target.y), 38 + pulse, 0, Math.PI * 2);
  ctx.stroke();

  ctx.lineWidth = 5;
  ctx.strokeStyle = color;
  ctx.beginPath();
  ctx.arc(sx(target.x), sy(target.y), 38 + pulse, 0, Math.PI * 2);
  ctx.stroke();

  ctx.fillStyle = color;
  ctx.beginPath();
  ctx.arc(sx(target.x), sy(target.y), 13, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = "#000";
  ctx.font = "bold 14px Arial";
  ctx.textAlign = "center";
  ctx.fillText(player.hasPackage ? "СДАТЬ" : "ВЗЯТЬ", sx(target.x), sy(target.y) - 46);

  const angle = Math.atan2(target.y - player.y, target.x - player.x);
  const ax = canvas.width / 2 + Math.cos(angle) * 82;
  const ay = canvas.height / 2 + Math.sin(angle) * 82;

  ctx.save();
  ctx.translate(ax, ay);
  ctx.rotate(angle);
  ctx.fillStyle = color;
  ctx.beginPath();
  ctx.moveTo(22, 0);
  ctx.lineTo(-13, -11);
  ctx.lineTo(-7, 0);
  ctx.lineTo(-13, 11);
  ctx.closePath();
  ctx.fill();
  ctx.restore();
}

function drawCars() {
  for (const c of cars) {
    const x = sx(c.x - c.w / 2);
    const y = sy(c.y - c.h / 2);

    ctx.fillStyle = "rgba(0,0,0,0.35)";
    ctx.fillRect(x + 4, y + 5, c.w, c.h);

    ctx.fillStyle = c.color;
    ctx.fillRect(x, y, c.w, c.h);

    ctx.fillStyle = "#111";
    if (c.w > c.h) {
      ctx.fillRect(x + 10, y + 4, 10, c.h - 8);
      ctx.fillRect(x + c.w - 20, y + 4, 10, c.h - 8);
    } else {
      ctx.fillRect(x + 4, y + 10, c.w - 8, 10);
      ctx.fillRect(x + 4, y + c.h - 20, c.w - 8, 10);
    }
  }
}

function drawPlayer() {
  const x = sx(player.x);
  const y = sy(player.y);

  ctx.fillStyle = "rgba(0,0,0,0.38)";
  ctx.beginPath();
  ctx.ellipse(x + 5, y + 9, 18, 10, 0, 0, Math.PI * 2);
  ctx.fill();

  if (player.hitTimer > 0 && Math.floor(performance.now() / 80) % 2 === 0) {
    ctx.globalAlpha = 0.45;
  }

  ctx.fillStyle = "#181818";
  ctx.beginPath();
  ctx.arc(x, y, player.r + 4, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = "#00ffcc";
  ctx.beginPath();
  ctx.arc(x, y, player.r, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = "#ffeb3b";
  ctx.fillRect(x - 7, y - 22, 14, 9);

  if (player.hasPackage) {
    ctx.fillStyle = "#ff8c00";
    ctx.fillRect(x - 20, y - 5, 11, 14);
    ctx.strokeStyle = "#fff";
    ctx.strokeRect(x - 20, y - 5, 11, 14);
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
  const panelW = Math.min(350, canvas.width - 24);

  ctx.fillStyle = "rgba(0,0,0,0.64)";
  ctx.fillRect(12, 74, panelW, 112);

  ctx.strokeStyle = "rgba(255,255,255,0.35)";
  ctx.lineWidth = 2;
  ctx.strokeRect(12, 74, panelW, 112);

  ctx.textAlign = "left";
  ctx.fillStyle = "#fff";
  ctx.font = "bold 18px Arial";
  ctx.fillText("ASTANA COURIER", 24, 102);

  ctx.font = "14px Arial";
  ctx.fillText("Уровень: " + player.level, 24, 126);
  ctx.fillText("Доставки: " + player.deliveries, 125, 126);
  ctx.fillText("Деньги: " + player.money + "₸", 235, 126);

  ctx.fillStyle = "#222";
  ctx.fillRect(24, 140, 175, 15);
  ctx.fillStyle = player.hp > 35 ? "#00ff75" : "#ff304f";
  ctx.fillRect(24, 140, 175 * player.hp / 100, 15);
  ctx.strokeStyle = "#fff";
  ctx.strokeRect(24, 140, 175, 15);

  ctx.fillStyle = "#fff";
  ctx.fillText("HP " + Math.floor(player.hp), 210, 153);

  if (quest) {
    ctx.font = "13px Arial";
    const text = player.hasPackage ? "Куда: " + quest.drop.name : "Где взять: " + quest.pickup.name;
    ctx.fillText(text.slice(0, 38), 24, 176);

    if (player.hasPackage) {
      const t = Math.ceil(quest.timeLeft);
      ctx.fillStyle = t < 10 ? "#ff304f" : "#ffe600";
      ctx.font = "bold 15px Arial";
      ctx.fillText("Время: " + t, panelW - 95, 176);
    }
  }

  if (messageTimer > 0) {
    const w = Math.min(canvas.width - 24, 620);
    ctx.fillStyle = "rgba(0,0,0,0.68)";
    ctx.fillRect(canvas.width / 2 - w / 2, 198, w, 42);
    ctx.strokeStyle = "rgba(255,255,255,0.35)";
    ctx.strokeRect(canvas.width / 2 - w / 2, 198, w, 42);

    ctx.fillStyle = "#fff";
    ctx.font = "bold 15px Arial";
    ctx.textAlign = "center";
    ctx.fillText(message.slice(0, 62), canvas.width / 2, 224);
  }

  drawMiniMap();
}

function drawMiniMap() {
  const mw = 150;
  const mh = 105;
  const mx = canvas.width - mw - 12;
  const my = 74;

  if (canvas.width < 430) return;

  ctx.fillStyle = "rgba(0,0,0,0.62)";
  ctx.fillRect(mx, my, mw, mh);

  ctx.strokeStyle = "rgba(255,255,255,0.4)";
  ctx.strokeRect(mx, my, mw, mh);

  const sxm = mw / WORLD_W;
  const sym = mh / WORLD_H;

  ctx.fillStyle = "#555";
  for (const x of verticalRoads) {
    ctx.fillRect(mx + (x - ROAD_W / 2) * sxm, my, ROAD_W * sxm, mh);
  }
  for (const y of horizontalRoads) {
    ctx.fillRect(mx, my + (y - ROAD_W / 2) * sym, mw, ROAD_W * sym);
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
  drawQuestMarker();
  drawCars();
  drawParticles();
  drawPlayer();
  drawBorder();
  drawUI();
}

function init() {
  generateBuildings();
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
