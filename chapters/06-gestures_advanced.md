# 第6章：ジェスチャー操作

## この章で学ぶこと

本章では、SwiftUIのジェスチャー機能を使い、タップ・長押し・ドラッグ・拡大縮小・回転などの基本操作を実装する方法を学ぶ。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第6章（応用）：Tinder風スワイプカードUI
// ============================================
// ドラッグジェスチャーとアニメーションを組み合わせて、
// カードを左右にスワイプして仕分けるUIを作ります。
// ============================================

import SwiftUI

// MARK: - データモデル

struct Animal: Identifiable {
    let id = UUID()
    let name: String
    let emoji: String
    let description: String
    let color: Color
}

extension Animal {
    static let sampleData: [Animal] = [
        Animal(name: "ネコ", emoji: "🐱", description: "自由気ままなマイペース派", color: .orange),
        Animal(name: "イヌ", emoji: "🐶", description: "忠実で人懐っこい", color: .brown),
        Animal(name: "ウサギ", emoji: "🐰", description: "おとなしくてかわいい", color: .pink),
        Animal(name: "ペンギン", emoji: "🐧", description: "南極のタキシード紳士", color: .cyan),
        Animal(name: "パンダ", emoji: "🐼", description: "笹が大好きなのんびり屋", color: .green),
        Animal(name: "フクロウ", emoji: "🦉", description: "夜型の知恵者", color: .purple)
    ]
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var animals: [Animal] = Animal.sampleData
    @State private var likedAnimals: [Animal] = []
    @State private var dislikedAnimals: [Animal] = []

    var body: some View {
        VStack(spacing: 20) {
            Text("好きな動物は？")
                .font(.title2)
                .bold()

            // スコア表示
            HStack(spacing: 40) {
                Label("\(dislikedAnimals.count)", systemImage: "hand.thumbsdown")
                    .foregroundStyle(.red)
                Label("\(likedAnimals.count)", systemImage: "hand.thumbsup")
                    .foregroundStyle(.green)
            }
            .font(.headline)

            // カードスタック
            ZStack {
                if animals.isEmpty {
                    VStack(spacing: 12) {
                        Text("完了！")
                            .font(.largeTitle)

                        Button("もう一度") {
                            animals = Animal.sampleData.shuffled()
                            likedAnimals = []
                            dislikedAnimals = []
                        }
                        .buttonStyle(.borderedProminent)
                    }
                } else {
                    // ↓これいらない
                    ForEach(animals.reversed()) { animal in
                        SwipeCardView(animal: animal) { direction in
                            removeCard(animal: animal, direction: direction)
                        }
                    }
                }
            }
            .frame(height: 400)

            // 手動ボタン
            if !animals.isEmpty {
                HStack(spacing: 40) {
                    Button {
                        if let top = animals.first {
                            removeCard(animal: top, direction: .left)
                        }
                    } label: {
                        Image(systemName: "xmark.circle.fill")
                            .font(.system(size: 50))
                            .foregroundStyle(.red)
                    }

                    Button {
                        if let top = animals.first {
                            removeCard(animal: top, direction: .right)
                        }
                    } label: {
                        Image(systemName: "heart.circle.fill")
                            .font(.system(size: 50))
                            .foregroundStyle(.green)
                    }
                }
            }

            Spacer()
        }
        .padding()
    }

    func removeCard(animal: Animal, direction: SwipeDirection) {
        withAnimation(.spring(duration: 0.3)) {
            animals.removeAll { $0.id == animal.id }
        }

        switch direction {
        case .left:
            dislikedAnimals.append(animal)
        case .right:
            likedAnimals.append(animal)
        }
    }
}

// MARK: - スワイプ方向

enum SwipeDirection {
    case left, right
}

// MARK: - スワイプカードビュー

struct SwipeCardView: View {
    let animal: Animal
    let onSwipe: (SwipeDirection) -> Void

    @State private var offset: CGSize = .zero
    @State private var rotation: Double = 0

    private let swipeThreshold: CGFloat = 100

    private var swipeProgress: CGFloat {
        min(abs(offset.width) / swipeThreshold, 1.0)
    }

    var body: some View {
        ZStack {
            // カード背景
            RoundedRectangle(cornerRadius: 20)
                .fill(animal.color.opacity(0.15))
                .overlay(
                    RoundedRectangle(cornerRadius: 20)
                        .stroke(animal.color.opacity(0.3), lineWidth: 2)
                )

            // カード内容
            VStack(spacing: 16) {
                Text(animal.emoji)
                    .font(.system(size: 80))

                Text(animal.name)
                    .font(.title)
                    .bold()

                Text(animal.description)
                    .font(.body)
                    .foregroundStyle(.secondary)
            }

            // いいね / NG オーバーレイ
            if offset.width > 0 {
                Text("LIKE")
                    .font(.system(size: 40, weight: .bold))
                    .foregroundStyle(.green)
                    .opacity(swipeProgress)
                    .rotationEffect(.degrees(-20))
                    .position(x: 80, y: 60)
            } else if offset.width < 0 {
                Text("NOPE")
                    .font(.system(size: 40, weight: .bold))
                    .foregroundStyle(.red)
                    .opacity(swipeProgress)
                    .rotationEffect(.degrees(20))
                    .position(x: 240, y: 60)
            }
        }
        .frame(width: 300, height: 380)
        .shadow(color: .black.opacity(0.1), radius: 8)
        .offset(offset)
        .rotationEffect(.degrees(rotation))
        .gesture(
            DragGesture()
                .onChanged { value in
                    offset = value.translation
                    rotation = Double(value.translation.width / 20)
                }
                .onEnded { value in
                    if value.translation.width > swipeThreshold {
                        // 右スワイプ → LIKE
                        withAnimation(.easeOut(duration: 0.3)) {
                            offset = CGSize(width: 500, height: 0)
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                            onSwipe(.right)
                        }
                    } else if value.translation.width < -swipeThreshold {
                        // 左スワイプ → NOPE
                        withAnimation(.easeOut(duration: 0.3)) {
                            offset = CGSize(width: -500, height: 0)
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                            onSwipe(.left)
                        }
                    } else {
                        // 元に戻す
                        withAnimation(.spring) {
                            offset = .zero
                            rotation = 0
                        }
                    }
                }
        )
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

Tinder風のアプリで、左右にドラッグし好みを分けることができる。

<img width="180" height="393" alt="swift_github_GestureAdvanced" src="https://github.com/user-attachments/assets/95111016-2821-4073-9d46-408dac2cb7c5" />

## コードの詳細解説

## メインビュー

```swift
struct ContentView: View {
    @State private var animals: [Animal] = Animal.sampleData
    @State private var likedAnimals: [Animal] = []
    @State private var dislikedAnimals: [Animal] = []

    var body: some View {
        VStack(spacing: 20) {
            Text("好きな動物は？")
                .font(.title2)
                .bold()

            // スコア表示
            HStack(spacing: 40) {
                Label("\(dislikedAnimals.count)", systemImage: "hand.thumbsdown")
                    .foregroundStyle(.red)
                Label("\(likedAnimals.count)", systemImage: "hand.thumbsup")
                    .foregroundStyle(.green)
            }
            .font(.headline)

            // カードスタック
            ZStack {
                if animals.isEmpty {
                    VStack(spacing: 12) {
                        Text("完了！")
                            .font(.largeTitle)

                        Button("もう一度") {
                            animals = Animal.sampleData.shuffled()
                            likedAnimals = []
                            dislikedAnimals = []
                        }
                        .buttonStyle(.borderedProminent)
                    }
                } else {
                    // ↓これいらない
                    ForEach(animals.reversed()) { animal in
                        SwipeCardView(animal: animal) { direction in
                            removeCard(animal: animal, direction: direction)
                        }
                    }
                }
            }
            .frame(height: 400)

            // 手動ボタン
            if !animals.isEmpty {
                HStack(spacing: 40) {
                    Button {
                        if let top = animals.first {
                            removeCard(animal: top, direction: .left)
                        }
                    } label: {
                        Image(systemName: "xmark.circle.fill")
                            .font(.system(size: 50))
                            .foregroundStyle(.red)
                    }

                    Button {
                        if let top = animals.first {
                            removeCard(animal: top, direction: .right)
                        }
                    } label: {
                        Image(systemName: "heart.circle.fill")
                            .font(.system(size: 50))
                            .foregroundStyle(.green)
                    }
                }
            }

            Spacer()
        }
        .padding()
    }

    func removeCard(animal: Animal, direction: SwipeDirection) {
        withAnimation(.spring(duration: 0.3)) {
            animals.removeAll { $0.id == animal.id }
        }

        switch direction {
        case .left:
            dislikedAnimals.append(animal)
        case .right:
            likedAnimals.append(animal)
        }
    }
}

// MARK: - スワイプ方向

enum SwipeDirection {
    case left, right
}

```

**何をしているのか：**

動物カードを表示し、左右へのスワイプやボタン操作によって、動物を「好き」と「好きではない」に分類する画面を作成している。
animalsには未判定の動物、likedAnimalsには好きと判定した動物、dislikedAnimalsには好きではないと判定した動物を保存している。
画面上部には好き・好きではないと判定した動物の数を表示し、中央には動物カードを重ねて表示している。すべてのカードを判定すると「完了！」と表示され、「もう一度」ボタンで最初からやり直せる。
カードを左右にスワイプするだけでなく、画面下部のバツボタンとハートボタンからも判定できる。

**なぜこう書くのか：**

@Stateを使用することで、動物の配列が変更されたときに画面の表示を自動的に更新できる。
ZStackを使用することで、複数の動物カードを重ねて表示できる。

```swift
ForEach(animals.reversed()) { animal in
```

では、動物の配列を逆順に表示している。ZStackでは後から表示されたViewが手前に配置されるため、animals.firstの動物を一番上のカードとして表示するためにreversed()を使用している。
カードがスワイプされたときは、SwipeCardViewから渡された方向をremoveCardに送り、対象のカードを削除して、それぞれの配列に追加している。

removeCardにカード削除と判定処理をまとめることで、スワイプ操作とボタン操作の両方から同じ処理を呼び出せる。

**もしこう書かなかったら：**

@Stateを使用しなかった場合、配列の内容を変更しても、カードや判定数の表示が正しく更新されない。
ZStackを使用しなかった場合、カードが重ならず、縦や横に並んで表示されてしまう。
ForEachを削除した場合、配列に保存されている動物カードを画面に表示できない。コメントには「これいらない」とあるが、複数のカードを重ねて表示する場合は必要な処理である。
reversed()を使用しなかった場合、配列の最後にある動物が一番上に表示される。そのため、ボタン操作で取得しているanimals.firstと、実際に画面上に表示されているカードが一致しなくなる。
removeCardを作成しなかった場合、スワイプ処理とボタン処理のそれぞれに同じ削除・分類処理を書く必要があり、コードが重複して修正しにくくなる。
animals.isEmptyによる条件分岐がなかった場合、すべてのカードを判定したあとも完了画面を表示できず、何も表示されていない画面になってしまう。

---

### スワイプカードビュー


```swift
// MARK: - スワイプカードビュー

struct SwipeCardView: View {
    let animal: Animal
    let onSwipe: (SwipeDirection) -> Void

    @State private var offset: CGSize = .zero
    @State private var rotation: Double = 0

    private let swipeThreshold: CGFloat = 100

    private var swipeProgress: CGFloat {
        min(abs(offset.width) / swipeThreshold, 1.0)
    }

    var body: some View {
        ZStack {
            // カード背景
            RoundedRectangle(cornerRadius: 20)
                .fill(animal.color.opacity(0.15))
                .overlay(
                    RoundedRectangle(cornerRadius: 20)
                        .stroke(animal.color.opacity(0.3), lineWidth: 2)
                )

            // カード内容
            VStack(spacing: 16) {
                Text(animal.emoji)
                    .font(.system(size: 80))

                Text(animal.name)
                    .font(.title)
                    .bold()

                Text(animal.description)
                    .font(.body)
                    .foregroundStyle(.secondary)
            }

            // いいね / NG オーバーレイ
            if offset.width > 0 {
                Text("LIKE")
                    .font(.system(size: 40, weight: .bold))
                    .foregroundStyle(.green)
                    .opacity(swipeProgress)
                    .rotationEffect(.degrees(-20))
                    .position(x: 80, y: 60)
            } else if offset.width < 0 {
                Text("NOPE")
                    .font(.system(size: 40, weight: .bold))
                    .foregroundStyle(.red)
                    .opacity(swipeProgress)
                    .rotationEffect(.degrees(20))
                    .position(x: 240, y: 60)
            }
        }
        .frame(width: 300, height: 380)
        .shadow(color: .black.opacity(0.1), radius: 8)
        .offset(offset)
        .rotationEffect(.degrees(rotation))
        .gesture(
            DragGesture()
                .onChanged { value in
                    offset = value.translation
                    rotation = Double(value.translation.width / 20)
                }
                .onEnded { value in
                    if value.translation.width > swipeThreshold {
                        // 右スワイプ → LIKE
                        withAnimation(.easeOut(duration: 0.3)) {
                            offset = CGSize(width: 500, height: 0)
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                            onSwipe(.right)
                        }
                    } else if value.translation.width < -swipeThreshold {
                        // 左スワイプ → NOPE
                        withAnimation(.easeOut(duration: 0.3)) {
                            offset = CGSize(width: -500, height: 0)
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                            onSwipe(.left)
                        }
                    } else {
                        // 元に戻す
                        withAnimation(.spring) {
                            offset = .zero
                            rotation = 0
                        }
                    }
                }
        )
    }
}
```

**何をしているのか：**

1枚の動物カードを表示し、ドラッグ操作によって左右にスワイプできるようにしている。
カードには動物の絵文字、名前、説明を表示し、右に動かすと「LIKE」、左に動かすと「NOPE」の文字が表示される。
スワイプした距離が100ポイントを超えた場合はカードを画面外へ移動させ、右なら.right、左なら.leftをonSwipeを通してメインビューへ伝える。
スワイプ距離が100ポイント未満の場合は、カードを元の位置に戻す。

**なぜこう書くのか：**

```swift
@State private var offset: CGSize = .zero
@State private var rotation: Double = 0
```

offsetでカードの移動位置、rotationでカードの傾きを管理している。@Stateを使うことで、ドラッグ中の位置や角度の変化がすぐに画面へ反映される。

```swift
let onSwipe: (SwipeDirection) -> Void
```

カード自身は、好き・嫌いの配列を直接変更せず、スワイプ方向だけを親のビューへ伝えている。これにより、カード表示とデータ管理の役割を分けられる。

```swift
private let swipeThreshold: CGFloat = 100
```

カードを判定するために必要なスワイプ距離を設定している。少し指を動かしただけで判定されることを防いでいる。

```swift
private var swipeProgress: CGFloat {
    min(abs(offset.width) / swipeThreshold, 1.0)
}
```

カードを横に動かした距離から、スワイプの進み具合を0〜1で計算している。この値を「LIKE」と「NOPE」の透明度に使用することで、カードを動かすほど文字が濃く表示される。

```swift
offset = value.translation

rotation = Double(value.translation.width / 20)
```

ドラッグした距離をカードの位置に反映し、横方向の移動量に応じてカードを傾けている。これにより、実際のカードをめくるような動きになる。

スワイプ成立時は、先にカードを画面外へ移動するアニメーションを実行し、その0.3秒後にonSwipeを呼び出している。これにより、カードが突然消えるのではなく、画面外へ飛んでから削除されるように見える。



**もしこう書かなかったら：**

offsetがなかった場合、指を動かしてもカードが移動せず、スワイプ操作を視覚的に確認できない。
rotationがなかった場合でもスワイプはできるが、カードは傾かず平行移動だけになるため、動きがやや不自然になる。
swipeThresholdがなかった場合、少し触っただけでもLIKEやNOPEとして判定される可能性がある。
swipeProgressがなかった場合、「LIKE」や「NOPE」の文字をスワイプ量に応じて徐々に表示できない。
abs(offset.width)を使わなかった場合、左スワイプ時には値がマイナスになるため、正しい進捗を計算できない。
min(..., 1.0)がなかった場合、スワイプ距離が100ポイントを超えたときに透明度が1を超える値になる。
onSwipeがなかった場合、カードビューからメインビューへスワイプ方向を伝えられず、動物を好き・嫌いの配列に分類できない。
DispatchQueue.main.asyncAfterを使わず、すぐにonSwipeを呼び出した場合、画面外へ移動するアニメーションが完了する前にカードが削除され、突然消えたように見える。
スワイプ距離が足りない場合にoffsetとrotationを元に戻さなければ、途中まで動かしたカードがその位置に残ってしまう。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| | | |
| | | |
| | | |


## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：

```swift

```

- 結果：

- わかったこと：

**実験2：**
- やったこと：
- 結果：
- わかったこと：

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**

```swift
 
```

2. **質問：**

```swift

```
   **得られた理解：**

3. **質問：**
   **得られた理解：**

## この章のまとめ

ジェスチャーだけではなく、状態管理と画面更新を組み合わせてUIを作ることを学んだ。

### 重要なポイントは主に下記の３つ

* DragGestureでスワイプ操作を検知する方法
* @Stateでカードの位置や回転などの状態を管理する方法
* スワイプ結果を親ビューへ通知し、データと画面を連携させる方法

