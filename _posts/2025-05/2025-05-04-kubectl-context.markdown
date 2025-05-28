---
layout: post
title: "kubectl contextの設定と確認"
date: 2025-05-04 12:00:00 +0900
categories: [blog]
tags: [k8s]
excerpt: "k8s"
---

k8sでコンテキストを切り替える動作に躓いてしまったので、`kubectl context`関連の設定と確認コマンドをまとめてみます。

**目次**
* ToC
{:toc}

---

### 1. EKSクラスターと `kubectl` の関係性

#### `eksctl`コマンドでのクラスター初期化

```bash
eksctl create cluster --name my-cluster --region ap-northeast-1 --nodes 2
```

eksctl を実行してクラスターを作成するだけでは、kubectl コンテキストの設定が自動的に反映されないので注意する必要があります。

#### `kubectl` 設定の取得と追加

eksctl create cluster コマンドでクラスターが作成された後、kubectl がそのクラスターにアクセスできるように設定するために、次のコマンドを実行します。

```bash
aws eks --region ap-northeast-1 update-kubeconfig --name my-cluster
```

`~/.kube/config` でcontext を確認できます。

#### context 切り替えと確認コマンド

```bash
kubectl config get-contexts
kubectl config use-context arn:aws:eks:ap-northeast-1:123456789012:cluster/my-cluster
```

---

### 2. Docker Desktop の K8s 初期化と設定

Docker Desktopを使うと、ローカルでk8sクラスターを構築することができます。

#### 初期化方法

Docker Desktop の設定画面で「Kubernetes を有効化（Enable Kubernetes）」をオンにします。これにより `docker-desktop` という context が自動的に `~/.kube/config` に追加されます。

#### context 切り替えと確認

```bash
kubectl config use-context docker-desktop
kubectl get nodes
```

---

### 3. `kubectl` の context 設定ファイルと管理方法

`kubectl` は `~/.kube/config` ファイルで複数のクラスタと context を管理します。

#### 主なコマンド

```bash
# context の一覧
kubectl config get-contexts

# 現在の context
kubectl config current-context

# context の切り替え
kubectl config use-context <context-name>
```

---

### 4. `istioctl` との関係性

#### `istioctl` の役割

`istio`はk8sネイティブのサービスメッシュツールなので、`istioctl`コマンドを使うと、`kubectl` と同様、現在アクティブな Kubernetes context を使って自動的にクラスタに接続します。


#### 使用例

```bash
# Istio のインストール（現在の context に対して）
istioctl install --set profile=demo -y

# Istio コンポーネントの状態確認
istioctl proxy-status

# サイドカー自動注入をネームスペースに適用
kubectl label namespace default istio-injection=enabled
```

---



