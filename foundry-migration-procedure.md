# Microsoft Foundry Migration Procedure (Resource Group → Resource Group)

**Purpose:** Step-by-step procedure to migrate a Microsoft Foundry environment (AI resource, project, model deployments, connections, agents, files, vector stores, RBAC) from a source resource group to a target resource group.

**Precondition (agreed with customer):** The real target is a different subscription. Because of test-environment constraints, verification is done RG → RG. To make the result valid for the subscription migration, this procedure uses a **rebuild (recreate) approach**, not Azure "Move resource". Moving a Foundry resource across subscriptions is not supported, so an RG move would not prove anything for the real migration.

**Shell:** Git Bash (MINGW64) on Windows with Azure CLI. Always run first:

```bash
export MSYS_NO_PATHCONV=1
```

Without it, Git Bash converts `/subscriptions/...` into a Windows path and Azure returns `MissingSubscription`. If an ID still fails, strip hidden carriage returns with `| tr -d '\r'`.

---

## What is migrated and how (summary)

| Item | Can it be moved as-is? | Method |
|---|---|---|
| Foundry resource (AI resource) | No (cross-subscription) | Recreate |
| Model deployments | No | Recreate with same name/version/capacity |
| Projects | No | Recreate with same name |
| Connections | No (secrets are not exportable) | Recreate + re-enter key or use Entra ID |
| Agents | No | Recreate from exported JSON definition |
| Files | No | Re-upload |
| Vector stores | No | Recreate and re-index |
| Conversation history (threads) | No | Not migrated (confirm acceptable) |
| RBAC role assignments | No | Reassign; new managed identities get new IDs |
| Endpoint URL / keys | — | Change; apps must be updated |

---

## Demo scenario

| Component | Source | Target |
|---|---|---|
| Resource group | `at-agent-research` | `<target-rg>` |
| Foundry resource | `foundry-demo-source-dig-resource` | `foundry-demo-target-dig-resource` |
| Project | `proj-demo` | `proj-demo` (same) |
| Model deployment | `gpt-41-mini` | `gpt-41-mini` (same) |
| Storage account | `<src-storage>` | `stdemotgt<initials>` |
| Connection | `conn-blob-demo` (account key) | `conn-blob-demo` (same) |
| Agent | `demo-faq-agent` (File Search) | `demo-faq-agent` (same) |
| File | `demo-faq.txt` | re-uploaded |
| Vector store | auto-created | recreated |
| RBAC | Foundry User on project | reassigned |

Keep project, deployment, connection and agent names **identical**. Then apps only need the endpoint changed. The Foundry resource and storage account names must differ because they are globally unique.

---

## Common variables

Set these in every new terminal:

```bash
export MSYS_NO_PATHCONV=1
SUB=$(az account show --query id -o tsv | tr -d '\r')
API=2025-06-01

SRC_RG=at-agent-research
SRC_ACCT=foundry-demo-source-dig-resource
SRC_ST=<src-storage>

TGT_RG=<target-rg>
TGT_ACCT=foundry-demo-target-dig-resource
TGT_ST=stdemotgt<initials>

PROJ=proj-demo
DEPLOY=gpt-41-mini
CONN=conn-blob-demo
AGENT=demo-faq-agent

LOC=$(az cognitiveservices account show -g $SRC_RG -n $SRC_ACCT --query location -o tsv | tr -d '\r')

SRC_ARM=https://management.azure.com/subscriptions/$SUB/resourceGroups/$SRC_RG/providers/Microsoft.CognitiveServices/accounts/$SRC_ACCT
TGT_ARM=https://management.azure.com/subscriptions/$SUB/resourceGroups/$TGT_RG/providers/Microsoft.CognitiveServices/accounts/$TGT_ACCT

SRC_DP=https://$SRC_ACCT.services.ai.azure.com/api/projects/$PROJ
TGT_DP=https://$TGT_ACCT.services.ai.azure.com/api/projects/$PROJ
```

---

# Phase 0 – Build demo source and take inventory

## Part A – Build the demo in the source RG

### A1. Foundry resource and project
1. Azure portal → search **Foundry** → **Create**.
2. Resource group: `at-agent-research`; name: `foundry-demo-source-dig-resource`; region: one where gpt-4.1-mini is available.
3. Default project name: `proj-demo`.
4. Networking: public (keep the demo simple).
5. Identity: system-assigned managed identity **on**.
6. Open the resource in the Foundry portal (ai.azure.com). Use the **new** Foundry experience, not classic.

### A2. Model deployment
1. Project → **Models** → Deploy → gpt-4.1-mini.
2. Deployment name `gpt-41-mini`, type Global Standard, small capacity (e.g., 10K TPM).
3. Test once in the playground.

### A3. Connection with a secret
1. Create storage account `<src-storage>` (Standard LRS) in the source RG.
2. Foundry project → **Connected resources** → **New connection** → Azure Blob Storage → select the storage account → auth **account key** → name `conn-blob-demo`.

### A4. Data and agent
1. Create `demo-faq.txt`:
   ```
   Contoso Demo FAQ
   Q: What is the office code word for the year 2026?
   A: The code word is BLUE-FALCON-42.
   Q: Who approves migration tasks?
   A: Migration tasks are approved by the Platform Team lead.
   ```
2. Foundry → **Agents** → New agent: name `demo-faq-agent`, model `gpt-41-mini`, instructions:
   *"Answer only from the uploaded files. If the answer is not in the files, say you don't know."*
3. Add tool **File search** → upload `demo-faq.txt` (a vector store is created automatically).
4. Playground test. Save a screenshot as the **baseline**.

### A5. RBAC
Project → **Access control (IAM)** → add a user with **Foundry User** (may display as *Azure AI User*).

## Part B – Inventory

```bash
mkdir -p ~/inventory-src && cd ~/inventory-src
```

### B1. Resources in the RG
```bash
az resource list -g $SRC_RG -o table
az resource list -g $SRC_RG > 01-resources.json
```

### B2. Foundry resource settings
```bash
az cognitiveservices account show -g $SRC_RG -n $SRC_ACCT > 02-account.json
```
Record: `location`, `sku`, `kind`, `customSubDomainName`, `publicNetworkAccess`, `disableLocalAuth`, `identity`, `properties.endpoints`.

### B3. Model deployments
```bash
az cognitiveservices account deployment list -g $SRC_RG -n $SRC_ACCT > 03-deployments.json
az cognitiveservices account deployment list -g $SRC_RG -n $SRC_ACCT \
  --query "[].{name:name, model:properties.model.name, ver:properties.model.version, sku:sku.name, cap:sku.capacity, filter:properties.raiPolicyName}" -o table
```

### B4. Projects
```bash
az rest --method get --url "$SRC_ARM/projects?api-version=$API" > 04-projects.json
```

### B5. Connections
```bash
az rest --method get --url "$SRC_ARM/connections?api-version=$API" > 05a-account-connections.json
az rest --method get --url "$SRC_ARM/projects/$PROJ/connections?api-version=$API" > 05b-project-connections.json
```
**Expected:** `credentials` is `null`. Azure never returns secrets on read. This proves connections cannot be exported/imported and the key must be re-entered. Check `authType` (`AccountKey` = key-based; `AAD` = Entra ID, no secret but needs RBAC). An empty account-level list is normal.

### B6. Role assignments
```bash
SRC_ACCT_ID=$(az cognitiveservices account show -g $SRC_RG -n $SRC_ACCT --query id -o tsv | tr -d '\r')
az role assignment list --scope "$SRC_ACCT_ID" --include-inherited -o table > 06a-rbac-account.txt
az role assignment list --scope "$SRC_ACCT_ID/projects/$PROJ" --include-inherited -o table > 06b-rbac-project.txt
```
Without `--include-inherited`, a blank result is normal (no roles assigned directly). Rows scoped to the RG/subscription are inherited; rows scoped to the account/project must be recreated.

### B7. Agents
```bash
az rest --method get --url "$SRC_DP/agents?api-version=v1" --resource https://ai.azure.com > 07a-agents.json
az rest --method get --url "$SRC_DP/agents/$AGENT?api-version=v1" --resource https://ai.azure.com > 07b-agent-$AGENT.json
```
`--resource https://ai.azure.com` is required for data-plane calls.

### B8. Files and vector stores
```bash
az rest --method get --url "$SRC_DP/openai/v1/vector_stores" --resource https://ai.azure.com > 08a-vector-stores.json
az rest --method get --url "$SRC_DP/openai/v1/files" --resource https://ai.azure.com > 08b-files.json
```
If 404, try `$SRC_DP/vector_stores?api-version=v1` and `$SRC_DP/files?api-version=v1`. Keep the original `demo-faq.txt`; files are re-uploaded, not downloaded from Foundry.

### B9. Inventory sheet
Columns: **Item | Name | Key settings | Source ID | Method | Target ID (fill later) | Status**

### Phase 0 checks
- [ ] All inventory files saved and not empty (blank RBAC without `--include-inherited` is fine)
- [ ] Connection JSON confirmed: `credentials = null`
- [ ] Agent JSON shows model, instructions, `file_search` tool, vector store ID
- [ ] Baseline screenshot saved
- [ ] 3 test questions fixed:
  1. "What is the office code word for 2026?" → `BLUE-FALCON-42`
  2. "Who approves migration tasks?" → Platform Team lead
  3. "What is the CEO's name?" → "I don't know"

---

# Phase 1 – Prepare the target

### 1.1 Confirm region
```bash
echo $LOC
az group show -n $TGT_RG --query "{name:name, location:location}" -o table
```
Create all target resources in `$LOC` (same as source).

### 1.2 Check your permissions
```bash
TGT_RG_ID=$(az group show -n $TGT_RG --query id -o tsv | tr -d '\r')
ME=$(az ad signed-in-user show --query id -o tsv | tr -d '\r')
az role assignment list --scope "$TGT_RG_ID" --include-inherited --assignee $ME -o table
```
Need **Owner**, or **Contributor + User Access Administrator** (role assignments in Phase 5).

### 1.3 Resource providers
```bash
az provider show -n Microsoft.CognitiveServices --query registrationState -o tsv
az provider show -n Microsoft.Storage --query registrationState -o tsv
```
Must be `Registered`. In a new subscription, register with `az provider register -n <name>`.

### 1.4 Model quota
```bash
az cognitiveservices usage list -l $LOC -o table | grep -i "gpt-4.1-mini"
```
Source and target share the subscription's regional quota. **Limit − CurrentValue** must be ≥ the source deployment capacity. Otherwise deploy the target with lower capacity for the test.

### 1.5 Policies
```bash
az policy assignment list --scope "$TGT_RG_ID" --disable-scope-strict-match -o table
```
Look for: allowed locations, disable local auth/keys, required private endpoints, required tags.

### 1.6 Target storage account
Record source settings:
```bash
az storage account show -g $SRC_RG -n $SRC_ST \
  --query "{sku:sku.name, kind:kind, tls:minimumTlsVersion, publicAccess:allowBlobPublicAccess, sharedKey:allowSharedKeyAccess}" -o table
```
Create target with the same settings:
```bash
az storage account create -g $TGT_RG -n $TGT_ST -l $LOC \
  --sku Standard_LRS --kind StorageV2 --min-tls-version TLS1_2 \
  --allow-blob-public-access false
```
If the source storage contains blobs or the connection points to a specific container, create the same container and copy data with AzCopy:
```bash
az storage container list --account-name $SRC_ST --auth-mode login -o table
az storage container create --account-name $TGT_ST -n <container> --auth-mode login
# azcopy copy "https://$SRC_ST.blob.core.windows.net/<container>?<SAS>" "https://$TGT_ST.blob.core.windows.net/<container>?<SAS>" --recursive
```

### Phase 1 checks
- [ ] Target RG confirmed; region = source region
- [ ] Owner (or Contributor + UAA) on target RG
- [ ] Providers registered
- [ ] Enough quota
- [ ] No blocking policy (or noted)
- [ ] Target storage created; containers/data copied if needed
- [ ] Naming map filled in

---

# Phase 2 – Foundry resource and model deployment

### 2.1 Create the Foundry resource

**Option A – Portal (recommended for the demo):** same steps as A1 with target RG and name `foundry-demo-target-dig-resource`. Match the settings recorded in `02-account.json` (region, SKU, network access, managed identity). If the portal asks for a default project, name it `proj-demo` and skip 3.1.

**Option B – CLI:**
```bash
az cognitiveservices account create -g $TGT_RG -n $TGT_ACCT -l $LOC \
  --kind AIServices --sku S0 \
  --custom-domain $TGT_ACCT \
  --assign-identity \
  --allow-project-management \
  --yes
```
If `--allow-project-management` is not recognized, update Azure CLI (`az upgrade`) or use Option A.

Verify:
```bash
az cognitiveservices account show -g $TGT_RG -n $TGT_ACCT \
  --query "{state:properties.provisioningState, endpoint:properties.endpoint, identity:identity.type, network:properties.publicNetworkAccess}" -o table
```

### 2.2 Recreate the model deployment
Use the exact values from `03-deployments.json`:
```bash
az cognitiveservices account deployment create -g $TGT_RG -n $TGT_ACCT \
  --deployment-name $DEPLOY \
  --model-name gpt-4.1-mini \
  --model-version "<version from 03-deployments.json>" \
  --model-format OpenAI \
  --sku-name GlobalStandard \
  --sku-capacity <capacity from 03-deployments.json>
```
If the source uses a custom content filter (`raiPolicyName` not default), recreate the filter in the portal (Guardrails + controls) and assign it to the deployment.

### 2.3 Quick model test
Foundry portal → target project → playground → select `gpt-41-mini` → send a prompt.

### Phase 2 checks
- [ ] Provisioning state `Succeeded`
- [ ] Managed identity enabled
- [ ] Network setting matches source
- [ ] Deployment name, model, version, SKU, capacity identical to source
- [ ] Content filter matches source
- [ ] Playground responds

---

# Phase 3 – Project and connections

### 3.1 Create the project (skip if created in 2.1)
```bash
cat > project.json <<EOF
{
  "location": "$LOC",
  "identity": { "type": "SystemAssigned" },
  "properties": { "displayName": "$PROJ", "description": "Migrated from $SRC_RG" }
}
EOF
az rest --method put --url "$TGT_ARM/projects/$PROJ?api-version=$API" --body @project.json
```
Verify:
```bash
az rest --method get --url "$TGT_ARM/projects/$PROJ?api-version=$API" \
  --query "{name:name, state:properties.provisioningState, principalId:identity.principalId}"
```
Record the project managed identity `principalId` (needed in Phase 5). It is **different** from the source.

### 3.2 Recreate the connection

**Option A – Portal (recommended):** target project → Connected resources → New connection → Azure Blob Storage → select `$TGT_ST` → account key → name `conn-blob-demo`.

**Option B – REST (scriptable):**
1. Open `05b-project-connections.json`, copy the `properties` of `conn-blob-demo`.
2. Replace the source storage name with the target storage name in `target` and `metadata`.
3. Add the key:
```bash
KEY=$(az storage account keys list -g $TGT_RG -n $TGT_ST --query "[0].value" -o tsv | tr -d '\r')
cat > connection.json <<EOF
{
  "properties": {
    "category": "AzureBlob",
    "target": "https://$TGT_ST.blob.core.windows.net/",
    "authType": "AccountKey",
    "credentials": { "key": "$KEY" },
    "metadata": {
      "AccountName": "$TGT_ST",
      "ContainerName": "<container from 05b>"
    }
  }
}
EOF
az rest --method put --url "$TGT_ARM/projects/$PROJ/connections/$CONN?api-version=$API" --body @connection.json
rm connection.json   # contains the key
```
Use the exact `category`, `authType` and `metadata` keys found in `05b`; the above is the typical shape.

**Recommended improvement for production:** use `authType: AAD` (Entra ID) instead of a key, and grant the project managed identity *Storage Blob Data Contributor* on the storage account (Phase 5). Then there is no secret to migrate.

### 3.3 Verify
```bash
az rest --method get --url "$TGT_ARM/projects/$PROJ/connections?api-version=$API" \
  --query "value[].{name:name, category:properties.category, auth:properties.authType, target:properties.target}" -o table
```
In the portal, open the connection and confirm it is connected.

### Phase 3 checks
- [ ] Project name identical; provisioning `Succeeded`
- [ ] Project managed identity principalId recorded
- [ ] Connection name, category and authType identical; target points to the **target** storage
- [ ] Connection works (portal shows connected)

---

# Phase 4 – Agent service setup

### 4.1 Basic setup (the demo)
Nothing to configure. Agent data (files, vector stores, threads) is stored in Microsoft-managed storage. Confirm the **Agents** page opens in the target project.

### 4.2 Standard setup (if the real environment uses it)
Check the source: if the project has connections to Cosmos DB, AI Search and Storage used for agents, it is a standard setup. Then, in the target:
1. Create new Cosmos DB for NoSQL (≥ 3000 RU/s total), Storage, AI Search, Key Vault in the target.
2. Create project connections to Cosmos DB, Storage and AI Search.
3. Assign roles to the project managed identity (Cosmos DB Operator, Storage Account Contributor, Search Index Data Contributor, Search Service Contributor, Storage Blob Data Contributor/Owner on the agent containers, Cosmos DB Built-in Data Contributor on the `enterprise_memory` database).
4. Create the **account** capability host, then the **project** capability host referencing the three connections.

Capability hosts cannot be updated after creation, so create the connections and roles correctly first. Old Cosmos/Search/Storage data is not reused by the new project; agents and data are recreated in Phase 6.

### Phase 4 checks
- [ ] Setup type identified (basic / standard) and recorded
- [ ] Agents page opens in target project
- [ ] (Standard) Both capability hosts `Succeeded`

---

# Phase 5 – RBAC

### 5.1 User and group roles
From `06b-rbac-project.txt`, recreate every assignment scoped to the account or project:
```bash
TGT_ACCT_ID=$(az cognitiveservices account show -g $TGT_RG -n $TGT_ACCT --query id -o tsv | tr -d '\r')

az role assignment create \
  --assignee "<user UPN or object ID>" \
  --role "Azure AI User" \
  --scope "$TGT_ACCT_ID/projects/$PROJ"
```
Use the role name exactly as shown in the inventory (*Azure AI User* or *Foundry User*).

### 5.2 Managed identity roles
Only needed if connections use Entra ID or standard setup is used:
```bash
PROJ_MI=$(az rest --method get --url "$TGT_ARM/projects/$PROJ?api-version=$API" --query identity.principalId -o tsv | tr -d '\r')
TGT_ST_ID=$(az storage account show -g $TGT_RG -n $TGT_ST --query id -o tsv | tr -d '\r')

az role assignment create --assignee-object-id $PROJ_MI --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Contributor" --scope "$TGT_ST_ID"
```

### 5.3 Consumer app identities
If an app calls the source with Entra ID, grant its identity the same role on the target account.

### 5.4 Verify
```bash
az role assignment list --scope "$TGT_ACCT_ID/projects/$PROJ" -o table
```
Compare with `06b-rbac-project.txt` (direct assignments only).

### Phase 5 checks
- [ ] Every direct assignment from source exists on target
- [ ] Managed identity roles assigned (if needed)
- [ ] Assigned user can open the target project and see agents
- [ ] A user without a role is denied

---

# Phase 6 – Data and agents

Agents, files and vector stores cannot be migrated; they are recreated. **IDs will change.** Keep a mapping table:

| Item | Source ID | Target ID |
|---|---|---|
| File demo-faq.txt | file-xxxx | |
| Vector store | vs_xxxx | |
| Agent | demo-faq-agent (version) | |

### Option A – Portal (recommended for the demo)
1. Target project → Agents → New agent → name `demo-faq-agent`.
2. Model `gpt-41-mini`.
3. Copy **instructions** exactly from `07b-agent-demo-faq-agent.json`.
4. Add the same tools as in `07b` (File search) → upload `demo-faq.txt`.
5. Set any other parameters shown in `07b` (temperature, top_p, etc.).
6. Save.

### Option B – REST (scriptable, for many agents)
```bash
TOKEN=$(az account get-access-token --resource https://ai.azure.com --query accessToken -o tsv | tr -d '\r')
```
1. Upload the file:
```bash
curl -s -X POST "$TGT_DP/openai/v1/files" \
  -H "Authorization: Bearer $TOKEN" \
  -F purpose=assistants -F file=@demo-faq.txt
```
Record the returned `id` (new file ID).

2. Create the vector store with the file:
```bash
curl -s -X POST "$TGT_DP/openai/v1/vector_stores" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"demo-faq-vs","file_ids":["<new file id>"]}'
```
Record the new vector store `id`. Check that indexing completed:
```bash
curl -s "$TGT_DP/openai/v1/vector_stores/<new vs id>" -H "Authorization: Bearer $TOKEN"
```
`file_counts.completed` should equal the number of files.

3. Create the agent. Copy the `definition` block from `07b`, and replace the old vector store ID with the new one:
```bash
cat > agent.json <<'EOF'
{
  "name": "demo-faq-agent",
  "definition": {
    "kind": "prompt",
    "model": "gpt-41-mini",
    "instructions": "Answer only from the uploaded files. If the answer is not in the files, say you don't know.",
    "tools": [ { "type": "file_search", "vector_store_ids": ["<new vs id>"] } ]
  }
}
EOF
az rest --method post --url "$TGT_DP/agents?api-version=v1" --resource https://ai.azure.com --body @agent.json
```
If the path `/openai/v1/...` returns 404, use the same alternative path that worked in Phase 0 (B8). Always follow the structure shown in your `07b` export; it reflects the API version used by your project.

### 6.1 Conversation history
Threads/conversations are not migrated. Confirm with the customer that this is acceptable. If history must be kept, it needs a custom export script (out of scope for this procedure).

### 6.2 Verify
```bash
az rest --method get --url "$TGT_DP/agents/$AGENT?api-version=v1" --resource https://ai.azure.com > ~/inventory-src/../tgt-07b-agent.json
```
Compare with `07b`: model, instructions and tools must match; only IDs differ.

### Phase 6 checks
- [ ] File uploaded; vector store indexing completed
- [ ] Agent name, model, instructions, tools identical to source
- [ ] Agent references the **new** vector store ID
- [ ] Mapping table filled
- [ ] Decision on conversation history recorded

---

# Phase 7 – Switch consumers (cutover)

1. List every app/script using the source (from inventory).
2. Update for each:
   - Endpoint: `https://foundry-demo-source-dig-resource.services.ai.azure.com/...` → `https://foundry-demo-target-dig-resource.services.ai.azure.com/...`
   - Project endpoint: `$TGT_DP`
   - API key (if key auth): get new key with
     `az cognitiveservices account keys list -g $TGT_RG -n $TGT_ACCT`
   - Agent reference: unchanged if referenced by **name**; update if referenced by an ID.
   - File/vector store IDs: update if the app uses them directly (use the mapping table).
3. Keep the source running until all tests pass (rollback option).

### Phase 7 checks
- [ ] All consumers updated
- [ ] No consumer still calling the source (check source metrics: requests should drop to zero)

---

# Phase 8 – Tests

Record each result as **Pass / Fail** with evidence (screenshot or command output).

### 8.1 Infrastructure
| # | Test | Expected |
|---|---|---|
| I-1 | `az cognitiveservices account show` on target | `Succeeded` |
| I-2 | Project GET | `Succeeded` |
| I-3 | Deployment list on target vs `03-deployments.json` | Same name, model, version, SKU, capacity |
| I-4 | Connection list on target vs `05b` | Same name, category, authType; target = new storage |
| I-5 | Network setting vs source | Same |
| I-6 | (Standard setup) capability hosts | `Succeeded` |

Quick comparison of deployments:
```bash
az cognitiveservices account deployment list -g $TGT_RG -n $TGT_ACCT \
  --query "[].{name:name, model:properties.model.name, ver:properties.model.version, sku:sku.name, cap:sku.capacity}" -o table
```

### 8.2 Model
| # | Test | Expected |
|---|---|---|
| M-1 | Send the same prompt to source and target deployment | Both respond, same format |
| M-2 | Send a prompt the source content filter blocks | Target blocks too |

### 8.3 Agent and data
| # | Test | Expected |
|---|---|---|
| A-1 | "What is the office code word for 2026?" | `BLUE-FALCON-42` (proves file + vector store migrated) |
| A-2 | "Who approves migration tasks?" | Platform Team lead |
| A-3 | "What is the CEO's name?" | "I don't know" (proves instructions migrated) |
| A-4 | Compare target agent JSON with `07b` | Same model, instructions, tools |
| A-5 | Each additional tool (code interpreter, function, OpenAPI, MCP) | One successful call per tool |

### 8.4 Connection
| # | Test | Expected |
|---|---|---|
| C-1 | Open connection in portal | Connected |
| C-2 | Use the connection (e.g., browse/add data from blob in the project) | Works |
| C-3 | Confirm source key does not work for the target storage | Fails (proves new secret is in use) |

### 8.5 Security
| # | Test | Expected |
|---|---|---|
| S-1 | Assigned user opens target project | Allowed |
| S-2 | User without role opens target project | Denied |
| S-3 | Source API key used against target endpoint | `401` |
| S-4 | (If private networking) access from outside VNet | Denied |

S-3 example:
```bash
SRC_KEY=$(az cognitiveservices account keys list -g $SRC_RG -n $SRC_ACCT --query key1 -o tsv | tr -d '\r')
curl -s -o /dev/null -w "%{http_code}\n" \
  "https://$TGT_ACCT.services.ai.azure.com/openai/deployments/$DEPLOY/chat/completions?api-version=2024-10-21" \
  -H "api-key: $SRC_KEY" -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"hi"}]}'
```
Expected `401`. Run again with the target key → `200`.

### 8.6 Application
| # | Test | Expected |
|---|---|---|
| P-1 | End-to-end run of each consumer app against target | Works |
| P-2 | Logs/monitoring (diagnostic settings, App Insights) | Receiving data from target |

### 8.7 Rollback
| # | Test | Expected |
|---|---|---|
| R-1 | Point one app back to source | Works (source untouched) |

### 8.8 Re-run inventory on target
Run Phase 0 Part B commands against the target into `~/inventory-tgt` and compare file by file with `~/inventory-src`. Differences should only be names/IDs/endpoints listed in the mapping table.

---

# Phase 9 – Cleanup and report

### 9.1 Cleanup (only after sign-off)
1. Keep the source for an agreed period (e.g., 1–2 weeks).
2. Then delete source agents, deployments, project, Foundry resource, and storage.
3. Note: deleted Foundry resources are soft-deleted; purge if the name must be reused:
   ```bash
   az cognitiveservices account purge -g $SRC_RG -n $SRC_ACCT -l $LOC
   ```
   Deleting/purging is irreversible; confirm with the customer first.

### 9.2 Report contents
1. Precondition: RG → RG test using rebuild method; valid for subscription migration.
2. Migration matrix (the summary table at the top).
3. What changes: endpoint URL, API keys, managed identity IDs, file/vector store IDs.
4. What is lost: conversation history.
5. Answer to "can connections be migrated?": No. Secrets are never returned by the API (`credentials = null`); connections must be recreated and keys re-entered, or switched to Entra ID.
6. Test results table (Phase 8) with evidence.
7. Issues found and workarounds (e.g., Git Bash `MSYS_NO_PATHCONV`, quota sharing).

### 9.3 Extra checks for the real subscription migration
| Item | Why |
|---|---|
| Resource provider registration | New subscriptions are often not registered |
| Model quota in new subscription | Quota is per subscription per region |
| Azure Policy in new subscription | May block keys, regions, public access |
| Same Entra tenant? | Different tenant = all identities and RBAC change |
| Private networking / DNS | VNet, private endpoints and DNS zones must be recreated |
| Cost/billing | New subscription billing owner |
