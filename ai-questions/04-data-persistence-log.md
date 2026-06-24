# AI質問ログ：第4章 データの永続化

## 使用した生成AIツール

ChatGPT 5.5Thinking

## 質問と回答の記録

### Q1

**質問：**

swiftUIにおける@AppStorageについて詳しく教えてください。
イメージは本体へ保存なのですが、認識にズレはありますか???

**AIの回答の要点：**

正確に言うなら @AppStorage は 「アプリ本体に保存」ではなく、「UserDefaults に保存」
* アプリ内で使う小さな設定値を、アプリ専用の保存領域に自動保存してくれるもの

>@State
>→ 画面上だけの一時的なメモ
>
>@AppStorage
>→ アプリを閉じても残るメモ

**自分の理解：**

自動保存でdefault扱いになるから呼び出しやすいのではないかと思った。
アプリに保存されるため、データが消えにくいメリットもあることがわかった。

### Q2

**質問：**

```swift
var displayedMemos: [Memo] {
        if sortByFavorite {
            return memos.sorted { $0.isFavorite && !$1.isFavorite }
        }
        return memos
    }
```
はsortByFavoriteをお気に入り順で並べ替えているのはわかるのですが、$0,!$1がなにかよくわかりません。
Excelやspreadsheetで他のシートを参照する時につかう「!」と「!$1」同じ意味と推測するのでいいのでしょうか??
その場合$0はお気に入りがついている、!$1はついていないになるのでしょうか??

**AIの回答の要点：**

Excel / Spreadsheet の ! とは完全に別物

>Swift の ! はこの場合、
>
>否定
>「〜ではない」
>true を false にする、false を true にする
>
>という意味です。

今回の場合はお気に入りがついているものを前へ、ついていないものを後ろへ並べている。

**自分の理解：**

並べ替え方の指定で細かく判定しているわけではなく、ついているかいないかで判定していることがわかった。

### Q3

**質問：**

```swift
    var body: some View {
        NavigationStack {
            Group {
                if memos.isEmpty {
                    ContentUnavailableView(
                        "メモがありません",
                        systemImage: "note.text",
                        description: Text("右上の＋ボタンからメモを追加してください")
                    )
                } else {
                    List {
                        ForEach(displayedMemos) { memo in
                            NavigationLink(destination: MemoEditView(memo: memo)) {
                                MemoRow(memo: memo)
                            }
                        }
                        .onDelete(perform: deleteMemos)
                    }
                }
            }
            .navigationTitle(userName.isEmpty ? "メモ帳" : "\(userName)のメモ帳")
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button {
                        isShowingSettings = true
                    } label: {
                        Image(systemName: "gear")
                    }
                }
                ToolbarItem(placement: .topBarTrailing) {
                    Button {
                        isShowingAddSheet = true
                    } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $isShowingAddSheet) {
                MemoAddView()
            }
            .sheet(isPresented: $isShowingSettings) {
                SettingsView(userName: $userName, sortByFavorite: $sortByFavorite)
            }
        }
    }
```
>はメモ追加前のデフォルト画面であると思いますが、
>elseの分岐以降で、記入されたメモを追加していくのでしょうか??

**AIの回答の要点：**

elseに役割はすでに追加されたメモを一覧表示している

>+ ボタンを押す
MemoAddView() が表示される
そこでメモを入力・保存する
memos にメモが追加される
memos.isEmpty が false になる
else 側の List が表示される

という流れです。

**自分の理解：**

ifはメモがない画面でdefault表示している。
elseでは追加したメモを表示し並べている。

## 今日の質問を振り返って

（どんな質問が良い質問だったか。生成AIの回答で間違いや不正確な部分はあったか。次回はどんな質問をしてみたいか。）
