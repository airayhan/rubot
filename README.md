 # 🩸 Blood Donor Approval System - n8n Workflow

## 🚀 Overview
This repository contains a fully automated n8n workflow designed to collect, validate, and manage blood donor submissions. It acts as a complete approval system connecting Google Sheets, Supabase, and Telegram.

## ⚙️ How It Works
The workflow is divided into two main logical parts:

**1. Data Collection & Notification (Flow 1):**
* **Trigger:** Automatically fetches new form responses from Google Sheets.
* **Validation:** Uses an IF node to ensure all required conditions are met (e.g., Department is exactly "IER", phone number format is valid 11-digits, and consent is given). Invalid data is ignored.
* **Database Entry:** Validated data is inserted into a Supabase PostgreSQL database with a default `pending` status.
* **Telegram Alert:** Sends a formatted message to the Admin's Telegram containing the donor's details along with inline `✅ Approve` and `❌ Reject` buttons.

**2. Decision Handling (Flow 2):**
* **Trigger:** Listens for `callback_query` updates when the Admin taps a button on Telegram.
* **Processing:** A Code node extracts the action (`approve` or `reject`) and the specific database Row ID.
* **Database Update:** The relevant donor's status is updated in Supabase to either `approved` or `rejected`, along with a timestamp.
* **UI Update:** Edits the original Telegram message to show the final decision, removing the buttons to prevent duplicate clicks.

## 🔑 Prerequisites & Credentials
To import and run this workflow, you will need to set up the following credentials in your n8n instance:
* **Google Sheets API:** To read new form responses.
* **Supabase API:** Project URL and API Key for database operations.
* **Telegram Bot API:** Bot token from BotFather to send and edit messages.

## 🛠️ Setup Instructions
1. Download the `Blood Donor Approval System.json` file from this repository.
2. Open your n8n workspace and go to your workflows.
3. Click on the `...` menu on the top right and select **Import from File**.
4. Upload the downloaded JSON file.
5. Re-authenticate all nodes with your own credentials (Google Sheets, Supabase, Telegram).
6. Activate the workflow!

---
*Built with [n8n](https://n8n.io/) - Advanced Workflow Automation.*
