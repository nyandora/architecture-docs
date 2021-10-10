書籍「Clean Architecure」は良書です。この記事では、この書籍の内容をできるだけ直感的に分かるように記してみたいと思います。

この記事を通して、書籍「Clean Architecure」に興味を持ってもらえたら嬉しいです。アーキテクトを目指す方には強くオススメします。

# アーキテクチャ設計の目的とは？
アーキテクチャ設計をする目的は、構築・保守にかかるコストを小さくすることです。最小限の工数で構築・保守できるアーキテクチャが良いアーキテクチャです。良いアーキテクチャであれば、システムの理解・開発・保守・デプロイが簡単になり、その結果としてコストが小さくなります。

一方、無策に開発を進めると、コンポーネントを分割する等の整理が疎かになってしまいます。なぜなら、コンポーネントを分割すると、開発作業が増えてしまって面倒だからです。例えば複数のコンポーネントを別々のサーバに分けるとすれば、開発時にそれらのサーバでコンポーネントを起動する必要があります。面倒ですよね。その結果、デプロイ・運用・保守をしづらいアーキテクチャとなってしまいます。例えば・・・

* デプロイのコストが高くつきます。デプロイ時に、コンポーネント間を接続することが大変となったり、コンポーネント（マイクロサービス）の起動タイミングなどが問題となり、関係者との調整が必要となったりします。簡単にデプロイできるようでないと、そのコストは高くつきます。

* 保守のコストが高くつきます。修正を加えるとき、他に影響を与えないように注意深く作業する必要がありますが、適切にコンポーネントが分割・設計されていなければ、不具合を混入するリスクが高まりますし、調査やテストのコストが高くついてしまいます。


オブジェクト指向プログラミングの本質

・実装の詳細を隠し、意識する必要がなくなる。生産性と品質が高まる。
・依存関係の方向を自在に制御できる。制御の流れと、ソースコードの依存関係を別物として扱うことができる。OCP, DIPに関わる。


　


関数型プログラミングの本質

競合状態、デッドロック、並行更新の原因が、すべて可変変数にある。なので、不変性という観点はアーキテクチャにおいてすごく重要。不変コンポーネントにできるだけ多くの処理を押し込み、可変コンポーネントからはできるだけ多くのコードを追い払うべき。関数型言語では可変変数を出来る限り排除しようとしている。この点で、関数型言語は優れている。

単一責任の原則(SRP:Single Responsibility Principle)
　アーキテクチャのレベルでいうと、モジュールはたったひとつのアクターに対して責務を負うべきである。
　モジュールはたったひとつだけの処理をこなすべき、という意味ではない。

オープン・クローズドの原則（OCP:Open-Closed Principle）
　ソフトウェアの構成要素は、拡張に対して開いていて、修正に対して閉じているべき。
　要するに、既存の成果物を変更せずに拡張できるようにすべき、ということ。

　これを実現するには、制御の流れと、コードの依存関係を独立させる。具体的には、呼び出し元が呼び出し先に依存しないようにする。
　呼び出し元が呼び出し先に依存しない、ということは、呼び出し元のソースコード中に、呼び出し先のコードが登場しないということなので、呼び出し先のコードが変更されても、コード上は呼び出し元は影響を受けない。具体的には、呼び出し元のコードを再コンパイル、再デプロイする必要はない。結果、コンポーネント間の独立性を高め、独立してデプロイできるようになる。呼び出し元が、呼び出し先の変更からのコード上の影響を受けないようにできる、ということ。

リスコフの置換原則(LSP: Liskov Substitution Principle)
　この本質は、呼び出し先を他のパーツに変更しても、呼び出し元がコード上の影響を受けないようにすべき、ということ。
　呼び出し先のクラスがAの場合、呼び出し元が特別な処理（if文など）をするようであってはいけない。このような「特別扱い」を排除する、ということ。

　単にクラスの継承、インターフェースの実装といった狭い領域だけに適用されるものではなく、
　アーキテクチャのレベルについても拡張して適用されるべき。

インターフェイス分離の原則(ISP: Interface Segregation Principle)
　呼び出し元は、呼び出し先のもつ要素のうち使っていないものに依存すべきではない。
　その使っていないものが変更になったときに、呼び出し元の再コンパイル・再デプロイが必要になってしまう。
　
　アーキテクチャのレベルでも同じことがいえる。
　アプリ→FW→DBライブラリ　という依存関係で、FWが使っていないDBライブラリの一部が変更された場合、連鎖的にアプリも影響を受けてしまう。

依存関係逆転の原則(DIP: Dependency Inversion Principle)
　制御の流れに対して、コード上の依存関係は逆行すべきというもの。具体的には、呼び出し元は具象ではなく、抽象に依存するようにする。
　変化しやすい具象要素に、呼び出し元が影響を受けないようにするのが目的。
　
コンポーネント＝デプロイの単位。JARなどにできる単位、ということ。


コンポーネントにどの部品（クラスなど）を含めて、どれを含まないべきかを決めるにあたって、
　・再利用・リリース等価の原則(REP)
　・閉鎖性共通の原則(CCP)
　・全再利用の原則(CRP)
といった原則を考慮すべき。

全再利用の原則(CRP)
使われるクラスと使われないクラスがコンポーネントに混在してはいけない。ISPのコンポーネント版。

各原則はトレードオフの関係にあって、バランスをとるようにすべき。そこが難しいところ。


コンポーネントの結合

・非循環依存関係の原則(ADP)
・安定依存の原則(SDP)
・安定度・抽象度等価の原則(SAP)


開発の初期段階で
　・DBを決定する必要はない
　・Webサーバー製品を決める必要はない
　・Webで配信するかどうかを決める必要すらない
　・RESTを採用する必要はない
　・MSAを採用する必要はない
　・DIフレームワークを導入する必要はない

なぜなら、比較したり実験する時間を確保したり、情報をそろえる時間を確保すべきだから。
イケてるアーキテクトは「まだ決まってない」と言って、多くの選択肢から選べる状態を維持しておくべき。

出来る限り、コアなビジネスルールを表すコンポーネントと、実装の詳細を表すようなコンポーネントとの間に境界線を引いて、実装の詳細の決定を後回しにしたいところ。そのためには、依存関係逆転の原則で、具象から抽象に向かって依存するようにするのが良い。

優れたアーキテクチャはユースケースを中心とするものであり、フレームワークやDBなどのツールを中心したものであってはならない。


クリーンアーキテクチャの同心円の図。
コアなコードがコアじゃないコードに依存しないようにして、前者を自由に変更できるようにすべき、ということ。
実態としてはコアなコードからコアじゃないコードに制御が移っていくが、そこは依存性逆転の原則に従って実装することで、その制御の移ろいを実現する。

ということで、コアなコードは（ビジネスルールは）
　・フレーフワーク非依存。フレームワークの制約でシステムを縛るのではなく、フレームワークをツールとして使うようにすべき。
　　Springを使うなら、ビジネスオブジェクトに`@autowired`を散りばめるのはNG。ビジネスオブジェクトのようなコアなコードが、特定のFWに依存するのはNG、ということ。
　　代わりに、MainコンポーネントにDIで依存性を注入する。Mainコンポーネントは通常の方法（依存関係逆転の技法など）によって依存関係を散りばめる。
　　データストアといった詳細などかな。

　・その他の要素がなくてもテストできる
　・UI非依存。他の要素を変更しなくてもUIだけを差し替えることができる。
　・データベース非依存。
　・外部とのインタフェースに依存しない。



モノリシックが悪い、ということではない。そういった中でもコンポーネントの分割や設計を適切に行うことで、コスト低下を実現できる。


第２７章「サービス：あらゆる存在」はすごく重要だと思う。
マイクロサービスに分割したとして、全サービスにわたって修正が必要になることはある。
アーキテクチャの境界はサービスとサービスの間にあるわけではなく、各サービスを横断して各サービス内にコンポーネント間の境界をもたらすことになる。
それに対応できるようなアーキテクチャにしておくべき。大きな粒度でSOLID原則を体現することで実現できる。（タクシー配車サービスに、子猫宅配機能を追加する例）


テストはプロダクションコードと強く結合してはいけない。
こうすると、プロダクションコードを少し変更しただけで多数のテストが失敗するようになったり、
それによってプロダクションコードの柔軟な変化が妨げられてしまう。
プロダクションコードのクラス１つに対してテストクラスを１つ用意するような構造的結合は最悪の形態。
解決策は、プロダクションコードとテストコードの間に１層を設けることによって、テストコードにプロダクションコードの構造を意識させないようにする。
おそらく、テストは抽象的なビジネスルールに対して行うようにするってことだと思う。その中ではもちろんGUIといった変化しやすいものに依存してはいけない。コアなビジネスルールのみをテストするようにする。
テストをシステムの設計にうまく統合する必要がある。



懸念
・普段の開発では他サービスをPC上に起動するのか？サーバ上に起動するなら自動テストはどうなるのか？
・自動テストでのモックはどうするのか？
・他サービスの死活を意識したデプロイをするはめになるのでは？その他、他サービスを意識したり調整が結局必要になり、独立してデプロイする、ということが実際にはできないかもしれない。
・単純な機能を実現するだけで、他サービスを呼びまくらないといけないとすれば、それはただの無駄ではないか？コストが高くつくのでは？
・サービス間のインターフェースが頻繁に変わることで、呼び出し元の再デプロイがバンバン必要になるような状況は想定されないか？
・サービス間で共有するデータというのものが実質的には存在する。ユーザーのデータなど。その構造に変更があれば、複数のサービスが影響を受けることになる。これはやりたかったことなのだろうか？
・このように複数や全てのサービスの修正が必要な、新機能の追加というのは普通にあり得る話で、それに対応できるようなアーキテクチャにしておく必要がある。
・他サービスが停止していた場合、どのように画面表示すべきか？例えばカートのサービスが停止してた場合、カートに入れられないんだからカートを表示しない、とかやるのか？



参考資料
Clean Architecture