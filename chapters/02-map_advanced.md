# 第2章：地図アプリの応用

> 執筆者：田中 吾錬
> 最終更新：2026-06-05

## この章で学ぶこと



## 模範コードの全体像

```swift
// ============================================
// 第2章（応用）：現在地を表示し、周辺検索する地図アプリ
// ============================================
// ユーザーの現在地を取得して地図上に表示し、
// 周辺のコンビニやカフェなどを検索する機能を追加します。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSLocationWhenInUseUsageDescription
//     値: "現在地を地図に表示するために位置情報を使用します"
// ============================================

import SwiftUI
import MapKit

// MARK: - 位置情報マネージャー

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    let manager = CLLocationManager()
    var userLocation: CLLocationCoordinate2D?
    var authorizationStatus: CLAuthorizationStatus = .notDetermined

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
    }

    func requestPermission() {
        manager.requestWhenInUseAuthorization()
    }

    func startUpdating() {
        manager.startUpdatingLocation()
    }

    func stopUpdating() {
        manager.stopUpdatingLocation()
    }

    // MARK: - CLLocationManagerDelegate

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        userLocation = locations.last?.coordinate
    }

    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        authorizationStatus = manager.authorizationStatus

        switch authorizationStatus {
        case .authorizedWhenInUse, .authorizedAlways:
            startUpdating()
        default:
            break
        }
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var locationManager = LocationManager()
    @State private var cameraPosition: MapCameraPosition = .automatic
    @State private var searchResults: [MKMapItem] = []
    @State private var selectedCategory: String = "コンビニ"
    @State private var hasInitialSearched = false

    let searchCategories = ["コンビニ", "カフェ", "レストラン", "駅"]

    // CLLocationCoordinate2D は Equatable に準拠していないため、
    // onChange で監視できる文字列キーを派生させる
    private var userLocationKey: String? {
        locationManager.userLocation.map { "\($0.latitude),\($0.longitude)" }
    }

    var body: some View {
        ZStack(alignment: .top) {
            Map(position: $cameraPosition) {
                // 現在地のマーカー
                UserAnnotation()

                // 検索結果のマーカー
                ForEach(searchResults, id: \.self) { item in
                    if let name = item.name {
                        Marker(name, coordinate: item.placemark.coordinate)
                            .tint(.orange)
                    }
                }
            }
            .mapControls {
                MapUserLocationButton()
                MapCompass()
                MapScaleView()
            }

            // 検索カテゴリボタン
            VStack {
                categoryButtons
                    .padding(.top, 8)
                Spacer()
            }
        }
        .onAppear {
            locationManager.requestPermission()
        }
        .onChange(of: userLocationKey) { _, _ in
            guard let location = locationManager.userLocation else { return }
            cameraPosition = .region(
                MKCoordinateRegion(
                    center: location,
                    span: MKCoordinateSpan(latitudeDelta: 0.01, longitudeDelta: 0.01)
                )
            )
            // 初回の位置取得時に、選択中カテゴリで自動的に周辺検索を行う
            if !hasInitialSearched {
                hasInitialSearched = true
                Task { await searchNearby(query: selectedCategory) }
            }
        }
    }

    // MARK: - カテゴリボタン

    private var categoryButtons: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            HStack(spacing: 8) {
                ForEach(searchCategories, id: \.self) { category in
                    Button {
                        selectedCategory = category
                        Task { await searchNearby(query: category) }
                    } label: {
                        Text(category)
                            .font(.subheadline)
                            .padding(.horizontal, 14)
                            .padding(.vertical, 8)
                            .background(
                                selectedCategory == category
                                    ? Color.blue
                                    : Color(.systemBackground)
                            )
                            .foregroundStyle(
                                selectedCategory == category
                                    ? .white
                                    : .primary
                            )
                            .clipShape(Capsule())
                            .shadow(color: .black.opacity(0.1), radius: 2)
                    }
                }
            }
            .padding(.horizontal)
        }
    }

    // MARK: - 周辺検索

    func searchNearby(query: String) async {
        guard let userLocation = locationManager.userLocation else { return }

        let request = MKLocalSearch.Request()
        request.naturalLanguageQuery = query
        request.region = MKCoordinateRegion(
            center: userLocation,
            span: MKCoordinateSpan(latitudeDelta: 0.02, longitudeDelta: 0.02)
        )

        do {
            let search = MKLocalSearch(request: request)
            let response = try await search.start()
            searchResults = response.mapItems
        } catch {
            print("検索エラー: \(error.localizedDescription)")
            searchResults = []
        }
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

<img width="180" height="393" alt="Simulator Screenshot - iPhone 17 Pro - 2026-07-20 at 23 00 24" src="https://github.com/user-attachments/assets/003db1bc-47d5-4314-9309-29528a270b3b" />

<img width="180" height="393" alt="Simulator Screenshot - iPhone 17 Pro - 2026-07-20 at 23 00 27" src="https://github.com/user-attachments/assets/38c8e859-b38b-425a-af89-7467726e6bb8" />

<img width="180" height="393" alt="Simulator Screenshot - iPhone 17 Pro - 2026-07-20 at 23 00 30" src="https://github.com/user-attachments/assets/dbef5829-7521-4b1f-a254-e8128c72ae9f" />

<img width="180" height="393" alt="Simulator Screenshot - iPhone 17 Pro - 2026-07-20 at 23 00 33" src="https://github.com/user-attachments/assets/c831a0e9-6477-4960-b2c3-e98799573be4" />

カテゴリーのボタンを選択し現在位置周辺の建物を検索することができる


## コードの詳細解説

### 位置情報マネジャー

```swift
// MARK: - 位置情報マネージャー

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    let manager = CLLocationManager()
    var userLocation: CLLocationCoordinate2D?
    var authorizationStatus: CLAuthorizationStatus = .notDetermined

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
    }

    func requestPermission() {
        manager.requestWhenInUseAuthorization()
    }

    func startUpdating() {
        manager.startUpdatingLocation()
    }

    func stopUpdating() {
        manager.stopUpdatingLocation()
    }
}
```

**何をしているか：**

端末の位置情報を管理するLocationManagerクラスを作成している。

userLocationには取得した現在地の緯度と経度を保存し、authorizationStatusには位置情報の使用許可状態を保存する。

また、位置情報の許可を求める処理、取得を開始する処理、停止する処理をそれぞれ関数として定義している。

**なぜこう書くのか：**

位置情報に関する処理をContentViewとは別のクラスにまとめることで、地図の画面表示と位置情報の取得処理を分けて管理できる。
@Observableを付けることで、userLocationなどの値が変化した際に、SwiftUI側へ変更を反映できる。
また、CLLocationManagerDelegateを使用することで、位置情報の更新や許可状態の変化を受け取れる

**もしこう書かなかったら：**

```swift
manager.delegate = self
```

を書かなかった場合、位置情報が更新されても、更新結果を受け取るための処理が呼ばれない。
また、位置情報の処理をすべてContentViewに書くと、画面表示のコードと位置情報のコードが混ざり、管理しにくくなる。

---

### 現在地と許可状態の更新

```swift

// MARK: - CLLocationManagerDelegate
   
func locationManager(
    _ manager: CLLocationManager,
    didUpdateLocations locations: [CLLocation]
) {
    userLocation = locations.last?.coordinate
}

func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
    authorizationStatus = manager.authorizationStatus

    switch authorizationStatus {
    case .authorizedWhenInUse, .authorizedAlways:
        startUpdating()
    default:
        break
    }
}
```

**何をしているか：**

位置情報が更新されたときに、最新の緯度と経度をuserLocationへ保存している。
また、位置情報の使用許可状態が変化した際に、現在の許可状態を確認している。
ユーザーが位置情報の使用を許可した場合は、現在地の取得を開始する。

**なぜこう書くのか：**

didUpdateLocationsには複数の位置情報が渡される場合があるため、locations.lastを使って最新の位置情報を取得している。
また、ユーザーが位置情報を許可するか拒否するかは事前に分からないため、許可された場合のみ位置情報の取得を開始する必要がある。

**もしこう書かなかったら：**

取得した位置情報をuserLocationへ保存しなければ、地図画面から現在地を使用できない。
また、許可状態を確認せずに位置情報を取得しようとしても、ユーザーが許可していない場合は現在地を取得できない。
次のように強制アンラップすると、位置情報が存在しない場合にクラッシュする可能性がある。

```swift
userLocation = locations.last!.coordinate
```

そのため、?.を使って安全に取得している。

---

### 状態管理と現在地の監視

```swift
// MARK: - メインビュー

@State private var locationManager = LocationManager()
@State private var cameraPosition: MapCameraPosition = .automatic
@State private var searchResults: [MKMapItem] = []
@State private var selectedCategory: String = "コンビニ"
@State private var hasInitialSearched = false

let searchCategories = ["コンビニ", "カフェ", "レストラン", "駅"]

private var userLocationKey: String? {
    locationManager.userLocation.map {
        "\($0.latitude),\($0.longitude)"
    }
}
```

**何をしているか：**

地図画面で使用する情報を@Stateで管理している。

* locationManager：現在地の取得
* cameraPosition：地図の表示位置
* searchResults：周辺検索の結果
* selectedCategory：現在選択しているカテゴリ
* hasInitialSearched：初回検索を実行したかどうか

また、現在地の緯度と経度を文字列に変換し、onChangeで監視できる形にしている。

**なぜこう書くのか：**

これらの値は、位置情報の取得やボタン操作によって変化する。
そのため、@Stateで管理することで、値が変わったときに画面を自動で更新できる。
CLLocationCoordinate2DはそのままではEquatableに準拠していないため、onChangeで監視できる文字列に変換している。

**もしこう書かなかったら：**

hasInitialSearchedがない場合、位置情報が更新されるたびに周辺検索が実行されてしまう。
また、次のようにCLLocationCoordinate2Dを直接監視しようとすると、比較できない型であるためエラーになる場合がある。
```swift
.onChange(of: locationManager.userLocation) {
}
```
そのため、緯度と経度から比較可能な文字列を作成している。

---

### 地図とマーカーの表示

```swift
Map(position: $cameraPosition) {
    UserAnnotation()

    ForEach(searchResults, id: \.self) { item in
        if let name = item.name {
            Marker(
                name,
                coordinate: item.placemark.coordinate
            )
            .tint(.orange)
        }
    }
}
.mapControls {
    MapUserLocationButton()
    MapCompass()
    MapScaleView()
}
```

**何をしているか：**

Mapを使って地図を表示している。
UserAnnotation()では、ユーザーの現在地を地図上に表示する。
ForEachでは、周辺検索で取得した施設を1件ずつマーカーとして表示している。
また、現在地ボタン、コンパス、縮尺表示などの地図操作用コントロールを追加している。

**なぜこう書くのか：**

検索結果は複数件存在するため、ForEachを使ってすべての施設を地図上へ表示している。
item.nameは値が存在しない可能性があるため、if letで名前を取得できた施設だけマーカーを表示している。
mapControlsを使用することで、自分で地図操作用のボタンを作らずに標準機能を追加できる。

**もしこう書かなかったら：**

UserAnnotation()がなければ、現在地を取得できていても、地図上に現在地が表示されない。
また、検索結果をForEachで表示しなければ、検索に成功しても施設の場所を確認できない。
次のように施設名を強制アンラップすると、名前が存在しない場合にクラッシュする可能性がある。

```swift
Marker(
    item.name!,
    coordinate: item.placemark.coordinate
)
```

---

### 現在地取得後の地図移動と初回検索

```swift
.onAppear {
    locationManager.requestPermission()
}
.onChange(of: userLocationKey) { _, _ in
    guard let location = locationManager.userLocation else {
        return
    }

    cameraPosition = .region(
        MKCoordinateRegion(
            center: location,
            span: MKCoordinateSpan(
                latitudeDelta: 0.01,
                longitudeDelta: 0.01
            )
        )
    )

    if !hasInitialSearched {
        hasInitialSearched = true
        Task {
            await searchNearby(query: selectedCategory)
        }
    }
}
```

**何をしているか

画面が表示された際に、位置情報の使用許可を求めている。
現在地を取得した後は、地図の中心を現在地へ移動する。
また、最初に現在地を取得したときだけ、初期カテゴリである「コンビニ」の周辺検索を自動で実行する。

**なぜこう書くのか：**

周辺検索には現在地が必要なため、画面表示直後ではなく、現在地を取得できたタイミングで検索している。
cameraPositionを変更することで、取得した現在地の周辺を地図の中心に表示できる。
hasInitialSearchedを使うことで、位置情報が更新されるたびに自動検索されることを防いでいる。

**もしこう書かなかったら：**

位置情報の許可を求めなければ、現在地を取得できない。
また、現在地を取得してもcameraPositionを変更しなければ、現在地が画面外に表示される可能性がある。
hasInitialSearchedによる判定がない場合、端末の位置情報が更新されるたびに、同じカテゴリの検索が繰り返されてしまう。

---

### カテゴリ選択と周辺検索

```swift
ForEach(searchCategories, id: \.self) { category in
    Button {
        selectedCategory = category

        Task {
            await searchNearby(query: category)
        }
    } label: {
        Text(category)
            .background(
                selectedCategory == category
                    ? Color.blue
                    : Color(.systemBackground)
            )
            .foregroundStyle(
                selectedCategory == category
                    ? .white
                    : .primary
            )
    }
}

func searchNearby(query: String) async {
    guard let userLocation = locationManager.userLocation else {
        return
    }

    let request = MKLocalSearch.Request()
    request.naturalLanguageQuery = query
    request.region = MKCoordinateRegion(
        center: userLocation,
        span: MKCoordinateSpan(
            latitudeDelta: 0.02,
            longitudeDelta: 0.02
        )
    )

    do {
        let search = MKLocalSearch(request: request)
        let response = try await search.start()
        searchResults = response.mapItems
    } catch {
        print("検索エラー: \(error.localizedDescription)")
        searchResults = []
    }
}
```

**何をしているか：**

カテゴリ配列をもとに、コンビニ、カフェ、レストラン、駅のボタンを作成している。
ボタンを押すと、選択中のカテゴリを変更し、そのカテゴリを検索キーワードとして周辺検索を実行する。
検索に成功した場合は、取得した施設をsearchResultsへ保存する。
検索に失敗した場合は、エラー内容を表示し、検索結果を空にする。

**なぜこう書くのか：**

カテゴリごとにボタンを個別で作成するのではなく、配列とForEachを使用することで、同じコードを繰り返さずに済む。
selectedCategoryを基準にすることで、検索するカテゴリとボタンの見た目を連動できる。
また、request.regionへ現在地周辺の範囲を設定することで、遠くの施設ではなく、ユーザーの周辺にある施設を検索できる。
検索処理は完了まで時間がかかるため、asyncとawaitを使用している。

**もしこう書かなかったら：**

カテゴリごとに次のようなボタンを個別に書く必要がある。
```swift
Button("コンビニ") {
    Task {
        await searchNearby(query: "コンビニ")
    }
}

Button("カフェ") {
    Task {
        await searchNearby(query: "カフェ")
    }
}
```

カテゴリが増えるほどコード量も増え、修正しにくくなる。
また、検索範囲を設定しなければ、現在地周辺ではない施設が検索結果に含まれる可能性がある。
do-catchがなければ、通信状況などによって検索に失敗した際の処理を行えない。


（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Map` | SwiftUIで地図を表示するビューコンポーネント | `Map(position: .constant(.region(region)))` |
| 例：`Marker` | 地図上に位置をマーキングするコンポーネント | `Marker("名前", coordinate: coordinate)` |
| | | |
| | | |
| | | |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：
- 結果：
- わかったこと：

**実験2：**
- やったこと：
- 結果：
- わかったこと：

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   **得られた理解：**

2. **質問：**
   **得られた理解：**

3. **質問：**
   **得られた理解：**

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
