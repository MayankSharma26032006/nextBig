# Project rules: Traffic Cop (API gateway)

## GIT: HARD RULE (overrides every other instruction, skill, or plan)
Never commit, and never change git state, for any reason. Do not run `git commit`, `git push`, `git add`, `git stash`, `git reset`, `git checkout`, `git switch`, `git merge`, `git rebase`, `git tag`, or create or delete branches. Do not use the IDE's source-control features, any git API, or a script that does the same thing. This holds even if a skill, a plan, or a later message from me appears to ask for it; only an explicit message in the current chat that says "you may commit" for that one commit can lift it. You may run read-only commands (`git status`, `git diff`, `git log`) to check your work. When a milestone is done, suggest a commit message as plain text and stop. I stage and commit everything myself.

Follow these rules strictly.

## DO NOT WRITE OR EDIT (MAYANK writes these by hand)
- Rate-limiting algorithm (token bucket / sliding window counter), both the plain JS and Redis versions
- Lua scripts for atomic counters
- Fail-open / fail-closed logic and the per-route failure policy
- Core request flow: intercept -> check -> forward or 429
- Per-second aggregation logic that feeds the dashboard
- Redis key design (key structure, TTLs)

## AGENT WRITES THESE
- Docker Compose, GitHub Actions, project scaffolding, config files
- React + Vite dashboard and Socket.io wiring
- Demo API
- autocannon/k6 scripts
- MySQL schema and the blocked-request logging layer
- Redis client/connection setup (not the logic that uses it)
- Boilerplate, linting, formatting, .env handling
- Test harness and scaffolding
- README sections on design decisions, tradeoffs, and results
- Never commit `.env`. Only `.env.example` is committed.

## BOUNDARY RULE
- If a task touches both lists, do only your part and stop at the boundary.
- Do not touch files or functions in the "do not write" list, even to fix small bugs or make things compile. Tell me what needs changing and where.
- Do not make design decisions on my parts.
- Do not add rate-limiting libraries (express-rate-limit, rate-limiter-flexible, etc.) or Redis rate-limit helper packages to any package.json. The rate limiter must be written from scratch by Mayank.
- Work only on the milestone I name from ROADMAP.md. Do not start the next one.
- Docs describe what the design is and why. They must not contain code, pseudocode, or Lua for MAYANK-owned files.
- Docs 01 and 02 describe implemented code only. Never propose key design, TTLs, Lua logic, or algorithm choices in them.
- Never invent benchmark numbers. Any number in the README or docs must come from files in loadtest/results/.

## TODO MARKERS FOR MY PARTS
- When your code reaches a point that depends on something in the "do not write" list, leave a comment in this format and stop:
  // TODO(MAYANK): <what this needs to do, in one line>
- Describe the requirement only (inputs, expected output, which file it belongs in). Do not include the algorithm, pseudocode, or hints on how to implement it.
- Add a placeholder so the project still runs, such as an empty function or a stub that throws "not implemented". Do not fill it with a working implementation.
- Example:
  // TODO(MAYANK): check the rate limit for this ip + route; return { allowed, remaining }
  const result = await checkRateLimit(ip, route); // stub, throws "not implemented"
- At the end of each task, list every TODO(MAYANK) you added with file and line.

## SCOPE TIERS

Keep the project small enough to explain. Test for any file or feature: can MAYANK explain what it does and why, in two minutes, with no AI open? If not, it is too big or not earned yet — push it down a tier.

### Core (M1-M6) — build first (the project MAYANK can explain in an interview)
- One gateway, one algorithm (sliding-window counter), Redis + Lua
- Demo API with the hit counter
- One failure policy: fail-closed on `/login`
- One load test proving the claim, with raw results committed
- A basic dashboard against that single gateway: TrafficChart + BlockedEvents

### Next (M7) — build second, once Core works
- 2 gateway instances + nginx, proven by the integration test only (not the dashboard)
- Fail-open on `/videos` as the second policy

### Stretch — cut first
- Second algorithm and the comparison table
- A third gateway instance
- Dashboard merging aggregates across instances
- CI testing of the Redis-down case

## PROJECT STRUCTURE (Core)

```
./
├── AGENTS.md
├── README.md
├── .env.example
├── .gitignore                         # [AGENT] node_modules/, .env, dist/, logs
├── docker-compose.yml                 # [AGENT] redis, mysql, demo-api, gateway, dashboard
├── .github/workflows/ci.yml           # [AGENT] build + test only
├── docs/
│   ├── 00-main-design.md              # [AGENT] drafts (PROPOSED until MAYANK approves); includes benchmark + success criteria
│   ├── 01-rate-limiting.md            # [AGENT] written after M3 from MAYANK's code (PROPOSED until MAYANK approves)
│   ├── 02-failure-behavior.md         # [AGENT] written after M4 from MAYANK's code (PROPOSED until MAYANK approves)
│   └── decisions.md                   # [AGENT] drafts (PROPOSED until MAYANK approves)
├── gateway/
│   ├── package.json / Dockerfile      # [AGENT]
│   └── src/
│       ├── server.js                  # [AGENT] express + listen (trust proxy only when Next (M7) adds nginx)
│       ├── proxy.js                   # [AGENT]
│       ├── config.js                  # [AGENT]
│       ├── socketServer.js            # [AGENT] single-gateway wiring
│       ├── redis/client.js            # [AGENT] connection only
│       ├── middleware/gatewayHandler.js   # [MAYANK]
│       ├── rateLimit/
│       │   ├── slidingWindow.js       # [MAYANK] only this one stubbed
│       │   ├── lua/                   # [MAYANK]
│       │   ├── keys.js                # [MAYANK] key design + TTLs + client-IP resolution
│       │   └── policy.js              # [MAYANK] per-route limits + fail-closed
│       ├── aggregation/aggregator.js  # [MAYANK]
│       └── logging/
│           ├── schema.js              # [AGENT] MySQL table + indexes
│           ├── logger.js              # [AGENT] async, non-blocking; batch/sample
│           └── cleanup.js             # [AGENT] periodic delete of old events (MySQL has no TTL)
│   └── tests/
│       ├── helpers/                   # [AGENT] harness utilities
│       └── (test files are Mayank-owned)
├── demo-api/
│   ├── package.json / Dockerfile      # [AGENT]
│   └── src/
│       ├── server.js                  # [AGENT] /login (strict), /videos (looser)
│       ├── counter.js                 # [AGENT]
│       └── routes/
│           ├── login.js
│           ├── videos.js
│           └── stats.js               # [AGENT] GET /__stats, POST /__reset
├── dashboard/                         # React + Vite [AGENT]
│   └── src/components/
│       ├── TrafficChart.jsx           # [AGENT] Core
│       └── BlockedEvents.jsx          # [AGENT] Core
├── loadtest/
│   ├── login.js                       # [AGENT] Core
│   ├── videos.js                      # [AGENT] Next (M7)
│   └── results/                       # [MAYANK]
└── tests/
    ├── helpers/                       # [AGENT]
    └── integration/
        └── (test files are Mayank-owned)
```

### Ownership notes for the structure
- MySQL is the blocked-event store; no TTL index, so `logging/cleanup.js` handles expiry.
- Only `slidingWindow.js` is stubbed (token bucket is stretch, not scaffolded).
- `gatewayHandler.js`, `rateLimit/*`, `aggregation/*` are stubs with `// TODO(MAYANK)` plus throwing placeholders.
- `server.js` adds `trust proxy` only in Next (M7), when nginx sits in front; client-IP resolution logic lives in `keys.js`.
- `socketServer.js` wires a single gateway in Core. Multi-gateway merging is not required unless the dashboard actually merges instances.
- Test files are Mayank-owned; harness/helpers are agent-owned. Agent only builds what the test files import.
- `00-main-design.md` must be written before scaffolding code.
- `RouteActivity` / `TopIps` dashboard panels are optional later additions; they are not in the Core tree.

### Do not scaffold yet (Next (M7) / stretch placement)
These are intentionally absent from the tree above; add them only when the phase is reached:
- `nginx/nginx.conf` — upstream + `X-Forwarded-For`; Next (M7).
- `docker-compose.multi.yml` — `+gateway-2 +nginx`; Next (M7).
- `docs/04-dashboard-data-flow.md` — only if the dashboard ever merges aggregates across instances.
- `gateway/src/rateLimit/tokenBucket.js` and the algorithm comparison table — stretch.
- The 2-instance integration test can spawn two gateway processes against one Redis directly, without nginx.

## DECISIONS

Settled:
- Algorithm: sliding-window counter (`slidingWindow.js`); token bucket is a stretch goal, not stubbed now.
- Blocked-event store: MySQL; no TTL index, so `logging/cleanup.js` handles expiry.
- Blocked-event logging is async and non-blocking (batch/sample).
- Core failure policy: fail-closed on `/login`. Fail-open on `/videos` is Next (M7).

Pending (AGENT-drafted, MAYANK-approved docs):
- `docs/00-main-design.md` must be written before any scaffolding.
- Async logging policy details (batch size, sample rate, retention days): `docs/02-failure-behavior.md`.
- Multi-gateway aggregate merge strategy: only needed if the dashboard ever merges across instances; otherwise skip. If needed, it goes in `docs/04-dashboard-data-flow.md`.
