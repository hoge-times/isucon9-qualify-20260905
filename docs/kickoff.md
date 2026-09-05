# ISUCON9予選 キックオフドキュメント

## 対象ホスト

`i1 i2 i3`（先頭ホスト i1）

出典: ユーザー指示（matsuu/aws-isucon 構成、ホストエイリアス i1/i2/i3, ssh ユーザー isucon）。i1 の実機確認済み。i2/i3 は同一AMIからの複製想定のため未確認（設定は同一である可能性が高いが個別確認はしていない）。

## webapp ディレクトリ

`/home/isucon/isucari/webapp`

出典: i1 実機確認（`ls ~` → `isucari`、`ls ~/isucari` → `webapp` を含む、`ls ~/isucari/webapp` → `go nodejs perl php python ruby sql public frontend docs README.md` を確認）。
Go実装の作業ディレクトリは `/home/isucon/isucari/webapp/go/`（`isucari.golang.service` の `WorkingDirectory` より）。

## 言語別のサービス名 (Go and PHP at least; list all present)

出典: i1 実機確認 `systemctl list-unit-files --no-legend | grep -i isu`

| service | 状態 |
|---|---|
| isucari.golang.service | enabled（現在起動中・稼働中） |
| isucari.nodejs.service | disabled |
| isucari.perl.service | disabled |
| isucari.php.service | disabled |
| isucari.python.service | disabled |
| isucari.ruby.service | disabled |

`systemctl list-units --type=service --state=running` では `isucari.golang.service`（"isucon9 qualifier main application in golang"）のみが running。

`isucari.golang.service` の内容（`systemctl cat`）:
```
WorkingDirectory=/home/isucon/isucari/webapp/go/
EnvironmentFile=/home/isucon/env.sh
ExecStart = /home/isucon/isucari/webapp/go/isucari
User = isucon / Group = isucon
Restart = always
```

## 言語切り替え手順

README.md・APPLICATION_SPEC.md には明示的な切り替え手順（マニュアル）の記載は無し（README記載の起動法はDocker/手動ビルド実行が中心）。以下は典型的なsystemctl切り替え手順（未確認・isucon標準作法）:

```bash
sudo systemctl disable --now isucari.golang.service
sudo systemctl enable --now isucari.<lang>.service
sudo systemctl status isucari.<lang>.service
```

切り替え後は `POST /initialize` を叩き、レスポンス JSON の `language` フィールドが切り替え先言語になっていることを確認する（README「ベンチマーク走行」節参照）。

## DB 接続情報

出典: i1 実機確認 `cat ~/env.sh`

```
MYSQL_HOST=127.0.0.1
MYSQL_PORT=3306
MYSQL_USER=isucari
MYSQL_DBNAME=isucari
MYSQL_PASS=isucari
```

`~/isucari/env.sh` は空/存在しない（実際に読み込まれるのは `~/env.sh` で、`isucari.golang.service` の `EnvironmentFile` がこれを指している）。

`sudo mysql` はパスワード不要（`sudo mysql -e 'select 1'` が成功、"sudo mysql: no password"）。MySQLはポート3306で `127.0.0.1` のみLISTEN（`ss -ltnp` 確認）。

## 初期化エンドポイント

`POST /initialize`（20秒以内に完了する必要あり）。

出典: README.md「ベンチマーク走行」節、および「`POST /initialize`での実装言語の出力」節。

- リクエスト例（README掲載の `initialize.json`）:
  ```json
  {
    "payment_service_url": "https://payment.t.isucon.pw",
    "shipment_service_url": "https://shipment.t.isucon.pw"
  }
  ```
- レスポンス（JSON、実装言語とキャンペーン還元率を返す）:
  ```json
  {
    "campaign": 0,
    "language": "実装言語"
  }
  ```
  - `campaign`: 0〜4の整数。0でキャンペーン無効。
  - `language`: 空だとベンチマーカー失敗扱い。

## ベンチ実行方法

`.claude/skills/isucon-deploy/contests.md` の isucon9-qualify 行（原文のまま）:

| key | 大会 | AMI | ログインユーザー | bench（サーバー上で実行） |
|---|---|---|---|---|
| isucon9-qualify | ISUCON9予選 (isucari) | ami-03b1b78bb1da5122f | ubuntu | `cd isucari && bin/benchmarker`（デフォルトターゲット = app :8000；nginx は :443；約70秒；素の状態のアプリではログインタイムアウトは正常；2026-09に検証、スコア2610） |

ベンチマーカーの主要オプション（README.md「実行オプション」節より）:

```
-target-url string
      target url (default "http://127.0.0.1:8000")
-payment-url string
      payment url (default "http://localhost:5555")
-shipment-url string
      shipment url (default "http://localhost:7001")
```

（他に `-target-host`, `-data-dir`, `-static-dir`, `-payment-port`, `-shipment-port`, `-allowed-ips` あり。）

## 特記事項

- **スコア計算**（README「スコア計算」節）: `取引が完了した商品（椅子）の価格の合計（イスコイン） - 減点 = スコア`。イスコイン還元キャンペーンの費用はスコアから引かれない。
- **失格条件**（README「スコア計算」節）:
  - 致命的なエラー（メッセージ末尾に `(critical error)`）: 1回以上で失格。
  - HTTPステータス/レスポンス内容の誤り: 1回で500イスコイン減点、10回以上で失格。
  - タイムアウト（メッセージ末尾に `（タイムアウトしました）`）: 200回超過後100回毎に5000イスコイン減点、失格なし。
  - 減点により0イスコイン以下になった場合は失格。
- **ベンチマーク走行フロー**（README）: ①`POST /initialize`（20秒以内）→ ②整合性チェック → ③負荷走行60秒 → ④走行後チェック。60秒経過後応答なしのリクエストは強制切断され、nginxアクセスログに499が記録されることがあるが減点対象外。
- **外部サービス**（EXTERNAL_SERVICE_SPEC.md）:
  - payment service: `POST /card`（カード番号→5分間有効なトークン発行、CORS対応）、`POST /token`（決済実行）。デフォルトポート5555。
  - shipment service: `POST /create`（集荷予約作成）、`POST /request`（集荷リクエスト、QRコード画像を返す）、`GET /accept`（発送、認証なし）、`GET /status`（配送ステータス確認、`initial→wait_pickup→shipping→done`）。Authorizationは `Bearer <APP_ID>`。デフォルトポート7001。
  - nginx経由で外部サービスを繋ぐ場合、`proxy_set_header Host $http_host;`（shipmentのみ必須）、`proxy_set_header X-Forwarded-Proto "https";`（HTTPS時のみ）、`proxy_set_header True-Client-IP $remote_addr;` が必要（README「ベンチマーカー」節）。
- **キャンペーン設定**（README「キャンペーン機能」節）: `POST /initialize` レスポンスの `campaign` フィールド（0〜4の整数、0で無効）でイスコイン還元率を設定。ユーザー増減に影響。詳細はAPPLICATION_SPEC.mdには「マニュアルを参照」とあるのみで、大会マニュアル自体は本リポジトリに含まれておらず未確認。
- **nginx listen ports / アプリのポート**（i1実機確認）:
  - nginx: `listen 443 ssl;`（`/etc/nginx/sites-available/isucari.conf`, `isucari.php.conf`）。80番はデフォルトvhostのみ（`sites-available/default`）。
  - nginx → app: `proxy_pass http://127.0.0.1:8000;`
  - app(isucari.golang)は `*:8000` でLISTEN（`ss -ltnp` 確認）。→ **アプリは8000番で待受、nginxが443で受けて8000へproxy_passしている**。
  - MySQLは `127.0.0.1:3306` でLISTEN。

## 主要エンドポイント一覧

APPLICATION_SPEC.md の「ISUCARI ステータス遷移表」から確認できるアクション（ページ／APIの主要パス、alp正規表現作成時の参考用）:

| パス（想定） | 操作 | WHO |
|---|---|---|
| `/sell` | 出品 | 出品者 |
| `/buy` | 購入 | 購入者 |
| `/ship` | 集荷予約 | 出品者 |
| `/ship_done` | 発送完了 | 出品者 |
| `/complete` | 取引完了 | 購入者 |

README.md「商品取得APIと更新の反映について」節に記載の商品取得系API（具体的なパスはAPPLICATION_SPEC.md/README.mdに明記なし、**未確認** — 実装コード `webapp/go/*.go` を要確認）:

- 新着一覧
- カテゴリ毎新着一覧
- ユーザ毎一覧
- 取引一覧
- 商品詳細

上記の商品取得系5APIについては、出品・編集時にカテゴリ毎新着一覧・ユーザ毎一覧・取引一覧・商品詳細への即時反映が必須。購入・編集時は全一覧・詳細取得APIへの即時反映が必須。古いデータの削除・非表示は不可、一覧が一度に返す件数は初期実装と同じである必要あり（README該当節）。

外部サービスAPI（EXTERNAL_SERVICE_SPEC.md、上記「特記事項」にも記載）:

- `POST /card`（payment）
- `POST /token`（payment）
- `POST /create`（shipment）
- `POST /request`（shipment）
- `GET /accept`（shipment）
- `GET /status`（shipment）

初期化:

- `POST /initialize`
