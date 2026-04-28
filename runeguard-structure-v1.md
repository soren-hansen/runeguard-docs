# RuneGuard — Entity Structure & Relationships v1
*Apr 28, 2026 | For: Vlad (technical reference)*

---

## Background

Commercial cleaning companies clean facilities they don't own, for clients who weren't there to watch. The job is done, but proving it — consistently, professionally, and at scale — is hard.

**RuneGuard solves this.** Crew members photograph each area before and after cleaning via a Telegram bot. The platform analyses the photos with AI, compiles a structured cleaning report, and delivers it to the facility manager by email. The cleaning company's supervisor reviews every report before it goes out — nothing reaches the customer without approval.

**The key entities and how they relate:**

- **Client** — a cleaning company using RuneGuard (e.g. Craddock's Cleaning Services). Each client gets their own isolated workspace on the platform.
- **Customer** — the facility the client cleans directly (e.g. Hospital Dallas, ACME Offices). Receives the finished report by email after every job.
- **Crew Member** — an individual cleaner. Authenticates via Telegram — no separate login. Submits before/after photos through the bot from the job site.
- **SubContractor** — an external cleaning company hired by the Client to execute a job. Their crew submits photos; the Client reviews and delivers the report.
- **PrimeContractor** — an external company that holds the facility contract and hires the Client to do the cleaning. The Client executes and reviews, but the report is delivered to the PrimeContractor, not the end facility.
- **Job** — one cleaning visit. Belongs to a Customer or a PrimeContractor, executed by own crew or a SubContractor, covers one or more areas, produces one report.
- **Report** — the AI-compiled output of a job. Reviewed and approved by a supervisor before delivery.

**A typical job, start to finish:**
> Craddock's crew arrives at Hospital Dallas. Each cleaner opens the Telegram bot, selects the job, and photographs the areas before they start. They clean. They photograph each area again. The bot submits everything to RuneGuard. The AI analyses the before/after pairs and drafts a report. Craddock's supervisor reviews it on the dashboard, approves it, and it lands in the facility manager's inbox — clean, professional, and timestamped.

**Prime and Sub-Contractors:**
A Client operates in different roles depending on the job:

1. **Client as Prime, own crew** — Craddock's cleans Hospital Dallas with their own crew. Craddock's owns the customer relationship, reviews the report, delivers it to the facility.
2. **Client as Prime, Sub executes** — Craddock's holds the contract with Hospital Dallas but uses a SubContractor crew to do the work. The Sub's crew submits photos. Craddock's reviews and delivers. The customer never knows a sub was used.
3. **Client as Sub** — Dallas Cleaning Inc. holds the contract with a facility and hires Craddock's to do the cleaning. Craddock's crew submits photos and reviews the report, but it is delivered to Dallas Cleaning Inc. (the PrimeContractor). The end facility only hears from Dallas Cleaning Inc.

---

## Entity Map

```
Platform
└── Client (Tenant)                       e.g. Craddock's Cleaning Services
    │
    ├── Customer                           e.g. Hospital Dallas — facility they clean directly
    ├── SubContractor                      e.g. Acme Crew Co. — hired to execute jobs for Client
    │   └── CrewMember(s)                  sub's crew, linked via Telegram
    ├── PrimeContractor                    e.g. Dallas Cleaning Inc. — Client works under them
    ├── CrewMember(s)                      Client's own crew, linked via Telegram
    │
    └── Job
        ├── customer_id                    → Customer     (set when Client is Prime)
        ├── sub_contractor_id              → SubContractor (set when Sub executes the job)
        ├── prime_contractor_id            → PrimeContractor (set when Client is Sub)
        ├── CrewMember(s)                  own crew or Sub's crew
        ├── Area(s)                        e.g. Kitchen, Reception, Server Room
        │   └── Media                      before + after photos per area
        ├── CrewNotes
        ├── AIAnalysis
        └── Report                         delivered to Customer OR PrimeContractor
```

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

### SubContractor
```
id
client_id               → Client (the prime who hired them)
name                    e.g. "Acme Crew Co."
contact_name
contact_email
phone                   optional
created_at
```

### PrimeContractor
```
id
client_id               → Client (the sub who works for them)
name                    e.g. "Dallas Cleaning Inc."
contact_name
contact_email           ← report delivered here
phone                   optional
created_at
```

### CrewMember
```
id
client_id               → Client
sub_contractor_id       → SubContractor (nullable — set if crew belongs to a sub)
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
customer_id             → Customer        (nullable — null when Client is acting as Sub)
sub_contractor_id       → SubContractor   (nullable — set when a Sub executes the job)
prime_contractor_id     → PrimeContractor (nullable — set when Client is acting as Sub)
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
sub_contractor_id       → SubContractor   (optional default sub for this schedule)
crew_member_id          → CrewMember      (default assignee)
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
| Client | SubContractor | 1 : many |
| Client | PrimeContractor | 1 : many |
| Client | CrewMember | 1 : many |
| Client | Job | 1 : many |
| SubContractor | CrewMember | 1 : many |
| Customer | Job | 1 : many |
| SubContractor | Job | 1 : many |
| PrimeContractor | Job | 1 : many |
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
[in_review]   ← Client supervisor reviews on dashboard
    ↓              ↓
[accepted]     [failed]   (with failure_reason)
    ↓
[delivered]   ← PDF generated + emailed
```

---

## Delivery Flow

**When Client is Prime (customer_id is set):**
1. Job accepted → PDF generated
2. Report emailed to `Customer.contact_email`
3. Job status → delivered

**When Client is Sub (prime_contractor_id is set):**
1. Job accepted → PDF generated
2. Report emailed to `PrimeContractor.contact_email`
3. Job status → delivered
4. End facility is not contacted — Prime handles that relationship

---

*Søren Hansen / LeadPillar — send to Vlad for review*
