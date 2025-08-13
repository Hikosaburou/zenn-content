---
title: "FastlyのRealtime Log Streamingで苦労したあれこれ"
emoji: "🪵"
type: "tech"
topics: ["fastly", "vcl", "bigquery"]
published: false
---

## これはなんですか

* FastlyのRealtime Log Streaming機能を使って試行錯誤したので、そこで知った細かいTipsを書きます。
* 以下のサンプルが広いユースケースをカバーしているので、これを読み解いて必要な要素をロギングすることをお勧めします。
    * [Copy of Comprehensive logging, one value per line - Fastly Fiddle](https://fiddle.fastly.dev/fiddle/89e5e296)

## やりたかったこと

* ワイルドカード表記のドメインでCDN Serviceをホストしています。
    * ex `*.myendpoint.example.com`
* ワイルドカードに当てはまるサブドメインごとにアクセスログを集計しようとしています。
    * アクセス数
    * データ転送量
    * Fastly Image Optimizer (IO) リクエスト数

## 前提

FastlyのLog Streamingによるログの記録はベストエフォートです。あくまで参考値である点に注意しましょう。

[About Fastly's real-time log streaming features | Fastly Documentation](https://www.fastly.com/documentation/guides/integrations/streaming-logs/about-fastlys-realtime-log-streaming-features/#how-real-time-log-streaming-works)

## BigQueryにFastlyのアクセスログを転送する

基本的にはドキュメントに書いてある通りに設定すればOKです。

[About Fastly's real-time log streaming features | Fastly Documentation](https://www.fastly.com/documentation/guides/integrations/streaming-logs/about-fastlys-realtime-log-streaming-features/#how-real-time-log-streaming-works)

TerraformでIAM Impersonation（アカウントに成り代わる権限を振ることで、キーを使わずに認証を通す方式）方式の設定をする際は、次のように`account_name`にサービスアカウント名（サービスアカウントを表すメールアドレスのユーザー名部分）を設定します。

[Configuring Google IAM service account impersonation to avoid storing keys on Fastly logging | Fastly Documentation](https://www.fastly.com/documentation/guides/integrations/streaming-logs/configuring-google-iam-service-account-impersonation-for-fastly-logging/)
[fastly_service_vcl | Resources | fastly/fastly | Terraform | Terraform Registry](https://registry.terraform.io/providers/fastly/fastly/latest/docs/resources/service_vcl#nestedblock--logging_bigquery)

```tf
  logging_bigquery {
    name               = "logging_to_bq_sample"
    project_id         = "your-gcp-project-id"

    # 'your-service-account-name@your-gcp-project-id.iam.gserviceaccount.com' のユーザー名部分
    account_name       = "your-service-account-name" 

    dataset            = "your_bq_dataset_name"
    table              = "your_bq_table_name"
    format             = "..."
  }
```

## Fastly Image Optimizerによる画像変換の有無を記録する

ドキュメントを追いきれなかったのでFastlyのサポートの方に聞きました。

結果、レスポンスヘッダの`Fastly-Stats`を使えば良いことがわかりました。

[Fastly-Stats | Fastly Documentation](https://www.fastly.com/documentation/reference/http/http-headers/Fastly-Stats/)

レスポンスヘッダはVCL上で`resp.http.{NAME}`でアクセスできるので、次のようにログフォーマットを設定しました。

```vcl:vcl_log
  {""fastly_is_transformed_by_io":"} if(resp.http.Fastly-Stats ~ "io=1", "true", "false") {", "}
```



## データ転送量集計用のメトリクスを記録する

こちらもFastlyのサポートの方に聞きました。
Websocketを使わない場合は次の2つの変数を使って上りと下りのデータ転送量を計算すればOKです。

* [bereq.bytes_written]()
    * バックエンドリクエスト全体のサイズ
* [resp.bytes_written]()
    * クライアントへのレスポンス全体のサイズ

## Shielding設定時の注意点

FastlyにはShieldingという機能があります。

[Shielding | Fastly Documentation](https://www.fastly.com/documentation/guides/getting-started/hosts/shielding/)

Fastlyはキャッシュサーバー群を[POP](https://www.fastly.com/documentation/guides/getting-started/concepts/using-fastlys-global-pop-network/)という単位でグループ化しています。Shieldingでは、指定したPOPがShield POPとなり、世界中のPOP（Edge POP）からのオリジンへのリクエストをShield POPが中継するという機能です。オリジンへのリクエストをShield POPが受け持つことにより、オリジンへのリクエスト回数を減らすことができるというメリットがあります。

一方で、Shieldingを有効にするとEdge POPとShield POPの両方で同じVCLが動くことになるため、ヘッダの操作やログ設定を工夫する必要が生じます。
また、シールド間の通信はFastlyの課金対象となるため、ログからデータ転送量を計算する際にはシールド間通信の有無を考慮する必要があります。

### ログをEdge POPでのみ記録する

Fastly VCLにはリクエストが通過したキャッシュサーバーの数を記録する変数として、`fastly.ff.visits_this_service`があります。

[fastly.ff.visits_this_service | Fastly Documentation](https://www.fastly.com/documentation/reference/vcl/variables/miscellaneous/fastly-ff-visits-this-service/)

この変数を使って、`vcl_log`サブルーチンでログを記録する条件を記述します。

[vcl_log | Fastly Documentation](https://www.fastly.com/documentation/reference/vcl/subroutines/log/#shielding-considerations)

```vcl:vcl_log
if (fastly.ff.visits_this_service == 0) {
  log "....";
}
```

こうすることで、シールド間通信が走った際にEdge POPでのみログを記録できます。

なお、Realtime Log StreamingにはConditionという設定が存在するので、そちらに条件式を書くことでも同様に設定できます。以下はTerraformの設定例です。

```terraform
  condition {
    name      = "log_only_on_edge"
    statement = "fastly.ff.visits_this_service == 0"
    type      = "RESPONSE"
  }

  logging_bigquery {
    name               = "log_to_bq"
    account_name       = "{{サービスアカウント名}}"
    project_id         = "{{Google CloudプロジェクトID}}"
    dataset            = "{{BQデータセット名}}"
    table              = "{{BQテーブル名}}"
    response_condition = "log_only_on_edge"
    format             = "{{ログフォーマット}}"
  }
```

### クライアントへのレスポンスから特定のヘッダを落とす

Shieldingが有効な場合、VCLはEdge POPとShield POPの両方で動作するため、`vcl_deliver`サブルーチンで単純に`unset resp.http.{NAME}`と書いてレスポンスヘッダを落とすと、Shield POPの時点でレスポンスヘッダが落ちてしまいます。Edge POPはShield POPのレスポンスを元に処理をするため、`resp.http.{NAME}`が使えなくなります。

レスポンスヘッダの値をログに使いつつクライアントへのレスポンスからのみ除外するには、次のように`fastly.ff.visits_this_service`変数を使って、Edge POPの場合のみレスポンスヘッダを落とすようにします。

```vcl
if (fastly.ff.visits_this_service == 0) {
  unset resp.http.log-origin;
}
```

### シールド間通信の有無を記録する

Edge POPとShield POP間のシールド間通信のデータ転送量はFastlyの課金対象です。したがって、ログからデータ転送量を集計する際には、シールド間通信の有無も記録する必要があります。シールド間通信の記録方法は、冒頭で紹介したサンプルがカバーしています。

[Copy of Comprehensive logging, one value per line - Fastly Fiddle](https://fiddle.fastly.dev/fiddle/89e5e296)

このサンプルは様々なログを記録しているため、シールド間通信の記録に必要な要素のみ追ってみます。まず、シールド間通信の有無は `fastly_is_shield` というフィールドで記録しています。 `vcl_log` サブルーチン内の該当箇所を抜粋します。

```vcl:vcl_log
  {""fastly_is_shield":"} if(req.http.log-origin:shield == server.datacenter, "true", "false") {", "}
```

このフィールドは、「リクエストがEdge POPを経由せず、直接Shield POPに到達した」場合に`true`となります。この判定には、`log-origin:shield`というカスタムヘッダと[`server.datacenter`変数](https://www.fastly.com/documentation/reference/vcl/variables/server/server-datacenter/)を使っています。`server.datacenter`はそのリクエストを処理するPOPのIDを表現します。

`log-origin:shield`カスタムヘッダがどのように定義されるかを追いかけます。`vcl_deliver`サブルーチンを見ると、次のように`log-origin`レスポンスヘッダの値を`log-origin`リクエストヘッダにコピーしています。

```vcl:vcl_deliver
if (resp.http.log-origin) {
  set req.http.log-origin = resp.http.log-origin;
}
```

先ほど出てきた`log-origin:shield`という表記は、`:`演算子を使って`log-origin`ヘッダーの`shield`サブフィールドにアクセスしていることを意味しています。したがって、`log-origin`をコピーすればサブフィールドである`shield`の値もコピーできます。

また、Fastly VCLではオリジンへのリクエスト以降、[リクエストヘッダ(`req.http.{NAME}`)](https://www.fastly.com/documentation/reference/vcl/variables/client-request/req-http/)の値は意味を持ちません。その性質を利用して、後続のサブルーチン(`vcl_deliver`,`vcl_log`)で変数のように使うことが多くあります。

`resp.http.log-origin`レスポンスヘッダを追いかけると、`vcl_fetch`サブルーチンで`beresp.http.log-origin`にアクセスしていることがわかります。

```vcl:fetch
if (req.backend.is_origin) {
  set beresp.http.log-origin:shield = server.datacenter;
}
```

`beresp`変数は後続のバックエンドから得たレスポンスを格納する変数です。`vcl_fetch`サブルーチンでのみ書き込みが可能で、その結果は後続の`vcl_deliver`、`vcl_log`サブルーチンで`resp`変数として利用できます。
[Variables in VCL | Fastly Documentation](https://www.fastly.com/documentation/reference/vcl/variables/#predefined-variables)

`vcl_fetch`サブルーチンの処理を見ると、[`req.backend.is_origin`変数](https://www.fastly.com/documentation/reference/vcl/variables/backend-connection/req-backend-is-origin/)が`true`の場合に`server.datacenter`変数の値を詰めています。これは、オリジンへのリクエストを行った場合に、VCLを実行しているPOPのIDを記録することを意味しています。

ここまでの内容を踏まえて、シールド間通信がある場合とない場合で`fastly_is_shield`の値がどうなるかを考えます。まず、リクエストを直接Shield POPで受け取ってオリジンまでリクエストを行う場合の処理順序を見ます。

1. Shield POP上でVCLにしたがってリクエストを処理し、`vcl_fetch`サブルーチンに到達する。
1. Shield POPはオリジンをバックエンドに持つため、`if(req.backend.is_origin) {...}`内の処理を実行し、`log-origin:shield`ヘッダに`server.datacenter`変数の値を詰める。
1. `log-origin`ヘッダは値を持つので、Shield POPは`vcl_deliver`サブルーチンの`if (resp.http.log-origin) {...}`を実行し、リクエストヘッダに`log-origin`ヘッダを設定する。
1. `vcl_log`サブルーチンで`log-origin:shield`リクエストヘッダの値を比較してログとして記録する。Shield POP上でのみ処理を行っているため`server.datacenter`の値は一致し、`true`となる。

次に、Edge POPからShield POPを経由してオリジンまでリクエストを行う場合の処理順序を見てみます。

1. Edge POP上でVCLにしたがってリクエストを処理する。
1. Edge POPからShield POPにリクエストを行う。
    * Shield POPでも同様にVCLにしたがってリクエストを処理する。前述の通り、Shield POPでは`resp.http.log-origin`が値を持つので、Edge POPへのレスポンスには`log-origin`ヘッダが含まれる。
1. Edge POP上で`vcl_fetch`サブルーチンに到達する。
1. Edge POPのバックエンドはShield POPであるため、Edge POPは`if(req.backend.is_origin) {...}`を実行せず、Shield POPが返した`beresp.http.log-origin`の値を保持する。
1. `log-origin`ヘッダは値を持つので、Edge POPは`vcl_deliver`サブルーチンで`if (resp.http.log-origin) {...}`を実行し、リクエストヘッダに`log-origin`ヘッダを設定する。
1. `vcl_log`サブルーチンで`log-origin:shield`リクエストヘッダの値を比較してログとして記録する。Edge POPとShield POPは異なる`server.datacenter`の値を持つので`false`となる。


## トラブルシュート用のAPIエンドポイント

Log Streamingがうまく動いているかどうかを確認するためのAPIがあります。トラブルシュートする際に便利です。

[Setting up remote log streaming | Fastly Documentation](https://www.fastly.com/documentation/guides/integrations/streaming-logs/setting-up-remote-log-streaming/#troubleshooting-common-logging-errors)


## 参考

そのほか、調査した中で読んだページを挙げます。

* [Fastlyについて知らないかもしれない30のこと – TravelBook Tech Blog](https://tech.travelbook.co.jp/posts/fastly-30-things/)
* [About Fastly VCL | Fastly Documentation](https://www.fastly.com/documentation/guides/full-site-delivery/fastly-vcl/about-fastly-vcl/#the-vcl-request-lifecycle)
* [VCL best practices | Fastly Documentation](https://www.fastly.com/documentation/guides/full-site-delivery/fastly-vcl/vcl-best-practices/#use-the--operator-for-cookies-or-other-header-subfields)
* [Clustering in VCL | Fastly Documentation](https://www.fastly.com/documentation/guides/full-site-delivery/fastly-vcl/clustering-in-vcl/)
* [Useful log formats | Fastly Documentation](https://www.fastly.com/documentation/guides/integrations/streaming-logs/useful-log-formats/)