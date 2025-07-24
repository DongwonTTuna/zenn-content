---
title: "ElasticsearchのNested型って何？"
emoji: "🔍"
type: "tech"
topics: ["Elasticsearch", "Lucene", "データモデリング", "検索エンジン", "パフォーマンス"]
published: false
publication_name: "nextbeat"
---

## はじめに：オブジェクト配列検索の罠

Elasticsearchを使っていると、オブジェクト配列の検索で思わぬ落とし穴にハマることがあります。

例えばプロジェクト管理システムで、以下のようなオブジェクト配列データを検索するとしてみましょう。

```json
// プロジェクトA
{
  "project": "AI開発プロジェクト",
  "members": [
    { "name": "田中", "role": "リーダー" },
    { "name": "佐藤", "role": "エンジニア" }
  ]
}

// プロジェクトB
{
  "project": "Web開発プロジェクト",
  "members": [
    { "name": "鈴木", "role": "デザイナー" },
    { "name": "田中", "role": "エンジニア" }
  ]
}
```

この時、「田中さんがエンジニアとして参加しているプロジェクト」を検索したら、当然プロジェクトBだけが返ってくると思いますよね？

ところが実際にElasticsearchで検索してみると、なぜかプロジェクトAまでヒットしてしまうんです。プロジェクトAの田中さんは「リーダー」なのに...

なぜこんなことが起きるかを説明していきたいと思います。

## オブジェクト配列の平坦化問題

### Elasticsearchの内部動作

[Elasticsearchの公式ドキュメント](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html)によると、Elasticsearchはオブジェクトの階層構造を理解しません。代わりに、オブジェクト階層をフィールド名と値のシンプルなリストに平坦化します。

先のプロジェクトAの文書をインデックスすると、内部的にこのようになります

```json
{
  "project": "AI開発プロジェクト",
  "members.name": ["田中", "佐藤"],
  "members.role": ["リーダー", "エンジニア"]
}
```

配列内の各オブジェクトがバラバラになり、フィールドごとにまとめられてしまいました。これを「平坦化（フラット化）」と呼びます。

この挙動は、Elasticsearchの基盤である[Apache Lucene](https://lucene.apache.org/)が根本的に平坦な構造しか扱えないことに起因します。

### 検索時の問題

この平坦化により、「`name: 田中` **AND** `role: エンジニア`」で検索すると
- 「田中」は確かに存在する（ただしリーダーとして）
- 「エンジニア」も確かに存在する（ただし佐藤さんの役職として）

両方の条件を満たすため、プロジェクトAがヒットしてしまうのです。

とはいえ、私たちは「田中さんがエンジニアとして参加しているプロジェクト」を探しているので、この結果は明らかに誤検出です。

では、どうすればこの問題を解決できるのでしょうか？
そこで登場するのがElasticsearchの[Nested型](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html)です。

## Nested型について

### Nested型とは

[Nested型](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html)は、オブジェクト配列の各要素の独立性を保つための特殊なデータ型です。

### Nested型の仕組み

Nested型を使うと、配列内の各オブジェクトが独立した「隠れた文書」として保存されます。公式ドキュメントによると：

> 各Nestedオブジェクトは独立したLucene文書としてインデックスされます。100個のユーザーオブジェクトを含む1つの文書をインデックスすると、101個のLucene文書が作成されます：親文書1個とNestedオブジェクト100個です。

これらの文書は同じLuceneセグメント内に物理的に隣接して配置されるため、効率的にクエリを実行できます。

### 使い方

#### マッピングの定義

```json
PUT /project-index
{
  "mappings": {
    "properties": {
      "project": { "type": "text" },
      "members": {
        "type": "nested",  // ここがポイント
        "properties": {
          "name": { "type": "text" },
          "role": { "type": "text" }
        }
      }
    }
  }
}
```

#### 基本的なクエリ

[Nestedクエリ](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-nested-query.html)を使って検索します：

```json
GET /project-index/_search
{
  "query": {
    "nested": {
      "path": "members",
      "query": {
        "bool": {
          "must": [
            { "match": { "members.name": "田中" }},
            { "match": { "members.role": "エンジニア" }}
          ]
        }
      }
    }
  }
}
```

これで、実際に「田中さんがエンジニアとして参加しているプロジェクト」だけが正確にヒットします。

## Nested型使用時の注意点・問題点

### パフォーマンスへの影響

Elasticsearchの公式ドキュメントは、Nested型のパフォーマンスコストについて明確に警告しています：

> Nested文書とクエリは通常高コストです。

#### インデックスサイズの増加

各Nestedオブジェクトが独立した文書として保存されるため、インデックスサイズが大幅に増加します。例えば：

- 1つのプロジェクト文書に10人のメンバーがいる場合
- 実際には11個の文書（親文書1個 + メンバー10個）が作成される
- 1000個のプロジェクトなら、11,000個の文書になる

#### 更新コストの問題

Nested型の最大の問題は更新コストです。1つのNestedオブジェクトを更新するだけで、**文書全体**を再インデックスする必要があります。

```
更新コスト = 親文書の再インデックス + (すべてのNested数 × Nested再インデックスコスト)
```

### Kibanaでの制限

公式ドキュメントによると：

> NestedフィールドはKibanaでの不完全なサポートしかありません。Discoverでは表示・検索可能ですが、Lensでビジュアライゼーションを構築することはできません。

### パフォーマンスのセーフガード

Elasticsearchは、パフォーマンス問題を防ぐためにいくつかの制限を設けています：

- インデックス内の異なるNestedマッピングの最大数：50（デフォルト）
- 1つの文書が含むことができるNestedオブジェクトの最大数に制限がある

## 注意点・問題点の解消方法

### inner_hitsの活用

Nested型の制限として、デフォルトではどのオブジェクトがマッチしたのか分からないという問題があります。これを解決するのが[inner_hits](https://www.elastic.co/guide/en/elasticsearch/reference/current/inner-hits.html)機能です。

例えば、プロジェクト管理システムで「エンジニア」という役職で検索した場合：

```json
GET /project-index/_search
{
  "query": {
    "nested": {
      "path": "members",
      "query": {
        "match": { "members.role": "エンジニア" }
      },
      "inner_hits": {
        "name": "matched_members",
        "size": 3,
        "_source": ["members.name", "members.role"],
        "highlight": {
          "fields": {
            "members.role": {}
          }
        }
      }
    }
  }
}
```

これにより、マッチしたメンバーの情報が`inner_hits`として返されます：

```json
{
  "hits": {
    "hits": [{
      "_source": {
        "project": "Web開発プロジェクト",
        "members": [...]
      },
      "inner_hits": {
        "matched_members": {
          "hits": [{
            "_nested": {
              "field": "members",
              "offset": 1
            },
            "_source": {
              "name": "田中",
              "role": "エンジニア"
            },
            "highlight": {
              "members.role": ["<em>エンジニア</em>"]
            }
          }]
        }
      }
    }]
  }
}
```

### include_in_parentの活用

[include_in_parent](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html)パラメータを使うと、Nestedフィールドをフラット構造でも検索できるようになります：

```json
PUT /project-index
{
  "mappings": {
    "properties": {
      "members": {
        "type": "nested",
        "include_in_parent": true,
        "properties": {
          "name": { "type": "text" },
          "role": { "type": "text" }
        }
      }
    }
  }
}
```

これにより、シンプルな検索には通常のクエリを使い、厳密な検索にはNestedクエリを使うという使い分けが可能になります。

### フィルタリングを併用

Nested検索は高コストなので、まず通常のフィルタで文書を絞り込んでから実行しましょう：

```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "project": "Web開発プロジェクト" } }  // まずプロジェクトで絞る
      ],
      "must": [
        {
          "nested": {
            "path": "members",
            "query": {
              "match": { "members.role": "エンジニア" }
            }
          }
        }
      ]
    }
  }
}
```

## フラット構造とNested型の比較検討

### Nested型が向いているケース

1. **プロジェクトメンバー管理**
   - メンバーの名前と役職を組み合わせて検索
   - 例：「田中さんがエンジニアとして参加しているプロジェクト」

2. **商品のバリエーション管理**
   - 色とサイズの正確な組み合わせ検索が必要
   - 例：`{color: "青", size: "M"}` の在庫確認

3. **更新頻度が低いデータ**
   - 一度登録したらあまり変更しないデータ
   - 例：過去のイベント参加者リスト

### フラット構造（通常のobject型）が向いているケース

1. **頻繁に更新されるデータ**
   - リアルタイムで更新が必要なデータ
   - 例：アクティブなチャットメッセージ

2. **大量のオブジェクトを含む配列**
   - 1つの文書に数百〜数千のオブジェクトがある場合
   - インデックスサイズの増大を避けたい場合

3. **Kibanaでの分析が必要な場合**
   - ダッシュボードやビジュアライゼーションを作成したい場合
   - Nested型はKibanaのLensで使用できない

### 性能比較

| 観点 | フラット構造 | Nested型 |
|------|------------|----------|
| インデックスサイズ | 小 | 大（オブジェクト数に比例） |
| 更新性能 | 高速 | 低速（文書全体の再インデックス） |
| クエリ精度 | 低（誤検出あり） | 高（正確な検索） |
| クエリ速度 | 高速 | 低速 |
| Kibanaサポート | 完全対応 | 部分対応（Discoverのみ） |

## まとめ

Nested型は、Elasticsearchでオブジェクト配列の関係性を保持する強力なツールです。プロジェクトメンバーの管理のように、オブジェクト内のフィールドの関連性を維持する必要がある場合には非常に有効です。

しかし、公式ドキュメントが警告するように「Nested文書とクエリは通常高コスト」であることを忘れてはいけません。

導入前のチェックリスト：
- [ ] 本当にオブジェクトの独立性が必要か？
- [ ] 更新頻度は低いか？
- [ ] オブジェクト数は適切か？（数個〜数十個程度）
- [ ] Kibanaでの分析は不要か？
- [ ] パフォーマンステストは実施したか？

多くの場合、データモデルを見直して非正規化する方が、シンプルで高速な解決策になることもあります。Nested型は強力ですが、適切な場面で使ってこそ真価を発揮します。

## 参考リンク

- [Nested field type | Elasticsearch公式ドキュメント](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html)
- [Nested query | Elasticsearch公式ドキュメント](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-nested-query.html)
- [Retrieve inner hits | Elasticsearch公式ドキュメント](https://www.elastic.co/guide/en/elasticsearch/reference/current/inner-hits.html)
- [Arrays | Elasticsearch公式ドキュメント](https://www.elastic.co/guide/en/elasticsearch/reference/current/array.html)
