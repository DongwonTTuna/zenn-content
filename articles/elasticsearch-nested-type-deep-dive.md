---
title: "Elasticsearchのnested型って何？"
emoji: "🔍"
type: "tech"
topics: ["Elasticsearch", "Lucene", "データモデリング", "検索エンジン", "パフォーマンス"]
published: false
---

## オブジェクト配列の検索で困ったことはありませんか

Elasticsearchでユーザー情報を検索していたら、こんな経験をしたことはないでしょうか。

「Alice White」というユーザーを検索したはずなのに、なぜか「John Smith」と「Alice White」の両方が入った文書がヒットしてしまう。データは正しく入っているはずなのに、検索結果がおかしい...

実はこれ、Elasticsearchの仕様によるものです。

## Elasticsearchがオブジェクト配列を平坦化する理由

以下のような文書をインデックスしてみましょう

```json
{
  "group": "fans",
  "user": [
    {
      "first": "john",
      "last": "smith"
    },
    {
      "first": "alice",
      "last": "white"
    }
  ]
}
```

驚くかもしれませんが、Elasticsearchは内部的にこのデータを次のように変換してしまいます

```json
{
  "group": "fans",
  "user.first": ["john", "alice"],
  "user.last": ["smith", "white"]
}
```

配列内のオブジェクトがバラバラになってしまいました。これで「first: alice AND last: smith」で検索すると、本来存在しない組み合わせでも文書がヒットしてしまうわけです。

なぜこんなことが起きるのかを言いますと、Elasticsearchの基盤であるLuceneが、根本的に平坦な構造しか扱えないからです。

## nested型の登場

この問題を解決するのがnested型です。

nested型を使うと、配列内の各オブジェクトを独立した「隠れた文書」として保存します。100個のオブジェクトを含む配列があれば、内部的には101個の文書（親文書1個 + 子文書100個）として管理されるイメージです。

※ これらの文書は同じLuceneセグメント内に物理的に隣接して配置されます。

## nested型の使い方

### 基本的なクエリ

通常のクエリとは少し違い、専用の`nested`クエリを使います

```json
GET /my-index/_search
{
  "query": {
    "nested": {
      "path": "user",
      "query": {
        "bool": {
          "must": [
            { "match": { "user.first": "alice" }},
            { "match": { "user.last": "white" }}
          ]
        }
      }
    }
  }
}
```

`path`パラメータでnestedフィールドを指定するのがポイントです。

### どのオブジェクトがマッチしたか知りたいとき

nested型の制限として、デフォルトではどのオブジェクトがマッチしたのか分からないという問題があります。

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

「実装」というキーワードでコメントを検索した場合、nestedクエリだけでは記事全体（すべてのコメントを含む）が返されるだけで、実際にどのコメントがマッチしたのかわかりません。

この問題を解決するのが`inner_hits`機能です。`inner_hits`を使うことで、検索条件にマッチした特定のnestedオブジェクトを正確に特定できます。

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
        "size": 5,                        // 返すオブジェクトの最大数（デフォルト：3）
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

## nested型を使うケース

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
   - nestedオブジェクトを1つ更新するだけで、文書全体を再インデックスする必要があります
   - 例：リアルタイムのコメント欄

2. **大量のオブジェクトを含む配列**
   - デフォルトでは1文書あたり10,000個が上限
   - それぞれが独立した文書として保存されるため、インデックスサイズが膨張します

3. **Kibanaでの分析が必要な場合**
   - Discoverでは検索できますが、LensやVisualizationでは使えません

## パフォーマンスへの影響

nested型は便利ですが、タダではありません

### インデックスサイズの増加

nested型では、各nestedオブジェクトが内部的に独立した文書として保存されるため、インデックスサイズも何倍も大きくなります。

```
総インデックスサイズ ≈ 親文書のサイズ × (1 + Σ(各フィールドのnested数))
```
例
- 親文書サイズ: 1KB
- nestedフィールドA: 平均10個のオブジェクト
- nestedフィールドB: 平均5個のオブジェクト

$$
\text{総サイズ} \approx 1\text{KB} \times (1 + 10 + 5) = 16\text{KB}
$$

つまり、上記の場合はNestを使うことで元の16倍のストレージが必要になります

### 更新コストの計算

```
更新コスト = 親文書の再インデックス + (nested数 × nested再インデックスコスト)
```

1000個のnestedオブジェクトを持つ文書の場合、1つのnestedオブジェクトの更新でも、$1 + 1000 = 1001$ 個の文書（親 + 1000個のnested）すべてが再インデックスされます

### その他のパフォーマンスへの影響

- クエリ速度が通常のフィールドより遅い（**ただしjoin型よりは5-10倍速い**）


## ベストプラクティス

### nested検索時にフィルタリングを行う

nested検索は重いので、まず通常のフィルタで文書を絞り込んでから実行しましょう

```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "group": "fans" } }  // まずこれで絞る
      ],
      "must": [
        {
          "nested": {
            "path": "user",
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

このオプションを有効にすると、 nested、object配列の両方の型としてインデックスされます。

これにより、nestedクエリを使わずに通常のフィールドとして検索することも可能になります。

```json
{
  "mappings": {
    "properties": {
      "user": {
        "type": "nested",
        "include_in_parent": true
      }
    }
  }
}
```

## まとめ

nested型は、Elasticsearchでオブジェクト配列の関係性を保持する強力なツールです。ただし、銀の弾丸ではありません。

更新頻度、データ量、パフォーマンス要件を総合的に判断して、本当にnested型が必要かを検討しましょう。多くの場合、データモデルを見直して非正規化する方が、シンプルで高速な解決策になることもあります。

実際のプロジェクトでnested型を採用する前に、小規模なデータでパフォーマンステストを実施することをお勧めします。思わぬボトルネックに後から気づくより、最初から適切な選択をする方が、長期的には楽になるはずです。