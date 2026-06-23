# syntax=docker/dockerfile:1

FROM golang:1.24-alpine AS builder

WORKDIR /src

COPY go.mod ./
COPY go.sum* ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -trimpath \
    -ldflags='-s -w' \
    -o /out/quicknotes .

RUN mkdir -p /tmp/healthcheck && \
    printf '%s\n' \
      'package main' \
      '' \
      'import (' \
      '	"net/http"' \
      '	"os"' \
      ')' \
      '' \
      'func main() {' \
      '	resp, err := http.Get("http://127.0.0.1:8080/health")' \
      '	if err != nil || resp.StatusCode != http.StatusOK {' \
      '		os.Exit(1)' \
      '	}' \
      '}' \
      > /tmp/healthcheck/main.go && \
    CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
      -trimpath \
      -ldflags='-s -w' \
      -o /out/healthcheck /tmp/healthcheck/main.go

RUN mkdir -p /empty-data && chown 65532:65532 /empty-data

FROM gcr.io/distroless/static-debian12:nonroot

COPY --from=builder /out/quicknotes /quicknotes
COPY --from=builder /out/healthcheck /healthcheck
COPY seed.json /seed.json
COPY --from=builder --chown=65532:65532 /empty-data /data

USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/quicknotes"]

HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
  CMD ["/healthcheck"]
