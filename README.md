# Malaria Monthly Form: Spring Boot to DHIS2

A small, complete example. A health worker fills a form with two numbers, **Suspected** and **Tested**, for one facility and one month. The Spring Boot app saves it and sends it to a matching data set in DHIS2.

The example is set up for the DHIS2 2.40 demo server (`https://play.im.dhis2.org/dev-2-40`, login `admin` / `district`), so it works without changing any UIDs. Only the token has to be added.

## What is in the project

```
malaria-form-dhis2/
├── dhis2/dhis2_metadata.json                    the DHIS2 form: 2 data elements + 1 data set
├── samples/
│   ├── 1_form_submission.json                   what the Spring Boot form sends when you press Save
│   ├── 2_saved_submission.json                  what Spring Boot keeps after checking it
│   └── 3_dhis2_dataValueSets.json               what Spring Boot sends to DHIS2
└── src/main/
    ├── resources/
    │   ├── application.yml                      DHIS2 address, token, data set, mapping files
    │   ├── mappings/data_element_mapping.csv    form field  ->  DHIS2 data element
    │   ├── mappings/facility_mapping.csv        facility code  ->  DHIS2 org unit
    │   └── static/index.html                    the form page
    └── java/sl/icf/malariaform/                 the Java code
```

## How the two sides match

| Spring Boot form field | DHIS2 data element | Data element UID | Category option combo UID |
|---|---|---|---|
| `suspected` | ICF Malaria Suspected (demo) | `IcfMalSus01` | `HllvX50cXC0` (default) |
| `tested` | ICF Malaria Tested (demo) | `IcfMalTst01` | `HllvX50cXC0` (default) |

| Facility code | Facility | DHIS2 org unit UID |
|---|---|---|
| `HF001` | Ngelehun CHC, Bo | `DiszpKrYNg8` |

Both data elements belong to the data set **ICF Malaria Monthly (demo)**, UID `IcfMalDS001`, with a monthly period. Neither is broken down by age or sex, so both use the DHIS2 default combo `HllvX50cXC0`.

## Step 1: The Spring Boot form

The form is `src/main/resources/static/index.html`. It has a facility dropdown (filled from `facility_mapping.csv`), a month, Suspected and Tested.

When you press **Save**, the page sends this JSON to `POST /api/submissions` (`samples/1_form_submission.json`):

```json
{
  "facilityCode": "HF001",
  "period": "2026-09",
  "suspected": 120,
  "tested": 118
}
```

The app checks that the facility is in the mapping file, that the month is valid and that both numbers are 0 or more. It then turns the month into DHIS2 format (`202609`) and keeps the form. Saving the same facility and month again replaces the earlier values.

The forms are kept in memory, so they are lost when the app stops. For real use, save them in your database.

## Step 2: The same form in DHIS2

`dhis2/dhis2_metadata.json` creates the two data elements and the data set, and assigns the data set to Ngelehun CHC.

1. Log in to `https://play.im.dhis2.org/dev-2-40` as `admin` / `district`.
2. Open the **Import/Export** app, then **Metadata import**.
3. Choose `dhis2_metadata.json`, set **Import mode** to *Validate* (dry run), and click **Start import**. It should report no errors.
4. Change **Import mode** to *Import* and start it again. You should see 2 data elements and 1 data set created.
5. Open the **Data Entry** app, pick Ngelehun CHC and the data set *ICF Malaria Monthly (demo)*. You will see the same two boxes as the Spring Boot form: Suspected and Tested.

The demo server resets every night, so repeat this step on any day you test.

## Step 3: The mapping files

`src/main/resources/mappings/data_element_mapping.csv`

```
source_field,data_element_name,data_element_uid,category_option_combo,category_option_combo_uid
suspected,ICF Malaria Suspected (demo),IcfMalSus01,default,HllvX50cXC0
tested,ICF Malaria Tested (demo),IcfMalTst01,default,HllvX50cXC0
```

`src/main/resources/mappings/facility_mapping.csv`

```
facility_code,facility_name,district,org_unit_uid
HF001,Ngelehun CHC,Bo,DiszpKrYNg8
```

The word in `source_field` must be exactly the name the form uses (`suspected`, `tested`). That is the link between the form and DHIS2.

To add a facility, find its UID with
`https://play.im.dhis2.org/dev-2-40/api/organisationUnits?fields=id,name&filter=name:ilike:Bumpeh&paging=false`
(change the name), add a row to `facility_mapping.csv`, and assign the facility to the data set in the **Maintenance** app.

## Step 4: application.yml

```yaml
server:
  port: 8080

dhis2:
  base-url: ${DHIS2_URL:https://play.im.dhis2.org/dev-2-40}
  token: ${DHIS2_TOKEN:put-your-personal-access-token-here}
  data-set: IcfMalDS001
  dry-run: true
  mapping:
    data-elements-file: classpath:mappings/data_element_mapping.csv
    facilities-file: classpath:mappings/facility_mapping.csv
```

Nothing here needs changing for the demo. The token comes from the `DHIS2_TOKEN` environment variable in the next step.

## Step 5: Get the access token

1. In DHIS2, click your initials (top right), then **Edit profile**.
2. In the left menu, click **Personal access tokens**, then **Generate new token**.
3. Choose **Server/script context**, pick an expiry such as 7 days, and click **Generate new token**.
4. Copy the token straight away. It starts with `d2pat_` and is shown only once.

Test it (Linux or Mac):

```bash
curl -H "Authorization: ApiToken d2pat_xxxxxxxx" "https://play.im.dhis2.org/dev-2-40/api/me?fields=username"
```

The answer should be `{"username":"admin"}`.

## Step 6: Run the app

Windows Command Prompt:

```
set DHIS2_TOKEN=d2pat_PASTE_YOUR_TOKEN_HERE
mvn spring-boot:run
```

Windows PowerShell:

```
$env:DHIS2_TOKEN="d2pat_PASTE_YOUR_TOKEN_HERE"
mvn spring-boot:run
```

Linux or Mac:

```
export DHIS2_TOKEN=d2pat_PASTE_YOUR_TOKEN_HERE
mvn spring-boot:run
```

When it starts, the log shows:

```
Loaded 2 data element mappings from [classpath:mappings/data_element_mapping.csv]
Loaded 1 facility mappings from [classpath:mappings/facility_mapping.csv]
```

Open `http://localhost:8080` in your browser.

## Step 7: Collect and send

1. Choose Ngelehun CHC, September 2026, Suspected 120, Tested 118, and press **Save**.
2. Press **Test send (dry run)**. DHIS2 checks the values but saves nothing. You want status `SUCCESS` and no conflicts.
3. Press **Send to DHIS2**. The values are saved.
4. In DHIS2 Data Entry, open Ngelehun CHC, the data set and September 2026. The boxes now show 120 and 118.

This is what the app sent (`samples/3_dhis2_dataValueSets.json`):

```json
{
  "dataSet": "IcfMalDS001",
  "dataValues": [
    { "dataElement": "IcfMalSus01", "categoryOptionCombo": "HllvX50cXC0",
      "orgUnit": "DiszpKrYNg8", "period": "202609", "value": "120" },
    { "dataElement": "IcfMalTst01", "categoryOptionCombo": "HllvX50cXC0",
      "orgUnit": "DiszpKrYNg8", "period": "202609", "value": "118" }
  ]
}
```

## If something goes wrong

| Message | Cause | Fix |
|---|---|---|
| DHIS2 rejected the token (401) | Token wrong, expired or wiped by the nightly reset | Generate a new token, set `DHIS2_TOKEN`, restart |
| Conflict: data element not found | Step 2 import not done (or the demo was reset) | Import `dhis2_metadata.json` again |
| Conflict: org unit not assigned to data set | Facility added to the CSV but not to the data set | Assign it in the Maintenance app |
| Cannot reach DHIS2 | Wrong address or no internet | Check `base-url` and your connection |
| Unknown facility | Facility code not in `facility_mapping.csv` | Add the row and restart |

## Moving to the real NMCP server

Import `dhis2_metadata.json` there (or create the data elements and data set by hand and put their UIDs in `data_element_mapping.csv`), put your real facilities in `facility_mapping.csv`, and set `DHIS2_URL` and `DHIS2_TOKEN` for that server. The Java code stays the same.
