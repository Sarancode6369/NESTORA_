# NESTORA — Setup and deployment

NESTORA is a smart township management app (Node.js 22, vanilla-JS front end, MySQL 8).

* Repository: <https://github.com/Sarancode6369/NESTORA_.git>
* Live application: <https://nestora-new.vercel.app/>

This file lists commands that exist in `package.json` and settings that the code reads. Nothing here contains a password,
API key or secret. Variable names only: put your own values in `.env` or in the Vercel dashboard.

## Prerequisites

* Node.js **22 or newer** (`engines` in `package.json`; the app uses the built-in `fetch` and `node:test`).
* MySQL **8.0.16 or newer** (InnoDB, utf8mb4; CHECK constraints are enforced from 8.0.16). MariaDB is untested.
* Git.
* A Groq API key is **optional**. Without it complaint triage uses the keyword classifier, duplicate suggestions use rule
  scoring only, and the resident assistant answers with a fixed "contact the help desk" message.

## 1. Clone and install

```bash
git clone https://github.com/Sarancode6369/NESTORA_.git
cd NESTORA_
npm install
```

`mysql2` is the only dependency.

## 2. Create the database (once)

Log in to MySQL. It prompts for your password; do not type it into a command or share it.

```bash
mysql -u root -p
```

```sql
CREATE DATABASE IF NOT EXISTS nestora_db CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
-- Recommended beyond a local laptop: a dedicated account instead of root.
CREATE USER 'nestora_app'@'localhost' IDENTIFIED BY '<choose-a-strong-password>';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER, INDEX, REFERENCES ON nestora_db.* TO 'nestora_app'@'localhost';
```

On a server, restrict the account to the application host instead of `localhost`. After the first run you may revoke
`CREATE, ALTER, INDEX`.

## 3. Configure environment variables

The `.env` file lives in the project root (next to `package.json`). `npm start` and every `npm run db:*` script read it
automatically. Variables already set in the real environment win over `.env`.

```bash
cp .env.example .env          # Windows PowerShell: Copy-Item .env.example .env
```

Edit `.env` and replace every placeholder. `.env` is git-ignored (so is `.env.*`, except `.env.example`). Never commit it.

| Variable | Meaning |
|---|---|
| `DATABASE_URL` | `mysql://localhost:3306/nestora_db`. Host, port and database only, **no user or password in the URL**. Add `?ssl=true` for managed MySQL. |
| `DB_USERNAME` | MySQL user |
| `DB_PASSWORD` | That user's password (read separately from the URL, so `@ : / #` need no escaping) |
| `JWT_SECRET` | Long random string used to sign login tokens. **Required** when `NODE_ENV=production` and on Vercel. Generate one with `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"` |
| `PORT` | HTTP port for `npm start` (default 3000) |
| `NODE_ENV` | Set to `production` for real deployments; this also disables `/api/demo/reset` |
| `SEED_DEMO_DATA` | `false` to stop the app loading the demo township into an empty database at start-up |
| `GROQ_API_KEY` | Optional. Backend only, never sent to the browser. Empty means fallback mode |
| `GROQ_MODEL` | Optional. Default `openai/gpt-oss-120b` |
| `AI_TIMEOUT_MS`, `AI_MIN_CONFIDENCE`, `AI_MAX_CALLS_PER_MIN` | Optional AI tuning (defaults 8000 ms, 0.6, 30) |
| `DB_SSL` | `true` to require TLS when it is not already in `DATABASE_URL` |
| `DB_SSL_CA` | Optional PEM certificate for private or custom CAs (for example Aiven) |
| `DB_SSL_REJECT_UNAUTHORIZED` | Defaults to strict verification. Set `false` only for local testing |

`.env.example` ships placeholder values (`your_mysql_username`, `your_mysql_password`). The app refuses to start with them
and says what to replace.

## 4. Initialise, seed, run

```bash
npm run db:init      # create tables, indexes and constraints (safe to repeat)
npm run db:seed      # load the demo township; works on an EMPTY database only
npm start            # API and front end on http://localhost:3000 (or PORT)
```

* The front end is static files in `public/`, served by the same process. There is no separate build step.
* `npm start` also creates missing tables and loads the demo data into an empty database on boot, unless
  `SEED_DEMO_DATA=false`.
* `npm run db:seed` stops with "Database already has data" if users exist. That is intended.
* Check it is up: `GET /api/health` returns `{"ok":true,"database":"mysql", ...}`.
* Other scripts: `npm run db:export` (JSON backup), `npm run db:seed-sql` (writes `database/seed.sql`),
  `node scripts/db.js import-json <file>` (import an old JSON snapshot into an empty database).
* **`npm run db:reset` wipes every table and reloads the demo data. Never run it on a database that holds real data.**

### Optional: SQLite for quick evaluation (separate from MySQL)

```bash
npm run dev:sqlite
```

This runs the same app on an in-memory SQLite engine with demo data and needs no MySQL. Data is lost when it exits. It is for
development and evaluation only. `npm start` and `npm run db:*` always use MySQL.

## 5. Tests

```bash
npm test
```

This runs `node --test test/*.test.js`. By default every test file uses its **own in-memory SQLite database** built from the
same `database/schema.sql`, so a normal run does not touch MySQL and does not need a `.env`.

Result recorded on 7 November 2026 (commit `804c9c8`, Node 22.23, Linux, no `GROQ_API_KEY`, `npm install` not run):
**122 tests, 120 passed, 1 failed, 1 skipped.**

* The skipped test is the live Groq API test. It runs only when `GROQ_API_KEY` is set.
* The failed test is the MySQL SSL configuration test. It failed with "mysql2 is not installed" because that run had no
  `node_modules`. Run `npm install` first, then `npm test`, and update these numbers with what you see.

To run the suite against a real MySQL server, set `TEST_DATABASE_URL`, `TEST_DB_USERNAME` and `TEST_DB_PASSWORD`.
**This wipes and re-seeds the tables it connects to, so use a throw-away database, never production.** That MySQL run has not
been recorded.

## 6. Demo logins

The demo OTP is always `123456`. No SMS is sent.

| Role | Phone |
|---|---|
| Resident (Neeraj Kulkarni, flat T07-1203) | `9000000001` |
| Resident (Swati Joshi, flat T12-0408) | `9000000002` |
| Resident (Aamir Shaikh, flat T21-1601) | `9000000003` |
| Help desk | `9000000010` |
| Committee | `9000000020` |
| Guard | `9000000030` |

These are fictional demo accounts. Payments are test-mode only; never enter real card or bank details.

## 7. Deploying to Vercel

`vercel.json` serves `public/**` as static files and runs the API from `api/[...path].js` (`@vercel/node`). Requests to
`/api/*` go to the function; everything else goes to `public/`.

1. Create a MySQL database that Vercel can reach (a managed service such as Aiven, or any host that allows the connection).
2. Import the repository into Vercel.
3. In **Settings → Environment Variables** add:
   * `DATABASE_URL` (`mysql://<host>:<port>/<database>?ssl=true`)
   * `DB_USERNAME`
   * `DB_PASSWORD`
   * `JWT_SECRET`
   * `NODE_ENV` = `production`
   * `DB_SSL_CA` only if the provider uses a private CA
   * `GROQ_API_KEY` and `GROQ_MODEL` only if you want the AI features
   * `SEED_DEMO_DATA` = `false` if you do not want demo data loaded into an empty database
4. Deploy, then open `/api/health` on the deployment. It should report `"database":"mysql"`.

Things to know before relying on it:

* On the first request the function opens the database, creates missing tables and (unless `SEED_DEMO_DATA=false`) loads the
  demo data into an empty database. It returns `500 Server initialization failed` if `JWT_SECRET` is missing or the database
  cannot be reached; check the function logs.
* The OTP attempt counter and the rate limiters are held in memory per function instance. On a serverless host,
  `request-otp` and `verify-otp` can reach different instances, and verification may then answer "Request an OTP first".
  Move this state to a shared store before relying on it.
* The hourly visitor-photo purge is a timer started by `npm start`. It has not been verified on Vercel.
* On the Vercel entry point the duplicate-complaint route does not receive the Groq client, so it uses rule scoring only.
* The demo reset route is disabled there (`allowDemoReset: false`).
* **Status of the live site `https://nestora-new.vercel.app/`:** its reachability, front end, back end, production database,
  sign-in and AI have not been verified by this document.

## 8. Running it yourself on a server (alternative)

* Run behind HTTPS (nginx, Caddy or a load balancer). The app sets security headers but does not terminate TLS.
* Set `NODE_ENV=production` and `JWT_SECRET`.
* Keep one app instance, or accept that OTP and rate-limit counters are per instance.
* Back up MySQL nightly, for example `mysqldump --single-transaction nestora_db`. Visitor photos are stored in the database
  and deleted after 90 days by the hourly job.
* Example systemd unit:

```ini
[Service]
WorkingDirectory=/opt/nestora
EnvironmentFile=/etc/nestora.env
ExecStart=/usr/bin/node server/index.js
Restart=always
User=nestora
```

* Staff can read the AI mode with `GET /api/ai/status` (`groq` or `keyword`; never the key).

## 9. Troubleshooting

* `DATABASE_URL is not set`: there is no `.env` in the project root, or it is empty.
* `DB_PASSWORD in .env is still the example placeholder`: edit `.env` and enter your own MySQL password.
* `MySQL rejected DB_USERNAME / DB_PASSWORD`: wrong credentials. Test them with `mysql -u <user> -p`.
* `Cannot reach MySQL at localhost:3306`: start the service (Windows: `Get-Service MySQL80`, then `Start-Service MySQL80`
  in an Administrator PowerShell).
* `Database "nestora_db" does not exist`: run the `CREATE DATABASE` statement in step 2.
* `mysql2 is not installed`: run `npm install`.
* `Database already has data` from `npm run db:seed`: expected when users exist. Do not reset a database you want to keep.
* A managed database refuses the connection: add `?ssl=true` to `DATABASE_URL` (or `DB_SSL=true`), and `DB_SSL_CA` if the
  provider uses its own certificate authority.
* `Server initialization failed` on Vercel: check that `JWT_SECRET`, `DATABASE_URL`, `DB_USERNAME` and `DB_PASSWORD` are set
  for the environment you deployed, and that the database accepts connections from Vercel.
* "Request an OTP first" on Vercel: see the in-memory OTP note in section 7.
* Every complaint shows *Keyword fallback*: `GROQ_API_KEY` is missing, wrong or rate-limited, or the model name is no longer
  offered by Groq. Check `GET /api/ai/status` and the server log (errors are logged without keys or complaint text).

## 10. Security warnings

* Never commit `.env`, a database dump, an API key or a JWT secret. Rotate any secret that has been pasted into a chat,
  an issue or a commit.
* The OTP is a fixed demo value and payments are test mode. Do not present a demo deployment as a secure system. For real use,
  set `SEED_DEMO_DATA=false`, remove the demo accounts, and add a real SMS gateway and payment gateway.
* Use a dedicated MySQL account with the minimum privileges, and keep TLS on for any database that is not on the same machine.
* `DB_SSL_REJECT_UNAUTHORIZED=false` turns off certificate checking. Use it for local testing only.
* Do not run `npm run db:reset` or the MySQL test mode (`TEST_DATABASE_URL`) against a database that holds real data.
