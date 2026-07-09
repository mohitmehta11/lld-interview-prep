# Tic-Tac-Toe and Chess

Both problems live in one file because they teach the same lesson from opposite ends: Tic-Tac-Toe is "how do you generalize a board so win-checking isn't hardcoded to 3×3," and Chess is "how do you model per-piece behavior without a Strategy explosion when polymorphism already does the job." Read them together.

## Requirements

### Tic-Tac-Toe

- "Fixed 3×3, or should the design generalize?" → **You decide**: generalize to N×N with a "get K in a row to win" rule — this is a near-guaranteed follow-up, so design for it up front rather than retrofitting.
- "How many players?" → 2, alternating turns. Out of scope: >2 players, AI opponent (mention Minimax as a follow-up talking point, don't implement it).
- "Draw detection?" → Yes — board full with no winner.

**In scope:** N×N board, alternating turns, win detection in O(1) per move (not a full-board rescan), draw detection.
**Out of scope:** AI opponent, network multiplayer, undo/redo (contrast with Chess, where undo is a natural follow-up).

### Chess

- "Full chess rules?" → **No — explicitly out of scope.** Castling, en passant, pawn promotion, threefold repetition, and full checkmate/stalemate detection are called out as follow-up talking points, not implemented. Say this out loud before writing code so the interviewer scopes with you instead of expecting FIDE-complete rules in 20 minutes.
- "What *is* in scope?" → Board representation, a `Piece` hierarchy with correct per-piece pseudo-legal move generation (Rook, Knight, Pawn, King shown; Bishop/Queen are the same sliding-piece pattern as Rook), move execution, and basic check detection (is a king's square currently attacked).
- "Move history / undo?" → Not core scope, but the design should visibly support it — see [Command](../patterns/03-behavioral-patterns.md#command) in Design patterns below.

**In scope:** 8×8 board, piece movement rules (pseudo-legal + "does this move leave your own king in check" filtering), turn alternation, check detection.
**Out of scope:** castling, en passant, promotion, checkmate/stalemate detection, draw-by-repetition, time controls, full legal-move enumeration for an engine.

## Core entities & relationships

### Tic-Tac-Toe

```
TicTacToeGame
  ├─ has-a[1] Board
  └─ has-a[*] Player

Board
  └─ has-a[*] Cell (Mark enum: EMPTY, X, O)
```

Board size lives as a constructor parameter (`int size`, `int winLength`), not a hardcoded `3`. `Mark` is a plain enum with no behavior — X/O don't do anything differently from each other, so there's no polymorphism to reach for here at all, unlike Chess's pieces.

### Chess

```
ChessGame
  ├─ has-a[1] Board
  └─ has-a[*] Player (has-a[1] Color)

Board
  └─ has-a[*] Piece (abstract)

Piece (abstract, shared state: Color, Position)
  ├─ Rook
  ├─ Knight
  ├─ Pawn
  └─ King
```

`Piece` is an **abstract class**, not an interface — every piece shares real state (`color`, `position`) and shared default behavior (bounds-checking, "is this square occupied by an enemy" helpers), which is exactly the decision rule in [Java OOP §2](../03-java-oop-essentials.md#2-abstract-class-vs-interface--the-decision-interviewers-watch-for): reach for `abstract class` when subtypes share state and some default logic, `interface` when you're only promising a capability. A `PieceType` enum would fail here the way it succeeded for the parking lot's `VehicleType` — piece *behavior* (how a Rook vs a Knight generates moves) genuinely differs, it isn't just a data lookup.

## Design patterns applied

- **Polymorphism, not Strategy, for `getValidMoves()`** — worth stating this distinction explicitly in an interview: each `Piece` subclass overriding `getValidMoves()` is ordinary polymorphism (the behavior is intrinsic to *being* a Rook, never swapped at runtime), whereas [Strategy](../patterns/03-behavioral-patterns.md#strategy) implies an *injectable, interchangeable* algorithm independent of the object's identity (like `PricingStrategy` in the parking lot problem, which any vehicle could in principle be charged under). Calling this "Strategy" in an interview is a common imprecision — say "this is polymorphism; I'd reach for Strategy if move rules needed to be swappable independent of piece identity, e.g. a chess-variant mode with custom piece rules," which shows you know the difference.
- [Command](../patterns/03-behavioral-patterns.md#command) — *not implemented in the core design, mentioned as the right extension point*: wrapping each executed move as a `MoveCommand` (with `execute()`/`undo()` and a reference to any captured piece) is how you'd add move-history/undo without polluting `Board` or `Piece` with history-tracking concerns. See Follow-ups.
- Composite is deliberately **not** used here — a chess board is a flat grid of independent pieces, not a part-whole tree where operations recurse (there's no "move this piece and everything nested inside it"); forcing Composite onto a board is a common overuse mistake worth naming and rejecting out loud.

## Java implementation

### Tic-Tac-Toe

```java
enum Mark { EMPTY, X, O }

final class Board {
    private final int size;
    private final Mark[][] grid;
    Board(int size) {
        this.size = size;
        this.grid = new Mark[size][size];
        for (Mark[] row : grid) Arrays.fill(row, Mark.EMPTY);
    }
    boolean placeMark(int row, int col, Mark mark) {
        if (grid[row][col] != Mark.EMPTY) return false;
        grid[row][col] = mark;
        return true;
    }
    int getSize() { return size; }
    Mark at(int row, int col) { return grid[row][col]; }
}

final class Player {
    final String name;
    final Mark mark;
    Player(String name, Mark mark) { this.name = name; this.mark = mark; }
}

final class TicTacToeGame {
    private final Board board;
    private final int winLength;
    private final Deque<Player> turnOrder;
    // O(1)-per-move win check: running sums per row/col/diagonal, not a full rescan.
    private final int[] rowCount, colCount;
    private int diagCount, antiDiagCount;
    private int movesPlayed = 0;

    TicTacToeGame(int size, int winLength, List<Player> players) {
        this.board = new Board(size);
        this.winLength = winLength;
        this.turnOrder = new ArrayDeque<>(players);
        this.rowCount = new int[size];
        this.colCount = new int[size];
    }

    Player play(int row, int col) {
        Player current = turnOrder.peekFirst();
        if (!board.placeMark(row, col, current.mark)) throw new IllegalArgumentException("Occupied");
        int delta = current.mark == Mark.X ? 1 : -1; // signed count: |count| == winLength means someone swept the line
        rowCount[row] += delta;
        colCount[col] += delta;
        if (row == col) diagCount += delta;
        if (row + col == board.getSize() - 1) antiDiagCount += delta;
        movesPlayed++;

        boolean won = Math.abs(rowCount[row]) == winLength || Math.abs(colCount[col]) == winLength
                   || Math.abs(diagCount) == winLength || Math.abs(antiDiagCount) == winLength;
        if (won) return current;
        if (movesPlayed == board.getSize() * board.getSize()) return null; // draw signal handled by caller checking movesPlayed
        turnOrder.addLast(turnOrder.removeFirst());
        return null;
    }

    boolean isDraw() { return movesPlayed == board.getSize() * board.getSize(); }
}
```

The running-sum trick generalizes to any `size`/`winLength` (e.g. Connect-4-style "4 in a row on a bigger board") without touching the win-check logic — it's the concrete payoff of not hardcoding a 3×3 scan.

### Chess

```java
enum Color { WHITE, BLACK }
record Position(int row, int col) {
    boolean inBounds() { return row >= 0 && row < 8 && col >= 0 && col < 8; }
}

abstract class Piece {
    protected final Color color;
    protected Position position;
    Piece(Color color, Position position) { this.color = color; this.position = position; }

    abstract List<Position> getValidMoves(Board board); // pseudo-legal: ignores self-check exposure, filtered by ChessGame

    Color getColor() { return color; }
    Position getPosition() { return position; }
    void moveTo(Position p) { this.position = p; }

    // shared helper, available to every subclass without reimplementation
    protected boolean canLandOn(Board board, Position p) {
        if (!p.inBounds()) return false;
        Piece occupant = board.pieceAt(p);
        return occupant == null || occupant.getColor() != this.color;
    }
}

final class Rook extends Piece {
    Rook(Color color, Position position) { super(color, position); }
    List<Position> getValidMoves(Board board) {
        List<Position> moves = new ArrayList<>();
        int[][] directions = {{1,0},{-1,0},{0,1},{0,-1}};
        for (int[] dir : directions) {
            int r = position.row() + dir[0], c = position.col() + dir[1];
            while (canLandOn(board, new Position(r, c))) {
                moves.add(new Position(r, c));
                if (board.pieceAt(new Position(r, c)) != null) break; // capture square is last step in this direction
                r += dir[0]; c += dir[1];
            }
        }
        return moves;
    }
}

final class Knight extends Piece {
    Knight(Color color, Position position) { super(color, position); }
    List<Position> getValidMoves(Board board) {
        int[][] deltas = {{1,2},{2,1},{-1,2},{-2,1},{1,-2},{2,-1},{-1,-2},{-2,-1}};
        List<Position> moves = new ArrayList<>();
        for (int[] d : deltas) {
            Position p = new Position(position.row() + d[0], position.col() + d[1]);
            if (canLandOn(board, p)) moves.add(p);
        }
        return moves;
    }
}

final class Pawn extends Piece {
    Pawn(Color color, Position position) { super(color, position); }
    List<Position> getValidMoves(Board board) {
        List<Position> moves = new ArrayList<>();
        int dir = color == Color.WHITE ? 1 : -1; // no en passant / no double-step-from-start / no promotion — see Out of scope
        Position forward = new Position(position.row() + dir, position.col());
        if (forward.inBounds() && board.pieceAt(forward) == null) moves.add(forward);
        for (int dc : new int[]{-1, 1}) {
            Position diag = new Position(position.row() + dir, position.col() + dc);
            if (diag.inBounds() && board.pieceAt(diag) != null && board.pieceAt(diag).getColor() != color) moves.add(diag);
        }
        return moves;
    }
}

final class King extends Piece {
    King(Color color, Position position) { super(color, position); }
    List<Position> getValidMoves(Board board) { // no castling
        List<Position> moves = new ArrayList<>();
        for (int dr = -1; dr <= 1; dr++)
            for (int dc = -1; dc <= 1; dc++) {
                if (dr == 0 && dc == 0) continue;
                Position p = new Position(position.row() + dr, position.col() + dc);
                if (canLandOn(board, p)) moves.add(p);
            }
        return moves;
    }
}

final class Board {
    private final Map<Position, Piece> occupied = new HashMap<>();
    void place(Piece p) { occupied.put(p.getPosition(), p); }
    Piece pieceAt(Position p) { return occupied.get(p); }
    void move(Piece p, Position to) {
        occupied.remove(p.getPosition());
        occupied.put(to, p);
        p.moveTo(to);
    }
    void remove(Position p) { occupied.remove(p); }
    List<Piece> piecesOf(Color color) {
        return occupied.values().stream().filter(pc -> pc.getColor() == color).toList();
    }
}

final class ChessGame {
    private final Board board = new Board();
    private Color turn = Color.WHITE;

    boolean isKingInCheck(Color kingColor) {
        Position kingPos = board.piecesOf(kingColor).stream()
            .filter(p -> p instanceof King).findFirst().orElseThrow().getPosition();
        Color enemy = kingColor == Color.WHITE ? Color.BLACK : Color.WHITE;
        return board.piecesOf(enemy).stream().anyMatch(p -> p.getValidMoves(board).contains(kingPos));
    }

    boolean makeMove(Piece piece, Position to) {
        if (piece.getColor() != turn) throw new IllegalStateException("Not your turn");
        if (!piece.getValidMoves(board).contains(to)) return false;
        Piece captured = board.pieceAt(to);
        Position from = piece.getPosition();
        board.move(piece, to); // tentative
        if (isKingInCheck(turn)) { // illegal: exposes own king — revert
            board.move(piece, from);
            if (captured != null) board.place(captured);
            return false;
        }
        turn = (turn == Color.WHITE) ? Color.BLACK : Color.WHITE;
        return true;
    }
}
```

## Python implementation

### Tic-Tac-Toe

```python
from dataclasses import dataclass
from enum import Enum
from collections import deque

class Mark(Enum):
    EMPTY = 0; X = 1; O = 2

@dataclass
class Player:
    name: str
    mark: Mark

class Board:
    def __init__(self, size: int):
        self.size = size
        self.grid = [[Mark.EMPTY] * size for _ in range(size)]

    def place_mark(self, row: int, col: int, mark: Mark) -> bool:
        if self.grid[row][col] != Mark.EMPTY:
            return False
        self.grid[row][col] = mark
        return True

class TicTacToeGame:
    def __init__(self, size: int, win_length: int, players: list[Player]):
        self.board = Board(size)
        self.win_length = win_length
        self.turn_order = deque(players)
        self.row_count = [0] * size
        self.col_count = [0] * size
        self.diag_count = 0
        self.anti_diag_count = 0
        self.moves_played = 0

    def play(self, row: int, col: int) -> Player | None:
        current = self.turn_order[0]
        if not self.board.place_mark(row, col, current.mark):
            raise ValueError("Occupied")
        delta = 1 if current.mark == Mark.X else -1  # signed running sum, O(1) win check
        self.row_count[row] += delta
        self.col_count[col] += delta
        size = self.board.size
        if row == col:
            self.diag_count += delta
        if row + col == size - 1:
            self.anti_diag_count += delta
        self.moves_played += 1

        won = (abs(self.row_count[row]) == self.win_length or abs(self.col_count[col]) == self.win_length
               or abs(self.diag_count) == self.win_length or abs(self.anti_diag_count) == self.win_length)
        if won:
            return current
        self.turn_order.rotate(-1)
        return None

    def is_draw(self) -> bool:
        return self.moves_played == self.board.size ** 2
```

### Chess

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from enum import Enum

class Color(Enum):
    WHITE = 1; BLACK = 2

@dataclass(frozen=True)
class Position:
    row: int
    col: int
    def in_bounds(self) -> bool:
        return 0 <= self.row < 8 and 0 <= self.col < 8

class Piece(ABC):
    def __init__(self, color: Color, position: Position):
        self.color = color
        self.position = position

    @abstractmethod
    def get_valid_moves(self, board: "Board") -> list[Position]: ...  # pseudo-legal; ChessGame filters self-check

    def can_land_on(self, board: "Board", p: Position) -> bool:  # shared helper, no reimplementation needed
        if not p.in_bounds():
            return False
        occupant = board.piece_at(p)
        return occupant is None or occupant.color != self.color

class Rook(Piece):
    def get_valid_moves(self, board):
        moves = []
        for dr, dc in ((1, 0), (-1, 0), (0, 1), (0, -1)):
            r, c = self.position.row + dr, self.position.col + dc
            while self.can_land_on(board, Position(r, c)):
                moves.append(Position(r, c))
                if board.piece_at(Position(r, c)) is not None:
                    break  # capture square is the last reachable step in this direction
                r += dr; c += dc
        return moves

class Knight(Piece):
    DELTAS = ((1,2),(2,1),(-1,2),(-2,1),(1,-2),(2,-1),(-1,-2),(-2,-1))
    def get_valid_moves(self, board):
        return [p for dr, dc in self.DELTAS
                if (p := Position(self.position.row + dr, self.position.col + dc)) and self.can_land_on(board, p)]

class Pawn(Piece):
    def get_valid_moves(self, board):
        moves = []
        direction = 1 if self.color == Color.WHITE else -1  # no double-step/en-passant/promotion — see Out of scope
        forward = Position(self.position.row + direction, self.position.col)
        if forward.in_bounds() and board.piece_at(forward) is None:
            moves.append(forward)
        for dc in (-1, 1):
            diag = Position(self.position.row + direction, self.position.col + dc)
            occupant = board.piece_at(diag) if diag.in_bounds() else None
            if occupant is not None and occupant.color != self.color:
                moves.append(diag)
        return moves

class King(Piece):
    def get_valid_moves(self, board):  # no castling
        moves = []
        for dr in (-1, 0, 1):
            for dc in (-1, 0, 1):
                if dr == 0 and dc == 0:
                    continue
                p = Position(self.position.row + dr, self.position.col + dc)
                if self.can_land_on(board, p):
                    moves.append(p)
        return moves

class Board:
    def __init__(self):
        self._occupied: dict[Position, Piece] = {}
    def place(self, piece: Piece) -> None:
        self._occupied[piece.position] = piece
    def piece_at(self, p: Position) -> Piece | None:
        return self._occupied.get(p)
    def move(self, piece: Piece, to: Position) -> None:
        del self._occupied[piece.position]
        self._occupied[to] = piece
        piece.position = to
    def remove(self, p: Position) -> None:
        self._occupied.pop(p, None)
    def pieces_of(self, color: Color) -> list[Piece]:
        return [p for p in self._occupied.values() if p.color == color]

class ChessGame:
    def __init__(self):
        self.board = Board()
        self.turn = Color.WHITE

    def is_king_in_check(self, king_color: Color) -> bool:
        king_pos = next(p.position for p in self.board.pieces_of(king_color) if isinstance(p, King))
        enemy = Color.BLACK if king_color == Color.WHITE else Color.WHITE
        return any(king_pos in p.get_valid_moves(self.board) for p in self.board.pieces_of(enemy))

    def make_move(self, piece: Piece, to: Position) -> bool:
        if piece.color != self.turn:
            raise RuntimeError("Not your turn")
        if to not in piece.get_valid_moves(self.board):
            return False
        captured = self.board.piece_at(to)
        origin = piece.position
        self.board.move(piece, to)  # tentative
        if self.is_king_in_check(self.turn):  # illegal: exposes own king — revert
            self.board.move(piece, origin)
            if captured is not None:
                self.board.place(captured)
            return False
        self.turn = Color.BLACK if self.turn == Color.WHITE else Color.WHITE
        return True
```

## Sample walkthrough

```python
# Tic-Tac-Toe, 3x3, standard rules
p1, p2 = Player("A", Mark.X), Player("B", Mark.O)
game = TicTacToeGame(size=3, win_length=3, players=[p1, p2])
game.play(0, 0)  # X
game.play(1, 1)  # O
winner = game.play(0, 1)  # X, no win yet -> None
winner = game.play(2, 2)  # O
winner = game.play(0, 2)  # X completes row 0 -> row_count[0] hits +3 -> returns p1

# Chess: White knight opens, Black responds
chess = ChessGame()
wn = Knight(Color.WHITE, Position(0, 1)); chess.board.place(wn)
bk = King(Color.BLACK, Position(7, 4)); chess.board.place(bk)
chess.make_move(wn, Position(2, 2))   # legal knight hop, turn flips to BLACK
```

## Follow-up questions

- **"Generalize Tic-Tac-Toe to Connect-4 (gravity, drop into a column)."** `Board.place_mark` changes from "place at (row, col)" to "drop into column, land on lowest empty row" — the win-check running-sum machinery in `TicTacToeGame.play` is untouched since it only cares about the final `(row, col)` a mark landed on, not how it got there.
- **"Add an undo button to the chess game."** Wrap `ChessGame.make_move` in a `MoveCommand` (holding piece, from, to, captured-piece) with an `undo()` that reverses `board.move`/`board.place` — per [Command](../patterns/03-behavioral-patterns.md#command); maintain a `Deque[MoveCommand]` history in `ChessGame`. This is additive: `Piece`/`Board` don't need to know history is being tracked.
- **"Detect checkmate, not just check."** Extend `isKingInCheck` to `isCheckmate`: king is in check *and* no legal move by any of that color's pieces resolves it — i.e., simulate `make_move` for every `(piece, destination)` pair for the side to move and see if any leaves `is_king_in_check == False`. Expensive but a direct extension of the existing self-check-filtering logic in `make_move`, not a new subsystem.
- **"Support castling / en passant / pawn promotion."** Each needs board-state beyond "current piece positions" — castling needs "has this king/rook ever moved," en passant needs "did the enemy pawn's *last* move double-step," promotion needs a user choice on reaching the back rank. Flag that this means `Board` needs a small amount of move-history-aware state (or `ChessGame` tracks it), and `Pawn`/`King` gain a few extra conditional branches — real work, correctly scoped out up front rather than discovered mid-interview.
- **"Two Tic-Tac-Toe / Chess clients over a network — where does state live?"** `TicTacToeGame`/`ChessGame` stay authoritative on a server; clients send `(row, col)` or `(piece, destination)` intents and receive the resulting state — the class boundary you already have (game owns board, exposes one `play`/`make_move` entry point) is exactly the seam a network layer wraps, not a redesign.

## Common mistakes on this problem

- Hardcoding Tic-Tac-Toe's win check to `size == 3` (eight explicit line checks) instead of the generalized running-sum approach — works for the stated problem, collapses the instant "what about 4×4" follow-up lands.
- Modeling chess pieces with a single `Piece` class and a `PieceType` enum plus a giant `switch` in `getValidMoves()` — this is the *wrong* lesson to take from the parking lot's `VehicleType`; piece move generation is genuinely different logic per type (sliding vs jumping vs single-step), which is precisely when polymorphism earns its keep over an enum.
- Forcing [Composite](../patterns/02-structural-patterns.md#composite) onto the chess board ("pieces and squares and the board are all `BoardComponent`") because "board games feel tree-shaped" — there's no part-whole recursion here; resist naming a pattern without a concrete use, per the overuse trap called out in [patterns/00-overview.md](../patterns/00-overview.md).
- Skipping the "does this move leave my own king in check" filter and calling pseudo-legal moves "valid moves" — a `Rook` correctly refusing to jump over pieces is necessary but not sufficient; a move that's otherwise legal but exposes your own king must still be rejected, and it's a common gap in rushed chess implementations.

## Continue

Next: [05-lru-cache-and-rate-limiter.md](05-lru-cache-and-rate-limiter.md)
