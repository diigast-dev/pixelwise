# Installation

This guide explains how to install PixelWise on a fresh **Ubuntu** server. It covers
both the automated path (`setup-server.sh`) and the equivalent manual steps.

PixelWise consists of two parts that are installed together:

- the **FastAPI backend** (`app/`), served by uvicorn on port `8000` and reverse-proxied
by nginx under `/api/`,
- the **Sulu/Symfony frontend** (`sulu/`), served directly by nginx + PHP-FPM.

## Prerequisites

- A fresh Ubuntu server with `sudo` access.
- A dedicated system user named `produser`, which owns the deployment and is granted a
narrow, passwordless `sudo` rule to restart the `pixelwise` service. Create it if it
doesn't exist yet:
  ```bash
  sudo useradd -m -s /bin/bash produser
  ```
- `git` installed (`sudo apt install -y git`).

## 1. Clone the repository

The deployment scripts, the nginx config, and the systemd units all assume the project
lives at `/opt/pixelwise`. Clone it there:

```bash
sudo git clone <repository-url> /opt/pixelwise
sudo chown -R produser:produser /opt/pixelwise
cd /opt/pixelwise
```

## 2. Configure environment variables

PixelWise has **two separate `.env` files** — one for the Python backend, one for the
Symfony frontend.

### Root `.env` (Python backend + `setup-server.sh`)

```bash
cp .env.example .env
```

Then edit `.env` and set:


| Variable         | Description                                                                                                                 |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `SECRET_API_KEY` | Secret key used to authenticate frontend requests; substituted into `app.js` on deploy.                                     |
| `DEBUG`          | Set to `false` in production.                                                                                               |
| `MODEL_REPO`     | Git URL of the repository that holds the trained model.                                                                     |
| `MODEL_VERSION`  | Git tag/branch to pull from `MODEL_REPO`.                                                                                   |
| `MODEL_PATH`     | Path to the `.pkl` file after it's copied into `models/` (must match the filename in `MODEL_REPO`).                         |
| `DATABASE_URL`   | SQLAlchemy connection string for the predictions database, e.g. `postgresql://pixelwise:<password>@localhost/pixelwise`.    |
| `DB_PASSWORD`    | Password used when `setup-server.sh` provisions the `pixelwise` PostgreSQL role. Must match the password in `DATABASE_URL`. |


### `sulu/.env.local` (Symfony frontend)

```bash
cp sulu/.env.example sulu/.env
```

Then create `sulu/.env.local` (uncommitted, environment-specific overrides) and set at
least:

```dotenv
APP_ENV=prod
APP_DEBUG=0
DATABASE_URL="postgresql://pixelwise:<password>@127.0.0.1:5432/pixelwise?serverVersion=16&charset=utf8"
```

Use the same PostgreSQL credentials as the root `.env` — Sulu and the FastAPI backend
share the same database server.

## 3. Automated installation

Once both `.env` files are in place, run:

```bash
cd /opt/pixelwise
./setup-server.sh
```

The script performs the full installation in one pass:

1. Installs system packages: nginx, PHP 8.5 (+ fpm/xml/gd/pgsql/sqlite3), PostgreSQL,
  Python 3, Composer.
2. Adds `produser` to the `www-data` group.
3. Creates a Python virtual environment (`.venv`) and installs `requirements.txt`.
4. Sets permissions on `/opt/pixelwise` and symlinks `sulu/` to `/var/www/pixelwise`.
5. Substitutes `SECRET_API_KEY` into the frontend's `app.js`.
6. Installs the nginx site config and reloads nginx.
7. Runs `composer install` and builds the Sulu production environment.
8. Enables `php8.5-fpm`.
9. Pulls the trained model from `MODEL_REPO`/`MODEL_VERSION` into `models/`.
10. Installs and starts the `pixelwise` systemd service (uvicorn).
11. Provisions the `pixelwise` PostgreSQL role and database.
12. Initializes the predictions table (`init_db.py`).
13. Grants `produser` passwordless `sudo` for restarting the `pixelwise` service only.
14. Installs and enables the `pixelwise-deploy` systemd timer for auto-deployment.

After it finishes, the frontend is served on port 80 and the API is reachable under
`/api/`.

> **Note:** `deploy/pixelwise.nginx` hard-codes `server_name 192.168.64.2`. Change this
> to your server's actual hostname or IP before (or after) running the installer, and
> reload nginx (`sudo nginx -t && sudo systemctl reload nginx`).

## 4. Manual installation

If you prefer to install step by step instead of running `setup-server.sh`:

- Install system dependencies:
  ```bash
  sudo apt update
  sudo apt install -y git python3 python3-pip python3-venv curl \
      postgresql nginx php8.5 php8.5-fpm php8.5-xml php8.5-gd \
      php8.5-pgsql php8.5-sqlite3 composer
  ```
- Add `produser` to the `www-data` group, so it can read/write the files nginx and
  PHP-FPM need access to:
  ```bash
  sudo usermod -aG www-data produser
  ```
- Create the Python virtual environment and install dependencies:
  ```bash
  python3 -m venv .venv
  source .venv/bin/activate
  pip install -r requirements.txt
  ```
- Provision the PostgreSQL role and database (matching `DB_PASSWORD` from `.env`):
  ```bash
  sudo -u postgres psql -c "CREATE USER pixelwise WITH PASSWORD '<DB_PASSWORD>';"
  sudo -u postgres createdb -O pixelwise pixelwise
  ```
- Set permissions on the project, then symlink the frontend and install the nginx site:
  ```bash
  sudo chgrp -R www-data /opt/pixelwise
  sudo chmod -R g+rX /opt/pixelwise
  sudo chmod -R g+rwX /opt/pixelwise/sulu/var
  sudo ln -sfn /opt/pixelwise/sulu /var/www/pixelwise
  sudo cp deploy/pixelwise.nginx /etc/nginx/sites-available/pixelwise
  sudo ln -sf /etc/nginx/sites-available/pixelwise /etc/nginx/sites-enabled/pixelwise
  sudo rm -f /etc/nginx/sites-enabled/default
  sudo nginx -t
  sudo systemctl reload nginx
  ```
  Remember to update `server_name` in `deploy/pixelwise.nginx` for your host first.
- Install PHP dependencies and build Sulu for production:
  ```bash
  cd /var/www/pixelwise
  composer install

  bin/console sulu:build prod --no-interaction
  OR
  bin/console sulu:build dev --no-interaction

  cd /opt/pixelwise
  ```
- Enable PHP-FPM:
  ```bash
  sudo systemctl enable --now php8.5-fpm
  ```
- Pull the model and initialize the database:
  ```bash
  mkdir -p models/
  git clone --depth 1 --branch "$MODEL_VERSION" "$MODEL_REPO" /tmp/pixelwise-model
  cp /tmp/pixelwise-model/*.pkl models/
  rm -rf /tmp/pixelwise-model
  python init_db.py
  ```
- Install and start the API service:
  ```bash
  sudo cp deploy/pixelwise.service /etc/systemd/system/pixelwise.service
  sudo systemctl daemon-reload
  sudo systemctl enable --now pixelwise
  ```
- (Optional) Grant `produser` passwordless `sudo` to restart the `pixelwise` service —
required for the auto-deploy timer below, which restarts the service as `produser`:
  ```bash
  sudo tee /etc/sudoers.d/pixelwise >/dev/null <<'EOF'
produser ALL=(root) NOPASSWD: /usr/bin/systemctl restart pixelwise
EOF
  sudo chmod 0440 /etc/sudoers.d/pixelwise
  sudo visudo -cf /etc/sudoers.d/pixelwise
  ```
- (Optional) Install the auto-deploy timer, which pulls `main`, runs the test suite,
and restarts the service on every green commit:
  ```bash
  sudo cp deploy/systemd/pixelwise-deploy.service /etc/systemd/system/
  sudo cp deploy/systemd/pixelwise-deploy.timer /etc/systemd/system/
  sudo systemctl daemon-reload
  sudo systemctl enable --now pixelwise-deploy.timer
  ```
  Auto-deploy only runs `pytest` if a `tests/` directory exists in the repository root.

## 5. Verify the installation

```bash
# Backend health check
curl http://localhost:8000/health

# Through the nginx proxy
curl http://localhost/api/health

# Service status
sudo systemctl status pixelwise --no-pager
sudo systemctl status php8.5-fpm --no-pager
```

Then open `http://<server>/` in a browser — you should see the Sulu site with the
digit-drawing pad, which submits to `/api/classify`.
