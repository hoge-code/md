---
layout: post
title: "socket.ioでHTTPヘッダーを使った認証"
date: 2025-02-21 12:00:00 +0900
categories: [blog]
tags: [Nodejs]
excerpt: "Node.js + socket.io + ミドルウェア"
---

`socket-io`を使って認証機能を実装する際につまづいたので記事に残しておきます。

### Socket.IOの通信プロトコル
`Socket.io`接続をする場合、HTTPから`ws`にアップグレードされますが、(HTTPSの場合は`wss://`)、初回のプロトコルアップグレード時のみHTTPリクエストとしてヘッダーやクエリパラメータを送信できます。それ以降はHTTPではなく、`ws`での通信となるので、テキストでもバイナリでも、データを送信するにはペイロード内で送る必要があります。

### クライアント側での初期設定
以下のように`extraHeaders` を使用してトークンを送信します。(フロントでトークンを保存している場合)この際、再接続処理や、そのバックオフも書いておきます。
```javascript
// クライアント側 (ブラウザ)
const socket = io('http://localhost:3000', {
  transports: ['websocket'],
  extraHeaders: {
    'Authorization': 'Bearer my_token_here'
  },
  reconnection: true, // 再接続を有効化
  reconnectionAttempts: 5, // 無限に再接続を試みる
  reconnectionDelay: 1000, // 再接続の待機時間（ミリ秒）
  reconnectionDelayMax: 5000, // 最大待機時間（ミリ秒）
  randomizationFactor: 0.5, // 再接続時の待機時間のランダム化
});
```
`React`などを使う場合は、上記で作成した`socket`を`useContext`などを使うと様々なコンポーネントで`socket.emit`の処理を書けるようになります。

### サーバー側での設定
`Node.js + Express`の場合、接続時に `socket.handshake` を使って、ヘッダーを確認することができます。`python-socketio`の場合は`environ`でヘッダーを確認できます。</br>
以下のサンプルコードの`isValidToken`関数では、`jsonwebtoken`ライブラリなどを使い、JWTペイロードからクレームを検証するような処理を実装します。認証サービスを利用している場合は、`cognito`や`keycloak`の公開鍵エンドポイントを使って公開鍵を取得して検証します。
```javascript
// サーバー側での認証処理例
const io = require('socket.io')(server);

io.use((socket, next) => {
  const token = socket.handshake.headers['authorization'];
  if (isValidToken(token)) {
    return next();  // 認証成功
  }
  return next(new Error('Authentication error'));  // 認証失敗
});

io.on('connection', (socket) => {
  console.log('User connected');
});
```

### テスト
調べたところ、`OAS`のように`AsyncAPI`のドキュメントを自動生成するフレームワークはなかったので、手動で`AsyncAPI`ドキュメントをyamlで作成した後に、`Postman`でトークンを環境変数に設定し、エンドポイントをテストするという流れになりそうです。