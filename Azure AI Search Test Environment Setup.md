# Azure AI Search: Test Environment Setup

Oct 8, 2026 · @Bunnyger

## What you'll build

You'll set up a small source search service in about 20 minutes. Its objects are chosen so that each part of the migration runbook has something real to work on. It uses the same variable names as the runbook, so once it's built you can go straight to Step 1 of the runbook.

| Test object | Type | What it tests in the runbook |
| --- | --- | --- |
| `hotel-synonyms` | Synonym map | Creation order (Step 5b); synonym query check (Step 7) |
| `hotels-push` | Index, 5 documents pushed over REST, semantic config | Option B dump and reload (Step 6) |
| `internalNote` field in `hotels-push` | Non-retrievable field | Shows Option B losing non-retrievable data |
| `docs-blob-ds` | Blob data source with a connection string | Secret redaction and refill (Step 5a) |
| `docs-blob` + `docs-blob-indexer` | Index filled by an indexer from 4 JSON blobs | Option A re-run indexers (Step 6) |

Cost: this uses the **Basic** tier, which is billed by the hour, plus a few cents of storage. The Free tier is limited to one service per subscription and has small limits, so it can't hold both the source and the target. Delete everything when you finish (see Clean up).

If you'd rather not use the CLI, the portal's **Import data** wizard with the built-in hotels sample is a quicker way to get a test index. It only exercises Option A, though.

## Step 1 — Set variables and create the resource group

Open Azure Cloud Shell (Bash) from the portal toolbar. It already has `az`, `curl` and `jq` installed and signed in.

1. Pick the subscription:

   ```bash
   az account list -o table
   az account set --subscription "<subscription name or id>"
   ```
2. Set the variables. Search service and storage account names must be globally unique, so a random suffix is added. **Copy the printed block into a note**: you'll paste it again in any new shell, including when you run the runbook.

   ```bash
   SUFFIX=$RANDOM
   cat > ~/search-test.env <<EOF
   SRC_RG=search-test-src-rg
   SRC_SVC=srchtest-src-$SUFFIX
   DST_RG=search-test-dst-rg
   DST_SVC=srchtest-dst-$SUFFIX
   STG=srchtest$SUFFIX
   LOCATION=centralindia
   API=2024-07-01
   SRC=https://srchtest-src-$SUFFIX.search.windows.net
   DST=https://srchtest-dst-$SUFFIX.search.windows.net
   EOF
   source ~/search-test.env && cat ~/search-test.env
   ```

   The file sits in your Cloud Shell home folder, which persists between sessions, so later you only need `source ~/search-test.env`.
3. Create the source resource group:

   ```bash
   az group create -n $SRC_RG -l $LOCATION -o table
   ```

## Step 2 — Create the search service

This creates a Basic service with semantic ranker and a managed identity, so the runbook's identity and semantic checks have something to test.

1. Create the service. It takes about 3–10 minutes:

   ```bash
   az search service create -g $SRC_RG -n $SRC_SVC -l $LOCATION \
     --sku basic --replica-count 1 --partition-count 1 \
     --semantic-search free \
     --identity-type SystemAssigned \
     --auth-options aadOrApiKey -o table
   ```

   If you get a capacity or region error, set `LOCATION=southindia` (or another nearby region) in `~/search-test.env`, run `source ~/search-test.env` again, and retry from Step 1.3.
2. Get the admin key and confirm the service responds:

   ```bash
   SRC_KEY=$(az search admin-key show -g $SRC_RG --service-name $SRC_SVC --query primaryKey -o tsv)
   curl -s "$SRC/servicestats?api-version=$API" -H "api-key: $SRC_KEY" | jq '.counters.indexesCount'
   ```

   It should print `{ "usage": 0, "quota": 15 }` or similar.

## Step 3 — Create a storage account with sample files

Four small JSON files in a blob container give the indexer something to read.

1. Create the storage account and the container:

   ```bash
   az storage account create -n $STG -g $SRC_RG -l $LOCATION --sku Standard_LRS --kind StorageV2 -o table
   STG_CONN=$(az storage account show-connection-string -n $STG -g $SRC_RG --query connectionString -o tsv)
   az storage container create -n docs --connection-string "$STG_CONN"
   ```
2. Create the sample files:

   ```bash
   mkdir -p ~/sample-docs && cd ~/sample-docs
   echo '{"title":"Password reset","content":"To reset your password, open Settings, choose Security and select Reset password. A link is emailed within five minutes.","category":"Account"}' > doc1.json
   echo '{"title":"Refund policy","content":"Refunds are issued to the original payment method within 7 business days after the returned item is received.","category":"Billing"}' > doc2.json
   echo '{"title":"Wi-Fi troubleshooting","content":"If the internet connection drops, restart the router, then forget and rejoin the network on your device.","category":"Support"}' > doc3.json
   echo '{"title":"Shipping times","content":"Standard shipping takes 3 to 5 days. Express shipping arrives the next business day for orders placed before noon.","category":"Orders"}' > doc4.json
   ls
   ```
3. Upload them:

   ```bash
   az storage blob upload-batch -d docs -s ~/sample-docs --connection-string "$STG_CONN" -o table
   ```

   You should see four rows, one per file.

## Step 4 — Push index with a synonym map and documents

This index is filled by your own REST calls, not an indexer. That makes it the test case for the runbook's Option B. It also has a deliberately non-retrievable field, `internalNote`.

1. Create the synonym map:

   ```bash
   cd ~ && cat > synonyms.json <<'EOF'
   {
     "name": "hotel-synonyms",
     "format": "solr",
     "synonyms": "inn, hotel, motel\nwifi, internet"
   }
   EOF
   curl -s -X PUT "$SRC/synonymmaps/hotel-synonyms?api-version=$API" \
     -H "api-key: $SRC_KEY" -H 'Content-Type: application/json' -d @synonyms.json | jq .name
   ```
2. Create the index:

   ```bash
   cat > hotels-index.json <<'EOF'
   {
     "name": "hotels-push",
     "fields": [
       {"name": "hotelId", "type": "Edm.String", "key": true, "filterable": true},
       {"name": "hotelName", "type": "Edm.String", "searchable": true, "sortable": true},
       {"name": "description", "type": "Edm.String", "searchable": true, "synonymMaps": ["hotel-synonyms"]},
       {"name": "category", "type": "Edm.String", "filterable": true, "facetable": true},
       {"name": "rating", "type": "Edm.Double", "filterable": true, "sortable": true},
       {"name": "internalNote", "type": "Edm.String", "searchable": true, "retrievable": false}
     ],
     "semantic": {
       "configurations": [{
         "name": "default",
         "prioritizedFields": {
           "titleField": {"fieldName": "hotelName"},
           "prioritizedContentFields": [{"fieldName": "description"}]
         }
       }]
     }
   }
   EOF
   curl -s -X PUT "$SRC/indexes/hotels-push?api-version=$API" \
     -H "api-key: $SRC_KEY" -H 'Content-Type: application/json' -d @hotels-index.json | jq .name
   ```
3. Upload five documents:

   ```bash
   cat > hotels-docs.json <<'EOF'
   {"value": [
     {"@search.action": "upload", "hotelId": "1", "hotelName": "Brahmaputra View Inn", "description": "Riverside inn with free wifi and a rooftop restaurant.", "category": "Boutique", "rating": 4.5, "internalNote": "Owner prefers email contact"},
     {"@search.action": "upload", "hotelId": "2", "hotelName": "Hill Top Resort", "description": "Quiet resort in the hills with a spa and mountain views.", "category": "Resort", "rating": 4.2, "internalNote": "Contract renews in March"},
     {"@search.action": "upload", "hotelId": "3", "hotelName": "City Centre Hotel", "description": "Business hotel near the station with meeting rooms and fast internet.", "category": "Business", "rating": 3.9, "internalNote": "Corporate discount 10%"},
     {"@search.action": "upload", "hotelId": "4", "hotelName": "Tea Garden Retreat", "description": "Heritage bungalow inside a working tea estate, with guided walks.", "category": "Boutique", "rating": 4.8, "internalNote": "Closed during monsoon maintenance"},
     {"@search.action": "upload", "hotelId": "5", "hotelName": "Highway Motel", "description": "Budget motel with parking, open 24 hours.", "category": "Budget", "rating": 3.1, "internalNote": "Renovation planned"}
   ]}
   EOF
   curl -s -X POST "$SRC/indexes/hotels-push/docs/index?api-version=$API" \
     -H "api-key: $SRC_KEY" -H 'Content-Type: application/json' -d @hotels-docs.json \
     | jq '[.value[] | .status] | length'
   ```

   It should print `5`. Each line should show `"hotels-push"` or `"hotel-synonyms"`; an `error` object means the JSON or key is wrong.

## Step 5 — Blob data source, index and indexer

This index is filled by an indexer from the four blobs, which is the test case for the runbook's Option A. The data source holds a connection string, which the runbook's export step will return as `null`.

1. Create the data source. The connection string is inserted with `jq` so it is quoted correctly:

   ```bash
   jq -n --arg cs "$STG_CONN" '{name: "docs-blob-ds", type: "azureblob", credentials: {connectionString: $cs}, container: {name: "docs"}}' \
     | curl -s -X PUT "$SRC/datasources/docs-blob-ds?api-version=$API" \
         -H "api-key: $SRC_KEY" -H 'Content-Type: application/json' -d @- | jq .name
   ```
2. Create the index:

   ```bash
   cat > docs-index.json <<'EOF'
   {
     "name": "docs-blob",
     "fields": [
       {"name": "id", "type": "Edm.String", "key": true},
       {"name": "title", "type": "Edm.String", "searchable": true},
       {"name": "content", "type": "Edm.String", "searchable": true},
       {"name": "category", "type": "Edm.String", "filterable": true, "facetable": true},
       {"name": "metadata_storage_name", "type": "Edm.String", "filterable": true}
     ]
   }
   EOF
   curl -s -X PUT "$SRC/indexes/docs-blob?api-version=$API" \
     -H "api-key: $SRC_KEY" -H 'Content-Type: application/json' -d @docs-index.json | jq .name
   ```
3. Create the indexer. It runs as soon as it's created, then every 2 hours:

   ```bash
   cat > docs-indexer.json <<'EOF'
   {
     "name": "docs-blob-indexer",
     "dataSourceName": "docs-blob-ds",
     "targetIndexName": "docs-blob",
     "parameters": {"configuration": {"parsingMode": "json"}},
     "fieldMappings": [
       {"sourceFieldName": "metadata_storage_path", "targetFieldName": "id", "mappingFunction": {"name": "base64Encode"}}
     ],
     "schedule": {"interval": "PT2H"}
   }
   EOF
   curl -s -X PUT "$SRC/indexers/docs-blob-indexer?api-version=$API" \
     -H "api-key: $SRC_KEY" -H 'Content-Type: application/json' -d @docs-indexer.json | jq .name
   ```
4. After about 30 seconds, check that the run succeeded:

   ```bash
   curl -s "$SRC/indexers/docs-blob-indexer/status?api-version=$API" -H "api-key: $SRC_KEY" \
     | jq '{status: .lastResult.status, processed: .lastResult.itemsProcessed, failed: .lastResult.itemsFailed}'
   ```

   You want `"status": "success"` and `"processed": 4`. If it shows `inProgress`, wait and rerun.

## Step 6 — Verify and hand off to the migration runbook

The test source is ready when both indexes return documents and these queries give the results shown below. Save the results: you'll compare the migrated copy against them.

1. Document counts (expect `hotels-push: 5`, `docs-blob: 4`):

   ```bash
   for i in hotels-push docs-blob; do
     echo "$i: $(curl -s "$SRC/indexes/$i/docs/\$count?api-version=$API" -H "api-key: $SRC_KEY")"
   done
   ```
2. Run the test queries:

   ```bash
   q() { curl -s -X POST "$SRC/indexes/$1/docs/search?api-version=$API" -H "api-key: $SRC_KEY" \
           -H 'Content-Type: application/json' -d "$2" | jq -c "$3"; }
   
   # Synonym: "motel" also matches "inn" via hotel-synonyms
   q hotels-push '{"search":"inn","select":"hotelName"}' '[.value[].hotelName]'
   
   # Semantic ranker
   q hotels-push '{"search":"quiet place to relax","queryType":"semantic","semanticConfiguration":"default","select":"hotelName","top":3}' '[.value[].hotelName]'
   
   # Non-retrievable field: searchable, but never returned
   q hotels-push '{"search":"monsoon","select":"hotelName"}' '[.value[].hotelName]'
   
   # Indexer-filled index
   q docs-blob '{"search":"refund","select":"title,category"}' '[.value[].title]'
   ```

   | Query | Expected result on the source |
   | --- | --- |
   | Synonym `inn` | Brahmaputra View Inn **and** Highway Motel |
   | Semantic | Hill Top Resort ranked first (exact order may vary) |
   | `monsoon` | Tea Garden Retreat, found through the hidden `internalNote` |
   | `refund` | Refund policy |
3. Now open the migration runbook and start at its Step 1. `source ~/search-test.env` already sets every variable it needs.

What to look for when you've migrated it: after Option B, the `monsoon` query on `hotels-push` returns **nothing**, because the backup tool couldn't read `internalNote`. That's the expected data loss the runbook warns about. After Option A, `docs-blob` returns all four documents.

## Clean up

Delete both resource groups when you finish testing. This removes the search services and the storage account, and Basic-tier billing stops.

```bash
source ~/search-test.env
az group delete -n $SRC_RG --yes --no-wait
az group delete -n $DST_RG --yes --no-wait
rm -rf ~/sample-docs ~/synonyms.json ~/hotels-index.json ~/hotels-docs.json ~/docs-index.json ~/docs-indexer.json
```

A few minutes later, confirm the groups are gone with `az group list -o table`.

If you open a new shell partway through, run `source ~/search-test.env` and then reload the keys:

```bash
SRC_KEY=$(az search admin-key show -g $SRC_RG --service-name $SRC_SVC --query primaryKey -o tsv)
STG_CONN=$(az storage account show-connection-string -n $STG -g $SRC_RG --query connectionString -o tsv)
```

You'll need `STG_CONN` again in runbook Step 5a, when you put the data source's connection string back.
