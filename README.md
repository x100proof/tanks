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
- **Drag the battlefield** — direction sets the angle, distance sets the power.
  Dragging never fires; press **FIRE!** to shoot.
- **Keyboard** — `←`/`→` angle, `↑`/`↓` power (hold `shift` for bigger steps),
  `space` to fire.

The battlefield is **square**, so the game plays the same held upright or
sideways, on a phone or a desktop — it never asks you to rotate anything. The
**Full** button in the top-right corner goes full screen, and turns into
**Exit** to come back (Escape works too).

## Run locally

```bash
node server.js       # http://127.0.0.1:3005
```

## Design

The design lives in [`architecture.md`](architecture.md) and is kept in step
with the implementation — see [`CLAUDE.md`](CLAUDE.md).

## Deployment (backend host)

Follows the battleship/kidage pattern — see `deploy/`:

- **systemd user service**: `deploy/tanks.service` →
  `~/.config/systemd/user/tanks.service`

  ```bash
  cp deploy/tanks.service ~/.config/systemd/user/tanks.service
  systemctl --user daemon-reload
  systemctl --user enable --now tanks.service
  ```

  Lingering is enabled for this user (`loginctl enable-linger`), so the service
  starts at boot without a login session, and `Restart=always` brings it back if
  the process dies.

- **nginx vhost**: `deploy/tanks.nginx` →
  `/etc/nginx/sites-available/tanks.example.com`, symlinked into `sites-enabled`.

  ```bash
  sudo cp deploy/tanks.nginx /etc/nginx/sites-available/tanks.example.com
  sudo ln -sfn /etc/nginx/sites-available/tanks.example.com /etc/nginx/sites-enabled/tanks.example.com
  sudo nginx -t && sudo systemctl reload nginx
  ```

  It listens on localhost and the tailnet IP and proxies to `127.0.0.1:3005`.
  The public front end (the edge host, see `~/code/sysadmin`)
  terminates TLS for `*.example.com` and proxies here over the private network; its wildcard
  vhost already covers this subdomain, so no change was needed there.
