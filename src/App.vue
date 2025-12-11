<script setup>
import { reactive, ref, computed, watch } from 'vue';
import { TheChessboard } from 'vue3-chessboard';
import 'vue3-chessboard/style.css';
import { Chess } from 'chess.js';

const boardApi = ref(null);

// chess.js instance – used for "no check" validation during placement
// and later for the standard chess phase (only for validation, not move rules).
const chess = ref(null);

// UI state for placement
const selectedPiece = ref('');
const selectedColor = ref('w');
const errorMessage = ref('');
const infoMessage = ref('');

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

// Files and ranks helpers
const files = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h'];
const whiteRanks = ['1', '2']; // first 2 ranks from White's perspective
const blackRanks = ['7', '8']; // last 2 ranks from White's perspective

// Random king squares (recomputed on reset)
const whiteKingSquare = ref('');
const blackKingSquare = ref('');

// Internal position map used to build FEN.
// Keys are squares like "e4", values are { color: 'white'|'black', role: 'king'|'queen'|... }.
const position = reactive({});

// Track how many placement moves have been made (excluding the initial kings).
// We want the repeating pattern:
// 1: white pawn
// 2: black pawn
// 3: white piece
// 4: black piece
// 5: white pawn
// 6: black pawn
// 7: white piece
// 8: black piece
// ...
const placementMoveCount = ref(1);

// Are we still in the placement phase?
const inPlacementPhase = ref(true);

// Track whose turn it is to place: 'white' or 'black'
const currentPlacementColor = computed(() => {
  const m = placementMoveCount.value;
  // Odd moves: white, even moves: black
  return m % 2 === 1 ? 'white' : 'black';
});

// Whether the current move must be a pawn (true) or a piece (false)
const isCurrentMovePawnMove = computed(() => {
  const m = placementMoveCount.value;
  // Pattern: pawn, pawn, piece, piece, pawn, pawn, piece, piece, ...
  const mod = (m - 1) % 4; // 0,1,2,3 repeating
  return mod === 0 || mod === 1; // first two in each block of 4 are pawns
});

// Convenience: computed description of what is allowed this move
const currentMoveRequirement = computed(() => {
  if (!inPlacementPhase.value) {
    return 'Placement phase finished. Standard chess game has started.';
  }
  const colorText = currentPlacementColor.value === 'white' ? 'White' : 'Black';
  if (isCurrentMovePawnMove.value) {
    return `${colorText} must place a pawn.`;
  }
  return `${colorText} must place a piece (non-pawn).`;
});

// Board configuration for placement phase
// IMPORTANT: we must not use a completely empty-board FEN because chess.js
// requires at least one white and one black king. We start with a minimal
// valid FEN and immediately overwrite it in resetGame() with our random-king position.
const boardConfig = reactive({
  fen: '4k3/8/8/8/8/8/8/4K3 w - - 0 1',
  coordinates: true,
  viewOnly: false,
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
    lastMove: true,
    check: true,
  },
  events: {
    select: handleSquareSelect,
  },
});

// Board configuration for standard chess phase
const standardBoardConfig = reactive({
  fen: 'start', // will be replaced with placement FEN when starting the game
  coordinates: true,
  viewOnly: false,
  movable: {
    free: false,
    color: 'white',
    showDests: true,
  },
  selectable: {
    enabled: true,
  },
  draggable: {
    enabled: true,
  },
  highlight: {
    lastMove: true,
    check: true,
  },
  events: {
    // no custom events needed; standard rules are handled by TheChessboard
  },
});

function handleBoardCreated(api) {
  boardApi.value = api;
  resetGame();
}

/**
 * Helpers for rule enforcement
 */

// Convert square like "e4" to file ('a'..'h') and rank (1..8)
function parseSquare(sq) {
  if (!/^[a-h][1-8]$/.test(sq)) return null;
  return {
    file: sq[0],
    rank: parseInt(sq[1], 10),
  };
}

// Determine light/dark color of a square
function squareColor(sq) {
  const parsed = parseSquare(sq);
  if (!parsed) return null;
  const fileIndex = parsed.file.charCodeAt(0) - 'a'.charCodeAt(0); // 0..7
  const rankIndex = parsed.rank - 1; // 0..7
  // a1 is dark in standard chess: (0 + 0) % 2 === 0 -> dark
  const sum = fileIndex + rankIndex;
  return sum % 2 === 0 ? 'dark' : 'light';
}

// Get all squares in the 3x3 "safe zone" around a king square
function kingSafeZoneSquares(kingSq) {
  const parsed = parseSquare(kingSq);
  if (!parsed) return [];
  const result = [];
  const fileIndex = parsed.file.charCodeAt(0) - 'a'.charCodeAt(0);
  const rank = parsed.rank;
  for (let df = -1; df <= 1; df += 1) {
    for (let dr = -1; dr <= 1; dr += 1) {
      const f = fileIndex + df;
      const r = rank + dr;
      if (f < 0 || f > 7 || r < 1 || r > 8) continue;
      const fileChar = String.fromCharCode('a'.charCodeAt(0) + f);
      result.push(`${fileChar}${r}`);
    }
  }
  return result;
}

// Safe zones (recomputed on reset)
const whiteKingSafeZone = ref([]);
const blackKingSafeZone = ref([]);

// Check if a square is in the opponent king's safe zone
function isSafeSquareForColor(square, color) {
  if (color === 'white') {
    // White cannot place on black king's safe squares
    return blackKingSafeZone.value.includes(square);
  }
  if (color === 'black') {
    // Black cannot place on white king's safe squares
    return whiteKingSafeZone.value.includes(square);
  }
  return false;
}

// Count bishops of a given color on light/dark squares
function countBishopsBySquareColor(color, squareColorTarget) {
  return Object.entries(position).filter(([sq, p]) => {
    if (p.color !== color || p.role !== 'bishop') return false;
    return squareColor(sq) === squareColorTarget;
  }).length;
}

// Count pieces of a given color and role
function countPieces(color, role) {
  return Object.values(position).filter(
    (p) => p.color === color && p.role === role,
  ).length;
}

// Count pawns of a given color
function countPawns(color) {
  return countPieces(color, 'pawn');
}

// Maximum piece counts per color (standard chess)
const maxPiecesPerColor = {
  pawn: 8,
  rook: 2,
  bishop: 2,
  knight: 2,
  queen: 1,
  king: 1, // kings are pre-placed and we don't allow placing more
};

// Check if the current move type (pawn vs piece) is legal for this placement
function isMoveTypeLegalForTurn(pieceCode) {
  const isPawn = pieceCode === 'p';
  const isPiece = !isPawn;

  const mustBePawn = isCurrentMovePawnMove.value;

  if (mustBePawn && !isPawn) return false;
  if (!mustBePawn && !isPiece) return false;
  return true;
}

/**
 * Compute all legal pawn squares (by rank band only) for a color.
 * This ignores occupancy and per-file limits.
 */
function getLegalPawnSquaresByRankBand(color) {
  const squares = [];
  for (const f of files) {
    for (let r = 1; r <= 8; r += 1) {
      const sq = `${f}${r}`;
      const parsed = parseSquare(sq);
      if (!parsed) continue;
      if (color === 'white') {
        if (parsed.rank < 2 || parsed.rank > 5) continue;
      } else {
        if (parsed.rank < 4 || parsed.rank > 7) continue;
      }
      squares.push(sq);
    }
  }
  return squares;
}

/**
 * Compute all currently empty legal pawn squares (respecting rank band only).
 */
function getEmptyLegalPawnSquares(color) {
  return getLegalPawnSquaresByRankBand(color).filter((sq) => !position[sq]);
}

/**
 * Compute all squares where a pawn of this color may actually be placed
 * given the current position, including:
 * - rank band
 * - empty square
 * - per-file "one pawn per file unless no pawnless files with legal squares" rule.
 */
function getTrulyLegalPawnSquares(color) {
  const result = [];

  // Precompute which files already have a pawn of this color
  const fileHasPawnMap = {};
  for (const f of files) {
    fileHasPawnMap[f] = Object.entries(position).some(([sq, p]) => {
      if (p.color !== color || p.role !== 'pawn') return false;
      const parsedSq = parseSquare(sq);
      if (!parsedSq) return false;
      return parsedSq.file === f;
    });
  }

  // Determine if there exists any pawnless file with at least one empty legal square
  let hasPawnlessFileWithLegalSquare = false;
  for (const f of files) {
    if (fileHasPawnMap[f]) continue; // not pawnless

    let hasEmptyLegalOnF = false;
    for (let r = 1; r <= 8; r += 1) {
      const sq = `${f}${r}`;
      const sqParsed = parseSquare(sq);
      if (!sqParsed) continue;

      if (color === 'white') {
        if (sqParsed.rank < 2 || sqParsed.rank > 5) continue;
      } else {
        if (sqParsed.rank < 4 || sqParsed.rank > 7) continue;
      }

      if (!position[sq]) {
        hasEmptyLegalOnF = true;
        break;
      }
    }

    if (hasEmptyLegalOnF) {
      hasPawnlessFileWithLegalSquare = true;
      break;
    }
  }

  // Now build the list of truly legal squares
  for (const f of files) {
    for (let r = 1; r <= 8; r += 1) {
      const sq = `${f}${r}`;
      const parsed = parseSquare(sq);
      if (!parsed) continue;

      // Must be empty
      if (position[sq]) continue;

      // Must be in rank band
      if (color === 'white') {
        if (parsed.rank < 2 || parsed.rank > 5) continue;
      } else {
        if (parsed.rank < 4 || parsed.rank > 7) continue;
      }

      const thisFileHasPawn = fileHasPawnMap[f];

      // If this file has no pawn yet, this square is legal
      if (!thisFileHasPawn) {
        result.push(sq);
        continue;
      }

      // This file already has a pawn.
      // We may only place here if there are no pawnless files with legal squares.
      if (!hasPawnlessFileWithLegalSquare) {
        result.push(sq);
      }
    }
  }

  return result;
}

// Check if pawn placement is legal according to rank/file and per-file limits
function isPawnPlacementLegal(square, color) {
  const parsed = parseSquare(square);
  if (!parsed) {
    return {
      ok: false,
      reason: 'Invalid square.',
    };
  }

  const { file, rank } = parsed;

  // Rank constraints: pawns must be 3 or more squares away from queening rank
  // White queening rank is 8 -> allowed ranks 2,3,4,5
  // Black queening rank is 1 -> allowed ranks 4,5,6,7
  if (color === 'white') {
    if (rank < 2 || rank > 5) {
      const legalSquares = getTrulyLegalPawnSquares(color).join(', ');
      return {
        ok: false,
        reason: `White pawns may only be placed on ranks 2–5. Legal white pawn squares given the current position: ${legalSquares}`,
      };
    }
  } else if (color === 'black') {
    if (rank < 4 || rank > 7) {
      const legalSquares = getTrulyLegalPawnSquares(color).join(', ');
      return {
        ok: false,
        reason: `Black pawns may only be placed on ranks 4–7. Legal black pawn squares given the current position: ${legalSquares}`,
      };
    }
  }

  // Global limit: at most 8 pawns per color
  const totalPawns = countPawns(color);
  if (totalPawns >= maxPiecesPerColor.pawn) {
    return {
      ok: false,
      reason: `You may not place more than ${maxPiecesPerColor.pawn} pawns for ${color}.`,
    };
  }

  // Determine if this file already has a pawn of this color
  const fileHasPawn = Object.entries(position).some(([sq, p]) => {
    if (p.color !== color || p.role !== 'pawn') return false;
    const parsedSq = parseSquare(sq);
    if (!parsedSq) return false;
    return parsedSq.file === file;
  });

  // If this file has no pawn yet, this square is allowed (rank band already checked)
  if (!fileHasPawn) {
    return { ok: true };
  }

  // This file already has a pawn of this color.
  // We must now check the "no pawnless files with legal squares" condition.
  //
  // A "pawnless file with a legal square" means:
  // - There exists some file F where this color has no pawn yet, AND
  // - There exists at least one empty square on F in the allowed rank band.
  let hasPawnlessFileWithLegalSquare = false;

  for (const f of files) {
    // Does this file already have a pawn of this color?
    const fileHasPawnOnF = Object.entries(position).some(([sq, p]) => {
      if (p.color !== color || p.role !== 'pawn') return false;
      const parsedSq = parseSquare(sq);
      if (!parsedSq) return false;
      return parsedSq.file === f;
    });

    if (fileHasPawnOnF) {
      continue; // not pawnless
    }

    // Check if there is at least one empty legal square on this pawnless file
    let hasEmptyLegalOnF = false;
    for (let r = 1; r <= 8; r += 1) {
      const sq = `${f}${r}`;
      const sqParsed = parseSquare(sq);
      if (!sqParsed) continue;

      if (color === 'white') {
        if (sqParsed.rank < 2 || sqParsed.rank > 5) continue;
      } else {
        if (sqParsed.rank < 4 || sqParsed.rank > 7) continue;
      }

      if (!position[sq]) {
        hasEmptyLegalOnF = true;
        break;
      }
    }

    if (hasEmptyLegalOnF) {
      hasPawnlessFileWithLegalSquare = true;
      break;
    }
  }

  if (!hasPawnlessFileWithLegalSquare) {
    // No pawnless file has any legal empty square -> we are allowed to stack
    return { ok: true };
  }

  // Otherwise, we must reject and show the currently available pawn squares
  const emptyLegalSquares = getTrulyLegalPawnSquares(color);
  const legalSquaresList = emptyLegalSquares.join(', ');

  return {
    ok: false,
    reason:
      `You may only have one pawn per file unless there are no pawnless files with legal pawn squares for this color. ` +
      `Currently available pawn squares for ${color}: ${legalSquaresList}`,
  };
}

// Check if bishop placement is legal (one light-squared and one dark-squared bishop per color)
function isBishopPlacementLegal(square, color) {
  const sqColor = squareColor(square);
  if (!sqColor) {
    return { ok: false, reason: 'Invalid square.' };
  }

  const existingCount = countBishopsBySquareColor(color, sqColor);
  if (existingCount >= 1) {
    return {
      ok: false,
      reason: `You may only have one ${sqColor}-squared bishop for ${color}.`,
    };
  }

  return { ok: true };
}

/**
 * Enforce per-piece maximum counts (rooks, knights, bishops, queens, pawns).
 */
function isPieceCountLegal(role, color) {
  const max = maxPiecesPerColor[role];
  if (max == null) return true;
  const current = countPieces(color, role);
  if (current >= max) {
    return {
      ok: false,
      reason: `You may not place more than ${max} ${role}${max > 1 ? 's' : ''} for ${color}.`,
    };
  }
  return { ok: true };
}

/**
 * Compute the automatically recommended piece for the current move,
 * following the priority:
 *   knight -> bishop -> rook -> queen
 * Only applies on "piece" moves (non-pawn). For pawn moves we always use pawn.
 */
function getRecommendedPieceForCurrentMove() {
  if (!inPlacementPhase.value) return '';

  const mustBePawn = isCurrentMovePawnMove.value;
  const color = currentPlacementColor.value;

  if (mustBePawn) {
    // Always pawn on pawn moves
    return 'p';
  }

  // Piece move: choose in order N, B, R, Q if there is remaining capacity
  const priorities = ['n', 'b', 'r', 'q'];
  const roleMap = {
    n: 'knight',
    b: 'bishop',
    r: 'rook',
    q: 'queen',
  };

  for (const code of priorities) {
    const role = roleMap[code];
    const check = isPieceCountLegal(role, color);
    if (check === true || check.ok) {
      return code;
    }
  }

  // If nothing is available (should be rare), fall back to pawn so UI has something
  return 'p';
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

/**
 * Check if placing a piece on a square would put the opponent in check.
 * Uses chess.js for move generation and check detection.
 *
 * Note: chess.js v1.4.0 uses inCheck(), not in_check().
 */
function wouldPlacePutOpponentInCheck(square, color, role) {
  const fen = positionToFen(position);
  const tempChess = new Chess(fen);

  // Determine piece letter for FEN
  const base = {
    king: 'k',
    queen: 'q',
    rook: 'r',
    bishop: 'b',
    knight: 'n',
    pawn: 'p',
  }[role];
  if (!base) return false;

  const sqParsed = parseSquare(square);
  if (!sqParsed) return false;

  // Build a new FEN by adding/overwriting this piece on the target square.
  // We'll do this by using chess.js's board() representation.
  const boardArr = tempChess.board();
  const fileIndex = square.charCodeAt(0) - 'a'.charCodeAt(0);
  const rankFromBottom = sqParsed.rank - 1;
  const rankIndex = 7 - rankFromBottom;

  boardArr[rankIndex][fileIndex] = {
    type: base,
    color: color === 'white' ? 'w' : 'b',
  };

  // Rebuild FEN piece placement from modified board
  let fenPlacement = '';
  for (let r = 0; r < 8; r += 1) {
    let empty = 0;
    for (let f = 0; f < 8; f += 1) {
      const cell = boardArr[r][f];
      if (!cell) {
        empty += 1;
      } else {
        if (empty > 0) {
          fenPlacement += String(empty);
          empty = 0;
        }
        const sym = cell.type;
        fenPlacement += cell.color === 'w' ? sym.toUpperCase() : sym;
      }
    }
    if (empty > 0) fenPlacement += String(empty);
    if (r < 7) fenPlacement += '/';
  }

  // Side to move should be the opponent; we want to know if they are in check.
  const sideToMove = color === 'white' ? 'b' : 'w';
  const newFen = `${fenPlacement} ${sideToMove} - - 0 1`;
  const finalChess = new Chess(newFen);

  // inCheck() answers "is side to move in check?"
  if (typeof finalChess.inCheck === 'function') {
    return finalChess.inCheck();
  }

  // Fallback: if API is different, do not block placement
  return false;
}

/**
 * Handle square selection on the board during placement.
 */
function handleSquareSelect(square) {
  if (!inPlacementPhase.value) {
    // In the standard chess phase, moves are handled by the standard board.
    return;
  }

  errorMessage.value = '';
  infoMessage.value = '';

  if (!boardApi.value) return;
  if (!selectedPiece.value) {
    errorMessage.value = 'Please select a piece first.';
    return;
  }
  if (!square || typeof square !== 'string') return;

  // Enforce alternating colors: ignore selectedColor and derive from move number
  const expectedColor = currentPlacementColor.value; // 'white' or 'black'
  const expectedColorCode = expectedColor === 'white' ? 'w' : 'b';

  // Keep the dropdown in sync with the actual color to move
  selectedColor.value = expectedColorCode;

  // Do not allow placing on an occupied square
  if (position[square]) {
    errorMessage.value = `Square ${square} is already occupied. Please choose an empty square.`;
    return;
  }

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

  const color = expectedColor;

  // Enforce move type (pawn vs piece) alternation
  if (!isMoveTypeLegalForTurn(selectedPiece.value)) {
    errorMessage.value = isCurrentMovePawnMove.value
      ? 'This move must be a pawn placement.'
      : 'This move must be a piece (non-pawn) placement.';
    return;
  }

  // Enforce per-piece maximum counts (except kings, which are pre-placed)
  if (role !== 'king') {
    const pieceCountCheck = isPieceCountLegal(role, color);
    if (!pieceCountCheck.ok) {
      errorMessage.value = pieceCountCheck.reason;
      return;
    }
  }

  // Prevent placing on opponent king's safe squares
  if (isSafeSquareForColor(square, color)) {
    errorMessage.value =
      'You may not place a piece on the 3×3 safe zone around the opponent king.';
    return;
  }

  // Enforce pawn-specific rules
  if (role === 'pawn') {
    const pawnCheck = isPawnPlacementLegal(square, color);
    if (!pawnCheck.ok) {
      errorMessage.value = pawnCheck.reason;
      return;
    }
  }

  // Enforce bishop-specific rules
  if (role === 'bishop') {
    const bishopCheck = isBishopPlacementLegal(square, color);
    if (!bishopCheck.ok) {
      errorMessage.value = bishopCheck.reason;
      return;
    }
  }

  // Enforce "You may NOT place your opponent in check."
  if (wouldPlacePutOpponentInCheck(square, color, role)) {
    errorMessage.value = 'You may not place a piece that puts the opponent in check.';
    return;
  }

  // Place the piece on that square in our position map (no overwriting allowed now)
  position[square] = { color, role };

  // Build a FEN string from the current position and push it to the board
  const fen = positionToFen(position);
  boardApi.value.setPosition(fen);

  // Advance placement move counter
  placementMoveCount.value += 1;
}

/**
 * Reset the entire game back to the start of the placement phase.
 * This must create an "empty board except for the two kings randomly placed
 * on the first two / last two ranks", per GAME_SPECIFICATION.md.
 */
function resetGame() {
  // Clear state
  Object.keys(position).forEach((key) => {
    delete position[key];
  });

  // Randomize kings on the correct rank ranges
  whiteKingSquare.value = randomChoice(files) + randomChoice(whiteRanks);
  blackKingSquare.value = randomChoice(files) + randomChoice(blackRanks);

  // Place only the two kings on the otherwise empty board
  position[whiteKingSquare.value] = { color: 'white', role: 'king' };
  position[blackKingSquare.value] = { color: 'black', role: 'king' };

  whiteKingSafeZone.value = kingSafeZoneSquares(whiteKingSquare.value);
  blackKingSafeZone.value = kingSafeZoneSquares(blackKingSquare.value);

  placementMoveCount.value = 1;
  inPlacementPhase.value = true;
  selectedPiece.value = '';
  // Keep this in sync with currentPlacementColor (white on move 1)
  selectedColor.value = 'w';
  errorMessage.value = '';
  infoMessage.value =
    'Placement phase: White pawn, Black pawn, White piece, Black piece, then repeat.';

  const fen = positionToFen(position);
  boardConfig.fen = fen;
  boardConfig.viewOnly = false;
  boardConfig.movable.free = false;
  boardConfig.events.select = handleSquareSelect;

  if (boardApi.value) {
    boardApi.value.setPosition(fen);
  }

  // Reset chess.js to this starting placement position
  chess.value = new Chess(fen);

  // Auto-select recommended piece for the first move
  selectedPiece.value = getRecommendedPieceForCurrentMove();
}

/**
 * Start the standard chess game after placement is done.
 * This locks further placement and hands control to vue3-chessboard's
 * standard chess rules on a separate board instance.
 */
function startChessGame() {
  if (!inPlacementPhase.value) {
    return;
  }

  const fen = positionToFen(position);

  // Initialize chess.js with this FEN (kept for potential future use)
  chess.value = new Chess(fen);

  // Configure the standard chess board to start from the placement FEN
  standardBoardConfig.fen = fen;
  standardBoardConfig.viewOnly = false;

  inPlacementPhase.value = false;
  errorMessage.value = '';
  infoMessage.value = 'Standard chess game started.';

  // The placement board remains mounted but becomes invisible via v-if.
  // The standard board becomes visible and handles all moves itself.
}

/**
 * Automatically recommend a piece whenever the move number changes
 * (i.e., after each successful placement) or when we re-enter placement phase.
 * Also keep the displayed color in sync with the side to place.
 * This does NOT prevent the user from manually changing the piece selection.
 */
watch(
  () => placementMoveCount.value,
  () => {
    if (!inPlacementPhase.value) return;
    selectedPiece.value = getRecommendedPieceForCurrentMove();
    // Keep color dropdown in sync with the actual side to place
    selectedColor.value = currentPlacementColor.value === 'white' ? 'w' : 'b';
  },
);
</script>

<template>
  <div class="app">
    <header class="app-header">
      <h1>Custom Chess Variant – Placement &amp; Play</h1>
      <p class="app-subtitle">
        Step 1.2: Place pieces according to the variant rules. Kings are
        pre-placed; safe zones, pawn limits, bishop limits, per-piece limits,
        alternating colors, and "no check" placement are enforced. Then play a
        standard chess game from the resulting position.
      </p>
    </header>

    <main class="app-main">
      <section class="controls">
        <div class="controls-row controls-row-buttons">
          <button type="button" class="primary-button" @click="resetGame">
            Reset game
          </button>
          <button
            type="button"
            class="secondary-button"
            @click="startChessGame"
          >
            Start chess game
          </button>
        </div>

        <div class="controls-row" v-if="inPlacementPhase">
          <label class="control-label" for="piece-select">Piece:</label>
          <select
            id="piece-select"
            v-model="selectedPiece"
            class="control-select"
            :disabled="!inPlacementPhase"
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

        <!-- Color selector shows whose turn it is; it is driven by the logic, not user choice -->
        <div class="controls-row" v-if="inPlacementPhase">
          <label class="control-label" for="color-select">Side to place:</label>
          <select
            id="color-select"
            v-model="selectedColor"
            class="control-select"
            disabled
          >
            <option value="w">White</option>
            <option value="b">Black</option>
          </select>
        </div>

        <p class="hint">
          {{ currentMoveRequirement }}
        </p>
        <p class="hint" v-if="inPlacementPhase">
          A recommended piece is auto-selected for you (Knight → Bishop → Rook
          → Queen on piece moves). You can still change the selection manually
          before clicking a square.
        </p>
        <p class="hint" v-if="inPlacementPhase">
          1. Confirm or change the selected piece. 2. Click a legal square on
          the board to place that piece. 3. Illegal placements are rejected
          with a message below.
        </p>
        <p class="hint" v-else>
          Standard chess phase: play normally from the custom starting
          position. All standard chess rules are enforced by the board.
        </p>

        <!-- Reserve vertical space for messages so the board doesn't move -->
        <div class="messages">
          <p v-if="infoMessage" class="info">
            {{ infoMessage }}
          </p>
          <p v-if="errorMessage" class="error">
            {{ errorMessage }}
          </p>
        </div>
      </section>

      <section class="board-wrapper">
        <!-- Placement-phase board -->
        <TheChessboard
          v-if="inPlacementPhase"
          :board-config="boardConfig"
          reactive-config
          @board-created="handleBoardCreated"
        />

        <!-- Standard chess-phase board -->
        <TheChessboard
          v-else
          :board-config="standardBoardConfig"
          reactive-config
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

.controls-row-buttons {
  justify-content: space-between;
}

.control-label {
  flex: 0 0 140px;
  font-size: 0.9rem;
}

.control-select {
  flex: 1;
  padding: 0.25rem 0.5rem;
  font-size: 0.9rem;
}

.primary-button,
.secondary-button {
  flex: 1;
  padding: 0.4rem 0.6rem;
  font-size: 0.9rem;
  border-radius: 0.4rem;
  border: none;
  cursor: pointer;
}

.primary-button {
  background-color: #1976d2;
  color: white;
}

.primary-button:hover {
  background-color: #145ca3;
}

.secondary-button {
  background-color: #e0e0e0;
  color: #333;
}

.secondary-button:hover {
  background-color: #cfcfcf;
}

.hint {
  margin: 0.25rem 0 0;
  font-size: 0.8rem;
  color: #666;
}

/* Fixed-height container for messages so layout doesn't jump */
.messages {
  min-height: 5.4rem; /* enough for a few short lines */
  margin-top: 0.25rem;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
}

.info {
  margin: 0;
  font-size: 0.8rem;
  color: #1565c0;
}

.error {
  margin: 0.15rem 0 0;
  font-size: 0.8rem;
  color: #b00020;
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
