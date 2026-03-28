# SHIPMENT TRACKER

**Power Apps · Dataverse · Power Automate**

Complete CodeApps / Vibe-Coding Specification
Version 1.0 | March 2026

## 1. Project Overview & Architecture

This document is the complete specification for the Shipment Tracker Power Apps application. It covers the Dataverse schema, every screen's UI/UX design, Power Automate flow definitions, role-based access rules, upload/export logic, and CodeApps vibe-coding prompts for each component.

### 1.1 Technology Stack

| Component     | Technology                   | Purpose                                                  |
| :------------ | :--------------------------- | :------------------------------------------------------- |
| Frontend app  | Power Apps Canvas App        | All UI screens — login, dashboard, CRUD, reports, admin |
| Database      | Microsoft Dataverse          | All persistent data — shipments, users, lookups, audit  |
| Automation    | Power Automate Cloud Flows   | Auth, upload parsing, export generation, notifications   |
| Reports       | Power Apps Charts + Power BI | KPI tiles, charts, advanced filtered reports             |
| File storage  | SharePoint / Dataverse Files | Excel upload templates, exported files                   |
| Maps/tracking | Azure Maps Control           | Live visual shipment tracking dashboard                  |
| Notifications | Outlook connector            | Email alerts on status changes, cutoff reminders         |

### 1.2 Role Hierarchy

| Role      | Permissions                                                                                     | Who                               |
| :-------- | :---------------------------------------------------------------------------------------------- | :-------------------------------- |
| Admin     | Full CRUD on all tables; manage Shipper, Carrier, Consignee, CSO, User tables; upload templates | Operations managers, IT admins    |
| User      | Create & edit shipments in own branch; view all shipments in branch; upload data; export        | Freight coordinators, CSO members |
| Read-Only | View all shipments and reports in own branch; export only                                       | Management, auditors              |

### 1.3 Branch-Level Data Isolation

Every Dataverse query is filtered by BranchID stored in a global app variable set at login. Users from Mumbai cannot see Singapore shipments. Admins can optionally toggle a global view.

---

## 2. Dataverse Table Schema

All tables use standard Dataverse naming conventions (schema prefix: `shp_`). Primary keys are auto-generated GUIDs. Lookup fields reference other tables via foreign key relationships.

### 2.1 Shipment (shp_Shipment) — Core table

| Column name             | Dataverse field      | Type                                   | Required | Notes                                      |
| :---------------------- | :------------------- | :------------------------------------- | :------- | :----------------------------------------- |
| Shipment No             | shp_ShipmentNo       | Text (50)                              | Yes      | Unique index — duplicate check on upload  |
| Branch                  | shp_BranchId         | Lookup → shp_Branch                   | Yes      | Drives data isolation                      |
| Direction               | shp_Direction        | Choice: Import/Export/Both             | Yes      | Controls which detail tab shows            |
| Shipment Status         | shp_Status           | Choice                                 | Yes      | Booked/In Transit/Arrived/Delivered/Closed |
| Cargo Type              | shp_CargoType        | Choice: FCL/LCL/Air/Breakbulk          | Yes      |                                            |
| Carrier Name            | shp_CarrierId        | Lookup → shp_Carrier                  | Yes      |                                            |
| Carrier SCAC Code       | shp_CarrierSCAC      | Text (10)                              | No       | Auto-filled from Carrier lookup            |
| Carrier Booking No      | shp_CarrierBookingNo | Text (50)                              | No       |                                            |
| Shipper Name            | shp_ShipperId        | Lookup → shp_Shipper                  | Yes      |                                            |
| Consignee Name          | shp_ConsigneeId      | Lookup → shp_Consignee                | Yes      |                                            |
| CSO Member              | shp_CSOId            | Lookup → shp_CSOMember                | No       |                                            |
| Consolidation Type      | shp_ConsolType       | Choice: Direct/Consol                  | No       |                                            |
| Consol Number           | shp_ConsolNo         | Text (50)                              | No       |                                            |
| Transport Mode          | shp_TransportMode    | Choice: Sea/Air/Road/Rail              | Yes      |                                            |
| Vessel Name / Voyage    | shp_VesselVoyage     | Text (100)                             | No       |                                            |
| HBL Number              | shp_HBLNo            | Text (50)                              | No       |                                            |
| MBL Number              | shp_MBLNo            | Text (50)                              | No       |                                            |
| POL (Port of Loading)   | shp_POL              | Text (100)                             | No       |                                            |
| POD (Port of Discharge) | shp_POD              | Text (100)                             | No       |                                            |
| Final Destination       | shp_FinalDestination | Text (100)                             | No       |                                            |
| Region                  | shp_Region           | Choice: Asia/Europe/Americas/ME/Africa | No       |                                            |
| ETD                     | shp_ETD              | Date Only                              | No       | Estimated departure                        |
| ETA                     | shp_ETA              | Date Only                              | No       | Estimated arrival                          |
| Shipment Received Date  | shp_ReceivedDate     | Date Only                              | No       |                                            |
| Comments                | shp_Comments         | Text Area (2000)                       | No       |                                            |
| Created By              | shp_CreatedBy        | Lookup → shp_User                     | Auto     | Set by app on create                       |
| Created On              | shp_CreatedOn        | Date & Time                            | Auto     | Set by system                              |
| Modified By             | shp_ModifiedBy       | Lookup → shp_User                     | Auto     |                                            |
| Modified On             | shp_ModifiedOn       | Date & Time                            | Auto     |                                            |

### 2.2 Import Detail (shp_ImportDetail)

One-to-one with shp_Shipment via shp_ShipmentId FK. Only populated when Direction = Import or Both.

| Column name                          | Dataverse field              | Type                               |
| :----------------------------------- | :--------------------------- | :--------------------------------- |
| Shipment (FK)                        | shp_ShipmentId               | Lookup → shp_Shipment (1:1)       |
| ATA                                  | shp_ATA                      | Date Only                          |
| Carrier Arrival Notice Received Date | shp_CarrierArrivalNoticeDate | Date & Time                        |
| Carrier DO Release Date              | shp_CarrierDOReleaseDate     | Date Only                          |
| DO Status                            | shp_DOStatus                 | Choice: Pending/Released/Collected |
| DPW Arrival Notice Date & Time       | shp_DPWArrivalNoticeDateTime | Date & Time                        |
| DPW Arrival Notice Status            | shp_DPWArrivalNoticeStatus   | Choice: Pending/Received/Actioned  |
| DPW DO Release Date                  | shp_DPWDOReleaseDate         | Date Only                          |
| Empty Container Return Date          | shp_EmptyReturnDate          | Date Only                          |
| Empty Container Return Status        | shp_EmptyReturnStatus        | Choice: Pending/Returned           |
| ETA at Final Destination             | shp_ETAFinalDest             | Date Only                          |
| LFD to Empty Return                  | shp_LFDEmptyReturn           | Date Only                          |
| LFD to Pick Up                       | shp_LFDPickUp                | Date Only                          |
| Picked Up Date                       | shp_PickedUpDate             | Date Only                          |
| MBL Type                             | shp_ImportMBLType            | Choice: OBL/Seaway Bill/Express    |
| SI Cut-off Date & Time               | shp_ImportSICutoff           | Date & Time                        |
| SI Instructions Received Date & Time | shp_ImportSIReceivedDate     | Date & Time                        |
| SI Status                            | shp_ImportSIStatus           | Choice: Pending/Received/Submitted |
| SI Submitted Date & Time             | shp_ImportSISubmitted        | Date & Time                        |
| VGM Cut-off Date & Time              | shp_ImportVGMCutoff          | Date & Time                        |
| VGM Received Date & Time             | shp_ImportVGMReceived        | Date & Time                        |
| VGM Status                           | shp_ImportVGMStatus          | Choice: Pending/Received/Submitted |
| VGM Submitted Date & Time            | shp_ImportVGMSubmitted       | Date & Time                        |
| Comments                             | shp_ImportComments           | Text Area (2000)                   |

### 2.3 Export Detail (shp_ExportDetail)

One-to-one with shp_Shipment via shp_ShipmentId FK. Only populated when Direction = Export or Both.

| Column name                          | Dataverse field            | Type                                   |
| :----------------------------------- | :------------------------- | :------------------------------------- |
| Shipment (FK)                        | shp_ShipmentId             | Lookup → shp_Shipment (1:1)           |
| ATD                                  | shp_ATD                    | Date Only                              |
| Customs Filing Date                  | shp_CustomsFilingDate      | Date Only                              |
| Customs Filing Status                | shp_CustomsFilingStatus    | Choice: Pending/Filed/Cleared          |
| Final HBL Release Date               | shp_FinalHBLReleaseDate    | Date Only                              |
| Final HBL Status                     | shp_FinalHBLStatus         | Choice: Pending/Released               |
| Final MBL Release Date               | shp_FinalMBLReleaseDate    | Date Only                              |
| First HBL Draft Sent Date & Time     | shp_FirstHBLDraftSent      | Date & Time                            |
| HBL Amendment Date & Time            | shp_HBLAmendDateTime       | Date & Time                            |
| HBL Amendment Occurred?              | shp_HBLAmendOccurred       | Yes/No                                 |
| HBL Amendments                       | shp_HBLAmendments          | Text Area (500)                        |
| HBL Draft Status                     | shp_HBLDraftStatus         | Choice: Pending/Draft Sent/Approved    |
| HBL Type                             | shp_HBLType                | Choice: OBL/Seaway Bill/Express        |
| MBL Type                             | shp_ExportMBLType          | Choice: OBL/Seaway Bill/Express        |
| Port Cut-off Date & Time             | shp_PortCutoff             | Date & Time                            |
| Pre-Alert Sent Date & Time           | shp_PreAlertSent           | Date & Time                            |
| Pre-Alert Acceptance Date            | shp_PreAlertAcceptanceDate | Date Only                              |
| Pre-Alert Status                     | shp_PreAlertStatus         | Choice: Pending/Sent/Accepted/Rejected |
| SI Cut-off Date & Time               | shp_ExportSICutoff         | Date & Time                            |
| SI Instructions Received Date & Time | shp_ExportSIReceivedDate   | Date & Time                            |
| SI Status                            | shp_ExportSIStatus         | Choice: Pending/Received/Submitted     |
| SI Submitted Date & Time             | shp_ExportSISubmitted      | Date & Time                            |
| VGM Cut-off Date & Time              | shp_ExportVGMCutoff        | Date & Time                            |
| VGM Received Date & Time             | shp_ExportVGMReceived      | Date & Time                            |
| VGM Status                           | shp_ExportVGMStatus        | Choice: Pending/Received/Submitted     |
| VGM Submitted Date & Time            | shp_ExportVGMSubmitted     | Date & Time                            |

### 2.4 Lookup Tables (Admin-managed)

| Table      | Dataverse name | Key fields                                                          |
| :--------- | :------------- | :------------------------------------------------------------------ |
| Shipper    | shp_Shipper    | ShipperID (PK), Name, Country, ContactEmail, ContactPhone, IsActive |
| Carrier    | shp_Carrier    | CarrierID (PK), Name, SCACCode, Country, ContactEmail, IsActive     |
| Consignee  | shp_Consignee  | ConsigneeID (PK), Name, Country, TaxID, ContactEmail, IsActive      |
| CSO Member | shp_CSOMember  | CSOID (PK), FullName, Email, Phone, Role, BranchID, IsActive        |
| Branch     | shp_Branch     | BranchID (PK), Name, Region, Country, TimeZone                      |

### 2.5 User & Auth Tables

| Table        | Field           | Type                        | Notes                                             |
| :----------- | :-------------- | :-------------------------- | :------------------------------------------------ |
| shp_User     | UserID (PK)     | GUID                        |                                                   |
| shp_User     | FullName        | Text (100)                  | Display name                                      |
| shp_User     | Email           | Text (200)                  | Unique — used as login ID                        |
| shp_User     | BranchID        | Lookup → shp_Branch        | Drives data isolation                             |
| shp_User     | Role            | Choice: Admin/User/ReadOnly |                                                   |
| shp_User     | IsActive        | Yes/No                      | Deactivated users cannot log in                   |
| shp_UserAuth | UserAuthID (PK) | GUID                        |                                                   |
| shp_UserAuth | UserID (FK)     | Lookup → shp_User          | 1:1                                               |
| shp_UserAuth | PasswordHash    | Text (200)                  | SHA-256 of password+salt                          |
| shp_UserAuth | Salt            | Text (50)                   | Unique per user, random GUID                      |
| shp_UserAuth | FailedAttempts  | Whole Number                | Reset on successful login                         |
| shp_UserAuth | LockedUntil     | Date & Time                 | Null = not locked; set to +30min after 5 failures |
| shp_UserAuth | LastLogin       | Date & Time                 |                                                   |
| shp_UserAuth | SessionToken    | Text (200)                  | Set on login, cleared on logout                   |
| shp_UserAuth | TokenExpiry     | Date & Time                 | 8 hours from login                                |

### 2.6 Supporting Tables

| Table           | Dataverse name     | Key fields                                                                                                       | Purpose                            |
| :-------------- | :----------------- | :--------------------------------------------------------------------------------------------------------------- | :--------------------------------- |
| Audit Log       | shp_AuditLog       | LogID, TableName, RecordID, Action (Create/Update/Delete), OldValues (JSON), NewValues (JSON), UserID, Timestamp | Immutable change history           |
| Upload Template | shp_UploadTemplate | TemplateID, VersionNo, FileName, FileURL (SharePoint), IsActive, UploadedBy, UploadedOn                          | Version-controlled Excel templates |
| Notification    | shp_Notification   | NotifID, ShipmentID, TriggerField, TriggerValue, RecipientEmail, SentOn, Status                                  | Alert history log                  |

---

## 3. App Screens — UI/UX Specification

**Screen 1 — Login**

- Route: `/login` | Roles: All (pre-auth)
- Elements: App logo, Email input (`txtEmail`), Password input (`txtPassword`), Login button (`btnLogin`), Error label (`lblError`), Forgot password link.
- Global vars set on login: `gblCurrentUser`, `gblUserRole`, `gblBranchID`, `gblSessionToken`.

**Screen 2 — Dashboard (Home)**

- Route: `/dashboard` | Roles: All | Default screen after login
- **KPI strip (top row — 4 tiles):**
  - Active shipments (Blue)
  - Overdue ETA (Red)
  - Pending DO release (Amber)
  - Shipments this month (Green)
- **Status pipeline (Kanban columns):** Horizontal scrollable gallery. Each column = one status value.
- **Recent activity feed:** Vertical gallery of last 20 `shp_AuditLog` records.
- **Upcoming cutoffs:** Gallery of shipments where any cutoff date is within the next 7 days.

**Screen 3 — Shipment List**

- Route: `/shipments` | Roles: All
- **Filter panel (left sidebar, 280px wide):** Direction toggle, Status checklist, ETD date range, ETA date range, Carrier dropdown, Consignee dropdown, Region dropdown, Free text search (ShipmentNo/HBL/MBL).
- **Results gallery:** Shipment No | Direction | Status | Carrier | Shipper | Consignee | ETD | ETA | HBL No.
- **Action buttons:** + New Shipment, Upload, Export (Excel/CSV).

**Screen 4 — Shipment Detail (Create / Edit / View)**

- Route: `/shipment/:id` | Roles: Admin, User (edit); ReadOnly (view only)
- **Tab navigation:**
  - `Basic Details`: Identity, Parties, Routing, Dates, Comments.
  - `Import Details`: Arrival, DO & Pickup, SI, VGM, Comments (Visible if Import/Both).
  - `Export Details`: Departure, HBL, MBL, SI, VGM, Pre-Alert, Comments (Visible if Export/Both).
- **Save logic:** Validate required fields. Auto-creates `shp_ImportDetail` / `shp_ExportDetail` records based on direction. Writes to `shp_AuditLog`.

**Screen 5 — Upload Center**

- Route: `/upload` | Roles: Admin, User
- **Step 1:** Download current template linking to active `shp_UploadTemplate` URL.
- **Step 2:** Upload file (accept `.xlsx`). Calls Upload Parse flow (Flow 4.3).
- **Step 3:** Preview & validation gallery (Green = valid, Red = error, Yellow = warning).
- **Step 4:** Confirm upload. Runs Commit Upload flow (Flow 4.4). Returns summary modal.

**Screen 6 — Reports**

- Route: `/reports` | Roles: All
- **Filter builder:** Select field, operator, and value. Combine with AND logic. Save presets to `localStorage`.
- **Chart section:**
  - Shipment count by status
  - Shipments by carrier (stacked by direction)
  - Monthly volume trend (12 months rolling)
  - ETA vs ATA variance (Import only)
- **Results grid:** Scrollable data table matching filters with an Export button.

**Screen 7 — Live Tracking Dashboard**

- Route: `/tracking` | Roles: All
- **Map view (Azure Maps):** Interactive map with shipment pins colored by status. Clicking reveals details card (ShipmentNo, Vessel, POL -> POD, ETA).
- **Timeline:** Horizontal visual timeline showing key dates (ETD, ETA, ATA, Cutoffs, DO Release). Past dates = solid, future = dotted, overdue = red.

**Screen 8 — Admin Panel**

- Route: `/admin` | Roles: Admin only.
- **Sub-sections (Tabs):** Shippers, Carriers, Consignees, CSO Members, Users, Upload Templates. Provides list/edit galleries to create, update, and deactivate rows. Password reset controls via Email for users.

---

## 4. Power Automate Flow Definitions

* **Flow 4.1 — User Login Authentication:** Validates inputEmail and inputPassword against `shp_UserAuth` table calculating SHA-256 hash. Locks account after 5 failed attempts. Generates sessionToken.
* **Flow 4.2 — Audit Log Writer:** Invoked on record changes. Writes to `shp_AuditLog`.
* **Flow 4.3 — Excel Upload Parse & Validate:** Converts base64 file, parses via Excel Online. Runs required field, duplicate, and lookup validations.
* **Flow 4.4 — Upload Commit:** Processes array from Flow 4.3. Creates or updates shipments, related details, and invokes Flow 4.2.
* **Flow 4.5 — Export to Excel / CSV:** Queries Dataverse via OData filterQuery. Generates CSV/Excel output.
* **Flow 4.6 — Status Change Notification:** Triggered on `shp_Status` change. Checks config and sends Office 365 Outlook email to CSO member.

---

## 5. CodeApps / Vibe-Coding Prompts

Prompts specify precise implementation instructions for Canvas App, theming, variable init, screen layouts, form layouts, map visuals, routing timelines, and Excel handling.

---

## 6. Excel Upload Template — Column Specification

The upload template is a single Excel workbook with one named table `ShipmentUpload`. It combines all 78 fields from Shipment, ImportDetail, and ExportDetail tables. First row is a frozen header; mandatory columns highlighted in yellow. *[Columns mapped corresponding to schema above]*

---

## 7. Recommended Build Sequence

Follow this sequence to always have a working, testable app at each milestone.

| Phase           | Sprint    | Deliverables                                                                   | Estimated effort |
| :-------------- | :-------- | :----------------------------------------------------------------------------- | :--------------- |
| 1 — Foundation | Sprint 1  | All 12 Dataverse tables created, relationships set, test data seeded           | 3–4 days        |
| 1 — Foundation | Sprint 2  | Login screen + Auth flow (4.1) working end-to-end with role + branch isolation | 2–3 days        |
| 2 — Core CRUD  | Sprint 3  | Shipment List screen with basic filters + column sort                          | 3 days           |
| 2 — Core CRUD  | Sprint 4  | Shipment Detail form — Basic Details tab + Save/Edit/Delete + Audit flow      | 4 days           |
| 2 — Core CRUD  | Sprint 5  | Import Details tab + Export Details tab + direction-based tab visibility       | 3 days           |
| 3 — Admin      | Sprint 6  | Admin panel — all 4 lookup table managers (Shipper, Carrier, Consignee, CSO)  | 3 days           |
| 3 — Admin      | Sprint 7  | User management (add, edit role, reset password) + Upload Template manager     | 3 days           |
| 4 — Data ops   | Sprint 8  | Upload Center — Parse flow (4.3) + preview validation gallery                 | 4 days           |
| 4 — Data ops   | Sprint 9  | Upload Commit flow (4.4) + Export flow (4.5) with Excel/CSV output             | 3 days           |
| 5 — Insights   | Sprint 10 | Dashboard KPI tiles + Status pipeline (Kanban view)                            | 3 days           |
| 5 — Insights   | Sprint 11 | Reports screen — dynamic filter builder + 4 charts + grid export              | 4 days           |
| 6 — Advanced   | Sprint 12 | Live tracking timeline + Azure Maps integration (optional)                     | 4–5 days        |
| 6 — Advanced   | Sprint 13 | Notification flow (4.6) + upcoming cutoffs alert on Dashboard                  | 2 days           |
| 7 — Polish     | Sprint 14 | Mobile layout testing, accessibility review, performance optimization          | 3 days           |

### 7.1 Key dependencies

- Power Platform environment with Dataverse must be provisioned before Sprint 1
- SharePoint site for file storage must be created before Sprint 8
- Azure Maps resource must be provisioned before Sprint 12

### 7.2 Naming conventions

| Item                | Convention                        | Example                                      |
| :------------------ | :-------------------------------- | :------------------------------------------- |
| Dataverse table     | `shp_` prefix + PascalCase      | `shp_ImportDetail`                         |
| Dataverse column    | `shp_` prefix + camelCase       | `shp_CarrierBookingNo`                     |
| Power Apps screen   | PascalCase                        | `ShipmentDetail`                           |
| Power Apps control  | 3-letter type prefix + PascalCase | `txtEmail`, `btnLogin`, `galShipments` |
| Power Automate flow | Descriptive verb+noun             | `ShipmentUploadParseFlow`                  |
| Global variable     | `gbl` prefix + camelCase        | `gblCurrentUser`                           |
| Local variable      | `loc` prefix + camelCase        | `locSelectedRecord`                        |
