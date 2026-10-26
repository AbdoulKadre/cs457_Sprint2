# Message Flow Sequence Diagrams — Tic-Tac-Toe Server

**Project:** Tic-Tac-Toe 
**Sprint:** Sprint 2 — Game Behavior & Server Concurrency Architecture Design
**Author:** Abdoul Kadre Tiemtore

Companion to `concurrency_design.md`. All diagrams use **Mermaid** syntax and render directly on GitHub.

## 1.Turn Synchronization

A move by Player X (Client 1) is validated, applied, and the new state is broadcast to both clients inside a single locked critical section on the server.

```mermaid
sequenceDiagram
    actor P1 as Client 1 (Player X)
    participant S as Game Server
    actor P2 as Client 2 (Player O)
    Note over P1,S: Turn: Player X
    P1->>S: MOVE (row: 0, col: 2)
    Note right of S: Acquire room_lock
    Note right of S: Validate: sender is current player? cell empty?
    Note right of S: Update board, flip turn to Player O
    S-->>P1: STATE_UPDATE (Board, Turn: Player O)
    S-->>P2: STATE_UPDATE (Board, Turn: Player O)
    Note over S: Release room_lock
    Note over P2,S: Player O's turn
    P2->>S: MOVE (row: 1, col: 1)
    Note right of S: Acquire room_lock, validate, update, flip turn
    S-->>P2: STATE_UPDATE (Board, Turn: Player X)
    S-->>P1: STATE_UPDATE (Board, Turn: Player X)
    Note over S: Release room_lock
```

## 2.Invalid Move / Not Your Turn Flow

The server rejects the request with `ERROR` and the authoritative state is unchanged.

```mermaid
sequenceDiagram
    actor P1 as Client 1 (Player X)
    participant S as Game Server
    actor P2 as Client 2 (Player O)
    Note over P1,S: Turn: Player O
    P1->>S: MOVE (row: 1, col: 0)
    Note right of S: Acquire room_lock
    Note right of S: Validate: sender is NOT the current player
    S-->>P1: ERROR "Not Your Turn"
    Note over S: Release room_lock (no state change)
    P2->>S: MOVE (row: 1, col: 0)
    Note right of S: Acquire room_lock
    Note right of S: Validate: cell already taken
    S-->>P2: ERROR "Invalid Move"
    Note over S: Release room_lock (no state change)
    Note over P1,S: Board and turn unchanged for both players
```

## 3.Win Detection & Score Broadcast

```mermaid
sequenceDiagram
    actor P1 as Client 1 (Player X)
    participant S as Game Server
    actor P2 as Client 2 (Player O)
    Note over P1,S: Turn: Player X
    P1->>S: MOVE (row: 2, col: 2)
    Note right of S: Acquire room_lock
    Note right of S: Validate and update board
    Note right of S: Check lines: Player X wins
    Note right of S: Update scoreboard (X: 2, O: 1) under room_lock
    S-->>P1: GAME_OVER (winner: X, scores)
    S-->>P2: GAME_OVER (winner: X, scores)
    Note over S: Release room_lock
```

## 4.Connection & Matchmaking

Two clients connect concurrently; the server spawns one worker thread per connection, pairs them into a room under `registry_lock`, then each room runs independently.

```mermaid
sequenceDiagram
    participant C1 as Client 1
    participant L as Listener (ThreadingTCPServer)
    participant MM as Matchmaker (registry_lock)
    participant R as Game Room (room_lock)
    participant C2 as Client 2
    C1->>L: TCP connect
    L->>MM: Spawn worker thread 1, JOIN(name)
    Note over MM: Acquire registry_lock, no opponent yet → waiting queue
    C2->>L: TCP connect
    L->>MM: Spawn worker thread 2, JOIN(name)
    Note over MM: Acquire registry_lock, pair both clients
    MM->>R: Create room (board, turn: X, scores)
    MM-->>C1: STATE_UPDATE (Turn: Player X)
    MM-->>C2: STATE_UPDATE (Turn: Player X)
    Note over R: Room now runs independently of all other rooms
```



- All validation, board mutation, and broadcasts happen server-side inside the room's `threading.Lock` critical section — see `concurrency_design.md` §2.3 and §3.1.
- The two `ERROR` cases (`"Invalid Move"`, `"Not Your Turn"`) leave the authoritative state untouched, so both clients stay consistent.
- Diagrams were validated with the Mermaid Live Editor (mermaid.live) and render natively on GitHub.