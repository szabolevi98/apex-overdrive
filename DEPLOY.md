# Apex Overdrive — deployment guide

This is a public, general-purpose deployment guide. SSH hosts, usernames,
ports and keys in the examples are placeholders; adapt them to your own
environment. Keep production access details, server inventories and secret
storage locations out of the repository.

Project repository: `https://github.com/szabolevi98/apex-overdrive`.
Public game: `https://racing.levente.net/`.

## Requirements and architecture

- Node.js 20 or later.
- Linux with systemd; Apache and a valid certificate for HTTPS.
- Optional MySQL/MariaDB for persistent results.
- Enough disk space for car, track and environment models.

The client uses native ES modules and does not require a separate build.
Apache serves `web/` and the `/shared/` route directly, forwarding `/api/`
and `/ws` requests to the local Node.js process.

```text
browser → Apache HTTPS → web/ and shared/
                       → /api/ and /ws → Node.js on loopback
```

Use a separate VirtualHost when hosting alongside other websites.

## SSH connection

Configure an SSH host alias on your own machine. This `~/.ssh/config`
snippet is an example, not the connection details of the public game:

```sshconfig
Host apex-overdrive-server
    HostName server.example.com
    User deploy
    Port 22
    IdentityFile ~/.ssh/example-deploy-key
    IdentitiesOnly yes
```

The remaining commands use this local alias. Never copy private keys into
the repository. Use a separate server-side deploy key for GitHub; read-only
access is sufficient if the server only downloads code. Verify the server's
SSH host key through a trusted channel.

## Code and assets

The examples use `/opt/racing` as the installation directory. If you choose
another path, adjust the systemd and Apache templates too.

```bash
git clone git@github.com:szabolevi98/apex-overdrive.git /opt/racing
cd /opt/racing
npm ci --omit=dev
cp .env.example .env
```

Large binary assets are excluded from Git: upload GLB, GLTF, HDR, EXR, BIN
and track textures separately. Hand-authored metadata such as spawn points,
gates, pit settings and zone maps is versioned.

From your local machine, with appropriate write permissions at the destination:

```bash
rsync -avP web/assets/ apex-overdrive-server:/opt/racing/web/assets/
```

Individual files can also be uploaded with `scp`. Create destination
directories first. New tracks require `collision.bin`, and cars need their
remote/ghost models under `web/assets/cars/compressed/` too. One car, one
track and one environment are sufficient to check an initial installation.

## Environment and database

Fill in `.env` only on the server and keep it out of Git. The password below
is explicitly a placeholder:

```ini
HOST=127.0.0.1
PORT=3000
ALLOW_DEV_WRITES=0
DB_HOST=127.0.0.1
DB_PORT=3306
DB_NAME=example_game
DB_USER=example_game_user
DB_PASSWORD=REPLACE_WITH_A_UNIQUE_SECRET
```

Create a separate database and a user with access only to that database.
The application creates its tables at startup. It can run without a
database, but results will not be persisted.

Set `ALLOW_DEV_WRITES=0` in production: developer save endpoints are not
intended for public use. `HOST=127.0.0.1` keeps Node.js behind the proxy;
do not expose its port directly to the internet.

Run the service as a dedicated system user without an interactive login.
It should be able to read the application code, but not modify it. Restrict
`.env` to the administrator and service user, for example with mode `640`.

## systemd

Use [deploy/systemd/racing.service](deploy/systemd/racing.service) as a
template. Adjust its system user and working directory, then install it in
the appropriate systemd directory.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now racing
sudo systemctl status racing
journalctl -u racing -n 50
```

The application reads `.env` from its working directory using `dotenv`.
The service template therefore does not specify a separate `EnvironmentFile`.

## Apache and HTTPS

VirtualHost template:
[deploy/apache/racing.levente.net-le-ssl.conf](deploy/apache/racing.levente.net-le-ssl.conf).
For another domain, adjust `ServerName`, certificate paths and directories.

Enable the required modules:

```bash
sudo a2enmod proxy proxy_http proxy_wstunnel ssl rewrite headers deflate
sudo apache2ctl configtest
```

Configure:

- `DocumentRoot`: the project's `web/` directory.
- `Alias /shared/`: the project's `shared/` directory.
- `/ws`: `ws://127.0.0.1:3000/ws`.
- `/api/`: `http://127.0.0.1:3000/api/`.
- Serve the root path through Apache rather than the Node.js proxy.
- Preserve appropriate MIME types, range support and cache headers for large assets.

Configure DNS and a valid TLS certificate for your domain, with verified
automatic renewal. Redirect HTTP requests to HTTPS. After a successful
configuration check, apply the changes:

```bash
sudo systemctl reload apache2
```

## Updates

On the server, as the user responsible for deployment:

```bash
cd /opt/racing
git pull --ff-only
# Only when dependencies change:
npm ci --omit=dev
# After server-code or car/track asset changes:
sudo systemctl restart racing
sudo systemctl is-active racing
```

The server caches its asset manifest, so new or modified game assets require
a restart. Git pull does not transfer models and collision files excluded
from Git. Updates limited to static branding files do not require a restart.

After uploading, compare local and remote file sizes or SHA-256 hashes.
Content-versioned URLs refresh the cache for large assets.

## Validation and troubleshooting

Replace the example host with your deployment's public domain:

```bash
curl -I https://game.example.com/
curl https://game.example.com/api/status
curl https://game.example.com/api/assets
```

The homepage and API should return HTTP 200, and the manifest should contain
the current assets. WebSocket upgrades should return
`101 Switching Protocols`; asset range requests should return `206`.
Also check the menu and multiplayer connection in a browser.

- Homepage errors: check the VirtualHost, file permissions and Apache logs.
- API errors: check the service, Node.js logs and loopback API access.
- WebSocket errors: check proxy modules and `/ws` rule ordering.
- Empty asset lists: check uploads, read access and manifest refresh.
- Missing track collisions: upload the matching `collision.bin`.

Store your server inventory, connection details and private operational
notes separately from the public repository.
