# 継承固有スキル情報取得

U-toolsから最新の継承固有スキル情報を取得し、`references/trainee.json` に保存する。

## 実行手順

### ステップ1: スキルID一覧の取得

`https://xn--gck1f423k.xn--1bvt37a.tools/skills/specials` を以下のプロンプトでWebFetchする:

> ページ内に存在する全スキルのリンク（href）を列挙してください。
> `/skills/` で始まるリンクを全て抽出し、IDと共に以下の形式でリストアップしてください:
> - ID, スキル名, キャラ名表示
> **900000～999999番台（継承固有）と9000000～9999999番台（進化固有）のみを対象**とし、
> 10000～99999番台（星２固有）および100000～199999番台（本体固有）は除外してください。

取得したIDを全て記録する。

### ステップ2: 各スキルの詳細取得（Playwright使用）

全IDに対して `https://xn--gck1f423k.xn--1bvt37a.tools/skills/{id}` を **Playwright ブラウザ** で開き、以下の手順で情報を取得する。

**rank_range の取得手順（Playwrightで実施）:**

1. `mcp__playwright__browser_navigate` でスキルページを開く
2. `mcp__playwright__browser_snapshot` でページ内容を取得し、発動条件エリアを確認する
3. **現在のモード判定**: 発動条件の条件ラベルテキストで判定する（以下のいずれかが表示される）
   - `チャンミ条件` → 現在は **チャンミ** の値が表示されている
   - `リグヒ条件` → 現在は **LoH** の値が表示されている
   - 補助確認: 表示されている順位マーカーの個数でも判定可能（9個 = チャンミ、12個 = LoH）
4. 現在表示されている順位範囲（min, max）を、判定したモードのキーに記録する
5. **`発動条件` 見出し内の切替ボタン** を `mcp__playwright__browser_click` でクリックする（見出しの右隣にあるボタン）
6. 再度 `mcp__playwright__browser_snapshot` でページを取得し、条件ラベルが切り替わったことを確認する
7. 切替後の条件ラベルからモードを判定し、もう一方のモード（チャンミ/LoH）のキーに順位範囲を記録する
8. `rank_range` フィールドに `{ "チャンミ": [min, max], "LoH": [min, max] }` として格納する
   - 順位範囲が存在しない（順位条件なし）スキルは `rank_range: null` とする

**その他のフィールドの取得:**

取得済みのページスナップショットから以下を読み取る:
- スキル名、キャラのフルネーム
- 発動条件: 順位割合(min/max)、走行距離%(min/max)、フェーズ、脚質
- 本人使用時の効果: 目標速度(m/s)、加速度(m/s²)、持久力回復率(%)、継続時間(秒 × 距離/1000)、クールダウン
- 継承時効果（200pt）: 目標速度、加速度、持久力回復率、継続時間(秒)、取得コスト(pt)

値が存在しない場合は `null` とする。

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

- スキルIDが `900000`～`999999` の範囲 → `"固有"`（継承固有）
- スキルIDが `9000000`～`9999999` の範囲 → `"進化固有"`
- 上記以外のID（星２固有・本体固有）は保存対象外のため、このステップには到達しない

> **補足**: 継承固有のIDは、対応する本体固有ID（100000～199999）の先頭1桁目 `1` を `9` に置き換えて得られる。
> 例: 本体固有 `123456` → 継承固有 `923456`

#### duration_multiplier フィールド

継続時間が「X秒 × レース距離/1000」形式の場合:
- `duration_base_sec` に X を格納
- `duration_multiplier` に `"distance_m / 1000"` を格納
固定秒数の場合は `duration_multiplier` を `null` にする。

#### phaseとphase_detail の判定

phaseは、"序盤"、"中盤"、"終盤"、"最終盤"、"指定なし"のいずれかとする。
| phase | 区間 |
|--------|--------|
| 序盤 | レース距離の0/6 ～ 1/6 |
| 中盤 | レース距離の1/6 ～ 4/6 |
| 終盤 | レース距離の4/6 ～ 5/6 |
| 最終盤 | レース距離の5/6 ～ 6/6 |

"最終盤"は、終盤やラストスパートという表記で記載されている場合があるが、
速度スキルの場合は"最終版"、加速スキルの場合は"終盤"として扱う

その他、詳細の条件をphase_detailに記載する。
最終版の速度スキルで、"残り400m"等の記載がある場合は、phase_detailに記載する。
終版の加速スキルで、"始めの方"もしくは"始めの方早め"等の記載がある場合は、phase_detailに"始め"もしくは"始め早め"を記載する。
phase_detailは距離に関する条件を記載するものとし、距離に関する条件ではないものは、他の項目もしくはnoteに記載する。

### ステップ4: 親進化フラグの手動補足

以下のキャラクターのエントリに限り、`parent_evolution`は`is_evolution_trigger: true`とする。
他の全キャラクターについて`parent_evolution`は `is_evolution_trigger: false`とする。

 * `ステイゴールド（通常）`
 * `[Eternal Fairytale] ラインクラフト`

`evolves_skill_id` は現時点では `null` とし、判明次第手動で更新する。

### ステップ5: 補足情報の記載

その他、既存のJSON構造に格納できなかった発動条件や効果等の補足情報は`note` フィールドに記載する。

### ステップ6: JSONの組み立て

全スキルのデータを以下のスキーマに従ったJSON配列として組み立てる:

```json
[
  {
    "id": "900271",
    "skill_name": "レッツ・アナボリック！",
    "character_name": "メジロライアン",
    "costume_name": "通常",
    "full_name": "通常ライアン",
    "category": "継承固有",
    "activation_conditions": {
      "rank_range": { "チャンミ": [6, null], "LoH": [8, null] },
      "rank_pct_min": 65,
      "rank_pct_max": 70,
      "distance_pct_min": null,
      "distance_pct_max": null,
      "phase": "終盤",
      "phase_detail": "開始直後",
      "running_styles": []
    },
    "effects": {
      "target_speed_mps": null,
      "acceleration_mps2": 0.2,
      "stamina_recovery_pct": null,
      "duration_sec": 2.4,
      "cost_pt": 200
    },
    "parent_evolution": {
      "is_evolution_trigger": false,
      "evolves_skill_id": null
    },
    "note": null
  }
]
```

### ステップ7: ファイル保存

組み立てたJSON配列を `references/trainee.json` に書き込む。
既存の内容は全て上書きする。

保存後、以下を報告する:
- 保存したスキル件数（継承固有 X 件、進化固有 Y 件）
- 親進化フラグを設定したキャラ名
- エラーや取得できなかったスキルがあれば一覧表示

## エラー処理

- WebFetchが失敗したIDはスキップし、最後にまとめて報告する
- 発動条件や効果値が読み取れない場合は該当フィールドを `null` にして処理を継続する
- ステップ1でIDが1件も取得できない場合は処理を中止してエラーを報告する
