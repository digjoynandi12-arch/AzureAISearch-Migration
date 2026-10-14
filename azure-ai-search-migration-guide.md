# Azure AI Search: Backup & Migration Guide

A step-by-step guide to copy an Azure AI Search service into a new resource group: rebuild the service, copy every object definition, and move the documents. Every step has the commands to run and the result to expect. The source service is not changed until the final step.

> Tested end to end on a sample service, including from Git Bash on Windows. Fixes for the errors found during testing are built into the scripts below.

---

## Contents

- [Overview](#overview)
- [Choose a data path per index](#choose-a-data-path-per-index)
- [Prerequisites](#prerequisites)
- [Setup: save settings once, load per terminal](#setup-save-settings-once-load-per-terminal)
- [Step 1 — Inventory the source service](#step-1--inventory-the-source-service)
- [Step 2 — Create the destination service](#step-2--create-the-destination-service)
- [Step 3 — Grant identity and network access](#step-3--grant-identity-and-network-access)
- [Step 4 — Export every object definition](#step-4--export-every-object-definition)
- [Step 5 — Restore secrets and import](#step-5--restore-secrets-and-import)
- [Step 6 — Move the documents](#step-6--move-the-documents)
- [Step 7 — Validate](#step-7--validate)
- [Step 8 — Cut over and retire the source](#step-8--cut-over-and-retire-the-source)
- [Errors found in testing](#errors-found-in-testing)
- [Troubleshooting and rollback](#troubleshooting-and-rollback)
- [Recap](#recap)

---

## Overview

Azure AI Search has no self-service backup or restore, so a copy is rebuilt in three layers:

| Layer | What it includes | How it moves |
| --- | --- | --- |
| **Service** | Tier, replicas, partitions, region, semantic ranker, managed identity, network rules | Azure CLI · Steps 2–3 |
| **Definitions** | Indexes, synonym maps, data sources, skillsets, indexers | REST export and import as JSON · Steps 4–5 |
| **Documents** | The indexed content | Re-run indexers, or dump and reload · Step 6 |

If you only want to **move** the existing service (same name and endpoint, no copy), use the portal: *Search service → Move → Move to another resource group*.

## Choose a data path per index

| | Option A · Re-run indexers | Option B · Dump and reload |
| --- | --- | --- |
| Use when | An indexer fills the index | Your app pushes documents |
| Copies | Everything, rebuilt from the source data | Only fields marked `retrievable` |
| Cost | Skillsets and embeddings run again | Bandwidth and your time |
| Time | Hours to days for large sets | Minutes to hours |
| Watch out | A full re-index; change tracking does not carry over | Non-retrievable fields (often vectors) arrive empty |

Most services mix both: use **A** for indexer-fed indexes and **B** for push indexes. Decide per index during Step 1 — if an indexer targets the index, it's Option A.

## Prerequisites

- [ ] Azure CLI 2.50+ signed in (`az login`), plus `curl` and `jq`, in a bash terminal
- [ ] Contributor on the source and destination resource groups
- [ ] Admin keys, or the Search Service Contributor role, on both services
- [ ] Connection strings and AI service keys for every data source and skillset — **exports blank them**
- [ ] .NET 8 SDK if you use Microsoft's backup tool for Option B
- [ ] A write-freeze window if apps push documents during the copy

> **On Windows?** Git Bash works, but its `jq` adds a hidden `\r` to every output line, which breaks object names in URLs. The setup below includes a one-line fix.

## Setup: save settings once, load per terminal

**1. Save your values** (edit the names first; the destination service name must be globally unique):

```bash
cat > ~/search-migration.env <<'EOF'
SRC_RG=old-rg
SRC_SVC=old-search
DST_RG=new-rg
DST_SVC=new-search
LOCATION=centralindia
API=2024-07-01
SRC=https://old-search.search.windows.net
DST=https://new-search.search.windows.net
EOF
```

**2. Run this at the start of every terminal session:**

```bash
set -a; source ~/search-migration.env; set +a
export SRC_KEY=$(az search admin-key show -g $SRC_RG --service-name $SRC_SVC --query primaryKey -o tsv)

# Git Bash on Windows only: strip the \r that jq adds
jq() { command jq "$@" | tr -d '\r'; }; export -f jq

# After Step 2 has created the destination service:
# export DST_KEY=$(az search admin-key show -g $DST_RG --service-name $DST_SVC --query primaryKey -o tsv)
```

`set -a` **exports** every variable. Without it, scripts started with `./` can't see them and stop with `unbound variable`.

---

## Step 1 — Inventory the source service

Record everything you'll compare against later.

```bash
mkdir -p ~/backup && cd ~/backup
az search service show -g $SRC_RG -n $SRC_SVC > service.json

# list every object
names() { curl -s "$1/$2?api-version=$API&\$select=name" -H "api-key: $3" | jq -r '.value[].name'; }
for t in indexes synonymmaps datasources skillsets indexers; do echo "== $t"; names $SRC $t $SRC_KEY; done

# baseline document counts, used again in Step 7
for i in $(names $SRC indexes $SRC_KEY); do
  echo "$i: $(curl -s "$SRC/indexes/$i/docs/\$count?api-version=$API" -H "api-key: $SRC_KEY")"
done | tee doc-counts.txt
```

- `service.json` holds the tier, replicas, partitions and semantic setting to copy in Step 2.
- Also note every app that calls this service; you repoint them in Step 8.
- Work from `~/backup` for the rest of the guide.

## Step 2 — Create the destination service

Create the resource group first if it doesn't exist (`az group create -n $DST_RG -l $LOCATION`).

```bash
az search service create -g $DST_RG -n $DST_SVC -l $LOCATION \
  --sku standard --replica-count 1 --partition-count 1 \
  --semantic-search free --identity-type SystemAssigned \
  --auth-options aadOrApiKey -o table

export DST_KEY=$(az search admin-key show -g $DST_RG --service-name $DST_SVC --query primaryKey -o tsv)
```

- **Match the source:** copy tier (`--sku`), replicas, partitions and `--semantic-search` from `service.json`. The tier must hold the source's storage and object counts.
- **New name, new endpoint:** clients need the new URL and key in Step 8.
- Provisioning takes about 3–10 minutes.

*Alternative:* portal → source service → Automation → **Export template**, change the service name, deploy into the new resource group. This also captures network settings.

## Step 3 — Grant identity and network access

The new service has a new managed identity with no roles. Do this before Step 5, or indexers fail on their first run. Skip the role part if your data sources use connection strings.

```bash
DST_MI=$(az search service show -g $DST_RG -n $DST_SVC --query identity.principalId -o tsv)
az role assignment create --assignee-object-id $DST_MI --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Reader" --scope <storage-account-resource-id>
```

| Resource | Role for the new identity |
| --- | --- |
| Blob / ADLS storage | Storage Blob Data Reader |
| Knowledge store, enrichment cache | Storage Blob Data Contributor |
| Azure OpenAI | Cognitive Services OpenAI User |
| Azure AI services | Cognitive Services User |
| Azure SQL | Contained database user with `db_datareader` |

To see what the old identity has: `az role assignment list --assignee <old-principal-id> --all -o table`. Role assignments can take ~10 minutes to apply.

**Also recreate:** IP firewall rules, private endpoints and DNS, shared private links (approve them again), app roles for Entra ID clients, customer-managed keys.

## Step 4 — Export every object definition

Save as `export-definitions.sh` in `~/backup`:

```bash
#!/usr/bin/env bash
# export-definitions.sh — run from the backup folder
set -uo pipefail
for t in synonymmaps indexes datasources skillsets indexers; do
  mkdir -p "$t"
  code=$(curl -s -o /tmp/list.json -w '%{http_code}' -H "api-key: $SRC_KEY" \
    "$SRC/$t?api-version=$API&\$select=name")
  [[ $code == 200 ]] || { echo "SKIP $t ($code)"; continue; }
  for n in $(jq -r '.value[].name' /tmp/list.json | tr -d '\r'); do
    code=$(curl -s -o /tmp/obj.json -w '%{http_code}' -H "api-key: $SRC_KEY" \
      "$SRC/$t/$n?api-version=$API")
    if [[ $code == 200 ]]; then
      jq 'del(."@odata.context", ."@odata.etag")' /tmp/obj.json > "$t/$n.json" && echo "exported $t/$n"
    else echo "FAIL $t/$n ($code)"; cat /tmp/obj.json; echo; fi
  done
done
```

Run it and check:

```bash
chmod +x export-definitions.sh && ./export-definitions.sh
find . -name '*.json' | sort
```

**Expect** one `exported` line per object. A `FAIL` prints the service's error underneath. Keep the `backup` folder — these JSON files are your configuration backup.

## Step 5 — Restore secrets and import

### 5a. Put the secrets back

Exports return every secret as `null`. Fill each one before importing.

| Exported file | Field returned as `null` | Set it to |
| --- | --- | --- |
| `datasources/*.json` | `credentials.connectionString` | Connection string, or `ResourceId=/subscriptions/.../storageAccounts/<acct>;` for managed identity |
| `skillsets/*.json` | `cognitiveServices.key`, `apiKey` | AI services or Azure OpenAI key (or remove to use the identity) |
| `indexes/*.json` | vectorizer `apiKey` | Azure OpenAI key |

Example for one data source (set `CONN` to the real connection string first; repeat per data source):

```bash
jq --arg cs "$CONN" '.credentials.connectionString = $cs' datasources/my-ds.json > tmp \
  && mv tmp datasources/my-ds.json

# should print "no blank secrets"
grep -rl '"connectionString": null\|"key": null\|"apiKey": null' . || echo "no blank secrets"
```

### 5b. Import in dependency order

Order: **synonym maps → indexes → data sources → skillsets → indexers**. Save as `import-definitions.sh`:

```bash
#!/usr/bin/env bash
# import-definitions.sh — run from the backup folder
set -uo pipefail
for t in synonymmaps indexes datasources skillsets indexers; do
  for f in "$t"/*.json; do
    [ -e "$f" ] || continue
    n=$(basename "$f" .json); body=$(cat "$f")
    [ "$t" = indexers ] && body=$(echo "$body" | jq '.disabled = true')
    code=$(curl -s -o /tmp/resp.json -w '%{http_code}' -X PUT "$DST/$t/$n?api-version=$API" \
      -H "api-key: $DST_KEY" -H 'Content-Type: application/json' -d "$body")
    if [[ $code == 20* ]]; then echo "OK   $t/$n"
    else echo "FAIL $t/$n ($code)"; cat /tmp/resp.json; echo; fi
  done
done
```

```bash
chmod +x import-definitions.sh && ./import-definitions.sh
```

**Expect** one `OK` line per object. Indexers are created **disabled**, so nothing runs until Step 6. A `FAIL` prints the error — fix that file and rerun; a PUT simply overwrites.

## Step 6 — Move the documents

### Option A — Re-run indexers (indexer-fed indexes)

```bash
for f in indexers/*.json; do n=$(basename "$f" .json)
  jq '.disabled = false' "$f" | curl -s -X PUT "$DST/indexers/$n?api-version=$API" \
    -H "api-key: $DST_KEY" -H 'Content-Type: application/json' -d @- > /dev/null
  curl -s -X POST "$DST/indexers/$n/run?api-version=$API" -H "api-key: $DST_KEY" -H 'Content-Length: 0'
  echo "started $n"
done
```

Check the document count, then the **whole** run history:

```bash
curl -s "$DST/indexes/<index>/docs/\$count?api-version=$API" -H "api-key: $DST_KEY"; echo
curl -s "$DST/indexers/<indexer>/status?api-version=$API" -H "api-key: $DST_KEY" \
  | jq '[.executionHistory[] | {status, processed: .itemsProcessed}]'
```

> `status: success` with `processed: 0` is fine when an earlier run already indexed everything. Enabling an indexer can start a run on its own, so you may see two runs back to back. Trust the count and the history, not only the last run.

A single run is time-limited; very large indexers continue on their next scheduled run, so make sure each indexer has a schedule.

### Option B — Dump and reload (push indexes)

**Small indexes (up to 1,000 documents)** — dump from the source, then upload:

```bash
curl -s -X POST "$SRC/indexes/<index>/docs/search?api-version=$API" -H "api-key: $SRC_KEY" \
  -H 'Content-Type: application/json' -d '{"search":"*","top":1000}' \
  | jq '{value: [.value[] | del(."@search.score") | . + {"@search.action":"upload"}]}' > docs.json

jq '.value | length' docs.json      # check the count before uploading

curl -s -X POST "$DST/indexes/<index>/docs/index?api-version=$API" -H "api-key: $DST_KEY" \
  -H 'Content-Type: application/json' -d @docs.json | jq -c '[.value[].status]'
```

The upload should print a list of `true`.

> Paste multi-line commands as they are. If you join them onto one line, **remove the backslashes** — otherwise bash drops the `-H` headers and `docs.json` comes out empty.

**Larger indexes** — use Microsoft's backup tool:

1. `git clone https://github.com/Azure-Samples/azure-search-dotnet-utilities.git`
2. In `index-backup-restore` (v11), fill `appsettings.json`: source and target service names, admin keys, index names, `BackupDirectory`. Don't commit this file — it holds both keys.
3. `dotnet run`, once per index. If the target index already exists (created empty in Step 5), delete it and run again.

**What Option B can't copy:**

- Fields with `retrievable: false` (often vectors) come back empty — use Option A for those indexes.
- Search can't page past 100,000 results with `skip`; export very large indexes in slices using a filter on a sortable field.

## Step 7 — Validate

**Finish Step 6 for every index first.** A push index shows 0 until Option B has run for it — that's a skipped step, not a failure.

Compare counts with the Step 1 baseline (expect `counts match`):

```bash
for i in $(curl -s "$DST/indexes?api-version=$API&\$select=name" -H "api-key: $DST_KEY" | jq -r '.value[].name'); do
  echo "$i: $(curl -s "$DST/indexes/$i/docs/\$count?api-version=$API" -H "api-key: $DST_KEY")"
done | diff doc-counts.txt - && echo "counts match"
```

Run the same query against both services and compare:

```bash
Q='{"search":"<a typical query>","top":5}'
for s in "$SRC $SRC_KEY" "$DST $DST_KEY"; do set -- $s
  curl -s -X POST "$1/indexes/<index>/docs/search?api-version=$API" -H "api-key: $2" \
    -H 'Content-Type: application/json' -d "$Q" | jq -c '[.value[] | ."@search.score"]'
done
```

Checklist:

- [ ] Object lists match Step 1
- [ ] Document counts match
- [ ] Same top results for sample queries
- [ ] Semantic, vector and synonym queries work
- [ ] Indexers succeed and keep their schedules
- [ ] Clients can query with the new key or role

Counts can lag a few seconds after indexing; recheck before treating a difference as a failure.

## Step 8 — Cut over and retire the source

1. **Freeze writes** — pause pushes to the source, or write to both, so no update is lost.
2. **Repoint clients** — new endpoint and key (or Entra role) in app settings, Key Vault, Azure OpenAI / AI Foundry connections, Logic Apps.
3. **Stop the source** — disable its indexers so enrichment isn't paid for twice:

   ```bash
   for f in indexers/*.json; do n=$(basename "$f" .json)
     jq '.disabled = true' "$f" | curl -s -X PUT "$SRC/indexers/$n?api-version=$API" \
       -H "api-key: $SRC_KEY" -H 'Content-Type: application/json' -d @- > /dev/null; done
   ```

4. **Watch, back up, delete** — monitor latency and throttling for a few days, take a final backup (rerun Step 4 and Option B), then delete:

   ```bash
   # irreversible: a deleted service or index can't be recovered
   az search service delete -g $SRC_RG -n $SRC_SVC --yes
   ```

---

## Errors found in testing

| What you see | Why | Fix |
| --- | --- | --- |
| `SRC: unbound variable` | Variables are set but not exported; `./script` runs in a child shell | `set -a; source ~/search-migration.env; set +a` |
| `HTTP 000`, or a file named `docs-blob$'\r'.json` | Git Bash `jq` ends lines with `\r\n` | `jq() { command jq "$@" \| tr -d '\r'; }; export -f jq` |
| Empty output from a pasted one-liner | A backslash before a space drops the `-H` header | Remove the `\` when joining lines into one |
| Indexer `success`, `processed: 0` | An earlier run already did the work | Check `docs/$count` and `executionHistory` |
| Script stops partway with no message | Old scripts used `curl -sf` with `set -e` | Use the scripts in this guide; they print each error and continue |

## Troubleshooting and rollback

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `403` on a REST call | Wrong key, or local auth disabled | Reload the key, or use `az account get-access-token --resource https://search.azure.com` and send `Authorization: Bearer <token>` |
| Index PUT: quota error | Destination tier too small | Raise tier or partitions, rerun import |
| Index PUT: semantic config error | Semantic ranker off | `az search service update -g $DST_RG -n $DST_SVC --semantic-search free` |
| Data source credentials error | Secret still `null`, or role missing | Redo Step 5a or Step 3; roles take ~10 min |
| Skill fails in indexer | AI key or OpenAI role missing | Fill `apiKey`, or grant Cognitive Services OpenAI User |
| Restored vectors empty | Field not retrievable | Use Option A for that index |
| Count lower than source | Indexer still running, or the 1,000 / 100,000 dump limit | Wait and recheck; use the backup tool or filtered slices |

**Rollback**, any time before the final delete: point clients back to the source and delete only the new service (`az search service delete -g $DST_RG -n $DST_SVC --yes`). Never delete a shared resource group.

---

## Recap

1. **Inventory** — settings, objects, counts
2. **Create service** — same tier and features
3. **Access** — identity roles, network
4. **Export** — definitions to JSON
5. **Import** — secrets back, in order
6. **Move data** — indexers (A) or dump and reload (B)
7. **Validate** — counts and queries match
8. **Cut over** — repoint, then retire

Keep the `backup` folder: the exported definitions plus dumped documents are your point-in-time backup. Schedule Step 4 and Option B against the new service if you want a standing backup.

### References

- [Azure AI Search FAQ: move, backup and restore](https://learn.microsoft.com/azure/search/search-faq-frequently-asked-questions)
- [Back up and restore an Azure AI Search index (sample)](https://learn.microsoft.com/en-us/samples/azure-samples/azure-search-dotnet-utilities/azure-search-backup-restore-index)
- [Microsoft Q&A: backing up Azure AI Search data](https://learn.microsoft.com/en-za/answers/questions/1792032/how-do-i-backup-data-of-azure-ai-search-service)
- [Microsoft Q&A: re-indexing after migration](https://learn.microsoft.com/en-us/answers/questions/1616243/avoid-re-indexing-all-documents-after-migration-of)
