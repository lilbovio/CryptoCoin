# SPEC.md — CryptoCoin (hash-chained database / mini-blockchain)

> Single source of truth for all agents. Language: English. If anything here conflicts with a worker's assumption, this file wins. Only ASTRA may change it, and only with the owner's approval.

## 1. Project

- **Course:** IH063 Cryptography — final project "Base de datos encadenada con hash (mini-blockchain)".
- **Theme:** a basic cryptocurrency ledger called **CryptoCoin** (a record of transfers).
- **Goal:** each record stores a hash that depends on the previous record. If any past record is modified, the chain breaks and the application detects it.
- **Out of scope (do NOT build):** consensus, mining / proof of work, P2P networking, wallets, digital signatures, web UI, external databases (MongoDB, D1), any runtime dependency outside the Python standard library.

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

### 2.2 Genesis block

- `id = 0`, `prev_hash = "0" * 64`, `data = {"from": "GENESIS", "to": "GENESIS", "amount": 0}`.
- Created automatically when the database is empty.

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
| F4 | **Persist** to SQLite: after closing and reopening, blocks remain and the chain can be verified |
| F5 | **Simulate an attack** (see §5) and then verify to show the corruption |
| F6 | **Show balance** of an account (supports the balance rule and the demo) |

### Interface

- Interactive menu (required): add, show, verify, simulate attack, balance, exit. Clear messages with ✅ / ❌.
- Non-interactive subcommands (recommended, makes tests and demo easier). ASTRA decides in the architecture.
- All user-facing messages in **Spanish**.
- Database path configurable; default `chain.db`. Database files are never committed (`.gitignore`).

## 4. Verification algorithm (F3)

Scan **all** blocks in order. **Never stop at the first error.** For each block `b` collect findings:

| Code | Condition |
|---|---|
| `GENESIS_INVALID` | Block 0 does not match §2.2 |
| `ID_GAP` | `b.id` is not previous `id` + 1 |
| `MALFORMED` | Missing field, wrong type or wrong format |
| `HASH_MISMATCH` | Recomputed hash ≠ stored `b.hash` (content altered) |
| `LINK_BROKEN` | `b.prev_hash` ≠ **stored** `hash` of the previous block |
| `INSUFFICIENT_BALANCE` | Replaying balances over the stored data, the sender lacks funds (semantic check, secondary) |

Output: ✅ valid chain with block count, or ❌ list of findings (block `id`, code, expected vs found). Exit code 0 if valid, 1 if not. A tampered block is reported as `HASH_MISMATCH` (content altered) or `LINK_BROKEN` on the next block (hash rewritten), clearly distinguished.

Known and accepted limitation (must appear in the report): an attacker who rewrites **all** subsequent hashes consistently is not detected, because there is no external anchor, signature or consensus.

## 5. Attack simulation (F5)

- Command/menu option that **modifies a stored record directly in the database, bypassing application logic** (raw update), e.g. change `amount` of block N.
- Must let the user choose the block `id` and the new value. Must not alter `hash`, unless a "rewrite hash too" option is selected, to demonstrate `LINK_BROKEN`.
- Provide a way to reset or restore the demo database (e.g., `reset`) so the demo is repeatable.

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
3. Block 2 `amount` 30 → 300, hash untouched: `HASH_MISMATCH` at block 2 and `INSUFFICIENT_BALANCE` at block 2.

## 7. Required tests (pytest)

- Reference vectors in §6 (hashes match exactly).
- Valid chain verifies with no findings.
- Tamper `data`, tamper `hash`, tamper `prev_hash`, tamper `timestamp`.
- Delete a middle block (`ID_GAP` and `LINK_BROKEN`).
- Multiple corrupted blocks: all are reported, not only the first.
- Persistence: add blocks, reopen the database, verify still valid.
- Rejected transactions: insufficient balance, non-positive amount, `from == to`; no block created.
- Genesis created once; it is not duplicated when reopening.

## 8. Technical constraints

- Python 3.10+, **standard library only** at runtime (`hashlib`, `json`, `sqlite3`, `argparse`, `datetime`). `pytest` allowed for development only.
- Cross-platform (Windows, macOS, Linux). No platform-specific paths or shell scripts required to run.
- Persistence behind a `Storage` interface (at least `load()` and `save(block)`), so another backend (e.g., Cloudflare D1, MongoDB) could be added later without touching the core.

## 9. Architecture — ASTRA proposes (mandatory, before any implementation)

ASTRA must design the architecture and document it in **`docs/ARQUITECTURA.md`** (Spanish), then **stop and wait for the owner's "ok"** before dispatching any implementation work. It must conform to this SPEC. It must include:

1. Layers and responsibilities (e.g., core / storage / interface), with the rationale for each.
2. Module diagram (Mermaid) and data flow for add, verify and attack.
3. Repository folder structure. Code, tests and scripts go outside `docs/`; the exact layout is ASTRA's proposal.
4. Public interfaces: classes, function signatures, error types, verifier report format.
5. Key design decisions with alternatives considered and why they were discarded.
6. Per-module task briefs (max 40 lines each) for the workers, derived from this SPEC.
7. Risks and how the design mitigates them (e.g., serialization mismatch, partial writes).

## 10. Documentation rules

- **All documentation lives in `docs/`.** No `.md` files outside `docs/`.
- Required documents:
  - `docs/SPEC.md` — this file (English).
  - `docs/ARQUITECTURA.md` — architecture (Spanish), written by ASTRA.
  - `docs/README.md` — install, run, usage examples, how to run tests, how to run the attack demo (Spanish).
  - `docs/INFORME.md` — technical report following the professor's paper template, 4–6 pages once exported to PDF (Spanish). A draft already exists; update it with the real architecture, outputs and results.
  - `docs/EXPLICACION.md` — per-file and per-decision explanation for the oral defense, with 8 likely professor questions and answers (Spanish).
- Code comments and docstrings in Spanish. Clean, readable, consistent naming.
- Never invent bibliographic references.

## 11. Definition of done (maps to the 100-point rubric)

- [ ] Data structure clear, coherent and documented (20).
- [ ] Hashing and chaining correct, consistent, robust; reference vectors pass (25).
- [ ] Verification scans everything and distinguishes altered content from broken link (25).
- [ ] Persistence works across restarts; messages clear and easy to understand (10).
- [ ] Report complete, with real screenshots, limitations and improvements (15).
- [ ] Code commented and clean (5).
- [ ] All tests pass; the demo sequence add → show → verify → attack → verify works end to end.
- [ ] The owner can explain every file orally (`docs/EXPLICACION.md` reviewed).

## 12. Process constraints

- Workers receive only their brief + this SPEC + the files they must touch.
- Workers reply with files or diffs and a report of at most 5 lines.
- Maximum 2 fix rounds per module; on the third failure, escalate to ASTRA with the exact error.
- Do not refactor code whose tests pass.