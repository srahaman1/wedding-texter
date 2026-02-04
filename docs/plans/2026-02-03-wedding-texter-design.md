# wedding-texter — Design Document

A local web tool for sending personalized SMS/MMS reminders to wedding guests via Twilio. Designed as a one-off personal tool first, with clean seams for future open-source generalization.

---

## Architecture

Three layers, single process:

- **Web layer** — Flask serves a local UI (Jinja2 templates, vanilla JS) at `localhost:5000`. Three pages: Dashboard, Guests, and Compose.
- **Data layer** — SQLite database, auto-created on first run. No database server to manage.
- **SMS layer** — Twilio handles all sending. Credentials stored in a `.env` file (gitignored). Supports both SMS and MMS.

A background scheduler (APScheduler) runs inside the same Flask process. It checks every 30 seconds for messages whose scheduled send time has arrived, fires them via Twilio, and updates delivery status.

### Running the app

```
pip install -r requirements.txt
python main.py
```

First run creates the SQLite database automatically. The app reads Twilio credentials from `.env` on startup and fails fast with a clear error if any are missing. A `TEST_MODE=true` flag in `.env` toggles Twilio's test mode (sends are accepted but not delivered) for development.

---

## Data Model

Three tables:

**guests**
| Column | Notes |
|---|---|
| id | Primary key |
| first_name | From CSV; optional (fallback greeting used if missing) |
| last_name | From CSV; optional |
| phone_number | Primary identifier for CSV sync; unique |
| active | Soft-delete flag; inactive guests are excluded from compose but preserved in send history |
| imported_at | Timestamp of last CSV import |

**messages**
| Column | Notes |
|---|---|
| id | Primary key |
| body | Message template; supports `{first_name}` and `{last_name}` placeholders |
| image_url | Optional; publicly accessible URL for MMS attachment |
| scheduled_at | Target send time; null means "send now" |
| created_at | Timestamp |

**sends**
| Column | Notes |
|---|---|
| id | Primary key |
| message_id | Foreign key → messages |
| guest_id | Foreign key → guests |
| status | `pending`, `sent`, `delivered`, `failed` |
| error | Twilio error message if failed; null otherwise |
| sent_at | Timestamp when actually sent; null if still pending |

---

## Message Flow

1. User composes a message in the Compose page: writes the body (with optional `{first_name}` personalization), optionally pastes an image URL or uses the shorten-URL button for any links, selects recipients, and chooses "Send Now" or a specific date/time.
2. One `messages` row is created. One `sends` row is created per selected recipient.
3. **Send Now:** each `sends` row is fired immediately via Twilio. Status updates to `sent`/`delivered`/`failed`.
4. **Scheduled:** `sends` rows stay `pending` until the background scheduler picks them up when the scheduled time arrives.
5. If the app is closed before a scheduled time, the sends don't fire until the app is restarted — the dashboard makes this visible so there are no surprises.
6. Failed sends can be retried individually from the Dashboard.

### Personalization

Message body supports `{first_name}` and `{last_name}` placeholders. Before sending, each message is rendered per-guest:
- If the guest has a first name, `{first_name}` is replaced with it.
- If not, a generic fallback (e.g., "Friend") is substituted.

The Compose screen shows a live preview using a real guest's name so you can see the output before sending.

---

## Pages

### Dashboard
- Upcoming scheduled messages with a countdown to send time.
- Recent send history with per-guest delivery status (pending / delivered / failed).
- Summary counts at a glance.
- Retry button on any failed send.
- Visual indicator if the app was down during a scheduled send time (messages that sent late).

### Guests
- Table of all guests with first name, last name, phone number, and active status.
- **Import CSV** button: drops or selects a `.csv` file. Syncs by phone number — new numbers are added, existing entries have their names updated. Guests missing from the new CSV are *not* auto-removed (prevents accidental deletion from an incomplete export).
- Manual removal from this page with a confirmation step. Removing a guest cancels any pending sends to them and flags them on the dashboard.
- Filter to show/hide inactive (removed) guests.

### Compose
- Message body textarea with live `{first_name}` preview using a real guest's name.
- Live character counter that updates as you type:
  - Tracks whether the message is in standard mode (160 chars/segment) or emoji mode (70 chars/segment).
  - Shows estimated segment count — highlights when you cross a segment boundary.
- Image URL field with a helper note: *"Host your image on Imgur, Google Drive (public link), etc. and paste the URL here. Twilio fetches the image directly — local files won't work."*
- Shorten URL button: paste a long URL, hit the button, it's replaced with a TinyURL short link (free API, no account required).
- Recipient selector: checkboxes per guest, or "Select All" / "Deselect All".
- Send options: "Send Now" button, or a date/time picker for scheduled sends.

---

## SMS / MMS Details

- **Provider:** Twilio. Pay-per-message, no monthly fee. A full wedding guest list will cost a few dollars total.
- **MMS:** Supported via Twilio's `MediaUrl` parameter. Requires a publicly accessible image URL — the Compose page makes this requirement clear.
- **Emojis:** Supported. Twilio handles encoding automatically. Using an emoji switches the message to UCS-2 encoding (70 chars/segment instead of 160). The character counter reflects this live.
- **Delivery status:** Twilio provides status callbacks (pending → sent → delivered → failed). Tracked in the `sends` table. Note: SMS does not support read receipts at the protocol level.
- **URL shortening:** Integrated via TinyURL's free API. A button in the Compose screen shortens any pasted URL in place.

---

## Guest List (CSV)

Expected columns:

| Column | Required |
|---|---|
| first_name | No (fallback greeting used if missing) |
| last_name | No |
| phone_number | Yes |

Import behavior:
- Matches on `phone_number`. New numbers are added; existing entries are updated.
- Missing guests are never auto-removed. Removal is always manual.
- Re-importing is safe to run at any time.

---

## Testing

- **Unit tests (pytest):** CSV parsing and guest syncing, template rendering (personalization + fallback), character counting and segment calculation, TinyURL integration.
- **Twilio test mode:** Set `TEST_MODE=true` in `.env` to use Twilio's test credentials. Sends are accepted and validated but never delivered. Swap to live credentials when ready to test a real send to your own phone.

---

## Out of Scope (Now) / Future Phases

- **RSVP page** — requires a publicly accessible web page (guests need to reach it from their phones). Likely a phase 2: deploy a lightweight public form (e.g., Vercel) that posts responses back. The current architecture doesn't block this.
- **Link click tracking** — a natural companion to RSVP. Route outgoing URLs through a local redirect that logs clicks before forwarding. Pairs with the URL shortener already in the design.
- **Local image upload** — currently requires a public URL due to how Twilio MMS works. A future enhancement could auto-host uploaded images (e.g., via ngrok or a simple file hosting service) to make this seamless.
- **Read receipts** — not possible over SMS at the protocol level. Link click tracking (above) is the practical substitute for engagement tracking.
