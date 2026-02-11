# Playbook Brain Integration Specification

> **Status**: APPROVED
> **Version**: 1.0.0
> **Target System**: Brain API (HuggingFace Spaces)
> **Consumer**: Playbook Client

---

## 1. System Overview

The **Playbook Client** is a production-ready application that orchestrates document processing workflows. It communicates with the **Brain API** (an external service running on HuggingFace Spaces) to perform intelligent extraction, training, and execution tasks.

### Request/Response Lifecycle

The interaction follows a strict sequence:

1.  **Startup Probe**: Playbook validates the Brain connection immediately upon launch.
2.  **Upload & Extract**: User uploads a file → Playbook sends `POST /extract` → Brain returns structured data.
3.  **Train**: User maps data columns → Playbook sends `POST /train` → Brain confirms mapping logic.
4.  **Finish Training**: User finalizes mapping → Playbook sends `POST /finish_training` → Brain returns execution plan.
5.  **Execute**: User runs the plan on new files → Playbook sends `POST /execute` → Brain processes files and returns results.
6.  **Opportunities**: Playbook periodically fetches `GET /opportunities` to display suggestions.

All communication is **synchronous HTTP** (JSON over REST).

---

## 2. Environment Variables

The Playbook Client requires the following environment variables to connect to the Brain. These are loaded from `playbook/credentials.py` (preferred) or the process environment.

| Variable | Description | Example |
| :--- | :--- | :--- |
| `BRAIN_API_URL` | The base URL of the Brain API. | `https://your-space-name.hf.space` |
| `BRAIN_API_KEY` | The secret key for authentication. | `gsk_...` |

**Authentication**:
All requests to the Brain must include the following header:
```http
Authorization: Bearer <BRAIN_API_KEY>
Content-Type: application/json
```

---

## 3. API Endpoint Contract

The Brain API must implement the following endpoints exactly as specified.

### 3.1 Health Check (Startup Probe)

**Note**: The Playbook Client does **not** use a dedicated `GET /health` endpoint. Instead, it performs a synthetic `POST /extract` request to verify connectivity and schema validation.

-   **Path**: `/extract`
-   **Method**: `POST`
-   **Purpose**: Validates that the Brain is reachable and authorized.

#### Request (Probe)
```json
{
  "session_id": "health-check-probe",
  "file": {
    "filename": "health_probe.txt",
    "content_b64": "SEVBTFRIIENIRUNL"  // Base64 for "HEALTH CHECK"
  }
}
```

#### Expected Response
The Brain **must** return a valid extraction response, even for this dummy payload.
```json
{
  "status": "ok",
  "session_id": "health-check-probe",
  "extracted": [],
  "ui_state": {
    "mode": "enterprise",
    "visual_action": "preview"
  }
}
```

---

### 3.2 Extract Data

-   **Path**: `/extract`
-   **Method**: `POST`
-   **Purpose**: Ingests a file and returns structured tabular data.

#### Request Schema
```json
{
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "file": {
    "filename": "invoice_feb.pdf",
    "content_b64": "<base64_encoded_string>"
  }
}
```

#### Response Schema
```json
{
  "status": "ok",
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "extracted": [
    {
      "row": 1,
      "Date": "2024-02-10",
      "InvoiceNumber": "INV-001",
      "Customer": "ACME Corp",
      "Amount": "1200.00"
    },
    {
      "row": 2,
      "Date": "2024-02-11",
      "InvoiceNumber": "INV-002",
      "Customer": "Globex",
      "Amount": "850.50"
    }
  ],
  "ui_state": {
    "mode": "enterprise",
    "visual_action": "preview"
  }
}
```

**Validation Rules**:
-   `extracted` must be a list of objects (rows).
-   Keys in the row objects represent column headers.
-   `ui_state` is required.

---

### 3.3 Train Mapping

-   **Path**: `/train`
-   **Method**: `POST`
-   **Purpose**: Registers user-defined mappings between source columns and target applications.

#### Request Schema
```json
{
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "mapping": [
    {
      "source": { "file": "invoice_feb.pdf", "field": "Amount" },
      "target": { "app": "excel", "field": "C" }
    },
    {
      "source": { "file": "invoice_feb.pdf", "field": "Customer" },
      "target": { "app": "salesforce", "field": "AccountName" }
    }
  ],
  "example_execution": {
    "preview": true
  }
}
```

#### Response Schema
```json
{
  "status": "ok",
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "summary": "I will copy 'Amount' to Excel column C and 'Customer' to Salesforce AccountName.",
  "ui_state": {
    "mode": "enterprise",
    "visual_action": "morph"
  }
}
```

---

### 3.4 Finish Training

-   **Path**: `/finish_training`
-   **Method**: `POST`
-   **Purpose**: Finalizes the training session and generates an execution plan.

#### Request Schema
```json
{
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "confirm": true
}
```

#### Response Schema
```json
{
  "status": "ok",
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "execution_plan": {
    "steps": [
      "Extract Amount",
      "Open Excel",
      "Paste to Column C",
      "Save"
    ]
  },
  "message": "Training complete. Ready to execute."
}
```

---

### 3.5 Execute

-   **Path**: `/execute`
-   **Method**: `POST`
-   **Purpose**: Runs the trained playbook on a batch of files.

#### Request Schema
```json
{
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "files": [
    {
      "filename": "invoice_march.pdf",
      "content_b64": "<base64_encoded_string>"
    }
  ],
  "execution_options": {
    "dry_run": false
  }
}
```

#### Response Schema
```json
{
  "status": "ok",
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "results": [
    {
      "file": "invoice_march.pdf",
      "row": 1,
      "status": "done",
      "details": "Copied 1500.00 to Excel C1"
    }
  ]
}
```

---

### 3.6 Get Opportunities

-   **Path**: `/opportunities`
-   **Method**: `GET`
-   **Purpose**: Retrieves suggested automations or insights.

#### Response Schema
```json
{
  "status": "ok",
  "opportunities": [
    {
      "id": "op-101",
      "title": "Automate Invoice Processing",
      "summary": "I noticed you handle many invoices manually. I can automate this flow."
    }
  ]
}
```

---

## 4. Startup & Runtime Requirements

The Brain service MUST:

1.  **Framework**: Run as a **FastAPI** application (or compatible Python server).
2.  **Port**: Listen on port **7860** (Standard for HuggingFace Spaces).
3.  **Latency**: Respond to `/extract` (probe) in **< 10 seconds**.
4.  **Schema Compliance**: **Never** return `null` for required fields (`status`, `session_id`, `ui_state`). Always return structured JSON.
5.  **Persistence**: Maintain `session_id` state in-memory during the lifecycle of the user session.

---

## 5. Docker Compatibility

The Brain container must be built with the following specifications:

-   **Base Image**: `python:3.10` or higher.
-   **Dependencies** (`requirements.txt`):
    -   `fastapi`
    -   `uvicorn`
    -   `pydantic`
    -   `python-multipart`
-   **Startup Command**:
    ```bash
    uvicorn app:app --host 0.0.0.0 --port 7860
    ```

---

## 6. File Ingestion Contract

The Brain must accept base64-encoded content for:

-   **PDF**: Application/pdf (extract text/tables).
-   **Excel**: .xlsx (extract rows).
-   **CSV**: .csv (extract rows).
-   **Images**: .png/.jpg (OCR extraction).
-   **Text**: .txt (raw content).

The `extracted` array in the response should normalize this data into a flat list of JSON objects (key-value pairs) representing rows/records.

---

## 7. Training Session Contract

-   **Session Scope**: The `session_id` provided by the client must be the key for all state.
-   **Mapping Logic**: Store the `source` (file/field) to `target` (app/field) rules.
-   **Memory**: In-memory storage (e.g., Python `dict`) is acceptable for the session duration. Global persistence is not required.

---

## 8. Execution Contract

When `/execute` is called:

1.  Retrieve the mappings stored for the given `session_id`.
2.  Process the new `files` (base64).
3.  Apply the transformation (e.g., "Extract Field X").
4.  Simulate the "Action" (e.g., "Pasting to Excel").
5.  Return a result log for each processed item.

---

## 9. Failure Behavior

If the Brain encounters an error (e.g., unreadable file, invalid session):

1.  Return a **non-200** HTTP status code (e.g., 400, 500).
2.  Return a JSON body with details:
    ```json
    {
      "detail": "Session not found"
    }
    ```
3.  **Critical**: Do NOT return a 200 OK with an error message inside the body. The Playbook client relies on HTTP status codes for error handling.

---

## 10. Verification

The Playbook Client validates the integration on startup.
**Success Criteria**:
-   `POST /extract` with dummy data returns HTTP 200.
-   Response JSON contains `"status": "ok"` and `"session_id"`.
-   Response JSON contains `"ui_state"`.

**Failure**:
-   If any of these conditions are not met, Playbook will terminate immediately and log a `KEY_VALIDATION_BLOCKER.md`.

---
**End of Specification**
