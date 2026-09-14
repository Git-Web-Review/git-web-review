# git-web-review

Development workspace for git-web-review.

## Docker compose

The root `docker-compose.yml` starts the full local topology:

- PostgreSQL
- Redis
- backend API
- frontend app
- websocket relay
- IRC relay
- email relay
- HTTP relay (webhooks)

Each service runs from its own Dockerfile. The IRC, email and HTTP relays consume Redis notification events and can run in dry-run mode until real transport credentials are configured.

## First run

Copy and edit the compose environment file:

```sh
cp example.env .env
```

## Firebase setup

The project uses Firebase in two different ways:

- the frontend uses the public Firebase Web app config through `VITE_FIREBASE_*` variables;
- the backend uses a private Firebase service account JSON to verify Firebase ID tokens.

### Frontend Firebase variables

The `VITE_FIREBASE_*` variables come from the Firebase Web app config. They are public browser-side values, not private secrets.

To find them:

1. Open `https://console.firebase.google.com`.
2. Select the Firebase project.
3. Open Project settings with the gear icon.
4. Go to the General tab.
5. In Your apps, create or open a Web app with the `</>` icon.
6. Firebase shows a `firebaseConfig` object.

Example Firebase config:

```ts
const firebaseConfig = {
  apiKey: "AIza...",
  authDomain: "my-project.firebaseapp.com",
  projectId: "my-project",
  storageBucket: "my-project.appspot.com",
  messagingSenderId: "123456789",
  appId: "1:123456789:web:abcdef",
};
```

Copy those values into `.env`:

```env
VITE_FIREBASE_API_KEY=AIza...
VITE_FIREBASE_AUTH_DOMAIN=my-project.firebaseapp.com
VITE_FIREBASE_PROJECT_ID=my-project
VITE_FIREBASE_APP_ID=1:123456789:web:abcdef
VITE_FIREBASE_MESSAGING_SENDER_ID=123456789
VITE_FIREBASE_STORAGE_BUCKET=my-project.appspot.com
```

These values are expected to be visible in the browser. Security comes from Firebase Auth, Firebase rules where relevant, and backend token verification.

### Backend Firebase service account

The backend needs a Firebase service account JSON to verify user tokens. This file is private and must not be committed.

To generate it:

1. Open `https://console.firebase.google.com`.
2. Select the Firebase project.
3. Open Project settings with the gear icon.
4. Go to the Service accounts tab.
5. Click Generate new private key.
6. Download the JSON file.
7. Store it in this workspace as `secrets/firebase-service-account.json`.

```sh
mkdir -p secrets
cp /path/to/downloaded-service-account.json secrets/firebase-service-account.json
```

Keep this `.env` value as-is for Docker Compose:

```env
GOOGLE_APPLICATION_CREDENTIALS=/run/secrets/firebase-service-account.json
```

Docker Compose mounts the local `secrets` directory at `/run/secrets` inside the backend and websocket containers. The stack can start without this file, but Firebase-protected API routes will reject tokens until it exists.

Also set this value in `.env`:

```env
FIREBASE_PROJECT_ID=my-project
```

### Firebase Auth provider

In Firebase Authentication, enable the OAuth provider used by the company. Also add local development domains if Firebase asks for authorized domains, for example:

- `localhost`
- the company development domain, if any

### Verified emails

The backend accepts any token your Firebase project issues, whatever provider
produced it — the frontend only offering one button does not restrict that. And
it decides two things from the address inside: whether the domain is allowed,
and whether the user is an admin. So an unverified address is refused:

- the token must carry `email_verified: true`;
- the address must not already belong to another identity.

Enabling a provider that does not verify the address — email/password above all
— would otherwise be enough to sign up as `admin@company.tld` and be handed the
ADMIN role. Do not enable one unless you mean to.

Some federated IdPs guarantee the address but omit the claim. List those, and
only those, so they are accepted without it:

```env
FIREBASE_TRUSTED_SIGN_IN_PROVIDERS=oidc.acme,saml.company
```

Use the Firebase provider ids exactly as they appear in `sign_in_provider`.
Listing `password` defeats the purpose of the check.

### When an account is already linked

Signing in with a Firebase account whose email already belongs to a different
identity is refused with `EMAIL_ALREADY_LINKED`, and the refusal names both the
existing user id and the rejected uid in the backend log. Two cases reach it:

- a **service account**'s user row, which no browser sign-in may ever take over;
- a person whose Firebase uid changed, typically because their account was
  deleted and recreated with the same address.

The second case is a lockout with no in-app remedy today: the admin delete
button refuses users that have a Firebase identity, so the stale row has to be
removed or repointed directly in the database.

A row created ahead of time from a commit trailer is not in this situation —
nobody occupies it, and the person claims it on their first sign-in, keeping
the reviews and comments already attributed to them.

## Internal API auth (service accounts)

Alongside Firebase sign-in, the backend accepts an internal JWT so agents, CI
jobs and scripts can call the API without a browser. Both flows travel on the
same `Authorization: Bearer <token>` header and every route accepts either one.

Enable it by setting a secret in `.env`:

```env
INTERNAL_JWT_SECRET=<at least 32 characters, e.g. openssl rand -base64 48>
INTERNAL_JWT_TTL_SECONDS=3600
```

Leaving `INTERNAL_JWT_SECRET` empty disables the internal flow entirely; a
secret shorter than 32 characters is refused and logged as an error.

### Creating a service account

In the app, go to Admin then the Service accounts tab, or call the admin API
directly. Either way you need to be an admin.

```sh
curl -X POST http://localhost:3005/v1/admin/service-accounts \
  -H "authorization: Bearer <admin token>" \
  -H "content-type: application/json" \
  -d '{"name": "Review bot", "description": "Automated reviews"}'
```

The response contains the plaintext `clientSecret` **once**. It is stored
salted and hashed with scrypt and cannot be read again; if it is lost, rotate
it.

Each service account owns a regular user row, so the agent can be assigned as
a reviewer, post comments and show up in the history like any teammate. The
user gets a technical email, `<client-id>@service.internal` by default. Pass
`"admin": true` to give that user the ADMIN role.

### Using the credentials

Exchange them for a short-lived token, then call the API with it:

```sh
TOKEN=$(curl -s -X POST http://localhost:3005/v1/auth/token \
  -H "content-type: application/json" \
  -d '{"clientId": "review-bot", "clientSecret": "gwr_sk_..."}' \
  | jq -r .accessToken)

curl http://localhost:3005/v1/me -H "authorization: Bearer $TOKEN"
```

The token expires after `INTERNAL_JWT_TTL_SECONDS`; the agent simply asks for a
new one with the same credentials. There is no refresh token to keep around.
The websocket relay authenticates through the backend, so the same token also
works to open a `ws://localhost:3001/ws` connection.

### Revoking

- **Disable**: `PATCH /v1/admin/service-accounts/:id` with `{"active": false}`.
  Blocks new tokens and rejects the ones already issued, and is reversible.
- **Rotate**: `POST /v1/admin/service-accounts/:id/rotate-secret`. Invalidates
  the old secret immediately; tokens already issued stay valid until they
  expire.
- **Delete**: `DELETE /v1/admin/service-accounts/:id`. Revokes the credentials
  for good. The agent's user row is kept so its reviews and comments stay
  attributed, and it can no longer authenticate through any flow.

To erase the agent itself, delete its user from the admin users page. See
[Deleting users](#deleting-users) below for what that destroys.

## Deleting users

Admin then Users has a delete button, also exposed as
`DELETE /v1/admin/users/:id`. It only accepts users **without a Firebase
identity**: service accounts, and anyone who never signed in. A Firebase user
would simply be recreated on their next login, so the button stays disabled for
them. You cannot delete yourself, nor the last admin.

The deletion cascades. Reviews the user owns are destroyed along with their
commits and **every comment other people left on them**. Its own comments,
reviewer assignments, acks, file views, notifications, settings and service
account go too.

Because that reaches other people's work, the dialog first calls
`GET /v1/admin/users/:id/deletion-preview` and lists exactly what will be lost
before you confirm. There is no undo.

## Webhook notifications

Next to email and IRC, every user can have their notifications POSTed to a URL
of their own. In Settings, enable **Webhook notifications**, paste the URL and
pick which categories to receive — the same per-category switches the other two
transports have.

The `http-relay` service does the sending. It subscribes to
`notifications:webhook` on Redis and POSTs the event as JSON — a small envelope
(`event`, `notificationId`, `deliveryId`, `createdAt`, `sentAt`, `url`, `user`)
around the backend's `payload`, forwarded untouched. It does no formatting and
has no templates: presenting the notification is the receiving end's job.

Deliveries retry on transport errors, `429` and `5xx` with exponential backoff;
other `4xx` are dropped as permanent. `HTTP_RELAY_DRY_RUN=true` logs the
requests instead of sending them.

The URL is user-supplied and the relay calls it from inside the Docker network,
so a target resolving to a loopback, private, CGNAT or link-local address is
refused by default:

```env
# Default. Turn off only if your endpoints are genuinely on private addresses.
WEBHOOK_BLOCK_PRIVATE_NETWORKS=true
# Optional, and independent of the above: restrict which hosts users may target.
WEBHOOK_ALLOWED_HOSTS=chat.company.tld,hooks.company.tld
```

`WEBHOOK_ALLOWED_HOSTS` reads like the other allow-lists: `*`, or an empty
value, allows every host, and a listed host also matches its subdomains. An
allow-listed host still has to pass the private-address check, so if your chat
server is on a private IP you need both settings, not one.

Both are read twice: by the `http-relay`, which enforces them on the address it
is about to dial, and by the backend, which refuses a disallowed URL when the
user saves it — with a message naming the reason, rather than accepting it and
dropping every delivery in silence.

See [`http-relay/README.md`](http-relay/README.md) for the full payload, the
headers and every setting.

## Database and Redis credentials

Neither has a default any more: compose refuses to start without
`POSTGRES_PASSWORD` and `REDIS_PASSWORD`, so a deployment cannot silently end up
with a value that is published in this repository. Generate both with
`openssl rand -hex 32`.

Redis sits on the internal Docker network, which is why it had no password at
all. That is thin: a container compromised through any other path gets not just
read access but the right to **publish** on the notification channels, and the
three relays trust what they read there — a forged event means an outbound POST
to an arbitrary URL, an IRC message, or an email.

### Rotating on an existing database

`POSTGRES_PASSWORD` is only read when the data volume is first initialised.
Changing it in `.env` alone leaves the database on its old password and the
backend fails to connect. Change it inside PostgreSQL as well, while the old
password still works:

```sh
docker compose exec -T postgres psql -U git_web_review -d git_web_review \
  -c "ALTER USER git_web_review WITH PASSWORD '<the new password>';"
```

Then `docker compose up -d`. Redis needs none of this: `--requirepass` applies
at start-up, so restarting is enough.

## Database migrations

The backend runs `prisma migrate deploy` at container start, then serves. The
schema is therefore versioned in `backend/prisma/migrations`, reviewed like the
rest of the code, and applied the same way in every environment.

### Upgrading a database created before migrations existed

Earlier versions ran `prisma db push`, which reconciles the schema without
recording anything. Such a database already has the tables but no migration
history, so `migrate deploy` stops with `P3005: The database schema is not
empty` and the backend will not start.

Rebuild the backend image first. Both commands below run inside it, and they
need the `prisma/migrations` directory that a stale image does not have — the
symptom is `P3017: The migration ... could not be found`.

```sh
docker compose build backend
```

Then check that the live schema really matches `prisma/schema`. Repeated
`db push` runs can leave a database that drifted from it, and baselining a
drifted database records a migration whose effects were never fully applied:

```sh
docker compose run --rm --no-deps --entrypoint sh backend -c \
  "npx prisma migrate diff --from-config-datasource --to-schema prisma/schema --script"
```

`-- This is an empty migration.` means the schema matches.

Anything else is drift: apply it, or review it, before going further.

Then tell Prisma the initial migration is already applied, once, and start the
stack normally:

```sh
docker compose run --rm --no-deps --entrypoint sh backend -c \
  "npx prisma migrate resolve --applied 20260914000000_init"
```

A brand-new database needs none of this: `migrate deploy` creates everything.
Wiping the local database with `docker compose down -v` is also a valid answer
in development, and skips the whole procedure.

`npm run prisma:migrate:status` inside the container reports what has been
applied. Use `npm run prisma:migrate` (that is `prisma migrate dev`) in
development to author the next migration; `prisma db push` stays for throwaway
local experiments only.

## Reading review sources

Creating, previewing or syncing a review makes the backend run `git ls-remote`
and `git fetch` against the remote the URL rule derives. That remote is built
from what the caller submitted, so the outbound call is bounded three ways:

| Setting | Default | Effect |
| --- | --- | --- |
| `GIT_ALLOWED_HOSTS` | empty (all) | Hosts the backend may contact, same spelling as the other allow-lists |
| `GIT_MAX_CONCURRENT` | `4` | Simultaneous git operations, all callers together |
| `GIT_TIMEOUT_MS` | `120000` | Per-operation timeout |

The default allow-list is permissive on purpose: company forges often live on
private addresses, so refusing them by default would break review creation
outright. Name your forges there on any instance whose users are not fully
trusted.

Git itself runs with `protocol.ext` and `protocol.file` refused,
`GIT_ALLOW_PROTOCOL` pinned to `git:http:https:ssh`, system and global config
neutralised, and a minimal environment — the backend's own secrets are not
handed to subprocesses.

A failing fetch reports a generic message. Git's stderr says whether a host
answered, whether a repository exists and whether a port is filtered, on an
address the caller chose; the detail goes to the backend log instead.

## Rate limiting

Every route is rate limited, per caller and per minute:

| Setting | Default | Applies to |
| --- | --- | --- |
| `THROTTLE_DEFAULT_PER_MINUTE` | `300` | every route |
| `THROTTLE_AUTH_PER_MINUTE` | `10` | `POST /v1/auth/token` |
| `THROTTLE_GIT_PER_MINUTE` | `20` | review create, preview and sync |

A caller is identified by its bearer token when there is one, and by IP address
otherwise. That distinction matters: a company reaches the API through a handful
of public addresses, so counting by IP would let one busy user throttle their
colleagues. The two stricter buckets cover the routes that cost the server real
work — an scrypt derivation on a public endpoint, and a `git fetch` towards a
remote host with a two-minute timeout.

Over the limit, the API answers `429` with the usual envelope and the
`TOO_MANY_REQUESTS` code.

Behind a reverse proxy, set `TRUST_PROXY` so the client address is read from
`X-Forwarded-For`. Leave it empty otherwise: with no proxy in front, anyone
could set that header and pick their own counter.

## Running the stack

Start everything:

```sh
docker compose up --build
```

Useful commands:

```sh
docker compose logs -f backend
docker compose down
docker compose down -v
```

Default local URLs:

- Backend: `http://localhost:3005`
- Swagger: `http://localhost:3005/api`
- Frontend: `http://localhost:5173`
- WebSocket relay: `ws://localhost:3001/ws`
- HTTP relay status: `http://localhost:3002/status`

The backend still listens on port `3000` inside Docker; only the host port defaults to `3005` to avoid common local conflicts with port `3000`.

With Docker Desktop on WSL, test published ports from a Windows browser or with `cmd.exe /c curl.exe http://localhost:3005/health` if WSL `curl` hangs on localhost forwarding.
