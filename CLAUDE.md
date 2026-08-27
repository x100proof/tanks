# CLAUDE.md — tanks

## Design lives in `architecture.md`

The design of this project **must always be written in
[`architecture.md`](architecture.md)**. That file is the single source of truth
for how the game is put together: the coordinate system, the game-state machine,
the physics model, the rendering/camera model, the audio design, and the
deployment topology.

**`architecture.md` must be kept up to date with the implementation at all
times.** It is not a historical record and not a design sketch — it describes
the code as it exists right now.

Practically, that means:

- **Any change to behaviour or structure updates `architecture.md` in the same
  change.** Never leave the document describing the old design, not even
  temporarily.
- **Design first.** When adding a feature, write the design into
  `architecture.md`, then implement it to match.
- **If code and document disagree, that is a bug.** Fix whichever is wrong —
  usually the document — rather than letting them drift.
- **Constants belong in the document.** World size, gravity, wind strength,
  blast radius, port numbers and similar tunables are stated in
  `architecture.md`; when you change one in the code, change it there too.
- **Deleting a feature deletes its section.** Stale sections are as harmful as
  missing ones.

Before finishing any piece of work in this repo, re-read `architecture.md` and
confirm every statement in it still holds.

## Shape of the project

- The entire game is one self-contained file, `public/index.html` — markup,
  styles, and game code, with no build step, no dependencies, and no external
  assets. Keep it that way unless the design in `architecture.md` says
  otherwise.
- `server.js` is a zero-dependency static file server and should stay boring.
- Deployment artefacts live in `deploy/` and are copied into place; see
  `architecture.md` for the topology and `README.md` for the commands.

## Testing

There is no test runner in the repo. Changes to the game are verified by
driving a real browser (Playwright) against `http://127.0.0.1:3005` — checking
the setup flow, terrain destruction, turn order, elimination, the win banner,
landscape layout, and touch drag-aiming. `window.__debug` exposes `G`, `view`,
`fire`, `newRound` and `startGame` for exactly this purpose; keep it exported.
