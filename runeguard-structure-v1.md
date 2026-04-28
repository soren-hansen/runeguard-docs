# RuneGuard — Entity Structure & Relationships v1
*Apr 28, 2026 | For: Vlad (technical reference)*

---

## Background

Commercial cleaning companies clean facilities they don't own, for clients who weren't there to watch. The job is done, but proving it — consistently, professionally, and at scale — is hard.

**RuneGuard solves this.** Crew members photograph each area before and after cleaning via a Telegram bot. The platform analyses the photos with AI, compiles a structured cleaning report, and delivers it to the facility manager by email. The cleaning company's supervisor reviews every report before it goes out — nothing reaches the customer without approval.

**The key entities and how they relate:**

- **Client** — a cleaning company using RuneGuard (e.g. Craddock's Cleaning Services). Each client gets their own isolated workspace on the platform.
- **Customer** — the facility the client cleans (e.g. Hospital Dallas, ACME Offices). The customer receives the finished report by email after every job.
- **Crew Member** — an individual cleaner. They authenticate via Telegram — no separate login needed. They submit before/after photos through the bot directly from the job site.
- **Job** — one cleaning visit. A job belongs to a customer, is executed by one or more crew members, covers one or more areas, and produces one report.
- **Report** — the AI-compiled output of a job. Reviewed and approved by a supervisor before it's emailed to the customer.

**A typical job, start to finish:**
> Craddock's crew arrives at Hospital Dallas. Each cleaner opens the Telegram bot, selects the job, and photographs the areas before they start. They clean. They photograph each area again. The bot submits everything to RuneGuard. The AI analyses the before/after pairs and drafts a report. Craddock's supervisor reviews it on the dashboard, approves it, and it lands in the facility manager's inbox — clean, professional, and timestamped.

**Prime and Sub-Contractors:**
A Client can operate in both roles depending on the job. When Craddock's cleans one of their own customers (Hospital Dallas, DOME in Dallas), Craddock's is the Prime — they own the customer relationship, approve the report, and deliver it. But Craddock's can also work as a Sub-Contractor for another Prime — for example, if Dallas Cleaning Inc. holds the contract with a facility and hires Craddock's to do the work, then Dallas Cleaning Inc. is the Prime. Craddock's crew submits the photos, but Dallas Cleaning Inc. reviews and delivers the report to the end customer. The facility only ever hears from the Prime.

---

## Entity Map

```
Platform
└── Client (Tenant)                      any cleaning company on the platform
    │
    ├── [as Prime Contractor]             e.g. Craddock's — owns the customer relationship
    │   ├── Customer                      e.g. Hospital Dallas, ACME Offices
    │   │   └── Location / Facility       e.g. Building A, Floor 3 (optional sub-level)
    │   └── Job
    │       ├── accountable_client        → Prime (approves report + delivers to customer)
    │       ├── executing_client          → Prime's own crew OR a Sub-Contractor Client
    │       ├── CrewMember(s)             linked via Telegram user ID
    │       ├── Area(s)                   e.g. Kitchen, Reception, Server Room
    │       │   └── Media                 before + after photos per area
    │       ├── CrewNotes
    │       ├── AIAnalysis
    │       └── Report                   always delivered from Prime — Sub is never visible
    │
    └── [as Sub-Contractor]              another Client working under a Prime
        ├── CrewMember(s)                their crew, linked via Telegram
        └── Jobs (executing_client)      submit photos only — Prime reviews + delivers
```

**Craddock's in practice:**
- **Prime** to their own customers (Hospital Dallas, DOME in Dallas, etc.) — always
- May use **Sub crews** on jobs — Sub submits photos, Craddock's supervisor approves, Craddock's delivers the report
- May act as **Sub** for a larger prime — in that case the prime is accountable_client and delivers to the end customer

---

## Entities & Fields

### Client (Tenant)
```
id
slug                    e.g. "craddocks"
name                    e.g. "Craddock's Cleaning Services"
contact_name
contact_email
created_at
```

### Customer
```
id
client_id               → Client
name                    e.g. "Hospital Dallas"
contact_name
contact_email           ← report delivery address
delivery_method         email (primary)
telegram_chat_id        optional
created_at
```

### CrewMember
```
id
client_id               → Client
name
telegram_user_id        ← identity (no separate login)
telegram_handle         e.g. "@bullet62"
linked                  boolean
created_at
```

### Job
```
id
client_id               → Client
customer_id             → Customer
name                    auto-generated: "{Customer} — {Date}" or crew-provided
status                  submitted | in_review | accepted | delivered | failed
failure_reason          text (populated when status = failed)
created_at
submitted_at
delivered_at
```

### JobCrewAssignment
```
id
job_id                  → Job
crew_member_id          → CrewMember
```

### Area
```
id
job_id                  → Job
label                   e.g. "Kitchen", "Lobby", "Restrooms"
crew_notes              free text
```

### Media
```
id
area_id                 → Area
job_id                  → Job
type                    before | after
file_url
telegram_file_id
uploaded_at
uploaded_by             → CrewMember
```

### AIAnalysis
```
id
job_id                  → Job
area_id                 → Area (nullable — can be job-level)
visible_outcome         AI-generated text description
flags                   e.g. ["missing_after_photo", "clutter_visible"]
generated_at
```

### Report
```
id
job_id                  → Job (1:1)
status                  draft | ready | sent
html_content
pdf_url                 (generated on approval)
sent_to_email
sent_at
approved_by             → Admin/Supervisor user
approved_at
```

### JobSchedule (recurring jobs)
```
id
client_id               → Client
customer_id             → Customer
crew_member_id          → CrewMember (default assignee)
frequency               daily | weekly | custom
cron_expression         e.g. "0 8 * * 2" (every Tuesday 8am)
areas                   JSON list of area labels
active                  boolean
next_run_at
```
*Each schedule run spawns a new Job instance.*

---

## Relationships Summary

| From | To | Cardinality |
|---|---|---|
| Client | Customer | 1 : many |
| Client | CrewMember | 1 : many |
| Client | Job | 1 : many |
| Customer | Job | 1 : many |
| Job | CrewMember | many : many (via JobCrewAssignment) |
| Job | Area | 1 : many |
| Area | Media | 1 : many |
| Job | AIAnalysis | 1 : many |
| Job | Report | 1 : 1 |
| JobSchedule | Job | 1 : many (spawned instances) |

---

## Report State Machine

```
[submitted]
    ↓
[in_review]   ← supervisor sees it on dashboard
    ↓              ↓
[accepted]     [failed]   (with failure_reason)
    ↓
[delivered]   ← PDF generated + sent to customer.contact_email
```

---

## Delivery Flow

1. Job reaches `accepted` status
2. System generates PDF from Report.html_content
3. Email sent to `Customer.contact_email` with PDF attached
4. Report.status → `sent`, Report.sent_at = now
5. Job.status → `delivered`, Job.delivered_at = now

---

---

*Søren Hansen / LeadPillar — send to Vlad for review*
