<!-- converted from ImportDefaultWorkflowTemplates.xlsx -->

## Sheet: Workflow Templates
| # | Workflow Category - select from drop down | Default Template
(Executed when Deal is created)
(Default: No) | Workflow Template Name | Workflow Template Description | Workflow Instance Naming Convention | Trigger Name | Trigger Description | Conditions |  |  | Actions |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|  |  |  |  |  |  |  |  | Field | Value | Condition (AND/OR) |  |
| 1 | Document workflow | No | Verify automated data extraction result | The automated data extraction has been successfully completed by AI. This template creates a task for the user to review and approve the result of the automated extraction. | Extraction_{Document_Type}_{File_Name} | New file in Data Room | A new file or a new verson of the file has been uploaded or linked to the Data Room | Extraction Status | Completed | AND | Task 1 |
|  |  |  |  |  |  |  |  | Extraction | Automated |  |  |
| 2 | Document workflow | No | Perform manual data extraction | The document is not yet supported by the automated data extraction process. This template creates a task for the user to perform the manual extraction. | Extraction_{File_Name} | New file in Data Room | A new file or a new verson of the file has been uploaded or linked to the Data Room | Extraction Status | Failed | AND | Task 2 |
|  |  |  |  |  |  |  |  | Extraction | Automated |  |  |
| 3 | Deal workflow | Yes | Set up a Deal | A new deal has been created. This template is assigned by default to each deal and has the list of steps to be done to set up the deal. | Setup_{Deal_Type}_{Deal_Name} | New deal | A default workflow for every newly created deal | Deal | Created |  | Task 3 |
| 4 | Document workflow | No | Verify automated data extraction result | The automated data extraction has been successfully completed by AI. This template creates a task for the user to review and approve the result of the automated extraction. | Extraction_{Document_Type}_{File_Name} | New file in Data Room | A new file or a new verson of the file has been uploaded or linked to the Data Room | Extraction Status | Completed | AND | Task 4 |
|  |  |  |  |  |  |  |  | Extraction | Automated |  |  |
| 5 | Document workflow | No | Perform manual data extraction | The document is not yet supported by the automated data extraction process. This template creates a task for the user to perform the manual extraction. | Extraction_{File_Name} | New file in Data Room | A new file or a new verson of the file has been uploaded or linked to the Data Room | Extraction Status | Failed | AND | Task 5 |
|  |  |  |  |  |  |  |  | Extraction | Automated |  |  |
| 6 | Deal workflow | Yes | Set up a Deal | A new deal has been created. This template is assigned by default to each deal and has the list of steps to be done to set up the deal. | Setup_{Deal_Type}_{Deal_Name} | New deal | A default workflow for every newly created deal | Deal | Created |  | Task 6 |
## Sheet: Task Details
| # | Task Naming Convention | Task Description | Due Date (In Days) | Responsible |  | Approver |  | Informed |  | Check-Lists |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|  |  |  |  | Name | Assignee Type | Name | Assignee Type | Name | Assignee Type |  |
| Task 1 | {Document_Type}_Verify data extraction_{File_Name} | This task is created for a new file or new version of the file appeared in the Data Room.
The file has been successfully categorized (document type is defined) and automatically extracted.

Please go to the attachment, review the extracted data, correct it where it's needed.

Responsible party to complete the first check-list.
Approver to complete the second check-list. | 2 | Team Member | Role | Manager | Role |  |  | Data extract verification (for Responsible):
[] Open an attachment
[] Review the document and its extracted data
[] Correct mistakes if any
[] Save changes if any
[] Mark task as "Review" when it's ready for approval and comment below "ready for approval" tagging the Approver

Data extract approval (for Approver):
[] If approved, check this box and mark task as "Done" and comment below "approved" tagging the Responsible Party
[] If edits are required, move status to "In progress" and comment below with the corrections tagging the Responsible Party |
| Task 2 | Manual data extraction_{File_Name} | This task is created for a new file or new version of the file appeared in the Data Room.
The file has failed to be categories and/or extracted automatically.

Please go to the attachment, review, define its document type and initiate the manual extraction.

Responsible party to complete the first check-list.
Approver to complete the second check-list. | 2 | Team Member | Role | Manager | Role |  |  | Data extraction (for Responsible):
[] Open an attachment
[] Review the document and define its document type
[] Verify that the extraction template is available to perform a manual extraction for this document type
[] Go to the Data Room and click to initiate the manual extraction flow
[] Open the file, highlight data points in the document for each field requires to be extracted
[] Save changes
[] Mark task as "Review" when it's ready for approval and comment below "ready for approval" tagging the Approver

Data extract approval (for Approver):
[] If approved, check this box and mark task as "Done" and comment below "approved" tagging the Responsible Party
[] If edits are required, move status to "In progress" and comment below with the corrections tagging the Responsible Party |
| Task 3 | Set up a Deal_{Deal_Name} | This task is created for a newly created Deal. It includes settings to be done around a deal.
Please go to the check-lists for more details and complete them to complete this task. | 5 | User(Deal Creator) | User |  |  |  |  | Data room connection:
[] Open Data room, connect data room to the folder from the availabale storage providers.
Pay attention: the connected folder can be "watched". It means all files from the provider side folder will be available and viewable in this deal's data room. If it's "unwatched", you can link separate files to the data room manually once the connect is set up.
[] Create folders and subfolders in the data room to organize the documents, for example, based on the financial period, based on the document purpose, based on the document type and so on.
[] Link files from the storage provider folder to the data room folder.

Select workflow templates:
[] Go to the Deal Settings and Workflows, add workflow templates that are applicable for this deal to be triggerd automatically. You can select multiple options.

Invite users to the deal:
[] Go to the Deal settings and User management there. Add users who are going to work and communicate around this deal, assign appropriate deal roles to them.
Pay attention: you need to have all the roles in your deal team that are required by workflow templates that you selected. This will ensure that automatically cretaed tasks have assignees. |
| Task 4 | {Document_Type}_Verify data extraction_{File_Name} | This task is created for a new file or new version of the file appeared in the Data Room.
The file has been successfully categorized (document type is defined) and automatically extracted.

Please go to the attachment, review the extracted data, correct it where it's needed.

Responsible party to complete the first check-list.
Approver to complete the second check-list. | 2 | Team Member | Role | Manager | Role |  |  | Data extract verification (for Responsible):
[] Open an attachment
[] Review the document and its extracted data
[] Correct mistakes if any
[] Save changes if any
[] Mark task as "Review" when it's ready for approval and comment below "ready for approval" tagging the Approver

Data extract approval (for Approver):
[] If approved, check this box and mark task as "Done" and comment below "approved" tagging the Responsible Party
[] If edits are required, move status to "In progress" and comment below with the corrections tagging the Responsible Party |
| Task 5 | Manual data extraction_{File_Name} | This task is created for a new file or new version of the file appeared in the Data Room.
The file has failed to be categories and/or extracted automatically.

Please go to the attachment, review, define its document type and initiate the manual extraction.

Responsible party to complete the first check-list.
Approver to complete the second check-list. | 2 | Team Member | Role | Manager | Role |  |  | Data extraction (for Responsible):
[] Open an attachment
[] Review the document and define its document type
[] Verify that the extraction template is available to perform a manual extraction for this document type
[] Go to the Data Room and click to initiate the manual extraction flow
[] Open the file, highlight data points in the document for each field requires to be extracted
[] Save changes
[] Mark task as "Review" when it's ready for approval and comment below "ready for approval" tagging the Approver

Data extract approval (for Approver):
[] If approved, check this box and mark task as "Done" and comment below "approved" tagging the Responsible Party
[] If edits are required, move status to "In progress" and comment below with the corrections tagging the Responsible Party |
| Task 6 | Set up a Deal_{Deal_Name} | This task is created for a newly created Deal. It includes settings to be done around a deal.
Please go to the check-lists for more details and complete them to complete this task.. | 5 | User(Deal Creator) | User |  |  |  |  | Data room connection:
[] Open Data room, connect data room to the folder from the availabale storage providers.
Pay attention: the connected folder can be "watched". It means all files from the provider side folder will be available and viewable in this deal's data room. If it's "unwatched", you can link separate files to the data room manually once the connect is set up.
[] Create folders and subfolders in the data room to organize the documents, for example, based on the financial period, based on the document purpose, based on the document type and so on.
[] Link files from the storage provider folder to the data room folder.

Select workflow templates:
[] Go to the Deal Settings and Workflows, add workflow templates that are applicable for this deal to be triggerd automatically. You can select multiple options.

Invite users to the deal:
[] Go to the Deal settings and User management there. Add users who are going to work and communicate around this deal, assign appropriate deal roles to them.
Pay attention: you need to have all the roles in your deal team that are required by workflow templates that you selected. This will ensure that automatically cretaed tasks have assignees. |
## Sheet: Validations
| Allowed Fields | Allowed Values | Allowed Logical Operator | Allowed Roles | Allowed WF Categories | RACI Assignee type | Default Template |
| --- | --- | --- | --- | --- | --- | --- |
| Extraction Status | Automated | AND | Administrator | Document workflow | Role | Yes |
| Extraction | Created | OR | Team Member | Deal workflow | User | No |
| Deal | Completed |  | Manager |  |  |  |
| Document Type | Not Supported |  | Guest |  |  |  |
|  | Failed |  | User(Deal Creator) |  |  |  |
|  | Rent Roll |  |  |  |  |  |
|  | Limited Partnership Agreement |  |  |  |  |  |
|  | Appraisal |  |  |  |  |  |
|  | Capital Call |  |  |  |  |  |
|  | Credit Agreement |  |  |  |  |  |
|  | Income Statement |  |  |  |  |  |
|  | Partners Capital Account Statement |  |  |  |  |  |
|  | Schedule of Investments |  |  |  |  |  |
|  | PA Agent Regulations |  |  |  |  |  |
|  | Balance Sheet |  |  |  |  |  |
|  | None |  |  |  |  |  |