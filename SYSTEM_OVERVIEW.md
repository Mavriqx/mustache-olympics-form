# Mustache Olympics — System Overview
_Last updated: 2025-08-29 • Owner: Mav • Event ID: `2025-mustache-olympics`_

> **Purpose:** One doc that explains how the submission system works end-to-end (website → S3 → DynamoDB → Lambdas), with all critical configs, policies, testing steps, and what to change later if we scale.

---

## 0) TL;DR (What exists today)

- **Frontend (Netlify)**: Single submission page (`index.html`) using **Uppy** (direct-to-S3 via multipart).  
  - Allowed video formats: **MP4, MOV**; Max size: **1 GB**; Progress + single file.
  - Submit button unlocks **only after** upload completes.

- **Storage (S3)**: `maxstache-uploads-2025-mustache-olympics` (Region **us-east-2**).  
  - Key format: `event={event_id}/participant_id={participant_id}/submission_id={submission_id}/video.mp4`  
  - Object tags: `event_id`, `participant_id`, `submission_id`, `content=video`, `source=direct-s3`

- **Database (DynamoDB)**: `MustacheOlympicsSubmissions`  
  - PK: `submissionId` (ISO string UUID)  
  - Stores ingest status + form fields. Idempotent writes.

- **Lambdas**  
  1) **Signer** (`maxstache-s3-signer`, Function URL) → `/create-mpu`, `/sign-part`, `/complete-mpu`  
  2) **Ingest Finalize** (`ingest-finalize`, S3 ObjectCreated) → tags object + updates Dynamo  
  3) **Submit API** (`maxstache-submit`, Function URL) → writes final record

- **CORS**: Restricted to Netlify site + `maxstache.com` and `www.maxstache.com` (+ localhost dev).

---

## 1) User Journey

1) **Registration (main website)**  
   - User lands on a page that emails them a unique link to the submission portal with `?email=<addr>`.
   - On load, the portal reads the email from the URL (or `localStorage`) and shows **“Submitting for: <email>”**.

2) **Upload (browser → S3 directly)**  
   - User selects a **.mp4/.mov ≤ 1 GB**.  
   - Uppy asks the **Signer Lambda** to:
     - `POST /create-mpu` → returns `{uploadId, key, submissionId}`  
     - `POST /sign-part` for each chunk  
     - `POST /complete-mpu` after all parts  
   - On success, the file exists at the canonical **S3 key** above.

3) **Ingest Finalize (S3 event)**  
   - S3 `ObjectCreated` → **Ingest Finalize Lambda** fires:  
     - Adds tags (`event_id`, `participant_id`, `submission_id`, `content=video`, `source=direct-s3`)  
     - `UpdateItem` in Dynamo: `backup_status=ok`, `s3_key`, `backup_ts`

4) **Submit (form → Submit API)**  
   - The page enables **Submit** only after upload completes.  
   - The payload (user fields + `submissionId` + `s3Key`) is posted to the **Submit Lambda**.  
   - Dynamo record is upserted **idempotently** (one item per `submissionId`).  
   - User is redirected to the thank-you page.

> **Note:** Legacy Uploadcare path is retired/disabled.

---

## 2) Frontend (Netlify)

- **Repo**: `mustache-olympics-form`  
  - `index.html` (submission page)  
  - `uppy.min.js`, `uppy.min.css` (self-hosted Uppy v3)  
  - `Black.png` (logo) and any icons

- **Key UI rules**  
  - Title: **“Enter the 2025 Mustache Olympics”**  
  - “Required Information” + “Optional Information” sections, styled.  
  - **Uploader**: inline Dashboard; **180px** height; **“Done”** button hidden; single progress bar; remove (X) enabled.  
  - Allowed files: **`video/mp4`, `video/quicktime`** only.  
  - Submit button: **burgundy, right-aligned**, disabled until upload completes.

- **Constants embedded in `index.html`**  
  - `SIGNER_BASE = https://2gflx5uo7lqzj2exr2dgybofrq0uvrhy.lambda-url.us-east-2.on.aws`  
  - `LAMBDA_ENDPOINT = https://lf5ekhxuvwgwuhc34tsis6eldm0ltwzr.lambda-url.us-east-2.on.aws/`  
  - `BUCKET = maxstache-uploads-2025-mustache-olympics`  
  - `REGION = us-east-2`

---

## 3) AWS Inventory (production)

### 3.1 S3 — Uploads Bucket
- **Name**: `maxstache-uploads-2025-mustache-olympics` (Region: **us-east-2**)
- **Public access**: Blocked  
- **Versioning**: ON  
- **CORS** (bucket-level):
  ```json
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
# Mustache Olympics — System Overview
_Last updated: 2025-08-29 • Owner: Mav • Event ID: `2025-mustache-olympics`_

> **Purpose:** One doc that explains how the submission system works end-to-end (website → S3 → DynamoDB → Lambdas), with all critical configs, policies, testing steps, and what to change later if we scale.

---

## 0) TL;DR (What exists today)

- **Frontend (Netlify)**: Single submission page (`index.html`) using **Uppy** (direct-to-S3 via multipart).  
  - Allowed video formats: **MP4, MOV**; Max size: **1 GB**; Progress + single file.
  - Submit button unlocks **only after** upload completes.

- **Storage (S3)**: `maxstache-uploads-2025-mustache-olympics` (Region **us-east-2**).  
  - Key format: `event={event_id}/participant_id={participant_id}/submission_id={submission_id}/video.mp4`  
  - Object tags: `event_id`, `participant_id`, `submission_id`, `content=video`, `source=direct-s3`

- **Database (DynamoDB)**: `MustacheOlympicsSubmissions`  
  - PK: `submissionId` (ISO string UUID)  
  - Stores ingest status + form fields. Idempotent writes.

- **Lambdas**  
  1) **Signer** (`maxstache-s3-signer`, Function URL) → `/create-mpu`, `/sign-part`, `/complete-mpu`  
  2) **Ingest Finalize** (`ingest-finalize`, S3 ObjectCreated) → tags object + updates Dynamo  
  3) **Submit API** (`maxstache-submit`, Function URL) → writes final record

- **CORS**: Restricted to Netlify site + `maxstache.com` and `www.maxstache.com` (+ localhost dev).

---

## 1) User Journey

1) **Registration (main website)**  
   - User lands on a page that emails them a unique link to the submission portal with `?email=<addr>`.
   - On load, the portal reads the email from the URL (or `localStorage`) and shows **“Submitting for: <email>”**.

2) **Upload (browser → S3 directly)**  
   - User selects a **.mp4/.mov ≤ 1 GB**.  
   - Uppy asks the **Signer Lambda** to:
     - `POST /create-mpu` → returns `{uploadId, key, submissionId}`  
     - `POST /sign-part` for each chunk  
     - `POST /complete-mpu` after all parts  
   - On success, the file exists at the canonical **S3 key** above.

3) **Ingest Finalize (S3 event)**  
   - S3 `ObjectCreated` → **Ingest Finalize Lambda** fires:  
     - Adds tags (`event_id`, `participant_id`, `submission_id`, `content=video`, `source=direct-s3`)  
     - `UpdateItem` in Dynamo: `backup_status=ok`, `s3_key`, `backup_ts`

4) **Submit (form → Submit API)**  
   - The page enables **Submit** only after upload completes.  
   - The payload (user fields + `submissionId` + `s3Key`) is posted to the **Submit Lambda**.  
   - Dynamo record is upserted **idempotently** (one item per `submissionId`).  
   - User is redirected to the thank-you page.

> **Note:** Legacy Uploadcare path is retired/disabled.

---

## 2) Frontend (Netlify)

- **Repo**: `mustache-olympics-form`  
  - `index.html` (submission page)  
  - `uppy.min.js`, `uppy.min.css` (self-hosted Uppy v3)  
  - `Black.png` (logo) and any icons

- **Key UI rules**  
  - Title: **“Enter the 2025 Mustache Olympics”**  
  - “Required Information” + “Optional Information” sections, styled.  
  - **Uploader**: inline Dashboard; **180px** height; **“Done”** button hidden; single progress bar; remove (X) enabled.  
  - Allowed files: **`video/mp4`, `video/quicktime`** only.  
  - Submit button: **burgundy, right-aligned**, disabled until upload completes.

- **Constants embedded in `index.html`**  
  - `SIGNER_BASE = https://2gflx5uo7lqzj2exr2dgybofrq0uvrhy.lambda-url.us-east-2.on.aws`  
  - `LAMBDA_ENDPOINT = https://lf5ekhxuvwgwuhc34tsis6eldm0ltwzr.lambda-url.us-east-2.on.aws/`  
  - `BUCKET = maxstache-uploads-2025-mustache-olympics`  
  - `REGION = us-east-2`

---

## 3) AWS Inventory (production)

### 3.1 S3 — Uploads Bucket
- **Name**: `maxstache-uploads-2025-mustache-olympics` (Region: **us-east-2**)
- **Public access**: Blocked  
- **Versioning**: ON  
- **CORS** (bucket-level):
  ```json
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
Object key convention:
event={event_id}/participant_id={participant_id}/submission_id={submission_id}/video.mp4

Object tags (finalized):
event_id, participant_id, submission_id, content=video, source=direct-s3

(Optional later) Lifecycle rules: abort stale MPU after 1 day; expire raw after 120 days.

### 3.2 DynamoDB — Submissions Table

- **Table Name:** `MustacheOlympicsSubmissions`
- **Partition Key (PK):** `submissionId` (String)

**Typical fields**
- submissionId (PK)
- participantId, eventId, email, firstName, lastName, city, state
- s3_key, backup_status (e.g., ok), backup_ts (ISO timestamp)
- Optional: phone, socialPlatform, socialHandle, eventSource, accordion fields

**Write pattern**
```text
ConditionExpression: attribute_not_exists(submissionId)
3.3 Lambdas
A) Signer — maxstache-s3-signer
Trigger: Lambda Function URL (Auth: NONE; CORS restricted to your origins)

Handler: app.handler (Python 3.12)

Endpoints

POST /create-mpu → { uploadId, key, submissionId }

POST /sign-part → { url }

POST /complete-mpu → { ok: true, location }

Env Vars

REGION=us-east-2

BUCKET=maxstache-uploads-2025-mustache-olympics

EVENT_ID=2025-mustache-olympics

ALLOWED_ORIGINS=https://mustache-olympics-submission.netlify.app,https://maxstache.com,https://www.maxstache.com,http://localhost:3000,http://localhost:5173

IAM Minimum

json
Copy code
{
  "Effect": "Allow",
  "Action": [
    "s3:CreateMultipartUpload",
    "s3:UploadPart",
    "s3:CompleteMultipartUpload",
    "s3:AbortMultipartUpload"
  ],
  "Resource": "arn:aws:s3:::maxstache-uploads-2025-mustache-olympics/*"
}
B) Ingest Finalize — ingest-finalize
Trigger: S3 ObjectCreated (All) on the uploads bucket.

What it does:

Adds final object tags

Updates Dynamo item with backup_status=ok, s3_key, backup_ts, event_id, participant_id

Env Var

DDB_TABLE=MustacheOlympicsSubmissions

IAM

json
Copy code
{
  "Effect": "Allow",
  "Action": ["dynamodb:UpdateItem"],
  "Resource": "arn:aws:dynamodb:us-east-2:760474663439:table/MustacheOlympicsSubmissions"
},
{
  "Effect": "Allow",
  "Action": ["s3:PutObjectTagging"],
  "Resource": "arn:aws:s3:::maxstache-uploads-2025-mustache-olympics/*"
}
C) Submit API — maxstache-submit
Trigger: Lambda Function URL (public; CORS restricted to your origins)

Handler: lambda_function.lambda_handler (Python 3.12)

What it does

Parses form payload; ensures submissionId present

Writes to Dynamo with attribute_not_exists(submissionId)

Env Vars

REGION=us-east-2

DDB_TABLE=MustacheOlympicsSubmissions

EVENT_ID=2025-mustache-olympics

IAM

json
Copy code
{
  "Effect": "Allow",
  "Action": ["dynamodb:PutItem"],
  "Resource": "arn:aws:dynamodb:us-east-2:760474663439:table/MustacheOlympicsSubmissions"
}
4) IAM Policies (exact)
4.1 Submit API Role
json
Copy code
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
4.2 Signer Role
json
Copy code
{
  "Version":"2012-10-17",
  "Statement":[
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
4.3 Ingest Finalize Role
json
Copy code
{
  "Version": "2012-10-17",
  "Statement": [
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
5) CORS & Security
Function URLs (Signer, Submit)

Allowed origins:

https://mustache-olympics-submission.netlify.app

https://maxstache.com

https://www.maxstache.com

http://localhost:3000

http://localhost:5173

Allowed headers: content-type

Allowed methods: OPTIONS, POST

S3 CORS set to the same origins.

Block Public Access: ON.

Versioning: ON.

6) Client-Side Validation
Uppy restricts file types: .mp4, .mov

Max file size: 1 GB

Consent checkbox required

7) Server-Side Validation
Submit API enforces submissionId uniqueness (attribute_not_exists).

Ingest Finalize only tags items matching naming convention.

MIME type validation still mandatory.

8) Data Model (DynamoDB)
Partition key

submissionId (String UUID; same as from create-mpu)

Attributes

submissionId (PK)

participantId (from email)

eventId (default: 2025-mustache-olympics)

email, firstName, lastName, city, state

s3_key

backup_status (ok, missing)

backup_ts (ISO)

Optional: phone, socialPlatform, socialHandle, eventSource, accordion answers

9) Testing Playbook
Before starting

Confirm Function URLs show correct CORS.

Confirm S3 bucket CORS includes Netlify origins.

Smoke test

Open submission page with ?email=test@example.com

Upload .mp4 or .mov (20–50 MB for quick test).

Watch progress bar → confirm “Upload complete”.

Submit → should redirect to thank-you page.

Check AWS

S3: Object at canonical key with tags

DynamoDB: Item with submissionId + backup_status=ok

CloudWatch: No errors in ingest-finalize, maxstache-submit, maxstache-s3-signer

10) Future Enhancements (backlog)
API Gateway + WAF in front of signer/submit

S3 Lifecycle: abort incomplete MPUs (1d), delete raw after 120d

CloudWatch dashboards/alarms (errors, throttles, SQS backlog if added)

Captcha (Turnstile/reCAPTCHA) on submit

SES email receipts with submissionId

Optional: S3→SQS→Lambda pipeline for extreme scale

🔑 Quick Reference
Bucket: maxstache-uploads-2025-mustache-olympics (us-east-2)

Table: MustacheOlympicsSubmissions

Event ID: 2025-mustache-olympics

Signer URL: https://2gflx5uo7lqzj2exr2dgybofrq0uvrhy.lambda-url.us-east-2.on.aws

Submit URL: https://lf5ekhxuvwgwuhc34tsis6eldm0ltwzr.lambda-url.us-east-2.on.aws/

yaml
Copy code
