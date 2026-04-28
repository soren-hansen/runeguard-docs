# RuneGuard — Entity Structure & Relationships v1
*Apr 28, 2026 | For: Vlad (technical reference)*

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
