# Cloud Trace Setup Guide (ADK + Notebook)

This guide documents the exact step-by-step setup for Cloud Traceability in this project, based on `main.ipynb`.

## 1. Prerequisites

- Python 3.12+
- Google Cloud SDK (`gcloud`) installed
- A Google Cloud project with Vertex AI and Cloud Trace enabled
- Access to at least one allowed model in your org policy

This project currently uses:
- Project: `dev-mq-tech-transfer`
- Recommended model: `gemini-2.5-flash`

## 2. Install Dependencies

From the project root:

```powershell
uv sync
```

## 3. Authenticate to Google Cloud

Authenticate ADC (Application Default Credentials):

```powershell
gcloud auth application-default login
```

Optional verification:

```powershell
gcloud auth application-default print-access-token
```

## 3.1 Enable services - VERY IMPORTANT STEP
```powershell
gcloud enable services cloudtrace.googleapis.com aiplatform.googleapis.com --project=<your_project_id>
```


## 4. Configure Environment Variables

Create or update `.env` with:

```env
PROJECT_ID=dev-mq-tech-transfer
PROJECT_NUMBER=35209068002
GOOGLE_CLOUD_LOCATION=global
GOOGLE_GENAI_USE_VERTEXAI=true
OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=EVENT_ONLY
MODEL_NAME=gemini-2.5-flash
```

Notes:
- `MODEL_NAME` must be allowed by your org policy.
- If your org policy blocks a model, you will get `FAILED_PRECONDITION` errors.

## 5. Notebook Flow (`main.ipynb`)

Run cells in order.

### Cell 1 - Import agent type

```python
from google.adk.agents.llm_agent import Agent
```

### Cell 2 - Initialize Cloud Trace providers

This is the key telemetry setup:

```python
import google.auth
from google.adk.telemetry import google_cloud
from google.adk.telemetry.setup import maybe_set_otel_providers

credentials, project_id = google.auth.default()
resource = google_cloud.get_gcp_resource(project_id=project_id)
hooks = google_cloud.get_gcp_exporters(
	enable_cloud_tracing=True,
	google_auth=(credentials, project_id),
)
maybe_set_otel_providers(otel_hooks_to_setup=[hooks], otel_resource=resource)
```

### Cell 3 - Create ADK agent

Use an allowed model:

```python
root_agent = Agent(
	name="capital_agent",
	model="gemini-2.5-flash",
	description="Answers geography questions.",
	instruction="Answer the user's question directly and concisely.",
)
```

### Cell 4/5 - Run the agent with `InMemoryRunner`

Define `ask(...)` and execute:

```python
await ask("What is the capital of France?")
await ask("What is the capital of India?")
```

This should produce model responses and telemetry spans.

## 6. Verify Traces in Google Cloud

1. Open Google Cloud Console.
2. Go to Trace Explorer.
3. Filter by your project (`dev-mq-tech-transfer`).
4. Look for recent traces generated after notebook execution.

If traces do not appear immediately, wait 30-120 seconds and refresh.


```powershell
".venv/Scripts/python.exe" agent_runner.py
```

## 8. Troubleshooting

### A) `FAILED_PRECONDITION` for disallowed model

Cause:
- Org policy (`constraints/vertexai.allowedGenAIModels`) does not allow the configured model.

Fix:
- Set `MODEL_NAME` to an allowed model (for this project, `gemini-2.5-flash`).
- Or ask org admin to allow the model.

### B) `ModuleNotFoundError: No module named 'vertexai'`

Cause:
- Notebook cell imports `vertexai` directly, but package is not installed.

Fix:
- Prefer using `gcloud` CLI checks instead of Python `vertexai` import for policy inspection.

### C) `Reauthentication is needed`

Fix:

```powershell
gcloud auth application-default login
```

### D) `Failed to export span batch code: 400`

Cause:
- Export destination/config mismatch (often OTLP endpoint config conflicts).

Fix options:
- Keep only Google Cloud trace exporter settings.
- Remove conflicting OTLP env vars unless explicitly needed.

## 9. Quick Validation Checklist

- `uv sync` completed
- `gcloud auth application-default login` completed
- `.env` has valid project/model settings
- Notebook Cell 2 initializes telemetry without errors
- Agent calls return answers
- New traces visible in Cloud Trace

