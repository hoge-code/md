---
layout: post
title: "主要DBで全文検索を試してみる"
date: 2025-05-04 12:00:00 +0900
categories: [blog]
tags: [elasticsearch, MySQL, sqlite]
excerpt: "elasticsearch, MySQL, sqlite"
---

最近は全文検索について学習しているので主要DBごとの全文検索の特徴を簡単にまとめてみます。

**目次**
* ToC
{:toc}

全文検索とハイライトはElasticsearchではネイティブに対応していますが、MySQL、PostgreSQL、SQLiteではそれぞれ手法が異なり、制限もあります。以下にそれぞれの全文検索＋ハイライトのクエリ例と（可能な限り）レスポンス例を示します。

---

### Elasticsearch

#### クエリ例（全文検索＋ハイライト）:

```json
GET /my_index/_search
{
  "query": {
    "match": {
      "content": "Elasticsearch tutorial"
    }
  },
  "highlight": {
    "fields": {
      "content": {}
    }
  }
}
```

#### レスポンス例:

```json
{
  "hits": {
    "hits": [
      {
        "_source": {
          "content": "This is an Elasticsearch tutorial for beginners."
        },
        "highlight": {
          "content": [
            "This is an <em>Elasticsearch</em> <em>tutorial</em> for beginners."
          ]
        }
      }
    ]
  }
}
```
Elasticsearchはタイトルに対する重み付けなどにも対応しており、ハイライトも`<em>`をつけてレスポンスが返されるのでフロントで簡単にUIに反映できます。

---

### MySQL

MySQLのFULLTEXTインデックスは `MATCH ... AGAINST` 構文で全文検索ができますが、ハイライトは自前で文字列を置換して処理する必要があります。

#### クエリ例:

```sql
SELECT 
  id,
  content,
  REPLACE(content, 'Elasticsearch', '<em>Elasticsearch</em>') AS highlighted_content
FROM articles
WHERE MATCH(content) AGAINST('Elasticsearch' IN NATURAL LANGUAGE MODE);
```

MySQLはPostgreに比べて機能が少ないので、部分一致や複数キーワードには非対応です。また、ハイライトも使えないので全文検索にはあまり向いていません。

---

### PostgreSQL 

tsvector + tsquery + ts\_headlineを使います。PostgreSQLは全文検索に強く、`ts_headline`でハイライトが可能です。

#### クエリ例:

```sql
SELECT 
  id,
  ts_headline('english', content, to_tsquery('Elasticsearch')) AS highlighted_content
FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('Elasticsearch');
```

#### 出力例:

```text
This is an <b>Elasticsearch</b> tutorial for beginners.
```

---

### SQLite

SQLiteはネイティブな全文検索 (`FTS5`) をサポートしていますが、ハイライト機能は組み込み関数 `highlight()` を使う必要があります。

#### クエリ例 (FTS5):

```sql
SELECT 
  highlight(articles_fts, 0, '<em>', '</em>') AS highlighted_content
FROM articles_fts
WHERE articles_fts MATCH 'Elasticsearch';
```

※ `articles_fts` は `CREATE VIRTUAL TABLE articles_fts USING fts5(content)` で作成された仮想テーブル。

---

### 感想

最近はPostgreSQLを使うことがあるのですが、全文検索だけでなく、B-Tree以外のインデックスが豊富だったり、カスタムデータ型が作れたりと、MySQLと比べてかなり高機能であるように感じます。

また、SQLiteはファイルベースの小規模なDBですが、基本的な全文検索やハイライトが出来たりと、PCやモバイルでローカルに使うには十分な機能があるように感じます。
