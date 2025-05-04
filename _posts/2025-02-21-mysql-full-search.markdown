---
layout: post
title: "MySQLで全文検索を試してみる"
date: 2025-02-21 12:00:00 +0900
categories: [blog]
tags: [MySQL]
excerpt: "MySQL + 全文検索"
---

Docker、VSCode(拡張機能はMySQLとDevContainer)、MySQLで全文検索を試してみます。RDBのLIKE検索は前方一致条件でない限りインデックスが利かないし、Elasticsearchを使ってCQRS的に実装するのは小規模なアプリだとオーバーなのでMySQLの全文検索を使った際の忘備録を残しておきます。

**目次**
* ToC
{:toc}

#### **Dockerコンテナ起動**

docker runだとコマンド入力が面倒なので、docker-compose.ymlを書きます。本当は.envファイルに環境変数を書くべきだけどベタ書きします。Docker拡張機能を入れているので、上のRun All Servicesボタンをクリックし、mysqlコンテナが起動することを確認します。

![](/md/assets/images/mysql_docker_compose.png)

#### **テーブル作成とFULLTEXTインデックスの設定**

全文検索を行うために、テーブル内の対象カラムに`FULLTEXT`インデックスを作成してテーブルを作成します。

```sql
CREATE TABLE articles (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    body TEXT NOT NULL,
    FULLTEXT(title, body) -- FULLTEXTインデックスを作成
);
```

適当にデータも挿入します。
```sql
INSERT INTO articles (title, body) VALUES
('MySQL Full-Text Search', 'This tutorial explains how to use full-text search in MySQL.'),
('Introduction to Databases', 'Databases store data and provide various query functionalities.'),
('Full-Text Search Features', 'MySQL full-text search supports natural language queries.');
```

#### クエリを試してみる

`MATCH(column_name) AGAINST('検索文字列')`を使用します。以下のように、idが1と3のデータのみが表示されます。
![](/md/assets/images/mysql_against.png)

次に、クエリのスコアを算出してランキングにしてみます。
![](/md/assets/images/mysql_ranking.png)


#### 設定ファイルを編集してみる
MySQLの全文検索では、最小文字数を指定し、その文字数以下の単語は検索対象外にできます。`Dev Containers`という拡張機能を使い、`etc/my.cnf`にアクセスし、以下のディレクティブを追加してみます。

```ini
[mysqld]
ft_min_word_len=2
```

![](/md/assets/images/mysql_ranking.png)


その後、インデックスを再構築します。
```sql
REPAIR TABLE articles QUICK;
```

#### 感想

小規模なアプリで全文検索を試したいと思った時、わざわざ全文検索エンジンを導入する必要がないので便利だと感じました。
