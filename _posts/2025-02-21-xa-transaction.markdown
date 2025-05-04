---
layout: post
title: "hyperターミナルを設定してみる"
date: 2025-02-21 12:00:00 +0900
categories: [blog]
tags: [ツール]
excerpt: "Hyper"
---

[Hyper](https://hyper.is/)はElectronを使ったターミナルで、プラグインを使った拡張や、tmuxのようにマルチタブが使えるらしいです。正直に言うと、AI補完やコマンド履歴を使える[WARP](https://www.warp.dev/)を使いたかったのですが、まだWindowsには対応していないのでHyperを使ってみることにしました。忘備録を兼ねて、セットアップ手順を書きます。

**目次**
* ToC
{:toc}

#### hyperをダウンロード
公式サイトにアクセスするか、chocolateyでインストールします。

#### 設定ファイルを開く
デフォルトでは以下のような画面になっています。

![default](/md/assets/images/hyper_default.png)
文字が小さく、シェルもcmd.exeのデフォルトのままなので、スタイルを整えて、git bashを導入します。設定ファイルは左上のハンバーメニューから開けますが、デフォルトではJSファイルはテキストエディタで開かれてしまうので、vscodeで開かれるように注意する必要があります。hyper.jsは自分の環境では以下のパスにありました。
`C:\Users\{username}\AppData\Roaming\Hyper\.hyper.js`

#### .hyper.jsを編集する

- **configディレクティブ**
・fontsizeを12→16に変更
・fontWeightをnormal→boldに変更

- **shell**
デフォルトの空文字からgit bashのパスに変更
この際、基本的なことですが、JSなので\をエスケープすることを忘れないようにします。

```json
shell: 'C:\\Program Files\\Git\\bin\\bash.exe',
```

- **plugins**
pluginsは詳しくないが、適当に追加してみます。

```json
    plugins: [
        'hyper-snazzy',
        'hyper-statusline',
        'hyper-search'
    ],
```

#### 編集後

以下のような見た目になりました。
![default](/md/assets/images/hyper.png)

画面分割すると以下のようになります。
![default](/md/assets/images/hyper_tmux.png)

#### 感想

設定ファイルを編集することで簡単にスタイル変更やプラグイン追加ができるのは便利だと感じました。その一方で、git bashだけでは補完が利かないのでxonshなど他のシェルを試してみたくなりました。