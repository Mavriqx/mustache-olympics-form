# Mustache Olympics Submission System Overview

This document describes the full stack of the Mustache Olympics submission system: client, AWS resources, IAM policies, and operational procedures.

---

## 1) Client (Front-End)

- Hosted at **Netlify** (`mustache-olympics-submission.netlify.app`).
- Repo: `mustache-olympics-form` (GitHub).
- Main file: `index.html` with embedded CSS/JS.
- Assets: favicon set, logo (Black.png), uppy.min.js, uppy.min.css.

### Key UI rules
- Title: **“Enter the 2025 Mustache Olympics”**
- Sections: **Required Information** + **Optional Information** with styled headers.
- Uploader: inline Dashboard, 180px height, single progress bar, no “Done” button, remove (X) enabled.
- Allowed file types: **video/mp4**, **video/quicktime** (MP4 or MOV).
- Max size: **1 GB**, max length: **30 seconds**.
- Submit button: burgundy, right-aligned, disabled until upload completes.

### Constants embedded in `index.html`
SIGNER_BASE = https://2gflx5uo7lqzj2exr2dgybofrq0uvrhy.lambda-url.us-east-2.on.aws
LAMBDA_ENDPOINT = https://lf5ekhxuvwgwuhc34tsis6eldm0ltwzr.lambda-url.us-east-2.on.aws/
BUCKET = maxstache-uploads-2025-mustache-olympics
REGION = us-east-2

pgsql
Copy code

---

## 2) AWS Inventory (Production)

### 2.1 S3 — Uploads Bucket
- **Name**: `maxstache-uploads-2025-mustache-olympics` (Region: us-east-2)
- Public access: Blocked
- Versioning: ON
- Object Ownership: Bucket owner enforced
- Lifecycle: *optional later*  
  - Abort incomplete MPUs after 1 day  
  - Expire raw uploads after 120 days

**CORS (bucket-level):**
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

bash
Copy code
event={event_id}/participant_id={participant_id}/submission_id={submission_id}/video.mp4
Tags (finalized objects):

bash
Copy code
event_id, participant_id, submission_id, content=video, source=direct-s3
2.2 DynamoDB — Submissions Table
Name: MustacheOlympicsSubmissions

PK: submissionId (String)

Typical fields:

submissionId (PK)

participantId, eventId, email, firstName, lastName, city, state

s3_key, backup_status, backup_ts

optional: phone, socialPlatform, socialHandle, eventSource, accordion fields

Write pattern:

sql
Copy code
ConditionExpression: attribute_not_exists(submissionId)
2.3 Lambdas
A) Signer — maxstache-s3-signer
Trigger: Lambda Function URL (Auth: NONE; CORS restricted)

Endpoints:

POST /create-mpu → { uploadId, key, submissionId }

POST /sign-part → { url }

POST /complete-mpu → { ok, location }

Env: REGION, BUCKET, EVENT_ID, ALLOWED_ORIGINS

IAM (minimum):

json
Copy code
{
  "Action": [
    "s3:CreateMultipartUpload",
    "s3:UploadPart",
    "s3:CompleteMultipartUpload",
    "s3:AbortMultipartUpload"
  ],
  "Resource": "arn:aws:s3:::maxstache-uploads-2025-mustache-olympics/*"
}
B) Ingest Finalize — maxstache-ingest-finalize
Trigger: S3 ObjectCreated (all)

What it does:

Adds final object tags

Updates DynamoDB (backup_status=ok, backup_ts, etc.)

Env: DDB_TABLE=MustacheOlympicsSubmissions

IAM:

json
Copy code
{
  "Action": ["dynamodb:UpdateItem"],
  "Resource": "arn:aws:dynamodb:us-east-2:760474663439:table/MustacheOlympicsSubmissions"
},
{
  "Action": ["s3:PutObjectTagging"],
  "Resource": "arn:aws:s3:::maxstache-uploads-2025-mustache-olympics/*"
}
C) Submit API — maxstache-submit
Trigger: Lambda Function URL (public, CORS restricted)

What it does:

Validates payload, requires submissionId

PutItem into DynamoDB with conditional insert

IAM:

json
Copy code
{
  "Action": ["dynamodb:PutItem"],
  "Resource": "arn:aws:dynamodb:us-east-2:760474663439:table/MustacheOlympicsSubmissions"
}
3) Security & CORS
Function URLs (Signer, Submit):

Allowed origins: Netlify app, maxstache.com, localhost (dev)

Allowed headers: content-type

Allowed methods: OPTIONS, POST

S3:

Block Public Access: Enabled

Versioning: Enabled

CORS as defined above

4) Client-Side Validation
Uppy enforces:

File type: MP4/MOV only

Max size: 1 GB

Single file

Consent checkbox required before submission

5) Server-Side Validation
submit API: idempotent on submissionId

ingest-finalize: only updates/tag objects matching convention

DynamoDB: conditional write prevents duplicates

6) Data Model (DynamoDB)
PK: submissionId

Fields:

participantId (derived from email)

eventId (2025-mustache-olympics)

email, name, city, state

s3_key, backup_status, backup_ts

optional social/phone/talent fields

7) Testing Playbook
Before starting

Confirm Function URLs CORS headers are correct

Confirm S3 CORS includes Netlify origin(s)

End-to-end smoke

Open form with ?email=your@email

Upload MP4/MOV (~20–50 MB test)

Watch progress → “Upload complete ✅”

Click Submit → redirected to Thank You page

Verify in AWS:

S3 object present, correct tags

DynamoDB item present, backup_status=ok

CloudWatch logs show no errors

If issues

Duplicate submissions blocked by DynamoDB

Upload stalls: check network tab + signer endpoints

Tags missing: confirm S3 event notification on bucket

8) Future Enhancements (Backlog)
API Gateway + WAF in front of signer/submit

Captcha/Turnstile on submit

SES email receipts

CloudWatch dashboards/alarms

S3→SQS→Lambda for extreme scale

S3 lifecycle (abort MPU after 1d, expire raw after 120d)

9) Quick Reference
Bucket: maxstache-uploads-2025-mustache-olympics

Region: us-east-2

DynamoDB: MustacheOlympicsSubmissions

Event ID: 2025-mustache-olympics

Signer URL: https://2gflx5uo7lqzj2exr2dgybofrq0uvrhy.lambda-url.us-east-2.on.aws

Submit URL: https://lf5ekhxuvwgwuhc34tsis6eldm0ltwzr.lambda-url.us-east-2.on.aws/

pgsql
Copy code
