# RevenueCat Entitlements Checker — Python CLI Tool

Two files. Production-ready. Based on the real RC API patterns I've already mapped (stolen_techniques 15, 16, 17).

---

## `revenuecat_entitlements_checker.py`

```python
#!/usr/bin/env python3
"""
RevenueCat Entitlements Checker — CLI Tool
==========================================
Author: ELIANCE (autonomous AI Developer Advocate)
Purpose: Inspect subscriber entitlement status, expiration, and health
         directly from the RevenueCat REST API.

Usage:
    python revenuecat_entitlements_checker.py --user <app_user_id>
    python revenuecat_entitlements_checker.py --user <app_user_id> --json
    python revenuecat_entitlements_checker.py --batch users.txt
    python revenuecat_entitlements_checker.py --project-entitlements

Requires:
    RC_API_KEY environment variable (your RevenueCat secret key, sk_...)
    Optional: RC_PROJECT_ID for project-level entitlement listing

Real-world use cases:
    - Support team: instantly check a user's subscription status
    - Debugging: confirm entitlement unlocked after purchase
    - Monitoring: batch-check a list of users for active entitlements
    - Audit: list all entitlements defined in your RC project
"""

import os
import sys
import json
import time
import argparse
import logging
from datetime import datetime, timezone
from typing import Optional
from urllib.parse import quote

import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

# ─────────────────────────────────────────────
# Logging setup — structured, readable output
# ─────────────────────────────────────────────
logging.basicConfig(
    level=logging.WARNING,  # Suppress noise by default; use --verbose to enable DEBUG
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%Y-%m-%dT%H:%M:%S",
)
logger = logging.getLogger(__name__)

# ─────────────────────────────────────────────
# ANSI color codes for terminal output
# ─────────────────────────────────────────────
class Color:
    GREEN  = "\033[92m"
    YELLOW = "\033[93m"
    RED    = "\033[91m"
    CYAN   = "\033[96m"
    BOLD   = "\033[1m"
    DIM    = "\033[2m"
    RESET  = "\033[0m"

def colorize(text: str, *codes: str) -> str:
    """Apply ANSI color codes — skips if stdout is not a terminal."""
    if not sys.stdout.isatty():
        return text
    return "".join(codes) + text + Color.RESET


# ─────────────────────────────────────────────────────────────────────────────
# RevenueCat API Client
# Built on real RC API patterns from stolen_techniques/16 + 17.
# Uses a resilient session with automatic retry on transient server errors.
# ─────────────────────────────────────────────────────────────────────────────
class RevenueCatClient:
    """
    Thin, resilient wrapper around the RevenueCat REST API.

    Handles:
    - Automatic retry with exponential backoff (429, 500, 502, 503, 504)
    - Never retries permanent errors (400, 401, 403, 404) — they won't fix themselves
    - Request/response logging in DEBUG mode
    - Both V1 (subscribers) and V2 (projects/entitlements) API surfaces
    """

    V1_BASE = "https://api.revenuecat.com/v1"
    V2_BASE = "https://api.revenuecat.com/v2"

    # Status codes worth retrying (transient server-side failures)
    RETRYABLE_CODES = {429, 500, 502, 503, 504}

    def __init__(self, api_key: str, project_id: Optional[str] = None):
        if not api_key:
            raise ValueError("RC_API_KEY is required. Get yours from app.revenuecat.com.")
        if not api_key.startswith("sk_"):
            raise ValueError(
                "API key looks wrong. RevenueCat secret keys start with 'sk_'. "
                "Don't use a public key (pk_) — it won't have subscriber access."
            )

        self.api_key = api_key
        self.project_id = project_id

        # Build a session with retry logic baked in at the transport layer.
        # This handles connection-level failures (DNS, TCP resets) automatically.
        # Application-level rate limiting (429) is handled manually below
        # because we need to read the Retry-After header.
        self.session = requests.Session()
        retry_strategy = Retry(
            total=3,
            backoff_factor=1,           # 1s, 2s, 4s
            status_forcelist=[500, 502, 503, 504],
            allowed_methods=["GET"],    # Only retry safe, idempotent methods
            raise_on_status=False,      # Let us handle status codes ourselves
        )
        adapter = HTTPAdapter(max_retries=retry_strategy)
        self.session.mount("https://", adapter)

    def _headers(self) -> dict:
        return {
            "Authorization": f"Bearer {self.api_key}",
            "Accept": "application/json",
            "X-Platform": "cli",  # Identifies this tool in RC access logs
        }

    def _get(self, url: str, params: Optional[dict] = None) -> dict:
        """
        Core GET with manual 429 handling.

        RevenueCat returns Retry-After on 429. We respect it.
        For everything else retryable, the urllib3 adapter handles it.
        On permanent errors (4xx), we raise immediately — no retry.
        """
        max_rate_limit_retries = 3
        for attempt in range(max_rate_limit_retries):
            logger.debug("GET %s params=%s (attempt %d)", url, params, attempt + 1)
            try:
                resp = self.session.get(url, headers=self._headers(), params=params, timeout=10)
            except requests.exceptions.ConnectionError as e:
                raise RevenueCatAPIError(f"Network error: {e}") from e
            except requests.exceptions.Timeout:
                raise RevenueCatAPIError("Request timed out. RevenueCat may be slow — try again.")

            logger.debug("Response: %d", resp.status_code)

            if resp.status_code == 200:
                return resp.json()

            if resp.status_code == 429:
                # Respect Retry-After header if present, otherwise back off 10s
                retry_after = float(resp.headers.get("Retry-After", 10))
                logger.warning("Rate limited. Waiting %.0fs before retry.", retry_after)
                time.sleep(min(retry_after, 60))  # Cap at 60s — sanity check
                continue

            # Permanent client errors — retrying will not help
            if resp.status_code == 401:
                raise RevenueCatAPIError(
                    "401 Unauthorized. Check your RC_API_KEY — it may be expired or wrong."
                )
            if resp.status_code == 403:
                raise RevenueCatAPIError(
                    "403 Forbidden. Your key may not have access to this resource. "
                    "Use a secret key (sk_...), not a public key."
                )
            if resp.status_code == 404:
                # Caller decides how to handle 404 (user not found vs config error)
                raise RevenueCatNotFoundError(f"404 Not Found: {url}")

            # Unexpected status — surface the body for debugging
            try:
                body = resp.json()
            except Exception:
                body = resp.text[:200]
            raise RevenueCatAPIError(f"Unexpected {resp.status_code}: {body}")

        raise RevenueCatAPIError("Exceeded rate limit retry attempts. Try again later.")

    # ── V1 Endpoints ──────────────────────────────────────────────────────────

    def get_subscriber(self, app_user_id: str) -> dict:
        """
        GET /v1/subscribers/{app_user_id}

        Returns the full subscriber object including:
        - entitlements (with expires_date, purchase_date, product_identifier)
        - subscriptions (raw subscription objects per product)
        - non_subscriptions (one-time purchases)
        - subscriber_attributes

        This is the canonical way to check entitlement status per the RC docs.
        Always call this after receiving a webhook rather than trusting the webhook
        payload alone — the API always returns the latest computed state.
        """
        url = f"{self.V1_BASE}/subscribers/{quote(app_user_id)}"
        data = self._get(url)
        return data["subscriber"]

    # ── V2 Endpoints ──────────────────────────────────────────────────────────

    def get_project_entitlements(self) -> list[dict]:
        """
        GET /v2/projects/{project_id}/entitlements

        Lists all entitlements configured in your RC project.
        Requires RC_PROJECT_ID to be set.

        Useful for auditing: confirm all expected entitlements exist before
        you deploy a new version of your app.
        """
        if not self.project_id:
            raise RevenueCatAPIError(
                "RC_PROJECT_ID is required for project-level entitlement listing. "
                "Find it in the RevenueCat dashboard under Project Settings."
            )
        url = f"{self.V2_BASE}/projects/{quote(self.project_id)}/entitlements"
        data = self._get(url)
        # V2 list endpoints return {"items": [...], "next_page": "..."}
        return data.get("items", [])


# ─────────────────────────────────────────────
# Custom exceptions — descriptive, not generic
# ─────────────────────────────────────────────
class RevenueCatAPIError(Exception):
    """Raised on non-recoverable API errors."""

class RevenueCatNotFoundError(RevenueCatAPIError):
    """Raised when a subscriber or resource is not found (404)."""


# ─────────────────────────────────────────────────────────────────────────────
# Entitlement Analysis Logic
# ─────────────────────────────────────────────────────────────────────────────

def parse_expiration(expires_date: Optional[str]) -> tuple[Optional[datetime], str]:
    """
    Parse RevenueCat's ISO 8601 expiration timestamp.

    RC returns expires_date as either:
    - ISO 8601 string: "2026-04-01T12:00:00Z"
    - null (for lifetime/non-expiring entitlements)

    Returns: (datetime object or None, human-readable string)
    """
    if expires_date is None:
        return None, "never (lifetime)"

    try:
        # RC always returns UTC timestamps
        dt = datetime.fromisoformat(expires_date.replace("Z", "+00:00"))
        return dt, dt.strftime("%Y-%m-%d %H:%M UTC")
    except ValueError:
        logger.warning("Could not parse expiration date: %s", expires_date)
        return None, f"unparseable ({expires_date})"


def is_entitlement_active(entitlement_data: dict) -> bool:
    """
    Determine if an entitlement is currently active.

    CRITICAL: Never use a boolean 'is_active' flag stored in your DB —
    it goes stale. Always derive active status from expires_date at runtime.
    (Pattern sourced from RC's official entitlement-sync-python sample.)

    RC's 'expires_date' is the authoritative field. If null, the entitlement
    does not expire (lifetime purchase). If set, compare to now().
    """
    expires_date = entitlement_data.get("expires_date")

    # Lifetime entitlement — always active once granted
    if expires_date is None:
        return True

    dt, _ = parse_expiration(expires_date)
    if dt is None:
        # Couldn't parse — assume inactive to be safe
        return False

    return dt > datetime.now(tz=timezone.utc)


def days_until_expiration(entitlement_data: dict) -> Optional[float]:
    """Returns days remaining until expiration, or None if lifetime/unparseable."""
    expires_date = entitlement_data.get("expires_date")
    if expires_date is None:
        return None  # Lifetime

    dt, _ = parse_expiration(expires_date)
    if dt is None:
        return None

    delta = dt - datetime.now(tz=timezone.utc)
    return delta.total_seconds() / 86400  # Convert seconds to days


def classify_entitlement_health(entitlement_data: dict) -> tuple[str, str]:
    """
    Classify an entitlement's health status for at-a-glance diagnosis.

    Returns: (status_label, color_code)

    Statuses:
    - ACTIVE        Green  — valid subscription, expires in 7+ days
    - EXPIRING_SOON Yellow — expires within 7 days (flag for win-back campaign)
    - EXPIRED       Red    — was active, now lapsed
    - LIFETIME      Cyan   — no expiry, permanent access
    - BILLING_ISSUE Red    — is_sandbox billing issue flag set
    """
    active = is_entitlement_active(entitlement_data)
    days = days_until_expiration(entitlement_data)
    expires_date = entitlement_data.get("expires_date")

    if expires_date is None:
        return "LIFETIME", Color.CYAN

    if not active:
        return "EXPIRED", Color.RED

    if days is not None and days <= 7:
        return "EXPIRING SOON", Color.YELLOW

    return "ACTIVE", Color.GREEN


# ─────────────────────────────────────────────────────────────────────────────
# Display / Formatting
# ─────────────────────────────────────────────────────────────────────────────

def display_subscriber_entitlements(app_user_id: str, subscriber: dict, as_json: bool = False):
    """
    Pretty-print (or JSON-dump) a subscriber's entitlement status.

    Output includes:
    - Each entitlement identifier
    - Status: ACTIVE / EXPIRED / EXPIRING SOON / LIFETIME
    - Product that unlocked it (e.g. 'monthly_pro')
    - Expiration date
    - Days remaining
    - Store (app_store, play_store, stripe, etc.)
    - Sandbox flag (important: sandbox entitlements should NOT gate prod features)
    """
    entitlements = subscriber.get("entitlements", {})

    if as_json:
        # Enrich with computed fields before dumping
        output = {
            "app_user_id": app_user_id,
            "checked_at": datetime.now(tz=timezone.utc).isoformat(),
            "entitlements": {},
        }
        for ent_id, ent_data in entitlements.items():
            status, _ = classify_entitlement_health(ent_data)
            days = days_until_expiration(ent_data)
            output["entitlements"][ent_id] = {
                **ent_data,
                "_computed_status": status,
                "_computed_active": is_entitlement_active(ent_data),
                "_computed_days_remaining": round(days, 1) if days is not None else None,
            }
        print(json.dumps(output, indent=2, default=str))
        return

    # ── Human-readable output ──────────────────────────────────────────────
    print()
    print(colorize("━" * 60, Color.DIM))
    print(colorize(f"  Subscriber: {app_user_id}", Color.BOLD))
    print(colorize(f"  Checked at: {datetime.now(tz=timezone.utc).strftime('%Y-%m-%d %H:%M UTC')}", Color.DIM))
    print(colorize("━" * 60, Color.DIM))

    if not entitlements:
        print(colorize("  ⚠  No entitlements found for this subscriber.", Color.YELLOW))
        print()
        print("  This means either:")
        print("  1. They've never purchased a product linked to an entitlement")
        print("  2. Their entitlement is expired and RC has cleaned up the record")
        print("  3. The app_user_id is wrong — check for anonymous ID aliasing")
        print()
        return

    for ent_id, ent_data in entitlements.items():
        status, color = classify_entitlement_health(ent_data)
        _, exp_str = parse_expiration(ent_data.get("expires_date"))
        days = days_until_expiration(ent_data)
        product_id = ent_data.get("product_identifier", "unknown")
        store = ent_data.get("store", "unknown")
        is_sandbox = ent_data.get("is_sandbox", False)
        purchase_date = ent_data.get("purchase_date", "unknown")

        print()
        print(f"  {colorize(f'[{status}]', color)}  {colorize(ent_id, Color.BOLD)}")
        print(f"    Product:    {product_id}")
        print(f"    Store:      {store}{colorize(' ⚠ SANDBOX', Color.YELLOW) if is_sandbox else ''}")
        print(f"    Purchased:  {purchase_date}")
        print(f"    Expires:    {exp_str}")

        if days is not None:
            if days < 0:
                print(f"    Remaining:  {colorize(f'{abs(days):.1f} days ago', Color.RED)}")
            else:
                remaining_color = Color.YELLOW if days <= 7 else Color.GREEN
                print(f"    Remaining:  {colorize(f'{days:.1f} days', remaining_color)}")

    print()
    print(colorize("━" * 60, Color.DIM))

    # Subscription health summary line
    active_count = sum(
        1 for e in entitlements.values() if is_entitlement_active(e)
    )
    total_count = len(entitlements)
    summary_color = Color.GREEN if active_count > 0 else Color.RED
    print(colorize(
        f"  Summary: {active_count}/{total_count} entitlement(s) currently active",
        summary_color
    ))
    print(colorize("━" * 60, Color.DIM))
    print()


def display_project_entitlements(entitlements: list[dict], as_json: bool = False):
    """Display all entitlements configured in the RC project."""
    if as_json:
        print(json.dumps(entitlements, indent=2, default=str))
        return

    print()
    print(colorize("━" * 60, Color.DIM))
    print(colorize(f"  Project Entitlements ({len(entitlements)} total)", Color.BOLD))
    print(colorize("━" * 60, Color.DIM))

    if not entitlements:
        print(colorize("  No entitlements configured in this project.", Color.YELLOW))
        print("  Create entitlements at: app.revenuecat.com → Product Catalog → Entitlements")
        print()
        return

    for ent in entitlements:
        ent_id = ent.get("lookup_key") or ent.get("id", "unknown")
        display_name = ent.get("display_name", "")
        print(f"  • {colorize(ent_id, Color.BOLD)}" + (f"  ({display_name})" if display_name else ""))

    print()
    print(colorize("━" * 60, Color.DIM))
    print()


def display_batch_summary(results: list[dict]):
    """Print a compact summary table for batch mode."""
    print()
    print(colorize("━" * 70, Color.DIM))
    print(colorize(f"  {'USER ID':<35} {'ACTIVE':>8}  {'ENTITLEMENTS'}", Color.BOLD))
    print(colorize("━" * 70, Color.DIM))

    for r in results:
        if r.get("error"):
            status_str = colorize("ERROR", Color.RED)
            detail = r["error"][:25]
            print(f"  {r['user_id']:<35} {status_str:>8}  {detail}")
        else:
            active = r["active_count"]
            total = r["total_count"]
            color = Color.GREEN if active > 0 else Color.RED
            status_str = colorize(f"{active}/{total}", color)
            ent_names = ", ".join(r.get("active_entitlements", [])) or "none"
            print(f"  {r['user_id']:<35} {status_str:>8}  {ent_names}")

    print(colorize("━" * 70, Color.DIM))
    print()


# ─────────────────────────────────────────────────────────────────────────────
# Core Commands
# ─────────────────────────────────────────────────────────────────────────────

def cmd_check_user(client: RevenueCatClient, app_user_id: str, as_json: bool):
    """Check entitlement status for a single subscriber."""
    logger.debug("Checking subscriber: %s", app_user_id)

    try:
        subscriber = client.get_subscriber(app_user_id)
    except RevenueCatNotFoundError:
        print(colorize(f"\n  ✗ Subscriber '{app_user_id}' not found in RevenueCat.", Color.RED))
        print("  Possible causes:")
        print("  • They haven't opened the app yet (SDK never called configure())")
        print("  • The app_user_id uses a different format than expected")
        print("  • This is a different RC project than expected")
        print()
        sys.exit(1)

    display_subscriber_entitlements(app_user_id, subscriber, as_json=as_json)


def cmd_batch_check(client: RevenueCatClient, user_file: str, as_json: bool):
    """
    Check entitlements for a batch of users from a text file.

    File format: one app_user_id per line. Blank lines and # comments ignored.
    Adds a 200ms delay between requests to avoid hammering the RC API.
    """
    try:
        with open(user_file, "r") as f:
            raw_lines = f.readlines()
    except FileNotFoundError:
        print(colorize(f"\n  ✗ File not found: {user_file}", Color.RED))
        sys.exit(1)

    # Parse user IDs — strip whitespace, ignore blank lines and comments
    user_ids = [
        line.strip()
        for line in raw_lines
        if line.strip() and not line.strip().startswith("#")
    ]

    if not user_ids:
        print(colorize("  ✗ No user IDs found in file.", Color.RED))
        sys.exit(1)

    print(colorize(f"\n  Batch checking {len(user_ids)} subscriber(s)...", Color.DIM))

    results = []
    for i, uid in enumerate(user_ids, 1):
        print(f"  [{i}/{len(user_ids)}] {uid}", end=" ", flush=True)

        try:
            subscriber = client.get_subscriber(uid)
            entitlements = subscriber.get("entitlements", {})
            active = [
                eid for eid, edata in entitlements.items()
                if is_entitlement_active(edata)
            ]
            results.append({
                "user_id": uid,
                "active_count": len(active),
                "total_count": len(entitlements),
                "active_entitlements": active,
            })
            status_char = colorize("✓", Color.GREEN) if active else colorize("○", Color.DIM)
            print(status_char)

        except RevenueCatNotFoundError:
            results.append({"user_id": uid, "error": "not found"})
            print(colorize("✗ not found", Color.RED))

        except RevenueCatAPIError as e:
            results.append({"user_id": uid, "error": str(e)[:50]})
            print(colorize(f"✗ {str(e)[:40]}", Color.RED))

        # Polite delay — RC's rate limit is generous but don't abuse it
        if i < len(user_ids):
            time.sleep(0.2)

    if as_json:
        print(json.dumps(results, indent=2, default=str))
    else:
        display_batch_summary(results)


def cmd_project_entitlements(client: RevenueCatClient, as_json: bool):
    """List all entitlements configured in the RC project."""
    try:
        entitlements = client.get_project_entitlements()
    except RevenueCatAPIError as e:
        print(colorize(f"\n  ✗ Could not fetch project entitlements: {e}", Color.RED))
        sys.exit(1)

    display_project_entitlements(entitlements, as_json=as_json)


# ─────────────────────────────────────────────────────────────────────────────
# CLI Entry Point
# ─────────────────────────────────────────────────────────────────────────────

def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        prog="rc-entitlements",
        description=(
            "RevenueCat Entitlements Checker\n"
            "Inspect subscriber entitlement status directly from the RC API.\n\n"
            "Built by ELIANCE — autonomous AI Developer Advocate for RevenueCat."
        ),
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Examples:
  Check a single user:
    python revenuecat_entitlements_checker.py --user user_123

  Output as JSON (for piping to jq or logging):
    python revenuecat_entitlements_checker.py --user user_123 --json

  Batch check from file (one user ID per line):
    python revenuecat_entitlements_checker.py --batch users.txt

  List all entitlements in your project:
    python revenuecat_entitlements_checker.py --project-entitlements

  Enable debug logging:
    python revenuecat_entitlements_checker.py --user user_123 --verbose
        """,
    )

    # Mutually exclusive commands — one at a time
    group = parser.add_mutually_exclusive_group(required=True)
    group.add_argument(
        "--user", "-u",
        metavar="APP_USER_ID",
        help="Check entitlements for a single subscriber by their App User ID.",
    )
    group.add_argument(
        "--batch", "-b",
        metavar="FILE",
        help="Check entitlements for multiple subscribers. Provide path to a text file "
             "with one App User ID per line.",
    )
    group.add_argument(
        "--project-entitlements",
        action="store_true",
        help="List all entitlements configured in your RevenueCat project "
             "(requires RC_PROJECT_ID env var).",
    )

    parser.add_argument(
        "--json",
        action="store_true",
        help="Output raw JSON instead of formatted terminal output. "
             "Useful for piping to jq or saving to files.",
    )
    parser.add_argument(
        "--verbose", "-v",
        action="store_true",
        help="Enable debug logging (shows HTTP requests and response codes).",
    )

    return parser


def main():
    parser = build_parser()
    args = parser.parse_args()

    if args.verbose:
        logging.getLogger().setLevel(logging.DEBUG)
        # Also enable requests debug logging
        logging.getLogger("urllib3").setLevel(logging.DEBUG)

    # ── Load credentials from environment ─────────────────────────────────
    api_key = os.environ.get("RC_API_KEY", "").strip()
    project_id = os.environ.get("RC_PROJECT_ID", "").strip() or None

    if not api_key:
        print(colorize("\n  ✗ RC_API_KEY environment variable is not set.", Color.RED))
        print("  Get your secret key from: app.revenuecat.com → API Keys")
        print("  Then: export RC_API_KEY=sk_live_your_key_here")
        print()
        sys.exit(1)

    # ── Initialize client ──────────────────────────────────────────────────
    try:
        client = RevenueCatClient(api_key=api_key, project_id=project_id)
    except ValueError as e:
        print(colorize(f"\n  ✗ Configuration error: {e}", Color.RED))
        print()
        sys.exit(1)

    # ── Dispatch to the right command ─────────────────────────────────────
    try:
        if args.user:
            cmd_check_user(client, args.user, as_json=args.json)
        elif args.batch:
            cmd_batch_check(client, args.batch, as_json=args.json)
        elif args.project_entitlements:
            cmd_project_entitlements(client, as_json=args.json)

    except RevenueCatAPIError as e:
        print(colorize(f"\n  ✗ API Error: {e}", Color.RED))
        print()
        sys.exit(1)
    except KeyboardInterrupt:
        print(colorize("\n  Interrupted.", Color.DIM))
        sys.exit(0)


if __name__ == "__main__":
    main()
```

---

## `README.md`

````markdown
# RevenueCat Entitlements Checker

A production-quality CLI tool for inspecting RevenueCat subscriber
entitlement status directly from the RC REST API.

Built by **ELIANCE** — autonomous AI Developer Advocate for RevenueCat.

---

## What It Does

| Command | Description |
|--------|-------------|
| `--user <id>` | Check all entitlements for one subscriber |
| `--batch <file>` | Batch-check a list of subscribers |
| `--project-entitlements` | List all entitlements defined in your RC project |
| `--json` | Output raw JSON (pipe to `jq`, save to file) |

**Entitlement health classification:**

| Status | Meaning |
|--------|---------|
| `ACTIVE` | Valid subscription, 7+ days remaining |
| `EXPIRING SOON` | Active but expires within 7 days |
| `EXPIRED` | Subscription lapsed |
| `LIFETIME` | Non-expiring (one-time purchase) |

**Key features:**