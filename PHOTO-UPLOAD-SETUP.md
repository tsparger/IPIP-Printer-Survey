# Printer photo uploads — setup

Adds two photo slots to every printer set: **Photo of printer** and **Photo of label stock**.
On a phone the control opens the camera directly.

Nothing appears in the survey until Part 3 is done and the URL is pasted in — the photo slots
are hidden while the upload flow URL is blank, so a live respondent never sees a control that
can't work. Follow the parts in order.

Budget about 25 minutes. Assumes [POWER-AUTOMATE-SETUP.md](POWER-AUTOMATE-SETUP.md) is already done.

---

## How it works

1. Respondent picks or takes a photo.
2. The browser downscales it to 1600px on the longest edge and re-encodes as JPEG at 0.72
   quality — a 4–5 MB phone photo becomes roughly 200–350 KB, with label text still readable.
3. That single photo POSTs to the **Upload Printer Photo** flow right away, while they keep
   filling out the form.
4. The flow writes it to a document library and returns the file URL.
5. The survey shows a thumbnail and holds only the URL.
6. On Submit, the URLs ride along in two new columns on the row.

Uploading per photo rather than all at once keeps the submit payload small, gives per-photo
error handling, and keeps image data out of the draft autosave — localStorage caps around 5 MB,
so base64 photos in a draft would break it.

---

## Part 1 — Create the document library

1. Same SharePoint site as **IPIP Printer Responses**.
2. **+ New → Document library**.
3. Name it **IPIP Printer Photos**. **Create**.
4. Keep permissions tight — the upload endpoint is anonymous (see the security note at the
   bottom), so treat this library as write-only from the internet and readable only by the
   IPIP team.

Use a dedicated library rather than an existing one. It keeps a size-heavy anonymous write
target isolated from anything else.

---

## Part 2 — Two new columns on IPIP Printer Responses

Add these to the existing list, exactly as named, no spaces:

| Column name | Type |
| --- | --- |
| `PrinterPhotoUrl` | Single line of text |
| `LabelPhotoUrl` | Single line of text |

Use **Single line of text**, not Hyperlink. The Hyperlink type expects an object with `Url` and
`Description` sub-fields, which complicates the Create item mapping for no benefit — the URL is
still clickable from the list view and from an export.

Leave both **not required**. A respondent may skip photos.

---

## Part 3 — Flow 3: "IPIP - Upload Printer Photo"

### 3.1 Create it

1. **+ Create → Instant cloud flow**, name **IPIP - Upload Printer Photo**,
   trigger **When an HTTP request is received**.
2. **Who can trigger the flow?** → **Anyone**
3. **Advanced parameters → Method** → **POST**
4. **Use sample payload to generate schema**, paste this, **Done**:

```json
{
  "action": "uploadPrinterPhoto",
  "fileName": "NCAL_DRV-HOSPITAL_DRV-1ST-FL-IP-RX-MAIN_set1_printer_ab12cd.jpg",
  "contentType": "image/jpeg",
  "fileContent": "BASE64STRING",
  "region": "NCAL",
  "location": "DRV-HOSPITAL",
  "department": "DRV 1ST FL IP RX MAIN",
  "printerSet": 1,
  "slot": "printer"
}
```

### 3.2 Reject oversized uploads

The browser already caps size, but an anonymous endpoint should not trust its caller.

1. **+ New step → Control → Condition**.
2. Left side — **Expression**:
   ```
   length(triggerBody()?['fileContent'])
   ```
3. Operator: **is less than or equal to** · Right side: `3000000`

   Base64 is about 4/3 the size of the bytes, so 3,000,000 characters is roughly a 2.2 MB
   file — comfortably above the ~350 KB the survey actually sends, and a hard stop on abuse.

4. Everything in 3.3 and 3.4 goes in the **If yes** branch.
5. In **If no**, add a **Response**: Status Code `413`, Body:
   ```json
   { "success": false, "error": "That photo is too large." }
   ```

### 3.3 Create the file (If yes branch)

1. **Add an action → SharePoint → Create file**
2. **Site Address**: your IPIP site
3. **Folder Path**: click the folder picker and choose **IPIP Printer Photos**
4. **File Name** — Expression:
   ```
   triggerBody()?['fileName']
   ```
5. **File Content** — Expression:
   ```
   base64ToBinary(triggerBody()?['fileContent'])
   ```

   `base64ToBinary` is required. Passing the base64 string straight through writes a text file
   full of base64 rather than a viewable image.

### 3.4 Respond with the URL (If yes branch, after Create file)

1. **+ Add an action → Request → Response**
2. Status Code `200`, header `Content-Type` = `application/json`
3. **Body**:
   ```json
   {
     "success": true,
     "url": "@{outputs('Create_file')?['body/{Link}']}",
     "path": "@{outputs('Create_file')?['body/Path']}"
   }
   ```

   Both are returned on purpose. `{Link}` is an absolute URL and is what you want, but it isn't
   present on every version of the SharePoint connector. The survey uses `url` when it's there
   and falls back to `path`, so this works either way — no need to check which you have.

### 3.5 Failure response

1. Add another **Response** after the Condition.
2. **⋯ → Run after** → uncheck **is successful**, check **has failed** and **has timed out**.
3. Status Code `500`, Body:
   ```json
   { "success": false, "error": "The photo could not be saved." }
   ```

### 3.6 Save and copy the URL

**Save**, reopen the trigger, copy the **HTTP POST URL**.

---

## Part 4 — Update Flow 1 (Submit Printer Survey)

Open **IPIP - Submit Printer Survey** → the **Create item** action inside the Apply to each,
and add two more field mappings:

| List field | Expression |
| --- | --- |
| PrinterPhotoUrl | `items('Apply_to_each_row')?['printerPhotoUrl']` |
| LabelPhotoUrl | `items('Apply_to_each_row')?['labelPhotoUrl']` |

Same naming rule as before — match your Apply to each's actual action name with spaces as
underscores. **Save**.

No trigger schema change is needed. The trigger schema is generated from a sample, and extra
properties in a real request are passed through rather than rejected. If you'd rather have the
schema exactly describe the payload, regenerate it from the sample in POWER-AUTOMATE-SETUP.md
with `"printerPhotoUrl": ""` and `"labelPhotoUrl": ""` added to the row object — but if you do
regenerate, you must re-check every field mapping afterward, since regenerating can clear them.

---

## Part 5 — Update Flow 2 (Get Printer Responses)

Open **IPIP - Get Printer Responses** → the **Select** action, and add two entries to the map:

```json
  "PrinterPhotoUrl": @{item()?['PrinterPhotoUrl']},
  "LabelPhotoUrl": @{item()?['LabelPhotoUrl']}
```

Add them inside the existing object, remembering the comma on the line before. **Save**.

---

## Part 6 — Turn it on in the survey

In `index.html`, fill in the constant:

```js
const EMBEDDED_UPLOAD_URL = 'paste the Upload Printer Photo POST URL here';
```

Commit and push. Once Pages redeploys, the two photo slots appear on every printer set. Leave
the constant blank and the survey behaves exactly as it does today.

---

## Part 7 — Test

1. Open the survey on a phone. Pick a region, facility, and department.
2. Tap **Photo of printer** — the camera should open directly.
3. Take a photo. Within a few seconds you should see a thumbnail, a green **Uploaded**, and a
   **Remove** button.
4. Check **IPIP Printer Photos** — a JPEG named
   `REGION_FACILITY_DEPT_set1_printer_xxxxxx.jpg`, viewable in the browser and a few hundred KB.
   If it's a text file or won't open, `base64ToBinary` is missing from step 3.3.
5. Do the same for **Photo of label stock**.
6. Submit, then check the row in **IPIP Printer Responses** — `PrinterPhotoUrl` and
   `LabelPhotoUrl` should both be populated and clickable.
7. Tap **Remove** on a photo before submitting and confirm the slot returns to an empty
   file input. Note the file stays in the library; Remove only detaches it from the submission.

---

## Notes

**The upload endpoint is anonymous**, like your existing submit flow. Anyone with the URL can
write files into that library — which is why the size cap in 3.2 and a dedicated library both
matter. If you ever need to cut it off, regenerate the flow URL (which requires a survey
redeploy with the new URL) or turn the flow off.

**Photos in a clinical setting.** The survey shows a warning next to the upload controls telling
respondents to photograph the printer and label stock only, and not to capture screens, patient
labels, paperwork, or people. That's a prompt, not an enforcement mechanism — someone will
eventually upload something they shouldn't. Worth agreeing with whoever owns privacy review for
the IPIP surveys how the library gets periodically reviewed, before this goes out broadly.

**Orphaned files.** A photo uploads as soon as it's picked, so a respondent who uploads and then
abandons the survey leaves a file with no matching row. Harmless, but the library will
accumulate some. To find them, compare filenames against the `PrinterPhotoUrl` / `LabelPhotoUrl`
values in the list.

**HEIC.** iPhones generally hand a JPEG to a web file input even when the photo is stored as
HEIC, so this works. If a photo ever fails to process, the slot shows
"Could not read this image. Try a JPG or PNG."
