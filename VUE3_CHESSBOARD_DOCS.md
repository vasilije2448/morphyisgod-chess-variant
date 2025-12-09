# vue3-chessboard – Consolidated Documentation

This document summarizes the main concepts and APIs described across the project’s documentation.

---

## 1. Overview

### 1.1 What is vue3-chessboard?

`vue3-chessboard` is a Vue 3 component library for rendering and interacting with a chessboard. It is written in TypeScript and built on top of:

- [chessground](https://github.com/lichess-org/chessground) – for board rendering and interaction
- [chess.js](https://github.com/jhlywa/chess.js) – for chess rules and move validation

The library exposes:

- A main component: `<TheChessboard />`
- A rich configuration object (`board-config`)
- A `BoardApi` instance (via the `board-created` event) for programmatic control
- Events and callbacks for reacting to game state changes
- Optional integration with UCI engines like Stockfish

---

## 2. Getting Started

### 2.1 Installation

Install the library:

```bash
# npm
npm install vue3-chessboard

# yarn
yarn add vue3-chessboard

# pnpm
pnpm install vue3-chessboard
```

### 2.2 Basic Usage

Minimal example in JavaScript:

```vue
<script setup>
import { TheChessboard } from 'vue3-chessboard';
import 'vue3-chessboard/style.css';
</script>

<template>
  <TheChessboard />
</template>
```

Minimal example in TypeScript:

```vue
<script setup lang="ts">
import { TheChessboard } from 'vue3-chessboard';
import 'vue3-chessboard/style.css';
</script>

<template>
  <TheChessboard />
</template>
```

The homepage (`src/index.md`) also demonstrates:

- Using `@board-created` to capture `boardAPI`
- Using `boardAPI` methods like `toggleOrientation`, `resetBoard`, `undoLastMove`, `toggleMoves`
- Listening to `@checkmate` and `@check` events to show alerts and reset the board

---

## 3. Core Component: `<TheChessboard />`

### 3.1 Main Props

From `src/props.md`:

- `board-config`  
  An object to configure the board (position, orientation, animation, events, etc.).

- `player-color`  
  The color that the local client is allowed to move. Opponent moves can be applied via `BoardApi.move`.  
  If omitted, turns switch locally.

- `reactive-config` (boolean)  
  When set, the `board-config` prop becomes reactive: changes to the config object propagate to the board in a non-destructive way.  
  Note: this is one-way; the board’s internal state (e.g. `fen`) does not update the prop.

### 3.2 Example: Passing `board-config`

```vue
<script setup>
import { TheChessboard } from 'vue3-chessboard';
import 'vue3-chessboard/style.css';

const boardConfig = {
  coordinates: false,
  autoCastle: false,
  orientation: 'black',
};
</script>

<template>
  <TheChessboard :board-config="boardConfig" />
</template>
```

### 3.3 Example: `player-color` for Multiplayer

```vue
<script setup>
import { TheChessboard } from 'vue3-chessboard';
import 'vue3-chessboard/style.css';

let boardApi;

// Client will only be able to play white pieces.
const playerColor = 'white';

// Receive move from socket/server/etc here.
function onRecieveMove(move) {
  boardApi?.move(move);
}
</script>

<template>
  <TheChessboard
    :player-color="playerColor"
    @board-created="(api) => (boardApi = api)"
  />
</template>
```

TypeScript version uses `MoveableColor` and `BoardApi` types.

### 3.4 Example: `reactive-config`

```vue
<script setup>
import { reactive } from 'vue';
import { TheChessboard } from 'vue3-chessboard';
import 'vue3-chessboard/style.css';

const boardConfig = reactive({
  coordinates: true,
  viewOnly: false,
  animation: { enabled: true },
  draggable: { enabled: true },
});
</script>

<template>
  <TheChessboard :board-config="boardConfig" reactive-config />
  <div>
    <button @click="boardConfig.coordinates = !boardConfig.coordinates">
      Toggle coordinates
    </button>
    <button @click="boardConfig.viewOnly = !boardConfig.viewOnly">
      Toggle view only
    </button>
    <button
      @click="boardConfig.animation.enabled = !boardConfig.animation.enabled"
    >
      Toggle animations
    </button>
    <button
      @click="boardConfig.draggable.enabled = !boardConfig.draggable.enabled"
    >
      Toggle draggable
    </button>
  </div>
</template>
```

---

## 4. Board Configuration (`board-config`)

### 4.1 Default Configuration (Summary)

From `src/props.md`, the default config includes:

- Position & orientation:
  - `fen`: starting position FEN
  - `orientation`: `'white' | 'black'`
  - `turnColor`: `'white' | 'black'`
  - `coordinates`: `true | false`
  - `autoCastle`: simplify castling moves
  - `viewOnly`: disable moves if `true`
  - `disableContextMenu`, `addPieceZIndex`, `blockTouchScroll`

- Highlighting:
  - `highlight.lastMove`
  - `highlight.check`

- Animation:
  - `animation.enabled`
  - `animation.duration`

- Movable:
  - `movable.free`
  - `movable.color`
  - `movable.showDests`
  - `movable.dests`
  - `movable.events`
  - `movable.rookCastle`

- Premovable / Predroppable:
  - `premovable.enabled`, `premovable.showDests`, `premovable.castle`, `premovable.events`
  - `predroppable.enabled`, `predroppable.events`

- Draggable:
  - `draggable.enabled`
  - `draggable.distance`
  - `draggable.autoDistance`
  - `draggable.showGhost`
  - `draggable.deleteOnDropOff`

- Selectable:
  - `selectable.enabled`

- Events:
  - `events` (see Callbacks section)

- Drawable:
  - `drawable.enabled`, `drawable.visible`
  - `drawable.defaultSnapToValidMove`
  - `drawable.eraseOnClick`
  - `drawable.shapes`, `drawable.autoShapes`
  - `drawable.brushes` (predefined colors like `green`, `red`, `blue`, `yellow`, `paleBlue`, etc.)

You can pass a partial config; it will be merged with defaults.

---

## 5. Board API

### 5.1 Accessing `BoardApi`

From `src/board-api.md` and `src/events/board-created.md`:

The `board-created` event emits a `BoardApi` instance:

```ts
defineEmits<{
  (e: 'board-created', boardApi: BoardApi): void;
}>();
```

Example usage:

```vue
<script setup>
import { onMounted } from 'vue';
import { TheChessboard } from 'vue3-chessboard';
import 'vue3-chessboard/style.css';

let boardAPI;

// access the boardAPI in the onMounted hook
onMounted(() => {
  console.log(boardAPI?.getBoardPosition());
});
</script>

<template>
  <TheChessboard @board-created="(api) => (boardAPI = api)" />
</template>
```

### 5.2 Key Methods (Summary)

From `src/board-api.md`, `BoardApi` exposes many methods, including:

- Game control:
  - `resetBoard()`
  - `undoLastMove()`
  - `setPosition(fen: string)`
  - `clearBoard()`
  - `move(move: string | { from: Key; to: Key; promotion?: Promotion }): boolean`

- Game state & info:
  - `getMaterialCount(): MaterialDifference`
  - `getCapturedPieces(): CapturedPieces`
  - `getTurnColor(): Color`
  - `getPossibleMoves(): Map<Key, Key[]> | undefined`
  - `getCurrentTurnNumber(): number`
  - `getCurrentPlyNumber(): number`
  - `getLastMove(): MoveEvent | undefined`
  - `getHistory(verbose?: boolean): MoveEvent[] | string[]`
  - `getFen(): string`
  - `getBoardPosition(): ({ type: PieceType; color: PieceColor; square: Square } | null)[][]`
  - `getPgn(): string`
  - `getPgnInfo(): { [key: string]: string | undefined }`
  - `getOpeningName(): Promise<string | null>`

- Game result helpers:
  - `getIsGameOver(): boolean`
  - `getIsCheckmate(): boolean`
  - `getIsCheck(): boolean`
  - `getIsStalemate(): boolean`
  - `getIsDraw(): boolean`
  - `getIsThreefoldRepetition(): boolean`
  - `getIsInsufficientMaterial(): boolean`

- Board inspection:
  - `getSquareColor(square: Square): SquareColor`
  - `getSquare(square: Square): Piece | null`

- Drawing & shapes:
  - `drawMoves()`
  - `hideMoves()`
  - `toggleMoves()`
  - `drawMove(orig: Square, dest: Square, brushColor: BrushColor)`
  - `setShapes(shapes: DrawShape[])`

- Config & history viewing:
  - `setConfig(config: BoardConfig, fillDefaults = false): void`
  - `viewHistory(ply: number): void`
  - `stopViewingHistory(): void`
  - `viewStart(): void`
  - `viewNext(): void`
  - `viewPrevious(): void`

The docs also show examples of:

- Using `BoardApi` to build a game history viewer
- Styling the board differently when viewing history (via a `viewingHistory` CSS class)

---

## 6. Callbacks (Config-Level Events)

From `src/callbacks.md`:

You can register callbacks in `board-config.events`. These are **not** Vue events; they are internal callbacks used by chessground.

Example:

```js
const boardConfig = {
  events: {
    change: () => {
      // called after the situation changes on the board
      console.log('Something changed!');
    },
    move: (from, to, capture) => {
      // fires after each move; from, to, capture are provided
      console.log(from, to, capture);
    },
    select(key) {
      // called when a square is selected
      console.log(key);
    },
  },
};
```

Important note:

> The `move` callback is executed **before** the internal board state is updated. Accessing `boardApi` here can lead to unexpected results. Use the `@move` Vue event instead when you need updated state.

Additional callbacks mentioned but not commonly needed:

- `insert` – fired when a new chessboard is mounted
- `dropNewPiece` – fired when a new piece is inserted (not currently used)

The callbacks page also includes a live example that tracks:

- A change counter
- The latest move (`from`, `to`, `capture`)
- The last selected square

---

## 7. Vue Events

There are two main event docs:

- `src/events.md` – overview and interactive examples
- `src/events/*.md` – per-event reference pages

### 7.1 Event List

From `src/events.md` and `src/events/board-events.md`:

- `board-created` – emitted when the board is created; includes `BoardApi`
- `checkmate` – emitted when a player is checkmated; includes the color
- `stalemate` – emitted when the game ends in stalemate
- `draw` – emitted when the game ends in a draw
- `check` – emitted when a player is in check; includes the color
- `promotion` – emitted when a pawn is promoted; includes promotion details
- `move` – emitted after each move; includes detailed move info

#### TypeScript `defineEmits` (events.md)

```ts
const emit = defineEmits<{
  (e: 'boardCreated', boardApi: BoardApi): void; // emits boardAPI
  (e: 'checkmate', isMated: PieceColor): void; // color of mated player
  (e: 'stalemate'): void;
  (e: 'draw'): void;
  (e: 'check', isInCheck: PieceColor): void; // color in check
  (e: 'promotion', promotion: PromotionEvent): void; // promotion info
  (e: 'move', move: MoveEvent): void; // move info
}>();
```

#### TypeScript `defineEmits` (board-events.md)

```ts
const emit = defineEmits<{
  (e: 'board-created', boardApi: BoardApi): void;
  (e: 'checkmate', isMated: PieceColor): void;
  (e: 'stalemate', isStalemate: boolean): void;
  (e: 'draw', isDraw: boolean): void;
  (e: 'check', isInCheck: PieceColor): void;
  (e: 'promotion', promotion: PromotionEvent): void;
  (e: 'move', move: MoveEvent): void;
}>();
```

> Note: There is a naming difference between `boardCreated` and `board-created` in the docs; the public event name used in examples is `@board-created`.

### 7.2 `move` Event

From `src/events/move.md`:

Definition:

```ts
defineEmits<{
  (e: 'move', move: MoveEvent): void;
}>();

type MoveEvent = {
  color: Color;
  from: Square;
  to: Square;
  piece: PieceSymbol;
  captured?: PieceSymbol;
  promotion?: PieceSymbol;
  flags: string;
  san: string;
  lan: string;
  before: string;
  after: string;
};
```

Usage:

```vue
<script setup>
import { TheChessboard } from 'vue3-chessboard';
import 'vue3-chessboard/style.css';

function handleMove(move) {
  console.log(move);
}
</script>

<template>
  <TheChessboard @move="handleMove" />
</template>
```

TypeScript version uses `MoveEvent` type.

### 7.3 `promotion` Event

From `src/events/promotion.md`:

Definition:

```ts
defineEmits<{
  (e: 'promotion', promotion: PromotionEvent): void;
}>();

export interface PromotionEvent {
  color: 'white' | 'black';
  sanMove: string;
  promotedTo: 'Q' | 'B' | 'R' | 'N';
}
```

Usage:

```vue
<script setup>
import { TheChessboard } from 'vue3-chessboard';
import 'vue3-chessboard/style.css';

function handlePromotion(promotion) {
  console.log(promotion);
}
</script>

<template>
  <TheChessboard @promotion="handlePromotion" />
</template>
```

### 7.4 `check` Event

From `src/events/check.md`:

Definition:

```ts
defineEmits<{
  (e: 'check', isInCheck: PieceColor): void;
}>();

export type PieceColor = 'white' | 'black';
```

Usage:

```vue
<script setup>
import { TheChessboard } from 'vue3-chessboard';
import 'vue3-chessboard/style.css';

function handleCheck(isInCheck) {
  alert(`${isInCheck} is in Check`);
}
</script>

<template>
  <TheChessboard @check="handleCheck" />
</template>
```

### 7.5 `checkmate` Event

From `src/events/checkmate.md`:

Definition:

```ts
defineEmits<{
  (e: 'checkmate', isMated: PieceColor): void;
}>();

export type PieceColor = 'white' | 'black';
```

Usage:

```vue
<script setup>
import { TheChessboard } from 'vue3-chessboard';
import 'vue3-chessboard/style.css';

function handleCheckmate(isMated) {
  alert(`${isMated} is mated`);
}
</script>

<template>
  <TheChessboard @checkmate="handleCheckmate" />
</template>
```

### 7.6 `stalemate` Event

From `src/events/stalemate.md`:

Definition:

```ts
defineEmits<{
  (e: 'stalemate'): void;
}>();
```

Usage:

```vue
<script setup>
import { TheChessboard } from 'vue3-chessboard';
import 'vue3-chessboard/style.css';

function handleStalemate() {
  alert('Stalemate');
}
</script>

<template>
  <TheChessboard @stalemate="handleStalemate" />
</template>
```

### 7.7 `draw` Event

From `src/events/draw.md`:

Definition:

```ts
defineEmits<{
  (e: 'draw'): void;
}>();
```

Usage:

```vue
<script setup>
import { TheChessboard } from 'vue3-chessboard';
import 'vue3-chessboard/style.css';

function handleDraw() {
  alert('Draw');
}
</script>

<template>
  <TheChessboard @draw="handleDraw" />
</template>
```

### 7.8 `board-created` Event

From `src/events/board-created.md`:

Definition:

```ts
defineEmits<{
  (e: 'board-created', boardApi: BoardApi): void;
}>();
```

Usage:

```vue
<script setup>
import { onMounted } from 'vue';
import { TheChessboard, type BoardApi } from 'vue3-chessboard';
import 'vue3-chessboard/style.css';

let boardApi: BoardApi | undefined;

onMounted(() => {
  console.log(boardApi?.getBoardPosition());
});
</script>

<template>
  <TheChessboard @board-created="(api) => (boardApi = api)" />
</template>
```

### 7.9 Interactive Event Examples

`src/events.md` includes interactive demos that:

- Programmatically play sequences of moves to trigger:
  - `check`
  - `checkmate`
  - `stalemate`
  - `draw`
  - `promotion`
- Use `BoardApi.move` to apply moves
- Use `@board-created` to capture multiple `boardAPI` instances
- Show emitted `MoveEvent` data in a `<pre>` block

---

## 8. Engine / Stockfish Integration

There are three main docs:

- `src/setup.md` – basic UCI engine setup (currently incomplete)
- `src/stockfish.md` – “Play vs Stockfish” example
- `src/stockfish-moves.md` – “Displaying Engine Moves” (hint arrows)

### 8.1 Installing Stockfish.js

From `src/setup.md`, `src/stockfish.md`, `src/stockfish-moves.md`:

```bash
# npm
npm i stockfish.js

# yarn
yarn add stockfish.js

# pnpm
pnpm install stockfish.js
```

Copy the engine files into your public folder:

```bash
cp node_modules/stockfish.js/stockfish.* public
```

PowerShell:

```powershell
Copy-Item -Path .\node_modules\stockfish.js\stockfish.* -Destination .\public\
```

### 8.2 Engine Basics

The docs use [stockfish.js](https://github.com/lichess-org/stockfish.js) by Lichess, which:

- Uses a WebAssembly build if supported
- Falls back to a JavaScript implementation otherwise
- Runs in a Web Worker to keep the UI responsive

There is also [stockfish.wasm](https://github.com/lichess-org/stockfish.wasm), but it requires HTTP headers not available on GitHub Pages.

### 8.3 Engine Class (Play vs Stockfish)

From `src/stockfish.md`:

The `Engine` class:

- Creates a `Worker` with either `stockfish.wasm.js` or `stockfish.js`
- Sends `uci` to initialize
- Sets options like `UCI_AnalyseMode` and `Analysis Contempt`
- On `bestmove`, if it’s black’s turn, calls `boardApi.move` with the engine’s move

Key points:

```ts
private handleEngineStdout(data: MessageEvent<unknown>) {
  const uciStringSplitted = (data.data as string).split(' ');

  if (uciStringSplitted[0] === 'uciok') {
    this.setOption('UCI_AnalyseMode', 'true');
    this.setOption('Analysis Contempt', 'Off');

    this.stockfish.postMessage('ucinewgame');
    this.stockfish.postMessage('isready');
    return;
  }

  if (uciStringSplitted[0] === 'readyok') {
    this.stockfish.postMessage('go movetime 1500');
    return;
  }

  if (uciStringSplitted[0] === 'bestmove' && uciStringSplitted[1]) {
    if (uciStringSplitted[1] !== this.bestMove) {
      this.bestMove = uciStringSplitted[1];
      if (this.boardApi.getTurnColor() === 'black') {
        this.boardApi.move({
          from: this.bestMove.slice(0, 2) as SquareKey,
          to: this.bestMove.slice(2, 4) as SquareKey,
        });
      }
    }
  }
}
```

`sendPosition(position: string)` sends:

```ts
this.stockfish.postMessage(`position startpos moves ${position}`);
this.stockfish.postMessage('go movetime 2000');
```

### 8.4 Vue Component (Play vs Stockfish)

From `src/stockfish.md`:

- `@board-created` instantiates `Engine` with `boardApi`
- `@move` collects history via `boardAPI.getHistory(true)` and sends LAN moves to the engine

```vue
<script setup lang="ts">
import { TheChessboard, type BoardApi } from 'vue3-chessboard';
import { Engine } from './Engine';

let boardAPI: BoardApi | undefined;
let engine: Engine | undefined;

function handleBoardCreated(boardApi: BoardApi) {
  boardAPI = boardApi;
  engine = new Engine(boardApi);
}

function handleMove() {
  const history = boardAPI?.getHistory(true);

  const moves = history?.map((move) => {
    if (typeof move === 'object') {
      return move.lan;
    } else {
      return move;
    }
  });

  if (moves) {
    engine?.sendPosition(moves.join(' '));
  }
}
</script>

<template>
  <TheChessboard
    @board-created="handleBoardCreated"
    @move="handleMove"
    :player-color="'white'"
  />
</template>
```

This lets you play as white against Stockfish as black.

### 8.5 Displaying Engine Moves (Hint Arrows)

From `src/stockfish-moves.md`:

A similar `Engine` class is used, but instead of making moves, it:

- Stores `bestMove`
- Uses `boardApi.drawMove(orig, dest, 'paleBlue')` to draw an arrow

On `bestmove`:

```ts
if (uciStringSplitted[0] === 'bestmove' && uciStringSplitted[1]) {
  if (uciStringSplitted[1] !== this.bestMove) {
    this.bestMove = uciStringSplitted[1];
    const orig = this.bestMove.slice(0, 2) as SquareKey;
    const dest = this.bestMove.slice(2, 4) as SquareKey;
    this.boardApi.drawMove(orig, dest, 'paleBlue');
  }
}
```

The Vue component:

- On `select`, if `engine.bestMove` exists, draws the arrow
- On `move`, hides moves (`boardAPI.hideMoves()`)
- On `@move`, recomputes the engine position from history and calls `engine.sendPosition`

This provides “best move” hints without the engine playing automatically.

---

## 9. Project Meta

### 9.1 README

`README.md` is minimal and points to this docs site:

```md
# vue3-chessboard-docs

Documentation for [vue3-chessboard](https://github.com/qwerty084/vue3-chessboard).
```

### 9.2 Contributing

From `src/contribute.md`:

- Clone the repo (HTTPS, SSH, or GitHub CLI)
- Install dependencies:

  ```sh
  npm install
  ```

- Start dev server:

  ```sh
  npm run dev
  ```

Before opening a PR, run:

- Format:

  ```sh
  npm run format
  ```

- Type check:

  ```sh
  npm run type-check
  ```

- Lint:

  ```sh
  npm run lint
  ```

- Test:

  ```sh
  npm run test
  ```

Then create a branch, commit, and push your changes.

### 9.3 Issues

From `src/issues.md`:

If you find a bug:

- Open an issue on GitHub:  
  <https://github.com/qwerty084/vue3-chessboard/issues>
- Or submit a pull request; contributions are welcome.

---

## 10. Summary

- Use `<TheChessboard />` with `board-config`, `player-color`, and `reactive-config` to control the board.
- Use `@board-created` to get `BoardApi` for programmatic control (moves, FEN/PGN, history, drawing, etc.).
- Use Vue events (`@move`, `@check`, `@checkmate`, `@stalemate`, `@draw`, `@promotion`) to react to game state changes.
- Use `board-config.events` callbacks for low-level chessground hooks, but prefer Vue events when you need updated state.
- Integrate Stockfish via a Web Worker to:
  - Play against the engine
  - Show best-move hints with arrows
- Follow the contributing guide and open issues/PRs on GitHub for bugs and improvements.

This `DOCS.md` is a high-level map of the documentation; for full code samples and live examples, refer to the individual `.md` files in `src/`.
