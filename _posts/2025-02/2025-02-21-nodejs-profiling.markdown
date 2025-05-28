---
layout: post
title: "nodejsをプロファイリングしてみる"
date: 2025-02-21 12:00:00 +0900
categories: [blog]
tags: [Nodejs]
excerpt: "Node.js + Clinic.js"
---
nodejsのデバッグは`inspect-brk`や、それを内部的に使用しているvscodeが有名ですが、プロファイリングに使われる`clinic.js`を試してみます。

**目次**
* ToC
{:toc}

### 使用してみる
まずはグローバルにインストールします。
```bash
npm install -g clinic
```
その後、以下のコマンドで分析します。
```bash
clinic docter -- node dist/app.js
```
HTMLが表示されるのでブラウザで結果を確認できます。

### 他のオプション
`clinic.js` には、以下のオプションがあります。

#### 1. **`clinic doctor`** 
一般的なパフォーマンス診断ツール。アプリケーションを監視して、ボトルネックを特定します。CPU使用率、イベントループの遅延、GCなどを確認できます。各関数の呼び出し回数や遅延を確認できるので、ボトルネックを確認しやすくなります。

#### 2. **`clinic flame`** 
Flamegraph（炎グラフ）を使用して、CPU のプロファイルを視覚化します。コマンドは`docter`を`flame`に変更するだけです。グラフが表示され、関数ごとのCPU使用率を視覚的に確認できます。

#### 3. **`clinic bubbleprof`** 
関数間の呼び出し関係を視覚化するツール。円の大きさによって関数の依存関係や、ボトルネックを確認できます。
