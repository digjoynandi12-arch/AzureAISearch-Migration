# Azure AI Search: Backup & Migration Runbook

Oct 7, 2026 · @Bunnyger

## Overview

Azure AI Search has no built-in backup or restore, so a copy into a new resource group is rebuilt in three layers: the service, the object definitions, and the indexed documents. Your source service stays untouched until you decommission it in Step 8.

| Layer | What it includes | How it moves |
| --- | --- | --- |
| Service | SKU, replicas, partitions, region, semantic ranker, identity, network rules | Recreated with Azure CLI (Steps 2–3) |
| Definitions | Indexes, synonym maps, aliases, data sources, skillsets, indexers | Exported and re-imported as JSON over REST (Steps 4–5) |
| Documents | The indexed content itself | Re-run indexers from source, or dump and reload JSON (Step 6) |

### Prerequisites

- [ ] Azure CLI 2.50+ signed in (`az login`) with Contributor on both resource groups; Azure Cloud Shell (Bash) works and already has `az`, `curl` and `jq`
- [ ] Admin key or the Search Service Contributor + Search Index Data Contributor roles on both services
- [ ] Connection strings / keys for every data source and any Azure AI services or Azure OpenAI resources used by skillsets (they are redacted on export)
- [ ] A maintenance window if apps write to the index during the copy
- [ ] For Option B in Step 6: .NET 8 SDK or Python 3.9+ on the machine running the backup tool

All scripts below use these variables; set them once per shell session:

```bash
SRC_RG=old-rg
SRC_SVC=old-search
DST_RG=new-rg
DST_SVC=new-search          # must be globally unique
LOCATION=centralindia
API=2024-07-01
SRC=https://$SRC_SVC.search.windows.net
DST=https://$DST_SVC.search.windows.net
```

## Step 1 — Inventory the source service

Record the service settings and every object before you build anything, so the new service can be checked against this list in Step 7.

1. Save the service configuration:

   ```bash
   mkdir -p backup && cd backup
   az search service show -g $SRC_RG -n $SRC_SVC > service.json
   jq '{sku: .sku.name, replicas: .replicaCount, partitions: .partitionCount, location, semanticSearch, hostingMode, publicNetworkAccess, identity: .identity.type, ipRules: .networkRuleSet.ipRules, authOptions, disableLocalAuth}' service.json
   ```
2. Get the source admin key:

   ```bash
   SRC_KEY=$(az search admin-key show -g $SRC_RG --service-name $SRC_SVC --query primaryKey -o tsv)
   ```
3. List every object and note the names:

   ```bash
   for t in indexes synonymmaps aliases datasources skillsets indexers; do
     echo "== $t"; curl -s "$SRC/$t?api-version=$API&\$select=name" -H "api-key: $SRC_KEY" | jq -r '.value[].name'
   done
   ```
4. Record document counts per index (your baseline for validation):

   ```bash
   for i in $(curl -s "$SRC/indexes?api-version=$API&\$select=name" -H "api-key: $SRC_KEY" | jq -r '.value[].name'); do
     echo "$i: $(curl -s "$SRC/indexes/$i/docs/\$count?api-version=$API" -H "api-key: $SRC_KEY")"
   done | tee doc-counts.txt
   ```
5. Note which apps, Function apps or Logic apps call this service, and whether they use keys or Entra ID. You will repoint them in Step 8.

If `aliases` returns an error, your service has none; skip it throughout.

## Step 2 — Create the new resource group and search service

Create the target service with the same SKU, scale and region you recorded in Step 1; the tier must be large enough to hold the source's storage and object counts.

1. Create the resource group:

   ```bash
   az group create -n $DST_RG -l $LOCATION
   ```
2. Create the search service, matching the values from `service.json`:

   ```bash
   az search service create -g $DST_RG -n $DST_SVC -l $LOCATION \
     --sku standard \
     --replica-count 1 --partition-count 1 \
     --semantic-search free \
     --identity-type SystemAssigned \
     --auth-options aadOrApiKey
   ```

   Change `--sku` (`basic`, `standard`, `standard2`, `standard3`, `storage_optimized_l1`…), replicas, partitions and `--semantic-search` (`disabled`, `free`, `standard`) to match the source. Provisioning usually takes a few minutes.
3. Get the target admin key:

   ```bash
   DST_KEY=$(az search admin-key show -g $DST_RG --service-name $DST_SVC --query primaryKey -o tsv)
   ```

Alternative: in the portal, open the source service → Automation → **Export template**, change the service name, and deploy it into the new resource group. This captures every setting at once, including network rules.

## Step 3 — Configure security, identity and networking

The new service has a new managed identity and no role assignments, so grant its access before creating indexers, or they will fail on first run.

1. Get the new service's identity:

   ```bash
   DST_MI=$(az search service show -g $DST_RG -n $DST_SVC --query identity.principalId -o tsv)
   ```
2. Grant it the same roles the old identity has on each data source and AI resource:

   | Resource | Role to grant |
   | --- | --- |
   | Storage account (blob, ADLS, table) | Storage Blob Data Reader (Storage Table Data Reader for tables) |
   | Storage account used for knowledge store or enrichment cache | Storage Blob Data Contributor |
   | Azure OpenAI / AI Foundry (embedding skill, vectorizer) | Cognitive Services OpenAI User |
   | Azure AI services (multi-service skills) | Cognitive Services User |
   | Cosmos DB | Cosmos DB Account Reader Role + a data-plane read role |
   | Azure SQL | A contained database user for the identity with `db_datareader` |

   ```bash
   az role assignment create --assignee-object-id $DST_MI --assignee-principal-type ServicePrincipal \
     --role "Storage Blob Data Reader" \
     --scope /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<account>
   ```

   To see what the old identity has: `az role assignment list --assignee <old principalId> --all -o table`.
3. Reassign user and app access. If apps use Entra ID instead of keys, give them Search Index Data Reader (query) or Search Index Data Contributor (write) on the new service.
4. Recreate networking: IP firewall rules (`az search service update --ip-rules`), private endpoints and DNS records, and shared private links for indexers that reach private data sources. Shared private links must be approved again on each target resource.
5. Re-enable customer-managed keys if the source used them; indexes and synonym maps then need their `encryptionKey` property (kept in the exported JSON).

## Step 4 — Export object definitions

This script saves each object as its own JSON file under `backup/<type>/`, which is your durable backup of the configuration.

```bash
#!/usr/bin/env bash
# export-definitions.sh — run from the backup folder
set -euo pipefail
for t in synonymmaps indexes aliases datasources skillsets indexers; do
  mkdir -p "$t"
  names=$(curl -sf "$SRC/$t?api-version=$API&\$select=name" -H "api-key: $SRC_KEY" | jq -r '.value[].name' || true)
  for n in $names; do
    curl -sf "$SRC/$t/$n?api-version=$API" -H "api-key: $SRC_KEY" \
      | jq 'del(."@odata.context", ."@odata.etag")' > "$t/$n.json"
    echo "exported $t/$n"
  done
done
```

Run it and check the output:

```bash
chmod +x export-definitions.sh && ./export-definitions.sh
find . -name '*.json' | sort
```

Keep the `backup` folder in source control or a storage account; from now on you can rebuild these objects anywhere from it.

## Step 5 — Fix secrets and recreate objects in order

Exported JSON has every secret blanked, so fill them in first, then create objects in dependency order: synonym maps → indexes → aliases → data sources → skillsets → indexers.

### 5a. Put the secrets back

| File | Field returned as `null` | Set it to |
| --- | --- | --- |
| `datasources/*.json` | `credentials.connectionString` | The full connection string, or `ResourceId=/subscriptions/.../storageAccounts/<acct>;` for managed identity |
| `skillsets/*.json` | `cognitiveServices.key` | Your Azure AI services key (or switch to `#Microsoft.Azure.Search.AIServicesByIdentity`) |
| `skillsets/*.json` | `apiKey` on Azure OpenAI embedding skills | The Azure OpenAI key, or remove it to use the managed identity |
| `indexes/*.json` | `vectorSearch.vectorizers[].azureOpenAIParameters.apiKey` | Same as above |

Example for a blob data source using managed identity:

```bash
jq '.credentials.connectionString = "ResourceId=/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<acct>;"' \
  datasources/my-blob-ds.json > tmp && mv tmp datasources/my-blob-ds.json
```

Check nothing is left blank before importing:

```bash
grep -rl '"connectionString": null\|"key": null\|"apiKey": null' . || echo "no blank secrets"
```

### 5b. Import into the new service

Indexers are created **disabled** so they don't start until you choose a data path in Step 6.

```bash
#!/usr/bin/env bash
# import-definitions.sh — run from the backup folder
set -euo pipefail
for t in synonymmaps indexes aliases datasources skillsets indexers; do
  [ -d "$t" ] || continue
  for f in "$t"/*.json; do
    [ -e "$f" ] || continue
    n=$(basename "$f" .json)
    body=$(cat "$f")
    [ "$t" = indexers ] && body=$(echo "$body" | jq '.disabled = true')
    code=$(curl -s -o /tmp/resp.json -w '%{http_code}' -X PUT \
      "$DST/$t/$n?api-version=$API" \
      -H "api-key: $DST_KEY" -H 'Content-Type: application/json' -d "$body")
    if [[ $code == 20* ]]; then echo "OK   $t/$n"; else echo "FAIL $t/$n ($code)"; cat /tmp/resp.json; fi
  done
done
```

```bash
chmod +x import-definitions.sh && ./import-definitions.sh
```

Every line should read `OK`. A `FAIL` prints the service's error message; fix that file and rerun the script, since a PUT simply overwrites what already exists.

## Step 6 — Migrate the data

Use Option A if indexers fill your indexes and Option B if your application pushes documents; Option B also gives you an offline copy of the documents.

|  | Option A: re-run indexers | Option B: dump and reload |
| --- | --- | --- |
| Best for | Indexer-based indexes, anything with vectors that aren't retrievable | Push-model indexes, offline backup, no access to the source data |
| Copies | Everything, rebuilt from the source | Only fields marked `retrievable` |
| Cost | Skillset and embedding calls run again | Your machine's time plus bandwidth |
| Duration | Hours to days for large sets with AI enrichment | Minutes to hours |

### Option A — Re-run indexers

1. Enable, reset and run each indexer:

   ```bash
   for f in indexers/*.json; do
     n=$(basename "$f" .json)
     jq '.disabled = false' "$f" | curl -s -X PUT "$DST/indexers/$n?api-version=$API" \
       -H "api-key: $DST_KEY" -H 'Content-Type: application/json' -d @- > /dev/null
     curl -s -X POST "$DST/indexers/$n/reset?api-version=$API" -H "api-key: $DST_KEY" -H 'Content-Length: 0'
     curl -s -X POST "$DST/indexers/$n/run?api-version=$API"   -H "api-key: $DST_KEY" -H 'Content-Length: 0'
     echo "started $n"
   done
   ```
2. Watch progress until each shows `success`:

   ```bash
   curl -s "$DST/indexers/<name>/status?api-version=$API" -H "api-key: $DST_KEY" \
     | jq '{status: .lastResult.status, processed: .lastResult.itemsProcessed, failed: .lastResult.itemsFailed, errors: .lastResult.errors[:3]}'
   ```

   A single run is capped (24 hours on most tiers, less for skillsets), so large indexers pick up where they stopped on their next scheduled run. Add a schedule to the indexer JSON if it has none.

### Option B — Dump and reload with Microsoft's backup tool

1. Get the sample: the .NET version is `index-backup-restore` in the [azure-search-dotnet-utilities](https://github.com/Azure-Samples/azure-search-dotnet-utilities) repo; a Python version is in [azure-search-python-samples](https://github.com/Azure-Samples/azure-search-python-samples).
2. Fill in `appsettings.json` (.NET):

   ```json
   {
     "SourceSearchServiceName": "old-search",
     "SourceAdminKey": "<source admin key>",
     "SourceIndexName": "my-index",
     "TargetSearchServiceName": "new-search",
     "TargetAdminKey": "<target admin key>",
     "TargetIndexName": "my-index",
     "BackupDirectory": "index-backup"
   }
   ```
3. Run it from the project folder with `dotnet run`. It writes the schema and documents as JSON under `BackupDirectory`, then uploads them to the target index. Repeat per index.
4. Keep the `BackupDirectory` output with your Step 4 definitions; together they are a complete point-in-time backup.

Things that catch people with Option B:

- Fields with `retrievable: false` (often vector fields, or `stored: false`) come back empty. Those indexes need Option A.
- Search can't page past 100,000 results with `skip`; very large indexes must be exported in slices using a filter on a sortable field (for example a date or ID range).
- Indexers on the new service start with no change-tracking state. Their first run after reload reprocesses every source document; leave them disabled until you accept that cost.

## Step 7 — Validate

The copy is ready when object lists, document counts and test queries match the source.

1. Compare document counts with your Step 1 baseline:

   ```bash
   for i in $(curl -s "$DST/indexes?api-version=$API&\$select=name" -H "api-key: $DST_KEY" | jq -r '.value[].name'); do
     echo "$i: $(curl -s "$DST/indexes/$i/docs/\$count?api-version=$API" -H "api-key: $DST_KEY")"
   done | diff doc-counts.txt - && echo "counts match"
   ```

   Counts can lag a few seconds after indexing ends; recheck before treating a difference as a failure.
2. Run the same query on both services and compare the top results:

   ```bash
   Q='{"search":"<a typical query>","top":5,"select":"<key field>"}'
   for s in "$SRC $SRC_KEY" "$DST $DST_KEY"; do set -- $s
     curl -s -X POST "$1/indexes/<index>/docs/search?api-version=$API" -H "api-key: $2" \
       -H 'Content-Type: application/json' -d "$Q" | jq -c '[.value[] | ."@search.score"]'
   done
   ```
3. Test the features that depend on settings, not data: a semantic query (`"queryType":"semantic"`), a vector or hybrid query, synonyms, and the alias name if your apps query through one.
4. Confirm indexers show `success` with zero failed items, and that their schedules are set.

- [ ] Object lists match Step 1
- [ ] Document counts match
- [ ] Sample queries return the same top results
- [ ] Semantic, vector and synonym queries work
- [ ] Indexers healthy and scheduled

## Step 8 — Cut over and decommission

Switch clients to the new endpoint, run both services side by side for a few days, then delete the old one only after a final backup.

1. If apps push documents, pause writes to the old service or send writes to both until cutover, so no updates are lost.
2. Update every client to the new endpoint `https://<new-search>.search.windows.net` and its key or Entra ID role. Common places: app settings, Key Vault secrets, Azure OpenAI "on your data" or AI Foundry connections, Logic Apps, Power Platform connectors.
3. Disable the old service's indexers so both services aren't paying for enrichment:

   ```bash
   for f in indexers/*.json; do n=$(basename "$f" .json)
     jq '.disabled = true' "$f" | curl -s -X PUT "$SRC/indexers/$n?api-version=$API" \
       -H "api-key: $SRC_KEY" -H 'Content-Type: application/json' -d @- > /dev/null; done
   ```
4. Monitor the new service for a few days (portal → Monitoring → Metrics: search latency, throttled queries).
5. Take a final backup of the old service (rerun Step 4 and Option B), then delete it. Deletion can't be undone:

   ```bash
   az search service delete -g $SRC_RG -n $SRC_SVC
   ```

To keep a standing backup from now on, schedule Step 4's export script and Option B's tool to run regularly against the new service.

## Troubleshooting and rollback

Rollback is simple until Step 8: the old service is unchanged, so point clients back to it and delete the new resource group (`az group delete -n $DST_RG`).

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `403` on any REST call | Wrong key, or local auth disabled | Recheck `SRC_KEY` / `DST_KEY`; if keys are off, use `az account get-access-token --resource https://search.azure.com` and send `Authorization: Bearer` |
| Index PUT fails with a quota error | Target tier has fewer indexes or less storage than the source needs | Raise the tier or partitions, then rerun the import |
| Index PUT fails on semantic or vector config | Semantic ranker disabled, or the region lacks a feature | `az search service update --semantic-search free`; check region support |
| Data source fails with credentials error | Secret still `null`, or managed identity lacks a role | Redo Step 5a or Step 3; role assignments can take 10+ minutes to apply |
| Indexer fails on skills | Azure OpenAI / AI services key or role missing | Fill `apiKey`, or grant Cognitive Services OpenAI User to the new identity |
| Restored index has empty vector fields | Fields not retrievable, so Option B couldn't read them | Use Option A for that index |
| Document count lower than source | Indexer still running, or Option B hit the 100,000 `skip` limit | Wait and recheck; export large indexes in filtered slices |

### Sources

- [Azure AI Search FAQ: move, backup and restore](https://learn.microsoft.com/azure/search/search-faq-frequently-asked-questions)
- [Back up and restore an Azure AI Search index (sample)](https://learn.microsoft.com/en-us/samples/azure-samples/azure-search-dotnet-utilities/azure-search-backup-restore-index)
- [Microsoft Q&A: backing up Azure AI Search data](https://learn.microsoft.com/en-za/answers/questions/1792032/how-do-i-backup-data-of-azure-ai-search-service)
- [Microsoft Q&A: re-indexing after migration](https://learn.microsoft.com/en-us/answers/questions/1616243/avoid-re-indexing-all-documents-after-migration-of)
