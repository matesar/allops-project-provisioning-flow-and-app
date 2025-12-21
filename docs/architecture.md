# Architecture – CapEx Project Provisioning (ALLOPS)

This document describes the technical architecture of the CapEx project provisioning solution in the **ALLOPS** environment, built on **Power Automate**, **Power Apps**, **SharePoint Online**, **Power BI**, **Outlook 365** and **Microsoft Teams**.

---

## 1. Scope and overview

When a user creates a new CapEx project through a SharePoint form or the Power Apps app, the system:

1. Registers the request in a SharePoint list.
2. Assigns a unique identifier to the project.
3. Creates the standard document structure from templates.
4. Updates a **master projects list** with key metadata and links.
5. (Optionally) creates a Teams channel for the project.
6. Sends an automatic email to the project owner with all relevant links.
7. Exposes project information and Power BI dashboards through a Power Apps front end.

The process is split into two Power Automate flows plus a Power Apps canvas app:

- **PARENT flow**: `CPXinitializationARParent-…`
- **CHILD flow**: `CPXProvisioningChild-…`
- **Power Apps app**: `CapexProjectInitApp.msapp`

---

## 2. Main components

### 2.1 SharePoint Online

1. **ALLOPS site**  
   SharePoint/Teams site where CapEx projects are centralized.

2. **Project request / project list** (initial creation)  
   - Source of the trigger for the PARENT flow.  
   - Typical columns:
     - `CompanyISOCod` (country/company)
     - `ProjectName` (project name)
     - `InternalID` (internal CPX ID or similar)
     - `Requester` (requesting person / PM)
     - `Complexity` (Low / Medium / High)
     - `FiscalYear`
     - Additional organization-specific fields.

   > In some implementations this is already the **master projects list**; in others there is a separate “requests” list and a “master” list.

3. **Master projects list**  
   Example name: `Projects Master list`  
   Stores consolidated project information once provisioning is complete:
   - Unique project ID (ALLOPS Project ID / CPX ID).
   - Metadata (country, plant, owner, FY, estimated amount, etc.).
   - Links to:
     - Project root folder.
     - Cost tracking file.
     - Gantt / task list / MS Project.
     - Power BI dashboard.
     - Shared / “blue” folder.
     - Teams channel.

4. **Template document library**  
   Example path:  
   `/Shared Documents/Templates/ALLOPS/CPX/Issued/Latest/`  
   Contains the “model” structure of folders and files:
   - Subfolders by area (EHSQ, Engineering, Finance, etc.).
   - Cost tracking workbook.
   - MS Project template.
   - User guides (PDF/Word).
   This library is the **source** for the `CreateCopyJobs` API.

---

### 2.2 Power Automate (cloud flows)

1. **PARENT flow**  
   File: `CPXinitializationARParent-….json`  

   **Role:**
   - Receive the project request from SharePoint.
   - Validate minimum data.
   - Calculate/generate the unique project ID.
   - Prepare a configuration object with all needed parameters.
   - Call the CHILD flow, passing project data.

2. **CHILD flow**  
   File: `CPXProvisioningChild-….json`  

   **Role:**
   - Create the project document structure using `CreateCopyJobs`.
   - Update the master projects list with generated links and metadata.
   - Create a Teams channel and welcome message (if enabled).
   - Send the automatic email to the project owner.
   - Log errors and return status.

3. **Main connectors**
   - `SharePoint` (standard actions + HTTP calls to `_api`).
   - `Office 365 Outlook` (email sending).
   - `HTTP` (Microsoft Graph for Teams).
   - Standard Power Automate actions for scopes, variables, conditions, loops, etc.

---

### 2.3 Power Apps canvas app

**File:** `powerapps/CapexProjectInitApp.msapp`  

Canvas application used by PMs and the engineering team as the main front end.

**Data sources:**

- SharePoint **master projects list** (main data source).
- Optionally:
  - SharePoint task lists / Gantt lists per project.
  - Additional lists (e.g., reference tables, complexity, plants, etc.).

**Key concepts:**

- Horizontal **navigation bar** (gallery or buttons) acting as “tabs” that switch between screens:
  - *New project* (minimal required fields).
  - *Project details* (extended fields).
  - *Dashboards* (embedded Power BI).
  - *Tasks / Gantt* (SharePoint task list).
  - *Links & resources* (shortcuts to documents, folders, etc.).

- The app uses the same SharePoint list as the flows, so there is no data duplication: a project created in the app is immediately visible to the flows and vice versa.

Details in section **6. Power Apps app – screens and behaviour**.

---

### 2.4 Microsoft Teams / Graph

- **ALLOPS Team** (the Team where project channels are created).  

The CHILD flow uses **Microsoft Graph** to:

- Create one **channel per project**:
  - `POST /teams/{team-id}/channels`
  - Channel name based on project ID + project name.

- Post a **welcome message**:
  - `POST /teams/{team-id}/channels/{channel-id}/messages`
  - Message includes description and main links (SharePoint, cost tracking, dashboard, etc.).

---

### 2.5 Outlook 365

Used by the CHILD flow to send an email to the PM:

- Personalized greeting (first name).
- Project summary.
- Links to:
  - Project site in SharePoint.
  - Root / shared folder.
  - Cost tracking.
  - Gantt / MS Project or task list.
  - Dashboard.
  - Teams channel.
- Links to user guides (RPA, sync, etc.) where applicable.

---

### 2.6 Power BI

- Dashboards showing:
  - Cost progress (committed / spent).
  - Overall CapEx status.
  - Consolidated project view.

- Selected **tiles** are embedded in the Power Apps app and filtered by project ID (CPX ID) using URL filter parameters (`&filter=...`).

---

## 3. End-to-end process (high level)

1. **New project creation**
   - The PM fills in a form:
     - directly on the SharePoint list, **or**
     - through the Power Apps app (simplified “New project” screen).
   - A new item is created with the minimum required fields.

2. **PARENT flow execution**
   - Triggered by the new item in SharePoint.
   - Validates key data.
   - Generates a **unique project ID**.
   - Updates the source item with that ID (for traceability).
   - Calls the CHILD flow with all necessary parameters.

3. **Provisioning (CHILD flow)**
   - Creates the document structure from templates using `CreateCopyJobs`.
   - Creates or links Gantt/task lists and cost tracking files.
   - Updates the master projects list with all generated links and metadata.
   - Creates the Teams channel and welcome message (if enabled).

4. **Notification**
   - Sends an automatic email to the PM containing:
     - Project summary.
     - Main ALLOPS link.
     - Links to user guides and dashboards.

5. **Front end usage (Power Apps)**
   - PM and team use the app to:
     - See project details.
     - Open all related resources from a single place.
     - View Power BI dashboards already filtered by project.
     - Navigate to Gantt / task lists and cost tracking files.

---

## 4. PARENT flow – detailed behaviour

### 4.1 Trigger

- **Trigger:** `When an item is created` (SharePoint).
- Source: project requests / master projects list.
- Form fields are read from `triggerBody()`.

### 4.2 Validation and ID assignment

Typical steps:

1. **Read key fields**
   - Country / company.
   - Project name.
   - Requester (email).
   - Internal ID (if provided).
   - Fiscal year, plant, etc.

2. **Calculate unique project ID**
   - Query the master projects list.
   - Count existing items or read the last sequence.
   - Generate a sequential ID (e.g. `AR-2025-0012` or `2025ARIT083`).
   - Implement retry logic (`Do until` / conditions) to avoid collisions.

3. **Update request item**
   - Write back the generated ID to the original list item.

### 4.3 Call to CHILD flow

- Action: **Run a child flow**.
- Typical parameters:
  - Unique project ID.
  - Request fields (country, plant, name, owner, FY, etc.).
  - Source/destination paths for templates.
  - Email addresses of PM and stakeholders.
  - Flags for “create Teams channel”, “send email”, etc. (if used).

---

## 5. CHILD flow – detailed behaviour

### 5.1 Input parameters

The CHILD flow expects at minimum:

- `ProjectId` (unique ID from the PARENT).
- `ProjectName`.
- `CountryCompany`.
- `RequesterEmail` / `PmEmail`.
- `FiscalYear`.
- SharePoint paths:
  - Template library.
  - Project destination library/root folder.
- URLs of guides (if used in the email).
- Optional flags: create Teams channel, send email, etc.

### 5.2 Document structure creation (`CreateCopyJobs`)

1. **Build JSON payload**
   - Request body for `CreateCopyJobs` includes:
     - `exportObjectUris`: source paths (template folders/files).
     - `destinationUri`: project-specific destination folder.

2. **HTTP call**
   - Method: `POST`
   - URI: `/_api/site/CreateCopyJobs?copy=true`
   - Headers:
     - `Accept: application/json;odata=nometadata`
     - `Content-Type: application/json;odata=nometadata`

3. **Result**
   - Retrieve the URL of the project root folder and relevant subfolders to be used in next steps and stored in the master list.

### 5.3 Teams channel and welcome message

1. **Create channel**
   - HTTP call to Microsoft Graph:
     - `POST /teams/{team-id}/channels`
   - Channel name typically uses the project ID and project name.

2. **Post welcome message**
   - `POST /teams/{team-id}/channels/{channel-id}/messages`
   - Content:
     - Short project description.
     - Links to root folder, cost tracking, dashboard, etc.

### 5.4 Master list update

- Update or create the corresponding item in the **master projects list** with:
  - Metadata from the request.
  - Root folder URL.
  - Gantt / tasks list URL.
  - Cost tracking URL.
  - Dashboard URL.
  - Teams channel URL.

### 5.5 Automatic email

- Action: `Send an email (V2)` (Outlook).
- Subject: dynamic, often including key fields (e.g. country, plant, project name).
- HTML body:
  - Greeting using the PM’s first name.
  - Project summary (ID, name, plant, FY).
  - List of links (SharePoint, cost tracking, Gantt, dashboard, Teams).
  - Links to user guides.

---

## 6. Power Apps app – screens and behaviour

### 6.1 Data connections

- Main data source: SharePoint **master projects list**.
- Optional additional data sources:
  - Gantt / task lists per project.
  - Lookup/reference lists (plants, complexity levels, etc.).

### 6.2 Navigation model

- A **horizontal navigation bar** (gallery or set of buttons) at the top of the app acts as a tab control:
  - Each button updates a variable (e.g. `varCurrentTab`) and calls `Navigate()` to the corresponding screen.
  - Example tabs:
    - `New project`
    - `Project details`
    - `Dashboards`
    - `Tasks / Gantt`
    - `Resources`

### 6.3 Core screens

1. **New project screen (simplified form)**
   - Shows only a **reduced set of mandatory fields**, e.g.:
     - Project name
     - Country / company
     - Fiscal year
     - Complexity
     - PM / requester
   - Uses a SharePoint form control in **New** mode bound to the master projects list.
   - *OnSuccess*: optionally:
     - Navigate to the **Project details** screen with the newly created record selected.

2. **Project details screen (extended form)**
   - Full SharePoint form in **Edit** mode with additional fields:
     - Budget, scope, plant, category, etc.
     - URL columns for cost tracking, Gantt, dashboards, “blue folder”, etc.
   - To avoid showing “dummy” values (e.g. `.` used for JSON-formatted hyperlink columns), the app:
     - Hides such columns in the form, and
     - Uses **icons or images** with `OnSelect` opening the corresponding URL (`Launch()`).

3. **Dashboards screen (embedded Power BI)**
   - Embeds a Power BI tile using:
     - The **Power BI tile control**, or
     - An **HTML text**/image control with the embed URL.
   - The URL includes a `filter=` parameter that uses the selected project’s CPX ID (e.g. variable `varCPXId`) to display metrics for that project only.
   - The selected project is usually taken from:
     - A gallery bound to the master list, **or**
     - The current form item.

4. **Tasks / Gantt screen**
   - Connects to a SharePoint task list (or multiple lists) where the project Gantt is stored.
   - Can show:
     - A gallery/table view of tasks.
     - Links to open the native SharePoint Gantt view or MS Project file.

5. **Resources / links screen**
   - Central place with clickable icons/images linking to:
     - Project root folder.
     - Shared / “blue” folder.
     - Cost tracking file.
     - Project guides and templates.
   - Links are taken from the master projects list columns.

### 6.4 Reuse and extension

- When customizing forms for other lists, screens can be **copied** between apps and re-bound to new data sources while keeping layout and navigation patterns.
- The tab navigation logic and the pattern of “hide dummy field, show icon with Launch()” can be reused across screens.

---

## 7. Data model (summary)

### 7.1 Project form / master list (minimal recommended fields)

- `CapExName` (project name)
- `CPX ID#` (primary key / unique project ID)
- `Country` / `CompanyISOCod`
- `Plant`
- `Requester` / `PmEmail`
- `FiscalYear`
- `Complexity`
- `CreatedBy`
- Status fields (e.g. `Status`, `ProvisioningStatus`)

### 7.2 Link / resource fields

Typical URL columns:

- `ProjectFolderUrl`
- `CostTrackingUrl`
- `GanttUrl` or `TasksListUrl`
- `DashboardUrl`
- `TeamsChannelUrl`
- `SharedFolderUrl` (blue folder)

These fields are:

- **Populated by the CHILD flow** during provisioning.
- **Consumed by the Power Apps app** to open resources and filter dashboards.

---

## 8. Error handling

- Each major block in the flows is wrapped inside a **Scope**:
  - `Scope – Create structure`
  - `Scope – Update master list`
  - `Scope – Create Teams channel`
  - `Scope – Send email`

- On failure:
  - A variable such as `varError` is updated with the name of the failing block.
  - The flow can:
    - Log the error in a dedicated SharePoint log list.
    - Update the master list item with an error status.
    - Send an alert email to the support team.

The Power Apps app can optionally read the error/status fields from the master list and display them to the user.

---

## 9. Security and permissions

The account(s) used by the flows must have:

- **Edit** permissions on:
  - Master projects list.
  - Template library.
  - Project destination library.
- Permissions to:
  - Send email (Outlook).
  - Create channels and post messages in the ALLOPS Team (Graph / Teams).

Recommendations:

- Use a dedicated **service account** for automation instead of a personal account.
- Restrict who can create new projects in the app and/or list through SharePoint permissions and Power Apps sharing settings.

---

## 10. Parameterization and adaptation

To adapt the solution to another country/site:

- Adjust SharePoint paths:
  - Site URL.
  - Template library.
  - Project destination library.
- Update list names:
  - Requests / master list.
- Review and update guide URLs.
- Adapt the unique ID format (country prefixes, fiscal year, etc.).
- In Power Apps:
  - Point data sources to the new lists.
  - Adjust tab labels and visible fields according to local requirements.
- In Power BI:
  - Ensure filters use the correct table/column names for the CPX ID and project keys.

---
