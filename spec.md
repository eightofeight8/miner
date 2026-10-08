# Техническая спецификация: Веб-игра «Сапёр» (Single-File Architecture)

**Версия:** 1.0.0  
**Целевой артефакт:** `index.html` (монолитный файл: HTML5 + CSS3 + Vanilla JS)  
**Зависимости:** 0 (без внешних библиотек, шрифтов и сетевых ассетов)  
**Целевой исполнитель:** Antigravity CLI / AI Code Generator

---

## 1. Архитектурные требования и ограничения

1. **Единый файл:** Вся разметка, CSS-стили и JavaScript-код обязаны находиться строго в одном файле `index.html`.
2. **Нулевой сетевой след (Offline-first):**
   * Запрещены теги `<link>` и `<script>` с внешними URL (CDN, Google Fonts и т.д.).
   * Графика, иконки и глифы реализуются строго через Unicode/Emoji (💣, 🚩, 🙂, 😮, 😎, 😵) либо инлайновый SVG / CSS-формы.
3. **Совместимость с мобильными устройствами:**
   * Отключение дефолтного зума по двойному тапу (`touch-action: manipulation`).
   * Полное подавление контекстного меню браузера на игровом поле (`contextmenu -> e.preventDefault()`).
   * Запрет на выделение текста (`user-select: none`).

---

## 2. Спецификация DOM-структуры

Корневая разметка файла должна соответствовать следующей иерархии ID и классов:

```html
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>Сапёр / Minesweeper</title>
  <style>/* CSS Спецификация */</style>
</head>
<body>
  <div id="game-app" class="game-container">
    <!-- Панель управления сложностью -->
    <header class="game-toolbar">
      <div class="difficulty-group" role="radiogroup" aria-label="Сложность">
        <button type="button" class="btn-diff active" data-difficulty="easy">9×9</button>
        <button type="button" class="btn-diff" data-difficulty="medium">16×16</button>
        <button type="button" class="btn-diff" data-difficulty="hard">30×16</button>
      </div>
      <!-- Мобильный тумблер режима (Копать / Флаг) -->
      <button type="button" id="mode-toggle-btn" class="btn-mode-toggle" aria-label="Режим клика">
        <span class="mode-icon">⛏️</span>
        <span class="mode-text">Копать</span>
      </button>
    </header>

    <!-- Основной блок игры (Windows Classic style) -->
    <main class="game-window">
      <!-- Верхний статус-бар -->
      <section class="status-panel">
        <div id="mine-counter" class="digital-display" aria-live="polite">010</div>
        <button type="button" id="face-reset-btn" class="face-btn" aria-label="Новая игра">🙂</button>
        <div id="timer-display" class="digital-display" aria-live="off">000</div>
      </section>

      <!-- Контейнер скролла для больших полей -->
      <section id="board-viewport" class="board-viewport">
        <div id="minefield" class="minefield" role="grid" aria-label="Игровое поле">
          <!-- Генерируется через JS -->
        </div>
      </section>
    </main>
  </div>
  <script>/* JS Спецификация */</script>
</body>
</html>
```

---

## 3. Спецификация стилей (CSS)

### 3.1. Дизайн-система и переменные
```css
:root {
  --bg-color: #c0c0c0;
  --border-light: #ffffff;
  --border-dark: #808080;
  --border-shadow: #000000;
  --display-bg: #000000;
  --display-red: #ff0000;

  /* Размеры ячеек */
  --cell-size: 32px;
  --cols: 9;
  --rows: 9;

  /* Палитра цифр */
  --c1: #0000ff; /* 1: Синий */
  --c2: #008000; /* 2: Зеленый */
  --c3: #ff0000; /* 3: Красный */
  --c4: #000080; /* 4: Темно-синий */
  --c5: #800000; /* 5: Бордовый */
  --c6: #008080; /* 6: Бирюзовый */
  --c7: #000000; /* 7: Черный */
  --c8: #808080; /* 8: Серый */
}
```

### 3.2. Правила компоновки и состояний ячеек
1. **Сетка `#minefield`:**
   ```css
   display: grid;
   grid-template-columns: repeat(var(--cols), var(--cell-size));
   grid-template-rows: repeat(var(--rows), var(--cell-size));
   user-select: none;
   touch-action: manipulation;
   margin: 0 auto;
   ```
2. **Базовая ячейка `.cell`:**
   * Размер: строго `var(--cell-size) × var(--cell-size)`.
   * Закрытая ячейка: выпуклый псевдо-3D бордер:
     `border-top: 3px solid var(--border-light); border-left: 3px solid var(--border-light); border-bottom: 3px solid var(--border-dark); border-right: 3px solid var(--border-dark); background: var(--bg-color);`.
3. **Открытая ячейка `.cell.revealed`:**
   * Плоский или вдавленный бордер: `border: 1px solid var(--border-dark); background: #dcdcdc;`.
   * Центрирование содержимого через `display: flex; align-items: center; justify-content: center; font-weight: bold; font-family: monospace;`.
4. **Специфические модификаторы ячеек:**
   * `.cell.flagged` — отображает иконку 🚩.
   * `.cell.exploded` — фон `#ff4444`, отображает 💣.
   * `.cell.mine` — фоновое отображение мины 💣 при поражении.
   * `.cell.false-flag` — флаг, под которым не было мины (перечеркнутый флаг или красный крест при Game Over).
   * Классы цифр `.cell[data-count="1"]` до `[data-count="8"]` с соответствующим `var(--cN)`.

### 3.3. Адаптивность и мобильный Viewport
```css
.board-viewport {
  max-width: 100vw;
  max-height: calc(100vh - 160px);
  overflow: auto;
  -webkit-overflow-scrolling: touch;
  padding: 6px;
}

@media (max-width: 600px) {
  :root {
    --cell-size: 34px; /* Оптимальная тач-зона */
  }
  .game-container {
    padding: 4px;
    width: 100%;
  }
}
```

---

## 4. Спецификация логики и структуры данных (JavaScript)

### 4.1. Конфигурация уровней
```javascript
const CONFIGS = {
  easy:   { rows: 9,  cols: 9,  mines: 10 },
  medium: { rows: 16, cols: 16, mines: 40 },
  hard:   { rows: 16, cols: 30, mines: 99 }
};
```

### 4.2. Модель состояния (Game State)
```javascript
const state = {
  difficulty: 'easy',
  rows: 9,
  cols: 9,
  totalMines: 10,
  flagsCount: 0,
  revealedCount: 0,
  status: 'IDLE', // 'IDLE' | 'PLAYING' | 'WON' | 'LOST'
  board: [],      // Array<Array<CellState>>
  firstClick: true,
  timer: 0,
  timerId: null,
  mobileMode: 'dig' // 'dig' | 'flag'
};
```

### 4.3. Структура сущности `CellState`
```javascript
/**
 * @typedef {Object} CellState
 * @property {number} r - Индекс строки (0 .. rows - 1)
 * @property {number} c - Индекс колонки (0 .. cols - 1)
 * @property {boolean} isMine - Наличие мины
 * @property {boolean} isRevealed - Открыта ли клетка
 * @property {boolean} isFlagged - Установлен ли флаг
 * @property {number} count - Количество соседних мин (0 .. 8)
 * @property {HTMLElement|null} element - Прямая ссылка на DOM-ноду ячейки
 */
```

---

## 5. Алгоритмические спецификации

### 5.1. Безопасная инициализация (Safe First Click)
Генерация расположения мин вызывается **строго при первом вызове** функции раскрытия ячейки `handleCellOpen(startR, startC)`:
1. Создается массив всех доступных координат `(r, c)`.
2. **Исключение зоны первого клика:** Координата `(startR, startC)` и её 8 непосредственных соседей исключаются из пула кандидатов для мин. Это гарантирует, что первый клик всегда открывает пустую зону со значением `count = 0` (каскадное открытие), создавая удобный старт.
   * *Fallback:* Если количество мин превышает число оставшихся ячеек поля (не применимо в базовых пресетах, но критично для архитектурной надежности), исключается только клетка `(startR, startC)`.
3. Из оставшегося пула случайным образом (алгоритм тасования Фишера — Йетса) выбираются `CONFIGS[current].mines` ячеек.
4. Выполняется проход по всем ячейкам с вычислением `count` — суммы мин во всех смежных ячейках в радиусе 1.

### 5.2. Каскадное открытие (Flood Fill via BFS)
Для предотвращения переполнения стека (Call Stack Overflow) на больших полях запрещено использовать неограниченную прямую рекурсию. Используется BFS-очередь:

```javascript
function floodFill(startR, startC) {
  const queue = [[startR, startC]];
  
  while (queue.length > 0) {
    const [r, c] = queue.shift();
    const cell = state.board[r][c];

    // Получаем 8 смежных ячеек
    const neighbors = getNeighbors(r, c);
    for (const n of neighbors) {
      if (!n.isRevealed && !n.isFlagged && !n.isMine) {
        n.isRevealed = true;
        state.revealedCount++;
        renderCell(n);

        // Если у соседа тоже 0 смежных мин, продолжаем каскад
        if (n.count === 0) {
          queue.push([n.r, n.c]);
        }
      }
    }
  }
}
```

### 5.3. Механика «Аккорд» (Chording)
Вызывается при клике на уже открытую ячейку с числом `count > 0`:
1. Подсчитывается количество флагов вокруг данной клетки (`flagCount`).
2. Если `flagCount === cell.count`:
   * Инициируется раскрытие всех закрытых соседей, на которых **нет** флага.
   * Если среди этих соседей оказывается мина (ошибочно выставленный игроком флаг) — игра немедленно переходит в статус `LOST`.

### 5.4. Условия завершения игры
* **Поражение (`LOST`):**
  1. Остановка таймера.
  2. Смайлик переключается на 😵.
  3. Взорванная мина получает класс `.exploded`.
  4. Раскрываются все остальные мины (`.mine`).
  5. Неверно выставленные флаги помечаются классом `.false-flag`.
  6. Все слушатели игровых действий блокируются.
* **Победа (`WON`):**
  1. Условие: `state.revealedCount === (state.rows * state.cols - state.totalMines)`.
  2. Остановка таймера.
  3. Смайлик переключается на 😎.
  4. На все оставшиеся закрытые мины автоматически выставляются флаги 🚩, счетчик мин сбрасывается в `000`.

---

## 6. Спецификация системы управления и событий

Для обеспечения полной кроссплатформенности используется связка Pointer Events и Context Menu API.

### 6.1. Матрица взаимодействий

| Устройство | Действие | Логика |
| :--- | :--- | :--- |
| **Mouse (Desktop)** | Клик ЛКМ | `handleCellOpen(r, c)` |
| | Клик ПКМ | `handleCellToggleFlag(r, c)` |
| | Клик ЛКМ по открытой цифре | Вызов алгоритма `Chording` |
| | Mousedown на поле | Смайлик переключается на 😮 |
| | Mouseup / Cancel | Смайлик возвращается в 🙂 |
| **Touch (Mobile)** | Тап в режиме `dig` | `handleCellOpen(r, c)` |
| | Тап в режиме `flag` | `handleCellToggleFlag(r, c)` |
| | Долгий тап (> 350мс) | `handleCellToggleFlag(r, c)` + `navigator.vibrate(40)` |
| | Движение пальца (> 10px) | Отмена таймера долгого тапа (скролл поля) |

### 6.2. Детальная обработка Touch / Long-Press
```javascript
let touchTimer = null;
let touchStartX = 0;
let touchStartY = 0;
let isLongPress = false;

function onPointerDown(e) {
  if (e.pointerType === 'touch') {
    isLongPress = false;
    touchStartX = e.clientX;
    touchStartY = e.clientY;
    touchTimer = setTimeout(() => {
      isLongPress = true;
      handleCellToggleFlag(r, c);
      if (navigator.vibrate) navigator.vibrate(40);
    }, 350);
  }
}

function onPointerMove(e) {
  if (touchTimer) {
    const diffX = Math.abs(e.clientX - touchStartX);
    const diffY = Math.abs(e.clientY - touchStartY);
    if (diffX > 10 || diffY > 10) {
      clearTimeout(touchTimer);
      touchTimer = null;
    }
  }
}

function onPointerUp(e) {
  if (touchTimer) {
    clearTimeout(touchTimer);
    touchTimer = null;
  }
  if (isLongPress) return; // Если сработал долгий тап, обычный клик игнорируется

  if (e.pointerType === 'touch') {
    if (state.mobileMode === 'flag') {
      handleCellToggleFlag(r, c);
    } else {
      handleCellOpen(r, c);
    }
  }
}
```

---

## 7. Требования к UI-компонентам

1. **Счетчики (`#mine-counter`, `#timer-display`):**
   * Всегда отображают 3 цифры (формат строк: `'000'`, `'042'`, `'-05'`).
   * В случае отрицательного значения флагов: знак минус и 2 цифры (например, `-01`).
   * Таймер инкрементируется раз в 1000 мс, максимальное значение — `999`.
2. **Кнопка сброса (`#face-reset-btn`):**
   * Клик по кнопке полностью сбрасывает поле в текущей выбранной сложности без перезагрузки всей веб-страницы (`window.location.reload` запрещен, пересоздается только `state`).
3. **Селектор сложности (`.btn-diff`):**
   * При клике активной кнопке назначается класс `.active`.
   * Поле перерисовывается с обновленными CSS переменными `--cols` и `--rows`.

---

## 8. Чек-лист валидации (Verification Checklist)

При проверке сгенерированного кода убедиться в соблюдении следующих критериев:

- [ ] **Single File Integrity:** Файл запускается локально через `file://`, все стили и скрипты работают без интернета.
- [ ] **First Click Safety:** Первое нажатие на любую ячейку никогда не взрывает мину и открывает гарантированную область с нулем (`count === 0`).
- [ ] **Boundary Conditions:** Клетки по углам поля (например, `(0,0)`, `(0, cols-1)`) корректно вычисляют от 0 до 3 соседей без ошибок `index out of bounds`.
- [ ] **Zero Leaks:** При смене сложности или рестарте старый таймер `setInterval` гарантированно очищается (`clearInterval`).
- [ ] **Input Separation:** ПКМ не открывает ячейку при случайном отпускании ЛКМ; контекстное меню браузера не появляется ни при каких обстоятельствах на поле.
- [ ] **Viewport Protection:** На мобильном устройстве при быстром тапе по ячейкам не происходит зумирование экрана. Поле сложности «Профессионал» (30 колонок) можно прокручивать горизонтально пальцем без ложных открытий клеток.