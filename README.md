# email-bridge

A small FastAPI service that turns an ordinary email mailbox into a
[RapidPro](https://github.com/rapidpro/rapidpro) **External (EX) channel**, so
an AI assistant (or any flow) can converse over email exactly like it does
over WhatsApp or SMS.

```
inbound:   IMAP poll → strip quoted history → courier /receive → RapidPro flow
outbound:  RapidPro broadcast/reply → courier EX send_url → POST /send → SMTP
```

## Features

- **Outbound SMTP** (`POST /send`): courier's EX `send_url` target. Returns the
  channel's `mt_response_check` token so courier marks messages delivered.
  Supports STARTTLS and implicit TLS.
- **In-band subjects**: RapidPro can only carry a text field, so upstream
  senders may prepend `Subject: <line>\n` — the bridge lifts it into the SMTP
  subject and falls back to `EMAIL_SUBJECT` otherwise.
- **Inbound IMAP poll**: baseline at UIDNEXT on first run (an existing backlog
  is never ingested), persistent UID state across restarts, per-cycle flood
  cap, quoted-history stripping, optional sender allowlist
  (`EMAIL_ALLOWED_SENDERS`) to scope ingestion server-side.
- **Shared-secret auth** between courier and the bridge (`EMAIL_SEND_AUTH`).

## Configuration

Everything is environment-driven (see the `Config` section at the top of
`email_bridge.py`): `EMAIL_SMTP_*`, `EMAIL_IMAP_*`, `EMAIL_FROM`,
`EMAIL_COURIER_RECEIVE_URL`, `EMAIL_SEND_AUTH`, `EMAIL_MAX_BODY`,
`EMAIL_POLL_INTERVAL`, `EMAIL_ALLOWED_SENDERS`.

## Channel provisioning

`create_email_channel.py` get-or-creates the RapidPro EX channel
(scheme `mailto`) idempotently. Run it inside the RapidPro Django context:

```bash
cd /path/to/rapidpro && \
  EMAIL_BOT_ADDR=bot@example.com EMAIL_SEND_AUTH='<secret>' \
  ./.venv/bin/python manage.py shell \
    -c "exec(open('/path/to/email-bridge/create_email_channel.py').read())"
```

## Running

```bash
uv sync
uvicorn email_bridge:app --host 127.0.0.1 --port 8096
```

Typically deployed as a systemd service alongside RapidPro courier.
