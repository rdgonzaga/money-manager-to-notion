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
2. Click **Create a new connection**.
3. Make sure its type is set to **Internal**. Name it (e.g., "Money Manager Sync").
4. Make sure you select the notion page you created under **Installable in** dropdown.
5. Under **Capabilities**, ensure "Read content", "Update content", and "Insert content" are checked.
6. Copy the **Internal Integration Token**.

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
   ```

2. **Create a `.env` file** in the project root with your credentials:
   ```bash
   cp .env.example .env
   ```

3. **Edit `.env`** with your values:
   ```
   NOTION_API_KEY=xxxxxxxxxxxxxxxxxxxx
   NOTION_FINANCE_DB_ID=your_database_id_here
   MM_DB_PATH=/path/to/Money/Manager/database.mmbak
   ```

---

## 3. Configuration

### A. Get Your Money Manager Database Path

The Money Manager app stores transaction data in a SQLite database. Find it:

- Check Money Manager app's backup settings
- Export the `.mmbak` file
   - You can send it to your email and just download it on the device where the script will run.

### B. Configure Environment Variables

Edit `.env` with the following:

| Variable | Description | Example |
| :--- | :--- | :--- |
| `NOTION_API_KEY` | Your Notion Internal Integration Token | `abc1234567...` |
| `NOTION_FINANCE_DB_ID` | Your Notion Finance database ID (36 chars, from URL) | `a1b2c3d4e5f6...` |
| `MM_DB_PATH` | Full path to Money Manager database file | `/path/to/Money/Manager/database.mmbak` |

---

## 4. Usage

Run the script:
```bash
python money_manager_to_notion.py
```

You'll see an interactive menu with 4 operations:

### Operation 1: Initial Setup (CSV Export)
- **When to use:** First time running the pipeline
- **What it does:**
  - Extracts all historical transactions from Money Manager
  - Saves them to `Notion_Initial_Load.csv`
  - Creates `.sync_state` file to track future syncs
- **Next steps:**
  1. Open the generated `Notion_Initial_Load.csv`
  2. Go to your Notion Finance Database
  3. Click **+ Add** and select **Merge with CSV**
  4. Upload the CSV file

### Operation 2: Incremental Sync (Notion API)
- **When to use:** After initial setup, to sync new transactions
- **What it does:**
  - Queries only new transactions since last sync
  - Directly uploads them to Notion via API
  - Updates `.sync_state` with the latest timestamp
- **Note:** Requires Option 1 to be run first to establish baseline
- **If `.sync_state` is missing:**
  - Script will warn you and ask confirmation to sync full history
  - Not recommended unless you want all transactions re-synced

### Operation 3: Reset Sync State
- **When to use:** When you want to restart the sync pipeline
- **What it does:**
  - Deletes `.sync_state` file
  - Resets pipeline to zero
- **Warning:** Next incremental sync will ask to sync full history
- **Use case:** After manual Notion database cleanup/modifications

### Operation 4: Exit
- Safely closes the program

---

## 5. Error Handling

### Common Error Messages

| Error | Cause | Solution |
| :--- | :--- | :--- |
| `NOTION_API_KEY not found in .env` | Missing Notion token | Add token to `.env` |
| `MM_DB_PATH not found in .env` | Missing database path | Add path to `.env` |
| `Database file not found at: /path/to/file` | Wrong path in `.env` | Verify path and update |
| `Invalid selection. Please enter 1-4` | User entered wrong menu option | Enter 1, 2, 3, or 4 |
| `Rate limit encountered` | Notion API throttling | Script auto-pauses 5 seconds and retries |
| `No sync state found` | First incremental sync | Run Option 1 first or confirm full history sync |

---

## 6. Troubleshooting

**Q: How do I get my Notion Database ID?**
- Open your Finance database in Notion
- Copy the URL: `https://www.notion.so/YOUR_ID?v=...`
- The ID is the 32-char string after `/`

**Q: The script says "Database connection failed"**
- Make sure the path in `.env` is correct and readable
- Try using the full absolute path instead of relative path

**Q: How often should I run Operation 2?**
- Run as frequently as you want (e.g., daily cron job, weekly manual)
- Each run syncs only new transactions since last run
- No duplicate entries will be created

**Q: Can I modify the Notion database structure?**
- Property names are **case-sensitive**: `Transaction`, `Amount`, `Category`, `Account`, `Date`
- If you rename properties, update the script's `payload` dictionary
- Adding extra properties is fine; script only manages the 5 core properties

**Q: How do I undo a sync?**
- Delete records manually from Notion (no automatic deletion in script)
- Use Option 3 to reset `.sync_state` and re-run from scratch if needed

---

## 7. Example Workflow

```
Day 1: Initial Setup
$ python money_manager_to_notion.py
> Select operation: 1
[SUCCESS] Exported 287 records to Notion_Initial_Load.csv
[INFO] Sync state established...

→ Upload CSV to Notion manually

Day 8: Incremental Sync
$ python money_manager_to_notion.py
> Select operation: 2
[INFO] Querying new records after timestamp 678900000...
[SUCCESS] Synced: 2024-05-08 | 45.50 | Bank Account
[SUCCESS] Synced: 2024-05-09 | 120.00 | Cash
[INFO] Sync state successfully updated.
```

---

## License

See [LICENSE](LICENSE) file for details