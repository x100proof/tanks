# CLAUDE.md — tanks

## Design lives in `ARCHITECTURE.md`

The design of this project **must always be written in
[`ARCHITECTURE.md`](ARCHITECTURE.md)**. That file is the single source of truth
for how the game is put together: the coordinate system, the game-state machine,
the physics model, the rendering/camera model, the audio design, and the
deployment topology.

**`ARCHITECTURE.md` must be kept up to date with the implementation at all
times.** It is not a historical record and not a design sketch — it describes
the code as it exists right now.

Practically, that means:

- **Any change to behaviour or structure updates `ARCHITECTURE.md` in the same
  change.** Never leave the document describing the old design, not even
  temporarily.
- **Design first.** When adding a feature, write the design into
  `ARCHITECTURE.md`, then implement it to match.
- **If code and document disagree, that is a bug.** Fix whichever is wrong —
  usually the document — rather than letting them drift.
- **Constants belong in the document.** World size, gravity, wind strength,
  blast radius, port numbers and similar tunables are stated in
  `ARCHITECTURE.md`; when you change one in the code, change it there too.
- **Deleting a feature deletes its section.** Stale sections are as harmful as
  missing ones.

Before finishing any piece of work in this repo, re-read `ARCHITECTURE.md` and
confirm every statement in it still holds.

## Shape of the project

- The entire game is one self-contained file, `public/index.html` — markup,
  styles, and game code, with no build step, no dependencies, and no external
  assets. Keep it that way unless the design in `ARCHITECTURE.md` says
  otherwise.
- `server.js` is a zero-dependency static file server and should stay boring.
- `DEPLOY.md` explains how anyone can host their own copy, and must stay
  generic — no hostnames, addresses or infrastructure specific to one machine.
  The shape of the author's deployment is described in `ARCHITECTURE.md`, and
  the unit and vhost files it uses are in `deploy/`. Nothing in the repository
  may name a real host, domain, address or account: use placeholders such as
  `tanks.example.com` and fill in the real values on the machine itself.

## Testing

There is no test runner in the repo. Changes to the game are verified by
driving a real browser (Playwright) against `http://127.0.0.1:3005` — checking
the setup flow, terrain destruction, turn order, elimination, the win banner,
landscape layout, and touch drag-aiming. `window.__debug` exposes `G`, `view`,
`fire`, `newRound` and `startGame` for exactly this purpose; keep it exported.
