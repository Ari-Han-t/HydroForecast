# HydroForecast

HydroForecast is a full-stack water tank monitoring and forecasting project that combines a Node.js dashboard with a Python prediction service. The application lets users register tanks, record water-level logs, review tank history, and open a prediction view that combines groundwater estimation, rainfall forecast data, and future tank-level forecasting.

The repository currently contains two connected applications:

- A Node.js + Express + MongoDB + EJS web app for authentication, tank management, and log management.
- A FastAPI + Prophet + geospatial analysis service that generates prediction data for a selected tank.

## What the project currently supports

The codebase already implements these user-facing flows:

- User sign-up and login with JWT-based authentication.
- Cookie-based session handling for the browser dashboard.
- Tank registration with name, capacity, location, current level, unit, and alert threshold.
- Dashboard listing for all tanks owned by the logged-in user.
- Manual tank log creation.
- Sensor-style automatic tank log creation through a dedicated API route that converts a measured distance into estimated stored volume.
- Viewing historical tank logs for each tank.
- Tank deletion.
- Prediction page for each tank that shows:
	- groundwater level estimate based on the tank location,
	- rainfall forecast from Open-Meteo,
	- future tank-level predictions from Prophet.

## Architecture

### 1. Web app

- Framework: Express
- Database: MongoDB via Mongoose
- Templating: EJS
- Auth: JWT in HTTP-only cookies, with header fallback for API clients
- UI: Server-rendered pages with vanilla JavaScript

### 2. Prediction service

- Framework: FastAPI
- Database access: PyMongo
- Forecasting: Prophet
- Geocoding: geopy with Nominatim
- Spatial lookup: KDTree over a groundwater CSV dataset
- Weather source: Open-Meteo daily precipitation forecast

### 3. Runtime interaction

1. The browser talks to the Node.js app.
2. The dashboard exposes a prediction link for a selected tank.
3. The Node.js route `/prediction/:tankId` calls the Python FastAPI endpoint at `http://127.0.0.1:8000/prediction`.
4. The Python service reads tank data and logs from MongoDB, performs enrichment and forecasting, and returns prediction data.
5. The Node.js app renders the prediction page using EJS and Chart.js.

## Repository structure

```text
HydroForecast/
├── package.json
├── backend/
│   ├── server.js
│   ├── config/
│   ├── controllers/
│   ├── middleware/
│   ├── models/
│   ├── public/
│   ├── routes/
│   ├── utils/
│   └── views/
└── AI/
		├── application.py
		├── requirements.txt
		├── setup.py
		├── test.py
		├── notebooks/
		└── src/
				├── components/
				├── pipeline/
				├── exception.py
				├── logger.py
				└── utils.py
```

## Data model overview

### User

- `name`
- `email`
- `password` (hashed before save)
- `location`
- `createdAt`

### Tank

- `user`
- `name`
- `capacity`
- `currentLevel`
- `location`
- `unit` (`liters`, `gallons`, `cubic_meters`)
- `alertThreshold`
- `status` (`active`, `inactive`, `maintenance`)
- `createdAt`
- `lastUpdated`
- virtual field: `percentageFilled`

### TankLog

- `tank`
- `user` (optional in the schema)
- `currentLevel`
- `rainfall`
- `usage`
- `timestamp`
- `notes`
- `logType`

## Frontend pages

- `/login`: login page for existing users.
- `/signup`: account creation page.
- `/dashboard`: authenticated dashboard for tank management.
- `/prediction/:tankId`: analytics page for one tank, including predicted tank levels and rainfall forecast charts.

## API overview

### Authentication

- `POST /api/auth/signup`
- `POST /api/auth/login`
- `POST /api/auth/logout`
- `GET /api/auth/me`

### Tanks

- `POST /api/tanks`
- `GET /api/tanks`
- `GET /api/tanks/:id`
- `PUT /api/tanks/:id`
- `DELETE /api/tanks/:id`

### Tank logs

- `POST /api/tanklogs`
- `GET /api/tanklogs/:tankId`
- `DELETE /api/tanklogs/:tankId/clear`
- `DELETE /api/tanklogs/:logId`
- `POST /api/tanklogs/auto`

### Prediction bridge

- `GET /prediction/:tankId` renders the prediction page through the Node.js app.
- `POST http://127.0.0.1:8000/prediction` is the Python service endpoint used by the web app.

## Local setup

## Prerequisites

- Node.js 18 or newer
- Python 3.10 or 3.11 recommended
- MongoDB instance

## 1. Install Node.js dependencies

The repository has a root `package.json`, so install JavaScript dependencies from the project root.

```bash
npm install
```

## 2. Install Python dependencies

From the `AI` directory:

```bash
cd AI
python -m venv .venv
source .venv/Scripts/activate
pip install -r requirements.txt
pip install scipy prophet
```

Notes:

- `scipy` and `prophet` are imported by the code but are not listed in `AI/requirements.txt`, so they need to be installed separately unless you update the dependency file.
- If you are using PowerShell instead of Git Bash, activate the virtual environment with `.venv\Scripts\Activate.ps1`.

## 3. Environment variables

The code expects environment variables for both the backend and the AI service.

### Backend environment

Create a `.env` file in the project root if you run the backend with `node backend/server.js`:

```env
MONGO_URI=mongodb://127.0.0.1:27017/hydroforecast
JWT_SECRET=replace_with_a_secure_secret
PORT=5000
NODE_ENV=development
```

### AI service environment

Create an `AI/.env` file if you run the FastAPI service from inside the `AI` folder:

```env
MONGO_URI=mongodb://127.0.0.1:27017/hydroforecast
WEATHER_API_KEY=optional_currently_unused
```

Notes:

- `WEATHER_API_KEY` is loaded in the Python service but is not currently used by the implemented weather call.
- Both services must point to the same MongoDB database.

## 4. Required local artifact for groundwater lookup

The prediction service expects this CSV file to exist:

```text
AI/src/components/artifacts/underground.csv
```

The CSV must include at least these columns:

- `LATITUDE`
- `LONGITUDE`
- `WL(mbgl)`

Without this file, the groundwater lookup path will fail.

## 5. Start the Python prediction service

From the `AI` directory:

```bash
uvicorn application:app --reload --host 127.0.0.1 --port 8000
```

## 6. Start the Node.js app

From the project root:

```bash
node backend/server.js
```

For development with reload:

```bash
npx nodemon backend/server.js
```

The dashboard will be available at:

```text
http://localhost:5000
```

## End-to-end usage flow

1. Start MongoDB.
2. Start the FastAPI service on port 8000.
3. Start the Express app on port 5000.
4. Open `http://localhost:5000`.
5. Sign up or log in.
6. Create a tank with a valid location.
7. Add a few manual logs.
8. Open the prediction page for that tank.

Important: the prediction logic requires enough historical log data for Prophet to work well. The implementation currently raises an error when fewer than 5 values are available.

## Prediction pipeline behavior

When the prediction endpoint is called, the Python service performs these steps:

1. Look up the selected tank in MongoDB.
2. Read the tank location.
3. Geocode that location using Nominatim.
4. Load the groundwater CSV and find the nearest groundwater record using KDTree.
5. Fetch rainfall forecast data from Open-Meteo.
6. Read historical tank logs from MongoDB.
7. Forecast future tank levels using Prophet.
8. Return structured prediction data to the Express app for rendering.

## Notes about the automatic log route

The route `POST /api/tanklogs/auto` is designed for a sensor-style input flow. It expects a `currentLevel` value that is treated as a distance from the top of a fixed-height tank, then converts that distance into water volume using:

- fixed tank height of `3`
- `distanceFromTop = currentLevel / 100`
- computed stored volume based on tank capacity

This is useful to understand before integrating real hardware, because the field name `currentLevel` in that route is not a direct stored water volume.

## Current implementation gaps and caveats

This README is intentionally honest about the current state of the repository.

### 1. Weather monitoring in `backend/server.js` is not currently functional

The weather monitoring service is intended to periodically create rainfall-based logs, but the HTTP request used to fetch forecast data is commented out while the code still references `response`. As written, that path cannot work reliably.

### 2. Groundwater predictions depend on a missing dataset artifact

The repository references `AI/src/components/artifacts/underground.csv`, but that file is not present in the current codebase snapshot.

### 3. Some Python support files are stubs or incomplete

- `AI/src/utils.py` is empty.
- `AI/src/pipeline/dataIngestionPipeline.py` is empty.
- `AI/src/pipeline/dataTransformationPipeline.py` is empty.
- `AI/src/pipeline/predictionPipeline.py` is empty.

### 4. Tank log routes are currently public

The log routes do not use the authentication middleware, so any client with access to the API can currently create, read, or delete logs if they know the identifiers.

### 5. Dependency metadata is incomplete

The Python code imports packages that are not fully represented in `AI/requirements.txt`, and the JavaScript root `package.json` does not currently define start scripts.

### 6. Prediction requires enough history

The Prophet-based prediction path requires at least 5 log entries. Tanks with very little historical data will not produce a forecast.

## Development notes

- The backend uses root-level JavaScript dependencies from `package.json`.
- The backend serves EJS pages directly and also exposes JSON APIs.
- The prediction page uses Chart.js loaded from a CDN.
- The AI service talks directly to MongoDB rather than through the Node.js API.
- `AI/test.py` is a manual exploratory script, not a full automated test suite.

## Suggested improvements

If you plan to continue this project, the most valuable next fixes are:

1. Restore or rewrite the backend weather-monitoring request path.
2. Add the missing groundwater dataset or document how to obtain it.
3. Protect tank-log routes with authentication and ownership checks.
4. Add proper start scripts to `package.json`.
5. Add automated tests for both the API and the prediction service.
6. Move hardcoded URLs and CORS origin values into environment variables.

## Summary

HydroForecast is best understood as a working prototype for smart tank monitoring and hydro-aware forecasting. The main dashboard flow is implemented, the prediction experience is integrated end-to-end, and the codebase is structured clearly enough for extension, but a few critical runtime and security gaps still need attention before the project is production-ready.
