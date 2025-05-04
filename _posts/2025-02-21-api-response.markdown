---
layout: post
title: "APIのレスポンス形式を統一する"
date: 2025-02-21 12:00:00 +0900
categories: [blog]
tags: [FastAPI, Nodejs]
excerpt: "FastAPIやNode.jsでレスポンス形式を統一する"
---

APIレスポンスの形式を統一する方法を考えてみます。

**目次**
* ToC
{:toc}

### FastAPIの場合

FastAPIはエンドポイントの`response_model`のDTOクラスの構造をもとにOASドキュメントを生成しますが、ミドルウェアを使って、レスポンス後に返却するJSONの構造を変更すると上手く表示されないのでDTOを使って実装してみます。

#### DTOの生成
適当な`Item`と汎用的な`ResponseDTO`を作成します。`ResponseDTO`も本来ならステータスコードや、エラーならエラーメッセージを詳細に含めるべきですが、簡略化します。

```python
from typing import TypeVar, Generic, List
from fastapi import FastAPI
from pydantic import BaseModel

# 総称型の定義
T = TypeVar('T')  # Tは任意の型

class ResponseDTO(Generic[T], BaseModel):
    success: bool
    data: T
    message: str

# レスポンス用のDTOクラス
class Item(BaseModel):
    id: int
    name: str
```

#### エンドポイントの作成
こちらも本来ならクエリパラメータなどを使ってページネーションを実装し、それらのメタデータもレスポンスに含めるべきですが簡略化します。
```typescript
app = FastAPI()

@app.get("/items", response_model=ResponseDTO[List[Item]])
async def get_items():
    items = [
        Item(id=1, name="Item 1"),
        Item(id=2, name="Item 2")
    ]
    return ResponseDTO(success=True, data=items, message="Items fetched successfully")
```

### Expressの場合
Expressでも一応`Response`の型定義にDTOを使うこともできますが、OASを使わないし、ミドルウェアで作成してみます。この際、`X-Forwarded-For`ヘッダーを消し、エラーレスポンスは標準の`RFC7807`形式で返却するように実装してみます。以下がサンプルコードです。

```typescript
import { Request, Response, NextFunction } from 'express';

// 成功時のレスポンス形式
interface SuccessResponse {
  status: string;
  data: any;
}

// エラーレスポンスのRFC7807形式
interface ErrorResponse {
  type: string;
  title: string;
  status: number;
  detail: string;
}

const responseMiddleware = (req: Request, res: Response, next: NextFunction) => {
  const originalJson = res.json;

  res.json = (body: any) => {
    // X-Forwarded-Forを消す
    res.removeHeader('X-Forwarded-For');

    if (res.statusCode >= 200 && res.statusCode < 300) {
      // 成功時のレスポンス形式
      const successResponse: SuccessResponse = {
        status: 'success',
        data: body,
      };
      return originalJson.call(res, successResponse);
    } else {
      // 失敗時のRFC7807形式
      const errorResponse: ErrorResponse = {
        type: 'about:blank',
        title: 'Error',
        status: res.statusCode,
        detail: body.message || 'Unknown error',
      };
      return originalJson.call(res, errorResponse);
    }
  };

  next();
};

export default responseMiddleware;
```

### まとめ
上記は単純な例ですが、一度レスポンス形式を統一する処理を書いておけば繰り返し使えそうです。
