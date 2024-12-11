---
title: "Browserslistの使い方とコツ"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "Browserslist", "フロントエンド", "ブラウザ互換性" ]
published: false
publication_name: "nextbeat"
---

## はじめに

ウェブ開発において様々なブラウザやバージョンを考慮することは重要です。
しかし、すべてのブラウザに対応するコードを書くのは手間がかかります。
この記事では、ブラウザ互換性を効率的に管理するためのツール「[Browserslist](https://github.com/browserslist/browserslist)」について紹介します。

## [Browserslist](https://github.com/browserslist/browserslist)とは？

Browserslistは、ターゲットとなるブラウザやそのバージョンを定義するためのツールです。
ESLint、Autoprefixer、PostCSS、StyleLintなどの様々なフロントエンドツールと連携して、指定されたブラウザ向けの最適化を自動で行います。

### 主な特徴

- 柔軟なクエリ構文でブラウザの対象範囲を指定。
- 複数のツールとの統合が可能。
- 共有設定によるプロジェクト間の一貫性維持。

## Browserslistの設定方法

### 1. `package.json`に設定を追加

`package.json`ファイルにbrowserslistフィールドを追加して設定します。

```json
{
  "browserslist": [
    "> 1%",
    "last 2 versions",
    "not dead"
  ]
}
```

### 2. `.browserslistrc`ファイルを使用

プロジェクトのルートに`.browserslistrc`ファイルを作成して設定を記述します。

```
> 1%
last 2 versions
not dead
```

### 3. 設定ファイルからの読み込み

別のパッケージからエクスポートされたBrowserslist設定を参照できます。

```json
{
  "browserslist": [
    "extends browserslist-config-mycompany"
  ]
}
```

このメソッドは安全性を重視しており、`browserslist-config-`接頭辞を含むパッケージのみ参照可能です。
npmスコープのパッケージも、`@scope/browserslist-config`のように命名することで参照できます。

例:

```json
{
  "browserslist": [
    "extends browserslist-config-mycompany/desktop",
    "extends browserslist-config-mycompany/mobile"
  ]
}
```

共有可能なBrowserslist設定ファイルを作成するには、下記のように配列をエクスポートすればOKです。

`browserslist-config-mycompany/index.js`

```js
module.exports = [
  'last 1 version',
  '> 1%',
  'not dead'
];
```

### 4. 環境ごとの設定

さらに環境に対応した設定も可能です。Browserslistは`BROWSERSLIST_ENV`や`NODE_ENV`の変数によって、適切なクエリを選択します。

`package.json`の例:

```json

{
  "browserslist": {
    "production": [
      "> 1%",
      "not dead"
    ],
    "modern": [
      "last 1 chrome version",
      "last 1 firefox version"
    ],
    "ssr": [
      "node 12"
    ]
  }
}
```

`.browserslistrc`の例:

```
[production]
> 1%
not dead

[modern]
last 1 chrome version
last 1 firefox version

[ssr]
node 12
```

### 5. CLIでの確認

設定が正しく機能しているかを確認するには、パッケージ下で以下のコマンドを実行すれば見れます。

```
npx browserslist
```

## 注意点

- last ... versionsを使用する場合は、以下のようなブラウザーを除くために`not dead`を追加することを推奨します。
  - サポートが終了したブラウザ
  - 長期に渡りアップデートされていないブラウザ
- [Opera Mini](https://ja.wikipedia.org/wiki/Opera_Mini)を除外することを推奨します。
  - Opera Miniは、HTMLレンダリングエンジンを搭載していないため、対応したら逆にその他のブラウザーで互換性の問題が発生する可能性があります。

## まとめ

Browserslistは現代のウェブ開発において欠かせないツールです。
効率的かつ正確にブラウザ互換性を管理することで、開発の手間を削減し、クリーンなコードを保つことができます。
ぜひプロジェクトで活用してみてください！
