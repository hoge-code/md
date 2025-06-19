---
layout: post
title: "「GitOpsクックブック」を読む"
date: 2025-06-16 12:00:00 +0900
categories: [blog]
tags: [書籍]
excerpt: "GitOps"
memo: "3, 6-8"
---

オライリーの`GitOpsクックブック`という書籍を読んだので、気になったことを中心に簡単な感想や要約を書きます。(網羅性はありません)

**目次**

- ToC
{:toc}

---

**サイト**

- <a href="https://www.oreilly.com/library/view/gitops/9798341635609/">オライリー</a>

---

### 1.はじめに
<!--
#### 1.1.GitOpsとは何か？

#### 1.2.なぜGitOpsなのか？

#### 1.3.Kubernetes CI/CD

#### 1.4.Kubernetes上でGitOpsを使ってアプリをデプロイする

#### 1.5.DevOpsとアジャイル
-->


### 2.必要条件

#### 2.1.コンテナレジストリへの登録

- docker.ioのアカウントを作成
- それ以外だとquay.ioが有名なレジストリサービス

#### 2.2.Git リポジトリに登録する

#### 2.3.ローカルKubernetesクラスタを作成する


### 3.コンテナ
<!--
#### 3.1.Dockerを使ってコンテナを構築する

#### 3.2.Dockerless Jibを使ってコンテナを構築する

#### 3.3.Buildahを使ってコンテナを作る

#### 3.4.ビルドパックでコンテナを構築する

#### 3.5.KubernetesでShipwrightとkanikoを使ってコンテナを構築する

#### 3.6.最終的な感想
-->


### 4.Kustomize

- 最初のYAMLの開発が重要
- その後は小規模な変更
    - イメージタグのversion, レプリカ数、設定値の更新など
- KustomizeはConfigMapが変更された際、ローリングアップデートを自動で実行

#### 4.1.KubernetesリソースのデプロイにKustomizeを使う

- `kustomize.yaml`を作成
- nameは以下のように書く

```yaml
metadata:
  labels:
    app.kubernetes.io/name: pacman-kikd
```
- `kustomize.yaml`サンプル

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization 
resources: 
- ./namespace.yaml
- ./deployment.yaml
- ./service.yaml
```
```bash
kubectl apply --dry-run=client -o yaml \ 
              -k ./ 
```
- -kオプションはkustomization yamlを使用
- ベースフォルダを作成し、別のフォルダの`kustomization.yaml`が参照する例

```plaintext
 ├── base
 │   ├── kustomization.yaml
 │   └── deployment.yaml
 ├── kustomization.yaml
 ├── configmap.yaml
```
- ベースディレクトリ

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ./deployment.yaml
```
- ルートディレクトリ

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ./base
- ./configmap.yaml
```
- ルートのファイルを適用する
- URLを指定できる

```yaml
resources:
- github.com/lordofthejars/mysql 
- github.com/lordofthejars/mysql?ref=test
```
- 上記は、ブランチtestのルートのyamlファイルを参照
- kubectl以外では、`kustomize`を使用できる
    - `kustomize build . | kubectl apply -f -`

#### 4.2.Kustomize でコンテナイメージを更新する

- deploymentファイルでイメージを更新する
- バグ修正、新機能追加後に、新しいバージョンにアップデートする場面
- registry/username/project:tag

```yaml
spec:
    containers:
        - image: lordofthejars/pacman-kikd:1.0.0 
        imagePullPolicy: Always
        name: pacman-kikd
```
- `kustomization.yaml`のimagesセクションを使用して更新する

```yaml
 apiVersion: kustomize.config.k8s.io/v1beta1
 kind: Kustomization
 resources:
 - ./namespace.yaml
 - ./deployment.yaml
 - ./service.yaml
 images: 
 - name: lordofthejars/pacman-kikd 
   newTag: 1.0.1
```
- その後、`kustomize build`
- 元の`deployment.yaml`は更新されない
- 以下のコマンドでも可能
    - `kustomize edit set image lordofthejars/pacman-kikd:1.0.2`

#### 4.3.Kustomizeで任意のKubernetesフィールドを更新する

- レプリカ数などを更新
- JSONで指定するには、`patches`セクションを使用
    - 注釈、ラベル、制限などを追加
- Deploymentのレプリカ数を3に変更する例

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
 - ./deployment.yaml
patches: 
  - target: 
      version: v1
      group: apps
      kind: Deployment
      name: pacman-kikd
      namespace: pacman
    patch: |- 
      - op: replace 
        path: /spec/replicas 
        value: 3 
```
- `replicas`フィールドも使用できる

```yaml
replicas:
- name: pacman-kikd 
  count: 3 
resources:
- deployment.yaml
```
- 値の変更だけでなく、追加、削除もできる

```yaml
 patch: |
      - op: add 
        path: /metadata/labels/testkey 
        value: testvalue 
```
- 外部のパッチファイルを参照できる

```yaml
patches:
  - target:
    ...
    path: external_patch.yaml
```
- SMP(strategic merge patch)で変更できる
    - 完成したYAMLにマージされる不完全なYAMLファイルのこと

#### 4.4.複数の環境にデプロイする

- Kustomizeを使用し、同じアプリを異なる名前空間にデプロイ
- namespaceフィールドを使用し、設定
    - 一個の名前空間をdev, 別のをprodなど
    - ベースファイルは同じで、名前空間、設定パラメータ、コンテナバージョンなどが違うだけ

```plaintext
 ├── base 
│  ├── deployment.yaml
 │  └── kustomization.yaml
 ├── production
 │   └── kustomization.yaml 
└── staging
    └── kustomization.yaml 
```
- devのkustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
 - ../base 
namespace: staging 
images:
- name: lordofthejars/pacman-kikd
  newTag: 1.2.0-beta
```
- 全てのリソースと参照の名前の前後に値を追加できる
- 環境やバージョンで名前を変更する場合に使う

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
 - ../base
namespace: staging
namePrefix: staging- 
nameSuffix: -v1-2-0 
images:
- name: lordofthejars/pacman-kikd
  newTag: 1.2.0-beta
```
- 以下のように出力される
    - `name: staging-pacman-kikd-v1-2-0`

#### 4.5.Kustomize で ConfigMap を生成する

- `ConfigMapGenerator`機能フィールドを使⽤
- あるいは、`ConfigMap`宣言
- これらはメタデータ名にハッシュを自動的に追加
- 設定ファイルの変更は、deploymentにも反映される
- 環境ごとに異なる設定ファイルがある場合に便利
- deployment.yaml

```yaml
    spec:
      containers:
      - image: lordofthejars/pacman-kikd:1.0.0
        imagePullPolicy: Always
        name: pacman-kikd
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        configMap:
          name: pacman-configmap
```
- kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ./deployment.yaml
configMapGenerator:
- name: pacman-configmap 
   literals:
   - db-timeout=2000 
   - db-username=Ada
```
- 結果：`name: pacman-configmap-96kb69b6t4`
- マージ戦略もサポート

```yaml
configMapGenerator:
- name: pacman-configmap
  behavior: merge 
  literals:
  - db-username=Alexandra
```
- `.properties`ファイルもサポート

```yaml
configMapGenerator:- name: 
pacman-configmap
 files: 
- ./connection.properties
```
- kustomizeではsecretsも簡単に扱える
- secretsを扱う簡単な方法はsealed secretsを使うこと

#### 4.6.最終的な感想


### 5.ヘルム

- バージョン管理可能、デプロイ可能
- goテンプレートを使用
- kustomizeとの一番の違いはチャート
  - 他チャートへの依存関係など
- helmはConfigMapが変更された際にローリングアップデートを自動で実行

#### 5.1.Helmプロジェクトを作成する

- インストールの複雑さや簡単なアップグレードに対処
- goテンプレート言語とSprigライブラリ
- deploymentファイル

```yaml
apiVersion: apps/v1
 kind: Deployment
 metadata:
  name: {{ .Chart.Name}} 
  labels:
    app.kubernetes.io/name: {{ .Chart.Name}}
    {{- if .Chart.AppVersion }} 
    app.kubernetes.io/version: {{ .Chart.AppVersion | quote }} 
    {{- end }}
 spec:
  replicas: {{ .Values.replicaCount }} 
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ .Chart.Name}}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ .Chart.Name}}
    spec:
      containers:
          - image: "{{ .Values.image.repository }}:
          {{ .Values.image.tag | default .Chart.AppVersion}}"  
            imagePullPolicy: {{ .Values.image.pullPolicy }} 
            securityContext:
              {{- toYaml .Values.securityContext | nindent 14 }}
            name: {{ .Chart.Name}}
            ports:
              - containerPort: {{ .Values.image.containerPort }}
                name: http
                protocol: TCP
```

- serviceファイル

```yaml
apiVersion: v1
 kind: Service
 metadata:
  labels:
    app.kubernetes.io/name: {{ .Chart.Name }}
  name: {{ .Chart.Name }}
 spec:
  ports:
    - name: http
      port: {{ .Values.image.containerPort }}
      targetPort: {{ .Values.image.containerPort }}
  selector:
    app.kubernetes.io/name: {{ .Chart.Name }}
```

- デフォルト値の設定

```yaml
 image: 
  repository: quay.io/gitops-cookbook/pacman-kikd 
  tag: "1.0.0"
  pullPolicy: Always
  containerPort: 8080
 replicaCount: 1
 securityContext:
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000
```
- ディレクトリの例

```plaintext
 pacman
    ├── Chart.yaml 
    ├── templates 
    │   ├── deployment.yaml 
    │   ├── service.yaml
    └── values.yaml
```
- ローカルでYAMLにレンダリング
  - `helm template .`
- デフォルト値の上書き
  - `helm template  --set replicaCount=3 .`
- デプロイ
  - `helm install pacman .`
- その後、`kubectl get pods, deploy`
- 履歴
  - `helm history pacman`
- 削除
  - `helm uninstall pacman`
- インストールプロセスで、依存チャートがインストールされる

#### 5.2.テンプレート間でinclude文で再利用する

- 複数のファイルでテンプレートステートメントを再利用したい
- 例えば、セレクターとして新しいラベルを追加する場合
  - 複数個所で更新が必要

- 以下ファイルを作成

```yaml
{{- define "pacman.selectorLabels" -}} 
app.kubernetes.io/name: {{ .Chart.Name}} 
{{- end }}
```

- includeキーワードを利用

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "pacman.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "pacman.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        # コンテナの定義はここに追加
```
- ローカルで実行
  - `helm template .`

#### 5.3.Helmでコンテナイメージを更新する

- helmでdeploymentファイルからコンテナイメージを更新
- values.yaml更新後、以下コマンドを実行
  - `helm upgrade pacman .`
- 以前のリビジョンにロールバックできる
  - `helm rollback packman 1`
- 新しいファイルをオーバーライドとして設定できる
  - `helm template pacman -f newvalues.yaml .`

#### 5.4.ヘルムチャートのパッケージ化と配布

- パッケージ化
  - `helm package .`
- 生成されるフォルダ

```plaintext
repo
 ├── index.yaml
 ├── pacman-0.1.0.tgz
```
- 署名ファイルを生成

```bash
helm package --sign --key 'me@example.com' \
  --keyring /home/me/.gnupg/secring.gpg  .
```
- xxx.tgzとxxx.tgz.provが生成される
  - これもGitHubにアップロード
- チャートの有効性を確認
  - `helm verify pacman-0.1.0.tgz`

#### 5.5.リポジトリからチャートをデプロイする

- 登録済みリポジトリ一覧
  - `helm repo list`
- `values.yaml`の確認
  - `helm show values bitnami/postgresql`

#### 5.6.依存関係を持つグラフをデプロイする

- 別のチャートの依存関係のHelm Chartをデプロイ
- 例：ウェブアプリで、DB, キャッシュなどの依存関係

```yaml
dependencies: 
  - name: postgresql 
    version: 10.16.2 
    repository: "https://charts.bitnami.com/bitnami"
```
- また、以下の項目を追加

```yaml
postgresql: 
 server: jdbc:postgresql://music-db-postgresql:5432/mydb
 postgresqlUsername: my-default
 secretName: music-db-postgresql
 secretKey: postgresql-password
```
- 依存関係をDLし、ディレクトリに保存
  - `helm dependency update`

```plaintext
music
 ├── Chart.lock
 ├── Chart.yaml
 ├── charts
 │    └── postgresql-10.16.2.tgz
 ...
```

#### 5.7.ローリングアップデートを自動的に起動する

- ConfigMap変更時に、deploymentのローリングアップデートをトリガー
- sha256sumテンプレート関数を使用し、deploymentに変更を生成
- KustomizeにはConfigMapGeneratorという機能 
  - 新しいハッシュでdeploymentを直接更新
- Helmの場合、任意のファイルのSHA256ハッシュを計算し、テンプレートに埋め込むテンプレート関数
- ファイルのSHA256値を計算し、アノテーションに設定
  - 変更後にローリングアップデートがトリガーされる

```yaml
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ .Chart.Name}}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

#### 5.8.最終的な感想

- 単純な変更のみが必要な場合はKustomizeを使用


### 6.クラウドネイティブCI/CD
<!--
#### 6.1.Tektonをインストールする

#### 6.2.Hello Worldタスクの作成

#### 6.3.Git からアプリをコンパイルしてパッケージ化するタスクを作成する

#### 6.4.プライベート Git からアプリをコンパイルしてパッケージ化するタスクを作成する

#### 6.5.TektonタスクとBuildahを使ってアプリケーションをコンテナ化する

#### 6.6.Tektonタスクを使ってKubernetesにアプリケーションをデプロイする

#### 6.7.アプリをビルドしてKubernetesにデプロイするTektonパイプラインを作成する

#### 6.8.Tektonトリガーを使って、Git上で変更があったときに自動的にアプリケーションをコンパイルしてパッケージ化する

#### 6.9.Kustomizeを使ってKubernetesリソースを更新し、変更をGitにプッシュする

#### 6.10.Helmを使ってKubernetesリソースを更新し、プルリクエストを作成する

#### 6.11.Droneを使ってKubernetes用のパイプラインを作成する

#### 6.12.GitHub Actions を CI に使う
-->


### 7.アルゴCD

- ArgoCDはkustomize, helmもサポート
- ローリングアップデートをスムーズにする
- いつ、どのようにデプロイするかなどを効率的にサポート

#### 7.1.Argo CDを使ってアプリケーションをデプロイする

- argocd名前空間を作成し、ArgoCDをインストール

```bash
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/v2.3.4/manifests/install.yaml
```
- 必須ではない
  - Argo CD CLIツールのインストール
  - Argo CDダッシュボードにアクセスするためのArgo CDサーバーサービスの公開

- argocd-server Kubernetes Serviceを公開する必要
  - ここでは、サービスを公開せずにAPIサーバに接続
  - Ingress, LoadBalancerとしての設定など

```bash
kubectl port-forward svc/argocd-server -n argocd 9090:443
```
- http://localhost:9090を使⽤してArgo CD サーバーにアクセス
- 初期パスワード

```bash
argoPass=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" 

argoURL=localhost:9090
 
argocd login --insecure --grpc-web $argoURL  --username admin --password $argoPass
```
- Applicationリソースの作成

```yaml
 apiVersion: argoproj.io/v1alpha1
 kind: Application
 metadata:
  name: bgd-app
  namespace: argocd 
spec:
  destination: 
    namespace: bgd
    server: https://kubernetes.default.svc 
  project: default 
  source:
    repoURL: https://github.com/gitops-cookbook/gitops-cookbook-sc.git 
    path: ch07/bgd 
    targetRevision: main
```
- apply後、`argocd`あるいはUIでステータスを確認できる
  - `argocd app list`
- STATUSは`OutOfSync`
  - 登録はされたが、現在の状態とGitの状態が異なる
- デフォルトでは自動的に同期しない
- 以下を実行
  - `argocd app sync bgd-app`
- あるいはUIからSYNCボタンをクリック
- `git push`後、`argocd app list`で確認
  - `Sync`になっている
  - しばらくすると、`OutOfSync`になる
- ロールバックは`git revert`後にpush

#### 7.2.自動同期

- `automated`ポリシーで`syncPolicy`セクションを使用
- 自動同期のメリットは、`argocdAPI`へのログインや`argocd`ツールが不要になること
  - シークレット管理などセキュリティ上の問題

```yaml
...
  source: 
  ...
  syncPolicy: 
    automated: {} 
```
- デフォルトの保守的な戦略を使用
  - 削除されたリソースのプルーニング
  - Git 経由ではなくk8sクラスターで直接変更が⾏われた場合にアプリケーションを⾃⼰修復
- ArgoCDは利用できないリソースを検出すると、削除せず、`OutOfSync`状態
- 手動で同期を呼び出す
  - ` argocd app sync --prune`
- 自動で削除する場合

```yaml
syncPolicy:
  automated:
    prune: true
```
- Argo CDは、クラスター内で⼿動で発⽣したドリフトを修正しないように設定
- 以下でドリフトを修正

```yaml
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
- 他にもArgoCD同期オプションは複数ある

#### 7.3.カスタマイズ統合

- 以下条件でKustomize プロジェクトを検出
  - kustomization.yaml 、Kustomizationのいずれかのファイルがある場合
- Applicationファイルで使用するツールを明示的に指定できる

```yaml
source:
  directory: 
    recurse: true
```
- 可能な戦略は, directory, chart, helm, kustomize, path, plugin

#### 7.4.ヘルムとの統合

- Helmプロジェクトを検出するとインストールする
- Argo CD はビルド環境変数をHelm マニフェストに⼊⼒
  - `ARCOCD_APP_xxx`など

```yaml
  source:
    path: ch07/bgd
    ...
    helm: 
      parameters: 
      - name: app 
        value: $ARGOCD_APP_NAME
```
- values.yamlより優先度が高い

#### 7.5.imageアップデータ

#### 7.6.プライベート Git リポジトリからデプロイする

#### 7.7.Kubernetesマニフェストを順序付ける

#### 7.8.同期ウィンドウを定義する


### 8.上級トピックス
<!--
#### 8.1.機密データを暗号化する（シールド・シークレット）

#### 8.2.ArgoCDで秘密を暗号化する (ArgoCD + HashiCorp Vault + 外部秘密)

#### 8.3.アプリケーションのデプロイを自動的にトリガーする (Argo CD Webhooks)

#### 8.4.複数のクラスタにデプロイする

#### 8.5.プルリクエストをクラスタにデプロイする

#### 8.6.高度なデプロイテクニックを使う
-->
