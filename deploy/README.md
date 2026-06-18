# Deploy

Spliti runs as its own systemd service behind the shared `fucku` cloudflared
tunnel, independent of the fucku GitHub App.

## Service

`spliti-app.service` runs `uvicorn spliti.app:split_app` on `127.0.0.1:8001`.

```sh
sudo cp deploy/spliti-app.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now spliti-app.service
```

## Cloud Firestore mirror (optional)

The unit wires `FIRESTORE_PROJECT_ID` / `FIRESTORE_CREDENTIALS` so each write is
mirrored to Cloud Firestore (SQLite stays the source of truth; see
`spliti/firestore_sync.py`). To enable it on the box:

```sh
# 1. install the extra dependency into the service venv
/home/ubuntu/spliti/.venv/bin/pip install "google-cloud-firestore>=2.16"

# 2. drop the service-account key where the unit points, locked down
install -m 600 gcp_sa.json /home/ubuntu/spliti/gcp_sa.json

# 3. (re)deploy the unit and restart
sudo cp deploy/spliti-app.service /etc/systemd/system/
sudo systemctl daemon-reload && sudo systemctl restart spliti-app.service
```

On startup the service back-fills all existing groups, then keeps Firestore
converged on every write. To run on SQLite alone, comment out the two
`Environment=FIRESTORE_*` lines. The service account needs the
`roles/datastore.user` IAM role on the project (Firestore Security Rules don't
apply to this server-side/admin access).

## Tunnel routing

The cloudflared tunnel (`~/.cloudflared/config.yml`, service
`cloudflared-fucku.service`) routes the Spliti hostnames to this service while
`bot.codexvault.org` stays on fucku (`:8000`):

```yaml
ingress:
  - hostname: bot.codexvault.org      # fucku
    service: http://127.0.0.1:8000
  - hostname: spliti.codexvault.org   # Spliti
    service: http://127.0.0.1:8001
  - service: http_status:404
```

After editing the config: `sudo systemctl restart cloudflared-fucku.service`.

> Note: fucku's `app/main.py` still has dead `app.host("spliti…")` mounts. They
> no longer receive traffic and can be removed from fucku independently.
