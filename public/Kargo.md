---
title: プロモーションプラットフォーム「Kargo」を調査した
tags:
  - 'Kubernetes'
  - 'Kargo'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

株式会社スリーシェイクで2ヶ月間の期限付きインターンを行いました。インターンではKargoというKubernetesのプロモーションプラットフォームについて技術調査を行いました。

この記事では、Kargoについて紹介した後、インターン期間で発見したKargoのメリットを述べて、最後にKargoのデモを行います。


:::note info
記事化については、株式会社スリーシェイク様の許可を得ております。
:::


## Kargoとは

[Kargo](https://kargo.io/)は[Akuity](https://akuity.io/)が開発している、検証環境からテスト、本番環境へのデプロイまでのデプロイフローを実装、管理するためのプロモーションプラットフォームです。オープンソースで公開されており、[https://github.com/akuity/kargo](https://github.com/akuity/kargo) から確認することができます。

Kargoでは、Gitリポジトリの更新やk8sへの反映など、デプロイ時に行う様々な手順を”プロモーションタスク”とみなしており、Kargoはこのプロモーションタスクの実装・管理を支援するためのツールとして作られたと説明しています。

また、KargoはKargo単体で使用することができますが、Argo CDなどの他のツールと併用することも可能です。

Kargoを使用すると、使用するFreightを指定するだけでコンテナのバージョンアップやYAMLファイルの編集とGitリポジトリの変更、Gitリポジトリへのpushなどが自動で行えます。

## Kargoで使用する単語

Kargoの説明をするにあたって、Kargo独自の単語や役割について、説明していきます。

ここでは、WarehouseとStage、Promotion Step、Promotion Task、Promotion Templateについて取り上げます。

なお、紹介時に使用しているスクリーンショットはKargo v1.5.3を使用しているため、一部公式ドキュメントで紹介されているスクリーンショットとUIが異なる場合があります。

### Warehouse

Warehouseは、Gitリポジトリやコンテナイメージ、Helmチャートリポジトリなどを監視し、変更を検知したら新しいバージョン含めたFreightリソースを作成します。

以下はWarehouseの例です。このWarehouseは”guestbook”という名前で、Dockerイメージ”[guestbook](https://github.com/users/satsuki2200/packages/container/package/guestbook)”とGitリポジトリ”[kargo-advanced.git](https://github.com/satsuki2200/kargo-advanced.git)”を監視しています。

![warehouse.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3772590/085f0ebe-7282-4cb6-aa10-48dbd6996a85.png)

### Freight

デプロイ可能なバージョン群が列挙される場所です。Warehouseで検知されると、Freightリソースが作成されてFreightに出現します。

以下はFreightの例です。Warehouseの例と同じDockerイメージ・Gitリポジトリを使用しています。1時間前に作成された、Gitリポジトリのコミットが`eb05af2` でコンテナイメージタグが`v0.0.10` のFreightと、1分前に作成された、Gitリポジトリのコミットが`eb05af2` でコンテナイメージタグが`v0.0.11` のFreightがあります。各Freightの左側にある赤や緑などの色は、後述するStageの色と対応しており、Freightが各色に対応するStageで適用されていることを示しています。

![freight.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3772590/f080ddf0-206f-484d-a498-df74eb4613e1.png)

### Stage

Stageとは、プロモーションにおけるターゲットのことです。各Stageにおいて、新たなFreightがpromoteされた際に実行すべき処理を定義することができます。

なお、Stageはしばしば環境と捉えられることもあります。以下の画像はdev, staging, ab-test, prodの4つのStageを用意した際のイメージです。それぞれのStageは開発環境、ステージング環境、テスト環境、本番環境をイメージして作られています。なお、テスト環境はテストの種類に応じて、本番環境は各リージョンに対応する形で、Stage数が増えていることに注意してください。

![stages.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3772590/24a660e4-5bde-477b-9643-5d83eafa0715.png)

### Promotion Step

KargoにはPromotion Taskと呼ばれる、Stageにpromoteした際に実行する処理をまとめる機構があります。

Promotion Stepでは、GitリポジトリにコミットやPRを作成する、Argo CDを更新する、HTTPリクエストを送信するなど、promoteされた時に実行したい処理のパラメータなどを定義することができます。

以下はPromotion Stepの例です。このStepでは、[git-clone](https://docs.kargo.io/user-guide/reference-docs/promotion-steps/git-clone/)を実装しています。一つずつ見ていきます。

varsではrepoURLとしてリポジトリURLを変数化しています。次にstepsのusesで使用するStepを指定し、configでリポジトリURLを指定しています。また、checkoutの箇所ではcheckoutするコミットIDとブランチ名を指定し、それぞれをcheckoutするpathを指定しています。

```yaml:git-clone
  vars:
  - name: repoURL
    value: https://github.com/satsuki2200/kargo-advanced.git
  steps:
  - uses: git-clone
    config:
      repoURL: ${{ vars.repoURL }}
      checkout:
      - commit: ${{ commitFrom(vars.repoURL).ID }}
        path: ./src
      - branch: env/${{ ctx.stage }}
        path: ./out
        create: true
```

### Promotion Task

Promotion TaskではいくつかのPromotion Step処理をひとまとめにすることができ、Stageにpromoteされたときに実行したい処理が複数ある際に、あらかじめPromotion Taskで定義しておくことができます。

このTaskは再利用可能であるため、複数のStageで使用することが可能です。

以下はPromotion Taskの例です。なお、[こちら](https://github.com/satsuki2200/kargo-advanced/blob/main/kargo/promotiontasks.yaml)からも確認できます。このYAMLファイルでは、Promotion Stepの[git-clone](https://docs.kargo.io/user-guide/reference-docs/promotion-steps/git-clone/)と[git-clear](https://docs.kargo.io/user-guide/reference-docs/promotion-steps/git-clear/)、[kustomize-build](https://docs.kargo.io/user-guide/reference-docs/promotion-steps/kustomize-build/)、[git-commit](https://docs.kargo.io/user-guide/reference-docs/promotion-steps/git-commit/)、[git-push](https://docs.kargo.io/user-guide/reference-docs/promotion-steps/git-push)、[argocd-update](https://docs.kargo.io/user-guide/reference-docs/promotion-steps/argocd-update/)を定義して、それらを`promote` というTask名で定義しています。

```yaml:promotiontasks.yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: PromotionTask
metadata:
  name: promote
  namespace: kargo-advanced
spec:
  vars:
  - name: repoURL
    value: https://github.com/satsuki2200/kargo-advanced.git
  - name: branch
    value: main
  - name: image
    value: ghcr.io/satsuki2200/guestbook
  steps:
  # Clone the Git repository that contains the Kustomize configuration
  # to the ./src directory, and the environment configuration to ./out.
  - uses: git-clone
    config:
      repoURL: ${{ vars.repoURL }}
      checkout:
      - commit: ${{ commitFrom(vars.repoURL).ID }}
        path: ./src
      - branch: env/${{ ctx.stage }}
        path: ./out
        # Create the branch if it does not exist.
        create: true
  # Following a "rendered branch" pattern, we delete all files in the
  # ./out directory to ensure that we only have the files that are
  # rendered from the Kustomize configuration in the ./src directory
  # of the repository after being rendered.
  - uses: git-clear
    config:
      path: ./out
  # Update the image in the Kustomize configuration located at ./src/env/<stage>
  # in the repository.
  - uses: kustomize-set-image
    as: update-image
    config:
      path: ./src/env/${{ ctx.stage }}
      images:
      - image: ${{ vars.image }}
        tag: ${{ imageFrom(vars.image).Tag}}
  # Build the Kustomize configuration in the ./src directory using the env/<stage>
  # overlay to the ./out directory.
  - uses: kustomize-build
    config:
      path: ./src/env/${{ ctx.stage }}
      outPath: ./out
  # Commit the changes to the Git repository.
  - uses: git-commit
    as: commit
    config:
      path: ./out
      message: ${{ task.outputs['update-image'].commitMessage }}
  # Push the changes to the Git repository.
  - uses: git-push
    config:
      path: ./out
  # Request a sync of the ArgoCD Application to apply the changes from the
  # commit.
  - uses: argocd-update
    config:
      apps:
      - name: guestbook-${{ ctx.stage }}
        sources:
        - repoURL: ${{ vars.repoURL }}
          desiredRevision: ${{ task.outputs['commit'].commit }}
```

### Promotion Template

Promotion Templateでは、新しいk8sマニフェストやコンテナイメージをStageに導入する際実行するTaskを定義します。

また、呼び出すTaskに変数が必要な場合は、Promotion Templateで定義します。


## Kargoを使用することで得られるメリット

ここでは、私がインターン期間にKargoを調査して発見したメリットを説明します。

私が発見したメリットは以下です。

- デプロイフローをWeb GUIで可視化することができる
- Podに変更を反映する前に行いたい処理を自動化できる
- 開発環境から本番環境までのデプロイフローを自由に構成できる

それぞれについて、説明していきます。

今回は[kargo-advanced](https://github.com/akuity/kargo-advanced/)というKargo提供元が公開しているサンプルリポジトリを使用します。このリポジトリでは、大きく分けて4つのStageを含んでおり、dev、staging、ab-test、prodのデプロイフローで構成されています。ab-testは同じテストを値の異なる2種類で検証するためにテストA用とテストB用の計2つ、prodは本番環境に複数のリージョンがあることを想定して作られているため、eastリージョン、westリージョン、centralリージョンが用意されており、それぞれにStageが存在することに注意してください。

### デプロイフローの可視化

Gitリポジトリに変更を加えた後、本番環境にデプロイするまで、様々な環境での検証やテストが行われることでしょう。現在のステップは本番環境デプロイまであとどれくらいなのか、そして現在のステップが終わったら次に何のステップを行うのか、Argo CDでApplicationsを眺めているだけでは分かりません。

しかしKargoを使用することで、これらの問題は解消され、一箇所で可視化することができます。

以下はkargo-advancedについてArgo CDのApplicationsを表示した画像です。この画像を見ただけでは、最初にどのApplicationで検証を行うべきかわかりません。

![argocd_applications.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3772590/ddfa6a93-1742-4e8a-b975-5548ae1771c6.png)

しかしKargoを使用すると、この問題は解決します。以下はkargo-advancedについてKargoのProjectを表示した画像です。この画像を見ると、最初にdev環境で検証を行い、次にstaging、そしてab-testの順に検証すれば良いことがわかるでしょう。

![kargo_project.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3772590/1171821c-d192-42c3-b486-fde3a42c4e1f.png)

また、Kargoでは現在実行しているコンテナのバージョンやGitリポジトリのコミットをハッシュ値で表示することができます。これにより、dev環境ではv0.0.11のコンテナを実行しており、それ以外の環境ではv0.0.10のコンテナを実行していることも一目でわかります。

### 処理の自動化

Podで動かしているコンテナのバージョンをあげたい、リポジトリの内容に変更があったので反映したい。そのような時、Promotion Taskに行いたい処理を定義しておくことで、自動化することができます。

Gitブランチやコミットの作成、GitリポジトリにPRを作成することやYAMLファイルの更新、Argo CDの更新、HTTPリクエストの送信など、様々なことを行えます。

### デプロイフロー構成の自由度

本番環境までに幾つのステップを通過する必要があるか、どんなテストをどれだけ行うのか、それは内容によって異なるかと思います。

Kargoでは、Stage数や各Stageで行うPromotion Taskを自由に設定できます。例えば、dev環境とuat環境、そして5つのテスト環境を用意するといったことも可能です。

また、それぞれの環境で行うPromotion Taskも柔軟に変更することができます。すべての環境で異なるTaskを動作させることも可能です。

## Kargoデモ

AkuityはKargoの使用例として複数のリポジトリを公開しています。今回はその1つである[kargo-advanced](https://github.com/akuity/kargo-advanced)をforkしてプロモーションの検証を行いました。

:::note warn
手元で実行する際は各自でリポジトリをforkし、KargoがリポジトリにpushできるようGitHub Personal Access Tokenを発行してArgo CDとKargoのそれぞれにSecretとして追加してください。
:::


検証環境は以下のとおりです。

|    名称    |   バージョン   |
|:----------:|:-----------:|
| macOS      | v15.5       |
| Docker     | v27.4.0     |
| kind       | v0.27.0     |
| Argo CD    |  client：v3.0.0+e98f483 <br /> server：v3.0.6+db93798  |
| Argo Rollouts | v1.8.3   |
| cert-manager | v1.18.1   |
| Kargo      | v1.5.3.     |

環境構築は自作したシェルスクリプトを使用しました。

```bash:install.sh
#!/bin/bash

# Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Argo Rollouts
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# cert-manager
helm install cert-manager cert-manager \
  --repo https://charts.jetstack.io \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true \
  --wait

# Kargo
# Password is 'admin'
helm install kargo \
  oci://ghcr.io/akuity/kargo-charts/kargo \
  --namespace kargo \
  --create-namespace \
  --version 1.5.3 \
  --set api.adminAccount.passwordHash='$2a$10$Zrhhie4vLz5ygtVSaif6o.qN36jgs6vjtMBdM6yrU1FOeiAAMMxOm' \
  --set api.adminAccount.tokenSigningKey=iwishtowashmyirishwristwatch \
  --wait
```

環境構築とリポジトリforkを行った後、ArgoCDとKargoのWeb GUI操作画面にアクセスするためにポートフォワードを行います。これらはターミナルで別タブを開き、実行したままにしておいてください。

```bash
# Argo CDのポートフォワード
kubectl port-forward services/argocd-server -n argocd 8080:443

# Kargoのポートフォワード
kubectl port-forward --namespace kargo svc/kargo-api 3000:443
```

Webで[https://localhost:8080](https://localhost:8080)と[https://localhost:3000](https://localhost:3000)にアクセスすると、それぞれArgoCDとKargoのログイン画面が表示されます。

Kargoの初期パスワードは`admin` です。Argo CDの初期パスワードは以下のコマンドで確認できます。

```bash
argocd admin initial-password -n argocd
```

CLIもログインします。

```bash
# Argo CD CLI login
$ argocd login localhost:8080
WARNING: server certificate had error: tls: failed to verify certificate: x509: certificate signed by unknown authority. Proceed insecurely (y/n)? y
Username: admin
Password:
'admin:login' logged in successfully
Context 'localhost:8080' updated
$ 
```

```bash
# Kargo CLI login
$ kargo login https://localhost:3000 --admin --insecure-skip-tls-verify
? Admin user password *****
$ 
```

次にArgo CDのプロジェクトとApplications、Kargoのプロジェクトなどを作成します。

```bash
argocd proj create -f ./argocd/appproj.yaml
argocd appset create ./argocd/appset.yaml
kargo apply -f ./kargo
```

すべて実行した後に[Argo CDの管理画面](https://localhost:8080)にアクセスすると、guestbookから始まるApplicationが複数存在し、すべてStatusがUnknownになっています。また、[Kargoの管理画面](https://localhost:3000)にアクセスすると、`kargo-advanced`というProjectがあるので、クリックしてみます。すると、以下のようにWarehouseや複数のStageが並んでいます。

![kargo-advanced-pre.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3772590/904d1a84-8ba1-4fae-8475-7827cdc56b51.png)

試しに`0e74c71`と`v0.0.11`の組み合わせのFreightをdevにpromoteしてみます。dev右側のトラックボタン→Promoteを順に押下すると、Freightが選択できるため、`0e74c71`と`v0.0.11`の組み合わせをSelect、右下緑色のPromoteボタンを押下します。

![dev_select.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3772590/8217d4fb-79fe-46a0-9641-59a41b369c92.png)

するとすぐに様々な処理が実行されました。すべてが完了した後、GitHubリポジトリを確認すると`/env/dev`というブランチ内にKargoによるコミットが確認できました。また、Argo CDの`guestbook-dev`もStatusがSyncになっていることが確認できました。

![dev_promote.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3772590/aeecda82-8271-49d0-a780-b3304809e338.png)

![dev_commit.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3772590/0ad0ca4a-48f2-4f06-a3d0-f0b5319945b7.png)

promoteしたときに何が起こったのでしょうか？

先ほど述べた”様々な処理”はPromotion Task(複数のPromotion Stepの集合)によるものです。これらは`promotiontasks.yaml`に`promote`という名前で定義されており、kargo-advancedではすべてのStageにおいてFreightをpromoteする際に実行されます。このPromotion Taskでは、git-clone、git-clear、kustomize-set-image、kustomize-build、git-commit、git-push、argocd-updateが順に行われます。

この時、argocd-updateも行われるため、Argo CDを通じてNamespace`guestbook-dev` もSyncされ、Podが動き始めます。

```yaml:promotiontasks.yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: PromotionTask
metadata:
  name: promote
  namespace: kargo-advanced
spec:
  vars:
  - name: repoURL
    value: https://github.com/satsuki2200/kargo-advanced.git
  - name: branch
    value: main
  - name: image
    value: ghcr.io/satsuki2200/guestbook
  steps:
  # Clone the Git repository that contains the Kustomize configuration
  # to the ./src directory, and the environment configuration to ./out.
  - uses: git-clone
    config:
      repoURL: ${{ vars.repoURL }}
      checkout:
      - commit: ${{ commitFrom(vars.repoURL).ID }}
        path: ./src
      - branch: env/${{ ctx.stage }}
        path: ./out
        # Create the branch if it does not exist.
        create: true
  # Following a "rendered branch" pattern, we delete all files in the
  # ./out directory to ensure that we only have the files that are
  # rendered from the Kustomize configuration in the ./src directory
  # of the repository after being rendered.
  - uses: git-clear
    config:
      path: ./out
  # Update the image in the Kustomize configuration located at ./src/env/<stage>
  # in the repository.
  - uses: kustomize-set-image
    as: update-image
    config:
      path: ./src/env/${{ ctx.stage }}
      images:
      - image: ${{ vars.image }}
        tag: ${{ imageFrom(vars.image).Tag}}
  # Build the Kustomize configuration in the ./src directory using the env/<stage>
  # overlay to the ./out directory.
  - uses: kustomize-build
    config:
      path: ./src/env/${{ ctx.stage }}
      outPath: ./out
  # Commit the changes to the Git repository.
  - uses: git-commit
    as: commit
    config:
      path: ./out
      message: ${{ task.outputs['update-image'].commitMessage }}
  # Push the changes to the Git repository.
  - uses: git-push
    config:
      path: ./out
  # Request a sync of the ArgoCD Application to apply the changes from the
  # commit.
  - uses: argocd-update
    config:
      apps:
      - name: guestbook-${{ ctx.stage }}
        sources:
        - repoURL: ${{ vars.repoURL }}
          desiredRevision: ${{ task.outputs['commit'].commit }}
```

このように、KargoでpromoteするとGitリポジトリの変更やArgo CDのUpdateなどの処理を一括して自動で行うことができました。

同じ要領でstagingとab-test、prodについても順にpromoteしていきます。

ab-testでは、Argo Rolloutsを使用してテストを行っています。内容は特定のポケモンのbase_experienceを[PokeAPI](https://pokeapi.co/)で取得し、その値が一定以下ならテストをパスするというものです。ab-test-aではピカチュウ、ab-test-bではリザードンでテストを行っています。

元リポジトリのままだとab-test-bはテストをクリアすることができません。テストをクリアさせたい場合は、`stages.yaml` でポケモン名を変更するか、`analysis.yaml`の閾値を変更することでテストをクリアすることができます。

<details><summary>stages.yaml</summary>

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: ab-test-b
  namespace: kargo-advanced
  annotations:
    kargo.akuity.io/color: green
spec:
  requestedFreight:
  - origin:
      kind: Warehouse
      name: guestbook
    sources:
      stages:
      - staging
  promotionTemplate:
    spec:
      steps:
      - task:
          name: promote

  verification:
    analysisTemplates:
    - name: pokemon-xp
    args:
    - name: pokemon
      value: charizard # charizardを変更
```
</details>

<details><summary>analysis.yaml</summary>

```yaml:analysis.yaml
# This is a sample analysis template that uses the PokeAPI to
# check the base experience of a Pokemon is under threshold.
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: pokemon-xp
  namespace: kargo-advanced
spec:
  args:
  - name: pokemon
  metrics:
  - name: pokemon-xp
    interval: 5s
    count: 5
    successCondition: result < 200 # 200を変更
    failureLimit: 3
    provider:
      web:
        url: https://pokeapi.co/api/v2/pokemon/{{args.pokemon}}
        jsonPath: "{$.base_experience}"
```
</details>

<br>

prodは3つのStageがありますが、prodという名前のStageでpromoteを行おうとすると、これまで”Promote”と書かれていた箇所が”Promote to downstream”になっています。このPromote to downstreamは、prodに紐づいているprod-central、prod-east、prod-westの3つすべてに同じFreightをpromoteすることができます。1つずつpromoteするのはめんどくさいので、Promote to downstreamで一気にpromoteしてしまいましょう。

<br>
<br>

すべての環境においてFreightを適用することができました🎉

![kargo-advanced-finish.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3772590/d439631c-4a45-44d2-acb8-a044364e307f.png)

このように、Stageによって行いたい処理を決めることができるので、デプロイまでのフローを自由に構築することができます。

また、リポジトリに新たなコミットがされた時やコンテナイメージに新しいイメージタグが発生すると、新しいFreightが作られてパイプラインに登場します。

新たなFreightをデプロイしたい時は、先ほどと同じようにpromoteしていくことでデプロイすることができます。

## まとめ

今回はKubernetesのコンテナオーケストレーションツールであるKargoについて調査を行い、また簡単な検証を行いました。

また、拙い文章を読んでいただきありがとうございました。

まだまだKubernetesの知識やCDの知識が足りていないので、さらに研鑽していく所存です。

### 参考

Kargo HP [https://kargo.io/](https://kargo.io/)

Kargo Docs [https://docs.kargo.io/](https://docs.kargo.io/)

Kargo Core Concepts [https://docs.kargo.io/user-guide/core-concepts](https://docs.kargo.io/user-guide/core-concepts)

Warehouse [https://docs.kargo.io/user-guide/core-concepts/#warehouses](https://docs.kargo.io/user-guide/core-concepts/#warehouses)

Freight [https://docs.kargo.io/user-guide/core-concepts/#freight](https://docs.kargo.io/user-guide/core-concepts/#freight)

Stage [https://docs.kargo.io/user-guide/core-concepts/#stages](https://docs.kargo.io/user-guide/core-concepts/#stages)

Promotion Step [https://docs.kargo.io/user-guide/reference-docs/promotion-steps/](https://docs.kargo.io/user-guide/reference-docs/promotion-steps/)

Promotion Task [https://docs.kargo.io/user-guide/reference-docs/promotion-tasks/](https://docs.kargo.io/user-guide/reference-docs/promotion-tasks/)

Promotion Template [https://docs.kargo.io/user-guide/reference-docs/promotion-templates/](https://docs.kargo.io/user-guide/reference-docs/promotion-templates/)

kargo-advanced [https://github.com/akuity/kargo-advanced](https://github.com/akuity/kargo-advanced)