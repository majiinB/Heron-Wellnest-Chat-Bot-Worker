# Heron Wellnest Chat Bot Worker

A background worker that processes Pub/Sub chat events from the front-facing chatbot API, generates AI responses with Gemini, and persists bot messages and session updates to PostgreSQL.

## Table of Contents
- Features
- Tech Stack
- Architecture
- Getting Started
- Configuration and Environment Variables
- Pub/Sub Processing Flow
- API Endpoint (Push Subscription)
- Project Structure
- Troubleshooting
- Testing
- Deployment
- License & Authors

## Features
- Processes chat events from Google Cloud Pub/Sub (pull worker) or HTTP push endpoint.
- Generates AI responses with Google Gemini and structured JSON outputs.
- Decrypts/encrypts chat content before storing bot responses.
- Enforces session hard-stop and near-stop limits for safe conversations.
- Sends counselor notification alerts via Pub/Sub when self-harm risk is detected.
- Supports async PostgreSQL access via SQLAlchemy + asyncpg.

## Tech Stack
- Python 3.12+
- FastAPI
- SQLAlchemy 2.0 (async), asyncpg
- Google Cloud Pub/Sub
- Google Gemini (google-genai)
- cryptography (AES-256-GCM)
- Docker (optional)

## Architecture
- Front-facing Chat API → publishes a chat event to Pub/Sub.
- Chat Bot Worker (this repo) → receives the event, loads session and messages, generates a Gemini response, stores the bot message, and updates session status.
- Optional: Push subscription → Pub/Sub delivers the event to `POST /pubsub/chat-bot`.

Service entrypoint: FastAPI app in `app/main.py`. The pull worker lives in `app/worker.py` and can be run as a standalone worker.

## Getting Started

### Prerequisites
- Python 3.12+
- PostgreSQL (or Supabase)
- Google Cloud project with Pub/Sub (or Pub/Sub emulator)
- Google Cloud credentials (ADC)

### Installation
1. Clone the repository

```bash
git clone <repository-url>
cd heron-wellnest-chat-bot-worker
```

2. Create and activate a virtual environment, then install dependencies (PowerShell example)

```bash
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

3. Create a `.env` file in the project root (see configuration below).

### Run the worker (pull subscription)

```bash
python -m app.worker
```

### Run the HTTP service (push subscription)

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8080
```

## Configuration and Environment Variables
The worker uses Pydantic settings from `app/config/env_config.py`. Create a `.env` file at the project root and set the required values.

Required:
- `CONTENT_ENCRYPTION_KEY` (>= 32 characters)
- `PUBSUB_CHAT_BOT_TOPIC`
- `PUBSUB_NOTIFICATION_TOPIC`
- `GEMINI_API_KEY`

Common:
- `ENVIRONMENT` (development | production | test; default: development)
- `PORT` (default: 8080)
- `DB_HOST` (default: localhost)
- `DB_PORT` (default: 5432)
- `DB_USER` (default: postgres)
- `DB_PASSWORD`
- `DB_NAME` (default: heron_wellnest)
- `GOOGLE_CLOUD_PROJECT_ID`
- `PUBSUB_CHAT_BOT_SUBSCRIPTION` (default: chat-bot-subscription)
- `CONTENT_ENCRYPTION_ALGORITHM` (default: aes-256-gcm)
- `CONTENT_ENCRYPTION_IV_LENGTH` (default: 16)
- `CONTENT_ENCRYPTION_KEY_LENGTH` (default: 32)
- `HARD_STOP_TOTAL` (default: 100)
- `HARD_STOP_ROLE` (default: 50)
- `NEAR_STOP_TOTAL` (default: 90)
- `NEAR_STOP_ROLE` (default: 45)
- `PH_HOTLINE` (default: 1553)

Example `.env`:

```dotenv
ENVIRONMENT=development
PORT=8080
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=postgres
DB_NAME=heron_wellnest
CONTENT_ENCRYPTION_KEY=replace-with-a-32-char-min-secret
CONTENT_ENCRYPTION_ALGORITHM=aes-256-gcm
GOOGLE_CLOUD_PROJECT_ID=your-gcp-project
PUBSUB_CHAT_BOT_TOPIC=chat-bot-topic
PUBSUB_CHAT_BOT_SUBSCRIPTION=chat-bot-subscription
PUBSUB_NOTIFICATION_TOPIC=notification-topic
GEMINI_API_KEY=replace-with-gemini-api-key
```

## Pub/Sub Processing Flow
1. Pub/Sub message arrives with payload: `userId`, `sessionId`, `messageId`.
2. Worker loads the chat session and validates status is `waiting_for_bot`.
3. Retrieves the current message and recent context messages.
4. Decrypts messages and builds the conversation context.
5. Calls Gemini, parses structured JSON response, checks for self-harm indications.
6. Stores the encrypted bot response and updates session status.
7. Sends counselor notifications when required.

## API Endpoint (Push Subscription)
`POST /pubsub/chat-bot`

Expected Pub/Sub envelope:

```json
{
  "message": {
    "data": "base64-encoded-json-payload",
    "messageId": "...",
    "publishTime": "..."
  },
  "subscription": "..."
}
```

Decoded payload (JSON inside `data`):

```json
{
  "userId": "...",
  "sessionId": "...",
  "messageId": "...",
  "timestamp": "...",
  "eventType": "..."
}
```

## Project Structure
```
.
├── app/
│   ├── config/                # env and datasource config
│   ├── controller/            # HTTP handlers for Pub/Sub push
│   ├── repository/            # DB access for sessions/messages/notifications
│   ├── routes/                # FastAPI routes
│   ├── service/               # ChatService workflow + Gemini integration
│   ├── utils/                 # crypto, db, logging, pubsub helpers
│   ├── worker.py              # Pub/Sub pull worker
│   └── main.py                # FastAPI application entry
├── Dockerfile
├── requirements.txt
└── README.md
```

Key files:
- `app/main.py` — FastAPI app and (dev-only) worker startup.
- `app/worker.py` — Pub/Sub pull worker with streaming subscription.
- `app/routes/chat_route.py` — Push endpoint for Pub/Sub.
- `app/controller/chat_controller.py` — Pub/Sub envelope parsing and validation.
- `app/service/chat_service.py` — End-to-end chat workflow and Gemini calls.
- `app/utils/crypto_utils.py` — Encryption/decryption helpers.

## Troubleshooting
- `Failed to create Pub/Sub subscriber client`:
  - Ensure ADC is configured: `gcloud auth application-default login`.
  - Or set `GOOGLE_APPLICATION_CREDENTIALS` to a service account JSON.
- `Session is not waiting for bot response`:
  - Confirm the upstream service sets session status to `waiting_for_bot` before publishing the event.
- Gemini errors:
  - Verify `GEMINI_API_KEY` is correct and the model is enabled for your project.
- Database errors:
  - Confirm DB credentials, host, and access permissions.

## Testing
- No automated tests are included yet.
- Suggested next steps: add unit tests for `ChatService.process_chat_message` and integration tests for the Pub/Sub push endpoint.

## Deployment
- Build and run with Docker:

```bash
docker build -t hw-chat-bot-worker .
docker run --env-file .env --rm hw-chat-bot-worker
```

- For production, inject secrets via your cloud provider's secret manager and use a managed Pub/Sub subscription.

## License & Authors
- Private / proprietary to the Heron Wellnest platform.
- Author / Maintainer: Arthur M. Artugue

---
Last Updated: 2026-05-27

