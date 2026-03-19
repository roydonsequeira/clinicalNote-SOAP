<div align="center">

# 🏥 Clinical SOAP Note Generator

**AI-powered clinical documentation workflows built on n8n**

Generate structured SOAP notes from text or voice — automatically.

![n8n](https://img.shields.io/badge/n8n-workflow-orange?style=for-the-badge&logo=n8n)
![Ollama](https://img.shields.io/badge/Ollama-Gemma_3-blue?style=for-the-badge)
![ElevenLabs](https://img.shields.io/badge/ElevenLabs-STT_%7C_TTS-purple?style=for-the-badge)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-FHIR-blue?style=for-the-badge&logo=postgresql)

</div>

---

## 📋 Overview

This project provides three production-ready n8n workflows for automated clinical note generation. A clinician or intake system sends raw clinical input — as typed text or a voice recording — and receives a fully structured **SOAP** (Subjective, Objective, Assessment, Plan) note in return, conforming to the master clinical progress note template.

| Workflow | File | Input | Output |
|---|---|---|---|
| **TTT** | `clinicalNote(TTT).json` | Raw clinical text | SOAP note JSON |
| **STT** | `clinicalNote(STT).json` | Audio recording | SOAP note JSON |
| **TTS** | `clinicalNote(TTS).json` | Raw clinical text | SOAP note + spoken audio |

---

## 🗂️ Master SOAP Template

> 📄 **[Download the full master template PDF](./Clinical-Progress-Note_Master.pdf)**

All workflows produce output conforming to the following clinical progress note structure:

```
Clinical Progress Note
══════════════════════════════════════════════════

Patient Information
  ├── Patient Name      → {{patient_name}}
  ├── Date of Birth     → {{patient_dob}}
  ├── Patient ID        → {{patient_id}}
  ├── Date of Visit     → {{visit_date}}
  ├── Practitioner      → {{practitioner}}, {{practitioner_credentials}}
  └── Chief Complaint   → {{chief_complaint}}

S — Subjective  (Patient's own words & reported history)
  ├── HPI Narrative           → {{hpi_narrative}}
  ├── Reported Symptoms       → {{symptoms_reported}}
  ├── Progress Since Last     → {{progress_since_last}}
  └── Compliance with Plan    → {{compliance_notes}}

O — Objective  (Measurable, observable clinical data)
  ├── Vital Signs
  │     ├── BP              → {{bp}}
  │     ├── Heart Rate      → {{heart_rate}}
  │     ├── Temperature     → {{temperature}}
  │     └── O2 Saturation   → {{o2_saturation}}
  ├── Physical Exam         → {{physical_exam_findings}}
  ├── Labs Reviewed         → {{lab_results}}
  └── Imaging               → {{imaging_results}}

A — Assessment  (Clinician's diagnosis & interpretation)
  ├── Primary Diagnosis (ICD-10) → {{primary_diagnosis}} ({{icd_10_code}})
  ├── Problem List               → {{problem_list}}
  └── Clinical Rationale         → {{assessment_summary}}

P — Plan  (Course of action)
  ├── Medication Changes     → {{medication_plan}}
  ├── Interventions          → {{intervention_details}}
  ├── Follow-up Instructions → {{follow_up_plan}}
  └── Referrals              → {{referrals}}
```

---

## ⚙️ Architecture

```
HTTP Request (Text or Audio)
           │
           ▼
    ┌─────────────┐
    │   Webhook   │  ← n8n entry point
    └──────┬──────┘
           │
     ┌─────┴──────┐
     ▼            ▼
┌─────────┐  ┌──────────┐     [STT only]
│ Verify  │  │ ElevenLabs│ ←── Audio transcription
│Practit. │  │  Scribe   │
└────┬────┘  └─────┬─────┘
     ▼             │
┌─────────┐        │
│ Verify  │        │
│ Patient │        │
└────┬────┘        │
     └──────┬──────┘
            ▼
     ┌─────────────┐
     │  codeParse  │  ← Normalize patient + practitioner + clinical input
     └──────┬──────┘
            ▼
     ┌─────────────┐
     │  Gemma 3 4B │  ← Local LLM via Ollama
     │ (LLM Chain) │
     └──────┬──────┘
            ▼
     ┌─────────────┐
     │ Clean JSON  │  ← Strip markdown, parse output
     └──────┬──────┘
            ▼
     ┌──────────────────────────┐
     │  Response                │
     │  JSON (TTT / STT)        │
     │  audio/mpeg (TTS)        │
     └──────────────────────────┘
```

---

## 🔌 Workflow Triggers & API Reference

### 1️⃣ TTT — Text to Text

> Accepts raw clinical text → returns SOAP note as JSON

**Endpoint**
```
POST /webhook/clinical-note/start
Content-Type: application/json
```

**Request Body**
```json
{
  "provider_email": "provider@clinic.com",
  "patient_email": "patient@example.com",
  "appointment_id": "APPT-XXXX",
  "mode": "text",
  "text_input": "<raw clinical notes from the encounter>",
  "metadata": {
    "source": "patient",
    "language": "en-US"
  }
}
```

**Example Clinical Input**
```
34-year-old male with 4 days of fever, cough, sore throat, congestion,
headache, chills, and fatigue. Cough initially dry, now mildly productive.
Denies chest pain, shortness of breath, nausea, vomiting, or diarrhea.
Partial relief with paracetamol. Vitals: Temp 37.8 C, BP 118/74, HR 88,
RR 16, SpO2 98% on room air. Mild pharyngeal erythema. Lungs clear.
COVID and rapid strep negative.
```

**Response**
```json
{
  "status": "success",
  "code": 200,
  "message": "Clinical SOAP note generated successfully.",
  "data": {
    "soap_note": {
      "patient_name": "{{patient_name}}",
      "patient_dob": "{{patient_dob}}",
      "patient_id": "{{patient_id}}",
      "visit_date": "{{visit_date}}",
      "practitioner": "{{practitioner_name}}",
      "practitioner_credentials": "MD",
      "chief_compliant": "Patient presents with cough, congestion, and subjective fever.",
      "subjective": {
        "hpi_narrative": "Patient reports 4 days of fever, cough, sore throat, congestion, headache, chills, and fatigue...",
        "symptoms_reported": "Fever, cough, sore throat, congestion, headache, chills, fatigue. Denies chest pain, shortness of breath.",
        "progress_since_last": "NA",
        "compliance_notes": "NA"
      },
      "objective": {
        "bp": "118/74",
        "heart_rate": "88",
        "temperature": "37.8 C",
        "o2_saturation": "98 percent on room air",
        "physical_exam_findings": "Mild pharyngeal erythema. Lungs clear bilaterally. Heart rhythm normal. Abdomen soft, non-tender.",
        "lab_results": "COVID negative. Rapid strep negative.",
        "imaging_results": "None"
      },
      "assessment": {
        "primary_diagnosis": "Upper Respiratory Infection",
        "icd_10_code": "J06.9",
        "problem_list": "Upper Respiratory Infection",
        "assessment_summary": "Presentation consistent with viral upper respiratory infection based on symptom duration, negative rapid tests, and clinical findings."
      },
      "plan": {
        "medication_plan": "Acetaminophen 325mg every 6 hours as needed. Saline nasal spray for congestion.",
        "intervention_details": "Patient educated on hydration and rest.",
        "follow_up_plan": "Return if symptoms worsen or do not improve within 72 hours.",
        "referrals": "None"
      }
    }
  },
  "timestamp": "2025-12-16T00:00:00.000Z"
}
```

---

### 2️⃣ STT — Speech to Text

> Accepts a clinical audio recording → transcribes it → returns SOAP note as JSON

**Endpoint**
```
POST /webhook/clinical-note-audio
Content-Type: multipart/form-data
```

**Request Fields**

| Field | Type | Description |
|---|---|---|
| `data` | File | Audio recording of the clinical encounter (`.wav`, `.mp3`, etc.) |
| `practitioner_email` | String | Practitioner's registered email |
| `patient_email` | String | Patient's registered email |

**cURL Example**
```bash
curl -X POST http://localhost:5678/webhook/clinical-note-audio \
  -F "data=@recording.wav" \
  -F "practitioner_email=provider@clinic.com" \
  -F "patient_email=patient@example.com"
```

**Response** — Same SOAP note JSON structure as TTT.

> Audio is transcribed using **ElevenLabs Scribe v1** before being processed by the LLM.

---

### 3️⃣ TTS — Text to Speech

> Accepts raw clinical text → generates SOAP note → returns spoken summary as audio

**Endpoint**
```
POST /webhook/clinical-note-tts
Content-Type: application/json
```

**Request Body** — Same structure as TTT.

**Response**
```
HTTP 200
Content-Type: audio/mpeg
Content-Disposition: inline; filename="clinical_note.mp3"

[binary audio stream]
```

The spoken summary includes:
- Chief Complaint
- History of Present Illness
- Assessment Summary
- Follow-up Plan

---

## 🧩 Node Reference

### Shared Nodes (all 3 workflows)

| Node | Type | Purpose |
|---|---|---|
| `clinicalNote` | Webhook | Entry point — receives the HTTP request |
| `verifyPractitioner` | PostgreSQL | Looks up practitioner FHIR resource by email |
| `verifyPatient` | PostgreSQL | Looks up patient FHIR resource by email |
| `codeParse` | Code | Normalizes patient, practitioner & clinical text into one payload |
| `promptChain` | LLM Chain | Sends structured prompt to Gemma 3 via Ollama |
| `cleanOutputJSON` | Code | Strips LLM markdown, returns parsed JSON |
| `respondSuccess` | Respond to Webhook | Returns final JSON response |

### STT-only Nodes

| Node | Type | Purpose |
|---|---|---|
| `speech2Text` | HTTP Request | Sends audio to ElevenLabs Scribe v1 for transcription |
| `extractTranscription` | Code | Extracts `text` field from transcription response |
| `merge` | Merge | Joins verified patient/practitioner data with transcription |

### TTS-only Nodes

| Node | Type | Purpose |
|---|---|---|
| `cleanOutputJSON` | Code | Extracts HPI, assessment & plan for the spoken script |
| `text2Speech` | HTTP Request | Sends script to ElevenLabs TTS API, receives audio |
| `respondSucessAudio` | Respond to Webhook | Returns binary audio response |

---

## 🗄️ Database Schema (FHIR)

Patient and practitioner data is stored as **FHIR R4** JSON resources in PostgreSQL:

```sql
CREATE TABLE practitioner (
  resource JSONB
);

CREATE TABLE patient (
  resource JSONB
);
```

**Lookup queries used internally:**

```sql
-- Find practitioner by email
SELECT resource FROM practitioner
WHERE resource @> jsonb_build_object(
  'telecom', jsonb_build_array(
    jsonb_build_object('system', 'email', 'value', $1)
  )
) LIMIT 1;

-- Find patient by email
SELECT resource FROM patient
WHERE resource->>'email' = $1 LIMIT 1;
```

---

## 🚀 Setup Guide

### Prerequisites

- [n8n](https://n8n.io/) (self-hosted)
- [Ollama](https://ollama.com/) with `gemma3:4b` pulled
- PostgreSQL with FHIR-structured patient/practitioner data
- [ElevenLabs](https://elevenlabs.io/) account (for STT and TTS workflows)

### Steps

**1. Import workflows into n8n**

Go to **Settings → Import Workflow** and import each `.json` file.

**2. Configure credentials in n8n**

| Credential | Where used | Notes |
|---|---|---|
| PostgreSQL | `verifyPractitioner`, `verifyPatient` | Your Cloud SQL or local Postgres |
| Ollama API | `gemma3` node | Default: `http://localhost:11434` |
| ElevenLabs API Key | `speech2Text`, `text2Speech` | Set as `xi-api-key` header |

**3. (Optional) Google Cloud SQL — use the Auth Proxy**

If your PostgreSQL is hosted on Google Cloud SQL, run the proxy locally before starting n8n:

```powershell
.\cloud-sql-proxy.exe YOUR_PROJECT:YOUR_REGION:YOUR_INSTANCE --port=5432
```

Then set Postgres host to `127.0.0.1` in your n8n credential.

**4. Activate workflows and test**

Use the example requests above or import into Postman/Insomnia.

---

## 🔐 Credential Placeholders

The workflow JSON files contain the following placeholders. Replace them with your actual values inside n8n:

| Placeholder | Replace with |
|---|---|
| `YOUR_ELEVENLABS_API_KEY` | Your ElevenLabs API key |
| `YOUR_POSTGRES_CREDENTIAL_ID` | Your n8n Postgres credential ID |
| `YOUR_OLLAMA_CREDENTIAL_ID` | Your n8n Ollama credential ID |
| `YOUR_N8N_INSTANCE_ID` | Your n8n instance ID |

> ⚠️ Never commit real API keys or credentials to version control.

---

## 📄 License

This project is for educational and portfolio purposes.
