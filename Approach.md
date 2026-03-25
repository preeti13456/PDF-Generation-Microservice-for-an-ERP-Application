1. High-Level Architecture
The service is Node.js microservice which incorporates two path one is sync and other is via async generation.The app deployed as a separate process on the same EC2 instance or as a sibling container.
It does not require re-architecting the Django monolith — Django calls it via HTTP.

**Two distinct request paths:**
**Sync** : In the sync functionality we uses Puppeteer to ensure a page is fully loaded , elements can render as well as authorization token is persistent 
instead of only relying on browser caching 
<img width="249" height="202" alt="puppetter" src="https://github.com/user-attachments/assets/2f0e2b86-083e-4193-b272-cb63178853fe" />

**Async**: For async we The request is accepted immediately, a job_id is returned, and the work is enqueued to Redis. 
Bulk workers consume jobs, render PDFs using the shared Puppeteer pool, upload to S3, and emit SSE progress events back to the client.
The client polls GET /jobs/:id/status or listens on an SSE stream.



2. Handling the Scenario Constraints (One by One)
**Constraint 1 — Existing Django monolith (r6i.large, 16GB RAM)**
Deploy the PDF service as a separate Node.js process (or Docker container) on the same or adjacent machine. The Django app calls it via http://localhost:3000 or an internal VPC endpoint. No changes to Django's architecture. Communication is plain HTTP/JSON — Django sends the payload, gets back a URL.
For peak load of ~1000 single requests/min (~17/sec): with each render taking 2–3s and consuming ~400MB, you can safely run 6–8 concurrent Puppeteer workers on a 16GB machine (leaving headroom for Django itself). Use p-limit or a semaphore to cap concurrency. Excess requests queue locally or via BullMQ — they do not get dropped.
**Constraint 2 — Variable document complexity / OOM errors for 500+ line item tables**
This is the most critical operational problem. The solution has three parts:
First, chunk large tables across pages in the HTML template itself. Use CSS page-break-inside: avoid and split the data into batches of 100 rows each, rendered as separate <table> elements. This prevents any single render from trying to lay out 500 rows at once.
Second, set a hard memory limit per Puppeteer page: new_context = await browser.newContext() with --max-old-space-size=350 flag per worker process. If a render exceeds the limit, Node throws ENOMEM — catch it, retry once with chunking forced on, then fail with a clear error code.
Third, recycle browser instances — don't create a new Chromium process per PDF. Maintain a warm browser pool (2–3 Chromium instances), and create a fresh page per render, closing it after use. This prevents tab-level memory accumulation.
**Constraint 3 — Unreliable client connectivity (bulk downloads failing mid-transfer)**
Never stream 100 PDFs directly. Instead:
For bulk downloads, assemble the zip server-side in S3 and return a presigned URL (valid for 24h). The client downloads one file from S3 directly. S3 natively supports HTTP range requests, so any S3-compatible download client can resume an interrupted download automatically.
For the zip assembly itself, stream each PDF from S3 into an archiver stream piped to a new S3 upload (using multipart upload) — never load the full zip into memory.
This means even a client on BSNL 2G who gets disconnected 3 times will eventually finish their download.
**Constraint 4 — Data consistency (stale data in the 87th PDF)**
Snapshot the ERP data at job enqueue time, not at render time.
When the Django app sends a bulk request, it includes the full JSON payload for all documents (or a list of IDs). The bulk job handler immediately serializes this entire payload and stores it in Redis (or S3 for large payloads) keyed to the job_id. Every worker reads from this snapshot, never from the live database.
If Django only sends IDs, the handler must query PostgreSQL once at job creation and store the snapshot. Workers never touch the DB. This guarantees all 100 PDFs in a batch reflect the same data state.
**Constraint 5 — Tamper evidence**
After each PDF is generated, before uploading to S3:

Compute SHA-256(pdf_bytes)
Store the hash in the S3 object's metadata (x-amz-meta-sha256) and also write it to a PostgreSQL pdf_audit table (job_id, document_id, sha256, generated_at, size_bytes)
Embed the hash as invisible metadata inside the PDF itself using pdf-lib — add it to the PDF's XMP metadata or as a document property

To verify: re-download the file, recompute SHA-256, compare against the database record. Any modification (even a single byte) changes the hash.
This requires zero additional infrastructure — just a small audit table and a pdf-lib post-processing step adding ~50ms per document.
**Constraint 6 — Budget constraint (₹12,500/month for ~30,000 PDFs/month)**
30,000 PDFs/month ≈ 1,000/day ≈ ~42/hour — very modest. The expensive part is Puppeteer's memory, not CPU.
Cost breakdown (AWS ap-south-1 Mumbai):

EC2 r6i.large (16GB) shared with Django: already paid — ₹0 marginal
RDS db.r6i.large: already paid
ElastiCache t3.micro (Redis for queues): ~₹800/month
S3 storage: 30,000 × 500KB avg = 15GB/month + bandwidth = ~₹400/month
Data transfer out: ~₹300/month

Total marginal cost: ~₹1,500/month — well under the ₹12,500 cap, leaving room for a t3.small upgrade if needed.
If load grows significantly, the right scale-up is a dedicated t3.medium spot instance for the PDF workers (~₹1,200/month spot) fronted by an SQS queue instead of Redis.

3. API Design
POST /generate-pdf
Body: { template: "invoice"|"purchase_order", data: {...} }
Response 200: { url: "https://s3.../presigned-url", expires_in: 3600 }

POST /generate-bulk
Body: { template: "invoice", documents: [{data: {...}}, ...] }  // max 100
Response 202: { job_id: "uuid", status_url: "/jobs/uuid/status" }

GET /jobs/:id/status
Response: { status: "queued|processing|done|failed", progress: 45, download_url?: "..." }

GET /jobs/:id/stream   (SSE)
Events: { type: "progress", completed: 45, total: 100 }
        { type: "done", download_url: "..." }
        { type: "error", code: "OOM", document_id: "..." }



<img width="828" height="488" alt="Screenshot 2026-03-25 at 11 27 44 PM" src="https://github.com/user-attachments/assets/2ee8a854-2a4b-428d-a34c-347225532c8e" />

5. What I Would Add in Production (Beyond MVP)
Dead-letter queue for permanently failed jobs, alerting when DLQ depth grows.
Per-template HTML sanitization to prevent injection.
Presigned URL rotation after download. S3 lifecycle rule to delete PDFs after 30 days (unless flagged for compliance archival).
Horizontal scaling by moving BullMQ workers to a separate t3.medium spot instance when 30k/month grows to 300k/month.
