# 補足資料 Podmanの主要コマンド

---

## 注釈

本資料は2025/05段階のGemini 2.5 Proによる出力を基板とし、著者が修正を加える形で作成している。
スクリーンショット以外の画像で出典元が明記されていない場合、ChatGPTによる作成物である。

---

## 目的

この資料は、Podman を利用する上で頻繁に使用される主要なコマンドとその基本的な使い方をまとめたものです。日々のコンテナ操作やイメージ管理の際のリファレンスとしてご活用ください。

---

## はじめに

Podman のコマンドラインインターフェース (CLI) は、**Docker の CLI と高い互換性**を持つように設計されています。そのため、既に Docker を利用した経験がある方は、多くの場合 `docker` を `podman` に置き換えるだけで同様の操作が可能です。

ここでは、特によく使われるコマンドを用途別に分類し、具体的な使用例とともに紹介します。

---

## レジストリ操作コマンド

コンテナレジストリへのログイン/ログアウトやイメージ検索を行います。

- **レジストリへのログイン:** プライベートリポジトリなど、認証が必要なレジストリにログインします。

    ```
    podman login ghcr.io 
    podman login --authfile ~/.config/containers/auth.json quay.io
    podman login -u testuser -p testpassword localhost:5000
    # ユーザー名とパスワード (またはトークン) を求められます
    ```

- **レジストリからのログアウト:**

    ```
    podman logout docker.io
    podman logout --all # すべてのレジストリからログアウト
    ```

- **イメージの検索:** レジストリ上の公開イメージを検索します。

    ```
    podman search nginx
    podman search python --limit 10 # 結果を10件に制限
    ```


---

## イメージ操作コマンド

コンテナイメージの取得、作成、管理に関するコマンドです。

- **イメージの取得:** レジストリ（例: Docker Hub, Quay.io, ghcr.io）からイメージをダウンロードします。

    ```
    # 例: 最新のUbuntuイメージを取得(※取得先のレジストリ指定がない場合、選択を求められる事がある)
    podman pull ubuntu:latest
    # 例: 特定バージョンのAlpineイメージを取得
    podman pull alpine:3.17
    # 例: GitHub Packagesからイメージを取得 (要ログイン)
    podman pull ghcr.io/mygithubuser/my-image:v1.0
    # 例: armアーキテクチャ用のイメージ取得
    podman pull --arch=arm arm32v7/debian:stretch
    ```
- イメージの保存/ロード: ローカル環境でのイメージの保存と読み込みを行います。イメージのバックアップ目的のほか、インターネット接続が限られる環境でのイメージの運搬に持ちいられます。
    ```
    #例: 'my-app:latest' イメージを 'my-app-image.tar' という名前で保存
    podman save -o my-app-image.tar my-app:latest
    #例: 上記保存したイメージを読み込み
    podman load -i ./my-app-image.tar
    ```
- **イメージのビルド:** `Containerfile` (または `Dockerfile`) を元に新しいイメージを作成します。
    ```
    # カレントディレクトリのContainerfileを元にイメージ 'my-app:1.0' をビルド
    podman build -t my-app:1.0 .
    # 特定のContainerfileを指定してビルド
    podman build -t another-app -f ./path/to/MyContainerfile .
    ```

- **ローカルイメージ一覧表示:** ローカルに保存されているイメージの一覧を表示します。

    ```
    podman images
    # または
    podman image list
    ```

- **ローカルイメージの削除:** ローカルのイメージを削除します。コンテナで使用中のイメージは削除できません。

    ```
    # イメージIDで削除
    podman rmi <イメージID>
    # イメージ名とタグで削除
    podman rmi my-app:1.0
    # 複数のイメージをまとめて削除
    podman rmi <イメージID1> <イメージID2>
    ```

- **イメージへのタグ付け:** 既存のイメージに別の名前（リポジトリ名やタグ）を付けます。主にレジストリへのPush前に使用します。

    ```
    # 'my-app:1.0' に 'registry.example.com/my-repo/my-app:v1.0' というタグを付ける
    podman tag my-app:1.0 registry.example.com/my-repo/my-app:v1.0
    ```

- **イメージのPush:** ローカルのイメージを指定したレジストリにアップロードします。事前に `podman login` が必要です。

    ```
    # 例: tagによりイメージ名を“完全修飾”しておく
    podman push registry.example.com/my-repo/my-app:v1.0
    # 例: push 時に “宛先” を明示する
    podman push my-app:v1.0 registry.example.com/my-repo
    ```

### 補足 Podman pushにおけるイメージ名とPush先の関連について

コンテナのイメージ名は通常次の要素によって成り立ちます。
```[<transport>://][ホスト名[:ポート番号]/][PATH][:<タグ>|@<ダイジェスト>]```
※PATH例: ```[NAMESPACE/]REPOSITORY```

イメージ名がレジストリ名を含めて完全修飾されているとき、Podmanはイメージ名に含まれるレジストリをPush先とみなします。
イメージ名にレジストリが含まれず、かつPush時の宛先も指定されていない場合、Podmanは`/etc/containers/registries.conf` の `unqualified-search-registries` に記載のあるレジストリにPushしようと試みます。ただし、意図しないレジストリへのPushを防ぐため、イメージ名で完全修飾するか、Push時に宛先を明示することを推奨します。

---

## コンテナ操作コマンド

コンテナの実行、停止、管理に関するコマンドです。

- **コンテナの実行:** イメージから新しいコンテナを作成し、実行します。

    ```
    # Ubuntuイメージでbashを対話的に実行し、終了後コンテナを削除
    podman run -it --rm ubuntu bash

    # Nginxコンテナをバックグラウンドで実行し、ホストの8080番をコンテナの80番に接続、コンテナ名を 'my-web' に設定
    podman run -d -p 8080:80 --name my-web nginx:latest

    # ホストの './data' ディレクトリをコンテナの '/app/data' にマウント
    podman run -v ./data:/app/data my-app:1.0

    # 'my-pod' という名前のPod内でコンテナを実行
    podman run --pod my-pod my-sidecar-app:latest
    ```

    - **よく使うオプション:**
        - `-d`: バックグラウンド実行 (Detached)
        - `-it`: インタラクティブなTTYを確保 (対話モード)
        - `--rm`: コンテナ終了時に自動的に削除
        - `-p <ホストポート>:<コンテナポート>`: ポートフォワーディング
        - `-v <ホストパス>:<コンテナパス>[:オプション]`: ボリュームマウント
        - `--name <コンテナ名>`: コンテナに名前を指定
        - `--pod <pod名>`: 既存のPodに参加させる
- **コンテナ一覧表示:** 実行中のコンテナ、またはすべてのコンテナを表示します。

    ```
    # 実行中のコンテナを表示
    podman ps
    # 停止中のコンテナも含め、すべてのコンテナを表示
    podman ps -a
    ```

- **コンテナの停止:** 実行中のコンテナを停止します。

    ```
    podman stop <コンテナID または コンテナ名>
    ```

- **コンテナの開始:** 停止しているコンテナを開始します。

    ```
    podman start <コンテナID または コンテナ名>
    ```

- **コンテナの再起動:** コンテナを再起動します。

    ```
    podman restart <コンテナID または コンテナ名>
    ```

- **コンテナの削除:** コンテナを削除します。通常、停止しているコンテナのみ削除可能です。

    ```
    podman rm <コンテナID または コンテナ名>
    # 実行中のコンテナを強制的に削除 (-f オプション)
    podman rm -f <コンテナID または コンテナ名>
    ```

- **コンテナログの表示:** コンテナの標準出力/標準エラー出力を表示します。

    ```
    podman logs <コンテナID または コンテナ名>
    # ログをリアルタイムで追跡 (-f オプション)
    podman logs -f <コンテナID または コンテナ名>
    ```

- **コンテナ内でのコマンド実行:** 実行中のコンテナの中に入って、または指定したコマンドを実行します。

    ```
    # 'my-web' コンテナ内でbashを対話的に実行
    podman exec -it my-web /bin/bash
    # 'my-app' コンテナ内で 'ls /app' を実行
    podman exec my-app ls /app
    ```

- **ファイルコピー:** ホストとコンテナ間でファイルやディレクトリをコピーします。

    ```
    # ホストの 'local.txt' をコンテナ 'my-app' の '/tmp/' にコピー
    podman cp ./local.txt my-app:/tmp/
    # コンテナ 'my-app' の '/app/config.yaml' をホストのカレントディレクトリにコピー
    podman cp my-app:/app/config.yaml .
    ```

---

## Pod操作コマンド

Podman固有の機能であるPod（関連するコンテナのグループ）を管理するコマンドです。

- **Podの作成:** 新しいPodを作成します。Pod自体はコンテナを持ちませんが、ネットワーク名前空間などを共有するグループを定義します。

    ```
    # 'my-webapp-pod' という名前で、ポート8080を公開するPodを作成
    podman pod create --name my-webapp-pod -p 8080:80
    ```

- **Podの一覧表示:** 作成されたPodの一覧を表示します。

    ```
    podman pod list
    # または
    podman pod ps
    ```

- **Podの開始:** Pod内のすべてのコンテナを開始します。

    ```
    podman pod start <pod ID または pod名>
    ```

- **Podの停止:** Pod内のすべてのコンテナを停止します。

    ```
    podman pod stop <pod ID または pod名>
    ```

- **Podの削除:** Podを削除します。Pod内のコンテナも一緒に削除されます。

    ```
    podman pod rm <pod ID または pod名>
    ```

---

## Kubernetes連携コマンド

Kubernetes YAMLファイルを使用してPodやコンテナを管理するコマンドです。

- **Kubernetes YAMLからのPod/コンテナ作成:** 指定されたKubernetes YAMLファイル（PodやDeploymentなど）に基づいてPodman Podおよびコンテナを作成・実行します。

    ```
    # 'my-kube-manifest.yaml' ファイルからリソースを作成
    podman kube play my-kube-manifest.yaml

    # 既存のリソースがある場合に削除して再作成 (--replace オプション)
    podman kube play --replace my-kube-manifest.yaml

    # ネットワーク設定を指定して作成
    podman kube play --network=my-network my-kube-manifest.yaml
    ```

    - このコマンドにより、Kubernetesのマニフェストファイルを使ってPodman環境でアプリケーションを簡単にデプロイできます。

---
## その他

- **Podmanシステム情報の表示:** ホスト環境、ストレージ、レジストリなどの情報を表示します。

    ```
    podman info
    ```

- **バージョン情報の表示:** Podmanクライアントとサーバー（もしあれば）のバージョンを表示します。

    ```
    podman version
    ```


---

## コマンド実行時の注意点

- **Rootlessモード:** 一般ユーザーでPodmanを実行している場合、1024未満のポートを直接利用するには追加の設定が必要な場合があります。また、ホストのファイルシステムへのアクセス権限もユーザー権限に基づきます。
- **Dockerからの移行:** Dockerに慣れている方は、シェルの設定ファイル（`.bashrc`, `.zshrc`など）に `alias docker=podman` を追加しておくと、既存の `docker` コマンドをそのまま利用できて便利な場合があります。

---

## より詳しい情報

- 各コマンドには多くのオプションがあります。詳細は `--help` フラグで確認できます。

    ```
    podman --help
    podman run --help
    podman pod create --help
    ```

- **Podman公式ドキュメント:** [https://docs.podman.io/](https://docs.podman.io/)

これらのコマンドを使いこなすことで、Podmanによるコンテナ管理がより効率的になります。

## 参考

- [https://docs.podman.io/](https://docs.podman.io/)
- [Reference | Docker Docs](https://docs.docker.com/reference/)
- [3.2. コンテナーレジストリーの設定 | Red Hat Product Documentation](https://docs.redhat.com/ja/documentation/red_hat_enterprise_linux/8/html/building_running_and_managing_containers/con_configuring-container-registries_working-with-container-registries#con_configuring-container-registries_working-with-container-registries)
- [Dockerfile リファレンス — Docker-docs-ja 24.0 ドキュメント](https://docs.docker.jp/engine/reference/builder.html)
