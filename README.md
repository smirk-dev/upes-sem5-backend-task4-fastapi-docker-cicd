# fastapi-docker-cicd

Backend Development (CSFS3008P) · UPES · B.Tech Full Stack Sem 5 · Theory · **Task 4**
Containerise a FastAPI app, push it to Docker Hub, and automate the build + push with GitHub Actions.

- Parent repository: [`smirk-dev/Backend-Development`](https://github.com/smirk-dev/Backend-Development) (this repo is mounted there as a git submodule at `Theory/Task4`)
- Docker Hub image: [`smirkdev/fastapi-app`](https://hub.docker.com/r/smirkdev/fastapi-app) (tag `v0.1`)
- Task brief: *Hands On Task* (Unit 4), published at <https://upessocs.github.io>

## Pipeline

```
Code change -> git push -> GitHub Actions -> docker build -> docker push (Docker Hub)
            -> docker pull / run -> /health OK -> logs OK -> two instances + nginx OK
```

## Layout

| File | Purpose |
|---|---|
| `main.py` | Student API: `/`, `/health`, `/students`, plus a request-logging middleware (`method= path= status= duration_ms=`) |
| `requirements.txt` | `fastapi==0.163.1`, `uvicorn==0.54.0` |
| `Dockerfile` | `python:3.12-slim`, installs requirements, runs uvicorn on port 8000 |
| `.dockerignore` | keeps `.env`, `.git` and `.github` out of the image (`COPY . .` would otherwise bake the token in) |
| `.github/workflows/ActionsYaml.yaml` | on push to `main`: checkout, docker login, build, push |
| `nginx.conf` | round-robin load balancer over two instances (8001, 8002), listening on 8080 |
| `.env.example` | template for the local-only, gitignored `.env` |

## Deviations from the brief

- `requirements.txt` is pinned to the versions pip resolved, so CI builds are reproducible (the brief leaves them unpinned).
- `.dockerignore` is an addition (reason above).
- `.env.example` is an addition; the real `.env` never enters git.

## Run it locally

```bash
docker build -t fastapi-app:local .
docker run --rm -p 8000:8000 fastapi-app:local
curl http://localhost:8000/health        # {"status":"healthy"}
curl http://localhost:8000/students
```

Or use the CI-built image: `docker run --rm -p 8000:8000 smirkdev/fastapi-app:v0.1`

### Two instances behind nginx

```bash
docker run -d --name fastapi-1 -p 8001:8000 smirkdev/fastapi-app:v0.1
docker run -d --name fastapi-2 -p 8002:8000 smirkdev/fastapi-app:v0.1
docker run -d --name student-lb -p 8080:8080 -v "$(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro" nginx:alpine
curl http://localhost:8080/health
docker stop fastapi-1 fastapi-2 student-lb && docker rm fastapi-1 fastapi-2 student-lb
```

`host.docker.internal` in `nginx.conf` works on Docker Desktop; on Linux Engine use a user-defined network and container names. On Git Bash for Windows, set `MSYS_NO_PATHCONV=1` and use `$(pwd -W)` for the mount path.

## CI/CD setup

Repository secrets (Settings > Secrets and variables > Actions):

| Secret | Value |
|---|---|
| `DOCKERUSERNAME` | Docker Hub username |
| `DOCKERTOKEN` | Docker Hub access token (Read & Write) |

The token is never committed. It is passed to `docker login` through `--password-stdin`, never `-p`.

## Verification (run 2026-11-17)

| Check | Result |
|---|---|
| Image builds and serves `/`, `/health`, `/students` locally | OK (200s; unknown path returns 404) |
| Structured request logs in `docker logs` | OK, e.g. `INFO method=GET path=/health status=200 duration_ms=2.28` |
| `.env` is not in the image | OK (`ls -a /app` shows only source files) |
| Two instances serve independently | OK (8001 and 8002 both healthy) |
| nginx splits traffic | OK: 11 requests through port 8080 landed 6 on `fastapi-1` and 5 on `fastapi-2` |
| Actions run 1 ("Add CI workflow"): login, build, push | OK, `v0.1` appeared on Docker Hub |
| CI-built image pulled and run | OK, `/health` returned `{"status":"healthy"}` |
| Part 8: code change adds SAP ID and location, pushed | Actions run succeeded |
| Pulled the updated image | OK, `/` returned `{"application":"Student API","framework":"FastAPI","status":"running","sapid":"590012069","location":"Dehradun"}` |

## Notes

- The tag `v0.1` is overwritten on every push, as the brief specifies. For real releases, tag with the commit SHA.
- The Student API is stateless, which is what makes running several instances behind a load balancer safe.
