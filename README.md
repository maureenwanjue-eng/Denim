# Denim

Deployment notes for a user-uploaded video platform with accounts, creator uploads, catalog discovery, and paid access to premium videos.

## Reference architecture

```text
Browser
  ├── Web application
  ├── API ───── PostgreSQL (accounts, catalog, purchases, entitlements)
  │      ├──── Redis / managed queue
  │      ├──── Private object storage (uploads and outputs)
  │      └──── Payment provider webhooks
  └── CDN ◄──── short-lived authorized HLS playback

Upload events → scan / validation → transcoding workers → moderation → approved catalog
```

Keep the database, workers, and object-storage origins private. The application API should be the authority for authentication, age eligibility, moderation status, and playback entitlements. Do not rely on browser-side checks to protect premium media.

## Suggested initial stack

| Concern | AWS reference | Alternatives |
|---|---|---|
| Web and API | ECS/Fargate behind an Application Load Balancer | Google Cloud Run, Azure Container Apps |
| Relational data | RDS PostgreSQL, Multi-AZ for production | Cloud SQL, Azure Database for PostgreSQL |
| Cache and queue | ElastiCache Redis plus SQS | Managed Redis and a managed queue from another cloud |
| Object storage | Private S3 buckets | R2, Backblaze B2, Google Cloud Storage |
| Transcoding | MediaConvert or isolated FFmpeg workers | Managed transcoding service or autoscaled worker pool |
| Video delivery | CloudFront with signed cookies or URLs | A CDN that supports private-origin authorization |
| Secrets | AWS Secrets Manager | Cloud Secret Manager, Key Vault, Vault |
| Observability | CloudWatch and OpenTelemetry | Grafana Cloud, Datadog, or equivalent |

Before selecting a host, CDN, storage provider, or payment processor, confirm that its current terms and account review process support the platform's content category and operating model.

## Repository and infrastructure layout

A practical infrastructure-as-code layout is:

```text
infra/
  modules/
    network/
    database/
    cache-queue/
    api/
    workers/
    storage-cdn/
  environments/
    staging/
    production/
```

Use Terraform or OpenTofu to define repeatable environments. Keep environment-specific configuration in the deployment system, not committed credentials. Use separate cloud accounts or projects for production and non-production where practical.

## Deployment sequence

1. Create DNS records, TLS certificates, private networking, and security groups.
2. Provision PostgreSQL, queue/cache services, and private storage buckets.
3. Configure secrets and workload identities; grant each service only the access it requires.
4. Apply database migrations as a dedicated release step.
5. Deploy the API and web application behind the load balancer.
6. Deploy upload-event consumers and autoscaled transcoding workers.
7. Configure CDN origin access so that private objects cannot be fetched directly.
8. Test the complete upload, review, publication, purchase, playback, refund, and access-revocation flows in staging.
9. Enable backups, monitoring, alerting, and a tested recovery procedure.
10. Roll out production traffic gradually and watch error rates, queue depth, and egress costs.

## Upload pipeline

Use direct-to-object-storage uploads to avoid proxying large files through API containers:

```text
Creator → API requests upload session
API → checks account state and upload quotas; creates pending record
API → returns a short-lived, object-scoped upload URL
Browser → uploads to private incoming bucket
Storage event → queue
Worker → validates format and size, scans, and transcodes
Review workflow → approves or rejects the asset
Publisher → moves/marks approved outputs and updates catalog state
```

Keep original uploads separate from approved outputs. Apply maximum size and duration limits, allowed-container checks, quotas, malware scanning, encryption at rest, and lifecycle rules for abandoned or rejected uploads. Use multipart upload for large files and clean up incomplete multipart sessions.

## Video processing and playback

Transcode approved media to an adaptive bitrate format such as HLS, with multiple resolutions and segment durations appropriate to the chosen player and CDN. Store manifests and segments in private object storage. Avoid exposing permanent origin URLs.

For playback, the API should verify all required conditions on every new playback session:

- Authenticated, active account.
- Required age/identity checks completed for the applicable jurisdiction and product policy.
- Video has passed review and is currently available.
- A valid purchase or subscription entitlement exists for premium media.
- The account and asset are not blocked or subject to a hold.

On success, issue a short-lived CDN token or signed cookie scoped to the specific asset. Ensure refunds, chargebacks, account suspension, and content takedowns revoke future playback authorization. Keep authorization decisions server-side.

## Accounts and data model

Use a maintained authentication library or identity provider where possible. If handling passwords directly, store only a modern password hash (for example, Argon2id), use secure recovery flows, and protect sessions with `HttpOnly`, `Secure`, and appropriately configured `SameSite` cookies. Add MFA for creators and administrators.

Core relational entities commonly include:

```text
users
videos
upload_jobs
moderation_reviews
purchases
entitlements
payment_webhook_events
reports
audit_events
creator_payouts
```

Keep a unique provider-event identifier for payment webhooks and process each event idempotently. Never store raw payment-card data; use a payment provider's hosted checkout or tokenized components. Maintain a ledger for financial movements and reconcile it against provider reports.

## Moderation and auditability

Make review a required state transition, not a UI-only label. Track uploader identity, provenance, required consent/age documentation, review status, reviewer, timestamps, decisions, reports, appeals, and takedown history. Restrict sensitive records to authorized reviewers, encrypt them, log access, and define retention and deletion policies.

A typical asset state machine is:

```text
created → uploaded → scanning → processing → pending_review
                                      ├── approved → published
                                      └── rejected
published → restricted / removed
```

Automated classification can help prioritize review but should not replace the required verification and review process. Provide reporting and takedown workflows and an audited administrative interface.

## Security and operations checklist

- TLS everywhere; private database and worker subnets.
- Private buckets with public access blocked and CDN origin access restricted.
- Short-lived upload and playback credentials scoped to individual resources.
- Least-privilege IAM/workload roles and managed secrets.
- Rate limits for sign-in, recovery, uploads, reports, checkout, and playback.
- CSRF protection for cookie-authenticated state-changing requests.
- Input validation, request-size limits, dependency scanning, and container scanning.
- Audit records for administrative, moderation, purchase, and entitlement changes.
- Redacted logs; never log passwords, tokens, identity documents, or payment secrets.
- Encrypted backups, point-in-time database recovery, and tested restore procedures.
- Alerts for failed payments, webhook errors, queue backlog, worker failures, storage growth, and unusual traffic.
- A documented incident response, content-removal, and account-recovery runbook.

## Cost and scaling notes

Video delivery and storage egress usually dominate infrastructure cost. Measure average bitrate, watch time, cache-hit ratio, transcode minutes, storage retention, and regional traffic before setting pricing. Use lifecycle policies for originals and renditions, CDN caching for approved segments, and autoscaling workers based on queue depth. Set budget alerts and per-account upload quotas from the start.

Begin with a small production footprint and load-test the API, upload initiation, queue, and playback authorization paths separately. Scale the API on request latency and CPU, and scale transcoding workers on queue age and pending job count. Avoid scaling solely on CPU when the limiting factor is storage or network throughput.

## Deployment provider checklist

For each candidate vendor, verify current written terms and service limits for:

- The exact content and payment categories.
- Upload size, storage, transcoding, and bandwidth/egress.
- CDN private-origin access and signed-request support.
- Data residency, encryption, backup, and deletion options.
- DDoS controls, support response time, and account escalation process.
- Webhook delivery guarantees and payment/refund/chargeback capabilities.
- Export and migration options if the service is discontinued or the account changes.

Treat this document as an architecture starting point; adapt service names and limits to the selected cloud region, provider account, expected traffic, and application implementation.
