# 🎫 AI Customer Complaint Triage: n8n + Google Gemini

A no-code / low-code **AI automation workflow** built in **n8n**. A customer complaint comes in through a webhook. **Google Gemini 3.1 Flash Lite** classifies it by category, priority and reason, and writes a reply. High-priority issues are escalated automatically, and the caller gets a structured JSON response.

![n8n](https://img.shields.io/badge/n8n-Workflow%20Automation-EA4B71?logo=n8n&logoColor=white)
![Gemini](https://img.shields.io/badge/Google-Gemini%203.1%20Flash%20Lite-4285F4?logo=google&logoColor=white)
![LangChain](https://img.shields.io/badge/n8n-Basic%20LLM%20Chain-1C3C3C)
![Webhook](https://img.shields.io/badge/Trigger-Webhook%20POST-orange)
![Postman](https://img.shields.io/badge/Tested%20with-Postman-FF6C37?logo=postman&logoColor=white)

---

## 📌 Overview

Support teams spend a lot of time reading every ticket just to decide **how urgent it is and who should handle it**. This workflow automates that first triage step:

| Step | What happens |
|---|---|
| 1️⃣ Receive | A `POST` request with the customer's name and complaint hits an n8n webhook |
| 2️⃣ Classify | Gemini reads the complaint and returns **Category, Priority, Reason, Customer Response** |
| 3️⃣ Decide | An **If** node checks whether the priority is **High** |
| 4️⃣ Escalate | High-priority tickets get `status: ESCALATED` and are routed to the Payment Support Team |
| 5️⃣ Respond | A JSON reply goes back to the caller in the same HTTP request |

---

## 🖼️ Workflow

![Workflow canvas](screenshots/01_workflow_canvas.png)

```
┌──────────┐   POST   ┌──────────────────┐        ┌──────┐  true  ┌─────────────┐      ┌─────────────────────┐
│ Webhook  │ ───────► │ Basic LLM Chain  │ ─────► │  If  │ ─────► │ Edit Fields │ ───► │ Respond to Webhook  │
└──────────┘          └────────┬─────────┘        └──┬───┘        └─────────────┘      └─────────────────────┘
                               │ model               │ false
                      ┌────────▼─────────┐           ▼
                      │ Google Gemini    │      (not connected yet)
                      │ Chat Model       │
                      └──────────────────┘
```

---

## 🧩 Node-by-Node Breakdown

### 1. Webhook (Trigger)
![Webhook](screenshots/02_webhook_trigger.png)

- **HTTP Method:** `POST`
- **Authentication:** None
- **Respond:** *Using 'Respond to Webhook' Node*. The HTTP connection stays open until the last node sends the answer.
- Incoming data is available as `$json.body` (`customer`, `message`).

### 2. Basic LLM Chain
![Basic LLM Chain](screenshots/03_basic_llm_chain.png)

Prompt (User Message):

```
classify this customer complaint:

{{ $json.body.message }}

return ONLY these 4 fields:
Category:
Priority:
Reason:
Customer Response:

Do not provide any additional explanation
```

The `{{ $json.body.message }}` expression puts the complaint text from the webhook into the prompt.

### 3. Google Gemini Chat Model
![Gemini Chat Model](screenshots/04_gemini_chat_model.png)

- **Model:** `gemini-3.1-flash-lite`, which is fast and cheap and fits a classification task
- **Credential:** Google Gemini (PaLM) API account, stored in n8n Credentials and not in the workflow
- **Token usage for the test run:** 53 prompt + 56 completion = **109 tokens**

### 4. If: Priority Check
![If node](screenshots/05_if_priority_check.png)

- **Condition:** `{{ $json.text }}` **contains** `Priority: High`
- **True branch** → escalate
- **False branch** → not connected yet (see *Limitations*)

### 5. Edit Fields (Set)
![Edit Fields](screenshots/06_edit_fields.png)

Manual mapping that builds a clean ticket object:

| Field | Value |
|---|---|
| `status` | `ESCALATED` |
| `priority` | `HIGH` |
| `action` | `Sent to Payment Support Team` |

### 6. Respond to Webhook
![Respond to Webhook](screenshots/07_respond_to_webhook.png)

Returns JSON to the caller:

```json
{
  "status": "ESCALATED",
  "priority": "HIGH",
  "action": "Sent to Payment Team",
  "message": "Your payment issue has been escalated to our payment support team."
}
```

---

## 🧪 Example Run

**Request** (sent from Postman):

```bash
curl -X POST "https://<your-instance>.app.n8n.cloud/webhook-test/<path>" \
  -H "Content-Type: application/json" \
  -d @sample_request.json
```

```json
{
  "customer": "Rahul",
  "message": "My UPI payment failed but 2000 rupees was debited from my account."
}
```

**Gemini classification:**

```
Category: Payment/Transaction Issue
Priority: High
Reason: Transaction failure with amount deduction
Customer Response: Apologies for the inconvenience. We have noted your transaction
details and our team is investigating the failed debit. Please provide the
Transaction ID so we can expedite your refund process.
```

**Result:** If → **True branch** → ticket escalated → JSON response returned to Postman ✅

---

## 📂 Project Structure

```
n8n-AI-Complaint-Triage-Gemini/
├── workflow.json            # Exported n8n workflow (import this)
├── sample_request.json      # Test payload for Postman / curl
├── screenshots/             # Canvas + every node's config and output
├── .env.example             # Only needed for self-hosted n8n
├── .gitignore
└── README.md
```

---

## 🚀 How to Run It

### 1. Import the workflow
1. Open n8n (Cloud or self-hosted)
2. **Workflows → Import from File** → choose `workflow.json`

### 2. Add your Gemini credential
1. Get a key from [Google AI Studio](https://aistudio.google.com/app/apikey)
2. In n8n: **Credentials → New → Google Gemini (PaLM) API** → paste the key
3. Open the **Google Gemini Chat Model** node and select that credential

> 🔐 Exported n8n workflows include only a credential *reference*, never the key itself, so `workflow.json` is safe to push.

### 3. Test
1. Open the **Webhook** node → **Listen for test event**
2. Send `sample_request.json` to the **Test URL** using Postman or curl
3. Watch every node turn green ✅

### 4. Go live
**Publish / activate** the workflow and switch your client to the **Production URL** (`/webhook/...` instead of `/webhook-test/...`).

### Self-hosting (optional)

```bash
docker run -it --rm -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

---

## 🧠 Key Takeaways

- **LLMs can act as routers.** One prompt turns free text into structured, actionable fields.
- **Expressions connect nodes.** `{{ $json.body.message }}` and `{{ $json.text }}` pass data through the workflow without writing code.
- **Synchronous webhooks.** *Respond to Webhook* lets the workflow behave like a real API endpoint.
- **Use small models for small jobs.** Flash Lite classified the ticket for only 109 tokens.
- **Keep credentials separate.** API keys live in n8n Credentials, not inside the workflow logic.

---

## ⚠️ Current Limitations

- **The False branch is empty.** Medium/Low complaints never reach *Respond to Webhook*, so the caller does not get the proper JSON answer.
- **Text matching is brittle.** `contains "Priority: High"` is case-sensitive, so `Priority: high` or `**Priority:** High` would go to the False branch.
- **Responses are hardcoded.** *Edit Fields* and *Respond to Webhook* use fixed values, so every High ticket says "Payment Team", even non-payment issues.
- **`action` text doesn't match:** *Edit Fields* says `Sent to Payment Support Team`, while the response body says `Sent to Payment Team`.
- **The Gemini customer reply is not returned.** The `Customer Response` the LLM wrote is never sent back to the caller.
- **The webhook has no authentication.** Anyone with the URL can call it.

---

## 🔮 Possible Extensions

- [ ] Turn on **Require Specific Output Format** with a Structured Output Parser so Gemini returns JSON (`{category, priority, reason, reply}`)
- [ ] Replace **If** with a **Switch** node: High → escalate, Medium → queue, Low → auto-reply
- [ ] Build the response with expressions such as `{{ $json.priority }}` instead of fixed text
- [ ] Return Gemini's `Customer Response` to the customer
- [ ] Log every ticket to **Google Sheets / Airtable / Postgres**
- [ ] Notify the support team on **Slack / Gmail** for escalations
- [ ] Add **Header Auth** to the webhook
- [ ] Add an **Enable Fallback Model** option for reliability

---

## 🛠️ Tech Stack

| Tool | Role |
|---|---|
| [n8n](https://n8n.io/) | Workflow automation / orchestration |
| Google Gemini 3.1 Flash Lite | Complaint classification + reply generation |
| n8n Basic LLM Chain | LangChain-based prompt → model wrapper |
| Postman | API testing |

---

## 👤 Author

**Kailas Wadje**
MSc Data Science & AI, University of Liverpool

- GitHub: [@Kailaswadje](https://github.com/Kailaswadje)
- LinkedIn: [linkedin.com/in/kwadaje](https://www.linkedin.com/in/kwadaje/)

---

⭐ If this workflow saved you some triage time, consider giving it a star!
