# Printer Survey — SharePoint list + Power Automate setup

Build this the same way the Department and Pharmacy surveys are built: one SharePoint list,
one **Submit** flow (HTTP POST, called when a respondent clicks Submit Survey), and one
**Get Responses** flow (HTTP GET, called by the survey for completion checkmarks and by the
dashboard for progress tracking).

Budget about 45 minutes.

---

## Part 1 — Create the SharePoint list

### 1.1 Create the list

1. Go to the same SharePoint site that holds **IPIP Responses** and **IPIP Pharmacy Responses**.
2. **+ New → List → Blank list**.
3. Name it **IPIP Printer Responses**. Leave "Show in site navigation" checked.
4. **Create**.

### 1.2 Add the columns

One row in this list = **one printer set** (one bin-label printer). A pharmacy with two
bin-label printers produces two rows that repeat the same Region / Location / Department.

Add each column with **+ Add column**. Type the names *exactly* as shown, with no spaces —
SharePoint turns a space into `_x0020_` in the internal name, and the flows reference
internal names.

| Column name | Type | Notes |
| --- | --- | --- |
| `Title` | Single line of text | Already exists. The flow fills it with `REGION - FACILITY - DEPARTMENT - Set N` |
| `RespondentName` | Single line of text | |
| `SubmittedAt` | Single line of text | ISO timestamp. Keep as text — a Date column will shift by timezone |
| `Region` | Single line of text | NCAL / SCAL / NW / HI |
| `Location` | Single line of text | The **Facility** |
| `Department` | Single line of text | The **Pharmacy Department Location** |
| `Address` | Single line of text | |
| `City` | Single line of text | |
| `State` | Single line of text | |
| `Zip` | Single line of text | |
| `PrinterSet` | Number | 1, 2, 3… within a department |
| `ZebraPrinterName` | Single line of text | |
| `ZebraPrinterIP` | Single line of text | |
| `ReportPrinterName` | Single line of text | |
| `ReportPrinterIP` | Single line of text | |
| `FullComputerName` | Single line of text | |
| `LabelPaperStock` | Multiple lines of text | **Turn off rich text** (see below) |
| `Comments` | Multiple lines of text | **Turn off rich text** |
| `RawData` | Multiple lines of text | **Turn off rich text.** JSON copy of the row, for rebuilding an entry later |

> **Turn off rich text on every multi-line column.** In the column's settings expand
> **More options** and set **Use enhanced rich text** to **No** (classic settings call it
> "Plain text"). If you skip this, SharePoint wraps stored values in `<div>` tags and the
> exports come back full of HTML — this is the same problem that had to be fixed on the
> Department survey.

### 1.3 Do NOT make any column required

The flow writes every field. A required column that the flow leaves blank fails the whole
Create item action.

### 1.4 Optional: a nicer default view

Add a view **By Location** grouped on `Region` then `Location`, showing `Department`,
`PrinterSet`, `ZebraPrinterName`, `ZebraPrinterIP`, `FullComputerName`, `LabelPaperStock`.

---

## Part 2 — Flow 1: "IPIP - Submit Printer Survey"

This receives the survey POST and writes one list item per printer set.

### 2.1 Create the flow

1. https://make.powerautomate.com → same environment as the other two IPIP flows.
2. **+ Create → Instant cloud flow**.
3. Name: **IPIP - Submit Printer Survey**.
4. Trigger: **When an HTTP request is received**. → **Create**.

### 2.2 Configure the trigger

1. Open the trigger card. Set **Who can trigger the flow?** to **Anyone**.
   Respondents hit this from GitHub Pages with no KP sign-in, so anonymous is required.
   The URL's `sig=` token is the only secret — treat the URL as sensitive.
2. Set **Method** to `POST` (under **Show advanced options**).
3. Click **Use sample payload to generate schema** and paste this, then **Done**:

```json
{
  "action": "submitPrinterSurvey",
  "respondentName": "Jane Doe",
  "submittedAt": "2026-08-27T18:00:00.000Z",
  "rows": [
    {
      "respondentName": "Jane Doe",
      "submittedAt": "2026-08-27T18:00:00.000Z",
      "region": "NCAL",
      "location": "DRV-HOSPITAL",
      "department": "DRV 1ST FL IP RX MAIN",
      "address": "4501 Sand Creek Road",
      "city": "Antioch",
      "state": "California",
      "zip": "94531",
      "printerSet": 1,
      "zebraName": "DRV-RX-ZEB01",
      "zebraIp": "10.20.30.40",
      "reportName": "DRV-RX-RPT01",
      "reportIp": "10.20.30.41",
      "computerName": "DRVRXWS01.kp.org",
      "paperStock": "Zebra Z-Select 4000D, 2 in x 1 in, direct thermal, 1 in core",
      "comments": "Behind the carousel",
      "title": "NCAL - DRV-HOSPITAL - DRV 1ST FL IP RX MAIN - Set 1",
      "raw": "{}"
    }
  ]
}
```

The generated schema will mark `printerSet` as `integer`. That is correct — the survey
always sends it as a number.

### 2.3 Add "Apply to each"

1. **+ New step → Control → Apply to each**.
2. **Select an output from previous steps**: click the dynamic-content field, choose
   **Expression**, and enter:
   ```
   triggerBody()?['rows']
   ```
   (Or pick **rows** from the dynamic content list.)
3. Rename the action to **Apply to each row** so the expressions below read clearly.

### 2.4 Inside the loop: Create item

1. Inside the Apply to each, **Add an action → SharePoint → Create item**.
2. **Site Address**: the IPIP site. **List Name**: **IPIP Printer Responses**.
3. Fill each field with an expression. Click the field → **Expression** tab → paste → **OK**.

| List field | Expression |
| --- | --- |
| Title | `items('Apply_to_each_row')?['title']` |
| RespondentName | `items('Apply_to_each_row')?['respondentName']` |
| SubmittedAt | `items('Apply_to_each_row')?['submittedAt']` |
| Region | `items('Apply_to_each_row')?['region']` |
| Location | `items('Apply_to_each_row')?['location']` |
| Department | `items('Apply_to_each_row')?['department']` |
| Address | `items('Apply_to_each_row')?['address']` |
| City | `items('Apply_to_each_row')?['city']` |
| State | `items('Apply_to_each_row')?['state']` |
| Zip | `items('Apply_to_each_row')?['zip']` |
| PrinterSet | `items('Apply_to_each_row')?['printerSet']` |
| ZebraPrinterName | `items('Apply_to_each_row')?['zebraName']` |
| ZebraPrinterIP | `items('Apply_to_each_row')?['zebraIp']` |
| ReportPrinterName | `items('Apply_to_each_row')?['reportName']` |
| ReportPrinterIP | `items('Apply_to_each_row')?['reportIp']` |
| FullComputerName | `items('Apply_to_each_row')?['computerName']` |
| LabelPaperStock | `items('Apply_to_each_row')?['paperStock']` |
| Comments | `items('Apply_to_each_row')?['comments']` |
| RawData | `items('Apply_to_each_row')?['raw']` |

> **The name inside `items(...)` must match your action's name with spaces replaced by
> underscores.** If you left it as "Apply to each", use `items('Apply_to_each')`. If Power
> Automate auto-numbered it "Apply to each 2", use `items('Apply_to_each_2')`. A mismatch
> here is the single most common reason this flow fails to save.

### 2.5 Speed the loop up

Click the **⋯** on **Apply to each row → Settings**, turn **Concurrency Control** **On**, and
set **Degree of Parallelism** to **20**. A pharmacy with several printer sets then writes in
one or two seconds instead of serially.

### 2.6 Respond to the survey

**Outside and after** the Apply to each (not nested inside it):

1. **+ New step → Request → Response**.
2. **Status Code**: `200`
3. **Headers**: add `Content-Type` = `application/json`
4. **Body**:
   ```json
   {
     "success": true,
     "rowsCreated": @{length(triggerBody()?['rows'])}
   }
   ```
   Paste this into the Body box in code view; the `@{...}` is an inline expression.

The survey only treats a submission as successful when the JSON it gets back has
`success: true`. Anything else surfaces as an error toast to the respondent.

### 2.7 Handle failures honestly

So a respondent is told when a write fails instead of seeing a false success:

1. Add a second **Response** action after the first.
2. On that action, **⋯ → Configure run after** → uncheck **is successful**, check
   **has failed** and **has timed out**.
3. Status Code `500`, Body:
   ```json
   { "success": false, "error": "One or more printer rows could not be saved." }
   ```
4. On the *first* Response action, leave run-after as **is successful** only.

### 2.8 Save and copy the URL

**Save**. Reopen the trigger card — the **HTTP POST URL** is now filled in. Copy it.

---

## Part 3 — Flow 2: "IPIP - Get Printer Responses"

This feeds the completion checkmarks in the survey and the progress numbers on the dashboard.

### 3.1 Create the flow

1. **+ Create → Instant cloud flow**, name **IPIP - Get Printer Responses**,
   trigger **When an HTTP request is received**.
2. Trigger settings: **Who can trigger the flow?** = **Anyone**, **Method** = `GET`.
   No request schema is needed.

### 3.2 Get items

1. **+ New step → SharePoint → Get items**.
2. **Site Address**: the IPIP site. **List Name**: **IPIP Printer Responses**.
3. **Show advanced options → Top Count**: `5000`.
4. **⋯ → Settings → Pagination**: **On**, **Threshold** `100000`.
   Without this you silently stop at 100 rows and the dashboard under-reports.

### 3.3 Shape the output

1. **+ New step → Data Operation → Select**.
2. **From**: the **value** output of Get items.
3. Switch the Map to text mode (the small icon at the right of the Map box) and paste:

```json
{
  "Region": @{item()?['Region']},
  "Location": @{item()?['Location']},
  "Department": @{item()?['Department']},
  "RespondentName": @{item()?['RespondentName']},
  "SubmittedAt": @{item()?['SubmittedAt']},
  "Address": @{item()?['Address']},
  "City": @{item()?['City']},
  "State": @{item()?['State']},
  "Zip": @{item()?['Zip']},
  "PrinterSet": @{item()?['PrinterSet']},
  "ZebraPrinterName": @{item()?['ZebraPrinterName']},
  "ZebraPrinterIP": @{item()?['ZebraPrinterIP']},
  "ReportPrinterName": @{item()?['ReportPrinterName']},
  "ReportPrinterIP": @{item()?['ReportPrinterIP']},
  "FullComputerName": @{item()?['FullComputerName']},
  "LabelPaperStock": @{item()?['LabelPaperStock']},
  "Comments": @{item()?['Comments']},
  "RawData": @{item()?['RawData']}
}
```

Keeping the payload to just these fields matters — returning the raw SharePoint objects
makes the response several times larger and can push the flow past its response timeout.

### 3.4 Respond

1. **+ New step → Request → Response**.
2. Status Code `200`, header `Content-Type` = `application/json`.
3. **Body** — click the field, choose **Expression**, and enter:
   ```
   json(concat('{"responses":', string(body('Select')), '}'))
   ```
   This is the exact shape the survey and the dashboard expect:
   `{ "responses": [ … ] }`.

### 3.5 Save and copy the URL

**Save**, reopen the trigger, copy the **HTTP GET URL**.

Sanity check: paste that URL into a browser tab. You should get
`{"responses":[]}` on an empty list.

---

## Part 4 — Wire the URLs in

### 4.1 The survey

Edit `index.html` in this repo and fill in the two constants near the top of the script:

```js
const EMBEDDED_FLOW_URL = 'paste the Submit Printer Survey POST URL here';
const EMBEDDED_RESPONSES_URL = 'paste the Get Printer Responses GET URL here';
```

Commit and push. GitHub Pages redeploys in a minute or two.

These must be hardcoded in the file — respondents open the page with empty localStorage, so
the Admin screen alone will not make the survey work for them. (`?admin=1` on the URL still
gives you a place to paste the URLs for your own browser while testing, before you commit.)

### 4.2 The dashboard

Either:

- Open https://tsparger.github.io/IPIP-Survey-Dashboard/, click **Settings**, paste the
  **Get Printer Responses** URL into **Printer Survey — Get Responses URL**, **Save**; or
- Better, so it works for everyone: in the dashboard repo's `index.html`, find the
  `printer` entry in the `SURVEYS` config and set `defaultUrl` to the Get Responses URL,
  then commit and push.

Until one of those is done, the dashboard's Printer Survey card reads
"Get Responses URL not configured — add it under Settings."

---

## Part 5 — Test end to end

1. Open the survey. Pick a Region, Facility, and Pharmacy Department Location; confirm the
   address auto-fills.
2. Fill one printer set and click **Submit Survey**. You should get the green success screen.
3. Check **IPIP Printer Responses** in SharePoint — one new item, `Title` reading
   `REGION - FACILITY - DEPARTMENT - Set 1`, and no `<div>` tags in `LabelPaperStock`.
4. Submit a second pharmacy with **two** printer sets. Confirm **two** rows land, both with
   the same Region / Location / Department and `PrinterSet` of 1 and 2.
5. Reload the survey — the department you submitted should now show a ✅ in the
   Pharmacy Department Location dropdown.
6. Open the dashboard, hit **Refresh**, click the **Printer Survey** tab. The submitted
   count and printer-set count should match what you entered.

---

## How the dashboard counts this survey

- **In scope** = all 130 pharmacy department locations in `survey-data.js` (the same list
  the Pharmacy Survey uses).
- **Submitted** = that Region + Location + Department has at least one row in the list,
  counted once no matter how many times it was submitted.
- **Printer sets** = total rows — one per bin-label printer.

There is no separate "Validated" metric for this survey. Submitting already requires every
printer field, so a validated count would always equal the submitted count. If you later add
a confirmation question (say a `PrinterInfoValidated` Yes/No column), set `hasValidated: true`
and supply an `isConfirmed` function on the `printer` entry in the dashboard's `SURVEYS`
config, and the second metric switches over automatically.

---

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| Flow won't save: "The expression is invalid" | The name inside `items('…')` doesn't match the Apply to each action name. Spaces become underscores |
| Survey shows "Submission failed" but rows appear in SharePoint | The Response action is nested *inside* the Apply to each, or the body isn't returning `success: true` |
| Survey shows a timeout message | The Apply to each is running serially — turn on Concurrency Control (2.5) |
| Dashboard shows 0 submitted but rows exist | The Select in 3.3 isn't emitting `Region` / `Location` / `Department` under those exact names |
| Dashboard stops counting past 100 rows | Pagination is off on Get items (3.2) |
| Exported paper stock full of `<div>` | Rich text wasn't disabled on the multi-line columns (1.2) |
| Respondents get a CORS or 401 error | The trigger's "Who can trigger the flow?" is not set to **Anyone** |
