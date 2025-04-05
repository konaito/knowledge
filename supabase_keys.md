# Supabaseにおける`supabase_url`・`anon_key`・`service_role_key`のセキュリティベストプラクティス

Supabaseを安全に利用するためには、プロジェクトのAPIエンドポイントである`supabase_url`と2種類のAPIキー（`anon_key`と`service_role_key`）を正しく取り扱うことが重要です。以下では各キーの役割や、フロントエンド/バックエンドでの適切な使い分け、公開してよい情報と秘密にすべき情報の基準、環境変数やCI/CDでの管理、キー漏洩時の対処とローテーション方法、さらにRow Level Security (RLS)との連携によるセキュリティ強化について解説します。

## 各キーの役割と用途の違い

- **Supabase URL (`supabase_url`)**: 各Supabaseプロジェクトに固有のAPIエンドポイントURLです。クライアント（フロントエンド）およびサーバー（バックエンド）の双方から、このURLを通じてデータベースのREST APIや認証機能にアクセスします。`supabase_url`自体は秘密情報ではなく、プロジェクトの識別子として機能します ([Managing Secrets (Environment Variables) | Supabase Docs](https://supabase.com/docs/guides/functions/secrets#:~:text=,connect%20directly%20to%20your%20database))。ただし、このURLのみではデータにはアクセスできず、後述するキーが必要です。

- **Anonキー (`anon_key`)**: 「匿名キー」あるいは「クライアントキー」とも呼ばれ、PostgreSQLの`anon`ロールに対応するAPIキーです ([Understanding API Keys | Supabase Docs](https://supabase.com/docs/guides/api/api-keys#:~:text=Supabase%20provides%20two%20default%20keys,keys%20in%20the%20API%20Settings))。権限は極めて限定的で、RLSポリシーによって厳しくアクセスが制御されます ([〖Supabase〗anon keyと service_role keyの使い分け #Security - Qiita](https://qiita.com/syukan3/items/8ec9b7288834e4c95f3b#:~:text=,%E8%AA%8D%E8%A8%BC%E7%8A%B6%E6%85%8B%E3%81%AB%E3%82%88%E3%82%8B%E3%83%AD%E3%83%BC%E3%83%AB%E3%81%AE%E5%A4%89%E5%8C%96%20%E3%83%A6%E3%83%BC%E3%82%B6%E3%83%BC%E3%81%8C%E3%83%AD%E3%82%B0%E3%82%A4%E3%83%B3%E3%81%99%E3%82%8B%E3%81%A8%E3%80%81%E8%87%AA%E5%8B%95%E7%9A%84%E3%81%AB%E3%80%8Cauthenticated%E3%80%8D%E3%83%AD%E3%83%BC%E3%83%AB%E3%81%AB%E5%88%87%E3%82%8A%E6%9B%BF%E3%82%8F%E3%82%8A%E3%80%81%E3%83%A6%E3%83%BC%E3%82%B6%E3%83%BC%E5%B0%82%E7%94%A8%E3%81%AE%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E6%A8%A9%E3%81%8C%E9%81%A9%E7%94%A8%E3%81%95%E3%82%8C%E3%81%BE%E3%81%99%E3%80%82))。Supabase Authを利用してユーザーがログインすると、クライアントのJWT内のロールが`anon`から`authenticated`（認証済みユーザー）に自動的に切り替わり、より適切なアクセス権が適用されます ([Understanding API Keys | Supabase Docs](https://supabase.com/docs/guides/api/api-keys#:~:text=If%20you%20are%20using%20Supabase,a%20user%20is%20logged%20in))。**用途:** ブラウザやモバイルアプリなどフロントエンドから直接Supabaseにリクエストを送る際に使用します。公開されているデータの読み取りや、認証済みユーザーの自身のデータ操作など、クライアント側で行う操作に利用されます ([〖Supabase〗anon keyと service_role keyの使い分け #Security - Qiita](https://qiita.com/syukan3/items/8ec9b7288834e4c95f3b#:~:text=,%E3%82%AF%E3%83%A9%E3%82%A4%E3%82%A2%E3%83%B3%E3%83%88%E5%81%B4%E3%81%A7%E5%88%A9%E7%94%A8%E3%81%97%E3%80%81%E6%9C%80%E4%BD%8E%E9%99%90%E3%81%AE%E6%A8%A9%E9%99%90%E3%81%A7%E5%AE%89%E5%85%A8%E3%81%AB%E3%83%87%E3%83%BC%E3%82%BF%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E3%82%92%E8%A1%8C%E3%81%84%E3%81%BE%E3%81%99%E3%80%82))。

- **サービスロールキー (`service_role_key`)**: サーバーサイド専用の秘密鍵で、PostgreSQLの`service_role`ロールに対応します ([Understanding API Keys | Supabase Docs](https://supabase.com/docs/guides/api/api-keys#:~:text=Supabase%20provides%20two%20default%20keys,keys%20in%20the%20API%20Settings))。このロールは高度な権限を持ち、**テーブルに設定されたRLSを無視してあらゆるデータにアクセス可能**です ([Understanding API Keys | Supabase Docs](https://supabase.com/docs/guides/api/api-keys#:~:text=The%20,used%20on%20a%20private%20server))。Supabase側では、管理タスクやバックエンド処理のために用意されたプリビレッドなキーと位置づけられています。**用途:** クラウド関数や自前のサーバーから管理者権限でデータベース操作を行う場合や、定期バッチ・分析処理などに使用します ([Understanding API Keys | Supabase Docs](https://supabase.com/docs/guides/api/api-keys#:~:text=A%20common%20use%20case%20for,table))。**重要:** このキーは極めて強力なため、決してフロントエンドに含めてはならず、厳重に管理する必要があります。

※なお、Supabaseでは今後キーの名称変更が予定されており、`anon_key`は「publishable key（公開用キー）」、`service_role_key`は「secret key（秘密キー）」と呼ばれるようになる見込みです ([〖Supabase〗anon keyと service_role keyの使い分け #Security - Qiita](https://qiita.com/syukan3/items/8ec9b7288834e4c95f3b#:~:text=%E4%BB%8A%E5%BE%8C%E3%80%81%E3%82%AD%E3%83%BC%E3%81%AE%E5%90%8D%E7%A7%B0%E3%81%AF%E4%BB%A5%E4%B8%8B%E3%81%AE%E3%82%88%E3%81%86%E3%81%AB%E5%A4%89%E6%9B%B4%E3%81%95%E3%82%8C%E3%82%8B%E4%BA%88%E5%AE%9A%E3%81%A7%E3%81%99%E3%81%8C%E3%80%81%E5%9F%BA%E6%9C%AC%E7%9A%84%E3%81%AA%E5%BD%B9%E5%89%B2%E3%81%AF%E5%A4%89%E3%82%8F%E3%82%8A%E3%81%BE%E3%81%9B%E3%82%93%E3%80%82))。名称が変わっても役割自体に大きな変更はなく、引き続き適切な運用と管理が求められます。

## フロントエンドとバックエンドでの安全な使い分け

**フロントエンド（ブラウザ/モバイルアプリ）**では、**`anon_key`**を使用してSupabaseクライアントを初期化し、直接データベースにアクセスします。Supabase公式ドキュメントによれば、RLSを有効にしていれば`anon_key`をフロントエンドに埋め込んでも安全に利用できます ([Securing your data | Supabase Docs](https://supabase.com/docs/guides/database/secure-data#:~:text=Your%20anon%20key%20is%20safe,logged%20in%20using%20Supabase%20Auth))。実際、認証前の匿名ユーザーや認証後のユーザーが、自分に許可された操作（例: 公開データの閲覧、自身のプロフィール取得など）を行う場合、フロントエンドから`anon_key`経由でSupabaseにアクセスする設計になっています。これにより専用のバックエンドサーバーを用意しなくても、クライアントから直接安全にDB操作が可能です ([Securing your data | Supabase Docs](https://supabase.com/docs/guides/database/secure-data#:~:text=,you%20create%20a%20Supabase%20client))。

一方、**バックエンド（サーバーサイド）**では、より高権限の**`service_role_key`**を必要に応じて使用します。`service_role_key`は前述の通り強力な管理者権限を持つため、サーバー上の安全な環境でのみ利用してください ([Understanding API Keys | Supabase Docs](https://supabase.com/docs/guides/api/api-keys#:~:text=The%20,used%20on%20a%20private%20server))。例えば、サーバーサイドで動くバッチ処理、管理者向け機能の実装、あるいは認証なしでデータを書き込む必要がある場合（例: Webhook受信でDBにレコード追加など）は、`service_role_key`を用いてSupabaseクライアントを作成し操作します。Supabaseのエッジ関数（Edge Functions）やクラウドラン、自社サーバー上のAPIなど、**ユーザーから直接見えない場所**にこのキーを置きましょう。

フロントエンドでは**絶対に`service_role_key`を使わない**ことが鉄則です ([Securing your data | Supabase Docs](https://supabase.com/docs/guides/database/secure-data#:~:text=,role%20key%20on%20the%20frontend))。万一これをクライアントに含めてしまうと、RLSによる保護をバイパスして誰でもデータベース全体にアクセスできてしまいます。また、フロントエンドでは`anon_key`のみでできない特権操作が必要な場合、バックエンド経由のAPIを用意してそこで`service_role_key`を使うなど、明確に責務を分離することが望ましいです。

## 公開してもよいキーと公開してはいけないキー

Supabaseの構成要素のうち、**「公開してもよい」もの**と**「決して公開してはいけない」もの**を正しく区別する必要があります。

- **公開してもよい:** `supabase_url`および`anon_key`。  
  - *Supabase URL*は単なるサービスエンドポイントであり、これ自体に機密性はありません。誰かにURLを知られても、それだけでデータを盗まれることはありません（適切なキーと権限が無ければアクセスはできません）。  
  - *Anonキー*も、RLSを有効化している限り公開して問題ないとされています ([Securing your data | Supabase Docs](https://supabase.com/docs/guides/database/secure-data#:~:text=Your%20anon%20key%20is%20safe,logged%20in%20using%20Supabase%20Auth))。実際、Supabaseのクライアント初期化コードは公開リポジトリやフロントエンドコード内に`anon_key`を含む形で配置されるケースが多いですが、RLSによって許可された操作範囲のデータしか触れないため、鍵そのものが露出しても大きなリスクとならない設計になっています ([Managing Secrets (Environment Variables) | Supabase Docs](https://supabase.com/docs/guides/functions/secrets#:~:text=,connect%20directly%20to%20your%20database))。例えば、あるテーブルが「匿名ロールには読み取りのみ許可」されていれば、`anon_key`を使う限りそのテーブルの読み取り以外（書き込みや削除等）はできませんし、読み取れるデータもRLSポリシーで制限された範囲に留まります。

- **公開厳禁:** `service_role_key`（サービスロールキー）。  
  このキーは**絶対に公開してはいけません** ([〖Supabase〗anon keyと service_role keyの使い分け #Security - Qiita](https://qiita.com/syukan3/items/8ec9b7288834e4c95f3b#:~:text=,%E3%82%AF%E3%83%A9%E3%82%A4%E3%82%A2%E3%83%B3%E3%83%88%E5%81%B4%E3%81%A7%E5%88%A9%E7%94%A8%E3%81%97%E3%80%81%E6%9C%80%E4%BD%8E%E9%99%90%E3%81%AE%E6%A8%A9%E9%99%90%E3%81%A7%E5%AE%89%E5%85%A8%E3%81%AB%E3%83%87%E3%83%BC%E3%82%BF%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E3%82%92%E8%A1%8C%E3%81%84%E3%81%BE%E3%81%99%E3%80%82))。仮に外部に漏えいすると、認証無しでデータベースを完全に操作されてしまう危険があります。Supabase公式も「`service_role`キーはブラウザなどユーザーが見える場所で公開しないように」強く警告しています ([Understanding API Keys | Supabase Docs](https://supabase.com/docs/guides/api/api-keys#:~:text=Never%20expose%20the%20,a%20user%20can%20see%20it))。基準としては、「RLSや認可ポリシーを無視できる権限を持つキー」はすべて秘密に管理すべきです。つまり`service_role_key`はもちろん、データベース直接接続用のURLやパスワード（`SUPABASE_DB_URL`など）も含めて、第三者に知られないようにする必要があります ([Managing Secrets (Environment Variables) | Supabase Docs](https://supabase.com/docs/guides/functions/secrets#:~:text=,connect%20directly%20to%20your%20database))。

要するに、**クライアント公開用に設計されている情報（`supabase_url`と`anon_key`）以外は一切外部に漏れないように管理**します。特に`service_role_key`はその名の通り“サービス用の秘密鍵”であり、GitHubリポジトリやフロントエンドコードに埋め込むことは避け、バックエンドの環境変数などで厳重に管理しましょう ([〖Supabase〗anon keyと service_role keyの使い分け #Security - Qiita](https://qiita.com/syukan3/items/8ec9b7288834e4c95f3b#:~:text=,%E3%82%AF%E3%83%A9%E3%82%A4%E3%82%A2%E3%83%B3%E3%83%88%E5%81%B4%E3%81%A7%E5%88%A9%E7%94%A8%E3%81%97%E3%80%81%E6%9C%80%E4%BD%8E%E9%99%90%E3%81%AE%E6%A8%A9%E9%99%90%E3%81%A7%E5%AE%89%E5%85%A8%E3%81%AB%E3%83%87%E3%83%BC%E3%82%BF%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E3%82%92%E8%A1%8C%E3%81%84%E3%81%BE%E3%81%99%E3%80%82))。

## 環境変数やCI/CDでの安全な管理方法

**環境変数（.envファイルなど）を活用して秘密情報を管理する**のが基本です。開発・運用フローにおいて、以下の点に注意してください。

- **ローカル開発**: APIキー類はプロジェクトルートの`.env`ファイルに保存し、アプリケーション起動時に読み込むようにします。例えば、`SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`といった変数名で環境変数を定義し、コード中では直接値を書かず環境変数参照で利用します。Supabaseもデフォルトでこれらの環境変数名を使用します ([Managing Secrets (Environment Variables) | Supabase Docs](https://supabase.com/docs/guides/functions/secrets#:~:text=,connect%20directly%20to%20your%20database))。**重要: .envファイルは必ずGitの追跡対象から除外**し、リポジトリにコミットしないでください ([Managing Secrets (Environment Variables) | Supabase Docs](https://supabase.com/docs/guides/functions/secrets#:~:text=Never%20check%20your%20,into%20Git))。

- **.gitignoreの設定**: `.env`やその他機密情報が書かれたファイル（例えば`supabase/config.yaml`などにキーを直書きしている場合）があれば、必ず`.gitignore`に追加し、ソース管理から除外します。誤ってコミット・プッシュするミスを防ぐために不可欠な手順です。

- **CI/CDパイプライン**: ビルドやデプロイ時には、各種キーをCI/CDサービスのシークレット変数/環境変数機能に登録して利用します。GitHub Actionsであればリポジトリの「Secrets」に、VercelやNetlify等のホスティングならプロジェクト設定の環境変数欄に、Supabaseの鍵を設定します。CI/CDのログや出力に鍵が露出しないよう、例えば`echo`で環境変数を表示しない、デバッグ情報に含めないといった配慮も必要です。

- **シークレットマネージャの活用**: 規模が大きくなったり、より安全性を高めたい場合は、HashiCorp Vaultやクラウドプロバイダのシークレットマネージャ（AWS Secrets ManagerやGoogle Secret Managerなど）に鍵を格納し、アプリ実行時に取得する方式も検討します。GitHubが提供する暗号化されたシークレットや、Supabase自身もVaultとの連携機能を提供しています ([Securing your data | Supabase Docs](https://supabase.com/docs/guides/database/secure-data#:~:text=Unlike%20your%20anon%20key%2C%20your,variable%20instead%20of%20hardcoding%20it)) ([Managing Secrets (Environment Variables) | Supabase Docs](https://supabase.com/docs/guides/functions/secrets#:~:text=,connect%20directly%20to%20your%20database))。

これらの対策によって、鍵の値がソースコード上やインフラ外部に露出するリスクを抑えることができます。特に`service_role_key`は環境変数などで厳重管理し、**ハードコーディングしない**ことが鉄則です ([Securing your data | Supabase Docs](https://supabase.com/docs/guides/database/secure-data#:~:text=,role%20key%20on%20the%20frontend))。

## GitHubへの誤公開対策

開発中にうっかりAPIキーをGitHubなどの公開リポジトリにプッシュしてしまう事故は後を絶ちません。以下の対策でそのリスクを低減しましょう。

- **コミット前のチェック**: コミット内容にAPIキーや秘密情報が含まれていないか確認します。例えばGitのフック（pre-commitフック）にスクリプトを仕込んで、特定のパターン（`supabase.co`, `eyJhbGciOi...`などJWTキーの特徴）を検知したら警告する、といった自動チェックを導入するのも有効です。

- **ツールの活用**: GitHubはデフォルトで公開リポジトリのシークレットスキャン機能を提供しており、SupabaseはGitHubと連携して**万一`service_role_key`がコミットされた場合、自動で検知・失効させる仕組み**を持っています ([Understanding API Keys | Supabase Docs](https://supabase.com/docs/guides/api/api-keys#:~:text=We%20have%20partnered%20with%20GitHub,your%20data%20against%20malicious%20actors))。これにより、公開リポジトリ上でサービスキーが漏洩しても迅速に無効化されます（Supabaseからユーザへ通知も行われます） ([Understanding API Keys | Supabase Docs](https://supabase.com/docs/guides/api/api-keys#:~:text=We%20have%20partnered%20with%20GitHub,your%20data%20against%20malicious%20actors))。また、サードパーティーのサービス（例: GitGuardian）を使ってプライベートリポジトリを含めた監視を行うこともできます。

- **誤って公開してしまった場合の対応**: 仮にコミットに含めてしまった場合、速やかにGitの履歴から該当キーを削除するとともに、Supabaseダッシュボードでキーのローテーション（再発行）を行います（具体的な手順は後述）。GitHub上でコミットを消しても履歴から完全に消去するのは困難なため、**鍵そのものを無効化・再発行**することが確実な対処となります。また、GitHubに公開後できるだけ早く対処すれば、スキャンで見つかって第三者に悪用されるリスクを減らせます。

- **オープンソースプロジェクトの場合**: 自分のSupabase鍵を他者と共有しない方が安全です。例えば公開リポジトリでSupabaseを使ったアプリを公開するなら、自分のプロジェクト用anonキーを直書きせず、`.env.example`ファイルを用意して「ここに各自SupabaseのURLとanonキーを入れてください」と促すのが良いでしょう。他者がForkした際に自分のデータベースにアクセスされる事態も防げます。

以上を徹底し、「鍵をコミットしない・公開しない」を習慣づけることが肝要です。GitHub上での自動検知機能は最後の砦と考え、まずは**流出させない予防策**を講じましょう。

## キーが漏洩した場合の対応とローテーション方法

万一、`anon_key`や`service_role_key`が漏洩してしまった場合には、迅速に対処する必要があります。基本戦略は**「当該キーを無効化し、新しいキーに切り替える（ローテーションする）」**ことです。

**Supabaseにおけるキーのローテーション:**  
Supabaseではプロジェクトの「API設定 (API Settings)」画面からJWTシークレットを再生成することで、新しい`anon_key`と`service_role_key`を発行できます ([Supabase Docs | Troubleshooting | Rotating Anon, Service, and JWT Secrets](https://supabase.com/docs/guides/troubleshooting/rotating-anon-service-and-jwt-secrets-1Jq6yd#:~:text=1,Find%20the%20JWT%20Secrets%20section))。手順は以下の通りです。

1. Supabase管理画面でプロジェクトの設定に移動し、「API」タブを開きます。  
2. 「JWT Secrets」というセクションを探し、「Generate new secret」ボタンをクリックします ([Supabase Docs | Troubleshooting | Rotating Anon, Service, and JWT Secrets](https://supabase.com/docs/guides/troubleshooting/rotating-anon-service-and-jwt-secrets-1Jq6yd#:~:text=1,Find%20the%20JWT%20Secrets%20section))。必要に応じてランダム生成かカスタムシークレットを選択できます。  
3. 確認ダイアログが表示されたら再度「Generate New Secret」を実行し、シークレットを再生成します ([Supabase Docs | Troubleshooting | Rotating Anon, Service, and JWT Secrets](https://supabase.com/docs/guides/troubleshooting/rotating-anon-service-and-jwt-secrets-1Jq6yd#:~:text=4,again))。  

これにより**JWT署名用のシークレットが新しくなり、既存の全てのAPIキー（anon, service_role）が即座に無効化**されます ([Supabase Docs | Troubleshooting | Rotating Anon, Service, and JWT Secrets](https://supabase.com/docs/guides/troubleshooting/rotating-anon-service-and-jwt-secrets-1Jq6yd#:~:text=4,connections%20to%20begin%20working%20again))。内部的にはPostgresサーバの再起動と設定更新が行われ、新しいシークレットに基づいた鍵が発行されます ([Supabase Docs | Troubleshooting | Rotating Anon, Service, and JWT Secrets](https://supabase.com/docs/guides/troubleshooting/rotating-anon-service-and-jwt-secrets-1Jq6yd#:~:text=Image%3A%20Screenshot%202023,new%20anon%20and%20service%20keys))。その結果、漏洩したキーではこれ以上アクセスできなくなります。

**ローテーション後の対応:**  
新しいキーが発行されたら、速やかにアプリケーションや環境変数を更新しデプロイし直します。旧キーは使えなくなっているため、アプリが新キーを使うようにしないとサービスに障害が生じます ([Supabase Docs | Troubleshooting | Rotating Anon, Service, and JWT Secrets](https://supabase.com/docs/guides/troubleshooting/rotating-anon-service-and-jwt-secrets-1Jq6yd#:~:text=4,again))。また、漏洩が`service_role_key`であった場合には、漏洩期間中に不正アクセスが行われていないか、データベースの監査ログやSupabaseの監視機能を確認することも大切です。必要に応じてユーザーへのパスワードリセット対応や、流出したデータがないかの調査も検討します。

**定期的なキーのローテーション:**  
仮に漏洩事故がなくとも、セキュリティポリシー上**定期的にキーをローテーションすること**が推奨される場合があります ([Remediating Supabase JWT Secret leaks | GitGuardian](https://www.gitguardian.com/remediation/supabase-jwt-secret#:~:text=,who%20have%20access%20to%20the))。長期間同じシークレットを使い続けると、それだけ露出のリスクも高まるため、例えば半年～年単位でキーを再生成する運用も検討できます（その際は利用中のサービスへの影響を考慮し、計画的に行うようにします）。

まとめると、**キーが漏れたら「急いで無効化（ローテーション）して被害を封じる」**ことが第一です。その後、新キーへの切り替えと影響範囲のチェックまで含めて対処しましょう。

## SupabaseのRLS（Row Level Security）との連携とセキュリティ強化

**Row Level Security (RLS)**はSupabase（PostgreSQL）が提供する強力なセキュリティ機構で、**各テーブルの行ごとにアクセス権限を制御できる**ものです。SupabaseのデータAPIはRLSと組み合わせて使う前提で設計されており、RLSを正しく使うことで`anon_key`の公開を前提としたフロントエンド直接アクセスでもデータを保護できます ([Securing your data | Supabase Docs](https://supabase.com/docs/guides/database/secure-data#:~:text=Your%20anon%20key%20is%20safe,logged%20in%20using%20Supabase%20Auth))。

- **RLSの有効化:** 新規に作成したテーブルに対しては**必ずRLSを有効**にします。Supabaseのダッシュボードからテーブル作成した場合、デフォルトでRLSが有効になっていることがありますが、SQLエディタ経由でテーブル追加した場合などは自分で有効化する必要があります ([Securing your API | Supabase Docs](https://supabase.com/docs/guides/api/securing-your-api#:~:text=Any%20table%20created%20through%20the,way%2C%20enable%20RLS%20like%20so))。有効化してポリシーを設定するまでは、匿名キー経由のアクセスはそのテーブルに対して拒否されるため、意図しないデータ漏洩を防ぐ初期状態となります ([SupabaseのRow Level Security (RLS) を理解する](https://push.co.jp/articles/supabase-row-level-security-rls#:~:text=RLS%E3%82%92%E6%9C%89%E5%8A%B9%E3%81%AB%E3%81%99%E3%82%8B%E3%81%A8%E3%80%81%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC%E3%82%92%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B%E3%81%BE%E3%81%A7public%E3%81%AE%20))。

- **ポリシーの設定:** RLSを有効にしただけではデータへのアクセスはすべてブロックされます。各テーブルごとにポリシー（ルール）を作成し、「どのロール（anonやauthenticated）が、どの条件下で、どの操作（SELECT/INSERT/UPDATE/DELETE）を許可するか」を定義します。例えば「anonロールには`profiles`テーブルの読み取りを許可するが書き込みは許可しない」や「authenticatedロールのユーザーは自分が所有する行（例: `user_id`が自分のUIDである行）のみ更新可」等の細かな制御が可能です ([Understanding API Keys | Supabase Docs](https://supabase.com/docs/guides/api/api-keys#:~:text=The%20,table)) ([Understanding API Keys | Supabase Docs](https://supabase.com/docs/guides/api/api-keys#:~:text=create%20policy%20,true))。

- **anonキーとRLSの関係:** 先述のように`anon_key`でのアクセスは`anon`ロールとして扱われます。このロールにはデフォルトでは権限がほとんどありませんが、**RLSポリシーで許可された操作だけが可能**になります ([〖Supabase〗anon keyと service_role keyの使い分け #Security - Qiita](https://qiita.com/syukan3/items/8ec9b7288834e4c95f3b#:~:text=,%E8%AA%8D%E8%A8%BC%E7%8A%B6%E6%85%8B%E3%81%AB%E3%82%88%E3%82%8B%E3%83%AD%E3%83%BC%E3%83%AB%E3%81%AE%E5%A4%89%E5%8C%96%20%E3%83%A6%E3%83%BC%E3%82%B6%E3%83%BC%E3%81%8C%E3%83%AD%E3%82%B0%E3%82%A4%E3%83%B3%E3%81%99%E3%82%8B%E3%81%A8%E3%80%81%E8%87%AA%E5%8B%95%E7%9A%84%E3%81%AB%E3%80%8Cauthenticated%E3%80%8D%E3%83%AD%E3%83%BC%E3%83%AB%E3%81%AB%E5%88%87%E3%82%8A%E6%9B%BF%E3%82%8F%E3%82%8A%E3%80%81%E3%83%A6%E3%83%BC%E3%82%B6%E3%83%BC%E5%B0%82%E7%94%A8%E3%81%AE%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E6%A8%A9%E3%81%8C%E9%81%A9%E7%94%A8%E3%81%95%E3%82%8C%E3%81%BE%E3%81%99%E3%80%82))。したがって、`anon_key`が外部に知られていたとしても、RLSによってポリシーで許可されたデータしか参照・操作できません。例えば、RLS無効のテーブルがあると**anonキーを持つ誰もがそのテーブルを自由に閲覧・操作できてしまう**のに対し ([Securing your API | Supabase Docs](https://supabase.com/docs/guides/api/securing-your-api#:~:text=Any%20table%20without%20RLS%20enabled,access%20to%20your%20project%27s%20data))、RLS有効＋適切なポリシー設定済みであれば匿名ユーザーには必要最低限のデータしか見せない、といったことが実現できます。

- **サービスロールとRLS:** これに対し、`service_role_key`でアクセスする場合はRLSが**無視**されます ([Securing your data | Supabase Docs](https://supabase.com/docs/guides/database/secure-data#:~:text=,role%20key%20on%20the%20frontend))。つまりサービスロールにはDB上のあらゆる行・あらゆる操作が許可されます。このため、サービスキーの漏洩は甚大な被害に直結しますが、逆に言えば**サービスキーさえ厳重に守っておけば、RLSによって公開用のanonキーからは保護されたデータは見られない**ということでもあります。RLSはデータベース側で強制されるルールのため、フロントエンドからの直接アクセスであっても、サーバーサイドを経由するアクセスであっても、一貫したセキュリティポリシーを適用できます。

RLSを有効活用することで、Supabaseは**「フロントエンド直アクセスの利便性」と「データ保護」の両立**を図っています。開発者はキーの管理と合わせてポリシーの管理にも注意を払い、想定どおりのアクセス制限になっているかテストすることが重要です。特に、誤って`using (true)`のポリシーを入れて全行アクセスを許可してしまった、逆に厳しすぎて必要なデータ取得もできなくなっていた、ということがないように細心の注意を払いましょう。

## 実運用でよくあるミスやトラブルの防止策

最後に、Supabaseの運用で陥りがちなミスやトラブルと、その防止策をまとめます。

- **サービスキーをフロントエンドに含めてしまうミス**: 初心者にありがちな誤りとして、`service_role_key`を誤ってクライアント側で使ってしまうケースがあります。これは重大なセキュリティホールとなります。防止策: サービスキーとAnonキーの用途をチーム内で周知徹底し、フロントエンドのコードレビュー時にキーの種類をチェックする運用を取り入れます。「サービスキーはサーバーのみ」という原則をドキュメント化しておくのも有効です ([〖Supabase〗anon keyと service_role keyの使い分け #Security - Qiita](https://qiita.com/syukan3/items/8ec9b7288834e4c95f3b#:~:text=,%E3%82%AF%E3%83%A9%E3%82%A4%E3%82%A2%E3%83%B3%E3%83%88%E5%81%B4%E3%81%A7%E5%88%A9%E7%94%A8%E3%81%97%E3%80%81%E6%9C%80%E4%BD%8E%E9%99%90%E3%81%AE%E6%A8%A9%E9%99%90%E3%81%A7%E5%AE%89%E5%85%A8%E3%81%AB%E3%83%87%E3%83%BC%E3%82%BF%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E3%82%92%E8%A1%8C%E3%81%84%E3%81%BE%E3%81%99%E3%80%82))。

- **RLSを有効にし忘れる/誤ったポリシー設定**: RLS無効のテーブルが残っていると、Anonキー経由で誰でもそのデータにアクセスできてしまいます ([Securing your API | Supabase Docs](https://supabase.com/docs/guides/api/securing-your-api#:~:text=Any%20table%20without%20RLS%20enabled,access%20to%20your%20project%27s%20data))。また、ポリシーの条件漏れやミスにより意図せずデータが公開状態になるリスクもあります。防止策: テーブル作成時にRLS有効をデフォルトとし、ポリシー設定チェックリストを作成すること。運用中も定期的にポリシーをレビューし、想定外に緩い設定がないか監査します。特に公開前のセキュリティテストで、Anonキーで保護すべきデータにアクセスできないことを確認することが重要です。

- **APIキーのハードコーディング**: ソースコード内に直接キーを書いてしまうと、公開リポジトリでなくとも誤って漏洩する可能性が高まります。防止策: 環境変数を必ず使用し、開発者全員が.env管理に統一すること。.envファイルの共有には安全なストレージや社内ツールを使い、メールやチャットで平文送信しないようにします。

- **Gitリポジトリへのキーコミット**: こちらも人的ミスですが、うっかりコミット・プッシュしてしまう事故です。防止策: 前述の.gitignore設定や自動検出ツールで未然に防ぐこと。万一公開リポジトリに流出した場合は、速やかにSupabaseでキーをローテーションしてください ([Supabase Docs | Troubleshooting | Rotating Anon, Service, and JWT Secrets](https://supabase.com/docs/guides/troubleshooting/rotating-anon-service-and-jwt-secrets-1Jq6yd#:~:text=Have%20you%20ever%20accidentally%20committed,keys%20for%20your%20Supabase%20project))。GitHubの秘密スキャンやSupabaseの自動失効機能にも気づき次第対応できるよう、登録メールの通知を見逃さないことも大切です ([Understanding API Keys | Supabase Docs](https://supabase.com/docs/guides/api/api-keys#:~:text=We%20have%20partnered%20with%20GitHub,your%20data%20against%20malicious%20actors))。

- **Anonキーの誤用**: Anonキーはあくまで限定的な操作のためのキーです。例えば、フロントエンドから大きな権限の操作（大量のデータ更新や削除等）を行おうとすると、設計上無理が生じたりセキュリティ上の懸念が出ます ([supabase, backend and frontend pros and cons : r/Supabase](https://www.reddit.com/r/Supabase/comments/1g2n57k/supabase_backend_and_frontend_pros_and_cons/#:~:text=The%20most%20secure%20approach%20is,end%20client%20for%20everything%20else))。防止策: フロントエンドからの操作は必要最小限に留め、複雑な処理や広範なデータ変更はバックエンド側のロジック（Edge Functionやサーバー）で実行するようにアーキテクチャを構築します。こうすることで、フロントにはAnonキーしか置かずに済み、安全性と拡張性が向上します ([supabase, backend and frontend pros and cons : r/Supabase](https://www.reddit.com/r/Supabase/comments/1g2n57k/supabase_backend_and_frontend_pros_and_cons/#:~:text=The%20most%20secure%20approach%20is,end%20client%20for%20everything%20else))。

- **キーの長期利用**: 運用が長くなると、開発当初に発行した鍵をそのままずっと使い続けてしまうことがあります。防止策: 定期的なセキュリティレビューを行い、必要に応じて鍵の再発行を計画に入れます（例えば大規模な人事異動やオープンソース公開前など節目で検討）。ローテーション実施時は、本番環境への影響を考慮し、メンテナンスウィンドウを設けるなど計画的に行いましょう。

以上のベストプラクティスを押さえておけば、Supabaseの`supabase_url`・`anon_key`・`service_role_key`を適切に管理でき、アプリケーションのセキュリティを高い水準で維持できます。**鍵の管理とRLSによるデータアクセス制御を両輪**として、安全かつ効率的なSupabase運用を心がけてください。必要に応じて公式ドキュメントやコミュニティの最新情報も参照し、継続的にセキュリティ対策をアップデートしていきましょう。 ([Securing your data | Supabase Docs](https://supabase.com/docs/guides/database/secure-data#:~:text=Your%20anon%20key%20is%20safe,logged%20in%20using%20Supabase%20Auth)) ([〖Supabase〗anon keyと service_role keyの使い分け #Security - Qiita](https://qiita.com/syukan3/items/8ec9b7288834e4c95f3b#:~:text=,%E3%82%AF%E3%83%A9%E3%82%A4%E3%82%A2%E3%83%B3%E3%83%88%E5%81%B4%E3%81%A7%E5%88%A9%E7%94%A8%E3%81%97%E3%80%81%E6%9C%80%E4%BD%8E%E9%99%90%E3%81%AE%E6%A8%A9%E9%99%90%E3%81%A7%E5%AE%89%E5%85%A8%E3%81%AB%E3%83%87%E3%83%BC%E3%82%BF%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E3%82%92%E8%A1%8C%E3%81%84%E3%81%BE%E3%81%99%E3%80%82))

