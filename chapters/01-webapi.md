# 第1章：WebAPIの基本

> 執筆者：田中 吾錬
> 最終更新：20260415

## この章で学ぶこと

>本章は、API（WebAPI）を、用いたアプリの復習を行う。
>今回はiTunes Search API を用いて音楽の検索、ジャケット写真及び、歌手名をリスト形式で表示するアプリ。

## 模範コードの全体像

```swift
import SwiftUI

// MARK: - データモデル

struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var songs: [Song] = []
    @State private var searchText: String = ""
    @State private var isLoading: Bool = false

    var body: some View {
        NavigationStack {
            VStack {
                // 検索バー
                HStack {
                    TextField("アーティスト名を入力", text: $searchText)
                        .textFieldStyle(.roundedBorder)

                    Button("検索") {
                        Task {
                            await searchMusic()
                        }
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(searchText.isEmpty)
                }
                .padding(.horizontal)

                // 検索結果リスト
                if isLoading {
                    ProgressView("検索中...")
                        .padding()
                    Spacer()
                } else if songs.isEmpty {
                    ContentUnavailableView(
                        "曲を検索してみよう",
                        systemImage: "music.note",
                        description: Text("アーティスト名を入力して検索ボタンを押してください")
                    )
                } else {
                    List(songs) { song in
                        SongRow(song: song)
                    }
                }
            }
            .navigationTitle("Music Search")
        }
    }

    // MARK: - API通信

    func searchMusic() async {
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else { return }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"
        // limitは検索結果の表示数

        guard let url = URL(string: urlString) else { return }

        isLoading = true

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)
            songs = response.results
        } catch {
            print("エラー: \(error.localizedDescription)")
            songs = []
        }

        isLoading = false
    }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                Color.gray.opacity(0.3)
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(song.trackName)
                    .font(.headline)
                    .lineLimit(1)

                Text(song.artistName)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }
        }
        .padding(.vertical, 4)
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

TextField にテキストを挿入し、音楽を検索する。
>
<img width="240" height="520" alt="Simulator Screenshot - iPhone 17 - 2026-04-15 at 15 57 24" src="https://github.com/user-attachments/assets/7e04e3c0-f0e4-4b5d-8902-24d3799e6c7e" />

>
> 
```swift
let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"
```
にて
```swift
limit=25
```
と指定されているので最大25件まで表示される。

この数字を変更すると表示数を変えることができる。
```swift
limit=1
```
のように変更すると
>
<img width="240" height="520" alt="Simulator Screenshot - iPhone 17 - 2026-04-15 at 16 00 39" src="https://github.com/user-attachments/assets/95473cb2-0dba-45ac-bd70-aff27f2733cb" />

のように1件のみの表示が行える。

また、アーティスト名でなく曲名での検索も可能である。
>
<img width="240" height="520" alt="Simulator Screenshot - iPhone 17 - 2026-04-15 at 16 06 57" src="https://github.com/user-attachments/assets/ef73cc57-a613-4a61-80e7-21b1028683be" />


>
（アプリの動作を自分の言葉で説明する。スクリーンショットを貼ってもよい。）

## コードの詳細解説

### データモデル（Codable構造体）

```swift
// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
（この部分が果たしている役割を説明する）

**なぜこう書くのか：**
（別の書き方ではなく、この書き方が選ばれている理由を説明する）

**もしこう書かなかったら：**
（この部分を省略したり変えたりすると何が起きるか。実際に試した結果があればここに書く）

---

### API通信の処理

```swift
func searchMusic() async {
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else { return }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"
        // limitは検索結果の表示数

        guard let url = URL(string: urlString) else { return }

        isLoading = true

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)
            songs = response.results
        } catch {
            print("エラー: \(error.localizedDescription)")
            songs = []
        }

        isLoading = false
    }
```

**何をしているか：**

**なぜこう書くのか：**

**もしこう書かなかったら：**

---

### ビューの構成

```swift
struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                Color.gray.opacity(0.3)
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(song.trackName)
                    .font(.headline)
                    .lineLimit(1)

                Text(song.artistName)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }
        }
        .padding(.vertical, 4)
    }
}
```

**何をしているか：**
曲名、歌手名の表示及びJSONで要求されたURL画像を拾ってきて、アルバム写真の表示を行っている。
また、画像が読み超えない間の待機時間には四角で、非表示を表している。
そして、拾ってきた画像をサイズ超世をしたのちサイズを合わせて設置している。

**なぜこう書くのか：**

綺麗なリストを制作するし、指示を簡素化するため。
一個の箱を作っていて、JSONを持ってくるURLにある**limit=25**の数によって箱を作る個数を決める。
データの数によって決まると言うこと。

**もしこう書かなかったら：**
表示件数、表示情報への配慮ができない。

曲は日々増え続ける点、そして**limit=25**のようにしているが、
これは最大表示件数になるので仮にした歌手が100曲あっても25件までしか表示させない処理。
そして10曲しかなければ10曲しか返されない。

limitで表示数を参照（出す個数まで）しているのが、
今回のように書かないと、[1曲目の曲名,歌手名、アルバム],[2曲目の曲名,歌手名、アルバム],[3曲目の曲名,歌手名、アルバム]のように
表示したい件数分codeを書く必要が出てくる。
またこのような場合は、 **「25」** と言う箱数があるのにも関わらず表示できるのが **「10」** しかない場合
無限に存在しない **「11」** 番目を探そうとしてエラーになる。



---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Codable` | JSONデータとSwiftの構造体を相互変換するプロトコル | `struct Song: Codable { ... }` |
| 例：`async/await` | 非同期処理を同期的に書ける構文 | `let data = try await URLSession.shared.data(from: url)` |
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
