# 第6章：ジェスチャー操作

> 執筆者：田中 吾錬
> 最終更新：2026-07-17

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

例：この章では、ユーザーの指の動きを検出するジェスチャー認識の方法を学ぶ。タップ・ロングプレス・ドラッグ・拡大縮小・回転など、各ジェスチャーの実装方法を学び、最終的にTinder風のスワイプUIで複数のジェスチャーを組み合わせた実装を題材にする。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第6章（基本）：ジェスチャーで操作するカードアプリ
// ============================================
// タップ、ロングプレス、ドラッグ、ピンチ、回転の
// 各ジェスチャーを実際に体験しながら学びます。
// ============================================

import SwiftUI

// MARK: - メインビュー

struct ContentView: View {
    var body: some View {
        NavigationStack {
            List {
                NavigationLink("タップ & ロングプレス") {
                    TapDemoView()
                }
                NavigationLink("ドラッグ") {
                    DragDemoView()
                }
                NavigationLink("ピンチ（拡大縮小）") {
                    MagnifyDemoView()
                }
                NavigationLink("回転") {
                    RotateDemoView()
                }
                NavigationLink("組み合わせ") {
                    CombinedDemoView()
                }
            }
            .navigationTitle("ジェスチャー体験")
        }
    }
}

// MARK: - タップ & ロングプレス

struct TapDemoView: View {
    @State private var tapCount = 0
    @State private var backgroundColor: Color = .blue
    @State private var isPressed = false

    var body: some View {
        VStack(spacing: 30) {
            Text("タップ回数: \(tapCount)")
                .font(.title)

            // シングルタップ
            RoundedRectangle(cornerRadius: 16)
                .fill(backgroundColor)
                .frame(width: 200, height: 200)
                .overlay {
                    Text("タップしてね")
                        .foregroundStyle(.white)
                        .font(.headline)
                }
                .onTapGesture {
                    tapCount += 1
                    backgroundColor = Color(
                        hue: Double.random(in: 0...1),
                        saturation: 0.7,
                        brightness: 0.9
                    )
                }

            // ロングプレス
            Circle()
                .fill(isPressed ? .green : .orange)
                .frame(width: 120, height: 120)
                .scaleEffect(isPressed ? 1.3 : 1.0)
                .overlay {
                    Text(isPressed ? "成功!" : "長押し")
                        .foregroundStyle(.white)
                        .font(.headline)
                }
                .animation(.spring(duration: 0.3), value: isPressed)
                .onLongPressGesture(minimumDuration: 1.0) {
                    isPressed = true
                    DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
                        isPressed = false
                    }
                }
        }
        .navigationTitle("タップ & ロングプレス")
    }
}

// MARK: - ドラッグ

struct DragDemoView: View {
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero

    var body: some View {
        VStack {
            Text("カードをドラッグしてみよう")
                .font(.headline)
                .padding()

            Spacer()

            RoundedRectangle(cornerRadius: 20)
                .fill(
                    LinearGradient(
                        colors: [.purple, .blue],
                        startPoint: .topLeading,
                        endPoint: .bottomTrailing
                    )
                )
                .frame(width: 200, height: 280)
                .shadow(radius: 8)
                .overlay {
                    VStack {
                        Image(systemName: "hand.draw")
                            .font(.system(size: 40))
                        Text("ドラッグ")
                            .font(.title3)
                    }
                    .foregroundStyle(.white)
                }
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = CGSize(
                                width: lastOffset.width + value.translation.width,
                                height: lastOffset.height + value.translation.height
                            )
                        }
                        .onEnded { _ in
                            lastOffset = offset
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    offset = .zero
                    lastOffset = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("ドラッグ")
    }
}

// MARK: - ピンチ（拡大縮小）

struct MagnifyDemoView: View {
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0

    var body: some View {
        VStack {
            Text("ピンチで拡大縮小")
                .font(.headline)
                .padding()

            Text(String(format: "倍率: %.1fx", scale))
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            Image(systemName: "star.fill")
                .font(.system(size: 100))
                .foregroundStyle(.yellow)
                // タッチ判定を300×300の透明な領域に広げる
                .frame(width: 300, height: 300)
                .contentShape(Rectangle())
                .scaleEffect(scale)
                .gesture(
                    MagnifyGesture()
                        .onChanged { value in
                            scale = lastScale * value.magnification
                        }
                        .onEnded { _ in
                            lastScale = scale
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    scale = 1.0
                    lastScale = 1.0
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("ピンチ")
    }
}

// MARK: - 回転

struct RotateDemoView: View {
    @State private var angle: Angle = .zero
    @State private var lastAngle: Angle = .zero

    var body: some View {
        VStack {
            Text("2本指で回転")
                .font(.headline)
                .padding()

            Text(String(format: "角度: %.0f°", angle.degrees))
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            Image(systemName: "arrow.up")
                .font(.system(size: 80))
                .foregroundStyle(.red)
                // タッチ判定を300×300の透明な領域に広げる
                .frame(width: 300, height: 300)
                .contentShape(Rectangle())
                .rotationEffect(angle)
                .gesture(
                    RotateGesture()
                        .onChanged { value in
                            angle = lastAngle + value.rotation
                        }
                        .onEnded { _ in
                            lastAngle = angle
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    angle = .zero
                    lastAngle = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("回転")
    }
}

// MARK: - 組み合わせ

struct CombinedDemoView: View {
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0
    @State private var angle: Angle = .zero
    @State private var lastAngle: Angle = .zero

    var body: some View {
        VStack {
            Text("ドラッグ・ピンチ・回転を同時に")
                .font(.headline)
                .padding()

            Spacer()

            Image(systemName: "photo.artframe")
                .font(.system(size: 120))
                .foregroundStyle(.indigo)
                // タッチ判定を300×300の透明な領域に広げる
                .frame(width: 300, height: 300)
                .contentShape(Rectangle())
                .scaleEffect(scale)
                .rotationEffect(angle)
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = CGSize(
                                width: lastOffset.width + value.translation.width,
                                height: lastOffset.height + value.translation.height
                            )
                        }
                        .onEnded { _ in
                            lastOffset = offset
                        }
                )
                // 複数のジェスチャーを「同時に」効かせるには
                // .gesture を重ねるのではなく .simultaneousGesture を使う
                .simultaneousGesture(
                    MagnifyGesture()
                        .onChanged { value in
                            scale = lastScale * value.magnification
                        }
                        .onEnded { _ in
                            lastScale = scale
                        }
                )
                .simultaneousGesture(
                    RotateGesture()
                        .onChanged { value in
                            angle = lastAngle + value.rotation
                        }
                        .onEnded { _ in
                            lastAngle = angle
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    offset = .zero
                    lastOffset = .zero
                    scale = 1.0
                    lastScale = 1.0
                    angle = .zero
                    lastAngle = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("組み合わせ")
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

画面タップしていろいろする

### タップとロングプレス
>
<img width="180" height="393" alt="Simulator Screenshot - iPhone 17 Pro - 2026-07-17 at 15 54 56" src="https://github.com/user-attachments/assets/42a5c454-0d88-4afc-b028-ef7dd35193fa" />

<img width="180" height="393" alt="swift_github_up" src="https://github.com/user-attachments/assets/088fc6f9-7a27-4fc3-91c9-92b51cbc312e" />

### ドラッグ

<img width="180" height="393" alt="Simulator Screenshot - iPhone 17 Pro - 2026-07-17 at 15 56 28" src="https://github.com/user-attachments/assets/d591aa80-60b0-4c2d-8968-63d465064e79" />

<img width="180" height="393" alt="Simulator Screenshot - iPhone 17 Pro - 2026-07-17 at 15 56 31" src="https://github.com/user-attachments/assets/37b4f139-b429-4aab-bc38-03033c01cff4" />

<img width="180" height="393" alt="Simulator Screenshot - iPhone 17 Pro - 2026-07-17 at 15 56 38" src="https://github.com/user-attachments/assets/a2907bdd-9000-4159-af6d-c09bdf335925" />

### ピンチ（拡大縮小）

<img width="180" height="393" alt="Simulator Screenshot - iPhone 17 Pro - 2026-07-17 at 15 56 43" src="https://github.com/user-attachments/assets/92d3a351-12ea-4b36-969f-b5a9a29002ed" />

<img width="180" height="393" alt="Simulator Screenshot - iPhone 17 Pro - 2026-07-17 at 15 56 51" src="https://github.com/user-attachments/assets/e0364ab7-d3b0-447c-bde6-5ce9799a15dd" />

<img width="180" height="393" alt="Simulator Screenshot - iPhone 17 Pro - 2026-07-17 at 15 56 57" src="https://github.com/user-attachments/assets/6b8a4919-226a-41ea-9558-f24d6eed2299" />

### 回転

<img width="180" height="393" alt="Simulator Screenshot - iPhone 17 Pro - 2026-07-17 at 15 57 02" src="https://github.com/user-attachments/assets/487ea667-9c62-4408-9175-8bfe3dbe998e" />

<img width="180" height="393" alt="Simulator Screenshot - iPhone 17 Pro - 2026-07-17 at 15 57 09" src="https://github.com/user-attachments/assets/8b0ef7ca-9fbd-42dd-95c9-ed0ffcac86ad" />

<img width="180" height="393" alt="Simulator Screenshot - iPhone 17 Pro - 2026-07-17 at 15 57 11" src="https://github.com/user-attachments/assets/92a7bc28-2917-4675-9885-63ed84e2f91d" />

### 組み合わせ


<img width="180" height="393" alt="Simulator Screenshot - iPhone 17 Pro - 2026-07-17 at 15 57 23" src="https://github.com/user-attachments/assets/a01a8a6b-247a-4078-8e0d-a3fff214c13a" />

<img width="180" height="393" alt="Simulator Screenshot - iPhone 17 Pro - 2026-07-17 at 15 57 20" src="https://github.com/user-attachments/assets/60d5efbe-9bc0-42dd-a318-7e8043f7f003" />

<img width="180" height="393" alt="Simulator Screenshot - iPhone 17 Pro - 2026-07-17 at 15 57 17" src="https://github.com/user-attachments/assets/0e87f73a-4b32-4fc6-95a8-f73ca115ad6e" />



## コードの詳細解説

### 基本ジェスチャー（タップ、ロングプレス）

```swift
// MARK: - タップ & ロングプレス

struct TapDemoView: View {
    @State private var tapCount = 0
    @State private var backgroundColor: Color = .blue
    @State private var isPressed = false

    var body: some View {
        VStack(spacing: 30) {
            Text("タップ回数: \(tapCount)")
                .font(.title)

            // シングルタップ
            RoundedRectangle(cornerRadius: 16)
                .fill(backgroundColor)
                .frame(width: 200, height: 200)
                .overlay {
                    Text("タップしてね")
                        .foregroundStyle(.white)
                        .font(.headline)
                }
                .onTapGesture {
                    tapCount += 1
                    backgroundColor = Color(
                        hue: Double.random(in: 0...1),
                        saturation: 0.7,
                        brightness: 0.9
                    )
                }

            // ロングプレス
            Circle()
                .fill(isPressed ? .green : .orange)
                .frame(width: 120, height: 120)
                .scaleEffect(isPressed ? 1.3 : 1.0)
                .overlay {
                    Text(isPressed ? "成功!" : "長押し")
                        .foregroundStyle(.white)
                        .font(.headline)
                }
                .animation(.spring(duration: 0.3), value: isPressed)
                .onLongPressGesture(minimumDuration: 1.0) {
                    isPressed = true
                    DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
                        isPressed = false
                    }
                }
        }
        .navigationTitle("タップ & ロングプレス")
    }
}
```

**何をしているか：**

四角形をタップすると、タップ回数が増えて背景色がランダムに変わる。
円を1秒間長押しすると、色がオレンジから緑に変わり、サイズが大きくなる。

**なぜこう書くのか：**

onTapGestureとonLongPressGestureを使うことで、タップや長押しなど、ユーザーの操作に応じた処理を実行できる。
また、@Stateで値を管理することで、回数・色・サイズの変化を画面へ反映できる。

**もしこう書かなかったら：**

タップや長押しを検知できず、タップ回数が増えたり、色やサイズが変化したりしなくなる。

---

### ドラッグジェスチャーとオフセット管理

```swift
// MARK: - ドラッグ

struct DragDemoView: View {
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero

    var body: some View {
        VStack {
            Text("カードをドラッグしてみよう")
                .font(.headline)
                .padding()

            Spacer()

            RoundedRectangle(cornerRadius: 20)
                .fill(
                    LinearGradient(
                        colors: [.purple, .blue],
                        startPoint: .topLeading,
                        endPoint: .bottomTrailing
                    )
                )
                .frame(width: 200, height: 280)
                .shadow(radius: 8)
                .overlay {
                    VStack {
                        Image(systemName: "hand.draw")
                            .font(.system(size: 40))
                        Text("ドラッグ")
                            .font(.title3)
                    }
                    .foregroundStyle(.white)
                }
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = CGSize(
                                width: lastOffset.width + value.translation.width,
                                height: lastOffset.height + value.translation.height
                            )
                        }
                        .onEnded { _ in
                            lastOffset = offset
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    offset = .zero
                    lastOffset = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("ドラッグ")
    }
}
```

**何をしているか：**

カードを指でドラッグすると、指の動きに合わせてカードが移動する。
ドラッグを終えた位置が保存され、次にドラッグするとその位置から移動できる。
「リセット」ボタンを押すと、アニメーション付きで元の位置に戻る。

**なぜこう書くのか：**

DragGestureを使うことで、指の移動量を取得してカードの位置に反映できる。
また、lastOffsetに前回の移動位置を保存することで、ドラッグを繰り返しても位置がずれず、続きから動かせる。

**もしこう書かなかったら：**

カードをドラッグして移動できなくなったり、ドラッグを始めるたびに元の位置へ戻ったりする。
また、リセットボタンを押してもカードを初期位置に戻せなくなる。

---

### 拡大縮小と回転

```swift
// MARK: - ピンチ（拡大縮小）

struct MagnifyDemoView: View {
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0

    var body: some View {
        VStack {
            Text("ピンチで拡大縮小")
                .font(.headline)
                .padding()

            Text(String(format: "倍率: %.1fx", scale))
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            Image(systemName: "star.fill")
                .font(.system(size: 100))
                .foregroundStyle(.yellow)
                // タッチ判定を300×300の透明な領域に広げる
                .frame(width: 300, height: 300)
                .contentShape(Rectangle())
                .scaleEffect(scale)
                .gesture(
                    MagnifyGesture()
                        .onChanged { value in
                            scale = lastScale * value.magnification
                        }
                        .onEnded { _ in
                            lastScale = scale
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    scale = 1.0
                    lastScale = 1.0
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("ピンチ")
    }
}

// MARK: - 回転

struct RotateDemoView: View {
    @State private var angle: Angle = .zero
    @State private var lastAngle: Angle = .zero

    var body: some View {
        VStack {
            Text("2本指で回転")
                .font(.headline)
                .padding()

            Text(String(format: "角度: %.0f°", angle.degrees))
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            Image(systemName: "arrow.up")
                .font(.system(size: 80))
                .foregroundStyle(.red)
                // タッチ判定を300×300の透明な領域に広げる
                .frame(width: 300, height: 300)
                .contentShape(Rectangle())
                .rotationEffect(angle)
                .gesture(
                    RotateGesture()
                        .onChanged { value in
                            angle = lastAngle + value.rotation
                        }
                        .onEnded { _ in
                            lastAngle = angle
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    angle = .zero
                    lastAngle = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("回転")
    }
}
```

**何をしているか：**

### ピンチ（拡大・縮小）
星の画像を2本指でピンチすると、指の動きに合わせて画像を拡大・縮小する。
ピンチ操作を終えたときの倍率を保存するため、次の操作でも現在の大きさから続けて拡大・縮小できる。
「リセット」ボタンを押すと、画像の大きさが元に戻る。

### 回転
矢印の画像を2本指で回転させると、指の動きに合わせて画像が回転する。
回転操作を終えたときの角度を保存するため、次の操作でも現在の角度から続けて回転できる。
「リセット」ボタンを押すと、画像の向きが元に戻る。

**なぜこう書くのか：**

### ピンチ（拡大・縮小）
MagnifyGestureでピンチ操作の倍率を取得し、scaleEffectに反映することで画像を拡大・縮小できるようにするため。
また、lastScaleに操作後の倍率を保存することで、次の操作でも現在の大きさから続けて拡大・縮小できる。

### 回転
RotateGestureで2本指の回転角度を取得し、rotationEffectに反映することで画像を回転できるようにするため。
また、lastAngleに操作後の角度を保存することで、次の操作でも現在の角度から続けて回転できる。

**もしこう書かなかったら：**

### ピンチ（拡大・縮小）
ピンチ操作をしても画像を拡大・縮小できなくなる。
また、lastScaleで倍率を保存しない場合、ピンチ操作をやり直すたびに大きさの基準が元に戻り、不自然な動きになる。

### 回転
2本指で回転操作をしても画像が回転しなくなる。
また、lastAngleで角度を保存しない場合、回転操作をやり直すたびに角度の基準が元に戻り、不自然な動きになる。



---

### ジェスチャーの組み合わせとアニメーション

```swift
// MARK: - 組み合わせ

struct CombinedDemoView: View {
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0
    @State private var angle: Angle = .zero
    @State private var lastAngle: Angle = .zero

    var body: some View {
        VStack {
            Text("ドラッグ・ピンチ・回転を同時に")
                .font(.headline)
                .padding()

            Spacer()

            Image(systemName: "photo.artframe")
                .font(.system(size: 120))
                .foregroundStyle(.indigo)
                // タッチ判定を300×300の透明な領域に広げる
                .frame(width: 300, height: 300)
                .contentShape(Rectangle())
                .scaleEffect(scale)
                .rotationEffect(angle)
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = CGSize(
                                width: lastOffset.width + value.translation.width,
                                height: lastOffset.height + value.translation.height
                            )
                        }
                        .onEnded { _ in
                            lastOffset = offset
                        }
                )
                // 複数のジェスチャーを「同時に」効かせるには
                // .gesture を重ねるのではなく .simultaneousGesture を使う
                .simultaneousGesture(
                    MagnifyGesture()
                        .onChanged { value in
                            scale = lastScale * value.magnification
                        }
                        .onEnded { _ in
                            lastScale = scale
                        }
                )
                .simultaneousGesture(
                    RotateGesture()
                        .onChanged { value in
                            angle = lastAngle + value.rotation
                        }
                        .onEnded { _ in
                            lastAngle = angle
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    offset = .zero
                    lastOffset = .zero
                    scale = 1.0
                    lastScale = 1.0
                    angle = .zero
                    lastAngle = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("組み合わせ")
    }
}
```

**何をしているか：**

画像をドラッグして移動したり、2本指のピンチ操作で拡大・縮小したり、2本指で回転させたりできるようにしている。
それぞれの操作終了時の位置・倍率・角度を保存するため、次の操作でも現在の状態から続けて動かせる。
「リセット」ボタンを押すと、位置・大きさ・角度がすべて元の状態に戻る。

**なぜこう書くのか：**

DragGesture、MagnifyGesture、RotateGestureを組み合わせることで、1つの画像に複数の操作を設定するため。
また、.simultaneousGestureを使うことで、ドラッグ・拡大縮小・回転を同時に認識できるようにしている。
さらに、lastOffset、lastScale、lastAngleに操作後の状態を保存することで、操作をやり直しても現在の状態から続けられる。

**もしこう書かなかったら：**

画像を移動・拡大縮小・回転できなくなる。
また、.simultaneousGestureを使わずに複数の.gestureを設定すると、一部のジェスチャーが正しく反応しない場合がある。
操作後の状態を保存しない場合は、操作をやり直すたびに位置・大きさ・角度の基準が元に戻り、不自然な動きになる。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| DragGesture | 指やマウスで押したまま動かす、ドラッグ操作を検知する。| `DragGesture()　.onChanged { value in　offset = value.translation　}` |
| MagnifyGesture | 2本指を広げたり縮めたりする、ピンチ操作を検知する。 | `MagnifyGesture()　.onChanged { value in　scale = value.magnification　}` |
| RotateGesture | 2本指をひねる、回転操作を検知する。 | `RotateGesture() .onChanged { value in angle = value.rotation }` |
| startLocation | ドラッグを開始した位置を取得する。 | `DragGesture() .onEnded { value in print(value.startLocation) }` |
| predictedEndTranslation | ドラッグの速さや方向から、どこまで移動しそうかを予測する。 | `DragGesture()　.onEnded { value in offset = value.predictedEndTranslation }` |

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
