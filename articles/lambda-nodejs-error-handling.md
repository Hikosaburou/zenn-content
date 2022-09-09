---
title: "AWS Lambda (Node.js) で同期実行時にカスタムエラーを投げる場合はnameプロパティも変えると良い"
emoji: "🥃"
type: "tech"
topics: ["aws", "nodejs", "lambda"]
published: true
---

# これはなんですか

* 表題の通りです。
* AWS Lambda (Node.js) を同期実行した際のエラーに含まれる `errorType` は[Errorオブジェクト](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Error)の `name` プロパティを参照します。


# AWS Lambda (Node.js) のエラーメッセージ

AWS Lambdaは同期実行をサポートしています。例えば [ドキュメント](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/invocation-sync.html) に記載のある通り、AWS CLIで `aws lambda invoke` コマンドを実行すると同期的にLambda Functionが実行されて、Functionの実行結果が取得できます。

```bash: Lambdaの同期実行例
aws lambda invoke --function-name my-function --cli-binary-format raw-in-base64-out --payload '{ "key": "value" }' response.json
```

Node.jsのLambda Functionを同期実行した場合、実行中にエラー終了すると、以下の例のようなメッセージが返却されます。([ドキュメント](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/nodejs-exceptions.html))

```json: エラーの例
{
    "errorType": "ReferenceError",
    "errorMessage": "x is not defined",
    "trace": [
      "ReferenceError: x is not defined",
      "    at Runtime.exports.handler (/var/task/index.js:2:3)",
      "    at Runtime.handleOnce (/var/runtime/Runtime.js:63:25)",
      "    at process._tickCallback (internal/process/next_tick.js:68:7)"
    ]
}
```

ここで、`errorType` はエラーの種類、`errorMessage` はエラー本文、`trace` はエラーのトレースです。

# `errorType` を使ってハンドリングしたいが……

Lambda Functionを同期実行して後続の処理でエラーハンドリングをするとき、 `errorType` を見て処理を変えたくなります。`errorType` はエラーの種類によって変わるようなので、動作確認用に以下のLambda Functionを作成します。

```js
class CustomError extends Error {
    constructor(message) {
        super(message);
    }
}

exports.handler = async (event) => {
    throw new CustomError('this is a error');
};
```

上記のFunctionを実行して得られたレスポンスは以下の通りです。

```json
{
  "errorType": "Error",
  "errorMessage": "this is a error",
  "trace": [
    "CustomError: this is a error",
    "    at Runtime.exports.handler (/var/task/index.js:9:11)",
    "    at Runtime.handleOnceNonStreaming (file:///var/runtime/index.mjs:1028:29)"
  ]
}
```

`errorType` はErrorのままです。


## CustomErrorの `name` 属性を設定する

どうやら、 `errorType` はErrorオブジェクトの `name` 属性を見ているようでした。Lambda Functionを以下のように書き換えます。

```javascript
class CustomError extends Error {
    constructor(message) {
        super(message);
        this.name = 'CustomError';
    }
}

exports.handler = async (event) => {
    throw new CustomError('this is a error');
};
```

実際のレスポンスは以下の通りです。

```json
{
  "errorType": "CustomError",
  "errorMessage": "this is a error",
  "trace": [
    "CustomError: this is a error",
    "    at Runtime.exports.handler (/var/task/index.js:9:11)",
    "    at Runtime.handleOnceNonStreaming (file:///var/runtime/index.mjs:1028:29)"
  ]
}
```

`"errorType": "CustomError"` となり、 `errorType` の値が変わっていることが確認できました。


# まとめ

AWS Lambda (Node.js) のエラーハンドリングで `errorType` を使う場合は、カスタムエラーオブジェクトの `name` 属性を変えると良いです。


# 参考

* [Node.js の AWS Lambda 関数エラー - AWS Lambda](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/nodejs-exceptions.html)
* [Error - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Error)