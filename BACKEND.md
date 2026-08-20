# Bloom Note — Multi-Agency Backend Design

Target: several agencies share one hosted backend, and no agency can reach
another's records. Modelled on how ProviderSoft operates — hosted, multi-tenant,
agencies log in.

Status: design only. Nothing here is built.

---

## The one rule

**The client asks, the server decides.**

Never send records to the browser and filter them there. The browser must never
receive data the signed-in user is not entitled to, because anything it receives
can be read.

Today's app hides data. A backend withholds it. That is the whole difference.

---

## Isolation lives in the database, not the code

Every table carries `agency_id`, and Postgres row-level security enforces it:

```sql
alter table children enable row level security;

create policy agency_isolation on children
  using (agency_id = current_setting('app.agency_id')::uuid);
```

Application-level filtering alone is one forgotten `WHERE` clause away from a
cross-agency leak. With RLS the database refuses regardless, so a bug in a query
returns nothing rather than someone else's child.

Therapist scoping is the same pattern one layer down:

```sql
create policy own_caseload on children
  using (
    agency_id = current_setting('app.agency_id')::uuid
    and (
      current_setting('app.role') in ('admin','superadmin','biller')
      or exists (
        select 1 from case_assignments ca
        where ca.child_id = children.id
          and ca.user_id = current_setting('app.user_id')::uuid
      )
    )
  );
```

A therapist crafting their own request still gets only their caseload.

---

## Schema

Every table below has `agency_id uuid not null` and an RLS policy. Omitted from
each definition for brevity — it is not optional on any of them.

### agencies
```
id            uuid pk
name          text
county        text          -- drives the rate column; NYC today
active        boolean
modules       jsonb         -- {"billing": true} — super admin only
created_at    timestamptz
```

### users
```
id            uuid pk
email         text unique
password_hash text
role          text          -- superadmin | admin | biller | therapist | coordinator
first_name    text
last_name     text
npi           text
credential    text
license_no    text
license_expires date
tax_id        text
street, city, state, zip, phone
active        boolean
```
`superadmin` is the only role that may write `agencies.modules`, and is the only
role not scoped to a single agency.

### children
```
id            uuid pk
name          text
dob           date
sex           text
ei_number     text
service_type  text
icd10         text
default_cpt   text
ifsp_start    date
street, city, state, zip
```

### case_assignments
```
child_id      uuid fk
user_id       uuid fk
primary key (child_id, user_id)
```

### authorizations
```
id            uuid pk
child_id      uuid fk
auth_number   text
service_type  text
hours         numeric
start_date    date
end_date      date
```

### doctors
```
id            uuid pk
first_name, last_name  text
npi           text          -- ten digits, checked against NPPES
medicaid_id   text
license_no    text
```

### scripts
```
id            uuid pk
child_id      uuid fk
doctor_id     uuid fk
discipline    text
icd10         text
start_date    date
end_date      date
```

### session_notes
```
id            uuid pk
child_id      uuid fk
author_id     uuid fk       -- the actual caregiver, per Program Records guidance
service_date  date
service_category text       -- Direct Therapy | Core Evaluation | Supplemental Evaluation | Screening
bilingual     boolean
header        text
narrative     text
goals         text
coaching      text
strategies    text
status        text          -- draft | signed | billed | cancelled
created_at    timestamptz
```

### note_revisions
```
id            uuid pk
note_id       uuid fk
changed_by    uuid fk
changed_at    timestamptz
field         text
before_value  text
after_value   text
```
Records may be altered, so the alteration must be documented. Append only —
never updated, never deleted.

### sc_contacts
```
id            uuid pk
child_id      uuid fk
coordinator_id uuid fk
contact_date  date
start_time, end_time  time
minutes       int
activity      text
summary       text
person_contacted, relationship, method, purpose  text
ifsp_connection text
signature     text
```

### signatures
```
id            uuid pk
note_id       uuid fk
sig_date      date          -- date only, deliberately
parent_image  text          -- data URI
provider_image text
slog_image    text
```

### rates
```
id            uuid pk
code          text
basis         text          -- unit | session | flat
rate          numeric
modifier      text
unit_minutes  int
session_minutes int
units         int
description   text
```
Rates are per agency because they are per county.

### claims
```
id            uuid pk
claim_number  text          -- must be unique per billing provider, ever
eihub_claim_id text         -- from the 277, needed for REF*F8
child_id      uuid fk
service_date  date
cpt           text
units         int
amount        numeric
status        text          -- submitted | accepted | rejected | paid | void
reason        text
frequency     text          -- 1 original | 7 corrected | 8 void
adjusts_claim uuid fk null
file_name     text
submitted_at  timestamptz
```

### progress_reports
```
id            uuid pk
child_id      uuid fk
report_type   text          -- six-month | annual
filed_date    date
filed_by      uuid fk
```

### audit_log
```
id            uuid pk
user_id       uuid fk
action        text          -- read | create | update | delete | export | submit
entity        text
entity_id     uuid
purpose       text
at            timestamptz
ip            inet
```
Append only. Reads are logged too, not just writes.

---

## Authentication

- Email and password, argon2 or bcrypt
- Session token in an httpOnly, Secure, SameSite cookie — not localStorage
- On every request the server resolves the token to a user, then sets
  `app.user_id`, `app.agency_id`, `app.role` for the connection before any query
  runs. RLS reads those.
- Session expiry, and forced logout when a user is deactivated
- Rate limit login attempts

The client never sends `agency_id`. It comes from the session, server-side.
A client-supplied agency is an attack, not a parameter.

---

## API surface

Roughly one endpoint group per store, all of them agency-scoped implicitly.

```
POST   /auth/login                 → sets session cookie
POST   /auth/logout
GET    /me                         → user, role, agency, modules

GET    /children                   → caseload-scoped
POST   /children
PATCH  /children/:id
POST   /children/:id/assignments

GET    /notes?child=&from=&to=
POST   /notes
PATCH  /notes/:id                  → writes a note_revision, never overwrites

GET    /sc-contacts
POST   /sc-contacts

GET    /doctors  POST /doctors
GET    /scripts  POST /scripts
GET    /rates    POST /rates       → admin only
GET    /claims   POST /claims/generate   → biller or admin only
POST   /claims/:id/adjust          → frequency 7 or 8

GET    /alerts                     → computed server-side
GET    /export/records             → readable export
GET    /export/backup              → full agency backup
```

Role checks belong on the server for every one of these. The current app hides
the Billing tab from a therapist; the backend must also refuse
`POST /claims/generate` from one.

---

## Porting the existing app

The refactor already done means the browser code changes in one place. `Store`
currently reads and writes localStorage:

```js
var Store = {
  read:  function(name, fallback){ /* localStorage */ },
  write: function(name, value){ /* localStorage */ }
};
```

It becomes an API client. Everything above it — the billing engine, rate
resolution, the scrub rules, alerts, the 837P generator — is untouched.

Two things do change shape:

- **Reads become asynchronous.** Call sites that expect a synchronous
  `childrenLoad()` need a loaded-state model, or the app hydrates once on login
  and works from memory, syncing writes.
- **Alerts move server-side.** They currently scan every record in the browser,
  which stops being possible once records are not all local.

---

## Order of work

1. Hosting model decided, vendor chosen, **BAA signed**
2. Schema and RLS policies, on synthetic data
3. Authentication
4. API, endpoint by endpoint
5. `Store` becomes an API client
6. Migrate existing localStorage data into an agency
7. Audit logging
8. Written risk analysis, security officer named, workforce training
9. Security review — **then** real children's records

Steps 1 and 8 are not paperwork to catch up on later. They are what makes
holding this data lawful.

---

## What this does not cover

- Whether NY EI requires an access log at all, given its exclusions for parents,
  municipality employees, EI providers and Department staff
- Retention beyond six years, and what deletion looks like afterwards
- Whether you are a Business Associate or the agencies are — the hosting model
  decides it, and it is worth an hour with a healthcare attorney
