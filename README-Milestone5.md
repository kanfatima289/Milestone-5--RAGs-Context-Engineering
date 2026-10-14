# Review Intelligence + Fee Explainer (n8n, with MCP approval gating)

An AI workflow for **Groww** that turns raw Play Store reviews into:

1. A **weekly internal product pulse** (max 250 words)
2. A **standard customer-facing fee explanation** (max 6 bullets, 2 official links, "Last checked" date)

After a human approves, the outputs are logged to a Google Doc and saved as an unsent Gmail draft through **MCP (Model Context Protocol)**. Nothing is ever auto-sent.

## Files in this submission

| File | What it is |
|---|---|
| `README.md` | This file: how to run, where approval happens, and the fee issue found |
| `Review_Intelligence___Fee_Explainer.json` | **Workflow A**, the main n8n workflow and MCP client (import this into n8n) |
| `MCP_Server-_Docs_and_Draft.json` | **Workflow B**, the n8n MCP server exposing the append-to-doc and create-draft tools (import this into n8n) |
| `reviews_last_12_weeks.csv` | Input data: the Groww Play Store reviews from the last 12 weeks, uploaded through the form in Workflow A |
| `MCP_Client_Workflow_Screenshot.png` | Screenshot of Workflow A's canvas, showing the approval step and the `Log via MCP` agent connected to the MCP client tool |
| `MCP_Server_Docs_and_Draft_Workflow_Screenshot.png` | Screenshot of Workflow B's canvas: the MCP Server Trigger with its `append_to_doc` and `create_email_draft` tools |
| `Weekly_Pdt_Pulse-_Milestone_5.pdf` | Sample output: the Google Doc log entry from a real run (2026-10-13), with the pulse, fee issue, explanation and sources |
| `Draft_screenshot-_Milestone_5.png` | Screenshot of the resulting unsent Gmail draft "Weekly Product Pulse + Customer Clarification", proving the MCP draft action worked |

## Architecture

```
WORKFLOW A (main)                                  WORKFLOW B (MCP server)
Form upload (CSV)                                  MCP Server Trigger
 -> Extract from File                                |- append_to_doc        (Google Docs)
 -> Clean and filter                                 `- create_email_draft   (Gmail draft)
 -> Review Intelligence (LLM 1)                              ^
 -> Parse and Validate                                       |
 -> Weekly Product Pulse (LLM 2)                             |
 -> Pulse Word Check                                         |
 -> Fee Explainer (LLM 3)                                    |
 -> Assemble Outputs                                         |
 -> Request Approval  (Gmail, send and wait)                 |
 -> Approved?                                                |
      |- true  -> Log via MCP (AI Agent + MCP Client) -------'
      `- false -> Rejected - Nothing Logged
```

Workflow A never writes to Google Docs or Gmail itself. It asks Workflow B to do it over MCP, and Workflow B exposes **only** an append-to-doc tool and a create-draft tool. There is no send-email tool, so sending is impossible by design.

---

## How to run

### 1. Prerequisites

- An n8n instance (n8n Cloud or self-hosted)
- An OpenAI-compatible credential for the chat models (the workflow uses `gpt-5-mini`)
- Google OAuth2 credentials for **Google Docs** and **Gmail**
- A reviews CSV with the columns `date`, `rating`, `review_text` (ideally the last 8 to 12 weeks)
- A Google Doc to use as the log

### 2. Import both workflows

In n8n: **Workflows -> Create workflow -> menu (...) -> Import from File**. Import both JSON files.

### 3. Set up Workflow B first (the MCP server)

1. Open `MCP Server- Docs and Draft`.
2. Reconnect credentials on the two tool nodes:
   - `append_to_doc`: Google Docs credential, and replace the **document URL** with your own Google Doc.
   - `create_email_draft`: Gmail credential, and set your own **recipient address** under Options.
3. Save and switch the workflow to **Active** (top right). The MCP production URL only works while it is active.
4. Open the **MCP Server Trigger** node and copy its **Production URL**.

### 4. Set up Workflow A (the main workflow)

1. Open `Review Intelligence + Fee Explainer`.
2. Reconnect credentials on the three OpenAI chat-model nodes and the **Request Approval** Gmail node.
3. In **Request Approval**, set `sendTo` to the address that should receive the approval email.
4. In the **docs_and_draft_mcp** node, replace the **Endpoint URL** with the Production URL you copied from Workflow B.
5. Save.

### 5. Run it

1. Make sure Workflow B is **Active**.
2. In Workflow A, click **Execute workflow**. The form page opens ("Upload Reviews CSV"). Upload your CSV.
3. The workflow analyses reviews, writes the pulse and the fee explainer, then **pauses** and emails you for approval (see below).
4. Click **Approve** in the email. The workflow resumes and logs via MCP.
5. Check results:
   - A new entry appears at the end of your Google Doc.
   - An **unsent** draft appears in Gmail Drafts with subject "Weekly Product Pulse + Customer Clarification - <date>".
6. To test the reject path, run again and click **Reject**. Nothing is written and no draft is created.

### What the workflow enforces

| Requirement | How |
|---|---|
| Review window | Last 12 weeks, anchored to the newest review in the CSV (so older datasets still work) |
| Volume cap | First 200 qualifying reviews are analysed, to stay within token limits |
| Max 5 themes, top 3, 3 quotes | LLM 1 returns structured JSON, then `Parse and Validate` checks it |
| Pulse <= 250 words | `Pulse Word Check` counts words in code and trims whole lines from the end if over the limit |
| Fee explainer <= 6 bullets | `Assemble Outputs` throws an error if the model returns more than 6 |
| 2 official links only | Links are filtered against a fixed allow-list in `Assemble Outputs`, so a made-up link cannot get through |
| "Last checked" date | Generated by n8n (`$now`, Asia/Kolkata), not guessed by the model |

---

## Where MCP approval happens

Approval happens in Workflow A, in two nodes, **before** any MCP call is made.

1. **`Request Approval`** (Gmail node, operation **Send and Wait for Response**)
   - Emails the full preview (pulse, fee explanation, sources) to the reviewer.
   - Uses a **double approval** with two buttons: **Approve** and **Reject**.
   - The workflow is **paused** here until the reviewer clicks. The wait limit is **24 hours**.
2. **`Approved?`** (IF node)
   - Checks `{{ $json.data.approved }}` is `true`.
   - **True** goes to `Log via MCP`.
   - **False** goes to `Rejected - Nothing Logged`. This also catches a timeout with no response, so no response means no action.

Only after the **true** branch does the MCP part run:

3. **`Log via MCP`** (AI Agent) with the **`docs_and_draft_mcp`** MCP Client Tool attached.
   - The agent is told to call exactly two tools, once each, with the content copied verbatim:
     1. `append_to_doc` appends the log entry (date, top themes, weekly pulse, fee issue, explanation bullets, source links) to the Google Doc.
     2. `create_email_draft` creates the Gmail **draft** (HTML body: weekly pulse plus the fee explanation as a reusable support snippet).
   - These tools live on Workflow B's MCP Server Trigger, which is why Workflow B must be Active.

**Safety:** there is no send tool on the MCP server, so the agent can only log and draft. A rejected or unanswered approval never reaches the MCP step.

---

## What fee issue was identified from reviews

**Product:** Groww (Play Store reviews)

**Fee issue identified:** **Hidden / external charges** (brokerage, service charge, and MTF haircut)

**What users misunderstand** (from the logged run on 2026-10-13): Users believe Groww is deducting unexplained or excessive "hidden" or "external" fees, such as service charges, high brokerage, and the haircut on leveraged stocks. They see these as arbitrary and not clearly shown or justified in the UI.

**Top themes from the same run:**

1. Performance and Stability
2. Customer Support and Account Issues
3. Fees and Charges

**Supporting review evidence (verbatim quote from the pulse):**

> "user friendly.. But Brokerage charges are too high. If monthly package means could be better for F&O" - Play Store user

**Key observation:** Fee confusion is acute. Users report unexplained "hidden" or external charges and cannot find a clear breakdown in the UI or in confirmations.

**Customer-facing explanation generated (summary of the support snippet):**

- "Hidden" or "external" charges are a label for third-party and statutory levies (exchange and clearing fees, stamp duty, GST, DP charges), not an extra created by the app.
- Brokerage and platform fees vary by product, segment and order type, and are published on Groww's pricing page and shown in each trade's contract note.
- An MTF haircut is a risk-based reduction in the value of collateral securities. It lowers usable margin and is not a separate cash fee charged to the user's P&L.
- Statutory charges appear separately in trade confirmations and account statements.
- The contract note and transaction history give a line-by-line breakdown of platform charges versus external levies.

**Official sources used:**

- https://groww.in/pricing
- https://www.sebi.gov.in/sebi_data/faqfiles/sep-2024/1727242783639.pdf

The full output of this run, including the "Last checked" date, is in `Weekly_Pdt_Pulse-_Milestone_5.pdf`.

---

## Limitations

- Only the first 200 reviews in the window are analysed (token limits). Very large CSVs would need batching.
- LLM output varies between runs. Themes, quotes, and the chosen fee can change slightly.
- Official links are restricted to a fixed allow-list in the workflow. The explanation is only as accurate as the model's use of those sources, so a human should verify the links before wider use. This is the purpose of the approval step.
- The MCP Server Trigger in the shared JSON has no authentication configured. For anything beyond a demo, add Bearer authentication on the server and the matching credential on the MCP Client Tool.
- Credentials are not included in the exported JSON files. They must be reconnected after import.
