# 第8章：ウィジェット

> 執筆者：田中 吾錬
> 最終更新：2026-07-29

## この章で学ぶこと

この章では、WidgetKitを使って、ホーム画面に日替わりの名言を表示するウィジェットの作り方を学ぶ。
TimelineProviderによる表示内容の更新、ウィジェットサイズに応じたレイアウトの切り替え、
メインアプリとウィジェットでデータを共有する方法について理解する。

## 模範コードの全体像

```swift
// ============================================
// 第8章：ウィジェットを作る
// ============================================
// 今日の名言をホーム画面に表示するウィジェットです。
// メインアプリとウィジェットの両方のコードを含みます。
//
// 【セットアップ手順】※最新のXcodeのウィジェットテンプレートに対応
//
// 1. Xcodeで File → New → Target → Widget Extension を選ぶ
//    ・名前（Product Name）は「QuoteWidget」にする
//    ・「Include Live Activity」と「Include Configuration App Intent」は
//      どちらもチェックを外す（後で消すファイルが減る。外せなくても手順2で消すのでOK）
//
// 2. 追加すると、プロジェクトに「QuoteWidget」フォルダができ、中に複数の
//    Swift ファイルが自動生成される。今回の名言ウィジェットに必要なのは
//    QuoteWidget.swift の 1本だけ。残りの QuoteWidget〜.swift は全部削除する
//    （ファイルを選び 右クリック → Delete → 「Move to Trash」）。
//    消す対象の例（生成された分だけ）：
//      ・QuoteWidgetBundle.swift      … @main が入っている「入口」。複数ウィジェットを
//                                        束ねる役割だが、今回はウィジェット1個なので不要
//      ・QuoteWidgetControl.swift     … コントロールセンター用ウィジェット（不要）
//      ・QuoteWidgetLiveActivity.swift … ライブアクティビティ用（生成された場合のみ）
//    → QuoteWidget フォルダに残す Swift ファイルは QuoteWidget.swift だけ、が目印。
//
// 3. データ型を共有する（Quote と QuoteStore の "両方"）：
//    ・新規ファイル QuoteStore.swift を作り、Quote 構造体と QuoteStore 構造体の
//      両方をそこに移す（ウィジェット側は Quote 型も使うため、片方だけでは
//      「Cannot find 'Quote' in scope」になる）
//    ・この ContentView.swift 側からは Quote と QuoteStore の定義を削除する
//      （両方に同じ定義が残ると「Invalid redeclaration」になる）
//    ・QuoteStore.swift を選び、右側インスペクタの Target Membership で
//      「メインアプリ」と「QuoteWidget Extension」の両方にチェックを入れる
//
// 4. ウィジェット本体を置き換える：
//    ・手順2で残した QuoteWidget.swift を開き、中身を「すべて削除」してから、
//      このファイル末尾の「ウィジェット側のコード」を貼り付ける
//      （先頭と末尾の /* */ は外す）
//    ・貼り付けるコードには @main の付いた QuoteWidget が入っている。手順2で
//      @main を持つ QuoteWidgetBundle.swift を削除済みなので、これがエクステン
//      ションで唯一の @main になる
//    ・QuoteWidgetBundle.swift を消し忘れると @main が二重になり
//      「'main' attribute can only apply to one type」エラーになる
//
// ※ App Group の設定は不要です（Quote / QuoteStore は静的データのため、
//   UserDefaults や共有ファイルでのデータ受け渡しを行いません）
// ============================================

// ============================================
// ■ メインアプリ側のコード（ContentView.swift）
// ============================================

import SwiftUI

// MARK: - 名言データ（アプリとウィジェットで共有）
//
//struct Quote: Identifiable, Codable {
//    let id: Int
//    let text: String
//    let author: String
//}
//
//struct QuoteStore {
//    static let quotes: [Quote] = [
//        Quote(id: 1, text: "為せば成る、為さねば成らぬ何事も", author: "上杉鷹山"),
//        Quote(id: 2, text: "千里の道も一歩から", author: "老子"),
//        Quote(id: 3, text: "継続は力なり", author: "ことわざ"),
//        Quote(id: 4, text: "失敗は成功のもと", author: "ことわざ"),
//        Quote(id: 5, text: "知ることは愛することの始まりである", author: "ことわざ"),
//        Quote(id: 6, text: "学びて思わざれば則ち罔し", author: "孔子"),
//        Quote(id: 7, text: "過ちて改めざる、是を過ちと謂う", author: "孔子"),
//    ]
//
//    static func todaysQuote() -> Quote {
//        let dayOfYear = Calendar.current.ordinality(of: .day, in: .year, for: Date()) ?? 0
//        let index = dayOfYear % quotes.count
//        return quotes[index]
//    }
//}

// MARK: - メインアプリのContentView

struct ContentView: View {
    let todaysQuote = QuoteStore.todaysQuote()
    @State private var allQuotes = QuoteStore.quotes

    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {
                // 今日の名言（ハイライト）
                VStack(spacing: 16) {
                    Text("今日の名言")
                        .font(.caption)
                        .foregroundStyle(.secondary)

                    Text("「\(todaysQuote.text)」")
                        .font(.title2)
                        .bold()
                        .multilineTextAlignment(.center)

                    Text("— \(todaysQuote.author)")
                        .font(.subheadline)
                        .foregroundStyle(.secondary)
                }
                .padding(24)
                .frame(maxWidth: .infinity)
                .background(
                    RoundedRectangle(cornerRadius: 16)
                        .fill(.blue.opacity(0.08))
                )
                .padding(.horizontal)

                // 全名言リスト
                List(allQuotes) { quote in
                    VStack(alignment: .leading, spacing: 4) {
                        Text(quote.text)
                            .font(.body)
                        Text("— \(quote.author)")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                    .padding(.vertical, 4)
                }
            }
            .navigationTitle("名言集")
        }
    }
}

#Preview {
    ContentView()
}


// ============================================
// ■ ウィジェット側のコード（自動生成された QuoteWidget.swift を "全置換"）
// ============================================
// ※ 下の /* ... */ を外し、自動生成ファイルの中身を全部消してから貼り付けます。
// ※ Quote と QuoteStore は手順3で QuoteStore.swift に移し、両ターゲットの
//    Target Membership に入れてあるので、ここでは再定義しません。
// ============================================

/*
import WidgetKit
import SwiftUI

// MARK: - タイムラインエントリ

struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}

// MARK: - タイムラインプロバイダ

struct QuoteProvider: TimelineProvider {
    // プレースホルダー（読み込み中の仮表示）
    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(
            date: Date(),
            quote: Quote(id: 0, text: "読み込み中...", author: "")
        )
    }

    // スナップショット（ウィジェットギャラリーでのプレビュー）
    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
        let entry = QuoteEntry(
            date: Date(),
            quote: QuoteStore.todaysQuote()
        )
        completion(entry)
    }

    // タイムライン（実際のウィジェット更新スケジュール）
    func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) {
        let currentDate = Date()
        let quote = QuoteStore.todaysQuote()
        let entry = QuoteEntry(date: currentDate, quote: quote)

        // 次の日の0時にウィジェットを更新
        let tomorrow = Calendar.current.startOfDay(
            for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
        )

        let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
        completion(timeline)
    }
}

// MARK: - ウィジェットのビュー

struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget
        case .systemMedium:
            mediumWidget
        default:
            mediumWidget
        }
    }

    // 小サイズ
    var smallWidget: some View {
        VStack(spacing: 4) {
            Image(systemName: "quote.opening")
                .font(.caption)
                .foregroundStyle(.blue)

            Text(entry.quote.text)
                .font(.caption)
                .bold()
                .multilineTextAlignment(.center)
                .lineLimit(3)

            Text(entry.quote.author)
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
        .padding(12)
    }

    // 中サイズ
    var mediumWidget: some View {
        HStack(spacing: 16) {
            Image(systemName: "quote.opening")
                .font(.title)
                .foregroundStyle(.blue)

            VStack(alignment: .leading, spacing: 4) {
                Text("今日の名言")
                    .font(.caption2)
                    .foregroundStyle(.secondary)

                Text(entry.quote.text)
                    .font(.subheadline)
                    .bold()

                Text("— \(entry.quote.author)")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }

            Spacer()
        }
        .padding()
    }
}

// MARK: - ウィジェット定義

@main
struct QuoteWidget: Widget {
    let kind: String = "QuoteWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: QuoteProvider()) { entry in
            QuoteWidgetEntryView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("今日の名言")
        .description("日替わりで名言を表示します")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}

// MARK: - プレビュー

#Preview(as: .systemMedium) {
    QuoteWidget()
} timeline: {
    QuoteEntry(date: .now, quote: QuoteStore.todaysQuote())
}
*/

```

**このアプリは何をするものか：**

このアプリは、複数の名言を一覧で確認できる名言集アプリである。メインアプリでは、その日の名言を画面上部に大きく表示し、その下に登録されているすべての名言を一覧表示する。

また、ホーム画面にウィジェットを追加すると、アプリを開かなくても今日の名言を確認できる。名言は日付に応じて自動的に切り替わり、小サイズと中サイズのウィジェットに対応している。

## コードの詳細解説

### TimelineProviderの仕組み

```swift
// MARK: - タイムラインプロバイダ

struct QuoteProvider: TimelineProvider {
    // プレースホルダー（読み込み中の仮表示）
    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(
            date: Date(),
            quote: Quote(id: 0, text: "読み込み中...", author: "")
        )
    }

    // スナップショット（ウィジェットギャラリーでのプレビュー）
    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
        let entry = QuoteEntry(
            date: Date(),
            quote: QuoteStore.todaysQuote()
        )
        completion(entry)
    }

    // タイムライン（実際のウィジェット更新スケジュール）
    func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) {
        let currentDate = Date()
        let quote = QuoteStore.todaysQuote()
        let entry = QuoteEntry(date: currentDate, quote: quote)

        // 次の日の0時にウィジェットを更新
        let tomorrow = Calendar.current.startOfDay(
            for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
        )

        let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
        completion(timeline)
    }
}
```

**何をしているか：**

この部分では、ウィジェットに表示するデータと、そのデータを更新するタイミングを設定している。
placeholderでは、ウィジェットの読み込み中に表示する仮のデータを作成している。ここでは「読み込み中…」という名言を持つQuoteEntryを返している。
getSnapshotでは、ウィジェットギャラリーなどに表示されるプレビュー用のデータを作成している。QuoteStore.todaysQuote()を使用して、その日の名言を取得している。
getTimelineでは、実際にホーム画面へ表示するデータと、次回の更新時刻を設定している。現在の日付から今日の名言を取得し、翌日の午前0時以降に新しい内容へ更新するタイムラインを作成している。

**なぜこう書くのか：**

ウィジェットは、通常のアプリ画面のように常にプログラムが動いているわけではない。そのため、いつ、どのデータを表示するかをあらかじめタイムラインとしてWidgetKitへ渡す必要がある。
今回のアプリでは名言が1日ごとに変わるため、頻繁に更新する必要はない。そこで、今日の名言を1件だけタイムラインへ登録し、更新方針を.after(tomorrow)にすることで、翌日の午前0時を過ぎた後に更新を依頼している。
また、completionを使用して作成したデータをWidgetKit側へ渡すことで、ウィジェットの表示に反映させている。

**もしこう書かなかったら：**

placeholderを実装しなかった場合、ウィジェットの読み込み中に表示する内容をWidgetKitへ渡せなくなる。
getSnapshotが正しく実装されていない場合、ウィジェットギャラリーやプレビューで、実際とは異なる表示になったり、データが表示されなかったりする可能性がある。
getTimelineを実装しなかった場合、ホーム画面へ表示するデータや更新タイミングを決められないため、ウィジェットが正しく動作しない。
また、更新方針を適切に設定しなければ、翌日になっても名言が切り替わらず、前日の名言がそのまま表示される可能性がある

---

### TimelineEntryとウィジェットビュー

```swift
// MARK: - タイムラインプロバイダ

struct QuoteProvider: TimelineProvider {
    // プレースホルダー（読み込み中の仮表示）
    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(
            date: Date(),
            quote: Quote(id: 0, text: "読み込み中...", author: "")
        )
    }

    // スナップショット（ウィジェットギャラリーでのプレビュー）
    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
        let entry = QuoteEntry(
            date: Date(),
            quote: QuoteStore.todaysQuote()
        )
        completion(entry)
    }

    // タイムライン（実際のウィジェット更新スケジュール）
    func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) {
        let currentDate = Date()
        let quote = QuoteStore.todaysQuote()
        let entry = QuoteEntry(date: currentDate, quote: quote)

        // 次の日の0時にウィジェットを更新
        let tomorrow = Calendar.current.startOfDay(
            for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
        )

        let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
        completion(timeline)
    }
}
```

**何をしているか：**

QuoteEntryは、ウィジェットに表示する1回分のデータを表している。
TimelineEntryに準拠するために、データを表示する時刻を表すdateを持っている。また、今回のウィジェットで表示する名言の内容として、quoteを持っている。
QuoteWidgetEntryViewは、QuoteProviderから受け取ったentryを使って、ウィジェットの画面を作成している。
さらに、@Environment(\.widgetFamily)を使用して現在のウィジェットサイズを取得し、小サイズの場合はsmallWidget、中サイズの場合はmediumWidgetを表示している。

**なぜこう書くのか：**

WidgetKitでは、タイムライン上の各時刻に表示するデータを、TimelineEntryに準拠した型で管理する必要がある。
表示時刻と名言をQuoteEntryという1つの型にまとめることで、どの時刻に、どの名言を表示するのかを分かりやすく管理できる。
また、ウィジェットのビューがentryを受け取る構造にすることで、QuoteProviderが作成したデータを画面へ反映できる。
QuoteProvider.Entryは、QuoteProviderが扱うエントリ型を表している。このコードでは、実際にはQuoteEntryとして扱われる。

**もしこう書かなかったら：**

QuoteEntryがTimelineEntryに準拠していない場合、WidgetKitのタイムラインで使用できず、コンパイルエラーになる。
dateを用意しなかった場合も、TimelineEntryに必要な条件を満たせない。
また、ウィジェットビューがentryを受け取らなければ、取得した名言を画面へ表示できない。
widgetFamilyを取得せず、すべてのサイズで同じレイアウトを使用すると、小サイズでは文字が収まらなかったり、中サイズでは空白が多くなったりする可能性がある。

---

### ウィジェットサイズごとのレイアウト

```swift
// MARK: - ウィジェットのビュー

struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget
        case .systemMedium:
            mediumWidget
        default:
            mediumWidget
        }
    }

    // 小サイズ
    var smallWidget: some View {
        VStack(spacing: 4) {
            Image(systemName: "quote.opening")
                .font(.caption)
                .foregroundStyle(.blue)

            Text(entry.quote.text)
                .font(.caption)
                .bold()
                .multilineTextAlignment(.center)
                .lineLimit(3)

            Text(entry.quote.author)
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
        .padding(12)
    }

    // 中サイズ
    var mediumWidget: some View {
        HStack(spacing: 16) {
            Image(systemName: "quote.opening")
                .font(.title)
                .foregroundStyle(.blue)

            VStack(alignment: .leading, spacing: 4) {
                Text("今日の名言")
                    .font(.caption2)
                    .foregroundStyle(.secondary)

                Text(entry.quote.text)
                    .font(.subheadline)
                    .bold()

                Text("— \(entry.quote.author)")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }

            Spacer()
        }
        .padding()
    }
}
```

**何をしているか：**

QuoteEntryがTimelineEntryに準拠していない場合、WidgetKitのタイムラインで使用できず、コンパイルエラーになる。
dateを用意しなかった場合も、TimelineEntryに必要な条件を満たせない。
また、ウィジェットビューがentryを受け取らなければ、取得した名言を画面へ表示できない。
widgetFamilyを取得せず、すべてのサイズで同じレイアウトを使用すると、小サイズでは文字が収まらなかったり、中サイズでは空白が多くなったりする可能性がある。

**なぜこう書くのか：**

ウィジェットはサイズによって使用できる表示領域が異なるため、同じレイアウトをそのまま使用すると、文字が見切れたり、余白が不自然になったりする。
小サイズでは縦方向に要素を並べ、文字数を制限することで、狭い領域でも名言を読みやすくしている。
中サイズでは横幅を活用して、アイコンと文章を横方向に並べることで、情報を見やすく整理している。
また、smallWidgetとmediumWidgetを別々の計算プロパティに分けることで、bodyの中身が複雑になりすぎず、サイズごとのレイアウトを修正しやすくしている。

**もしこう書かなかったら：**

サイズごとのレイアウトを分けなかった場合、小サイズで文字が途中までしか表示されなかったり、レイアウトが窮屈になったりする可能性がある。
逆に、中サイズでも小サイズと同じ小さな文字や縦並びを使用すると、利用できる横幅をうまく活用できず、余白の多い表示になる。
また、lineLimit(3)を指定しなければ、長い名言が多くの行を使用し、作者名が画面外へ押し出される可能性がある。
defaultを用意しなかった場合、将来別のウィジェットサイズが渡されたときに、すべての場合を処理できないとしてコンパイルエラーになる可能性がある。

---

### メインアプリとの連携

```swift
// MARK: - ウィジェット定義

@main
struct QuoteWidget: Widget {
    let kind: String = "QuoteWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: QuoteProvider()) { entry in
            QuoteWidgetEntryView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("今日の名言")
        .description("日替わりで名言を表示します")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}
```

**何をしているか：**

この部分では、ウィジェット全体の設定を行っている。
@mainを付けたQuoteWidgetが、ウィジェット拡張機能の開始地点になる。
StaticConfigurationでは、ウィジェットを識別するためのkind、表示データを作成するQuoteProvider、データを画面へ表示するQuoteWidgetEntryViewを指定している。
.configurationDisplayNameでは、ウィジェット追加画面に表示される名前を設定している。
.descriptionでは、ウィジェットがどのような機能を持つかを説明している。
.supportedFamiliesでは、このウィジェットが小サイズと中サイズに対応していることを指定している。
また、メインアプリとウィジェットは、両方のTarget Membershipに登録したQuoteStore.swiftを使用することで、同じQuoteとQuoteStoreのデータを参照している。

**なぜこう書くのか：**

WidgetKitに、この構造体がウィジェット本体であることを伝えるために、Widgetプロトコルへの準拠と@mainが必要になる。
今回は、利用者がウィジェットの内容を設定する必要がなく、常に今日の名言を表示するため、StaticConfigurationを使用している。
providerにQuoteProvider()を指定することで、タイムラインから受け取った名言をウィジェットビューへ渡せる。
また、メインアプリとウィジェットでQuoteStore.swiftを共有することで、同じ名言データを重複して記述する必要がなくなる。
どちらか一方のデータだけを変更して内容が食い違うことも防げる。

**もしこう書かなかったら：**

@mainがなければ、ウィジェット拡張機能の開始地点が分からず、ウィジェットとして起動できない。
反対に、QuoteWidgetBundle.swiftなど別のファイルにも@mainが残っていると、開始地点が複数存在するため、'main' attribute can only apply to one typeというエラーが発生する。
StaticConfigurationにQuoteProviderを指定しなければ、表示する名言や更新スケジュールを取得できない。
.supportedFamiliesから.systemSmallや.systemMediumを外すと、そのサイズではウィジェットを追加できなくなる。
また、QuoteとQuoteStoreをメインアプリ側だけに置き、ウィジェットのTarget Membershipに含めなかった場合、ウィジェット側から型を参照できず、Cannot find 'Quote' in scopeなどのエラーが発生する。
逆に、メインアプリ側とウィジェット側の両方へ同じ型を別々に定義すると、同じターゲット内で定義が重複した場合にInvalid redeclarationエラーが発生する。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`TimelineProvider` | ウィジェットを更新するタイミングとコンテンツを定義 | `struct QuoteProvider: TimelineProvider { ... }` |
| 例：`@main` + `WidgetConfiguration` | ウィジェットのエントリーポイント | `@main struct QuoteWidget: Widget { ... }` |
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

本章のまとめ

この章では、WidgetKitを使って、ホーム画面に日替わりの名言を表示するウィジェットを作成した。

ウィジェットでは、TimelineEntryに表示するデータをまとめ、TimelineProviderを使ってデータの取得方法や更新タイミングを設定する。
また、widgetFamilyを使うことで、小サイズと中サイズそれぞれに適したレイアウトへ切り替えられることを学んだ。

さらに、QuoteとQuoteStoreをメインアプリとウィジェットの両方で共有することで、
同じデータを利用できることを確認した。WidgetKitを使うと、アプリを開かなくても必要な情報をホーム画面から確認できる機能を実装できる。
