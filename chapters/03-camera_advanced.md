# 第3章：カメラの応用

> 執筆者：田中 吾錬
> 最終更新：2026-07-22

## この章で学ぶこと

本章では、PhotosPickerを使ってフォトライブラリから写真を選択し、Core Imageを用いて写真にフィルターを適用します。
また、フィルターごとの加工結果をサムネイルとして表示し、選択したフィルターを写真全体へ反映します。
加工した写真をフォトライブラリへ保存し、保存結果をアラートで表示する方法についても学びます。

## 模範コードの全体像

```swift
// ============================================
// 第3章（応用）：写真にフィルターをかけて保存するアプリ
// ============================================
// 選択した写真にCoreImageフィルターを適用し、
// フォトライブラリに保存する機能を追加します。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSPhotoLibraryAddUsageDescription
//     値: "加工した写真を保存するためにフォトライブラリを使用します"
// ============================================

import SwiftUI
import PhotosUI
import Photos
import CoreImage
import CoreImage.CIFilterBuiltins

// MARK: - フィルター定義

enum PhotoFilter: String, CaseIterable, Identifiable {
    case original = "オリジナル"
    case sepia = "セピア"
    case mono = "モノクロ"
    case chrome = "クローム"
    case fade = "フェード"
    case bloom = "ブルーム"

    var id: String { rawValue }

    func apply(to inputImage: CIImage, context: CIContext) -> CIImage? {
        switch self {
        case .original:
            return inputImage
        case .sepia:
            let filter = CIFilter.sepiaTone()
            filter.inputImage = inputImage
            filter.intensity = 0.8
            return filter.outputImage
        case .mono:
            let filter = CIFilter.photoEffectMono()
            filter.inputImage = inputImage
            return filter.outputImage
        case .chrome:
            let filter = CIFilter.photoEffectChrome()
            filter.inputImage = inputImage
            return filter.outputImage
        case .fade:
            let filter = CIFilter.photoEffectFade()
            filter.inputImage = inputImage
            return filter.outputImage
        case .bloom:
            let filter = CIFilter.bloom()
            filter.inputImage = inputImage
            filter.radius = 10
            filter.intensity = 0.8
            return filter.outputImage
        }
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var selectedItem: PhotosPickerItem?
    @State private var originalUIImage: UIImage?
    @State private var displayImage: Image?
    @State private var currentFilter: PhotoFilter = .original
    @State private var isSaving = false
    @State private var showSaveAlert = false
    @State private var saveMessage = ""

    private let context = CIContext()

    var body: some View {
        NavigationStack {
            VStack(spacing: 16) {
                // 画像表示
                if let image = displayImage {
                    image
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .frame(maxHeight: 350)
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                        .padding(.horizontal)
                } else {
                    placeholderView
                }

                // フィルター選択
                if originalUIImage != nil {
                    filterSelector
                }

                // ボタン群
                HStack(spacing: 16) {
                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選ぶ", systemImage: "photo")
                    }
                    .buttonStyle(.bordered)

                    if displayImage != nil {
                        Button {
                            saveFilteredImage()
                        } label: {
                            Label("保存", systemImage: "square.and.arrow.down")
                        }
                        .buttonStyle(.borderedProminent)
                        .disabled(isSaving)
                    }
                }
                .padding()

                Spacer()
            }
            .navigationTitle("フォトフィルター")
            .onChange(of: selectedItem) { _, newItem in
                Task { await loadOriginalImage(from: newItem) }
            }
            .onChange(of: currentFilter) { _, _ in
                applyFilter()
            }
            .alert("保存結果", isPresented: $showSaveAlert) {
                Button("OK") {}
            } message: {
                Text(saveMessage)
            }
        }
    }

    // MARK: - プレースホルダー

    private var placeholderView: some View {
        RoundedRectangle(cornerRadius: 12)
            .fill(.gray.opacity(0.1))
            .frame(height: 300)
            .overlay {
                VStack(spacing: 8) {
                    Image(systemName: "camera.filters")
                        .font(.system(size: 48))
                        .foregroundStyle(.gray)
                    Text("写真を選んでフィルターを試そう")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .padding(.horizontal)
    }

    // MARK: - フィルター選択UI

    private var filterSelector: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            HStack(spacing: 12) {
                ForEach(PhotoFilter.allCases) { filter in
                    VStack(spacing: 4) {
                        // フィルタープレビュー（サムネイル）
                        if let thumbnail = createThumbnail(filter: filter) {
                            Image(uiImage: thumbnail)
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                                .frame(width: 60, height: 60)
                                .clipShape(RoundedRectangle(cornerRadius: 8))
                                .overlay(
                                    RoundedRectangle(cornerRadius: 8)
                                        .stroke(
                                            currentFilter == filter ? Color.blue : Color.clear,
                                            lineWidth: 3
                                        )
                                )
                        }

                        Text(filter.rawValue)
                            .font(.caption2)
                            .foregroundStyle(
                                currentFilter == filter ? .blue : .secondary
                            )
                    }
                    .onTapGesture {
                        currentFilter = filter
                    }
                }
            }
            .padding(.horizontal)
        }
    }

    // MARK: - 画像処理

    func loadOriginalImage(from item: PhotosPickerItem?) async {
        guard let item = item else { return }

        do {
            if let data = try await item.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                originalUIImage = uiImage
                currentFilter = .original
                displayImage = Image(uiImage: uiImage)
            }
        } catch {
            print("画像読み込みエラー: \(error)")
        }
    }

    func applyFilter() {
        guard let uiImage = originalUIImage,
              let ciImage = CIImage(image: uiImage) else { return }

        guard let outputImage = currentFilter.apply(to: ciImage, context: context) else { return }

        if let cgImage = context.createCGImage(outputImage, from: ciImage.extent) {
            // 元画像の scale と imageOrientation を引き継がないと
            // EXIF回転情報が落ちて画像が上下反転して表示される
            displayImage = Image(uiImage: UIImage(cgImage: cgImage,
                                                  scale: uiImage.scale,
                                                  orientation: uiImage.imageOrientation))
        }
    }

    func createThumbnail(filter: PhotoFilter) -> UIImage? {
        guard let uiImage = originalUIImage,
              let ciImage = CIImage(image: uiImage) else { return nil }

        guard let output = filter.apply(to: ciImage, context: context) else { return nil }

        if let cgImage = context.createCGImage(output, from: ciImage.extent) {
            return UIImage(cgImage: cgImage,
                           scale: uiImage.scale,
                           orientation: uiImage.imageOrientation)
        }
        return nil
    }

    func saveFilteredImage() {
        guard let uiImage = originalUIImage,
              let ciImage = CIImage(image: uiImage),
              let output = currentFilter.apply(to: ciImage, context: context),
              let cgImage = context.createCGImage(output, from: ciImage.extent) else { return }

        // PHAssetChangeRequest は imageOrientation を尊重して保存するため、
        // ここで orientation を渡しておかないと保存ファイルも上下反転する
        let finalImage = UIImage(cgImage: cgImage,
                                 scale: uiImage.scale,
                                 orientation: uiImage.imageOrientation)
        isSaving = true

        // PHPhotoLibrary を使うと、保存の成否をコールバックで受け取れる
        PHPhotoLibrary.shared().performChanges {
            PHAssetChangeRequest.creationRequestForAsset(from: finalImage)
        } completionHandler: { success, error in
            DispatchQueue.main.async {
                isSaving = false
                if success {
                    saveMessage = "写真を保存しました"
                } else {
                    saveMessage = "保存に失敗しました\n\(error?.localizedDescription ?? "原因不明のエラー")"
                }
                showSaveAlert = true
            }
        }
    }
}

#Preview {
    ContentView()
}
```

**このアプリは何をするものか：**

フォトライブラリから写真を選択し、選択した写真にフィルターを適用するアプリです。
写真を選択すると、オリジナル、セピア、モノクロ、クローム、フェード、ブルームの加工結果がサムネイルとして表示されます。
サムネイルをタップすると、そのフィルターが画面上の写真に反映されます。
加工した写真は保存ボタンを押すことで、フォトライブラリへ保存できます。


## コードの詳細解説

### フィルターの定義と適用

```swift
enum PhotoFilter: String, CaseIterable, Identifiable {
    case original = "オリジナル"
    case sepia = "セピア"
    case mono = "モノクロ"
    case chrome = "クローム"
    case fade = "フェード"
    case bloom = "ブルーム"

    var id: String { rawValue }

    func apply(to inputImage: CIImage, context: CIContext) -> CIImage? {
        switch self {
        case .original:
            return inputImage

        case .sepia:
            let filter = CIFilter.sepiaTone()
            filter.inputImage = inputImage
            filter.intensity = 0.8
            return filter.outputImage

        case .mono:
            let filter = CIFilter.photoEffectMono()
            filter.inputImage = inputImage
            return filter.outputImage

        case .chrome:
            let filter = CIFilter.photoEffectChrome()
            filter.inputImage = inputImage
            return filter.outputImage

        case .fade:
            let filter = CIFilter.photoEffectFade()
            filter.inputImage = inputImage
            return filter.outputImage

        case .bloom:
            let filter = CIFilter.bloom()
            filter.inputImage = inputImage
            filter.radius = 10
            filter.intensity = 0.8
            return filter.outputImage
        }
    }
}
```

**何をしているか：**

アプリで使用するフィルターの種類と、フィルターを画像へ適用する処理を定義している。
PhotoFilterには、オリジナル、セピア、モノクロ、クローム、フェード、ブルームの6種類が登録されている。
apply関数では、選択されたフィルターに応じてCore Imageのフィルターを作成し、引数として受け取った画像へ適用している。

```swift
let filter = CIFilter.sepiaTone()
filter.inputImage = inputImage
filter.intensity = 0.8
```

inputImageに加工対象の画像を設定し、intensityでフィルターの強さを設定している。

**なぜこう書くのか：**

フィルターの種類と加工処理を1つのenumにまとめることで、どのフィルターを選んだときにどの処理を行うのか分かりやすくなる。
CaseIterableを付けることで、登録されているすべてのフィルターを次のように取得できる。

```swift
PhotoFilter.allCases
```

これにより、フィルターごとのサムネイルをForEachで自動的に作成できる。
また、Identifiableに準拠することで、SwiftUIがそれぞれのフィルターを区別できる。

```swift
var id: String { rawValue }
```

では、フィルターの表示名を識別用のIDとして使用している。


**もしこう書かなかったら：**

フィルターごとに別々の関数を作成する必要があり、同じような処理が増えてしまう。

```swift
func applySepia() {
    // セピアを適用する処理
}
func applyMono() {
    // モノクロを適用する処理
}
```

また、CaseIterableがなければPhotoFilter.allCasesを使用できないため、表示するフィルターを個別に指定する必要がある。

---

### PhotosPickerによる写真選択と非同期読み込み

```swift
@State private var selectedItem: PhotosPickerItem?
@State private var originalUIImage: UIImage?
@State private var displayImage: Image?

PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("写真を選ぶ", systemImage: "photo")
}
.buttonStyle(.bordered)

.onChange(of: selectedItem) { _, newItem in
    Task {
        await loadOriginalImage(from: newItem)
    }
}

func loadOriginalImage(from item: PhotosPickerItem?) async {
    guard let item = item else {
        return
    }

    do {
        if let data = try await item.loadTransferable(type: Data.self),
           let uiImage = UIImage(data: data) {
            originalUIImage = uiImage
            currentFilter = .original
            displayImage = Image(uiImage: uiImage)
        }
    } catch {
        print("画像読み込みエラー: \(error)")
    }
}

```

**何をしているか：**

PhotosPickerを使用して、フォトライブラリから写真を選択している。

```swift
matching: .images
```

を指定することで、選択できる対象を画像のみに限定している。
写真が選択されると、selectedItemの変化をonChangeで検知し、loadOriginalImageを実行する。
loadOriginalImageでは、選択された写真をDataとして読み込み、UIImageへ変換している。
読み込んだ画像は、元画像としてoriginalUIImageへ保存し、画面表示用のdisplayImageにも設定している。


**なぜこう書くのか：**

PhotosPickerItemは、選択された画像そのものではなく、写真を読み込むための情報を持つ型である。
そのため、次の処理で画像データを読み込む必要がある。
```swift
try await item.loadTransferable(type: Data.self)
```
画像の読み込みには時間がかかる場合があるため、asyncとawaitを使って非同期で処理している。
また、originalUIImageとdisplayImageを分けて保存することで、フィルターを変更するたびに元画像から加工し直せる。


**もしこう書かなかったら：**

onChangeがなければ、ユーザーが写真を選択しても画像の読み込み処理が実行されない。
また、Taskを使わずに非同期関数を呼び出すことはできない。
元画像を保存せず、加工後の画像だけを使い続けると、フィルターを変更するたびに加工が重ねられてしまう。
例えば、セピア加工後の画像へさらにモノクロ加工を適用するような状態になる。

---

### Core Imageによる画像加工

```swift
@State private var currentFilter: PhotoFilter = .original
private let context = CIContext()

.onChange(of: currentFilter) { _, _ in
    applyFilter()
}
func applyFilter() {
    guard let uiImage = originalUIImage,
          let ciImage = CIImage(image: uiImage) else {
        return
    }
    guard let outputImage = currentFilter.apply(
        to: ciImage,
        context: context
    ) else {
        return
    }
    if let cgImage = context.createCGImage(
        outputImage,
        from: ciImage.extent
    ) {
        displayImage = Image(
            uiImage: UIImage(
                cgImage: cgImage,
                scale: uiImage.scale,
                orientation: uiImage.imageOrientation
            )
        )
    }
}

```

**何をしているか：**

現在選択されているフィルターが変化したときに、元画像へフィルターを適用している。
まず、UIImageとして保存されている元画像を、Core Imageで扱えるCIImageへ変換する。
```swift
let ciImage = CIImage(image: uiImage)
```

次に、現在選択されているフィルターのapply関数を呼び出し、加工後の画像を取得する。
```swift
currentFilter.apply(to: ciImage, context: context)
```
その後、加工したCIImageをCGImageへ変換し、最終的にSwiftUIのImageとして画面へ表示している。


**なぜこう書くのか：**

Core Imageのフィルターは、SwiftUIのImageやUIImageへ直接適用するのではなく、CIImageに対して適用する。

そのため、画像を次の順番で変換している。

    UIImage
    ↓
    CIImage
    ↓
    フィルター適用
    ↓
    CGImage
    ↓
    UIImage
    ↓
    SwiftUIのImage

また、CIContextは、Core Imageで加工した画像を実際の画像として描画するために使用する

```swift
context.createCGImage(outputImage, from: ciImage.extent)
```

画像をUIImageへ戻す際に、元画像のscaleとimageOrientationを引き継いでいる。

```swift
scale: uiImage.scale
orientation: uiImage.imageOrientation
```

これにより、元画像の向きを維持したまま表示できる。

**もしこう書かなかったら：**

CIImageへ変換しなければ、Core Imageのフィルターを適用できない。
また、次のように向きの情報を渡さずにUIImageを作成すると、写真によっては回転したり、上下が反転したりする可能性がある。

```swift
UIImage(cgImage: cgImage)
```

onChangeがなければ、フィルターを選択してもapplyFilterが実行されず、表示画像が更新されない。

---

### 

```swift
private var filterSelector: some View {
    ScrollView(.horizontal, showsIndicators: false) {
        HStack(spacing: 12) {
            ForEach(PhotoFilter.allCases) { filter in
                VStack(spacing: 4) {
                    if let thumbnail = createThumbnail(filter: filter) {
                        Image(uiImage: thumbnail)
                            .resizable()
                            .aspectRatio(contentMode: .fill)
                            .frame(width: 60, height: 60)
                            .clipShape(RoundedRectangle(cornerRadius: 8))
                            .overlay(
                                RoundedRectangle(cornerRadius: 8)
                                    .stroke(
                                        currentFilter == filter
                                            ? Color.blue
                                            : Color.clear,
                                        lineWidth: 3
                                    )
                            )
                    }

                    Text(filter.rawValue)
                        .font(.caption2)
                        .foregroundStyle(
                            currentFilter == filter
                                ? .blue
                                : .secondary
                        )
                }
                .onTapGesture {
                    currentFilter = filter
                }
            }
        }
        .padding(.horizontal)
    }
}

func createThumbnail(filter: PhotoFilter) -> UIImage? {
    guard let uiImage = originalUIImage,
          let ciImage = CIImage(image: uiImage) else {
        return nil
    }

    guard let output = filter.apply(
        to: ciImage,
        context: context
    ) else {
        return nil
    }

    if let cgImage = context.createCGImage(
        output,
        from: ciImage.extent
    ) {
        return UIImage(
            cgImage: cgImage,
            scale: uiImage.scale,
            orientation: uiImage.imageOrientation
        )
    }

    return nil
}
```

すべてのフィルターを横方向に並べ、フィルターごとの加工結果をサムネイルとして表示している。

```swift
ForEach(PhotoFilter.allCases)
```

によって、PhotoFilterに登録されているフィルターを1件ずつ表示している。
createThumbnailでは、元画像に指定されたフィルターを適用し、サムネイル用のUIImageを作成している。
フィルターの項目をタップすると、そのフィルターをcurrentFilterへ設定する。

```swift
.onTapGesture {
    currentFilter = filter
}
```

選択中のフィルターには青色の枠線と文字色を付けている。



**何をしているか：**

すべてのフィルターを横方向に並べ、フィルターごとの加工結果をサムネイルとして表示している。

```swift
ForEach(PhotoFilter.allCases)
```

によって、PhotoFilterに登録されているフィルターを1件ずつ表示している。
createThumbnailでは、元画像に指定されたフィルターを適用し、サムネイル用のUIImageを作成している。
フィルターの項目をタップすると、そのフィルターをcurrentFilterへ設定する。

```swift
.onTapGesture {
    currentFilter = filter
}
```

選択中のフィルターには青色の枠線と文字色を付けている。

**なぜこう書くのか：**

フィルター名だけでは、どのような加工結果になるのか分かりにくい。
そのため、実際の元画像にフィルターを適用したサムネイルを表示している。
また、PhotoFilter.allCasesとForEachを使うことで、フィルターごとに同じUIを個別で書かずに済む。
currentFilterを基準にすることで、選択中のフィルターと表示画像、枠線の見た目を連動できる。


**もしこう書かなかったら：**

サムネイルを表示しなければ、ユーザーはフィルター名だけで加工結果を判断する必要がある。
また、フィルターごとに次のようなボタンを個別に作成する必要がある。


```swift
Button("セピア") {
    currentFilter = .sepia
}

Button("モノクロ") {
    currentFilter = .mono
}
```

選択中の枠線や文字色がなければ、どのフィルターが選ばれているのか分かりにくくなる。

---

### 加工した写真の保存

```swift
Button {
    saveFilteredImage()
} label: {
    Label("保存", systemImage: "square.and.arrow.down")
}
.buttonStyle(.borderedProminent)
.disabled(isSaving)

func saveFilteredImage() {
    guard let uiImage = originalUIImage,
          let ciImage = CIImage(image: uiImage),
          let output = currentFilter.apply(
              to: ciImage,
              context: context
          ),
          let cgImage = context.createCGImage(
              output,
              from: ciImage.extent
          ) else {
        return
    }

    let finalImage = UIImage(
        cgImage: cgImage,
        scale: uiImage.scale,
        orientation: uiImage.imageOrientation
    )

    isSaving = true

    PHPhotoLibrary.shared().performChanges {
        PHAssetChangeRequest.creationRequestForAsset(
            from: finalImage
        )
    } completionHandler: { success, error in
        DispatchQueue.main.async {
            isSaving = false

            if success {
                saveMessage = "写真を保存しました"
            } else {
                saveMessage =
                    "保存に失敗しました\n\(error?.localizedDescription ?? "原因不明のエラー")"
            }

            showSaveAlert = true
        }
    }
}

.alert("保存結果", isPresented: $showSaveAlert) {
    Button("OK") {}
} message: {
    Text(saveMessage)
}

```

**何をしているか：**

現在選択されているフィルターを元画像へ適用し、加工後の画像をフォトライブラリへ保存している。
保存前に、元画像をCIImageへ変換し、選択中のフィルターを適用している。
加工後の画像はUIImageへ変換し、PHPhotoLibraryを使って保存する。
保存が完了すると、成功したか失敗したかを確認し、その結果をアラートで表示している。

**なぜこう書くのか：**

画面に表示しているdisplayImageはSwiftUIのImage型であり、そのままフォトライブラリへ保存する処理には使いにくい。
そのため、保存時にも元画像からフィルター処理を行い、保存可能なUIImageを作成している。
PHPhotoLibrary.shared().performChangesを使うことで、画像を保存するだけでなく、保存の成功・失敗を受け取れる。

```swift
completionHandler: { success, error in
```

保存完了後の処理では@Stateを変更して画面を更新するため、メインスレッドへ処理を戻している。

```swift
DispatchQueue.main.async
```

また、保存中はisSavingをtrueにして、保存ボタンを押せないようにしている。

```
.disabled(isSaving)
```

これにより、同じ画像を連続して保存してしまうことを防げる。

**もしこう書かなかったら：**

PHPhotoLibraryを使わなければ、保存に成功したか失敗したかを確認しにくくなる。
DispatchQueue.main.asyncを使わずに、保存処理の完了後に直接@Stateを変更すると、画面更新に関する警告や不具合が発生する可能性がある。
また、isSavingによる制御がなければ、保存処理中に保存ボタンを何度も押せてしまい、同じ写真が複数保存される可能性がある。
Info.plistに次の項目を追加していない場合、フォトライブラリへの保存許可を正常に求められない。

    NSPhotoLibraryAddUsageDescription

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `async` | 非同期処理 | `PoadImage(from item: PhotosPickerItem?) async { guard let item = item else { return }` |
| `do-catch` | 失敗する可能性のある場所をエラーが起きても実行できる様に安全に処理する設計 | `do{ if (略） } catch { (エラー処理）{ ` |
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

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
