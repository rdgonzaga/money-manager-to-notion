# Money Manager to Notion ETL

An automated Python ETL (Extract, Transform, Load) pipeline that migrates your mobile financial data from the **Money Manager** app directly into a **Notion** database.

## Features
* **Bypasses SQLite Locks:** Extracts data safely even if the app is syncing by using read-only URI modes.
* **Smart Filtering:** Automatically detects and filters out internal transfers (e.g., movements between your own accounts) using database-level `ZDO_TYPE` flags to prevent inflated cash flow reports.
* **Timestamp Correction:** Converts Apple CoreData timestamps (seconds since 2001-01-01) to standard ISO dates.
* **Incremental Sync:** Utilizes a local state file (`.sync_state`) to track the high-water mark of synced transactions, ensuring only new data is pushed via the API.
* **Interactive CLI:** Provides a menu-driven interface for initial bulk loads via CSV or incremental updates via the Notion API.

## Prerequisites
* Python 3.9+
* A Notion account (Personal or Pro).
* A Money Manager (Realbyte) backup file (`.mmbak` or `.sqlite`).

---

## 1. Notion Setup (The Target)

To ensure the pipeline functions correctly, your Notion database must be configured with specific property types.

### A. Create the Master Database
1. Create a new page in Notion.
2. Type `/database` and select **Database - Full page**.
3. Name it **Master Finance DB** (or your preferred name).

### B. Configure Properties
Set up the following properties exactly as listed (names are case-sensitive for the script):

| Property Name | Property Type | Description |
| :--- | :--- | :--- |
| **Transaction** | Title | The description or note of the transaction. |
| **Amount** | Number | The monetary value. Set format to your local currency. |
| **Category** | Select | The category (e.g., Food, Transport, Salary). |
| **Account** | Select | The source account (e.g., Cash, Bank, GCash). |
| **Date** | Date | The date the transaction occurred. |

### C. Create an Internal Connection
1. Go to [Notion My-Integrations](https://www.notion.so/my-integrations).
2. Click **+ New integration**.
3. Set the Type to **Internal**. Name it (e.g., "Money Manager Sync").
4. Under **Capabilities**, ensure "Read content", "Update content", and "Insert content" are checked.
5. Copy the **Internal Integration Token** (starts with `secret_`).

### D. Connect the Integration to the Database
1. Go to your **Master Finance DB** in Notion.
2. Click the three dots (`...`) in the top-right corner.
3. Scroll down to **Connections** > **Add connections**.
4. Search for your integration name and select it.

---

## 2. Installation

1. **Clone the repository:**
   ```bash
   git clone [https://github.com/rdgonzaga/money-manager-to-notion](https://github.com/rdgonzaga/money-manager-to-notion)
   cd money-manager-to-notion
   pip install -r requirements.txt
   python money_manager_to_notion.py