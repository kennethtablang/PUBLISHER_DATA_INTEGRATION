# ETL Sequence — Publisher Data Integration (high-level)

1. **Partner / Fund manager** uploads a file to `IncomingBatchContainer` (blob).
2. **Batch_Blob** (BlobTrigger) fires:
   - If ZIP: extracts entries, registers batch (PDIBatch.LoadBatch), saves entries to `processing/{entry}`, enqueue DTO messages to `%PDI_QueueName%`, move original to `archive/{name}`.
   - If XLSX: records a batch or single job via `PDIBatch.RecordBatch`/`RecordSingle`, moves file to `processing/{name}`, enqueues message to `%PDI_QueueName%`.
3. **Queue_Process** (QueueTrigger on `%PDI_QueueName%`) fires:
   - Opens `processing/{file}` via `PDIStream`, loads template if needed, calls `Orchestration.ProcessFile(curStream, templateStream, retryCount, log)`.
   - If successful and `orch.FileStatus` true: moves `processing/{file}` → `importing/{file}` and enqueues DTO to `%PUB_QueueName%`.
   - If failed: moves file to `rejected/{file}` and logs / notifies.
4. **QueueImport** (QueueTrigger on `%PUB_QueueName%`) fires:
   - Opens `importing/{file}`, initializes `Orchestration.PublisherImport(jobID, log)`, on success sends notification and moves file to `completed/{file}`, on failure moves to `rejected/{file}` and sends error notifications.
5. **Publisher** consumes the import request, renders documents (PDF/HTML), and stores artifacts in the long-term archive. Delivery and audit steps are then performed by downstream systems.

---

**Correlation & traceability:** `BatchID`, `File_ID`, `Job_ID`, and `FileRunID` (ExecutionContext invocation id) are used to trace a file end-to-end across functions, queues and DB records.
