# 專案介紹: PhotoPainter_B

## 專案概述
PhotoPainter_B 是一個基於 Raspberry Pi Pico 2 (RP2350) 與 Waveshare 7.3 吋 6 色電子紙 (e-Paper) 的電子相框開源專案。相框從 SD 卡讀取圖片並顯示在低功耗的電子紙上，支援兩種圖片格式：

- **BMP**：事先用 `convert.py` 轉好的 6 色抖動圖片，讀取最快。
- **JPG**：直接放原圖，由相框自己解碼、縮放裁切並做 Floyd-Steinberg 抖動，不必先在電腦上轉檔。

專案內建電池管理、RTC (即時時鐘) 定時喚醒、按鍵換圖，以及 **USB 磁碟模式**：接上電腦後長按按鍵，SD 卡就會以隨身碟的形式出現在電腦上，不必拔卡就能更換圖片。

---

## 系統工作流程 (Workflow)

系統的主要執行流程位於 `main.c`，核心邏輯圍繞在「極致省電」的原則下運行。步驟如下：

0. **開機模式判斷** (`usb_disk_boot_check()`):
   - 開機第一步先鎖住電池電源 (`BAT_OFF` 拉高)，再判斷這次要進入哪一種模式。
   - 若是長按 NEXT 後重開機，或接著 USB 開機時按住 NEXT 3 秒，就直接進入 **USB 磁碟模式** (見下方專節)，不會執行後續的相框流程。
   - 若是剛離開 USB 磁碟模式，開機後會立刻刷新一張圖片，讓新放入的圖片馬上生效。

1. **硬體與模組初始化**:
   - 初始化周邊硬體 (GPIO, SPI, I2C 等介面)。
   - 啟動看門狗計時器 (Watchdog, 8 秒) 防止系統死機。
   - 初始化 RTC (PCF85063) 模組並設定下一次喚醒時間 (預設 1 小時，見 `main.c` 的 `alarmTime`)。
   - 註冊充電狀態中斷回呼函數。

2. **電池電量檢測**:
   - 系統啟動後，立刻透過 ADC 讀取電池電壓 (`measureVBAT()`)。
   - 若電壓過低 (< 3.1V)，則進入低電量保護模式：LED 閃爍提示、關閉 RTC 喚醒功能，並自動切斷電源 (`powerOff()`) 避免電池過放。

3. **SD 卡掃描與文件讀取**:
   - 檢測是否插入 SD 卡 (`sdTest()`)。若上次執行有異常 (例如被看門狗重設)，會把原因寫進 `usbdisk_log.txt` (見「診斷記錄」)。
   - 圖片放在 SD 卡的 `pic/` 資料夾，副檔名為 `.bmp`、`.jpg` 或 `.jpeg` 的檔案才會被列入播放清單 `fileList.txt`；隱藏檔與 macOS 的 `._` 檔案會被略過。
   - 播放模式由 `main.c` 的 `PlayMode` 決定 (預設為 3)：
     - **Mode 0**: 自動掃描 `pic/` 並依檔名排序。
     - **Mode 1**: 自動掃描 `pic/`，不排序。
     - **Mode 2**: 不自動掃描，使用者需自行建立 `fileList.txt` 指定圖片順序。
     - **Mode 3**: 自動掃描 `pic/`，隨機播放 (不會連續顯示同一張)。

4. **顯示與電源狀態管理**:
   - 顯示時依副檔名分流：`.bmp` 走原本的 BMP 讀取，`.jpg`/`.jpeg` 由相框解碼 (見「支援的圖片格式」)。
   - **未充電狀態 (使用電池)**:
     讀取並顯示圖片 (`run_display()`)，重新設定 RTC 喚醒時間，最後**直接切斷系統總電源** (`powerOff()`)。系統將完全斷電，直到下一次 RTC 喚醒或使用者按下按鍵。
     若刷新途中接上了 USB，斷電後系統仍由 USB 供電，此時會自動回到充電狀態的迴圈，按鍵照常可用。
   - **充電狀態 (連接 USB VBUS)**:
     進入持續運行循環 (每 200 ms 檢查一次)，監聽充電狀態並切換 LED 指示燈。
     - **短按 NEXT**：刷新顯示下一張圖片。
     - **長按 NEXT 3 秒**：重開機進入 USB 磁碟模式。
     - 充電時預設不輪詢 GPIO6 的 RTC 中斷 (`enChargingRtc` 為 0)。依硬體註解，S1 開啟時 RTC 會改經由 `BAT_STATE` 的喚醒電路觸發刷新。

---

## 支援的圖片格式

| | BMP | JPG |
|---|---|---|
| 來源 | 用 `convert.py` 事先轉好 | 直接放原圖 |
| 格式要求 | 24-bit，800×480 或 480×800 | baseline JPEG (一般相機、手機拍的照片) |
| 尺寸 | 必須剛好是面板尺寸 | 任意尺寸，自動縮放並置中裁切 |
| 讀取時間 | 約 1～2 秒 | 480×800、500 KB 以下約 1～2 秒；大圖會更久 |

### JPG 的處理方式
JPG 解碼程式在 `lib/GUI/GUI_JPGfile.c`，使用 ChaN 的 TJpgDec (`lib/tjpgd/`)。為了在 RP2350 有限的記憶體裡處理，整張圖不會一次解開，而是邊解碼邊處理：

1. TJpgDec 一次解出一列 MCU (8 或 16 列像素)。原圖很大時，會在解碼階段先縮小為 1/2、1/4 或 1/8。
2. 縮放並置中裁切成 800×480 (橫圖) 或 480×800 (直圖)，規則與 `convert.py` 的預設 `scale` 模式相同。
3. 用與 `convert.py` 相同的 6 色調色盤做 Floyd-Steinberg 抖動，直接寫進 192 KB 的畫面緩衝。

除了 192 KB 的畫面緩衝，解碼時額外約需 16 KB 解碼器工作區、最多約 77 KB 的條帶緩衝，以及約 10 KB 的抖動誤差緩衝。

### JPG 的限制
- **不支援 progressive JPEG**。部分修圖軟體或「儲存為網頁用」會輸出這種格式，畫面會顯示 `JPG error 8: unsupported JPEG (progressive?)`。重新存成一般 (baseline) JPEG 即可。
- **不支援 HEIC**。iPhone 請在「設定 > 相機 > 格式」改成「最相容」，或先轉成 JPG。
- **不讀 EXIF 旋轉資訊**。直拍照片若依賴 EXIF 才轉正，在相框上會是橫躺的，請先把圖片本身轉正。
- 縮放採用最近鄰取樣，畫質比 `convert.py` 粗一些。檔案已經是 800×480 或 480×800 時不經過縮放，不受影響。

### 錯誤顯示
圖片無法顯示時，畫面會印出錯誤訊息與檔名，不會卡住：
- `JPG error N: ...`：JPG 解碼失敗，N 為 TJpgDec 的錯誤代碼 (例如 8 為 progressive JPEG)。
- `Not a 24-bit 800x480 BMP`：BMP 不是 24-bit，或尺寸超過面板。

序列埠 (USB CDC) 會印出 `[JPG]` 與 `[TIME]` 開頭的 log，列出解碼各階段與電子紙刷新所花的時間。

---

## USB 磁碟模式

不必拔出 SD 卡，就能從電腦存取卡片內容。程式位於 `usb_msc/`。

### 進入方式
相框需接著 USB (有資料線的電腦連接埠，不是單純的充電器)：
- 開機狀態下**長按 NEXT 3 秒**，相框會重開機進入磁碟模式；或
- **按住 NEXT 再插上 USB**，持續按住 3 秒。

進入後電腦會出現一個隨身碟 (`PhotoPnt SD Card`)，直接把圖片複製到 `pic/` 資料夾即可。

### 燈號
| ACT 燈 | 意義 |
|---|---|
| 恆亮 | 磁碟模式，閒置中 |
| 快速閃爍 | 正在讀寫 SD 卡 |
| 每秒約 5 下 | SD 卡初始化失敗 (檢查卡片是否插好) |

PWR 燈的意義與一般模式相同：亮代表正在充電。

### 離開方式
- **在電腦上「退出」磁碟 (建議)**：相框會自動重開機並刷新一張圖片。
- **短按 NEXT**：同樣會重開機並刷新。請先在電腦上退出，避免檔案還沒寫完。
- **拔掉 USB**：USB 電源中斷 300 ms 以上才會判定為拔除，避免傳輸中電壓瞬間下降造成誤判。
- **15 秒內沒有連上電腦**：例如接的是充電器，會自動回到一般模式。

### 注意事項
- 磁碟模式期間由電腦掌管檔案系統，相框本身不會讀寫 SD 卡上的檔案。
- 進入磁碟模式時，SD 卡與電子紙共用的電源 (`EPD_POWER_EN`) 會保持開啟，並先等待 250 ms 讓卡片穩定。
- SD 卡在大量寫入時可能長時間忙碌。驅動程式在有上限的等待迴圈中會重設看門狗，所以慢卡只會讓傳輸變慢，不會被誤判為當機。若最慢寫入時間經常超過數秒，建議換一張速度較快的卡。

---

## 診斷記錄 (`usbdisk_log.txt`)

相框在異常結束或離開磁碟模式後，下次開機會把原因附加到 SD 卡根目錄的 `usbdisk_log.txt`，同時印在序列埠上。只有在有事件可記錄時才會寫入，平常開機不會。可以進入 USB 磁碟模式後，從電腦直接打開這個檔案。

| 記錄內容 | 意義 |
|---|---|
| `USB disk mode ended: ejected by the computer, slowest SD write X s` | 正常退出；附上這次最慢的一次 SD 寫入時間 |
| `USB disk mode ended: NEXT key pressed` / `USB power lost` / `no USB host for 15 s` | 按鍵、拔線或沒連上電腦而離開 |
| `USB disk mode ended: watchdog timeout (stuck > 8 s in SD write ...)` | 磁碟模式中卡住超過 8 秒，並註明是否卡在 SD 讀寫 |
| `..., SD errors: write N, read M` | 磁碟模式中 SD 讀寫失敗的次數 |
| `watchdog timeout (stuck > 8 s) during <步驟>, detail N` | 一般模式卡住的位置，例如 `JPG decode` (N 為處理到第幾列)、`BMP read`、`e-Paper refresh` |
| `boot after brown-out` | 電源電壓不足導致重開機 |

原理：重開機時 RP2350 的 watchdog scratch 暫存器會保留內容。韌體在執行各步驟時把進度與原因寫進暫存器，下次開機再讀出來 (見 `usb_msc/usb_disk.c` 的 `record_last_exit()` 與 `lib/Config/DEV_Config.c` 的 `DEV_SetStage()`)。完全斷電時暫存器會被清空，所以斷電造成的重開機不會有記錄。

---

## 各部件功能說明 (Component Functions)

專案目錄結構主要由 `lib` (底層硬體與模組驅動)、`examples` (應用層代碼) 與 `usb_msc` (USB 磁碟模式) 等資料夾組成，分別負責不同的功能：

### 1. 核心控制與硬體設定
- **`main.c`**: 程式主入口，負責開機模式判斷、總體流程控制、電源管理與按鍵事件 (短按換圖、長按進入 USB 磁碟模式)。
- **`lib/Config/`**: 硬體設定層。`DEV_Config` 定義 Pico 腳位分配 (GPIO mapping)、硬體介面初始化 (`DEV_Module_Init()`；磁碟模式用的 `DEV_Hardware_Init()` 不啟動 USB 序列埠) 與進度記錄 `DEV_SetStage()`。`hw_config.c` 設定 SD 卡使用的 SPI 腳位與驅動強度。

### 2. 電子紙與顯示驅動 (Display)
- **`lib/e-Paper/`**: 電子紙底層驅動。包含與 Waveshare 7.3 吋電子紙 (`EPD_7in3e`) 通訊、發送初始化參數、刷新指令與傳送圖片數據的驅動程式碼。
- **`lib/GUI/` 及 `lib/Fonts/`**: 點陣圖形繪製庫 (畫點、線、圓、矩形、文字) 以及字庫。圖片讀取也放在這裡：
  - `GUI_BMPfile.c`：讀取 24-bit BMP，不符合格式的檔案會回報錯誤。
  - `GUI_JPGfile.c`：JPG 串流解碼、縮放裁切與 6 色抖動。
- **`lib/tjpgd/`**: ChaN 的 TJpgDec R0.03 JPEG 解碼器。設定 (`tjpgdcnf.h`) 調整為 `JD_SZBUF 2048`、`JD_FASTDECODE 2`，用約 16 KB RAM 換取較快的解碼速度。

### 3. 文件系統與存儲 (Storage)
- **`lib/FatFs_SPI/`**: 基於 SPI 介面的 SD 卡底層驅動及開源的 FatFs 文件系統。用於掛載 SD 卡、讀取目錄結構與讀寫檔案。
- **`examples/run_File.c`**: 應用層的文件處理邏輯，負責掃描 `pic/` 中的圖片檔、管理播放清單與索引，以及寫入診斷記錄 (`sdAppendLog()`)。
- **`examples/EPD_7in3e_test.c`**: 顯示流程 `EPD_7in3e_display_BMP()`：依副檔名選擇 BMP 或 JPG 讀取、處理錯誤顯示，並記錄各階段時間。

### 4. USB 磁碟模式 (`usb_msc/`)
- **`usb_disk.c`**: 磁碟模式本體，以 TinyUSB 的 MSC (Mass Storage) 類別把 SD 卡區塊直接提供給電腦，並負責進出模式的判斷、燈號與診斷記錄。
- **`usb_descriptors.c`** / **`tusb_config.h`**: USB 描述元與 TinyUSB 設定。一般模式只提供 CDC (USB 序列埠 log)，磁碟模式只提供 MSC。

### 5. 電源與時間管理 (Power & Time)
- **`lib/RTC/`**: 外部即時時鐘模組 (`PCF85063`) 驅動程式。負責時間的設定與讀取，最重要的是提供獨立的定時喚醒 (Alarm) 中斷訊號，實現電子相框在斷電下的定時更新。
- **`lib/led/`**: LED 狀態指示燈控制，用於在系統低電量、充電中或充電完成時顯示不同的燈號，以及電源鎖存 (`powerOff()`)。

### 6. 圖片轉換支援 (`convert.py`)
相框已能直接顯示 JPG，`convert.py` 現在是選用工具。適合用在想先在電腦上預覽抖動結果、使用 `cut` (留白) 模式，或想讓相框讀取更快的情況。腳本會把圖片調整為面板解析度，透過 Floyd-Steinberg 抖動轉成 6 色，輸出 `.bmp` 檔案。

**環境需求**:
執行此腳本需要安裝 Python 及 Pillow (PIL) 影像處理套件：
```bash
pip install Pillow
```

**常用指令**:
- **基本轉換** (預設使用 scale 模式及 Floyd-Steinberg 抖動):
  ```bash
  python convert.py input.jpg
  ```
- **指定顯示方向**:
  可以使用 `--dir` 指定 `landscape` (橫向 800x480) 或 `portrait` (直向 480x800)；未指定時依原圖長寬自動判斷：
  ```bash
  python convert.py input.jpg --dir landscape
  ```
- **指定畫面處理模式**:
  可以使用 `--mode` 指定 `scale` (等比例放大填滿畫面，超出部分置中裁切，預設值) 或 `cut` (等比例縮小放進畫面，空白處補白)：
  ```bash
  python convert.py input.jpg --mode cut
  ```
- **關閉色彩抖動** (純色塊顯示):
  ```bash
  python convert.py input.jpg --dither 0
  ```
轉換完成後，腳本會在同目錄下生成帶有後綴的 `.bmp` 檔案 (例如 `input_scale_output.bmp`)，將其複製到 SD 卡的 `pic/` 資料夾即可。

---

## 開發備註

修改 SD 卡或電源相關程式前，值得先知道的幾件事：

- **`hw_config.c` 的欄位覆蓋問題**：原廠版本誤把驅動強度常數寫進 `.mosi_gpio`、`.sck_gpio`、`.ss_gpio`，蓋掉了腳位號碼，導致真正的 CS (GPIO5) 從未被驅動程式控制。讀取大多能動，但多區塊寫入會卡住。上游 `a1ce5e7` 已修正，本專案也已合併。
- **與上游的差異**：SD 的 MOSI/SCK 驅動強度設為 4 mA (RP2350 預設值，實測大檔案傳輸穩定；上游為 2 mA)。`lib/FatFs_SPI/sd_driver/spi.c` 保留實測通過的 DMA 等待方式，未採用上游的輪詢版本，之後可再單獨評估。
- **播放清單的行尾**：`f_gets` 會保留 `\r\n`。FatFs 開檔時會忽略它，但判斷副檔名時不會，所以 `fil2array()` 讀出路徑後會先去掉行尾空白。
- **看門狗 (8 秒)**：長時間的迴圈都必須定期呼叫 `watchdog_update()` (或使用會順便更新的 `DEV_Delay_ms()`)。SD 驅動的忙碌等待迴圈 (`sd_wait_ready()`、`sd_wait_token()`) 也會更新看門狗。
- **ACT 燈 (GPIO25)**：SD 驅動在選取／取消選取卡片時也會控制這顆燈 (`spi.h` 的 `LED_ON()`/`LED_OFF()`)，所以 SD 存取期間燈號會跟著變化。
- **建置**：`build/` 已設定好 CMake，執行 `make -C build epd` 會產生 `build/epd.uf2`。
