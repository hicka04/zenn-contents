---
title: "Combineを使ったVIPERアーキテクチャの実装"
emoji: "🐍"
type: "tech"
topics:
  - "iOS"
  - "Swift"
  - "Combine"
  - "VIPER"
  - "SwiftUI"
published: true
---

# はじめに
## VIPERアーキテクチャとは
以下記事にまとめているので、詳しくはこちらをご覧ください

https://qiita.com/hicka04/items/09534b5daffec33b2bec

* 各コンポーネントの頭文字を取ってVIPER
    * View, Interactor, Presenter, Entity, Router
* 単一責任の原則に則って適切に分割する
* 各コンポーネントはprotocolを介してやり取りする
    * Mockに差し替えてテストコードを書きやすくする

## Combineとは
以下記事をとても参考にさせていただいているので、詳しくはこちらをご覧ください

https://qiita.com/shiz/items/5efac86479db77a52ccc

* Apple公式の関数型リアクティブプログラミングフレームワーク
    * iOS界隈ではRxSwiftやReactiveSwiftなどのOSSが有名
* 時間とともに非同期的に変化する値を処理する方法を宣言的に書くことができる

# Combine × VIPERアーキテクチャ
## 考慮したこと
* Combineを使わない手続き的なVIPERアーキテクチャから移行しやすいこと
* ViewをSwiftUIに移行しやすいこと
* テストコードが書けること

## 完成形
[サンプルコード](https://github.com/hicka04/viper-combine-sample)
* QiitaのAPIを叩いて一覧表示
* 一覧をタップして詳細遷移

### Presenter
Combineを導入するにあたって、一番変化が大きかった

#### ポイント
* Presenterを `ObservableObject` に準拠させる
    * 実装を共通化するための [`Presentation` protocol](https://github.com/hicka04/viper-combine-sample/blob/main/ViperCombineSample/Modules/Presentation.swift)が `ObservableObject` に準拠するよう実装している
* ViewからPresenterへのイベント通知に `PassthroughSubject` を利用
    * Viewで発生するイベントの一覧をenumで定義し、 `PassthroughSubject` に流す
    * 流れてきたイベントに応じてInteractorやRouterなどの各所に処理の依頼を投げる
* Presenterに `@Published` なプロパティを公開しておく
    * 画面更新の流れは後述のViewの章で
    * 各画面のPresenter用のprotocolは作成しない
        * Presenterのprotocolを作ったパターンも実装してみたが、Presenterで公開したい `@Published` な変数をprotocolで公開するのが面倒だったため諦めた
            * Viewのテストを書かない場合はprotocolなしでも問題はないと判断し、書きやすさを優先した
        * 詳しくは以下の記事をご覧ください
            * [[iOS]Published attributeが付与されたプロパティをprotocolのrequirementsに含められない](https://dev.classmethod.jp/articles/published_attribute_in_protocol/)
* できるだけPresenterのinit内で宣言しておき、メソッドやプロパティを極力作らない
    * View → Presenterはprotocolなしで参照するため、privateではないメソッドやプロパティを作ってしまうと密結合を生んでしまうため

#### サンプルコード
[ArticleSearchPresenter.swift](https://github.com/hicka04/viper-combine-sample/blob/main/ViperCombineSample/Modules/ArticleSearch/Presenter/ArticleSearchPresenter.swift)
```swift:ArticleSearchPresenter.swift
import Foundation
import Combine
import CombineSchedulers

enum ArticleSearchViewEvent {
    case viewDidLoad
    case refreshControlValueChanged
    case didSelect(article: ArticleModel)
}

final class ArticleSearchPresenter: Presentation {
    private var cancellables: Set<AnyCancellable> = []
    let viewEventSubject = PassthroughSubject<ArticleSearchViewEvent, Never>()
    
    @Published var articles: [ArticleModel] = []
    @Published var articleSearchError: ArticleSearchError?
    
    init<
        Router: ArticleSearchWireframe,
        ArticleSearchInteractor: ArticleSearchUsecase
    >(
        mainScheduler: AnySchedulerOf<DispatchQueue> = .main,
        router: Router,
        articleSearchInteractor: ArticleSearchInteractor
    ) {
        let searchKeywordSubject = PassthroughSubject<String, Never>()
        
        // 受け取ったイベントをもとに処理の依頼を各所に投げる
        viewEventSubject
            .sink { event in
                switch event {
                case .viewDidLoad, .refreshControlValueChanged:
                    searchKeywordSubject.send("Swift")
                    
                case .didSelect(let article):
                    router.navigationSubject.send(.articleDetail(article))
                }
            }.store(in: &cancellables)
        
        // キーワードが変更されたら記事を検索して結果を`@Published`のプロパティにセット
        searchKeywordSubject
            .flatMap { searchKeyword in
                articleSearchInteractor
                    .execute(searchKeyword)
                    .convertToResultPublisher()
            }.receive(on: mainScheduler)
            .sink { [weak self] result in
                switch result {
                case .success(let articles):
                    self?.articles = articles
                    
                case .failure(let error):
                    self?.articleSearchError = error
                }
            }.store(in: &cancellables)
    }
}
```

### View
#### ポイント
* Presenterに公開されている `@Published` の変数の変更を監視して画面更新を実施
    * 状態とUIを同期させやすくなった
* 同じPresenterを利用し、画面だけUIKitとSwiftUIのどちらでも実装
    * 画面単位で徐々にSwiftUIへの移行が可能

#### サンプルコード
##### UIKit
[ArticleSearchViewController.swift](https://github.com/hicka04/viper-combine-sample/blob/main/ViperCombineSample/Modules/ArticleSearch/View/UIKit/ArticleSearchViewController.swift)
```swift:ArticleSearchViewController.swift
import UIKit
import Combine
import CombineCocoa

class ArticleSearchViewController: UICollectionViewController {
    private let presenter: ArticleSearchPresenter
    
    // ...

    override func viewDidLoad() {
        super.viewDidLoad()
        
        // ...
        
        // 記事の一覧が更新されたらリストを更新
        presenter.$articles
            .sink { [weak self] articles in
                var snapshot = NSDiffableDataSourceSnapshot<Int, ArticleModel>()
                snapshot.appendSections([0])
                snapshot.appendItems(articles, toSection: 0)
                self?.dataSource.apply(snapshot, animatingDifferences: true) {
                    self?.collectionView.refreshControl?.endRefreshing()
                }
            }.store(in: &cancellables)
        
        // エラーが発生したらアラートを出す
        presenter.$articleSearchError
            .compactMap { $0 }
            .sink { [weak self] error in
                let alert = UIAlertController(
                    title: error.errorDescription,
                    message: error.recoverySuggestion,
                    preferredStyle: .alert
                )
                alert.addAction(.init(title: "OK", style: .default, handler: nil))
                self?.present(alert, animated: true) {
                    self?.collectionView.refreshControl?.endRefreshing()
                }
            }.store(in: &cancellables)
        
        // Viewで発生したイベントをPresenterに通知
        presenter.viewEventSubject.send(.viewDidLoad)
    }
}
```

##### SwiftUI
[ArticleSearchView.swift](https://github.com/hicka04/viper-combine-sample/blob/main/ViperCombineSample/Modules/ArticleSearch/View/SwiftUI/ArticleSearchView.swift)
```swift:ArticleSearchView.swift
import SwiftUI

struct ArticleSearchView: View {
    @ObservedObject var presenter: ArticleSearchPresenter
    
    var body: some View {
        ArticleListView(
            articles: presenter.articles, // 記事の一覧が更新されたらリストを更新
            onTapArticle: { article in
                presenter.viewEventSubject.send(.didSelect(article: article))
            }
        )
        .alert(item: $presenter.articleSearchError) { error in // エラーが発生したらアラートを出す
            Alert(
                title: .init(error.errorDescription),
                message: error.recoverySuggestion.map { .init($0) },
                dismissButton: nil
            )
        }.navigationBarTitle(Text("Articles"), displayMode: .large)
        .onAppear {
            // Viewで発生したイベントをPresenterに通知
            presenter.viewEventSubject.send(.viewDidLoad)
        }
    }
}

extension ArticleSearchView {
    struct ArticleListView: View {
        let articles: [ArticleModel]
        let onTapArticle: (ArticleModel) -> Void
        
        var body: some View {
            List(articles) { article in
                HStack {
                    Text(article.title)
                    Spacer()
                }
                .contentShape(Rectangle())
                .onTapGesture {
                    onTapArticle(article)
                }
            }.listStyle(.plain)
        }
    }
}
```

### Router
#### ポイント
* 画面遷移依頼通知に `PassthroughSubject` を利用
    * 遷移先をenumで定義し、 `PassthroughSubject` に流す

#### サンプルコード
[ArticleSearchRouter.swift](https://github.com/hicka04/viper-combine-sample/blob/main/ViperCombineSample/Modules/ArticleSearch/Router/ArticleSearchRouter.swift)
```swift:ArticleSearchRouter.swift
import UIKit
import SwiftUI
import Combine

enum ArticleSearchDestination: Equatable {
    case articleDetail(_ article: ArticleModel)
}

protocol ArticleSearchWireframe: Wireframe where Destination == ArticleSearchDestination {}

final class ArticleSearchRouter: ArticleSearchWireframe {
    private var cancellables: Set<AnyCancellable> = []
    fileprivate weak var viewController: UIViewController?
    let navigationSubject = PassthroughSubject<ArticleSearchDestination, Never>()
    
    private init() {
        // 画面遷移
        navigationSubject
            .sink { destination in
                switch destination {
                case .articleDetail(let article):
                    let articleDetailView = ArticleDetailRouter.assembleModules(article: article)
                    self.viewController?.navigationController?.pushViewController(articleDetailView, animated: true)
                }
            }.store(in: &cancellables)
    }
    
    // DI
    static func assembleModules() -> UIViewController {
        let router = ArticleSearchRouter()
        let qiitaDataStore = QiitaDataStore()
        let articleSearchInteractor = ArticleSearchInteractor(qiitaRepository: qiitaDataStore)
        let presenter = ArticleSearchPresenter(router: router, articleSearchInteractor: articleSearchInteractor)
        let view = ArticleSearchViewController(presenter: presenter)
        
        router.viewController = view
        
        return view
    }
}
```

### Interactor
#### ポイント
* Interactorで実施するビジネスロジックの結果を返却する際にPublisherを利用

#### サンプルコード
[ArticleSearchInteractor.swift](https://github.com/hicka04/viper-combine-sample/blob/main/ViperCombineSample/Domain/Usecases/ArticleSearchInteractor/ArticleSearchInteractor.swift)
```swift:ArticleSearchInteractor.swift
import Foundation
import Combine

struct ArticleSearchError: UsecaseError, Identifiable {
    private let error: QiitaRepositoryError
    
    var errorDescription: String {
        // ...
    }
    
    var recoverySuggestion: String? {
        // ...
    }
    
    var id: String {
        error.localizedDescription
    }
    
    init(error: QiitaRepositoryError) {
        self.error = error
    }
}

protocol ArticleSearchUsecase: Usecase
where Input == String,
      Output == [ArticleModel],
      Failure == ArticleSearchError {}

final class ArticleSearchInteractor {
    private let qiitaRepository: QiitaRepository
    
    init(qiitaRepository: QiitaRepository) {
        self.qiitaRepository = qiitaRepository
    }
}

extension ArticleSearchInteractor: ArticleSearchUsecase {
    func execute(_ input: String) -> AnyPublisher<[ArticleModel], ArticleSearchError> {
        qiitaRepository
            .searchArticles(keyword: input)
            .mapError { .init(error: $0) }
            .eraseToAnyPublisher()
    }
}
```

## テストコード
* [combine-schedulers](https://github.com/pointfreeco/combine-schedulers)というライブラリを使って実装
    * iOSDC2021の｢Combineを使ったコードのテストをSchedulerで操る方法とその仕組み｣というセッションが大変参考になります
        * [動画](https://www.youtube.com/watch?v=2MlCr72eUiY&list=PLod2oSGQp3W4cUoZ19vooF4dlStBIBdd0)
        * [スライド](https://www.docswell.com/s/kalupas226/ZJ4PE5-2021-10-05-075350)

### PresenterTests
[ArticleSearchPresenterTests.swift](https://github.com/hicka04/viper-combine-sample/blob/main/ViperCombineSampleTests/Modules/ArticleSearch/ArticleSearchPresenterTests.swift)
```swift:ArticleSearchPresenterTests.swift
@testable import ViperCombineSample
import Quick
import Nimble
import Combine
import CombineSchedulers
import OrderedCollections

final class ArticleSearchPresenterTests: QuickSpec {
    override func spec() {
        var cancellables: Set<AnyCancellable> = []
        var testScheduler: TestSchedulerOf<DispatchQueue>!
        
        var presenter: ArticleSearchPresenter!
        var router: MockArticleSearchRouter!
        var articleSearchInteractor: MockArticleSearchInteractor!
        
        var articlesOutputs: [[ArticleModel]] = []
        var articleSearchErrorOutputs: [ArticleSearchError?] = []
        var navigationOutputs: [ArticleSearchDestination] = []
        
        beforeEach {
            testScheduler = DispatchQueue.test
            
            router = .init()
            articleSearchInteractor = .init()
            presenter = .init(
                mainScheduler: testScheduler.eraseToAnyScheduler(),
                router: router,
                articleSearchInteractor: articleSearchInteractor
            )
            
            presenter.$articles.sink { articlesOutputs.append($0) }.store(in: &cancellables)
            presenter.$articleSearchError.sink { articleSearchErrorOutputs.append($0) }.store(in: &cancellables)
            router.navigationSubject.sink { navigationOutputs.append($0) }.store(in: &cancellables)
        }
        
        afterEach {
            cancellables = []
            articlesOutputs = []
            articleSearchErrorOutputs = []
            navigationOutputs = []
        }
        
        describe("viewDidLoad") {
            beforeEach {
                testScheduler.schedule {
                    presenter.viewEventSubject.send(.viewDidLoad)
                }
            }
            
            context("articleSearchInteractorの返却値がエラーのとき") {
                let error = ArticleSearchError(error: .connectionError(NSError(domain: "hoge", code: -1, userInfo: nil)))
                
                beforeEach {
                    testScheduler.schedule(after: testScheduler.now.advanced(by: 10)) {
                        articleSearchInteractor.executeResult.send(completion: .failure(error))
                    }
                    
                    testScheduler.advance(by: 10)
                }
                
                it("articleSearchErrorが更新される") {
                    expect(articleSearchErrorOutputs) == [
                        nil,
                        error
                    ]
                }
            }
            
            context("articleSearchInteractorの返却値が成功のとき") {
                let articles = [
                    ArticleModel(id: .init(rawValue: "article_id"), title: "article_title", body: "article_body")
                ]
                
                beforeEach {
                    testScheduler.schedule(after: testScheduler.now.advanced(by: 10)) {
                        articleSearchInteractor.executeResult.send(articles)
                    }
                    
                    testScheduler.advance(by: 10)
                }
                
                it("articlesが更新される") {
                    expect(articlesOutputs) == [
                        [],
                        .init(articles)
                    ]
                }
            }
        }

        // ...
    }
}
```

### InteractorTests
[ArticleSearchInteractorTests.swift](https://github.com/hicka04/viper-combine-sample/blob/main/ViperCombineSampleTests/Domain/Usecases/ArticleSearch/ArticleSearchInteractorTests.swift)
```swift:ArticleSearchInteractorTests.swift
@testable import ViperCombineSample
import Foundation
import Quick
import Nimble
import Combine
import CombineSchedulers

final class ArticleSearchInteractorTests: QuickSpec {
    override func spec() {
        var cancellables: Set<AnyCancellable> = []
        var testScheduler: TestSchedulerOf<DispatchQueue>!
        var executeOutputs: [Result<ArticleSearchInteractor.Output, ArticleSearchInteractor.Failure>] = []
        
        var interactor: ArticleSearchInteractor!
        var qiitaDataStore: MockQiitaDataStore!
        
        let input: ArticleSearchInteractor.Input = "Swift"
        let error = NSError(domain: "hoge", code: -1, userInfo: nil)
        
        beforeEach {
            testScheduler = DispatchQueue.test
            
            qiitaDataStore = .init()
            interactor = .init(qiitaRepository: qiitaDataStore)
        }
        
        afterEach {
            cancellables = []
            executeOutputs = []
        }
        
        describe("execute") {
            beforeEach {
                testScheduler.schedule {
                    interactor.execute(input)
                        .convertToResultPublisher()
                        .sink { executeOutputs.append($0) }
                        .store(in: &cancellables)
                }
            }

            context("dataStoreの返却値がconnectionErrorのとき") {
                let connectionError: QiitaRepositoryError = .connectionError(error)
                
                beforeEach {
                    testScheduler.schedule(after: testScheduler.now.advanced(by: 10)) {
                        qiitaDataStore.searchArticlesResult.send(completion: .failure(connectionError))
                    }
                    
                    testScheduler.advance(by: 10)
                }
                
                it("エラーが返却される") {
                    expect(executeOutputs) == [
                        .failure(.init(error: connectionError))
                    ]
                }
            }

            // ...
            
            context("dataStoreの返却値が成功のとき") {
                let response = [
                    ArticleModel(id: .init(rawValue: "article_id"), title: "article_title", body: "article_body")
                ]
                
                beforeEach {
                    testScheduler.schedule(after: testScheduler.now.advanced(by: 10)) {
                        qiitaDataStore.searchArticlesResult.send(response)
                    }
                    
                    testScheduler.advance(by: 10)
                }
                
                it("[ArticleModel]が返却される") {
                    expect(executeOutputs) == [
                        .success(response)
                    ]
                }
            }
        }
    }
}
```

# 今後の展望
## フルSwiftUI化
TBD

### 考慮したいこと
* 画面遷移周りをどうするか
    * SwiftUIの画面遷移は `NavigationLink` を使うため、現状のようにRouterがViewを参照して画面遷移を外から行うことは難しい
    * 現在RouterはPresenterが参照しているが、Viewが直接参照する方法を検討中
        * Routerに｢遷移先のenumを受け取って、各caseに対応するViewを返却するメソッド｣を実装すれば、受け取ったViewをもとに `NavigationLink` を生成できるのではないか?
    * [こちらの記事](https://blog.personal-factory.com/2020/05/31/viper-architecture-in-swiftui/)ではPresenterに `NavigationLink` を作るメソッドを作成する方法を取っている
* `@Binding` によってデータの変更が伝えられるUIコンポーネントを使う場合にPresenterへのイベント通知方法をどうするか
    * ex `TextField` など
    * 今回は `PassthroughSubject` を作ってPresenterへのイベント通知を1本化したが、ここもまとめられるか?

## Swift Concurrencyとの共存/置き換え等
TBD

### 考慮したいこと
* どこをCombineで書くか、どこをConcurrencyで書くか、すべて置き換えるか等の線引き
    * ViewにデータをBindするところはCombineのほうが良さそう? (SwiftUIへの移行も考えると)
    * それ以外はConcurrencyでも書けそうだが…
* Combineのオペレーターからasyncなメソッド呼べるのか…? Taskからしか呼べないならどっちかに全振りしかできない…?
    * [こちらの記事](https://www.swiftbysundell.com/articles/calling-async-functions-within-a-combine-pipeline/)が参考になりそう
        * Futureを使っているためAsyncSequenceのように複数回結果を流すものは対応できないかも? 要調査

# まとめ
* Combineを使っていないコードからの移行のしやすさ、SwiftUIへの移行のしやすさを考慮して実装してみた
* 宣言的に書けるようになったため、状態とUIを同期させやすい
    * initやviewDidLoadのコード量が多くなるため、doを使った一時的なスコープを作るなど可読性を上げる工夫を検討する必要が出てくるかも
* combine-schedulersを利用してテストコードも実装できた