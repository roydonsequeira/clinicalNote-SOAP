<div align="center">

<img src="https://img.shields.io/badge/-%F0%9F%8F%A5%20Healthcare%20Automation-1a1a2e?style=for-the-badge" />

# Appointment Scheduler

### Rule-based Healthcare Appointment Management REST API

*Book · Reschedule · Cancel · Retrieve — all through clean HTTP endpoints*

<br/>

[![n8n](https://img.shields.io/badge/n8n-Workflow_Engine-FF6D5A?style=for-the-badge&logo=n8n&logoColor=white)](https://n8n.io)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-FHIR_Database-336791?style=for-the-badge&logo=postgresql&logoColor=white)](https://postgresql.org)
[![Google Calendar](https://img.shields.io/badge/Google_Calendar-Availability-4285F4?style=for-the-badge&logo=googlecalendar&logoColor=white)](https://calendar.google.com)
[![Gmail](https://img.shields.io/badge/Gmail-Notifications-EA4335?style=for-the-badge&logo=gmail&logoColor=white)](https://gmail.com)

<br/>

![Status](https://img.shields.io/badge/Status-Production_Ready-brightgreen?style=flat-square)
![License](https://img.shields.io/badge/License-Portfolio-blue?style=flat-square)
![Standard](https://img.shields.io/badge/Standard-FHIR_R4-purple?style=flat-square)

</div>

---

<div align="center">

## ✨ Why This Exists

</div>

> A **fully deterministic**, no-LLM appointment backend that any frontend, mobile app, or EHR system can integrate with via REST. No AI ambiguity — every request follows an exact, predictable pipeline through conflict checks, calendar validation, database persistence, and email delivery.

<br/>

<div align="center">

| 🔒 Zero AI Dependency | ⚡ Low Latency | 📋 FHIR R4 | 🗓️ Calendar Sync | 📧 Auto Notifications |
|:---:|:---:|:---:|:---:|:---:|
| Fully rule-based | Direct node pipeline | Standards compliant | Google Calendar | HTML emails via Gmail |

</div>

---

## 🗺️ API Endpoints at a Glance

<div align="center">

| Method | Endpoint | Action |
|:---:|:---|:---|
| `POST` | `/webhook/appointments/create` | 📌 Book a new appointment |
| `POST` | `/webhook/appointments/update` | 🔄 Reschedule an appointment |
| `POST` | `/webhook/appointments/cancel` | ❌ Cancel an appointment |
| `POST` | `/webhook/appointments/get` | 🔍 Fetch appointment by ID |
| `POST` | `/webhook/appointments/upcoming` | 📅 List upcoming appointments |

</div>

---

## 🏗️ Flow Architecture

<details open>
<summary><b>📌 Book Appointment</b></summary>

<br/>

```
POST /appointments/create
         │
         ▼
  ┌─────────────────┐
  │ Validate Input  │  checks: organization_id, patient_email,
  └───────┬─────────┘          practitioner_email, start, end
          │
          ▼
  ┌─────────────────┐
  │ checkConflict   │  PostgreSQL OVERLAPS — is the practitioner free?
  └───────┬─────────┘
          │
          ▼
  ┌─────────────────┐
  │ checkCalendar   │  Google Calendar free/busy check
  └───────┬─────────┘
          │
     ─────┴──────────────────────────────────
     │ available = false          │ available = true
     ▼                            ▼
 ┌──────────┐            ┌─────────────────┐
 │ 409 Error│            │  mergeResource  │  generate logical_id
 └──────────┘            │  build FHIR obj │  appt-{uuid}
                         └───────┬─────────┘
                                 │
                                 ▼
                         ┌─────────────────┐
                         │ createAppointment│  INSERT → PostgreSQL
                         └───────┬─────────┘
                                 │
                                 ▼
                         ┌─────────────────┐
                         │  createEvent    │  → Google Calendar
                         └───────┬─────────┘
                                 │
                                 ▼
                         ┌─────────────────┐
                         │  Send Email     │  → Gmail HTML confirmation
                         └───────┬─────────┘
                                 │
                                 ▼
                           ✅ 200 Success
```

</details>

<details>
<summary><b>🔄 Reschedule Appointment</b></summary>

<br/>

```
POST /appointments/update
         │
         ▼
  fetchAppointment  →  SELECT by logical_id + org_id
         │
    ─────┴──────────────────
    │ not found             │ found
    ▼                       ▼
 404 Error        checktimeConflict1  (excludes current appt)
                           │
                  checkAvailability1  (Google Calendar)
                           │
                   mergeResource1     patch start / end / description
                           │
                  updateAppointment   UPDATE + version_id++
                           │
                   updateEvent        PATCH Google Calendar
                           │
                  updateConfirmation  Gmail reschedule email
                           │
                     ✅ 200 Success
```

</details>

<details>
<summary><b>❌ Cancel Appointment</b></summary>

<br/>

```
POST /appointments/cancel
         │
         ▼
  fetchAppointment  →  SELECT by logical_id + org_id
         │
         ▼
  mergeResource2    →  status = "cancelled" + cancellationReason
         │
         ▼
  cancelAppointment →  UPDATE PostgreSQL
         │
         ▼
  cancelEvent       →  DELETE Google Calendar event
         │
         ▼
  cancelConfirmation → Gmail cancellation email
         │
         ▼
    ✅ 200 Success
```

</details>

<details>
<summary><b>🔍 Get / 📅 Upcoming</b></summary>

<br/>

```
POST /appointments/get              POST /appointments/upcoming
          │                                    │
          ▼                                    ▼
  SELECT by logical_id              SELECT future booked appts
  + organization_id                 WHERE start >= NOW()
          │                         ORDER BY start ASC
          ▼                                    │
  200 JSON response                  200 JSON response
```

</details>

---

## 🔌 API Reference

### 📌 Book Appointment

```http
POST /webhook/appointments/create
Content-Type: application/json
```

<details open>
<summary>Request · Response · Errors</summary>

**Request**
```json
{
  "organization_id":    "uuid-of-organization",
  "patient_email":      "patient@example.com",
  "practitioner_email": "provider@clinic.com",
  "start":              "2025-12-22T10:00:00+05:30",
  "end":                "2025-12-22T10:30:00+05:30",
  "purpose_of_visit":   "Follow-up consultation"
}
```

**✅ Success — 200**
```json
{
  "status": "success",
  "appointment_logical_id": "appt-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "message": "Appointment created successfully"
}
```

**❌ Errors**

| Code | Reason |
|:---:|:---|
| `400` | Missing required fields |
| `409` | Time slot already booked in database |
| `409` | Time unavailable on Google Calendar |

</details>

---

### 🔄 Reschedule Appointment

```http
POST /webhook/appointments/update
Content-Type: application/json
```

<details>
<summary>Request · Response</summary>

```json
{
  "appointment_logical_id": "appt-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "organization_id":        "uuid-of-organization",
  "practitioner_email":     "provider@clinic.com",
  "start":                  "2025-12-23T14:00:00+05:30",
  "end":                    "2025-12-23T14:30:00+05:30",
  "purpose_of_visit":       "Updated reason"
}
```

**✅ Success — 200** → Same structure as create.

</details>

---

### ❌ Cancel Appointment

```http
POST /webhook/appointments/cancel
Content-Type: application/json
```

<details>
<summary>Request · Response</summary>

```json
{
  "appointment_logical_id": "appt-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "organization_id":        "uuid-of-organization",
  "reason":                 "Patient request"
}
```

**✅ Success — 200**
```json
{
  "status": "success",
  "message": "Appointment cancelled successfully"
}
```

</details>

---

### 🔍 Get Appointment

```http
POST /webhook/appointments/get
Content-Type: application/json
```

<details>
<summary>Request</summary>

```json
{
  "appointment_logical_id": "appt-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "organization_id":        "uuid-of-organization"
}
```

</details>

---

### 📅 Get Upcoming Appointments

```http
POST /webhook/appointments/upcoming
Content-Type: application/json
```

<details>
<summary>Request</summary>

```json
{
  "appointment_organization_id": "uuid-of-organization",
  "practitioner_email":          "provider@clinic.com",
  "limit":                       10
}
```

</details>

---

## 🧩 Node Map

<details>
<summary><b>📌 Book Flow nodes</b></summary>

| Node | Type | Role |
|---|:---:|---|
| `bookAppointment` | Webhook | Entry — receives POST request |
| `checktimeConflict` | PostgreSQL | OVERLAPS query against booked slots |
| `checkAvailibilty` | Google Calendar | Free/busy check |
| `avalibilityConflict` | IF | Branch: available vs unavailable |
| `calConflict` | Respond | 409 — calendar unavailable |
| `mergeResouce` | Code | Generates `appt-{uuid}`, builds FHIR resource |
| `createAppintment` | PostgreSQL | INSERT with patient/practitioner JOIN |
| `createEvent1` | Google Calendar | Create event with attendees |
| `createConfirmation` | Gmail | HTML booking confirmation email |
| `success` | Respond | 200 final response |

</details>

<details>
<summary><b>🔄 Reschedule Flow nodes</b></summary>

| Node | Type | Role |
|---|:---:|---|
| `updateAppointment` | Webhook | Entry |
| `fetchAppointment` | PostgreSQL | SELECT by logical_id + org |
| `appointmentFound` | IF | 404 branch if missing |
| `notFound` | Respond | 404 error |
| `checktimeConflict1` | PostgreSQL | Conflict check excluding current appt |
| `checkAvailibilty1` | Google Calendar | Free/busy for new slot |
| `mergeResouce1` | Code | Patch start/end, increment version_id |
| `updateAppointment1` | PostgreSQL | UPDATE resource + version |
| `updateEvent` | Google Calendar | PATCH calendar event |
| `updateConfirmation` | Gmail | Reschedule email |

</details>

<details>
<summary><b>❌ Cancel Flow nodes</b></summary>

| Node | Type | Role |
|---|:---:|---|
| `cancelAppointment` | Webhook | Entry |
| `fetchAppointment1` | PostgreSQL | Fetch existing record |
| `mergeResouce2` | Code | Set `status = cancelled` + reason |
| `cancelAppointment1` | PostgreSQL | UPDATE with cancelled resource |
| `cancelEvent` | Google Calendar | Delete calendar event |
| `cancelConfirmation` | Gmail | Cancellation email |

</details>

---

## 🗄️ Database Schema

```sql
CREATE TABLE appointment (
  id               SERIAL PRIMARY KEY,
  logical_id       TEXT UNIQUE NOT NULL,          -- "appt-{uuid}"
  version_id       INTEGER DEFAULT 1,             -- increments on each update
  patient_id       UUID REFERENCES patient(id),
  practitioner_id  UUID REFERENCES practitioner(id),
  organization_id  UUID NOT NULL,
  resource         JSONB NOT NULL,                -- FHIR R4 Appointment
  created_at       TIMESTAMPTZ DEFAULT NOW(),
  last_updated_at  TIMESTAMPTZ DEFAULT NOW()
);
```

**FHIR R4 `resource` structure:**

```json
{
  "resourceType": "Appointment",
  "id":           "appt-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "status":       "booked",
  "start":        "2025-12-22T10:00:00+05:30",
  "end":          "2025-12-22T10:30:00+05:30",
  "description":  "Follow-up consultation"
}
```

<details>
<summary><b>Conflict Detection SQL</b></summary>

```sql
SELECT 1
FROM   appointment a
JOIN   practitioner pr ON pr.id = a.practitioner_id
WHERE  a.organization_id        = $1::uuid
  AND  pr.logical_id            = $2::text
  AND  a.resource->>'status'    = 'booked'
  AND  (
         (a.resource->>'start')::timestamptz,
         (a.resource->>'end')::timestamptz
       )
       OVERLAPS
       ( $3::timestamptz, $4::timestamptz )
LIMIT  1;
```

</details>

---

## 📧 Email Notifications

> All emails are sent as styled **HTML** via Gmail OAuth2 and include: Appointment ID, Practitioner, Date, Time, and Purpose of visit.

| Event | Subject Line |
|---|---|
| ✅ New booking | `Your Appointment Has Been Confirmed` |
| 🔄 Reschedule | `Your Appointment Has Been Updated` |
| ❌ Cancellation | `Appointment Cancelled` |

---

## 🆚 This vs. AI Agent Approach

> This workflow is the **deterministic REST API counterpart** to [`dataAgent`](https://github.com/roydonsequeira/dataAgent) — an LLM-powered conversational booking system. Two architectures, one problem.

| | **appointment-scheduler** | **dataAgent** |
|---|:---:|:---:|
| Trigger | REST API | Chat interface |
| AI / LLM | ✗ None | ✓ Llama 3.1 8B |
| Calendar | Google Calendar | Cal.com |
| Flow control | Deterministic nodes | LLM agent decisions |
| Integration | Any backend / EHR | Patient-facing chat |
| Latency | ⚡ Low | 🐢 Higher (inference) |
| Predictability | 100% deterministic | Probabilistic |

---

## 🚀 Quick Start

### Prerequisites

- [n8n](https://n8n.io/) self-hosted instance
- PostgreSQL with `patient`, `practitioner`, `appointment` tables
- Google Calendar + Gmail connected via OAuth2 in n8n

### Setup

**1 — Import**

In n8n: **Settings → Import Workflow** → select `appointment.json`

**2 — Connect credentials**

| Credential | Nodes |
|---|---|
| PostgreSQL | All DB nodes |
| Google Calendar OAuth2 | `checkAvailibilty`, `createEvent1`, `updateEvent`, `cancelEvent` |
| Gmail OAuth2 | `createConfirmation`, `updateConfirmation`, `cancelConfirmation` |

**3 — Set Calendar ID**

In `checkAvailibilty`, `checkAvailibilty1`, and `createEvent1` — replace `YOUR_GOOGLE_CALENDAR_ID` with the practitioner's calendar email.

**4 — (Optional) Cloud SQL Proxy**

```powershell
.\cloud-sql-proxy.exe YOUR_PROJECT:YOUR_REGION:YOUR_INSTANCE --port=5432
```

**5 — Test**

```bash
curl -X POST http://localhost:5678/webhook/appointments/create \
  -H "Content-Type: application/json" \
  -d '{
    "organization_id":    "your-org-uuid",
    "patient_email":      "patient@example.com",
    "practitioner_email": "provider@clinic.com",
    "start":              "2025-12-22T10:00:00+05:30",
    "end":                "2025-12-22T10:30:00+05:30",
    "purpose_of_visit":   "Follow-up consultation"
  }'
```

---

## 🔐 Credential Placeholders

| Placeholder | Replace with |
|---|---|
| `YOUR_POSTGRES_CREDENTIAL_ID` | n8n Postgres credential ID |
| `YOUR_GOOGLE_CALENDAR_CREDENTIAL_ID` | n8n Google Calendar OAuth2 credential ID |
| `YOUR_GMAIL_CREDENTIAL_ID` | n8n Gmail OAuth2 credential ID |
| `YOUR_GOOGLE_CALENDAR_ID` | Practitioner's Google Calendar ID (their email) |
| `YOUR_N8N_INSTANCE_ID` | Your n8n instance ID |

> ⚠️ **Never** commit real credentials, API keys, or personal email addresses to version control.

---

<div align="center">

Made for portfolio purposes · FHIR R4 · n8n · PostgreSQL · Google Workspace

</div>
