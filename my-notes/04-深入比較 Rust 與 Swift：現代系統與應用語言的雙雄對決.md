# 深入比較 Rust 與 Swift：現代系統與應用語言的雙雄對決

**Rust**（由 Mozilla 發起，現由 Rust 基金會維護）與 **Swift**（由 Apple 主導開源）是 2010 年代誕生的兩大現代編譯式程式語言。兩者底層皆採用 **LLVM** 編譯架構，也都大量吸收了函數式程式設計的精華，追求**高安全性**與**接近 C/C++ 的執行效能**。

然而，這兩個語言的核心設計哲學卻截然不同：
* **Rust 的哲學是「絕對控制與零成本抽象」**：絕不隱藏任何效能開銷，將所有記憶體與併發風險在編譯期徹底消滅，哪怕這意味著較陡峭的學習曲線。
* **Swift 的哲學是「漸進式揭露（Progressive Disclosure）」**：讓日常應用開發盡可能直覺、優雅且低心智負擔；同時在開發者需要榨乾硬體效能時，才逐步暴露底層的控制工具。

---

## 一、核心維度快速對照表

| 比較維度 | Rust | Swift |
| :--- | :--- | :--- |
| **主要定位** | 系統程式設計、高併發後端、CLI、Wasm、嵌入式 | Apple 全平台應用開發（iOS/macOS/visionOS）、跨平台應用與部分伺服器端 |
| **記憶體管理** | 編譯期所有權（Ownership）與借用檢查（無執行期開銷） | 自動參考計數（ARC）＋ 值型別寫時複製（Copy-on-Write） |
| **物件導向支援** | 無 `class` 與繼承；純粹的 `struct`/`enum` ＋ `trait` | 提倡協定導向（`struct` ＋ `protocol`），但保留 `class` 與單一繼承 |
| **空值處理** | `Option<T>`（無特殊語法糖，靠模式匹配與方法鏈） | `Optional<T>`（具備 `T?`、`?.`、`??`、`guard let` 等極致語法糖） |
| **錯誤處理** | 基於回傳值：`Result<T, E>` ＋ `?` 運算子 | 基於語法機制：`do-try-catch` ＋ `throws`（支援 Typed Throws） |
| **非同步與併發** | 編譯期 `Send`/`Sync` 檢查；`async/await` 不綁定執行期（需搭 Tokio 等） | 內建結構化併發（Structured Concurrency）、`actor` 模型與編譯期資料競爭檢查 |
| **跨平台生態** | 極強（Linux、Windows、macOS、Wasm、裸機嵌入式皆為一等公民） | Apple 生態無敵；Linux/Windows/Wasm 持續強化中，但第三方生態較小 |
| **學習曲線** | 初期陡峭（需跨越所有權與生命週期門檻） | 初期平緩，進階（底層效能調校與嚴格併發）才變陡峭 |

---

## 二、六大核心技術深度解析

### 1. 記憶體管理：編譯期所有權 vs. ARC 自動參考計數

這是兩者最根本的差異，也直接決定了它們的效能上限與開發體驗。

* **Rust（所有權與借用）**：
  Rust 不使用垃圾回收（GC），也不預設使用參考計數。每個值在編譯期就決定了唯一的「所有者」，離開作用域即刻釋放記憶體。
  * **優點**：執行期**零開銷**（沒有原子加減計數器的成本），且預設幾乎不可能產生循環參考（Reference Cycle）導致的記憶體洩漏。
  * **缺點**：開發者必須向編譯器證明參考的生命週期合法，撰寫複雜資料結構（如雙向鏈結串列、圖形結構）或 GUI 狀態樹時會非常吃力。

* **Swift（ARC ＋ 值型別優先）**：
  Swift 將資料分為「值型別（`struct`、`enum`）」與「參考型別（`class`）」。對於 `class` 實例，Swift 使用 **ARC（Automatic Reference Counting）**，由編譯器自動在適當時機插入 `retain`（計數 +1）與 `release`（計數 -1）指令；而陣列、字典等標準集合則是 `struct`，搭配**寫時複製（Copy-on-Write, CoW）**來兼顧安全與效能。
  * **優點**：沒有 GC 的停頓問題，且開發者不需要與「借用檢查器」搏鬥，開發 UI 與商業邏輯非常順手。
  * **缺點**：ARC 的計數增減在多執行緒環境下需要原子操作（Atomic Operations），會產生些許執行期效能開銷；此外，使用 `class` 時必須小心**循環參考**，需手動標註 `weak` 或 `unowned`。

> **近年來的匯流趨勢**：Swift 近年引入了深受 Rust 啟發的**不可複製型別（Noncopyable Types, `~Copyable`）**以及 `borrowing` / `consuming` 參數修飾詞。這代表 Swift 開發者在撰寫極致效能或系統底層程式碼時，現在也能選擇繞過 ARC 與複製開銷，獲得類似 Rust 的所有權控制力！

---

### 2. 多型與抽象設計：Trait 系統 vs. Protocol 導向程式設計

兩者在語法設計上非常相似，都強烈推崇「組合優於繼承」，並將「資料」與「行為」分離。

* **Rust 的 `trait`**：
  Rust 徹底捨棄了 `class` 與繼承。所有多型都透過 `trait` 實現，且**強制開發者明確區分效能成本**：
  * **靜態分派（Static Dispatch）**：透過泛型 `fn draw<T: Drawable>(item: &T)`，編譯期單態化（Monomorphization）展開，零執行期開銷。
  * **動態分派（Dynamic Dispatch）**：必須明確寫出 `dyn Drawable`（Trait Object），透過 vtable 進行執行期派發。

* **Swift 的 `protocol` 與 `class`**：
  Swift 提出「協定導向程式設計（Protocol-Oriented Programming, POP）」，其 `protocol` 與 `extension` 的組合幾乎對應 Rust 的 `trait` 與 `impl`。
  * 不同之處在於，為了相容 Apple 龐大的 Objective-C 歷史遺產（UIKit、AppKit 等）以及方便表達具備「共享可變狀態」的物件，Swift 保留了 `class` 與單一繼承。
  * 在分派語法上，Swift 近年也透過 `some Protocol`（不透明型別，靜態分派）與 `any Protocol`（存在型別，動態分派），讓開發者能像 Rust 一樣精準區分效能開銷。

---

### 3. 空值安全與錯誤處理

兩者都徹底消滅了傳統 C/Java 中惡名昭彰的空指標例外（Null Pointer Exception），但在語法風格上體現了不同的取捨：

* **空值處理**：
  * **Rust** 的 `Option<T>` 只是標準函式庫中一個普通的 `enum`，沒有特殊的語言層級語法糖，主要透過 `match`、`if let` 或 `.map()`、`.unwrap_or()` 等函數式方法處理。
  * **Swift** 的 `Optional<T>` 雖然本質上也是 `enum`，但語言提供了極為豐富的語法糖：`?`（可選鏈結）、`??`（空值合併）、`if let` 與 `guard let`（提早返回），在處理層層嵌套的資料時比 Rust 簡潔許多。

* **錯誤處理**：
  * **Rust** 將錯誤視為普通的值（`Result<T, E>`）。函式回傳 `Result`，呼叫端使用 `?` 運算子即可簡潔地向上傳遞錯誤。這讓錯誤處理可以無縫融入迭代器與函數式管線中。
  * **Swift** 採用 `throws` 與 `do-try-catch` 語法（雖然語法長得像傳統例外，但底層實作其實類似 Rust 的回傳值機制，沒有傳統 C++ 例外展開的沉重開銷）。從 Swift 6 開始，更支援了**具型別拋出（Typed Throws，如 `throws(NetworkError)`）**，大幅拉近了與 Rust 強型別錯誤處理的距離。

---

### 4. 併發與非同步模型

* **Rust（無畏併發 ＋ 執行期無關）**：
  * **安全保證**：透過內建的 `Send`（可跨執行緒轉移所有權）與 `Sync`（可跨執行緒共享不可變參考）兩個標記 Trait，在編譯期就徹底杜絕資料競爭。
  * **非同步機制**：Rust 的 `async/await` 採用零成本的狀態機（State Machine）編譯，且**標準函式庫刻意不內建非同步執行期（Runtime）**。開發者可以依場景自由選擇 `Tokio`（後端微服務首選）、`Embassy`（嵌入式微控制器首選）等不同執行期。代價是生態系會出現些許碎片化，且 `Pin`、`Unpin` 等底層概念較難理解。

* **Swift（開箱即用的結構化併發 ＋ Actor）**：
  * **安全保證**：Swift 6 正式啟用了完整的編譯期資料競爭檢查（透過 `Sendable` 協定與隔離域檢查），達到與 Rust 同等級的併發安全性。
  * **非同步機制**：Swift 在語言與標準執行期中**原生整合**了結構化併發（`async/await`、`Task`、`TaskGroup`）以及語言級別的 **`actor` 型別**（類似 `class`，但自動保證內部狀態同一時間只會被一個任務存取，從根本上避免鎖的濫用）。對於應用程式開發者來說，上手門檻遠低於 Rust。

---

### 5. 底層控制與效能天花板

* **Rust 是純正的系統語言**：
  Rust 可以完全不依賴作業系統標準函式庫（`#![no_std]`），產出的執行檔體積極小、記憶體佈局完全可預測、沒有隱藏的記憶體配置或執行期（Runtime）。因此它能直接寫進 Linux 核心、Windows 核心、作業系統 Bootloader 與極度受限的微控制器。
* **Swift 是高效能的通用語言（正向底層延伸）**：
  Swift 的運算效能非常優異，在許多演算法基準測試中能與 Rust 和 C++ 並駕齊驅。不過傳統上 Swift 依賴一定的執行期環境（ARC、動態派發元資料等）。雖然 Apple 近年推出了 **Embedded Swift**（裁減執行期與反射機制以支援嵌入式與 Wasm），但在裸機（Bare-metal）與作業系統底層領域的成熟度，目前仍遠不及 Rust。

---

### 6. 生態系、工具鏈與跨平台能力

* **Rust**：
  * **工具鏈**：`Cargo` ＋ `Crates.io` 是公認現代程式語言中最優秀的套件與建置體驗之一，跨平台交叉編譯（Cross-compilation）極為成熟。
  * **C/C++ 互通**：與 C 語言的 FFI（外部函式介面）極為順暢；與 C++ 互通則需要透過 `cxx` 或 `autocxx` 等橋接工具。
  * **應用版圖**：雲端基礎設施、高效能 Web 後端、CLI 工具、資料庫引擎、WebAssembly、區塊鏈、系統核心。

* **Swift**：
  * **工具鏈**：官方的 `Swift Package Manager (SwiftPM)` 相當好用，但在 Apple 平台上許多開發者仍高度依賴 Xcode。
  * **C/C++ 互通**：Swift 在編譯器層級原生支援與 C、Objective-C 以及 **C++ 的雙向互通**，這在整合既有 C++ 函式庫時是一大優勢。
  * **應用版圖**：在 iOS、iPadOS、macOS、watchOS、visionOS 領域擁有絕對統治力（搭配 SwiftUI）。在伺服器端（如 Vapor、Hummingbird 框架）與 Linux/Windows 平台雖有官方持續投入，但社群規模與套件豐富度仍明顯落後於 Rust 與 Go。

---

## 三、程式碼風格對照

以下以「定義代數資料型別（Enum 攜帶資料）並處理可能失敗的操作」為例，感受兩者在語法表達上的異同：

### Rust 寫法
```rust
enum ApiError {
    NotFound,
    Unauthorized(String),
}

// 顯式回傳 Result<成功型別, 錯誤型別>
fn fetch_user(id: u32) -> Result<String, ApiError> {
    if id == 0 {
        return Err(ApiError::Unauthorized(String::from("無效的權限")));
    }
    if id == 1 {
        Ok(String::from("Alice"))
    } else {
        Err(ApiError::NotFound)
    }
}

fn main() {
    match fetch_user(1) {
        Ok(name) => println!("找到使用者：{}", name),
        Err(ApiError::NotFound) => println!("錯誤：找不到使用者"),
        Err(ApiError::Unauthorized(msg)) => println!("權限不足：{}", msg),
    }
}
```

### Swift 寫法
```swift
enum ApiError: Error {
    case notFound
    case unauthorized(String)
}

// 使用 throws(ApiError) 宣告具型別的錯誤拋出
func fetchUser(id: UInt32) throws(ApiError) -> String {
    if id == 0 {
        throw .unauthorized("無效的權限")
    }
    if id == 1 {
        return "Alice"
    } else {
        throw .notFound
    }
}

do {
    let name = try fetchUser(id: 1)
    print("找到使用者：\(name)")
} catch .notFound {
    print("錯誤：找不到使用者")
} catch .unauthorized(let msg) {
    print("權限不足：\(msg)")
}
```

從上述範例可以看出，兩者在**型別安全**與**模式匹配**的思維上高度一致；但 Swift 的語法讀起來更接近自然語言且省略了許多樣板細節，而 Rust 則更加顯式且結構嚴謹。

---

## 四、總結：該如何選擇？

### 優先選擇 **Rust**，如果您的目標是：
1. **跨平台底層開發或系統程式**：作業系統、驅動程式、嵌入式裝置、遊戲引擎底層。
2. **高吞吐、低延遲的後端與基礎設施**：微服務、API Gateway、資料庫、訊息佇列、高效能 CLI 工具。
3. **WebAssembly（Wasm）**：需要在瀏覽器中運行極速且體積輕量的運算模組。
4. **追求跨平台共用核心邏輯**：例如寫一套核心引擎（Rust），再透過 FFI 同時供 iOS、Android、Web 與桌面端呼叫。

### 優先選擇 **Swift**，如果您的目標是：
1. **Apple 生態系應用開發**：開發 iOS、macOS、visionOS 應用程式，Swift 搭配 SwiftUI 是無可取代的唯一首選。
2. **重視開發流暢度與 UI 互動體驗**：在不需要極端壓榨最後 5% 記憶體或 CPU 效能的情境下，Swift 的 ARC 與語法糖能大幅提升開發效率。
3. **Apple 團隊的全端整合**：如果團隊已經精通 Swift，使用 Server-side Swift（如 Vapor / Hummingbird）可以達成前後端共享資料模型與語言技能。