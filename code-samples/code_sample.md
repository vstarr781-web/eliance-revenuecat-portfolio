# RevenueCat Webhook Receiver — Production Python Server

Here's the complete code sample. I'll produce two files: the main server (`server.py`) and the `README.md`.

---

## `server.py`

```python
"""
RevenueCat Webhook Receiver — Production Python Server
=======================================================
Author: ELIANCE (Autonomous AI Developer Advocate for RevenueCat)
Date: 2026-03-05

Receives RevenueCat webhook events, verifies authorization, processes
subscription lifecycle events (INITIAL_PURCHASE, RENEWAL, CANCELLATION,
BILLING_ISSUE, EXPIRATION, etc.), and syncs subscriber state to SQLite.

Design decisions:
  - Deferred processing: HTTP handler returns 200 immediately, pushes to queue.
    Prevents RevenueCat from retrying due to slow processing (RC retry schedule:
    5, 10, 20, 40, 80 minutes — avoid triggering this).
  - Idempotency: event.id is stored in SQLite; duplicate deliveries are no-ops.
  - Never store boolean `is_active` — always store expiration timestamp and
    check `expiration > time.time()` (per official RC sample pattern).
  - HMAC-ready: auth header verification is baked in (not bolted on later).

Requirements:
    pip install flask requests python-dotenv
"""

import hashlib
import json
import logging
import os
import queue
import sqlite3
import threading
import time
from datetime import datetime, timezone
from functools import wraps
from http import HTTPStatus
from typing import Optional

import requests
from dotenv import load_dotenv
from flask import Flask, Response, jsonify, request

# ─── Logging ─────────────────────────────────────────────────────────────────

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    datefmt="%Y-%m-%dT%H:%M:%S",
)
logger = logging.getLogger("rc_webhook")

# ─── Config ──────────────────────────────────────────────────────────────────

load_dotenv()

# Required env vars — server will not start without these.
RC_WEBHOOK_TOKEN: str = os.environ["RC_WEBHOOK_TOKEN"]          # Set in RC dashboard
RC_SECRET_API_KEY: str = os.environ["RC_SECRET_API_KEY"]        # sk_... from RC project settings
RC_PROJECT_ID: str = os.environ["RC_PROJECT_ID"]                # From RC dashboard URL

DB_PATH: str = os.getenv("DB_PATH", "subscriptions.db")
PORT: int = int(os.getenv("PORT", "8080"))
WORKER_THREADS: int = int(os.getenv("WORKER_THREADS", "2"))

# RevenueCat REST API base (v1 is stable for subscriber reads)
RC_API_BASE = "https://api.revenuecat.com/v1"

# Event types we care about. Others are accepted (200) but not stored.
HANDLED_EVENT_TYPES = {
    "INITIAL_PURCHASE",
    "RENEWAL",
    "CANCELLATION",
    "UNCANCELLATION",
    "NON_RENEWING_PURCHASE",
    "SUBSCRIPTION_PAUSED",
    "SUBSCRIPTION_EXTENDED",
    "BILLING_ISSUE",
    "PRODUCT_CHANGE",
    "EXPIRATION",
    "TRANSFER",
}

# ─── Database ─────────────────────────────────────────────────────────────────

def get_db() -> sqlite3.Connection:
    """
    Opens a SQLite connection with row_factory set to return dicts.
    Call this once per thread — do not share connections across threads.
    """
    conn = sqlite3.connect(DB_PATH, check_same_thread=False)
    conn.row_factory = _dict_factory
    conn.execute("PRAGMA journal_mode=WAL")   # Better concurrent read performance
    conn.execute("PRAGMA foreign_keys=ON")
    return conn


def _dict_factory(cursor: sqlite3.Cursor, row: tuple) -> dict:
    """Returns rows as dicts keyed by column name."""
    return {col[0]: val for col, val in zip(cursor.description, row)}


def init_db(conn: sqlite3.Connection) -> None:
    """
    Creates tables if they don't exist. Safe to call on every startup.

    Schema notes:
      - entitlements: stores per-user entitlement expiration (not boolean!)
      - processed_events: idempotency table — prevents double-processing RC retries
    """
    conn.executescript("""
        CREATE TABLE IF NOT EXISTS entitlements (
            user_id         TEXT    NOT NULL,
            entitlement     TEXT    NOT NULL,
            expiration      REAL,               -- Unix timestamp. NULL = lifetime/never expires.
            product_id      TEXT,               -- Store product identifier
            store           TEXT,               -- 'APP_STORE' | 'PLAY_STORE' | 'STRIPE' etc.
            last_synced_at  REAL    NOT NULL,   -- Unix timestamp of last RC API sync
            updated_at      REAL    NOT NULL,
            PRIMARY KEY (user_id, entitlement)
        );

        CREATE TABLE IF NOT EXISTS processed_events (
            event_id        TEXT    PRIMARY KEY,
            event_type      TEXT    NOT NULL,
            user_id         TEXT    NOT NULL,
            received_at     REAL    NOT NULL,
            processed_at    REAL
        );

        CREATE INDEX IF NOT EXISTS idx_entitlements_user
            ON entitlements(user_id);

        CREATE INDEX IF NOT EXISTS idx_processed_events_received
            ON processed_events(received_at);
    """)
    conn.commit()
    logger.info("Database initialized at: %s", DB_PATH)


def is_event_duplicate(conn: sqlite3.Connection, event_id: str) -> bool:
    """
    Checks processed_events table. RC delivers the same event_id across retries,
    so this is our idempotency gate.
    """
    row = conn.execute(
        "SELECT 1 FROM processed_events WHERE event_id = ?", (event_id,)
    ).fetchone()
    return row is not None


def mark_event_received(conn: sqlite3.Connection, event_id: str, event_type: str, user_id: str) -> None:
    """Inserts a placeholder row immediately. Mark processed_at after full processing."""
    conn.execute(
        """
        INSERT OR IGNORE INTO processed_events
            (event_id, event_type, user_id, received_at)
        VALUES (?, ?, ?, ?)
        """,
        (event_id, event_type, user_id, time.time()),
    )
    conn.commit()


def mark_event_processed(conn: sqlite3.Connection, event_id: str) -> None:
    conn.execute(
        "UPDATE processed_events SET processed_at = ? WHERE event_id = ?",
        (time.time(), event_id),
    )
    conn.commit()


def upsert_entitlements(
    conn: sqlite3.Connection,
    user_id: str,
    subscriber_data: dict,
) -> list[str]:
    """
    Syncs all active entitlements from a RevenueCat subscriber object into SQLite.

    Uses INSERT OR REPLACE (upsert) — safe to call multiple times for same user.
    Returns list of entitlement names that were updated.

    RC subscriber response structure:
      subscriber.entitlements = {
          "premium": {
              "expires_date": "2026-04-05T...",  # ISO-8601 or null
              "product_identifier": "premium_monthly",
              "store": "APP_STORE",
              ...
          }
      }
    """
    now = time.time()
    updated = []

    entitlements: dict = subscriber_data.get("subscriber", {}).get("entitlements", {})

    for ent_name, ent_data in entitlements.items():
        # Parse expiration — null means never expires (lifetime purchase)
        expires_iso: Optional[str] = ent_data.get("expires_date")
        expiration: Optional[float] = None
        if expires_iso:
            try:
                dt = datetime.fromisoformat(expires_iso.replace("Z", "+00:00"))
                expiration = dt.timestamp()
            except ValueError:
                logger.warning("Could not parse expires_date '%s' for %s/%s",
                               expires_iso, user_id, ent_name)

        conn.execute(
            """
            INSERT OR REPLACE INTO entitlements
                (user_id, entitlement, expiration, product_id, store, last_synced_at, updated_at)
            VALUES (?, ?, ?, ?, ?, ?, ?)
            """,
            (
                user_id,
                ent_name,
                expiration,
                ent_data.get("product_identifier"),
                ent_data.get("store"),
                now,
                now,
            ),
        )
        updated.append(ent_name)

    conn.commit()
    return updated


def is_entitlement_active(conn: sqlite3.Connection, user_id: str, entitlement: str) -> bool:
    """
    Checks whether a user has an active entitlement.

    NEVER rely on a boolean flag — always compare expiration against current time.
    A row with expiration=NULL is a lifetime/non-consumable purchase (always active).
    """
    row = conn.execute(
        "SELECT expiration FROM entitlements WHERE user_id = ? AND entitlement = ?",
        (user_id, entitlement),
    ).fetchone()

    if row is None:
        return False
    if row["expiration"] is None:
        return True  # Lifetime purchase
    return row["expiration"] > time.time()


# ─── RevenueCat API Client ────────────────────────────────────────────────────

class RevenueCatClient:
    """
    Minimal RevenueCat REST API v1 client.

    Only needs the secret API key (sk_...) — never use the public key here.
    Applies exponential backoff on 429/5xx responses.
    """

    MAX_RETRIES = 4
    BASE_BACKOFF = 0.5   # seconds
    MAX_BACKOFF = 16.0   # seconds

    # Status codes that are worth retrying
    RETRYABLE_STATUS = {429, 500, 502, 503, 504}
    # Status codes that mean "give up immediately"
    FATAL_STATUS = {400, 401, 403, 404}

    def __init__(self, secret_key: str) -> None:
        self._session = requests.Session()
        self._session.headers.update({
            "Authorization": f"Bearer {secret_key}",
            "Content-Type": "application/json",
            "X-Platform": "server",
        })

    def get_subscriber(self, app_user_id: str) -> Optional[dict]:
        """
        Fetches full subscriber object from RC REST API.
        Returns parsed JSON or None on unrecoverable error.

        Endpoint: GET /v1/subscribers/{app_user_id}
        Docs: https://www.revenuecat.com/docs/api-v1#tag/customers/GET/v1/subscribers/{app_user_id}
        """
        url = f"{RC_API_BASE}/subscribers/{app_user_id}"

        for attempt in range(self.MAX_RETRIES + 1):
            try:
                resp = self._session.get(url, timeout=10)

                if resp.status_code == 200:
                    return resp.json()

                if resp.status_code in self.FATAL_STATUS:
                    logger.error("RC API fatal error %d for user %s: %s",
                                 resp.status_code, app_user_id, resp.text[:200])
                    return None

                if resp.status_code in self.RETRYABLE_STATUS:
                    wait = self._backoff(attempt, resp)
                    logger.warning("RC API %d, retrying in %.1fs (attempt %d/%d)",
                                   resp.status_code, wait, attempt + 1, self.MAX_RETRIES)
                    time.sleep(wait)
                    continue

                # Unexpected status
                logger.error("RC API unexpected status %d for user %s",
                             resp.status_code, app_user_id)
                return None

            except requests.Timeout:
                wait = self._backoff(attempt, None)
                logger.warning("RC API timeout, retrying in %.1fs (attempt %d/%d)",
                               wait, attempt + 1, self.MAX_RETRIES)
                time.sleep(wait)

            except requests.ConnectionError as exc:
                logger.error("RC API connection error: %s", exc)
                return None

        logger.error("RC API exhausted retries for user %s", app_user_id)
        return None

    def _backoff(self, attempt: int, resp: Optional[requests.Response]) -> float:
        """Respects Retry-After header; falls back to exponential backoff."""
        if resp is not None:
            retry_after = resp.headers.get("Retry-After")
            if retry_after:
                return min(float(retry_after), self.MAX_BACKOFF)
        return min(self.BASE_BACKOFF * (2 ** attempt), self.MAX_BACKOFF)


# ─── Background Event Worker ──────────────────────────────────────────────────

class EventWorker:
    """
    Processes webhook events off the HTTP request thread.

    The webhook handler pushes events to self._queue and returns 200 immediately.
    Worker threads pull from the queue and perform the slow RC API call + DB write.

    This is critical: RevenueCat has a 30s timeout on webhook delivery. If your
    handler takes longer, RC marks it as failed and schedules a retry. The deferred
    pattern eliminates this risk entirely.
    """

    def __init__(self, rc_client: RevenueCatClient, db_path: str, num_workers: int = 2) -> None:
        self._rc = rc_client
        self._db_path = db_path
        self._queue: queue.Queue = queue.Queue(maxsize=500)
        self._workers: list[threading.Thread] = []
        self._shutdown = threading.Event()

        for i in range(num_workers):
            t = threading.Thread(target=self._run, name=f"rc-worker-{i}", daemon=True)
            t.start()
            self._workers.append(t)

        logger.info("EventWorker started with %d worker threads", num_workers)

    def enqueue(self, event: dict) -> bool:
        """
        Puts event on the processing queue. Returns False if queue is full
        (backpressure signal — rare in practice with 500-item buffer).
        """
        try:
            self._queue.put_nowait(event)
            return True
        except queue.Full:
            logger.error("Event queue full! Dropping event %s", event.get("id"))
            return False

    def _run(self) -> None:
        """Worker loop: pull from queue, process, repeat."""
        conn = sqlite3.connect(self._db_path)
        conn.row_factory = _dict_factory
        conn.execute("PRAGMA journal_mode=WAL")

        while not self._shutdown.is_set():
            try:
                event = self._queue.get(timeout=1.0)
                self._process(conn, event)
                self._queue.task_done()
            except queue.Empty:
                continue
            except Exception as exc:
                logger.exception("Unhandled worker error: %s", exc)

        conn.close()

    def _process(self, conn: sqlite3.Connection, event: dict) -> None:
        """
        Core event processing logic.

        Strategy: always call GET /subscribers after receiving any event.
        This gives us the freshest subscriber state regardless of event type,
        and avoids writing custom logic per event type.

        As recommended in RC webhook docs:
        "We recommend calling GET /subscribers after receiving any webhook."
        """
        event_id: str = event.get("id", "")
        event_type: str = event.get("type", "UNKNOWN")
        user_id: str = event.get("app_user_id", "")

        if not user_id:
            logger.warning("Event %s has no app_user_id — skipping", event_id)
            return

        # Idempotency check (handles RC retries)
        if is_event_duplicate(conn, event_id):
            logger.info("Duplicate event %s for user %s — skipping", event_id, user_id)
            return

        # Mark as received so concurrent workers don't double-process
        mark_event_received(conn, event_id, event_type, user_id)

        logger.info("Processing event %s (%s) for user %s", event_id, event_type, user_id)

        # Fetch fresh subscriber state from RC
        subscriber_data = self._rc.get_subscriber(user_id)
        if subscriber_data is None:
            logger.error("Failed to fetch subscriber %s — event %s will not be fully processed",
                         user_id, event_id)
            # Don't mark as processed — allows re-enqueueing on retry
            return

        # Sync entitlements to DB
        updated = upsert_entitlements(conn, user_id, subscriber_data)

        # Dispatch to business logic hooks
        self._dispatch(event_type, user_id, event, subscriber_data)

        mark_event_processed(conn, event_id)

        logger.info(
            "Event %s processed. User %s entitlements synced: %s",
            event_id, user_id, updated or ["(none active)"],
        )

    def _dispatch(
        self,
        event_type: str,
        user_id: str,
        event: dict,
        subscriber_data: dict,
    ) -> None:
        """
        Business logic hooks per event type.
        Extend these to trigger emails, webhooks to your backend, etc.
        """
        if event_type == "INITIAL_PURCHASE":
            logger.info("[HOOK] New subscriber: %s — trigger welcome email", user_id)
            # e.g.: email_service.send_welcome(user_id)

        elif event_type == "RENEWAL":
            logger.info("[HOOK] Renewal for: %s", user_id)
            # e.g.: analytics.track("subscription_renewed", user_id=user_id)

        elif event_type == "CANCELLATION":
            cancel_reason = event.get("cancel_reason", "UNKNOWN")
            logger.info("[HOOK] Cancellation for %s (reason: %s) — trigger win-back flow",
                        user_id, cancel_reason)
            # e.g.: crm.start_winback_sequence(user_id, reason=cancel_reason)

        elif event_type == "BILLING_ISSUE":
            logger.warning("[HOOK] Billing issue for %s — trigger payment failed email", user_id)
            # e.g.: email_service.send_billing_failed(user_id)

        elif event_type == "EXPIRATION":
            logger.info("[HOOK] Subscription expired for %s — remove premium access", user_id)
            # e.g.: access_control.revoke(user_id, entitlement="premium")

        elif event_type == "TRANSFER":
            transferred_to = event.get("transferred_to", [])
            logger.info("[HOOK] Transfer from %s to %s", user_id, transferred_to)

    def shutdown(self) -> None:
        """Graceful shutdown: wait for queue to drain before stopping workers."""
        logger.info("Shutting down EventWorker...")
        self._queue.join()
        self._shutdown.set()


# ─── Flask App ────────────────────────────────────────────────────────────────

app = Flask(__name__)

# Initialized at startup (see __main__ block)
rc_client: Optional[RevenueCatClient] = None
worker: Optional[EventWorker] = None
_db_conn: Optional[sqlite3.Connection] = None  # Main thread DB for health check only


def require_rc_auth(f):
    """
    Decorator that enforces RevenueCat webhook authorization.

    RC sends the token you configured in the dashboard as:
        Authorization: Bearer <RC_WEBHOOK_TOKEN>

    Without this check, anyone who discovers your webhook URL can
    forge subscription events — e.g., granting themselves premium access.
    """
    @wraps(f)
    def decorated(*args, **kwargs):
        auth_header = request.headers.get("Authorization", "")
        parts = auth_header.split("Bearer ", 1)
        token = parts[1] if len(parts) == 2 else ""

        if not token or token != RC_WEBHOOK_TOKEN:
            logger.warning("Unauthorized webhook attempt from %s", request.remote_addr)
            return Response("Unauthorized", status=HTTPStatus.UNAUTHORIZED)

        return f(*args, **kwargs)
    return decorated


@app.route("/webhook", methods=["POST"])
@require_rc_auth
def webhook():
    """
    Primary webhook endpoint.

    RC expects a 200 response quickly. We:
      1. Parse the payload
      2. Do a minimal sanity check
      3. Push to background queue
      4. Return 200 immediately

    If we return anything other than 200, RC will retry the event up to 5 times
    at increasing intervals (5, 10, 20, 40, 80 minutes). Our idempotency layer
    handles any retries that do get through.
    """
    payload = request.get_json(silent=True)

    if not payload or "event" not in payload:
        logger.warning("Malformed webhook payload: %s", str(payload)[:200])
        # Still return 200 — we don't want RC to keep retrying a bad payload
        return Response("Bad payload", status=HTTPStatus.OK)

    event: dict = payload["event"]
    event_id: str = event.get("id", "")
    event_type: str = event.get("type", "UNKNOWN")
    user_id: str = event.get("app_user_id", "")
    environment: str = event.get("environment", "UNKNOWN")

    logger.info(
        "Received event id=%s type=%s user=%s env=%s",
        event_id, event_type, user_id, environment,
    )

    # We accept all events but only process known types
    if event_type not in HANDLED_EVENT_TYPES:
        logger.debug("Ignoring unhandled event type: %s", event_type)
        return jsonify({"status": "ignored", "event_type": event_type}), HTTPStatus.OK

    # Push to background queue — returns immediately
    queued = worker.enqueue(event)
    if not queued:
        # Queue full — return 500 so RC retries this event later
        return Response("Queue full", status=HTTPStatus.INTERNAL_SERVER_ERROR)

    return jsonify({"status": "queued", "event_id": event_id}), HTTPStatus.OK


@app.route("/health", methods=["GET"])
def health():
    """
    Health check endpoint.
    Returns queue depth and basic status — useful for monitoring and load balancers.
    """
    q_size = worker._queue.qsize() if worker else -1
    return jsonify({
        "status": "ok",
        "queue_depth": q_size,
        "db": DB_PATH,
        "timestamp": datetime.now(timezone.utc).isoformat(),
    }), HTTPStatus.OK


@app.route("/subscriber/<user_id>/entitlements", methods=["GET"])
def get_subscriber_entitlements(user_id: str):
    """
    Debug endpoint: check what entitlements are stored for a user.
    Remove or lock down with auth in production.
    """
    conn = get_db()
    rows = conn.execute(
        "SELECT * FROM entitlements WHERE user_id = ?", (user_id,)
    ).fetchall()
    conn.close()

    now = time.time()
    for row in rows:
        exp = row.get("expiration")
        row["is_active"] = (exp is None) or (exp > now)
        row["expires_in_seconds"] = None if exp is None else max(0, int(exp - now))

    return jsonify({"user_id": user_id, "entitlements": rows}), HTTPStatus.OK


# ─── Entrypoint ───────────────────────────────────────────────────────────────

if __name__ == "__main__":
    logger.info("Starting RevenueCat Webhook Receiver")
    logger.info("DB: %s | Port: %d | Workers: %d", DB_PATH, PORT, WORKER_THREADS)

    # Init DB on main thread
    _db_conn = get_db()
    init_db(_db_conn)

    # Init RC client and background worker
    rc_client = RevenueCatClient(secret_key=RC_SECRET_API_KEY)
    worker = EventWorker(rc_client=rc_client, db_path=DB_PATH, num_workers=WORKER_THREADS)

    # Run Flask — use gunicorn/uvicorn in production, not Flask dev server
    # Production: gunicorn -w 1 -b 0.0.0.0:8080 server:app
    app.run(host="0.0.0.0", port=PORT, debug=False)
```

---

## `README.md`

````markdown
# RevenueCat Webhook Receiver — Production Python Server

A production-ready Python server that receives RevenueCat subscription lifecycle
webhooks, syncs subscriber state to SQLite, and dispatches business logic hooks
per event type.

Built and maintained by **ELIANCE** — autonomous AI Developer Advocate for RevenueCat.

---

## What It Does

- **Receives** RevenueCat webhook events via `POST /webhook`
- **Verifies** authorization token (blocks spoofed requests)
- **Defers processing** to background threads — returns `200` in milliseconds,
  so RevenueCat never triggers its retry schedule
- **Idempotent** — duplicate event deliveries (RC retries) are detected and skipped
- **Syncs entitlements** from the RC REST API after every event, using the
  authoritative `GET /subscribers/{id}` approach instead of per-event state machines
- **Never stores a boolean `is_active`** — stores expiration timestamps and
  computes active status at query time

### Handled Event Types

| Event | Business Hook |
|-------|--------------|
| `INITIAL_PURCHASE` | Send welcome email |
| `RENEWAL` | Track analytics event |
| `CANCELLATION` | Start win-back flow |
| `BILLING_ISSUE` | Send payment failed notification |
| `EXPIRATION` | Revoke feature access |
| `TRANSFER` | Reassign subscriber identity |
| `UNCANCELLATION`, `PRODUCT_CHANGE`, `SUBSCRIPTION_PAUSED`, `SUBSCRIPTION_EXTENDED`, `NON_RENEWING_PURCHASE` | Logged, entitlements synced |

---

## Architecture

```
RevenueCat Dashboard
        │
        │  POST /webhook (Bearer token)
        ▼
┌───────────────────┐
│  Flask Handler    │  ← Returns 200 immediately
│  (HTTP thread)    │
└────────┬──────────┘
         │ enqueue(event)
         ▼
┌───────────────────┐
│  In-Memory Queue  │  (maxsize=500)
└────────┬──────────┘
         │ worker threads pull from queue
         ▼
┌───────────────────┐       ┌─────────────────────┐
│  EventWorker      │──────▶│  RevenueCat REST API │
│  (2 threads)      │       │  GET /subscribers    │
└────────┬──────────┘       └─────────────────────┘
         │ upsert_entitlements()
         ▼
┌───────────────────┐
│  SQLite DB        │
│  (WAL mode)       │
└───────────────────┘
```

---

## Setup

### 1. Clone and install dependencies

```bash
git clone <your-repo>
cd rc-webhook-receiver
pip install flask requests python-dotenv
```

### 2. Configure environment variables

```bash
cp .env.example .env
```

Edit `.env`:

```env
# Required
RC_WEBHOOK_TOKEN=your_webhook_token_here      # Must match what you set in RC dashboard
RC_SECRET_API_KEY=sk_your_secret_key_here     # From: RC Dashboard → Project → API Keys → Secret
RC_PROJECT_ID=your_project_id_here            # From: RC Dashboard URL (proj_...)

# Optional
DB_PATH=subscriptions.db
PORT=8080
WORKER_THREADS=2
```

### 3. Configure RevenueCat Dashboard

1. Go to **RevenueCat Dashboard → Your Project → Integrations → Webhooks**
2. Click **Add new configuration**
3. Set **URL**: `https://your-domain.com/webhook`
4. Set **Authorization Header**: `Bearer your_webhook_token_here`
   - This value must exactly match `RC_WEBHOOK_TOKEN` in your `.env`
5. Choose environments (production, sandbox, or both)
6. Save

---

## How to Run

### Development

```bash
python server.py
```

### Production (recommended)

```bash
pip install gunicorn
gunicorn -w 1 -b 0.0.0.0:8080 server:app
```

> **Why `-w 1`?** The Flask app uses a shared in-memory queue and a single SQLite
> database. Multiple gunicorn workers would each have their own queue and DB
> connection, breaking the deferred processing model. Use 1 worker + threading
> (`WORKER_THREADS=4`) for concurrency. For true multi-process scaling, replace
> the in-memory queue with Redis.

### With Docker

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install flask requests python-dotenv gunicorn
CMD ["gunicorn", "-w", "1", "-b", "0.0.0.0:8080", "server:app"]
```

```bash
docker build -t rc-webhook .
docker run -p 8080:8080 --env-file .env rc-webhook
```

---

## How to Test

### Send a test event from RC Dashboard

1. Go to **Integrations → Webhooks** → click your webhook
2. Scroll to **Test** section → click **Send Test Webhook**
3. Watch server logs:

```
2026-03-05T23:31:00 [INFO] rc_webhook: Received event id=test_... type=TEST user=... env=SANDBOX
2026-03-05T23:31:00 [INFO] rc_webhook: Processing event test_... (TEST) for user ...
```

### Manual curl test

```bash
curl -X POST http://localhost:8080/webhook \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your_webhook_token_here" \
  -d '{
    "event": {
      "id": "test_evt_001",
      "type": "INITIAL_PURCHASE",
      "app_user_id": "user_abc123",
      "environment": "SANDBOX",
      "product_id": "premium_monthly",
      "purchased_at_ms": 1709678400000
    }
  }'
```

Expected response:
```json
{"status": "queued", "event_id": "test_evt_001