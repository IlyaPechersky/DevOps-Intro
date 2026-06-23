# Lab 6 Submission

Host used for the run: Ubuntu 24.04.3 LTS x86_64 cloud server with Docker 29.4.2 and Compose v5.1.3.

## Task 1 - Multi-Stage Dockerfile

`app/Dockerfile` is committed in the repository. Key points:

- Builder: `golang:1.24-alpine`
- Runtime: `gcr.io/distroless/static-debian12:nonroot`
- Static build with `CGO_ENABLED=0`, `-trimpath`, `-ldflags='-s -w'`
- `USER nonroot:nonroot`, `EXPOSE 8080`, exec-form `ENTRYPOINT`
- `go.mod` copied before source for layer caching
- A tiny static `/healthcheck` binary is built in the builder stage for HTTP probes
- `/data` is seeded in the image with UID 65532 so named volumes inherit writable ownership

Build and image size:

```text
$ docker build -t quicknotes:lab6 ./app
...
$ docker images quicknotes:lab6
IMAGE             ID             DISK USAGE   CONTENT SIZE   EXTRA
quicknotes:lab6   f63a1637227a       22.5MB          5.6MB

$ docker images golang:1.24-alpine
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
golang:1.24-alpine   8bee1901f1e5        395MB         83.5MB
```

`docker inspect` excerpt:

```text
User=nonroot:nonroot
ExposedPorts={"8080/tcp":{}}
Entrypoint=["/quicknotes"]
Healthcheck={"Test":["CMD","/healthcheck"],"Interval":10000000000,"Timeout":3000000000,"StartPeriod":5000000000,"Retries":3}
```

Runtime verification:

```text
$ docker run -d --name qn-lab6-run -p 18081:8080 \
  -e ADDR=0.0.0.0:8080 -e DATA_PATH=/data/notes.json -e SEED_PATH=/seed.json \
  -v qn-lab6-run-data:/data quicknotes:lab6
$ curl -s http://127.0.0.1:18081/health
{"notes":4,"status":"ok"}
```

### Layer-cache comparison

Bad order (`COPY . .` before `go mod download`), rebuild after touching one source file:

```text
#10 [builder 4/5] RUN go mod download
#12 [builder 5/5] RUN CGO_ENABLED=0 ... go build ...
real 0.72
```

Good order (`COPY go.mod` + `go mod download` before `COPY . .`), rebuild after the same change:

```text
#20 [builder 5/9] RUN go mod download
#20 CACHED
#17 [builder 7/9] RUN CGO_ENABLED=0 ... go build ...
real 1.30
```

The bad Dockerfile invalidates dependency download on every source edit. The good Dockerfile keeps `go mod download` cached and only rebuilds the binary layer.

### Design answers

a) Layer order matters because Docker caches each instruction. Putting `COPY . .` before `go mod download` ties the dependency layer to every source change, so `go mod download` reruns even when `go.mod` is unchanged. Copying `go.mod` first keeps dependency download cached across normal code edits.

b) `CGO_ENABLED=0` produces a fully static binary with no libc dependency. Distroless static has no dynamic linker; a CGO-enabled binary fails at startup with errors like `no such file or directory` when the loader is missing.

c) `gcr.io/distroless/static:nonroot` is a minimal runtime image with just the app, CA certs, `/etc/passwd`, and timezone data. It has no shell, package manager, or extra utilities. Fewer packages means a much smaller attack surface and fewer OS-level CVEs to patch.

d) `-s -w` strips the symbol table and DWARF debug info to shrink the binary. `-trimpath` removes local filesystem paths from the binary for reproducible builds. The cost is harder post-mortem debugging from the stripped binary alone.

## Task 2 - Compose + Healthcheck + Persistent Volume

Root `compose.yaml`:

```yaml
services:
  quicknotes:
    build:
      context: ./app
      dockerfile: Dockerfile
    image: quicknotes:lab6
    ports:
      - "8080:8080"
    environment:
      ADDR: "0.0.0.0:8080"
      DATA_PATH: "/data/notes.json"
      SEED_PATH: "/seed.json"
    volumes:
      - quicknotes-data:/data
    restart: unless-stopped
    cap_drop:
      - ALL
    read_only: true
    tmpfs:
      - /tmp
    security_opt:
      - no-new-privileges:true
    healthcheck:
      test: ["CMD", "/healthcheck"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 5s

volumes:
  quicknotes-data:
```

Persistence test:

```text
$ docker compose up --build -d
$ curl -s http://127.0.0.1:8080/health
{"notes":4,"status":"ok"}

$ curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"title":"durable","body":"survive a restart"}' \
  http://127.0.0.1:8080/notes
{"id":5,"title":"durable","body":"survive a restart","created_at":"2026-06-23T20:27:39.237347063Z"}

--- after POST ---
$ curl -s http://127.0.0.1:8080/notes | grep durable
..."title":"durable"...

$ docker compose down
$ docker compose up -d

--- after down/up ---
$ curl -s http://127.0.0.1:8080/notes | grep durable
..."title":"durable"...

$ docker compose down -v
$ docker compose up -d

--- after down -v ---
durable gone (expected)
```

Compose status after startup:

```text
NAME                       IMAGE             STATUS                    PORTS
devops-lab6-quicknotes-1   quicknotes:lab6   Up 12 seconds (healthy)   0.0.0.0:8080->8080/tcp
```

### Design answers

e) Distroless has no shell, so a shell-form healthcheck cannot work. I built a tiny static `/healthcheck` binary in the builder stage and use exec-form `CMD ["/healthcheck"]` to call `http://127.0.0.1:8080/health`. That gives a real HTTP probe without adding a shell to the runtime image.

f) `volumes: [quicknotes-data:/data]` survives `docker compose down` because Compose removes containers and networks, not named volumes. The data stays in Docker's volume store until `docker compose down -v`, `docker volume rm`, or `docker volume prune`.

g) `depends_on` without `condition: service_healthy` only waits for the dependency container to start, not for the app inside it to be ready. A dependent service can send traffic before QuickNotes is listening and see connection errors.

## Bonus - 6 Security Defaults

Hardened `services.quicknotes` block is shown above (`cap_drop`, `read_only`, `tmpfs`, `security_opt`, nonroot image, distroless base).

Verification:

```text
$ docker inspect quicknotes:lab6 --format '{{ .Config.User }}'
nonroot:nonroot

$ docker compose exec quicknotes sh
OCI runtime exec failed: exec failed: unable to start container process: exec: "sh": executable file not found in $PATH

$ docker inspect <container> --format '{{ .HostConfig.CapDrop }}'
[ALL]

$ docker inspect <container> --format '{{ .HostConfig.ReadonlyRootfs }}'
true

$ docker inspect <container> --format '{{ .HostConfig.SecurityOpt }}'
[no-new-privileges:true]

$ docker inspect <container> --format '{{json .State.Health.Status}}'
"healthy"
```

Read-only root was also confirmed via `ReadonlyRootfs=true`; there is no shell in the runtime image to run `touch /etc/test` inside the container itself.

Trivy scan:

```text
$ docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:0.59.1 image --severity HIGH,CRITICAL --no-progress \
  quicknotes:lab6

quicknotes:lab6 (debian 12.14)
Total: 0 (HIGH: 0, CRITICAL: 0)
```

Trivy also reported Go stdlib findings inside the compiled binaries; the distroless OS layer itself had zero HIGH/CRITICAL findings.

The best security-per-line default here is `cap_drop: [ALL]`. It removes Linux capabilities from the container process namespace with one YAML line and directly reduces kernel-level escape options. Distroless and nonroot matter a lot too, but capability dropping is the clearest enforcement knob in Compose.
