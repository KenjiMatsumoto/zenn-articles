---
title: "【徹底解明】GitHub Token流出から始まったAPI改ざんとその対策"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CICD", "github", "AWS", "Token"]
published: false
publication_name: omakase
---

## 前回の振り返り

前回の記事では、60 億円の流出が発生したのはトランザクションが書き換えられ、悪意のあるコードが実行されてしまったことが原因でした。ではどのような攻撃手法でトランザクションを書き換えることができたのでしょうか。続報([リンク](https://www.kiln.fi/post/kiln-sabisunozai-jia-dong-oyobisekiyuriteiinsidentoniguan-suruqing-bao))をもとに分析していきます。

## 根本原因はGithubAccessTokenの漏洩

インシデントの詳細([リンク](https://www.kiln.fi/post/kiln-sabisunozai-jia-dong-oyobisekiyuriteiinsidentoniguan-suruqing-bao))に記載されている事件の根本原因はインフラエンジニアの GithubAccessToken の漏洩とされています。ただ内容には GithubAccessToken としか記載されておらず Token の種別は言及されていません。また Token の流出原因についても決定的な証拠が見つかっていません。そのため、侵害されてしまう Token にはどんな種別があるのか。Token はどのような経路で流出してしまうのか。を詳細に追ってみましょう。

### Tokenの種別

Token の種別が明確に言及されていないため、今回の事件に該当すると考えられる Token の種別を可能性が高い順に見ていきましょう。

1. Personal Access Token  
   詳細を記載する。(補足：PAT については Github API から取得することは不可能)
1. OAuth App の Access Token  
   詳細を記載する。
1. GitHub App User OAuth Token  
   詳細を記載する。
1. GitHub App Install Access Token  
   詳細を記載する。

**補足：GITHUB_TOKEN**
Actions ジョブ開始時に自動発行される短命・リポジトリ限定のトークンのため、今回の事件には適さないことを記載する。

### 一般的な流出パターン

Token の流出原因については決定的な証拠が見つかっていません。  
そのため、一般的な流出パターンを見ていきましょう。

1. 誤コミット/README や設定ファイルへの混入からの漏洩  
   詳細を記載する。
1. CI/CD のログ・アーティファクトからの漏洩  
   詳細を記載する。
1. Actions の権限/トリガー設計の不備からの漏洩  
   詳細を記載する。
1. セルフホスト Runner の隔離不備からの漏洩  
   詳細を記載する。
1. 端末側が侵害（開発者マシンのマルウェア/ブラウザセッション乗っ取り）され漏洩  
   詳細を記載する。
1. サードパーティ連携（OAuth/GitHub App/Webhooks）の侵害からの漏洩  
   詳細を記載する。
1. コンテナ/イメージ/キャッシュ層に残置しての漏洩  
   詳細を記載する。
1. チケット/チャット/スクショ共有からの漏洩  
   詳細を記載する。

他にも人やツール、周辺 Saas にまで広げると際限なくパターンは出てきます。しかしながら、今回はあくまでコアなパターンとして８つを取り上げました。

## 流出したTokenでの攻撃経路

それでは実際に漏洩した GithubAccessToken を利用してどのように攻撃し、流出までに至ったのかをフロー図を交えて見ていきましょう。

①Token→②CI 実行の悪用→③Secrets 収集→④k8s 侵入→⑤API 改ざん→⑥署名→⑦資産流出という流れのフロー図を記載し、下記リストに記載の内容とリンクさせます。

1. GitHubAccessToken 漏洩  
   詳細を記載する。
1. CI 実行の悪用（branch create/delete でトリガー）  
   詳細を記載する。
1. Secrets 収集・クラウド横展開  
   詳細を記載する。
1. Kubernetes 侵入（Connect API Pod）  
   詳細を記載する。
1. API ロジックを改ざん  
   詳細を記載する。
1. 署名（raw signing）  
   詳細を記載する。
1. 資産流出  
   詳細を記載する。

## 今回の攻撃手法への対策

では今回の攻撃手法に対してどのような防御策が考えられるでしょうか。流出を防ぐ、流出後の被害を最小限に留めるという２つの観点で見ていきましょう。

### Tokenを流出させないための対策

Token の流出原因については、一般的な流出パターンで確認しました。ではそれらをどのように防ぐのかについてみていきましょう。

1. 誤コミット/README や設定ファイルへの混入からの漏洩の対策を記載
1. CI/CD のログ・アーティファクトからの漏洩の対策を記載
1. Actions の権限/トリガー設計の不備からの漏洩の対策を記載
1. セルフホスト Runner の隔離不備からの漏洩の対策を記載
1. 端末側が侵害（開発者マシンのマルウェア/ブラウザセッション乗っ取り）され漏洩の対策を記載
1. サードパーティ連携（OAuth/GitHub App/Webhooks）の侵害からの漏洩の対策を記載
1. コンテナ/イメージ/キャッシュ層に残置しての漏洩の対策を記載
1. チケット/チャット/スクショ共有からの漏洩の対策を記載

### 被害を最小限に留めるための対策

1. 本番に影響を与える CI/CD の経路を限定する
   - Secrets は Environment を利用して分離し、レビュワーの承認を強制
   - 権限最小化
   - リポジトリシークレットの本番情報の利用禁止
   - 外部 Action のコミット SHA 固定
   - permissions: read を Default 設定し、必要時に権限昇格
   - push: branches:[main]などに限定して Secrets を渡す
   - 上記の内容を Sample Yaml で記載
1. 長期シークレットの撤廃
   - クラウド認証を OIDC に置換（導入手順と Sample Yaml などを記載）
   - コンテナレジストリのトークン短命化（ECR、GHCR の Sample Yaml を踏まえ記載）
   - PAT の原則禁止と導入せざるを得ない場合の対応法（fine-grained PAT＆期限必須などの短命化）
   - ORG 設定の Secret scanning と Push protection の有効化
1. 署名前デコードと不正トランザクションの検知  
   前回の記事に記載されているため、割愛([前回記事](https://zenn.dev/omakase/articles/b2e85e51ca45ef))

## まとめ

記事のまとめを記載します。

## 参考リンク

掲載した各種リンク記載します。
