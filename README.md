# AI-Whatsapp-Real-Estate-Agent


# Real Estate WhatsApp AI Agent — local n8n + local Ollama

A WhatsApp assistant that finds available plots/properties from a Google Sheet, shows the top matches, and books viewings (name → email → date → time), then saves the appointment and sends Gmail notifications.

* **n8n (local)** runs the workflow and owns all logic and data writes.
* **Ollama (local)** only *understands* the message (JSON extraction) and writes friendly wording. It never writes to your spreadsheet and never supplies prices or availability.
* **Google Sheets** is the database. **WhatsApp Cloud API** is the channel. **Gmail** sends the notifications.

## Files in this package

| File | Purpose |
|---|---|
| `real-estate-whatsapp-agent.workflow.json` | Main n8n workflow (41 nodes) |
| `real-estate-error-alert.workflow.json` | Optional: emails you when the main workflow fails |
| `Properties.csv` | 20 sample listings (replace with yours) |
| `Appointments.csv` | Headers only |
| `Customers.csv` | Headers only |
| `ConversationState.csv` | Headers only |

> The CSVs contain a few more columns than the original spec: `LastResultIDs`, `LastCriteria`, `LastMessageID` (ConversationState) and `CustomerEmailStatus`, `AdminEmailStatus` (Appointments). They are needed to remember which listings were shown, to ignore duplicate webhook deliveries, and to track email failures separately from the booking. **Do not remove or rename columns.**

---

## 1. How it works

```
Customer ──WhatsApp──> Meta Cloud API ──> n8n webhook (POST, replies 200 at once)
   Config → Parse Incoming (drops statuses/own msgs/duplicates)
   → read ConversationState, Customers, Properties (Google Sheets)
   → Analyze Message (deterministic parsing: IDs, email, dates, budgets, locations)
   → [Ollama: extract intent JSON]  (skipped when not needed)
   → Resolve Action (state machine) ──┬─ reply            ─────────────┐
                                      ├─ llm_reply → Ollama wording ───┤
                                      └─ save_booking → check duplicate/slot → append Appointments
                                           → Gmail (customer) → Gmail (admin) → status row ───┤
   → Build Outbound → WhatsApp send → upsert ConversationState → upsert Customers
```

Conversation stages (`ConversationState.Stage`): `idle`, `results_shown`, `ask_name`, `ask_email`, `ask_date`, `ask_time`, `completed`.

Customer commands (any time): **MENU / RESET / START OVER** (restart), **CANCEL** (abort a booking), **CHANGE DETAILS** (during date/time step, to re-enter name and email).

Safeguards built in:

* Results come only from rows with `Status = Available`; if nothing matches, the bot says so (and suggests real locations / the cheapest real price) — it never invents listings.
* Ollama's wording is rejected and replaced with a safe template if it contains currency symbols or numbers that weren't in the customer's message.
* Dates are validated: not in the past, within `maxAdvanceDays`, allowed weekdays, inside visiting hours.
* Slot check: a slot is refused if an active (non-cancelled) appointment already exists at the same date/time for the same property **or the same agent**. A repeat of an identical booking returns the existing reference instead of creating another.
* Duplicate Meta deliveries are ignored (message-ID cache in workflow static data + `LastMessageID` in the sheet).
* The final WhatsApp confirmation is only sent after the appointment row is saved. If saving fails, the customer is told it was *not* recorded and can retry.
* Email failure never blocks the booking; the result is written to `CustomerEmailStatus` / `AdminEmailStatus` so you can retry manually.
* If Ollama is down or times out, the workflow falls back to deterministic parsing and template replies.

---

## 2. Prerequisites

* A computer/server that stays on (n8n and Ollama run locally).
* Node.js 20+ **or** Docker for n8n.
* A Google account (Sheets + Gmail), a Meta developer account with a WhatsApp Business account.
* A way to expose local n8n over HTTPS (Meta cannot call `localhost`): ngrok or Cloudflare Tunnel (free options below).
* Hardware for Ollama: `qwen2.5:3b` runs on ~4 GB RAM; `qwen2.5:7b` wants ~8 GB+.

---

## 3. Install Ollama and the model

1. Install from https://ollama.com/download (Windows, macOS, Linux).
2. Pull a model and test it:
   ```bash
   ollama pull qwen2.5:3b        # lighter hardware
   # ollama pull qwen2.5:7b      # better understanding, needs more RAM
   ollama run qwen2.5:3b "say hi"
   curl http://localhost:11434/api/tags      # should list the model
   ```
3. Keep it warm (optional, avoids slow first replies): the workflow already sends `keep_alive: 30m`.

If you change the model, edit `ollamaModel` in the **Config** node.

---

## 4. Run n8n locally with a public HTTPS URL

### 4a. Create a tunnel (pick one)

**Cloudflare quick tunnel (no account):**
```bash
cloudflared tunnel --url http://localhost:5678
```
**ngrok:**
```bash
ngrok http 5678
```
Copy the `https://….` URL it prints. Quick-tunnel URLs change on restart; for permanent use create a named Cloudflare tunnel or a reserved ngrok domain, otherwise you must update Meta every time.

### 4b. Start n8n with that URL

**Docker (recommended):**
```bash
docker run -it --rm --name n8n -p 5678:5678 \
  -e WEBHOOK_URL=https://YOUR-TUNNEL-URL/ \
  -e GENERIC_TIMEZONE=Africa/Lagos -e TZ=Africa/Lagos \
  --add-host=host.docker.internal:host-gateway \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```
Because n8n is inside Docker, `localhost` means the container. Therefore:
* In **Config**, set `ollamaUrl` to `http://host.docker.internal:11434`.
* Make Ollama listen on all interfaces: set the environment variable `OLLAMA_HOST=0.0.0.0` and restart Ollama (Linux systemd: `sudo systemctl edit ollama` → add `Environment="OLLAMA_HOST=0.0.0.0"`).

**npm (no Docker):**
```bash
npm install -g n8n
WEBHOOK_URL=https://YOUR-TUNNEL-URL/ GENERIC_TIMEZONE=Africa/Lagos n8n start
```
Here `ollamaUrl` stays `http://localhost:11434`.

Open http://localhost:5678 and create your owner account. (Windows PowerShell: use `$env:WEBHOOK_URL="https://…"; n8n start`.)

---

## 5. Create the Google Sheet

1. Go to https://sheets.google.com → **Blank spreadsheet**. Name it `Real Estate AI Agent Database`.
2. For **each** CSV: **File → Import → Upload** the file → Import location **Insert new sheet(s)** → **untick “Convert text to numbers, dates, and formulas”** → Import data.
   Importing creates a tab named after the file: `Properties`, `Appointments`, `Customers`, `ConversationState`.
3. Delete the empty default `Sheet1`.
4. Verify the four tab names are *exactly*: `Properties`, `Appointments`, `Customers`, `ConversationState` (case-sensitive) and row 1 holds the headers.
5. Copy the spreadsheet ID from the URL: `https://docs.google.com/spreadsheets/d/<THIS-PART>/edit`.

Properties data rules:
* `Status` must be `Available` for a plot to be offered (Sold/Reserved/… are hidden).
* `Price` is a plain number; `Currency` is a code (USD, NGN, EUR, GBP…). The bot only filters by currency when the customer explicitly mentions one.
* Use consistent names in `Location` (city/region) and `Area` (neighbourhood): customers' words are matched against both.
* Use IDs shaped like `P001` (letter(s) + digits); the bot also understands `p1`, `P-001`.
* `Size` like `500 sqm`, `2 acres`, `1 hectare`.
* `ImageURL` must be a public JPEG/PNG URL if you enable image sending.
* Keep phone numbers as text (`+2348…`).

---

## 6. Google credentials in local n8n (Sheets + Gmail)

1. https://console.cloud.google.com → create a project.
2. **APIs & Services → Library**: enable **Google Sheets API**, **Gmail API** (and **Google Drive API**).
3. **OAuth consent screen**: External → add your Google account under **Test users**.
4. **Credentials → Create credentials → OAuth client ID → Web application**. Add this **Authorized redirect URI**:
   ```
   http://localhost:5678/rest/oauth2-credential/callback
   ```
   (Use your tunnel URL instead if you open n8n through it.) Copy Client ID and Secret.
5. In n8n: **Credentials → New → Google Sheets OAuth2 API** → paste ID/secret → **Sign in with Google**. Repeat with **Gmail OAuth2 API**.

> ⚠️ While the consent screen is in **Testing**, Google expires refresh tokens after 7 days and you must reconnect. For production, set the publishing status to **In production** (you'll see an “unverified app” warning for personal use, which is fine).

---

## 7. WhatsApp Cloud API setup

1. https://developers.facebook.com → **Create App** → type **Business** → add the **WhatsApp** product.
2. **WhatsApp → API Setup**: note the **Phone number ID** (not the phone number) and add your own phone as a test recipient (the free test number can only message approved recipients).
3. **Token**: the page shows a *temporary* 24-hour token. For production create a **System User** token in Business Settings (permissions `whatsapp_business_messaging`, `whatsapp_business_management`) so it does not expire.
4. In n8n create credential **Header Auth** named exactly `WhatsApp Cloud API Token`:
   * Name: `Authorization`
   * Value: `Bearer YOUR_ACCESS_TOKEN`
5. Webhook (do this **after** step 8 below, once the workflow is active): **WhatsApp → Configuration → Webhook → Edit**
   * Callback URL: `https://YOUR-TUNNEL-URL/webhook/whatsapp`
   * Verify token: same value as `verifyToken` in the Config node
   * Verify and save, then **Manage → subscribe to `messages`**.

Customer-service window: the bot only replies to customer-initiated chats within 24 hours of their last message, which is how this workflow behaves. Messaging first/after 24 h needs approved templates (not included).

---

## 8. Import and configure the workflow

1. n8n → **Workflows → ⋯ → Import from file** → `real-estate-whatsapp-agent.workflow.json`.
2. Open the **Config** node and edit:

| Field | Meaning |
|---|---|
| `businessName` | Shown in greetings/emails |
| `verifyToken` | Any secret string; must match Meta |
| `sheetId` | Spreadsheet ID from step 5 |
| `phoneNumberId` | WhatsApp Phone number ID (events for other numbers are ignored) |
| `graphApiVersion` | Use a currently supported Graph API version (Meta auto-upgrades old ones) |
| `adminEmail` | Receives new-appointment alerts |
| `ollamaUrl` / `ollamaModel` | See section 4; e.g. `qwen2.5:3b` |
| `timezone` | IANA zone, used for “today/tomorrow”, past-date checks |
| `dateOrder` | `DMY` (20/10/2026 = 20 Oct) or `MDY` |
| `maxResults` | Listings returned (default 5) |
| `ranking` | `price_asc`, `price_desc`, or `closest_to_budget` |
| `sendImages` | `true` sends each result's `ImageURL` as a photo after the text |
| `visitDays` | Allowed weekdays, 0=Sun … 6=Sat (default Mon–Sat) |
| `visitEarliest` / `visitLatest` | Visiting hours, 24h `HH:MM` |
| `maxAdvanceDays` | How far ahead visits can be booked |
| `sessionTimeoutMinutes` | A half-finished booking older than this is reset |

3. **Assign credentials** on these nodes (nodes with a red warning): all Google Sheets nodes (Read…, Append Appointment, Update Notification Status, Save State, Save Customer), the two Gmail nodes, and **Send WhatsApp** (Header Auth).
4. Optional: import `real-estate-error-alert.workflow.json`, set its Gmail credential and recipient, then in the main workflow **Settings → Error workflow** select it.
5. Toggle the main workflow **Active**.

---

## 9. Testing

**Step A – verification handshake** (replace values):
```bash
curl "https://YOUR-TUNNEL-URL/webhook/whatsapp?hub.mode=subscribe&hub.verify_token=YOUR_VERIFY_TOKEN&hub.challenge=12345"
# should print: 12345   (wrong token → Forbidden)
```

**Step B – simulate a customer without Meta** (the reply is attempted to that number via WhatsApp; use your approved test number; change `id` for every test, otherwise it is treated as a duplicate):
```bash
curl -X POST https://YOUR-TUNNEL-URL/webhook/whatsapp -H "Content-Type: application/json" -d '{
 "entry":[{"changes":[{"value":{
  "metadata":{"phone_number_id":"YOUR_PHONE_NUMBER_ID","display_phone_number":"15550000000"},
  "contacts":[{"profile":{"name":"Test"}}],
  "messages":[{"from":"YOUR_TEST_NUMBER_DIGITS","id":"wamid.TEST001","type":"text","text":{"body":"I need a plot in Washington, budget $50,000"}}]}}]}]}'
```
Watch **Executions** in n8n.

**Step C – real chat.** Message the business number:

1. `Hi` → greeting
2. `I need a plot in Washington. My budget is $50,000.` → up to five real listings
3. `2` (or `P001`) → asks for your name
4. `Michael Johnson` → asks for email → `michael@example.com`
5. `October 20 at 11 AM` (also works: `tomorrow 3pm`, `next Friday at 2:30 pm`, `20/10/2026 14:00`, or date and time in separate messages)
6. Confirmation arrives; check the new row in `Appointments`, both emails, and the state rows.

Also test: no matches (`plot in Atlantis`), over-budget, `MENU`, `CANCEL`, sending an image, an invalid email, a Sunday, a past date, booking the same slot twice, a sold property ID.

---

## 10. Customising

* **Add listings**: just add rows to `Properties`. Nothing in the workflow changes.
* **Ranking**: change `ranking` in Config.
* **Reply wording**: edit the templates in **Resolve Action**; AI-written parts are the intro sentence of search results and general answers.
* **Appointment status**: change `Pending` → `Confirmed`/`Cancelled` in the sheet; `Cancelled` frees the slot.
* **Bigger model**: set `ollamaModel` to `qwen2.5:7b` for better handling of messy messages. The workflow does not depend on the model for correctness; only for understanding free text.

---

## 11. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Meta says “callback URL couldn't be validated” | Workflow not **Active**, wrong path (`/webhook/whatsapp`, not `/webhook-test/`), `WEBHOOK_URL` not set to the tunnel, or verify token mismatch |
| Webhook runs but nothing happens | `phoneNumberId` in Config doesn't match Meta's, or the event was a delivery status (ignored on purpose) |
| Same test payload ignored | Duplicate protection; use a new `messages[0].id` |
| No WhatsApp reply, `Send WhatsApp` shows an error | Expired token, wrong Phone number ID, recipient not in the test allow-list, or >24 h since the customer's last message |
| `ECONNREFUSED` on Ollama nodes | n8n in Docker: use `http://host.docker.internal:11434` and `OLLAMA_HOST=0.0.0.0` |
| Slow first answer | Model loading on CPU; replies still work (Ollama nodes have 3–4 min timeouts and fallbacks) |
| “Sheet not found” / “Unable to parse range” | Tab names must match exactly: `Properties`, `Appointments`, `Customers`, `ConversationState` |
| New columns appear in a sheet | You renamed/removed a header. Restore the CSV headers |
| Google nodes fail after ~7 days | OAuth app still in *Testing*; publish it or reconnect |
| Dates read wrongly (03/04) | Set `dateOrder` correctly |
| “Today” is off by hours | Set `timezone` in Config (and `GENERIC_TIMEZONE` for n8n) |
| Phone `+` signs lost in sheet | Re-import with “Convert text…” unticked |

---

## 12. Security & production checklist

* Use a **permanent System User token**; never paste tokens into prompts or messages.
* Use a long random `verifyToken` and consider a random webhook path.
* **Webhook signature (X-Hub-Signature-256)** is *not* verified by this workflow: it needs the raw request body and Node's `crypto`, which n8n blocks in Code nodes by default. For hardening, either put a reverse proxy in front that validates the signature, or enable the Webhook “Raw Body” option and `NODE_FUNCTION_ALLOW_BUILTIN=crypto` and add a verification Code node.
* Back up the spreadsheet (File → Version history), and run n8n with a persistent volume.
* Set n8n execution-data pruning (`EXECUTIONS_DATA_PRUNE=true`) because executions contain customer messages.
* Tell customers how their data is used (WhatsApp/GDPR/NDPR consent) and restrict who can access the sheet.
* Run n8n and Ollama as services (Docker `--restart unless-stopped`, systemd) so they survive reboots.

## 13. Known limitations

* **Google Sheets is not transactional.** Two simultaneous bookings for the same slot could both pass the check. Fine for low volume; move to Postgres/Supabase for heavy use.
* Only the first message of a webhook batch is processed (Meta normally delivers one message per call).
* Only text messages are understood; other types get a polite “please type it” reply.
* Whole tabs are read on each message (fast for thousands of rows, not for hundreds of thousands).
* A real calendar for agents is not included; “availability” means no existing appointment for the same property/agent at that exact date and time.
* Local tunnels (quick URLs) change when restarted; use a permanent tunnel/domain in production.
* Verification note: the Code-node logic (parsing, state machine, booking, conflicts) was exercised offline with a simulated conversation. The first run against your real Google, Meta and Ollama accounts still needs the credential steps above, so work through section 9 before going live.
