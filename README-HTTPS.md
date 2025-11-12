HTTPS setup for verspritescheduling.io

This repository contains the static scheduling landing page (served from `docs/`) and helper artifacts to enable HTTPS for a custom domain. The site is deployed automatically to GitHub Pages using a native Pages workflow.

Summary of files added
- `deploy/nginx/verspritescheduling.conf` - nginx site file (HTTP + ACME challenge). Copy to `/etc/nginx/sites-available/verspritescheduling` on the host and enable it.
- `deploy/acme/install_acme.sh` - shell script to install `acme.sh`, issue certificates using webroot `/workspaces/landingschedule/docs`, install certs to `/etc/letsencrypt/live/verspritescheduling.io`, and configure nginx reload on renewal.

Preconditions (host)
- DNS A record for `verspritescheduling.io` and `www.verspritescheduling.io` points to the host IP.
- Ports 80 and 443 are reachable from the Internet.
- nginx is installed on the host and can serve files from `/workspaces/landingschedule/docs`.
- Run the following commands on the host (not in the devcontainer):

1) Install nginx (if not present)
```bash
sudo apt update
sudo apt install -y nginx
```

2) Copy nginx site file and enable it
```bash
sudo cp deploy/nginx/verspritescheduling.conf /etc/nginx/sites-available/verspritescheduling
sudo ln -s /etc/nginx/sites-available/verspritescheduling /etc/nginx/sites-enabled/verspritescheduling || true
sudo nginx -t
sudo systemctl reload nginx
```

3) Run the acme installer script (will use webroot method)
```bash
# run on host
bash deploy/acme/install_acme.sh
```

4) Update nginx to use installed certificates (if the HTTPS block isn't already present in the site file, add this block):
```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name verspritescheduling.io www.verspritescheduling.io;

    root /workspaces/landingschedule/docs;
    index landing.html index.html;

    ssl_certificate     /etc/letsencrypt/live/verspritescheduling.io/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/verspritescheduling.io/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-Content-Type-Options nosniff;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

5) Test and reload nginx
```bash
sudo nginx -t
sudo systemctl reload nginx
```

Verification
- Open https://verspritescheduling.io in your browser
- Check certificate: `sudo openssl s_client -connect verspritescheduling.io:443 -servername verspritescheduling.io` or `sudo certbot certificates` if you used certbot

Notes
- This flow uses `acme.sh` (no snapd required). The `install_acme.sh` script installs acme.sh into the home directory and issues certificates using the repository webroot.
- The script requires `sudo` when installing certificates into `/etc/letsencrypt` and when reloading nginx.
- Renewals are handled by `acme.sh`'s cron job; the install step sets `--reloadcmd` to reload nginx after renewals.

Deployment (GitHub Pages)
-------------------------
Branch: `main`
Source: GitHub Actions workflow (`.github/workflows/deploy-pages.yml`) uploads `docs/` as a Pages artifact.
Target: GitHub Pages environment (serves from the `gh-pages` internal Pages deployment, not the historical branch contents).

Typical flow:
1. Edit files under `docs/` (e.g. `docs/index.html`).
2. Commit + push to `main`.
3. Workflow builds and publishes (usually < 2 minutes).
4. Site available at: https://kcybervs.github.io/VerSpriteScheduling/ (or your configured custom domain).

Legacy workflow `deploy-gh-pages.yml` has been disabled (converted to a no-op) to avoid double publishing.

If you'd like, I can also generate the HTTPS nginx server block automatically and apply the file edits here in the repo. If you want me to, tell me to "apply nginx HTTPS block" and I'll update `deploy/nginx/verspritescheduling.conf` with the HTTPS server and http->https redirect enabled.