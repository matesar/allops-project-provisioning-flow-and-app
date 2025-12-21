## General architecture

High-level flow:

1. **Trigger**  
   - Trigger: **When an item is created** in a SharePoint list (functional form hosted inside the Power App within the same Solution).  
   - The form is completed by the **Project Manager** or project owner.  
   - Fields to be filled in:
     - Project name  
     - Organizational area  
     - Fiscal year  
     - Plant  
   - The **PARENT** flow assigns a unique ID to the record for each country (by counting existing items and adding 1, with retry logic to avoid collisions).

2. **Initialization of configuration variables**  
   - A configuration object `cfg` is initialized with:
     - Base SharePoint paths where the project folders will be created.  
     - Link to the project record in the ALLOPS site.  
     - Template URLs (Gantt, MS Project, cost tracking, etc.).  
   - `varError` is initialized to record where a failure occurs in the flow (each main block is wrapped in a **Scope** to simplify error handling).

3. **Creation of document structure**  
   - HTTP action to SharePoint:  
     `POST /_api/site/CreateCopyJobs?copy=true`  
   - From a **templates** folder (model structure), it copies:
     - The project root folder.  
     - Subfolders by area (e.g. EHSQ, Engineering, Finance, etc.).  
     - Base files: cost tracking workbook, MS Project template, instructions, etc.  
   - The structure is created under a project-specific path, for example:  
     `/Shared Documents/ALLOPS/CPX/<Year>/<ProjectID>_<Name>/…`

4. **Update / creation of item in master list**  
   - A record is updated or created in a **master projects list** (e.g. `AR Projects Master list`).  
   - It stores:
     - Key project metadata (ID, country, plant, owner, fiscal year, estimated amount, initial status, etc.).  
     - Generated links: project root folder, cost tracking, Gantt, dashboard, Teams channel, etc.  
   - This list acts as the **single source of truth** to check the status and links of any CapEx project in the company’s LATAM region.

5. **Teams channel creation and automatic email to owner**

   - Using Microsoft Graph, a **Teams channel** is created for the project (in a specific ALLOPS Team) and a welcome message is posted with the main links.  
   - An email is sent to the project owner:
     - Dynamic subject (example):  
       `@{triggerBody()?['text_3']}_@{triggerBody()?['text_1']}_@{triggerBody()?['text_4']}`  
       > This combines project name, country/company and an internal ID.  
     - HTML body with:
       - Personalized greeting using the PM’s **first name**.  
       - Main link to the project in SharePoint.  
       - Links to instructions (RPA, MS Project sync, cost tracking update, etc.).  
       - Direct links to:
         - Gantt / MS Project  
         - Cost tracking workbook  
         - Monitoring dashboard  
         - Project shared folder  
         - “Blue folder” (documentation accessible to non-technical areas, if applicable)

6. **Logging and basic error handling**

   - Each critical block (SharePoint copies, list updates, channel creation, email sending) runs inside a **Scope**.  
   - If any action fails:
     - `varError` is updated indicating in which stage the error occurred.  
     - Details are logged (for example in a logs list or in the project item itself).  
     - Optionally, an alert email can be sent to the Engineering / IT team.
