---
layout: post
title: "自動補完のローカルと全文検索の比較"
date: 2025-05-04 12:00:00 +0900
categories: [blog]
tags: [elasticsearch, React]
excerpt: "elasticsearch, fuse.js"
---

検索機能などで使われるオートコンプリートをクライアント側のJSONファイルと、リモートの全文検索を実装する方法を比較して考えてみます。

**目次**
* ToC
{:toc}

---

### ローカルで実装する場合

今回は`React`を使い、オートコンプリートライブラリに`fuse.js`、UIライブラリに`MUI`を使用してみます。

#### 依存

```bash
npm install @mui/material fuse.js
```

#### 検索対象例（data.js）

```js
export const data = [
  { title: 'Apple Pie', id: 1 },
  { title: 'Banana Bread', id: 2 },
  { title: 'Cherry Tart', id: 3 }
];
```

#### Autocomplete コンポーネント

```jsx
import { Autocomplete, TextField } from '@mui/material';
import Fuse from 'fuse.js';
import { useState } from 'react';
import { data } from './data';

const fuse = new Fuse(data, {
  keys: ['title'],
  includeScore: true,
});

export default function Search() {
  const [options, setOptions] = useState([]);

  const handleInput = (e) => {
    const input = e.target.value;
    const result = fuse.search(input).map(r => r.item);
    setOptions(result);
  };

  return (
    <Autocomplete
      options={options}
      getOptionLabel={(option) => option.title}
      renderInput={(params) => (
        <TextField {...params} label="Search" onChange={handleInput} />
      )}
    />
  );
}
```

都道府県情報のようにあまり変化しないデータや、ローカルストレージに保存した検索履歴などを使用する場合にはパフォーマンス的によさそうです。その一方で、動的なデータやデバイス間でデータを共有したい場合には向いていません。

---

### Elasticsearchを使用する場合

#### API例（/api/search）

```ts
app.post('/api/search', async (req, res) => {
  const q = req.body.query;
  const result = await es.search({
    index: 'documents',
    body: {
      query: {
        match: { title: q }
      },
      highlight: {
        fields: { title: {} }
      }
    }
  });

  const hits = result.hits.hits.map(hit => ({
    id: hit._id,
    title: hit._source.title,
    highlight: hit.highlight?.title?.[0] || hit._source.title
  }));

  res.json(hits);
});
```

#### フロントエンド側

```jsx
import { Autocomplete, TextField } from '@mui/material';
import { useState } from 'react';
import axios from 'axios';

export default function Search() {
  const [options, setOptions] = useState([]);

  const handleInput = async (e) => {
    const input = e.target.value;
    const res = await axios.post('/api/search', { query: input });
    setOptions(res.data);
  };

  return (
    <Autocomplete
      options={options}
      getOptionLabel={(option) => option.title}
      renderOption={(props, option) => (
        <li {...props} dangerouslySetInnerHTML={{ __html: option.highlight }} />
      )}
      renderInput={(params) => (
        <TextField {...params} label="Search" onChange={handleInput} />
      )}
    />
  );
}
```

大規模な検索には向いていますが、入力ごとにAPIに送信していてはパフォーマンスに問題が生じてしまうので、適切なキャッシュ戦略やデバウンス処理を考える必要がありそうです。

### Elasticsearchを使用する場合2

また、`Elasticsearch`にはオートコンプリートに特化した機能が存在します。厳密な候補リストを素早く提示する場合は`completion`機能、全文検索ベースで自然な補完をしたい場合は`search_as_you_type + bool_prefix`機能で実装します。

---

#### 方法1: `completion` サジェスター（サジェスト専用）

##### インデックス設定（mapping）

```json
PUT /my_index
{
  "mappings": {
    "properties": {
      "title_suggest": {
        "type": "completion"
      }
    }
  }
}
```

##### ドキュメント登録

```json
POST /my_index/_doc/1
{
  "title": "Apple Pie",
  "title_suggest": {
    "input": ["Apple Pie", "Pie"]
  }
}
```

##### サジェストクエリ

```json
POST /my_index/_search
{ 
  "suggest": {
    "title-suggest": {
      "prefix": "app",
      "completion": {
        "field": "title_suggest"
      }
    }
  }
}
```

---

#### 方法 2: `search_as_you_type` フィールドタイプ

##### インデックス設定

```json
PUT /my_index
{
  "mappings": {
    "properties": {
      "title": {
        "type": "search_as_you_type"
      }
    }
  }
}
```

##### ドキュメント登録

```json
POST /my_index/_doc/1
{
  "title": "Banana Bread"
}
```

##### クエリ（部分一致）

```json
POST /my_index/_search
{
  "query": {
    "multi_match": {
      "query": "ban",
      "type": "bool_prefix",
      "fields": [
        "title",
        "title._2gram",
        "title._3gram"
      ]
    }
  }
}
```
---

