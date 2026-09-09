# myapp

A minimal Go HTTP server (stdlib only, no dependencies) with:

- `GET /`        → `{"message": "hello from myapp"}`
- `GET /healthz` → `{"status": "ok"}`
- `GET /readyz`  → `{"status": "ready"}`

These match the liveness/readiness probe paths in the Helm chart from the
CI/CD pipeline, so this app deploys as-is.

## Run locally without Docker

```bash
go run ./cmd/app
curl localhost:8080/
```

## Run with Docker

```bash
docker build -t myapp:local .
docker run --rm -p 8080:8080 myapp:local
```

Then in another terminal:

```bash
curl localhost:8080/
curl localhost:8080/healthz
curl localhost:8080/readyz
```

Override the port with `-e PORT=9090` if you don't want 8080.

## Project layout

```
cmd/app/main.go   # entrypoint
go.mod / go.sum   # module def (no external deps yet)
Dockerfile        # multi-stage build, distroless runtime
```

When you add real dependencies, run `go mod tidy` locally to populate
`go.sum` before building the Docker image — `go mod download` in the
Dockerfile needs it to match.
