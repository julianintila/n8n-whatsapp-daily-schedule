# WhatsApp Schedule Workflow (n8n + WAHA)

An n8n workflow that watches a WhatsApp group, picks out the daily schedule message posted by one specific person, and forwards it automatically, so the schedule no longer has to be copied and pasted every day.

## How it works

```
Webhook → If (right group + sender) → If1 (schedule keyword) → Edit Fields → Send a message (Gmail)
```

| Node | What it does |
|---|---|
| **Webhook** | Receives every incoming WhatsApp message from WAHA (`POST /webhook/whatsapp-schedule`) |
| **If** | Continues only if the message is from the target group **and** from the target sender |
| **If1** | Continues only if the message contains a schedule keyword (`schedule`, `sched`, `sked`, `schedules`), case-insensitive |
| **Edit Fields** | Builds `date`, `time_received` (Asia/Manila) and `message` |
| **Send a message** | Emails the schedule (subject: `Schedule for <date>`) |

Messages from other people or other chats, and messages without the keyword, are dropped.

## Requirements

- Docker (n8n and WAHA both run as containers)
- [n8n](https://n8n.io/)
- [WAHA](https://waha.devlike.pro/) (WhatsApp HTTP API), free Core version
- A WhatsApp account linked to WAHA that is a member of the target group
- A Gmail credential in n8n (only for the email step)

## Setup

1. **Run WAHA** and point its webhook at n8n:
   - `WHATSAPP_HOOK_URL=http://host.docker.internal:5678/webhook/whatsapp-schedule`
   - `WHATSAPP_HOOK_EVENTS=message`
2. **Link WhatsApp:** open the WAHA dashboard, start the `default` session, and scan the QR code (WhatsApp → Linked Devices).
3. **Import the workflow:** in n8n, create a new workflow, then **⋯ → Import from file** and choose `schedule-workflow.json`.
4. **Fill in the placeholders** in the imported workflow:

   | Placeholder | Where | Replace with |
   |---|---|---|
   | `YOUR_GROUP_ID@g.us` | `If` node | The group's ID |
   | `YOUR_BOSS_ID@lid` | `If` node | The sender's ID |
   | `YOUR_EMAIL@gmail.com` | Gmail node | The address to send to |

5. **Connect your Gmail credential** in the Gmail node.
6. **Publish** (or activate) the workflow.

## Finding the group and sender IDs

With the workflow active, post a test message in the group. In n8n, open **Executions**, click the newest run, and open the Webhook node's output:

- `body.payload.from` → the group ID (ends in `@g.us`)
- `body.payload.participant` → the sender ID (ends in `@lid` or `@c.us`)

You can also list groups from WAHA's Swagger page with `GET /api/default/groups`.

## Notes

- WAHA only fires the `message` event for messages **received from others**, not ones sent from the linked number itself.
- The PC running Docker, n8n and WAHA must be on when the schedule is posted, and the WAHA session must show **WORKING**.
- Change the keyword regex in `If1` to match how the schedule message starts (for example `^\s*TS SCHEDULE` to match only messages that begin with that phrase).
- WAHA is an unofficial WhatsApp bridge. Use it at your own risk and consider a spare number.

## Security

This repo contains only the workflow. Do **not** commit:

- WAHA API keys, dashboard or Swagger passwords
- The WAHA `sessions` folder (it holds the logged-in WhatsApp session)
- Real group IDs, sender IDs, phone numbers or email addresses
- Google credentials

## Possible next steps

- Replace the Gmail step with a WAHA `sendText` call to forward the schedule to a WhatsApp chat or group
- Append each schedule to a Google Sheet
- Parse the message into structured fields (date, time, personnel)
