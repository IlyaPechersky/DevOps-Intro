# 📖 Reading 12 (Bonus) — WebAssembly Containers: A Different Kind of Portability

> 🎁 **This is a bonus reading**, paired with Lab 12. Optional, for students chasing the frontier of portable runtimes.

---

## 1. The Problem WebAssembly Solves (on the Server)

Containers solved "**ships across machines**". But they did *not* solve:

* 🐌 **Cold-start time** — a Go container starts in ~1-2 s. A WASM module starts in **~1 ms**
* 📦 **Size** — even with multi-stage, a Go container is ~15 MB. A WASM module is **~1-3 MB**
* 🏛️ **Isolation** — containers share the host kernel (Lecture 5). WASM modules run in a **sandbox with no system calls** by default — stronger isolation than containers, lighter than VMs
* 🌐 **CPU portability** — `linux/amd64` vs `linux/arm64` vs Apple Silicon. One WASM module runs on **all of them**, unchanged

For high-fan-out, latency-sensitive workloads (edge functions, plugin systems, multi-tenant SaaS), these wins matter.

> 🤔 **Think:** Cloud Run cold-started QuickNotes in ~1.5 s in Lab 10. What workloads would benefit from booting in 1 ms instead?

---

## 2. A Compressed History

* 🌐 **2015** — Mozilla, Google, Microsoft, Apple announce **WebAssembly** as a successor to asm.js
* 📜 **March 2017** — WebAssembly 1.0 specification published
* 🪟 **2017-2019** — Adopted by **all four major browsers** as a stable feature
* 🖥️ **2019** — **WASI** (WebAssembly System Interface) proposed — a POSIX-like ABI for running WASM **outside** the browser (CLI tools, server-side modules)
* 🚀 **2020** — Fermyon founded; **Spin** framework launches the "WASM containers" pattern
* 🧩 **2022-2023** — containerd gains **WASM shims**; Kubernetes can schedule a `crun-wasmtime` "container" that's actually a WASM module
* 🌍 **2024-2026** — Cloudflare Workers, Fastly Compute@Edge, Fermyon Cloud — mainstream edge-compute platforms ship WASM-first

---

## 3. WASM 101

```mermaid
graph LR
    S["📜 source<br/>Go / Rust / C / AssemblyScript"] -- "compile" --> W["📦 .wasm binary<br/>(stack-based VM bytecode)"]
    W -- "run" --> R1["🌐 Browser"]
    W -- "run" --> R2["🖥️ wasmtime CLI"]
    W -- "run" --> R3["🐳 containerd-wasm"]
    W -- "run" --> R4["☁️ Spin / Cloudflare Worker"]
```

* 🧠 WebAssembly is a **stack-based virtual machine instruction set** — like a portable, compact CPU
* 🔒 **Capability-based sandbox**: by default no filesystem, no network, no clock. You grant *only* what's needed
* 📦 Multiple source languages compile to it: Rust (best support), C/C++, Go (via TinyGo), AssemblyScript

```text
.wasm = bytecode + module imports/exports + type signatures
```

---

## 4. WASI: WebAssembly System Interface

In the browser, WASM accesses the world through JavaScript bindings. On the server, **WASI** is the standard ABI:

| WASI API | What it gives |
|----------|---------------|
| `wasi:cli/stdio` | stdin/stdout/stderr |
| `wasi:filesystem` | Pre-opened directory handles only (no `/`) |
| `wasi:sockets` | TCP/UDP (limited) |
| `wasi:clocks` | Monotonic + wall clocks |
| `wasi:random` | Entropy source |

* 🔒 **Capability model**: the runtime mounts a directory like `--dir=.::./data` — the module sees `./data` but **cannot** see `/etc/passwd` or `/`
* 🪶 No syscalls smuggled in — every interaction with the OS is an explicit WASI import

---

## 5. Spin: The "WASM Containers" Pattern

[Spin](https://www.fermyon.com/spin) (created by Fermyon, donated to the **CNCF** in 2024 → SDK now lives under the `spinframework` org) is the easiest entry to server-side WASM. You write a normal-looking Go HTTP handler; the Spin SDK adapts it to a `wasi-http` component:

```go
// main.go — Spin Go SDK
package main

import (
    "fmt"
    "net/http"
    spinhttp "github.com/spinframework/spin-go-sdk/v2/http"
)

func init() {
    spinhttp.Handle(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        fmt.Fprintln(w, `{"status":"ok"}`)
    })
}

func main() {}
```

```toml
# spin.toml
spin_manifest_version = 2
[application]
name = "quicknotes-wasm"
version = "0.1.0"
[[trigger.http]]
route = "/time"
component = "moscow-time"
[component.moscow-time]
source = "main.wasm"
allowed_outbound_hosts = []
[component.moscow-time.build]
command = "tinygo build -target=wasip1 -buildmode=c-shared -no-debug -o main.wasm ."
```

```bash
$ spin new -t http-go moscow-time   # scaffold the CURRENT template
$ spin build                        # runs tinygo under the hood
$ spin up                           # serves on :3000
$ spin deploy                       # → Fermyon Cloud (free tier)
```

* ⚡ Spin instantiates a fresh WASM instance per request — true serverless. Cold starts in **single-digit milliseconds**
* 🌐 Each component carries its own capability list — it cannot reach code or hosts it wasn't granted
* 🔑 **`-buildmode=c-shared`** is required: it makes TinyGo export the handler symbols the Spin host calls (omit it → HTTP 500)

> ⚠️ **Tooling churn warning.** WASM server tooling moves fast. The SDK import path changed from `github.com/fermyon/spin/sdk/go/v2` to `github.com/spinframework/spin-go-sdk/v2` with the CNCF donation. **Always `spin new -t http-go` to get the current template** rather than copy-pasting an old `spin.toml`.

---

## 6. The Older Model: WAGI ("CGI for WASM") — Deprecated

Before the `wasi-http` Component Model matured, Spin offered a **WAGI** executor that mapped HTTP onto stdin/stdout, CGI-style: the module read request info from environment variables, wrote the response to stdout. It was the simplest possible Go-to-WASM port.

```toml
# ❌ DEPRECATED — removed in Spin 3.x
[component.quicknotes]
source = "main.wasm"
executor = { type = "wagi" }      # `spin up` rejects this field in 3.x
```

* 🪦 **Spin 3.x removed the inline WAGI executor.** Modern Spin uses the `wasi-http` component model (Section 5)
* 🧰 The WAGI *pattern* still lives on in bare `wasmtime run`: a standalone WASI module reads env + stdin, writes stdout. Lab 12's Bonus uses exactly this to contrast the two execution models — Spin's persistent `wasi-http` server vs wasmtime's per-invocation CLI

---

## 7. Compared with Traditional Containers

| Dimension | Docker container (Lab 6) | WASM module (Lab 12) |
|-----------|--------------------------|----------------------|
| Cold start | ~200 ms – 2 s | ~1 ms |
| Size | 15-200 MB | 1-3 MB |
| CPU portability | Per-arch image | Single artifact |
| Isolation | Linux kernel namespaces | Capability sandbox (stronger than namespaces, weaker than VM) |
| Mature ecosystem | ✅ massive | 🟡 growing fast, still rough edges |
| Multi-tenant safety | OK | Excellent |
| Long-running, stateful workloads | ✅ | ⚠️ (single-process model) |
| `apt install` arbitrary OS deps | ✅ | ❌ (no shell, no syscalls beyond WASI) |

* 🎯 **Sweet spot for WASM**: edge, plugin systems, multi-tenant SaaS request handlers, IoT, browser companion code
* 🪤 **Bad fit for WASM (today)**: heavy DB clients, persistent processes, anything needing arbitrary syscalls

---

## 8. WASM in Kubernetes: containerd Shims

```mermaid
graph TB
    K["☸️ Kubernetes Pod"] --> CR["🐳 containerd"]
    CR --> R1["🥪 runc (Linux containers)"]
    CR --> R2["🧪 containerd-shim-wasmtime<br/>or -wasmedge -spin"]
    R2 --> W["📦 .wasm artifact"]
```

* 🧩 A `RuntimeClass: wasmtime` (Kubernetes feature) lets a pod's containers run as **WASM modules** instead of Linux containers
* 🌐 Major cloud K8s providers (GKE, AKS, EKS) support this in preview as of 2024-2025
* 🎁 **K8s scheduling + WASM runtime** = put microservice plugins next to their users at the edge, with strong isolation

---

## 9. The Cloudflare / Fastly / Vercel Edge Model

These platforms run WASM modules at **300+ POPs** worldwide:

* 🌍 Your request hits the **nearest** POP (typically <30 ms RTT)
* ⚡ A fresh WASM instance starts per request — measured in **microseconds**
* 💸 Pricing is per-request, not per-instance-hour — perfect for spiky traffic
* 🛡️ Each tenant's WASM is sandboxed from every other tenant's — multi-tenant by design

* 🎯 *This* is where WASM-on-server has decisively won — edge platforms, where Cold-Start latency dominates the user experience

---

## 10. The Honest Trade-offs (2026)

| ✅ WASM wins | ⚠️ WASM struggles |
|------------|-------------------|
| Cold start, size, portability | Library ecosystem (esp. databases) |
| Multi-tenant isolation | Long-running stateful workloads |
| Browser + server with same artifact | Debugging tools (still maturing) |
| Sandbox is "deny by default" | TinyGo (the Go-to-WASM compiler) doesn't support all of Go's stdlib |
| Standard ABI (WASI) growing | Cgo, reflection, large goroutine fleets |

> 💡 The Go-specific gotcha: **standard `go build -o main.wasm` produces a WASM module that runs only in browsers** (with the Go runtime + JS glue). For server-side WASM you typically want **TinyGo** (`tinygo build -target=wasi`), which produces a smaller, WASI-compliant binary with a stripped-down stdlib.

---

## 11. Lab 12 Preview

Lab 12 is the **WASM bonus lab** — itself worth 10 pts, structured 4+4+2:

* 🔨 **Task 1 (4 pts):** Build a minimal `time` endpoint (returns current Moscow time as JSON) in Go → WASM via TinyGo → packaged as a Spin SDK `wasi-http` component, served by `spin up`. Compare deployed size vs the QuickNotes Docker image
* 🏎️ **Task 2 (4 pts):** Benchmark request latency: warm Lab 6 Docker container vs warm `spin up`. Then compare cold-starts. Explain what dominates each curve
* 🎁 **Bonus (2 pts):** Rebuild the same logic as a standalone WASI CLI module, run under bare `wasmtime run`, and contrast the two execution models (Spin's persistent `wasi-http` server vs wasmtime's per-invocation CLI)
* 📜 Deliverable: `submissions/lab12.md` — spin.toml, build sizes, perf numbers, written reflection

---

## 12. Resources

* 📕 *WebAssembly: The Definitive Guide* — Brian Sletten (O'Reilly, 2022)
* 📗 [WebAssembly Spec](https://webassembly.github.io/spec/) — the actual spec; surprisingly readable
* 📘 [WASI documentation](https://wasi.dev/) — capability-based system interface
* 📗 [Fermyon Spin docs](https://developer.fermyon.com/spin) — quickest start to server-side WASM
* 🎥 [Lin Clark — *A cartoon introduction to WebAssembly*](https://hacks.mozilla.org/2017/02/a-cartoon-intro-to-webassembly/) — the canonical visual primer
* 📝 [Cloudflare Workers — WASM at the edge](https://blog.cloudflare.com/webassembly-on-cloudflare-workers/) — production case study
* 📝 [WASI Preview 2 update (2024)](https://bytecodealliance.org/articles/wasi-preview-2-launch) — the latest milestone

> 🎯 **Remember:** Containers won the 2014-2020 era by making "ship anywhere on Linux" trivial. WebAssembly is making a credible bid for the 2025-2030 era by adding **"ship to any CPU, with capability-based safety, in 1 millisecond"**. Whether it dethrones containers or coexists with them is the open question of the decade.
