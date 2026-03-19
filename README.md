# Clinical SOAP Note Generator — n8n Workflows

An AI-powered clinical documentation system built on [n8n](https://n8n.io/) that automatically generates structured SOAP (Subjective, Objective, Assessment, Plan) notes from clinical input. Supports three input modes: **Text-to-Text (TTT)**, **Speech-to-Text (STT)**, and **Text-to-Speech (TTS)**.

---

## Architecture Overview

```
Input (Text or Audio)
        │
        ▼
  Webhook Trigger
        │
        ├──► Verify Practitioner (PostgreSQL / FHIR)
        │
        ├──► Verify Patient (PostgreSQL / FHIR)
        │
        ▼
   CodeParse Node  ←── Merges patient + practitioner + clinical input
        │
        ▼
  Gemma 3 (Ollama) via LLM Chain
        │
        ▼
   Clean Output JSON
        │
        ▼
  Response (JSON / Audio)
```

---

## Tech Stack

| Component | Tool |
|---|---|
| Workflow Automation | n8n |
| LLM | Gemma 3 4B (via Ollama) |
| Database | PostgreSQL (FHIR-structured) |
| Speech-to-Text | ElevenLabs Scribe v1 |
| Text-to-Speech | ElevenLabs Multilingual v2 |

---

## Workflows

### 1. TTT — Text to Text (`clinicalNote(TTT).json`)

Accepts raw clinical text and returns a structured SOAP note as JSON.

**Webhook Trigger**
```
POST /webhook/clinical-note/start
Content-Type: application/json
```

**Request Body**
```json
{
  "provider_email": "mccoy@enterprise.med",
  "patient_email": "roydon@asendhealth.com",
  "appointment_id": "APPT-1234",
  "mode": "text",
  "text_input": "34-year-old male with 4 days of fever, cough, sore throat, congestion, headache, chills, and fatigue. Cough initially dry, now mildly productive. Denies chest pain, shortness of breath, nausea, vomiting, or diarrhea. Partial relief with paracetamol. Recent sick contacts at work. Past medical history none. Medications paracetamol 500mg as needed. No known drug allergies. Vitals: Temp 37.8 C, BP 118/74, HR 88, RR 16, SpO2 98 percent on room air. Physical exam shows mild pharyngeal erythema, lungs clear, heart normal, abdomen soft and non-tender. COVID and rapid strep negative.",
  "metadata": {
    "source": "patient",
    "language": "en-US"
  }
}
```

**Response**
```json
{
  "status": "success",
  "code": 200,
  "message": "Clinical SOAP note generated successfully.",
  "data": {
    "soap_note": {
      "patient_name": "Roydon Sequiera",
      "patient_dob": "1971-03-10",
      "patient_id": "roydon",
      "visit_date": "2025-12-16",
      "practitioner": "Leonard H. McCoy",
      "practitioner_credentials": "MD",
      "chief_compliant": "Patient presents with cough, congestion, and subjective fever.",
      "subjective": {
        "hpi_narrative": "Patient reports experiencing a persistent cough, nasal congestion, and subjective fever. Patient denies chest pain or shortness of breath. Reports feeling generally unwell.",
        "symptoms_reported": "Cough, nasal congestion, subjective fever, denies chest pain, denies shortness of breath.",
        "progress_since_last": "NA",
        "compliance_notes": "NA"
      },
      "objective": {
        "bp": "118/74",
        "heart_rate": "88",
        "temperature": "37.8 C",
        "o2_saturation": "98 percent on room air",
        "physical_exam_findings": "Exam reveals clear lungs bilaterally. Mucous membranes are moist. No rash or lesions noted. Pharyngeal exam reveals mild erythema.",
        "lab_results": "None",
        "imaging_results": "None"
      },
      "assessment": {
        "primary_diagnosis": "Upper Respiratory Infection",
        "icd_10_code": "J06.9",
        "problem_list": "Upper Respiratory Infection",
        "assessment_summary": "Patient presents with symptoms consistent with an upper respiratory infection. The elevated temperature and associated symptoms support this diagnosis."
      },
      "plan": {
        "medication_plan": "Recommend symptomatic treatment with acetaminophen 325mg every 6 hours as needed for fever and discomfort. Instruct patient to use saline nasal spray for congestion.",
        "intervention_details": "Educated patient on proper hydration and rest.",
        "follow_up_plan": "Return to clinic if symptoms worsen or fail to improve within 72 hours. Instruct patient to monitor temperature and cough frequency.",
        "referrals": "None"
      }
    }
  },
  "timestamp": "2025-12-16T00:00:00.000Z"
}
```

---

### 2. STT — Speech to Text (`clinicalNote(STT).json`)

Accepts an audio recording of a clinical encounter, transcribes it using ElevenLabs Scribe, and returns the same structured SOAP note JSON as TTT.

**Webhook Trigger**
```
POST /webhook/clinical-note-audio
Content-Type: multipart/form-data
```

**Request Fields**

| Field | Type | Description |
|---|---|---|
| `data` | File (audio) | The clinical audio recording (e.g. `.wav`, `.mp3`) |
| `practitioner_email` | String | Practitioner's registered email |
| `patient_email` | String | Patient's registered email |

**Example (cURL)**
```bash
curl -X POST http://localhost:5678/webhook/clinical-note-audio \
  -F "data=@Recording.wav" \
  -F "practitioner_email=mccoy@enterprise.med" \
  -F "patient_email=roydon@asendhealth.com"
```

**Response** — Same SOAP note JSON structure as TTT above.

---

### 3. TTS — Text to Speech (`clinicalNote(TTS).json`)

Accepts raw clinical text, generates a SOAP note, then converts a spoken clinical summary into an audio file returned directly in the response.

**Webhook Trigger**
```
POST /webhook/clinical-note-tts
Content-Type: application/json
```

**Request Body** — Same structure as TTT.

**Response**
```
Content-Type: audio/mpeg
Content-Disposition: inline; filename="clinical_note.mp3"
```

Returns a binary `.mp3` audio file containing a spoken summary of:
- Chief Complaint
- History of Present Illness
- Assessment
- Follow-up Plan

---

## Node Flow Detail

### Shared nodes across all 3 workflows

| Node | Purpose |
|---|---|
| `clinicalNote` (Webhook) | Entry point — receives the HTTP request |
| `verifyPractitioner` | Queries PostgreSQL FHIR `practitioner` table by email |
| `verifyPatient` | Queries PostgreSQL FHIR `patient` table by email |
| `codeParse` | Normalizes patient + practitioner + clinical input into a single payload |
| `promptChain` | Sends structured prompt to Gemma 3 via Ollama |
| `cleanOutputJSON` | Strips markdown from LLM output, parses clean JSON |
| `respondSuccess` / `respondSucessAudio` | Returns final response to caller |

### STT-specific nodes

| Node | Purpose |
|---|---|
| `speech2Text` | Sends audio to ElevenLabs Scribe v1 API for transcription |
| `extractTranscription` | Extracts the `text` field from Scribe response |
| `merge` | Combines the verified patient/practitioner data with the transcription |

### TTS-specific nodes

| Node | Purpose |
|---|---|
| `cleanOutputJSON` | Also extracts chief complaint, HPI, assessment, and plan for the spoken script |
| `text2Speech` | Sends the script to ElevenLabs TTS API and receives audio |

---

## Database Schema (FHIR)

The workflows rely on a PostgreSQL database with FHIR-structured JSON columns:

```sql
-- Practitioners table
CREATE TABLE practitioner (
  resource JSONB
);

-- Patients table
CREATE TABLE patient (
  resource JSONB
);
```

Patient and practitioner records are stored as FHIR R4 JSON resources. The workflows look up records by `email` stored in the `telecom` array.

---

## Setup & Configuration

### Prerequisites
- [n8n](https://n8n.io/) (self-hosted)
- [Ollama](https://ollama.com/) running locally with `gemma3:4b` pulled
- PostgreSQL with FHIR-structured patient and practitioner data
- [ElevenLabs](https://elevenlabs.io/) account (for STT and TTS workflows)

### Steps

1. Import the workflow JSON files into your n8n instance via **Settings → Import Workflow**
2. Set up credentials in n8n:
   - **Postgres**: add your Cloud SQL or local PostgreSQL connection
   - **Ollama**: point to your local Ollama instance (default: `http://localhost:11434`)
   - **ElevenLabs**: add your API key (`xi-api-key`) in the HTTP Request nodes
3. Activate the workflows
4. Test using the example requests above

> **Note:** If connecting to Google Cloud SQL, use the [Cloud SQL Auth Proxy](https://cloud.google.com/sql/docs/postgres/connect-auth-proxy):
> ```powershell
> .\cloud-sql-proxy.exe YOUR_PROJECT:YOUR_REGION:YOUR_INSTANCE --port=5432
> ```
> Then set your Postgres host to `127.0.0.1` in n8n.

---

## Environment Placeholders

The following placeholders appear in the workflow JSON files and must be replaced with real values in your n8n credential settings:

| Placeholder | What to replace with |
|---|---|
| `YOUR_ELEVENLABS_API_KEY` | Your ElevenLabs API key |
| `YOUR_POSTGRES_CREDENTIAL_ID` | Your n8n Postgres credential ID |
| `YOUR_OLLAMA_CREDENTIAL_ID` | Your n8n Ollama credential ID |
| `YOUR_N8N_INSTANCE_ID` | Your n8n instance ID |
