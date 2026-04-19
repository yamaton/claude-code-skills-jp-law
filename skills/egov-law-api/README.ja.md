# egov-law-api スキル — 解説（日本語）

このディレクトリにある `SKILL.md` は、 **Claude Code / Claude Agent SDK に読み込ませる「スキル定義ファイル」** です。同ファイルは AI エージェント（Anthropic の Claude 等）を e-Gov 法令 API v2 と接続するときの挙動指針として機能します。

本ドキュメントは `SKILL.md` の役割と設計意図を、e-Gov API に精通した読者向けに日本語で解説するものです。API そのものの仕様には踏み込まず、AI エージェント連携の観点のみを扱います。

**出典**: `SKILL.md` の記述は e-Gov 法令 API v2 の公式仕様（OpenAPI / Swagger UI: <https://laws.e-gov.go.jp/api/2/swagger-ui>）に準拠しています。

---

## 1. スキル定義ファイル (`SKILL.md`) とは

Claude Code は、ユーザーのリクエスト内容に応じて関連スキルを**自動的にコンテキストへ注入**します。判定材料は `SKILL.md` 冒頭の YAML フロントマターの `description` フィールドです。

```yaml
---
name: egov-law-api
description: "e-Gov Law API v2 (...) for programmatic access to Japanese law data.
  Covers article text retrieval, metadata search, full-text keyword search,
  revision history, attachment files (images/PDFs), point-in-time snapshots,
  and file format downloads (XML/JSON/HTML/RTF/DOCX). No authentication required."
---
```

ユーザーが「e-Gov API を使って◯◯の条文を取得して」「法令検索 API で◯◯を探して」といった発話をした瞬間、Claude は description 文と照合し、合致すればこのスキルの本文を読み込んでからアクションを計画します。開発者が `import` 相当の操作をする必要はなく、発話コンテキストがトリガーです。

スキル本文（YAML 以降の Markdown）にはエンドポイント一覧・パラメータ表・落とし穴集・エラー辞典を記載してあり、Claude はこれらを参照しながら HTTP リクエストを組み立てます。

### インストール方法（参考）

- **プロジェクトスコープ**: 対象プロジェクトの `.claude/skills/egov-law-api/` に配置
- **ユーザースコープ**: `~/.claude/skills/egov-law-api/` に配置

配置するだけで、以降その環境の Claude Code セッションでは自動トリガーの対象になります。

---

## 2. AI エージェント向けの設計ポイント

`SKILL.md` に書かれている個々のトピックは、単なる API 説明ではなく **「Claude がハマりやすい・失敗しやすい箇所の事前予防」** として選定されています。主な設計判断は次のとおりです。

### 2.1 `elm` パラメータの強調
大規模法令（例: 地方税法は 1,376 条）の全文を取得すると、LLM のコンテキスト窓を一気に圧迫し後続ターンで破綻します。そこで `SKILL.md` では**初手で `elm` による条項単位の部分抽出を案内**し、必要な粒度だけ切り出すことを既定戦略としています。

### 2.2 `law_full_text_format` の一致指示
`response_format` と `law_full_text_format` が一致しないと、法令本文が Base64 でエンコードされて返ります。これはエージェントにとって実質不可読で、追加の復号ステップと大量のトークン浪費を招きます。`SKILL.md` では両方を揃える旨を繰り返し明示しています。

### 2.3 `Misc` 法令種別が空である事実の明記
告示・訓令・通達は `law_type=Misc` として API 上に型定義はあるものの、v2 本番環境では 0 件しか返りません。これを明記しておかないと、エージェントが「告示を探せばある」と推論して空振りのリクエストを繰り返します。

### 2.4 消費改正法の 404 パターン
`公職選挙法の一部を改正する法律` のような改正実施後に「消滅」扱いとなる法令は v2 API から 404 になります。知らないと Claude は法令番号を変えて再試行し続けるため、事前に代替経路（例: NDL 日本法令索引 API）を提示して無限ループを避けています。

### 2.5 `asof` の制約（2017-04-01 以降）
時点指定は 2017-04-01 より前では 400 エラー（コード `400044`）になります。この制約を知らないまま Claude が昭和・平成初期の日付で条文取得を試みると即座にエラーで詰まるため、明記しています。

### 2.6 レート制限の非文書化
e-Gov API にはレート制限の明文規定がない一方で、バーストリクエストはサーバ負荷やタイムアウトの原因になります。大量バッチ処理には公式の XML 一括ダウンロードに誘導する記述を置いています。

---

## 3. エージェント対話例（実機検証済）

以下は Claude が本スキルを参照しながら API を呼び出す流れの具体例です。記載の URL・パラメータ・結果はいずれも実機で確認したものです（2026-04-19 時点）。

### 例 A: 特定時点の条文取得

**ユーザー**: 「労働基準法第32条を2018年4月時点でどうなっていたか見せて」

**エージェントの内部推論**:
1. `SKILL.md` の「Endpoint selection」より、条文取得は `/law_data/{id}` が適切
2. `/laws` を使って law_id を解決（`322AC0000000049`）
3. `SKILL.md`「Common parameters」「Gotchas」より:
   - `asof=2018-04-01` が 2017-04-01 以降なので有効
   - `elm=MainProvision-Article_32` で 32 条のみ抽出（LLM 窓の節約）
   - `response_format=json` と `law_full_text_format=json` を揃える

**実行されるリクエスト**:
```bash
curl 'https://laws.e-gov.go.jp/api/2/law_data/322AC0000000049?asof=2018-04-01&elm=MainProvision-Article_32&response_format=json&law_full_text_format=json'
```

**レスポンスの骨格** (抜粋):
```json
{
  "law_info": { "law_id": "322AC0000000049", ... },
  "revision_info": {
    "law_revision_id": "322AC0000000049_20170602_429AC0000000045",
    "law_title": "労働基準法",
    "amendment_enforcement_date": "2017-06-02",
    ...
  },
  "law_full_text": {
    "tag": "Article", "attr": { "Num": "32" },
    "children": [
      { "tag": "ArticleCaption", "children": ["（労働時間）"] },
      { "tag": "ArticleTitle", "children": ["第三十二条"] },
      { "tag": "Paragraph", "attr": { "Num": "1" }, ... }
    ]
  }
}
```

**エージェントの最終応答**: 第32条の見出し（労働時間）・第1項（週40時間制限）・第2項（1日8時間制限）を整形して提示。

### 例 B: 全文キーワード検索

**ユーザー**: 「『育児休業』に言及している法令を上位3件探して」

**エージェントの内部推論**:
1. `SKILL.md`「Endpoint selection」より、全文検索は `/keyword`
2. `limit=3` で上位のみ取得（本スキルの pagination ガイダンスに従う）
3. `SKILL.md`「Gotchas」の `sentence_text_size` を考慮（デフォルト 100 文字で十分なら省略）

**実行されるリクエスト**:
```bash
curl 'https://laws.e-gov.go.jp/api/2/keyword?keyword=育児休業&limit=3&response_format=json'
```

**レスポンスの骨格** (抜粋):
```json
{
  "total_count": 1912,
  "sentence_count": 3,
  "next_offset": 3,
  "items": [
    {
      "law_info": { "law_title": "健康保険法", ... },
      "sentences": [
        { "position": "caption", "text": "（<span>育児休業</span>等を終了した際の改定）" },
        { "position": "mainprovision", "text": "保険者等は、<span>育児休業</span>、介護休業等..." }
      ]
    },
    ...
  ]
}
```

**エージェントの最終応答**: 全 1,912 件の中から上位 3 件（健康保険法ほか）を法令名・マッチ位置・該当文つきで提示。後続質問に応じて `next_offset=3` で続きを取得可能。

---

## 4. 本スキルを通じた価値

上記のとおり `SKILL.md` は単なる API リファレンスではなく、 **エージェントが失敗せずに価値ある応答を返すための「事前ブリーフィング」** として機能します。同様のパターンで、自社の法令データベースや業務システムを Claude 経由で扱いたい場合も、スキル定義を整備することでゼロコードに近い統合が可能です。

導入・カスタマイズ・別ドメイン（顧問法務・コンプライアンス管理・契約レビュー等）への展開については別途ご相談ください。
