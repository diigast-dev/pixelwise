# PixelWise

PixelWise is a handwritten digit classifier. Visitors draw a digit on a canvas in the
browser; the drawing is sent to a scikit-learn model, which predicts the digit and shows
the result, alongside per-class confidence scores. Past predictions are persisted to
PostgreSQL and exposed through a small read API.

## Architecture

```
 Browser (canvas pad)
        │  POST /api/classify
        ▼
      nginx  ──────────────────────────────┐
        │  static files / PHP-FPM          │ proxy_pass /api/
        ▼                                  ▼
 Sulu CMS (Symfony, PHP 8.5)        FastAPI + uvicorn (:8000)
        │                                  │
        ▼                                  ▼
   PostgreSQL  ◄────────────────  scikit-learn model + SQLAlchemy
```

- **Frontend** — a [Sulu](https://sulu.io/) (Symfony) CMS page with a 28×28 canvas
drawing pad (`sulu/public/js/app.js`). Strokes are downsampled and POSTed as pixel
data to the backend.
- **Backend** (`app/`) — a FastAPI service (`app/main.py`) that loads a pickled
scikit-learn pipeline (`app/classifier.py`) to classify the digit, then logs the
prediction (`app/models.py`) to PostgreSQL via SQLAlchemy.
- **Reverse proxy** — nginx serves the Sulu frontend directly and proxies `/api/` to the
FastAPI backend (`deploy/pixelwise.nginx`).
- **Deployment** — `setup-server.sh` provisions a full Ubuntu server in one pass
(packages, database, nginx, systemd). `deploy/pixelwise.service` runs the API under
systemd, and a systemd timer (`deploy/systemd/pixelwise-deploy.{service,timer}`) polls
`origin/main`, runs the test suite, and restarts the service on green commits.

## API


| Endpoint    | Method | Description                                                                                   |
| ----------- | ------ | --------------------------------------------------------------------------------------------- |
| `/health`   | GET    | Liveness check, returns the loaded model version.                                             |
| `/classify` | POST   | Classifies a 28×28 pixel grid, returns the predicted digit, confidence, and per-class scores. |
| `/results`  | GET    | Returns the 20 most recent predictions.                                                       |


## Project layout

```
app/                  FastAPI backend (classifier, models, API routes)
sulu/                 Sulu/Symfony CMS frontend
deploy/               nginx site, systemd units, auto-deploy script
setup-server.sh        One-shot Ubuntu server provisioning script
init_db.py             Creates the predictions table
predict.py              Standalone script to sanity-check the model against MNIST samples
.env.example            Backend environment variable template
installation.md         Full installation guide
```

## Quick start

```bash
sudo git clone <repository-url> /opt/pixelwise
cd /opt/pixelwise
cp .env.example .env        # then edit .env
cp sulu/.env.example sulu/.env
./setup-server.sh
```

See `[installation.md](installation.md)` for the full prerequisites, environment  
variable reference, and a manual step-by-step alternative.

## Create a Sulu admin user

To log into the Sulu admin panel, create a user from `/var/www/pixelwise` (or
`sulu/` in a local checkout):

```bash
bin/console sulu:security:user:create
```

The command prompts interactively for username, email, first/last name, password, and
role.
