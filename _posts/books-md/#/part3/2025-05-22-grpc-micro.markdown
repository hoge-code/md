---
layout: post
title: "「gRPC Microservices in Go」を読む"
date: 2025-05-22 12:00:00 +0900
categories: [blog]
tags: [書籍, 設計, Go]
excerpt: "Go, GRPC"
memo: "1-5, 6-9"
---

Manning 出版の`gRPC Microservices in Go`という書籍を読んだので、気になったことを中心に簡単な感想や要約を書きます。(網羅性はありません)

この書籍はGRPC、クリーンアーキテクチャ、マイクロサービスで必要となるレジリエンスのGoでの実装方法、k8sや`Opentelemetory`などのデブオペ技術について基礎的な内容が網羅されています。

日本語の同様の内容の書籍では、エンジニア選書より「クラウドネイティブで実現する マイクロサービス開発・運用 実践ガイド<a href="https://gihyo.jp/book/2023/978-4-297-13783-0">(Gihyo)</a>」が存在します。 

扱う技術自体に重複はあまりないので両方読んでも問題ありません。

**目次**

- ToC
{:toc}

---

**サイト**

- <a href="https://www.oreilly.com/library/view/grpc-microservices-in/9781633439207/">オライリー</a>

---

## パート 1. gRPC とマイクロサービスアーキテクチャ

### 1 Go gRPC マイクロサービス入門

- grpc は LB、トレース、フォールトトレランス、セキュリティまでもサポート

#### 1.1 gRPC マイクロサービスの利点

##### 1.1.1 パフォーマンス

- HTTP/2 のサーバーサイドプッシュ、多重化、ヘッダー圧縮

##### 1.1.2 コード生成と相互運用性

##### 1.1.3 フォールトトレランス

- 冪等性はフォールトトレランスに重要
  - 再試行した場合にリソースの内容が変更されないようにするため
- 冪等性ユースケースに適していない場合
  - 再試行オペを停止するかを伝える検証エラーを提供する必要
- フォールトトレランスの実現方法
  - レート制限、サーキットブレイカー、フォールトインジェクション

冪等性を満たすコードを考えるのは難しいですが、フォールトトレランスを実現する多くの方法は、goのライブラリ、k8s、istioなどが解決してくれます。

##### 1.1.4 セキュリティ

- TLS, ALTS(Application Layer Transport Security)、トークンベース認証

##### 1.1.5 ストリーミング

- ページネーションの代わりになる

#### 1.2 REST と RPC

#### 1.3 gRPC を使用する場合

- `proto`ファイルの維持は容易ではない
  - 小規模なアプリには適さない

#### 1.4 本番環境レベルのユースケース

##### 1.4.1 マイクロサービス

- 今回構築する EC サイトのサービス
  - 製品、カート、チェックアウト、⽀払い、配送
- 依存関係(例：Checkout→Cart)の管理方法
  - CI を使って、スタブ生成などを自動化
- 各マイクロサービスは言語だけでなく、使用する DB も自由
  - 例：MySQL, Mongo, Neo4j

##### 1.4.2 コンテナランタイム

##### 1.4.3 CI/CD パイプライン

- CI/CD の例 
  - MySQL、Kubernetes、AWS などのサードパーティの統合を確認するための適切な統合テスト 
  - サービス間通信の契約テスト
- k8s などを使ってもデプロイ後のロールバックは難しい
  - エラー率、ユーザーフィードバックなどをもとに考える
  - ロールバックか、修正プログラムの導入かを考える

統合テストをCICDで実行する場合は時間がかかるので、プッシュごとなどではなく、適切な頻度で実行する必要がありそうです。また、
k8sを使えばコマンドで簡単にロールバックできると考えていましたが、そう単純ではないと感じました。

##### 1.4.4 監視と観測可能性

- メトリクス、ログ、トレース
- 特に、コンテキストのトレースはリクエストのライフサイクル確認に重要
- トレース ID をつけてグループ化
  - ダウンストリームサービスを含めた流れを把握
  - エラーだけでなく、遅延検出も重要
- 監視ダッシュボードで確認したい

##### 1.4.5 パブリックアクセス

- k8s の Nginx コントローラや API ゲートウェイなど

APIゲートウェイやBFFを使わない場合は、`Ingress`コントローラや`Istio`で公開する場合もありそうです。


### 2 gRPC とマイクロサービスの融合

#### 2.1 モノリシックアーキテクチャ

- 初期バージョンでは有効
- 製品を定期的に評価して、マイクロサービスへの移行時期かを考える

##### 2.1.1 開発

- ファイルが多いと、IDE が重くなる可能性
- コンパイルやテスト時間が長い

##### 2.1.2 デプロイメント

- 他が変更されていなくても、すべてのテストを実行する必要
- デプロイ全体が中断される可能性

##### 2.1.3 スケーリング

#### 2.2 スケールキューブ

- 三元でのスケーリング

##### 2.2.1 X 軸のスケーリング

##### 2.2.2 Z 軸スケーリング

##### 2.2.3 Y 軸のスケーリング

#### 2.3 マイクロサービスアーキテクチャ

- テストが容易
- 障害が発生しても、他のサービスは引き続き利用できる

##### 2.3.1 データの一貫性の取り扱い

##### 2.3.2 サガパターン

##### 2.3.3 振り付けベースのサガ

- それぞれのサービスがイベントを送信し合う(例：order_created_event)
- replyToChannel パラメーターをデータに含める
  - 発行元が次のサービスを知る必要
- Order→(Payment command channel)→Payment→(Create order reply channel)→Order

##### 2.3.4 オーケストレーターベースのサガ

- Pub/Sub
  - ブローカーが単一障害点になりやすい
- ロールバックは Payment:refund()、Order:cancel()など

オーケストラサーガは必ずしも正解ではないと感じました。

#### 2.4 サービス検出

- クライアント側のサービス検出
  - サービスレジストリに接続し、特定のサービスの場所を確認
- サーバー側のサービス検出
- k8s 使えばサービスディスカバリは不要

#### 2.5 サービス間通信に gRPC を使用する

##### 2.5.1 プロトコルバッファの操作

##### 2.5.2 Go ソースコードの生成

- 支払い要求などはストリーミングモードがいい

```go
service Payment {
     Create(stream CreatePaymentRequest)
     returns (CreatePaymentResponse){}
}
```

- ⾮同期的に処理されるため、失敗した操作をマークする⼀意の識別⼦を提供

##### 2.5.3 配線の接続

- タイムアウトは非常に重要
- 10 秒以内に結果が得られない場合に実⾏をキャンセルする例

```go
ctx, cancel := context.WithTimeout(context.Background(),
10 * time.Second)
defer cancel()
payment := pb.NewPaymentClient(conn)
...
```

## パート 2. gRPC マイクロサービスアプリケーションの開発、テスト、デプロイ

### 3 gRPC と Golang を使ってみる

#### 3.1 プロトコルバッファ

##### 3.1.1 メッセージタイプの定義

- 複数は`repeated`を使う
- フィールド番号は下位互換性のために変更しない
- フィールドを削除する場合は`reserved`を使って予約
  - 同じフィールド番号や名前が使われないようにするため
- `to`で範囲を指定

```go
message CreateOrderRequest {
    reserved 1, 2, 3 to 7; ❶
    reserved "customer_id"; ❷
    int64 user_id = 7;
    repeated Item items = 8;
    float amount = 9;
}
```

- 上記では`user_id`が使われているので`customer_id`を予約している
- 1-15 までは 1 バイトなので頻繁に使うフィールド用に予約

##### 3.1.2 プロトコルバッファエンコーディング

- マーシャリングは、メタデータとデータ自体のエンコード情報を含む

#### 3.2 スタブの生成

##### 3.2.1 プロトコルバッファコンパイラのインストール

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

##### 3.2.2 プロトコルバッファコンパイラの使用

- protocコマンドで生成

```proto
syntax = “proto3”;
option go_package=" GitHub/huseyinbabal/microservices/order";
...
```

- `option`はパッケージ名

#### 3.3 .proto ファイルの管理

##### 3.3.1 プロトプロジェクトの構造

- 以下のような構造になる

```plaintext
├── golang
│ ├── order
│ │ ├── go.mod
│ │ ├── go.sum
│ │ ├── order.pb.go
│ │ └── order_grpc.pb.go
```

- Golang の依存関係バージョン
  - Git をタグ、ブランチ、コミットハッシュとして参照

```bash
git tag -a golang/order/v1.2.3 -m "golang/order/v1.2.3"
...
git push --tags
go get -u github.com/huseyinbabal/microservices-proto/golang/order@v1.2.3
```

- `latest`を使わずに参照する

##### 3.3.2 ソースコード生成の自動化

- マトリックスを使うとできること

  - 1 つのサービスに対して異なるバージョンを使用して多数のバイナリをビルドできる

- order、payment、shipping サービスのソースコード生成が実行

```yaml
on:
    push:
    tags:
        - v**
jobs:
    protoc:
    name: "Generate"
    runs-on: ubuntu-latest
    strategy:
        matrix:
            service: ["order", "payment", "shipping"]

steps:
    ...
    - name: Etract Release Version
      run: echo "RELEASE_VERSION=${GITHUB_REF#refs/*/}" >> $GITHUB_ENV ❸
    - name: "Generate for Golang"
      shell: bash
      run: |
        chmod +x "${GITHUB_WORKSPACE}/run.sh"
        ./run.sh ${{ matrix.service }} ${{ env.RELEASE_VERSION }}
```

- 上の例のシェルスクリプトでは、GRPC のスタブ生成のコマンドを書く

#### 3.4 後方互換性と前方互換性

##### 3.4.1 新しいフィールドの追加

##### 3.4.2 サーバーをアップグレードするがクライアントはアップグレードしない

##### 3.4.3 クライアントをアップグレードするがサーバーをアップグレードしない

##### 3.4.4 フィールドの追加/削除

##### 3.4.5 フィールドを他のフィールドへ移動する

GRPCはデバッグが難しいことと同様に、互換性を維持するのも非常に難しいと感じた章でした。


### 4 マイクロサービスプロジェクトのセットアップ

#### 4.1 六角形アーキテクチャ

- ポートとアダプター

##### 4.1.1 アプリケーション

- ポートには読み取りと書き込みクエリがある

##### 4.1.2 アクター

##### 4.1.3 ポート

##### 4.1.4 アダプタ

#### 4.2 注文サービスの実装

- フォルダ構造は以下のようになる

```plaintext
- cmd
    - main.go
-config
    - config.go
-internal
    - adapters
        - db
            - db.go
        - grpc
            - grpc.go
            - server.go
    - application
        - core
            - api
                - api.go
            - domain
                - order.go
    - ports
        - api.go
        - db.go
    go.mod
```

- 内部ポートは`application`で、外部ポートは`adapters`で具象化する

##### 4.2.1 プロジェクトフォルダ

##### 4.2.2 Go プロジェクトの初期化

```bash
go mod init GitHub.com/<username>/microservices/order
```

- `go mod`コマンドは VCS の URL を引数にとり、`go.mod`を作成

```go
# go.mod
module github.com/huseyinbabal/microservices/order
go 1.17
...
```

##### 4.2.3 アプリケーションコアの実装

- `domain/order.go`に構造体を定義

```go
type OrderItem struct {
    ProductCode string `json:"product_code"`
    UnitPrice float32 `json:"unit_price"`
    Quantity int32 `json:"quantity"`
    ...
}
```

- `application/api.go`では DB ポートに依存するアプリを作成

```go
type Application struct {
    db ports.DBPort
}

func NewApplication(db ports.DBPort) *Application {
    return &Application{
        db: db,
    }
}

func (a Application) PlaceOrder(order domain.Order) (domain.Order, error) {
    err := a.db.Save(&order)
    ...
    }
```

- DB ポートのインターフェースで定義した`Save`メソッドを呼び出します
  - 実際には、DB インターフェースを実装したアダプタが呼び出されます

##### 4.2.4 ポートの実装

- `internal/ports/api.go` を使用して API ポートの実装
- 先ほど定義した`Application`構造体はこのインターフェースを実装したことになる

```go
type APIPort interface {
    PlaceOrder(order domain.Order) (domain.Order, error)
}
```

- DB ポートも同様に定義する

```go
type DBPort interface {
    Get(id string) (domain.Order, error)
    Save(*domain.Order) error
}
```

##### 4.2.5 アダプタの実装

```go
type Adapter struct {
    db *gorm.DB
}

func NewAdapter(dataSourceUrl string) (*Adapter, error) {
    ...
}

type Order struct {
     gorm.Model
     CustomerID int64
     Status string
     OrderItems []OrderItem
}

func (a Adapter) Get(id string) (domain.Order, error) {
    ...
}
```

- 上記は`Gorm`に依存し、DB ポートを実装している

##### 4.2.6 gRPC アダプタの実装

```go
type Adapter struct {
     api ports.APIPort
     port int
     order.UnimplementedOrderServer
     ...
}
```

- アプリサービス、ポート番号、上位互換性対応が依存関係の構造体

- 次に、`Run()`メソッドを定義する

```go
func (a Adapter) Run() {
    grpcServer := grpc.NewServer()
    order.RegisterOrderServer(grpcServer, a)
    if config.GetEnv() == "development" {
        reflection.Register(grpcServer)
    }
 ...
}
```

- 上記では注文`GRPC`サーバを初期化している
- 開発環境でのみ、`grpcurl`のためのリフレクションを有効にしている

- 次に、`proto`ファイルで作成した関数を実装する

```go
func (a Adapter) Create(ctx context.Context, request *order.CreateOrderRequest) (*order.CreateOrderResponse, error) {
    ...
    return &order.CreateOrderResponse{OrderId: result.ID}, nil
}
```

##### 4.2.7 依存性注入とアプリケーションの実行

- `config.go`では環境変数の読み取り関数などを定義する

- `main.go`で以下のように`main`メソッドを定義

```go
func main() {
    dbAdapter, err := db.NewAdapter(config.GetDataSourceURL())
    if err != nil {
        log.Fatalf("Failed to connect to database. Error: %v", err)
    }
    application := api.NewApplication(dbAdapter)
    grpcAdapter := grpc.NewAdapter(application, config.GetApplicationPort())
    grpcAdapter.Run()
}
```

- 依存関係エラーが出た場合は、`go mod tidy`で依存関係を再編成し、再実行

##### 4.2.8 gRPC エンドポイントの呼び出し

- `grpcurl`コマンドで呼び出せる


### 5 サービス間通信

- GRPC でも HTTP と同様に、応答ステータスやエラーの種類が重要

#### 5.1 gRPC サービス間通信

##### 5.1.1 サーバー側の負荷分散

##### 5.1.2 クライアント側の負荷分散

- サーバーサイド LB
  - 設定が簡単
  - スループットの制限
- クライアント LB
  - パフォーマンスがいい
  - 単一障害点なし
  - 設定が難しい

#### 5.2 モジュールに依存し、ポートとアダプタを実装する

- Order サービスで以下のコマンドで依存関係を追加
- `go get -u github.com/huseyinbabal/microservices-proto/golang/payment`

##### 5.2.1 支払いポート

##### 5.2.2 決済アダプタ

##### 5.2.3 支払いポートの実装

- `ports/payment.go`

```go
type PaymentPort interface {
     Charge(*domain.Order) error
}
```

##### 5.2.4 支払いアダプタの実装

- `adapters/payment/payment.go`

```go
type Adapter struct {
     payment payment.PaymentClient
}
```

- 上記の`payment`は、`go get`で依存関係を追加したので`Order`サービスから利用できる

```go
func NewAdapter(paymentServiceUrl string) (*Adapter, error) {
    var opts []grpc.DialOption 
    opts = append(opts,grpc.WithTransportCredentials(insecure.NewCredentials())) 
    conn, err := grpc.Dial(paymentServiceUrl, opts...) 
    if err != nil {
        return nil, err
    }
    defer conn.Close() 
    client := payment.NewPaymentClient(conn) 
    return &Adapter{payment: client}, nil
}
```
- 上記では`Payment`サービスのクライアントGRPCを作成
- その後、`Order`サービスのアダプターに依存関係を追加
- また、TLSを無効にしている

```go
func (a *Adapter) Charge(order *domain.Order) error {
     _, err := a.payment.Create(context.Background(), &payment.CreatePaymentRequest{ 
        UserId: order.CustomerID,
        OrderId: order.ID,
        TotalPrice: order.TotalPrice(),
    })
    return err
}
```
- 上の例では、`Order`エンティティに`TotalPrice()`というメソッドを定義している

##### 5.2.5 支払明細書のクライアント設定

```go
func main() {
     ...
     application := api.NewApplication(dbAdapter, paymentAdapter) 
     grpcAdapter := grpc.NewAdapter(application,config.GetApplicationPort())
     grpcAdapter.Run()
}
```
- 上記では、DBアダプターと、`Payment`サービスのクライアントGRPCアダプターをアプリの依存関係に追加
- その後、アプリを`Order`サービスの依存関係に追加し、実行している

##### 5.2.6 gRPC での支払いアダプタの使用

#### 5.3 エラー処理

##### 5.3.1 ステータスコード

- HTTPと似たようなステータスも多い
    - `INVALID_ARGUMENT`, `NOT_FOUND`, `PERMISSION_DENIED`など

##### 5.3.2 エラーコードとメッセージを返す

- 以下のようにしてエラーを作成する

```go
err = status.Errorf(
    codes.InvalidArgument,
    fmt.Sprintf("failed to charge user: %d", request.UserId))
```
- `status`ライブラリ、`codes`ライブラリを使用

##### 5.3.3 詳細エラー

```go
func (a Application) PlaceOrder(order domain.Order) (domain.Order, error) {
    ...
    paymentErr := a.payment.Charge(&order)
    if paymentErr != nil {
        st, _ := status.FromError(paymentErr)  
        fieldErr := &errdetails.BadRequest_FieldViolation{
            Field:       "payment",
            Description: st.Message(),
        }  
        
        // BadRequestエラーの設定
        badReq := &errdetails.BadRequest{}  
        badReq.FieldViolations = append(badReq.FieldViolations, fieldErr)  

        // 注文失敗のステータスを作成
        orderStatus := status.New(codes.InvalidArgument, "order creation failed")  
        statusWithDetails, _ := orderStatus.WithDetails(badReq)  

        return domain.Order{}, statusWithDetails.Err()
    }

    // 注文成功
    return order, nil
}
```

- 上のように書くと、他のサービスのエラーを使用できる
- また、エラーコードだけでなく、エラーフィールドも追加している

##### 5.3.4 クライアント側でのエラー処理

- マイクロサービスはエラーの集約も重要
    - 複数のダウンストリームサーバがあることが多いから 
- 以下のようにして、複数のエラーを扱う

```go
 paymentErr := a.payment.Charge(&order)
 if paymentErr != nil {
    st := status.Convert(paymentErr) 
    var allErrors []string 
        for _, detail := range st.Details(
            switch t := detail.(type) {
            case *errdetails.BadRequest: 
                for _, violation := range t.GetFieldViolations() {
                    allErrors = append(allErrors, violation.Description)
                 }
            }
            ...
 }
``` 

##### 5.3.5 支払いサービスの実行


### 6 回復力のあるコミュニケーション

<!--
#### 6.1 回復力パターン

##### 6.1.1 タイムアウトパターン

##### 6.1.2 再試行パターン

##### 6.1.3 サーキットブレーカーパターン

#### 6.2 エラー処理

##### 6.2.1 gRPC エラーモデル

##### 6.2.2 gRPCエラー応答

#### 6.3 gRPC通信のセキュリティ保護

##### 6.3.1 TLSハンドシェイク

##### 6.3.2 証明書の生成

##### 6.3.3 gRPC TLS 認証情報

#### まとめ
-->

### 7 マイクロサービスのテスト

<!--
#### 7.1 テストピラミッド

#### 7.2 ユニットテストによるテスト

##### 7.2.1 テスト対象システム

##### 7.2.2 テストワークフロー

##### 7.2.3 モックの操作

##### 7.2.4 モックの実装

##### 7.2.5 自動モック生成

#### 7.3 統合テスト

##### 7.3.1 テストスイートの準備

##### 7.3.2 テストコンテナの操作

#### 7.4 エンドツーエンドテスト

##### 7.4.1 仕様

##### 7.4.2 Docker Composeサービス定義の理解

##### 7.4.3 エンドツーエンドのテストフォルダ構造

##### 7.4.4 データベース層

##### 7.4.5 支払いサービス層

##### 7.4.6 注文サービス層

##### 7.4.7 スタックに対するテストの実行

#### 7.5 テスト範囲

##### 7.5.1 カバレッジ情報

##### 7.5.2 CIパイプラインでのテスト

#### まとめ
-->

### 8 展開

<!--
#### 8.1 ドッカー

##### 8.1.1 イメージの構築

#### 8.2 Kubernetes

##### 8.2.1 Kubernetesアーキテクチャ

##### 8.2.2 Kubernetesリソース

##### 8.2.3 マイクロサービス展開のイーグルビュー

##### 8.2.4 ポッド

##### 8.2.5 デプロイメント

##### 8.2.6 サービス

##### 8.2.7 NGINX Ingressコントローラ

#### 8.3 証明書管理

##### 8.3.1 インストール

##### 8.3.2 クラスタ発行者

##### 8.3.3 Ingressでの証明書の使用

##### 8.3.4 クライアント側の証明書

#### 8.4 展開戦略

##### 8.4.1 ローリングアップデート

##### 8.4.2 ブルーグリーンデプロイメント

##### 8.4.3 カナリアデプロイメント

##### 8.4.4 展開に関する最終的な考察

#### まとめ
-->

## パート 3. gRPC とマイクロサービスアーキテクチャ

### 9 可観測性

<!--
#### 9.1 可観測性

##### 9.1.1 トレース

##### 9.1.2 メトリクス

##### 9.1.3 ログ

#### 9.2 オープンテレメトリー

##### 9.2.1 計器の位置

##### 9.2.2 計装

##### 9.2.3 メトリックバックエンド

##### 9.2.4 サービスパフォーマンス監視

#### 9.3 Kubernetesにおける可観測性

##### 9.3.1 イェーガー オールインワン

##### 9.3.2 OpenTelemetryコレクター

##### 9.3.3 プロメテウス

##### 9.3.4 Jaegerのインストール

##### 9.3.5 注文サービス用の OpenTelemetry インターセプター

##### 9.3.6 注文サービスの指標を理解する

##### 9.3.7 アプリケーションログ

##### 9.3.8 ログ収集

##### 9.3.9 ロギングバックエンドとしてのElasticsearch

##### 9.3.10 ログダッシュボードとしての Kibana

#### まとめ
-->
