# Cost_Optimization_Challenge
## Managing Billing Records in Azure Serverless Architecture

---

## Step-by-Step Implementation Plan to Cost-Optimize Billing Records in a Serverless Azure Cosmos DB Architecture

### 1. Architecture Overview

![Architecture Diagram](./architecture.png)

'''


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

'''



### 2. Implement Data Archival

Our goal is to move records older than 3 months from Cosmos DB to Azure Blob Storage on a schedule.

#### a) Create an Azure Function to Archive Data  
- **Trigger:** Timer (for example, daily at UTC/PST time)  
- **Steps:**  
  1. Query Cosmos DB for records older than 3 months:  
     ```
     SELECT * FROM c WHERE c.createdAt < @cutoff
     ```  
     where `@cutoff` is calculated as the date 90 days before today.  
  2. For each record:  
     - Write to Blob Storage (as a JSON file), organized in folders by year/month for efficient lookup.  
     - Verify the write succeeded.  
     - Delete or soft-delete the record from Cosmos DB only if the above succeeded.  
  3. Log all activity and errors for traceability.

#### Python Pseudocode for Archival Function

import datetime
import json
from azure.cosmos import CosmosClient, exceptions
from azure.storage.blob import BlobServiceClient

COSMOS_CONN_STR = "<your_cosmos_connection_string>"
BLOB_CONN_STR = "<your_blob_connection_string>"
DATABASE_NAME = "Billing"
CONTAINER_NAME = "Records"
ARCHIVE_CONTAINER_NAME = "billing-archive"
BATCH_SIZE = 100 # Number of records to process per batch

def archive_old_billing_records():
cosmos = CosmosClient(COSMOS_CONN_STR)
blob_service = BlobServiceClient.from_connection_string(BLOB_CONN_STR)
db = cosmos.get_database_client(DATABASE_NAME)
container = db.get_container_client(CONTAINER_NAME)
archive_container = blob_service.get_container_client(ARCHIVE_CONTAINER_NAME)

text
cutoff_date = (datetime.datetime.utcnow() - datetime.timedelta(days=90)).isoformat()
query = f"SELECT * FROM c WHERE c.createdAt < '{cutoff_date}'"

try:
    # Use continuation tokens for batch processing
    items_iterable = container.query_items(
        query=query, enable_cross_partition_query=True
    )
    batch = []
    for item in items_iterable:
        batch.append(item)
        if len(batch) >= BATCH_SIZE:
            process_batch(batch, archive_container, container)
            batch = []
    # Process remaining items in the last batch
    if batch:
        process_batch(batch, archive_container, container)

except exceptions.CosmosHttpResponseError as e:
    print(f"Cosmos DB query error: {e}")
except Exception as ex:
    print(f"Unexpected error during archival: {ex}")
def process_batch(batch, archive_container, container):
for record in batch:
try:
folder = record.get("createdAt", "")[:7] # YYYY-MM
blob_path = f"{folder}/{record['id']}.json"
# Upload record to Blob Storage
archive_container.upload_blob(blob_path, json.dumps(record), overwrite=True)

text
        # Soft delete in Cosmos DB: set a flag instead of hard delete (if schema supports it)
        record['isArchived'] = True  # Example flag, depends on your schema
        container.upsert_item(record)

        # Optionally: For hard delete, uncomment next line after verifying archival
        # container.delete_item(item=record['id'], partition_key=record['partitionKey'])
        
        print(f"Archived and soft-deleted record id: {record['id']}")

    except Exception as e:
        print(f"Error processing record id {record['id']}: {e}")
if name == "main":
archive_old_billing_records()

text

---

### 3. Unified Access Layer

Wrap Cosmos DB and Blob Storage access in a logic layer within your API or Azure Function:

- **On read:**  
  1. Attempt to fetch from Cosmos DB.  
  2. If not found, fetch from Blob Storage.  
  3. Return the record to the client.

- **On write:**  
  1. Always write to Cosmos DB (no logic change).

#### Pseudocode for Reading Billing Records

def get_billing_record(record_id, partition_key):
try:
return cosmosdb_client.read_item(record_id, partition_key)
except cosmosdb_client.exceptions.NotFound:
folder = determine_folder_based_on_id_logic(record_id)
blob = blob_client.download_blob(f"{folder}/{record_id}.json")
return json.loads(blob.readall())

text

---

### 4. Operations

- **Deployment:** Use Azure CLI / Bicep / ARM templates to automate deployment of Functions, storage containers, and schedules.  
- **Code management:** Store functions and API logic in version control with clear documentation.  
- **Monitoring:** Enable Application Insights and Blob Storage logging for real-time monitoring and alerting.  
- **Recovery:** Use soft-delete with grace periods when moving records; regularly test restoring data from Blob Storage back to Cosmos DB.

---

### 5. Handling Edge Cases & Reliability

- If Blob Storage reads are slow, consider using Azure Blob Storage Hot access tier or caching “hot” cold records in-memory.  
- Use Blob Storage soft-delete and versioning to protect against accidental deletions.  
- If users update old archived records, retrieve from Blob, restore into Cosmos DB, then allow the update operation.  
- Run periodic audits (e.g., weekly) to reconcile data consistency between Cosmos DB and Blob Storage.

---

## Summary Table

| Goal                     | Solution Step                          |
|-------------------------|--------------------------------------|
| Simplicity & maintainability | Azure Functions + foldered Blob Storage |
| No downtime / data loss    | Verify-then-delete, retries, logs      |
| No API changes           | Unified access logic layer             |

---

## References

- [Azure Cosmos DB Well-Architected Guidance](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/cosmos-db)

---

