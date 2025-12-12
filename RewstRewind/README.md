# General – MSP Rewind (Rewst Workflow)

This repository contains the **General – MSP Rewind** Rewst workflow.

The workflow pulls 180-day usage reports from **Microsoft Graph** for:

- Exchange Online (email)
- Microsoft Teams
- Viva Engage / Yammer

It then stores the parsed results into Rewst templates and exposes a per-user summary in the workflow output.

> ⚠️ Before you run this workflow, you **must** configure a few variables in Rewst:
> - `long_company`
> - `short_company`
> - `email_template_id`
> - `teams_template_id`
> - `viva_template_id`

---

## Quickstart

If you just want this up and running, follow these steps:

1. **Import the workflow**  
   - In Rewst, go to **Automations → Workflows → Import** and import the `General – MSP Rewind` workflow JSON from this repo.

2. **Set the workflow variables**  
   - Edit the imported workflow and set:
     - `long_company`, `short_company`
     - `email_template_id`, `teams_template_id`, `viva_template_id`

3. **Import the AppBuilder page**  
   - Open **AppBuilder**, create or open a page, and use **⋮ → Load Page State** to import the `pageState.json` file from this repo.

4. **Assign the workflow to the page**  
   - In the page **Settings → Workflows (array)**, add the **General – MSP Rewind** workflow.
   - Publish the page.

After those four steps, you should have a working **MSP Rewind** page driven by this workflow.

---

## How it works (high level)

1. **Optional refresh (controlled by `refresh_data`)**
   - Calls Microsoft Graph reporting endpoints:
     - `reports/getEmailActivityUserDetail(period='D180')`
     - `reports/getTeamsUserActivityUserDetail(period='D180')`
     - `reports/getYammerActivityUserDetail(period='D180')`
   - Follows the export URLs and downloads the CSV data.
   - Parses the CSV into JSON and **updates three Rewst templates**:
     - Email usage template (`email_template_id`)
     - Teams usage template (`teams_template_id`)
     - Viva/Yammer usage template (`viva_template_id`)

2. **Per-user lookup**
   - For the currently logged-in Rewst user (`CTX.user.username`), the workflow looks up their record in each of the three templates.
   - The workflow output object includes:
     - `teams`
     - `email`
     - `viva`
   - These can then be used downstream (dashboards, emails, other automations, etc.).

---

## Configuration

All configuration is done in the edit Workflow settings in Rewst.

### Required variables

Click the pencil to edit the workflow and update the variables.

| Variable            | Required | Description                                                                                                         |
| ------------------- | -------- | ------------------------------------------------------------------------------------------------------------------- |
| `long_company`      | No       | Full, “long form” company name. **Change this to your organization’s full name.**                                  |
| `short_company`     | No       | Short company name or acronym. **Change this to your preferred short name.**                                       |
| `email_template_id` | Yes      | ID of a Rewst template that will store the parsed **Email activity** JSON.                                         |
| `teams_template_id` | Yes      | ID of a Rewst template that will store the parsed **Teams activity** JSON.                                         |
| `viva_template_id`  | Yes      | ID of a Rewst template that will store the parsed **Viva/Yammer activity** JSON.                                   |

The export you see here includes defaults, but **you should set these to values that make sense in your own Rewst tenant**.

#### 1. Set company name variables

In Rewst, open the workflow and go to **Vars / Schema**:

- `long_company`  
  - Set this to your full company name (e.g. `Contoso Technology Services`).

- `short_company`  
  - Set this to your short name or acronym (e.g. `Contoso` or `CTS`).

These variables are intended for use in messaging, dashboards, and any user-facing output that references your company name.

#### 2. Create / locate the Rewst templates

You need **three** templates in Rewst that will hold the parsed Microsoft Graph report data:

- **Email usage template**
  - Will contain the JSON-converted results of `getEmailActivityUserDetail`.
- **Teams usage template**
  - Will contain the JSON-converted results of `getTeamsUserActivityUserDetail`.
- **Viva/Yammer usage template**
  - Will contain the JSON-converted results of `getYammerActivityUserDetail`.

For each of those templates:

1. Create (or identify) the template in Rewst.
2. Copy the **Template ID** from Rewst.
3. Paste the ID into the corresponding variable in the workflow:

   - `email_template_id` → Email usage template ID  
   - `teams_template_id` → Teams usage template ID  
   - `viva_template_id` → Viva/Yammer usage template ID  

The tasks named `update_email_template`, `update_teams_template`, and `update_viva_template` will write into these templates.

---

## Typical usage

1. **Prime the data**
   - From the workflow **Test** tab, run the workflow with:
     - `refresh_data = true`
   - This will pull fresh data from Microsoft Graph and populate the three templates.

---

## AppBuilder integration (MSP Rewind page)

To use this workflow with the provided AppBuilder page:

### 1. Load pageState

1. In this repository, locate the exported **page state** file (e.g. `pageState.json`).
2. In AppBuilder, create or open a page (the provided config assumes):
   - **Page Name:** `MSP Rewind`
   - **Path:** `msp-rewind`
3. With that page open, click the **⋮ (three dots)** menu in the top-right.
4. Select **Load Page State**.
5. Upload the `pageState.json` file from this repo.
6. Confirm the load when prompted.

### 2. Add the workflow to the page in AppBuilder

1. In the same page, open **Settings → Page Settings**.
2. Scroll to **Workflows (array)**.
3. Add the workflow **General – MSP Rewind** to the list.
4. Publish the page.

This will apply the pre-built layout and components that expect the `General – MSP Rewind` workflow output and ensure the page is powered by this workflow.
