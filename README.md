# HistoCrypt — Full-Stack Personal Project

In this project I built a cryptogram puzzle application with Django, Django REST Framework, PostgreSQL, and React. The API has full CRUD and JWT cookie authentication capabilities, and the whole stack is containerized with Docker Compose and deployed to AWS EC2 behind Nginx.

## Scenario

Cryptogram puzzles pair the satisfaction of clever deduction with a small reward for solving them. HistoCrypt pulls a real quote from a third-party API, encodes it with a substitution cipher, and asks the player to break it. Solving a puzzle reveals the quote, its author, and a short biography pulled from Wikipedia. Players can adjust the difficulty, save games in progress, keep a collection of favorite quotes with personal notes, and track what they have solved.

## Set Up

- Install Docker Desktop (with WSL2 integration on Windows)
- Clone the repository and change directory into it
- Copy `server/.env.example` to `server/.env`
- Fill in the placeholder values in `server/.env`:
  - `DJANGO_SECRET_KEY` — generate with `python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"`
  - `POSTGRES_PASSWORD` — any value for local development
  - `ZENQUOTES_API_KEY` — a ZenQuotes API key
- Build and start the stack
- Apply migrations inside the running backend container

```bash
git clone https://github.com/SimonGBaum/histocrypt_personal_project.git
cd histocrypt_personal_project
cp server/.env.example server/.env
docker compose up -d --build
docker compose exec backend python manage.py migrate
```

The app is served at `http://localhost` and the API is proxied at `http://localhost/api/v1/`.

## Usage

```sh
docker compose logs backend -f
docker compose exec backend bash
docker compose down
```

> NOTE: `docker compose down` leaves the `pgdata` volume in place, so your database survives a restart. Adding `-v` destroys it, which means re-running migrations and losing every account and saved game.

## Functionality

Functionality the app is expected to have includes:

- A puzzle is a real quote between 50 and 180 characters, encoded with a randomly shifted substitution alphabet
- Difficulty controls how much of the answer is prefilled — 10% on easy, 5% on medium, nothing on hard
- Character type controls whether cells display letters or the numbers 1 through 26
- Prefilled cells are read-only, and every cell becomes read-only once the puzzle is solved
- A player can hold no more than three saved games at once, and the API returns 409 on the fourth
- Saving a favorite twice returns 400 rather than creating a duplicate
- Solving a saved puzzle deletes the saved game automatically
- Leaving the game page with unsaved progress prompts for confirmation
- All routes other than `/` require an authenticated session, enforced by a default permission class on the API and by a redirect on the client
- On bad urls, redirect to a 404 page without the app layout

## Test Details

The backend tests live inside each app and run against a throwaway database. The end-to-end tests are located within the `client/cypress/e2e` folder and represent the 8 individual tests that exercise the app through a real browser.

The Django tests can be run all at once from inside the backend container:

```bash
docker compose exec backend python manage.py test
```

In order to run the Cypress tests, the full stack must already be up, because `cypress.config.js` points `baseUrl` at `http://localhost` — that is Nginx serving the built bundle with the API proxied behind it, not the Vite dev server.

```bash
cd client
npx cypress open
```

## Deployment

The application is deployed to an AWS EC2 instance running Ubuntu, with a static Elastic IP associated to it and ports 22, 80, and 443 open in its security group.

- Connect to the instance over SSH using the `.pem` key pair
- Clone the repository onto the instance
- Recreate `server/.env` from `server/.env.example`, with the Elastic IP added to `DJANGO_ALLOWED_HOSTS`
- Bring the stack up in detached mode and apply migrations

```bash
ssh -i ~/.ssh/histocrypt-personal-project.pem ubuntu@<elastic-ip>
git clone https://github.com/SimonGBaum/histocrypt_personal_project.git
cd histocrypt_personal_project
cp server/.env.example server/.env
nano server/.env
docker compose up -d --build
docker compose exec backend python manage.py migrate
```

> NOTE: `.env` is gitignored and never reaches the instance through the clone, so recreating it from the example file is a required step, not an optional one.

## Tech Stack

**Backend** — Django 6.0, Django REST Framework, SimpleJWT, PostgreSQL 15, psycopg 3

**Frontend** — React, Vite, React Router, Axios, React Bootstrap

**Infrastructure** — Docker, Docker Compose, Gunicorn, Nginx, AWS EC2

**External APIs** — ZenQuotes, Wikipedia REST