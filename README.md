# 🎲 Monopoly Online — Realtime Architecture

## Goal

An **online Monopoly clone**, fully **real-time**, ensuring a **consistent global state** across all players.

---

## Tech Stack

- **Frontend**
    - React + **SVG** board
    - Realtime animations via **WebSocket**
    - Global state handled with **Redux** (UI = dumb components)
- **Backend**
    - Node.js + **Fastify**
    - **REST API** for init/queries
    - **WebSocket** hub for events
- **Databases**
    - **Neo4j** → persistent entities (players, pawns, tiles, properties, houses, hotels, cash)
    - **MongoDB** → event-driven log (turns, transactions, mortgages, doubles, history)

---

## High-Level Flow

```
[React+Redux] <—REST—> [Fastify API] <—> [Neo4j] (board & cash)
     ↑  ↓ WS
  (events) <—WS—> [Fastify WS hub] <—> [MongoDB] (event log / turns)

```

- Cold reads (join/refresh) = **Neo4j state** + last events from **Mongo**
- Game actions (`roll`, `buy`, `pay`) → REST **command** → validated → Neo4j + Mongo update → WS broadcast `game.state.updated`
- Clients resync if behind (`/sync?since=version`)

---

## Example Data Models

### Neo4j (persistent)

```
MERGE (g:Game {gameId:$gameId}) SET g.version=1;
UNWIND $tiles AS t
  MERGE (tile:Tile {tileId:t.id})
  SET tile.index=t.index, tile.type=t.type, tile.label=t.label
  MERGE (g)-[:HAS_TILE]->(tile);

```

### MongoDB (event log)

```json
{
  "gameId": "g1",
  "version": 42,
  "type": "player.moved",
  "ts": 1737904123456,
  "payload": { "playerId":"p1","from":10,"to":24,"dice":[6,8],"passedGo":true }
}

```

---

## API Contracts

### REST

- `POST /games` → create game
- `POST /games/:gid/join` → add player
- `GET /games/:gid/state` → aggregated state + version
- `POST /games/:gid/roll` { playerId }
- `POST /games/:gid/buy` { playerId, propId }

### WebSocket

- `game.state.updated` { version, diffs }
- `player.moved` { playerId, from, to, dice }
- `property.bought` { propId, by, price }
- `rent.paid` { from, to, amount }

---

## Redux Store (simplified)

```tsx
type RootState = {
  game: { gameId: string; version: number; players: Record<string, Player> };
  ui:   { ws: { connected: boolean } };
}

```

Reducers = pure, aligned with **WS events**.

Middleware handles **WebSocket connection & dispatch**.

---

## Quick Start

```bash
pnpm --filter front dev   # run frontend
pnpm --filter back dev    # run backend
pnpm --filter back seed:board

```

```yaml
# docker-compose.yml
services:
  mongo:
    image: mongo:7
    ports: ["27017:27017"]
  neo4j:
    image: neo4j:5
    environment: { NEO4J_AUTH: neo4j/test }
    ports: ["7474:7474","7687:7687"]

```

---

⚡ **Roadmap**: seed board → WS hub → roll/move → buy/pay → mortgages → snapshots/resync → full UI.
