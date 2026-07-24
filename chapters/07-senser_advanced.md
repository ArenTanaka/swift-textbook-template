# 第7章：センサーの活用

> 執筆者：（氏名）
> 最終更新：YYYY-MM-DD

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

例：この章では、iPhoneに搭載されている加速度・ジャイロなどのセンサーにアクセスして、デバイスの動きや姿勢を検出する方法を学ぶ。具体的にはCoreMotionを使った水平器アプリ、CMPedometerとCoreLocationを組み合わせた活動トラッカーを題材にして、センサーデータの読み取りと処理の実装を学ぶ。

## 模範コードの全体像

### 基本編

```swift
// ============================================
// 第7章（基本）：加速度センサーで動く水平器アプリ
// ============================================
// CoreMotionを使って端末の傾きをリアルタイムで取得し、
// 水平器（水準器）として表示するアプリです。
//
// 【注意】シミュレータではセンサーが使えません。
//         実機（iPhone / iPad）でテストしてください。
// ============================================

import SwiftUI
import CoreMotion

// MARK: - モーションマネージャー

@Observable
class MotionManager {
    private let motionManager = CMMotionManager()

    var pitch: Double = 0    // 前後の傾き
    var roll: Double = 0     // 左右の傾き
    var yaw: Double = 0      // 水平方向の回転
    var isAvailable: Bool

    init() {
        // 初回 body 評価時点で正しい値を返すよう、init で同期的にセット
        isAvailable = motionManager.isDeviceMotionAvailable
    }

    func startUpdates() {
        guard isAvailable else { return }

        motionManager.deviceMotionUpdateInterval = 1.0 / 60.0

        motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
            guard let self = self, let motion = motion else { return }

            self.pitch = motion.attitude.pitch
            self.roll = motion.attitude.roll
            self.yaw = motion.attitude.yaw
        }
    }

    func stopUpdates() {
        motionManager.stopDeviceMotionUpdates()
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var motionManager = MotionManager()

    var body: some View {
        NavigationStack {
            if motionManager.isAvailable {
                VStack(spacing: 30) {
                    // 水平器の円
                    LevelIndicator(
                        pitch: motionManager.pitch,
                        roll: motionManager.roll
                    )

                    // 数値表示
                    DataDisplay(
                        pitch: motionManager.pitch,
                        roll: motionManager.roll,
                        yaw: motionManager.yaw
                    )
                }
                .padding()
                .navigationTitle("水平器")
            } else {
                ContentUnavailableView(
                    "センサーが利用できません",
                    systemImage: "iphone.slash",
                    description: Text("このアプリは実機（iPhone）で動作します。\nシミュレータではセンサーが使えません。")
                )
            }
        }
        .onAppear {
            motionManager.startUpdates()
        }
        .onDisappear {
            motionManager.stopUpdates()
        }
    }
}

// MARK: - 水平器インジケーター

struct LevelIndicator: View {
    let pitch: Double
    let roll: Double

    private let maxOffset: CGFloat = 100

    private var xOffset: CGFloat {
        CGFloat(roll) * maxOffset
    }

    private var yOffset: CGFloat {
        CGFloat(pitch) * maxOffset
    }

    private var isLevel: Bool {
        abs(pitch) < 0.03 && abs(roll) < 0.03
    }

    var body: some View {
        ZStack {
            // 外側の円
            Circle()
                .stroke(.gray.opacity(0.3), lineWidth: 2)
                .frame(width: 250, height: 250)

            // 中心の十字線
            Path { path in
                path.move(to: CGPoint(x: 125, y: 0))
                path.addLine(to: CGPoint(x: 125, y: 250))
                path.move(to: CGPoint(x: 0, y: 125))
                path.addLine(to: CGPoint(x: 250, y: 125))
            }
            .stroke(.gray.opacity(0.2), lineWidth: 1)
            .frame(width: 250, height: 250)

            // 中間の円
            Circle()
                .stroke(.gray.opacity(0.2), lineWidth: 1)
                .frame(width: 125, height: 125)

            // バブル（傾きに応じて移動）
            Circle()
                .fill(isLevel ? .green : .red)
                .frame(width: 40, height: 40)
                .opacity(0.8)
                .shadow(color: isLevel ? .green : .red, radius: 8)
                .offset(
                    x: max(-maxOffset, min(maxOffset, xOffset)),
                    y: max(-maxOffset, min(maxOffset, yOffset))
                )
                .animation(.spring(duration: 0.1), value: xOffset)
                .animation(.spring(duration: 0.1), value: yOffset)

            // 水平時の表示
            if isLevel {
                Text("水平!")
                    .font(.headline)
                    .foregroundStyle(.green)
                    .offset(y: 140)
            }
        }
    }
}

// MARK: - 数値データ表示

struct DataDisplay: View {
    let pitch: Double
    let roll: Double
    let yaw: Double

    var body: some View {
        VStack(spacing: 12) {
            DataRow(
                label: "前後の傾き（Pitch）",
                value: pitch,
                icon: "arrow.up.and.down"
            )
            DataRow(
                label: "左右の傾き（Roll）",
                value: roll,
                icon: "arrow.left.and.right"
            )
            DataRow(
                label: "水平回転（Yaw）",
                value: yaw,
                icon: "arrow.triangle.2.circlepath"
            )
        }
        .padding()
        .background(
            RoundedRectangle(cornerRadius: 12)
                .fill(.gray.opacity(0.05))
        )
    }
}

struct DataRow: View {
    let label: String
    let value: Double
    let icon: String

    var body: some View {
        HStack {
            Image(systemName: icon)
                .frame(width: 30)
                .foregroundStyle(.blue)

            Text(label)
                .font(.caption)

            Spacer()

            Text(String(format: "%.3f rad", value))
                .font(.system(.caption, design: .monospaced))
                .foregroundStyle(.secondary)

            Text(String(format: "(%.1f°)", value * 180 / .pi))
                .font(.system(.caption, design: .monospaced))
                .foregroundStyle(.secondary)
                .frame(width: 60, alignment: .trailing)
        }
    }
}

#Preview {
    ContentView()
}
```

### 応用編
```swift
// ============================================
// 第7章（応用）：歩数計・移動距離トラッカー
// ============================================
// CoreMotion（歩数計）とCoreLocation（移動距離）を
// 組み合わせて、今日の活動を記録するアプリです。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSMotionUsageDescription
//     値: "歩数を計測するためにモーションセンサーを使用します"
//   - NSLocationWhenInUseUsageDescription
//     値: "移動距離を計測するために位置情報を使用します"
// ============================================

import SwiftUI
import CoreMotion
import CoreLocation

// MARK: - 活動トラッカー

@Observable
class ActivityTracker: NSObject, CLLocationManagerDelegate {
    // 歩数関連
    private let pedometer = CMPedometer()
    var stepCount: Int = 0
    var distance: Double = 0     // メートル
    var isPedometerAvailable: Bool = false

    // 位置関連
    private let locationManager = CLLocationManager()
    var currentSpeed: Double = 0  // m/s
    var locations: [CLLocationCoordinate2D] = []

    // 状態
    var isTracking: Bool = false
    var startTime: Date?
    var endTime: Date?

    override init() {
        super.init()
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.requestWhenInUseAuthorization()
        isPedometerAvailable = CMPedometer.isStepCountingAvailable()
    }

    func startTracking() {
        isTracking = true
        startTime = Date()
        endTime = nil
        stepCount = 0
        distance = 0
        locations = []

        // 歩数計の開始
        if isPedometerAvailable {
            pedometer.startUpdates(from: Date()) { [weak self] data, error in
                guard let self = self, let data = data else { return }

                DispatchQueue.main.async {
                    self.stepCount = data.numberOfSteps.intValue
                    if let dist = data.distance {
                        self.distance = dist.doubleValue
                    }
                }
            }
        }

        // 位置情報の開始
        locationManager.startUpdatingLocation()
    }

    func stopTracking() {
        isTracking = false
        endTime = Date()
        pedometer.stopUpdates()
        locationManager.stopUpdatingLocation()
    }

    // MARK: - CLLocationManagerDelegate

    func locationManager(_ manager: CLLocationManager, didUpdateLocations newLocations: [CLLocation]) {
        guard let location = newLocations.last else { return }
        currentSpeed = max(0, location.speed)
        locations.append(location.coordinate)
    }

    // MARK: - 計算プロパティ

    var distanceInKm: Double {
        distance / 1000
    }

    var speedInKmh: Double {
        currentSpeed * 3.6
    }

    var caloriesBurned: Double {
        // 簡易計算：歩数 × 0.04 kcal（目安）
        Double(stepCount) * 0.04
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var tracker = ActivityTracker()

    // 経過時間（startTime からの差分。計測中は現在時刻 date、停止後は endTime で固定）
    private func elapsedTime(at date: Date) -> TimeInterval {
        guard let start = tracker.startTime else { return 0 }
        let end = tracker.endTime ?? date
        return max(0, end.timeIntervalSince(start))
    }

    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(spacing: 20) {
                    // タイマー表示
                    timerSection

                    // メイン統計
                    statsGrid

                    // スタート/ストップボタン
                    controlButton

                    // 速度メーター
                    if tracker.isTracking {
                        SpeedMeter(speed: tracker.speedInKmh)
                    }
                }
                .padding()
            }
            .navigationTitle("活動トラッカー")
        }
    }

    // MARK: - タイマーセクション

    private var timerSection: some View {
        // TimelineView が1秒ごとに context.date を更新し、この中だけが再描画される
        TimelineView(.periodic(from: .now, by: 1)) { context in
            VStack(spacing: 4) {
                Text(formatTime(elapsedTime(at: context.date)))
                    .font(.system(size: 48, weight: .thin, design: .monospaced))

                if tracker.isTracking {
                    Text("計測中")
                        .font(.caption)
                        .foregroundStyle(.green)
                }
            }
            .padding()
        }
    }

    // MARK: - 統計グリッド

    private var statsGrid: some View {
        LazyVGrid(columns: [
            GridItem(.flexible()),
            GridItem(.flexible()),
        ], spacing: 16) {
            StatCard(
                icon: "figure.walk",
                value: "\(tracker.stepCount)",
                unit: "歩",
                color: .blue
            )
            StatCard(
                icon: "map",
                value: String(format: "%.2f", tracker.distanceInKm),
                unit: "km",
                color: .green
            )
            StatCard(
                icon: "flame",
                value: String(format: "%.0f", tracker.caloriesBurned),
                unit: "kcal",
                color: .orange
            )
            StatCard(
                icon: "speedometer",
                value: String(format: "%.1f", tracker.speedInKmh),
                unit: "km/h",
                color: .purple
            )
        }
    }

    // MARK: - コントロールボタン

    private var controlButton: some View {
        Button {
            if tracker.isTracking {
                tracker.stopTracking()
            } else {
                tracker.startTracking()
            }
        } label: {
            HStack {
                Image(systemName: tracker.isTracking ? "stop.fill" : "play.fill")
                Text(tracker.isTracking ? "ストップ" : "スタート")
            }
            .font(.title3)
            .frame(maxWidth: .infinity)
            .padding()
            .background(tracker.isTracking ? Color.red : Color.green)
            .foregroundStyle(.white)
            .clipShape(RoundedRectangle(cornerRadius: 16))
        }
    }

    // MARK: - 時間フォーマット

    func formatTime(_ interval: TimeInterval) -> String {
        let hours = Int(interval) / 3600
        let minutes = Int(interval) / 60 % 60
        let seconds = Int(interval) % 60
        return String(format: "%02d:%02d:%02d", hours, minutes, seconds)
    }
}

// MARK: - 統計カード

struct StatCard: View {
    let icon: String
    let value: String
    let unit: String
    let color: Color

    var body: some View {
        VStack(spacing: 8) {
            Image(systemName: icon)
                .font(.title2)
                .foregroundStyle(color)

            Text(value)
                .font(.title)
                .bold()

            Text(unit)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .frame(maxWidth: .infinity)
        .padding()
        .background(
            RoundedRectangle(cornerRadius: 12)
                .fill(color.opacity(0.08))
        )
    }
}

// MARK: - 速度メーター

struct SpeedMeter: View {
    let speed: Double

    var body: some View {
        VStack(spacing: 8) {
            Text("現在の速度")
                .font(.caption)
                .foregroundStyle(.secondary)

            ZStack {
                Circle()
                    .trim(from: 0, to: 0.75)
                    .stroke(.gray.opacity(0.2), lineWidth: 8)
                    .rotationEffect(.degrees(135))

                Circle()
                    .trim(from: 0, to: min(speed / 15.0, 1.0) * 0.75)
                    .stroke(speedColor, style: StrokeStyle(lineWidth: 8, lineCap: .round))
                    .rotationEffect(.degrees(135))
                    .animation(.spring, value: speed)

                VStack {
                    Text(String(format: "%.1f", speed))
                        .font(.system(size: 32, weight: .bold, design: .monospaced))
                    Text("km/h")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .frame(width: 150, height: 150)
        }
        .padding()
    }

    var speedColor: Color {
        if speed < 4 { return .green }
        if speed < 8 { return .orange }
        return .red
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

（アプリの動作を自分の言葉で説明する。スクリーンショットを貼ってもよい。）

## コードの詳細解説

### CoreMotionの基本（CMMotionManager）

```swift
// MARK: - モーションマネージャー

@Observable
class MotionManager {
    private let motionManager = CMMotionManager()

    var pitch: Double = 0    // 前後の傾き
    var roll: Double = 0     // 左右の傾き
    var yaw: Double = 0      // 水平方向の回転
    var isAvailable: Bool

    init() {
        // 初回 body 評価時点で正しい値を返すよう、init で同期的にセット
        isAvailable = motionManager.isDeviceMotionAvailable
    }

    func startUpdates() {
        guard isAvailable else { return }

        motionManager.deviceMotionUpdateInterval = 1.0 / 60.0

        motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
            guard let self = self, let motion = motion else { return }

            self.pitch = motion.attitude.pitch
            self.roll = motion.attitude.roll
            self.yaw = motion.attitude.yaw
        }
    }

    func stopUpdates() {
        motionManager.stopDeviceMotionUpdates()
    }
}
```

**何をしているか：**

CMMotionManagerを使って、iPhoneの加速度センサーやジャイロセンサーから端末の動きや傾きを取得している。
```swift
private let motionManager = CMMotionManager()
```
で、モーションセンサーを管理するオブジェクトを作成している。

```swift
isAvailable = motionManager.isDeviceMotionAvailable
```

では、使用している端末でモーションデータを取得できるか確認しています。

```swift
motionManager.deviceMotionUpdateInterval = 1.0 / 60.0
```

では、センサーの情報を1秒間に約60回取得するように設定しています。

```swift
motionManager.startDeviceMotionUpdates(to: .main)
```

によって、端末の傾きや回転に関する情報の取得を開始しています。

**なぜこう書くのか：**

端末の傾きをリアルタイムで画面に反映し、水平器として動作させるためです。
CMMotionManagerを使用すると、加速度センサーやジャイロセンサーの情報をまとめたdeviceMotionを取得できます。
また、更新間隔を設定することで、バブルの動きを滑らかに表示できます

**もしこう書かなかったら：**

CMMotionManagerを作成しなければ、端末の傾きや回転を取得できない。
startDeviceMotionUpdatesを書かなければ、端末を傾けてもpitch、roll、yawの値が更新されず、水平器のバブルも動かない。
また、isDeviceMotionAvailableによる確認を書かない場合、センサーが利用できないシミュレータなどでも取得処理を開始しようとする。

---

### デバイスの姿勢データ（pitch/roll/yaw）

```swift
// MARK: - 水平器インジケーター

struct LevelIndicator: View {
    let pitch: Double
    let roll: Double

    private let maxOffset: CGFloat = 100

    private var xOffset: CGFloat {
        CGFloat(roll) * maxOffset
    }

    private var yOffset: CGFloat {
        CGFloat(pitch) * maxOffset
    }

    private var isLevel: Bool {
        abs(pitch) < 0.03 && abs(roll) < 0.03
    }

    var body: some View {
        ZStack {
            // 外側の円
            Circle()
                .stroke(.gray.opacity(0.3), lineWidth: 2)
                .frame(width: 250, height: 250)

            // 中心の十字線
            Path { path in
                path.move(to: CGPoint(x: 125, y: 0))
                path.addLine(to: CGPoint(x: 125, y: 250))
                path.move(to: CGPoint(x: 0, y: 125))
                path.addLine(to: CGPoint(x: 250, y: 125))
            }
            .stroke(.gray.opacity(0.2), lineWidth: 1)
            .frame(width: 250, height: 250)

            // 中間の円
            Circle()
                .stroke(.gray.opacity(0.2), lineWidth: 1)
                .frame(width: 125, height: 125)

            // バブル（傾きに応じて移動）
            Circle()
                .fill(isLevel ? .green : .red)
                .frame(width: 40, height: 40)
                .opacity(0.8)
                .shadow(color: isLevel ? .green : .red, radius: 8)
                .offset(
                    x: max(-maxOffset, min(maxOffset, xOffset)),
                    y: max(-maxOffset, min(maxOffset, yOffset))
                )
                .animation(.spring(duration: 0.1), value: xOffset)
                .animation(.spring(duration: 0.1), value: yOffset)

            // 水平時の表示
            if isLevel {
                Text("水平!")
                    .font(.headline)
                    .foregroundStyle(.green)
                    .offset(y: 140)
            }
        }
    }
}

// MARK: - 数値データ表示

struct DataDisplay: View {
    let pitch: Double
    let roll: Double
    let yaw: Double

    var body: some View {
        VStack(spacing: 12) {
            DataRow(
                label: "前後の傾き（Pitch）",
                value: pitch,
                icon: "arrow.up.and.down"
            )
            DataRow(
                label: "左右の傾き（Roll）",
                value: roll,
                icon: "arrow.left.and.right"
            )
            DataRow(
                label: "水平回転（Yaw）",
                value: yaw,
                icon: "arrow.triangle.2.circlepath"
            )
        }
        .padding()
        .background(
            RoundedRectangle(cornerRadius: 12)
                .fill(.gray.opacity(0.05))
        )
    }
}

struct DataRow: View {
    let label: String
    let value: Double
    let icon: String

    var body: some View {
        HStack {
            Image(systemName: icon)
                .frame(width: 30)
                .foregroundStyle(.blue)

            Text(label)
                .font(.caption)

            Spacer()

            Text(String(format: "%.3f rad", value))
                .font(.system(.caption, design: .monospaced))
                .foregroundStyle(.secondary)

            Text(String(format: "(%.1f°)", value * 180 / .pi))
                .font(.system(.caption, design: .monospaced))
                .foregroundStyle(.secondary)
                .frame(width: 60, alignment: .trailing)
        }
    }
}

```

**何をしているか：**

端末の姿勢を表すpitch、roll、yawを取得しています。

```swift
self.pitch = motion.attitude.pitch
self.roll = motion.attitude.roll
self.yaw = motion.attitude.yaw
```

それぞれの値は、次のような傾きや回転を表します。

* pitch：端末の前後方向の傾き
* roll：端末の左右方向の傾き
* yaw：端末の水平方向の回転

このコードでは、pitchとrollを水平器のバブルの移動に使用している。


```swift
private var xOffset: CGFloat {
    CGFloat(roll) * maxOffset
}

private var yOffset: CGFloat {
    CGFloat(pitch) * maxOffset
}
```

rollを横方向、pitchを縦方向の移動量に変換している。

```swift
abs(pitch) < 0.03 && abs(roll) < 0.03
```
によって、端末がほぼ水平になっているかを判定しています。

**なぜこう書くのか：**

端末がどの方向に、どの程度傾いているかを画面上で表現するため。
pitchとrollをバブルの位置に反映することで、実際の水平器のように、端末を傾けるとバブルが移動する。
yawは水平器のバブル移動には使用していないが、端末が水平方向にどの程度回転しているかを数値で確認するために表示している。

**もしこう書かなかったら：**

pitchとrollを取得しなければ、端末の前後左右の傾きを判定できません。そのため、バブルは中央から動かず、水平器として機能しません。

yawを取得しなければ、端末の水平方向の回転を表示できません。ただし、水平判定にはpitchとrollを使っているため、yawがなくても水平器自体は動作します。

---

### 歩数計（CMPedometer）

```swift
歩数計（CMPedometer)

// MARK: - 活動トラッカー

@Observable
class ActivityTracker: NSObject, CLLocationManagerDelegate {
    // 歩数関連
    private let pedometer = CMPedometer()
    var stepCount: Int = 0
    var distance: Double = 0     // メートル
    var isPedometerAvailable: Bool = false

    // 位置関連
    private let locationManager = CLLocationManager()
    var currentSpeed: Double = 0  // m/s
    var locations: [CLLocationCoordinate2D] = []

    // 状態
    var isTracking: Bool = false
    var startTime: Date?
    var endTime: Date?

    override init() {
        super.init()
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.requestWhenInUseAuthorization()
        isPedometerAvailable = CMPedometer.isStepCountingAvailable()
    }

    func startTracking() {
        isTracking = true
        startTime = Date()
        endTime = nil
        stepCount = 0
        distance = 0
        locations = []

        // 歩数計の開始
        if isPedometerAvailable {
            pedometer.startUpdates(from: Date()) { [weak self] data, error in
                guard let self = self, let data = data else { return }

                DispatchQueue.main.async {
                    self.stepCount = data.numberOfSteps.intValue
                    if let dist = data.distance {
                        self.distance = dist.doubleValue
                    }
                }
            }
        }

        // 位置情報の開始
        locationManager.startUpdatingLocation()
    }

    func stopTracking() {
        isTracking = false
        endTime = Date()
        pedometer.stopUpdates()
        locationManager.stopUpdatingLocation()
    }
```

**何をしているか：**

CMPedometerを使って、計測開始後の歩数と移動距離を取得している。

```swift
private let pedometer = CMPedometer()
```
で歩数計を管理するオブジェクトを作成している。
```swift
isPedometerAvailable = CMPedometer.isStepCountingAvailable()
```
では、使用している端末が歩数計測に対応しているかを確認している。

計測開始時には、
```swift
pedometer.startUpdates(from: Date()) { [weak self] data, error in
```
を実行し、現在時刻からの歩数データを継続的に取得しています。
```swift
self.stepCount = data.numberOfSteps.intValue
```
で歩数を保存し、
```swift
self.distance = dist.doubleValue
```
で移動距離を保存している。

**なぜこう書くのか：**

端末の加速度データから自分で歩行を判定しなくても、iPhoneが計算した歩数や移動距離を簡単に取得できるためです。
基本編で使用したCMMotionManagerでも端末の動きは取得できますが、歩数を自分で判定するには複雑な計算が必要です。
CMPedometerを使用することで、歩数計測に必要な処理を簡単に実装できる。
また、取得した値は画面表示に使用するため、

```swift
DispatchQueue.main.async
```
を使ってメインスレッド上で更新しています。

**もしこう書かなかったら：**

CMPedometerを使用しなければ、歩数や歩行距離を取得できない。
startUpdatesを書かなければ、スタートボタンを押しても歩数が更新されない。
また、
```swift
pedometer.stopUpdates()
```
を書かなければ、ストップボタンを押した後も歩数の取得処理が続く可能性がある。

---

### CoreLocationとの連携

```swift
// 該当部分のコードを抜粋して貼る
```

**何をしているか：**

CoreLocationを使用して、現在の移動速度と通過した位置情報を取得しています。

```swift
private let locationManager = CLLocationManager()
```

で位置情報を管理するオブジェクトを作成しています。

```swift
locationManager.delegate = self
```
では、取得した位置情報をActivityTrackerで受け取れるようにしています。

```swift
locationManager.requestWhenInUseAuthorization()
```

によって、アプリ使用中の位置情報利用許可をユーザーに求めています。

計測開始時には、

```swift
locationManager.startUpdatingLocation()
```

で位置情報の取得を開始します。
位置情報が更新されると、次のメソッドが呼び出されます。

```swift
func locationManager(
    _ manager: CLLocationManager,
    didUpdateLocations newLocations: [CLLocation]
)
```

その中で、
```swift
currentSpeed = max(0, location.speed)
```

により現在の移動速度を取得しています。

location.speedはメートル毎秒で取得されるため、

```swift
currentSpeed * 3.6
```

によってkm/hに変換しています。
また、

```swift
locations.append(location.coordinate)
```

によって、移動中に取得した緯度と経度を配列へ保存しています。

**なぜこう書くのか：**

CMPedometerでは歩数や推定移動距離を取得できますが、現在地や現在の移動速度は取得できません。
そのため、CoreLocationを組み合わせて、GPSから次の情報を取得しています。

* 現在の移動速度
* 現在地
* 移動した経路

CoreMotionとCoreLocationを組み合わせることで、歩数だけでなく、速度や移動位置も記録できる活動トラッカーになります。

**もしこう書かなかったら：**

CoreLocationを使用しなければ、現在の速度や通過した位置情報を取得できません。
その場合、速度表示や速度メーターは0km/hのままになり、移動経路も記録されません。
delegateを設定しなければ、位置情報が更新されてもdidUpdateLocationsが呼び出されません。
startUpdatingLocation()を書かなければ、位置情報の取得が開始されません。

また、
```swift
locationManager.stopUpdatingLocation()
```

を書かなければ、計測を停止した後もGPSが動き続け、バッテリーを余計に消費する可能性があります。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`CMMotionManager` | 加速度・ジャイロ・気圧などのセンサーデータを取得 | `motionManager.startDeviceMotionUpdates(to: .main) { ... }` |
| 例：`CMPedometer` | 歩数や歩行距離をカウント | `pedometer.queryPedometerData(from: startDate, to: Date())` |
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

この章では、CoreMotionを使って端末の傾きや歩数を取得する方法を学びました。
基本編では、CMMotionManagerからpitch・roll・yawを取得し、端末の傾きを水平器の表示に反映する方法を確認しました。
応用編では、CMPedometerによる歩数や移動距離の取得と、CoreLocationによる現在の速度や位置情報の取得方法を学びました。
この章を通して、iPhoneのモーションセンサーや位置情報をアプリの表示や機能に活用する方法を理解しました。
