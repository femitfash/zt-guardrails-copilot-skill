# ZeroTrusted.ai Guardrails — Copilot UI Integration

Claude Code skill for adding ZeroTrusted.ai guardrails (PII detection, anonymization,
deanonymization, and hallucination checking) to any AI copilot or chat UI. Includes
complete React components, API routes, and the full privacy-safe conversation flow.

> **Prerequisite**: Familiarity with the base guardrails API. See `@zt-guardrails-api.md`
> for endpoint reference and authentication details.

---

## 1. Architecture Overview

### Guardrailed Copilot Flow

```
User types message
  -> PII Pre-Check (detect-sensitive-keywords)
     -> PII found + mode=block  -> Block message, show warning
     -> PII found + mode=warn   -> Show modal with detected entities
        -> User clicks "Anonymize"
           -> Call anonymize-sensitive-keywords
           -> Show original vs anonymized mappings
           -> User reviews + clicks "Use Anonymized Content"
              -> Replace user message in chat with anonymized text
              -> Send ONLY anonymized text to LLM (history scrubbed)
              -> LLM responds with anonymized placeholders
              -> Auto-deanonymize response (restore original values)
        -> User clicks "Send Anyway" -> bypass, send as-is
     -> No PII  -> Send directly to LLM
  -> LLM responds via SSE streaming
  -> Post-response: Hallucination / Reliability check
     -> Auto (runs immediately) or Manual (button click)
     -> Score >= 70 = reliable, < 70 = flagged
```

### File Structure

```
src/
  app/
    api/
      copilot/
        pii-check/route.ts         # Detect PII in user input
        anonymize/route.ts          # Anonymize detected PII
        deanonymize/route.ts        # Restore original values
        hallucination-check/route.ts # Evaluate response reliability
        route.ts                    # Main copilot streaming endpoint
  features/
    copilot/
      components/
        CopilotPanel.tsx            # Main UI with guardrails integrated
  shared/
    lib/
      guardrails.ts                 # ZeroTrusted API config (reads from env)
```

### Environment Variables

```bash
# .env.local
ZT_GUARDRAILS_API_KEY=zt-your-api-key-here
ZT_ENCRYPTED_PROVIDER_KEY=your-encrypted-provider-key
```

---

## 2. Shared Config

```typescript
// src/shared/lib/guardrails.ts

export const GUARDRAILS_URL = process.env.ZT_GUARDRAILS_URL
  || "https://dev-guardrails.zerotrusted.ai/api/v3";

export const GUARDRAILS_TOKEN = process.env.ZT_GUARDRAILS_API_KEY || "";

export const HALLUCINATION_URL = process.env.ZT_HALLUCINATION_URL
  || "https://dev-agents.zerotrusted.ai/api/v1/responses/evaluate-reliability";

export const HALLUCINATION_TOKEN = process.env.ZT_AGENTS_API_KEY
  || process.env.ZT_GUARDRAILS_API_KEY || "";

export const PII_ENTITY_TYPES = [
  "email", "email address", "person", "organization",
  "phone number", "address", "passport number", "credit card number",
  "social security number", "date time", "date of birth",
  "bank account number", "driver's license number",
  "tax identification number", "ip address", "iban", "cvv",
  "health insurance id number", "medical condition",
].join(", ");
```

---

## 3. API Routes

### 3.1 PII Detection Route

```typescript
// src/app/api/copilot/pii-check/route.ts
import { NextRequest } from "next/server";
import { GUARDRAILS_URL, GUARDRAILS_TOKEN, PII_ENTITY_TYPES } from "@/shared/lib/guardrails";

export async function POST(request: NextRequest) {
  const { text, api_key } = await request.json();
  if (!text) return Response.json({ error: "text is required" }, { status: 400 });

  try {
    const scanText = text.length > 10000 ? text.slice(0, 10000) : text;
    const token = api_key || GUARDRAILS_TOKEN;

    const formData = new FormData();
    formData.append("user_prompt", scanText);
    formData.append("pii_entity_types", PII_ENTITY_TYPES);

    const res = await fetch(`${GUARDRAILS_URL}/detect-sensitive-keywords`, {
      method: "POST",
      headers: { "X-API-Key": token },
      body: formData,
    });

    if (!res.ok) {
      const errText = await res.text();
      return Response.json({ error: `PII check failed: ${res.status} ${errText}` }, { status: 502 });
    }

    const data = await res.json();
    const piiEntities: [string, string][] = data?.privacy_result?.pii_entities || [];

    return Response.json({
      has_pii: piiEntities.length > 0,
      detections: piiEntities.map(([text, entityType]: [string, string]) => ({
        entity_type: entityType,
        text,
      })),
    });
  } catch (err: unknown) {
    const msg = err instanceof Error ? err.message : "Unknown error";
    return Response.json({ error: `PII check failed: ${msg}` }, { status: 500 });
  }
}
```

### 3.2 Anonymization Route

```typescript
// src/app/api/copilot/anonymize/route.ts
import { NextRequest } from "next/server";
import { GUARDRAILS_URL, GUARDRAILS_TOKEN, PII_ENTITY_TYPES } from "@/shared/lib/guardrails";

export const maxDuration = 30; // Extend timeout for Vercel

async function callAnonymize(scanText: string, token: string): Promise<Response> {
  const formData = new FormData();
  formData.append("user_prompt", scanText);
  formData.append("pii_entity_types", PII_ENTITY_TYPES);
  formData.append("response_language", "EN");

  return fetch(`${GUARDRAILS_URL}/anonymize-sensitive-keywords`, {
    method: "POST",
    headers: { "X-API-Key": token },
    body: formData,
  });
}

export async function POST(request: NextRequest) {
  const { text, api_key } = await request.json();
  if (!text) return Response.json({ error: "text is required" }, { status: 400 });

  try {
    const scanText = text.length > 10000 ? text.slice(0, 10000) : text;
    const token = api_key || GUARDRAILS_TOKEN;

    // Retry once on failure (endpoint can be intermittent)
    let res = await callAnonymize(scanText, token);
    if (!res.ok) res = await callAnonymize(scanText, token);

    if (!res.ok) {
      return Response.json({ error: `Anonymize failed: ${res.status}` }, { status: 502 });
    }

    const data = await res.json();
    const privacyResult = data?.privacy_result;

    return Response.json({
      anonymized_text: privacyResult?.processed_text || text,
      original_text: privacyResult?.original_text || text,
      mappings: (privacyResult?.anonymized_keywords_mapping || []).map(
        (m: { original: string; anonymized: string }) => ({
          original: m.original,
          anonymized: m.anonymized,
        })
      ),
    });
  } catch (err: unknown) {
    const msg = err instanceof Error ? err.message : "Unknown error";
    return Response.json({ error: `Anonymize failed: ${msg}` }, { status: 500 });
  }
}
```

### 3.3 Deanonymization Route

```typescript
// src/app/api/copilot/deanonymize/route.ts
import { NextRequest } from "next/server";
import { GUARDRAILS_URL, GUARDRAILS_TOKEN } from "@/shared/lib/guardrails";

export async function POST(request: NextRequest) {
  const { text, mappings, api_key } = await request.json();
  if (!text) return Response.json({ error: "text is required" }, { status: 400 });
  if (!mappings || !Array.isArray(mappings) || mappings.length === 0) {
    return Response.json({ error: "mappings array is required" }, { status: 400 });
  }

  try {
    const token = api_key || GUARDRAILS_TOKEN;

    const formData = new FormData();
    formData.append("user_prompt", text);
    formData.append("anonymized_keywords_mapping", JSON.stringify(mappings));

    const res = await fetch(`${GUARDRAILS_URL}/deanonymize-sensitive-keywords`, {
      method: "POST",
      headers: { "X-API-Key": token },
      body: formData,
    });

    if (!res.ok) {
      return Response.json({ error: `Deanonymize failed: ${res.status}` }, { status: 502 });
    }

    const data = await res.json();
    return Response.json({
      deanonymized_text: data?.privacy_result?.processed_text || text,
    });
  } catch (err: unknown) {
    const msg = err instanceof Error ? err.message : "Unknown error";
    return Response.json({ error: `Deanonymize failed: ${msg}` }, { status: 500 });
  }
}
```

### 3.4 Hallucination Check Route

```typescript
// src/app/api/copilot/hallucination-check/route.ts
import { NextRequest } from "next/server";
import { HALLUCINATION_URL, HALLUCINATION_TOKEN } from "@/shared/lib/guardrails";

const ENCRYPTED_PROVIDER_KEY = process.env.ZT_ENCRYPTED_PROVIDER_KEY || "";

export async function POST(request: NextRequest) {
  const { user_prompt, ai_response, model, api_key } = await request.json();

  if (!user_prompt || !ai_response) {
    return Response.json({ error: "user_prompt and ai_response are required" }, { status: 400 });
  }
  if (!ENCRYPTED_PROVIDER_KEY) {
    return Response.json({ error: "ZT_ENCRYPTED_PROVIDER_KEY not configured" }, { status: 501 });
  }

  const truncatedResponse = ai_response.slice(0, 3000);
  const truncatedPrompt = user_prompt.slice(0, 1000);

  try {
    const token = api_key || HALLUCINATION_TOKEN;

    const res = await fetch(`${HALLUCINATION_URL}?service=openai`, {
      method: "POST",
      headers: {
        "X-API-Key": token,
        "content-type": "application/json",
      },
      body: JSON.stringify({
        provider_api_key: ENCRYPTED_PROVIDER_KEY,
        evaluator_model: model || "gpt-4.1",
        candidate_responses: [
          { model: "assistant", response: truncatedResponse },
        ],
        user_prompt: truncatedPrompt,
        is_provider_api_key_encrypted: true,
        response_language: "EN",
      }),
    });

    if (!res.ok) {
      return Response.json({ error: `Reliability check failed: ${res.status}` }, { status: 502 });
    }

    const data = await res.json();
    let score = 50;
    let explanation = "";

    if (data.success && data.data) {
      try {
        const parsed = typeof data.data === "string" ? JSON.parse(data.data) : data.data;
        const result = Object.values(parsed).find(
          (v: unknown) => typeof v === "object" && v !== null && "score" in (v as Record<string, unknown>)
        ) as { score: string; explanation?: string } | undefined;
        if (result) {
          score = parseInt(result.score, 10) || 50;
          explanation = result.explanation || "";
        }
      } catch { /* use defaults */ }
    }

    return Response.json({
      reliable: score >= 70,
      score,
      explanation,
      details: data,
    });
  } catch (err: unknown) {
    const msg = err instanceof Error ? err.message : "Unknown error";
    return Response.json({ error: `Reliability check failed: ${msg}` }, { status: 500 });
  }
}
```

---

## 4. Message Types

```typescript
interface Message {
  id: string;
  role: "user" | "assistant";
  content: string;
  timestamp: Date;
  isStreaming?: boolean;

  // Guardrails fields
  validation?: {
    status: "scanning" | "passed" | "failed" | "skipped";
    details?: string;
  };
  reliability?: {
    score: number;
    reliable: boolean;
    checking?: boolean;
    explanation?: string;
  };
  anonymizeMappings?: { original: string; anonymized: string }[];
  deanonymized?: { text: string; checking?: boolean };
}
```

---

## 5. Safety Settings

Users can configure guardrail behavior through a settings UI:

```typescript
type SafetyMode = "block" | "warn" | "allow";
type HallucinationMode = SafetyMode | "manual";

interface SafetySettings {
  pii_detection: SafetyMode;          // "block" | "warn" | "allow"
  hallucination_check: HallucinationMode; // "block" | "warn" | "allow" | "manual"
  guardrails_api_key?: string;        // Optional user-provided key override
  agents_api_key?: string;            // Optional key for agents endpoints
}

// Default settings
const defaults: SafetySettings = {
  pii_detection: "warn",
  hallucination_check: "manual",
};

// Persist in localStorage
localStorage.setItem("safety_settings", JSON.stringify(settings));
```

### Mode Behaviors

| Setting | Block | Warn | Allow | Manual |
|---------|-------|------|-------|--------|
| **PII Detection** | Message blocked, not sent to LLM | Modal shown with anonymize option | No PII check | N/A |
| **Hallucination** | Response hidden if score < 70 | Badge shown with score | No check | Button shown for on-demand check |

---

## 6. UI Components

### 6.1 PII Scanning Animation

Displayed while the PII detection API is processing:

```tsx
const SCAN_STEPS = [
  "Scanning for Social Security Numbers",
  "Scanning for credit card numbers",
  "Scanning for email addresses",
  "Scanning for phone numbers",
  "Scanning for names and addresses",
  "Scanning for dates of birth",
  "Cross-referencing entity patterns",
  "Validating detection results",
];

function ScanningSteps() {
  const [step, setStep] = useState(0);
  useEffect(() => {
    const interval = setInterval(() => {
      setStep((s) => Math.min(s + 1, SCAN_STEPS.length - 1));
    }, 2200);
    return () => clearInterval(interval);
  }, []);
  return (
    <span className="inline-flex items-center gap-1.5">
      {SCAN_STEPS[step]}
      <span className="inline-flex gap-0.5 ml-0.5">
        <span className="w-1 h-1 rounded-full bg-current animate-bounce" style={{ animationDelay: "0ms" }} />
        <span className="w-1 h-1 rounded-full bg-current animate-bounce" style={{ animationDelay: "150ms" }} />
        <span className="w-1 h-1 rounded-full bg-current animate-bounce" style={{ animationDelay: "300ms" }} />
      </span>
    </span>
  );
}
```

### 6.2 Validation Badge

Shown below each message indicating PII scan status:

```tsx
{message.validation && (
  <div className={`flex items-center gap-1.5 mt-1.5 text-[10px] font-medium px-2 py-1 rounded-md w-fit ${
    message.validation.status === "scanning"
      ? "bg-blue-50 dark:bg-blue-900/20 text-blue-600 dark:text-blue-400"
      : message.validation.status === "passed"
        ? "bg-green-50 dark:bg-green-900/20 text-green-600 dark:text-green-400"
        : message.validation.status === "failed"
          ? "bg-red-50 dark:bg-red-900/20 text-red-600 dark:text-red-400"
          : "bg-gray-50 dark:bg-gray-800 text-gray-500 dark:text-gray-400"
  }`}>
    <span>
      {message.validation.status === "scanning" && <ScanningSteps />}
      {message.validation.status === "passed" && "Sensitive Data Validation: Passed"}
      {message.validation.status === "failed" && "Sensitive Data Validation: Failed"}
      {message.validation.status === "skipped" && "Sensitive Data Validation: Skipped"}
    </span>
    <span className="text-[9px] opacity-60">ZeroTrusted.ai</span>
  </div>
)}
```

### 6.3 PII Warning Modal

Full modal shown when PII is detected in "warn" mode:

```tsx
{piiWarning && (
  <div className="absolute inset-0 z-50 flex items-center justify-center bg-black/40 rounded-2xl">
    <div className="w-[90%] max-w-lg bg-white dark:bg-gray-900 rounded-xl border border-red-300
                    dark:border-red-700 shadow-xl max-h-[85%] flex flex-col">
      {/* Header */}
      <div className="flex items-center justify-between px-4 py-3 border-b border-red-200
                      dark:border-red-800 bg-red-50 dark:bg-red-900/20 rounded-t-xl">
        <div className="flex items-center gap-2">
          <span className="text-red-600">&#9888;</span>
          <span className="text-sm font-semibold text-red-700 dark:text-red-400">
            PII Detected in Content
          </span>
        </div>
        <span className="text-[9px] text-red-400/60">ZeroTrusted.ai</span>
      </div>

      {/* Body */}
      <div className="overflow-y-auto flex-1 p-4 space-y-3">
        {!anonymizeResult ? (
          <>
            <p className="text-sm text-gray-700 dark:text-gray-300">
              Sensitive personal information was found.
              This data has <strong>not</strong> been sent to the AI.
            </p>
            {/* Detection list */}
            {piiWarning.detections.map((d, i) => (
              <div key={i} className="flex items-center gap-2 px-3 py-2 rounded-lg border
                                      border-red-100 dark:border-red-900 text-xs">
                <span className="font-medium text-red-600 dark:text-red-400 uppercase
                                 text-[10px] bg-red-50 dark:bg-red-900/30 px-1.5 py-0.5 rounded">
                  {d.entity_type}
                </span>
                <span className="font-mono text-gray-700 dark:text-gray-300 truncate">
                  {d.text}
                </span>
              </div>
            ))}
          </>
        ) : (
          <>
            <p className="text-sm text-gray-700 dark:text-gray-300">
              PII has been replaced with anonymized values. Review the changes below:
            </p>
            {/* Anonymization mappings */}
            <div className="space-y-1.5 max-h-48 overflow-y-auto">
              {anonymizeResult.mappings.map((m, i) => (
                <div key={i} className="flex items-center gap-2 px-3 py-2 rounded-lg border
                                        border-gray-200 dark:border-gray-700 text-xs">
                  <span className="text-red-600 dark:text-red-400 line-through font-mono
                                   flex-1 truncate">{m.original}</span>
                  <span className="text-gray-400">&rarr;</span>
                  <span className="text-green-600 dark:text-green-400 font-mono
                                   flex-1 truncate">{m.anonymized}</span>
                </div>
              ))}
            </div>
            {/* Auto-deanonymize toggle */}
            <label className="flex items-center gap-2 mt-3 cursor-pointer">
              <input
                type="checkbox"
                checked={autoDeanonymize}
                onChange={(e) => setAutoDeanonymize(e.target.checked)}
                className="rounded border-gray-300 dark:border-gray-600
                           text-purple-600 focus:ring-purple-500 cursor-pointer"
              />
              <span className="text-xs text-gray-600 dark:text-gray-400">
                Auto-deanonymize AI response
              </span>
            </label>
          </>
        )}
      </div>

      {/* Actions */}
      <div className="px-4 py-3 border-t border-gray-200 dark:border-gray-700
                      flex items-center justify-between">
        <button onClick={() => { setPiiWarning(null); setAnonymizeResult(null); }}
          className="px-3 py-1.5 rounded-lg text-sm font-medium bg-gray-100
                     dark:bg-gray-700 text-gray-700 dark:text-gray-300">
          Cancel
        </button>
        <div className="flex gap-2">
          {!anonymizeResult ? (
            <>
              <button onClick={handleAnonymize} disabled={anonymizing}
                className="px-3 py-1.5 rounded-lg text-sm font-medium bg-blue-600
                           text-white hover:bg-blue-700 disabled:opacity-50">
                {anonymizing ? "Anonymizing..." : "Anonymize"}
              </button>
              <button onClick={handleSendAnyway}
                className="px-3 py-1.5 rounded-lg text-sm font-medium bg-amber-600
                           text-white hover:bg-amber-700">
                Send Anyway
              </button>
            </>
          ) : (
            <button onClick={handleUseAnonymized}
              className="px-3 py-1.5 rounded-lg text-sm font-medium bg-green-600
                         text-white hover:bg-green-700">
              Use Anonymized Content
            </button>
          )}
        </div>
      </div>
    </div>
  </div>
)}
```

### 6.4 Reliability / Hallucination Badge

```tsx
{message.reliability ? (
  <div className={`mt-2 text-xs px-2.5 py-1.5 rounded-lg font-medium ${
    message.reliability.checking
      ? "bg-blue-100 dark:bg-blue-900/30 text-blue-700 dark:text-blue-400"
      : message.reliability.reliable
        ? "bg-green-100 dark:bg-green-900/30 text-green-700 dark:text-green-400"
        : "bg-red-100 dark:bg-red-900/30 text-red-700 dark:text-red-400"
  }`}>
    {message.reliability.checking
      ? "Checking reliability..."
      : message.reliability.reliable
        ? `Reliability: High (${message.reliability.score}%)`
        : `Reliability: Low (${message.reliability.score}%) — ${message.reliability.explanation}`}
  </div>
) : (
  /* Manual check button */
  message.role === "assistant" && !message.isStreaming && message.content &&
  safetySettings.hallucination_check !== "allow" && (
    <div className="mt-2 flex items-center gap-2">
      <button onClick={() => runReliabilityCheck(message.id)}
        className="text-[11px] font-bold text-red-500 dark:text-red-400
                   hover:text-red-700 cursor-pointer transition-colors">
        Check Reliability &amp; Hallucination
      </button>
      <span className="text-[9px] text-gray-400/50">ZeroTrusted.ai</span>
    </div>
  )
)}
```

### 6.5 Deanonymize Loading Indicator

```tsx
{message.anonymizeMappings && message.deanonymized?.checking && (
  <div className="mt-2 text-xs px-2.5 py-1.5 rounded-lg font-medium
                  bg-purple-100 dark:bg-purple-900/30 text-purple-700 dark:text-purple-400">
    <span className="inline-flex items-center gap-1.5">
      Deanonymizing response
      <span className="inline-flex gap-0.5 ml-0.5">
        <span className="w-1 h-1 rounded-full bg-current animate-bounce" />
        <span className="w-1 h-1 rounded-full bg-current animate-bounce"
              style={{ animationDelay: "150ms" }} />
        <span className="w-1 h-1 rounded-full bg-current animate-bounce"
              style={{ animationDelay: "300ms" }} />
      </span>
    </span>
  </div>
)}
```

---

## 7. Core Logic: sendMessage with Guardrails

### 7.1 State Variables

```typescript
const [safetySettings, setSafetySettings] = useState<SafetySettings>({
  pii_detection: "warn",
  hallucination_check: "manual",
});
const [piiWarning, setPiiWarning] = useState<PiiWarning | null>(null);
const [anonymizeResult, setAnonymizeResult] = useState<AnonymizeResult | null>(null);
const [anonymizing, setAnonymizing] = useState(false);
const [autoDeanonymize, setAutoDeanonymize] = useState(true);
```

### 7.2 PII Pre-Check Flow

```typescript
// Inside sendMessage callback:

// ── PII Pre-Check (BEFORE any data reaches the LLM) ──
if (!bypassPii && safetySettings.pii_detection !== "allow") {
  const piiRes = await fetch("/api/copilot/pii-check", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ text: messageToSend, api_key: safetySettings.guardrails_api_key }),
  });
  const piiData = await piiRes.json();

  if (piiData.has_pii && piiData.detections?.length > 0) {
    if (safetySettings.pii_detection === "block") {
      // Block: show error, do NOT send to LLM
      return;
    } else {
      // Warn: show PII modal for user review
      setPiiWarning({ detections: piiData.detections, pendingText: text, pendingFile: currentFile });
      return;
    }
  }
}
```

### 7.3 History Scrubbing (Critical for Privacy)

When anonymized content is sent, the conversation history must be scrubbed to prevent
original PII from leaking to the LLM via context:

```typescript
// Build history — replace original PII in last user message with anonymized text
let historyMessages = messages.filter((m) => m.id !== "welcome");
if (bypassPii && anonMappings) {
  const lastUserIdx = historyMessages.findLastIndex((m) => m.role === "user");
  if (lastUserIdx >= 0) {
    historyMessages = historyMessages.map((m, i) =>
      i === lastUserIdx ? { ...m, content: anonymizedDisplayText } : m
    );
  }
}
const history = historyMessages.map((m) => ({ role: m.role, content: m.content }));
```

> **IMPORTANT**: React's `setMessages` is async. The `messages` variable in the callback
> closure is stale. You must rebuild history explicitly with the anonymized text rather
> than relying on state updates.

### 7.4 Auto-Deanonymize After LLM Response

```typescript
// After streaming completes, if anonymized content was sent:
if (anonMappings && autoDeanonymize && fullText) {
  // Show loading indicator
  setMessages((prev) =>
    prev.map((m) =>
      m.id === assistantId
        ? { ...m, deanonymized: { text: "", checking: true } }
        : m
    )
  );

  const dRes = await fetch("/api/copilot/deanonymize", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      text: fullText,
      mappings: anonMappings,
      api_key: safetySettings.guardrails_api_key,
    }),
  });
  const dData = await dRes.json();

  if (dRes.ok && dData.deanonymized_text) {
    // Replace message content with deanonymized version
    setMessages((prev) =>
      prev.map((m) =>
        m.id === assistantId
          ? { ...m, content: dData.deanonymized_text, deanonymized: { text: dData.deanonymized_text } }
          : m
      )
    );
  }
}
```

### 7.5 On-Demand Reliability Check

```typescript
const runReliabilityCheck = useCallback(async (messageId: string) => {
  const msgIndex = messages.findIndex((m) => m.id === messageId);
  if (msgIndex < 0) return;
  const assistantMsg = messages[msgIndex];

  // Find preceding user message for context
  let userPrompt = "";
  for (let i = msgIndex - 1; i >= 0; i--) {
    if (messages[i].role === "user") { userPrompt = messages[i].content; break; }
  }

  // Show checking indicator
  setMessages((prev) =>
    prev.map((m) =>
      m.id === messageId ? { ...m, reliability: { score: 0, reliable: true, checking: true } } : m
    )
  );

  const res = await fetch("/api/copilot/hallucination-check", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      user_prompt: userPrompt,
      ai_response: assistantMsg.content,
      api_key: safetySettings.agents_api_key,
    }),
  });
  const data = await res.json();

  if (res.ok) {
    setMessages((prev) =>
      prev.map((m) =>
        m.id === messageId
          ? { ...m, reliability: { score: data.score ?? 50, reliable: data.reliable !== false, explanation: data.explanation } }
          : m
      )
    );
  }
}, [messages, safetySettings.agents_api_key]);
```

---

## 8. Settings UI

A settings page or panel where users configure safety modes and optional API key overrides:

```tsx
<div className="space-y-4">
  <div>
    <label className="block text-sm font-medium">PII Detection</label>
    <select value={settings.pii_detection}
            onChange={(e) => updateSetting("pii_detection", e.target.value)}>
      <option value="block">Block — prevent sending PII to AI</option>
      <option value="warn">Warn — show modal, offer anonymization</option>
      <option value="allow">Allow — no PII checking</option>
    </select>
  </div>

  <div>
    <label className="block text-sm font-medium">Hallucination Check</label>
    <select value={settings.hallucination_check}
            onChange={(e) => updateSetting("hallucination_check", e.target.value)}>
      <option value="block">Block — hide unreliable responses</option>
      <option value="warn">Warn — show reliability score automatically</option>
      <option value="manual">Manual — show check button</option>
      <option value="allow">Allow — no hallucination checking</option>
    </select>
  </div>

  <div>
    <label className="block text-sm font-medium">
      ZeroTrusted Guardrails API Key (optional override)
    </label>
    <input type="password" placeholder="zt-..."
           value={settings.guardrails_api_key || ""}
           onChange={(e) => updateSetting("guardrails_api_key", e.target.value)} />
  </div>
</div>
```

---

## 9. Integration Checklist

When adding ZeroTrusted.ai guardrails to an existing copilot:

1. **Add environment variables** to `.env.local`:
   - `ZT_GUARDRAILS_API_KEY` (required)
   - `ZT_ENCRYPTED_PROVIDER_KEY` (required for hallucination check)

2. **Create API routes** (Section 3):
   - `/api/copilot/pii-check`
   - `/api/copilot/anonymize`
   - `/api/copilot/deanonymize`
   - `/api/copilot/hallucination-check`

3. **Add Message fields** (Section 4):
   - `validation`, `reliability`, `anonymizeMappings`, `deanonymized`

4. **Add state variables** (Section 7.1):
   - `safetySettings`, `piiWarning`, `anonymizeResult`, `anonymizing`, `autoDeanonymize`

5. **Wrap sendMessage** with PII pre-check (Section 7.2)

6. **Scrub history** when sending anonymized content (Section 7.3)

7. **Add auto-deanonymize** after streaming (Section 7.4)

8. **Add UI components**:
   - Validation badge on each message (Section 6.2)
   - PII warning modal (Section 6.3)
   - Reliability badge + manual check button (Section 6.4)
   - Deanonymize loading indicator (Section 6.5)

9. **Add settings UI** for safety mode configuration (Section 8)

---

## 10. Common Pitfalls

### Stale Closure on History

React state updates are async. When building conversation history after `setMessages`,
the `messages` variable in the callback closure still holds old values. Always explicitly
rebuild history with anonymized text rather than reading from state.

### Payload Size Limits

- Truncate to ~10KB for detect/anonymize endpoints
- Truncate to ~3KB for hallucination check
- Large payloads cause 502 gateway timeouts on Vercel/Azure

### API Key Header

The correct header is `X-API-Key` (not `Authorization`, not `X-Custom-Token`).

### Retry on Intermittent Failures

The anonymize endpoint can be intermittent. Implement a single retry:

```typescript
let res = await callAnonymize(text, token);
if (!res.ok) res = await callAnonymize(text, token);
```

### Vercel Function Timeout

Add `export const maxDuration = 30;` to API routes that call ZeroTrusted.ai,
as the default 10s timeout is often insufficient.
