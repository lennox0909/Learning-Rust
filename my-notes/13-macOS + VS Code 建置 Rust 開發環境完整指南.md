# macOS + VS Code 建置 Rust 開發環境完整指南

在 macOS 上開發 Rust 的體驗非常優秀且直覺。不同於 Windows 需要安裝龐大的 Visual Studio Build Tools，macOS 只需要安裝輕量級的 **Xcode Command Line Tools** 取得底層連結器（Linker），接著透過一行指令就能完成整個環境建置。

跟著以下五個步驟，大約 5～10 分鐘即可完成設定。

---

## 步驟一：安裝 Xcode Command Line Tools（取得 C 連結器）

Rust 編譯器在最後打包執行檔時，需要呼叫 macOS 內建的 C 連結器（`cc` / `ld`）。你**不需要**從 App Store 下載數十 GB 的完整版 Xcode，只需安裝命令列工具即可。

打開 macOS 內建的「終端機（Terminal）」，輸入以下指令：

```zsh
xcode-select --install
```

畫面會跳出系統視窗詢問是否安裝，點擊「**安裝**」並等待完成即可。（如果系統提示 `already installed`，代表你已經裝好了，直接跳下一步。）

---

## 步驟二：安裝 Rust 工具鏈（`rustup`）

### 強烈推薦：使用官方安裝腳本
在終端機輸入以下官方指令：

```zsh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

出現安裝選項時，直接輸入 **`1`**（Proceed with standard installation）並按下 Enter。

> **⚠️ 給 Homebrew 使用者的避坑提醒**：
> 請**不要**使用 `brew install rust` 來安裝 Rust！透過 Homebrew 直接安裝的單一版本缺少官方管理工具 `rustup`，未來在更新版本、安裝 `clippy` / `rustfmt`、或是新增跨平台編譯目標（如 Wasm、iOS、ESP32）時會非常麻煩。請一律使用上述官方 `rustup` 腳本安裝。

### 讓環境變數生效並驗證安裝
安裝完成後，**關閉並重新開啟終端機**（或執行 `source "$HOME/.cargo/env"`），接著輸入以下指令確認：

```zsh
rustc --version
cargo --version
```

若正常顯示版本號（例如 `rustc 1.xx.x` 與 `cargo 1.xx.x`），代表 Rust 已安裝成功！

---

## 步驟三：設定 VS Code 與安裝必備擴充套件

### 1. 開啟終端機的 `code` 快捷指令（推薦）
開啟 VS Code，按下 **`Cmd + Shift + P`** 叫出命令面板，輸入並執行：
> **`Shell Command: Install 'code' command in PATH`**

設定完成後，之後在終端機裡就能直接用 `code .` 快速用 VS Code 打開任何專案資料夾。

### 2. 安裝必備擴充套件（Extensions）
按下 **`Cmd + Shift + X`** 開啟擴充功能面板，搜尋並安裝以下套件：

* **`rust-analyzer`**（官方維護，作者 `rust-lang`）：
  * **核心必裝！** 提供即時型別提示、語法檢查、自動補全、跳轉定義與程式碼上方的執行／除錯按鈕。
* **`CodeLLDB`**（作者 `Vadim Chugunov`）：
  * macOS 上最順手的 Rust 除錯器，支援設定中斷點、單步執行與變數檢視，完整支援 Apple Silicon（M1～M4）與 Intel Mac。
* **`Even Better TOML`**（作者 `tamasfe`）：
  * 讓 `Cargo.toml` 設定檔擁有語法高亮與格式檢查。
* **推薦效率外掛（選裝）**：
  * **`Error Lens`**：直接將錯誤與警告訊息顯示在程式碼右側，寫 Rust 時能大幅減少來回查看的時間。
  * **`Dependi`**：在 `Cargo.toml` 中即時提示套件是否有新版本。

---

## 步驟四：建立、執行與除錯第一個 Rust 專案

打開終端機，切換到你用來放專案的資料夾（例如 `~/dev`），執行以下指令：

```zsh
# 1. 建立開發資料夾並進入
mkdir -p ~/dev && cd ~/dev

# 2. 建立名為 hello_rust 的新專案
cargo new hello_rust

# 3. 用 VS Code 開啟專案
code hello_rust
```

打開左側的 `src/main.rs`：

```rust
fn main() {
    let os = "macOS";
    println!("Hello, Rust on {}!", os);
}
```

### 執行與除錯方式
1. **點選編輯器內的快捷按鈕（最方便）**：
   等待右下角 `rust-analyzer` 索引完成後，`fn main()` 上方會出現 **`▶ Run | Debug`** 按鈕：
   * 點 **`Run`** 直接編譯並執行。
   * 在行號左側點一下設紅點（中斷點），再點 **`Debug`** 即可透過 `CodeLLDB` 進行單步除錯。
2. **在 VS Code 內建終端機（`` Ctrl + ` ``）下指令**：
   * `cargo run`：編譯並執行。
   * `cargo check`：只做編譯期與借用檢查，不產出執行檔（速度最快）。
   * `cargo build --release`：產出完整最佳化的正式執行檔。

---

## 步驟五：推薦的 VS Code 進階設定（`settings.json`）

按下 **`Cmd + Shift + P`**，輸入並選擇 **`Preferences: Open User Settings (JSON)`**，加入以下設定：

```json
{
    // 存檔時自動使用官方 rustfmt 格式化程式碼
    "[rust]": {
        "editor.defaultFormatter": "rust-lang.rust-analyzer",
        "editor.formatOnSave": true
    },

    // 存檔時改用 clippy 檢查，提供更道地的 Rust 語法與效能建議
    "rust-analyzer.check.command": "clippy",

    // 若覺得行內型別提示太佔畫面，可改為 "offUntilPressed"（按住 Ctrl+Option 才顯示）
    "editor.inlayHints.enabled": "on"
}
```

---

## 步驟六：macOS 專屬避坑與效能優化技巧

在 macOS 上寫 Rust 時，有三個非常常見的「隱形地雷」，提前避開會讓開發順暢很多：

### 1. 千萬不要把 Rust 專案放在 iCloud 同步資料夾！
許多 macOS 使用者開啟了「iCloud 雲碟同步桌面與文件資料夾」。Rust 專案在編譯時，會在 `target/` 資料夾內瞬間產生**數百 MB 到數 GB 的龐大暫存檔**。
* **後果**：iCloud 會瘋狂上傳這些編譯產物，導致網路塞爆、硬碟空間異常，甚至鎖住檔案導致編譯失敗。
* **解法**：請務必將專案建立在**不會被 iCloud 同步**的家目錄路徑下，例如 **`~/dev`** 或 **`~/projects`**。

### 2. Apple Silicon（M1 / M2 / M3 / M4）確認沒有誤跑在 Rosetta 2 下
如果你的終端機不小心勾選了「使用 Rosetta 打開」，`rustup` 可能會誤裝成 Intel (`x86_64-apple-darwin`) 版本，導致編譯速度大打折扣。
* 在終端機執行 `rustc -vV`，確認輸出中的 `host:` 為：
  * Apple Silicon Mac：`host: aarch64-apple-darwin`
  * Intel Mac：`host: x86_64-apple-darwin`

### 3. macOS 系統更新後出現 `xcrun: error: invalid active developer path`
每當 macOS 進行大版本更新（例如升級到新版 macOS）後，原本的 Xcode Command Line Tools 路徑有時會失效，導致 `cargo build` 報錯找不到連結器。
* **解法**：只要在終端機重新執行一次以下指令即可修復：
  ```zsh
  xcode-select --install
  ```