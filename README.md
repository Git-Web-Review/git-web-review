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

The URL is user-supplied, so the relay calls whatever it is given from inside
the Docker network. Where that matters, restrict it in `.env`:

```env
WEBHOOK_ALLOWED_HOSTS=chat.company.tld,hooks.company.tld
WEBHOOK_BLOCK_PRIVATE_NETWORKS=true
```

`WEBHOOK_ALLOWED_HOSTS` reads like the other allow-lists: `*`, or an empty
value, allows every host, and a listed host also matches its subdomains.

See [`http-relay/README.md`](http-relay/README.md) for the full payload, the
headers and every setting.

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
