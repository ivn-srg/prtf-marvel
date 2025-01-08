<img src="https://github.com/ivn-srg/prtf-marvel/blob/main/appIcon.jpg" alt="Лого" style="width: 50px; height: 50px;"/>

# Marvel Heroes iOS App

## Описание проекта

Проект представляет собой мобильное приложения для просмотра героев киновселенной Marvel. Основные функциональности включают в себя: просмотр всех имеющихся персонажей Марвел, а также детализированной информации по ним через открытое API Marvel.

## Галерея
<img src="https://github.com/ivn-srg/prtf-marvel/blob/main/Simulator%20Screen%20Recording%20-%20iPhone%2016%20Pro%20-%202025-01-08%20at%2022.32.21.gif" alt="main screen" width="250">

## Моя роль в проекте

Данный проект является индивидуальным, соответственно, все стадии разработки, начиная от проектирования, создания макетов и заканчивая сетевым менеджером, разрабатывались мною лично.

## Технологии, используемые в проекте

![Realm](https://img.shields.io/badge/Realm-5B8C4A?style=for-the-badge&logo=realm&logoColor=white)
![Swift](https://img.shields.io/badge/Swift-F05138?style=for-the-badge&logo=swift&logoColor=white)
![Apple UIKit](https://img.shields.io/badge/UIKit-007AFF?style=for-the-badge&logo=apple&logoColor=white)
![SnapKit](https://img.shields.io/badge/SnapKit-FF4B30?style=for-the-badge&logo=swift&logoColor=white)
![Async/Await](https://img.shields.io/badge/Async%20Await-4BCB1A?style=for-the-badge&logo=swift&logoColor=white)
![MVVM](https://img.shields.io/badge/MVVM-FF3D00?style=for-the-badge&logo=swift&logoColor=white)

## Архитектура проекта

Применены архитектурные паттерны Clean Architecture, MVVM, ориентированные на разделение проекта по функциональным блокам для улучшения масштабируемости и удобства поддержки и тестирования.

<img src="https://github.com/ivn-srg/prtf-marvel/blob/main/Снимок%20экрана%202024-11-05%20в%2019.01.01.png" alt="xcode code structure screen" width="250">


## Принципы и инструменты разработки
- Система контроля версий: Git
- IDE: Xcode

## Код: Предпросмотр

В данном разделе представлены отрывки кода, демонстрирующие ключевые моменты и методы, использованные в проекте. Эти фрагменты кода отражают стиль программирования, архитектурные решения и технологические практики, применённые в ходе работы.

<details>
  <summary>Сетевой менеджер</summary>
  
  ```swift
  import Foundation
  import CryptoKit
  import RealmSwift
  import UIKit
  
  protocol ApiServiceProtocol: AnyObject {
      func performRequest<T: Decodable>(
          from urlString: String,
          modelType: T.Type
      ) async throws -> T
      
      func makeHTTPRequest<T: Decodable>(
          for request: URLRequest,
          codableModelType: T.Type
      ) async throws -> T
      
      func getImage(url: String) async throws -> UIImage
      
      func urlString(endpoint: APIType, offset: Int?, entityId: Int?, finalURL: String?) throws -> String
  }
  
  enum HTTPMethod: String {
      case get = "GET"
      case post = "POST"
      case patch = "PATCH"
      case delete = "DELETE"
  }
  
  enum APIType {
      case getHeroes, getHero, getComics, getOneComics, getSeries, getOneSeries,
           getStories, getStory, getCreators, getCreator, getEvents, getEvent,
           finalURL, clearlyURL
      
      private var baseURL: String {
          "https://gateway.marvel.com/v1/public/"
      }
      
      private var path: String {
          switch self {
          case .getHeroes, .getHero: "characters"
          case .getComics, .getOneComics: "comics"
          case .getSeries, .getOneSeries: "series"
          case .getStories, .getStory: "stories"
          case .getCreators, .getCreator: "creators"
          case .getEvents, .getEvent: "events"
          case .finalURL, .clearlyURL: ""
          }
      }
      
      var request: String {
          "\(baseURL)\(path)"
      }
  }
  
  enum HeroError: Error, LocalizedError {
      case invalidURL, parsingError(Error), serializationError(Error), noInternetConnection, timeout,
           otherNetworkError(Error), notFoundEntity, cashingFailed(Error), unexpectedData, parsingFailureModelError(Error)
  }
  
  final class ApiServiceConfiguration {
      public static let shared = ApiServiceConfiguration()
  
      private init() {}
  
      var apiService: ApiServiceProtocol {
          if shouldUseMockingService {
              return APIMockManager.shared
          } else {
              return APIManager.shared
          }
      }
  
      private var shouldUseMockingService: Bool = false
  
      func setMockingServiceEnabled() {
          shouldUseMockingService = true
      }
  }
  
  final class APIManager: ApiServiceProtocol {
      public static let shared = APIManager()
      
      private var currentTimeStamp: Int {
          Int(Date().timeIntervalSince1970)
      }
      private var md5Hash: String {
          MD5(string: "\(currentTimeStamp)\(PRIVATE_KEY)\(API_KEY)")
      }
      
      func urlString(endpoint: APIType, offset: Int? = nil, entityId: Int? = nil, finalURL: String? = nil) throws -> String {
          let limit = 30
          
          switch endpoint {
          case .getHeroes, .getComics, .getCreators, .getEvents, .getSeries, .getStories:
              guard let offset = offset else { throw HeroError.invalidURL }
              return "\(endpoint.request)?limit=\(limit)&offset=\(offset)&ts=\(currentTimeStamp)&apikey=\(API_KEY)&hash=\(md5Hash)"
          case .getHero, .getEvent, .getStory, .getOneComics, .getOneSeries, .getCreator:
              guard let entityId = entityId else { throw HeroError.invalidURL }
              return "\(endpoint.request)/\(entityId)?ts=\(currentTimeStamp)&apikey=\(API_KEY)&hash=\(md5Hash)"
          case .finalURL:
              guard let finalURL = finalURL else { throw HeroError.invalidURL }
              return "\(finalURL)?ts=\(currentTimeStamp)&apikey=\(API_KEY)&hash=\(md5Hash)"
          case .clearlyURL:
              guard let finalURL = finalURL else { throw HeroError.invalidURL }
              return finalURL
          }
      }
      
      func performRequest<T: Decodable>(
          from urlString: String,
          modelType: T.Type
      ) async throws -> T {
          guard let url = URL(string: urlString) else { throw HeroError.invalidURL }
          
          var request = URLRequest(url: url)
          request.httpMethod = HTTPMethod.get.rawValue
          request.addValue("application/json", forHTTPHeaderField: "accept")
          
          return try await makeHTTPRequest(for: request, codableModelType: modelType)
      }
      
      func makeHTTPRequest<T: Decodable>(
          for request: URLRequest,
          codableModelType: T.Type
      ) async throws -> T {
          do {
              let (data, _) = try await URLSession.shared.data(for: request)
              guard data != notFoundEntityResponseData else { throw HeroError.notFoundEntity }
              do {
                  let result = try JSONDecoder().decode(codableModelType, from: data)
                  return result
              } catch {
                  print("data = \(String(decoding: data, as: UTF8.self))")
                  let errorModel = try JSONDecoder().decode(ResponseFailureModel.self, from: data)
                  let errorMessage = StringError(errorModel.status)
                  throw HeroError.parsingFailureModelError(errorMessage)
              }
          } catch let error as DecodingError {
              print("request \(request)")
              throw HeroError.parsingError(error)
          } catch let error as URLError {
              print("request \(request)")
              switch error.code {
              case .notConnectedToInternet:
                  throw HeroError.noInternetConnection
              case .timedOut:
                  throw HeroError.timeout
              default:
                  throw HeroError.otherNetworkError(error)
              }
          } catch {
              print("request1 \(request)")
              guard let error = error as? HeroError else { throw HeroError.otherNetworkError(error) }
              throw error
          }
      }
      
      // MARK: - getting Image funcs
      func getImage(url: String) async throws -> UIImage {
          if url == "entity." || url == "entity" {
              return emptyEntityImage
          } else if url == "hero." || url == "hero" || url == "\(imageNotAvailable).jpg" {
              return MockUpImage
          }
          
          if let cachedImage = await RealmManager.shared.fetchCachedImage(url: url),
              let imageData = cachedImage.imageData,
              let image = UIImage(data: imageData) {
              return image
          }
          
          // if image isn't cached
          return try await getImageForHeroFromNet(url: url)
      }
      
      private func getImageForHeroFromNet(url: String) async throws -> UIImage {
          guard let url = URL(string: url) else { throw HeroError.invalidURL }
          
          var request = URLRequest(url: url)
          request.httpMethod = "GET"
          request.allHTTPHeaderFields = ["Accept": "application/json,image/png,image/jpeg,image/gif"]
          
          let (data, _) = try await URLSession.shared.data(for: request)
          
          if let uiImage = UIImage(data: data) {
              let cachedImageData = CachedImageData(
                  url: url.absoluteString,
                  imageData: data
              )
              
              do {
                  try await MainActor.run {
                      let realm = try Realm()
                      
                      try realm.write {
                          realm.add(cachedImageData, update: .modified)
                      }
                  }
              } catch {
                  throw HeroError.cashingFailed("Error saving image to Realm cache: \(error)".errorString)
              }
              return uiImage
          } else {
              print("Error loading image: \(url)")
              return emptyEntityImage
          }
      }
      
      // MARK: - private utility func
      private func MD5(string: String) -> String {
          let digest = Insecure.MD5.hash(data: string.data(using: .utf8) ?? Data())
          return digest.map {
              String(format: "%02hhx", $0)
          }.joined()
      }
  }
  ```
</details>

<details>
  <summary>Экран со списком героев</summary>

  ```swift
  import UIKit
  import SnapKit
  
  final class HeroListViewController: UIViewController {
      
      // MARK: - Fields
      
      private let viewModel: HeroListViewModel
      
      private var itemW: CGFloat {
          screenWidth * 0.7
      }
      
      private var itemH: CGFloat {
          screenHeight * 0.6
      }
      
      // MARK: - UI components
      
      private lazy var box: UIView = {
          let vc = UIView()
          vc.translatesAutoresizingMaskIntoConstraints = false
          return vc
      }()
      
      private lazy var marvelLogo: UIImageView = {
          let logo = UIImageView()
          logo.translatesAutoresizingMaskIntoConstraints = false
          logo.contentMode = .scaleAspectFit
          return logo
      }()
      
      private lazy var chooseHeroText: UILabel = {
          let txt = UILabel()
          txt.translatesAutoresizingMaskIntoConstraints = false
          txt.font = UIFont(name: Font.InterBold, size: 28)
          txt.textAlignment = .center
          txt.numberOfLines = 2
          return txt
      }()
      
      private lazy var customLayout: CustomHeroItemLayer = {
          let lt = CustomHeroItemLayer()
          lt.itemSize.width = itemW
          lt.scrollDirection = .horizontal
          lt.minimumLineSpacing = itemW * 0.18
          return lt
      }()
      
      private lazy var collectionView: UICollectionView = {
          let collectionView = UICollectionView(frame: .zero, collectionViewLayout: customLayout)
          collectionView.translatesAutoresizingMaskIntoConstraints = false
          collectionView.contentInsetAdjustmentBehavior = .never
          collectionView.backgroundColor = .clear
          collectionView.register(HeroCollectionViewCell.self, forCellWithReuseIdentifier: HeroCollectionViewCell.identifier)
          collectionView.contentInset = UIEdgeInsets(top: 0, left: itemW * 0.25, bottom: 0, right: itemW * 0.25)
          collectionView.alwaysBounceHorizontal = true
          collectionView.showsHorizontalScrollIndicator = false
          collectionView.accessibilityIdentifier = "heroCollection"
          return collectionView
      }()
      
      private lazy var triangleView: TriangleView = {
          let tv = TriangleView(frame: view.bounds)
          tv.translatesAutoresizingMaskIntoConstraints = false
          tv.accessibilityIdentifier = "triangleView"
          return tv
      }()
      
      private lazy var panRecognize: UIPanGestureRecognizer = {
          let gestureRecognizer = UIPanGestureRecognizer()
          gestureRecognizer.addTarget(self, action: #selector(pull2refresh))
          return gestureRecognizer
      }()
      
      private let activityIndicator: UIActivityIndicatorView = {
          let activityIndicator = UIActivityIndicatorView(style: .medium)
          activityIndicator.hidesWhenStopped = true
          activityIndicator.color = UIColor.themeRed
          activityIndicator.translatesAutoresizingMaskIntoConstraints = false
          return activityIndicator
      }()
      
      // MARK: - lifecycle
      
      init(vm: HeroListViewModel) {
          self.viewModel = vm
          super.init(nibName: nil, bundle: nil)
      }
      
      required init?(coder: NSCoder) {
          fatalError("init(coder:) has not been implemented")
      }
      
      override func viewDidLoad() {
          super.viewDidLoad()
          
          setupUI()
          executeWithErrorHandling {
              try await self.viewModel.fetchHeroesData(into: self.collectionView)
          }
          
          collectionView.dataSource = self
          collectionView.delegate = self
      }
      
      // MARK: - UI functions
      
      private func setupUI() {
          
          box.backgroundColor = UIColor.bgColor
          
          navigationController?.setNavigationBarHidden(true, animated: false)
          
          marvelLogo.image = Logo
          
          chooseHeroText.text = mainScreenTitle
          
          view.addGestureRecognizer(panRecognize)
          
          view.addSubview(activityIndicator)
          activityIndicator.snp.makeConstraints {
              $0.top.equalTo(view.safeAreaLayoutGuide.snp.top).offset(view.frame.height * 0.05)
              $0.centerX.equalTo(view.safeAreaLayoutGuide.snp.centerX)
          }
          
          view.addSubview(box)
          box.snp.makeConstraints{
              $0.edges.equalTo(view.safeAreaLayoutGuide.snp.edges)
          }
          
          triangleView.backgroundColor = .clear
          box.addSubview(triangleView)
          triangleView.snp.makeConstraints{
              $0.height.equalToSuperview().multipliedBy(0.75)
              $0.bottom.horizontalEdges.equalToSuperview()
          }
          
          box.addSubview(marvelLogo)
          marvelLogo.snp.makeConstraints{
              $0.top.equalTo(box.snp.top).offset(20)
              $0.width.equalTo(box.snp.width).multipliedBy(0.4)
              $0.height.equalTo(box.snp.height).multipliedBy(0.09)
              $0.centerX.equalToSuperview()
          }
          
          box.addSubview(chooseHeroText)
          chooseHeroText.snp.makeConstraints{
              $0.top.equalTo(marvelLogo.snp.bottom).offset(20)
              $0.width.equalToSuperview()
          }
          
          box.addSubview(collectionView)
          collectionView.snp.makeConstraints{
              $0.top.equalTo(chooseHeroText.snp.bottom)
              $0.bottom.width.equalToSuperview()
          }
      }
      
      private func moveFocusOnItem() {
          let itemCount = collectionView.numberOfItems(inSection: 0)
          var (currentPage, _) = UDManager.shared.getCurrentPosition()
          currentPage = currentPage > itemCount ? itemCount - 1 : currentPage
          
          let indexPath = IndexPath(item: currentPage, section: 0)
          collectionView.scrollToItem(at: indexPath, at: .centeredHorizontally, animated: true)
          
          setupCell(at: currentPage)
      }
      
      @objc func pull2refresh(_ gesture: UIPanGestureRecognizer) {
          
          let translation = gesture.translation(in: box)
          let newY = max(translation.y, 0)
          let maxPullDownDistance = self.box.frame.height * 0.2
          
          if newY <= maxPullDownDistance {
              box.transform = CGAffineTransform(translationX: 0, y: newY)
          }
          
          if gesture.state == .began {
              activityIndicator.startAnimating()
          }
          
          if gesture.state == .ended {
              if newY > maxPullDownDistance {
                  executeWithErrorHandling {
                      try await self.viewModel.fetchHeroesData(into: self.collectionView, needRefresh: true)
                  }
              }
              UIView.animate(withDuration: 0.3) {
                  self.box.transform = CGAffineTransform.identity
                  self.activityIndicator.stopAnimating()
              }
          }
      }
      
      private func updateTriangleViewColor(didLoadImage: UIImage?) {
          guard let image = didLoadImage else { return }
          let averageColor = image.averageColor()
          triangleView.accessibilityLabel = averageColor.cgColor.toString()
          triangleView.updateTriangleColor(averageColor)
      }
  }
  
  // MARK: - Extensions
  
  extension HeroListViewController: UICollectionViewDelegate, UICollectionViewDataSource {
      func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
          viewModel.countOfRow()
      }
      
      func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
          guard let cell = collectionView.dequeueReusableCell(withReuseIdentifier: HeroCollectionViewCell.identifier, for: indexPath) as? HeroCollectionViewCell else { return UICollectionViewCell() }
          let hero = viewModel.dataSource[indexPath.row]
          
          cell.configure(viewModel: HeroCollectionViewCellViewModel(hero: HeroRO(heroData: hero)))
          
          moveFocusOnItem()
          
          return cell
      }
      
      func collectionView(_ collectionView: UICollectionView, didSelectItemAt indexPath: IndexPath) {
          let hero = viewModel.dataSource[indexPath.row]
          
          if indexPath.item == customLayout.currentPage {
              let vc = DetailHeroViewController(hero: HeroRO(heroData: hero))
              self.navigationController?.pushViewController(vc, animated: true)
          } else {
              collectionView.scrollToItem(at: indexPath, at: .centeredHorizontally, animated: true)
              
              customLayout.currentPage = indexPath.item
              customLayout.previousOffset = customLayout.updateOffset(collectionView)
              setupCell()
          }
      }
  }
  
  extension HeroListViewController: UICollectionViewDelegateFlowLayout {
      
      func collectionView(
          _ collectionView: UICollectionView,
          layout collectionViewLayout: UICollectionViewLayout,
          sizeForItemAt indexPath: IndexPath
      ) -> CGSize {
          CGSize(
              width: collectionView.frame.width * 0.7,
              height: collectionView.frame.height * 0.75
          )
      }
  }
  
  extension HeroListViewController {
      
      func scrollViewWillBeginDecelerating(_ scrollView: UIScrollView) {
          setupCell()
      }
      
      func collectionView(_ collectionView: UICollectionView, willDisplay cell: UICollectionViewCell, forItemAt indexPath: IndexPath) {
          let lastRow = indexPath.row
          if lastRow == viewModel.countOfRow() - 1 {
              let totalRows = collectionView.numberOfItems(inSection: indexPath.section)
              
              if lastRow >= totalRows - 1 {
                  if collectionView.contentOffset.x > 0 {
                      executeWithErrorHandling {
                          try await self.viewModel.fetchHeroesData(into: self.collectionView, needsLoadMore: true)
                      }
                  }
              }
          }
      }
      
      private func setupCell(at indexPathRow: Int? = nil) {
          let indexPath = IndexPath(item: indexPathRow == nil ? customLayout.currentPage : indexPathRow!, section: 0)
          guard let cell = collectionView.cellForItem(at: indexPath) as? HeroCollectionViewCell else { return }
  
          updateTriangleViewColor(didLoadImage: cell.heroImage)
          transformCell(cell)
      }
      
      private func transformCell(_ cell: UICollectionViewCell, isEffect: Bool = true) {
          if !isEffect {
              cell.transform = CGAffineTransform(scaleX: 1.2, y: 1.2)
              return
          }
          
          UIView.animate(withDuration: 0.2) {
              cell.transform = CGAffineTransform(scaleX: 1.2, y: 1.2)
          }
          
          for otherCell in collectionView.visibleCells {
              if let indexPath = collectionView.indexPath(for: otherCell) {
                  if indexPath.item != customLayout.currentPage {
                      UIView.animate(withDuration: 0.2) {
                          otherCell.transform = .identity
                      }
                  }
              }
          }
      }
  }
  ```
</details>

# Основные достижения и результаты

- Успешная реализация приложения для просмотра списка героев Марвел.
- Реализация юнит и UI тестов в проекте при помощи библиотеки XCTest.
- Оптимизация производительности UI части, гарантирующая понятную и отзывчивую работу приложения.
- Реализация кеширования данных приложения на основе Realm.
- Создание детализировнаного экрана с персонажем с информацией по нему.

## Требования

- iOS 16 и выше.

## Связь

[Telegram](https://t.me/ivn_srg)

## Автор

Иванов Сергей 2024.

## Ссылки
Код проекта находится [здесь](https://github.com/ivn-srg/MarvelHeroesiOSApp/tree/develop)
