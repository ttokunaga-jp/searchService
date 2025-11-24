# searchService API 統合ガイド

searchService は Elasticsearch と Qdrant を組み合わせたハイブリッド検索マイクロサービスです。本ドキュメントは、他のマイクロサービスから searchService と API 通信を行う開発者向けに、サービスの役割と API 仕様をまとめたものです。詳細な protobuf 定義は `api/proto/search/v1/search.proto` を参照してください。

## サービスの位置付け

- **責務:** gRPC/HTTP 経由で検索リクエストを受け、Elasticsearch (キーワード) と Qdrant (ベクトル) を統合したスコアで結果を返します。インデックス作成／更新イベントは Kafka から取り込みます。
- **上流 (Callers):** Web/API サーバー、レコメンド基盤、管理ポータルなど検索結果を必要とするマイクロサービス。
- **下流 (Dependencies):**
  - Elasticsearch クラスタ (`SEARCH_ELASTICSEARCH_ADDRESSES`)
  - Qdrant コレクション (`SEARCH_QDRANT_ADDRESS`)
  - Kafka ブローカ (`SEARCH_KAFKA_BROKERS`)：`SEARCH_KAFKA_TOPIC` でインデックス upsert/delete イベントを購読
  - Observability: OpenTelemetry Exporter (`SEARCH_OBSERVABILITY_TRACING_ENDPOINT`), Prometheus `/metrics`
- **通信経路:**
  1. gRPC (`:50051` デフォルト) および gRPC-Gateway 経由の HTTP/JSON (`/v1/...`) でリクエストを受信
  2. 内部で Elasticsearch/Qdrant へクエリ & 結果統合
  3. Kafka コンシューマが index イベントを処理し、両データストアを更新

## 接続エンドポイント

| 用途 | プロトコル | デフォルト | 備考 |
| --- | --- | --- | --- |
| Search API | gRPC | `localhost:50051` | proto: `api.proto.search.v1.SearchService` |
| Search API (HTTP) | HTTP/JSON | `http://localhost:8080` (例) | gRPC-Gateway が `/v1/indexes/...` を公開 |
| メトリクス | HTTP | `http://localhost:9464/metrics` | Prometheus 形式 |
| トレース | OTLP/HTTP | `http://localhost:4318` | 外形監視用 |

> **認証:** 現状、リポジトリには明示的な認証/認可実装はありません。上流サービス側で mTLS や API Gateway 等を用意してください。

## API 共通仕様

- **フォーマット:** gRPC (proto3)。HTTP クライアントは JSON ボディを同一スキーマで送信できます (`body: "*"`)。
- **ヘッダー:** `x-request-id` や `traceparent` を送るとトレーシングに引き継がれます。
- **エラー:** gRPC の `status.Code` と HTTP ステータスをマッピング。`INVALID_ARGUMENT`→400、`NOT_FOUND`→404、`INTERNAL`→500 など。
- **ページネーション:** `page_size` と `page_token`（次トークンはレスポンス `next_page_token`）で実装。
- **フィルタ/ソート:** `Filter` と `SortBy` は複数指定可能。フィールド名は作成時のスキーマと一致させてください。

## API リファレンス

### SearchDocuments

- **gRPC:** `SearchService.SearchDocuments`
- **HTTP:** `POST /v1/indexes/{index_name}/search`
- **用途:** 指定インデックスに対するハイブリッド検索。テキスト（`query_text`）のみ、ベクトル（`query_vector`）のみ、または両方の組合せをサポート。

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `index_name` | string | ✔ | ターゲットインデックス。例: `exams` |
| `query_text` | string |  | BM25 等のキーワード検索用クエリ |
| `query_vector` | repeated float |  | Qdrant へ送る埋め込みベクトル。`IndexSchema.vector_config.dimension` と一致させる |
| `filters` | `Filter` |  | フィールド条件 (`field`, `operator`, `value`) |
| `sort_by` | `SortBy` |  | 指定しない場合はハイブリッドスコア降順 |
| `page_size` | int32 |  | 1〜100 推奨。未指定はサーバー既定（例: 20） |
| `page_token` | string |  | 次ページトークン。初回は空 |

**レスポンス**

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `results` | `SearchResult[]` | `document_id`, `score`, `fields` (任意キー値) |
| `total_count` | int64 | マッチ総数 |
| `next_page_token` | string | 追加ページがある場合のみ返却 |

**HTTP リクエスト例**

```http
POST /v1/indexes/exams/search HTTP/1.1
Host: search-service.local
Content-Type: application/json
X-Request-ID: 7d8b8a85-7d3f-4f08-94a7-a3c3f9d9f5ff

{
  "query_text": "線形代数 期末",
  "query_vector": [0.12, -0.04, ...],
  "filters": [
    {"field": "year", "operator": "OPERATOR_EQUAL", "value": 2024},
    {"field": "difficulty", "operator": "OPERATOR_LESS_THAN", "value": 4}
  ],
  "page_size": 10
}
```

**grpcurl 例**

```sh
grpcurl -plaintext \
  -d '{"index_name":"exams","query_text":"linear algebra","page_size":5}' \
  localhost:50051 api.proto.search.v1.SearchService/SearchDocuments
```

### CreateIndex

- **gRPC:** `SearchService.CreateIndex`
- **HTTP:** `POST /v1/indexes`
- **用途:** Elasticsearch と Qdrant のスキーマを同時に作成。主に管理系サービス／運用ツールから呼び出します。

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `index_name` | string | ✔ | 一意なインデックス名。変更は不可のため命名規則に注意 |
| `schema.fields[]` | `FieldDefinition` | ✔ | フィールド名と型（TEXT/KEYWORD/INTEGER/FLOAT/BOOLEAN/DATE） |
| `schema.vector_config` | `VectorConfig` | ✔ | `dimension` と `distance` (COSINE/EUCLID/DOT_PRODUCT) |

**レスポンス**

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `success` | bool | true で作成成功 |
| `message` | string | 成功/失敗の詳細理由 |

**リクエスト例**

```json
{
  "index_name": "exams",
  "schema": {
    "fields": [
      {"name": "title", "type": "FIELD_TYPE_TEXT"},
      {"name": "year", "type": "FIELD_TYPE_INTEGER"},
      {"name": "difficulty", "type": "FIELD_TYPE_FLOAT"}
    ],
    "vector_config": {
      "dimension": 768,
      "distance": "DISTANCE_COSINE"
    }
  }
}
```

## 代表的な統合フロー

### 検索リクエストフロー

1. 呼び出し元サービスが gRPC/HTTP で `SearchDocuments` を呼ぶ（リクエスト ID を付与）。
2. searchService がクエリを解析し、Elasticsearch へキーワード検索、Qdrant へベクトル検索を実施。
3. 取得結果をスコア正規化＋重み付け統合し、`SearchResult` として返却。
4. メトリクス (`grpc_server_request_duration_seconds`, `search_index_ops_total`) とトレースが送信される。

### インデックス更新フロー

1. データ投入パイプラインが `SEARCH_KAFKA_TOPIC` に upsert/delete イベントを publish。
2. `internal/adapter/message` が Kafka からイベントを pull し、`internal/service/indexer` が Elasticsearch/Qdrant を更新。
3. 失敗時は DLQ もしくは再試行ロジックでリカバリ（詳細はソース参照）。

## 運用上の注意点

- **スキーマ互換性:** `CreateIndex` で定義したフィールドは両ストレージに反映されるため、破壊的変更時は新規インデックス作成→再投入を推奨。
- **タイムアウト/リトライ:** gRPC クライアントは 1〜3 秒のデッドライン設定と指数バックオフを推奨。HTTP クライアントは 429/503 で再試行。
- **観測性:** エラーレートやレイテンシ SLO (P95 ≤ 500ms, エラー率 ≤ 0.1%) を Prometheus アラートで監視。
- **ローカル開発:** Docker Compose (`make docker-up`) で依存コンポーネントをまとめて起動できます。API は `localhost:50051`（gRPC）にバインドされます。

## 追加リソース

- `README.md`: アーキテクチャ概要とセットアップ
- `docs/ci-cd.md`: CI パイプラインと品質ゲート
- `docs/operational-readiness.md`: 運用チェックリスト
- `docs/security.md`: ログ・認証・シークレットの方針

本ドキュメントでカバーされていない詳細は、該当ディレクトリのコード／プロトコル定義を参照してください。
