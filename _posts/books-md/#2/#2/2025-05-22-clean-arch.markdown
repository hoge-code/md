---
layout: post
title: "「Get Your Hands Dirty on Clean Architecture」を読む"
date: 2025-05-22 12:00:00 +0900
categories: [blog]
tags: [書籍, 設計, Go]
excerpt: "Go"
memo: "4-6, 7-15"
---

Packt社の`Get Your Hands Dirty on Clean Architecture`という書籍を読んだので、気になったことを中心に簡単な感想や要約を書きます。日本語にも翻訳されていて、こちらは「手を動かしてわかるクリーンアーキテクチャ」というタイトルになっています。

また、この書籍はJavaで書かれていますが、ソースコードが全て画像(文字ではない)になっていて、コピーできないのでGoに変換して書いてみます。

**目次**
* ToC
{:toc}

---

**サイト**

- <a href="https://www.oreilly.com/library/view/get-your-hands/9781805128373/">オライリー</a>

---

### 第1章 保守性
<!--
#### 保守性とは一体何を意味するのでしょうか?

#### 保守性により機能性が向上

#### 保守性は開発者の喜びを生み出す

#### 保守性は意思決定をサポートする

#### 保守性の維持
-->

### 第2章: レイヤーの何が問題なのか?
<!--
#### 彼らはデータベース駆動型設計を推進している

#### 彼らは近道をする傾向がある

#### テストが難しくなる

#### ユースケースを隠している

#### 並行作業を困難にする

#### これは保守可能なソフトウェアの構築にどのように役立ちますか?
-->


### 第3章 依存関係の反転
<!--
#### 単一責任原則

#### 副作用についての物語

#### 依存性逆転の原則

#### クリーンアーキテクチャ

#### 六角形の建築

#### これは保守可能なソフトウェアの構築にどのように役立ちますか?
-->

### 第4章 コードの整理

- パッケージ構造を整理しても、ファサードになりがち
    - インポートすべきでない他のパッケージからインポートしていること
- この章は、SendMoneyユースケースについて考える

#### レイヤー別に整理する

```plaintext
domain
    - Account
    - ...
    - AccountService
persistence
    - XXXRepositoryImpl
web
    - XXXController
```

- 上記の問題点
    - ドメインの機能の間にパッケージ境界がない
    - 構造化しないと、クラスの混乱を引き起こす

こうしたフォルダ構造は、一般的なウェブフレームワークのデフォルトで設定されていることが多いです。

#### 機能別に整理

```plaintext
account
    - Account
    - XXXController
    - XXXRepositoryImpl
    - ...
    - SendMoneyService
```

- 外部アクセス制限のために`package-private`可視性を利用
    - これにより、パッケージ境界を強制できる
- `AccountService`の名前を`SendMoneyService`に変更
    - サービス層のクラス名を具体的にするのは特に重要
- ただし、アダプター、ポート層が分かりづらい

こうした機能別のフォルダ構造は、`NestJS`や`Angular`などで見られます。

#### アーキテクチャに表現力豊かなパッケージ構造

- 一般的なヘキサゴナルアーキテクチャのフォルダ構造

```plaintext
adapter
    - in
        - web
            - SendMoneyController
    - out
        - persistence
            - AccountPersistenceAdapter
            - SpringDataAccountRepository
-application
    - domain
        - model
            -　Account
        - service
            - SendMoneyService
    - port
        - in
            - SendMoneyUseCase
        - out
            - UpdateAccountStatePort
- common
```

**フォルダ構造の解説**
- `adapter`パッケージ
    - `web`と`persistence`アダプター
    - これらは受信、送信ポートの実体
- ポートは`application`パッケージ内に配置する
- このフォルダ構造なら、サードパーティAPIの変更なども容易
    - `adapter/out/xxx`パッケージの変更で済む
- 複数のドメインを管理する場合は、`domain`内にサブパッケージを配置

**可視性**
- ただし、フォルダ構造が多いので可視性について考える必要
    - `adapter`は呼び出されないので、パッケージプライベートでいい
    - ただし、ポート、ドメインモデルはパブリックにする必要
    - ドメインサービスはプライベートでいい

こうしたフォルダ構造はクリーンアーキテクチャでは一般的ですが、パッケージ可視性や、長期に渡って維持するための規律など、考えることが多そうだと感じました。

#### 依存性注入の役割

- ここでは便宜のために、`interface`には`I...`とつけています

```plaintext
SendMoneyController→ISendMoneyUseCase
　　　　　　　　　　　　　　↑
                    SendMoneyService
                    　　　↓
                    IUpdateAccountStatePort←AccountPersistenceAdapter
```

- コントローラは、受信ポート(サービスが実装)を呼び出す
- サービスは、送信ポート(アダプタが実装)を呼び出す

Javaでは`SpringBoot`がDIでインターフェースを実装した具象クラスを自動で呼び出します。

Goでは自分で`main.go`などで依存関係を書くか、`wire`などのライブラリを使います。JSでは`InversifyJS`などが有名です。

#### これは保守可能なソフトウェアの構築にどのように役立ちますか?

この章では、クリーンアーキテクチャの一般的なフォルダ構造や、呼び出しの流れが説明されていました。


### 第5章: ユースケースの実装

- この章では、DDD的にドメインエンティティから始めて、ユースケースを組み立てる

#### ドメインモデルの実装

- 別の口座に送信するユースケースを考える
- まずは、`Account`エンティティを作成
    - お金を引き出し、送信できる機能を持つ

- 以下はGoのサンプルコード
    - Javaでは`@AllArgsConstructor`の他に、`withoutId`関数で初期化できる
    - また、`AccountId`はクラス内クラスになっている

```go
type AccountId struct {
	Value int64
}

// Account は残高と最近のアクティビティを持つ銀行口座
type Account struct {
	id              *AccountId
	BaselineBalance Money
	ActivityWindow  ActivityWindow
}

// IDなしで新しいアカウントを作成（新規作成用）
func NewAccountWithoutId(baselineBalance Money, activityWindow ActivityWindow) *Account {
	return &Account{
		id:              nil,
		BaselineBalance: baselineBalance,
		ActivityWindow:  activityWindow,
	}
}

// ID付きでアカウントを作成（既存エンティティの復元用）
func NewAccountWithId(id *AccountId, baselineBalance Money, activityWindow ActivityWindow) *Account {
	return &Account{
		id:              id,
		BaselineBalance: baselineBalance,
		ActivityWindow:  activityWindow,
	}
}

// アカウントIDを返します（存在しない場合はnil）
func (a *Account) GetId() *AccountId {
	return a.id
}

// アカウントの現在の合計残高を計算
func (a *Account) CalculateBalance() Money {
	return a.BaselineBalance.Add(a.ActivityWindow.CalculateBalance(a.id))
}

// 指定された金額を引き出し
// 残高が足りていればアクティビティを追加し、trueを返す
func (a *Account) Withdraw(money Money, targetAccountId *AccountId) bool {
	if !a.mayWithdraw(money) {
		return false
	}

	withdrawal := Activity{
		OwnerAccountId:  a.id,
		SourceAccountId: a.id,
		TargetAccountId: targetAccountId,
		Timestamp:       time.Now(),
		Money:           money,
	}

	a.ActivityWindow.AddActivity(withdrawal)
	return true
}

// 指定された金額を引き出すことが可能か判定
func (a *Account) mayWithdraw(money Money) bool {
	return a.CalculateBalance().Add(money.Negate()).IsPositiveOrZero()
}

// 指定された金額を入金
// アクティビティを追加し、常にtrueを返します
func (a *Account) Deposit(money Money, sourceAccountId *AccountId) bool {
	deposit := Activity{
		OwnerAccountId:  a.id,
		SourceAccountId: sourceAccountId,
		TargetAccountId: a.id,
		Timestamp:       time.Now(),
		Money:           money,
	}

	a.ActivityWindow.AddActivity(deposit)
	return true
}
```
- `Account`エンティティ
    - 現在の口座アカウントのスナップショットを提供
    - `ActivityWindows`値オブジェクトは最新の一部のウィンドウのみ所持
        - 全てを読み込むのはメモリ効率が悪いため
    - `baselineBalance`属性
        - アクティビティの直前に口座が持っていた残高
    - `withdraw()`と`deposit()`を使って引き出しと入金を行う
- `Activity`エンティティ
    - 全ての引き出しと入金をキャプチャ

上記により、お金の引き出しと入金ができる`Account`エンティティを作成できたので、これを中心にユースケースを構築していきます。

#### ユースケースの要約

- 入力アダプターから入力を受け取る
    - `input validation`と`business rule validation`の違いを意識
    - DTOの検証に書きすぎないということ

- 以下はGoのサンプルコード
    - Javaでは`@Transactional`でトランザクションを実装している

```go
package service

import (
	...
)

// SendMoneyService は送金ユースケースの実装
type SendMoneyService struct {
	LoadAccountPort         outbound.LoadAccountPort
	AccountLock             outbound.AccountLock
	UpdateAccountStatePort  outbound.UpdateAccountStatePort
	MoneyTransferProperties MoneyTransferProperties
}

// NewSendMoneyService はSendMoneyServiceのコンストラクタ
func NewSendMoneyService(
	loadAccountPort outbound.LoadAccountPort,
	accountLock outbound.AccountLock,
	updateAccountStatePort outbound.UpdateAccountStatePort,
	properties MoneyTransferProperties,
) *SendMoneyService {
	return &SendMoneyService{
		LoadAccountPort:         loadAccountPort,
		AccountLock:             accountLock,
		UpdateAccountStatePort:  updateAccountStatePort,
		MoneyTransferProperties: properties,
	}
}

// SendMoney は送金処理を行う
func (s *SendMoneyService) SendMoney(command inbound.SendMoneyCommand) (bool, error) {
    // TODO: ビジネスルールの検証
    // TODO: モデルの状態の変更
    // TODO: 戻り値を返す
}
```

- 上記は`SendMoneyUseCase`受信ポートの実装
- `LoadAccountPort`送信ポートを呼び出してアカウントを読み込む
- `UpdateAccountState`ポートを呼び出して、アカウントをDBに書き込む


#### 入力の検証

- 入力の検証をDTOで行うか、ユースケースで行うかは意見が分かれる

- 以下はGoのサンプルコード
    - Javaでは`record`を使用して`immutable`にしている
    - 受信ポートパッケージ内にある
    - DTOなので、0以上やnullでないなど、簡単な検証のみ行う

```go
package inbound

import (
    ...
)

// SendMoneyCommand は送金のコマンド
type SendMoneyCommand struct {
	SourceAccountId model.AccountId
	TargetAccountId model.AccountId
	Money           model.Money
}

// NewSendMoneyCommand は送金コマンドを作成
// バリデーションが失敗した場合はエラーを返します
func NewSendMoneyCommand(sourceAccountId model.AccountId, targetAccountId model.AccountId, money model.Money) (*SendMoneyCommand, error) {
	command := &SendMoneyCommand{
		SourceAccountId: sourceAccountId,
		TargetAccountId: targetAccountId,
		Money:           money,
	}

	// バリデーション
	if err := validateSendMoneyCommand(command); err != nil {
		return nil, err
	}

	return command, nil
}

// validateSendMoneyCommand は送金コマンドのバリデーション
func validateSendMoneyCommand(command *SendMoneyCommand) error {
	if command.SourceAccountId == (model.AccountId{}) {
		return errors.New("送金元アカウントIDが必須です")
	}

	if command.TargetAccountId == (model.AccountId{}) {
		return errors.New("送金先アカウントIDが必須です")
	}

	if err := validation.ValidatePositiveMoney(command.Money); err != nil {
		return err
	}

	return nil
}
```

- これは腐敗防止層
    - 簡易的に不適切な入力を跳ね返す役割を持つ
- CommandはGoFのコマンドパターンではないことに注意
    - GoFの場合は`execute()`を持つ
- 筆者は`SendMoneyDTO`よりコマンドという用語をお勧めしている
    - モデルの状態変更を分かりやすくするため

#### コンストラクタの力

- コンストラクタの引数が多い場合
    - 長いコンストラクタをプライベートにし、ビルダーパターンの利用
        - `SendMoneyCommandBuilder`というクラスを作成
        - 新しいフィールドが増える場合、コンストラクタとビルダーを追加する必要
            - 追加を忘れやすい
    - 別の方法は、引数を値オブジェクトにまとめること

#### さまざまなユースケースに応じたさまざまな入力モデル

- 異なるユースケースに同じ入力モデルを使いたくなる
    - `Register account`と`Update account`など
        - 上記はidが不要、必要かという違い
            - nullを許可するのは微妙
    - そこで、ユースケースごとに入力モデルを分ける

言語によって実装は異なりますが、ベースとなる型を拡張することでDRYを実現できます。FastAPIなどで特に一般的な手法です。Goでは例えば以下のように実装できます。

```go
type UpdateAccountCommand struct {
	AccountDetails
	ID string `json:"id" validate:"required"`
}
```

#### ビジネスルールの検証

- 「ソース勘定の残⾼を確認するためにモデルの現在の状態にアクセスする必要」
    - これはビジネスロジックなのでDTOに書いてはいけない
    - ドメインルールはエンティティ内(Accountなど)に記述
        - `mayWithdraw()`など
- エンティティで検証が難しい場合はユースケースに書く
    - `sendMoney()`内で、`requireAccountExists()`を呼び出すなど

#### 豊富なドメインモデルと貧血ドメインモデル

- 貧血ドメインモデルはゲッター、セッターのみなどのこと

#### さまざまなユースケースに応じたさまざまな出力モデル

- ユースケースは呼び出し元に何を返す必要があるか？
    - 本当に必要なデータのみを返す
        - bool, errorなど
        - 例えば、SendMoneyのユースケースで完全なAccountを呼び出し元に返すか？
            - もし必要なら、新しいユースケースを作成すべき
        - 同じ出力モデルの共有はユースケース間で密結合になる
            - 出力モデルの分離
            - ドメインエンティティを出力モデルにしたい誘惑
                - エンティティを入力、出力モデルに利用する方法は後の章で解説

個人的に、コマンド関数でもモデル全体を返してしまうことがあるので、`error`などでシンプルに返すことを意識したいです。

#### 読み取り専用ユースケースの場合はどうでしょうか?

- 読み取り専用のユースケース
    - UIでの残高の表示など
    - クエリ専用の受信ポートの実装

CQRSの考え方はマイクロサービスだけでなく、サービスファイル内の関数の分割にも有効です。

#### これは保守可能なソフトウェアの構築にどのように役立ちますか?

- ユースケースの入力と出力を個別にモデル化
    - 副作用の回避、ただし面倒
    - ユースケースを明確に理解できるため、長期的には維持しやすい


### 第6章: Webアダプタの実装

#### 依存性の逆転

- コントローラとサービスの間にレイヤーを追加するとテストしやすくなる
- WebSocketなど双方向の場合
    - WebSocketControllerは`port.in`, `port.out`の両方に依存

#### Webアダプタの役割

- リクエストオブジェクトをユースケースの入力モデルに変換 
- ユースケースの呼び出し
- ユースケースの出力をHTTPに変換

#### スライスコントローラー

- できるだけ狭く、多く作るべき
- 一般的なアプローチは`AccountController`の作成
    - ただし、同じ返却クラスの共有は誤解を招く
        - 例：`create`APIではidは不要だが、`update`APIでは必要
- 操作ごとのコントローラを作成
    - 場合によっては個別にパッケージに分ける
    - できるだけユースケースに近い命名

- 以下はGoのサンプルコード

```go
package main

import (
	...
)

// SendMoneyController は送金処理を管理するコントローラー
type SendMoneyController struct {
	SendMoneyUseCase in.SendMoneyUseCase
}

// NewSendMoneyController は新しいコントローラーを作成
func NewSendMoneyController(sendMoneyUseCase in.SendMoneyUseCase) *SendMoneyController {
	return &SendMoneyController{
		SendMoneyUseCase: sendMoneyUseCase,
	}
}

// SendMoney は送金処理を実行するエンドポイント
func (ctrl *SendMoneyController) SendMoney(c *fiber.Ctx) error {
	// パスパラメータから値を取得
	sourceAccountIdStr := c.Params("sourceAccountId")
	targetAccountIdStr := c.Params("targetAccountId")
	amountStr := c.Params("amount")

	// 文字列から数値に変換
	sourceAccountId, err := strconv.ParseInt(sourceAccountIdStr, 10, 64)
	if err != nil {
		return c.Status(fiber.StatusBadRequest).SendString("Invalid sourceAccountId")
	}
	...

	// コマンドを作成
	command := in.SendMoneyCommand{
		SourceAccountId: model.AccountId{Value: sourceAccountId},
		TargetAccountId: model.AccountId{Value: targetAccountId},
		Money:           model.Money{Amount: amount},
	}

	// 送金処理を実行
	err = ctrl.SendMoneyUseCase.SendMoney(command)
	if err != nil {
		return c.Status(fiber.StatusInternalServerError).SendString("Failed to send money")
	}

	// 成功した場合
	return c.SendString("Money sent successfully")
}
```

#### これは保守可能なソフトウェアの構築にどのように役立ちますか?

個人的にコントローラファイルは大きく書くことが多かったので、ユースケースを意識し、ファイルを小さく保つという考え方は非常に参考になりました。

また、RESTは一方向ですが、`WebSocket`、`RabbitMQ`など双方向通信が必要な場合はフォルダ構造を別に考える必要がありそうです。


### 第7章: 永続化アダプタの実装
<!--
#### 依存性の逆転

#### 永続化アダプタの役割

#### スライスポートインターフェース

#### 永続アダプタのスライス

#### Spring Data JPAの例

#### データベーストランザクションはどうですか?

#### これは保守可能なソフトウェアの構築にどのように役立ちますか?
-->


### 第8章: アーキテクチャ要素のテスト
<!--
#### テストピラミッド

#### ユニットテストによるドメインエンティティのテスト

#### ユニットテストによるユースケースのテスト

#### 統合テストによるWebアダプタのテスト

#### 統合テストによる永続化アダプタのテスト

#### システムテストによるメインパスのテスト

#### どの程度のテストで十分でしょうか?

#### これは保守可能なソフトウェアの構築にどのように役立ちますか?
-->


### 第9章 境界間のマッピング
<!--
#### 「マッピングなし」戦略

#### 「双方向」マッピング戦略

#### 「完全」マッピング戦略

#### 「一方通行」マッピング戦略

#### どのマッピング戦略をいつ使用すればよいですか?

#### これは保守可能なソフトウェアの構築にどのように役立ちますか?
-->


### 第10章: アプリケーションの組み立て
<!--
#### なぜ組み立てを気にする必要があるのでしょうか?

#### プレーンコードによるアセンブル

#### Springのクラスパススキャンによるアセンブル

#### SpringのJava Configを使ったアセンブル

#### これは保守可能なソフトウェアの構築にどのように役立ちますか?
-->


### 第11章：意識的に近道をする
<!--
#### ショートカットが割れた窓のような理由

#### クリーンなスタートの責任

#### ユースケース間でのモデルの共有

#### ドメインエンティティを入力または出力モデルとして使用する

#### 着信ポートをスキップする

#### サービスをスキップする

#### これは保守可能なソフトウェアの構築にどのように役立ちますか?
-->


### 第12章 アーキテクチャ境界の強制
<!--
#### 境界と依存関係

#### 可視性修飾子

#### コンパイル後の適合関数

#### ビルドアーティファクト

#### これは保守可能なソフトウェアの構築にどのように役立ちますか?
-->


### 第13章: 複数の境界付きコンテキストの管理
<!--
#### 境界付けられたコンテキストごとに 1 つの六角形ですか?

#### 分離された境界付きコンテキスト

#### 適切に結合された境界付きコンテキスト

#### これは保守可能なソフトウェアの構築にどのように役立ちますか?
-->


### 第14章: ソフトウェアアーキテクチャへのコンポーネントベースのアプローチ
<!--
#### コンポーネントによるモジュール性

#### ケーススタディ – 「チェックエンジン」コンポーネントの構築

#### コンポーネント境界の強制

#### これは保守可能なソフトウェアの構築にどのように役立ちますか?
-->


### 第15章 アーキテクチャスタイルの決定
<!--
#### シンプルに始める

#### ドメインを進化させる

#### 自分の経験を信じましょう

#### 場合によります
-->