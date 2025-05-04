---
layout: post
title: "github actionsのファイル分割"
date: 2025-05-04 12:00:00 +0900
categories: [blog]
tags: [ツール]
excerpt: "github-actions"
---

**目次**
* ToC
{:toc}

---

最近は`GitLabCI`について学習していて、`GitHubActions`でも`GitLabCI`のように柔軟に他のyamlファイルをインポートできないか気になったので調べてみました。

### 1. 再利用可能なワークフロー

#### workflow を extends と uses で再利用する

uses キーワードを使用して、共通のアクションを呼び出すことができます。

#### 再利用可能なワークフローの例
```yaml
# .github/workflows/main.yml
jobs:
  build:
    uses: ./.github/workflows/build.yml
    with:
      some-input: "value"
```

```yaml
# .github/workflows/build.yml
name: Build
on:
  workflow_call:
    inputs:
      some-input:
        required: true
        type: string
jobs:
  build-job:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2
      - name: Run Build
        run: echo "Building the project with input: ${{ inputs.some-input }}"
```

また、`GitLabCI`と同様に別リポジトリのyamlファイルも読み込むことができます。
```yaml
jobs:
  lint:
    uses: my-org/shared-workflows/.github/workflows/lint.yml@v1
```

この方法を使うことで、build.yml のようなワークフローを複数の他のワークフローから呼び出すことができ、GitLab CI のようなファイルの分割と再利用が可能になります。

---

### 2. アクションを作成する方法

TSを使ってサードパーティーアクションも作成できますが、リポジトリを分ける必要があるので面倒です。

---

### 感想

デブオペについて体系的に学習してこなかったため、プログラムと同様にCICDファイルも分割するという発想がありませんでしたが、ファイル分割することでより柔軟にCICDを構築できると感じました。




