# 継承固有スキル情報取得

U-toolsから最新の継承固有スキル情報を取得し、`references/trainee.json` に保存する。

## 実行手順

### ステップ1: スキルID一覧の取得

`https://xn--gck1f423k.xn--1bvt37a.tools/skills/specials` を以下のプロンプトでWebFetchする:

> ページ内に存在する全スキルのリンク（href）を列挙してください。
> `/skills/` で始まるリンクを全て抽出し、IDと共に以下の形式でリストアップしてください:
> - ID, スキル名, キャラ名表示
> 10000番台（本体固有）と100000番台（進化固有）の両方を含めてください。

取得したIDを全て記録する。

### ステップ2: 各スキルの詳細取得（並列処理）

全IDに対して `https://xn--gck1f423k.xn--1bvt37a.tools/skills/{id}` を5件ずつ並列でWebFetchする。

各ページから以下のプロンプトで情報を抽出する:

> このスキルページから以下の全フィールドを抽出してJSON形式で返してください:
> - スキル名
> - キャラのフルネーム（例: "通常ゴルシ" や "[スペシャルドリーマー] スペシャルウィーク" など、U-toolsの表示名をそのまま）
> - 発動条件の順位範囲: ページに表示されている順位範囲（デフォルト表示）を `LoH` 値として記録する。
>   次に、発動条件の右にある頭数切替アイコンを押下した場合の順位範囲（9頭立て表示）を `チャンミ` 値として記録する。
>   アイコン切替後の値が取得できない場合は `チャンミ: [min, min(max, 9)]` （max が null の場合はそのまま null）で計算する。
> - 発動条件: 順位割合(min/max)、走行距離%(min/max)、フェーズ、脚質
> - 本人使用時の効果: 目標速度(m/s)、加速度(m/s²)、持久力回復率(%)、継続時間(秒 × 距離/1000)、クールダウン
> - 継承時効果（200pt）: 目標速度、加速度、持久力回復率、継続時間(秒)、取得コスト(pt)
> 値が存在しない場合は null を返してください。

### ステップ3: データの整形

取得した情報を以下のルールで整形する。

#### full_name / character_name / costume_name の分解

| U-toolsの表示 | full_name | character_name | costume_name |
|-------------|-----------|----------------|--------------|
| `通常ゴルシ` | `通常ゴルシ` | `ゴールドシップ` | `通常` |
| `[スペシャルドリーマー] スペシャルウィーク` | `[スペシャルドリーマー] スペシャルウィーク` | `スペシャルウィーク` | `スペシャルドリーマー` |
| `[Eternal Fairytale] ラインクラフト` | `[Eternal Fairytale] ラインクラフト` | `ラインクラフト` | `Eternal Fairytale` |

- `[]` で囲まれた衣装名がある場合: `character_name` は `]` 以降のスペース除去後の名前、`costume_name` は `[]` 内の文字列
- `[]` がない場合: 先頭の「通常」「SS」などが `costume_name`、それ以降が `character_name`（判断できない場合は `full_name` と同じ値を `character_name` に入れ `costume_name` は `null`）

#### category の判定

- スキルIDが `10000`〜`99999` の範囲 → `"固有"`
- スキルIDが `100000` 以上 → `"進化固有"`

#### duration_multiplier フィールド

継続時間が「X秒 × レース距離/1000」形式の場合:
- `duration_base_sec` に X を格納
- `duration_multiplier` に `"distance_m / 1000"` を格納
固定秒数の場合は `duration_multiplier` を `null` にする。

### ステップ4: 親進化フラグの手動補足

以下のキャラクターのエントリに限り `parent_evolution` フィールドを手動で設定する。
他の全キャラクターは `is_evolution_trigger: false`、`evolves_skill_id: null`、`note: null` とする。

| キャラ（full_name） | is_evolution_trigger | note |
|------------------|---------------------|------|
| `ステイゴールド`（通常） | `true` | `"親として使用時に特定の継承先の固有スキルが進化する"` |
| `[Eternal Fairytale] ラインクラフト` | `true` | `"親として使用時に特定の継承先の固有スキルが進化する"` |

`evolves_skill_id` は現時点では `null` とし、判明次第手動で更新する。

### ステップ5: JSONの組み立て

全スキルのデータを以下のスキーマに従ったJSON配列として組み立てる:

```json
[
  {
    "id": "10071",
    "skill_name": "波乱注意砲！",
    "character_name": "ゴールドシップ",
    "costume_name": "通常",
    "full_name": "通常ゴルシ",
    "category": "固有",
    "activation_conditions": {
      "rank_range": { "チャンミ": [7, 9], "LoH": [7, 12] },
      "rank_pct_min": null,
      "rank_pct_max": 50,
      "distance_pct_min": 50,
      "distance_pct_max": 60,
      "phase": null,
      "running_styles": []
    },
    "effects": {
      "target_speed_mps": 0.15,
      "acceleration_mps2": null,
      "stamina_recovery_pct": null,
      "duration_base_sec": 6.0,
      "duration_multiplier": "distance_m / 1000",
      "cooldown_base_sec": 500
    },
    "inherited_effects": {
      "target_speed_mps": 0.05,
      "acceleration_mps2": null,
      "stamina_recovery_pct": null,
      "duration_sec": 3.6,
      "cost_pt": 200
    },
    "parent_evolution": {
      "is_evolution_trigger": false,
      "evolves_skill_id": null,
      "note": null
    }
  }
]
```

### ステップ6: ファイル保存

組み立てたJSON配列を `references/trainee.json` に書き込む。
既存の内容は全て上書きする。

保存後、以下を報告する:
- 保存したスキル件数（本体固有 X 件、進化固有 Y 件）
- 親進化フラグを設定したキャラ名
- エラーや取得できなかったスキルがあれば一覧表示

## エラー処理

- WebFetchが失敗したIDはスキップし、最後にまとめて報告する
- 発動条件や効果値が読み取れない場合は該当フィールドを `null` にして処理を継続する
- ステップ1でIDが1件も取得できない場合は処理を中止してエラーを報告する
