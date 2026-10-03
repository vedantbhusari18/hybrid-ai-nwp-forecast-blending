# Hybrid AI–NWP Forecast Blending

An operational-style workspace that blends physical NWP, AI weather models, and an ensemble mean into one forecast. Weights change with **region**, **season**, **lead time**, and **variable**, using historical skill. The demo runs on a deterministic synthetic archive over India so it works without GFS, ECMWF, or ERA5 access.

On that archive (seed 42, last 10 days of each season held out) the blend lowers normalized MAE by about **11%** versus the best single source, and it wins on rainfall, minimum temperature, maximum temperature, and wind separately.

| Variable | MAE reduction vs best bias-corrected model |
| --- | --- |
| Rainfall | 15% |
| Minimum temperature | 12% |
| Maximum temperature | 17% |
| Wind speed | 7% |

The strongest single source in the archive is the ensemble mean, because it is already a smoothed equal-weight mix. Adaptive weights still beat that flat mix.

## What you can do in the UI

- **Dashboard** — blended rainfall, min/max temperature, or wind on a map; lead time from 0 to 168 h; optional overlay of which model has the highest weight; click a cell for the lead-time trace.
- **Models & weights** — MAE, RMSE, bias, and CRPS; skill versus lead time; a map of the locally preferred model by season and lead.
- **Extremes** — heavy rain (≥ 64.5 mm), heat (≥ 40 °C), and high wind (≥ 15 m/s), with probability and model-agreement.
- **Operations** — job history, logs, and **Run blending now**. A scheduler repeats the cycle every 6 hours.
- **Settings** — region, metric/imperial display units, light/dark theme.

Sign in with `analyst@example.com` / `Forecast#2026`.

## Architecture

```text
frontend (Next.js)  --HTTP/JWT-->  backend (FastAPI)
                                      |
                                      +-- blender/
                                      |     data_loader -> skill_estimator
                                      |     weight_model -> blender -> extremes
                                      |
                                      +-- SQLite (users, skill scorecard, jobs)
                                      +-- APScheduler (every 6 hours)
```

```mermaid
flowchart LR
  sources[GFS ECMWF AI-A AI-B Ensemble] --> bias[Bias correction]
  history[Training window skill] --> weights[Skill softmax plus gating net]
  bias --> mix[Weighted consensus]
  weights --> mix
  mix --> field[Blended field]
  mix --> extremes[Extreme probabilities]
  history --> score[Held-out MAE RMSE CRPS]
```

### How the weights work

1. On training days only, remove each model's mean error for that season, lead, and grid cell.
2. Measure MAE, smooth it across neighbouring cells, and shrink it toward the domain mean so one noisy tile cannot dominate.
3. Turn MAE into weights with a softmax: lower error, higher weight. Weights always sum to 1.
4. Mix in a small linear gating network (14%) trained on lead time, latitude, longitude, season, and variable. This is the mixture-of-experts hook.
5. The published forecast is the weighted mean of the bias-corrected members. Wind direction uses the same weights as a vector mean.

Extreme probabilities treat each bias-corrected member as the center of a Gaussian whose width is that member's historical MAE, then average the threshold probabilities with the same weights. Confidence is higher when the members agree. Rainfall is skewed, so this is guidance, not a calibrated warning.

## Repository layout

```text
frontend/          Next.js 14 app (App Router, Tailwind, Leaflet, Recharts)
backend/app/       FastAPI routers, JWT auth, jobs, database
backend/blender/   data loading, skill, weights, blending, extremes
backend/scripts/   train_weights.py
infra/             Dockerfiles and compose file
docker-compose.yml
```

## Run with Docker

From the repository root:

```bash
docker compose up --build
```

- App: http://localhost:3000
- API docs: http://localhost:8000/docs

The first backend start trains the weight model (about 10 seconds) before the health check passes. `NEXT_PUBLIC_API_URL` is baked into the frontend at image build time and defaults to `http://localhost:8000`, which is what the browser on your machine should call.

## Run locally

Python 3.11+ and Node 20+.

**API**

```bash
cd backend
python -m venv .venv
```

Windows PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
copy .env.example .env
python scripts/train_weights.py
uvicorn app.main:app --reload --port 8000
```

macOS / Linux:

```bash
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
python scripts/train_weights.py
uvicorn app.main:app --reload --port 8000
```

`train_weights.py` writes `backend/data/artifacts.npz`, `gating_weights.npz`, and `scorecard.json`. The API also retrains on startup from the same seed, so the demo does not depend on those files being present.

**UI**

```bash
cd frontend
npm install
npm run dev
```

Open http://localhost:3000.

## Tests

```bash
cd backend && python -m pytest
cd frontend && npm test
```

Backend tests check that weights sum to 1, that the blend beats every model on held-out normalized MAE and on each variable, that extreme probabilities stay inside 0–1, and that the API auth and forecast routes respond. Frontend tests cover unit conversion, compass labels, and the color ramp.

## API

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `/api/auth/register` | Create an account |
| POST | `/api/auth/login` | Email and password, returns JWTs |
| POST | `/api/auth/refresh` | Refresh token |
| GET | `/api/auth/me` | Current user |
| GET | `/api/forecasts/blended` | Blended GeoJSON for a variable, lead, region, init |
| GET | `/api/forecasts/series` | Lead-time trace at a lat/lon |
| GET | `/api/forecasts/models` | Model catalogue and headline skill |
| GET | `/api/weights/maps` | Dominant-model weight map |
| GET | `/api/metrics/skill` | MAE, RMSE, CRPS, bias |
| GET | `/api/extremes` | Extreme-weather probabilities |
| GET | `/api/extremes/overview` | All hazards across lead times |
| POST | `/api/jobs/run` | Run a blending cycle now |
| GET | `/api/jobs` | Schedule and history |

All routes except `/api/health` and `/api/auth/*` require `Authorization: Bearer <access_token>`.

## Using real forecasts

Set `DATA_SOURCE=directory` and `FORECAST_DIR` to a folder that contains `cube.npz`. The expected arrays are documented at the top of `backend/blender/data_loader.py`.

Practical path:

1. Pull GFS and ECMWF GRIB (or ECMWF Open Data) and AI outputs such as GraphCast or Pangu NetCDF.
2. Regrid every source onto one lat/lon grid. xarray + cfgrib is the usual tool.
3. Use ERA5, IMDAA, or IMD gridded observations as `truth`, aligned to the same valid times.
4. Keep the axis order `(date, lead, lat, lon, variable, model)` and mark a training/test split that does not leak the evaluation days into the bias or the weights.
5. Point the API at that directory. `train_weights.py` and the startup job will refit without code changes.

To replace the weight model, edit `AdaptiveWeightModel` in `backend/blender/weight_model.py`. A gradient-boosted gate or a small neural net can implement the same `fit` / `predict` contract. `blender.py` only consumes the final weight array.

## Deploying

SQLite and APScheduler are the demo defaults. For a shared deployment:

- Set `DATABASE_URL` to PostgreSQL, for example `postgresql+psycopg://blend:blend@host:5432/hybrid`. Add `psycopg[binary]` to `requirements.txt` when you do this.
- Put the API and the web app on Render, Railway, Cloud Run, or ECS. Build the frontend with `NEXT_PUBLIC_API_URL` set to the public API origin.
- Store `JWT_SECRET` as a secret. Change the demo password.
- Move `execute_blend_job` in `backend/app/services/runtime.py` onto Celery or Cloud Tasks if cycles must survive multiple web replicas. Redis is the broker; the function body stays the same.
- Mount a disk for the forecast directory, or read GRIB from object storage inside `data_loader.py`.

Do not treat the extreme probabilities as public warnings until they are checked against real observations.

## Stack

Next.js 14, TypeScript, Tailwind CSS, Radix primitives (shadcn-style), TanStack Query, Zustand, Leaflet, Recharts, FastAPI, Pydantic, SQLAlchemy, PyJWT, NumPy, APScheduler.
