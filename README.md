# Kaphila API Production API Documentation

> Version: 2.0.0  
> Last updated: 2026-02-17  
> Service: Outbound telephony, call control, recordings, webhooks, and customer analytics

---

## 1) Overview

### API Name
**Kaphila API Production API**

### Base URL
`http://72.60.206.114:3000`

### Purpose
This API lets you:
- originate outbound calls via Kaphila API
- control live calls (hangup, TTS, playback, DTMF gather)
- track active calls and recordings
- receive call lifecycle webhooks
- view usage analytics for your API key

### Request/Response Format
- Request: JSON (`Content-Type: application/json`)
- Response: JSON (except recording download endpoints, which stream WAV files)

---

## 2) Getting Started

### Prerequisites
- Node.js >= 18
- Kaphila with API enabled
- PostgreSQL with required tables (`api_keys`, `call_logs`, etc.)
- `ffmpeg` available in PATH (for TTS conversion)

### Install and run
```bash
npm install
node api.js
```

### Health check
```bash
curl -X GET "http://72.60.206.114:3000/health"
```

Example response:
```json
{
  "status": "healthy",
  "service": "Kaphila API Telephony API",
  "version": "2.0.0",
  "timestamp": "2026-02-17T10:20:30.000Z",
  "stats": { "activeCalls": 0 },
  "features": {
    "recording": "bridge-only",
    "amd": "enhanced",
    "tts": "google-cloud-fallback-gtts",
    "playback": "bridge-safe"
  }
}
```

---

## 3) Authentication

This customer API uses **API Key Bearer authentication**.

## 3.1 API Key (Bearer)
Used for call operations and user dashboard endpoints.

Header:
```http
Authorization: Bearer ak_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### Authentication examples

#### cURL (API key)
```bash
curl -X GET "http://72.60.206.114:3000/api/me" \
  -H "Authorization: Bearer ak_live_123456" \
  -H "Content-Type: application/json"
```

#### JavaScript (axios)
```js
import axios from "axios";

const api = axios.create({
  baseURL: "http://72.60.206.114:3000",
  headers: {
    Authorization: "Bearer ak_live_123456",
    "Content-Type": "application/json"
  }
});

const me = await api.get("/api/me");
console.log(me.data);
```

#### Python (requests)
```python
import requests

base_url = "http://72.60.206.114:3000"
headers = {
    "Authorization": "Bearer ak_live_123456",
    "Content-Type": "application/json"
}

resp = requests.get(f"{base_url}/api/me", headers=headers, timeout=15)
print(resp.status_code, resp.json())
```

---

## 4) Common Headers

| Header | Required | Value |
|---|---:|---|
| Content-Type | Yes (POST/PUT) | application/json |
| Authorization | Yes (protected endpoints) | `Bearer <api_key>` |
| User-Agent | Recommended | Your client app name/version |

---

## 5) Error Handling

### Error envelope
Most errors follow:
```json
{
  "success": false,
  "error": "Human-readable message"
}
```

### Error codes table

| HTTP | Meaning | Typical causes |
|---:|---|---|
| 400 | Bad Request | Missing required fields, invalid filename, invalid params |
| 401 | Unauthorized | Missing/invalid API key |
| 402 | Payment Required | Insufficient credits |
| 403 | Forbidden | Reserved for policy/permission denial (not consistently used in code) |
| 404 | Not Found | Unknown call ID, recording, API key, file |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | API/db/file/system command failures |
| 503 | Service Unavailable | No trunks assigned |

---

## 6) Rate Limiting

- Applied to: `/makecall`, `/batchcall`, `/voice`, `/play`, `/gather`, `/hangup`
- Window: 1 hour
- Limit: `api_keys.rate_limit` (default fallback: 100 req/hour)
- Exceeded response:
```json
{ "success": false, "error": "Rate limit exceeded" }
```

---

## 7) Pagination

### `/api/call-logs`
- Inputs: `page`, `page_size`, optional `status`
- Returns `total_count` and pagination object

---

## 8) Webhooks

When `webhookUrl` is provided during call creation, the API sends POST events to that URL.

### Transport
- Method: `POST`
- Content-Type: `application/json`
- User-Agent: `Kaphila-API-API/2.0`
- Timeout: 5000ms

### Event types
- `call.initiated`
- `call.answered`
- `call.ended`
- `recording.started`
- `recording.completed`
- `recording.stopped`
- `dtmf.received`
- `gather.started`
- `gather.progress`
- `gather.complete`
- `gather.timeout`
- `tts.played`

### Example webhook payload
```json
{
  "callId": "1720012345.88",
  "timestamp": "2026-02-17T10:22:11.233Z",
  "event": "call.ended",
  "status": "completed",
  "endReason": "normal_hangup",
  "hangupCause": "16",
  "wasAnswered": true,
  "amd": {
    "status": "HUMAN",
    "cause": "INITIALSILENCE",
    "confidence": 0.9
  },
  "recording": {
    "filename": "call-1720012345.88-1739781234567.wav",
    "recordingId": "call-1720012345.88-1739781234567",
    "active": false
  },
  "callDuration": 43
}
```

---

## 9) Endpoint Reference

## 9.1 Public

### GET /health
- Description: service health and runtime capability snapshot.
- Authentication Required: No

**Headers:** none required  
**Query Params:** none  
**Request Body:** none

**Success 200**
```json
{
  "status": "healthy",
  "service": "Kaphila API Telephony API",
  "version": "2.0.0",
  "timestamp": "2026-02-17T10:20:30.000Z",
  "stats": { "activeCalls": 2 },
  "features": {
    "recording": "bridge-only",
    "amd": "enhanced",
    "tts": "google-cloud-fallback-gtts",
    "playback": "bridge-safe"
  }
}
```

**Error Responses:** 500 (rare)  
**Example Request:**
```bash
curl -X GET "http://72.60.206.114:3000/health"
```

**Notes & Edge Cases**
- If API is disconnected, process may fail at startup rather than returning unhealthy status.

---

## 9.2 Call Control (API Key)

### POST /makecall
- Description: originate an outbound call with trunk round-robin + failover.
- Authentication Required: Yes (API Key)

**Headers**
- `Authorization: Bearer <api_key>`
- `Content-Type: application/json`

**Query Params:** none

**Request Body Schema**
```json
{
  "number": "string, required",
  "webhookUrl": "string, optional, https URL",
  "callerId": "string, optional",
  "useAmd": "boolean, optional, default false",
  "voiceName": "string, optional, default en-US-Neural2-A",
  "ringTimeout": "number, optional, seconds, default 30"
}
```

**Success 200**
```json
{
  "success": true,
  "callId": "1720012345.88",
  "status": "ringing",
  "amdEnabled": true,
  "voiceName": "en-US-Neural2-A",
  "credits": 49,
  "ringTimeoutSeconds": 30,
  "trunk": "mycarrier-endpoint",
  "trunkAttempts": 1,
  "totalTrunks": 3,
  "timestamp": "2026-02-17T10:24:51.001Z"
}
```

**Error Responses**
- 400: missing `number`
- 401: invalid/missing API key
- 402: insufficient credits
- 404: n/a
- 429: rate limit exceeded
- 500: origination failure
- 503: no trunks assigned

**Example Request (cURL)**
```bash
curl -X POST "http://72.60.206.114:3000/makecall" \
  -H "Authorization: Bearer ak_live_123" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "14155550100",
    "webhookUrl": "https://example.com/hooks/calls",
    "callerId": "19172452367",
    "useAmd": true,
    "voiceName": "en-US-Neural2-A",
    "ringTimeout": 30
  }'
```

**Example Response (failure all trunks)**
```json
{
  "success": false,
  "error": "All trunk originate attempts failed",
  "code": "ORIGINATION_FAILED",
  "attemptedTrunks": ["a-endpoint", "b-endpoint"],
  "totalTrunks": 2
}
```

**Notes & Edge Cases**
- Number is dialed as `PJSIP/00<number>@<trunk>`, so country-prefix behavior matters.
- Credit is deducted when call is answered, not at originate time.

---

### POST /batchcall
- Description: call origination endpoint intended for batch workflows.
- Authentication Required: Yes (API Key)

**Headers:** same as `/makecall`  
**Query Params:** none

**Request Body Schema:** same as `/makecall`

**Success 200**
```json
{
  "success": true,
  "callId": "1720012346.91",
  "status": "ringing",
  "amdEnabled": false,
  "voiceName": "en-US-Neural2-A",
  "credits": 20,
  "ringTimeoutSeconds": 25,
  "trunk": "carrier2-endpoint",
  "trunkAttempts": 2,
  "totalTrunks": 4,
  "timestamp": "2026-02-17T10:25:24.200Z"
}
```

**Error Responses:** 400, 401, 402, 429, 500, 503

**Example Request**
```bash
curl -X POST "http://72.60.206.114:3000/batchcall" \
  -H "Authorization: Bearer ak_live_123" \
  -H "Content-Type: application/json" \
  -d '{"number":"14155550123","ringTimeout":20}'
```

**Notes & Edge Cases**
- Current code uses fixed endpoint format internally during originate in one path; validate in staging before high-volume use.

---

### POST /hangup
- Description: terminate an active call.
- Authentication Required: Yes (API Key)

**Request Body**
```json
{ "callId": "string, required" }
```

**Success 200**
```json
{ "success": true, "message": "Call 1720012345.88 terminated", "callId": "1720012345.88" }
```

**Error Responses:** 401, 404, 429, 500

**Example Request**
```bash
curl -X POST "http://72.60.206.114:3000/hangup" \
  -H "Authorization: Bearer ak_live_123" \
  -H "Content-Type: application/json" \
  -d '{"callId":"1720012345.88"}'
```

**Notes**
- Returns 404 if call already ended/cleaned from memory.

---

### POST /voice
- Description: synthesize and play TTS into call or bridge.
- Authentication Required: Yes (API Key)

**Request Body**
```json
{
  "callId": "string, required",
  "text": "string, required",
  "playTo": "bridge|channel, optional, default bridge"
}
```

**Success 200**
```json
{
  "success": true,
  "message": "TTS played successfully",
  "text": "Hello, this is a reminder call...",
  "playbackId": "tts-1720012345.88-1739781234567",
  "method": "bridge"
}
```

**Error Responses:** 400, 401, 404, 429, 500

**Example Request**
```bash
curl -X POST "http://72.60.206.114:3000/voice" \
  -H "Authorization: Bearer ak_live_123" \
  -H "Content-Type: application/json" \
  -d '{
    "callId":"1720012345.88",
    "text":"Press 1 for sales, press 2 for support",
    "playTo":"bridge"
  }'
```

**Notes**
- TTS engine can be Google or gTTS fallback based on settings.

---

### POST /play
- Description: play a server-side sound file in active call.
- Authentication Required: Yes (API Key)

**Request Body**
```json
{
  "callId": "string, required",
  "file": "string, required, Kaphila sound name",
  "playTo": "bridge|channel, optional, default bridge"
}
```

**Success 200**
```json
{
  "success": true,
  "message": "Audio file played: custom/welcome",
  "file": "custom/welcome",
  "playbackId": "play-1720012345.88-1739781234999",
  "method": "bridge"
}
```

**Error Responses:** 401, 404, 429, 500

**Example Request**
```bash
curl -X POST "http://72.60.206.114:3000/play" \
  -H "Authorization: Bearer ak_live_123" \
  -H "Content-Type: application/json" \
  -d '{"callId":"1720012345.88","file":"custom/welcome"}'
```

---

### POST /gather
- Description: play prompt and collect DTMF digits.
- Authentication Required: Yes (API Key)

**Request Body**
```json
{
  "callId": "string, required",
  "text": "string, required",
  "numDigits": "number, optional, default 1",
  "timeout": "number ms, optional, default 10000",
  "playTo": "bridge|channel, optional, default bridge"
}
```

**Success 200**
```json
{
  "success": true,
  "message": "Gathering 4 digits with 12000ms timeout",
  "callId": "1720012345.88",
  "expectedDigits": 4,
  "timeoutMs": 12000
}
```

**Error Responses:** 401, 404, 429, 500

**Example Request**
```bash
curl -X POST "http://72.60.206.114:3000/gather" \
  -H "Authorization: Bearer ak_live_123" \
  -H "Content-Type: application/json" \
  -d '{"callId":"1720012345.88","text":"Enter 4 digit pin","numDigits":4,"timeout":12000}'
```

**Notes**
- Completion is sent to webhook (`gather.complete` or `gather.timeout`).

---

### GET /calls
- Description: list active calls for the authenticated API key.
- Authentication Required: Yes (API Key)

**Success 200**
```json
{
  "success": true,
  "totalCalls": 1,
  "activeCalls": 1,
  "calls": [
    {
      "callId": "1720012345.88",
      "status": "answered",
      "number": "14155550100",
      "amd": { "status": "HUMAN", "cause": "INITIALSILENCE", "confidence": 0.9 },
      "gather": null,
      "recording": {
        "active": true,
        "filename": "call-1720012345.88-1739781234567.wav",
        "recordingId": "call-1720012345.88-1739781234567",
        "method": "bridge"
      },
      "voiceName": "en-US-Neural2-A",
      "hasBridge": true
    }
  ],
  "timestamp": "2026-02-17T10:30:00.000Z"
}
```

**Error Responses:** 401

**Example Request**
```bash
curl -X GET "http://72.60.206.114:3000/calls" \
  -H "Authorization: Bearer ak_live_123"
```

---

### GET /calls/:callId
- Description: get detailed active call state.
- Authentication Required: Yes (API Key)

**Path Params**
- `callId` (required)

**Success 200**
```json
{
  "success": true,
  "callId": "1720012345.88",
  "status": "answered",
  "number": "14155550100",
  "amd": { "status": "HUMAN", "cause": "INITIALSILENCE", "confidence": 0.9 },
  "recording": {
    "active": true,
    "filename": "call-1720012345.88-1739781234567.wav",
    "recordingId": "call-1720012345.88-1739781234567",
    "method": "bridge"
  },
  "hasBridge": true,
  "hasSnoop": false
}
```

**Error Responses:** 401, 404

**Example Request**
```bash
curl -X GET "http://72.60.206.114:3000/calls/1720012345.88" \
  -H "Authorization: Bearer ak_live_123"
```

---

## 9.3 Recordings (API Key)

### GET /recordings
- Description: list active call recording states + WAV files on server.
- Auth: Yes (API Key)

**Success 200**
```json
{
  "success": true,
  "active": [
    {
      "callId": "1720012345.88",
      "filename": "call-1720012345.88-1739781234567.wav",
      "recordingId": "call-1720012345.88-1739781234567",
      "active": true
    }
  ],
  "files": [
    {
      "filename": "call-1720012345.88-1739781234567.wav",
      "size": 114320,
      "createdAt": "2026-02-17T10:29:58.000Z"
    }
  ]
}
```

**Error Responses:** 401, 500

**Example**
```bash
curl -X GET "http://72.60.206.114:3000/recordings" \
  -H "Authorization: Bearer ak_live_123"
```

---

### GET /recordings/:callId
- Description: get recording metadata for an active call.
- Auth: Yes

**Success 200**
```json
{
  "success": true,
  "callId": "1720012345.88",
  "recordingId": "call-1720012345.88-1739781234567",
  "filename": "call-1720012345.88-1739781234567.wav",
  "active": true
}
```

**Error Responses:** 401, 404

**Example**
```bash
curl -X GET "http://72.60.206.114:3000/recordings/1720012345.88" \
  -H "Authorization: Bearer ak_live_123"
```

---

### GET /recordings/:callId/download
- Description: download WAV by active call ID.
- Auth: Yes

**Success 200**
- Binary stream (`audio/wav`)

**Error Responses:** 401, 404

**Example**
```bash
curl -L "http://72.60.206.114:3000/recordings/1720012345.88/download" \
  -H "Authorization: Bearer ak_live_123" \
  -o call.wav
```

---

### GET /recordings/file/:filename/download
- Description: download WAV by filename.
- Auth: Yes

**Path Params**
- `filename` must end with `.wav`

**Error Responses:** 400, 401, 404, 500

**Example**
```bash
curl -L "http://72.60.206.114:3000/recordings/file/call-1720012345.88-1739781234567.wav/download" \
  -H "Authorization: Bearer ak_live_123" \
  -o call.wav
```

---

### POST /recordings/:callId/stop
- Description: marks active recording as stopped in runtime state.
- Auth: Yes

**Success 200**
```json
{
  "success": true,
  "message": "Recording stopped successfully",
  "filename": "call-1720012345.88-1739781234567.wav",
  "recordingId": "call-1720012345.88-1739781234567"
}
```

**Error Responses:** 401, 404, 500

**Example**
```bash
curl -X POST "http://72.60.206.114:3000/recordings/1720012345.88/stop" \
  -H "Authorization: Bearer ak_live_123"
```

---

## 9.4 User Dashboard (API Key)

### POST /api/dashboard
- Description: current API key owner dashboard stats + latest calls.
- Auth: Yes

**Request Body**: `{}` (no required fields)

**Success 200**
```json
{
  "success": true,
  "stats": {
    "total_credits": 78,
    "total_calls": 340,
    "successful_calls": 301,
    "success_rate": 88.53,
    "calls_today": 22,
    "successful_calls_today": 18,
    "calls_this_week": 102
  },
  "recent_calls": [
    {
      "call_id": "1720012345.88",
      "number": "14155550100",
      "status": "completed",
      "amd_status": "HUMAN",
      "duration": 46,
      "created_at": "2026-02-17T10:22:00.000Z"
    }
  ]
}
```

**Error Responses:** 401, 500

**Example**
```bash
curl -X POST "http://72.60.206.114:3000/api/dashboard" \
  -H "Authorization: Bearer ak_live_123" \
  -H "Content-Type: application/json" \
  -d '{}'
```

---

### POST /api/call-logs
- Description: paginated call history for current API key.
- Auth: Yes

**Request Body**
```json
{
  "page": 1,
  "page_size": 20,
  "status": "completed"
}
```

**Success 200**
```json
{
  "success": true,
  "logs": [
    {
      "call_id": "1720012345.88",
      "number": "14155550100",
      "status": "completed",
      "amd_status": "HUMAN",
      "duration": 46,
      "created_at": "2026-02-17T10:22:00.000Z"
    }
  ],
  "total_count": 112,
  "pagination": {
    "page": 1,
    "page_size": 20,
    "total_pages": 6
  }
}
```

**Error Responses:** 401, 500

**Example**
```bash
curl -X POST "http://72.60.206.114:3000/api/call-logs" \
  -H "Authorization: Bearer ak_live_123" \
  -H "Content-Type: application/json" \
  -d '{"page":1,"page_size":20,"status":"completed"}'
```

---

### POST /api/analytics
- Description: user-level analytics for last N days.
- Auth: Yes

**Request Body**
```json
{ "days": 7 }
```

**Success 200**
```json
{
  "success": true,
  "period": "Last 7 days",
  "overview": {
    "totalCalls": 112,
    "completedCalls": 97,
    "failedCalls": 7,
    "noAnswerCalls": 8,
    "humanCalls": 84,
    "machineCalls": 28,
    "avgDuration": 42.8,
    "successRate": 86.61
  },
  "dailyVolume": [
    { "date": "2026-02-17", "total_calls": 22, "completed": 19, "failed": 1 }
  ],
  "statusBreakdown": [
    { "status": "completed", "count": 97, "percentage": 86.61 }
  ]
}
```

**Error Responses:** 401, 500

**Example**
```bash
curl -X POST "http://72.60.206.114:3000/api/analytics" \
  -H "Authorization: Bearer ak_live_123" \
  -H "Content-Type: application/json" \
  -d '{"days":30}'
```

---

### GET /api/me
- Description: returns authenticated API key metadata.
- Auth: Yes

**Success 200**
```json
{
  "success": true,
  "apiKey": {
    "id": 12,
    "keyId": "key_98aa10cd11",
    "name": "Campaign API",
    "credits": 78,
    "totalCalls": 340,
    "successfulCalls": 301,
    "rateLimit": 200,
    "createdAt": "2025-12-01T00:00:00.000Z",
    "lastUsed": "2026-02-17T10:30:11.000Z"
  }
}
```

**Error Responses:** 401, 500

**Example**
```bash
curl -X GET "http://72.60.206.114:3000/api/me" \
  -H "Authorization: Bearer ak_live_123"
```

---

## 10) SDK Usage Examples

### cURL workflow: place call with existing API key
```bash
# 1) Set your issued API key
API_KEY="ak_live_123"

# 2) Make call
curl -X POST "http://72.60.206.114:3000/makecall" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"number":"14155550100","webhookUrl":"https://example.com/hook"}'
```

### JavaScript SDK-style helper
```js
import axios from "axios";

export function createClient({ baseURL, token }) {
  return axios.create({
    baseURL,
    headers: {
      Authorization: `Bearer ${token}`,
      "Content-Type": "application/json"
    },
    timeout: 15000
  });
}

const api = createClient({ baseURL: "http://72.60.206.114:3000", token: "ak_live_123" });
const call = await api.post("/makecall", {
  number: "14155550100",
  webhookUrl: "https://example.com/hooks/calls",
  useAmd: true
});
console.log(call.data);
```

### Python SDK-style helper
```python
import requests

class APIApiClient:
    def __init__(self, base_url: str, token: str):
        self.base_url = base_url.rstrip("/")
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/json",
        })

    def make_call(self, number: str, webhook_url: str = None):
        payload = {"number": number}
        if webhook_url:
            payload["webhookUrl"] = webhook_url
        resp = self.session.post(f"{self.base_url}/makecall", json=payload, timeout=20)
        resp.raise_for_status()
        return resp.json()

client = APIApiClient("http://72.60.206.114:3000", "ak_live_123")
print(client.make_call("14155550100", "https://example.com/hooks/calls"))
```

### Flask (Python) — IVR + Voicemail Detection (AMD)

This example does three things:
1. Starts a call with `useAmd: true`
2. Uses webhook `call.answered` AMD status to split HUMAN vs MACHINE
3. Runs IVR (`/gather`) for humans and voicemail drop (`/voice`) for machines

```python
import os
import requests
from flask import Flask, request, jsonify

app = Flask(__name__)

BASE_URL = os.getenv("KAPHILA_BASE_URL", "http://72.60.206.114:3000")
API_KEY = os.getenv("KAPHILA_API_KEY", "ak_live_123")

HEADERS = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json",
}


def post_api(path: str, payload: dict):
    resp = requests.post(f"{BASE_URL}{path}", json=payload, headers=HEADERS, timeout=15)
    resp.raise_for_status()
    return resp.json()


@app.post("/start-call")
def start_call():
    body = request.get_json(force=True) or {}
    number = body.get("number")
    if not number:
        return jsonify({"ok": False, "error": "number is required"}), 400

    webhook_url = body.get("webhookUrl", "https://your-domain.com/webhooks/kaphila")

    result = post_api("/makecall", {
        "number": number,
        "webhookUrl": webhook_url,
        "useAmd": True,
        "ringTimeout": 30,
        "voiceName": "en-US-Neural2-A"
    })
    return jsonify({"ok": True, "call": result})


@app.post("/webhooks/kaphila")
def kaphila_webhook():
    payload = request.get_json(force=True) or {}
    event = payload.get("event")
    call_id = payload.get("callId")

    if not call_id:
        return jsonify({"ok": True, "ignored": "missing callId"})

    if event == "call.answered":
        amd_status = ((payload.get("amd") or {}).get("status") or "UNKNOWN").upper()

        if amd_status == "MACHINE":
            post_api("/voice", {
                "callId": call_id,
                "playTo": "bridge",
                "text": "Hello. This is a reminder call from Kaphila. Please call us back at 1-800-555-0100."
            })
        else:
            post_api("/gather", {
                "callId": call_id,
                "playTo": "bridge",
                "text": "Press 1 for sales, press 2 for support.",
                "numDigits": 1,
                "timeout": 10000
            })

    elif event == "gather.complete":
        digits = str(payload.get("digits", ""))
        if digits == "1":
            post_api("/voice", {
                "callId": call_id,
                "playTo": "bridge",
                "text": "Connecting you to sales shortly."
            })
        elif digits == "2":
            post_api("/voice", {
                "callId": call_id,
                "playTo": "bridge",
                "text": "Connecting you to support shortly."
            })
        else:
            post_api("/voice", {
                "callId": call_id,
                "playTo": "bridge",
                "text": "Invalid input. Goodbye."
            })
            post_api("/hangup", {"callId": call_id})

    return jsonify({"ok": True})


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```

### Node.js (Express) — IVR + Voicemail Detection (AMD)

```js
import express from "express";
import axios from "axios";

const app = express();
app.use(express.json());

const BASE_URL = process.env.KAPHILA_BASE_URL || "http://72.60.206.114:3000";
const API_KEY = process.env.KAPHILA_API_KEY || "ak_live_123";

const api = axios.create({
  baseURL: BASE_URL,
  headers: {
    Authorization: `Bearer ${API_KEY}`,
    "Content-Type": "application/json",
  },
  timeout: 15000,
});

app.post("/start-call", async (req, res) => {
  try {
    const { number, webhookUrl } = req.body;
    if (!number) return res.status(400).json({ ok: false, error: "number is required" });

    const call = await api.post("/makecall", {
      number,
      webhookUrl: webhookUrl || "https://your-domain.com/webhooks/kaphila",
      useAmd: true,
      ringTimeout: 30,
      voiceName: "en-US-Neural2-A",
    });

    return res.json({ ok: true, call: call.data });
  } catch (err) {
    return res.status(500).json({ ok: false, error: err.message });
  }
});

app.post("/webhooks/kaphila", async (req, res) => {
  const payload = req.body || {};
  const event = payload.event;
  const callId = payload.callId;

  try {
    if (!callId) return res.json({ ok: true, ignored: "missing callId" });

    if (event === "call.answered") {
      const amd = String(payload?.amd?.status || "UNKNOWN").toUpperCase();

      if (amd === "MACHINE") {
        await api.post("/voice", {
          callId,
          playTo: "bridge",
          text: "Hello. This is a reminder call from Kaphila. Please call us back at 1-800-555-0100.",
        });
      } else {
        await api.post("/gather", {
          callId,
          playTo: "bridge",
          text: "Press 1 for sales, press 2 for support.",
          numDigits: 1,
          timeout: 10000,
        });
      }
    }

    if (event === "gather.complete") {
      const digits = String(payload.digits || "");
      if (digits === "1") {
        await api.post("/voice", { callId, playTo: "bridge", text: "Connecting you to sales shortly." });
      } else if (digits === "2") {
        await api.post("/voice", { callId, playTo: "bridge", text: "Connecting you to support shortly." });
      } else {
        await api.post("/voice", { callId, playTo: "bridge", text: "Invalid input. Goodbye." });
        await api.post("/hangup", { callId });
      }
    }

    return res.json({ ok: true });
  } catch (err) {
    return res.status(500).json({ ok: false, error: err.message });
  }
});

app.listen(5000, () => {
  console.log("Webhook server running on :5000");
});
```

### cURL test flow (quick)

```bash
# 1) Place call with AMD + webhook
curl -X POST "http://72.60.206.114:3000/makecall" \
  -H "Authorization: Bearer ak_live_123" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "14155550100",
    "webhookUrl": "https://your-domain.com/webhooks/kaphila",
    "useAmd": true,
    "ringTimeout": 30
  }'

# 2) Your webhook receives call.answered with amd.status = HUMAN or MACHINE
# 3) HUMAN -> POST /gather ; MACHINE -> POST /voice (voicemail drop)
```

### AMD decision map

| `amd.status` | Recommended action |
|---|---|
| `HUMAN` | Start IVR menu with `/gather` |
| `MACHINE` | Play voicemail message with `/voice`, then optionally `/hangup` |
| `UNKNOWN` / `NOAMD` | Retry short IVR prompt or send fallback message |

---

## 11) Best Practices & Security

1. Store secrets in environment variables; never hardcode API keys.
2. Rotate API keys regularly.
3. Validate phone numbers in E.164 format before submission.
4. Enforce webhook signature verification on receiver side.
5. Add idempotency keys for retried call creation requests.
6. Monitor `402`, `429`, and call failure rates.
7. Keep Kaphila, Node dependencies, and OS patched.

---

## 12) FAQ

### Q1) When is credit deducted?
When a call is answered (`StasisStart` flow), not at initial originate request.

### Q2) Why do I get `503 No trunks assigned`?
Your account is not currently routed for outbound traffic. Contact support to enable outbound routing.

### Q3) Why does gather not return digits directly?
Digits are emitted asynchronously through webhook events (`gather.progress`, `gather.complete`, `gather.timeout`).

### Q4) Can I download old recordings not tied to active calls?
Yes, if WAV exists in recording directory and you know filename (`/recordings/file/:filename/download`).

### Q5) How do I raise rate limits?
Contact support with your expected requests/hour and use case.

---

## 13) Changelog Template

```markdown
# Changelog

## [Unreleased]
### Added
- 

### Changed
- 

### Fixed
- 

### Deprecated
- 

### Removed
- 

### Security
- 

## [2.0.0] - 2026-02-17
### Added
- API key auth + credits
- Trunk failover and round-robin routing
- Recording management endpoints
```

---

## 14) Production Readiness Checklist

- [ ] Enable HTTPS/TLS everywhere
- [ ] Replace default JWT secret
- [ ] Remove hardcoded credentials from source
- [ ] Add centralized request logging and audit trails
- [ ] Add webhook signing + replay protection
- [ ] Add OpenAPI schema and contract tests
- [ ] Add backup/restore runbook for PJSIP config changes
