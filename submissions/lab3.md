# Lab 3 Submission

Path picked: GitHub Actions. My fork and previous labs are on GitHub, so this path keeps the workflow close to the normal PR flow.

## Task 1 - PR Gate

CI config: `.github/workflows/ci.yml`

The workflow runs on:

- pushes to `main`
- pull requests to `main`
- only when `app/**` or `.github/workflows/ci.yml` changes

Jobs:

- `vet (go 1.23)` and `vet (go 1.24)` -> `go vet ./...`
- `test (go 1.23)` and `test (go 1.24)` -> `go test -race -count=1 ./...`
- `lint` -> `golangci-lint run`, pinned to `v2.5.0`

The runner is pinned to `ubuntu-24.04`. GitHub Actions are SHA-pinned:

```text
actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
actions/setup-go@d35c59abb061a4a6fb18e82ac0862c26744d6ab5 # v5.5.0
actions/cache@5a3ec84eff668545956fd18022155c47e93e2684 # v4.2.3
```

`permissions:` is set to least privilege:

```yaml
permissions:
  contents: read
```

Green run after the fix:

```text
https://github.com/IlyaPechersky/DevOps-Intro/actions/runs/27637608480

lint          pass  1m14s
test (go 1.23) pass 28s
test (go 1.24) pass 11s
vet (go 1.23)  pass 9s
vet (go 1.24)  pass 24s
```

The PR used to prove the gate on my fork:

```text
https://github.com/IlyaPechersky/DevOps-Intro/pull/1
```

Branch protection on `main` requires these checks:

```text
lint
test (go 1.23)
test (go 1.24)
vet (go 1.23)
vet (go 1.24)
```

It also requires branches to be up to date, PR review, signed commits, and linear history.

### Deliberate failure and fix

Failing run:

```text
https://github.com/IlyaPechersky/DevOps-Intro/actions/runs/27636570148

test (go 1.23) failed
test (go 1.24) failed
vet jobs passed
lint passed
```

Fail commit:

```text
2713fec test(lab3): demonstrate failing CI gate
```

I changed `TestHealth_ReportsCount` to expect 2 notes instead of 1. Both matrix test jobs failed.

Fix commit:

```text
f98f3a3 test(lab3): restore passing health test
```

After the fix, the same gate passed.

### Design answers

a) I pinned `ubuntu-24.04` instead of `ubuntu-latest` because `latest` moves. A new image can change preinstalled tools, shell behavior, or library versions and break a pipeline without any repository change.

b) Splitting vet, test, and lint shows exactly what failed. A combined job is simpler, but one red mark hides whether the issue was static analysis, tests, or style.

c) SHA pinning protects against action supply-chain attacks. The March 2025 `tj-actions/changed-files` incident showed that a trusted action tag can be moved or compromised and leak secrets from CI.

d) `permissions:` controls the default `GITHUB_TOKEN` scope. I set `contents: read` because CI only needs to read the repository; this follows least privilege.

e) In GitLab, a stage is an ordering group and a job is the actual unit that runs commands. `dependencies:` controls which artifacts a job downloads from earlier jobs; it does not define execution order by itself.

## Task 2 - Cache, Matrix, Path Filter

Optimizations applied:

- `actions/setup-go` cache is enabled with `app/go.mod` as the dependency key.
- `vet` and `test` run as a matrix on Go `1.23` and `1.24`.
- `fail-fast: false` keeps all matrix cells running even if one fails.
- `pull_request.paths` and `push.paths` skip docs-only changes.

Docs-only PR that skipped CI:

```text
https://github.com/IlyaPechersky/DevOps-Intro/pull/2
status checks reported: 0
```

Timing table:

| Scenario | Wall-clock |
|----------|-----------:|
| Baseline: first green run, cache not useful yet | 1m05s |
| With cache key fixed | 1m20s |
| With cache + matrix + final tuning | 1m14s |

The wall-clock is dominated by the slowest job (`lint`). Vet and test finish earlier, so running them in parallel keeps the total close to lint time.

### Design answers

f) The cache key should follow deterministic inputs. `go.mod` / `go.sum` describe dependencies, while build outputs can depend on OS, Go version, flags, and paths.

g) `fail-fast: false` lets every matrix cell finish, so I can see whether Go 1.23 and 1.24 behave differently. `fail-fast: true` is useful for expensive pipelines where one failure already proves the PR is blocked.

h) A malicious PR could try to write poisoned cache entries that later trusted branches restore. GitHub mitigates this with cache scoping: PRs can read base caches, but caches created by untrusted PR refs are isolated from protected branches.

## Bonus - Performance Investigation

Additional optimizations beyond Task 2:

| Optimization applied | Before | After | Saving |
|----------------------|-------:|------:|-------:|
| Run vet/test/lint in parallel jobs | about 2m+ sequential | about 1m14s | about -45s |
| Cache `golangci-lint` binary | lint installs every run | install skipped on cache hit | variable |
| `GOFLAGS=-buildvcs=false` | Go may inspect VCS metadata | VCS stamping skipped | small |

Final full run:

```text
https://github.com/IlyaPechersky/DevOps-Intro/actions/runs/27637608480
Total wall-clock: about 1m14s
```

Remaining bottleneck is lint, mostly because the runner installs and starts tooling before doing the actual lint work. QuickNotes itself is tiny, so changing application code would not save much time. The bigger win would be a prebuilt image or a warmer runner with the linter already available. I would stop optimizing around one minute, because the pipeline is already short enough for normal PR review.

## Submission PR

Upstream PR:

```text
https://github.com/inno-devops-labs/DevOps-Intro/pull/1075
```
