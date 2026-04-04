# minimal-keys 改動說明

## 目錄

- [第一部分：這把鍵盤的硬體](#第一部分這把鍵盤的硬體)
- [第二部分：左半翻轉 180°](#第二部分左半翻轉-180)
  - [為什麼要翻轉](#為什麼要翻轉)
  - [翻轉後的物理位置對應](#翻轉後的物理位置對應)
  - [翻轉後的配列](#翻轉後的配列)
- [第三部分：moNa2 風格層結構](#第三部分mona2-風格層結構)
  - [9 層設計](#9-層設計)
  - [Combo 組合鍵](#combo-組合鍵)
  - [Hold-Tap 設定](#hold-tap-設定)
  - [Encoder 旋鈕](#encoder-旋鈕)
  - [C++ 巨集](#c-巨集)
- [第四部分：PMW3610 驅動衝突事件（踩坑紀錄）](#第四部分pmw3610-驅動衝突事件踩坑紀錄)
  - [發生了什麼](#發生了什麼)
  - [嘗試一：移除外部驅動，用 Zephyr 內建的](#嘗試一移除外部驅動用-zephyr-內建的)
  - [嘗試二：用 Kconfig 關掉內建驅動](#嘗試二用-kconfig-關掉內建驅動)
  - [最終解法：降版 ZMK 到 v0.3-branch](#最終解法降版-zmk-到-v03-branch)
  - [教訓與建議](#教訓與建議)
- [第五部分：所有改動的檔案清單](#第五部分所有改動的檔案清單)
- [第六部分：CI/CD 與刷韌體](#第六部分cicd-與刷韌體)

---

## 第一部分：這把鍵盤的硬體

minimal-keys 是基於 [roBa](https://github.com/kumamuk-git/roBa) / roBaish 設計的分離式鍵盤，由 [hyhy3dp (minimal-keys)](https://hyhy3dp.base.shop/) 組裝販售。

```
┌─────────────────────┐        BLE        ┌─────────────────────┐
│      左半（L）        │ ◄──────────────► │      右半（R）        │
│    Peripheral        │                   │    Central           │
│                      │                   │                      │
│  • 43 鍵（含 encoder）│                   │  • 43 鍵             │
│  • EC11 旋鈕          │                   │  • PMW3610 軌跡球     │
│  • RGB LED           │                   │  • RGB LED           │
│  MCU: XIAO BLE       │                   │  MCU: XIAO BLE       │
│  (nRF52840)          │                   │  (nRF52840)          │
└─────────────────────┘                   └─────────────────────┘
```

**特殊之處：**
- **Encoder 在 Q 的位置**（左半左上角）：旋鈕下壓 = 按鍵，旋轉 = 感測器輸入
- **PMW3610 軌跡球**在右半，使用 [hyhy-masa 的客製驅動](https://github.com/hyhy-masa/zmk-pmw3610-driver)
- **43 鍵**：4 行，Row 1 有 10 鍵，Row 2-3 各 12 鍵，Row 4（拇指）9 鍵
- **RGB LED**：使用 [caksoylar/zmk-rgbled-widget](https://github.com/caksoylar/zmk-rgbled-widget) + `rgbled_adapter` shield

---

## 第二部分：左半翻轉 180°

### 為什麼要翻轉

這把鍵盤的 **Encoder（旋鈕）在 Q 的位置（左上角）**。旋鈕的下壓開關阻力很大，導致 Q 鍵很難按。

**解法：把左半物理翻轉 180°。** 翻轉後：
- Encoder 從左上角移到**右下角**（拇指附近），變成自然的捲輪位置
- Q 從 encoder 搬到正常的按鍵位置，打字不再費力
- 拇指列（原本在底部）翻到頂部，頂部字母列翻到底部

### 翻轉後的物理位置對應

翻轉 180° 後，每個 keymap 位置（硬體矩陣）對應的**視覺位置**反轉：

```
原始（未翻轉）：                    翻轉 180° 後：
Row 1: [0] [1] [2] [3] [4]        → 變成底部 thumb row（右到左）
Row 2: [10][11][12][13][14][15]    → 變成第 3 行（右到左）
Row 3: [22][23][24][25][26][27]    → 變成第 2 行（右到左）
Row 4: [34][35][36][37][38][39]    → 變成頂部行（右到左）
```

**具體對應表：**

| 視覺位置（翻轉後） | Keymap 位置 | 按鍵 |
|-------------------|-----------|------|
| 頂行左1 (pinky) | [39] | Q |
| 頂行左2 | [38] | W (hold=DESKTOP) |
| 頂行左3 | [37] | E |
| 頂行左4 | [36] | R |
| 頂行左5 | [35] | T |
| 頂行右 (inner) | [34] | CAPS |
| 中行左1 | [27] | A (hold=CTRL) |
| 中行左2 | [26] | S |
| 中行左3 | [25] | D |
| 中行左4 | [24] | F |
| 中行左5 | [23] | G |
| 中行右 (inner) | [22] | MB1 (滑鼠左鍵) |
| 下行左1 | [15] | Z (hold=SHIFT) |
| 下行左2 | [14] | X |
| 下行左3 | [13] | C |
| 下行左4 | [12] | V |
| 下行左5 | [11] | B |
| 下行右 (inner) | [10] | MB2 (滑鼠右鍵) |
| 拇指左1 | [4] | CTRL |
| 拇指左2 | [3] | OPT |
| 拇指左3 | [2] | CMD |
| 拇指左4 | [1] | SPACE (hold=VIM) |
| 拇指右 (encoder) | [0] | &none (encoder 下壓太重不用) |

> **重點：在 keymap 檔案中，左半的按鍵順序是「反」的。** 因為硬體位置不變（[0] 永遠是 Row 1 Col 0），但物理翻轉後 [0] 跑到右下角。所以 keymap 裡 Row 2 寫 `MB2 B V C X mt(LS,Z)`，視覺上看是 `Z X C V B MB2`。

### 翻轉後的配列

**左半：**
```
Q        W(桌面)  E       R       T       CAPS
A(CTRL)  S        D       F       G       MB1
Z(SHIFT) X        C       V       B       MB2
CTRL     OPT      CMD     SPC(VIM)        encoder(捲輪)
```

**右半（不翻轉，moNa2 風格）：**
```
Y       U        I       O       P
ESC     H        J       K       L       ;
RALT    N        M       ,       .       /(RSHIFT)
ENT(NUM)  BS(SYM)                        DEL(BT)
```

**設計理由：**
- **A hold=CTRL**（`mt(LCTRL, A)`）：因為翻轉後 thumb row 只有 4 個可用鍵（第 5 個是 encoder），CTRL 擠不下。把 CTRL 放在 A 的 hold 上，跟 [Home Row Mods](https://precondition.github.io/home-row-mods) 的理念相同
- **Thumb row = CTRL-OPT-CMD-SPC**：模仿 MacBook 筆電的 modifier 排列（Ctrl → Option → Command → Space）
- **MB1/MB2 在 inner column**（[22] 和 [10]）：翻轉後這些位置在左半靠近分體中間的邊緣，食指可及。右手操作軌跡球，左手食指按 MB1/MB2，分工明確
- **Encoder 旋轉 = 縱向捲動**：encoder 翻到拇指旁邊後，拇指自然地轉動就是捲動網頁

---

## 第三部分：moNa2 風格層結構

### 9 層設計

| 層 | 名稱 | 觸發方式 | 用途 |
|---|------|---------|------|
| 0 | Default | 預設 | QWERTY 打字 |
| 1 | SYM | 按住 Backspace [41] | 符號 `!@#$%^&*()` 等 |
| 2 | NUM | 按住 Enter [40] | 數字 1-0 + F1-F12 |
| 3 | MEDIA | 按住 左ALT [3] | 音量、亮度、播放控制 |
| 4 | BLUETOOTH | 按住 Delete [42] | 藍牙配對/切換/清除 |
| 5 | MOUSE | 軌跡球自動啟用 | 左手 MB1/MB2 已在 default 層 |
| 6 | SCROLL | 軌跡球自動捲動 | 驅動層級設定 |
| 7 | VIM | 按住 Space [1] | H=左 J=下 K=上 L=右 + Home/End/PgUp/PgDn |
| 8 | DESKTOP | 按住 W [38] | macOS 桌面切換 + Mission Control |

> 層的概念：鍵盤像一疊透明膠片。按住某個鍵時啟用上面的層，那一層有定義的鍵會覆蓋下面的，沒定義的（`&trans`）會透過去用下面的。[ZMK Layers 文件](https://zmk.dev/docs/keymaps#layers)

### Combo 組合鍵

| 組合 | 位置 | 功能 | 特殊設定 |
|------|------|------|---------|
| W + E | [38]+[37] | Tab | `slow-release` |
| J + K | [18]+[19] | Enter | `slow-release` |
| S + D | [26]+[25] | 切換輸入法 | `slow-release` + `&input_tap` macro |

> **注意：** 因為左半翻轉，combo 的位置編號跟 moNa2 不同（W 在 [38] 而非 [1]），但視覺上按的鍵是一樣的。

**`slow-release`**：兩個鍵都放開後才結束 combo。沒有這個的話，先放一個鍵可能導致 combo 重複觸發。[ZMK Combo 文件](https://zmk.dev/docs/keymaps/combos)

**`&input_tap` macro**：輸入法切換用 macro 代替 `&kp LC(SPACE)`。`&kp` 在按住期間會持續送出 Ctrl+Space，macOS 會不斷循環切換輸入法。macro 不管按多久只送一次。[ZMK Macro 文件](https://zmk.dev/docs/keymaps/behaviors/macros)

### Hold-Tap 設定

```c
&mt {  // Mod-Tap：按住=修飾鍵，點按=字母
    flavor = "balanced";
    quick-tap-ms = <0>;
    require-prior-idle-ms = <150>;  // 打字中直接判定 tap，防誤觸 Shift/Ctrl
};

&lt {  // Layer-Tap：按住=切層，點按=按鍵
    tapping-term-ms = <200>;        // 按住超過 200ms = hold（切層）
    quick-tap-ms = <200>;           // 200ms 內連按同鍵 = tap（如連打 Backspace）
    flavor = "balanced";
    // 不加 require-prior-idle-ms：讓切層隨時響應
};
```

**為什麼 `&mt` 有 `require-prior-idle-ms` 但 `&lt` 沒有？**
- `&mt` 管修飾鍵（A→CTRL、Z→SHIFT），打字快時容易誤觸，需要防護
- `&lt` 管切層（Backspace→SYM），是你主動要做的操作，加防護反而讓切層不順

> [ZMK Hold-Tap 文件](https://zmk.dev/docs/keymaps/behaviors/hold-tap)

### Encoder 旋鈕

翻轉後 encoder 在左手拇指旁邊（右下角），按層不同功能：

| 層 | 旋轉功能 | 說明 |
|---|---------|------|
| Default | 縱向捲動 | 瀏覽網頁最常用 |
| SYM | 水平捲動 | 看程式碼時橫向捲 |
| MEDIA | 音量 | 在媒體層控制音量 |
| 其他 | 縱向捲動 | 預設 |

Encoder 的**下壓**設為 `&none`（不觸發任何動作），因為旋鈕下壓阻力太大。

### C++ 巨集

SYM 層包含兩個 C++ 常用符號的 macro：

| 位置 | 符號 | 用途 | 實作 |
|------|------|------|------|
| `*` 下方 | `::` | scope resolution（`std::vector`） | `&kp LS(SEMICOLON) &kp LS(SEMICOLON)` |
| `\` 下方 | `->` | arrow operator（`ptr->member`） | `&kp MINUS &kp LS(DOT)` |

---

## 第四部分：PMW3610 驅動衝突事件（踩坑紀錄）

### 發生了什麼

改完 keymap 後 build 失敗，錯誤訊息：

```
devicetree error: both zmk-pmw3610-driver/dts/bindings/pixart,pmw3610.yml
and zephyr/dts/bindings/input/pixart,pmw3610.yaml
have 'compatible: pixart,pmw3610' and 'on-bus: spi'
```

**白話翻譯：** 有兩個地方都定義了「我是 PMW3610 的驅動」，系統不知道要用哪一個。

**根本原因：**

這把鍵盤使用 [hyhy-masa 的 PMW3610 客製驅動](https://github.com/hyhy-masa/zmk-pmw3610-driver)。這個驅動提供了很多客製功能（自動滑鼠層、捲動模式、移動閾值過濾等），是 ZMK 標準功能之外的。

但 ZMK 的 `main` 分支最近升級到了 **Zephyr 4.1**，而 [Zephyr 4.1 把 PMW3610 驅動加進了官方內建](https://docs.zephyrproject.org/4.1.0/boards/seeed/xiao_ble/doc/index.html)。於是：

```
┌──────────────────────────┐
│  hyhy-masa 外部驅動        │  compatible = "pixart,pmw3610"
│  (zmk-pmw3610-driver)    │  ← 有客製功能：automouse, scroll-layers, threshold
└──────────────────────────┘
            ⚡ 衝突！
┌──────────────────────────┐
│  Zephyr 4.1 內建驅動       │  compatible = "pixart,pmw3610"
│  (zephyr/drivers/input)  │  ← 基本功能：CPI, invert, 無 automouse
└──────────────────────────┘
```

兩個驅動用了**完全相同的 `compatible` 字串** (`pixart,pmw3610`)。Device Tree 編譯器（dtc）遇到兩個同名的 binding 就直接報錯，連 Kconfig 都還沒跑到。

### 嘗試一：移除外部驅動，用 Zephyr 內建的

**做法：**
1. 從 `west.yml` 刪除 `hyhy-masa/zmk-pmw3610-driver`
2. 改用 Zephyr 內建的 PMW3610 驅動
3. 把 overlay 改成內建驅動的語法（`motion-gpios` 取代 `irq-gpios`、加 `zephyr,axis-x/y` 等）
4. 用 ZMK 的 `zip_temp_layer` 輸入處理器取代驅動層級的 `automouse-layer`

**結果：Build 通過，但軌跡球異常（一直斜移）。**

**原因：** Zephyr 內建驅動缺少 hyhy-masa 驅動的客製功能：

| 功能 | hyhy-masa 驅動 | Zephyr 內建 |
|------|--------------|------------|
| 自動滑鼠層 (automouse-layer) | ✅ 驅動層級 | ❌ 需用 input processor |
| 捲動層 (scroll-layers) | ✅ 驅動層級 | ❌ 需用 input processor |
| 移動閾值過濾 (MOVEMENT_THRESHOLD) | ✅ `CONFIG_PMW3610_MOVEMENT_THRESHOLD=5` | ❌ 不支援 |
| 每軸獨立靈敏度 (X_SCALE/Y_SCALE) | ✅ `CONFIG_PMW3610_X_SCALE=130` | ❌ 不支援 |
| Polling Rate 設定 | ✅ `CONFIG_PMW3610_POLLING_RATE_250=y` | ❌ 用預設值 |
| 省電管理 (RUN_DOWNSHIFT) | ✅ `CONFIG_PMW3610_RUN_DOWNSHIFT_TIME_MS=3264` | ❌ 不同機制 |

> 缺少 `MOVEMENT_THRESHOLD` 是斜移的最可能原因：原本小於 5 單位的微動會被過濾掉，沒有過濾後微小雜訊就會讓游標飄移。

### 嘗試二：用 Kconfig 關掉內建驅動

**做法：** 在 `.conf` 加 `CONFIG_INPUT_PMW3610=n`，試圖關掉 Zephyr 內建驅動，讓 hyhy-masa 的驅動獨佔。

**結果：Build 失敗，錯誤一模一樣。**

**原因：** 衝突發生在 **Device Tree binding 層**，不是 Kconfig 層。

```
編譯順序：
1. Device Tree 編譯（dtc）  ← 在這裡就炸了，兩個 .yml 檔案衝突
2. Kconfig 處理              ← 根本跑不到這裡
3. C 編譯
```

`CONFIG_INPUT_PMW3610=n` 是 Kconfig 設定，在步驟 2 才生效。但步驟 1 就已經因為 binding 衝突而失敗了。所以 Kconfig 無法解決這個 Device Tree 層級的衝突。

> [Zephyr Device Tree 文件](https://docs.zephyrproject.org/latest/build/dts/index.html) — binding 檔案定義了 `compatible` 字串和驅動屬性

### 最終解法：降版 ZMK 到 v0.3-branch

**做法：** 把 ZMK 從 `main`（Zephyr 4.1）降到 `v0.3-branch`（Zephyr 3.5）。v0.3-branch **沒有內建 PMW3610 驅動**，所以不會衝突。

**改動：**

| 檔案 | 改前 | 改後 | 原因 |
|------|------|------|------|
| `west.yml` ZMK | `revision: main` | `revision: v0.3-branch` | 避免 Zephyr 4.1 內建驅動衝突 |
| `west.yml` rgbled-widget | `revision: main` | `revision: v0.3-branch` | 版本配套 |
| `west.yml` pmw3610-driver | 被刪除 | **恢復** `hyhy-masa/main` | 恢復客製驅動 |
| `build.yaml` board | `xiao_ble/nrf52840/zmk` | `seeeduino_xiao_ble` | v0.3 用舊版 board 名稱 |
| `.github/workflows/build.yml` | `@main` | `@v0.3.0` | workflow 也要配套 |
| `R.conf` | Zephyr 內建設定 | **恢復**原始 hyhy-masa 設定 | 恢復所有客製功能 |
| `R.overlay` | Zephyr 語法 | **恢復**原始 hyhy-masa 語法 | 恢復 automouse/scroll-layers |

**結果：Build 通過 ✅，軌跡球正常運作 ✅。**

### 教訓與建議

1. **ZMK `main` 不等於穩定版。** main 分支會持續更新，可能引入不相容的變更（如 Zephyr 大版本升級）。如果你的鍵盤依賴外部驅動模組，**用 `v0.3-branch`（穩定版）比 `main` 安全**。

2. **Device Tree 衝突無法用 Kconfig 解決。** 如果兩個模組定義了相同的 `compatible` 字串，唯一的解法是移除其中一個（要嘛不用外部驅動，要嘛用沒有內建驅動的 Zephyr 版本）。

3. **Board 名稱隨 Zephyr 版本變。** Zephyr 4.1 改了命名格式：
   - v0.3 / Zephyr 3.5：`seeeduino_xiao_ble`
   - main / Zephyr 4.1：`xiao_ble/nrf52840/zmk`（[Hardware Model v2](https://zmk.dev/blog/2025/12/09/zephyr-4-1#zmk-board-variant)）

4. **外部驅動 ≠ 劣質。** hyhy-masa 的驅動提供了很多 Zephyr 內建沒有的功能（automouse、scroll-layers、movement threshold、per-axis scaling）。在 Zephyr 內建驅動追上這些功能之前，外部驅動仍然是更好的選擇。

5. **原始 repo 可能已經壞了。** [hyhy-masa 的原始 repo](https://github.com/hyhy-masa/minimal-keys-release) 使用 `zmk@main`，這代表如果現在 fork 並 build，也會遇到同樣的衝突。原始 repo 能成功 build 是因為它是在 Zephyr 4.1 合併之前建的。

> 相關連結：
> - [ZMK v0.3 Release Notes](https://zmk.dev/docs)
> - [Zephyr 4.1 PMW3610 驅動](https://docs.zephyrproject.org/4.1.0/build/dts/api/bindings/input/pixart,pmw3610.html)
> - [hyhy-masa/zmk-pmw3610-driver](https://github.com/hyhy-masa/zmk-pmw3610-driver)
> - [ZMK Board Naming Changes (Zephyr 4.1)](https://zmk.dev/blog/2025/12/09/zephyr-4-1#zmk-board-variant)
> - [Zephyr Device Tree Bindings](https://docs.zephyrproject.org/latest/build/dts/bindings.html)

---

## 第五部分：所有改動的檔案清單

| 檔案 | 改動 | 說明 |
|------|------|------|
| `config/minimal-keys.keymap` | **重寫** | 左半翻轉 + moNa2 風格 9 層 |
| `config/boards/shields/minimal-keys/minimal-keys_R.overlay` | **修改** | 更新 automouse-layer=5、scroll-layers=6 |
| `config/boards/shields/minimal-keys/minimal-keys_R.conf` | **修改** | CPI 400→1600，恢復原始驅動設定 |
| `config/west.yml` | **修改** | ZMK main→v0.3-branch，恢復 hyhy-masa 驅動 |
| `build.yaml` | **修改** | board 名稱改回 seeeduino_xiao_ble |
| `.github/workflows/build.yml` | **修改** | workflow 配套改成 @v0.3.0 |

### Keymap 改動細節

**左半（翻轉 180°）：**
- Q 從 encoder 搬到正常按鍵 [38]
- Encoder [0] → rotation=捲輪、press=&none
- A hold=CTRL（`mt(LCTRL, A)`）
- Z hold=SHIFT（`mt(LSHIFT, Z)`）
- Inner column [22]=MB1、[10]=MB2
- Thumb：CTRL-OPT-CMD-SPC(VIM)

**右半（moNa2 風格）：**
- ESC 和 RALT 在 inner column
- ENTER(NUM)、BS(SYM)、DEL(BT) 在 thumb
- `mt(RSHIFT, SLASH)` 在右 pinky

**BT 層：**
- BT_CLR 和 BT_CLR_ALL 在左半 inner column（食指可及）
- 方向鍵在右半 K/M/,/. 位置
- bootloader 在右半

### Overlay 改動

```
scroll-layers = <3>;     →  scroll-layers = <6>;    // SCROLL 層號更新
（無 automouse-layer）     →  automouse-layer = <5>;  // 新增 MOUSE 層自動切換
```

### Conf 改動

```
CONFIG_PMW3610_CPI=400   →  CONFIG_PMW3610_CPI=1600  // 提高靈敏度，多螢幕夠快
```
其餘原始設定完整保留。

---

## 第六部分：CI/CD 與刷韌體

### 流程

```
改設定檔 → git push → GitHub Actions 自動編譯 → 下載 UF2 → 刷入鍵盤
```

### 刷入步驟

1. USB 接**右半**（Central）
2. 快速**雙擊 RST** → 出現 XIAO-SENSE 磁碟
3. 拖入 `minimal-keys_R` 的 UF2
4. 左半重複步驟（用 `minimal-keys_L` 的 UF2）

### 何時需要 settings_reset

- 左右半無法連線
- 要清除所有藍牙配對
- 刷完 reset 後：先右半再左半刷正常韌體
- 電腦上**刪除舊的 minimal-keys 藍牙**，重新配對

### 版本相容性總結

| 元件 | 版本 | 原因 |
|------|------|------|
| ZMK | `v0.3-branch` | 避免 Zephyr 4.1 PMW3610 衝突 |
| Board | `seeeduino_xiao_ble` | v0.3 的 board 名稱格式 |
| Workflow | `@v0.3.0` | 配套 ZMK 版本 |
| zmk-pmw3610-driver | `hyhy-masa/main` | 客製驅動，提供 automouse 等功能 |
| zmk-rgbled-widget | `caksoylar/v0.3-branch` | LED 狀態顯示，配套 ZMK 版本 |
