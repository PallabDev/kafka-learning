# Kafka Learning

## Project overview

This project is a small real-time location sharing demo built with Express, Socket.IO, and Kafka. Users sign in through an external OIDC-compatible auth service, connect to the server with an access token, and stream browser geolocation updates through Kafka. The server consumes those Kafka messages and broadcasts live map updates back to all connected clients.

## Tech stack

- Node.js with native ESM
- Express 5
- Socket.IO
- KafkaJS
- Apache Kafka via Docker Compose
- Leaflet with OpenStreetMap tiles
- Browser Geolocation API

## Setup steps

1. Install dependencies:

```bash
npm install
```

2. Start Kafka with Docker:

```bash
docker compose up -d
```

3. Create the Kafka topic:

```bash
node kafka-admin.js
```

4. Copy `env.sample` to `.env` and replace the placeholder auth values with real values from your auth provider.

5. Start the main app.

If you want Node to load values from `.env` directly:

```bash
node --env-file=.env index.js
```

If you prefer using the existing dev script, make sure the same environment variables are already available in your shell:

```bash
npm run dev
```

6. Optional: start the sample downstream Kafka consumer that represents database processing:

```bash
node database-processor.js
```

7. Open `http://localhost:5000`.

## Environment variables

The app reads these values from `process.env`:

| Variable | Required | Description |
| --- | --- | --- |
| `PORT` | No | HTTP port for the Express and Socket.IO server. Defaults to `5000`. |
| `AUTH_ORIGIN` | Yes | Base URL of the external auth server used for JWKS and token exchange. |
| `AUTHORIZATION_ENDPOINT` | No | Login endpoint. Defaults to `${AUTH_ORIGIN}/user/login`. |
| `AUTH_REDIRECT_URI` | No | Callback URL handled by this app. Defaults to `http://<host>/auth`. |
| `AUTH_CLIENT_ID` | Yes | OIDC client ID used during auth code exchange. |
| `AUTH_CLIENT_SECRET` | Yes | OIDC client secret used during auth code exchange. |

Example file:

```env
PORT=5000
AUTH_ORIGIN=https://auth.example.com
AUTHORIZATION_ENDPOINT=https://auth.example.com/user/login
AUTH_REDIRECT_URI=http://localhost:5000/auth
AUTH_CLIENT_ID=demo-client-id
AUTH_CLIENT_SECRET=demo-client-secret
```

## OIDC auth setup

1. A visitor opens `/`.
2. `public/login.html` checks local storage for an unexpired access token.
3. If no valid token exists, the browser is redirected to `/login`.
4. `/login` builds the authorization URL using `client_id` and `redirect_uri`, then redirects the user to the external auth server.
5. After login, the auth provider redirects back to `/auth?code=...`.
6. `public/auth.html` posts the authorization code to `/auth/exchange`.
7. `/auth/exchange` sends the code, client ID, client secret, and redirect URI to the token endpoint.
8. The server validates the returned JWT using the JWKS exposed by `${AUTH_ORIGIN}/certs`.
9. The access token and normalized user object are saved in local storage.
10. The browser opens `/home`, where Socket.IO connects with `handshake.auth.accessToken`.

## Socket event flow

Client to server:

- `client:location:update`
  - Sent by the browser whenever geolocation returns a fresh latitude and longitude.

Server to client:

- `server:session`
  - Sent immediately after authenticated socket connection.
  - Contains the socket ID and resolved user profile.
- `server:location:update`
  - Sent whenever a Kafka location message is consumed.
  - Contains `id`, `userName`, `latitude`, `longitude`, and `updatedAt`.

Runtime flow:

1. The browser connects to Socket.IO with the saved access token.
2. The server validates the token before accepting the socket connection.
3. The browser gets the current location and starts `watchPosition`.
4. Each location update is emitted as `client:location:update`.
5. The server publishes that payload to the Kafka topic `location-updates`.
6. The Kafka consumer inside `index.js` reads the message and broadcasts `server:location:update` to all connected clients.
7. Each browser updates the Leaflet map markers in real time.

## Kafka event flow

Topic used:

- `location-updates`

Producers:

- The main app in `index.js` publishes location updates received from Socket.IO clients.

Consumers:

- The main app in `index.js` consumes the same topic and rebroadcasts messages to connected sockets.
- `database-processor.js` is a second consumer that simulates writing consumed events into a database.

Lifecycle:

1. `kafka-admin.js` creates the `location-updates` topic.
2. A socket event arrives with live coordinates.
3. `index.js` serializes the event and sends it to Kafka with the socket ID as the message key.
4. Kafka stores the event in the topic partitions.
5. The Socket.IO server consumer reads the event and emits it back to browsers.
6. The database processor consumer reads the same event independently for downstream processing.

## Demo video link

Add your demo video link here:

`https://your-demo-link.example`

## Assumptions and limitations

- This project assumes an external auth server already exists and supports the endpoints used in `index.js`.
- JWT validation currently checks signature and time-based claims, but it does not validate issuer or audience claims.
- The server currently contains fallback auth values in code, which is convenient for local development but not ideal for production.
- The Kafka broker is configured for local development with a single broker and no replication.
- The sample database processor only logs events; it does not persist data anywhere.
- Browser geolocation permissions are required for the live map experience.
- The frontend stores the access token in `localStorage`, which is acceptable for a demo but should be reviewed for production security requirements.
