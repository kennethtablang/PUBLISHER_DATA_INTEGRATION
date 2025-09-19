# Runbook: Reprocess a failed file (PDI_Azure_Function)

**When to use:** a batch or file failed during validation, orchestration, or Publisher import and you need to re-run it. Use QA first to validate the steps.

**Prerequisites**

- Ops access to Azure Storage (blob containers) and queue (PDI & PUB) or use Azure Storage Explorer.
- Access to function logs (Application Insights or Function App logs).
- Contact info for dev if errors persist.

---

## High-level steps (safe, manual)

1. **Identify failed file**  
   - Find the filename (e.g., `myfile.xlsx`) in `rejected/` or examine the status page (`/api/Status`) for recent failures.
2. **Inspect logs & validation messages**  
   - Use the Status page link `ValidationMessages/{fileName}` or check function logs for `Batch_Blob` / `Queue_Process` / `Queue_Import`.
   - If errors are content/schema related, identify and fix source file (ask partner to correct).
3. **If failure was transient (infrastructure/timeout)**:
   - Copy or move the blob from `rejected/{file}` or `completed/` back to `processing/{file}` in `MainContainer`.
   - Re-create the DTO on `%PDI_QueueName%` with the same `FileName` and `Batch_ID`. Use Azure Storage Explorer or queue tooling:
     - Create base64 JSON like: `{"FileName":"myfile.xlsx","Batch_ID":"123","RetryCount":0}` and add to queue.
4. **If failure is content-related and source fixed**:
   - Upload corrected file to `incoming/` and let BatchBlob create a new BatchID and workflow.
5. **Monitor**  
   - Watch function logs and the status page. Confirm the blob moves `processing/` → `importing/` → `completed/`.
6. **Notify stakeholders**  
   - Update BA with file name, original error, action taken, and final outcome (completed or rejected).

---

## Quick checks & troubleshooting

- **Queue length is growing**: check function scale limits and App Insights for throttling. Consider reprocessing in smaller batches.
- **Template missing**: ensure `templates/{templateName}` exists in `MainContainer`; if missing, retrieve from template repository and upload.
- **Email notifications not coming**: verify SMTP / SendGrid config environment variables and mailbox quotas.

---
