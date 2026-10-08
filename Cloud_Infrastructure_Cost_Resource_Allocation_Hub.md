

## 1. Executive Summary

In modern software engineering, development teams frequently spin up cloud virtual machines, databases, and storage buckets across cloud providers (AWS, GCP, Azure) for testing and staging environments. However, developers often forget to terminate or downsize these instances once testing ends. This leads to **cloud sprawl** and significant budget overruns due to idle infrastructure.

The **Cloud Infrastructure Cost & Resource Allocation Hub** is an enterprise-grade FinOps platform built on Zoho Creator. It manages the complete lifecycle of cloud resources, enforces department-level budget controls, automates idle asset detection, routes multi-level approval workflows, and triggers automated instance termination via REST APIs.

---

## 2. Relational Database Schema (5 Forms)

The application architecture consists of 5 linked relational forms to ensure proper data normalization and access control.

### Form 1: `Departments Master`

* **Department Name:** Single Line (`AI/ML Team`, `DevOps`, `QA Testing`, `Backend Dev`)
* **Department Head:** Single Line (Email)
* **Overall Quarterly Budget ($):** Currency / Decimal

### Form 2: `Cloud Projects Master`

* **Project Name:** Single Line (`NLP-Model-Training`, `Auth-Service-Dev`)
* **Department:** Lookup field to `Departments Master`
* **Lead Engineer:** Single Line (Email)
* **Cloud Provider:** Dropdown (`AWS`, `Google Cloud Platform (GCP)`, `Microsoft Azure`)
* **Allocated Project Budget ($):** Currency / Decimal
* **Project Status:** Dropdown (`Active`, `Frozen / Over Budget`)

### Form 3: `Cloud Resource Registry` (Active Assets)

* **Resource ID:** Single Line (`i-0a8bf29381`, `gcp-db-main-01`)
* **Project Name:** Lookup field to `Cloud Projects Master`
* **Resource Type:** Dropdown (`EC2 Compute Instance`, `RDS Database`, `S3 Storage Bucket`, `Kubernetes Cluster`)
* **Instance Size / Tier:** Dropdown (`t3.micro`, `t3.large`, `m5.2xlarge`, `db.r5.xlarge`)
* **Hourly Cost ($):** Decimal (Auto-calculated via Deluge)
* **Idle CPU Utilization (%):** Decimal (Simulated metric, e.g., `2.5%`)
* **Resource Lifecycle Status:** Dropdown (`Running`, `Idle / Waste`, `Terminated`)

### Form 4: `Cloud Resource Requisitions` (Approval Gateway)

* **Requisition ID:** Auto-Number (`REQ-5001`)
* **Project Name:** Lookup field to `Cloud Projects Master`
* **Requested By:** Single Line (Developer Email)
* **Requested Instance Tier:** Dropdown (`t3.micro`, `t3.large`, `m5.2xlarge`)
* **Justification:** Multi-line Text
* **Estimated Monthly Cost ($):** Decimal (Auto-calculated)
* **Approval Status:** Dropdown (`Pending Department Head`, `Approved`, `Rejected`)

### Form 5: `Resource Cleanup Logs` (Audit Trail)

* **Log ID:** Auto-Number (`LOG-1001`)
* **Resource Name:** Lookup field to `Cloud Resource Registry`
* **Project Name:** Single Line
* **Wasted Dollars Saved ($):** Decimal
* **Action Taken:** Dropdown (`Auto-Stopped via API`, `Developer Notified`, `Manually Retained`)
* **Execution Timestamp:** Date-Time

---

## 3. Workflow Architecture

The application demonstrates three distinct workflow engines within Zoho Creator.

### A. Form Workflows (`On User Input` & `On Validate`)

1. **Dynamic Hourly Cost Calculation (`On User Input`):**
   * **Trigger:** Selecting an instance tier in `Cloud Resource Requisitions` or `Cloud Resource Registry`.
   * **Logic:** Automatically maps instance size to hourly pricing (`$0.0104/hr` for `t3.micro`, `$0.3840/hr` for `m5.2xlarge`) and computes projected monthly costs.

2. **Budget Pre-Validation (`On Validate`):**
   * **Trigger:** Submitting a new resource requisition.
   * **Logic:** Compares projected instance costs against the project's remaining budget. Blocks submission if the request breaches the allocated limit.

### B. Multi-Level Approval Workflow

1. **Trigger:** Submission of a new request in `Cloud Resource Requisitions`.

2. **Execution Flow:**
   * Notification and task routing sent to the **Department Head** configured in `Departments Master`.
   * **If Rejected:** Status updates to `Rejected`, and notification email with reviewer comments is sent to the developer.
   * **If Approved:** Status updates to `Approved`, triggering a Deluge `On Approval` script to automatically provision an active record in `Cloud Resource Registry`.

### C. Schedule Workflow (`Scheduled Task`)

1. **Trigger:** Deluge Scheduled Task running every Friday at 8:00 PM.

2. **Execution Flow:**
   * Scans `Cloud Resource Registry` for running resources with average CPU utilization below `5.0%`.
   * Automatically updates asset status to `Idle / Waste`.
   * Calculates total wasted dollars and logs an audit entry in `Resource Cleanup Logs`.
   * Dispatches an alert email to the project lead.

---

## 4. Deluge Scripting & REST API Integration (`invokeUrl`)

To demonstrate external system interaction, a custom action button (**"Stop Idle Instance"**) triggers an outbound HTTP POST call via Deluge `invokeurl` to an external API endpoint/webhook.

### Deluge API Trigger Code

```deluge
// Endpoint representing External Cloud API Gateway / Webhook
apiUrl = "https://webhook.site/YOUR-UNIQUE-WEBHOOK-ID";

headers = Map();
headers.put("Content-Type", "application/json");
headers.put("Authorization", "Bearer FINOPS_SECRET_TOKEN");

// Construct JSON Payload
paramMap = Map();
paramMap.put("instance_id", input.Resource_ID);
paramMap.put("action", "STOP_INSTANCE");
paramMap.put("cloud_provider", input.Project_Name.Cloud_Provider);
paramMap.put("requested_by", zoho.adminuser);

// Execute HTTP POST Request
response = invokeurl
[
    url : apiUrl
    type : POST
    headers : headers
    string : paramMap.toString()
];

// Update local Creator record on successful response
input.Resource_Lifecycle_Status = "Terminated";
alert "Instance " + input.Resource_ID + " successfully stopped via REST API call.";
```

---

## 5. Analytical Dashboard Layout (Page Builder)

The executive FinOps dashboard is built using Zoho Creator Page Builder and includes:

* **3 KPI Cards:**
  * Total Monthly Spend
  * Total Cloud Waste
  * Total Active Instances
* **Pie Chart:** Cost Allocation by Department (AI/ML vs DevOps vs QA).
* **Bar Chart:** Top 5 Most Expensive Cloud Projects.
* **Real-time Report Grid:** Filtered view displaying resources tagged as `Idle / Waste` pending administrative action.

---

## 6. Technical Stack & Deliverables Summary

| Component | Technical Implementation |
|---|---|
| **Platform** | Zoho Creator (Free Student / Trial Account) |
| **Backend Language** | Deluge Scripting Language |
| **Relational Forms** | 5 Forms (Departments, Projects, Registry, Requisitions, Logs) |
| **Workflows** | Form Validation, Multi-Level Approval Engine, Scheduled Sweeper |
| **API Integration** | Outbound HTTP POST via `invokeUrl` to Webhook/Mock Server |
| **User Interface** | Page Builder Analytics Dashboard with KPI Widgets |

---

## 7. Workflow / Form Relationship Diagram

The following diagram represents the intended relationship and workflow between the five forms:

```text
                         ▲
                         │
              [Form 1: Departments Master]
                         │
                         │
              [Form 2: Cloud Projects Master] ◄───────────────
                         │                                    │
                         │                                    │
        ┌────────────────┴─────────────────┐                  │
        │                                  │                  │
        ▼                                  ▼                  │
[Form 4: Cloud Resource             (Project / Department     │
 Requisitions - Approval)             relationships)          │
        │                                                     │
        │ On Approval                                         │
        ▼                                                     │
[Form 3: Cloud Resource Registry] ────────────────────────────┘
        │
        │ Cleanup / API action
        ▼
[Form 5: Resource Cleanup Logs]
```

The uploaded reference image also depicts the overall flow among the five forms:

- **Form 1: Departments Master** acts as the department-level master.
- **Form 2: Cloud Projects Master** maintains project-level cloud configuration and links projects to departments.
- **Form 4: Cloud Resource Requisitions** serves as the approval gateway.
- **Form 3: Cloud Resource Registry** contains approved/active cloud resources.
- **Form 5: Resource Cleanup Logs** records cleanup and resource-management actions.

---

## 8. Project Deliverables

The completed project should demonstrate:

1. A normalized five-form relational data model.
2. Lookup relationships between departments, projects, requisitions, and resources.
3. Automated cost calculation using Deluge.
4. Budget validation before resource requisition submission.
5. Department-head approval workflow.
6. Automatic resource registration after approval.
7. Scheduled detection of low-utilization resources.
8. Cloud-waste calculation and audit logging.
9. REST API integration using `invokeurl`.
10. A FinOps analytics dashboard with KPIs and charts.
11. An auditable cleanup history through `Resource Cleanup Logs`.

---

## 9. Expected Outcome

The proposed **Cloud Infrastructure Cost & Resource Allocation Hub (FinOps Engine)** provides a centralized low-code solution for controlling cloud infrastructure expenditure. By combining relational data management, Deluge automation, approval workflows, scheduled resource monitoring, REST API integration, and executive analytics, the application demonstrates practical implementation of cloud governance and FinOps principles within Zoho Creator.
