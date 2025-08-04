
# Cost_Optimization_Challenge
## Managing Billing Records in Azure Serverless Architecture

---

## Step-by-Step Implementation Plan to Cost-Optimize Billing Records in a Serverless Azure Cosmos DB Architecture

### 1. Architecture Overview

![Architecture Diagram](./architecture.png)

```
┌────────────┐       ┌────────────┐        ┌─────────────┐
│   Client   │──────▶│   API/App  │───────▶│ Unified Data│
└────────────┘       └────────────┘        │   Access    │
   ▲                                         │  Layer      │
   │                         ┌────────────┐  └─────────────┘
   │                         │ Cosmos DB  │
   │           (Check Recent │ (≤3 months)│
   │          records first) └────────────┘
   │                         ┌──────────────┐
   │                         │ Blob Storage │
   │    (Else Get from OLD)  │ (>3 months)  │
   └─────────────────────────┴──────────────┘
```

---

### 2. Implement Data Archival

Our goal is to move records older than 3 months from Cosmos DB to Azure Blob Storage on a schedule.

#### a) Create an Azure Function to Archive Data  
- **Trigger:** Timer (e.g., daily at UTC/PST)
- **Steps:**
  1. Query Cosmos DB for records older than 3 months:
     ```sql
     SELECT * FROM c WHERE c.createdAt < @cutoff
     ```
     where `@cutoff` is the date 90 days before today.
  2. For each record:
     - Write to Blob Storage (as a JSON file), organized by year/month folders
     - Verify success
     - Delete or soft-delete the Cosmos DB record only if the write succeeded
  3. Log all activities and errors

---

#### Python Pseudocode for Archival Function

<details>
<summary>Click to expand code</summary>

```python
import datetime
import json
from azure.cosmos import CosmosClient, exceptions
from azure.storage.blob import BlobServiceClient

COSMOS_CONN_STR = "<your_cosmos_connection_string>"
BLOB_CONN_STR = "<your_blob_connection_string>"
DATABASE_NAME = "Billing"
CONTAINER_NAME = "Records"
ARCHIVE_CONTAINER_NAME = "billing-archive"
BATCH_SIZE = 100  # Number of records per batch

def archive_old_billing_records():
    cosmos = CosmosClient(COSMOS_CONN_STR)
    blob_service = BlobServiceClient.from_connection_string(BLOB_CONN_STR)
    db = cosmos.get_database_client(DATABASE_NAME)
    container = db.get_container_client(CONTAINER_NAME)
    archive_container = blob_service.get_container_client(ARCHIVE_CONTAINER_NAME)

    cutoff_date = (datetime.datetime.utcnow() - datetime.timedelta(days=90)).isoformat()
    query = f"SELECT * FROM c WHERE c.createdAt < '{cutoff_date}'"

    try:
        items_iterable = container.query_items(
            query=query, enable_cross_partition_query=True
        )
        batch = []
        for item in items_iterable:
            batch.append(item)
            if len(batch) >= BATCH_SIZE:
                process_batch(batch, archive_container, container)
                batch = []
        if batch:
            process_batch(batch, archive_container, container)

    except exceptions.CosmosHttpResponseError as e:
        print(f"Cosmos DB query error: {e}")
    except Exception as ex:
        print(f"Unexpected error during archival: {ex}")

def process_batch(batch, archive_container, container):
    for record in batch:
        try:
            folder = record.get("createdAt", "")[:7]  # YYYY-MM
            blob_path = f"{folder}/{record['id']}.json"
            archive_container.upload_blob(blob_path, json.dumps(record), overwrite=True)

            # Soft delete (mark as archived)
            record['isArchived'] = True
            container.upsert_item(record)

            # Optional: Hard delete
            # container.delete_item(item=record['id'], partition_key=record['partitionKey'])

            print(f"Archived and soft-deleted record id: {record['id']}")

        except Exception as e:
            print(f"Error processing record id {record['id']}: {e}")

if __name__ == "__main__":
    archive_old_billing_records()
```
</details>

---

### 3. Unified Access Layer

Wrap Cosmos DB and Blob Storage access in your logic/API layer:

- **On read:**
  1. Try Cosmos DB
  2. If not found, try Blob Storage
  3. Return to client

- **On write:**
  - Always write to Cosmos DB

#### Pseudocode for Reading Billing Records

```python
def get_billing_record(record_id, partition_key):
    try:
        return cosmosdb_client.read_item(record_id, partition_key)
    except cosmosdb_client.exceptions.NotFound:
        folder = determine_folder_based_on_id_logic(record_id)
        blob = blob_client.download_blob(f"{folder}/{record_id}.json")
        return json.loads(blob.readall())
```

---

### 4. Operations

- **Deployment:** Use Azure CLI / Bicep / ARM templates
- **Code management:** Version-controlled with docs
- **Monitoring:** Application Insights + Blob Storage logs
- **Recovery:** Use soft-delete and restore testing

---

### 5. Handling Edge Cases & Reliability

- Slow blob reads? Use Hot access tier or cache
- Use Blob soft-delete & versioning for protection
- For updates on archived records, restore to Cosmos DB
- Periodic audits to ensure data consistency

---

## Summary Table

| Goal                      | Solution Step                          |
|---------------------------|----------------------------------------|
| Simplicity & maintainability | Azure Functions + foldered Blob Storage |
| No downtime / data loss     | Verify-then-delete, retries, logs       |
| No API changes            | Unified access logic layer              |

---

## References

- [Azure Cosmos DB Well-Architected Guidance](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/cosmos-db)

---
