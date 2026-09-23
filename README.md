# Orbinum Validator Ops

Runbook and helper scripts for deploying and operating an **Orbinum Network**
testnet validator node, based on the official docs:
https://docs.orbinum.network/validators/running-a-validator

This repo does **not** replace `orbinum/node-deploy` (which holds the actual
Compose file and chain spec) — it wraps it with scripts and checklists so the
setup is repeatable and each step is verified before moving to the next one.

---

## Prerequisites

- A server meeting the [validator requirements](https://docs.orbinum.network/validators/requirements)
- Docker + the Compose plugin ([install guide](https://docs.orbinum.network/nodes/installation))
- No registry token needed — the validator image is public
- A funded validator account (fund it from the [faucet](https://docs.orbinum.network/getting-started/faucet) before step 6)

---

## Quick start

```bash
./scripts/01_clone_configure.sh      # clone node-deploy, copy .env.example
# → edit node-deploy/testnet/validator/.env (see table below)
./scripts/02_firewall.sh             # open 30333/tcp + 22/tcp only
./scripts/03_start_node.sh           # docker compose pull && up -d
./scripts/04_check_sync.sh           # poll system_health until isSyncing:false
./scripts/05_generate_session_keys.sh <YOUR_SS58_ADDRESS>
# → copy `keys` + `proof` from the output
# → submit session.setKeys(keys, proof) manually in Polkadot.js Apps (step 6, no CLI path — see below)
./scripts/07_verify_keystore.sh <KEYS_HEX_FROM_STEP_5>
```

Each script is a thin, checked wrapper around the exact commands in the docs —
read them before running if you want to know exactly what they do.

---

## Step-by-step

### 1. Clone `node-deploy` and configure

`scripts/01_clone_configure.sh` clones the repo and copies `.env.example` to
`.env`. You then edit `.env` yourself — **do not** script this part blindly,
since some values are security-sensitive (`VALIDATOR_NODE_KEY`) or
environment-specific (`METRICS_BIND`).

| Variable             | Value                                 | Why                                                          |
| --------------------- | -------------------------------------- | -------------------------------------------------------------- |
| `VALIDATOR_NAME`     | anything identifiable                 | Shown on telemetry                                            |
| `VALIDATOR_NODE_KEY` | `openssl rand -hex 32`                | Your stable libp2p identity                                   |
| `TELEMETRY_URL`      | leave the default                     | Level `1` publishes your validator address to the dashboard   |
| `METRICS_BIND`       | your private IP, or `127.0.0.1`       | Default fails closed and exposes nothing                      |
| `PUBLIC_ADDR`        | **leave empty**                       | Node auto-detects its public address                          |
| `RESERVED_NODES`     | **leave empty**                       | Not additive — a partial list isolates the node               |
| `SYNC_MODE`          | **leave empty** unless bootstrapping  | Full sync is the safe default (warp sync only on a fresh volume) |

No bootnodes to add — `testnet-spec.json` already ships with them and the
Compose file already mounts it.

### 2. Open the firewall

`scripts/02_firewall.sh` opens **only**:
- `30333/tcp` — P2P, must be reachable from the internet
- `22/tcp` — SSH

Explicitly does **not** open `9944` (RPC) or `9615` (metrics) — those stay
private per the docs.

### 3. Start the node

`scripts/03_start_node.sh` runs `docker compose pull`, `up -d`, then tails
logs. Watch for the peer identity at startup, then import messages, until the
node reports idling at the chain head.

If you edit `.env` later, you must **recreate**, not restart:
```bash
docker compose up -d --force-recreate orbinum-validator
```

### 4. Confirm it is synced

`scripts/04_check_sync.sh` polls `system_health` via `docker exec` (the RPC
port is only ever reachable at `9944` *inside* the container) until
`isSyncing` is `false` and `peers >= 2`.

`peers: 0` → check that `30333/tcp` is genuinely reachable from outside and
that `RESERVED_NODES` is empty.

### 5. Generate session keys

```bash
./scripts/05_generate_session_keys.sh <YOUR_SS58_ADDRESS>
```

Looks up your account's hex public key, then calls
`author_rotateKeysWithOwner` bound to that account. Must be run **once, on a
synced node** — running it again generates a new pair and invalidates
anything already submitted on-chain.

Output has two fields you need for step 6:
- `keys` — 128 hex chars: Aura (sr25519) + GRANDPA (ed25519) pubkeys, concatenated
- `proof` — 256 hex chars: one signature per key over your account id

Private key halves stay in the node's keystore
(`/data/chains/orbinum_testnet/keystore`) and never leave the server.

### 6. Submit `session.setKeys` (manual — Polkadot.js Apps)

This is an on-chain extrinsic, not a CLI step, and must be signed by your
validator account (the same one used as `owner` in step 5). Fund it from the
[faucet](https://docs.orbinum.network/getting-started/faucet) first.

1. Open https://polkadot.js.org/apps/ and connect to `wss://rpc-1.testnet.orbinum.io`
2. **Developer → Extrinsics**
3. Select your validator account → `session` → `setKeys(keys, proof)`
4. Paste `keys` and `proof` from step 5
5. Submit and sign

**Verify:** **Developer → Chain state** → `session.nextKeys(yourAccount)` must
return the same `keys` you submitted. If empty, the extrinsic didn't finalize
— step 7 (and `addValidator`) will fail until it does.

> Your validator account is an ordinary account and needs no relationship to
> your Aura key — that only held for genesis validators.

### 7. Confirm the keystore matches the chain

```bash
./scripts/07_verify_keystore.sh <KEYS_HEX_FROM_STEP_5>
```

Calls `author_hasSessionKeys`. `true` = this node can sign for the on-chain
keys, done. `false` = keystore/chain mismatch (e.g. keys were rotated on a
different machine, or the container volume was recreated) — go back to step 5
**on this node** and resubmit `setKeys`.

> **`false` is silent** — no log or telemetry warning. The node just skips
> every slot it's scheduled for. Always check this before considering setup
> complete.

---

## After this repo: applying to the validator set

Once `session.nextKeys` returns your keys and `author_hasSessionKeys` returns
`true`, you're in the state `addValidator` requires. Next steps (not covered
here):
- [Apply to Join the Set](https://docs.orbinum.network/validators/apply)
- [Relay Setup and Rewards](https://docs.orbinum.network/validators/relay-setup)

---

## Notes / gotchas from the docs

- `docker exec` is used for every RPC call because inside the container the
  RPC port is always `9944`, regardless of what `RPC_PORT` is set to on the
  host — sidesteps a common source of "connection refused" confusion.
- The binary forces `--rpc-methods Unsafe` on every role; the RPC port must
  never be exposed to the internet (Compose only publishes it on loopback).
- `Session.InvalidProof` on `setKeys` means one of: proof is empty/`0x00`,
  the signing account isn't the `owner` passed in step 5, or you rotated keys
  again after copying — the fix is always: redo step 5, resubmit step 6.

## Repo layout

```
orbinum-validator-ops/
├── README.md
└── scripts/
    ├── 01_clone_configure.sh
    ├── 02_firewall.sh
    ├── 03_start_node.sh
    ├── 04_check_sync.sh
    ├── 05_generate_session_keys.sh
    └── 07_verify_keystore.sh
```

(Step 6 has no script on purpose — it's a manual, signed on-chain action.)
