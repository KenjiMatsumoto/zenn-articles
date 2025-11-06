---
title: "【60億円流出】GitHub Token流出から始まったAPI改ざんとその対策"
emoji: "🔐"
type: "tech"
topics: ["CICD", "github", "AWS", "Token", "security"]
published: false
publication_name: omakase
---

## 前回の振り返り

[前回の記事](https://zenn.dev/omakase/articles/b2e85e51ca45ef)では、ステーキングプロバイダーの API レスポンスが実行時に改ざんされ、利用者がそれに気づかず署名してしまったことで資産が流出してしまった、というインシデントについて整理しました。具体的には、Unstake のトランザクションに悪意のある権限移譲の処理が追加されていて、それをデコードして確認しないまま署名してしまったのが原因でした。なので、「署名する前にデコードして中身を確認する」という対策などについて記載しました。

ただ、前回は「なぜ API が改ざんされてしまったのか」という一番大事な根本原因が判明していませんでした。今回は、その後にステーキングプロバイダーから公開された[続報](https://www.kiln.fi/post/kiln-sabisunozai-jia-dong-oyobisekiyuriteiinsidentoniguan-suruqing-bao)を基に、攻撃の全貌を掘り下げていきます。インフラエンジニアの GitHub Access Token が漏洩し、そこから CI/CD パイプラインを乗っ取られ、最終的に本番で動いている API が**実行時に改ざん**されてしまった経緯を解説します。

## 根本原因はGitHub Access Tokenの漏洩

続報で明らかになった一番の根本原因、それは**インフラエンジニアのGitHub Access Tokenが漏洩したこと**でした。
Token がどうやって漏洩したか、その具体的な経路までは「現時点では、従業員の GitHub access token がどのように侵害されたのかについての決定的な証拠は得られていません。」と記載されており、特定までには至っていませんが、「**盗まれた Token → CI/CD の悪用 → 本番環境への侵入**」という攻撃の主要な流れは判明しています。

この記事では、漏洩した Token がどのように使われて、最終的に本番 API の**実行時改ざん**という恐ろしい事態に至ったのか。そのプロセスを、公式発表で分かっている事実と、そこから考えられる推測を切り分けながら、図を交えて分かりやすく解説していきます。

## 流出したTokenでの攻撃経路

では早速、漏洩した GitHub Access Token がどのように悪用され、資産流出に至ったのか、その具体的な流れを図と一緒に見ていきましょう。

![攻撃経路フロー図](/images/validator-security-incident-2025/攻撃経路フロー図.png)

この図の流れを、各ステップごとにもう少し詳しく見ていきます。

1. **GitHub Access Token 漏洩**
   まず全ての始まりは、インフラエンジニアの GitHub Access Token が何らかの形で攻撃者の手に渡ってしまったことである。漏洩した Token の種別（例えば Personal Access Token なのか、OAuth アプリの Token なのか）は公式発表では言及されていないが、これが攻撃の起点となった。この Token の種別については、後ほど詳しく考察する。

2. **CI実行の悪用**
   Token を手に入れた攻撃者は、それを使って Infrastructure as Code (IaC)が管理されている Repository にアクセスした。そして、**新しいブランチを作成し、すぐに削除する**という操作をしている。なぜこんなことをしたかというと、ブランチの作成や削除をトリガーにして CI/CD のワークフローを実行させるためである。加えて、その際に大量のファイルを変更することで、ブランチが作られたことや必要箇所の変更などを分かりにくくするという隠蔽工作までしていた。

3. **Secrets収集・クラウド横展開**
   悪意を持って実行された CI/CD ワークフローの中から、攻撃者は環境変数などに保存されていた**クラウド環境の認証情報（AWSやGCPのAPIキーなど）を盗み出した。** これにより、GitHub の Repository への改変だけでなく、本番サーバーが動いているクラウド環境へと侵入することが可能になった。

4. **Kubernetes侵入**
   盗んだクラウドの認証情報を使い、攻撃者は本番の API サーバーが稼働している Kubernetes クラスタに侵入した。そして、API が動いている Pod（Pod は Kubernetes でアプリが動く最小単位）に対して、悪意のあるプログラム（ペイロード）を直接注入した。ここが今回の攻撃の非常に高度な部分で、サーバー上のファイルやコンテナイメージを直接書き換えるのではなく、**メモリ上で動いているプロセスに直接介入した**と推測される。これにより、検知が非常に困難となってしまった。

5. **APIロジックを改ざん**
   悪意のあるプログラムによって、API のロジックが実行中に書き換えられてしまった。その結果、前回の記事で解説したように、ユーザーが「Unstake（ステーキング解除）」をリクエストした際に、正常なトランザクションに加えて、**攻撃者のものにWithdraw権限を移譲する悪意のあるトランザクションが追加で返される**ようになってしまった。

6. **署名**
   最後に、ユーザー（今回はステーキングプロバイダーの顧客企業）は、API から返された改ざん済みのトランザクションの中身を十分に確認しないまま署名してしまった。これにより、意図せず攻撃者へ資産の引き出し権限を与えてしまい、資産流出へと繋がった、という流れである。ちなみに、ステーキングプロバイダーはインシデント発生前から、安全のため署名前にトランザクションをデコードし、検証することを推奨していたと続報には記載されている。

## 今回の攻撃手法への対策と前提知識

ここまでで大まかな攻撃の流れが見えてきました。では、我々エンジニアは、このような攻撃からシステムをどのようにして守ればいいのでしょうか。

ここからは、具体的な対策の話に入っていきます。ただ、その前に、対策をより深く理解するための前提知識として、以下の 2 点について先に整理しておきましょう。

1. GitHub Token の種別
2. Token が漏洩する一般的なパターン

その上で、以下の 2 つの観点から見ていきます。

1. Token を漏洩させないための対策
2. 万が一漏洩してしまった場合に被害を最小限に食い止めるための対策

### 前提知識

まずは、今回の事件を理解するうえで欠かせない 2 つの前提知識を確認していきましょう。

#### GitHub Tokenの種別

今回漏洩した Token の種別は特定されていません。そこで、今回の攻撃に使われた可能性のある GitHub Token を、可能性の高そうな順に見ていきましょう。それぞれの Token がどんなもので、どんな権限を持っているのかを知ることが、対策を考えるうえで非常に重要になります。

| Token種別                       | Prefix | 特徴                                                                                               | 有効期限                                     | 考えられる漏洩時の影響範囲                                                                 |
| ------------------------------- | ------ | -------------------------------------------------------------------------------------------------- | -------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **Personal Access Token (PAT)** | `ghp_` | **【最有力】** 個人ユーザーに紐づく。リポジトリの読み書き、CI/CDのトリガーなど、幅広い操作が可能。 | **無期限**も設定可能（現在は期限設定が推奨） | 非常に広い。トークンに与えられたスコープ（権限）によっては、ほぼ全ての操作ができてしまう。 |
| OAuth App の Access Token       | `gho_` | 外部のSaaSやツールが、ユーザーの代わりにGitHubを操作するために使用する。                           | 通常は有効期限あり（リフレッシュ可能）       | アプリに許可された範囲の操作に限られるが、CI/CD関連の権限があれば同様の攻撃が可能。        |
| GitHub App User OAuth Token     | `ghu_` | ユーザーがGitHub Appを認可した際に発行される、ユーザーに紐づくトークン。                           | 短命（8時間）、リフレッシュ可能              | Appに許可されたリポジトリと権限に限定される。                                              |
| GitHub App Install Access Token | `ghs_` | GitHub Appが特定のリポジトリにインストールされた際に発行される、App自身に紐づくトークン。          | 短命（最大1時間）                            | Appに許可されたリポジトリと権限に限定される。                                              |

#### 補足：GITHUB_TOKEN

GitHub Actions のワークフロー実行中に自動で発行される `GITHUB_TOKEN` というものもあります。これは有効期限がジョブ終了までと非常に短く、権限もそのリポジトリ内に限定されているため、今回の攻撃のように長期間にわたって外部から侵入するようなシナリオには使いにくく、可能性は低いと考えられます。

今回のケースでは、インフラエンジニア個人の Token が漏洩したとされていること、そして攻撃者がリポジトリを横断して活動している可能性を考えると、**もっとも可能性が高いのは Personal Access Token (PAT)**と推測されます。特に、もしその PAT が無期限で、広い権限（`repo` や `workflow` スコープ）を持っていたとしたら、攻撃者にとっては非常に好都合な状況だったと考えられます。

#### Token が漏洩する一般的なパターン

次に、Token がどうやって漏洩してしまうのか、その一般的なパターンを見ていきましょう。原因が分からないことには、対策の立てようがありません。ここでは、特にありがちな 8 つのパターンを挙げています。

1. **誤コミット/READMEや設定ファイルへの混入**
   これが一番よくあるパターン。うっかり Token をソースコードや設定ファイル、README に書いてしまい、そのまま `git push` してしまうケース。

2. **CI/CDのログ・アーティファクトからの漏洩**
   デバッグのために CI/CD の実行ログに Token を出力してしまったり、ビルド成果物（アーティファクト）に Token が含まれてしまったりするケース。

3. **Actionsの権限/トリガー設計の不備**
   外部のコントリビューターが送ってきた Pull Request を無条件に信頼して実行してしまったり、`pull_request_target` のような強力なトリガーを安易に使ってしまったりすることで、悪意のあるコードに Secrets を盗まれるケース。

4. **セルフホストRunnerの隔離不備**
   自前で CI/CD の実行環境（セルフホスト Runner）を運用している場合に、複数のジョブで環境が共有されていて、あるジョブから別のジョブの Token が盗めてしまうようなケース。

5. **端末側の侵害（開発者マシンのマルウェア/ブラウザセッション乗っ取り）**
   開発者の PC がマルウェアに感染し、クリップボードの中身やブラウザに保存されている情報、SSH エージェントなどが盗まれ、そこから Token が漏洩するケース。今回の事例も、この可能性は否定できない。

6. **サードパーティ連携（OAuth/GitHub App/Webhooks）の侵害**
   連携している外部の SaaS やツールのセキュリティが甘く、そこから Token が漏洩したり、Webhook の署名検証を怠ったために偽のイベントを受け入れてしまうケース。

7. **コンテナ/イメージ/キャッシュ層に残置**
   コンテナイメージをビルドする際に、イメージのレイヤーやビルドキャッシュに Token が残ってしまい、後からそれを再利用した人に見られてしまうケース。

8. **チケット/チャット/スクリーンショット共有からの漏洩**
   意外と見落としがちだが、Issue や Pull Request のコメント、Slack などのチャットツールにデバッグ目的で Token を貼り付けてしまったり、Token が写り込んだスクリーンショットを共有してしまったりするケース。

もちろん、これ以外にもパターンは無限に考えられますが、まずはこの 8 つのコアなパターンを意識するだけでも、セキュリティレベルは格段に上がるはずです。

### Tokenを漏洩させないための対策

ここからが本題の対策編です。まずは「Token を漏洩させない」ための予防策を、先ほどの漏洩パターンと対応させながら見ていきましょう。

1. **誤コミット/README・設定混入対策**
   - **Secret scanningとPush protectionを有効化する**: GitHub が提供している機能で、リポジトリにプッシュされたコードをスキャンして、Token らしき文字列があったら警告・ブロックしてくれる。これは必ず有効にすべきである。
   - **`.gitignore` と `pre-commit` フックを活用する**: そもそも Token を Git の管理対象に含めないように `.gitignore` を正しく設定し、さらに `gitleaks` のようなツールを `pre-commit` フックに仕込んで、コミットする前にローカルで検知・ブロックする仕組みも有効である。

2. **CI/CD ログ・アーティファクト対策**
   - **ログにTokenを出力しない**: GitHub Actions の `::add-mask::` というコマンドを使えば、特定の文字列をログ上で `***` のようにマスクできる。また、デバッグでよく使う `set -x` のようなコマンドは、意図せず環境変数を展開してしまう可能性があるので、本番のワークフローでは使用を禁止するルールを設ける必要がある。
   - **アーティファクトとキャッシュは最小限に**: ビルド成果物（アーティファクト）やキャッシュには、本当に必要なものだけを含めるようにし、保持期間も短く設定（YAML でも `retention-days` で設定可能）する。使い終わったら速やかに削除する運用も重要である。

3. **Actions 権限/トリガー不備対策**
   - **外部からのPull Requestは手動承認を必須に**: Organization の設定で、フォークされたリポジトリからの Pull Request でワークフローが自動実行されないようにし、必ずレビュワーの承認を必須にする。これにより、悪意のある PR によって Secrets が盗まれるリスクを大幅に減らせる。

4. **セルフホストRunner 隔離対策**
   - **Runnerは使い捨て（Ephemeral）で運用する**: セルフホスト Runner を使う場合は、ジョブごとに新しい環境を起動し、ジョブが終わったら破棄する「使い捨て」モデルが理想である。また、本番環境にデプロイする Runner と、開発中の PR をテストする Runner は、ネットワークや権限を完全に分離する。

5. **端末侵害対策**
   - **SSOと2FA（多要素認証）を必須にする**: GitHub へのログインは、パスワードだけでなく、SSO（シングルサインオン）や 2FA を全開発者に強制する。
   - **EDRを導入する**: EDR (Endpoint Detection and Response) ツールを導入し、万が一マルウェアに感染しても、その活動を早期に検知・対応できる体制を整える。
   - **認証情報を安易に保存しない**: ブラウザのパスワードマネージャーやクリップボード、SSH エージェントなどに Token を安易に保存しない、という基本的なセキュリティ意識も重要である。

6. **サードパーティ連携対策**
   - **OAuth/GitHub Appのベストプラクティスを遵守する**: 連携アプリを開発・利用する際は、PKCE フローの利用、リダイレクト URI の厳格な固定、Webhook の署名検証などセキュリティのベストプラクティスを徹底する。

7. **コンテナ/イメージ/キャッシュ残置対策**
   - **マルチステージビルドと `--mount=type=secret` を活用する**: Dockerfile でマルチステージビルドを使い、最終的なイメージに不要なビルド時の情報が残らないようにする。また、ビルド時に Token が必要な場合は、`--mount=type=secret` オプションを使うことで、イメージレイヤーに Token が残るのを防げる。

8. **チケット/チャット/スクリーンショット対策**
   - **とにかく貼らない、写さない**: これはもう運用と文化の話になるが、「認証情報は絶対にパブリックな場所へは共有しない」というルールを徹底し、もし誤って共有してしまった場合は**即座にそのTokenを無効化（ローテーション）する**フローを確立しておくことが重要である。

### 万が一漏洩してしまった場合に被害を最小限に食い止めるための対策

どんなに予防策を講じても、攻撃を 100%防ぐことは不可能です。そこで重要になるのが、「万が一 Token が漏洩してしまっても、被害を最小限に食い止める」ための対策、いわゆる**多層防御**の考え方です。

#### 1. 本番に影響を与えるCI/CDの経路を限定する

攻撃者は盗んだ Token を使って CI/CD を悪用しました。ならば、その CI/CD から本番環境へ至る道を、できるだけ狭く、そして監視の行き届いたものにしておくことが重要です。具体的には、以下の 5 つの対策が非常に有効です。

![CICD経路限定フロー図](/images/validator-security-incident-2025/CICD経路限定フロー図.png)

- **SecretsはEnvironmentを利用して分離し、レビュワーの承認を強制する**
  GitHub Actions の**Environments**という機能を使うと、「本番環境用」「ステージング環境用」といったように、環境ごとに Secrets を分離できる。さらに、「`production` 環境へのデプロイは、セキュリティチームの誰かが承認しないと実行できない」といった**承認ルール**を設定できる。これにより、たとえ攻撃者が CI/CD をトリガーできても、承認者のチェックを突破しない限り本番環境には手出しできなくなる。
  _設定方法: リポジトリの `Settings` > `Environments` で `production` 環境を作成し、`Required reviewers` に承認者を設定する。_

- **権限最小化（`permissions: read` をDefault設定し、必要時のみ権限昇格）**
  ワークフローに与える `GITHUB_TOKEN` の権限は、デフォルトでは読み取り専用（`permissions: read-all`）にしておき特定のジョブで書き込み権限が必要な場合にのみ、そのジョブ単位で権限を昇格させる（例: `permissions: contents: write`）ようにする。これにより、ワークフロー全体が乗っ取られたとしても、被害の範囲を限定できる。
  _設定方法: Organizationまたはリポジトリの `Settings` > `Actions` > `General` の `Workflow permissions` で `Read repository contents and packages permissions` を選択する。_

- **リポジトリシークレットに本番情報を置かない**
  リポジトリ単位の Secrets は、そのリポジトリのコラボレーターなら誰でもワークフローから使えてしまう可能性がある。本番環境の認証情報のような重要情報は、リポジトリの Secrets には置かず、前述の**Environments**の Secrets に格納して、厳格にアクセスを管理する。

- **外部ActionのコミットSHA固定**
  ワークフローで利用するサードパーティ製の Action は、`actions/checkout@v4` のようにタグで指定するのではなく、`actions/checkout@a12a3943b4bde6ff22b8f99495b33c59aed7c3d2` のように**コミットハッシュで固定**する。これにより、Action の作者が悪意のあるバージョンをタグに上書きする、といったサプライチェーン攻撃のリスクを低減できる。
  _設定方法: Organizationの `Settings` > `Actions` > `General` で `Allow Omakase, and select non-Omakase, actions and reusable workflows` を選択し、`Require actions to be pinned to a full-length commit SHA` にチェックを入れる_

- **`push: branches: [main]` などに限定してSecretsを渡す**
  本番環境の Secrets を利用できるワークフローは、`main` ブランチへの push など、本当に信頼できるイベントによってのみトリガーされるように厳しく制限し、開発中のブランチから安易に本番環境の Secrets へアクセスできないようにすることが重要である

これらの設定を盛り込んだ Sample YAML を以下に示します。

```yaml
name: Secure Deploy

on:
  push:
    branches:
      - main # 5. mainブランチへのpushでのみトリガー

# 2. デフォルトは読み取り権限のみ
permissions:
  contents: read

jobs:
  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    # 1. production環境を指定し、承認を強制
    environment:
      name: production
      url: https://example.com

    # OIDCを使うため、id-tokenの書き込み権限をジョブ単位で付与
    permissions:
      id-token: write

    steps:
      - name: Checkout
        # 4. コミットSHAでActionを固定
        uses: actions/checkout@a12a3943b4bde6ff22b8f99495b33c59aed7c3d2

      - name: Deploy to Production
        env:
          # 3. EnvironmentのSecretを利用
          APP_CONFIG: ${{ secrets.APP_CONFIG }}
        run: |
          echo "Deploying to production..."
          # ./deploy.sh
```

#### 2. 長期シークレットの撤廃（OIDCの活用）

そもそも、PAT のような有効期限の長い「長期シークレット」の存在自体が大きなリスクです。可能な限り、有効期限の短い「短期シークレット」に置き換えていきましょう。そのための**切り札となる対策がOIDC (OpenID Connect) 連携**です。

![長期シークレット撤廃フロー図](/images/validator-security-incident-2025/長期シークレット撤廃フロー図.png)

OIDC 連携を使えば、クラウドの長期的なアクセスキーを GitHub の Secrets に保存する必要がなくなります。ワークフローが実行されるたびに、クラウド側から有効期限の短い一時的な認証情報が発行され、それを使って安全にクラウドを操作できます。これは現代の CI/CD セキュリティにおける**最重要対策の一つ**です。

以下は、AWS と OIDC 連携する場合のワークフローの例です。

```yaml
name: Deploy to AWS with OIDC

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write # OIDCに必須
      contents: read

    steps:
      - name: Checkout
        uses: actions/checkout@a12a3943b4bde6ff22b8f99495b33c59aed7c3d2

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionRole # 事前にAWSで作成したIAM Role
          aws-region: ap-northeast-1

      - name: Deploy to S3
        run: |
          aws s3 sync ./dist s3://my-production-bucket
```

#### 3. GitHub監査ログの常時転送＆検知

本稿のインシデントでは、攻撃者はブランチを大量に作成・削除して CI をトリガーしました。このような異常なアクティビティを早期に検知するためには、GitHub の監査ログを常時 SIEM（Security Information and Event Management）ツールや Slack などに転送し、監視する仕組みを構築しましょう。

![Github監視フロー図](/images/validator-security-incident-2025/Github監視フロー図.png)

特に、以下のようなイベントはアラートの対象とすべきです。

- ブランチの作成・削除が短時間に大量発生
- `workflow_dispatch`（手動実行）の急増
- 新しい Runner の登録や削除
- Secrets の更新

これをやっていれば、今回のケースのような攻撃の兆候を、侵入の初期段階で掴めた可能性があります。

#### 4. 署名前デコードと不正トランザクションの検知

最後に、これは前回の記事の復習になりますが、たとえ API が改ざんされてしまっても最終的な署名の段階でユーザーが中身をしっかり確認すれば、被害は防げます。特に暗号資産のトランザクションのように、一度署名したら取り消せない操作については、「**デコードされていない署名は絶対にしない**」を徹底し、必ず人間が読める形にデコードして意図しない操作が含まれていないかを確認するプロセスを挟むことが最後の砦として非常に重要です。

## 補足：その他の防御策

ここまで、今回のインシデントに直接関連する対策を見てきましたが、別の攻撃パターンに備えるための防御策も簡単に触れておきます。

### `.github/workflows/` の変更にセキュリティ承認を必須化

今回の攻撃では、攻撃者はワークフローファイル自体を改ざんすることはしませんでした。しかし、**別の攻撃シナリオでは、ワークフローファイルを書き換えてSecretsを盗み出す**という手法も十分に考えられます。そのため、`.github/workflows/` ディレクトリ配下のファイル変更には、必ずセキュリティチームの承認を必須とする設定を入れておくことを推奨します。

_設定方法:_

1. リポジトリのルートに `CODEOWNERS` ファイルを作成し、以下のように記述する

   ```text
   # .github/workflows/配下のファイルは @organization/security-team の承認が必須
   /.github/workflows/ @organization/security-team
   ```

2. リポジトリの `Settings` > `Branches` で `main` ブランチの保護ルールを開き（または `Settings` > `Rules` > `Rulesets` で新規作成し）、`Require a pull request before merging` と `Require review from Code Owners` を有効にする

これにより、ワークフローを改ざんして Secrets を盗み出そうとするような攻撃を、Pull Request のレビュー段階で防ぐことができます。今回のケースには直接効果はありませんでしたが、**多層防御の一環として、ぜひ導入しておきたい対策**です。

## まとめ

今回は、GitHub Access Token の漏洩を起点とした API 改ざんインシデントを題材に、その攻撃手法と具体的な対策を深掘りしてみました。

今回の攻撃が非常に巧妙だったのは、単にサーバーに侵入するだけでなく、CI/CD パイプラインという「システムの作り方」そのものを悪用し、さらにファイルなどを残さずメモリ上で直接 API のロジックを書き換えるという検知が非常に困難な手法を取っていた点です。

この記事で紹介した対策をまとめると、以下のようになります。

- **予防策（Tokenを漏洩させない）**: Secret scanning の有効化、`pre-commit` フックの導入、ログへのマスキング、外部 PR の手動承認、Runner の分離、SSO/2FA の必須化など、基本的なセキュリティ対策を徹底する。
- **被害最小化策（漏洩しても大丈夫なようにする）**: CI/CD の経路を Environment や承認ルールで厳格に管理し、OIDC を活用して長期シークレットを撤廃する。そして、監査ログで異常を監視し、最後の砦として署名前のデコードを徹底する。

セキュリティに「絶対」はありません。だからこそ、1 つの対策に頼るのではなく、複数の防御策を組み合わせる**「多層防御」**の考え方が何よりも重要です。今回のインシデントを他人事と捉えず、ぜひ皆さんの開発現場でも、セキュリティ対策を見直すきっかけにしていただければ幸いです。

## 参考リンク

<!-- textlint-disable ja-technical-writing/sentence-length -->

[1][Kiln サービスの再稼働およびセキュリティインシデントに関する情報](https://www.kiln.fi/post/kiln-sabisunozai-jia-dong-oyobisekiyuriteiinsidentoniguan-suruqing-bao)
[2][【60億円流出】Solanaステーキングで何が起きたのかーAPI悪用の可能性と対策](https://zenn.dev/omakase/articles/b2e85e51ca45ef)
[3][GitHub Docs: Secret scanning](https://docs.github.com/ja/code-security/secret-scanning/about-secret-scanning)
[4][GitHub Docs: Using environments for deployment](https://docs.github.com/ja/actions/deployment/targeting-different-environments/using-environments-for-deployment)
[5][GitHub Docs: Security hardening for GitHub Actions](https://docs.github.com/ja/actions/security-guides/security-hardening-for-github-actions)
[6][GitHub Docs: About code owners](https://docs.github.com/ja/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)

<!-- textlint-enable ja-technical-writing/sentence-length -->
