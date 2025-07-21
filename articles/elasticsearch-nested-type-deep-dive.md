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

例えばElasticsearchで、以下のようなオブジェクト配列データを検索するとしてみましょう。

```json
// プロジェクトA
{
  "project": "projectA",
  "users": [
    { "first": "Alice", "last": "Smith" }, // Alice Smith
    { "first": "Bob", "last": "White" } // Bob White
  ]
}

// プロジェクトB
{
  "project": "projectB",
  "users": [
    { "first": "Carol", "last": "Johnson" }, // Carol Johnson
    { "first": "Alice", "last": "White" } // Alice White
  ]
}
```
- プロジェクトA：「Alice Smith」と「Bob White」が参加
- プロジェクトB：「Carol Johnson」と「Alice White」が参加

この時、「`Alice White`」で検索したら、当然プロジェクトB（本当にAlice Whiteさんがいる）だけが返ってくると思いますよね？

ところが実際にElasticsearchで検索してみると、なぜかプロジェクトAまでヒットしてしまうんです。プロジェクトAには「`Alice White`」という人はいないのに...

なぜこんなことが起きるのかというと、実は「`Alice`」（Alice Smithの名前）と「`White`」（Bob Whiteの苗字）が別々の人物なのに、Elasticsearchはこれらを組み合わせて「`Alice White`」として認識してしまうからです。

## Elasticsearchはオブジェクト配列を内部的に平坦化します

先のプロジェクトAの文書をインデックスしてみましょう。

```json
{
   "project": "projectA",
   "users": [
      { "first": "Alice", "last": "Smith" },
      { "first": "Bob", "last": "White" }
   ]
}
```

すると驚くことに、Elasticsearchは内部的にこのデータを次のように変換してしまいます

```json
{
  "project": "projectA",
  "users.first": ["Alice", "Bob"],
  "users.last": ["Smith", "White"]
}
```

配列内のオブジェクトがバラバラになってしまいました。

これで「`first: alice` **AND** `last: white`」で検索すると、本来存在しない「`Alice White`」という組み合わせでも文書がヒットしてしまうわけです。

なぜこんなことが起きるのかを言いますと、Elasticsearchの基盤であるLuceneが、根本的に平坦な構造しか扱えないからです。

## ここでNested型の登場

この問題を解決するのがNested型です。

Nested型を使うと、配列内の各オブジェクトを独立した「隠れた文書」として保存します。100個のオブジェクトを含む配列があれば、内部的には101個の文書（親文書1個 + 子文書100個）として管理されるイメージです。

※ これらの文書は同じLuceneセグメント内に物理的に隣接して配置されます。

## Nested型の使い方

### 基本的なクエリ

通常のクエリとは少し違い、専用の`Nested`クエリを使います

```json
GET /my-index/_search
{
  "query": {
    "nested": {
      "path": "users",
      "query": {
        "bool": {
          "must": [
            { "match": { "users.first": "alice" }},
            { "match": { "users.last": "white" }}
          ]
        }
      }
    }
  }
}
```

`path`パラメータでNestedフィールドを指定するのがポイントです。

### どのオブジェクトがマッチしたか知りたいとき

Nested型の制限として、デフォルトではどのオブジェクトがマッチしたのか分からないという問題があります。

例えば、コメント機能を持つブログシステムを考えてみましょう

```json
{
  "title": "Elasticsearchの活用法",
  "author": "山田太郎",
  "comments": [
    { "user": "Alice", "content": "とても分かりやすい記事でした！", "likes": 10 },
    { "user": "Bob", "content": "実装例があって助かりました", "likes": 5 },
    { "user": "Charlie", "content": "もっと詳しく知りたいです", "likes": 3 }
  ]
}
```

「実装」というキーワードでコメントを検索した場合、Nestedクエリだけでは記事全体（すべてのコメントを含む）が返されるだけで、実際にどのコメントがマッチしたのかわかりません。

この問題を解決するのが`inner_hits`機能です。`inner_hits`を使うことで、検索条件にマッチした特定のNestedオブジェクトを正確に特定できます。

#### 基本的な使い方

```json
GET /blog/_search
{
  "query": {
    "nested": {
      "path": "comments",
      "query": {
        "match": {
          "comments.content": "実装"
        }
      },
      "inner_hits": {} 
    }
  }
}
```

#### オプションの活用した使い方

`inner_hits`には以下のようなオプションを指定できます

```json
GET /blog/_search
{
  "query": {
    "nested": {
      "path": "comments",
      "query": {
        "match": {
          "comments.content": "実装"
        }
      },
      "inner_hits": {
        "name": "matched_comments",        // 結果に名前を付ける
        "size": 5,                        // 返すオブジェクトの最大数
        "from": 0,                        // ページング用のオフセット
        "sort": [                         // マッチしたオブジェクトの並び順
          { "comments.likes": { "order": "desc" } }
        ],
        "_source": ["comments.user", "comments.content"],  // 必要なフィールドのみ取得
        "highlight": {                    // 検索語をハイライト表示
          "fields": {
            "comments.content": {}
          }
        }
      }
    }
  }
}
```

#### 実際の検索結果

上記のクエリを実行すると、以下のような結果が返ってきます
ポイントは、どのコメントがマッチしたのか、`inner_hits`内に表示されている点です

```json
{
  "hits": {
    "hits": [
      {
        "_source": {
          "title": "Elasticsearchの活用法",
          "author": "山田太郎",
          "comments": [...]  // 全コメントが含まれる
        },
        "inner_hits": {
          "matched_comments": {
            "hits": {
              "total": { "value": 1 },
              "hits": [
                {
                  "_nested": {
                    "field": "comments",
                    "offset": 1  // 配列内のインデックス
                  },
                  "_source": {
                    "user": "Bob",
                    "content": "実装例があって助かりました"
                  },
                  "highlight": {
                    "comments.content": [
                      "<em>実装</em>例があって助かりました"
                    ]
                  }
                }
              ]
            }
          }
        }
      }
    ]
  }
}
```

### パフォーマンスへの注意点

- `max_inner_result_window`設定を超える過度なページングは避けてください。
- パフォーマンスが気になる場合は、必要最小限のフィールドのみ`_source`で指定することで、パフォーマンスを改善できます

## Nested型を使うケース

### 向いているケース

1. **商品のバリエーション管理**
   - 例：Tシャツの色とサイズの組み合わせ
   - `{color: "blue", size: "M"}` のような正確な組み合わせ検索が必要

2. **ブログのコメント**
   - 投稿者と内容をセットで検索したい
   - ただし更新頻度が低い場合

3. **イベントの参加者リスト**
   - 参加者の属性を組み合わせて検索

### 向いていないケース

1. **頻繁に更新されるデータ**
   - Nestedオブジェクトを1つ更新するだけで、文書全体を再インデックスする必要があります
   - 例：リアルタイムのコメント欄

2. **大量のオブジェクトを含む配列**
   - それぞれが独立した文書として保存されるため、インデックスサイズが膨張します

3. **Kibanaでの分析が必要な場合**
   - Discoverでは検索できますが、LensやVisualizationでは使えません

## パフォーマンスへの影響

Nested型は便利ですが、タダではありません

### インデックスサイズの増加

Nested型では、各Nestedオブジェクトが内部的に独立した文書として保存されるため、インデックスサイズも何倍も大きくなります。

:::message
以下の計算式は概念を理解するための簡略化されたモデルです。実際のディスク使用量は、圧縮、最適化、フィールドの内容、インデックス設定など多くの要因によって変動します。
:::

```
総インデックスサイズ ≈ 親文書のサイズ × (1 + Σ(各フィールドのNested数))
```
例
- 親文書サイズ: 1KB
- NestedフィールドA: 平均10個のオブジェクト
- NestedフィールドB: 平均5個のオブジェクト

$$
\text{総サイズ} \approx 1\text{KB} \times (1 + 10 + 5) = 16\text{KB}
$$

つまり、上記の場合はNestを使うことで元の16倍のストレージが必要になります

### 更新コストの計算

```
更新コスト = 親文書の再インデックス + (Nested数 × Nested再インデックスコスト)
```

1000個のNestedオブジェクトを持つ文書の場合、1つのNestedオブジェクトの更新でも、$1 + 1000 = 1001$ 個の文書（親 + 1000個のNested）すべてが再インデックスされます

:::message
実際の更新パフォーマンスは、ハードウェア、クラスタ構成、書き込み負荷、マージポリシーなど多くの要因に依存します。
:::

### その他のパフォーマンスへの影響

- クエリ速度が通常のフィールドより遅い（**ただしjoin型よりは5-10倍速い**）


## ベストプラクティス

### Nested検索時にフィルタリングを行う

Nested検索は重いので、まず通常のフィルタで文書を絞り込んでから実行しましょう

```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "project": "projectA" } }  // まずこれで絞る
      ],
      "must": [
        {
          "nested": {
            "path": "users",
            "query": { ... }
          }
        }
      ]
    }
  }
}
```

### include_in_parentを活用
`include_in_parent`は検索の柔軟性とパフォーマンスの両立を図るオプションです。

このオプションを有効にすると、 Nested、object配列の両方の型としてインデックスされます。

これにより、Nestedクエリを使わずに通常のフィールドとして検索することも可能になります。

```json
{
  "mappings": {
    "properties": {
      "users": {
        "type": "nested",
        "include_in_parent": true
      }
    }
  }
}
```

## まとめ

Nested型は、Elasticsearchでオブジェクト配列の関係性を保持する強力なツールです。ただし、銀の弾丸ではありません。

更新頻度、データ量、パフォーマンス要件を総合的に判断して、本当にNested型が必要かを検討しましょう。多くの場合、データモデルを見直して非正規化する方が、シンプルで高速な解決策になることもあります。

実際のプロジェクトでNested型を採用する前に、小規模なデータでパフォーマンステストを実施することをお勧めします。思わぬボトルネックに後から気づくより、最初から適切な選択をする方が、長期的には楽になるはずです。