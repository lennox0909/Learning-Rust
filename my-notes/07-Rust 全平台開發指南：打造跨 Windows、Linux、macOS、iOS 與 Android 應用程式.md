# Rust 全平台開發指南：打造跨 Windows、Linux、macOS、iOS 與 Android 應用程式

Rust 本身具備極佳的**跨平台編譯能力（Cross-Compilation）**，透過 LLVM 後端，同一份 Rust 核心程式碼可以輕鬆編譯成 Windows、Linux、macOS、iOS 與 Android 的原生機器碼。

然而，在實務上開發「跨平台應用程式」時，最大的挑戰通常不在於底層邏輯，而在於**使用者介面（UI）與作業系統原生 API（如相機、推播、檔案系統、生物辨識）的整合**。

目前業界使用 Rust 開發五大平台應用，主要分為以下**四種主流架構路線**。

---

## 一、四大主流跨平台架構路線

### 路線 1：Tauri 2.0（Web 前端 UI ＋ Rust 原生核心）— ★ 綜合開發效率最高
如果你希望用**同一套程式碼**同時輸出 Windows、Linux、macOS、iOS 與 Android 應用程式，**Tauri 2.0** 是目前最成熟、最受歡迎的選擇。

* **運作原理**：
  * **畫面層（UI）**：使用 HTML / CSS / JavaScript（可搭配 React、Vue、Svelte、Solid 等前端框架），並直接呼叫作業系統內建的系統級 WebView（Windows 的 WebView2、macOS/iOS 的 WKWebView、Android 的 WebView、Linux 的 WebKitGTK）來渲染畫面，因此打包體積遠小於 Electron。
  * **邏輯與系統層**：使用 Rust 撰寫核心邏輯、檔案處理、加密與多執行緒運算，並透過 IPC（行程間通訊）與前端互動。
  * **行動端原生整合**：Tauri 2.0 支援撰寫 Swift（iOS）與 Kotlin（Android）插件，讓 Rust 與前端能無縫呼叫相機、GPS、通知等手機硬體功能。
* **適合對象**：具備 Web 前端經驗、希望快速覆蓋五大平台的中小型團隊或個人開發者。

### 路線 2：Rust 共用核心 ＋ 原生 UI（Shared Core + Native UI）— ★ 大型商業 App 黃金標準
許多知名跨平台軟體（如 **1Password、Dropbox、Signal、Discord、Mozilla Firefox**）都採用這種架構。

* **運作原理**：
  * **核心層（Rust）**：將商業邏輯、網路請求、本機資料庫（如 SQLite）、加密演算法、狀態同步全部寫在一個純 Rust 函式庫中。
  * **介面層（各平台原生或跨平台 UI 框架）**：
    * iOS / macOS：使用 **SwiftUI**
    * Android：使用 **Jetpack Compose（Kotlin）**
    * Windows / Linux：使用 WinUI、GTK，或直接搭配 **Flutter** 來統一五大平台的 UI。
* **關鍵橋接工具（FFI 綁定生成器）**：
  * **`UniFFI`**（Mozilla 開發）：目前最推薦的工具！只需在 Rust 函式加上巨集標註，就能自動產生 Swift、Kotlin、Python 等語言的原生綁定程式碼，省去手寫 C FFI 的痛苦。
  * **`flutter_rust_bridge`**：如果你選擇用 **Flutter** 寫五大平台 UI、用 **Rust** 寫高效能底層，這是社群最成熟的一鍵橋接方案。
  * **`Crux`**：將 UI 視為純粹的「視圖殼（Shell）」，連應用程式狀態機也統一收歸 Rust 管理，再透過 UniFFI 驅動各平台原生 UI。
* **適合對象**：對 UI 原生體驗要求極高、且有資源維護各平台原生介面（或 Flutter）的商業產品。

### 路線 3：純 Rust 全端 GUI 框架（Pure Rust UI）
如果你完全不想碰 JavaScript、Swift 或 Kotlin，想用 100% Rust 完成畫面與邏輯，可以考慮以下現代 Rust GUI 框架：

| 框架名稱 | UI 開發風格 | 五大平台支援度與特色 |
| :--- | :--- | :--- |
| **Dioxus** | 類 React（RSX 語法） | 支援 Desktop、Web、iOS、Android。開發體驗極佳，支援熱重載（Hot Reload），是目前純 Rust 全平台最具潛力的框架之一。 |
| **Slint** | 專屬宣告式 `.slint` 語法 | 效能極佳、記憶體占用極低，支援桌面、行動與嵌入式螢幕，商業級完整度高（需留意其雙重授權條款）。 |
| **egui** | 即時模式（Immediate Mode） | 透過 `eframe` 支援桌面、Web 與 Android（iOS 需額外封裝）。非常適合開發工程工具、除錯面板、內部工具，但外觀較不像系統原生 App。 |
| **Iced** | 類 Elm 架構（訊息驅動） | 桌面（Windows/macOS/Linux）與 Web 非常成熟（System76 的 COSMIC 桌面環境即採用它），但行動端支援仍不如前兩者完整。 |

> **實務提醒**：純 Rust GUI 框架在**桌面端**已經相當好用，但在 **iOS 與 Android 行動端**上，面對複雜的中文輸入法（IME）、系統無障礙（Accessibility）、原生捲動手感與系統權限整合時，成熟度仍略遜於 Tauri 2.0 或 Flutter + Rust。

### 路線 4：遊戲與高幀率圖形應用
如果你要開發遊戲或 2D/3D 互動繪圖應用：
* **Bevy**：Rust 生態最熱門的 ECS 資料導向遊戲引擎，底層使用 `wgpu`（自動對應 DirectX 12、Vulkan、Metal），支援五大平台與 WebAssembly。
* **Macroquad**：極度輕量、開箱即用的 2D 遊戲框架，交叉編譯到 Android 與 iOS 的流程非常簡單。

---

## 二、架構選型快速對照表

| 評估維度 | Tauri 2.0 (Web UI + Rust) | Rust 核心 + Flutter UI | Rust 核心 + 各平台原生 UI | 純 Rust GUI (如 Dioxus/Slint) |
| :--- | :---: | :---: | :---: | :---: |
| **五大平台完整度** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐☆☆（行動端演進中） |
| **開發速度** | 極快 | 快 | 較慢（需寫多套 UI） | 中等 |
| **UI 流暢度與原生感** | 良（受限於 WebView） | 優（自繪引擎 60/120fps） | 極佳（100% 原生） | 良～優 |
| **需要學習的語言** | Rust + TS/JS + HTML/CSS | Rust + Dart | Rust + Swift + Kotlin (+ C#/C++) | 僅 Rust |
| **最佳適用場景** | 跨平台桌面/行動 App、工具軟體 | 追求一致高顏值 UI 的跨平台 App | 大型旗艦商業軟體、安全/加密核心 | 系統工具、嵌入式介面、極客專案 |

---

## 三、底層基礎：交叉編譯（Cross-Compilation）與工具鏈

無論選擇哪種架構，底層都離不開 Rust 的**目標平台（Target Triples）**設定。透過 `rustup` 可以輕鬆安裝不同作業系統與 CPU 架構的標準函式庫：

### 1. 常見五大平台 Target Triples

```bash
# Windows (64位元 MSVC / ARM64)
rustup target add x86_64-pc-windows-msvc
rustup target add aarch64-pc-windows-msvc

# macOS (Apple Silicon 與 Intel)
rustup target add aarch64-apple-darwin
rustup target add x86_64-apple-darwin

# Linux (x86_64 與 ARM64)
rustup target add x86_64-unknown-linux-gnu
rustup target add aarch64-unknown-linux-gnu

# iOS (真機 ARM64 與 模擬器)
rustup target add aarch64-apple-ios
rustup target add aarch64-apple-ios-sim

# Android (現代主流 64位元 ARM、舊款 32位元 ARM、模擬器 x86_64)
rustup target add aarch64-linux-android
rustup target add armv7-linux-androideabi
rustup target add x86_64-linux-android
```

### 2. 各平台必備的輔助編譯工具
光安裝 Target 還不夠，編譯原生程式還需要對應平台的**連結器（Linker）與 SDK**：

* **Android**：需安裝 **Android NDK**，並強烈推薦搭配 **`cargo-ndk`** 工具，它能自動設定複雜的 NDK 環境變數，一鍵編譯出 `.so` 動態函式庫並放進 Android 專案的 `jniLibs` 目錄。
* **iOS / macOS**：必須在 **macOS 電腦**上安裝 **Xcode** 進行編譯與簽章（Apple 授權與工具鏈限制）。實務上常使用 `xcodebuild -create-xcframework` 將 iOS 真機與模擬器的 `.a` 靜態函式庫打包成單一的 `.xcframework`。
* **Windows / Linux 跨平台編譯**：如果想在單一台機器（例如 Linux CI 伺服器）交叉編譯出 Windows 或其他架構的 Linux 執行檔，推薦使用 **`cargo-zigbuild`**（借用 Zig 強大的交叉編譯連結器）或 **`cross`**（基於 Docker 容器）。

---

## 四、程式碼實戰：撰寫跨平台核心與條件編譯

### 1. 使用 `UniFFI` 建立跨平台共用核心
透過 `UniFFI` 的程序巨集，你可以用極簡的 Rust 程式碼，自動對外暴露給 Swift（iOS/macOS）與 Kotlin（Android）呼叫：

```rust
// 在 Cargo.toml 加入 uniffi 依賴
uniffi::setup_scaffolding!();

// 加上 #[uniffi::export]，UniFFI 會自動產生對應的 Swift 與 Kotlin 函式
#[uniffi::export]
pub fn calculate_hash(input: String) -> String {
    // 這裡放高效能的共用核心邏輯（如加密、解析、壓縮等）
    format!("HASHED_{}", input.to_uppercase())
}

// 甚至可以直接匯出 Struct 與物件方法
#[derive(uniffi::Record)]
pub struct UserProfile {
    pub id: u64,
    pub name: String,
}
```
編譯並執行 `uniffi-bindgen` 後，Swift 端就能直接寫出完全符合原生習慣的 `let result = calculateHash(input: "hello")`，Kotlin 端亦然，完全不需要手動處理 C 指標與記憶體釋放！

### 2. 處理各作業系統的底層差異（條件編譯）
當你的核心程式碼需要針對不同作業系統執行不同邏輯（例如取得設定檔路徑或呼叫系統 API）時，可使用 Rust 內建的 `#[cfg(...)]` 屬性，編譯器只會編譯符合當前目標平台的程式碼：

```rust
pub fn get_platform_name() -> &'static str {
    #[cfg(target_os = "windows")]
    { "Windows" }

    #[cfg(target_os = "macos")]
    { "macOS" }

    #[cfg(target_os = "linux")]
    { "Linux" }

    #[cfg(target_os = "ios")]
    { "iOS" }

    #[cfg(target_os = "android")]
    { "Android" }
}
```

---

## 五、實務開發與 CI/CD 專案結構建議

若要長期維護五大平台專案，建議採用 **Cargo Workspace** 將程式碼分層，避免平台相依套件互相污染：

```text
my-cross-platform-app/
├── Cargo.toml               # Workspace 根設定檔
├── crates/
│   ├── app-core/            # 100% 純 Rust 商業邏輯（不含任何 UI 或平台綁定，方便寫單元測試）
│   └── app-ffi/             # UniFFI / FFI 橋接層（負責將 app-core 暴露給行動端/桌面端）
├── apps/
│   ├── desktop-tauri/       # Tauri 2.0 專案（或是各自的 ios/、android/ 原生專案）
│   ├── ios/                 # Xcode / SwiftUI 專案
│   └── android/             # Android Studio / Kotlin 專案
└── .github/workflows/       # CI/CD 自動化建置（使用 ubuntu、windows、macos 三種 Runner）
```

### 給新手的起步建議：
1. **想最快做出一個五平台都能跑的 App**：直接從 **Tauri 2.0** 開始，官方 CLI（`cargo tauri init`、`cargo tauri android init`、`cargo tauri ios init`）已經幫你把 90% 的環境與打包腳本都處理好了。
2. **已有現成 iOS/Android App，想導入 Rust 提升效能或共用邏輯**：從 **UniFFI ＋ `cargo-ndk`** 著手，先將一個獨立的模組（如加密、資料解析或離線演算法）抽離成 Rust Crate 整合進去。
3. **不想寫任何前端 JS，想嘗試純 Rust**：從 **Dioxus**（偏向一般 App）或 **Slint**（偏向精巧高效能介面）開始體驗。