# Config map — PDI_Azure_Function (keys & purpose)

> This file lists environment variable names used by the PDI_Azure_Function and what they control. **Do not** store secret values here — map these names to KeyVault or your secret manager.

| Key name | Purpose / usage |
|---|---|
| `sapdi` | Azure Storage connection string used for all blob and queue operations. |
| `PDI_ConnectionString` | Connection string to the PDI / staging SQL database used by PDIBatch, Logger, Generic. |
| `PUB_ConnectionString` | Connection string to Publisher DB used by Orchestration.PublisherImport and related DB calls. |
| `IncomingBatchContainer` | Container name where partner uploads appear (BlobTrigger on Batch_Blob). |
| `MainContainer` | Main working container that holds processing/, importing/, templates/, completed/, rejected/, archive/. |
| `PDI_QueueName` | Queue name (PDI) that `Batch_Blob` and others write to and `Queue_Process` listens to. |
| `PUB_QueueName` | Queue name (Publisher) that `Queue_Process` writes to and `QueueImport` consumes. |
| `SMTP_Password` | Password or API key used by `PDISendGrid` to send emails (store in KeyVault). |
| `SMTP_FromEmail` | From address used for notification emails. |
| `SMTP_FromName` | Display name for notification emails. |

**Operational note:** Ensure all secrets (connection strings and SMTP secrets) are stored in KeyVault and not in function app settings in plain text for Production. Use Managed Identity where possible.
