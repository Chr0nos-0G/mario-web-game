const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
const timeEl = document.getElementById('time');
const coinsEl = document.getElementById('coins');

const TILE = 32;
const GRAVITY = 0.7;
const WORLD_WIDTH = 220 * TILE;
const WORLD_HEIGHT = 15 * TILE;
const FLAG_X = 200 * TILE;

const keys = {};

const level = [
  '..........................................................................................',
  '..........................................................................................',
  '..........................................................................................',
  '..........................................................................................',
  '..........................................................................................',
  '..........c..............................?............................?.................c.....',
  '..........................................................................................',
  '......................................?....................?.............................',
  '..........................................................................................',
  '......................#####.................................#####........................',
  '..............................................................#......................',
  '....??..............??....####..........................??....####..............??........',
  '..........................................................................................',
  '.......o...........o..............o..........o.........o.................................',
  '################################################################################################',
  '################################################################################################'
];

const map = [];
for (let y = 0; y < level.length; y++) {
  const row = [];
  for (let x = 0; x < level[y].length; x++) {
    const ch = level[y][x];
    if (ch === '#') row.push('ground');
    else if (ch === '?') row.push('brick');
    else if (ch === 'c') row.push('coin');
    else if (ch === 'o') row.push('platform');
    else row.push(null);
  }
  map.push(row);
}

const bricks = [];
const coins = [];
const enemies = [];

function addBrick(x, y) {
  bricks.push({ x, y, w: TILE, h: TILE, solid: true, type: 'brick' });
}

function addCoin(x, y) {
  coins.push({ x: x + TILE * 0.25, y: y + TILE * 0.25, r: 8, collected: false });
}

function buildLevel() {
  for (let y = 0; y < map.length; y++) {
    for (let x = 0; x < map[y].length; x++) {
      const tile = map[y][x];
      if (!tile) continue;
      const px = x * TILE;
      const py = y * TILE;

      if (tile === 'ground') {
        addBrick(px, py);
      } else if (tile === 'brick') {
        addBrick(px, py);
      } else if (tile === 'platform') {
        bricks.push({ x: px, y: py, w: TILE, h: TILE / 2, solid: true, type: 'platform' });
      } else if (tile === 'coin') {
        addCoin(px, py);
      }
    }
  }

  enemies.push({ x: 18 * TILE, y: 13 * TILE - 16, w: 24, h: 24, vx: -1.2, alive: true });
  enemies.push({ x: 56 * TILE, y: 13 * TILE - 16, w: 24, h: 24, vx: 1.2, alive: true });
  enemies.push({ x: 116 * TILE, y: 13 * TILE - 16, w: 24, h: 24, vx: -1.1, alive: true });
}

buildLevel();

const player = {
  x: 2 * TILE,
  y: 10 * TILE,
  w: 28,
  h: 32,
  vx: 0,
  vy: 0,
  speed: 3.4,
  jumpForce: 12.5,
  onGround: false,
  facing: 1,
  lives: 3,
  invincible: 0,
  coins: 0,
  won: false,
};

let cameraX = 0;
let timer = 300;
let lastTime = 0;

function rectsOverlap(a, b) {
  return (
    a.x < b.x + b.w &&
    a.x + a.w > b.x &&
    a.y < b.y + b.h &&
    a.y + a.h > b.y
  );
}

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}

function handleInput() {
  let move = 0;
  if (keys.ArrowLeft || keys.a) move -= 1;
  if (keys.ArrowRight || keys.d) move += 1;

  player.vx = move * player.speed;
  if (move !== 0) player.facing = move;

  if ((keys.ArrowUp || keys.w || keys[' ']) && player.onGround) {
    player.vy = -player.jumpForce;
    player.onGround = false;
  }
}

function updatePhysics() {
  player.vy += GRAVITY;
  player.x += player.vx;
  player.y += player.vy;

  player.onGround = false;

  for (const block of bricks) {
    const hit = rectsOverlap(player, block);
    if (!hit) continue;

    const prevX = player.x - player.vx;
    const prevY = player.y - player.vy;

    if (prevY + player.h <= block.y + 8 && player.vy >= 0) {
      player.y = block.y - player.h;
      player.vy = 0;
      player.onGround = true;
    } else if (prevY >= block.y + block.h - 8 && player.vy < 0) {
      player.y = block.y + block.h;
      player.vy = 0;
    } else if (prevX + player.w <= block.x + 8 && player.vx > 0) {
      player.x = block.x - player.w;
      player.vx = 0;
    } else if (prevX >= block.x + block.w - 8 && player.vx < 0) {
      player.x = block.x + block.w;
      player.vx = 0;
    }
  }

  player.x = clamp(player.x, 0, WORLD_WIDTH - player.w);
  if (player.y > canvas.height + 200) {
    resetPlayer();
  }

  for (const coin of coins) {
    if (!coin.collected) {
      const coinBox = { x: coin.x, y: coin.y, w: 16, h: 16 };
      if (rectsOverlap(player, coinBox)) {
        coin.collected = true;
        player.coins += 1;
        coinsEl.textContent = String(player.coins);
      }
    }
  }

  for (const enemy of enemies) {
    if (!enemy.alive) continue;
    enemy.x += enemy.vx;

    const groundY = 13 * TILE - enemy.h;
    if (enemy.x < 10 * TILE || enemy.x > 150 * TILE) {
      enemy.vx *= -1;
    }

    enemy.x = clamp(enemy.x, 0, WORLD_WIDTH - enemy.w);

    const playerHit = rectsOverlap(
      { x: player.x, y: player.y, w: player.w, h: player.h },
      { x: enemy.x, y: enemy.y, w: enemy.w, h: enemy.h }
    );

    if (playerHit) {
      if (player.vy > 0 && player.y + player.h - enemy.y < 18) {
        enemy.alive = false;
        player.vy = -8;
      } else {
        resetPlayer();
      }
    }
  }

  if (player.x > FLAG_X) {
    player.won = true;
  }
}

function resetPlayer() {
  player.x = 2 * TILE;
  player.y = 10 * TILE;
  player.vx = 0;
  player.vy = 0;
  player.onGround = false;
}

function drawBackground() {
  ctx.fillStyle = '#7ec9ff';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  ctx.fillStyle = '#bfe7ff';
  for (let i = 0; i < 6; i++) {
    const x = ((i * 180) - cameraX * 0.35) % (canvas.width + 200);
    const y = 80 + (i % 2) * 40;
    ctx.beginPath();
    ctx.arc(x, y, 28, 0, Math.PI * 2);
    ctx.arc(x + 28, y - 12, 24, 0, Math.PI * 2);
    ctx.arc(x + 56, y, 28, 0, Math.PI * 2);
    ctx.fill();
  }

  ctx.fillStyle = '#6bbb59';
  for (let i = 0; i < 10; i++) {
    const hillX = ((i * 160) - cameraX * 0.5) % (canvas.width + 220);
    ctx.beginPath();
    ctx.moveTo(hillX, 420);
    ctx.quadraticCurveTo(hillX + 50, 250, hillX + 120, 420);
    ctx.fill();
  }
}

function drawBlocks() {
  for (const block of bricks) {
    const x = block.x - cameraX;
    const y = block.y;

    if (x + block.w < -20 || x > canvas.width + 20) continue;

    if (block.type === 'platform') {
      ctx.fillStyle = '#9a6a3d';
      ctx.fillRect(x, y, block.w, block.h);
      ctx.fillStyle = '#b07e52';
      ctx.fillRect(x + 2, y + 2, block.w - 4, 6);
    } else {
      ctx.fillStyle = '#b96a2f';
      ctx.fillRect(x, y, block.w, block.h);
      ctx.fillStyle = '#dea873';
      ctx.fillRect(x + 2, y + 2, block.w - 4, 6);
      ctx.fillStyle = '#8f4b23';
      ctx.fillRect(x + 4, y + 12, block.w - 8, block.h - 14);
    }
  }
}

function drawCoins() {
  for (const coin of coins) {
    if (coin.collected) continue;
    const cx = coin.x - cameraX;
    const cy = coin.y;

    ctx.fillStyle = '#ffd84d';
    ctx.beginPath();
    ctx.arc(cx, cy, 8, 0, Math.PI * 2);
    ctx.fill();
    ctx.fillStyle = '#ffed9b';
    ctx.fillRect(cx - 2, cy - 8, 4, 16);
  }
}

function drawEnemies() {
  for (const enemy of enemies) {
    if (!enemy.alive) continue;
    const drawX = enemy.x - cameraX;
    const drawY = enemy.y;

    ctx.fillStyle = '#7d4d2a';
    ctx.fillRect(drawX, drawY + 8, enemy.w, enemy.h - 8);
    ctx.fillStyle = '#c98342';
    ctx.fillRect(drawX + 4, drawY, enemy.w - 8, 10);
    ctx.fillStyle = '#000';
    ctx.fillRect(drawX + 5, drawY + 6, 3, 3);
    ctx.fillRect(drawX + enemy.w - 8, drawY + 6, 3, 3);
  }
}

function drawPlayer() {
  const drawX = player.x - cameraX;
  const drawY = player.y;

  ctx.fillStyle = '#f8c27b';
  ctx.fillRect(drawX + 5, drawY + 2, 18, 12);

  ctx.fillStyle = '#c73d2e';
  ctx.fillRect(drawX + 3, drawY + 12, 22, 8);

  ctx.fillStyle = '#2b6dff';
  ctx.fillRect(drawX + 6, drawY + 20, 16, 12);

  ctx.fillStyle = '#000';
  ctx.fillRect(drawX + 7, drawY + 12, 3, 3);
  ctx.fillRect(drawX + 18, drawY + 12, 3, 3);

  if (player.facing === -1) {
    ctx.fillStyle = '#000';
    ctx.fillRect(drawX + 1, drawY + 18, 4, 9);
    ctx.fillRect(drawX + 23, drawY + 18, 4, 9);
  } else {
    ctx.fillStyle = '#000';
    ctx.fillRect(drawX + 23, drawY + 18, 4, 9);
    ctx.fillRect(drawX + 1, drawY + 18, 4, 9);
  }
}

function drawFlag() {
  const x = FLAG_X - cameraX;
  ctx.fillStyle = '#d8d8d8';
  ctx.fillRect(x, 220, 4, 180);
  ctx.fillStyle = '#ff3b30';
  ctx.fillRect(x + 4, 230, 70, 40);
  ctx.fillStyle = '#fff';
  ctx.fillRect(x + 10, 238, 58, 18);
}

function drawGroundShadow() {
  ctx.fillStyle = '#4d8a2d';
  ctx.fillRect(0, 440, canvas.width, 100);
  ctx.fillStyle = '#6dc14b';
  ctx.fillRect(0, 430, canvas.width, 20);
}

function render() {
  drawBackground();
  drawGroundShadow();
  drawBlocks();
  drawCoins();
  drawEnemies();
  drawFlag();
  drawPlayer();

  if (player.won) {
    ctx.fillStyle = 'rgba(0,0,0,0.45)';
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    ctx.fillStyle = '#fff';
    ctx.font = 'bold 40px Arial';
    ctx.fillText('Niveau terminé !', canvas.width / 2 - 180, canvas.height / 2);
    ctx.font = '20px Arial';
    ctx.fillText('Appuie sur R pour rejouer', canvas.width / 2 - 120, canvas.height / 2 + 40);
  }
}

function update(delta) {
  if (player.won) {
    if (keys.r) {
      location.reload();
    }
    return;
  }

  handleInput();
  updatePhysics();

  cameraX = clamp(player.x - canvas.width * 0.35, 0, WORLD_WIDTH - canvas.width);

  timer -= delta * 0.06;
  if (timer < 0) timer = 0;
  timeEl.textContent = Math.ceil(timer);
}

function gameLoop(timestamp) {
  const delta = Math.min(32, timestamp - lastTime || 16);
  lastTime = timestamp;
  update(delta);
  render();
  requestAnimationFrame(gameLoop);
}

window.addEventListener('keydown', (event) => {
  const k = event.key;
  keys[k] = true;
  keys[k.toLowerCase()] = true;
  if (k === ' ') event.preventDefault();
});

window.addEventListener('keyup', (event) => {
  const k = event.key;
  keys[k] = false;
  keys[k.toLowerCase()] = false;
});

requestAnimationFrame(gameLoop);
