# LinkedIn Post Generator (n8n Workflow)

An automated human-in-the-loop content pipeline built on n8n. This workflow collects post topics through an interactive n8n Form trigger, generates an initial draft using a Writer AI Agent, refines tone and structure with a Reviewer AI Agent, requests approval via an interactive email using Gmail (pausing execution until a response is received), and appends approved posts into Google Sheets.

---

## 📌 Features

- **Interactive Form Trigger:** Allows on-demand topic submission via a hosted form.
- **Two-Stage AI Pipeline:**
  - **Writer Agent:** Drafts engaging post copy tailored for LinkedIn.
  - **Reviewer Agent:** Polishes the copy for maximum engagement, formatting, and clear calls to action.
- **Human-in-the-Loop Approval:** Utilizes n8n's **Gmail (Send and Wait for Response)** operation to pause execution until you click **Approve** or **Reject**.
- **Conditional Branching:** Evaluates approval decisions using an **If** node.
- **Automated Logging:** Appends approved posts directly into Google Sheets with structured metadata (`Date`, `Topic`, `Final Post`, `Status`).

---

## 🛠 Workflow Architecture

```text
[ On form submission ] ➔ [ Writer Agent ] ➔ [ Reviewer Agent ] ➔ [ Send message and wait for response ]
                                │                   │                                 │
                         [ Chat Model ]      [ Chat Model ]                      [ If Node ]
                                                                                      │
                                                                           (TRUE) ────┴──── (FALSE)
                                                                             │                │
                                                                    [ Google Sheets ]      [ Stop ]
```

---

## 🚀 Step-by-Step Setup Guide

### Step 1: Create and Test the Form Trigger

1. Add an **n8n Form Trigger** node to your workflow.
2. **Node Name:** `On form submission`
3. **Form Customization:**
   - **Form Title:** `LinkedIn Post Generator`
   - **Form Elements:** Add a required **Text** field named `Text` (or `Topic`).
4. **Test Run:**
   - Click **Test Step** and open the form URL.
   - Enter a test topic (e.g., `About AI Agent`) and submit the form.
   - Verify that output fields `Text`, `submittedAt`, and `formMode` appear in the node preview.

---

### Step 2: Configure the Dual AI Pipeline

#### 1. Writer Agent Node
- **Node Name:** `Writer Agent`
- **Node Type:** `AI Agent`
- **Prompt Source:** `Define below`
- **Prompt:**
  ```text
  Draft an engaging, high-performing LinkedIn post about the following topic:
  {{ $json.Text }}

  Keep it professional, well-spaced, and under 250 words.
  ```
- **Connected Sub-Node:** Attach an **OpenAI Chat Model** node to the `Chat Model` input port (e.g., `gpt-4o-mini`).

#### 2. Reviewer Agent Node
- **Node Name:** `Reviewer Agent`
- **Node Type:** `AI Agent`
- **Prompt Source:** `Define below`
- **Prompt:**
  ```text
  You are an expert LinkedIn content editor. Review and improve the following post draft:

  {{ $json.output }}

  Optimize formatting, line spacing, hook impact, and CTA. Return ONLY the finalized post copy without introductory or closing commentary.
  ```
- **Connected Sub-Node:** Attach a second **OpenAI Chat Model** node to its `Chat Model` input port.

---

### Step 3: Configure Approval Email & Branching

#### 1. Gmail Node (Send and Wait for Response)
- **Node Type:** `Gmail`
- **Operation:** `Send message and wait for response`
- **To:** Your email address
- **Subject Expression:**
  ```text
  Approval Needed: LinkedIn Post on "{{ $('On form submission').item.json.Text }}"
  ```
- **Message Body Expression:**
  ```text
  Here is the finalized post draft for review:

  --------------------------------------------------
  {{ $json.output }}
  --------------------------------------------------

  Please click one of the buttons below to process this post.
  ```
- **Response Options:** Configure option values for `Approve` and `Reject`.

#### 2. IF Node (Conditional Routing)
- **Node Type:** `If`
- **Condition:**
  - **Value 1:** `{{ $json.approved }}` (or `{{ $json.response }}`)
  - **Operator:** `Equal` / `Boolean Is True`
  - **Value 2:** `true` (or `Approve`)

---

### Step 4: Configure Google Sheets Integration

1. Create a Google Sheet named **`LinkedIn Content Tracker`**.
2. Add the following exact headers in **Row 1**:

| Date | Topic | Final Post | Status |
| :--- | :--- | :--- | :--- |

3. Open your Google Cloud Console and enable the **Google Sheets API** on your project.
4. Add a **Google Sheets** node in n8n on the **TRUE** output branch of the `If` node.
5. **Node Settings:**
   - **Operation:** `Append Row`
   - **Document:** Select `LinkedIn Content Tracker` by ID or file path.
   - **Sheet:** Select `Sheet1`.
   - **Column Mapping:** Set mode to **Map Each Column Manually**:
     - **`Date`**: `{{ $now.format('YYYY-MM-DD') }}`
     - **`Topic`**: `{{ $('On form submission').item.json.Text }}`
     - **`Final Post`**: `{{ $('Reviewer Agent').item.json.output }}`
     - **`Status`**: `Approved`

---

## 🛠 Common Errors & Troubleshooting

### 1. Error: `[Referenced node doesn't exist]` in Gmail Node
* **Cause:** Referencing `$('n8n Form Trigger')` when the actual canvas trigger node is named `$('On form submission')`.
* **Fix:** Use the expression `{{ $('On form submission').item.json.Text }}`.

### 2. Workflow runs but Google Sheet does not update
* **Check IF Node:** Open execution history and verify if the IF node evaluated to `true`. If it routed to `false`, adjust the string or boolean comparison to match the exact output of the Gmail wait action.
* **Check Column Headers:** Ensure Row 1 of your Google Sheet has exact headers matching your mapping (`Date`, `Topic`, `Final Post`, `Status`).
* **Check Sheet Range:** Ensure no trailing blank rows exist between Row 1 and Row 100+, as data may append out of immediate view.

---

## 📧 Execution Output Example

```text
2026-10-28 | About AI Agent | Struggling to create LinkedIn posts that truly engage... | Approved
```

## 📝 Save Workflow

Save your completed workflow in n8n under the name **`LinkedIn Post Approval`**.
