# 使用 Rust 開發 Edge Device 與嵌入式專案指南（ESP32、Arduino 與微控制器實戰）

長期以來，嵌入式與邊緣裝置（Edge Devices）的世界幾乎完全由 C 與 C++ 統治。但近年來，**Rust 已成為嵌入式開發最具爆發力的新選擇**——包含 Espressif（樂鑫，ESP32 原廠）、Raspberry Pi、Nordic Semiconductor 等晶片大廠都已投入大量資源支援 Rust。

這份指南將帶您從零開始，了解如何用 Rust 打造穩定、省電且現代化的 Edge Device 專案。

---

## 一、先釐清觀念：Edge Device 的兩大類

在開始之前，先區分您手上的邊緣裝置屬於哪一種層級，因為兩者的 Rust 開發方式截然不同：

1. **Linux 邊緣運算裝置（MPU，例如 Raspberry Pi、NVIDIA Jetson、BeagleBone）**
   * **特點**：上面跑的是完整的 Linux 作業系統，記憶體動輒 512MB～16GB。
   * **開發方式**：**跟寫一般的電腦程式完全一樣！** 可以直接使用完整的標準函式庫（`std`）、`Tokio`、`Reqwest`、AI 推論框架（如 `candle`、`ort`）。只需在開發機上透過 `cross` 或 `cargo-zigbuild` 交叉編譯成 ARM Linux 執行檔，丟進裝置裡跑即可。
2. **微控制器（MCU，例如 ESP32、Arduino、STM32、Raspberry Pi Pico）**
   * **特點**：沒有完整作業系統（或只跑輕量級 RTOS），RAM 往往只有幾 KB 到幾百 KB。
   * **開發方式**：這才是真正的「嵌入式 Rust（Embedded Rust）」，需要直接與硬體暫存器、腳位（GPIO）互動，也是本文接下來要深入探討的重點。

---

## 二、為什麼用 Rust 寫微控制器（比 C/C++ 好在哪）？

1. **型別系統防呆（Type-State Pattern）**：
   在 C 語言中，你不小心把「設為輸入（Input）」的腳位拿來做「輸出（Output）」，或是兩個模組同時搶用同一個 I2C 匯流排，編譯器完全不會管你，直到通電燒毀或當機才發現。
   在 Rust 中，**硬體腳位也是具備「所有權」的物件**。如果你把 `GPIO2` 交給了 LED 驅動程式，其他地方想再拿 `GPIO2` 來用，**編譯期就會直接報錯**！
2. **無 GC 且極致省資源**：
   Rust 沒有垃圾回收器，記憶體配置完全可預測，不會在感測器中斷（Interrupt）的關鍵時刻發生卡頓，且編譯出來的韌體體積與執行效能與 C 語言同級。
3. **告別痛苦的套件管理**：
   在傳統嵌入式 C 開發中，引入一個溫濕度感測器或 OLED 螢幕的驅動往往要手動複製貼上 `.c`/`.h` 檔、改 `CMakeLists.txt`。在 Rust 中，只要在 `Cargo.toml` 加上一行相依套件，就能直接使用社群寫好的跨晶片驅動程式。

---

## 三、嵌入式 Rust 的核心架構：`#![no_std]` 與 `embedded-hal`

在沒有作業系統的微控制器上，Rust 程式碼開頭通常會看到這兩行宣告：

```rust
#![no_std]  // 不連結依賴作業系統的標準函式庫 (std)，只使用核心庫 (core)
#![no_main] // 不使用作業系統的標準 main 進入點，改由微控制器的硬體啟動流程接管
```

為了讓「同一份感測器驅動程式」能同時在 ESP32、STM32、Arduino、Pico 上運行，嵌入式 Rust 社群建立了一套極為優雅的**四層抽象架構**：

```
┌───────────────────────────────────────────────────┐
│ 應用程式 (你的業務邏輯，例如溫濕度監控、MQTT 上報)  │
├───────────────────────────────────────────────────┤
│ 跨平台驅動程式 (Driver Crates，例如 ssd1306 螢幕)   │  ◄── 只依賴 embedded-hal 介面
├───────────────────────────────────────────────────┤
│ 硬體抽象層 (HAL，如 esp-hal / avr-hal / rp-hal)    │  ◄── 實作 embedded-hal Trait
├───────────────────────────────────────────────────┤
│ 周邊存取層 (PAC，自動從晶片廠 SVD 檔產生的暫存器定義)│
└───────────────────────────────────────────────────┘
```

只要某個感測器套件（例如 BME280 氣壓計）是基於標準的 **`embedded-hal`** Trait 寫的，不管你今天把晶片從 STM32 換成 ESP32-C6，你的感測器程式碼幾乎**一行都不用改**！

---

## 四、實戰路線 1：ESP32 系列（目前 Rust 支援度最頂尖的 IoT 晶片）

如果你要做具備 Wi-Fi / 藍牙功能的 Edge Device，**ESP32 是目前 Rust 體驗最好的硬體平台**，因為原廠 Espressif 有專門團隊長期維護 Rust 工具鏈。

### 1. 挑選晶片架構的小眉角
* **RISC-V 架構（如 ESP32-C3、C6、H2、P4）**：**強烈推薦新手首選！** 因為官方 `rustc` 原生支援 RISC-V，不需要額外安裝特殊編譯器。
* **Xtensa 架構（如經典 ESP32、ESP32-S2、S3）**：由於 LLVM 上游對 Xtensa 支援較晚，需要透過官方工具 **`espup`** 安裝 Espressif 維護的專用 Rust 工具鏈。

### 2. ESP32 開發的兩條路線

| 路線 | 核心套件 | 特色與適用場景 |
| :--- | :--- | :--- |
| **裸機純 Rust（`no_std`）** | **`esp-hal`** ＋ **`Embassy`** | **目前官方主推的新世代路線**。100% 純 Rust、不依賴 C/ESP-IDF、編譯極快、韌體體積小、支援原生 `async/await` 與 Wi-Fi/BLE。 |
| **標準函式庫（`std`）** | **`esp-idf-hal`** ＋ **`esp-idf-svc`** | 底層封裝官方 C 語言的 ESP-IDF 與 FreeRTOS。優點是可以使用 Rust 標準函式庫 `std`（如 `Vec`、`String`、`std::thread`）以及所有 ESP-IDF 進階網路協定；缺點是編譯環境較重。 |

### 3. 快速建立與燒錄 ESP32 專案（`no_std` 路線）

```bash
# 1. 安裝燒錄工具與官方專案產生器
cargo install espflash esp-generate

# （若使用 Xtensa 晶片如 ESP32/S3，需額外安裝工具鏈）
# cargo install espup && espup install

# 2. 互動式建立新專案（可勾選晶片型號、是否啟用 Embassy 非同步、Wi-Fi 等）
esp-generate --chip=esp32c3 my-esp-project
cd my-esp-project

# 3. 插上 USB 線，一鍵編譯、燒錄並開啟序列埠監控！
cargo run --release
```

### 4. ESP32 程式碼範例（使用 `esp-hal` 讓 LED 閃爍）

```rust
#![no_std]
#![no_main]

use esp_backtrace as _;
use esp_hal::{
    delay::Delay,
    gpio::{Level, Output, OutputConfig},
    main,
};

#[main]
fn main() -> ! {
    // 1. 初始化晶片周邊，取得所有硬體資源的「所有權」
    let peripherals = esp_hal::init(esp_hal::Config::default());

    // 2. 將 GPIO2 設定為輸出模式（所有權移交給 led，其他人無法再誤用 GPIO2）
    let mut led = Output::new(peripherals.GPIO2, Level::Low, OutputConfig::default());
    let delay = Delay::new();

    loop {
        led.toggle(); // 切換亮暗
        delay.delay_millis(500);
    }
}
```

---

## 五、實戰路線 2：Arduino 系列

提到 Arduino 時，必須先區分你手上的是**「傳統 8-bit AVR 核心」**還是**「現代 32-bit 核心」**：

* **傳統 Arduino（Uno R3、Nano、Mega 2560）**：使用 Microchip AVR 8-bit 晶片。
  * **開發工具**：使用社群維護的 **`avr-hal`** 搭配 **`ravedude`**（負責呼叫 `avrdude` 燒錄並開啟序列埠監控）。
  * **注意事項**：由於 AVR 架構較老舊，目前通常需要切換到 **Nightly Rust**，並使用專案樣板來設定複雜的編譯參數。
* **現代 Arduino（Arduino Uno R4、Nano ESP32、Nano RP2040）**：底層其實就是主流的 ARM Cortex-M（Renesas RA4M1 / RP2040）或 ESP32 晶片，直接走對應晶片的現代 Rust HAL 即可，體驗好上許多。

### 傳統 Arduino Uno（AVR）快速上手

```bash
# 1. 安裝專案產生器與燒錄輔助工具 ravedude
cargo install cargo-generate ravedude

# 2. 使用 avr-hal 官方樣板建立專案（過程中選擇 Arduino Uno）
cargo generate --git https://github.com/Rahix/avr-hal-template.git

# 3. 插上 Arduino Uno，直接編譯並燒錄
cd my-arduino-project
cargo run
```

### Arduino Uno 程式碼範例（`avr-hal`）

```rust
#![no_std]
#![no_main]

use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);

    // 取得板載 LED（Pin 13）並設為輸出模式
    let mut led = pins.d13.into_output();

    loop {
        led.toggle();
        arduino_hal::delay_ms(1000);
    }
}
```

> **給新手的真心建議**：如果您還沒購買硬體，單純想學習用 Rust 寫嵌入式系統，**強烈建議從 ESP32-C3 / C6 或 Raspberry Pi Pico（RP2040 / RP2350）開始**，而不是傳統 8-bit 的 Arduino Uno。前兩者價格極便宜（約台幣 100～200 元）、支援 Stable Rust、記憶體寬裕，且能完整體驗現代 Rust 嵌入式生態的威力。

---

## 六、現代嵌入式 Rust 的殺手鐧：Embassy 非同步框架

在傳統 C 語言寫微控制器時，如果你想「同時閃爍 LED、每 5 秒讀取感測器、同時等待 Wi-Fi 封包與按鈕中斷」，你通常有兩種痛苦的選擇：
1. 手寫極難維護的龐大狀態機（Super-loop）。
2. 引入 FreeRTOS 等即時作業系統，為每個任務分配獨立的堆疊記憶體（浪費寶貴的 RAM）。

Rust 社群打造的 **`Embassy`** 框架徹底改變了這件事：它讓你可以在**沒有作業系統、沒有堆積記憶體（Heap）的裸機微控制器上，直接寫原生的 `async / await` 程式碼！**

```rust
// 概念範例：在極小的微控制器上同時「併發」執行兩個任務，且零 OS 負擔
#[embassy_executor::task]
async fn blink_task(mut led: Output<'static>) {
    loop {
        led.toggle();
        Timer::after_millis(500).await; // 等待時 CPU 自動進入低功耗休眠！
    }
}

#[embassy_executor::task]
async fn sensor_task() {
    loop {
        let temp = read_temperature().await;
        send_to_mqtt(temp).await;
        Timer::after_secs(10).await;
    }
}
```
Embassy 支援 **ESP32、STM32、Raspberry Pi Pico（RP2040/RP2350）與 Nordic nRF**，且因為編譯器會將 `async` 轉為極輕量的靜態狀態機，當沒有任務在跑時會自動讓晶片進入省電模式（WFI/Sleep），非常適合電池供電的 Edge Device。

---

## 七、嵌入式 Rust 必備工具箱（包含零硬體模擬器！）

1. **`Wokwi`（瀏覽器線上模擬器 — 強烈推薦！）**
   手邊還沒買開發板沒關係！[Wokwi](https://wokwi.com/) 是一個免費的線上硬體模擬器，**原生支援 Rust 編寫 ESP32、Raspberry Pi Pico 與 Arduino**，你可以直接在瀏覽器裡接上虛擬 LED、OLED 螢幕、溫濕度感測器，甚至模擬 Wi-Fi 連線來測試你的 Rust 程式碼。
2. **`probe-rs`（ARM 與 RISC-V 除錯神器）**
   如果你使用 STM32、Pico 或 nRF 搭配除錯器（如 ST-Link、J-Link、CMSIS-DAP），只要安裝 `probe-rs`，就能直接用 `cargo run` 完成編譯、燒錄，並即時在終端機看到晶片印出的 Log，甚至直接在 VS Code 設中斷點單步除錯。
3. **`defmt`（Deferred Formatting）**
   在微控制器上做 `println!` 字串格式化非常佔用 Flash 空間與 CPU 時間。`defmt` 透過黑科技把字串格式化的工作「延後到你的電腦端才處理」，微控制器只傳送幾個 Byte 的編號，讓除錯 Log 既詳細又不佔空間。

---

## 八、總結：您的第一步可以怎麼走？

* **想馬上動手體驗，但手邊沒板子**：去 **Wokwi** 開啟一個 **Rust on ESP32-C3** 或 **RP2040** 範例直接寫 Code。
* **要做聯網 IoT 裝置（Wi-Fi / 藍牙 / Matter）**：買一塊 **ESP32-C3 或 ESP32-C6**，使用 `esp-generate` 搭配 **`esp-hal` + `Embassy`**。
* **要做工控、馬達控制或學習底層除錯**：買一塊 **Raspberry Pi Pico** 或 **STM32 Nucleo**，搭配 **`Embassy` + `probe-rs`**。
* **要做影像辨識 / AI 邊緣推論**：直接買 **Raspberry Pi 5** 或 **Jetson Orin Nano**，當作標準 Linux 環境開發一般 Rust 程式即可。