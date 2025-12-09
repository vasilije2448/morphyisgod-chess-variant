<script setup>
import { reactive, ref } from 'vue';
import { TheChessboard } from 'vue3-chessboard';
import 'vue3-chessboard/style.css';

const boardApi = ref(null);

// UI state for placement
const selectedPiece = ref('');
const selectedColor = ref('w');

// Simple list of pieces to place
const pieceOptions = [
  { value: 'k', label: 'King' },
  { value: 'q', label: 'Queen' },
  { value: 'r', label: 'Rook' },
  { value: 'b', label: 'Bishop' },
  { value: 'n', label: 'Knight' },
  { value: 'p', label: 'Pawn' },
];

// Utility to get a random element from an array
function randomChoice(arr) {
  return arr[Math.floor(Math.random() * arr.length)];
}

// Generate random starting squares for the kings
const files = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h'];
const whiteRanks = ['1', '2']; // first 2 ranks from White's perspective
const blackRanks = ['7', '8']; // last 2 ranks from White's perspective

const whiteKingSquare = randomChoice(files) + randomChoice(whiteRanks);
const blackKingSquare = randomChoice(files) + randomChoice(blackRanks);

// Internal position map used to build FEN.
// Keys are squares like "e4", values are { color: 'white'|'black', role: 'king'|'queen'|... }.
const position = reactive({
  [whiteKingSquare]: { color: 'white', role: 'king' },
  [blackKingSquare]: { color: 'black', role: 'king' },
});

// Board configuration for placement phase 1.1
const boardConfig = reactive({
  // Start from a valid chess position: at least one white and one black king.
  // The exact squares are randomized; we keep a simple "kings only" FEN as a placeholder.
  fen: '4k3/8/8/8/8/8/8/4K3 w - - 0 1',
  coordinates: true,
  viewOnly: false,
  // We are not using chess.js move rules yet; we just place pieces.
  movable: {
    free: false,
    color: 'white',
    showDests: false,
  },
  selectable: {
    enabled: true,
  },
  draggable: {
    enabled: false,
  },
  highlight: {
    lastMove: false,
    check: false,
  },
  // Use chessground's internal select callback instead of a Vue @select event
  events: {
    select: handleSquareSelect,
  },
});

function handleBoardCreated(api) {
  boardApi.value = api;
  // Ensure the board position matches our internal map (random kings)
  const fen = positionToFen(position);
  boardApi.value.setPosition(fen);
}

/**
 * Handle square selection on the board.
 * For phase 1.1 we simply place the currently selected piece on the clicked square.
 *
 * This is wired via boardConfig.events.select, which receives the square key
 * directly as a string like "e4".
 */
function handleSquareSelect(square) {
  if (!boardApi.value) return;
  if (!selectedPiece.value || !selectedColor.value) return;
  if (!square || typeof square !== 'string') return;

  // Map our piece codes to chessground/chess.js roles
  const roleMap = {
    k: 'king',
    q: 'queen',
    r: 'rook',
    b: 'bishop',
    n: 'knight',
    p: 'pawn',
  };

  const role = roleMap[selectedPiece.value];
  if (!role) return;

  const color = selectedColor.value === 'w' ? 'white' : 'black';

  // Place or overwrite the piece on that square in our position map
  position[square] = { color, role };

  // Build a FEN string from the current position and push it to the board
  const fen = positionToFen(position);
  boardApi.value.setPosition(fen);
}

/**
 * Convert our simple square->piece map into a full FEN string.
 * We always ensure there is at least one white king and one black king
 * by defaulting them to e1/e8 if missing.
 */
function positionToFen(pos) {
  // Start from an empty 8x8 board
  const board = Array.from({ length: 8 }, () => Array(8).fill(null));

  // Helper to convert square like "e4" to [rankIndex, fileIndex]
  const squareToIndices = (sq) => {
    if (!/^[a-h][1-8]$/.test(sq)) return null;
    const file = sq.charCodeAt(0) - 'a'.charCodeAt(0); // 0..7
    const rankFromBottom = parseInt(sq[1], 10) - 1; // 0..7
    const rank = 7 - rankFromBottom; // 0 = 8th rank
    return [rank, file];
  };

  // Map role/color to FEN symbol
  const symbolFor = (piece) => {
    if (!piece) return null;
    const base = {
      king: 'k',
      queen: 'q',
      rook: 'r',
      bishop: 'b',
      knight: 'n',
      pawn: 'p',
    }[piece.role];
    if (!base) return null;
    return piece.color === 'white' ? base.toUpperCase() : base;
  };

  // Place all pieces from the position map
  Object.entries(pos).forEach(([sq, piece]) => {
    const idx = squareToIndices(sq);
    if (!idx) return;
    const [rank, file] = idx;
    const sym = symbolFor(piece);
    if (!sym) return;
    board[rank][file] = sym;
  });

  // Ensure at least one white king and one black king exist.
  // If missing, place them on e1/e8 if those squares are empty.
  const hasWhiteKing = board.some((rank) => rank.includes('K'));
  const hasBlackKing = board.some((rank) => rank.includes('k'));

  if (!hasWhiteKing) {
    const e1 = squareToIndices('e1');
    if (e1) {
      const [r, f] = e1;
      if (!board[r][f]) board[r][f] = 'K';
    }
  }

  if (!hasBlackKing) {
    const e8 = squareToIndices('e8');
    if (e8) {
      const [r, f] = e8;
      if (!board[r][f]) board[r][f] = 'k';
    }
  }

  // Convert board array to FEN piece placement
  const ranks = board.map((rankArr) => {
    let fenRank = '';
    let empty = 0;
    for (let i = 0; i < 8; i += 1) {
      const cell = rankArr[i];
      if (!cell) {
        empty += 1;
      } else {
        if (empty > 0) {
          fenRank += String(empty);
          empty = 0;
        }
        fenRank += cell;
      }
    }
    if (empty > 0) fenRank += String(empty);
    return fenRank || '8';
  });

  // Full FEN: piece placement + simple defaults
  return `${ranks.join('/')}` + ' w - - 0 1';
}
</script>

<template>
  <div class="app">
    <header class="app-header">
      <h1>Custom Chess Variant – Placement Phase</h1>
      <p class="app-subtitle">
        Step 1.1: Click a piece and color above, then click a square on the
        board to place it. No rules are enforced yet.
      </p>
    </header>

    <main class="app-main">
      <section class="controls">
        <div class="controls-row">
          <label class="control-label" for="piece-select">Piece:</label>
          <select
            id="piece-select"
            v-model="selectedPiece"
            class="control-select"
          >
            <option disabled value="">-- choose a piece --</option>
            <option
              v-for="option in pieceOptions"
              :key="option.value"
              :value="option.value"
            >
              {{ option.label }}
            </option>
          </select>
        </div>

        <div class="controls-row">
          <label class="control-label" for="color-select">Color:</label>
          <select
            id="color-select"
            v-model="selectedColor"
            class="control-select"
          >
            <option value="w">White</option>
            <option value="b">Black</option>
          </select>
        </div>

        <p class="hint">
          1. Select a piece and color. 2. Click any square on the board to place
          that piece. 3. Clicking an occupied square overwrites the piece.
        </p>
      </section>

      <section class="board-wrapper">
        <TheChessboard
          :board-config="boardConfig"
          reactive-config
          @board-created="handleBoardCreated"
        />
      </section>
    </main>
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  padding: 1rem;
  box-sizing: border-box;
  gap: 1rem;
}

.app-header {
  text-align: center;
}

.app-header h1 {
  margin: 0;
  font-size: 1.5rem;
}

.app-subtitle {
  margin: 0.5rem 0 0;
  font-size: 0.95rem;
  color: #555;
}

.app-main {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
}

/* Controls above the board for mobile friendliness */
.controls {
  width: 100%;
  max-width: 480px;
  padding: 0.75rem 1rem;
  border-radius: 0.5rem;
  background: #f7f7f7;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.08);
  box-sizing: border-box;
}

.controls-row {
  display: flex;
  align-items: center;
  margin-bottom: 0.5rem;
  gap: 0.5rem;
}

.control-label {
  flex: 0 0 110px;
  font-size: 0.9rem;
}

.control-select {
  flex: 1;
  padding: 0.25rem 0.5rem;
  font-size: 0.9rem;
}

.hint {
  margin: 0.5rem 0 0;
  font-size: 0.8rem;
  color: #666;
}

.board-wrapper {
  width: 100%;
  max-width: 480px;
}

/* Make the chessboard responsive */
.board-wrapper :deep(.cg-wrap) {
  width: 100%;
  max-width: 100%;
  aspect-ratio: 1 / 1;
}
</style>
