# Trinetra

**Trinetra** (त्रिनेत्र — “three-eyed”) is a women’s safety workflow on [Kestra](https://kestra.io). If someone cannot use their phone, it still alerts trusted contacts with an **urgent message** and **live location**.

> **Prerequisites:** [Docker](https://docs.docker.com/get-docker/) and a free [Groq API key](https://console.groq.com/keys). Jump to [Quick start](#quick-start).

---

## Why I built this

My sister works **night shifts** and often travels home alone. In a stressful moment she may not be able to unlock her phone and tap SOS.

Trinetra starts help **automatically**: a signal triggers a workflow that alerts family, shares location, waits **60 seconds**, then **escalates** (sends a stronger reminder) if nobody confirms they are helping.

---

## How it works

One Kestra flow runs **7 steps**. A full run takes about **70 seconds** (mostly the 60s wait for family to respond).

| Step | What happens |
|------|----------------|
| 1 | Read who needs help, what happened, and where they are |
| 2 | AI writes an urgent message + Google Maps link |
| 3 | Send alert to trusted contacts (logged in the demo) |
| 4 | Wait 60 seconds for a response |
| 5 | Check if family acknowledged |
| 6 | If no response → send a second, stronger reminder |
| 7 | Log final status |

### Flow diagram

**Visual flowchart** (from Kestra Topology after a successful run):

![Trinetra pipeline flowchart in Kestra Topology view](docs/screenshots/01-flow-topology.png)


## Architecture Diagram

<p align="center">
  <img src="https://quickchart.io/graphviz?graph=digraph%20G%20%7B%0A%20%20rankdir%3DTB%3B%0A%20%20bgcolor%3Dtransparent%3B%0A%20%20node%20%5Bshape%3Dbox%2C%20style%3D%22rounded%2Cfilled%22%2C%20fontname%3DHelvetica%2C%20fontsize%3D12%2C%20fillcolor%3D%22%23F8FAFC%22%2C%20color%3D%22%23334155%22%2C%20penwidth%3D1.5%5D%3B%0A%20%20edge%20%5Bcolor%3D%22%2364758B%22%2C%20penwidth%3D1.4%2C%20arrowsize%3D0.8%5D%3B%0A%0A%20%20subgraph%20cluster_triggers%20%7B%0A%20%20%20%20label%3D%22Emergency%20Triggers%22%3B%0A%20%20%20%20labelloc%3Dt%3B%0A%20%20%20%20fontsize%3D13%3B%0A%20%20%20%20fontname%3DHelvetica%3B%0A%20%20%20%20color%3D%22%23CBD5E1%22%3B%0A%20%20%20%20style%3D%22rounded%22%3B%0A%20%20%20%20timeout%20%5Blabel%3D%22Timeout%5CnNo%20safe%20check-in%22%2C%20fillcolor%3D%22%23FEE2E2%22%5D%3B%0A%20%20%20%20voice%20%20%20%5Blabel%3D%22Voice%5Cn%27Help%20me%27%20detected%22%2C%20fillcolor%3D%22%23FEE2E2%22%5D%3B%0A%20%20%20%20sos%20%20%20%20%20%5Blabel%3D%22SOS%20Button%22%2C%20fillcolor%3D%22%23FEE2E2%22%5D%3B%0A%20%20%20%20watch%20%20%20%5Blabel%3D%22Smartwatch%5CnFall%20or%20abnormal%20HR%22%2C%20fillcolor%3D%22%23FEE2E2%22%5D%3B%0A%20%20%7D%0A%0A%20%20webhook%20%5Blabel%3D%22Kestra%20Webhook%20Trigger%22%2C%20fillcolor%3D%22%23DBEAFE%22%5D%3B%0A%20%20parse%20%20%20%5Blabel%3D%22Parse%20Emergency%20Payload%22%2C%20fillcolor%3D%22%23DBEAFE%22%5D%3B%0A%20%20claude%20%20%5Blabel%3D%22Claude%20Generates%5CnAI%20Alert%20Message%22%2C%20fillcolor%3D%22%23EDE9FE%22%5D%3B%0A%20%20notify%20%20%5Blabel%3D%22Notify%20Trusted%20Contacts%22%2C%20fillcolor%3D%22%23DCFCE7%22%5D%3B%0A%20%20wait60%20%20%5Blabel%3D%22Wait%2060%20Seconds%22%2C%20fillcolor%3D%22%23FEF3C7%22%5D%3B%0A%20%20ack%20%20%20%20%20%5Blabel%3D%22Acknowledged%3F%22%2C%20shape%3Ddiamond%2C%20fillcolor%3D%22%23FEF3C7%22%5D%3B%0A%20%20escalate%20%5Blabel%3D%22Escalate%20Alert%22%2C%20fillcolor%3D%22%23FECACA%22%5D%3B%0A%20%20summary%20%20%5Blabel%3D%22Final%20Summary%22%2C%20fillcolor%3D%22%23F3E8FF%22%5D%3B%0A%0A%20%20timeout%20-%3E%20webhook%3B%0A%20%20voice%20%20-%3E%20webhook%3B%0A%20%20sos%20%20%20%20-%3E%20webhook%3B%0A%20%20watch%20%20-%3E%20webhook%3B%0A%0A%20%20webhook%20-%3E%20parse%20-%3E%20claude%20-%3E%20notify%20-%3E%20wait60%20-%3E%20ack%3B%0A%20%20ack%20-%3E%20summary%20%5Blabel%3D%22Yes%22%5D%3B%0A%20%20ack%20-%3E%20escalate%20%5Blabel%3D%22No%22%5D%3B%0A%20%20escalate%20-%3E%20summary%3B%0A%7D" alt="Trinetra Architecture Diagram" width="900"/>
</p>


| Kestra task | Step |
|-------------|------|
| `parse_emergency` | 1 |
| `generate_alert_message` | 2 |
| `notify_family` | 3 |
| `wait_for_acknowledgement` | 4 |
| `check_acknowledgement_status` | 5 |
| `escalate_if_needed` | 6 (only if no response) |
| `send_summary` | 7 |

### Triggers

| Trigger | When it fires |
|---------|----------------|
| `timeout` | Safe arrival not confirmed in time |
| `voice` | Distress phrase detected |
| `sos` | SOS button pressed |
| `watch` | Fall or abnormal vitals on a wearable |

---

## Execution walkthrough

After [running the demo](#quick-start), open **Executions** → your run → follow steps 1–5 below.

| Step | Image | Where in Kestra |
|------|-------|-----------------|
| 1 | `01-flow-topology.png` | **Topology** |
| 2 | `02-gantt-success.png` | **Gantt** |
| 3 | `03-ai-agent-alert-log.png` | `generate_alert_message` → **Logs** |
| 4 | `04-notify-family-log.png` | `notify_family` → **Logs** |
| 5 | `05-escalation-and-summary-log.png` | `escalate_alert` / `send_summary` → **Logs** |

---

### Step 1 — Full pipeline

![Step 1 - Full Trinetra pipeline in Kestra Topology](docs/screenshots/01-flow-topology.png)

Every task from trigger to summary, with green checkmarks when successful. Read top to bottom: parse → AI alert → notify → wait → check → optional escalation → summary.

---

### Step 2 — Timing (Gantt)

![Step 2 - Gantt chart showing 60 second wait as longest step](docs/screenshots/02-gantt-success.png)

Shows how long each task took. The **60-second wait** should be the longest bar. AI generation is usually **10–15 seconds**. Other steps are quick.

---

### Step 3 — AI urgent alert

![Step 3 - AI agent log with urgent message and maps link](docs/screenshots/03-ai-agent-alert-log.png)

Groq writes the alert text. Look for **URGENT**, the trigger type (`voice`, `sos`, etc.), and a **Google Maps URL**. This message is reused in Steps 4 and 5 if needed.

Requires `GROQ_API_KEY` in Kestra KV Store (namespace `safety`).

---

### Step 4 — Notify family (first alert)

![Step 4 - Notify family log with alert and location](docs/screenshots/04-notify-family-log.png)

**First contact attempt:** family receives the urgent message and live location. In production, replace the log with SMS, push, or WhatsApp. The flow then waits 60 seconds.

---

### Step 5 — Escalation and summary

![Step 5 - Escalation and final summary logs](docs/screenshots/05-escalation-and-summary-log.png)

If nobody responded within 60 seconds, you see `ESCALATION — no acknowledgement within 60 seconds` (**second reminder**). The summary shows `Acknowledged: false` — **normal in a default demo run**.

---

## Stack

| Tool | Role |
|------|------|
| [Kestra](https://kestra.io) | Workflow engine (wait, branch, webhook) |
| Groq | AI alert text |
| Docker | Local Kestra + Postgres |
| Python | Tasks in `flows/trinetra.yml` |

---

## Quick start

**1. Start Kestra**

```bash
cd trinetra
docker compose up -d
```

Open **http://localhost:8080** — login: `admin@kestra.io` / `Admin1234!`

**2. Import flow** — namespace `safety`, id `emergency`, paste `flows/trinetra.yml` → Save

**3. API key** — **KV Store** (namespace `safety`) → add `GROQ_API_KEY` from [console.groq.com/keys](https://console.groq.com/keys)

**4. Run** — flow **safety / emergency** → **Execute** (defaults) → wait ~70s → check **Executions** against [screenshots](#screenshots-walkthrough)

**5. Webhook (optional)**

```bash
curl -X POST "http://localhost:8080/api/v1/main/executions/webhook/safety/emergency/emergency" \
  -H "Content-Type: application/json" \
  -u "admin@kestra.io:Admin1234!" \
  -d @sample-payloads/voice.json
```

---

## Sample payload

```json
{
  "name": "User",
  "destination": "Home",
  "trigger": "voice",
  "location": {
    "latitude": 12.9716,
    "longitude": 77.5946,
    "mapsUrl": "https://maps.google.com/?q=12.9716,77.5946"
  },
  "timestamp": "2026-05-14T23:15:00Z"
}
```

More samples: `sample-payloads/sos.json`, `timeout.json`, `watch.json`

---

## Configuration

| Item | Value |
|------|--------|
| Namespace | `safety` |
| Flow ID | `emergency` |
| Webhook | `http://localhost:8080/api/v1/main/executions/webhook/safety/emergency/emergency` |
| Required KV | `GROQ_API_KEY` |
| Optional KV | `TRINETRA_SIMULATE_ACK` = `true` |

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Mermaid error in preview | Use the **Topology screenshot** above; the text `graph TD` diagram is a backup |
| Images not in README preview | Open `trinetra/README.md`; enable `markdown.preview.securityLevel: allowLocalImages` |
| Port 8080 down | `docker compose down -v && docker compose up -d` |
| AI task fails | Add `GROQ_API_KEY`; keep Docker running |
| Always see escalation | Expected in demo; set `TRINETRA_SIMULATE_ACK=true` to test skip path |

---

## License

MIT
