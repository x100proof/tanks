# Tank Duel

A hot-seat artillery game for 2–4 players on one device, hosted at
<https://tanks.example.com>.

Pick how many players and name them, then take turns lobbing shells across
randomly generated hills. Set **angle** and **power** and fire. The ground is
destructible — shells blow craters, and a tank whose ground is dug out from
under it falls. **One hit knocks a player out**; the last tank standing wins the
round, and scores carry across rounds.

There is **no wind**: the same dials always produce the same shot. It is a game
for young children, and being able to nudge the power up and see exactly the
result you expected is the whole point.

Everything is drawn as vector graphics on a canvas and every sound is
synthesised — no images, no fonts, no audio files, no dependencies, no build
step. The whole game is one file: [`public/index.html`](public/index.html).

## Controls

- **Sliders** — angle (0–180°) and power (10–100). Power fills in from the left
  so you can see how strong the shot is at a glance.
- **The − and + buttons** either side of each slider nudge it one step at a
  time, for fine aiming. Hold one down to keep going.
- **Drag the battlefield** — direction sets the angle, distance sets the power.
  Dragging never fires; press **FIRE!** to shoot.
- **Keyboard** — `←`/`→` angle, `↑`/`↓` power (hold `shift` for bigger steps),
  `space` to fire.

## Setting up a game

Besides the number of players (2–4) and their names, the setup screen sets:

- **Tank size**, from 0.35× to 1.75×. The preview shows the tank at exactly the
  size it will be in the game, and a bigger tank really is a bigger target.
- **Blast radius**, from 0 to 84. The preview draws the lethal radius as a ring
  around the tank at the same scale. **Set it to 0 and only direct hits count** —
  no splash at all.

The battlefield is **square**, so the game plays the same held upright or
sideways, on a phone or a desktop — it never asks you to rotate anything. The
**Full** button in the top-right corner goes full screen, and turns into
**Exit** to come back (Escape works too).

## Playing it

Open [`public/index.html`](public/index.html) in a browser — that is the whole
game, and it runs straight off the filesystem.

To serve it instead:

```bash
node server.js       # http://127.0.0.1:3005
```

## Hosting it yourself

One self-contained HTML file, no build step, no dependencies and no server-side
state, so it will run on anything that can serve a web page — a static host, a
spare Raspberry Pi, or the bundled zero-dependency server.
[`DEPLOY.md`](DEPLOY.md) walks through the options.

## Design

The design lives in [`ARCHITECTURE.md`](ARCHITECTURE.md) and is kept in step
with the implementation — see [`CLAUDE.md`](CLAUDE.md).

