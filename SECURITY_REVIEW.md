# Security Review (Local Source Analysis)

Scope: static source analysis and local validation only.

## Candidate Findings

### 1) Queue endpoints accept unauthenticated execution requests in local world
- **Severity**: High (when exposed beyond localhost), Medium (default local-only usage)
- **Files / functions**:
  - `packages/world-local/src/queue.ts` — `createQueueHandler`
  - `packages/core/src/runtime.ts` — `workflowEntrypoint` (binds queue handler)
  - `packages/core/src/runtime/step-handler.ts` — `stepEntrypoint` (binds queue handler)
- **Why reachable**:
  - Queue HTTP handlers are mounted under `/.well-known/workflow/v1/flow` and `/.well-known/workflow/v1/step` by integrations.
  - Handler validation checks only that required queue headers exist and match prefix; there is no HMAC/signature/auth token verification in `world-local` queue handler.
- **Impact**:
  - If these endpoints are reachable from untrusted networks, attackers can trigger workflow/step execution, causing unauthorized job execution and potential abuse/DoS.
- **Evidence**:
  - `createQueueHandler` accepts requests based on queue headers/prefix and deserializes body directly, without authenticating sender.
- **Minimal local repro idea**:
  - Run a local app using `world-local`, then `curl` POST `/.well-known/workflow/v1/flow` with `x-vqs-*` headers and valid queue payload shape; observe handler execution.

### 2) Sensitive workflow payloads are logged on queue errors
- **Severity**: Medium
- **Files / functions**:
  - `packages/world-local/src/queue.ts` — async worker loop in `createQueue` (`console.error` on queue failure)
- **Why reachable**:
  - Any step/workflow handler exception path returns an error response; caller-side queue retry logic logs response text plus serialized body.
- **Impact**:
  - Secrets in workflow arguments/event payloads can be written to logs, expanding blast radius (log sinks, CI logs, shared terminals).
- **Evidence**:
  - Error logging includes `body: body.toString()` and response text/headers.
- **Minimal local repro idea**:
  - Trigger a workflow with a secret value in args and force handler failure; inspect local logs for leaked payload.

### 3) Workflow sandbox exposes full `process.env` to workflow code
- **Severity**: Medium
- **Files / functions**:
  - `packages/core/src/vm/index.ts` — `createContext`
- **Why reachable**:
  - Every workflow run builds VM context via `createContext`, which sets `g.process.env = Object.freeze({ ...process.env })`.
- **Impact**:
  - Workflow code (the less-privileged orchestration layer) can read all environment variables, including secrets unrelated to workflow orchestration.
  - This weakens isolation claims between workflow and step contexts and increases risk from untrusted or compromised workflow code.
- **Evidence**:
  - Explicit propagation of host environment variables into VM global `process.env`.
- **Minimal local repro idea**:
  - Add a workflow that reads `process.env` key and returns/logs it; observe value in run output/logs.

### 4) Demo start endpoint allows unauthenticated remote workflow invocation
- **Severity**: Medium (demo app), High if reused in production pattern
- **Files / functions**:
  - `workbench/nextjs-turbopack/app/api/workflows/start/route.ts` — `POST`
- **Why reachable**:
  - Public POST route parses JSON body and starts selected workflow with caller-supplied args; there is no authn/authz check before `start(workflowFn, workflowArgs)`.
- **Impact**:
  - Remote users can trigger expensive workflows and side effects, causing abuse, cost amplification, and unauthorized actions.
- **Evidence**:
  - Route directly maps user-controlled `workflowName`/`args` to workflow execution.
- **Minimal local repro idea**:
  - Start workbench app and POST to `/api/workflows/start` from an unauthenticated client with valid workflow name.

## Notes
- This review excludes production penetration testing and external services.
- Findings above are concrete code paths with observable behavior from local execution.
