# Architecture — Tank Duel

The design of this project. It describes the code as it exists now; see
[`CLAUDE.md`](CLAUDE.md) for the rule that this file is kept in step with the
implementation at all times.

Tank Duel is a hot-seat artillery game for 2–4 players on one device.

---

## 1. Files

| Path | Role |
|---|---|
| `public/index.html` | The entire game — markup, CSS and JS in one file. No build step, no dependencies, no external assets. |
| `server.js` | Zero-dependency Node static file server for `public/`. |
| `DEPLOY.md` | How anyone can host their own copy. Deliberately generic. |
| `deploy/tanks.service` | systemd **user** unit used on the host that runs the game. |
| `deploy/tanks.nginx` | nginx vhost on that host, proxying to the Node server. |

Everything the browser needs is inside `public/index.html`: the graphics are
drawn from geometry, the sounds are synthesised, and the favicon is an inline
SVG data URI. There are no images, fonts or audio files to fetch.

---

## 2. Coordinate system and the camera

All game logic — physics, terrain, tank sizes, blast radii — is expressed in a
fixed **square world of 700 × 700 units** (`SIZE`). No device pixels appear
anywhere in the simulation, so behaviour is identical on every screen.

The battlefield is square on purpose. A square is the one shape that suits
every viewport equally: it is scaled to fit whatever it lands in, and the
leftover screen along the long axis is filled with more of the same
battlefield (see *Overscan*). A phone held upright, a phone on its side and a
desktop therefore all get exactly the same fight, only framed differently —
there is no orientation the game prefers, and no rotation prompt.

Rendering is **fully vector**: paths, arcs and gradients on a 2D canvas. Nothing
is rasterised or resampled.

The `view` object maps world units to the screen each time the canvas is sized:

- `s` — a single **uniform** scale, `min(cssW/W, cssH/H)` ("contain"), so the
  whole world is always visible and nothing is ever cropped or distorted.
- `offX`, `offY` — framing offsets. Horizontally the world is centred;
  vertically it sits low in the frame (80% of the slack above it), so spare
  height on a tall screen becomes sky that shells fly through rather than a
  dead band of dirt beneath the battlefield.
- `x0, x1, y0, y1` — the visible rectangle **in world coordinates**. Because the
  fit is "contain", the screen is usually wider than the world; these bounds run
  past `0..W`.
- `inset` — world units hidden behind the floating control bar (§8).

The canvas backing store is sized to `CSS pixels × devicePixelRatio`, with dpr
capped at 2, and the whole transform is folded into one `setTransform` call, so
drawing code only ever speaks world units.

### Overscan

Because the visible rectangle is wider than the world, the renderer deliberately
draws **past the world edges** rather than leaving letterbox bars:

- The sky gradient fills the entire visible rectangle.
- The terrain path runs from `x0-2` to `x1+2`, indexing the heightfield through
  a clamp, so the edge columns extend outwards and the ground continues to the
  screen edge.

Tanks are always placed inside the world, so the overscan region is scenery
only and gameplay is unaffected by screen shape. The extended ground is
nonetheless **solid**: shell/ground collision uses the same clamped heightfield
across the whole visible area, so a shell can never fly through scenery the
player can see. Beyond the edges of the screen there is nothing to look wrong,
and a shell out there may still be blown back into play.

---

## 3. Terrain

The terrain is a **heightfield**: `Float32Array(W)`, one surface `y` per world
column. A parallel `Uint8Array(W)` (`burn`) marks columns scorched by a blast.

**Generation** sums four octaves of sine with random amplitude, wavelength and
phase. The surface is confined to a band:

- `top = H * 0.20`
- `floor = H - 46 - clamp(view.inset, 0, H * 0.34)`

Octave amplitudes are scaled by `k = band / (H - 46 - top)` so hills are
*scaled into* whatever band is available rather than clipped flat by it — this
is what keeps the terrain interesting when the control bar floats over the field
on a short screen. The centre line sits at `top + band * 0.57`.

`flatten(x, 26 * tankScale)` levels a blended landing pad under each tank so
hulls sit flush instead of clipping into a slope.

**Starting positions** spread the players across the full width: the outermost
two begin near opposite edges, and each is jittered by a fraction of the gap so
no two rounds look alike. A minimum-separation pass then stops the jitter ever
bunching neighbours together. Measured over 300 rounds, two players always start
at least 51% of the battlefield apart (average 73%), and with three or four the
outermost pair always reach the left and right thirds.

**Destruction** (`carveCrater`) walks the columns under the blast and lowers the
surface to the bottom of a circle of radius `BLAST_R`, never below `BEDROCK`
(`H - 2`), marking each touched column in `burn`. Craters are therefore
geometric edits to the heightfield, not pixel operations, and remain crisp at
any resolution.

The surface is drawn as one filled path (dirt gradient) plus a per-column skin:
grass where `burn` is clear, scorched dirt where it is set.

---

## 4. Physics

| Constant | Value | Meaning |
|---|---|---|
| `GRAV` | 320 | downward acceleration, world units/s² |
| `MUZZLE_V` | √(`GRAV`·`W`·1.15)/100 ≈ 5.08 | muzzle speed per unit of power, derived so full power carries about 1.15 battlefields — the far side is always reachable, never trivially so |
| `blastR` | 0–84, default 48 | crater and kill radius, chosen on the setup screen (§6) |
| `TANK_R` | 20 | tank hit radius, multiplied by `tankScale` |
| `STEP` | 1/300 s | fixed integration step |

**There is no wind.** The game is for young children, and identical dials
producing an identical shot every single time is what makes it learnable: a
child can watch a shot fall short, nudge the power up, and see exactly the
result they expected. Nothing in the flight is random.

The shell integrates on a **fixed timestep accumulator** (`STEP`), independent
of frame rate, capped at 400 substeps per frame. Each step applies gravity to
`vy`, then advances position and tests, in order:

1. **Direct hit** — within `TANK_R` of any live tank's centre `(x, y-14)`.
2. **Ground** — `y >= terrain[x]`, across the world and the visible overscan.
3. **Lost** — `y > H + 300`, or `|x|` more than 900 beyond the world, at which
   point the shot is a miss.

The ground test covers the visible overscan as well as the world proper, so
what looks solid is solid (§2).

On impact the blast is resolved **before** the ground changes shape: the tank
the shell physically struck, plus every live tank within
`blastR + TANK_R * 0.6 * tankScale` of the impact point, is eliminated. Then the
crater is carved, so a tank is judged where it stood.

**At `blastR = 0` there is no splash whatsoever** — only the tank the shell
actually strikes dies, so the game becomes direct-hits-only. The struck tank is
therefore passed into `impact()` rather than being re-derived from a radius,
which would fail at zero. A zero-radius shell still scorches the ground where it
lands (`carveCrater` marks a minimum band without digging), so a near miss still
shows a child exactly where the shot went.

Aim is `angle` in degrees (0 = due right, 90 = straight up, 180 = due left) and
`power` in `10..100`, both persisted per tank between turns. The barrel is drawn
in world space from the turret pivot, so it always shows the true firing angle
even when the hull is tilted by the slope.

---

## 5. Game state machine

`G.state` is one of:

| State | Meaning | Leaves when |
|---|---|---|
| `setup` | Setup screen: player count and names | **Start battle** pressed |
| `aim` | Current player is aiming | **Fire** pressed |
| `fly` | Shell in flight | shell hits or is lost |
| `boom` | Explosion particles, screen shake | after 0.45 s |
| `settle` | Tanks fall into fresh craters | all tanks at rest |
| `over` | Round finished, banner shown | **Next round** / **New game** |

**Falling.** After a blast, any tank whose ground has been dug out from under it
falls (at `GRAV * 2.2`) until it lands, thudding if it hits hard. `endTurn` is
deferred until everything is at rest, so eliminations and craters resolve before
the next player aims.

**Turn resolution.** `endTurn` counts survivors:

- **more than one** — the turn passes to the next living player (`nextAlive`,
  which skips eliminated tanks and wraps).
- **exactly one** — that player wins the round and scores a point.
- **none** — a draw, and nobody scores.

One hit eliminates a tank, so a 2-player round ends on the first hit, while 3-
and 4-player rounds continue until one tank remains. A player who blasts
themselves hands the round to the others. Scores accumulate across rounds;
**Next round** regenerates terrain and revives everyone, rotating who shoots
first.

---

## 6. Setup options

Before a game starts the setup screen chooses, besides player count and names:

**Tank size** (`TANK_SIZES`, 0.35×–1.75×, default 1×). `tankScale` multiplies
*everything* about a tank together — the drawing, the turret pivot, the barrel
and muzzle, the landing pad flattened under it, and `TANK_R` — so a bigger tank
is genuinely a bigger target and not merely a bigger picture.

The preview beside it is drawn at **exactly the battlefield's own scale**
(`tankScale * view.s`), so the tank shown is the same number of pixels as the
tank that will be played with. Its frame is sized to hold the largest option of
either setting, so stepping through the options never rescales the picture —
only what is inside it changes.

**Blast radius** (`BLAST_SIZES`, 0–84, default 48). The preview draws the lethal
radius as a dashed ring around the tank at the same true scale, so the two
settings can be judged against each other. At zero the ring is replaced by a
crosshair on the tank itself and the screen says *direct hits only* — a note
that drops onto its own line on a narrow screen rather than crowding the row.

Both are rendered with `tankBody()` and `paintBarrel()`, the same routines the
battlefield uses — there is no second, drifting copy of what a tank looks like.

---

## 7. Controls

Three input methods drive the same two values (`angle`, `power`), and all three
stay in sync through `syncUI`:

- **Sliders** — angle `0..180`, power `10..100`, each flanked by a **−/+
  button** that steps it by one for fine aiming. The angle slider is rendered
  right-to-left (`direction: rtl`), so pushing its thumb right — or tapping
  its **+** button — *lowers* the angle and swings the barrel **clockwise**,
  while **−** raises it and swings anti-clockwise: the control moves the same
  way the barrel does. Its value stays the true angle in degrees, so assistive
  tech announces the real number. The two dials keep a deliberately wide gap
  between them in every layout (56px between the side-by-side columns, 44px in
  the short-viewport overlay, 28px between the stacked portrait rows — FIRE
  keeps its tight 8px row gap), so a fat finger on the angle **+** can't land
  on the power **−** next to it — the game is played by young children. The
  buttons repeat when held
  (350 ms before the first repeat, then every 55 ms), disable themselves at the
  ends of their range, and route through the same `syncUI` as everything else so
  the slider, the readout, the power fill and the barrel all stay in step. A tap
  acts on `pointerdown`; the `click` handler fires only for keyboard activation
  (`event.detail === 0`), so a tap can never count twice. The two are
  deliberately
  distinguishable at a glance: angle wears a pale tint of the player's colour,
  power wears the full colour **and fills in from the left up to the ball**, so
  how full a shot is can be read without reading the number. The fill stops
  under the middle of the thumb (`--stop` accounts for the thumb width) rather
  than running past it.
- **Drag anywhere on the field** (touch or mouse) — the direction from the
  turret to the finger sets the angle, and the distance sets the power
  (`10 + (dist - 60) * 90/340`, clamped). A reticle tracks the finger. Dragging
  never fires; committing a shot is always the **FIRE** button, so a stray touch
  cannot waste a turn. Pointer coordinates are mapped back through the camera
  transform, so aiming is correct at any scale or aspect ratio.
- **Keyboard** — arrows adjust angle/power (with shift for coarse steps), space
  or enter fires.

A short dashed lead line of fixed length (the length full power used to draw)
shows the aim direction — it is purely an angle guide; the power slider's fill
is what shows the stored energy. It deliberately does **not** plot the
trajectory: finding the arc is the game.

---

## 8. Layout

The page is a full-height flex column: scoreboard, playfield, controls. The
canvas fills the stage exactly and the game draws edge to edge (§2).

**Both orientations, no rotation prompt.** Because the battlefield is square
(§2), neither orientation is privileged and the game never asks the player to
turn the device. A **Full** button is still offered: it requests fullscreen and,
where the browser allows it, `screen.orientation.lock('landscape')` — a
convenience, not a requirement.

**Portrait** (`max-width: 560px`) has height to spare and none to waste
sideways, so the angle and power dials stack — each on a full-width row — and
the FIRE button drops onto its own full-width row below them; the flexing
stage gives up the height the taller panel needs. The scoreboard keeps only
the colour chips and scores — the name of whoever is up is on the turn line
directly below. The setup screen stacks its name fields.

**Short viewports** (`max-height: 560px` — a phone on its side) have the
opposite problem, so the control bar becomes a translucent overlay across the
bottom of the field instead of taking space in the column. To stop it covering
the action, `view.inset` measures its height in world units and §3 keeps the
terrain surface — and therefore every tank — above it. The class is set from JS
rather than a media query so a single rule decides it.

The top bar carries the scores, the round number, and a **full screen** button
in the top-right corner — the familiar four-corner expand glyph, drawn as SVG
paths so it needs no icon font, beside the words *Full Screen*. The glyph flips
to the inward "collapse" corners and the label to *Exit Full Screen* while
full screen is active. Portrait hides the round counter to keep four scores
legible. The same button leaves full screen again, and its
label tracks `fullscreenchange`, so pressing Escape or using a system gesture
keeps it honest. iOS Safari has no element fullscreen, so there the button
hides itself rather than offering something that cannot work.

Scores, the round counter, and the turn indicator are sized with `clamp()` so
they stay readable from phone to desktop. The active player is marked on the
scoreboard, and the FIRE button and slider thumbs take that player's colour.

---

## 9. Frame cost

Drawing is the most expensive thing this game does, so two rules keep it cheap.

**The battlefield is cached.** Painting the terrain means one path plus a
per-column surface skin — with overscan that was ~2,400 `fillRect` and ~1,200
`lineTo` calls *per frame*, about 150,000 `fillRect` calls a second. It is now
rendered once into an offscreen canvas (`rebuildTerrain`) and blitted, and only
re-rendered when the heightfield actually changes: a new round, a crater, or a
change of scale. `markTerrainDirty()` is called from `generateTerrain`,
`flatten` and `carveCrater`. The overscan either side is flat by construction,
so it is a couple of rectangles rather than a loop, overlapped by a unit to hide
the antialiasing seam. Sky and dirt gradients are likewise built once, not per
frame.

**Idle frames are throttled.** Most of a turn is spent with a player thinking,
and repainting a full screen 60 times a second to show nothing new is pure
waste. The loop keeps updating every frame — the simulation is cheap — but only
*draws* when something is moving (`fly`, `boom`, `settle`, live particles or
screen shake), when `requestDraw()` has been called because input changed what
is on screen, or when `IDLE_FPS` (10) says it is time to drift the clouds along.

Measured in headless Chromium, which software-rasterises and so overstates the
absolute numbers: idle cost fell from **53% of a core to 13%**, and canvas
calls from ~150,000/s to a few hundred. A browser with GPU compositing pays
much less than either figure. `IDLE_FPS` is the knob if it ever needs to go
lower; the remaining cost is pixel fill, which is proportional to frames drawn.

---

## 10. Audio

Every sound is synthesised with the Web Audio API; there are no audio files. The
context is created on the **Start battle** click, which satisfies browser
autoplay rules.

- **Fire** — a filtered noise burst plus a falling sine thump.
- **Shell whistle** — the Scorched Earth effect: a square-wave oscillator
  through a lowpass, started at launch, whose frequency **tracks the shell's
  altitude** (`190 + alt * 760` Hz, smoothed with `setTargetAtTime`). Because
  pitch follows height rather than time, it rises as the shell climbs and falls
  as it descends, and it is cut with a short fade on impact.
- **Explosion** — a low filtered noise burst plus a deep falling tone.
- **Landing thud** and a four-note **victory** arpeggio.
- **Angle ratchet** — every angle change ticks like a dial detent: a 30 ms
  square blip, rate-limited to one per 28 ms so a fast sweep clatters like a
  ratchet instead of buzzing. Fires from all four aiming inputs (slider, ±
  buttons, arrow keys, drag-aiming), and only when the value actually moves —
  pinning against a limit is silent.
- **Power tone** — while power is being adjusted, a single triangle oscillator
  hums at a pitch mapped exponentially to the stored energy:
  `75 · 2^(2.5·(p−10)/90)` Hz, i.e. ≈75 Hz at power 10 rising ~2.5 octaves to
  ≈425 Hz at 100. The oscillator is retuned with `setTargetAtTime` as the value
  changes and fades out 180 ms after the last change (or immediately on fire).

A speaker-icon toggle mutes everything and stops any whistle or power tone in
flight. The icons are inline SVGs in the classic flat speaker style (a
rectangle-plus-horn silhouette — no emoji, no external assets), and the one
shown is what pressing it **will do**, not the current state: while sound is
on it shows a speaker with a slash through it (press to mute), and while muted
it shows the speaker with sound-wave arcs (press to unmute). Its `title` and
`aria-label` carry the matching verb ("Mute"/"Unmute").

---

## 11. Server and deployment

> This section records the *shape* of the deployment. For hosting a copy
> yourself, see [`DEPLOY.md`](DEPLOY.md). Real hostnames, addresses and domain
> names are deliberately not written down anywhere in this repository.

`server.js` serves `public/` over HTTP on `127.0.0.1:3005` (override with
`PORT`/`HOST`). It resolves and normalises the request path and refuses anything
that escapes the document root, sends `Cache-Control: no-cache` so a reload
always picks up a new build, and has no dependencies.

The game sits behind two layers of nginx: a public edge host that terminates
TLS and forwards a wildcard subdomain over a private network (a VPN mesh) to a
backend host, which runs its own per-site nginx vhost and the Node process:

```
browser
  └── https://tanks.<domain>
        └── edge host (public nginx, terminates TLS)
              │  *.<domain> wildcard vhost → backend host, over the private network
              └── backend host
                    └── nginx vhost tanks.<domain>  (loopback + private-network address, :80)
                          └── proxy_pass 127.0.0.1:3005
                                └── tanks.service (systemd --user) → node server.js
```

- **Port 3005**, chosen because 3000–3004 were already taken on the backend
  host.
- **systemd user service.** `deploy/tanks.service` runs as an unprivileged
  user with `Restart=always` and `WantedBy=default.target`. Lingering is
  enabled for that user (`loginctl enable-linger`), so the service starts at
  boot with no login session and survives both a crash and a host restart.
- **Backend vhost only.** `deploy/tanks.nginx` is the backend host's vhost. The
  edge already proxies the wildcard subdomain and DNS already resolves it, so
  adding this one vhost was enough to publish the new subdomain; the edge
  needed no changes. The placeholders in the file (`tanks.example.com`, the
  private-network listen address, the log paths) are filled in on the host and
  never committed.

---

## 12. Debug handle

`window.__debug` exposes `{ G, view, fire, newRound, startGame, worldW, worldH,
tankScale, sizeIdx, sizes, previewScale, gameTankScale, blastR, blastIdx,
blasts }`. It is what the
browser-driven tests use to inspect terrain, force deterministic shots and step
rounds. It is intentionally shipped: this is a local hot-seat game with nothing
to protect, and the handle makes the game testable.
