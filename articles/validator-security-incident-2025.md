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

本記事では、ステーキングプロバイダーから公開された[続報](https://www.kiln.fi/post/kiln-sabisunozai-jia-dong-oyobisekiyuriteiinsidentoniguan-suruqing-bao)をもとに、「なぜ API が改ざんされてしまったのか」という根本原因について、その攻撃の全貌を掘り下げていきます。

## 根本原因はGitHub Access Tokenの漏洩

続報で明らかになった一番の根本原因、それは**エンジニアのGitHub Access Tokenが漏洩したこと**でした。
現時点では、エンジニアの GitHub access token がどのように漏洩したのか、その具体的な漏洩経路までは特定されていません。しかし、「**盗まれた Token → CI/CD の悪用 → 本番環境への侵入**」という攻撃の主要な流れは判明しています。それではステーキングプロバイダーの続報を元に、その攻撃のプロセスについて深掘りしていきましょう。

## 流出したTokenでの攻撃経路

では早速、漏洩した GitHub Access Token がどのように悪用され、資産流出に至ったのか、その具体的な流れを図と一緒に見ていきましょう。

![攻撃経路フロー図](/images/validator-security-incident-2025/攻撃経路フロー図.png)

この図の流れを、各ステップごとにもう少し詳しく見ていきます。

<!-- textlint-disable no-mix-dearu-desumasu -->
<!-- textlint-disable ja-technical-writing/no-mix-dearu-desumasu -->

1. **GitHub Access Token 漏洩**

   まず全ての始まりは、エンジニアの GitHub Access Token が何らかの形で攻撃者の手に渡ってしまったことです。漏洩した Token の種別（例えば Personal Access Token なのか、OAuth アプリケーションの Token なのか）は公式発表では言及されていませんが、これが攻撃の起点となりました。この Token の種別については、後ほど詳しく考察していきます。

2. **CI実行の悪用**

   Token を手に入れた攻撃者は、それを使って Infrastructure as Code (IaC)が管理されている Repository にアクセスしました。そして、**新しいブランチを作成し、すぐに削除する**という操作を行なっています。なぜこんなことをしたかというと、ブランチの作成や削除をトリガーにして CI/CD のワークフローを実行させるためです。加えて、その際に大量のファイルを変更することで、ブランチが作られたことや必要箇所の変更などを分かりにくくするという隠蔽工作までしていました。

3. **Secrets収集・クラウド横展開**

   悪意を持って実行された CI/CD ワークフローの中から、攻撃者は環境変数などに保存されていた**クラウド環境の認証情報（AWSやGCPのAPIキーなど）を盗み出しました。** これにより、GitHub の Repository への改変だけでなく、本番サーバーが動いているクラウド環境へと侵入することが可能となってしまいました。

4. **Kubernetes侵入**

   盗んだクラウドの認証情報を使い、攻撃者は本番の API サーバーが稼働している Kubernetes クラスタに侵入しました。そして、API が動いている Pod（Pod は Kubernetes でアプリケーションが動く最小単位）に対して、悪意のあるプログラム（ペイロード）を直接注入しています。ここが今回の攻撃の非常に高度な部分です。サーバー上のファイルやコンテナイメージを直接書き換えるのではなく、実行中のコンテナ内で悪意のあるプロセスを起動します。そこから iptables や eBPF などを使ってネットワークトラフィックをリダイレクトすることでレスポンスを改ざんした可能性が推測されます。この手法ならば、アプリケーションのメモリやコードを直接触ることなく、実行中のみ有効な改ざんが可能です。

5. **APIロジックを改ざん**

   悪意のあるプログラムによって、API のロジックが実行中に書き換えられてしまいました。その結果、前回の記事で解説したように、ユーザーが「Unstake（ステーキング解除）」をリクエストした際に、正常なトランザクションに加えて、**攻撃者のものにWithdraw権限を移譲する悪意のあるトランザクションが追加で返される**ようになってしまいました。

6. **署名**

   最後に、ユーザー（今回はステーキングプロバイダーの顧客企業）は、API から返された改ざん済みのトランザクションの中身を十分に確認しないまま署名してしまいました。これにより、意図せず攻撃者へ資産の引き出し権限を与えてしまい、資産流出へと繋がった、という流れです。ちなみに、ステーキングプロバイダーはインシデント発生前から、安全のため署名前にトランザクションをデコードし、検証することを推奨していたと続報には記載されています。
   <!-- textlint-disable ja-technical-writing/no-mix-dearu-desumasu -->
   <!-- textlint-disable no-mix-dearu-desumasu -->

## 今回の攻撃手法への対策と前提知識

ここまでで大まかな攻撃の流れが見えてきました。では我々エンジニアは、このような攻撃からどのようにシステムを守ればいいのでしょうか。

### 前提知識

対策をより深く理解するための前提知識として、以下の 2 点について先に整理しておきましょう。

1. GitHub Token の種別
2. Token が漏洩する一般的なパターン

#### GitHub Tokenの種別

今回漏洩した Token の種別は特定されていません。そこで、今回の攻撃に使われた可能性のある GitHub Token を、可能性の高そうな順に見ていきましょう。それぞれの Token がどんなもので、どんな権限を持っているのかを知ることが、対策を考えるうえで非常に重要になります。

| Token種別                              | Prefix        | 特徴                                                                                                                                   | 有効期限                               | 考えられる漏洩時の影響範囲                                                                    |
| -------------------------------------- | ------------- | -------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------- | --------------------------------------------------------------------------------------------- |
| **Personal Access Token (classic)**    | `ghp_`        | **【有力】** 個人ユーザーに紐づく。 Repository の読み書き、CI/CDのトリガーなど、幅広い操作が可能。無期限も設定可能だが、現在は非推奨。 | **無期限**も設定可能（現在は非推奨）   | 非常に広い。トークンに与えられたスコープ（権限）によっては、ほぼ全ての操作ができてしまう。    |
| **Fine-grained Personal Access Token** | `github_pat_` | **【有力】** 個人ユーザーに紐づく。 Repository 単位で権限を細かく制御可能。**有効期限の設定**が必須で、セキュリティが強化されている。  | **必須**（最大1年）                    | classic PATより限定的だが、必要な Repository と権限が付与されていれば同様の攻撃が可能。       |
| OAuth App の Access Token              | `gho_`        | 外部のSaaSやツールが、ユーザーの代わりにGitHubを操作するために使用する。                                                               | 通常は有効期限あり（リフレッシュ可能） | アプリケーションに許可された範囲の操作に限られるが、CI/CD関連の権限があれば同様の攻撃が可能。 |
| GitHub App User OAuth Token            | `ghu_`        | ユーザーがGitHub Appを認可した際に発行される、ユーザーに紐づくトークン。                                                               | 短命（8時間）、リフレッシュ可能        | Appに許可された Repository と権限に限定される。                                               |
| GitHub App Install Access Token        | `ghs_`        | GitHub Appが特定の Repository にインストールされた際に発行される、App自身に紐づくトークン。                                            | 短命（最大1時間）                      | Appに許可された Repository と権限に限定される。                                               |

#### 補足 1 ：Personal Access Tokenの2種類について

GitHub の Personal Access Token には、classic と Fine-grained の 2 種類があります。

- Personal Access Token (classic): 従来型のトークンで、無期限設定が可能だが、現在は非推奨とされている。権限の範囲が広く、Organization 全体やユーザーが持つ全ての Repository に影響を及ぼす可能性があるため、漏洩時のリスクが非常に高い。
- Fine-grained Personal Access Token: 2022 年に導入された新しいトークンで、有効期限の設定が必須（最大 1 年）。 Repository 単位で権限を細かく制御でき、最小権限の原則に基づいた運用が可能となる。
  今回の事件では、どちらの PAT も使われた可能性があります。 ただし、classic PAT で無期限設定だった場合は、攻撃者にとって特に都合の良い状況だったと考えられます。一方、 Fine-grained PAT でも、必要な Repository へのアクセス権限と CI/CD トリガーの権限が付与されていれば、同様の攻撃が可能です。

#### 補足 2 ：GITHUB_TOKEN

GitHub Actions のワークフロー実行中に自動で発行される `GITHUB_TOKEN` というものもあります。これは有効期限がジョブ終了までと非常に短く、権限もその Repository 内に限定されているため、今回の攻撃のように長期間にわたって外部から侵入するようなシナリオには使いにくく、可能性は低いと考えられます。

今回のケースでは、エンジニア個人の Token が漏洩したとされていること、そして攻撃者が Repository を横断して活動している可能性を考えると、**もっとも可能性が高いのは Personal Access Token** (PAT)と推測されます。特に、もしその PAT が無期限で、広い権限（`repo` や `workflow` スコープ）を持っていたとしたら、攻撃者にとっては非常に好都合な状況だったと考えられます。

### 重要な前提：今回の事件はPrivate Repositoryで発生しました

Private Repository の場合、「どうせ非公開だから」とセキュリティ意識が薄れてしまうエンジニアも少なくありません。しかし、今回の事件が示すように、ひとたび Token が漏洩すれば、攻撃者は Repository 内のすべてのコード、設定、履歴へアクセス可能になります。この前提を念頭に置いて、以降の漏洩パターンと対策を確認していきましょう。

#### Token が漏洩する一般的なパターン

以降の対策は主に Private Repository を前提としていますが、多くは Public Repository にも適用可能です。一部 Public Repository 特有の対策については、その旨を明記しています。

次に、Token がどうやって漏洩してしまうのか、その一般的なパターンを見ていきましょう。原因が分からないことには、対策の立てようがありません。ここでは、特にありがちな 8 つのパターンを挙げています。

1. **誤コミット/READMEや設定ファイルへの混入**
   うっかり Token をソースコードや設定ファイル、README に書いてしまい、そのまま `git push` してしまうケース。

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

### 攻撃手法への対策

ではその前提のもと、攻撃手法への対策を、以下の **3 つの防御層（多層防御）**の観点から見ていきます。

1. 【予防層】秘匿情報を漏洩させないための対策
2. 【被害軽減層】万が一漏洩してしまった場合に被害を最小限に食い止めるための対策
3. 【検知・対応層】漏洩の早期検知と迅速な対応をするための対策

### 【予防層】秘匿情報を漏洩させないための対策

> **目的**：Token や API キーなどの秘匿情報が、そもそも漏洩しないようにする対策です。これらは「漏洩を100%防ぐ」ことを目指す、最初の防線です。

#### 1. **誤コミット/README・設定混入対策**

- **Secret scanningとPush protectionを有効化する**: GitHub が提供している機能で、 Repository にプッシュされたコードをスキャンして、Token や API キーなどの秘匿情報らしき文字列があったら警告・ブロックしてくれる。これは必ず有効にすべきである。
- **`.gitignore` と `pre-commit` フックを活用する**: そもそも Token やパスワードなどの秘匿情報を Git の管理対象に含めないよう `.gitignore` を正しく設定し、さらに `gitleaks` のようなツールを `pre-commit` フックに仕込んで、コミットする前にローカルで検知・ブロックする仕組みも有効である。
- **環境変数ファイルの暗号化と `.env` ファイルの管理**: 開発環境で使用する `.env` ファイルなどには秘匿情報を含むケースが多いため、これらのファイルは Git で管理しないようにし、必要に応じて暗号化して保管する。また、本番環境では環境変数を直接設定するか、`dotenvx` のようなツールで安全に管理する。

#### 2. **CI/CD ログ・アーティファクト対策**

- **ログに秘匿情報を出力しない**: GitHub Actions の `::add-mask::` というコマンドを使えば、特定の文字列をログ上で `***` のようにマスクできる。Token、パスワード、API キーなどの秘匿情報は絶対にログへ出力してはいけない。また、デバッグでよく使う `set -x` のようなコマンドは、意図せず環境変数を展開してしまう可能性があるため、本番のワークフローでは使用を禁止するルールを設ける必要がある。
- **アーティファクトとキャッシュは最小限に**: ビルド成果物（アーティファクト）やキャッシュには、本当に必要なものだけを含めるようにし、秘匿情報が含まれていないことを確認してから保存する。保持期間も短く設定（YAML でも `retention-days` で設定可能）し、使い終わったら速やかに削除する運用も重要である。

#### 3. **Actions 権限/トリガー不備対策（Public Repository の場合）**

- **外部からのPull Requestは手動承認を必須に**: Organization の設定で、フォークされた Repository からの Pull Request でワークフローが自動実行されないようにし、必ずレビュワーの承認を必須にする。これにより、悪意のある PR によって Secrets が盗まれるリスクを大幅に減らせる。

#### 4. **セルフホストRunner 隔離対策**

- **Runnerは使い捨て（Ephemeral）で運用する**: セルフホスト Runner を使う場合は、ジョブごとに新しい環境を起動し、ジョブが終わったら破棄する「使い捨て」モデルが理想である。また、本番環境にデプロイする Runner と、開発中の PR をテストする Runner は、ネットワークや権限を完全に分離する。

#### 5. **端末侵害対策（アクセス制御と基本的なセキュリティ意識）**

- **SSOと2FA（多要素認証）を必須にする**: GitHub へのログインは、パスワードだけでなく、SSO（シングルサインオン）や 2FA を全開発者に強制する。これにより、開発者の認証情報が盗まれた場合でも、Token へのアクセスを防ぐことができる。
- **認証情報を安易に保存しない**: ブラウザのパスワードマネージャーやクリップボード、SSH エージェントなどに Token を安易に保存しない、という基本的なセキュリティ意識も重要である。

#### 6. **外部ActionのコミットSHA固定**

- **コミットハッシュを固定する**: ワークフローで利用するサードパーティ製の Action は、`actions/checkout@v4` のようなタグの指定ではなく、`actions/checkout@a12a3943b4bde6ff22b8f99495b33c59aed7c3d2` のように**コミットハッシュで固定**します。これにより、Action の作成者が悪意のあるバージョンをタグに上書きするといった、サプライチェーン攻撃のリスクを予防できます。
  _設定方法: Organizationの `Settings` > `Actions` > `General` で `Allow Omakase, and select non-Omakase, actions and reusable workflows` を選択し、`Require actions to be pinned to a full-length commit SHA` にチェックを入れる。_

#### 7. **チケット/チャット/スクリーンショット対策（秘密情報の共有禁止）**

- **とにかく貼らない、写さない**: これはもう運用と文化の話になるが、「秘密情報は絶対にパブリックな場所へは共有しない」というルールを彸底する。Issue や Pull Request のコメント、Slack などのチャットツールに Token、パスワード、API キーなどを貼り付けてはいけない。

---

### 【被害軽減層】万が一漏洩してしまった場合に被害を最小限に食い止めるための対策

> **目的**：予防策をすり抜けて Token が漏洩してしまった場合に、攻撃者が悪用できる範囲を制限し、被害を最小限に留める対策です。これは「多層防御」の考え方に基づいています。

#### 1. **本番に影響を与えるCI/CDの経路を限定する**

攻撃者は盗んだ Token を使って CI/CD を悪用しました。ならば、その CI/CD から本番環境へ至る道を、できるだけ狭く、そして監視の行き届いたものにしておくことが重要です。具体的には、以下の 5 つの対策が非常に有効です。

![CICD経路限定フロー図](/images/validator-security-incident-2025/CICD経路限定フロー図.png)

- **SecretsはEnvironmentを利用して分離し、レビュワーの承認を強制する**
  GitHub Actions の**Environments**という機能を使うと、「本番環境用」「ステージング環境用」といったように、環境ごとに Secrets を分離できます。さらに、「`production` 環境へのデプロイは、信頼できるメンバーの誰かが承認しないと実行できない」といった**承認ルール**を設定できます。これにより、たとえ攻撃者が CI/CD をトリガーできても、承認者のチェックを突破しない限り本番環境には手出しできなくなります。
  _設定方法: Repository の `Settings` > `Environments` で `production` 環境を作成し、`Required reviewers` に承認者を設定する。_

- **権限最小化（`permissions: read` をDefault設定し、必要時のみ権限昇格）**
  ワークフローに与える `GITHUB_TOKEN` の権限は、デフォルトでは読み取り専用（`permissions: read-all`）にしておき特定のジョブで書き込み権限が必要な場合にのみ、そのジョブ単位で権限を昇格させる（例: `permissions: contents: write`）ようにします。これにより、ワークフロー全体が乗っ取られたとしても、被害の範囲を限定できます。
  _設定方法: Organizationまたは Repository の `Settings` > `Actions` > `General` の `Workflow permissions` で `Read repository contents and packages permissions` を選択する。_

- **Repository シークレットに本番情報を置かない**
  Repository 単位の Secrets は、**Tokenが漏洩した場合**、攻撃者がワークフローを実行して使えてしまいます。重要なのは、「Repository Secrets が危険なのではなく、**保護ルールなしで重要なシークレットが利用できてしまう状態**が危険」ということです。今回の事件のように Private Repository だったとしても Token が漏洩すれば同じリスクがあるため、本番環境の認証情報のような重要情報は、 Repository の Secrets には置かず、前述の**Environments**の Secrets に格納して、厳格にアクセスを管理します。

  **Environments の真価**は、シークレットの格納場所を変えることではなく、**シークレットへのアクセスに「保護ルール（例：承認者の必須化）」を強制できる点**にあります。これにより、たとえ攻撃者が CI/CD をトリガーできても、承認者のチェックを突破しない限り本番環境の Secrets にはアクセスできなくなります。さらに、環境ごとに異なる値を設定可能（例：`production` と `staging` で異なるクレデンシャルを使用）なため、被害の範囲をさらに限定できます。

- **`push: branches: [main]` などに限定してSecretsを渡す**
  本番環境の Secrets を利用できるワークフローは、`main` ブランチへの push など、本当に信頼できるイベントによってのみトリガーされるように厳しく制限し、開発中のブランチから安易に本番環境の Secrets へアクセスできないようにすることが重要です。

- **`.github/workflows/` の変更にセキュリティ承認を必須化**

  今回の攻撃では、攻撃者はワークフローファイル自体を改ざんすることはしませんでした。しかし、**別の攻撃シナリオでは、ワークフローファイルを書き換えてSecretsを盗み出す**という手法も十分に考えられます。そのため、`.github/workflows/` ディレクトリ配下のファイル変更には、必ず第三者の承認を必須とする設定を入れておくことを推奨します。これにより、ワークフローを改ざんして Secrets を盗み出そうとするような攻撃を、Pull Request のレビュー段階で防ぐことができます。**今回のケースでは、攻撃者はワークフローファイル自体を改ざんせず、既存のワークフローをトリガーする形で攻撃したため、この対策は直接的な効果がありませんでした。** しかし、別の攻撃シナリオ（例えば、ワークフローファイルを書き換えて Secrets を環境変数として出力させる、など）に対しては**非常に有効**となります。**多層防御の一環として、ぜひ導入しておきたい対策**です。

これらの設定を盛り込んだ Sample YAML と workflows の変更にセキュリティ承認を設定する方法を以下に示します。

_Sample YAML:_

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

_セキュリティ承認の設定方法:_

1. Repository のルートに `CODEOWNERS` ファイルを作成し、以下のように記述する

   ```text
   # .github/workflows/配下のファイルは @organization/security-team の承認が必須
   /.github/workflows/ @organization/security-team
   ```

2. Repository の `Settings` > `Branches` で `main` ブランチの保護ルールを開き（または `Settings` > `Rules` > `Rulesets` で新規作成し）、`Require a pull request before merging` と `Require review from Code Owners` を有効にする

#### 2. **長期シークレットの撤廃（OIDCの活用）**

そもそも、有効期限の長い「長期シークレット」は大きなリスクです。GitHub の PAT（Personal Access Token）にしても、クラウドプロバイダー（例：AWS のアクセスキーとシークレットキー）にしても、一度漏洩すれば攻撃者に長期間悪用される可能性があります。可能な限り、有効期限の短い「短期トークン」に置き換えていきましょう。そのための**切り札となる対策がOIDC (OpenID Connect) 連携**です。

![長期シークレット撤廃フロー図](/images/validator-security-incident-2025/長期シークレット撤廃フロー図.png)

OIDC 連携を使えば、クラウドの長期的なアクセスキーを GitHub の Secrets に保存する必要がなくなります。ワークフローが実行されるたびに、クラウド側から有効期限の短い一時的な認証情報が発行され、それを使って安全にクラウドを操作できます。

**重要なのは、OIDC 連携でクラウド側（例：AWS）へ作成する IAM Role に最小限の権限のみを付与することです。**これにより、万が一攻撃者がトークンを入手したとしても、その攻撃者が実行できる操作を厳しく制限できます。これは現代の CI/CD セキュリティにおいて**最重要対策の一つ**です。

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

#### 3. **サードパーティ連携のセキュリティ強化（影響範囲の限定）**

外部の SaaS やツールと連携する際に、Token が漏洩した場合の影響範囲を限定することが重要です。

- **OAuth/GitHub Appのベストプラクティスを遵守する**: 連携アプリケーションを開発・利用する際は、PKCE フローの利用、リダイレクト URI の厳格な固定、Webhook の署名検証などセキュリティのベストプラクティスを徹底する。これにより、外部ツール経由での Token 漏洩リスクを低減できます。

#### 4. **コンテナイメージへの秘匿情報の残置対策（漏洩経路の遮断）**

Token がコンテナイメージのレイヤーに残ってしまうと、イメージを入手した攻撃者に Token が漏洩してしまいます。

- **マルチステージビルドと `--mount=type=secret` を活用する**: Dockerfile でマルチステージビルドを使い、最終的なイメージに不要なビルド時の情報が残らないようにする。また、ビルド時に Token が必要な場合は、`--mount=type=secret` オプションを使うことで、イメージレイヤーに Token が残るのを防げる。

#### 5. **秘匿情報の漏洩時のローテーション手順の確立**

万が一、秘匿情報が漏洩してしまった場合に備えて、迅速に対応するための手順を事前に確立しておくことが重要です。

- **ローテーション手順の確立**: もし誤って秘匿情報を共有してしまった場合は、**即座にそれらの秘匿情報を無効化（ローテーション）する**フローを確立しておくことが重要である。具体的には、以下のような手順を用意しておくべきである：
  1. 漏洩を発見した時点で、チーム全体に通知する
  2. 漏洩した Token/キーを即座に無効化する
  3. 新しい Token/キーを発行する
  4. すべての環境で新しい認証情報に置き換える
  5. ログを確認して、漏洩した認証情報がどの程度悪用されたかを調査する

---

### 【検知・対応層】漏洩の早期検知と迅速な対応をするための対策

> **目的**：Token 漏洩が発生した場合に、それを早期に検知し、迅速に対応するための対策です。予防と被害軽減がすり抜けられた場合の最後の砦です。

#### 1. **GitHub監査ログの常時転送＆検知**

本稿のインシデントでは、攻撃者はブランチを大量に作成・削除して CI をトリガーしました。このような異常なアクティビティを早期に検知するためには、GitHub の監査ログを常時 SIEM（Security Information and Event Management）ツールや Slack などに転送し、監視する仕組みを以下のフロー図のように構築しましょう。

![Github監視フロー図](/images/validator-security-incident-2025/Github監視フロー図.png)

特に、以下のようなイベントはアラートの対象とすべきです。

- ブランチの作成・削除が短時間に大量発生
- `workflow_dispatch`（手動実行）の急増
- 新しい Runner の登録や削除
- Secrets の更新
- Repository の設定変更

これをやっていれば、今回のケースのような攻撃の兆候を、侵入の初期段階で掴めた可能性があります。

#### 2. **Kubernetes Podへの操作の異常検知**

今回の攻撃経路では、攻撃者は最終的に CI/CD パイプラインを通じて本番環境にデプロイされた Pod へアクセスし、内部で悪意のあるコマンドを実行する可能性があります。GitHub Actions の監視だけでは、この「デプロイ後の活動」を検知できません。そこで、Kubernetes のランタイムセキュリティを強化し、Pod 内部での不審な振る舞いを検知する仕組みが重要になります。

##### 具体的な手法：Falcoによるランタイムセキュリティ監視

ここでは一例として、Falco を用いた監視例を紹介します。[Falco](https://falco.org/) は、CNCF（Cloud Native Computing Foundation）のインキュベーションプロジェクトであり、コンテナランタイムセキュリティのデファクトスタンダードツールです。eBPF という技術を用いて、カーネルレベルでシステムコールを監視し、あらかじめ定義されたルールに違反する不審なアクティビティをリアルタイムで検知します。

##### 攻撃シナリオと検知ルール

攻撃者が Pod へ侵入した後に行う可能性のある行動と、それを検知するための Falco ルールの例を以下に示します。

| 攻撃者の行動                     | 検知する Falco ルール（例）                      | 正当な運用との区別                                             |
| :------------------------------- | :----------------------------------------------- | :------------------------------------------------------------- |
| **シェルを起動する**             | `Terminal shell in container`                    | ホワイトリスト化：許可されたユーザー・時間帯のみ除外           |
| **機密ファイルを読み取る**       | `Read sensitive file untrusted`                  | 環境変数ファイル、秘密鍵など特定ファイルへのアクセスは常に監視 |
| **外部に不正な通信を行う**       | `Unexpected outbound connection destination`     | 許可されたエンドポイント以外への通信を検知                     |
| **パッケージをインストールする** | `Launch package management process in container` | コンテナ内での `apt`、`yum` 実行は常に検知（本来は不要）       |

続けて、以下ではシェル起動の検知ルールの設定例を紹介します。

##### 正当な運用と攻撃を区別する実装方法

###### 1. **コンテキスト情報の活用による判定**

`kubectl exec` による正当なアクセスと、攻撃者による不正なシェル起動を区別するために、以下のコンテキスト情報を活用します。

| 判定項目             | 正当な `kubectl exec`                         | 攻撃者による侵入                                   |
| :------------------- | :-------------------------------------------- | :------------------------------------------------- |
| **親プロセス**       | `kubelet` 経由（PID トレースで確認可能）      | 不正な入口（リバースシェル、ネットワーク経由など） |
| **実行ユーザー**     | Pod 内の標準ユーザー（例：`app`, `www-data`） | 権限昇格を試みる可能性（`root` への変更）          |
| **実行時刻**         | 業務時間内、または事前に通知された時間        | 深夜や休日など不規則な時刻                         |
| **ネットワーク接続** | 社内ネットワークから、または VPN 経由         | 外部からの直接接続、プロキシ経由など               |

Falco では、これらのコンテキスト情報をシステムコール監視から抽出し、ルールの判定に組み込むことができます。

###### 2. **ホワイトリスト化による除外**

正当な `kubectl exec` を識別するために、Falco ルールにホワイトリストを設定します。

```yaml
# Falco ルールの例（YAML形式）
- rule: Terminal shell in container
  desc: Detect shell spawning in container
  condition: >
    spawned_process and container and shell_procs
    and not (user.name in (allowed_operators))
    and not (container.privileged)
  output: >
    Shell spawned in container (user=%user.name container=%container.name pod=%k8s.pod.name)
  priority: WARNING
```

**設定のポイント**：

- `allowed_operators`：本番環境へのアクセスが許可されたユーザー名（例：`devops-team`, `sre-oncall`）をリスト化
- `container.privileged`：特権コンテナでの実行は除外（ただし本来は避けるべき）
- 時間帯ベースの除外：営業時間外のシェル起動は常に検知

##### 監視フロー

![K8s監視フロー図](/images/validator-security-incident-2025/K8s監視フロー図.png)

1. **イベント発生**: 攻撃者が不正な方法で Pod に侵入し、シェルを起動する。
2. **システムコール監視**: 各ノードに DaemonSet としてデプロイされた Falco が、カーネルのシステムコールを監視する。
3. **コンテキスト判定**: Falco が親プロセス、実行ユーザー、実行時刻などを確認し、ホワイトリストに該当するか判定。
4. **異常検知**: ホワイトリストに該当しない場合、Falco が「不正なシェル起動」として検知。
5. **アラート通知**: Falco が設定された通知チャネル（例：Webhook, Slack, SIEM）にアラートを送信。
6. **インシデント対応**: SOC 担当者や開発者がアラートを受け取り、Pod の隔離、ログ調査、復旧を実施。

このように、GitHub の監査ログ監視と Kubernetes のランタイムセキュリティ監視を組み合わせることで、攻撃の検知範囲を「CI/CD パイプライン」から「本番環境のコンテナ内部」まで広げ、より堅牢なセキュリティ体制を構築できます。

#### 3. **EDR（Endpoint Detection and Response）の導入**

開発者のマシンが Token の漏洩源となる可能性があります。EDR ツールを導入することで、マルウェア感染やクリップボード盗聴などの異常を早期に検知できます。

- **EDRを導入する**: EDR (Endpoint Detection and Response) ツールを導入し、万が一マルウェアに感染しても、その活動を早期に検知・対応できる体制を整える。これにより、開発者マシンからの Token 漏洩を早期に発見できる可能性が高まります。

#### 3. **インシデント対応計画の策定**

Token 漏洩が検知された場合に、迅速に対応するための計画を事前に策定しておくことが重要です。

- **インシデント対応計画の策定**: Token 漏洩が疑われる場合の対応フローを事前に定義しておく。具体的には以下を含める：
  1. 漏洩検知時の通知先と通知方法
  2. 初期対応（Token の無効化、ログ確認）
  3. 影響範囲の調査
  4. 復旧手順
  5. 事後分析とプロセス改善

> **注記**：API 改ざん対策の最終防衛ラインである「署名前デコード」については、前回の記事「[【60億円流出】Solanaステーキングで何が起きたのかーAPI悪用の可能性と対策](https://zenn.dev/omakase/articles/b2e85e51ca45ef)」で詳しく解説しているため、本記事では参照に留めます。

## 補足：その他の防御策

ここまで、今回のインシデントに直接関連する対策を見てきましたが、別の攻撃パターンに備えるための防御策も簡単に触れておきます。

### `.github/workflows/` の変更にセキュリティ承認を必須化

今回の攻撃では、攻撃者はワークフローファイル自体を改ざんすることはしませんでした。しかし、**別の攻撃シナリオでは、ワークフローファイルを書き換えてSecretsを盗み出す**という手法も十分に考えられます。そのため、`.github/workflows/` ディレクトリ配下のファイル変更には、必ず第三者の承認を必須とする設定を入れておくことを推奨します。

_設定方法:_

1. Repository のルートに `CODEOWNERS` ファイルを作成し、以下のように記述する

   ```text
   # .github/workflows/配下のファイルは @organization/security-team の承認が必須
   /.github/workflows/ @organization/security-team
   ```

2. Repository の `Settings` > `Branches` で `main` ブランチの保護ルールを開き（または `Settings` > `Rules` > `Rulesets` で新規作成し）、`Require a pull request before merging` と `Require review from Code Owners` を有効にする

これにより、ワークフローを改ざんして Secrets を盗み出そうとするような攻撃を、Pull Request のレビュー段階で防ぐことができます。
**今回のケースでは、攻撃者はワークフローファイル自体を改ざんせず、既存のワークフローをトリガーする形で攻撃したため、この対策は直接的な効果がありませんでした。** しかし、別の攻撃シナリオ（例えば、ワークフローファイルを書き換えて Secrets を環境変数として出力させる、など）に対しては**非常に有効**となります。**多層防御の一環として、ぜひ導入しておきたい対策**です。

## まとめ

今回は、GitHub Access Token の漏洩を起点とした API 改ざんインシデントを題材に、その攻撃手法と具体的な対策を深掘りしてみました。

今回の攻撃が非常に巧妙だったのは、単にサーバーに侵入するだけでなく CI/CD パイプラインという「システムの作り方」そのものを悪用し、さらにファイルやコンテナイメージを残さず、**実行中のコンテナ内で iptables や eBPF などを使ってネットワークトラフィックをリダイレクトし　API レスポンスを改ざんする**という検知が非常に困難な手法を取っていた点です。

この記事で紹介した対策をまとめると、以下のようになります。

- **予防策（Tokenを漏洩させない）**: Secret scanning の有効化、`pre-commit` フックの導入、ログへのマスキング、外部 PR の手動承認、Runner の分離、SSO/2FA の必須化など、基本的なセキュリティ対策を徹底する。
- **被害最小化策（漏洩しても大丈夫なようにする）**: CI/CD の経路を Environment や承認ルールで厳格に管理し、OIDC を活用して長期シークレットを撤廃する。そして、監査ログで異常を監視し、最後の砦として署名前のデコードを徹底する。

セキュリティに「絶対」はありません。だからこそ、1 つの対策に頼るのではなく、複数の防御策を組み合わせる「**多層防御**」の考え方が何よりも重要です。今回のインシデントを他人事と捉えず、ぜひ皆さんの開発現場でも、セキュリティ対策を見直すきっかけにしていただければ幸いです。

## 参考リンク

<!-- textlint-disable ja-technical-writing/sentence-length -->

[1][Kiln サービスの再稼働およびセキュリティインシデントに関する情報](https://www.kiln.fi/post/kiln-sabisunozai-jia-dong-oyobisekiyuriteiinsidentoniguan-suruqing-bao)
[2][【60億円流出】Solanaステーキングで何が起きたのかーAPI悪用の可能性と対策](https://zenn.dev/omakase/articles/b2e85e51ca45ef)
[3][GitHub Docs: Secret scanning](https://docs.github.com/ja/code-security/secret-scanning/about-secret-scanning)
[4][GitHub Docs: Using environments for deployment](https://docs.github.com/ja/actions/deployment/targeting-different-environments/using-environments-for-deployment)
[5][GitHub Docs: Security hardening for GitHub Actions](https://docs.github.com/ja/actions/security-guides/security-hardening-for-github-actions)
[6][GitHub Docs: About code owners](https://docs.github.com/ja/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)

<!-- textlint-enable ja-technical-writing/sentence-length -->
