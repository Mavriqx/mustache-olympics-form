# Mustache Olympics Submission System — System Overview

This document captures the full setup end-to-end so any new chat (or teammate) can understand and help without guesswork.

---

## 1) Client (Front-End)

- **Host:** Netlify — `mustache-olympics-submission.netlify.app`
- **Repo (local):** `mustache-olympics-form`
- **Primary file:** `index.html` (embedded CSS/JS)
- **Assets:** `Black.png`, `uppy.min.js`, `uppy.min.css`

### 1.1 UI/UX Highlights
- Page title: **“Enter the 2025 Mustache Olympics”**
- Two sections with styled headers:
  - **Required Information**
  - **Optional Information** (subtitle: *“It only takes 30 seconds — complete the optional questions below to shape next year’s Mustache Olympics.”*)
- Email is passed in via `?email=` param and displayed in a banner.
- Inline **Uppy Dashboard** uploader (compact ~180px), single file.
- **Allowed file types:** MP4 (`video/mp4`) and MOV (`video/quicktime`) only.
- **Max size:** 1 GB.
- “Done” button hidden; **remove (X)** enabled so entrants can replace a file.
- Submit button: **burgundy**, right-aligned, disabled until upload completes.

### 1.2 Front-End Constants (embedded in `index.html`)
- `SIGNER_BASE = https://2gflx5uo7lqzj2exr2dgybofrq0uvrhy.lambda-url.us-east-2.on.aws`
- `LAMBDA_ENDPOINT = https://lf5ekhxuvwgwuhc34tsis6eldm0ltwzr.lambda-url.us-east-2.on.aws/`
- `BUCKET = maxstache-uploads-2025-mustache-olympics`
- `REGION = us-east-2`

### 1.3 Client Validation
- Uppy restricts: **single file**, **.mp4/.mov**, **≤ 1 GB**.
- Consent checkbox must be checked before submit.
- Submit is disabled until upload completes successfully.

---

## 2) End-User Flow

1. Entrant opens the submission page via a personalized link (`?email=name@example.com`).
2. Entrant fills **Required Information** and (optionally) the **Optional Information**.
3. Entrant uploads a **MP4/MOV** video via Uppy:
   - Browser calls **Signer Lambda** to:
     - `POST /create-mpu` → returns `{uploadId, key, submissionId}`
     - `POST /sign-part` → returns pre-signed `PUT` URL(s)
     - `POST /complete-mpu` → finalizes MPU
   - Object is created in S3 at canonical key (see §3.1).
4. **S3 ObjectCreated** event triggers **Ingest Finalize Lambda**:
   - Adds final tags (`event_id`, `participant_id`, `submission_id`, `content=video`, `source=direct-s3`)
   - Updates **DynamoDB** record (`backup_status=ok`, `s3_key`, `backup_ts`, etc.)
5. Entrant clicks **Submit**:
   - Browser sends form JSON (includes `submissionId`, `s3Key`) to **Submit API Lambda**
   - Lambda writes to DynamoDB using a conditional insert to prevent duplicates
   - Browser redirects to **Thank You** page on success

---

## 3) AWS Inventory (Production)

### 3.1 S3 — Uploads Bucket
- **Bucket:** `maxstache-uploads-2025-mustache-olympics` (Region: `us-east-2`)
- **Block Public Access:** ON
- **Versioning:** ON
- **Object Ownership:** Bucket owner enforced

**CORS (bucket-level):**
    [
      {
        "AllowedOrigins": [
          "https://maxstache.com",
          "https://www.maxstache.com",
          "https://mustache-olympics-submission.netlify.app",
          "http://localhost:3000",
          "http://localhost:5173"
        ],
        "AllowedMethods": ["POST","PUT","GET","HEAD"],
        "AllowedHeaders": ["*"],
        "ExposeHeaders": ["ETag","Location"],
        "MaxAgeSeconds": 3000
      }
    ]

**Object key convention:**
    event={event_id}/participant_id={participant_id}/submission_id={submission_id}/video.mp4

**Final object tags:**
    event_id, participant_id, submission_id, content=video, source=direct-s3

*(Optional backlog)* Lifecycle rules:
- Abort incomplete MPUs after **1 day**
- Expire raw uploads after **120 days**

---

### 3.2 DynamoDB — Submissions
- **Table:** `MustacheOlympicsSubmissions`
- **Partition key (PK):** `submissionId` (String; same UUID returned by `create-mpu`)

**Typical attributes**
- Required pipeline fields:
  - `submissionId` (PK)
  - `participantId` (derived from email)
  - `eventId` (default: `2025-mustache-olympics`)
  - `s3_key`
  - `backup_status` (e.g., `ok`)
  - `backup_ts` (ISO 8601)
- Form fields:
  - `email`, `firstName`, `lastName`, `city`, `state`
  - Optional: `phone`, `socialPlatform`, `socialHandle`, `eventSource`
  - Optional ‘accordion’ values for Talent / ’Stache / About You

**Write pattern (idempotent):**
    ConditionExpression: attribute_not_exists(submissionId)

---

### 3.3 Lambdas

#### A) Signer — `maxstache-s3-signer`
- **Trigger:** Lambda Function URL (Auth: NONE; CORS locked to your origins)
- **Handler/Runtime:** `app.handler` (Python 3.12)
- **Env:**
  - `REGION=us-east-2`
  - `BUCKET=maxstache-uploads-2025-mustache-olympics`
  - `EVENT_ID=2025-mustache-olympics`
  - `ALLOWED_ORIGINS=https://mustache-olympics-submission.netlify.app,https://maxstache.com,https://www.maxstache.com,http://localhost:3000,http://localhost:5173`
- **Endpoints:**
  - `POST /create-mpu` → `{ uploadId, key, submissionId }`
  - `POST /sign-part`  → `{ url }` (pre-signed PUT for part)
  - `POST /complete-mpu` → `{ ok: true, location }`
- **IAM (minimum):**
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect":"Allow",
          "Action":[
            "s3:CreateMultipartUpload",
            "s3:UploadPart",
            "s3:CompleteMultipartUpload",
            "s3:AbortMultipartUpload"
          ],
          "Resource":"arn:aws:s3:::maxstache-uploads-2025-mustache-olympics/*"
        }
      ]
    }

#### B) Ingest Finalize — `ingest-finalize`
- **Trigger:** S3 **ObjectCreated (All)** on uploads bucket (single rule; no overlapping filters)
- **Handler/Runtime:** `lambda_function.lambda_handler` (Python 3.12)
- **Env:** `DDB_TABLE=MustacheOlympicsSubmissions`
- **Behavior:**
  - `PutObjectTagging` on the new object with final tags
  - `UpdateItem` in DynamoDB:
    - `backup_status = ok`
    - `s3_key` = canonical key
    - `backup_ts` = now
    - `event_id`, `participant_id`
- **IAM (minimum):**
    {
      "Version":"2012-10-17",
      "Statement":[
        {
          "Effect":"Allow",
          "Action":["dynamodb:UpdateItem"],
          "Resource":"arn:aws:dynamodb:us-east-2:760474663439:table/MustacheOlympicsSubmissions"
        },
        {
          "Effect":"Allow",
          "Action":["s3:PutObjectTagging"],
          "Resource":"arn:aws:s3:::maxstache-uploads-2025-mustache-olympics/*"
        }
      ]
    }

#### C) Submit API — `maxstache-submit`
- **Trigger:** Lambda Function URL (public; CORS locked to your origins)
- **Handler/Runtime:** `lambda_function.lambda_handler` (Python 3.12)
- **Env:**
  - `REGION=us-east-2`
  - `DDB_TABLE=MustacheOlympicsSubmissions`
  - `EVENT_ID=2025-mustache-olympics`
  - *(Legacy/unused now)* `QUEUE_URL` (for Uploadcare path)
- **Behavior:**
  - Validates presence of `submissionId` and key
  - `PutItem` to DynamoDB with **conditional insert** (no duplicate `submissionId`)
- **IAM (minimum):**
    {
      "Version":"2012-10-17",
      "Statement":[
        {
          "Effect":"Allow",
          "Action":["dynamodb:PutItem"],
          "Resource":"arn:aws:dynamodb:us-east-2:760474663439:table/MustacheOlympicsSubmissions"
        }
      ]
    }

*(Note: The old Uploadcare pipeline exists in history but is **disabled**; Direct-to-S3 is the only path in use.)*

---

## 4) CORS & Security

**Function URLs (Signer, Submit):**
- **Allowed origins:**  
  `https://mustache-olympics-submission.netlify.app`, `https://maxstache.com`, `https://www.maxstache.com`, `http://localhost:3000`, `http://localhost:5173`
- **Allowed headers:** `content-type`
- **Allowed methods:** `OPTIONS, POST`
- Each function also returns `Access-Control-Allow-*` headers and handles `OPTIONS`.

**S3:**
- CORS as defined in §3.1
- Block Public Access: **Enabled**
- Versioning: **Enabled**

*(Backlog hardening ideas: TLS-only & SSE-required bucket policy, API Gateway + WAF in front of Function URLs, Turnstile/recaptcha.)*

---

## 5) Server-Side Validation & Idempotency

- **Submit API** uses DynamoDB conditional insert:
  - `ConditionExpression: attribute_not_exists(submissionId)`
- **Ingest Finalize** only tags/updates when key follows canonical convention.
- **Signer** produces keys under the canonical prefix for the current event.
- **Client** constraints (MP4/MOV, ≤1 GB) are enforced before submit is enabled.

---

## 6) Data Model (DynamoDB)

**Partition key:** `submissionId` (String; UUID from `create-mpu`)

**Common Attributes**
- `submissionId`, `participantId`, `eventId`
- `email`, `firstName`, `lastName`, `city`, `state`
- `s3_key`, `backup_status`, `backup_ts`
- Optional profile fields:
  - `phone`, `socialPlatform`, `socialHandle`, `eventSource`
  - Talent / ’Stache / About You selections

---

## 7) Testing Playbook (Manual)

**Pre-checks**
- Function URLs show correct CORS (origins & headers).
- S3 CORS includes your Netlify origin(s).

**End-to-End Smoke**
1. Open the submission page with `?email=your@email`.
2. Upload a **.mp4** or **.mov** (20–50 MB is fine).
3. Confirm **single** progress bar and “Upload complete ✅”.
4. Click **Submit** → redirected to Thank You page.
5. Verify in AWS:
   - **S3**: Object exists at canonical key; object **tags** include `event_id`, `participant_id`, `submission_id`, `content=video`, `source=direct-s3`.
   - **DynamoDB**: Item exists with `submissionId`, `backup_status=ok`, `s3_key`, `backup_ts`.
   - **CloudWatch Logs**: No errors in `ingest-finalize`, `maxstache-s3-signer`, or `maxstache-submit`.

**If something fails**
- **Duplicate** submission: DynamoDB conditional insert will block overwrite (check the item).
- **Upload stalls**: check browser Network tab for `sign-part` calls and part PUTs; signer endpoints should be 200s.
- **Missing tags**: ensure the bucket’s **single** `ObjectCreated (All)` event is enabled and points to `ingest-finalize` (no overlapping rules).

---

## 8) Known Good Values (Quick Reference)

- **Bucket:** `maxstache-uploads-2025-mustache-olympics`
- **Region:** `us-east-2`
- **Event ID:** `2025-mustache-olympics`
- **DynamoDB Table:** `MustacheOlympicsSubmissions`
- **Signer URL:** `https://2gflx5uo7lqzj2exr2dgybofrq0uvrhy.lambda-url.us-east-2.on.aws`
- **Submit URL:** `https://lf5ekhxuvwgwuhc34tsis6eldm0ltwzr.lambda-url.us-east-2.on.aws/`

---

## 9) Backlog / Nice-to-Have (Not required today)

- API Gateway + WAF in front of Function URLs (rate limits, managed rules)
- S3 Lifecycle (abort incomplete MPU after 1 day; delete raw after 120 days)
- CloudWatch Dashboards + Alarms (5xx, throttles, S3 anomalies)
- SES email receipt on successful submission
- Optional: S3 → SQS → Lambda for extreme spikes

---

*Last updated: 2025-08-29 (US/Eastern)*
