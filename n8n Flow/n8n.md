# n8n Flow: Automated Bug Report Generator from Error Logs

## Problem Statement

Build an `n8n` workflow that automatically generates a structured bug report whenever a new error log is provided. The workflow should read the error log, send it to an LLM, and return a formatted bug report with:

- `Title`
- `Severity`
- `Steps to Reproduce`
- `Expected Result`
- `Actual Result`

The solution must avoid hallucinations and use only the information available in the log.

## Context

The QA team receives error logs from the application's monitoring system and currently writes bug reports manually. This workflow reduces that manual effort by converting a raw log entry into a clean, tester-friendly bug report.

## Workflow Requirements

1. Use a `Manual Trigger` or `Chat Trigger` node to accept the error log as input.
2. Connect the input to an `AI Agent` or `LLM` node.
3. Use a Groq model such as `llama-3.3-70b-versatile`, or another free model if needed.
4. Define a strict system prompt with the role of `QA Engineer`.
5. Add the anti-hallucination rule: `Use ONLY the log data provided. Mark unknown fields as [UNKNOWN].`
6. Output the final structured bug report using `Respond to Chat` or a `Set` node.

## Recommended n8n Flow

```text
Manual Trigger / Chat Trigger
  -> Set Input
  -> AI Agent or Basic LLM Chain
  -> Set Output
  -> Respond to Chat / Final Output
```

## Best Practical Version

The simplest and most presentation-friendly version is:

1. `Chat Trigger`
2. `Set` node
3. `AI Agent` node
4. `Groq Chat Model` node
5. `Respond to Chat`

If `Chat Trigger` is not convenient, use:

1. `Manual Trigger`
2. `Set` node with a sample `error_log`
3. `AI Agent` or `Basic LLM Chain`
4. `Set` node for formatted response

## Node-by-Node Design

### 1. Trigger Node

Use one of these options:

- `Chat Trigger` if you want to paste logs directly into the n8n chat UI
- `Manual Trigger` if you want a simple demo workflow

For a classroom or demo submission, `Manual Trigger` is often easier because it is stable and easy to screenshot.

### 2. Set Input Node

Add a `Set` node after the trigger and create a field called `error_log`.

Example value:

```text
[ERROR] 2026-02-15 14:32:01 | POST /api/checkout | 500 Internal Server Error | user_id=4521 | cart_items=3 | payment_gateway_timeout after 30s
```

If using `Chat Trigger`, map the incoming message into `error_log` so the rest of the workflow uses one consistent field name.

Suggested output of this node:

```json
{
  "error_log": "[ERROR] 2026-02-15 14:32:01 | POST /api/checkout | 500 Internal Server Error | user_id=4521 | cart_items=3 | payment_gateway_timeout after 30s"
}
```

### 3. LLM Model Node

Connect a `Groq Chat Model` node or equivalent LLM node.

Suggested model:

- `llama-3.3-70b-versatile`

If that exact model is unavailable in your n8n setup, use any accessible free chat model and mention the substitution in your notes.

### 4. AI Agent or LLM Chain Node

Use an `AI Agent` node or a simple chain node that sends the `error_log` to the model with a fixed system prompt.

The goal is not tool use. This is a clean prompt-to-output workflow, so keep it simple and deterministic.

## Recommended System Prompt

Use this as the system prompt in the AI node:

```text
You are a QA Engineer who converts raw application error logs into structured bug reports.

Use ONLY the log data provided by the user.
Do NOT invent missing details.
If any field cannot be determined from the log, write [UNKNOWN].

Return the result in exactly this format:

Title: "<short bug title>"
Severity: <Low | Medium | High | Critical | [UNKNOWN]>
Steps to Reproduce: <clear reproduction step based only on the log>
Expected Result: <expected successful behavior inferred from the request and status, otherwise [UNKNOWN]>
Actual Result: <actual failure observed in the log>

Rules:
- Keep the wording concise and professional.
- Do not add explanations outside the requested format.
- Do not include markdown bullets.
- Do not include fields that were not requested.
```

## Recommended User Prompt

Use the `error_log` field as the user message:

```text
Generate a bug report from this error log:

{{$json.error_log}}
```

## Expected Output Format

For the sample log:

```text
[ERROR] 2026-02-15 14:32:01 | POST /api/checkout | 500 Internal Server Error | user_id=4521 | cart_items=3 | payment_gateway_timeout after 30s
```

The expected result is:

```text
Title: "Checkout API returns 500 due to payment gateway timeout"
Severity: High
Steps to Reproduce: Send a POST request to /api/checkout with a valid cart containing 3 items.
Expected Result: The checkout API should return a successful order confirmation response.
Actual Result: The API returned 500 Internal Server Error after a payment gateway timeout of 30 seconds.
```

## Why This Output Is Acceptable

- `POST /api/checkout` gives the action and endpoint.
- `500 Internal Server Error` indicates a server-side failure.
- `payment_gateway_timeout after 30s` provides the apparent cause.
- The expected result is a safe inference from the failed checkout action.
- No extra facts are invented beyond what the log strongly implies.

If your instructor wants even stricter handling, replace the expected result with:

```text
Expected Result: [UNKNOWN]
```

That version is even more conservative.

## Suggested Severity Logic

If the model needs guidance, this is a safe rule of thumb:

- `Critical` for system-wide outage, data loss, or security failure clearly visible in the log
- `High` for failed payment, checkout, login, or core API failure
- `Medium` for partial feature failure
- `Low` for minor or non-blocking issues
- `[UNKNOWN]` if the impact cannot be inferred

This logic should only be used if the log supports it.

## Example n8n Configuration Notes

### Option A: Chat Demo Flow

```text
Chat Trigger
  -> Set
     error_log = {{$json.chatInput}}
  -> AI Agent
     System Prompt = QA Engineer prompt
     User Message = Generate a bug report from this error log: {{$json.error_log}}
  -> Respond to Chat
```

### Option B: Manual Demo Flow

```text
Manual Trigger
  -> Set
     error_log = [sample log text]
  -> AI Agent or Basic LLM Chain
  -> Set
     bug_report = {{$json.text || $json.output || $json.response}}
```

The exact final expression depends on the output field name produced by your chosen AI node.

## Anti-Hallucination Control

This is one of the most important grading points. To make the workflow safer:

- keep the prompt strict
- request only the required fields
- instruct the model to use `[UNKNOWN]` for missing data
- avoid asking for root cause unless the log explicitly provides it
- avoid asking for stack traces, screenshots, or modules not present in the log

## Evaluation Criteria Mapping

### 1. Approach

This workflow uses the correct structure:

- trigger to receive input
- `Set` node to normalize the log field
- LLM node to transform the log into a bug report
- final output node to display the response

### 2. Working Flow

The flow is lightweight and should run successfully with one log entry at a time. It does not depend on external parsers or custom code for the core use case.

### 3. Output Quality

The system prompt enforces:

- structured formatting
- concise bug-report writing
- no hallucinated details
- fallback to `[UNKNOWN]`

## Submission Checklist

Before submitting, make sure you have:

1. built the workflow in `n8n`
2. tested it with the sample error log
3. verified the output matches the required format
4. exported the workflow JSON or captured a screenshot
5. pushed the project to GitHub
6. submitted the repository link

## Final Short Explanation

This workflow accepts a raw error log, passes it to an LLM configured as a QA Engineer, and returns a clean bug report in a fixed format. The anti-hallucination prompt ensures the model only uses the provided log data and marks missing information as `[UNKNOWN]`.

## Optional Next Step

If we continue from here, the strongest follow-up is to create the actual `n8n` workflow JSON skeleton that matches this design so you can import it directly.
