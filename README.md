#!/usr/bin/env bash
# export-definitions.sh — run from the backup folder
set -euo pipefail

# Use exported variables if present, otherwise load the saved file
if [ -z "${SRC:-}" ] && [ -f ~/search-test.env ]; then source ~/search-test.env; fi
: "${SRC:?SRC is not set - run: set -a; source ~/search-test.env; set +a}"
SRC_KEY=${SRC_KEY:-$(az search admin-key show -g "$SRC_RG" --service-name "$SRC_SVC" --query primaryKey -o tsv)}

for t in synonymmaps indexes aliases datasources skillsets indexers; do
  mkdir -p "$t"
  names=$(curl -sf "$SRC/$t?api-version=$API&\$select=name" -H "api-key: $SRC_KEY" | jq -r '.value[].name' || true)
  for n in $names; do
    curl -sf "$SRC/$t/$n?api-version=$API" -H "api-key: $SRC_KEY" \
      | jq 'del(."@odata.context", ."@odata.etag")' > "$t/$n.json"
    echo "exported $t/$n"
  done
done
