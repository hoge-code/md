---
layout: post
title: "GitHubでの複数アカウント登録"
date: 2025-03-16 12:00:00 +0900
categories: [blog]
tags: [ツール]
excerpt: "GitHub"
---

GitHubで複数アカウントを登録する際にいくつかの問題に直面したため、忘備録として残しておきます。基本的な内容ですが、登録したのがかなり前で、忘れており、忘れやすいない内容のため、記事に残しておきます。

**目次**
* ToC
{:toc}

### GitHubの認証方式

GitHubの認証方式には、主に2つの方法があります。

1. **HTTPでのトークン認証**
2. **SSH接続方式**

### 1. HTTPでのトークン認証

GitHubでは、ユーザー設定画面 > Developer settings から**パーソナルアクセストークン**を生成することができます。トークンを生成した後、以下のコマンドでリモートURLを変更します。

```bash
git remote set-url origin https://your-username@github.com/username/repository.git
```

`git push` 時に、通常はトークンが求められます。以前はこの方法で設定していましたが、私の環境では**2つ目以降のアカウント**でトークンが求められず、トークンの設定もできなかったため、**403エラー**が発生し続けました。

`credential-manager`を使っても設定が上手くいかず、原因がよく理解できていません。理解できたら追記して再度試みます。

### 2. SSH接続

SSH接続を使用する場合は、`git-bash` などでSSHキーを生成します。以下のコマンドでSSHキーを生成します。

```bash
ssh-keygen -t rsa -b 4096 -C "your-email@example.com"
```

生成されたSSHキーは、通常は `~/.ssh/id_rsa` として保存されますが、異なるファイル名を指定して保存することもできます。その後、GitHubのSSH設定画面で公開鍵を追加します。

#### `~/.ssh/config` の設定例

複数のGitHubアカウントを使い分けるためには、`~/.ssh/config` を設定します。以下はその設定例です。

```bash
# ~/.ssh/config の例
Host github.com-Account1
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_rsa_account1

Host github.com-Account2
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_rsa_account2
```

その後、リモートリポジトリのURLを以下のコマンドで変更します。

```bash
git remote set-url origin git@github.com-Account1:username/repository.git
```

### 3. SSH接続の確認

SSHキーが正しく設定されているか確認するためには、以下のコマンドで接続のテストができます。

```bash
ssh -T git@github.com
```

正常に接続できれば、GitHubから以下のメッセージが返ってきます。

```
Hi username! You've successfully authenticated, but GitHub does not provide shell access.
```

もしこのメッセージが表示されない場合、設定に問題がある可能性があります。

### 4. SSHキーの登録確認

`ssh-agent` を使用してSSHキーが正しく登録されているか確認します。

1. `ssh-agent` が起動していることを確認します。以下のコマンドを実行します。

   ```bash
   eval "$(ssh-agent -s)"
   ```

2. 次に、生成したSSHキーを `ssh-agent` に追加します。

   ```bash
   ssh-add ~/.ssh/id_rsa
   ```

   別名でSSHキーを生成した場合は、ファイル名を適宜変更してください。

   ```bash
   ssh-add ~/.ssh/id_rsa_github
   ```

3. `ssh-agent` に登録されているSSHキーを確認します。

   ```bash
   ssh-add -l
   ```

   登録されているキーのリストが表示されればOKです。もし何も表示されない場合は、SSHキーが正しく登録されていません。

### 5. ブランチ名の確認

初期状態で、`git push --set-upstream origin master` を実行した場合にエラーが発生することがあります。特にファイルを一切作成せずにウェブ上でリポジトリを作成した場合は、認証前にリモートブランチを手動で作成し、同じ名前のローカルブランチを作成し、pushします。

### 結論

- `git@github.com` のユーザー名は 常に `git` で、GitHubでのユーザー名は関係ありません。
- SSH接続時には、`ssh-agent` の設定が重要です。SSHキーが正しく登録されているか確認します。
- 初期のブランチ名の設定に注意します。