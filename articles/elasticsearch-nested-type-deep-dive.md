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

[Nested型](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html)は、オブジェクト配列の各要素を独立した文書として保存し、要素間の関係性を保持する特殊なデータ型です。

### Nested型の仕組み

Nested型を使うと、配列内の各オブジェクトが独立した「隠れた文書」として保存されます。

例えば100個のユーザーオブジェクトを含む1つの文書をインデックスすると、101個のLucene文書が作成されます。
:::message
各Nestedオブジェクトは独立したLucene文書としてインデックスされます。
:::
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

[Nestedクエリ](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-nested-query.html)を使って検索します

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

[Elasticsearchの公式ドキュメント](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/nested)では、Nested型のパフォーマンスコストについて明確に警告しています。

> Nested documents and queries are typically expensive 
> (Nested文書とクエリは通常高コストです。)

各Nestedオブジェクトが独立した文書として保存されるため、通常のオブジェクト型のインデックスと比べるとインデックスサイズが大幅に増加します。

例えば

- 1つのプロジェクト文書に10人のメンバーがいる場合
- 実際には11個の文書（親文書1個 + メンバー10個）が作成される
- 1000個のプロジェクトなら、11,000個の文書になる

また、Nested型の問題はスペースだけではありません。

Nested型の最大の問題は更新コストです。
1つのNestedオブジェクトを更新するだけで、**文書全体**を再インデックスする必要があります。

```
更新コスト = 親文書の再インデックス + (すべてのNested数 × Nested再インデックスコスト)
```

たった1つのフィールドを更新するだけで、文書全体を再インデックスする必要があるなんて、非常に非効率的ですよね。

これらのスペースの浪費と更新コストを未然に防ぐために、ElasticsearchはNested型の使用にいくつかの制限を設けています。

- インデックス内の異なるNestedマッピングの最大数：デフォルトで50
- 1つの文書が含むことができるNestedオブジェクトの最大数に制限がある
  
長所もあれば短所もありますので、多方面から慎重に考慮・検討する必要があります。
何故ESの公式ドキュメントでNested型の使用は慎重に検討するように警告されているかがわかりますね。

### Kibanaでの制限

パフォーマンス的な問題もあれば、今度はKibanaでの制限もあります。
以下[公式ドキュメント](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/nested)から抜粋した内容です。

> Nested fields have incomplete support in Kibana. While they are visible and searchable in Discover, they cannot be used to build visualizations in Lens.
> (NestedフィールドはKibanaでの不完全なサポートしかありません。Discoverでは表示・検索可能ですが、Lensでビジュアライゼーションを構築することはできません。)

KibanaのDiscoverではNested型のフィールドは表示できますが、Lensなどのビジュアライゼーションツールでは使用できないとのことです。

もしKibanaでの分析やダッシュボード作成を重視する場合、Nested型は避けた方が良いでしょう。

## 注意点・問題点やその解消方法

### inner_hitsの活用

Nested型のクエリ検索時に、デフォルトではその条件に当てはまるドキュメント単位で返ってくるので、具体的にどのNestedオブジェクトがマッチしたのか分からないという問題があります。これを解決するのが[inner_hits](https://www.elastic.co/guide/en/elasticsearch/reference/current/inner-hits.html)機能です。

例えば、プロジェクト管理システムで「エンジニア」という役職で検索した場合

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

これにより、マッチしたメンバーの情報が`inner_hits`として返されます

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

[include_in_parent](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html)パラメータを設定すると、データがNested型とフラット構造の両方でインデックスされるため、Nestedクエリと通常のクエリの両方で検索できるようになります。

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

Nested検索は高コストなので、まず通常のフィルタで文書を絞り込んでから実行しましょう

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

## フラット構造とNested型の比較

| 観点 | フラット構造 | Nested型 |
|------|------------|----------|
| **インデックスサイズ** | 小 | 大（オブジェクト数に比例） |
| **更新性能** | 高速 | 低速（文書全体の再インデックス） |
| **クエリ精度** | 低（誤検出あり） | 高（正確な検索） |
| **クエリ速度** | 高速 | 低速 |
| **Kibanaサポート** | 完全対応 | 部分対応（Discoverのみ） |
| **向いているケース** | • 頻繁な更新<br>• 大量のオブジェクト<br>• Kibana分析が必要 | • 組み合わせ検索が必要<br>• 正確な検索精度が重要<br>• 更新頻度が低い |
| **使用例** | • チャットメッセージ<br>• アクセスログ<br>• メトリクスデータ | • プロジェクトメンバー管理<br>• 商品バリエーション<br>• イベント参加者リスト |

## まとめ

Nested型は、Elasticsearchでオブジェクト配列の関係性を保持する強力なツールです。プロジェクトメンバーの管理のように、オブジェクト内のフィールドの関連性を維持する必要がある場合には非常に有効です。

しかし、公式ドキュメントが警告するように「Nested文書とクエリは通常高コスト」であることを忘れてはいけません。

多くの場合、データモデルを見直して非正規化する方が、シンプルで高速な解決策になることもあります。

なのでもし導入を検討しているなら、慎重に要件を分析し、Nested型の使用が本当に必要かどうかを検討してください。

## 参考リンク

- [Nested field type | Elasticsearch公式ドキュメント](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html)
- [Nested query | Elasticsearch公式ドキュメント](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-nested-query.html)
- [Retrieve inner hits | Elasticsearch公式ドキュメント](https://www.elastic.co/guide/en/elasticsearch/reference/current/inner-hits.html)
- [Arrays | Elasticsearch公式ドキュメント](https://www.elastic.co/guide/en/elasticsearch/reference/current/array.html)
