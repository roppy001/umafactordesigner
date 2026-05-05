# ウマ娘因子設計ツール — 要件定義

## プロジェクト概要

ウマ娘の因子設計を支援するための3つのスキルで構成されるツール群。
指定のレース・距離・脚質に対して、最適な親・祖父母の構成を推奨することを目的とする。

---

## ディレクトリ構造

```
umafactordesigner/
├── CLAUDE.md                  # 本要件定義
├── references/
│   ├── trainee.json           # 継承固有スキル情報（スキル1が生成）
│   └── racetrack.json         # レース場・コース情報（スキル2が生成）
└── output/
    └── {レース場}_{路面}_{距離}m.md   # 因子構成推奨結果（スキル3が生成）
```

### `references/trainee.json` スキーマ

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
      "phase_detail": null,
      "running_styles": []
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

#### フィールド定義

| フィールド | 型 | 説明 |
|-----------|---|------|
| `id` | string | U-toolsのスキルID |
| `skill_name` | string | スキル名 |
| `character_name` | string | キャラ名（衣装なし） |
| `costume_name` | string | 衣装名（通常/コスチューム名など） |
| `full_name` | string | U-tools表示名（`[衣装名] キャラ名` 形式） |
| `category` | string | `"継承固有"` または `"進化固有"` |
| `activation_conditions` | object | 発動条件 |
| `inherited_effects` | object | 継承時の効果 |
| `parent_evolution.is_evolution_trigger` | boolean | 親として使用した際に固有進化を引き起こすか |
| `parent_evolution.evolves_skill_id` | string\|null | 進化先スキルのID |
| `parent_evolution.note` | string\|null | 手動補足メモ |

### categoryの定義

| category | スキルID範囲 | 説明 |
|---------|------------|------|
| 星２固有 | 10000～99999 | 星１、星２キャラの固有スキル（継承不可） |
| 本体固有 | 100000～199999 | 各キャラの固有スキル（継承不可） |
| 継承固有 | 900000～999999 | 他キャラで継承したときの固有スキル |
| 進化固有 | 9000000～9999999 | 継承固有が親の時に進化可能な進化版固有スキル |

#### phaseとphase_detail の定義

phaseは、"序盤"、"中盤"、"終盤"、"最終盤"、"指定なし"のいずれかとする。
| phase | 区間 | 別名 |
|--------|--------|--------|
| 序盤 | レース距離の0/6 ～ 1/6 | early |
| 中盤 | レース距離の1/6 ～ 4/6 | middle |
| 終盤 | レース距離の4/6 ～ 5/6 | late |
| 最終盤 | レース距離の5/6 ～ 6/6 | last_sprint |
| 指定なし | レース距離の0/6 ～ 6/6 | - |

終盤の`"phase_detail" : "始め"`とは、終盤区間の前半1/2でのランダム発動を意味し、レース距離の16/24 ～ 18/24区間を表す
終盤の`"phase_detail" : "始め早め"`とは、終盤区間の前半1/4でのランダム発動を意味し、レース距離の16/24 ～ 17/24区間を表す

### `references/racetrack.json` スキーマ

```json
[
  {
    "course_id": "10501",
    "venue": "中山",
    "distance_m": 1200,
    "surface": "芝",
    "direction": "右",
    "distance_category": "短距離",
    "phases": {
      "early":         { "start_m": 0,    "end_m": 200  },
      "middle":        { "start_m": 200,  "end_m": 800  },
      "late":          { "start_m": 800,  "end_m": 1000 },
      "last_sprint":   { "start_m": 1000, "end_m": 1200 },
      "position_keep": { "start_m": 0,    "end_m": 500  }
    },
    "slopes": [
      { "type": "downhill", "start_m": 0,    "end_m": 200,  "gradient_pct": -1.5 },
      { "type": "uphill",   "start_m": 1025, "end_m": 1135, "gradient_pct": 2.0  }
    ],
    "corners": [
      { "number": 3, "start_m": 300, "end_m": 650 },
      { "number": 4, "start_m": 650, "end_m": 890 }
    ],
    "straights": [
      { "name": "直線1", "start_m": 890, "end_m": 1200, "length_m": 310 }
    ],
    "skill_aptitudes": ["右回り", "根幹距離", "中山レース場"]
  }
]
```

#### フィールド定義

| フィールド | 型 | 説明 |
|-----------|---|------|
| `course_id` | string | U-toolsのコースID |
| `venue` | string | レース場名 |
| `distance_m` | number | 距離（メートル） |
| `surface` | string | `"芝"` または `"ダート"` |
| `direction` | string | `"右"` または `"左"` |
| `distance_category` | string | `"短距離"` / `"マイル"` / `"中距離"` / `"長距離"` |
| `phases` | object | 各フェーズの開始・終了地点（m） |
| `slopes` | array | 坂情報（type: `"uphill"` / `"downhill"`、勾配%） |
| `corners` | array | コーナー番号と区間（m） |
| `straights` | array | 直線区間と距離（m） |
| `skill_aptitudes` | array | 適正スキル名一覧 |

---

## スキル1: 継承固有スキル情報取得

### 概要

U-toolsから最新の継承固有スキル情報を取得し、`references/trainee.json` に保存する。

### データソース

- URL: `https://xn--gck1f423k.xn--1bvt37a.tools/skills/specials`
- 各スキル詳細: `https://xn--gck1f423k.xn--1bvt37a.tools/skills/{id}`

### 取得対象

以下のカテゴリ表のうち、継承固有、進化固有のスキルを取得対象とする。
星２固有、本体固有はtrainee.jsonに保存しないようにする。

| カテゴリ | スキルID範囲 | 説明 |
|---------|------------|------|
| 星２固有 | 10000～99999 | 星１、星２キャラの固有スキル（継承不可） |
| 本体固有 | 100000～199999 | 各キャラの固有スキル（継承不可） |
| 継承固有 | 900000～999999 | 他キャラで継承したときの固有スキル |
| 進化固有 | 9000000～9999999 | 継承固有が親の時に進化可能な進化版固有スキル |

#### phaseとphase_detail の判定

phaseは、"序盤"、"中盤"、"終盤"、"最終盤"、"指定なし"のいずれかとする。
| phase | 区間 |
|--------|--------|
| 序盤 | レース距離の0/6 ～ 1/6 |
| 中盤 | レース距離の1/6 ～ 4/6 |
| 終盤 | レース距離の4/6 ～ 5/6 |
| 最終盤 | レース距離の5/6 ～ 6/6 |

終盤の"始め"とは、終盤区間の前半1/2でのランダム発動を意味し、レース距離の16/24 ～ 18/24区間を表す
終盤の"始め早め"とは、終盤区間の前半1/4でのランダム発動を意味し、レース距離の16/24 ～ 17/24区間を表す

---

## スキル2: レース場情報取得

### 概要

U-toolsから全107コースのレース場情報を取得し、`references/racetrack.json` に保存する。

### データソース

- コース一覧: `https://xn--gck1f423k.xn--1bvt37a.tools/race/tracks`
- 各コース詳細: `https://xn--gck1f423k.xn--1bvt37a.tools/race/courses/{id}`

### 取得対象コース

全107コース（以下を含む）:

| 区分 | レース場 |
|-----|---------|
| 国内 | 札幌・函館・新潟・福島・中山・東京・中京・京都・阪神・小倉・盛岡 |
| 地方 | 大井・川崎・船橋 |
| 海外 | ロンシャン・サンタアニタパーク・デルマー |


---

## スキル3: 因子構成推奨

### 概要

レース場・路面・距離を入力として受け取り、逃げ・先行・差し・追込の4脚質それぞれについて
推奨する親・祖父母の構成をMarkdown形式でファイルに出力する。

### 入力

| 項目 | 例 | 必須 |
|-----|---|------|
| レース場 | `東京` | ✅ |
| 路面 | `芝` | ✅ |
| 距離（m） | `2400` | ✅ |
| レースモード | `チャンミ` または `LoH` | ✗（デフォルト: `チャンミ`） |
| ユーザー制約条件 | `親は加速持ちにしたい` | ✗（任意） |

#### レースモードと頭数

| モード | 頭数 | 説明 |
|-------|-----|------|
| `チャンミ` | 9頭 | チャンピオンズミーティング（デフォルト） |
| `LoH` | 12頭 | リーグオブヒーローズ |

`rank_range` は `チャンミ`（9頭）と `LoH`（12頭）の2種類を事前に格納済み。推奨ロジックではレースモードに対応するキーを直接参照する。
`rank_range` が `null` のスキルは順位条件なし（除外しない）。

### 推奨ロジック

#### 基本（自動推奨）

`references/trainee.json` と `references/racetrack.json` のデータを組み合わせ、
以下の観点でコース特性にマッチするキャラを優先推奨する:

1. **フェーズ一致**: スキルの発動フェーズがコースのキーフェーズ（終盤・ラストスパート）と一致
2. **距離%一致**: スキルの発動距離%がコースの主要フェーズ区間と重なる
3. **方向一致**: 右回り/左回り適正スキルとコース方向が一致
4. **坂一致**: 上り坂・下り坂スキルとコースの坂位置が一致
5. **脚質一致**: 発動条件に含まれる脚質指定がターゲット脚質と一致

#### ユーザー制約条件（追加フィルター）

ユーザーが自由記述で追加条件を指定できる。指定がある場合は、基本推奨結果をさらに絞り込む。

例:
- `「親は加速持ちにしたい」` → `effects.acceleration_mps2 != null` のキャラを優先
- `「スタミナ回復スキル持ちを入れたい」` → `effects.stamina_recovery_pct != null` のキャラを優先

### 出力

#### ファイルパス

```
output/{レース場}_{路面}_{距離}m_{race_mode}.md
```

例: `output/東京_芝_2400m_チャンミ.md` / `output/東京_芝_2400m_LoH.md`

#### Markdown形式

```markdown
# 東京 芝 2400m（チャンミ・9頭） — 因子構成推奨

> 生成日時: 2026-05-04

## 逃げ

| 役割 | 推奨キャラ | 根拠スキル |
|-----|-----------|-----------|
| 親1 | [衣装名] キャラ名 | スキル名 |
| 祖父母11 | [衣装名] キャラ名 | スキル名 |
| 祖父母12 | [衣装名] キャラ名 | スキル名 |
| 親2 | [衣装名] キャラ名 | スキル名 |
| 祖父母21 | [衣装名] キャラ名 | スキル名 |
| 祖父母22 | [衣装名] キャラ名 | スキル名 |

## 先行

（同形式）

## 差し

（同形式）

## 追込

（同形式）
```

---

## 実装上の注意

- **キャラ名の表記**: 衣装違いが識別できるよう `full_name`（例: `[Eternal Fairytale] ラインクラフト`）を使用する
- **親進化フラグ**: `parent_evolution.is_evolution_trigger = true` のキャラ（ステイゴールド、`[Eternal Fairytale] ラインクラフト`）は推奨時に進化トリガーである旨を注記する
- **データ鮮度**: スキル1・スキル2は定期的に再実行してデータを最新化する
- **エラー処理**: 指定コースが `racetrack.json` に存在しない場合は候補コース一覧を提示する
