# Hosting your own copy

Tank Duel is a single self-contained HTML file. There is **no build step, no
dependencies, no database, and no server-side state** — every part of the game
runs in the browser. That makes it about as easy to host as a web page gets.

Pick whichever of these suits you; they are in order of least effort.

---

## 1. Just open the file

```bash
git clone https://github.com/x100proof/tanks.git
cd tanks
xdg-open public/index.html      # or: open public/index.html   (macOS)
```

The game works straight off the filesystem. Nothing is fetched over the
network, so there is nothing to break.

This is fine for playing at home. Use one of the options below if you want it
on a real URL, or on a phone or tablet.

## 2. Any static web host

Everything the game needs is in `public/`. Upload that directory anywhere that
serves static files — GitHub Pages, Netlify, Cloudflare Pages, S3, or a
`public_html` on shared hosting. There is no configuration to do.

To try it locally first, any static server will do:

```bash
python3 -m http.server 8000 --directory public
# then visit http://localhost:8000
```

## 3. The bundled Node server

The repository includes `server.js`, a zero-dependency static file server, if
you would rather not install anything else:

```bash
node server.js                  # http://127.0.0.1:3005
```

Two environment variables configure it:

| Variable | Default | Meaning |
|---|---|---|
| `PORT` | `3005` | port to listen on |
| `HOST` | `127.0.0.1` | address to bind |

`127.0.0.1` only accepts connections from the same machine. To reach it from a
phone on the same wifi, bind to all interfaces:

```bash
HOST=0.0.0.0 PORT=8080 node server.js
```

…then browse to `http://<your-computer's-ip>:8080` from the phone. Do this only
on a network you trust; see *Exposing it to the internet* below.

---

## Keeping it running

To have the game come back after a reboot or a crash, run it under a service
manager. On Linux with systemd, a **user** service needs no root:

```ini
# ~/.config/systemd/user/tanks.service
[Unit]
Description=Tank Duel
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=%h/tanks
Environment=PORT=3005
ExecStart=/usr/bin/node server.js
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now tanks.service
systemctl --user status tanks.service
```

By default a user service stops when you log out. To let it run without a login
session — which is what you want on a server — enable lingering once:

```bash
sudo loginctl enable-linger "$USER"
```

Adjust `WorkingDirectory` to wherever you cloned the repository, and
`ExecStart` if `node` lives elsewhere (`command -v node` will tell you). If you
installed Node through a version manager such as nvm, systemd will not see it on
its `PATH`; give the absolute path, or start it through a login shell:

```ini
ExecStart=/bin/bash -lc 'exec node server.js'
```

## Putting it on a domain

If you want a proper hostname and HTTPS, run a web server in front of it.

**Serving the files directly** — simplest, and no Node process at all:

```nginx
server {
    listen 80;
    server_name tanks.example.com;
    root /srv/tanks/public;
    index index.html;
}
```

**Or proxying to `server.js`**, if you are already running it:

```nginx
server {
    listen 80;
    server_name tanks.example.com;

    location / {
        proxy_pass http://127.0.0.1:3005;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

For HTTPS, [certbot](https://certbot.eff.org/) will obtain a certificate and
rewrite the config for you:

```bash
sudo certbot --nginx -d tanks.example.com
```

Apache, Caddy or anything else works just as well — there is nothing unusual
about what is being served. Caddy in particular gets you HTTPS with two lines:

```
tanks.example.com {
    root * /srv/tanks/public
    file_server
}
```

## Exposing it to the internet

The game keeps no accounts, no scores on the server, and no data of any kind —
there is nothing in it to steal. The usual cautions still apply to the machine
it runs on: prefer a proper web server on 80/443 over binding `server.js` to a
public interface, keep HTTPS on, and do not open a port on your home router
unless you understand what that exposes.

---

## Updating

```bash
git pull
systemctl --user restart tanks.service    # only if you use the Node server
```

The server sends `Cache-Control: no-cache`, so a reload always picks up a new
version. On a static host, clear the CDN cache if it has one.

## Notes

- **Sound** needs a user gesture before it can start, which the **Start battle**
  button provides. Nothing to configure.
- **Full screen** uses the Fullscreen API. It is unavailable on iOS Safari, where
  the button hides itself; every other modern browser supports it.
- **No analytics, no telemetry, no external requests.** The page is genuinely
  self-contained, which is also why it works offline.

The design of the game itself is in [`architecture.md`](architecture.md).
