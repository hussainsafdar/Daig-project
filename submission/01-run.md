# Phase 1 — Run it & the exit-78 exercise

## Task 1: make up / seed / smoke

make up, make seed, make smoke all completed successfully. A real order was placed end-to-end through the browser UI (localhost:8080), confirmed with HTTP 201 and a downstream status of 200, proving the request crossed orders -> kitchen -> dispatch.

## Task 2: The broken image

Command: make broken
Result: docker build --target broken -f services/orders/Dockerfile -t tkxel/daig-orders:broken . -- built successfully, naming to docker.io/tkxel/daig-orders:broken

Command: docker run --rm tkxel/daig-orders:broken
Output:
{"level":"info","msg":"telemetry started","service":"orders","endpoint":"http://otel-collector:4318"}
{"level":"fatal","service":"secrets","msg":"No secret store configured while NODE_ENV=production","exit_code":78,"remedy":"Configure OpenBao (BAO_ADDR/BAO_ROLE_ID/BAO_SECRET_ID), or deliver secrets as *_FILE Docker/Swarm secrets. See vault/README.md."}
Exit code: 78

## Task 3: What is misconfigured, and how the code signalled it

The broken image sets NODE_ENV=production but supplies no secret store (no BAO_ADDR/BAO_ROLE_ID/BAO_SECRET_ID, and no *_FILE Docker/Swarm secrets). services/_shared/secrets.js bootstrap function walks a deliberate three-path resolution order: try OpenBao, then fall back to mounted secret files or plain environment variables (but only outside production), and otherwise refuse. Because NODE_ENV=production here and no file-backed secrets are present, the code calls die(), which logs a single structured JSON line naming exactly what is missing and exits with code 78, never falling back silently to guessing or a stack trace.

## Checkpoint: what is exit code 78?

78 is not an arbitrary number - it is EX_CONFIG from the Unix sysexits.h convention (see the comment in services/_shared/guard.js line 6), meaning configuration error. It is defined in two places: services/_shared/secrets.js line 20 and services/_shared/guard.js line 6. Both call process.exit(EX_CONFIG) after logging a structured fatal message that names the specific missing configuration and suggests a remedy.

Failing this way is better than a stack trace because a stack trace tells you where in the code something broke, but not why - you have to read source to reconstruct the cause. A named, structured fatal log tells you immediately what is missing and what to do about it, and a distinct, documented exit code lets orchestration tooling distinguish a config problem from a crash worth retrying.
