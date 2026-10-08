# Azure AI Search: Backup & Migration Runbook

Oct 7, 2026 · @Bunnyger

## Overview

Azure AI Search has no built-in backup or restore, so a copy into a new resource group is rebuilt in three layers: the service, the object definitions, and the indexed documents. Your source service stays untouched until you decommission it in Step 8.

This version is set up to run against the test environment from **Azure AI Search: Test Environment Setup**. It uses that guide's saved variables, your existing resource groups and its test objects (`hotels-push`, `docs-blob`, `hotel-synonyms`, `docs-blob-ds`, `docs-blob-indexer`), and shows the expected result at each step.

| Layer | What it includes | How it moves |
| --- | --- | --- |
| Service | SKU, replicas, partitions, region, semantic ranker, identity, network rules | Recreated with Azure CLI (Steps 2–3) |
| Definitions | Indexes, synonym maps, aliases, data sources, skillsets, indexers | Exported and re-imported as JSON over REST (Steps 4–5) |
| Documents | The indexed content itself | Re-run indexers from source, or dump and reload JSON (Step 6) |

### Prerequisites

- [ ] Test environment built and verified (Test Environment Setup guide, Steps 1–6)
- [ ] Local bash terminal with `az` (signed in), `curl`, `jq` and `git`
- [ ] Contributor on both resource groups
- [ ] .NET 8 SDK for Option B in Step 6 (check with `dotnet --version`)

All scripts below use the variables saved by the test setup guide. Run this at the start of every new terminal session:

```bash
source ~/search-test.env
SRC_KEY=$(az search admin-key show -g $SRC_RG --service-name $SRC_SVC --query primaryKey -o tsv)
STG_CONN=$(az storage account show-connection-string -n $STG -g $SRC_RG --query connectionString -o tsv)
# After Step 2 has created the destination service, also run:
# DST_KEY=$(az search admin-key show -g $DST_RG --service-name $DST_SVC --query primaryKey -o tsv)
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
5. Note which apps call this service, so you can repoint them in Step 8. The test environment has none, so skip this.

Expected in the test environment: indexes `docs-blob` and `hotels-push`, synonym map `hotel-synonyms`, data source `docs-blob-ds`, indexer `docs-blob-indexer`, no skillsets, and counts `docs-blob: 4` and `hotels-push: 5`. The `aliases` line may print a `jq` error or nothing, because API version 2024-07-01 has no aliases; ignore it.

## Step 2 — Create the destination search service

Your destination resource group already exists, so only the service is created. It matches the test source: Basic tier, 1 replica, 1 partition, free semantic ranker and a managed identity.

1. Create the service. It takes about 3–10 minutes:

   ```bash
   az search service create -g $DST_RG -n $DST_SVC -l $LOCATION \
     --sku basic --replica-count 1 --partition-count 1 \
     --semantic-search free \
     --identity-type SystemAssigned \
     --auth-options aadOrApiKey -o table
   ```

   `$LOCATION` is the source's region. To test a copy into another region, use a different value here.
2. Check that it matches the source:

   ```bash
   for g in "$SRC_RG $SRC_SVC" "$DST_RG $DST_SVC"; do set -- $g
     az search service show -g $1 -n $2 | jq -c '{sku: .sku.name, replicas: .replicaCount, partitions: .partitionCount, semanticSearch, identity: .identity.type}'
   done
   ```

   Both lines should be identical.
3. Get the destination admin key:

   ```bash
   DST_KEY=$(az search admin-key show -g $DST_RG --service-name $DST_SVC --query primaryKey -o tsv)
   ```

For a production service, match `--sku`, replicas, partitions and `--semantic-search` to `service.json`, or deploy the source's exported template (portal → Automation → **Export template**) under a new name.

## Step 3 — Configure security, identity and networking

The test data source signs in with a connection string, so the destination needs no role assignments, and both test services are public, so there are no network rules to copy.

1. Confirm the destination has a managed identity (it should print an ID):

   ```bash
   DST_MI=$(az search service show -g $DST_RG -n $DST_SVC --query identity.principalId -o tsv)
   echo $DST_MI
   ```
2. Optional: to also test managed-identity sign-in, grant the identity read access to the test storage account, then use the `ResourceId` option in Step 5a. The role can take 5–10 minutes to apply.

   ```bash
   az role assignment create --assignee-object-id $DST_MI --assignee-principal-type ServicePrincipal \
     --role "Storage Blob Data Reader" \
     --scope $(az storage account show -n $STG -g $SRC_RG --query id -o tsv)
   ```
3. In a production migration, this step is also where you recreate role assignments on every data source and AI resource, app access roles, IP rules, private endpoints, shared private links and customer-managed keys.

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

Expect five files besides `service.json`: `synonymmaps/hotel-synonyms.json`, `indexes/docs-blob.json`, `indexes/hotels-push.json`, `datasources/docs-blob-ds.json` and `indexers/docs-blob-indexer.json`. The `aliases` and `skillsets` folders stay empty.

Keep the `backup` folder; from now on you can rebuild these objects anywhere from it.

## Step 5 — Fix secrets and recreate objects in order

Exported JSON has every secret blanked, so fill them in first, then create objects in dependency order: synonym maps → indexes → aliases → data sources → skillsets → indexers.

### 5a. Put the secrets back

In the test, one secret needs filling: the connection string of `docs-blob-ds`. There are no skillsets or vectorizers. To confirm it was blanked:

```bash
jq .credentials datasources/docs-blob-ds.json
```

Put the storage connection string back (`STG_CONN` was loaded at the top of this runbook). Run only one of these:

```bash
# Connection string (default)
jq --arg cs "$STG_CONN" '.credentials.connectionString = $cs' \
  datasources/docs-blob-ds.json > tmp && mv tmp datasources/docs-blob-ds.json

# Managed identity (only if you did Step 3.2)
# jq --arg cs "ResourceId=$(az storage account show -n $STG -g $SRC_RG --query id -o tsv);" \
#   '.credentials.connectionString = $cs' datasources/docs-blob-ds.json > tmp && mv tmp datasources/docs-blob-ds.json
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

Expect five `OK` lines: `synonymmaps/hotel-synonyms`, `indexes/docs-blob`, `indexes/hotels-push`, `datasources/docs-blob-ds` and `indexers/docs-blob-indexer`. A `FAIL` prints the service's error message; fix that file and rerun the script, since a PUT simply overwrites what already exists.

## Step 6 — Migrate the data

Use Option A if indexers fill your indexes and Option B if your application pushes documents; Option B also gives you an offline copy of the documents.

|  | Option A: re-run indexers | Option B: dump and reload |
| --- | --- | --- |
| Best for | Indexer-based indexes, anything with vectors that aren't retrievable | Push-model indexes, offline backup, no access to the source data |
| Copies | Everything, rebuilt from the source | Only fields marked `retrievable` |
| Cost | Skillset and embedding calls run again | Your machine's time plus bandwidth |
| Duration | Hours to days for large sets with AI enrichment | Minutes to hours |

### Option A — Re-run indexers (for `docs-blob`)

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
   curl -s "$DST/indexers/docs-blob-indexer/status?api-version=$API" -H "api-key: $DST_KEY" \
     | jq '{status: .lastResult.status, processed: .lastResult.itemsProcessed, failed: .lastResult.itemsFailed, errors: .lastResult.errors[:3]}'
   ```

   Expected after about a minute: `"status": "success"`, `"processed": 4`, `"failed": 0`. If it shows a credentials error, recheck Step 5a.

   A single run is capped (24 hours on most tiers, less for skillsets), so large indexers pick up where they stopped on their next scheduled run. Add a schedule to the indexer JSON if it has none.

### Option B — Dump and reload with Microsoft's backup tool (for `hotels-push`)

1. Get the sample and find its project folder:

   ```bash
   cd ~ && git clone https://github.com/Azure-Samples/azure-search-dotnet-utilities.git
   APPDIR=$(dirname "$(find ~/azure-search-dotnet-utilities/index-backup-restore -name appsettings.json -path '*v11*' | head -1)")
   echo "$APPDIR"
   ```

   If nothing prints, run `ls -R ~/azure-search-dotnet-utilities/index-backup-restore` and set `APPDIR` to the v11 folder that holds `appsettings.json`.
2. Fill in its settings from your variables, keeping any other keys the file already has:

   ```bash
   jq --arg s "$SRC_SVC" --arg sk "$SRC_KEY" --arg t "$DST_SVC" --arg tk "$DST_KEY" \
     '. + {SourceSearchServiceName: $s, SourceAdminKey: $sk, SourceIndexName: "hotels-push",
           TargetSearchServiceName: $t, TargetAdminKey: $tk, TargetIndexName: "hotels-push",
           BackupDirectory: "index-backup"}' \
     "$APPDIR/appsettings.json" > /tmp/appsettings.json && mv /tmp/appsettings.json "$APPDIR/appsettings.json"
   ```

   The file now holds both admin keys, so don't commit it.
3. Run the tool:

   ```bash
   cd "$APPDIR" && dotnet run
   ```

   It backs up `hotels-push` to JSON under `index-backup`, then uploads it to the destination. If it stops because `hotels-push` already exists there (created empty in Step 5), delete it and run again:

   ```bash
   curl -s -X DELETE "$DST/indexes/hotels-push?api-version=$API" -H "api-key: $DST_KEY"
   dotnet run
   ```
4. Confirm five documents arrived:

   ```bash
   curl -s "$DST/indexes/hotels-push/docs/\$count?api-version=$API" -H "api-key: $DST_KEY"
   ```
5. Keep the `index-backup` output with your Step 4 definitions; together they are a complete point-in-time backup. Return to the backup folder before Step 7: `cd ~/backup` (or wherever you ran Step 1).

Things that catch people with Option B:

- Fields with `retrievable: false` (often vector fields, or `stored: false`) come back empty. Those indexes need Option A.
- Search can't page past 100,000 results with `skip`; very large indexes must be exported in slices using a filter on a sortable field (for example a date or ID range).
- Indexers on the new service start with no change-tracking state. Their first run after reload reprocesses every source document; leave them disabled until you accept that cost.

## Step 7 — Validate

The copy is ready when object lists, document counts and test queries match the source.

1. Compare document counts with your Step 1 baseline (run from the backup folder):

   ```bash
   for i in $(curl -s "$DST/indexes?api-version=$API&\$select=name" -H "api-key: $DST_KEY" | jq -r '.value[].name'); do
     echo "$i: $(curl -s "$DST/indexes/$i/docs/\$count?api-version=$API" -H "api-key: $DST_KEY")"
   done | diff doc-counts.txt - && echo "counts match"
   ```

   Expected: `counts match`. Counts can lag a few seconds after indexing; recheck before treating a difference as a failure.
2. Run the test queries on both services side by side:

   ```bash
   q() { curl -s -X POST "$1/indexes/$2/docs/search?api-version=$API" -H "api-key: $3" \
           -H 'Content-Type: application/json' -d "$4" | jq -c '[.value[] | (.hotelName // .title)]'; }
   
   for side in "SOURCE $SRC $SRC_KEY" "DESTINATION $DST $DST_KEY"; do set -- $side; echo "== $1"
     echo -n "synonym  : "; q $2 hotels-push $3 '{"search":"inn","select":"hotelName"}'
     echo -n "semantic : "; q $2 hotels-push $3 '{"search":"quiet place to relax","queryType":"semantic","semanticConfiguration":"default","select":"hotelName","top":3}'
     echo -n "hidden   : "; q $2 hotels-push $3 '{"search":"monsoon","select":"hotelName"}'
     echo -n "indexer  : "; q $2 docs-blob $3 '{"search":"refund","select":"title"}'
   done
   ```

   | Query | Source | Destination (expected) |
   | --- | --- | --- |
   | synonym | Brahmaputra View Inn, Highway Motel | Same |
   | semantic | Hill Top Resort first | Same (order may vary slightly) |
   | hidden | Tea Garden Retreat | **Empty**: Option B couldn't copy the non-retrievable `internalNote` field |
   | indexer | Refund policy | Same |

   The empty `hidden` result is the expected data loss for Option B. In a real migration, any index with non-retrievable fields needs Option A instead.
3. Confirm the destination indexer is healthy and kept its schedule (expect `"success"` and `"PT2H"`):

   ```bash
   curl -s "$DST/indexers/docs-blob-indexer/status?api-version=$API" -H "api-key: $DST_KEY" | jq .lastResult.status
   curl -s "$DST/indexers/docs-blob-indexer?api-version=$API" -H "api-key: $DST_KEY" | jq .schedule.interval
   ```

- [ ] Object lists match Step 1
- [ ] Document counts match
- [ ] Sample queries return the same top results
- [ ] Semantic, vector and synonym queries work
- [ ] Indexers healthy and scheduled

## Step 8 — Cut over and decommission

The test environment has no client apps to switch, so this step just retires the source the way you would in production.

1. Disable the source's indexers so both services aren't processing the same data (run from the backup folder):

   ```bash
   for f in indexers/*.json; do n=$(basename "$f" .json)
     jq '.disabled = true' "$f" | curl -s -X PUT "$SRC/indexers/$n?api-version=$API" \
       -H "api-key: $SRC_KEY" -H 'Content-Type: application/json' -d @- > /dev/null; done
   curl -s "$SRC/indexers/docs-blob-indexer?api-version=$API" -H "api-key: $SRC_KEY" | jq .disabled
   ```

   It should print `true`.
2. In production you would now point every client at `$DST` with its key or Entra ID role, and monitor for a few days. Skip this for the test.
3. When you're done testing, run **Clean up** in the Test Environment Setup guide. It deletes both search services and the storage account and leaves your resource groups in place.

For a real migration, take a final backup of the source (rerun Step 4 and Option B) before deleting it, since deletion can't be undone.

## Troubleshooting and rollback

Rollback is simple until Step 8: the source service is unchanged, so point clients back to it and delete only the destination service (`az search service delete -g $DST_RG -n $DST_SVC --yes`). Don't delete the resource group; it held other resources before this test.

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
