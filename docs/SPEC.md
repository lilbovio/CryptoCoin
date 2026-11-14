# SPEC.md — CryptoCoin (hash-chained database / mini-blockchain)

> Single source of truth for all agents. Language: English. If anything here conflicts with a worker's assumption, this file wins. Only ASTRA may change it, and only with the owner's approval.
> Revision 2: monorepo with `core` / `server` / `client` / `tools`.

## 1. Project

- **Course:** IH063 Cryptography — final project "Base de datos encadenada con hash (mini-blockchain)".
- **Theme:** a basic cryptocurrency ledger called **CryptoCoin** (a record of transfers).
- **Goal:** each record stores a hash that depends on the previous record. If any past record is modified, the chain breaks and the application detects it.
- **Out of scope (do NOT build):** consensus, mining / proof of work, P2P networking, wallets, digital signatures, graphical or web UI (the client is a CLI), external databases (MongoDB, D1), any dependency beyond those listed in §8.

## 2. Block contract

Each record ("block") has exactly these fields:

| Field | Type | Rule |
|---|---|---|
| `id` | integer | Genesis = 0; each new block = previous `id` + 1 (no gaps) |
| `timestamp` | string | UTC, format `YYYY-MM-DDTHH:MM:SS.ffffffZ` (microseconds, trailing `Z`). Stored and hashed exactly as generated; never reformatted |
| `data` | object | Transaction `{"from": str, "to": str, "amount": int}` |
| `prev_hash` | string | 64 lowercase hex chars; `hash` of the previous block. Genesis: 64 zeros |
| `hash` | string | 64 lowercase hex chars; computed as below |

### 2.1 Hash formula (canonical serialization)

```
hash = SHA-256( UTF-8( f"{id}|{timestamp}|{canonical_json(data)}|{prev_hash}" ) )  → lowercase hex
canonical_json(data) = json.dumps(data, sort_keys=True, separators=(",", ":"), ensure_ascii=False)
```

- `|` is the separator. It cannot cause ambiguity: `id`, `timestamp` and `prev_hash` have fixed formats that never contain `|`, and `prev_hash` is always last.
- The formula is H(ID || timestamp || data || prev_hash), as required by the course.
- The stored `data` must be re-serialized with `canonical_json` every time a hash is computed.
- This formula exists in exactly **one** place (`core`). Server, client and tools never reimplement it.

### 2.2 Genesis block

- `id = 0`, `prev_hash = "0" * 64`, `data = {"from": "GENESIS", "to": "GENESIS", "amount": 0}`.
- Created automatically when the database is empty. Never duplicated when the server restarts.

### 2.3 Transaction rules (checked when adding a block)

- `amount` is a **positive integer** (genesis excepted). No floats.
- `from` and `to` are non-empty strings, and `from != to`.
- The special sender `SYSTEM` can issue new coins (no balance check). Any other sender must have `balance >= amount`, where balance = sum received − sum sent over the existing chain.
- An invalid transaction is rejected with a clear error; **no block is created**.

## 3. Functional requirements

| ID | Requirement |
|---|---|
| F1 | **Add** a new block at the end of the chain (computes `id`, `timestamp`, `prev_hash`, `hash`) |
| F2 | **Show** all blocks in order with all fields, including full hashes |
| F3 | **Verify** integrity (see §4) and report whether the chain is valid |
| F4 | **Persist** to SQLite: after restarting the server, blocks remain and the chain can be verified |
| F5 | **Simulate an attack** with a standalone tool that edits the database directly (see §5), then verify through the client |
| F6 | **Show balance** of an account (supports the balance rule and the demo) |

### 3.1 Components

| Component | Responsibility | May import |
|---|---|---|
| `core` | Pure logic: block, hashing, chain rules, transaction rules, verifier, `Storage` interface. No network, no SQLite, no printing | stdlib only |
| `server` | HTTP API. SQLite implementation of `Storage`. Calls `core` for every rule | `core` |
| `client` | CLI: interactive menu (+ recommended non-interactive subcommands). Talks to the server **only via HTTP** | HTTP client library; never `core` |
| `tools` | Attack simulator and demo reset. Edits the database file directly | `core` (hash function only), `sqlite3` |

Dependency direction: `client → (HTTP) → server → core`; `tools → core`. `core` imports nothing from the other packages.

### 3.2 API (minimum contract, ASTRA may extend but not break)

| Method and path | Body / result |
|---|---|
| `POST /transactions` | Body `{"from","to","amount"}` → `201` with the created block. Invalid → `422` |
| `GET /chain` | `200` list of all blocks in order |
| `GET /verify` | `200` `{"valid": bool, "block_count": int, "findings": [{"block_id": int, "code": str, "expected": str\|null, "found": str\|null}]}` |
| `GET /balance/{account}` | `200` `{"account": str, "balance": int}` (unknown account = 0) |
| `GET /health` | `200` `{"status": "ok"}` |

Rules:
- The JSON key is `from`. In Python it is a reserved word: use a field alias (e.g., `from_` with alias `from`). The hashed `data` always uses the key `from`.
- Errors: JSON `{"error": "<CODE>", "detail": "<message>"}` with codes `INVALID_AMOUNT`, `SAME_ACCOUNT`, `EMPTY_ACCOUNT`, `INSUFFICIENT_BALANCE`.
- **The server must never cache the chain.** Every request reads storage fresh, so `/verify` always reflects the current content of the database file, including edits made behind the server's back.
- Adding a block is **atomic** (read last block, validate, compute, write inside one lock/transaction) so concurrent requests cannot fork the chain.
- The API exposes no endpoint that edits or deletes existing blocks.
- Server address configurable: default `http://127.0.0.1:8000`; the client accepts `--server` or env `CRYPTOCOIN_SERVER`. Database path via env `CRYPTOCOIN_DB`, default `chain.db`.

### 3.3 Client

- Interactive menu (required): add, show, verify, balance, exit. Clear messages with ✅ / ❌.
- If the server is unreachable, show a clear message (not a stack trace).
- Non-interactive subcommands recommended (`add`, `show`, `verify`, `balance`); `verify` exits 0 if valid, 1 if not.
- All user-facing messages in **Spanish**.
- Database files are never committed (`.gitignore`).

## 4. Verification algorithm (F3)

Implemented **only** in `core` (operates on an in-memory list of blocks); the server's `GET /verify` calls it. Scan **all** blocks in order. **Never stop at the first error.** For each block `b` collect findings:

| Code | Condition |
|---|---|
| `GENESIS_INVALID` | Block 0 does not match §2.2 |
| `ID_GAP` | `b.id` is not previous `id` + 1 |
| `MALFORMED` | Missing field, wrong type or wrong format |
| `HASH_MISMATCH` | Recomputed hash ≠ stored `b.hash` (content altered) |
| `LINK_BROKEN` | `b.prev_hash` ≠ **stored** `hash` of the previous block |
| `INSUFFICIENT_BALANCE` | Replaying balances over the stored data, the sender lacks funds (semantic check, secondary) |

Output: valid with block count, or a list of findings (block `id`, code, expected vs found). A tampered block is reported as `HASH_MISMATCH` (content altered) or `LINK_BROKEN` on the next block (hash rewritten), clearly distinguished.

Known and accepted limitations (must appear in the report):
1. An attacker who rewrites **all** subsequent hashes consistently is not detected (no external anchor, signature or consensus).
2. Altering the **last** block and recomputing its hash is not detected: no later block contradicts it.
3. **Truncating** the tail of the chain (deleting the last blocks) is not detected: the shorter chain remains valid.

## 5. Attack simulation (F5)

- A standalone script in `tools/` that **modifies a stored record directly in the SQLite file, bypassing the server and the application logic** (raw SQL). It is NOT exposed by the API or the client.
- Options: database path, block `id`, new `amount`; `--rewrite-hash` (recomputes the altered block's hash using the `core` hash function, to demonstrate `LINK_BROKEN`); `--delete-id` (delete a block, to demonstrate `ID_GAP`).
- A reset script/option recreates the demo database so the demo is repeatable.
- Must work while the server is running; the next `GET /verify` must see the change (§3.2, no cache).

## 6. Reference test vectors

Computed with the exact formula in §2.1 (use them as unit tests).

| id | timestamp | data (canonical JSON) | prev_hash | hash |
|---|---|---|---|---|
| 0 | `2026-11-14T00:00:00.000000Z` | `{"amount":0,"from":"GENESIS","to":"GENESIS"}` | `0`×64 | `d8b7ce7af6a1457fd7974f2b5b25268ffd3bb4537945db9844b9bb6c0c62b412` |
| 1 | `2026-11-14T00:01:00.000000Z` | `{"amount":100,"from":"SYSTEM","to":"alice"}` | hash of 0 | `ef4ac89c5f7942475656f2b1196602671078ba3e1da09232cb5e1e4e1ae934ef` |
| 2 | `2026-11-14T00:02:00.000000Z` | `{"amount":30,"from":"alice","to":"bob"}` | hash of 1 | `eb8c3c63480c7535ace675c2ac105695145452b49bed54bdbb2fe93c987d2c00` |

Expected tamper outcomes on that chain:

1. Block 1 `amount` 100 → 1000, hash untouched: `HASH_MISMATCH` at block 1 (recomputed `c6865260f6753728e27bcb8ab993c933cb1d8efcb5098933d5be5a92c5a6e9cb`). Nothing else.
2. Same change, but block 1 `hash` rewritten to `c6865260…e9cb`: block 1 passes; `LINK_BROKEN` at block 2.
3. Block 2 `amount` 30 → 300, hash untouched: `HASH_MISMATCH` and `INSUFFICIENT_BALANCE` at block 2.

## 7. Required tests (pytest)

`tests/core` (no server, no database):
- Reference vectors in §6 (hashes match exactly).
- Valid chain verifies with no findings.
- Tamper `data`, `hash`, `prev_hash`, `timestamp`.
- Delete a middle block (`ID_GAP` and `LINK_BROKEN`).
- Multiple corrupted blocks: all are reported, not only the first.
- Rejected transactions: insufficient balance, non-positive amount, `from == to`, empty account.
- Documented limitations behave as stated in §4 (last-block rewrite and truncation are not detected).

`tests/server` (temporary database, API test client):
- Add, show, verify and balance through the API; rejected transaction returns `422` and creates no block.
- Persistence: restart the app on the same database file; blocks remain and verify is valid.
- Genesis is created once and not duplicated on restart.
- Tamper the database file directly, then `GET /verify` reports the findings (proves there is no cache).
- Concurrent adds do not fork the chain.

## 8. Technical constraints

- Python 3.10+. Cross-platform (Windows, macOS, Linux); the owner develops on Windows with PowerShell.
- `core`: **standard library only** (`hashlib`, `json`, `datetime`, `typing`/`dataclasses`).
- `server`: **FastAPI + uvicorn**; persistence with stdlib `sqlite3`.
- `client`: **httpx** plus stdlib (`argparse`).
- Development: `pytest`.
- Runtime dependencies pinned in `requirements.txt`; development ones in `requirements-dev.txt`, both at the repo root.
- `Storage` interface defined in `core` (at least load and append), SQLite implementation in `server`, so another backend (Cloudflare D1, MongoDB) could be added later without touching `core`.

## 9. Architecture — ASTRA proposes (mandatory, before any implementation)

The top-level layout is **fixed** (the folders already exist):

```
CryptoCoin/
├── docs/
├── core/
├── server/
├── client/
├── tools/
├── tests/
│   ├── core/
│   └── server/
├── requirements.txt
├── requirements-dev.txt
└── .gitignore
```

ASTRA designs everything inside it and documents it in **`docs/ARQUITECTURA.md`** (Spanish), then **stops and waits for the owner's "ok"** before dispatching any implementation work. It must conform to this SPEC and include:

1. Internal module layout of each package, and how packages import each other on Windows (e.g., packaging with `pyproject.toml` or an editable install), justified.
2. Public interfaces: classes, function signatures, error types, `Storage` interface, API request/response models, verifier report format.
3. Module diagram (Mermaid) and sequence diagrams for add, verify and attack.
4. Key design decisions with alternatives considered and why they were discarded.
5. How atomic append, no-cache reads and `from` aliasing are implemented.
6. Run instructions (start server, run client, run tests, run attack demo) in PowerShell.
7. Risks and mitigations (e.g., serialization mismatch, partial writes, shared database file).
8. Appendix: per-module task briefs (max 40 lines each) derived from this SPEC. Each brief names the module document the worker must write or update (§10).

### Build order and gates

1. **Gate 1 — core:** all `tests/core` pass, including reference vectors; `docs/modulos/CORE.md` written.
2. **Gate 2 — server and tools:** all `tests/server` pass; the attack tool works against the server's database; `docs/modulos/SERVER.md`, `docs/modulos/API.md` and `docs/modulos/TOOLS.md` written.
3. **Gate 3 — client and end to end:** the demo sequence works: start server → client add → show → verify → attack tool → verify; `docs/modulos/CLIENT.md` written.

No phase starts before the previous gate is passed.

## 10. Documentation rules

- **All documentation lives in `docs/`.** No `.md` files outside `docs/`.
- Documents:
  - `docs/SPEC.md` — this file (English).
  - `docs/PRD.md` — product requirements for understanding and maintaining the project (Spanish). Owned by the owner; ASTRA proposes edits, never rewrites it.
  - `docs/ORQUESTADOR.md` — orchestrator prompt (English).
  - `docs/ARQUITECTURA.md` — architecture (Spanish), written by ASTRA.
  - `docs/README.md` — install, run, usage examples, tests, attack demo (Spanish).
  - `docs/INFORME.md` — technical report following the professor's paper template, 4–6 pages once exported to PDF (Spanish). A draft exists; update it with the real architecture, outputs and results.
  - `docs/EXPLICACION.md` — per-file and per-decision explanation for the oral defense, with 8 likely professor questions and answers (Spanish).
- **Module documentation (living docs).** Every component has its own document, created and updated **in the same task** that creates or changes the component. Spanish, max ~100 lines each, no copying of this SPEC (reference its sections instead):
  - `docs/modulos/CORE.md` — public interface of `core`, hashing and verifier explained, how to run its tests.
  - `docs/modulos/SERVER.md` — structure, configuration (environment variables), storage, how to start it, error handling.
  - `docs/modulos/API.md` — API reference: every endpoint with method, path, request, response, status codes, error codes and a PowerShell example (`Invoke-RestMethod` or `curl.exe`). Must match the live OpenAPI schema.
  - `docs/modulos/CLIENT.md` — commands and menu options, configuration, example sessions, error messages.
  - `docs/modulos/TOOLS.md` — attack simulator and reset options, the demo recipe, and which finding code each option triggers.
  - `docs/CHANGELOG.md` — one line per merged change: date, component, what changed.
- Each module document contains: purpose, files and responsibilities, public interface, configuration, how to run and test, examples, known limitations.
- Any new endpoint, command, option, error code or environment variable must appear in its document **before** the task is reported as done. If behavior changes, the same task updates the document and adds a CHANGELOG line.
- A gate (§9) does not pass unless the module's document exists and matches the code.
- Code comments and docstrings in Spanish. Clean, readable, consistent naming.
- Never invent bibliographic references.

## 11. Definition of done (maps to the 100-point rubric)

- [ ] Data structure clear, coherent and documented (20).
- [ ] Hashing and chaining correct, consistent, robust; reference vectors pass (25).
- [ ] Verification scans everything and distinguishes altered content from broken link (25).
- [ ] Persistence works across restarts; messages clear and easy to understand (10).
- [ ] Report complete, with real screenshots, limitations and improvements (15).
- [ ] Code commented and clean (5).
- [ ] All tests pass at the three gates; the end-to-end demo works.
- [ ] Every module document in `docs/modulos/` exists, matches the code, and each change is listed in `docs/CHANGELOG.md`.
- [ ] The owner can explain every file orally (`docs/EXPLICACION.md` reviewed).

## 12. Process constraints

- Workers receive only their brief + this SPEC + the files they must touch. They never read `docs/PRD.md`.
- Workers reply with files or diffs and a report of at most 5 lines.
- Maximum 2 fix rounds per module; on the third failure, escalate to ASTRA with the exact error.
- Do not refactor code whose tests pass.
- If git is available: one branch per module (`feat/core`, `feat/server`, `feat/client`, `feat/tools`), merged only after its gate passes.
- A task is not done until its module document (§10) is updated. The worker's report includes a `DOCS:` line.