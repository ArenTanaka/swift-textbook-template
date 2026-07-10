# AI質問ログ：第5章 機能統合の実践

## 使用した生成AIツール

（例：ChatGPT 無料版 / Claude 無料版 / Gemini など）

## 質問と回答の記録

### Q1

**質問：**

```swift
NavigationStack {
            Form {
                Section("写真") {
                    if let image = previewImage {
                        image
                            .resizable()
                            .aspectRatio(contentMode: .fit)
                            .frame(maxHeight: 200)
                            .clipShape(RoundedRectangle(cornerRadius: 8))
                    }

                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選択", systemImage: "photo")
                    }
                }

                Section("情報") {
                    TextField("タイトル", text: $title)
                    TextField("メモ（任意）", text: $memo, axis: .vertical)
                        .lineLimit(3...6)
                }

                Section("位置情報") {
                    if let location = locationManager.currentLocation {
                        Text("緯度: \(location.latitude, specifier: "%.4f")")
                        Text("経度: \(location.longitude, specifier: "%.4f")")
                    } else {
                        Text("位置情報を取得中...")
                            .foregroundStyle(.secondary)
                    }
                }
            }
            .navigationTitle("新しい記録")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        saveRecord()
                    }
                    .disabled(title.isEmpty || locationManager.currentLocation == nil)
                }
            }
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    if let data = try? await newItem?.loadTransferable(type: Data.self) {
                        selectedImageData = data
                        if let uiImage = UIImage(data: data) {
                            previewImage = Image(uiImage: uiImage)
                        }
                    }
                }
            }
        }
```

場所が分からないと画像あっても決定できないのはこのcodeが関係ありますか?


**AIの回答の要点：**

このコードが関係している。
```swift
.disabled(title.isEmpty || locationManager.currentLocation == nil)
```
保存ボタンは、次のどちらかに当てはまると押せなくなる。

```swift
title.isEmpty
```
タイトルが未入力。
```swift
locationManager.currentLocation == nil
```
位置情報を取得できていない。

つまり、画像を選択できていても、位置情報が取得できていなければ保存ボタンは無効です。

位置情報なしで保存可能に変更する場合は、saveRecord()側でも緯度・経度を必須としていないか確認が必要です。


**自分の理解：**
（回答を受けて自分がどう理解したか。納得できたか、さらに疑問が生まれたか）

### Q2

**質問：**

**AIの回答の要点：**

**自分の理解：**

### Q3

**質問：**

**AIの回答の要点：**

**自分の理解：**

（質問は何個でも追加してください。多ければ多いほど良いです。）

## 今日の質問を振り返って

（どんな質問が良い質問だったか。生成AIの回答で間違いや不正確な部分はあったか。次回はどんな質問をしてみたいか。）
