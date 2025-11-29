# Technical Task: RGS Platform — Full Stack

## Position
**Senior Fullstack Developer**

## Timeframe
**2-3 days**

## Task Goal
Build a **complete RGS (Remote Gaming Server) platform from scratch**:
1. Create a simple game (Game Provider)
2. Build RGS platform for game integration
3. Create an operator (Casino Backend)
4. Integrate everything together — so the game launches from the operator

Use **only Claude Code** as the primary development tool.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        OPERATOR (Casino)                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
│  │  Casino UI  │───▶│Casino Backend│───▶│   Wallet    │          │
│  │  (React)    │    │  (NestJS)   │    │  (MongoDB)  │          │
│  └─────────────┘    └──────┬──────┘    └─────────────┘          │
│                            │                                     │
└────────────────────────────┼─────────────────────────────────────┘
                             │ Seamless Wallet API
                             │ (authenticate, bet, win, rollback)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                          RGS PLATFORM                            │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
│  │   Adapter   │◀──▶│  RGS Core   │◀──▶│   Session   │          │
│  │  (Operator) │    │  (NestJS)   │    │   (Redis)   │          │
│  └─────────────┘    └──────┬──────┘    └─────────────┘          │
│                            │                                     │
└────────────────────────────┼─────────────────────────────────────┘
                             │ Game API
                             │ (init, play, history)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                       GAME PROVIDER                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
│  │  Game UI    │◀──▶│Game Backend │◀──▶│    RNG      │          │
│  │  (React)    │    │ (NestJS)    │    │  (crypto)   │          │
│  └─────────────┘    └─────────────┘    └─────────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Three independent services:**
1. **Game Provider** — game with simple logic
2. **RGS Platform** — integration layer
3. **Operator** — casino with player wallets

---

## Part 1: Game Provider (Game)

Create a simple game — for example **Dice** or **Coin Flip**.

### Game Backend (NestJS)

```typescript
// POST /game/init
Request:  { sessionId: string }
Response: {
  balance: number,
  availableBets: [10, 25, 50, 100],
  gameInfo: { name: "Dice", rtp: 96 }
}

// POST /game/play
Request:  { sessionId: string, betAmount: number, choice: "high" | "low" }
Response: {
  result: number,        // 1-100
  win: boolean,
  winAmount: number,
  balance: number,
  roundId: string
}
```

**Game Logic (Dice):**
- Player chooses "high" (>50) or "low" (≤50)
- A number 1-100 is generated
- If guessed correctly — win = bet × 1.9
- RNG via `crypto.randomInt()`

### Game Frontend (React, minimal)

```
┌─────────────────────────────────────────┐
│            🎲 DICE GAME                 │
├─────────────────────────────────────────┤
│  Balance: 1000.00 USD                   │
├─────────────────────────────────────────┤
│  Bet: [10] [25] [50] [100]              │
│                                         │
│     [LOW ≤50]      [HIGH >50]           │
├─────────────────────────────────────────┤
│  Result: 73 — HIGH WINS! +190           │
└─────────────────────────────────────────┘
```

---

## Part 2: RGS Platform (Integration Layer)

RGS receives requests from the game and communicates with the operator.

### RGS API

```typescript
// POST /rgs/session/create
Request:  { token: string, gameCode: string, operatorCode: string }
Response: { sessionId: string, balance: number, currency: string }

// POST /rgs/transaction/bet
Request:  { sessionId: string, amount: number, roundId: string }
Response: { success: boolean, balance: number, transactionId: string }

// POST /rgs/transaction/win
Request:  { sessionId: string, amount: number, roundId: string }
Response: { success: boolean, balance: number, transactionId: string }

// POST /rgs/transaction/rollback
Request:  { sessionId: string, originalTransactionId: string }
Response: { success: boolean, balance: number }
```

### Operator Adapter

Abstraction for different operators:

```typescript
export abstract class BaseAdapter {
  abstract authenticate(token: string): Promise<AuthResult>;
  abstract bet(session: Session, amount: number, roundId: string): Promise<BetResult>;
  abstract win(session: Session, amount: number, roundId: string): Promise<WinResult>;
  abstract rollback(session: Session, transactionId: string): Promise<RollbackResult>;
}

// Implementation for a specific operator
export class DemoOperatorAdapter extends BaseAdapter {
  private generateHmac(body: object): string {
    const bodyString = JSON.stringify(body);
    return createHmac('sha256', this.secretKey).update(bodyString).digest('hex');
  }

  async bet(session, amount, roundId) {
    // 1. Check idempotency
    // 2. Call operator API with HMAC
    // 3. Save transaction
    // 4. Return result
  }
}
```

### Key RGS Requirements:

1. **Session Management** — Redis with TTL
2. **Idempotency** — repeated requests return the same result
3. **HMAC signature** — all requests to operator are signed
4. **Retry mechanism** — for win operations
5. **Transactions** — MongoDB for history storage

---

## Part 3: Operator (Casino Backend)

Casino backend with Seamless Wallet API.

### Wallet API (receives requests from RGS)

```typescript
// POST /api/wallet/authenticate
Request:  { token: string }
Headers:  X-Signature: HMAC-SHA256
Response: { userId: string, balance: number, currency: string }

// POST /api/wallet/bet
Request:  { sessionId: string, amount: number, transactionId: string, roundId: string }
Headers:  X-Signature: HMAC-SHA256
Response: { balance: number, transactionId: string }

// POST /api/wallet/win
Request:  { sessionId: string, amount: number, transactionId: string, roundId: string }
Headers:  X-Signature: HMAC-SHA256
Response: { balance: number, transactionId: string }

// POST /api/wallet/rollback
Request:  { sessionId: string, transactionId: string, originalTransactionId: string }
Headers:  X-Signature: HMAC-SHA256
Response: { balance: number }
```

### Operator Requirements:

1. **HMAC validation** — verify incoming request signatures
2. **Player Wallet** — MongoDB schema for balance
3. **Idempotency** — transactionId as key
4. **Atomicity** — MongoDB transactions for bet/win

### Casino UI (minimal)

```
┌─────────────────────────────────────────┐
│          🎰 DEMO CASINO                 │
├─────────────────────────────────────────┤
│  Player: john_doe                       │
│  Balance: 1000.00 USD                   │
├─────────────────────────────────────────┤
│  Available Games:                       │
│  [🎲 Dice]  [🪙 Coin Flip]              │
├─────────────────────────────────────────┤
│  Click game to play →                   │
└─────────────────────────────────────────┘
```

When clicking on a game → opens iframe with Game UI.

---

## Part 4: Integration (Everything Together)

### Full Game Flow:

```
1. Player opens Casino UI
2. Player clicks "Dice"
3. Casino creates token and opens Game UI in iframe
4. Game UI → Game Backend → RGS → Operator (authenticate)
5. Player places bet
6. Game UI → Game Backend → RGS → Operator (bet)
7. Game generates result (RNG)
8. Game UI → Game Backend → RGS → Operator (win)
9. Player sees result and updated balance
```

### Launch URL Format:

```
https://game.example.com/dice?token={TOKEN}&operator={OPERATOR_CODE}
```

---

## Infrastructure

### Docker Compose

```yaml
services:
  # Databases
  mongodb:
    image: mongo:7
    ports: ["27017:27017"]

  redis:
    image: redis:7
    ports: ["6379:6379"]

  # Operator (Casino)
  operator-backend:
    build: ./operator/backend
    ports: ["5555:5555"]
    environment:
      - HMAC_SECRET=operator-secret-key
    depends_on: [mongodb]

  operator-frontend:
    build: ./operator/frontend
    ports: ["5556:5556"]

  # RGS Platform
  rgs:
    build: ./rgs
    ports: ["3000:3000"]
    environment:
      - OPERATOR_URL=http://operator-backend:5555
      - OPERATOR_SECRET=operator-secret-key
    depends_on: [mongodb, redis, operator-backend]

  # Game Provider
  game-backend:
    build: ./game/backend
    ports: ["4000:4000"]
    environment:
      - RGS_URL=http://rgs:3000
    depends_on: [rgs]

  game-frontend:
    build: ./game/frontend
    ports: ["4001:4001"]
```

### Project Structure

```
project/
├── operator/
│   ├── backend/          # NestJS — Wallet API
│   └── frontend/         # React — Casino UI
├── rgs/                   # NestJS — RGS Platform
├── game/
│   ├── backend/          # NestJS — Game Logic
│   └── frontend/         # React — Game UI
├── docker-compose.yml
├── CLAUDE.md
└── README.md
```

---

## Key Checks (What Should Work)

### 1. End-to-End Flow
```
1. Open Casino UI → login
2. Click on Dice game
3. Game opens with player balance
4. Place bet → balance decreased
5. Get result → if win, balance increased
6. Close game → balance in Casino UI updated
```

### 2. HMAC Validation (RGS → Operator)
```
1. RGS sends request without X-Signature → Operator responds 401
2. RGS sends with wrong signature → 401
3. RGS sends with correct signature → 200
```

### 3. Idempotency
```
1. Bet (txn_001) → balance = 900
2. Bet (txn_001) again → balance = 900 (NOT 800!)
3. Win (txn_002) → balance = 1090
4. Win (txn_002) again → balance = 1090 (NOT 1280!)
```

### 4. Rollback
```
1. Bet (txn_001) → balance = 900
2. Error in game
3. Rollback (txn_001) → balance = 1000
```

### 5. Session Expiry
```
1. Create session
2. Wait > TTL (e.g., 30 minutes)
3. Attempt play → error "Session expired"
```

---

## Development Process Requirements

### Mandatory

1. **CLAUDE.md file in project root**
   - Full system architecture
   - API contracts for each service
   - Commands to run
   - Instructions for Claude Code

2. **Git history**
   - Meaningful commits
   - Minimum 15-20 commits (across all services)

3. **README.md**
   - How to run (docker-compose up)
   - How to test
   - Architecture description

### Desirable

- Unit tests for critical modules
- Swagger documentation
- Transaction logging

---

## Evaluation Criteria

### 1. Working with Claude Code (30%)

| Criterion | Description | Points |
|-----------|-------------|--------|
| Prompt engineering | Task decomposition, prompt formulation | 0-8 |
| AI code review | Checking and fixing generated code | 0-7 |
| Architectural planning | Using AI for design | 0-8 |
| CLAUDE.md quality | Usefulness of the file for AI assistant | 0-7 |

### 2. Game Provider (15%)

| Criterion | Description | Points |
|-----------|-------------|--------|
| Game logic | RNG, win calculation, RTP | 0-8 |
| Game API | init, play endpoints work | 0-7 |

### 3. RGS Platform (25%)

| Criterion | Description | Points |
|-----------|-------------|--------|
| Operator Adapter | HMAC, operator API calls | 0-10 |
| Session Management | Redis, TTL, validation | 0-8 |
| Idempotency | Transaction deduplication | 0-7 |

### 4. Operator (Casino) (20%)

| Criterion | Description | Points |
|-----------|-------------|--------|
| Wallet API | authenticate, bet, win, rollback | 0-10 |
| HMAC validation | Incoming signature verification | 0-5 |
| Atomicity | MongoDB transactions | 0-5 |

### 5. Integration and Infrastructure (10%)

| Criterion | Description | Points |
|-----------|-------------|--------|
| Docker Compose | All services start | 0-5 |
| End-to-End Flow | Full cycle works | 0-5 |

**Maximum: 100 points**

---

## What to Submit

1. **GitHub repository**
2. **Screencast** (10-15 minutes):
   - Launch via docker-compose
   - Demo Casino UI → launch game
   - Gameplay (bet → win/lose)
   - Balance update in Casino
   - Show rollback (if possible)
3. **Brief report** (1-2 pages):
   - What prompts you used for Claude Code
   - Architectural decisions
   - Problems and how you solved them
   - What you would improve with more time

---

## Technology Stack

### Backend (all services)
- NestJS 10+
- TypeScript 5+
- MongoDB + Mongoose
- Redis + ioredis
- crypto (for HMAC and RNG)

### Frontend (Game UI and Casino UI)
- React 18+ or Vue 3+
- TypeScript
- Vite
- Any UI kit (or plain CSS)

### Infrastructure
- Docker + Docker Compose
- Node.js 20+

---

## Bonus Tasks (Optional)

- [ ] Second game (Coin Flip)
- [ ] Second operator with different API format
- [ ] Provably Fair (show seed after game)
- [ ] WebSocket for real-time balance
- [ ] Transaction history in Casino UI
- [ ] Rate limiting

---

## FAQ

**Q: Do I need beautiful graphics in the game?**
A: No. UI is minimal — buttons and text. Focus on integration.

**Q: Can I combine services?**
A: No. Game, RGS, and Operator must be separate services.

**Q: What is the HMAC format?**
A: `HMAC-SHA256(secretKey, JSON.stringify(body))` in hex, header `X-Signature`.

**Q: Can I use other AI besides Claude Code?**
A: No, only Claude Code. This is a key part of the evaluation.

**Q: What if I don't finish in 2-3 days?**
A: Priorities:
   1. RGS + Operator integration (CRITICAL)
   2. Game Provider (important)
   3. UI (minimum — working buttons)

   Better fewer features, but a working E2E flow.
