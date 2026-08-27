# Architecture — Tank Duel

The design of this project. It describes the code as it exists now; see
[`CLAUDE.md`](CLAUDE.md) for the rule that this file is kept in step with the
implementation at all times.

Tank Duel is a hot-seat artillery game for 2–4 players on one device, hosted at
<https://tanks.example.com>.

---

## 1. Files

| Path | Role |
|---|---|
| `public/index.html` | The entire game — markup, CSS and JS in one file. No build step, no dependencies, no external assets. |
| `server.js` | Zero-dependency Node static file server for `public/`. |
| `deploy/tanks.service` | systemd **user** unit. |
| `deploy/tanks.nginx` | nginx vhost for the backend host. |

Everything the browser needs is inside `public/index.html`: the graphics are
drawn from geometry, the sounds are synthesised, and the favicon is an inline
SVG data URI. There are no images, fonts or audio files to fetch.

---

## 2. Coordinate system and the camera

All game logic — physics, terrain, tank sizes, blast radii — is expressed in a
fixed **world coordinate system of 1200 × 700 units** (`W` × `H`). No device
pixels appear anywhere in the simulation, so behaviour is identical on every
screen.

Rendering is **fully vector**: paths, arcs and gradients on a 2D canvas. Nothing
is rasterised or resampled.

The `view` object maps world units to the screen each time the canvas is sized:

- `s` — a single **uniform** scale, `min(cssW/W, cssH/H)` ("contain"), so the
  whole world is always visible and nothing is ever cropped or distorted.
- `offX`, `offY` — centring offsets.
- `x0, x1, y0, y1` — the visible rectangle **in world coordinates**. Because the
  fit is "contain", the screen is usually wider than the world; these bounds run
  past `0..W`.
- `inset` — world units hidden behind the floating control bar (§7).

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

Tanks are always placed inside `0..W`, so the overscan region is scenery only
and gameplay is unaffected by screen shape.

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

`flatten(x, 26)` levels a blended landing pad under each tank so hulls sit flush
instead of clipping into a slope.

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
| `WIND_ACC` | 5.2 | horizontal acceleration per unit of wind |
| `MAX_WIND` | 10 | wind is a random integer in `-10..10`, re-rolled every turn |
| `MUZZLE_V` | 6.4 | muzzle speed per unit of power (power 100 → 640 u/s) |
| `BLAST_R` | 48 | crater radius and kill radius |
| `TANK_R` | 20 | tank hit radius |
| `STEP` | 1/300 s | fixed integration step |

The shell integrates on a **fixed timestep accumulator** (`STEP`), independent
of frame rate, capped at 400 substeps per frame. Each step applies wind to `vx`
and gravity to `vy`, then advances position and tests, in order:

1. **Direct hit** — within `TANK_R` of any live tank's centre `(x, y-14)`.
2. **Ground** — `y >= terrain[x]` while inside the world.
3. **Lost** — `y > H + 300`, or `|x|` more than 1400 beyond the world, at which
   point the shot is a miss. The generous horizontal margin lets a strong wind
   blow a shell back into play.

On impact the blast is resolved **before** the ground changes shape: every live
tank within `BLAST_R + TANK_R * 0.6` of the impact point is eliminated. Then the
crater is carved, so a tank is judged where it stood.

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
  which skips eliminated tanks and wraps), and the wind is re-rolled.
- **exactly one** — that player wins the round and scores a point.
- **none** — a draw, and nobody scores.

One hit eliminates a tank, so a 2-player round ends on the first hit, while 3-
and 4-player rounds continue until one tank remains. A player who blasts
themselves hands the round to the others. Scores accumulate across rounds;
**Next round** regenerates terrain and revives everyone, rotating who shoots
first.

---

## 6. Controls

Three input methods drive the same two values (`angle`, `power`), and all three
stay in sync through `syncUI`:

- **Sliders** — angle `0..180`, power `10..100`.
- **Drag anywhere on the field** (touch or mouse) — the direction from the
  turret to the finger sets the angle, and the distance sets the power
  (`10 + (dist - 60) * 90/340`, clamped). A reticle tracks the finger. Dragging
  never fires; committing a shot is always the **FIRE** button, so a stray touch
  cannot waste a turn. Pointer coordinates are mapped back through the camera
  transform, so aiming is correct at any scale or aspect ratio.
- **Keyboard** — arrows adjust angle/power (with shift for coarse steps), space
  or enter fires.

A short dashed lead line shows the aim direction, its length scaled by power. It
deliberately does **not** plot the trajectory: finding the arc is the game.

---

## 7. Layout

The page is a full-height flex column: scoreboard, playfield, controls. The
canvas fills the stage exactly and the game draws edge to edge (§2).

**Landscape only on touch devices.** Under
`(orientation: portrait) and (pointer: coarse)` the game and setup screen are
hidden and a "turn your device sideways" prompt is shown. A **Full** button
requests fullscreen and then `screen.orientation.lock('landscape')`, which is
the only way to actually pin orientation on Android Chrome; iOS does not support
it, so the rotate prompt is the fallback there.

**Floating controls on short viewports.** Under `max-height: 560px` — a phone in
landscape — the control bar becomes a translucent overlay across the bottom of
the field instead of taking space in the column. This roughly halves the chrome
and gives the battlefield the height it needs. To stop the bar covering the
action, `view.inset` measures its height in world units and §3 keeps the terrain
surface — and therefore every tank — above it.

Scores, the wind gauge, and the turn indicator are sized with `clamp()` so they
stay readable from phone to desktop. The active player is marked on the
scoreboard, and the FIRE button and slider thumbs take that player's colour.

---

## 8. Audio

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

A **Sound** toggle mutes everything and stops any whistle in flight.

---

## 9. Server and deployment

`server.js` serves `public/` over HTTP on `127.0.0.1:3005` (override with
`PORT`/`HOST`). It resolves and normalises the request path and refuses anything
that escapes the document root, sends `Cache-Control: no-cache` so a reload
always picks up a new build, and has no dependencies.

The topology follows the other `*.example.com` sites on this host:

```
browser
  └── https://tanks.example.com
        └── edge host (public nginx, terminates TLS)
              │  *.example.com wildcard vhost → upstream backend, over the private network
              └── backend host (this host)
                    └── nginx vhost tanks.example.com  (127.0.0.1:80, <private-network-address>:80)
                          └── proxy_pass 127.0.0.1:3005
                                └── tanks.service (systemd --user) → node server.js
```

- **Port 3005**, chosen because 3000–3004 were already taken on this host.
- **systemd user service.** `tanks.service` runs as an unprivileged user with
  `Restart=always` and `WantedBy=default.target`. Lingering is enabled for the
  user (`loginctl enable-linger`), so the service starts at boot with no login
  session and survives both a crash and a host restart.
- **No edge changes were needed.** the edge host already proxies the
  `*.example.com` wildcard to this host, and DNS already resolves the wildcard, so
  adding the backend vhost was sufficient to publish the new subdomain.

---

## 10. Debug handle

`window.__debug` exposes `{ G, view, fire, newRound, startGame }`. It is what the
browser-driven tests use to inspect terrain, force deterministic shots and step
rounds. It is intentionally shipped: this is a local hot-seat game with nothing
to protect, and the handle makes the game testable.
