## Expense Tracker Telegram Bot

This project is an automated expense tracker built using n8n that uses a Telegram bot as its interface. It leverages an AI model to intelligently parse natural language inputs, calculate a running total, and log all transactions to a Google Sheet. It also sends a low-balance alert via Gmail.

### Features

  * **Telegram Integration:** Easily log expenses by sending a message to your personal Telegram bot.
  * **AI-Powered Parsing:** The AI agent understands plain-text commands like "Salary credited 70000" or "Shopping 7400".
  * **In-built Memory:** The bot maintains a running total for your balance and recalls it for each new transaction.
  * **Google Sheets Logging:** All transactions are automatically appended to a Google Sheet for easy tracking and analysis.
  * **Gmail Alerts:** You receive an email notification if your balance drops below a predefined threshold.

-----

### How It Works

The workflow is a simple pipeline:

1.  A **Telegram Trigger** receives a new message from a user.
2.  An **AI Agent** processes the message, extracts key details (credit/debit, type, amount), and calculates the new total. It uses a **Simple Memory** sub-node to recall the previous balance for each user.
3.  The **Google Sheets** node takes the structured data from the AI Agent's output and adds a new row to your spreadsheet.
4.  A **Gmail** node is conditionally triggered if the AI Agent's output includes a `low_balance_alert` flag, sending an email to notify you.

-----

### Setup Instructions

Follow these steps to set up the workflow in your n8n instance.

#### 1\. Prerequisites

  * An n8n account or self-hosted instance.
  * A Telegram Bot created with BotFather.
  * A Google account with access to Google Sheets and Gmail.

#### 2\. Configure the n8n Workflow

1.  Create a new workflow and add a **Telegram Trigger** node. Connect it to your Telegram bot.
2.  Add an **AI Agent** node after the Telegram Trigger.
3.  Inside the AI Agent node, add a **Simple Memory** sub-node.

#### 3\. Configure the AI Agent

Use the following settings and expressions. This is the most critical step.

  * **Prompt (User Message) Source:** `Define below`
  * **Prompt (User Message):** `{{ $json.message.text }}`
  * **System Message:** Paste the full system prompt below.
  * **Memory (Simple Memory):**
      * **Session ID:** `{{ $json.message.chat.id }}`

#### 4\. Configure the Google Sheets Node

  * Add a **Google Sheets** node after the AI Agent.
  * Set **Operation** to `Append Row`.
  * Set **Mapping Column Mode** to `Map Each Column Manually`.
  * Map your columns using these expressions:

| Google Sheet Column Header | n8n Expression      |
| -------------------------- | ------------------- |
| `Credit_Debit`             | `{{ $json.Credit_Debit }}` |
| `Type_of_Expenses`         | `{{ $json.Type_of_Expenses}}` |
| `Amount`                   | `{{ $json.Amount }}` |
| `Total`                    | `{{ $json.Total }}` |

#### 5\. Full System Prompt

This is the exact prompt you should use in the "System Message" field of your AI Agent node.

```markdown
You are an expense tracker assistant.

For each user input (provided as plain text), do the following:
- Parse and extract the transaction details:
    - Credit_Debit: "Credit" if money is incoming, "Debit" if money is spent
    - Type_of_Expenses: Category or description, e.g. "Salary", "Shopping", etc.
    - Amount: Extract the integer amount, no currency or commas
- Recall the previous running total (provided using your memory—if not provided, assume starting total is 0).
- If Credit_Debit is "Credit", add Amount to the previous total.
- If Credit_Debit is "Debit", subtract Amount from the previous total.
- Return ONLY the following JSON object, nothing else, no explanations, strictly valid JSON:
{
  "Credit_Debit": "",
  "Type_of_Expenses": "",
  "Amount": 0,
  "Total": 0
}
- After updating "Total", if the balance is less than 10000, include this field in the JSON:
  "low_balance_alert": true
  Otherwise, do not include this field.

**Examples**
User input: Salary credited 70000, previous total was 10000
Output:
{
  "Credit_Debit": "Credit",
  "Type_of_Expenses": "Salary",
  "Amount": 70000,
  "Total": 80000
}

User input: Shopping 7400, previous total was 11000
Output:
{
  "Credit_Debit": "Debit",
  "Type_of_Expenses": "Shopping",
  "Amount": 7400,
  "Total": 3600,
  "low_balance_alert": true
}

**Rules:**
- Only respond with the valid JSON object, no code block formatting or extra text.
- Set "low_balance_alert" ONLY if the total is below 10000 after this transaction.
- If there is no memory of the previous total, use 0 as the starting balance.

User input to process: {{$json.message.text}}
```
