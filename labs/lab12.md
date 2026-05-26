# Lab 12 — Bonus: WebAssembly Containers — A QuickNotes Endpoint on Spin

![difficulty](https://img.shields.io/badge/difficulty-advanced-red)
![topic](https://img.shields.io/badge/topic-WebAssembly-blue)
![points](https://img.shields.io/badge/points-4%2B4%2B2-orange)
![tech](https://img.shields.io/badge/tech-Spin%20SDK%20%2B%20TinyGo-informational)

> **Goal:** Build a single QuickNotes-style HTTP endpoint as a WebAssembly component using the **Spin Go SDK**, run it with `spin up`, and compare cold-start + size against the Lab 6 Docker container. Bonus: rebuild the same logic as a standalone WASI module and run it under `wasmtime` to compare the two WASM execution models.
> **Deliverable:** A PR from `feature/lab12` to the course repo with `wasm/` + `submissions/lab12.md`. Submit the PR link via Moodle.

> 🎁 **Bonus lab.** 10 pts total, structured as Task 1 (4) + Task 2 (4) + Bonus Task (2). This lab is the bonus; its full 10 pts count toward the bonus-labs grade weight.

> ⚠️ **WASM tooling moves fast — pin your versions.** This lab was validated in May 2026 with **Spin 3.4**, **TinyGo 0.41**, and the **`github.com/spinframework/spin-go-sdk/v2`** SDK (note: the SDK moved from `github.com/fermyon/spin/sdk/go/v2` to the CNCF `spinframework` org when Spin was donated to the CNCF). Older guides reference a **WAGI executor** (`executor = { type = "wagi" }`) — **that was removed in Spin 3.x.** If a tutorial tells you to use WAGI, it's out of date. Always scaffold from `spin new -t http-go` to get the current template for whatever Spin version you have.

---

## Overview

Read [Reading 12](../lectures/reading12.md) first. By the end of this lab:
- A Go HTTP handler compiled to a **WebAssembly component** via the Spin SDK + TinyGo
- `spin up` serves it at `http://127.0.0.1:3000/time` returning Moscow time as JSON
- A measured comparison: cold-start, size, warm latency — WASM/Spin vs the Lab 6 Docker container
- *(Bonus)* The same logic rebuilt as a standalone WASI module run under `wasmtime` — comparing Spin's persistent wasi-http server model with wasmtime's per-invocation CLI model

---

## Project State

**Starting point:** Lab 6 image works (the comparison baseline). QuickNotes source in `app/`.

**After this lab:** A `wasm/` directory with the Spin component + reproducible perf numbers.

---

## Prerequisites

- **Spin 3.x** (`spin --version`) — [developer.fermyon.com/spin](https://developer.fermyon.com/spin)
- **TinyGo 0.41+** (`tinygo version` — must say "using go version go1.24" or later)
- `hyperfine` (or `wrk`) for benchmarking
- *(Bonus)* `wasmtime` ([wasmtime.dev](https://wasmtime.dev/))

> 💡 **Check the compatibility matrix.** TinyGo's release notes list which Go versions each TinyGo supports. TinyGo 0.34 only supports Go ≤ 1.23; if your host Go is 1.24+ you need TinyGo 0.40+. Run `tinygo version` and confirm before you build.

---

## Task 1 — Build a WASM Endpoint with the Spin SDK (4 pts)

### 1.1: Scaffold from the current template

**Do not hand-write `spin.toml` from an old tutorial.** Scaffold the canonical layout for your Spin version:

```bash
mkdir -p wasm && cd wasm
spin new -t http-go moscow-time --accept-defaults
```

This generates `go.mod` (with the correct SDK import path for your Spin version), `spin.toml` (with the correct `tinygo build` command including `-buildmode=c-shared`), and a `main.go` stub.

### 1.2: Requirements for your handler

Edit the generated `main.go` so the handler:

1. Uses the **Spin SDK's `spinhttp.Handle(...)`** registration (the scaffold shows the import path — in May 2026 it's `github.com/spinframework/spin-go-sdk/v2/http`)
2. Responds to a `GET /time` request with a **JSON** body containing at least:
   - `unix` (epoch seconds)
   - `iso` (RFC3339 timestamp)
   - `hour_minute` (e.g. `"15:42"`)
   - the Moscow time (UTC+3)
3. Sets `Content-Type: application/json`

### 1.3: Requirements for `spin.toml`

- HTTP trigger routed at `/time`
- `allowed_outbound_hosts = []` (no outbound network — least privilege)
- The build command uses TinyGo's `wasip1` target with `-buildmode=c-shared` (the scaffold sets this — don't change it)

### 1.4: Design questions — answer in your submission

- a) **Browser WASM vs server WASM:** `go build -o m.wasm -target=js/wasm` vs `tinygo build -target=wasip1`. What's missing in the server target, and what do you gain?
- b) **Why does the build command need `-buildmode=c-shared`?** (Hint: what does the Spin host expect the module to export? Try removing it and see what `spin up` does.)
- c) **`allowed_outbound_hosts = []`** is the strictest setting. Explain the capability-based security model and compare it to Docker's `--network none`.
- d) **TinyGo stdlib gaps:** which part of upstream Go's stdlib does TinyGo *not* fully support that you hit during this lab? (Time-zone data and reflection-heavy `encoding/json` of `map[string]any` are common ones.)

### 1.5: Where to start

- 📖 [Spin — Building Spin components in Go](https://developer.fermyon.com/spin/v3/go-components)
- 📖 [Spin Go SDK (spinframework)](https://github.com/spinframework/spin-go-sdk)
- 📖 [TinyGo with WASI](https://tinygo.org/docs/guides/webassembly/wasi/)

### 1.6: Run + verify

```bash
spin build        # runs tinygo under the hood
spin up           # serves on :3000
# in another terminal:
curl -s http://127.0.0.1:3000/time | python3 -m json.tool
```

### 1.7: Document

In `submissions/lab12.md`:
- `main.go` + `spin.toml` (paste or link)
- `spin build` output (the wasm size)
- `curl` response showing valid Moscow-time JSON
- Design questions a-d answered

---

## Task 2 — Perf Comparison vs Lab 6 Container (4 pts)

### 2.1: Measurements

Boot both: `spin up` (Task 1) and the Lab 6 Docker container. Measure:

1. **Warm latency** — `hyperfine --warmup 5 --runs 50` against each `/time` (Spin) and `/health` (Docker). Capture p50.
2. **Cold start** — kill the runtime; restart; time to first successful response. ≥ 5 samples each.
3. **Artifact size** — `main.wasm` vs `quicknotes:lab6` image size.

### 2.2: Table

| Dimension              | Lab 6 Docker | Lab 12 WASM/Spin |
|------------------------|-------------:|-----------------:|
| Artifact size          |            ? |                ? |
| Cold start (p50)       |            ? |                ? |
| Warm latency p50       |            ? |                ? |
| Warm latency p95       |            ? |                ? |

### 2.3: Design questions

- e) **What dominates each platform's cold start?** (Container: image extract + namespace init. Spin: wasmtime instantiation + WASM module load.)
- f) **For what workloads is WASM clearly better, and where is Docker still right?** (See Reading 12's trade-offs table.)
- g) **Multi-tenant safety:** WASM's capability sandbox is stronger than Linux namespaces. What concrete attack does a WASM platform make harder?

### 2.4: Document

In `submissions/lab12.md`: the full perf table from your real measurements, your test rig (machine, OS), and design questions e-g answered.

---

## Bonus Task — Two WASM Execution Models (2 pts)

### B.1: Goal

The Spin component from Task 1 is a `wasi-http` component — it only runs inside a wasi-http host (Spin, or `wasmtime serve`). Rebuild the **same Moscow-time logic** as a **standalone WASI CLI module** (the older "CGI-over-WASM" / WAGI-shaped model: read request from env vars + stdin, write response to stdout) and run it under bare **`wasmtime run`**. Compare the two execution models.

### B.2: Requirements

1. In a second directory (`wasm-cli/`), write a `main()` (no Spin SDK) that reads `REQUEST_METHOD` / `PATH_INFO` from the environment and prints the same Moscow-time JSON to stdout
2. Build it: `tinygo build -o main.wasm -target=wasi -no-debug ./main.go`
3. Run it: `wasmtime run --env REQUEST_METHOD=GET --env PATH_INFO=/time main.wasm`
4. Compare cold-start (per-invocation `wasmtime run`) vs Spin's persistent server, and the two module sizes

### B.3: Design questions

- h) **Why can't the Task 1 Spin component run under bare `wasmtime run`?** (Hint: what does it export — a `_start` entrypoint, or a wasi-http handler?)
- i) **Spin uses wasmtime internally.** So what does Spin *add* on top of bare wasmtime? (Instance pooling, the wasi-http server loop, the manifest/routing layer, outbound-host policy.)
- j) **Two execution models — when does each fit?** Per-invocation `wasmtime run` (like CGI) vs Spin's persistent wasi-http server. List one workload that fits each.

### B.4: Document

In `submissions/lab12.md`: both build commands, both run commands, a size + cold-start comparison, and design questions h-j answered.

---

## How to Submit

1. `wasm/` directory in your fork with `main.go`, `go.mod`, `go.sum`, `spin.toml` (and `wasm-cli/` for the Bonus)
2. (Optional) `main.wasm` artifacts — small, you may commit them as evidence
3. `submissions/lab12.md` covers all attempted tasks
4. PR from `feature/lab12` → course repo's `main`
5. Submit the PR URL via Moodle

---

## Acceptance Criteria

### Task 1 (4 pts)
- ✅ Scaffolded from `spin new -t http-go` (correct SDK + build command for your Spin version)
- ✅ `spin build` produces `main.wasm`
- ✅ `spin up` serves `/time` returning Moscow-time JSON (HTTP 200, valid JSON)
- ✅ Design questions a-d answered

### Task 2 (4 pts)
- ✅ Full perf table with real numbers from your hardware
- ✅ Cold + warm + size captured
- ✅ Design questions e-g answered

### Bonus Task (2 pts)
- ✅ Same logic rebuilt as a standalone WASI CLI module, runs under `wasmtime run`
- ✅ Comparison of the two execution models (size, cold-start)
- ✅ Design questions h-j answered

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Spin SDK component serving /time | **4** | Scaffolded correctly, builds, `spin up` serves valid JSON, design questions |
| **Task 2** — Perf comparison vs Lab 6 | **4** | Real table, cold + warm + size, design questions |
| **Bonus** — Two WASM execution models | **2** | Standalone wasmtime module + comparison, design questions |
| **Total** | **10** | (bonus lab — contributes toward bonus-labs grade weight) |

> 📝 **Lab 12 itself is a bonus lab** — its full 10 pts go into the bonus-labs grade component (20% of the final grade; see the course [README](../README.md)).

---

## Common Pitfalls

- 🪤 **Following an old WAGI tutorial.** `executor = { type = "wagi" }` was REMOVED in Spin 3.x. `spin up` errors with `unknown field 'executor'`. Always scaffold from `spin new -t http-go`
- 🪤 **Wrong SDK import path.** Fermyon donated Spin to the CNCF; the SDK moved from `github.com/fermyon/spin/sdk/go/v2` to `github.com/spinframework/spin-go-sdk/v2`. The scaffold uses the current one
- 🪤 **Missing `-buildmode=c-shared`.** Without it the module doesn't export the handler Spin's host expects; `spin up` returns HTTP 500 with empty component logs
- 🪤 **TinyGo version too old.** TinyGo 0.34 supports only Go ≤ 1.23. With Go 1.24+ you need TinyGo 0.40+. Check `tinygo version`
- 🪤 **`time.LoadLocation("Europe/Moscow")` fails in TinyGo** — no embedded tzdata. Use `time.Now().UTC().Add(3 * time.Hour)` for Moscow (UTC+3), or embed tzdata
- 🪤 **`json.NewEncoder(w).Encode(map[string]any{...})` may fail under TinyGo** (reflection limits). Building the JSON string with `fmt.Sprintf` and `%q` verbs is more robust
- 🪤 **Spin component can't run under bare `wasmtime run`** — it's a wasi-http component, not a CLI module. That's the Bonus's whole point. Use `wasmtime serve` if you want wasmtime to host the wasi-http component, or build the separate CLI module for `wasmtime run`

---

## Guidelines

- Don't try to port *all* of QuickNotes to WASM — TinyGo stdlib gaps will frustrate you. One endpoint is the lab's scope
- Pin every tool version (Spin, TinyGo, SDK) — WASM tooling churns fast; an unpinned tutorial breaks in months
- Read [Reading 12](../lectures/reading12.md) first — its trade-offs table is the answer key for design questions f and j

---

## Resources

- 📖 [Spin — Go components](https://developer.fermyon.com/spin/v3/go-components)
- 📖 [Spin Go SDK (spinframework, CNCF)](https://github.com/spinframework/spin-go-sdk)
- 📖 [TinyGo with WASI](https://tinygo.org/docs/guides/webassembly/wasi/)
- 📖 [wasmtime docs](https://docs.wasmtime.dev/) — for the Bonus runtime
- 📖 [WASI documentation](https://wasi.dev/)
- 🎥 [Lin Clark — *A cartoon introduction to WebAssembly*](https://hacks.mozilla.org/2017/02/a-cartoon-intro-to-webassembly/)
- 🛠️ [`hyperfine`](https://github.com/sharkdp/hyperfine), [`wasmtime`](https://wasmtime.dev/)
