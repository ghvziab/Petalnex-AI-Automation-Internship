# Candidate Screening Automation

## Problem Statement
Recruitment teams often face the time-consuming and repetitive task of manually filtering through candidate data to determine who meets specific role requirements. Manually verifying degrees, counting matching skills, calculating years of experience, and sending individual follow-up emails creates bottlenecks and increases the risk of human error.

## Objective
The objective of this workflow is to automate the initial screening of job applicants. It retrieves applicant data, validates the presence of required information, evaluates candidates against predefined technical and educational criteria, updates their status in the tracking system, and automatically emails shortlisted candidates to invite them to the next stage of the interview process.

## Workflow Architecture
The workflow follows a linear pipeline structure with conditional filtering:
1. **Trigger**: The workflow is initiated via a manual trigger.
2. **Extraction**: Candidate data is read from a linked Google Sheet.
3. **Validation**: A filter node ensures that no critical fields (Name, Email, Degree, Skills, Experience, Availability) are empty.
4. **Evaluation**: Custom JavaScript logic assigns a status of "Shortlisted", "Review Required", or "Rejected" based on specific criteria.
5. **Rejection Filtering**: Candidates marked as "Rejected" are filtered out of the subsequent update process.
6. **Data Update**: The original tracking spreadsheet is updated with the new statuses for the remaining candidates.
7. **Notification Pipeline**: A final filter isolates "Shortlisted" candidates and triggers an automated Gmail message containing interview next steps.

## Technologies Used
* **n8n**: Primary workflow automation and orchestration platform.
* **Google Sheets API**: Used as the database to read candidate records and update their evaluated statuses.
* **JavaScript**: Used for custom business logic to parse skills and evaluate candidate qualifications.
* **Gmail API**: Used to dispatch automated emails to successful applicants.

## Nodes Used
* **When clicking ‘Execute workflow’** (`n8n-nodes-base.manualTrigger`): Manually starts the execution.
* **Get row(s) in sheet** (`n8n-nodes-base.googleSheets`): Fetches data from the "Candidates" sheet.
* **Data Validation** (`n8n-nodes-base.filter`): Checks that essential fields are not empty.
* **Code in JavaScript** (`n8n-nodes-base.code`): Executes the screening algorithm.
* **Rejection filtering** (`n8n-nodes-base.filter`): Passes only "Review Required" or "Shortlisted" statuses.
* **Append or update row in sheet** (`n8n-nodes-base.googleSheets`): Maps back to the sheet using the Email field and updates the Status.
* **Filter** (`n8n-nodes-base.filter`): Isolates candidates strictly marked as "Shortlisted".
* **Send a message** (`n8n-nodes-base.gmail`): Sends the standardized HR interview invitation email.

## Setup Instructions
1. Import the workflow JSON into your n8n instance.
2. Set up the required connections for Google Sheets and Gmail.
3. Ensure your data source, **candidate_screening_dataset.xlsx**, is formatted correctly with the following column headers: `Name`, `Email`, `Degree`, `Skills`, `Experience`, and `Availability`.
4. Verify that the Google Sheets nodes are pointing to the correct Document ID and Sheet ID for your specific Google Drive environment.
5. Modify the custom JavaScript in the **Code in JavaScript** node if you need to adjust the required experience, degrees, or skills for a different role.
6. Activate the workflow or use the manual execution button to run it.

## Credentials Required
* Google Sheets account
* Gmail account

## Workflow Explanation
1. Upon clicking execute, the **Get row(s) in sheet** node pulls all applicant rows from the "Candidates" sheet of the provided dataset.
2. The **Data Validation** node inspects the incoming JSON payload. It drops any records where `Name`, `Email`, `Degree`, `Skills`, `Experience`, or `Availability` are empty.
3. The **Code in JavaScript** node takes the validated dataset and applies the screening logic:
   * **Shortlisted:** Requires >= 3.0 years of experience, a specific degree ("MS Artificial Intelligence", "BS Data Science", or "MS Software Engineering"), and at least 2 matching skills ("Python", "TensorFlow", "PyTorch").
   * **Review Required:** Requires >= 0 years of experience and at least 1 matching skill.
   * **Rejected:** Assigned by default to anyone who fails the above conditions.
4. The **Rejection filtering** node drops all "Rejected" profiles, allowing only "Review Required" and "Shortlisted" candidates to proceed.
5. The **Append or update row in sheet** node takes these remaining profiles and maps them back to "Sheet1" using their `Email` to update their `Status` column.
6. A secondary **Filter** node isolates only the profiles with a "Shortlisted" status.
7. Finally, the **Send a message** node uses Gmail to send an automated email from "Petalnex Pvt. Ltd." to the shortlisted candidates, addressing them by their `Name`.

## Test Cases
* **Test Case 1 (Ideal Candidate):** A candidate with an "MS Artificial Intelligence" degree, "Python, TensorFlow" as skills, and 4 years of experience. *Expected Result:* Assigned "Shortlisted" status, sheet is updated, and an email is dispatched.
* **Test Case 2 (Borderline Candidate):** A candidate with a "BS Data Science" degree, "Python, Java" as skills, and 1 year of experience. *Expected Result:* Assigned "Review Required" status, sheet is updated, no email is sent.
* **Test Case 3 (Missing Data):** A candidate whose `Email` or `Experience` column is blank. *Expected Result:* Dropped immediately by the Data Validation node.
* **Test Case 4 (Unqualified Candidate):** A candidate with a "BA English" degree, "Word, Excel" as skills, and 0 years of experience. *Expected Result:* Assigned "Rejected" status in the Code node, filtered out by the Rejection filtering node, and no sheet update or email occurs.

## Error Handling
* **Incomplete Data:** Handled explicitly by the **Data Validation** filter node, which securely drops malformed rows before they hit the JavaScript evaluation.
* **Default State assignment:** The JavaScript block safely initializes every candidate to "Rejected" before applying logic, preventing null or undefined status errors.

## Known Limitations
* **Hardcoded Rules:** The required parameters for degrees, skills, and experience are hardcoded directly into the JavaScript node, requiring manual developer intervention to change criteria for different jobs.
* **Unrecorded Rejections:** Because "Rejected" candidates are filtered out at the **Rejection filtering** node, their status is never written back to the Google Sheet.
* **Lack of Explicit Error Nodes:** The workflow does not include a dedicated error trigger to notify administrators if the Gmail or Google Sheets API fails.

## Future Improvements
* **Dynamic Screening Criteria:** Abstract the required skills, degrees, and experience into a separate configuration sheet or workflow variables to make the system modular for different job postings.
* **Rejected Candidate Handling:** Create an alternative branch to update "Rejected" statuses in the spreadsheet and send automated, polite rejection emails.
* **Error Catching:** Implement the n8n Error Trigger node to send a Slack or Microsoft Teams notification to the HR team if the automation fails during execution.
