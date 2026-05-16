# annealing-mock

`openapi.json` (Annealing Engine API v1, OpenAPI 3.1) を **[Prism](https://github.com/stoplightio/prism)** で配信するローカルモックサーバーです。クライアント開発・契約テスト用途。実ソルバーは含みません。

- ハンドラ実装なし: spec をそのまま起動するだけ
- 4 形状の `oneOf` リクエスト (sparse/dense × constraint/penalty) を spec 駆動で検証
- 名前付き examples を埋め込み済み。`Prefer: example=<name>` でレスポンスを切替可能
- `Prefer: code=4xx|5xx` でエラー応答も再現可能

---

## 必要要件

- Node.js 18+ (Prism CLI 5.x が動作する版)
- pnpm 9+ (本リポジトリは `packageManager: pnpm@11.1.1` を指定。corepack 経由での導入を推奨)

pnpm が未インストールなら、Node.js 同梱の corepack で:

```bash
corepack enable pnpm                                 # 要 root: /usr/bin に shim
# あるいはユーザーローカルに置く場合
corepack enable --install-directory ~/.local/bin pnpm
```

## セットアップ

```bash
pnpm install
```

## 起動

| コマンド | 内容 |
|---|---|
| `pnpm mock` | 静的モード。spec 内の examples を返却 (port 4010) |
| `pnpm mock:dynamic` | 動的モード。JSON Schema Faker でレスポンスを毎回生成 |
| `pnpm mock:errors` | 違反時に Prism が spec の 4xx/5xx を返すよう設定 |
| `pnpm mock:lan` | `0.0.0.0:4010` で listen (LAN 内共有用) |

デフォルトバインドは `127.0.0.1:4010`。ヘルスチェック:

```bash
curl http://127.0.0.1:4010/v1/health
# {"status":"ok","version":"1.1.0"}
```

---

## API 概要

`openapi.json` が**唯一の真実**です。本 README はその要約にすぎません。

| メソッド | パス | 概要 |
|---|---|---|
| GET | `/v1/health` | サービス稼働確認 |
| POST | `/v1/sync/solve` | 同期実行のソルブ |

`POST /v1/sync/solve` のリクエストボディは以下 4 形状の `oneOf`：

|              | Sparse 目的関数 (`sparse_objective`) | Dense 目的関数 (`dense_objective`) |
|--------------|--------------------------------------|------------------------------------|
| **Constraint** | `RequestV1SolveSparseConstraint` (`constraints`)   | `RequestV1SolveDenseConstraint` (`constraints`)   |
| **Penalty**    | `RequestV1SolveSparsePenalty` (`penalties`, `penalty_weight_calibration`) | `RequestV1SolveDensePenalty` (`penalties`, `penalty_weight_calibration`) |

レスポンス: 200 → `ResponseV1SyncSolve` / 400, 429, 500 → `ResponseOnError`。

### スキーマで間違えやすい点

- **`PolynomialV1`** … `[係数, 変数idx_1, ..., 変数idx_k]` のリスト。`k` は 0〜4 (多項式次数 ≤ 4)。変数インデックスは `uint32`。
- **`DenseObjectiveV1.matrix`** … **上三角・ラギッド配列**。`[[a_11..a_1n], [a_22..a_2n], ..., [a_nn]]` 形式で、n×n の正方行列ではない。
- **`ConstraintV1`** … `expression` に加えて `lower` / `upper` の**少なくとも一方**が必須。
- **`PenaltyV1.weight`** … `> 0` (exclusive)。`threshold` は `≥ 0`。
- **`Solution.values`** … `0|1` の整数配列 (boolean ではない)。
- **`Solution.status`** … `Infeasible` / `Feasible` / `Optimal` のいずれか。
- **`ResponseV1SyncSolve.version`** … SemVer (`^\d+\.\d+\.\d+(-.*)?(\+.*)?$`)。
- **`ResponseV1SyncSolve.solutions`** … `minItems: 1` (成功時は空配列禁止)。
- **`num_gpus: 0`** … リクエストでは「利用可能な全 GPU を使用」の特例値。レスポンスでは実際の使用数 (≥ 0)。

---

## サンプル JSON

`examples/` 配下に共通サンプルを置いています。

```
examples/
├── request_sparse_constraint_knapsack.json   sparse + constraints (0/1 ナップサック)
├── request_dense_constraint_qubo.json        dense  + constraints (3 変数 QUBO)
├── request_sparse_penalty_pubo.json          sparse + penalties   (3 次項を含む PUBO)
├── request_dense_penalty_partition.json      dense  + penalties   (数分割)
├── response_health_ok.json                   /v1/health 200
├── response_solve_optimal.json               solve 200 (単一解 Optimal)
├── response_solve_feasible_multi.json        solve 200 (Optimal + Feasible 複数解)
├── response_solve_infeasible.json            solve 200 (Infeasible)
├── response_error_400.json                   solve 400
├── response_error_429.json                   solve 429
└── response_error_500.json                   solve 500
```

これらは `openapi.json` の `examples` ブロックにも埋め込み済みなので、Prism の `Prefer: example=<name>` で個別に呼び出せます。

### 使い方

```bash
# 起動
pnpm mock

# 別タブで叩く
curl -X POST http://127.0.0.1:4010/v1/sync/solve \
  -H 'Content-Type: application/json' \
  -d @examples/request_sparse_constraint_knapsack.json
```

---

## `Prefer` ヘッダによる挙動制御

Prism は HTTP の `Prefer` ヘッダで応答を切り替えられます。

### `Prefer: example=<name>` — 特定の名前付き examples を返す

| エンドポイント | name | 内容 |
|---|---|---|
| `GET /v1/health` | `ok` | 正常稼働 (default) |
| `GET /v1/health` | `degraded` | 縮退運転 |
| `POST /v1/sync/solve` (200) | `optimal_single` | 単一 Optimal 解 |
| `POST /v1/sync/solve` (200) | `feasible_multi` | 複数解 (Optimal + Feasible) + warnings |
| `POST /v1/sync/solve` (200) | `infeasible` | Infeasible 解 + pre-release version |
| `POST /v1/sync/solve` (400) | `invalid_matrix_shape` | dense matrix が上三角でない |
| `POST /v1/sync/solve` (400) | `constraint_missing_bound` | constraint に lower/upper なし |
| `POST /v1/sync/solve` (429) | `rate_limited` | 同時実行数上限 |
| `POST /v1/sync/solve` (500) | `internal_error` | ソルバー内部エラー |

### `Prefer: code=<status>` — 特定の HTTP コードのレスポンスを強制

```bash
curl -X POST http://127.0.0.1:4010/v1/sync/solve \
  -H 'Content-Type: application/json' \
  -H 'Prefer: code=429' \
  -d @examples/request_dense_constraint_qubo.json
# {"error":"Too many requests: 同時実行数の上限に達しました..."}
```

### `Prefer: dynamic=true` — 単発で動的生成

静的モード起動中でも、特定リクエストだけ JSON Schema Faker による動的レスポンスを得たいとき。

### 複数ヘッダの組み合わせ

```
Prefer: code=200, example=feasible_multi
Prefer: code=200, dynamic=true
```

---

## 想定リクエスト/レスポンス例

### 1. Sparse + Constraint (0/1 ナップサック)

リクエスト (`examples/request_sparse_constraint_knapsack.json`):
```json
{
  "time_limit_ms": 5000,
  "num_gpus": 1,
  "duplicate_solutions": false,
  "sparse_objective": [
    [-3, 0], [-4, 1], [-5, 2], [-6, 3]
  ],
  "constraints": [
    { "expression": [[2,0],[3,1],[4,2],[5,3]], "upper": 8 }
  ]
}
```

レスポンス (`examples/response_solve_optimal.json`, status=Optimal, values=[0,1,0,1]):
```json
{
  "solutions": [{
    "time_stamp_ms": 1234.5,
    "objective": -10,
    "values": [0, 1, 0, 1],
    "status": "Optimal"
  }],
  "execution_time_ms": 1234.5,
  "queue_time_ms": 12.3,
  "submitted_at": "2026-05-16T10:00:00.000Z",
  "started_at":   "2026-05-16T10:00:00.012Z",
  "num_gpus": 1,
  "num_samplings": 1000,
  "num_flips": 50000,
  "warnings": [],
  "version": "1.1.0"
}
```

### 2. Dense + Constraint (3 変数 QUBO)

```json
{
  "dense_objective": {
    "matrix": [[1, -2, 0], [3, -1], [2]],
    "constant": 0
  },
  "constraints": [
    { "expression": [[1,0],[1,1],[1,2]], "lower": 1, "upper": 2 }
  ]
}
```
> `matrix` は上三角ラギッド。`n=3` のとき各行の長さは 3, 2, 1。

### 3. Sparse + Penalty (3 次項を含む PUBO)

```json
{
  "num_gpus": 0,
  "sparse_objective": [
    [-2, 0], [-1, 1], [-3, 2],
    [5, 0, 1, 2]
  ],
  "penalties": [
    { "expression": [[1, 0, 1]], "weight": 10, "threshold": 0 }
  ],
  "penalty_weight_calibration": true
}
```
> `[5, 0, 1, 2]` = 係数 5 の 3 次項 `x_0 * x_1 * x_2`。`num_gpus: 0` は「全 GPU 利用」。

### 4. Dense + Penalty (3 要素数分割)

```json
{
  "dense_objective": {
    "matrix": [[-20, 16, 24], [-32, 48], [-36]],
    "constant": 36
  },
  "penalties": [
    { "expression": [[1, 0]], "weight": 0.5, "threshold": 0 }
  ],
  "penalty_weight_calibration": false
}
```

---

## Prism 利用上の注意

CLAUDE.md にもまとめてありますが、共有時の Q&A 頻出ポイント:

- **`oneOf` の選択順**: Prism は最初に required を満たす sub-schema にマッチさせます。リクエストに `sparse_objective` と `dense_objective` を両方混在させると順序依存になるので、必ずどちらか一方だけ送ること。
- **リクエスト検証エラーは 422 か 4xx**: spec が `400` を宣言しているため、Prism は spec 上の `400` を選びます。`--errors` を付けると違反時に spec の 4xx/5xx を能動的に選択。
- **動的モードのレスポンス**: schema 上は valid ですが、`solutions[].values` の長さと多項式の変数数の整合などはランダム。整合性が必要なテストは `Prefer: example=...` で静的例に切替を推奨。
- **`PolynomialV1` の表記**: 本リポジトリでは Prism の AJV が `prefixItems` を十分に解釈できない事象を回避するため、`items: [...]` + `additionalItems` の Draft-7 互換タプル構文に書き換えています。意味 (先頭=係数 number、以降=変数インデックス integer ≥ 0、長さ 1〜5) は spec オリジナルと同じ。

---

## リポジトリ構成

```
.
├── README.md            このファイル
├── CLAUDE.md            Claude Code 用ガイド (背景・選定理由・運用ノート)
├── openapi.json         API 契約 (source of truth)。examples を埋め込み済み
├── package.json         Prism CLI を pin、packageManager を pnpm に固定
├── pnpm-lock.yaml
├── .gitignore
└── examples/            リクエスト 4 + レスポンス 7 の JSON サンプル
```

`openapi.json` を変更する際は CLAUDE.md の方針に従い、API 挙動の変更時には `info.version` を bump してください。
