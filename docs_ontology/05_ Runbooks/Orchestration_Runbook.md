# Runbook — Orchestration basic checks

## Quick health check

- Verify Processing records: check `pdi_Processing_Queue_Log` for `Job_ID` status and timestamps.
- Check A.I. / Function logs for Orchestration errors (`ErrorMessage`) and processing stage changes.

## Test a full flow (QA)

1. Upload a small valid sample to `IncomingBatchContainer`.
2. Confirm `Batch_Blob` -> `processing/{file}` -> `Queue_Process` -> `Orchestration.ProcessFile(...)` runs (check function logs).
3. Confirm validation step (FileIntegrityCheck) passes (or appropriate validation records are logged).
4. Confirm `Extract` wrote staging rows and `GetStagingPivotTable(jobID)` returns expected rows.
5. Confirm `DocumentProcessing.processStaging` updated/created rows in `pdi_Publisher_Documents`.
6. If running local/import enabled: confirm `LoadTempTable` created temp table and bulk-copy succeeded, and `sp_pdi_IMPORT_DATA_temp` / `sp_pdi_IMPORT_DATA` returned `"Complete"`.
7. Verify final processing stage set to `Complete` in `pdi_Processing_Queue_Log`.

## Re-run / retry

- To retry a failed run increment `RetryCount` and call ProcessFile with a fresh DTO (Azure queue or via function). Orchestration will call `Cleanup(jobID)` if `RetryCount > 0` — verify cleanup succeeded before reprocessing.
