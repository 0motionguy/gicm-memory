# @gicm/memory

Four-layer memory for AI agents: hot, warm, cold and archive tiers, hybrid keyword and vector search, and automatic migration between tiers.

The engine keeps recent entries in process memory and ages older ones down through SQLite-backed tiers, shortening them as they go. It is for developers building agents that should remember across sessions without running a vector database or calling an embeddings API. Search blends keyword matching with a small local vector, so results are approximate rather than learned-semantic.

Part of [gICM](https://github.com/0motionguy/gICM), a family of OpenClaw agent packages. API reference and examples: [docs/DEVELOPMENT.md](docs/DEVELOPMENT.md).

## What it does

- **Remember:** `remember(key, value, { type, metadata, expiresAt })` stores an entry in the hot layer with a token estimate and a 64-dimension embedding. Types are `episode`, `fact`, `improvement`, `goal` and `context`.
- **Migrate:** `flush()` moves hot entries to warm after 5 minutes or when the layer is full, warm to cold after 7 days (values cut to 200 characters) and cold to archive after 90 days (first sentence kept as a fact). Each layer has a capacity cap and is compacted. Ages and caps are configurable.
- **Search:** `search(query, { layers, limit, threshold })` scores 0.6 keyword match (exact, whole-word, substring) plus 0.4 cosine similarity; the default threshold is 0.3.
- **Isolate:** a `namespace` keeps several agents' memories apart.
- **Observe:** events `memory:added`, `memory:flushed`, `memory:searched` and `memory:compacted`.
- **Persist:** warm, cold and archive tiers use SQLite (WAL mode) through the optional `better-sqlite3` peer dependency and fall back to in-process storage with a console warning when it cannot be loaded.

## Example

```ts
import { MemoryEngine } from "@gicm/memory";

const memory = new MemoryEngine({ namespace: "support-agent", dbPath: "./memory.db" });
await memory.remember("preferred-language", "TypeScript", { type: "fact" });
const hits = await memory.search("language preference");
await memory.flush();
```

## How it's built

TypeScript (ESM) bundled with tsup and tested with vitest; zod validates the config. Embeddings come from `simpleEmbed`, a character-hashing function into 64 dimensions, so no model or API is involved. SQLite access is isolated in `db.ts`.

## Repository layout

- `src/memory-engine.ts`: layers, flush, search and scoring
- `src/db.ts`: SQLite storage with an in-memory fallback
- `src/embeddings.ts`: hashed embedding and cosine similarity
- `src/__tests__/`: vitest suite (37 test cases)
- `skill/SKILL.md`: the OpenClaw skill definition

## Run it locally

```bash
pnpm install
pnpm build   # tsup -> dist/
pnpm test    # vitest (add --run for a single pass)
```

## Status

Beta. Version 1.0.0, a single extracted release (February 2026). Known gap: `db.ts` loads `better-sqlite3` with `require()` inside an ESM-only build, which plain Node ESM does not provide, so as written persistence falls back to memory with a warning; switch it to `createRequire` or a dynamic import before relying on it.
