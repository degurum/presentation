# コンテナ技術入門 (Podman編) とJavaアプリデモ

---

## 注釈

本資料は2025/05段階のGemini 2.5 Proによる出力を基板とし、著者が修正を加える形で作成している。
スクリーンショット以外の画像で出典元が明記されていない場合、ChatGPTによる作成物である。

---

## 本日のゴール

- コンテナ技術の基本的な概念を理解する。
- **Podman** を使ってコンテナを操作する基本を学ぶ。
- 簡単なJavaアプリケーションをビルドし、イメージを作成、**AWS ECR** にPushし、コンテナとして動作させる手順を体験する。

---

## 対象者

- コンテナ技術について初めて学ぶ方
- アプリケーション開発の効率化、モダンなデプロイ手法に興味がある方
- Docker以外のコンテナツール（特にPodman）に関心がある方

---

## 従来の開発・デプロイの課題

- **「自分の環境では動くのに...」**
    - 開発環境と本番環境の違い（OS、ライブラリ、ミドルウェアのバージョンなど）による問題が頻発。
- **環境構築に時間がかかる**
    - 新しいメンバーが加わるたび、新しいサーバーを準備するたびに環境構築が必要。
- **リソースの無駄**
    - 物理サーバーや仮想マシンは、OSごと起動するため、起動が遅く、リソース消費が大きい。

---

## そこで登場！コンテナ技術

- アプリケーションとその実行に必要な依存関係（ライブラリ、設定ファイルなど）を**ひとまとめ**にして、**隔離された環境**で動かす技術。
- 「コンテナ」という軽量な箱に入れて、どこでも同じように動かせるようにします。
このことを｢**可搬性**(Portability)｣と言います。

![[コンテナ技術図示.png]](https://github.com/degurum/presentation/blob/main/container/podman/img/%E3%82%B3%E3%83%B3%E3%83%86%E3%83%8A%E6%8A%80%E8%A1%93%E5%9B%B3%E7%A4%BA.png?raw=true)

---

## 仮想マシンとの違い

| **特徴**    | **コンテナ**               | **仮想マシン (例: VMware, VirtualBox)** |
| --------- | ---------------------- | --------------------------------- |
| **隔離レベル** | プロセスレベル                | OSレベル                             |
| **起動速度**  | 速い (秒単位)               | 遅い (分単位)                          |
| **リソース**  | 軽量 (ホストOSの**カーネル**を共有) | 重量 (ゲスト**OS**が必要)                 |
| **用途**    | アプリケーションの実行環境          | OS全体の仮想化                          |


---

## コンテナのメリット

- **可搬性:** 
    - 開発環境、テスト環境、本番環境で同じコンテナを動かせる。環境差異の問題を解消。
- **軽量・高速:**
    - 適切な設計の元では起動が速くリソース消費が少ない。1台のサーバーでより多くのアプリを分離し実行可能。
- **スケーラビリティ:**
    - コンテナはVMと比較して軽量であるため、サーバー上でコンテナ化させた複数のアプリケーションを並列に動作させることが容易。

---

## コンテナ技術: Podmanを使ってみよう

- Dockerと互換性のあるコマンドラインインターフェースを持つ、**デーモンレス**なコンテナエンジン。
- Red Hat社が開発を主導しており、エンタープライズ環境での利用も増えています。

**なぜPodmanか？**

1. **デーモンレスアーキテクチャ:**
    - Dockerのように常駐する管理デーモンが不要。
    - コンテナはユーザープロセスとして直接起動されるため、システムへの影響が少なく、管理がシンプル。
    - デーモンを介さないため、ユーザーごとの操作やリソース管理が独立し、責任範囲が明確になりやすい。
2. **セキュリティ (Rootless):**
    - 一般ユーザー権限でのコンテナ実行（Rootlessモード）が容易かつ標準的。セキュリティリスクを低減。
3. **Kubernetes (k8s) との親和性:**
    - Pod（複数のコンテナのグループ）の概念をサポート。
    - k8sのマニフェストファイルを生成する機能など、連携がスムーズ。
4. **開発元の信頼性:**
    - Red Hat社による開発とサポートは、特にエンタープライズ利用において安心感があります。

---

**Dockerとの主な違い:**

- **デーモンの有無:** Podmanはデーモンレス、Dockerはデーモン（dockerd）が必要。
- **コマンド:** ほとんどの基本コマンド（`run`, `build`, `push`, `pull`など）は互換性がありますが、一部異なるオプションやコマンドも存在します (`docker` を `podman` に置き換えて使えることが多い)。

注釈: 開発現場ではいまだDockerが標準ですが、今後の潮流として十分Podmanが主軸となる可能性を期待しています。基本的には同じ様に動作するため今回はPodmanを採用しましたが、Dockerを利用いただいても問題ありません。

---

## Podmanの基本要素

1. **コンテナイメージ (Image):**
    - コンテナの「設計図」。Dockerイメージと互換性があります。
2. **Containerfile:**
    - コンテナイメージの作成手順を記述したファイル。`Dockerfile` という名前でも認識されますが、Podmanでは `Containerfile` が推奨されます。構文はDockerfileとほぼ同じです。
3. **コンテナ (Container):**
    - イメージを実行した「実体」。隔離されたプロセスとして動作。
4. **コンテナレジストリ (Registry):**
    - コンテナイメージを保存・共有する場所。
    - Docker Hub、Quay.io、そして今回利用する **AWS Elastic Container Registry (ECR)** など、様々なレジストリを利用できます。

---

## デモ: JavaアプリケーションをPodmanとAWS ECRで扱う

これから、簡単なWebサーバー機能を持つJavaアプリケーションをビルドし、Podmanでイメージを作成、AWS ECRにPushし、コンテナとして実行するデモを行います。

---

## デモのステップ

1. **準備:**
    - [ ] **Podman** のインストール
    - [ ] **AWSアカウント** と **AWS CLI** のインストールおよび設定 (認証情報、デフォルトリージョン)
    - [x] **ECRリポジトリ:** イメージをPushするためのECRリポジトリを事前に作成しておく。 (例: `my-java-app-repo`)
    - [x] **IAM権限:** 使用するAWSユーザーまたはロールにECRへのPush権限 (`ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:CompleteLayerUpload`, `ecr:InitiateLayerUpload`, `ecr:PutImage`, `ecr:UploadLayerPart` など) が必要。
    - **Javaアプリケーション:** (例: `demo` ディレクトリに配置、Maven or Gradle)
2. **Containerfileの作成:** Javaアプリをビルド・実行する手順を記述。
3. **Podmanイメージのビルド:** `podman build` コマンドでイメージ作成。
4. **ローカルでのコンテナ起動確認:** `podman run` コマンドで動作確認。
5. **AWS ECRへのログイン:** `aws ecr get-login-password` コマンドと `podman login` を組み合わせて認証。
6. **イメージへのタグ付け:** ECRの命名規則 (`<AWSアカウントID>.dkr.ecr.<リージョン>.amazonaws.com/<リポジトリ名>:<タグ>`) に合わせて `podman tag` でタグ付け。
7. **AWS ECRへのイメージPush:** `podman push` コマンドでアップロード。
8. **(任意) Pushしたイメージからの起動確認:** ローカルイメージを削除し、`podman pull` & `podman run` で再確認。

---

## 準備 (補足)

- **Podmanのインストール:**
    - Windows: Podman Desktop Installer または WSL2 上で Linux ディストリビューションのパッケージマネージャを利用
    - Linux (Fedora/CentOS/RHEL): `sudo dnf install podman`
    - Linux (Debian/Ubuntu): `sudo apt update && sudo apt install podman`
- **AWS CLIの設定:**
    - `aws configure` コマンドでアクセスキー、シークレットキー、デフォルトリージョンを設定します。
- **ECRリポジトリの作成:** AWSマネジメントコンソールやAWS CLI (`aws ecr create-repository --repository-name my-java-app-repo`) で作成します。
- **Javaアプリケーション:** 

---

## Containerfileの作成

プロジェクトのルートディレクトリに `Containerfile` という名前でファイルを作成します。

内容は[アプリケーションリポジトリ](https://github.com/degurum/podman_traning)を確認してください。


---

## Podmanイメージのビルド

ターミナルを開き、`Containerfile` があるディレクトリで以下のコマンドを実行します。

Bash

```
# podman build -t <イメージ名>:<タグ> .
# 例: イメージ名を 'my-java-app-podman', タグを 'latest' とする
podman build -t my-java-app-podman:latest .
```

- `-t`: イメージ名とタグを指定します。
- `.`: Containerfileがある現在のディレクトリを指定します。

---

## ローカルでのコンテナ起動確認

ビルドしたイメージをローカルで実行してみます。

Bash

```
# podman run -d --rm -p <ホストのポート>:<コンテナのポート> <イメージ名>:<タグ>
# 例: ホストの8080をコンテナの8080にマッピング、コンテナ終了時に自動削除(--rm)
podman run -d --rm -p 8080:8080 my-java-app-podman:latest
```

- `-d`: バックグラウンド実行
- `--rm`: コンテナ停止時に自動で削除
- `-p 8080:8080`: ポートマッピング

Webブラウザで http://localhost:8080 にアクセスし、"Hello from Podman Container!" と表示されるか確認します。

確認後、podman ps でコンテナIDを確認し、podman stop <コンテナID> で停止します (--rm をつけていれば停止と同時に削除されます)。

---

## AWS ECRへのログイン

AWS ECRにイメージをPushするために、Podmanでログインします。AWS CLIを使って一時的な認証トークンを取得し、それをPodmanに渡します。

```
# aws ecr get-login-password --region <リージョン> | podman login --username AWS --password-stdin <AWSアカウントID>.dkr.ecr.<リージョン>.amazonaws.com
# 例: リージョンが ap-northeast-1, AWSアカウントIDが 457227455307 の場合
aws ecr get-login-password --region ap-northeast-1 | podman login --username AWS --password-stdin 457227455307.dkr.ecr.ap-northeast-1.amazonaws.com
```

- AWS CLIが認証情報を取得し、そのパスワードを標準入力経由 (`--password-stdin`) で `podman login` に渡します。ユーザー名は常に `AWS` です。
- このコマンドを実行する前に、AWS CLIが正しく設定され、ECRへのアクセス権限があることを確認してください。

---

## イメージへのタグ付け (AWS ECR用)

AWS ECRへPushするイメージには、Push先のリポジトリを示すために以下の形式でタグを付けます。

`<AWSアカウントID>.dkr.ecr.<リージョン>.amazonaws.com/<リポジトリ名>:<タグ>`

```
# podman tag <ローカルイメージ名>:<タグ> <ECR用イメージ名>:<タグ>
# 例: アカウントID 457227455307, リージョン ap-northeast-1, リポジトリ名 my-java-app-repo の場合
podman tag my-java-app-podman:latest 457227455307.dkr.ecr.ap-northeast-1.amazonaws.com/my-java-app-repo:latest
```


---

## AWS ECRへのイメージPush

タグ付けしたイメージをAWS ECRにPushします。

リポジトリ作成
```
aws ecr create-repository --repository-name my-java-app-repo --region ap-northeast-1
```

```
# podman push <ECR用イメージ名>:<タグ>
podman push 457227455307.dkr.ecr.ap-northeast-1.amazonaws.com/my-java-app-repo:latest
```

- `<AWSアカウントID>`, `<リージョン>`, `<リポジトリ名>` はご自身の環境に合わせて置き換えてください。
- Pushが完了すると、AWSマネジメントコンソールのECRリポジトリ内で確認できます。

---

## (任意) Pushしたイメージからの起動確認

正しくPushされたか、またレジストリからPullできるかを確認します。

1. **ローカルイメージの削除 (任意):**

    ```
    podman rmi my-java-app-podman:latest
    podman rmi 457227455307.dkr.ecr.ap-northeast-1.amazonaws.com/my-java-app-repo:latest
    ```

2. **AWS ECRからのPull:** (事前に再度 `aws ecr get-login-password ... | podman login ...` が必要になる場合があります。ECRの認証トークンは通常12時間以内有効です)

    ```
    podman pull 457227455307.dkr.ecr.ap-northeast-1.amazonaws.com/my-java-app-repo:latest
    ```

3. **Pullしたイメージでコンテナ起動:**

    ```
    podman run -d --rm -p 8080:8080 457227455307.dkr.ecr.ap-northeast-1.amazonaws.com/my-java-app-repo:latest
    ```

4. ブラウザで `http://localhost:8080` を確認し、コンテナを停止 (`podman stop`) します。

---

## Lambdaで公開する

[Lambda](https://ap-northeast-1.console.aws.amazon.com/lambda/home?region=ap-northeast-1#/begin)から、関数の作成を選択。

以下のパラーメーターで作成を行います。

- 関数名: my-java-app
- コンテナイメージURI: GUI上から先ほどアップロードしたイメージを指定
- アーキテクチャ: x86_64
- アクセス権限>デフォルトの実行ロールの変更>既存のロールを使用する
    - 既存のロール: lambda_basic_execution

画面遷移後、設定>関数URLから認証タイプNONEで保存をしてください。
その後表示される関数URLでコンテナが公開されています。

ただし、このままだとLambda側のリソースが不足しています。
そのため設定>一般設定からメモリを 512 MB、タイムアウトを10秒に変更します。


---
## まとめ

- **Podman** はデーモンレスでセキュア、k8sとの親和性も高いコンテナエンジン。
- `Containerfile` (または `Dockerfile`) でコンテナイメージの構築手順を定義できる。
- 基本的なコマンドはDockerと互換性があり、移行も比較的容易。
- **AWS ECR** はスケーラブルで高可用性なコンテナイメージレジストリ。AWS CLI経由で認証し `podman push` 可能。
- Docker Hubの代替として、開発ワークフローに適したレジストリを選択することが重要。

---

## 今後の学習ステップ

- **Podの管理:** Podman固有の機能であるPodを使い、関連する複数のコンテナ（例: WebアプリとDB）をまとめて管理・実行する方法を学ぶ。`podman pod create`, `podman run --pod <pod名>` など。
- **Kubernetes (k8s):** コンテナオーケストレーションのデファクトスタンダード。Podmanで作成したイメージや、`podman generate kube` で生成したマニフェストファイルを使い、k8sクラスタ上でのデプロイ、スケーリング、管理方法を学ぶ。
- **CI/CD連携:** AWS CodePipeline, GitHub Actions, JenkinsなどのCI/CDツールと連携し、ソースコードの変更をトリガーに、自動でテスト、イメージビルド、AWS ECRへのPush、さらにはECSやEKSへのデプロイなどを行うパイプラインを構築する。

---
## 参考資料
- [コンテナ化とは? - コンテナ化の説明 - AWS](https://aws.amazon.com/jp/what-is/containerization/)
- [Podman とは？をわかりやすく解説](https://www.redhat.com/ja/topics/containers/what-is-podman)
- [コンテナランタイムとしての AWS Lambda - 変化を求めるデベロッパーを応援するウェブマガジン | AWS](https://aws.amazon.com/jp/builders-flash/202402/lambda-container-runtime/)
- [AWS Lambda 関数の設定 - AWS Lambda](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/lambda-functions.html)

## ご清聴ありがとうございました

**質疑応答**

## 補足　IAM権限: 使用するAWSユーザーまたはロールにECRへのPush権限

今回はIAM Identity Centerにて研修用のログインアカウントを作成している。
許可セットについては信頼可能なユーザーのため広めに設定しており次の権限を割り当てた。

`aws configure` コマンドでアクセスキー、シークレットキー、デフォルトリージョンの設定は省略しているが、IAM Identity Centerであればログイン後にアクセスキーを容易に確認できる。

- AmazonEC2ContainerRegistryFullAccess
- AmazonEC2ReadOnlyAccess
- AmazonS3FullAccess
- AWSLambda_FullAccess

## 補足 実行環境

WSL2環境のUbuntu(24.04)で動作確認済み。

aws cliおよびpodmanが動作すれば良い。
Podman Desktopは利用していないため、任意の環境で実行可能。
本資料を説明時にはEC2上のUbuntuを利用する想定。
