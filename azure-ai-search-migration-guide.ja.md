# Azure AI Search バックアップおよび移行ガイド

Azure AI Search サービスを新しいリソースグループへ複製するための手順書です。サービスを再構築し、すべてのオブジェクト定義をコピーしたうえで、ドキュメントを移行します。各ステップには、実行するコマンドと想定される結果を記載しています。移行元サービスは、最終ステップまで一切変更されません。

> 本ガイドは、Windows の Git Bash を含む環境で、サンプルサービスを用いてエンドツーエンドの検証を実施済みです。検証中に判明したエラーへの対処は、以下のスクリプトにあらかじめ組み込まれています。

---

## 目次

- [概要](#概要)
- [インデックスごとのデータ移行方法の選択](#インデックスごとのデータ移行方法の選択)
- [前提条件](#前提条件)
- [セットアップ — 設定の保存と読み込み](#セットアップ--設定の保存と読み込み)
- [ステップ 1 — 移行元サービスの棚卸し](#ステップ-1--移行元サービスの棚卸し)
- [ステップ 2 — 移行先サービスの作成](#ステップ-2--移行先サービスの作成)
- [ステップ 3 — ID とネットワークアクセスの付与](#ステップ-3--id-とネットワークアクセスの付与)
- [ステップ 4 — すべてのオブジェクト定義のエクスポート](#ステップ-4--すべてのオブジェクト定義のエクスポート)
- [ステップ 5 — シークレットの復元とインポート](#ステップ-5--シークレットの復元とインポート)
- [ステップ 6 — ドキュメントの移行](#ステップ-6--ドキュメントの移行)
- [ステップ 7 — 検証](#ステップ-7--検証)
- [ステップ 8 — 切り替えと移行元の廃止](#ステップ-8--切り替えと移行元の廃止)
- [テストで判明したエラー](#テストで判明したエラー)
- [トラブルシューティングとロールバック](#トラブルシューティングとロールバック)
- [まとめ](#まとめ)

---

## 概要

Azure AI Search には、セルフサービスのバックアップ・復元機能が用意されていません。そのため、複製は次の 3 つのレイヤーに分けて再構築します。

| レイヤー | 対象 | 移行方法 |
| --- | --- | --- |
| **サービス** | 価格レベル、レプリカ、パーティション、リージョン、セマンティックランカー、マネージド ID、ネットワークルール | Azure CLI · ステップ 2～3 |
| **定義** | インデックス、シノニムマップ、データソース、スキルセット、インデクサー | REST API による JSON のエクスポートとインポート · ステップ 4～5 |
| **ドキュメント** | インデックスに格納されたコンテンツ | インデクサーの再実行、またはダンプと再読み込み · ステップ 6 |

複製ではなく、既存サービスを**移動**するだけの場合（名前とエンドポイントは変わりません）は、ポータルで *検索サービス → 移動 → 別のリソースグループに移動* を使用してください。

## インデックスごとのデータ移行方法の選択

| | 方法 A · インデクサーの再実行 | 方法 B · ダンプと再読み込み |
| --- | --- | --- |
| 適用するケース | インデクサーでデータを投入しているインデックス | アプリケーションからドキュメントをプッシュしているインデックス |
| コピーされる範囲 | すべて（移行元データから再構築） | `retrievable` が設定されたフィールドのみ |
| コスト | スキルセットおよび埋め込みを再実行 | 通信量と作業時間 |
| 所要時間 | 大規模なデータでは数時間～数日 | 数分～数時間 |
| 注意点 | 全件の再インデックスが発生し、変更追跡は引き継がれません | 取得不可のフィールド（多くの場合ベクター）は空になります |

多くのサービスでは両方を併用します。インデクサーで投入しているインデックスには **A**、プッシュ型のインデックスには **B** を使用してください。どちらを使うかは、ステップ 1 でインデックスごとに判断します。インデクサーの投入先になっていれば方法 A です。

## 前提条件

- [ ] bash ターミナルで、Azure CLI 2.50 以降（`az login` でサインイン済み）、`curl`、`jq` が使用できること
- [ ] 移行元および移行先のリソースグループに対する共同作成者（Contributor）権限
- [ ] 両サービスの管理キー、または Search Service Contributor ロール
- [ ] すべてのデータソースおよびスキルセットの接続文字列と AI サービスキー（**エクスポート時に空になります**）
- [ ] 方法 B で Microsoft のバックアップツールを使用する場合は .NET 8 SDK
- [ ] 移行中にアプリからドキュメントがプッシュされる場合は、書き込み停止の時間帯

> **Windows をご利用の場合：** Git Bash で実行できますが、Git Bash の `jq` は出力の各行末に見えない `\r` を付加するため、URL 内のオブジェクト名が壊れます。以下のセットアップに、これを解消する 1 行の対処を含めています。

## セットアップ — 設定の保存と読み込み

**1. 設定値を保存します**（事前に名前を編集してください。移行先サービス名はグローバルで一意である必要があります）。

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

**2. ターミナルのセッションを開始するたびに、以下を実行します。**

```bash
set -a; source ~/search-migration.env; set +a
export SRC_KEY=$(az search admin-key show -g $SRC_RG --service-name $SRC_SVC --query primaryKey -o tsv)

# Windows の Git Bash のみ：jq が付加する \r を除去します
jq() { command jq "$@" | tr -d '\r'; }; export -f jq

# ステップ 2 で移行先サービスを作成した後に実行します：
# export DST_KEY=$(az search admin-key show -g $DST_RG --service-name $DST_SVC --query primaryKey -o tsv)
```

`set -a` によって、すべての変数が**エクスポート**されます。これがないと、`./` で起動したスクリプトから変数を参照できず、`unbound variable` で停止します。

---

## ステップ 1 — 移行元サービスの棚卸し

後の工程で比較に使用する情報をすべて記録します。

```bash
mkdir -p ~/backup && cd ~/backup
az search service show -g $SRC_RG -n $SRC_SVC > service.json

# すべてのオブジェクトを一覧表示
names() { curl -s "$1/$2?api-version=$API&\$select=name" -H "api-key: $3" | jq -r '.value[].name'; }
for t in indexes synonymmaps datasources skillsets indexers; do echo "== $t"; names $SRC $t $SRC_KEY; done

# ドキュメント数のベースライン（ステップ 7 で再度使用）
for i in $(names $SRC indexes $SRC_KEY); do
  echo "$i: $(curl -s "$SRC/indexes/$i/docs/\$count?api-version=$API" -H "api-key: $SRC_KEY")"
done | tee doc-counts.txt
```

- `service.json` には、ステップ 2 で引き継ぐ価格レベル、レプリカ、パーティション、セマンティック設定が含まれます。
- このサービスを呼び出しているアプリケーションもすべて控えておいてください。ステップ 8 で接続先を切り替えます。
- 以降の作業はすべて `~/backup` で行います。

## ステップ 2 — 移行先サービスの作成

リソースグループが存在しない場合は、先に作成してください（`az group create -n $DST_RG -l $LOCATION`）。

```bash
az search service create -g $DST_RG -n $DST_SVC -l $LOCATION \
  --sku standard --replica-count 1 --partition-count 1 \
  --semantic-search free --identity-type SystemAssigned \
  --auth-options aadOrApiKey -o table

export DST_KEY=$(az search admin-key show -g $DST_RG --service-name $DST_SVC --query primaryKey -o tsv)
```

- **移行元に合わせる：** 価格レベル（`--sku`）、レプリカ、パーティション、`--semantic-search` を `service.json` の値に合わせてください。価格レベルは、移行元のストレージ容量とオブジェクト数を収容できるものが必要です。
- **新しい名前と新しいエンドポイント：** ステップ 8 で、クライアントに新しい URL とキーを設定する必要があります。
- プロビジョニングには約 3～10 分かかります。

*別の方法：* ポータルで移行元サービス → オートメーション → **テンプレートのエクスポート** を開き、サービス名を変更して新しいリソースグループへデプロイします。この方法ではネットワーク設定も引き継がれます。

## ステップ 3 — ID とネットワークアクセスの付与

新しいサービスには、ロールが割り当てられていない新しいマネージド ID が付与されます。ステップ 5 より前に対応しないと、インデクサーが初回実行時に失敗します。データソースで接続文字列を使用している場合、ロールの割り当ては不要です。

```bash
DST_MI=$(az search service show -g $DST_RG -n $DST_SVC --query identity.principalId -o tsv)
az role assignment create --assignee-object-id $DST_MI --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Reader" --scope <storage-account-resource-id>
```

| リソース | 新しい ID に割り当てるロール |
| --- | --- |
| Blob / ADLS ストレージ | Storage Blob Data Reader |
| ナレッジストア、エンリッチメントキャッシュ | Storage Blob Data Contributor |
| Azure OpenAI | Cognitive Services OpenAI User |
| Azure AI サービス | Cognitive Services User |
| Azure SQL | `db_datareader` を持つ包含データベースユーザー |

旧 ID に割り当てられているロールは、`az role assignment list --assignee <old-principal-id> --all -o table` で確認できます。ロールの割り当てが反映されるまでに約 10 分かかる場合があります。

**あわせて再作成が必要なもの：** IP ファイアウォールルール、プライベートエンドポイントと DNS、共有プライベートリンク（再承認が必要）、Entra ID クライアント用のアプリロール、カスタマーマネージドキー。

## ステップ 4 — すべてのオブジェクト定義のエクスポート

`~/backup` に `export-definitions.sh` として保存します。

```bash
#!/usr/bin/env bash
# export-definitions.sh — バックアップフォルダーで実行します
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

実行して結果を確認します。

```bash
chmod +x export-definitions.sh && ./export-definitions.sh
find . -name '*.json' | sort
```

オブジェクトごとに `exported` が 1 行ずつ表示されれば**正常**です。`FAIL` の場合は、その下にサービスからのエラーが表示されます。`backup` フォルダーは保管してください。これらの JSON ファイルが構成のバックアップとなります。

## ステップ 5 — シークレットの復元とインポート

### 5a. シークレットの再設定

エクスポートでは、すべてのシークレットが `null` として返されます。インポートの前に、それぞれ値を設定してください。

| エクスポートされたファイル | `null` で返されるフィールド | 設定する値 |
| --- | --- | --- |
| `datasources/*.json` | `credentials.connectionString` | 接続文字列、またはマネージド ID の場合は `ResourceId=/subscriptions/.../storageAccounts/<acct>;` |
| `skillsets/*.json` | `cognitiveServices.key`、`apiKey` | AI サービスまたは Azure OpenAI のキー（ID を使用する場合は削除） |
| `indexes/*.json` | ベクトライザーの `apiKey` | Azure OpenAI のキー |

1 つのデータソースに対する例です（事前に `CONN` に実際の接続文字列を設定し、データソースごとに繰り返してください）。

```bash
jq --arg cs "$CONN" '.credentials.connectionString = $cs' datasources/my-ds.json > tmp \
  && mv tmp datasources/my-ds.json

# "no blank secrets" と表示されれば問題ありません
grep -rl '"connectionString": null\|"key": null\|"apiKey": null' . || echo "no blank secrets"
```

### 5b. 依存関係の順序に沿ったインポート

順序：**シノニムマップ → インデックス → データソース → スキルセット → インデクサー**。`import-definitions.sh` として保存します。

```bash
#!/usr/bin/env bash
# import-definitions.sh — バックアップフォルダーで実行します
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

オブジェクトごとに `OK` が 1 行ずつ表示されれば**正常**です。インデクサーは**無効化された状態**で作成されるため、ステップ 6 までは実行されません。`FAIL` の場合はエラー内容が表示されます。該当ファイルを修正して再実行してください。PUT は既存のオブジェクトを上書きするだけです。

## ステップ 6 — ドキュメントの移行

### 方法 A — インデクサーの再実行（インデクサーで投入しているインデックス）

```bash
for f in indexers/*.json; do n=$(basename "$f" .json)
  jq '.disabled = false' "$f" | curl -s -X PUT "$DST/indexers/$n?api-version=$API" \
    -H "api-key: $DST_KEY" -H 'Content-Type: application/json' -d @- > /dev/null
  curl -s -X POST "$DST/indexers/$n/run?api-version=$API" -H "api-key: $DST_KEY" -H 'Content-Length: 0'
  echo "started $n"
done
```

ドキュメント数を確認したうえで、実行履歴を**すべて**確認します。

```bash
curl -s "$DST/indexes/<index>/docs/\$count?api-version=$API" -H "api-key: $DST_KEY"; echo
curl -s "$DST/indexers/<indexer>/status?api-version=$API" -H "api-key: $DST_KEY" \
  | jq '[.executionHistory[] | {status, processed: .itemsProcessed}]'
```

> 以前の実行ですでに全件がインデックスされている場合、`status: success` かつ `processed: 0` となっても問題ありません。インデクサーを有効化すると自動的に実行が開始される場合があるため、実行が 2 回続けて記録されることがあります。最新の実行結果だけでなく、ドキュメント数と実行履歴で判断してください。

1 回の実行には時間の上限があります。大規模なインデクサーは次回のスケジュール実行で処理を継続するため、各インデクサーにスケジュールが設定されていることを確認してください。

### 方法 B — ダンプと再読み込み（プッシュ型のインデックス）

**小規模なインデックス（1,000 ドキュメントまで）** — 移行元からダンプし、移行先へアップロードします。

```bash
curl -s -X POST "$SRC/indexes/<index>/docs/search?api-version=$API" -H "api-key: $SRC_KEY" \
  -H 'Content-Type: application/json' -d '{"search":"*","top":1000}' \
  | jq '{value: [.value[] | del(."@search.score") | . + {"@search.action":"upload"}]}' > docs.json

jq '.value | length' docs.json      # アップロード前に件数を確認

curl -s -X POST "$DST/indexes/<index>/docs/index?api-version=$API" -H "api-key: $DST_KEY" \
  -H 'Content-Type: application/json' -d @docs.json | jq -c '[.value[].status]'
```

アップロードの結果、`true` の一覧が表示されれば正常です。

> 複数行のコマンドは、そのまま貼り付けてください。1 行にまとめる場合は、**バックスラッシュを削除**してください。削除しないと bash が `-H` ヘッダーを読み落とし、`docs.json` が空になります。

**大規模なインデックス** — Microsoft のバックアップツールを使用します。

1. `git clone https://github.com/Azure-Samples/azure-search-dotnet-utilities.git`
2. `index-backup-restore`（v11）の `appsettings.json` に、移行元・移行先のサービス名、管理キー、インデックス名、`BackupDirectory` を設定します。両方のキーが含まれるため、このファイルはコミットしないでください。
3. インデックスごとに `dotnet run` を実行します。移行先インデックスがすでに存在する場合（ステップ 5 で空の状態で作成済み）は、削除してから再実行してください。

**方法 B でコピーできないもの：**

- `retrievable: false` のフィールド（多くの場合ベクター）は空で返されます。該当するインデックスには方法 A を使用してください。
- 検索の `skip` では 100,000 件を超えてページングできません。非常に大規模なインデックスは、並べ替え可能なフィールドでフィルターをかけ、分割してエクスポートしてください。

## ステップ 7 — 検証

**先にすべてのインデックスでステップ 6 を完了させてください。** プッシュ型のインデックスは、方法 B を実行するまでドキュメント数が 0 と表示されます。これは手順の未実施であり、失敗ではありません。

ステップ 1 のベースラインとドキュメント数を比較します（`counts match` と表示されれば正常です）。

```bash
for i in $(curl -s "$DST/indexes?api-version=$API&\$select=name" -H "api-key: $DST_KEY" | jq -r '.value[].name'); do
  echo "$i: $(curl -s "$DST/indexes/$i/docs/\$count?api-version=$API" -H "api-key: $DST_KEY")"
done | diff doc-counts.txt - && echo "counts match"
```

両方のサービスに同じクエリを実行し、結果を比較します。

```bash
Q='{"search":"<a typical query>","top":5}'
for s in "$SRC $SRC_KEY" "$DST $DST_KEY"; do set -- $s
  curl -s -X POST "$1/indexes/<index>/docs/search?api-version=$API" -H "api-key: $2" \
    -H 'Content-Type: application/json' -d "$Q" | jq -c '[.value[] | ."@search.score"]'
done
```

チェックリスト：

- [ ] オブジェクトの一覧がステップ 1 と一致している
- [ ] ドキュメント数が一致している
- [ ] サンプルクエリの上位結果が一致している
- [ ] セマンティック、ベクター、シノニムの各クエリが動作する
- [ ] インデクサーが正常に完了し、スケジュールが維持されている
- [ ] クライアントが新しいキーまたはロールでクエリを実行できる

インデックス作成の直後は、ドキュメント数の反映が数秒遅れることがあります。差異を失敗と判断する前に、再度確認してください。

## ステップ 8 — 切り替えと移行元の廃止

1. **書き込みの停止** — 更新の欠落を防ぐため、移行元へのプッシュを停止するか、両方のサービスに書き込みます。
2. **クライアントの接続先変更** — アプリ設定、Key Vault、Azure OpenAI / AI Foundry の接続、Logic Apps に、新しいエンドポイントとキー（または Entra ロール）を設定します。
3. **移行元の停止** — エンリッチメントの二重課金を避けるため、移行元のインデクサーを無効化します。

   ```bash
   for f in indexers/*.json; do n=$(basename "$f" .json)
     jq '.disabled = true' "$f" | curl -s -X PUT "$SRC/indexers/$n?api-version=$API" \
       -H "api-key: $SRC_KEY" -H 'Content-Type: application/json' -d @- > /dev/null; done
   ```

4. **監視、バックアップ、削除** — 数日間レイテンシとスロットリングを監視し、最終バックアップ（ステップ 4 と方法 B の再実行）を取得してから削除します。

   ```bash
   # 元に戻せません：削除したサービスやインデックスは復元できません
   az search service delete -g $SRC_RG -n $SRC_SVC --yes
   ```

---

## テストで判明したエラー

| 発生した事象 | 原因 | 対処 |
| --- | --- | --- |
| `SRC: unbound variable` | 変数が設定されているがエクスポートされておらず、`./script` は子シェルで実行される | `set -a; source ~/search-migration.env; set +a` |
| `HTTP 000`、または `docs-blob$'\r'.json` という名前のファイル | Git Bash の `jq` が行末に `\r\n` を出力する | `jq() { command jq "$@" \| tr -d '\r'; }; export -f jq` |
| 1 行に貼り付けたコマンドの出力が空 | スペースの前のバックスラッシュにより `-H` ヘッダーが失われる | 1 行にまとめる際は `\` を削除する |
| インデクサーが `success` で `processed: 0` | 以前の実行ですでに処理が完了している | `docs/$count` と `executionHistory` を確認する |
| スクリプトがメッセージなしで途中停止する | 旧スクリプトが `set -e` と `curl -sf` を使用していた | 本ガイドのスクリプトを使用する（エラーを表示して処理を継続します） |

## トラブルシューティングとロールバック

| 症状 | 想定される原因 | 対処 |
| --- | --- | --- |
| REST 呼び出しで `403` | キーの誤り、またはローカル認証が無効 | キーを再取得する。または `az account get-access-token --resource https://search.azure.com` でトークンを取得し、`Authorization: Bearer <token>` を送信する |
| インデックスの PUT でクォータエラー | 移行先の価格レベルが不足している | 価格レベルまたはパーティションを引き上げ、インポートを再実行する |
| インデックスの PUT でセマンティック構成エラー | セマンティックランカーが無効 | `az search service update -g $DST_RG -n $DST_SVC --semantic-search free` |
| データソースで認証情報のエラー | シークレットが `null` のまま、またはロールが不足している | ステップ 5a またはステップ 3 をやり直す（ロールの反映には約 10 分） |
| インデクサーでスキルが失敗する | AI キーまたは OpenAI のロールが不足している | `apiKey` を設定する、または Cognitive Services OpenAI User を付与する |
| 復元したベクターが空 | フィールドが取得不可（retrievable でない） | 該当インデックスには方法 A を使用する |
| ドキュメント数が移行元より少ない | インデクサーが実行中、または 1,000 件 / 100,000 件のダンプ上限 | 時間をおいて再確認する。バックアップツールまたはフィルターによる分割を使用する |

**ロールバック**は、最終的な削除の前であればいつでも可能です。クライアントの接続先を移行元に戻し、新しいサービスのみを削除してください（`az search service delete -g $DST_RG -n $DST_SVC --yes`）。共有のリソースグループは削除しないでください。

---

## まとめ

1. **棚卸し** — 設定、オブジェクト、ドキュメント数
2. **サービスの作成** — 同じ価格レベルと機能
3. **アクセス** — ID のロール、ネットワーク
4. **エクスポート** — 定義を JSON に保存
5. **インポート** — シークレットを戻し、順序どおりに作成
6. **データの移行** — インデクサー（A）またはダンプと再読み込み（B）
7. **検証** — ドキュメント数とクエリ結果の一致
8. **切り替え** — 接続先を変更し、移行元を廃止

`backup` フォルダーは保管してください。エクスポートした定義とダンプしたドキュメントが、特定時点のバックアップとなります。継続的なバックアップが必要な場合は、新しいサービスに対してステップ 4 と方法 B を定期実行するよう設定してください。

### 参考資料

- [Azure AI Search FAQ: move, backup and restore](https://learn.microsoft.com/azure/search/search-faq-frequently-asked-questions)
- [Back up and restore an Azure AI Search index (sample)](https://learn.microsoft.com/en-us/samples/azure-samples/azure-search-dotnet-utilities/azure-search-backup-restore-index)
- [Microsoft Q&A: backing up Azure AI Search data](https://learn.microsoft.com/en-za/answers/questions/1792032/how-do-i-backup-data-of-azure-ai-search-service)
- [Microsoft Q&A: re-indexing after migration](https://learn.microsoft.com/en-us/answers/questions/1616243/avoid-re-indexing-all-documents-after-migration-of)
