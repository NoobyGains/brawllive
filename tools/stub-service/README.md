# Stub matchmaking service (sub-project 1, Task 7)

Zero-dependency Node.js 24 TypeScript server that speaks BrawlLive protocol v1 well enough to pair two agents and walk them through `match_found → host → join → start`. Used by the two-instance milestone harness and by anyone hacking on the Dolphin agent without the real service.

Files to be created by Task 7 of the plan: `package.json`, `server.ts`, `main.ts`, `server.test.ts`.

```powershell
node --test server.test.ts
node main.ts --port 8787 --mode direct --countdown 3
```

Debug endpoints: `GET /debug/state`, `POST /debug/reset`.
