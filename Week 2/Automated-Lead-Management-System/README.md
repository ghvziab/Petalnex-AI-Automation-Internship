# Automated-Lead-Management-System

## Problem Statement
Sales and marketing teams often struggle with manually sorting through incoming lead data, standardizing formatting, determining lead quality, and assigning immediate actions. This manual approach leads to delayed responses for high-value prospects, inefficient data tracking, and a lack of organized segmentation for future marketing efforts.

## Objective
The objective of this project is to automate the ingestion, validation, scoring, and routing of incoming leads. It aims to ensure high-priority prospects immediately trigger notifications to the team, while systematically segmenting all leads into appropriate categories (Viable, Nurture, Rejected) for streamlined data tracking and future action.

## Workflow Architecture
1. **Trigger:** The workflow begins with a Webhook receiving lead data via a POST request.
2. **Data Cleaning:** A custom code node flattens and normalizes the input payloads (handling both arrays and objects).
3. **Validation:** An IF condition routes valid leads forward and sends malformed leads (missing name, invalid email, missing service) to a "Rejected" path.
4. **Scoring & Enrichment:** Valid leads run through a JavaScript engine that normalizes text (titlecase for names, lowercase for emails) and assigns a numerical score based on budget, email domain (business vs. free), and company presence. Based on the score, a priority of HIGH, MEDIUM, or LOW is assigned.
5. **Routing & Delivery:**
   * **HIGH Priority:** Sends a Slack message to a designated channel, then merges with Medium priority leads.
   * **MEDIUM Priority:** Merges directly with High priority leads. Both are appended to a "Viable Leads" Google Sheet.
   * **LOW Priority:** Appended to a "Nurture" Google Sheet.
   * **Rejected Leads:** Appended to a "Rejected" Google Sheet.

## Technologies Used
* **n8n:** Workflow automation tool.
* **JavaScript:** Used for custom data cleaning and lead scoring logic.
* **Google Sheets:** Used as a database to store and segment incoming leads. (Note: You may also maintain offline backups or initial imports using files like Leads.xlsx).
* **Slack:** Used for real-time team notifications.

## Nodes Used
* Webhook
* Input Cleaning (Code in JavaScript)
* If
* Code in JavaScript (Scoring)
* Switch
* Send a message (Slack)
* Merge
* Append row in sheet (Google Sheets - Viable Leads)
* Append row in sheet1 (Google Sheets - Nurture)
* Append row in sheet2 (Google Sheets - Rejected)

## Setup Instructions
1. Import the provided `Automated-Lead-Management-System.json` workflow into your n8n instance.
2. Open the **Webhook** node and note your Test and Production URLs for receiving POST requests.
3. Authenticate the required Slack and Google Sheets credentials in n8n.
4. Prepare your target Google Spreadsheet (named "Leads") with the following distinct sheets: "Viable Leads", "Nurture", and "Rejected". Alternatively, you can use the data structure from your local `Leads.xlsx` file to map headers accordingly.
5. In the **Send a message** Slack node, ensure the channel ID (currently set to `C0BMFP7TL3E`) points to your desired target channel.
6. Save and activate the workflow.

## Credentials Required
* Slack account 2
* Google Sheets account

## Workflow Explanation
* **Ingestion & Cleaning:** A Webhook receives lead data via POST request. The "Input Cleaning" JavaScript code node processes the payload to ensure that arrays and single objects are parsed properly for downstream nodes.
* **Validation:** An `If` node checks that `name` exists, `email` contains "@", and `service` exists. Invalid records are immediately routed to the "Rejected" Google Sheet.
* **Enrichment & Scoring:** A JavaScript node standardizes names to titlecase and emails to lowercase. It assigns a score out of 100 based on budget criteria (e.g., `>= 10000` adds 45 points), email domain type (non-free domains score higher), and whether a company name is provided. Priorities are assigned based on score: HIGH (`>=70`), MEDIUM (`>=40`), LOW (`<40`).
* **Routing:** A `Switch` node routes leads based on their assigned priority. HIGH leads trigger a Slack alert containing the lead's name, email, service, and budget. HIGH and MEDIUM leads are merged and appended to the "Viable Leads" sheet. LOW leads go straight to the "Nurture" sheet.

## Test Cases
1. **High Priority Lead:** Send a POST request with a corporate email (e.g., `user@enterprise.com`), budget of `$15,000`, company name, name, and service. 
   * *Expected Result:* Slack notification sent; row appended to "Viable Leads" sheet.
2. **Medium Priority Lead:** Send a POST request with a free email (e.g., `user@gmail.com`), budget of `$5,000`, name, and service. 
   * *Expected Result:* No Slack notification; row appended to "Viable Leads" sheet.
3. **Low Priority Lead:** Send a POST request with a free email, budget of `$500`, name, and service. 
   * *Expected Result:* Row appended to "Nurture" sheet.
4. **Invalid Lead:** Send a POST request missing an email or missing the `@` symbol in the email string. 
   * *Expected Result:* Row appended to "Rejected" sheet.

## Error Handling
* **Payload Structure Handling:** The "Input Cleaning" node includes conditional logic to process both arrays and standard object structures, preventing loop errors if multiple leads are batched into one webhook call.
* **Missing Data:** Leads missing critical validation parameters fallback gracefully via the False branch of the `If` node to the "Rejected" path rather than crashing the workflow.

## Known Limitations
* **Hardcoded Scoring Logic:** The scoring weights (e.g., +45 for budget >=10000) are hardcoded into a JavaScript node. Any changes to the scoring model require editing the actual code.
* **Static Domain Checks:** The script relies on a hardcoded array of free email domains (`gmail.com`, `yahoo.com`, `outlook.com`, `hotmail.com`, `aol.com`, `icloud.com`, `live.com`). It will not catch less common free or disposable email providers.

## Future Improvements
* **CRM Integration:** Output data directly to a CRM system (like HubSpot or Salesforce) instead of relying strictly on Google Sheets or offline files like `Leads.xlsx`.
* **Dynamic Validation API:** Integrate a third-party email validation/enrichment API (like Clearbit or ZeroBounce) for more accurate domain checking and auto-filling missing company details.
* **Deduplication:** Add a node to cross-reference incoming emails with the existing database to prevent processing duplicate leads multiple times.
