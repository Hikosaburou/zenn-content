---
title: "gcloud components install でインストールしたコマンドのパスを通す(fish, homebrew)"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['gcp']
published: true
---

# これはなんですか

* `cbt` コマンドを使うための設定ログです。
* この設定をすることで、 `bq`, `gsutil` 等、`gcloud components install` でインストールしたコマンドが実行できるようになります。


# 前提・作業環境

* macOS Monterey バージョン 12.6.1
* fish shell version 3.5.1
* Homebrew 3.6.15

gcloud は Homebrew でインストール済みです。

```sh
$ which gcloud
/opt/homebrew/bin/gcloud
```

# 手順

## cbtのインストール

[ドキュメント](https://cloud.google.com/bigtable/docs/cbt-overview?hl=ja)に従って `gcloud components install` コマンドでインストールします。

```sh
$ gcloud components update
$ gcloud components install cbt
```

## コマンドのパスを確認する

`gcloud` コマンドのパスは `which` で確認できます。Homebrewで入れた場合はシンボリックリンクが張られているので、 `ls` コマンドで確認します。

```sh
$ ls -l (which gcloud)
lrwxr-xr-x  1 user.name  admin  74  5 11  2022 /opt/homebrew/bin/gcloud@ -> /opt/homebrew/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/bin/gcloud
```

`gcloud components install` でインストールしたコマンドは `gcloud` と同じパス（ドキュメントでは google-cloud-sdk/bin と説明されている）に配置されています。

```sh
$ ls /opt/homebrew/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/bin/
anthoscli*                bq*                       dev_appserver.py*         gcloud*                   gke-gcloud-auth-plugin*   java_dev_appserver.sh*
bootstrapping/            cbt*                      docker-credential-gcloud* git-credential-gcloud.sh* gsutil*
```

## パスを通す

環境変数 `PATH` に上記のパスを追加します。

```sh($HOME/.config/fish/config.fish)
... (略)

fish_add_path /opt/homebrew/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/bin/
```

## 動作確認

コマンドが実行できることを確認します。

```
$ cbt listinstances
2022/12/22 14:00:29 -creds flag unset, will use gcloud credential
2022/12/22 14:00:29 -project flag unset, will use gcloud active project
2022/12/22 14:00:30 missing -project
```

# 参考

[cbt ツールの概要  |  Cloud Bigtable ドキュメント  |  Google Cloud](https://cloud.google.com/bigtable/docs/cbt-overview?hl=ja)

以下の通り環境変数の記載があります。

> 注: gcloud CLI をホーム ディレクトリ以外のディレクトリにインストールする場合は、PATH 環境変数に google-cloud-sdk/bin へのパスを設定する必要があります。