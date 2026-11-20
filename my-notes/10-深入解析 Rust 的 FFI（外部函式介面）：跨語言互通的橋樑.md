# 深入解析 Rust 的 FFI（外部函式介面）：跨語言互通的橋樑

**FFI（Foreign Function Interface，外部函式介面）** 是一種讓程式語言能夠呼叫「由另一種程式語言撰寫的函式或函式庫」，或反過來「被其他語言呼叫」的機制。

在現代軟體工程中，很少有系統會將幾十年的舊程式碼全部推倒重寫。Rust 之所以能在短短幾年內成功滲透進 Python（如 `Polars`、`Pydantic v2`、`Ruff`）、JavaScript（如 `SWC`、`Rolldown`）、C/C++（如 Linux 核心、Firefox）以及行動端 App 中，最大的關鍵就是 **Rust 具備「零成本、無執行期負擔」的頂級 FFI 能力**。

---

## 一、為什麼 Rust 特別適合做 FFI？

相較於 Go、Java 或 Python，Rust 在跨語言呼叫上有兩個無可取代的先天優勢：

1. **沒有垃圾回收器（GC）與龐大的執行期（Runtime）**：
   當你把 Go 或 Java 打包成函式庫給 C 或 Python 呼叫時，目標程式必須同時載入一整套 GC 與排程器，跨語言邊界呼叫的成本極高。而 Rust 編譯出來就是純粹的原生機器碼，**呼叫 Rust 函式就跟呼叫普通的 C 函式一樣快，幾乎零額外開銷**。
2. **原生相容 C ABI**：
   在電腦科學中，**C 語言的 ABI（Application Binary Interface，應用程式二進位介面）** 是所有作業系統與程式語言的「共同世界語」。只要 Rust 能說一口流利的 C ABI，它就能與世界上幾乎所有程式語言溝通。

---

## 二、Rust FFI 的五大核心基礎機制

在深入寫程式碼之前，必須先理解 Rust 為了對齊 C ABI 所提供的幾個關鍵語法與概念：

### 1. `extern "C"`（指定呼叫慣例）
Rust 語言本身**刻意不固定自己的 ABI**（這樣編譯器才能在未來版本自由最佳化記憶體佈局與呼叫方式）。因此，跨語言溝通時必須明確寫出 `extern "C"`，告訴編譯器：「這個函式的參數傳遞、暫存器使用與堆疊清理方式，請完全遵守 C 語言標準。」

### 2. `#[unsafe(no_mangle)]`（禁止名稱修飾）
為了支援模組與泛型，Rust 編譯器預設會把函式名稱改寫成一串包含雜湊值的亂碼（稱為 Name Mangling，例如 `add` 可能變成 `_ZN4core3add17h9b...E`）。
若要讓外部語言找得到這個函式，必須加上此屬性，強制保留原本的函式名稱。
*(註：在 Rust 2024 Edition 之前寫作 `#[no_mangle]`；自 Rust 2024 Edition 起，為了強調符號名稱衝突的潛在風險，改為 `#[unsafe(no_mangle)]`。)*

### 3. `#[repr(C)]`（強制 C 語言記憶體佈局）
為了節省記憶體空間，Rust 編譯器預設會**自動重新排列 `struct` 裡面的欄位順序**。如果這個結構體要傳給 C 語言讀取，欄位順序亂掉會直接導致記憶體讀取錯誤。加上 `#[repr(C)]` 可以強制 Rust 按照「程式碼中宣告的順序」與 C 語言的對齊規則來排列記憶體。

```rust
#[repr(C)]
pub struct Point {
    pub x: f64,
    pub y: f64,
}
```

### 4. `unsafe`（安全邊界）
Rust 的借用檢查器只能保證 Rust 程式碼內部的安全，它完全不知道外部的 C 函式會不會亂改指標、造成緩衝區溢位或空指標解參考。因此，**所有呼叫外部 FFI 函式的動作都必須包在 `unsafe { ... }` 區塊中**，由開發者親自向編譯器擔保其安全性。

### 5. `std::ffi` 與 `core::ffi`（型別與字串轉換）
Rust 與 C 的基本型別與字串結構截然不同，標準函式庫提供了專門的橋接型別：
* **基本型別**：`c_int`、`c_char`、`c_void` 等（對應不同平台上的 C 型別寬度）。
* **字串轉換**：Rust 的 `String` 是 UTF-8 編碼且記錄長度（允許中間包含 `\0`）；C 的字串則是 `char*` 指標，以空字元 `\0` 結尾。因此跨邊界時必須透過 **`CString`**（Rust 轉 C）與 **`CStr`**（C 轉 Rust）進行安全轉換。

---

## 三、雙向實戰範例

### 方向 A：在 Rust 中呼叫 C 函式

以下示範如何在 Rust 中直接呼叫作業系統底層 C 標準函式庫（`libc`）的 `abs` 函式：

```rust
use std::ffi::c_int;

// 宣告外部 C 函式的簽名（Rust 2024 Edition 需加上 unsafe extern）
unsafe extern "C" {
    fn abs(input: c_int) -> c_int;
}

fn main() {
    let x = -42;
    // 呼叫外部函式必須放在 unsafe 區塊中
    let result = unsafe { abs(x) };
    println!("C 函式庫計算的絕對值為：{}", result); // 輸出：42
}
```

### 方向 B：將 Rust 編譯成函式庫，供 C（或其他語言）呼叫

假設我們想寫一個 Rust 函式庫，編譯成動態連結函式庫（Windows `.dll`、macOS `.dylib`、Linux `.so`）給外部程式呼叫。

**1. 設定 `Cargo.toml`：**
告訴 Cargo 產出 C 相容的動態函式庫（`cdylib`）或靜態函式庫（`staticlib`）：
```toml
[lib]
crate-type = ["cdylib"]
```

**2. 撰寫 `src/lib.rs`：**
```rust
use std::ffi::{CStr, CString, c_char};

// 1. 簡單數值運算：直接暴露給外部呼叫
#[unsafe(no_mangle)]
pub extern "C" fn rust_add(a: i32, b: i32) -> i32 {
    a + b
}

// 2. 進階範例：接收 C 字串，回傳一個由 Rust 分配的新 C 字串
#[unsafe(no_mangle)]
pub extern "C" fn rust_greet(name_ptr: *const c_char) -> *mut c_char {
    // 防禦性檢查：絕不信任外部傳來的指標不是 null
    if name_ptr.is_null() {
        return std::ptr::null_mut();
    }

    let c_str = unsafe { CStr::from_ptr(name_ptr) };
    let name = c_str.to_str().unwrap_or("陌生人");
    let greeting = format!("你好，{}！這是來自 Rust 的問候。", name);

    // 轉成 CString 並交出原始指標所有權（into_raw 會阻止 Rust 立刻釋放記憶體）
    match CString::new(greeting) {
        Ok(c_string) => c_string.into_raw(),
        Err(_) => std::ptr::null_mut(),
    }
}

// 3. 必須提供對應的釋放函式，讓外部用完後歸還記憶體給 Rust 釋放
#[unsafe(no_mangle)]
pub extern "C" fn rust_free_string(ptr: *mut c_char) {
    if !ptr.is_null() {
        unsafe {
            // 重新取回 CString 所有權，離開作用域時自動安全釋放
            let _ = CString::from_raw(ptr);
        }
    }
}
```

---

## 四、寫 FFI 時絕對不能踩的四大安全地雷

在跨語言邊界上，Rust 的安全護欄暫時撤除了，實務開發必須嚴守以下鐵律：

1. **嚴禁讓 Rust 的 `panic!` 跨越 FFI 邊界**
   如果暴露給 C 呼叫的 Rust 函式內部發生了 `panic!`，堆疊展開（Stack Unwinding）衝進 C 語言的程式碼是**未定義行為（Undefined Behavior）**，會導致程式直接崩潰甚至記憶體損毀。
   * **解法**：使用 `std::panic::catch_unwind` 攔截所有 panic，並將其轉成錯誤碼回傳給呼叫端；或在 `Cargo.toml` 設定 `panic = "abort"`。
2. **「誰配置，誰釋放」原則（Allocator Mismatch）**
   Rust 內部使用的記憶體配置器（如 jemalloc、mimalloc 或系統配置器）可能與 C/C++ 或 Python 不同。**絕對不能**在 Rust 裡配置記憶體，卻讓 C 語言呼叫 `free()` 去釋放！必須像上方範例一樣，由 Rust 提供專屬的 `rust_free_xxx()` 函式。
3. **永遠假設外部傳入的指標可能是 `NULL` 或無效 UTF-8**
   Rust 的參考（`&T`）在語言層級保證「絕對不為空」；但 C 指標（`*const T`）隨時可能是 `NULL`。在將原始指標轉為 Rust 參考之前，務必先檢查 `.is_null()`。
4. **設計「安全封裝層（Safe Wrapper）」**
   業界標準做法是將 FFI 拆成兩個 Crate：
   * `xxx-sys`：只放赤裸裸的、充滿 `unsafe` 與原始指標的底層 C 綁定。
   * `xxx`：把 `xxx-sys` 包裝起來，利用 Rust 的 `Result`、`Option` 與 `Drop` Trait 實作自動資源釋放，對外提供 **100% 安全的慣用 Rust API**。

---

## 五、現代開發必知：自動化 FFI 生態系工具

在實務開發中，除非是極底層的系統對接，否則開發者**很少需要手動撰寫繁瑣的 `extern "C"` 與指標轉換**。Rust 社群已經針對各大主流語言打造了極為成熟的高階綁定工具：

| 目標語言 / 平台 | 推薦工具 | 核心特點與知名應用案例 |
| :--- | :--- | :--- |
| **C 語言** | **`bindgen`** / **`cbindgen`** | `bindgen` 讀 C 的 `.h` 檔自動生成 Rust 程式碼；`cbindgen` 讀 Rust 程式碼自動生成 C 的 `.h` 標頭檔。 |
| **C++** | **`cxx`** / **`autocxx`** | 打破傳統 C ABI 限制，在編譯期靜態檢查 Rust 與 C++ 之間的型別安全（支援 `std::string`、`std::vector` 直接互通）。 |
| **Python** | **`PyO3`** ＋ **`maturin`** | 讓 Rust 直接寫成 Python 原生擴充模組，速度提升數十倍。著名案例：**`Ruff`**、**`Polars`**、**`Pydantic v2`**。 |
| **JavaScript / Node.js** | **`napi-rs`** | 基於 Node-API，輕鬆將 Rust 打包成跨平台 Node.js 原生套件。著名案例：**`SWC`**、**`Rolldown`**、**`Oxc`**。 |
| **Swift / Kotlin / Python / Ruby** | **`UniFFI`**（Mozilla） | 寫一份 Rust 邏輯，自動同時產出 Swift、Kotlin、Python、Ruby 的高階原生介面，跨平台 App 首選。 |
| **Web (瀏覽器)** | **`wasm-bindgen`** | 負責 Rust 與 JavaScript 之間的高階型別（字串、物件、Promise）在 WebAssembly 上的雙向橋接。 |
| **Dart / Flutter** | **`flutter_rust_bridge`** | 專為 Flutter 打造，支援零樣板程式碼的 Dart ↔ Rust 非同步呼叫與資料串流。 |

---

## 總結

Rust 的 FFI 設計完美體現了這門語言的務實哲學：
* **在底層**，它透過 `extern "C"`、`#[repr(C)]` 與 `unsafe` 提供與 C 語言完全同級的零成本互通能力。
* **在實務上**，開發者可以善用 `PyO3`、`napi-rs`、`cxx`、`UniFFI` 等現代化工具，把效能瓶頸或核心安全邏輯抽出來用 Rust 重寫，無痛嵌入既有的 Python、Node.js、C++ 或行動端專案中，達成「漸進式導入 Rust」的最佳實踐。