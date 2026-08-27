# IPIP Printer Validation Survey

KP NPI IPIP survey for validating bin-label and report printer information at each
pharmacy department location.

**Live survey:** https://tsparger.github.io/IPIP-Printer-Survey/
**Dashboard:** https://tsparger.github.io/IPIP-Survey-Dashboard/

## Files

| File | Purpose |
| --- | --- |
| `index.html` | The whole survey — form, validation, draft autosave, Power Automate submit |
| `survey-data.js` | Region / Facility / Pharmacy Department Location list + addresses |
| `POWER-AUTOMATE-SETUP.md` | Step-by-step build of the SharePoint list and the two flows |

## Questions asked

Per pharmacy department location, one or more **printer sets**. Each set captures:

1. OneLink Zebra (Bin Label) Printer Name
2. OneLink Zebra (Bin Label) Printer IP Address
3. Report Printer Name
4. Report Printer IP Address
5. Full Computer Name
6. Label paper stock info (include dimensions)
7. Comments (optional)

Region, Facility, and Pharmacy Department Location are picked from dropdowns; Address /
City / State / Zip auto-fill from the selection and stay editable.

## Setup checklist

1. Build the SharePoint list and both flows — see [POWER-AUTOMATE-SETUP.md](POWER-AUTOMATE-SETUP.md).
2. Paste the **Submit** flow URL into `EMBEDDED_FLOW_URL` in `index.html`.
3. Paste the **Get Responses** flow URL into `EMBEDDED_RESPONSES_URL` in `index.html`.
4. Commit and push — GitHub Pages redeploys automatically.
5. Paste the Get Responses URL into the dashboard's **Settings → Printer Survey** field,
   or set `defaultUrl` on the `printer` entry in the dashboard's `SURVEYS` config.

Append `?admin=1` to the survey URL to reveal the Admin tab, where the same two URLs can
be set for your browser only (useful for testing before you hardcode them).

## Refreshing the location list

`survey-data.js` is generated from the pharmacy survey so the two always agree on
Region / Facility / Pharmacy Department Location. To regenerate after the pharmacy list
changes, from the parent folder of both repos:

```bash
node -e "
const fs=require('fs');
const src=fs.readFileSync('IPIP-Pharmacy-Survey/survey-data.js','utf8');
const D=(new Function(src+'; return SURVEY_DATA;'))();
const sub={regions:D.regions,pharmacyLocations:D.pharmacyLocations,pharmacyDepartments:D.pharmacyDepartments};
const header='// Location data for the IPIP Printer Validation Survey.\n'
 +'// Regenerated from the IPIP Pharmacy Survey so Region / Facility / Pharmacy Department\n'
 +'// Location always match across surveys. To refresh after the pharmacy list changes, run\n'
 +'// the snippet in README.md. Keys mirror the pharmacy file so the dashboard can reuse them.\n';
fs.writeFileSync('IPIP-Printer-Survey/survey-data.js', header+'const SURVEY_DATA = '+JSON.stringify(sub)+';\n');
"
```

Bump the `?v=` query string on the `survey-data.js` script tag in `index.html` so browsers
pick up the new file.
