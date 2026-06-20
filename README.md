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

Each service runs from its own Dockerfile. The IRC and email relays consume Redis notification events and can run in dry-run mode until real transport credentials are configured.

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

The backend still listens on port `3000` inside Docker; only the host port defaults to `3005` to avoid common local conflicts with port `3000`.

With Docker Desktop on WSL, test published ports from a Windows browser or with `cmd.exe /c curl.exe http://localhost:3005/health` if WSL `curl` hangs on localhost forwarding.
