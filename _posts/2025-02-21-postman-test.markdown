---
layout: post
title: "postmanのテスト"
date: 2025-02-21 12:00:00 +0900
categories: [blog]
tags: [ツール]
excerpt: "Stoplight + Postman + pm + newman"
---

### Stoplight Studio で yaml をダウンロード

自分でOASを書いてもいいのですが、今回は Stoplight Studio でデフォルトの API コレクションの yaml ファイルをダウンロードします。
![](/md/assets/images/postman_export.png)

### Postman にアクセス

Postman で先ほどダウンロードした yaml をインポートし、コレクションを作成します。また、コレクションを右クリックし、モックサーバーを建ててみます。この際、モックサーバーが生成した URL を環境変数に入れます。</br>

試しに pm ライブラリを使った簡易的なテストを書いてみます。(モックレスポンスを書いていないので当然 404 になりますが・・・)
![](/md/assets/images/postman_test.png)

### newman でテスト

次に、作成したコレクションを右クリックし、json 形式でエキスポートします。また、`newman`という Postman コレクションをコマンドラインで実行するためのツールをインストールし、テストを実行してみます。

```bash
npm install -g newman
newman run ./api.postman_collection.json
```
結果は以下のように表示されます。
![](/md/assets/images/postman_newman.png)

今回は使いませんが、newmanはCICDで使うこともできます。GitHubActionsの場合、以下のようなコードを書くことになります。

```yaml
      # npmを使ってNewmanをインストール
      - name: Install Newman
        run: |
          npm install -g newman

      # Postmanコレクションと環境を指定してテストを実行
      - name: Run Postman tests
        run: |
          newman run postman_collection.json 

      # テスト結果の報告
      - name: Upload test results
        uses: actions/upload-artifact@v2
        with:
          name: postman-test-results
          path: ./newman/*.json
```

### newmanでパフォーマンステスト

PostmanのRunnerからでも行えますが、newmanを使って繰り返しテストを行い、htmlで出力するようにしてみます。
```bash
npm install newman-reporter-html
newman run ./api.postman_collection.json -n 10 --reporters cli,html --reporter-html-export result.html
```
![](/md/assets/images/postman_html.png)

高度なテストならk6やJMeterなどを使うべきですが、簡易的な負荷テストならPostmanのCollection RunnerやNewmanでCIを使って実行できそうです。

#### 感想

OASを自動生成してくれるフレームワークに頼るだけでなく、OAS関連の様々なツールに慣れていきたいと思いました。