# Rust 的設計哲學：捨棄傳統繼承與融入函數式特性

Rust 在設計上非常務實：它觀察了過去幾十年軟體工程的經驗，選擇性地捨棄了容易引發維護災難的傳統設計（如類別繼承），同時大量吸收了函數式程式設計（Functional Programming）的優點，讓程式碼既安全又具高度表達力。

以下分為兩個部分詳細說明。

---

## 第一部分：為什麼 Rust 不採用傳統的物件導向（OOP）繼承？

在 Java、C++、Python 等傳統物件導向語言中，「類別繼承（Class Inheritance）」常被用來達成**程式碼重用（Code Reuse）**與**多型（Polymorphism）**。但隨著大型軟體系統的演進，軟體工程界逐漸發現繼承帶來了許多難以解決的痛點。

### 1. 傳統繼承的四大痛點

* **脆弱的基底類別問題（Fragile Base Class Problem）**：
  當子類別繼承父類別時，兩者之間會產生極高的耦合度。只要父類別的內部實作稍微修改，就可能意外破壞多個子類別的行為，導致牽一髮而動全身。
* **強制綁定資料與行為（「大猩猩與香蕉」問題）**：
  程式設計師常開玩笑說：「你只是想要一根香蕉，但繼承給了你一隻拿著香蕉的大猩猩，外加整座叢林。」在傳統繼承中，為了重用父類別的某個方法，子類別往往被迫繼承了一堆自己不需要的資料欄位與方法，這不僅讓設計變得混亂，更會造成**記憶體浪費**——這對系統程式語言來說是大忌。
* **繼承樹膨脹與多重繼承混亂**：
  現實世界的邏輯往往無法完美套進「樹狀階層」中（例如：企鵝是鳥，但不會飛；鴨嘴獸是哺乳類，但會下蛋）。當需求變更時，修改繼承架構的成本極高；若引入多重繼承，又會引發著名的「菱形繼承問題（Diamond Problem）」。
* **隱藏的效能開銷**：
  傳統 OOP 的多型通常依賴虛擬函式表（vtable）進行**動態分派（Dynamic Dispatch）**，且物件經常需要配置在堆積（Heap）記憶體上。這與 Rust 追求「零成本抽象」與「精準控制記憶體佈局」的目標背道而馳。

### 2. Rust 的替代方案：組合優於繼承 ＋ Trait 系統

Rust 徹底實踐了軟體工程的黃金準則——**「組合優於繼承（Composition over Inheritance）」**，並將「資料」與「行為」明確分離：

1. **用 `struct` 組合資料**：如果型別 B 需要型別 A 的資料，直接把 A 當作 B 的一個欄位即可，結構清晰且沒有多餘的隱藏欄位。
2. **用 `trait`（特徵）定義共享行為**：`trait` 類似其他語言的介面（Interface），用來定義「某個型別能做什麼」，甚至可以提供預設實作。任何 `struct` 都可以實作多個 `trait` 來達成多型，而且不會有繼承樹的耦合問題。
3. **明確選擇效能成本**：Rust 預設使用泛型搭配 `trait`（**靜態分派**，編譯期展開，效能零開銷）；只有在開發者真的需要執行期多型時，才明確寫出 `dyn Trait`（**動態分派**）。

### 程式碼範例：Rust 如何用 `struct` 與 `trait` 取代繼承

```rust
// 1. 定義共享的「行為」（Trait），而非父類別
trait Speak {
    fn speak(&self) -> String;
}

// 2. 定義純粹的「資料」結構（Struct）
struct Dog {
    name: String,
}

struct Robot {
    model: String,
}

// 3. 各自實作 Speak 行為（無須讓 Dog 和 Robot 繼承同一個莫名其妙的父類別）
impl Speak for Dog {
    fn speak(&self) -> String {
        format!("{} 汪汪叫！", self.name)
    }
}

impl Speak for Robot {
    fn speak(&self) -> String {
        format!("機器人 {} 發出嗶嗶聲。", self.model)
    }
}

// 4. 透過泛型與 Trait 實現零成本多型（靜態分派）
fn make_noise<T: Speak>(entity: &T) {
    println!("{}", entity.speak());
}

fn main() {
    let dog = Dog { name: String::from("小白") };
    let bot = Robot { model: String::from("R2-D2") };

    make_noise(&dog);
    make_noise(&bot);
}
```

---

## 第二部分：什麼是「函數式特性（Functional Programming Features）」？

### 1. 什麼是函數式程式設計（FP）？

**函數式程式設計**是一種將程式運算視為「數學函式求值」的程式設計典範。相較於傳統指令式程式設計（一步步告訴電腦「怎麼修改狀態」），函數式程式設計更強調：

* **宣告式思維**：描述資料「要轉換成什麼樣子」，而不是寫一堆迴圈去手動控制索引值。
* **避免可變狀態與副作用（Side Effects）**：資料一旦建立就盡量不修改，而是透過函式產生新資料。這能大幅減少 Bug，並讓多執行緒併發變得極為安全。
* **函式為一等公民（First-class Functions）**：函式可以像數字或字串一樣，被當作參數傳遞、當作結果回傳，或存放在變數中。

### 2. Rust 吸收了哪些核心的函數式特性？

雖然 Rust 並非純函數式語言（如 Haskell），但它借鏡了大量來自 OCaml 與 Haskell 的精華，並將其與高效能系統開發完美結合：

#### ① 預設不可變性（Immutability by Default）
在 Rust 中，所有變數預設都是不可變的（Immutable），除非明確加上 `mut`。這迫使開發者思考「這個狀態真的需要被改變嗎？」，從源頭減少意外修改資料造成的 Bug。

#### ② 閉包（Closures）與高階函式（Higher-Order Functions）
Rust 支援輕量級的匿名函式（閉包），可以捕獲周圍環境的變數，並將其傳遞給其他函式。

#### ③ 迭代器（Iterators）與資料流轉換
處理集合資料時，你不需要寫繁瑣的 `for` 迴圈與暫存變數，而是可以直接串接 `filter`、`map`、`fold` 等函數式方法。
> **Rust 的獨到之處**：Rust 的迭代器是**惰性求值（Lazy）**且**零成本抽象**的。編譯器會將這串優雅的高階函數式寫法，最佳化成與手寫底層 `for` 迴圈一樣快（甚至更快）的機器碼！

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6];

    // 函數式風格：挑出偶數 -> 每個乘以 10 -> 全部加總
    let sum: i32 = numbers
        .iter()
        .filter(|&&x| x % 2 == 0) // 傳入閉包過濾偶數
        .map(|&x| x * 10)         // 傳入閉包轉換數值
        .sum();                   // 彙總結果

    println!("計算結果：{}", sum); // 輸出：120 (20 + 40 + 60)
}
```

#### ④ 代數資料型別（ADTs）與模式匹配（Pattern Matching）
這是函數式語言中最強大的工具之一。Rust 的 `enum` 不只是簡單的常數列表，它的每個變體（Variant）都可以攜帶不同型別的資料。搭配 `match` 語法，編譯器會強制檢查你是否處理了「所有可能的情況」。
* Rust 用 `Option<T>` 取代了容易造成崩潰的 `null`。
* Rust 用 `Result<T, E>` 取代了難以追蹤流程的 `try-catch` 例外。

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
}

fn process_message(msg: Message) {
    // 模式匹配：編譯器會確保所有分支都被處理到，絕不遺漏
    match msg {
        Message::Quit => println!("收到離開指令"),
        Message::Move { x, y } => println!("移動到座標 ({}, {})", x, y),
        Message::Write(text) => println!("收到文字訊息：{}", text),
    }
}
```

#### ⑤ 基於表達式（Expression-based）
在 Rust 中，幾乎所有的控制結構（如 `if/else`、`match`、甚至大括號區塊 `{}`）本身都會「求值並回傳結果」，這也是典型的函數式風格。

```rust
let score = 85;
// 直接將 if/else 的運算結果賦值給變數
let status = if score >= 60 { "及格" } else { "不及格" };
```

---

## 總結

Rust 之所以受到現代開發者的高度推崇，正是因為它做了一次非常成功的設計取捨：

* **捨棄了傳統 OOP 繼承**：避開了深層繼承樹的耦合與隱藏的效能損耗，改用更乾淨、靈活的 **`struct` 組合 ＋ `trait` 多型**。
* **擁抱了函數式特性**：透過**預設不可變、閉包、零成本迭代器、代數資料型別與模式匹配**，讓系統級程式碼也能寫得像高階語言一樣簡潔、安全且容易維護。