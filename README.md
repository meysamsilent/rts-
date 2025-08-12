# rts-
<!DOCTYPE html>
<html lang="fa">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>بازی RTS بهتر</title>
<style>
  body {
    font-family: Tahoma, sans-serif;
    background: #1e1e2f;
    color: #eee;
    margin: 0; padding: 0;
    display: flex; flex-direction: column; align-items: center;
  }
  h1 {
    margin: 10px;
  }
  #game {
    margin-top: 15px;
    display: grid;
    grid-template-columns: repeat(8, 50px);
    grid-template-rows: repeat(8, 50px);
    gap: 2px;
    background: #333;
    border-radius: 8px;
    user-select: none;
  }
  .cell {
    width: 50px; height: 50px;
    background: #556;
    border-radius: 5px;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    position: relative;
    transition: background-color 0.3s;
  }
  .cell:hover {
    background-color: #778899;
  }
  .base-player {
    background: linear-gradient(45deg, #22aa22, #117711);
    box-shadow: 0 0 8px #22ff22;
  }
  .base-enemy {
    background: linear-gradient(45deg, #aa2222, #771111);
    box-shadow: 0 0 8px #ff2222;
  }
  .soldier {
    width: 36px; height: 36px;
    pointer-events: none;
  }
  #info {
    margin-top: 15px;
    background: #222;
    padding: 15px;
    border-radius: 8px;
    width: 420px;
    text-align: center;
  }
  button {
    margin: 5px;
    padding: 8px 14px;
    font-size: 15px;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    background: #3a7;
    color: #fff;
    transition: background-color 0.3s;
  }
  button:hover {
    background: #5c9;
  }
  #resources {
    font-weight: bold;
    font-size: 18px;
  }
  .selected {
    outline: 3px solid #ff0;
  }
  .enemy-soldier {
    filter: drop-shadow(0 0 3px red);
  }
</style>
</head>
<body>

<h1>بازی RTS ساده و بهتر</h1>

<div id="game"></div>

<div id="info">
  <div>منابع: <span id="resources">100</span></div>
  <div>
    ساخت سرباز:
    <button data-type="1">سرباز سریع (30)</button>
    <button data-type="2">سرباز قوی (50)</button>
    <button data-type="3">سرباز جان سخت (40)</button>
  </div>
  <div>روی سرباز خودت کلیک کن و سپس خانه مقصد برای حرکت یا حمله انتخاب کن.</div>
</div>

<script>
  const gridSize = 8;
  const gameDiv = document.getElementById('game');
  const resourcesSpan = document.getElementById('resources');

  let resources = 100;
  let selectedSoldier = null;

  // سربازها با مشخصات و آیکون SVG ساده
  const soldierTypes = {
    1: {
      name: 'سریع',
      hp: 30,
      attack: 10,
      speed: 3,
      cost: 30,
      svg: `<svg class="soldier" viewBox="0 0 64 64" fill="gold" xmlns="http://www.w3.org/2000/svg">
        <circle cx="32" cy="32" r="28" stroke="orange" stroke-width="3"/>
        <polygon points="32,10 45,54 19,54" fill="goldenrod"/>
      </svg>`
    },
    2: {
      name: 'قوی',
      hp: 50,
      attack: 20,
      speed: 1,
      cost: 50,
      svg: `<svg class="soldier" viewBox="0 0 64 64" fill="#0ff" xmlns="http://www.w3.org/2000/svg">
        <rect x="16" y="16" width="32" height="32" rx="8" ry="8" stroke="cyan" stroke-width="3"/>
        <circle cx="32" cy="32" r="12" fill="#0ff" />
      </svg>`
    },
    3: {
      name: 'جان سخت',
      hp: 40,
      attack: 15,
      speed: 2,
      cost: 40,
      svg: `<svg class="soldier" viewBox="0 0 64 64" fill="magenta" xmlns="http://www.w3.org/2000/svg">
        <ellipse cx="32" cy="32" rx="20" ry="14" stroke="purple" stroke-width="3"/>
        <line x1="12" y1="12" x2="52" y2="52" stroke="purple" stroke-width="3"/>
        <line x1="12" y1="52" x2="52" y2="12" stroke="purple" stroke-width="3"/>
      </svg>`
    }
  };

  // نقشه 2بعدی: null یا پایگاه یا سرباز
  let map = [];
  for(let y=0; y<gridSize; y++) {
    map[y] = [];
    for(let x=0; x<gridSize; x++) {
      map[y][x] = null;
    }
  }

  // پایگاه‌ها
  const playerBase = {x:0, y:gridSize-1};
  const enemyBase = {x:gridSize-1, y:0};
  map[playerBase.y][playerBase.x] = 'base-player';
  map[enemyBase.y][enemyBase.x] = 'base-enemy';

  // سربازهای بازیکن و دشمن
  let soldiers = [];
  let enemySoldiers = [];

  function drawMap() {
    gameDiv.innerHTML = '';
    for(let y=0; y<gridSize; y++) {
      for(let x=0; x<gridSize; x++) {
        const cell = document.createElement('div');
        cell.classList.add('cell');

        const val = map[y][x];
        if(val === 'base-player') cell.classList.add('base-player');
        else if(val === 'base-enemy') cell.classList.add('base-enemy');
        else if(val && val.owner) {
          if(val.owner === 'player') {
            cell.innerHTML = soldierTypes[val.typeId].svg;
            if(selectedSoldier === val) cell.classList.add('selected');
          } else {
            cell.innerHTML = soldierTypes[val.typeId].svg;
            cell.firstChild.classList.add('enemy-soldier');
          }
        }

        cell.dataset.x = x;
        cell.dataset.y = y;

        cell.addEventListener('click', () => onCellClick(x,y));

        gameDiv.appendChild(cell);
      }
    }
  }

  function createSoldier(typeId) {
    const t = soldierTypes[typeId];
    if(resources < t.cost) {
      alert('منابع کافی نیست!');
      return;
    }
    resources -= t.cost;
    resourcesSpan.textContent = resources;

    const soldier = {
      x: playerBase.x,
      y: playerBase.y,
      hp: t.hp,
      attack: t.attack,
      speed: t.speed,
      typeId: typeId,
      owner: 'player',
      movedThisTurn: false,
    };
    soldiers.push(soldier);
    map[soldier.y][soldier.x] = soldier;
    drawMap();
  }

  function onCellClick(x,y) {
    const cellVal = map[y][x];

    if(selectedSoldier) {
      // اگر انتخاب شدی، بررسی حرکت یا حمله
      const dist = Math.abs(selectedSoldier.x - x) + Math.abs(selectedSoldier.y - y);
      if(dist > selectedSoldier.speed) {
        alert('نمی‌توانی بیش از سرعت خود حرکت کنی.');
        selectedSoldier = null;
        drawMap();
        return;
      }
      if(selectedSoldier.movedThisTurn) {
        alert('این سرباز در این نوبت حرکت کرده.');
        selectedSoldier = null;
        drawMap();
        return;
      }

      if(cellVal === null) {
        // حرکت
        map[selectedSoldier.y][selectedSoldier.x] = null;
        selectedSoldier.x = x;
        selectedSoldier.y = y;
        map[y][x] = selectedSoldier;
        selectedSoldier.movedThisTurn = true;
        selectedSoldier = null;
        drawMap();
        enemyTurn();
      } else if(cellVal.owner === 'enemy') {
        // حمله
        cellVal.hp -= selectedSoldier.attack;
        selectedSoldier.movedThisTurn = true;
        if(cellVal.hp <= 0) {
          enemySoldiers = enemySoldiers.filter(s => s !== cellVal);
          map[y][x] = null;
        }
        selectedSoldier = null;
        drawMap();
        enemyTurn();
      } else if(cellVal === 'base-enemy') {
        alert('پیروزی! پایگاه دشمن نابود شد!');
        resetGame();
      } else {
        alert('نمی‌توانی به این خانه حرکت کنی.');
        selectedSoldier = null;
        drawMap();
      }

    } else {
      // انتخاب سرباز خودی
      if(cellVal && cellVal.owner === 'player' && !cellVal.movedThisTurn) {
        selectedSoldier = cellVal;
        drawMap();
      }
    }
  }

  function enemyTurn() {
    // بازنشانی حرکت سربازان خودی
    soldiers.forEach(s => s.movedThisTurn = false);

    // دشمن سرباز می‌سازد اگر کمتر از 3 تا دارد و منابع دارد
    if(enemySoldiers.length < 3) {
      const etypeId = 1 + Math.floor(Math.random() * 3);
      const etype = soldierTypes[etypeId];
      if(resources >= etype.cost) {
        resources -= etype.cost;
        resourcesSpan.textContent = resources;
        const esoldier = {
          x: enemyBase.x,
          y: enemyBase.y,
          hp: etype.hp,
          attack: etype.attack,
          speed: etype.speed,
          typeId: etypeId,
          owner: 'enemy',
          movedThisTurn: false,
        };
        enemySoldiers.push(esoldier);
        map[enemyBase.y][enemyBase.x] = esoldier;
      }
    }

    // حرکت ساده دشمن به سمت پایگاه بازیکن
    enemySoldiers.forEach(es => {
      if(es.movedThisTurn) return;
      const dx = playerBase.x - es.x;
      const dy = playerBase.y - es.y;
      const dist = Math.abs(dx) + Math.abs(dy);

      if(dist === 1) {
        alert('شکست! پایگاه شما نابود شد!');
        resetGame();
        return;
      }

      let nx = es.x;
      let ny = es.y;
      if(Math.abs(dx) > Math.abs(dy)) {
        nx += dx > 0 ? 1 : -1;
      } else {
        ny += dy > 0 ? 1 : -1;
      }
      if(map[ny][nx] === null) {
        map[es.y][es.x] = null;
        es.x = nx;
        es.y = ny;
        map[ny][nx] = es;
      }
      es.movedThisTurn = true;
    });

    drawMap();
  }

  function resetGame() {
    if(confirm('می‌خواهید دوباره بازی کنید؟')) {
      location.reload();
    }
  }

  document.querySelectorAll('#info button').forEach(btn => {
    btn.addEventListener('click', () => {
      createSoldier(parseInt(btn.dataset.type));
    });
  });

  drawMap();
  resourcesSpan.textContent = resources;

</script>

</body>
</html>
