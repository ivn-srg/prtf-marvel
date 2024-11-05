<img src="https://github.com/ivn-srg/prtf-marvel/blob/main/appIcon.jpg" alt="Лого" style="width: 50px; height: 50px;"/>

# Marvel Heroes iOS App

## Описание проекта

Проект представляет собой мобильное приложения для просмотра героев киновселенной Marvel. Основные функциональности включают в себя: просмотр всех имеющихся персонажей Марвел, а также детализированной информации по ним через открытое API Marvel.

## Галерея
<img src="https://github.com/ivn-srg/prtf-marvel/blob/main/Simulator%20Screenshot%20-%20iPhone%2016%20Pro%20-%202024-11-05%20at%2018.56.12.gif" alt="main screen" width="250">

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
- Система контроля версий: Github
- IDE: Xcode

## Код: Предпросмотр

В данном разделе представлены отрывки кода, демонстрирующие ключевые моменты и методы, использованные в проекте. Эти фрагменты кода отражают стиль программирования, архитектурные решения и технологические практики, применённые в ходе работы.

<details>
  <summary>Сетевой менеджер</summary>
  
  ```swift
  import Foundation
  import Alamofire
  import CryptoKit
  import Kingfisher
  import RealmSwift
  import UIKit
  
  protocol ApiServiceProtocol: AnyObject {
      func fetchHeroesData(from offset: Int, completion: @escaping (Result<Heroes, Error>) -> Void)
      func fetchHeroData(heroItem: HeroRO, completion: @escaping (Result<HeroModel, Error>) -> Void)
      func getImageForHero(url: String, imageView: UIImageView)
  }
  
  enum APIType {
      
      case getHeroes
      case getHero
      
      private var baseURL: String {
          "https://gateway.marvel.com/v1/public/"
      }
      
      private var path: String {
          switch self {
          case .getHeroes: "characters"
          case .getHero: "characters"
          }
      }
      
      var request: String {
          "\(baseURL)\(path)"
      }
  }
  
  enum HeroError: Error, LocalizedError {
      
      case unknown
      case invalidURL
      case invalidUserData
      case custom(description: String)
      
      var errorDescription: String? {
          switch self {
          case .invalidUserData:
              return "This is an invalid data. Please try again."
          case .invalidURL:
              return "Invalid url, man"
          case .unknown:
              return "Hey, this is an unknown error!"
          case .custom(let description):
              return description
          }
      }
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
      
      func fetchHeroesData(from offset: Int, completion: @escaping (Result<Heroes, Error>) -> Void) {
          let limit = 30
          let path = "\(APIType.getHeroes.request)?limit=\(limit)&offset=\(offset)&ts=\(currentTimeStamp)&apikey=\(API_KEY)&hash=\(md5Hash)"
          let urlString = String(format: path)
          
          AF.request(urlString)
              .validate()
              .responseDecodable(of: ResponseModel.self, queue: .global(), decoder: JSONDecoder()) { (response) in
                  switch response.result {
                  case .success(let heroesData):
                      completion(.success(heroesData.data.results))
                      break
                  case .failure(let error):
                      
                      if let err = self.getHeroError(error: error, data: response.data) {
                          completion(.failure(err))
                      } else {
                          completion(.failure(error))
                      }
                      
                      break
                  }
              }
      }
      
      func fetchHeroData(heroItem: HeroRO, completion: @escaping (Result<HeroModel, Error>) -> Void) {
          let path = "\(APIType.getHero.request)/\(heroItem.id)?ts=\(currentTimeStamp)&apikey=\(API_KEY)&hash=\(md5Hash)"
          let urlString = String(format: path)
          
          AF.request(urlString)
              .validate()
              .responseDecodable(of: ResponseModel.self, queue: .global(), decoder: JSONDecoder()) { (response) in
                  switch response.result {
                  case .success(let heroesData):
                      let model = heroesData.data.results.first ?? mockUpHeroData
                      completion(.success(model))
                      break
                  case .failure(let error):
                      if let err = self.getHeroError(error: error, data: response.data) {
                          completion(.failure(err))
                      } else {
                          completion(.failure(error))
                      }
                      break
                  }
              }
      }
      
      @MainActor func getImageForHero(url: String, imageView: UIImageView) {
          do {
              let realm = try Realm()
              let cachedImage = realm.objects(CachedImageData.self).filter { $0.url == url }.first
              
              if let cachedImage = cachedImage, let imageData = cachedImage.imageData, let image = UIImage(data: imageData) {
                  imageView.image = image
                  return
              }
          } catch {
              print("Error saving image to Realm cache: \(error)")
              imageView.image = MockUpImage
          }
          
          // if image isn't cached
          getImageForHeroFromNet(url: url, imageView: imageView)
      }
      
      // MARK: - private func
      
      @MainActor private func getImageForHeroFromNet(url: String, imageView: UIImageView) {
          let url = URL(string: url)
          let processor = RoundCornerImageProcessor(cornerRadius: 20)
          let indicatorStyle = UIActivityIndicatorView.Style.large
          let indicator = UIActivityIndicatorView(style: indicatorStyle)
          
          indicator.autoresizingMask = [.flexibleWidth, .flexibleHeight]
          imageView.kf.indicatorType = .activity
          (imageView.kf.indicator?.view as? UIActivityIndicatorView)?.color = .white
          
          imageView.kf.setImage(with: url, options: [.processor(processor), .transition(.fade(0.2))]){ result in
              switch result {
              case .success(let imageResult):
                  imageView.image = imageResult.image
                  
                  let cachedImageData = CachedImageData(
                      url: url?.absoluteString ?? "",
                      imageData: imageResult.image.pngData()
                  )
                  
                  do {
                      let realm = try Realm()
                      
                      try realm.write {
                          realm.add(cachedImageData, update: .modified)
                      }
                      print("Downloaded and cached image for \(String(describing: url))")
                  } catch {
                      print("Error saving image to Realm cache: \(error)")
                  }
                  break
              case .failure(let error):
                  imageView.image = MockUpImage
                  print("Error loading image: \(error)")
                  
                  DispatchQueue.main.async {
                      LoadingIndicator.stopLoading()
                  }
                  break
              }
          }
      }
      
      private func getHeroError(error: AFError, data: Data?) -> Error? {
          if let data = data,
             let failure = try? JSONDecoder().decode(ResponseFailureModel.self, from: data) {
              let message = failure.message
              return HeroError.custom(description: message)
          } else {
              return nil
          }
      }
      
      private func MD5(string: String) -> String {
          let digest = Insecure.MD5.hash(data: string.data(using: .utf8) ?? Data())
          return digest.map {
              String(format: "%02hhx", $0)
          }.joined()
      }
  }
  
  final class APIMockManager: ApiServiceProtocol {
      
      public static let shared = APIMockManager()
      
      func fetchHeroesData(from offset: Int, completion: @escaping (Result<Heroes, any Error>) -> Void) {
          let response = [
              HeroModel(id: 1, name: "Deadpool", heroDescription: "This is the craziest hero in Marvel spacs!", thumbnail: ThumbnailModel(path: "Deadpool", extension: "")),
              HeroModel(id: 2, name: "Iron Man", heroDescription: "Robert is a clever guy", thumbnail: ThumbnailModel(path: "Iron Man", extension: ""))
          ]
          completion(.success(response))
      }
      
      func fetchHeroData(heroItem: HeroRO, completion: @escaping (Result<HeroModel, any Error>) -> Void) {
          let heroInfo = HeroModel(id: heroItem.id, name: heroItem.name, heroDescription: heroItem.heroDescription, thumbnail: ThumbnailModel(thumbRO: heroItem.thumbnail ?? ThumbnailRO()))
          completion(.success(heroInfo))
      }
      
      func getImageForHero(url: String, imageView: UIImageView) {
          if url == "Deadpool." {
              imageView.image = UIImage(named: "deadPool")
          } else if url == "Iron Man." {
              imageView.image = UIImage(named: "ironMan")
          } else {
              imageView.image = UIImage(named: "mockup")
          }
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
          txt.textColor = .white
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
          activityIndicator.color = loaderColor
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
          viewModel.fetchHeroesData(into: collectionView)
          
          self.collectionView.dataSource = self
          self.collectionView.delegate = self
      }
      
      // MARK: - UI functions
      
      private func setupUI() {
          
          box.backgroundColor = bgColor
          
          self.navigationController?.setNavigationBarHidden(true, animated: false)
          
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
              $0.verticalEdges.equalTo(self.view.safeAreaLayoutGuide.snp.verticalEdges)
              $0.horizontalEdges.equalToSuperview()
          }
          
          triangleView.backgroundColor = .clear
          box.addSubview(triangleView)
          triangleView.snp.makeConstraints{
              $0.top.equalTo(self.box.snp.top).offset(self.view.frame.height * 0.25)
              $0.bottom.horizontalEdges.equalToSuperview()
          }
          
          box.addSubview(marvelLogo)
          marvelLogo.snp.makeConstraints{
              $0.top.equalTo(self.box.snp.top).offset(20)
              $0.width.equalTo(self.box.snp.width).multipliedBy(0.4)
              $0.height.equalTo(self.box.snp.height).multipliedBy(0.09)
              $0.centerX.equalToSuperview()
          }
          
          box.addSubview(chooseHeroText)
          chooseHeroText.snp.makeConstraints{
              $0.top.equalTo(self.marvelLogo.snp.bottom).offset(20)
              $0.width.equalToSuperview()
          }
          
          box.addSubview(collectionView)
          collectionView.snp.makeConstraints{
              $0.top.equalTo(chooseHeroText.snp.bottom)
              $0.bottom.width.equalToSuperview()
          }
      }
      
      private func moveFocusOnFirstItem() {
          if customLayout.currentPage == 0 {
              let indexPath = IndexPath(item: 0, section: 0)
              collectionView.scrollToItem(at: indexPath, at: .centeredHorizontally, animated: true)
              
              setupCell()
          }
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
                  viewModel.fetchHeroesData(into: collectionView, needRefresh: true)
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
          
          moveFocusOnFirstItem()
          
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
      
      func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, sizeForItemAt indexPath: IndexPath) -> CGSize {
          
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
                      viewModel.fetchHeroesData(into: collectionView, needsLoadMore: true)
                  }
              }
          }
      }
      
      private func setupCell() {
          let indexPath = IndexPath(item: customLayout.currentPage, section: 0)
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
