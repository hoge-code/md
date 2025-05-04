---
layout: post
title: "ClientCredential認証"
date: 2025-02-21 12:00:00 +0900
categories: [blog]
tags: [Nodejs]
excerpt: "Node.js"
---

OAuth認証のうち、<br>

- `Resource Owner Password Credentials Grant`
    - モバイルアプリで使われるらしいがよく分からない
- `Implicit Grant`
    - 危険なので非推奨
- `Authorization Code Grant`
    - `amplify + Cognito`や`Firebase Authentication`を使えばバックエンドで`/callback`エンドポイントすら作成せずにフロントエンドのみで完結できてしまう

というわけで最後の`Client Credentials`認証を少しだけ理解できたような気がするのでまとめてみます。

**目次**
* ToC
{:toc}

### Client Credentials認証の流れ
`Keycloak`の場合は以下のような手順になります。
- **レルムを作成**
- **クライアント登録**
  クライアントを作成し、`Client Credentials`フローを有効化。

- **クライアントIDとクライアントシークレットの取得** 
  登録したクライアントの設定から、`Client ID`と`Client Secret`を取得。

- **アクセストークンのリクエスト**  
  `Keycloak`のトークンエンドポイントにリクエストを送信。 
```bash
  POST /realms/<realm-name>/protocol/openid-connect/token
  Content-Type: application/x-www-form-urlencoded

  client_id=<client-id>&client_secret=<client-secret>&grant_type=client_credentials
```

- **APIリクエストの実行**  
  アクセストークンを使用して、保護されたリソースやAPIにアクセス。  
  例：  
```bash
  GET /protected-api
  Authorization: Bearer <access_token>
```
プログラムで実装する場合は、トークンが存在しない場合に`keycloak`から取得し、そのトークンをヘッダーに設定して他のマイクロサービスに送信するという流れになりそうです。
```typescript
  import axios from 'axios';

  let accessToken = null;

  async function getToken() {
  const response = await axios.post('https://<keycloak-server>/realms/<realm-name>/protocol/openid-connect/token', new URLSearchParams({
    client_id: '<client-id>',
    client_secret: '<client-secret>',
    grant_type: 'client_credentials'
  }));
  accessToken = response.data.access_token;
  return accessToken;
  }
  
  async function callMicroservice() {
  if (!accessToken) {
    await getToken();
  }

  const response = await axios.get('https://<microservice-url>/api/resource', {
    headers: {
      Authorization: `Bearer ${accessToken}`
    }
  });

  console.log(response.data);
  }
  
  callMicroservice();
```

- **サーバー側での検証**
サーバー側で`keycloak`の公開鍵エンドポイントから公開鍵を取得し、トークンを検証します。`keycloak`と`cognito`の公開鍵エンドポイントは以下のようになります。
```
- Keycloak公開鍵エンドポイント:  
  `https://<keycloak-server>/realms/<realm-name>/protocol/openid-connect/certs`

- Amazon Cognito公開鍵エンドポイント:  
  `https://cognito-idp.<region>.amazonaws.com/<user-pool-id>/.well-known/jwks.json`
```
`express`を使用したサンプルコードは以下のようになります。

```javascript
import express from 'express';
import axios from 'axios';
import jwt from 'jsonwebtoken';

const app = express();
const keycloakUrl = 'https://<keycloak-server>/realms/<realm-name>';
let publicKey = null;

// Keycloak公開鍵の取得
async function getPublicKey() {
  if (!publicKey) {
    const response = await axios.get(`${keycloakUrl}/protocol/openid-connect/certs`);
    const keys = response.data.keys;
    publicKey = keys[0].x5c[0]; // 最初の鍵を使用 (通常、複数の鍵がある場合)
  }
  return publicKey;
}

// JWTを検証するミドルウェア
async function verifyToken(req, res, next) {
  try {
    const token = req.headers['authorization']?.split(' ')[1];

    if (!token) {
      return res.status(401).send('Token is required');
    }

    const publicKey = await getPublicKey();
    const decoded = jwt.verify(token, `-----BEGIN CERTIFICATE-----\n${publicKey}\n-----END CERTIFICATE-----`);

    req.user = decoded; // ユーザー情報をリクエストオブジェクトに追加
    next();
  } catch (err) {
    return res.status(401).send('Invalid or expired token');
  }
}

// ミドルウェアを使用
app.use(verifyToken);
```

### まとめ
`EKS`など`k8s`環境のマイクロサービス間認証では、`IAM`や`k8s`の`ServiceAcount`を使用する方法や、`AppMesh`などサービスメッシュを使用した`mTLS`が主流らしいですが、どちらも環境構築の時点で難しすぎるので、手軽な`Client Credential`認証について学習してみました。
