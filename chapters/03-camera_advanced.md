# 第3章：カメラの応用

> 執筆者：田中 吾錬
> 最終更新：2026-07-22

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

例：この章では、PhotosPickerでフォトライブラリから写真を選択し、UIImagePickerControllerでカメラ撮影した画像を扱う方法を学ぶ。具体的には非同期で画像データを読み込み、UIViewControllerRepresentableを使ってUIKitをSwiftUIに統合し、Coordinatorパターンを使ってカメラ機能と連携するアプリを題材にする。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

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


## コードの詳細解説

### PhotosPickerによる写真選択

```swift
```

**何をしているか：**
（この部分が果たしている役割を説明する）

**なぜこう書くのか：**
（別の書き方ではなく、この書き方が選ばれている理由を説明する）

**もしこう書かなかったら：**
（この部分を省略したり変えたりすると何が起きるか。実際に試した結果があればここに書く）

---

### 画像の非同期読み込み

```swift

```

**何をしているか：**


**なぜこう書くのか：**



**もしこう書かなかったら：**



---

### UIViewControllerRepresentableによるカメラ連携

```swift

```

**何をしているか：**


**なぜこう書くのか：**

**もしこう書かなかったら：**


---

### Coordinatorパターン

```swift

```

**何をしているか：**


**なぜこう書くのか：**


**もしこう書かなかったら：**


---

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
