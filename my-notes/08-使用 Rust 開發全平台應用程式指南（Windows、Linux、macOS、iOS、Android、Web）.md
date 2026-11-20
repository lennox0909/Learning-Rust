# 使用 Rust 開發全平台應用程式指南（Windows、Linux、macOS、iOS、Android、Web）

Rust 憑藉著「無龐大執行期（No heavy runtime）」、「極佳的 C 語言介面（FFI）互通性」以及「對 WebAssembly 的一等公民支援」，已成為現代跨平台開發極具競爭力的語言。

不過，**Rust 本身並不像 Flutter 或瀏覽器那樣內建一套官方的跨平台圖形介面（GUI）引擎**。因此，要用 Rust 同時覆蓋桌面端、行動端與網頁端這六大平台，實務上必須先選擇適合團隊的**架構路線**。

---

## 一、全平台開發的四大主流架構路線

| 架構路線 | 代表工具 / 框架 | UI 實作方式 | 優點 | 缺點 / 限制 |
| :--- | :--- | :--- | :--- | :--- |
| **1. Web 介面 ＋ Rust 核心** | **Tauri 2** | HTML / CSS / JS（React, Vue, Svelte 等） | 開發最快、前端生態極豐富、套件體積遠小於 Electron | 依賴系統內建 WebView；複雜動畫效能略遜原生 |
| **2. 共享 Rust 核心 ＋ 原生 UI** | **UniFFI** ＋ **wasm-bindgen**（或 **Crux**） | SwiftUI (iOS/macOS)、Kotlin Compose (Android)、WinUI/Web 等 | 各平台擁有 100% 原生體驗與效能；大型企業首選 | 需維護多套 UI 程式碼，人力成本較高 |
| **3. Flutter UI ＋ Rust 核心** | **flutter_rust_bridge** | Flutter（Dart） | UI 高度一致、元件庫極成熟、六大平台開箱即用 | 團隊需同時維護 Dart 與 Rust 兩種語言 |
| **4. 純 Rust 跨平台框架** | **Dioxus**、**Slint**、**egui**、**Bevy** | 全部使用 Rust 撰寫 | 單一語言走天下、無跨語言橋接（FFI）負擔 | UI 生態系與行動端原生元件整合仍在演進中 |

---

## 二、四大路線深度解析

### 路線 1：Tauri 2（最快覆蓋全平台的務實方案）
**Tauri 2** 正式支援 Windows、macOS、Linux、iOS 與 Android，而你的前端介面（React、Vue、Svelte 或 Leptos）本身就可以直接部署成 **Web 網站**。
* **運作原理**：使用作業系統內建的 WebView（Windows 的 WebView2、macOS/iOS 的 WKWebView、Linux 的 WebKitGTK、Android 的 Android WebView）來渲染介面，並透過 IPC（行程間通訊）呼叫底層的 Rust 程式碼處理檔案系統、加密、資料庫或高效能運算。
* **適合對象**：熟悉網頁前端技術、希望一套介面跑遍桌面、手機與網頁的團隊。

### 路線 2：Rust 核心 ＋ 各平台原生 UI（大型商業產品的黃金標準）
許多知名跨平台軟體（如 **1Password、Signal、Dropbox、Discord**）都採用這種「**邏輯集中、介面原生**」的架構：
* **運作原理**：將網路通訊、資料同步、加解密、本地資料庫（SQLite）等核心商業邏輯全部寫成一個純 Rust 函式庫；再透過橋接工具自動產生各平台的綁定程式碼（Bindings）：
  * **iOS / macOS（Swift）與 Android（Kotlin）**：使用 Mozilla 開發的 **`UniFFI`**，能根據 Rust 程式碼自動生成原生的 Swift 與 Kotlin 介面。
  * **Web（JavaScript / TypeScript）**：使用 **`wasm-bindgen`** 與 **`wasm-pack`**，將同一個 Rust 核心編譯成 WebAssembly（`.wasm`）供網頁呼叫。
  * **進階架構（Crux）**：如果想把「狀態管理與商業流程」也收斂到 Rust 裡，可以參考 Red Badger 推出的 **Crux** 框架。
* **適合對象**：對各平台使用者體驗要求極致（必須符合 iOS/Android/macOS 各自設計規範）的中大型團隊。

### 路線 3：Flutter（UI）＋ Rust（核心引擎）
如果你想要「單一 UI 程式碼庫」且需要比 WebView 更流暢的 60/120fps 渲染效能，**Flutter ＋ Rust** 是極為成熟的組合。
* **運作原理**：利用 **`flutter_rust_bridge`** 自動生成 Dart 與 Rust 之間的非同步橋接層。Flutter 負責所有六大平台的 UI 繪製與平台外掛（相機、推播、權限）；Rust 負責影像處理、音訊解碼、AI 推論、網路協定或高併發運算。
* **適合對象**：需要高度客製化動畫、穩定行動端表現，同時又需要 Rust 底層算力的 App。

### 路線 4：純 Rust 跨平台框架（All-in-Rust）
如果你希望從頭到尾只寫 Rust，不碰 JS、Swift、Kotlin 或 Dart，可依產品類型挑選：
* **Dioxus**：語法深受 React 啟發（具備類似 JSX 的 `rsx!` 巨集與狀態 Hook）。支援 Web（Wasm）、Desktop（WebView 或原生渲染器 Blitz）與 Mobile（iOS/Android）。
* **Slint**：專為嵌入式與跨平台設計的宣告式 UI 框架，使用專屬的 `.slint` 語法描述介面，執行效能極高、記憶體占用極低，支援桌面、行動端、Web 與微控制器。
* **egui**：即時模式（Immediate Mode）GUI，透過 `eframe` 可以極為輕鬆地一次編譯到桌面端、Web（Wasm/WebGL）與行動端，特別適合開發**工程工具、除錯面板、編輯器與儀表板**。
* **Bevy**：現代化純 Rust 遊戲引擎（ECS 架構），支援全部六大平台，適合 2D/3D 遊戲與高度互動的圖形應用。

---

## 三、底層關鍵：六大平台的交叉編譯（Target Triples）

無論選擇哪一種架構，Rust 都需要將程式碼編譯成各平台對應的目標架構（Target Triple）。透過 `rustup` 可以輕鬆安裝各平台的標準函式庫目標：

```bash
# 1. Windows (x64)
rustup target add x86_64-pc-windows-msvc

# 2. Linux (x64)
rustup target add x86_64-unknown-linux-gnu

# 3. macOS (Apple Silicon & Intel)
rustup target add aarch64-apple-darwin x86_64-apple-darwin

# 4. iOS (實機 & 模擬器)
rustup target add aarch64-apple-ios aarch64-apple-ios-sim

# 5. Android (現代手機主要是 ARM64，模擬器常需 x86_64)
rustup target add aarch64-linux-android armv7-linux-androideabi x86_64-linux-android

# 6. Web (WebAssembly)
rustup target add wasm32-unknown-unknown
```

### 各平台必備的建置輔助工具
由於不同作業系統的連結器（Linker）與 SDK 差異極大，實務上會搭配以下工具簡化打包流程：
* **Android**：使用 **`cargo-ndk`**（自動對接 Android NDK，產出各架構的 `.so` 動態函式庫）。
* **iOS / macOS**：必須在 macOS 環境下使用 Xcode 工具鏈編譯，常搭配 **`cargo-swift`**（配合 UniFFI 一鍵產出 Swift Package / XCFramework）。
* **Web**：使用 **`wasm-pack`**（將 Rust 打包成 NPM 套件）或 **`trunk`**（純 Rust Web 應用的建置與熱重載工具）。
* **桌面端跨系統編譯**：使用 **`cargo-zigbuild`**（借用 Zig 的交叉編譯連結器，在 macOS/Linux 上輕鬆編譯其他平台的執行檔）或透過 **GitHub Actions** 的各系統虛擬機自動建置。

---

## 四、實務專案架構與程式碼設計技巧

要讓同一份 Rust 程式碼順利在六大平台編譯與運行，建議遵循以下三大原則：

### 1. 使用 Cargo Workspace 拆分模組
將「完全不依賴平台的核心邏輯」與「各平台的介面膠水層」徹底分離：

```text
my-cross-platform-app/
├── Cargo.toml             # Workspace 根設定檔
├── crates/
│   ├── app-core/          # 純商業邏輯、資料模型、演算法（6 平台完全共用）
│   ├── app-ffi/           # 給 iOS / Android 用的 UniFFI 封裝層
│   ├── app-wasm/          # 給 Web 用的 wasm-bindgen 封裝層
│   └── app-desktop/       # 桌面端進入點（如 Tauri / Dioxus / egui）
└── apps/                  # 各平台專案殼層（ios/, android/, web/）
```

### 2. 善用條件編譯（Conditional Compilation）
當某些底層行為（例如檔案路徑、系統通知、執行緒模型）在各平台不同時，使用 `#[cfg(...)]` 屬性在編譯期切換實作：

```rust
pub fn get_platform_name() -> &'static str {
    #[cfg(target_os = "windows")]
    return "Windows";

    #[cfg(target_os = "macos")]
    return "macOS";

    #[cfg(target_os = "linux")]
    return "Linux";

    #[cfg(target_os = "ios")]
    return "iOS";

    #[cfg(target_os = "android")]
    return "Android";

    #[cfg(target_arch = "wasm32")]
    return "Web (WebAssembly)";
}
```

### 3. 特別注意 Web（Wasm）環境的限制
在六大平台中，**Web（`wasm32-unknown-unknown`）是最特殊的環境**，寫 `app-core` 時必須預先避開幾個地雷：
* **無法直接使用 `std::thread` 與 `std::time::Instant`**：瀏覽器沙盒不允許直接建立系統執行緒，計時器也需改用支援 Wasm 的套件（如 `web-time`）。
* **非同步執行期（Async Runtime）差異**：原生平台（Win/Mac/Linux/iOS/Android）通常使用 `tokio` 的多執行緒排程器；但 Web 端只能使用瀏覽器的事件迴圈（搭配 `wasm-bindgen-futures`）。
* **優先選用跨平台套件**：例如網路請求選用 **`reqwest`**（它會自動在原生端用底層 HTTP 客戶端，在 Wasm 端自動切換成瀏覽器的 `fetch` API），資料序列化選用 **`serde`**。

---

## 五、總結：該如何選擇？

* **你是網頁前端背景，想最快做出全平台產品** ➔ 選 **Tauri 2**（桌面＋手機＋網頁一次搞定）。
* **你追求極致的 UI 質感與穩定的行動端生態** ➔ 選 **Flutter ＋ `flutter_rust_bridge`**。
* **你是專業 App 開發團隊，要求 100% 原生體驗** ➔ 選 **Rust Core ＋ `UniFFI` ＋ `wasm-bindgen`**。
* **你想開發工程工具、除錯器，或堅持 100% 純 Rust** ➔ 選 **Dioxus**、**Slint** 或 **egui**。