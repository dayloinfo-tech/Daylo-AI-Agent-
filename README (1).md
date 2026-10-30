# Daylo AI: n8n Workflow

Daylo is an adaptive AI workday planner. You talk to "Ashley", its assistant, to save tasks and commitments, ask what is most urgent, plan your day, or move a task to a new time. This repository contains the "n8n workflow" that powers Ashley. It runs on "Nebius Token Factory" with "NVIDIA Nemotron" open models.

- Live app: https://dayloai.netlify.app
- Track: Best Apps and Agents

## What it does

| You say | Ashley does |
|---|---|
| "Send the proposal to Mr. Okafor by Friday." | Saves a task with the person and due date |
| "I promised Amaka a budget review next Tuesday." | Saves a commitment |
| "What should I work on next?" | Summarizes what is most urgent and why |
| "Plan my day." | Places open tasks into real time slots inside work hours |
| "Move the client reply to tomorrow morning." | Finds a new free slot and updates that task |
| Anything else | Friendly chat reply |

## Models (via Nebius Token Factory)

| Model | Model ID | Used for |
|---|---|---|
| NVIDIA Nemotron 3 Nano 30B A3B | `nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B` | Intent classification, task extraction, schedule drafting, general chat |
| NVIDIA Nemotron 3 Super 120B A12B | `nvidia/nemotron-3-super-120b-a12b` | Urgency summary ("what is most urgent and why") |

- API: Nebius Token Factory OpenAI-compatible API, `POST https://api.tokenfactory.nebius.com/v1/chat/completions`
- Public serverless endpoints, billed per token. No GPUs to manage.
- No fine-tuning. Prompts return strict labels or JSON, reasoning is turned off for speed, and temperatures are low (0 to 0.6).

## How it works

```
Webhook (POST /founder-assistant)
  -> Verify Firebase ID token (reject with 401 if invalid)
  -> Nemotron Nano: classify intent
  -> Switch by intent
       store_task / store_commitment -> Nano extracts fields -> save to Firestore
       query_status                  -> load open tasks -> Nemotron Super summarizes
       plan_day / reschedule_task    -> load open tasks -> Nano drafts schedule
                                        -> code validates slots -> save times to Firestore
       general_chat                  -> Nano replies
  -> Respond to webhook with { "reply": "..." }
```

"Safety net for scheduling." The model proposes time slots, then a Code node checks each one. It replaces any slot that overlaps another, starts in the past, or falls outside work hours, and only then saves the result. Default work hours are 09:00 to 18:00, with a 09:00 to 11:00 peak window and a 1.5x duration multiplier.

"Security" The user id (`founder_id`) always comes from the verified Firebase token, never from the request body.

## Setup

1. "Import" `workflow/daylo-ai.json` into n8n (Workflows > Import from file).
2. "Nebius credential." Create a "Header Auth" credential: Name `Authorization`, Value `Bearer <YOUR_NEBIUS_API_KEY>`. Select it on the 5 Nemotron HTTP Request nodes. Get a key from [Nebius Token Factory](https://tokenfactory.nebius.com/).
3. "Firebase."
   - Create a Google "Service Account" credential for your Firebase project and select it on the 3 Firestore nodes.
   - Replace `YOUR_FIREBASE_PROJECT_ID` in the Firestore nodes.
   - Replace `YOUR_FIREBASE_WEB_API_KEY` in the "Verify Firebase Token" node URL.
4. "Activate" the workflow. Your webhook URL is `https://<your-n8n-host>/webhook/ai-assistant`.
5. Point your front end at that URL.

## Webhook contract

```
POST /webhook/ai-assistant
Authorization: Bearer <Firebase ID token>
Content-Type: application/json

{ "message": "Plan my day", "timezone": "Africa/Lagos" }
```

Response: `{ "reply": "Here's your plan: ..." }`. An invalid token returns HTTP 401.

Quick test:

```bash
curl -X POST "https://<your-n8n-host>/webhook/ai-assistant" \
  -H "Authorization: Bearer <FIREBASE_ID_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"message":"Send the proposal to Mr. Okafor by Friday","timezone":"Africa/Lagos"}'
```

## Firestore data

Collection `tasks`, one document per task: `uid`, `content`, `linked_person`, `due_date`, `category` (`task` or `commitment`), `status`, `duration_hours`, `priority`, `missed_count`, `created_at`, `scheduled_start`, `scheduled_end`, `updated_at`.

Use Firestore security rules so users can only read and write documents where `uid` matches their own account.

## Repository layout

```
.
├── workflow/daylo-ai.json   # n8n workflow (credentials and keys removed)
├── README.md
└── LICENSE
```

## License

MIT. See [LICENSE](LICENSE).
