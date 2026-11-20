# Windows 11 + VS Code 建置 Rust 開發環境完整指南

在 Windows 11 上使用 VS Code 撰寫 Rust 的體驗非常流暢，但在開始之前，Windows 平台比 macOS / Linux 多了一個關鍵前置步驟——**安裝 C++ 連結器（MSVC Build Tools）**。

跟著以下五個步驟，大約 10～15 分鐘就能完成一套專業級的 Rust 開發環境。

---

## 步驟一：安裝 Rust 與 C++ 建置工具（MSVC）

在 Windows 上，Rust 預設需要微軟的 C++ 連結器（`link.exe`）與 Windows SDK 才能把程式碼打包成 `.exe` 執行檔。

### 方法 A：使用官方安裝程式（最簡單，會自動引導安裝 C++ 工具）
1. 前往 Rust 官網下載安裝檔：[https://www.rust-lang.org/zh-TW/tools/install](https://www.rust-lang.org/zh-TW/tools/install)
2. 下載並執行 **`rustup-init.exe`**（64 位元）。
3. 執行後會跳出終端機視窗。如果電腦尚未安裝 Visual Studio C++ Build Tools，安裝程式會直接詢問你是否要自動下載並安裝它：
   * 選擇 **`1) Quick install via the Visual Studio Community installer`**（或先手動安裝 Visual Studio Build Tools，勾選「**使用 C++ 的桌面開發**」工作負載）。
4. 接著看到 Rust 安裝選項時，直接按下 **`1`**（Proceed with standard installation）並按 Enter。

### 方法 B：使用 Windows 11 內建的 `winget` 指令安裝（極客首選）
打開 **Windows Terminal（PowerShell）**，依序貼上以下指令：

```powershell
# 1. 安裝 Visual Studio 2022 Build Tools（含 C++ 桌面開發工具）
winget install Microsoft.VisualStudio.2022.BuildTools --override "--wait --passive --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"

# 2. 安裝 Rust 工具鏈管理器 (rustup)
winget install Rustlang.Rustup
```

### 驗證安裝是否成功
安裝完成後，**務必關閉並重新開啟終端機**，輸入以下指令確認版本：

```powershell
rustc --version
cargo --version
```
若畫面正常顯示版本號（例如 `rustc 1.xx.x` 與 `cargo 1.xx.x`），代表 Rust 已經準備就緒！

---

## 步驟二：安裝 VS Code 必備擴充套件（Extensions）

開啟 VS Code，點擊左側的「擴充功能（`Ctrl + Shift + X`）」，搜尋並安裝以下套件：

### 1. 必裝核心套件
* **`rust-analyzer`**（官方維護）：
  * **最重要！** 提供即時語法檢查、型別提示（Inlay Hints）、自動補全、跳轉定義、重構與程式碼執行按鈕。
  * *注意：請認明作者是 `rust-lang`，不要安裝到已棄用的舊版套件。*
* **`CodeLLDB`**（作者：Vadim Chugunov）：
  * 在 VS Code 裡設定中斷點（Breakpoint）、單步執行、查看變數記憶體內容必備的除錯器，與 `rust-analyzer` 完美整合。
* **`Even Better TOML`**（作者：tamasfe）：
  * 讓設定檔 `Cargo.toml` 擁有語法高亮、自動排版與格式檢查。

### 2. 強烈推薦的效率外掛（選裝）
* **`Error Lens`**：直接將編譯錯誤與警告的文字顯示在程式碼該行的右側，寫 Rust 時不用一直把滑鼠移到紅線上看錯誤訊息，效率大增。
* **`Dependi`**（或 `crates`）：在 `Cargo.toml` 中直接顯示你引入的套件是否有最新版本可更新。

---

## 步驟三：建立、執行與除錯第一個 Rust 專案

打開 VS Code 的內建終端機（快捷鍵 `` Ctrl + ` ``），切換到你想放專案的**全英文路徑**（例如 `C:\dev`），執行以下指令：

```powershell
# 1. 建立名為 hello_rust 的新專案
cargo new hello_rust

# 2. 用 VS Code 開啟該資料夾
code hello_rust
```

進入新開啟的 VS Code 視窗後，點開左側的 `src/main.rs`：

```rust
fn main() {
    let name = "Windows 11";
    println!("Hello, Rust on {}!", name);
}
```

### 你有兩種方式執行與除錯：
1. **點選程式碼上方的快捷按鈕（最方便）**：
   等待左下角 `rust-analyzer` 讀取完成後，你會發現 `fn main()` 的正上方出現灰色的 **`▶ Run | Debug`** 小字：
   * 點 **`Run`** 即可直接編譯並執行。
   * 在行號左側點一下設個紅點（中斷點），再點 **`Debug`**，就會自動啟動 `CodeLLDB` 進入單步除錯模式！
2. **在終端機下指令**：
   * `cargo run`：編譯並執行。
   * `cargo check`：只做語法與借用檢查（不產出執行檔，速度極快）。
   * `cargo build --release`：產出經過完整最佳化的正式版 `.exe`（位於 `target/release/`）。

---

## 步驟四：推薦的 VS Code 進階設定（`settings.json`）

按 `Ctrl + Shift + P`，輸入並選擇 **`Preferences: Open User Settings (JSON)`**，加入以下設定能大幅提升寫 Rust 的爽度：

```json
{
    // 存檔時自動使用官方 rustfmt 排版程式碼
    "[rust]": {
        "editor.defaultFormatter": "rust-lang.rust-analyzer",
        "editor.formatOnSave": true
    },

    // 存檔時改用 clippy 取代預設的 check，即時給你更道地的 Rust 寫法建議
    "rust-analyzer.check.command": "clippy",

    // （選用）如果覺得畫面上自動顯示的型別提示太雜亂，可設為 "offUntilPressed"（按住 Ctrl+Alt 才顯示）
    "editor.inlayHints.enabled": "on"
}
```

---

## 步驟五：Windows 11 專屬避坑與加速技巧

在 Windows 11 寫 Rust 時，新手最常遇到以下三個問題，提前處理可以省下大量時間：

### 1. 解決 Windows Defender 拖慢編譯速度（超有感！）
Rust 編譯時會在 `target` 資料夾內短時間頻繁讀寫大量小檔案，Windows Defender 的即時掃描會讓編譯時間變慢 2～3 倍。
* **設定方法**：開啟 **Windows 安全性** → **病毒與威脅防護** → **管理設定** → **新增或移除排除項目**，將以下三個資料夾加入排除：
  1. `C:\Users\<你的使用者名稱>\.cargo`
  2. `C:\Users\<你的使用者名稱>\.rustup`
  3. 你專門放程式專案的總資料夾（例如 `C:\dev`）

### 2. 避免使用中文資料夾路徑
雖然現代 Rust 工具鏈對 Unicode 支援已進步許多，但部分底層 C/C++ 函式庫或建置腳本在 Windows 上遇到中文路徑（例如 `C:\Users\王小明\桌面\專案`）仍可能報錯。強烈建議在 `C:\dev` 或 `D:\projects` 等**純英文、無空白**的路徑下開發。

### 3. 常見報錯：`error: linker 'link.exe' not found`
這代表系統找不到微軟的 C++ 連結器。請開啟 **Visual Studio Installer**，點擊「修改（Modify）」，確認已經勾選 **「使用 C++ 的桌面開發（Desktop development with C++）」**，且右側詳細資訊中有勾選 **MSVC 建置工具** 與 **Windows 11 SDK**，安裝完畢後重開 VS Code 即可解決。