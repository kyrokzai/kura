<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8" />
  <title>Astana Courier 32-bit</title>
  <style>
    html, body {
      margin: 0;
      overflow: hidden;
      background: #111;
      font-family: Arial, sans-serif;
      color: white;
    }

    canvas {
      display: block;
      width: 100vw;
      height: 100vh;
      image-rendering: pixelated;
      background: #45d96d;
    }

    #hint {
      position: fixed;
      left: 14px;
      bottom: 12px;
      background: rgba(0,0,0,0.55);
      padding: 10px 14px;
      border: 2px solid rgba(255,255,255,0.4);
      border-radius: 10px;
      font-size: 14px;
      line-height: 1.45;
      pointer-events: none;
    }
  </style>
</head>
<body>
<canvas id="game"></canvas>

<div id="hint">
  <b>Astana Courier</b><br>
  WASD / стрелки — движение<br>
  Shift — бежать быстрее<br>
  E — взять / сдать заказ<br>
  Доставляй посылки, повышай уровень
</div>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

function resize() {
  canvas.width = Math.floor(window.innerWidth);
  canvas.height = Math.floor(window.innerHeight);
}
window.addEventListener("resize", resize);
resize();

const WORLD_W = 2600;
const WORLD_H = 1800;
const ROAD_W = 100;

const keys = new Set();
let interactPressed = false;

window.addEventListener("keydown", e => {
  keys.add(e.key.toLowerCase());
  if (e.key.toLowerCase() === "e") interactPressed = true;
});

window.addEventListener("keyup", e => {
  keys.delete(e.key.toLowerCase());
});

function clamp(v, a, b) {
  return Math.max(a, Math.min(b, v));
}

function dist(a, b) {
  return Math.hypot(a.x - b.x, a.y - b.y);
}

function circleRectCollision(cx, cy, r, rect) {
  const nx = clamp(cx, rect.x, rect.x + rect.w);
  const ny = clamp(cy, rect.y, rect.y + rect.h);
  return Math.hypot(cx - nx, cy - ny) < r;
}

function rectsOverlap(a, b) {
  return a.x < b.x + b.w &&
         a.x + a.w > b.x &&
         a.y < b.y + b.h &&
         a.y + a.h > b.y;
}

let seed = 123456;
function rand() {
  seed = (seed * 16807) % 2147483647;
  return (seed - 1) / 2147483646;
}

function choice(arr) {
  return arr[Math.floor(rand() * arr.length)];
}

const verticalRoads = [250, 620, 980, 1340, 1700, 2100, 2400];
const horizontalRoads = [220, 520, 820, 1120, 1420, 1640];

const player = {
  x: 250,
  y: 220,
  r: 15,
  speed: 190,
  sprint: 280,
  health: 100,
  money: 0,
  score: 0,
  deliveries: 0,
  level: 1,
  hasPackage: false,
  hitCooldown: 0
};

let camera = { x: 0, y: 0 };

let buildings = [];
let cars = [];
let particles = [];
let message = "Найди жёлтую точку и возьми заказ";
let messageTimer = 4;

const colors = [
  "#ff4fd8", "#3bdcff", "#ffe347", "#ff7a3d", "#8cff45",
  "#b366ff", "#ff4c4c", "#39ff88", "#38a0ff", "#ffb000"
];

function addMessage(text, time = 3) {
  message = text;
  messageTimer = time;
}

function makeBuildings() {
  buildings = [];

  const xCells = [
    [20, 200],
    [310, 570],
    [680, 930],
    [1040, 1290],
    [1400, 1650],
    [1760, 2050],
    [2160, 2350],
    [2460, 2580]
  ];

  const yCells = [
    [20, 170],
    [280, 470],
    [580, 770],
    [880, 1070],
    [1180, 1370],
    [1480, 1580],
    [1700, 1780]
  ];

  let id = 0;

  for (const xc of xCells) {
    for (const yc of yCells) {
      if (rand() < 0.18) continue;

      const w = clamp((xc[1] - xc[0]) - 30 - rand() * 45, 80, 280);
      const h = clamp((yc[1] - yc[0]) - 30 - rand() * 45, 60, 200);
      const x = xc[0] + 15 + rand() * Math.max(1, (xc[1] - xc[0] - w - 15));
      const y = yc[0] + 15 + rand() * Math.max(1, (yc[1] - yc[0] - h - 15));

      buildings.push({
        id: id++,
        x, y, w, h,
        color: choice(colors),
        roof: choice(["flat", "stripe", "dots"]),
        name: "Дом " + id,
        solid: true
      });
    }
  }

  // Яркие достопримечательности в стиле Астаны
  buildings.push({
    id: id++,
    x: 1140,
    y: 335,
    w: 140,
    h: 140,
    color: "#ffd400",
    name: "Байтерек",
    landmark: "baiterek",
    solid: true
  });

  buildings.push({
    id: id++,
    x: 1850,
    y: 325,
    w: 230,
    h: 130,
    color: "#00e1ff",
    name: "Хан Шатыр",
    landmark: "tent",
    solid: true
  });

  buildings.push({
    id: id++,
    x: 1450,
    y: 920,
    w: 180,
    h: 180,
    color: "#c989ff",
    name: "Пирамида",
    landmark: "pyramid",
    solid: true
  });
}

const deliveryPoints = [
  { x: 250, y: 220, name: "Старый вокзал" },
  { x: 620, y: 220, name: "Кофейня на Абая" },
  { x: 980, y: 220, name: "Маркет у парка" },
  { x: 1340, y: 220, name: "Бизнес-центр" },
  { x: 1700, y: 220, name: "Северный склад" },
  { x: 2100, y: 220, name: "ТЦ Nomad" },
  { x: 2400, y: 220, name: "Западный район" },

  { x: 250, y: 520, name: "Пекарня" },
  { x: 620, y: 520, name: "Аптека" },
  { x: 980, y: 520, name: "Офис курьеров" },
  { x: 1340, y: 520, name: "Байтерек Плаза" },
  { x: 1700, y: 520, name: "IT Hub" },
  { x: 2100, y: 520, name: "Кинотеатр" },
  { x: 2400, y: 520, name: "Автостанция" },

  { x: 250, y: 820, name: "Южный рынок" },
  { x: 620, y: 820, name: "Ресторан Lagman" },
  { x: 980, y: 820, name: "Фудкорт" },
  { x: 1340, y: 820, name: "Министерский квартал" },
  { x: 1700, y: 820, name: "Казпочта" },
  { x: 2100, y: 820, name: "Сервисный центр" },
  { x: 2400, y: 820, name: "Кампус" },

  { x: 250, y: 1120, name: "Склад Wild Steppe" },
  { x: 620, y: 1120, name: "Цветочный салон" },
  { x: 980, y: 1120, name: "Бар Neon" },
  { x: 1340, y: 1120, name: "Пирамида офис" },
  { x: 1700, y: 1120, name: "Дворец доставки" },
  { x: 2100, y: 1120, name: "Техномагазин" },
  { x: 2400, y: 1120, name: "Велостанция" },

  { x: 250, y: 1420, name: "Южный склад" },
  { x: 620, y: 1420, name: "Шаурма 24" },
  { x: 980, y: 1420, name: "Отель Steppe" },
  { x: 1340, y: 1420, name: "Банк" },
  { x: 1700, y: 1420, name: "Гипермаркет" },
  { x: 2100, y: 1420, name: "Гараж" },
  { x: 2400, y: 1420, name: "Набережная" },

  { x: 250, y: 1640, name: "Левый берег" },
  { x: 620, y: 1640, name: "Музыкальная школа" },
  { x: 980, y: 1640, name: "Курьерский хаб" },
  { x: 1340, y: 1640, name: "Ночной маркет" },
  { x: 1700, y: 1640, name: "Склад электроники" },
  { x: 2100, y: 1640, name: "Парк фонтанов" },
  { x: 2400, y: 1640, name: "Восточная точка" }
];

let quest = null;

function createQuest() {
  let pickup = choice(deliveryPoints);
  let drop = choice(deliveryPoints);

  while (drop === pickup || Math.hypot(drop.x - pickup.x, drop.y - pickup.y) < 500) {
    drop = choice(deliveryPoints);
  }

  const d = Math.hypot(drop.x - pickup.x, drop.y - pickup.y);
  const baseTime = clamp(d / 95 + 28 - player.level * 2, 25, 95);
  const reward = Math.floor(40 + d / 16 + player.level * 15);

  quest = {
    pickup,
    drop,
    reward,
    timeLeft: baseTime,
    maxTime: baseTime
  };

  player.hasPackage = false;
  addMessage(`Новый заказ: ${pickup.name} → ${drop.name}`, 4);
}

function makeTraffic(count) {
  for (let i = 0; i < count; i++) {
    const horizontal = rand() < 0.55;

    if (horizontal) {
      const y = choice(horizontalRoads) + (rand() < 0.5 ? -22 : 22);
      const dir = rand() < 0.5 ? 1 : -1;
      cars.push({
        x: rand() * WORLD_W,
        y,
        w: 44,
        h: 24,
        dirX: dir,
        dirY: 0,
        speed: 90 + rand() * 80 + player.level * 10,
        color: choice(["#ff304f", "#00f0ff", "#ffe600", "#7cff00", "#ff7b00", "#ffffff"])
      });
    } else {
      const x = choice(verticalRoads) + (rand() < 0.5 ? -22 : 22);
      const dir = rand() < 0.5 ? 1 : -1;
      cars.push({
        x,
        y: rand() * WORLD_H,
        w: 24,
        h: 44,
        dirX: 0,
        dirY: dir,
        speed: 90 + rand() * 80 + player.level * 10,
        color: choice(["#ff304f", "#00f0ff", "#ffe600", "#7cff00", "#ff7b00", "#ffffff"])
      });
    }
  }
}

function init() {
  makeBuildings();
  makeTraffic(22);
  createQuest();
}

function canMoveTo(x, y) {
  if (x < player.r || y < player.r || x > WORLD_W - player.r || y > WORLD_H - player.r) {
    return false;
  }

  for (const b of buildings) {
    if (b.solid && circleRectCollision(x, y, player.r, b)) {
      return false;
    }
  }

  return true;
}

function movePlayer(dt) {
  let dx = 0;
  let dy = 0;

  if (keys.has("w") || keys.has("arrowup")) dy -= 1;
  if (keys.has("s") || keys.has("arrowdown")) dy += 1;
  if (keys.has("a") || keys.has("arrowleft")) dx -= 1;
  if (keys.has("d") || keys.has("arrowright")) dx += 1;

  const len = Math.hypot(dx, dy);
  if (len > 0) {
    dx /= len;
    dy /= len;
  }

  const speed = keys.has("shift") ? player.sprint : player.speed;
  const nx = player.x + dx * speed * dt;
  const ny = player.y + dy * speed * dt;

  if (canMoveTo(nx, player.y)) player.x = nx;
  if (canMoveTo(player.x, ny)) player.y = ny;

  if (len > 0 && rand() < 0.28) {
    particles.push({
      x: player.x - dx * 12,
      y: player.y - dy * 12,
      life: 0.3,
      color: keys.has("shift") ? "#ffff00" : "#ffffff"
    });
  }
}

function updateCars(dt) {
  for (const c of cars) {
    c.x += c.dirX * c.speed * dt;
    c.y += c.dirY * c.speed * dt;

    if (c.x < -80) c.x = WORLD_W + 80;
    if (c.x > WORLD_W + 80) c.x = -80;
    if (c.y < -80) c.y = WORLD_H + 80;
    if (c.y > WORLD_H + 80) c.y = -80;

    const rect = { x: c.x - c.w / 2, y: c.y - c.h / 2, w: c.w, h: c.h };
    if (player.hitCooldown <= 0 && circleRectCollision(player.x, player.y, player.r, rect)) {
      player.health -= 18;
      player.hitCooldown = 1.1;
      addMessage("Осторожно! Машина сбила курьера!", 2);

      for (let i = 0; i < 12; i++) {
        particles.push({
          x: player.x,
          y: player.y,
          vx: (rand() - 0.5) * 160,
          vy: (rand() - 0.5) * 160,
          life: 0.7,
          color: "#ff3030"
        });
      }

      if (player.health <= 0) {
        player.health = 100;
        player.x = 250;
        player.y = 220;
        player.money = Math.max(0, player.money - 50);
        player.hasPackage = false;
        createQuest();
        addMessage("Ты потерял сознание. Минус 50₸ штрафа. Респавн.", 4);
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
      player.score = Math.max(0, player.score - 30);
      player.money = Math.max(0, player.money - 20);
      createQuest();
      addMessage("Время вышло! Заказ провален.", 3);
      return;
    }
  }

  const target = player.hasPackage ? quest.drop : quest.pickup;
  const near = Math.hypot(player.x - target.x, player.y - target.y) < 55;

  if (near && interactPressed) {
    if (!player.hasPackage) {
      player.hasPackage = true;
      addMessage(`Посылка получена. Доставь в: ${quest.drop.name}`, 3);
    } else {
      const timeBonus = Math.floor(Math.max(0, quest.timeLeft) * 2);
      const total = quest.reward + timeBonus;

      player.money += total;
      player.score += total;
      player.deliveries += 1;
      player.hasPackage = false;

      for (let i = 0; i < 30; i++) {
        particles.push({
          x: player.x,
          y: player.y,
          vx: (rand() - 0.5) * 220,
          vy: (rand() - 0.5) * 220,
          life: 0.9,
          color: choice(["#ffe600", "#00ff99", "#00eaff", "#ff3cff"])
        });
      }

      addMessage(`Заказ доставлен! +${total}₸`, 3);

      if (player.deliveries % 3 === 0) {
        player.level += 1;
        player.health = Math.min(100, player.health + 25);
        makeTraffic(5);
        addMessage(`Уровень ${player.level}! Больше трафика, выше награды.`, 4);
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

  if (player.hitCooldown > 0) player.hitCooldown -= dt;
  if (messageTimer > 0) messageTimer -= dt;

  camera.x = clamp(player.x - canvas.width / 2, 0, WORLD_W - canvas.width);
  camera.y = clamp(player.y - canvas.height / 2, 0, WORLD_H - canvas.height);
}

function sx(x) { return Math.floor(x - camera.x); }
function sy(y) { return Math.floor(y - camera.y); }

function drawRoads() {
  ctx.fillStyle = "#38434f";

  for (const x of verticalRoads) {
    ctx.fillRect(sx(x - ROAD_W / 2), sy(0), ROAD_W, WORLD_H);
  }

  for (const y of horizontalRoads) {
    ctx.fillRect(sx(0), sy(y - ROAD_W / 2), WORLD_W, ROAD_W);
  }

  // Разметка
  ctx.strokeStyle = "#f6ff61";
  ctx.lineWidth = 4;
  ctx.setLineDash([25, 25]);

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

function drawRiver() {
  ctx.fillStyle = "#18b7ff";
  ctx.beginPath();
  ctx.moveTo(sx(0), sy(690));
  ctx.bezierCurveTo(sx(500), sy(630), sx(850), sy(750), sx(1300), sy(700));
  ctx.bezierCurveTo(sx(1800), sy(650), sx(2100), sy(760), sx(WORLD_W), sy(700));
  ctx.lineTo(sx(WORLD_W), sy(770));
  ctx.bezierCurveTo(sx(2100), sy(835), sx(1750), sy(735), sx(1300), sy(790));
  ctx.bezierCurveTo(sx(850), sy(840), sx(500), sy(720), sx(0), sy(780));
  ctx.closePath();
  ctx.fill();

  ctx.strokeStyle = "#fff";
  ctx.globalAlpha = 0.35;
  ctx.lineWidth = 3;
  ctx.beginPath();
  ctx.moveTo(sx(0), sy(725));
  ctx.bezierCurveTo(sx(500), sy(665), sx(850), sy(785), sx(1300), sy(735));
  ctx.bezierCurveTo(sx(1800), sy(685), sx(2100), sy(795), sx(WORLD_W), sy(735));
  ctx.stroke();
  ctx.globalAlpha = 1;
}

function drawBuildings() {
  for (const b of buildings) {
    const x = sx(b.x);
    const y = sy(b.y);

    ctx.fillStyle = "rgba(0,0,0,0.35)";
    ctx.fillRect(x + 8, y + 8, b.w, b.h);

    if (b.landmark === "baiterek") {
      ctx.fillStyle = "#ffd400";
      ctx.beginPath();
      ctx.arc(x + b.w / 2, y + b.h / 2, 60, 0, Math.PI * 2);
      ctx.fill();

      ctx.fillStyle = "#ffffff";
      ctx.beginPath();
      ctx.arc(x + b.w / 2, y + b.h / 2, 25, 0, Math.PI * 2);
      ctx.fill();

      ctx.fillStyle = "#ff8800";
      ctx.fillRect(x + b.w / 2 - 9, y + 35, 18, 100);
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
      for (let i = 1; i < 5; i++) {
        ctx.beginPath();
        ctx.moveTo(x + b.w * i / 5, y + b.h);
        ctx.lineTo(x + b.w / 2, y);
        ctx.stroke();
      }
    } else {
      ctx.fillStyle = b.color;
      ctx.fillRect(x, y, b.w, b.h);

      ctx.fillStyle = "rgba(255,255,255,0.35)";
      if (b.roof === "stripe") {
        for (let i = 0; i < b.w; i += 18) {
          ctx.fillRect(x + i, y, 8, b.h);
        }
      } else if (b.roof === "dots") {
        for (let px = 12; px < b.w; px += 28) {
          for (let py = 12; py < b.h; py += 28) {
            ctx.fillRect(x + px, y + py, 8, 8);
          }
        }
      } else {
        ctx.fillRect(x + 8, y + 8, b.w - 16, 8);
      }
    }
  }
}

function drawQuestMarkers() {
  if (!quest) return;

  const target = player.hasPackage ? quest.drop : quest.pickup;
  const color = player.hasPackage ? "#00ff66" : "#ffe600";
  const pulse = 12 + Math.sin(performance.now() / 130) * 5;

  ctx.strokeStyle = "#000";
  ctx.lineWidth = 7;
  ctx.beginPath();
  ctx.arc(sx(target.x), sy(target.y), 35 + pulse, 0, Math.PI * 2);
  ctx.stroke();

  ctx.strokeStyle = color;
  ctx.lineWidth = 5;
  ctx.beginPath();
  ctx.arc(sx(target.x), sy(target.y), 35 + pulse, 0, Math.PI * 2);
  ctx.stroke();

  ctx.fillStyle = color;
  ctx.beginPath();
  ctx.arc(sx(target.x), sy(target.y), 12, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = "#000";
  ctx.font = "bold 14px Arial";
  ctx.textAlign = "center";
  ctx.fillText(player.hasPackage ? "СДАТЬ" : "ВЗЯТЬ", sx(target.x), sy(target.y) - 45);

  // Стрелка к цели
  const dx = target.x - player.x;
  const dy = target.y - player.y;
  const angle = Math.atan2(dy, dx);

  const ax = canvas.width / 2 + Math.cos(angle) * 85;
  const ay = canvas.height / 2 + Math.sin(angle) * 85;

  ctx.save();
  ctx.translate(ax, ay);
  ctx.rotate(angle);
  ctx.fillStyle = color;
  ctx.beginPath();
  ctx.moveTo(20, 0);
  ctx.lineTo(-12, -10);
  ctx.lineTo(-6, 0);
  ctx.lineTo(-12, 10);
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

    ctx.fillStyle = "#ffffff";
    ctx.fillRect(x + 3, y + 3, 5, 5);
  }
}

function drawPlayer() {
  const x = sx(player.x);
  const y = sy(player.y);

  ctx.fillStyle = "rgba(0,0,0,0.35)";
  ctx.beginPath();
  ctx.ellipse(x + 5, y + 9, 18, 10, 0, 0, Math.PI * 2);
  ctx.fill();

  if (player.hitCooldown > 0 && Math.floor(performance.now() / 80) % 2 === 0) {
    ctx.globalAlpha = 0.45;
  }

  ctx.fillStyle = "#222";
  ctx.beginPath();
  ctx.arc(x, y, player.r + 3, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = "#00ffcc";
  ctx.beginPath();
  ctx.arc(x, y, player.r, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = "#ffeb3b";
  ctx.fillRect(x - 7, y - 22, 14, 9);

  if (player.hasPackage) {
    ctx.fillStyle = "#ff8c00";
    ctx.fillRect(x - 19, y - 4, 10, 13);
    ctx.strokeStyle = "#ffffff";
    ctx.strokeRect(x - 19, y - 4, 10, 13);
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
  ctx.fillStyle = "rgba(0,0,0,0.62)";
  ctx.fillRect(16, 16, 340, 118);

  ctx.strokeStyle = "rgba(255,255,255,0.35)";
  ctx.lineWidth = 2;
  ctx.strokeRect(16, 16, 340, 118);

  ctx.fillStyle = "#fff";
  ctx.font = "bold 20px Arial";
  ctx.textAlign = "left";
  ctx.fillText("ASTANA COURIER", 30, 44);

  ctx.font = "15px Arial";
  ctx.fillText(`Уровень: ${player.level}`, 30, 68);
  ctx.fillText(`Доставки: ${player.deliveries}`, 140, 68);
  ctx.fillText(`Деньги: ${player.money}₸`, 250, 68);

  ctx.fillStyle = "#222";
  ctx.fillRect(30, 82, 180, 16);
  ctx.fillStyle = player.health > 35 ? "#00ff66" : "#ff304f";
  ctx.fillRect(30, 82, 180 * player.health / 100, 16);
  ctx.strokeStyle = "#fff";
  ctx.strokeRect(30, 82, 180, 16);

  ctx.fillStyle = "#fff";
  ctx.fillText(`HP ${Math.floor(player.health)}`, 220, 96);

  if (quest) {
    ctx.fillStyle = "#fff";
    const text = player.hasPackage
      ? `Куда: ${quest.drop.name}`
      : `Где взять: ${quest.pickup.name}`;
    ctx.fillText(text.slice(0, 34), 30, 122);

    if (player.hasPackage) {
      const t = Math.ceil(quest.timeLeft);
      ctx.fillStyle = t < 10 ? "#ff304f" : "#ffe600";
      ctx.font = "bold 18px Arial";
      ctx.fillText(`Время: ${t} сек`, 225, 122);
    }
  }

  if (messageTimer > 0) {
    ctx.fillStyle = "rgba(0,0,0,0.65)";
    ctx.fillRect(canvas.width / 2 - 330, 22, 660, 46);
    ctx.strokeStyle = "rgba(255,255,255,0.35)";
    ctx.strokeRect(canvas.width / 2 - 330, 22, 660, 46);

    ctx.fillStyle = "#fff";
    ctx.font = "bold 17px Arial";
    ctx.textAlign = "center";
    ctx.fillText(message, canvas.width / 2, 52);
  }

  drawMiniMap();
}

function drawMiniMap() {
  const mw = 210;
  const mh = 145;
  const mx = canvas.width - mw - 18;
  const my = 18;

  ctx.fillStyle = "rgba(0,0,0,0.65)";
  ctx.fillRect(mx, my, mw, mh);

  ctx.strokeStyle = "rgba(255,255,255,0.45)";
  ctx.strokeRect(mx, my, mw, mh);

  const scaleX = mw / WORLD_W;
  const scaleY = mh / WORLD_H;

  ctx.fillStyle = "#555";
  for (const x of verticalRoads) {
    ctx.fillRect(mx + (x - ROAD_W/2) * scaleX, my, ROAD_W * scaleX, mh);
  }
  for (const y of horizontalRoads) {
    ctx.fillRect(mx, my + (y - ROAD_W/2) * scaleY, mw, ROAD_W * scaleY);
  }

  if (quest) {
    const target = player.hasPackage ? quest.drop : quest.pickup;
    ctx.fillStyle = player.hasPackage ? "#00ff66" : "#ffe600";
    ctx.beginPath();
    ctx.arc(mx + target.x * scaleX, my + target.y * scaleY, 5, 0, Math.PI * 2);
    ctx.fill();
  }

  ctx.fillStyle = "#00eaff";
  ctx.beginPath();
  ctx.arc(mx + player.x * scaleX, my + player.y * scaleY, 5, 0, Math.PI * 2);
  ctx.fill();
}

function drawWorldBorder() {
  ctx.strokeStyle = "#ffffff";
  ctx.lineWidth = 8;
  ctx.strokeRect(sx(0), sy(0), WORLD_W, WORLD_H);
}

function draw() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  ctx.fillStyle = "#43d96a";
  ctx.fillRect(sx(0), sy(0), WORLD_W, WORLD_H);

  // Парки
  ctx.fillStyle = "#1fbf55";
  ctx.fillRect(sx(40), sy(40), 160, 120);
  ctx.fillRect(sx(2180), sy(1500), 260, 120);
  ctx.fillRect(sx(760), sy(1180), 160, 180);

  drawRiver();
  drawRoads();
  drawBuildings();
  drawQuestMarkers();
  drawCars();
  drawParticles();
  drawPlayer();
  drawWorldBorder();
  drawUI();
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
