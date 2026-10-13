#!/usr/bin/env bash
# export-definitions.sh — run from the backup folder
# Reports each failure and carries on, instead of stopping silently
set -uo pipefail

# Use exported variables if present, otherwise load the saved file
if [ -z "${SRC:-}" ] && [ -f ~/search-test.env ]; then source ~/search-test.env; fi
: "${SRC:?SRC is not set - run: set -a; source ~/search-test.env; set +a}"
SRC_KEY=${SRC_KEY:-$(az search admin-key show -g "$SRC_RG" --service-name "$SRC_SVC" --query primaryKey -o tsv)}

for t in synonymmaps indexes aliases datasources skillsets indexers; do
  mkdir -p "$t"
  code=$(curl -s -o /tmp/list.json -w '%{http_code}' "$SRC/$t?api-version=$API&\$select=name" -H "api-key: $SRC_KEY")
  if [[ $code != 200 ]]; then echo "SKIP $t (HTTP $code)"; continue; fi
  for n in $(jq -r '.value[].name' /tmp/list.json); do
    code=$(curl -s -o /tmp/obj.json -w '%{http_code}' "$SRC/$t/$n?api-version=$API" -H "api-key: $SRC_KEY")
    if [[ $code == 200 ]]; then
      jq 'del(."@odata.context", ."@odata.etag")' /tmp/obj.json > "$t/$n.json" && echo "exported $t/$n"
    else
      echo "FAIL $t/$n (HTTP $code)"; cat /tmp/obj.json; echo
    fi
  done
done


-------------------

#!/usr/bin/env bash
# import-definitions.sh — run from the backup folder
set -euo pipefail

# Use exported variables if present, otherwise load the saved file
if [ -z "${DST:-}" ] && [ -f ~/search-test.env ]; then source ~/search-test.env; fi
: "${DST:?DST is not set - run: set -a; source ~/search-test.env; set +a}"
DST_KEY=${DST_KEY:-$(az search admin-key show -g "$DST_RG" --service-name "$DST_SVC" --query primaryKey -o tsv)}

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
