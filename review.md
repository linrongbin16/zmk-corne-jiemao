# ZMK Config 评审记录

| 项目 | 值 |
| --- | --- |
| 评审日期 | 2026-11-09 |
| 评审对象 | `linrongbin16/zmk-corne-jiemao` @ `6aef6a2`（"Swap GUI and Alt keys in the Mac layers, split Fun/Num/Sym into Mac and Win variants (#34)"） |
| ZMK 版本 | v0.3（tag `3cf9006`，commit `edf5c08`，配套 Zephyr 3.5.0） |
| 证据来源 | CI run `42566944808` / job `128995435853`（`corneish_zen_v2_left` + snippet `nrf52840-nosd`，构建日志 257KB）+ ZMK v0.3 源码树 |

## 总体结论

| 状态 | 数量 | 条目 |
| --- | --- | --- |
| ✅ 已修复 | 1 | 一、5 个死文件 |
| 📌 已确认（设计决定） | 1 | 二、墨水屏关闭 |
| 🔴 待处理 | 2 | 三、underglow 全死；四、DCDC 无处安放 |
| 🟡 待处理 | 4 | 五～八 |
| 🟢 待处理 | 1 | 九、零碎问题 |

这份 config **能正常构建、能正常使用**（CI 全绿，DTS/Kconfig 均无 error），但：

1. **有 5 个文件从未被任何构建路径读取**（`config/*_defconfig`、`config/Kconfig`、`config/Kconfig.board`、`config/Kconfig.defconfig`），写在里面的配置全部无效；→ **已修复**（PR #35，见第一条）
2. 因为 `CONFIG_ZMK_DISPLAY=n`，**墨水屏连同 SPI / EPD / LVGL 整体没编译**，overlay 里约 60 行面向屏幕的配置是纯 dead weight；→ **已确认是有意关闭屏幕**（见第二条），剩下的是清理工作
3. **underglow 从未生效**（`CONFIG_ZMK_RGB_UNDERGLOW` 未开启，`led_strip` 节点不参与编译）；
4. DCDC 的两次提交（`6037d1d` → `492c431`）来回挪动，**两次都无效**，因为位置始终不在 BOARD_ROOT 下。

实质起作用的只有 3 个文件：`config/corneish_zen.keymap`、`config/corneish_zen.conf`、`config/corneish_zen_v2_{left,right}.overlay`。

---

## 🔴 一、5 个文件是死文件（最严重）

> ### ✅ 状态：已修复（PR #35 · `717dac2`）
>
> 已删除以下 5 个文件，零功能变化——它们从未被读取：
>
> - `config/corneish_zen_v2_left_defconfig`
> - `config/corneish_zen_v2_right_defconfig`
> - `config/Kconfig`
> - `config/Kconfig.board`
> - `config/Kconfig.defconfig`
>
> **验证**：删除前后三个 UF2 产物（left 440832 B / right 341504 B / settings_reset 92672 B）**逐字节相同**，`image/` 下 SVG 与 YAML 亦相同。
>
> **遗留**：DCDC（第四条）需要单独决策——删掉死文件后 DCDC 仍然不生效，若要启用需走 `config/boards/` 布局真正接管 board 定义。

构建日志里只有这两行配置加载记录：

```
Loaded configuration '.../zmk/app/boards/arm/corneish_zen/corneish_zen_v2_left_defconfig'
Merged configuration '.../config/corneish_zen.conf'
```

仓库里的 `config/corneish_zen_v2_left_defconfig` / `config/corneish_zen_v2_right_defconfig` **一次都没有被加载**。

### 原因

`app/keymap-module/modules/modules.cmake`：

- defconfig 只按 `find_path(NOMES ${BOARD}_defconfig PATHS ${root}/boards/*/*)`（:95-103）在 BOARD_ROOT 下查找；
- `ZMK_CONFIG` 只有在存在 `config/boards/` 或 `config/dts/` 时才会被追加进 BOARD_ROOT（:45-62），而这条路径本身已打上 DEPRECATION。

本仓库这两个目录**从来没有过**（git log 确认）。

### 铁证

`config/corneish_zen_v2_left_defconfig:48` 写了 `CONFIG_USB_DEVICE_MANUFACTURER="corne_epd"`，但最终 `.config` 里是：

```
CONFIG_USB_DEVICE_MANUFACTURER="ZMK Project"
```

`corne_epd` 在 257KB 构建日志里出现 **0 次**。

### 另外三个 Kconfig 文件同类

`app/Kconfig:711-712` 只 `osource` 了两条路径：

```cmake
osource "$(ZMK_CONFIG)/boards/shields/*/Kconfig.defconfig"
osource "$(ZMK_CONFIG)/boards/shields/*/Kconfig.shield"
```

所以 `config/Kconfig`、`config/Kconfig.board`、`config/Kconfig.defconfig` **永远不会被 source**。其中 `config/Kconfig.board` 与上游 `app/boards/arm/corneish_zen/Kconfig.board` **逐字节相同**，等于白抄一份。

### 后果

- `CONFIG_BOARD_ENABLE_DCDC` / `_HV`
- `CONFIG_SPI_NRFX`
- `CONFIG_USB_DEVICE_MANUFACTURER`
- `Kconfig.defconfig` 里的 `select SOC_DCDC_NRF52X` / `_HV`

全部没有生效。

### 修法（二选一，不要一半留一半）

- **A. 删掉这 5 个文件**（204 行），只保留 `conf` + `overlay` + `keymap` —— 最贴近上游、最干净；**← 已采用**
- **B. 真正接管 board 定义**，把它们挪进 `config/boards/arm/corneish_zen/`（0.3 仍支持，会打弃用警告），此时 `config/*_defconfig` 与 Kconfig 才会被读取。**← 若将来要启用 DCDC，需走这条路**

---

## 🔵 二、墨水屏被关掉，连带 SPI / EPD / LVGL 全部失活

> ### 📌 状态：确认关闭墨水屏（设计决定，非缺陷）
>
> 经确认，`CONFIG_ZMK_DISPLAY=n` 是**有意为之**——关闭墨水屏以换取更长续航。这是本机型的取舍，不是配置失误，**不需要修复**。
>
> 但结论要落到代码上：既然确定不要屏幕，`overlay` 里那套面向面板的配置应一并清掉，不要留半套。待清理清单见本节末尾。

`config/corneish_zen.conf:29` 的 `CONFIG_ZMK_DISPLAY=n` 覆盖了上游 defconfig 的 `=y`，而 overlay 里仍完整配着面板：

- `config/corneish_zen_v2_left.overlay:94-115`：`&spi0` + `epd: il0323@0`（`gooddisplay,il0323`，80×128，4MHz）
- `config/corneish_zen_v2_left.overlay:63-92`：`spi0` / `spi3` 的 pinctrl
- chosen 里 `zephyr,display = &epd`、`zmk,display = &epd`

### 实际结果

- `CONFIG_IL0323`、`CONFIG_SPI`、`CONFIG_SPI_NRFX` 均不在最终 `.config`；
- 日志对应的告警：`SPI_NRFX ... got the value 'n'. Check these unsatisfied dependencies: SPI (=n)`；
- 上游 `app/boards/arm/corneish_zen/CMakeLists.txt` 把 `custom_status_screen.c` 与整个 `widgets/` 包在 `if(CONFIG_ZMK_DISPLAY)` 内，只有 `empty_file.c` 在条件外 → **屏幕 UI 一行都没编译**；
- Kconfig 多出 5 条「写了但被丢弃」的警告：`ZMK_DISPLAY_DEDICATED_THREAD_STACK_SIZE='4096'`、`LV_Z_VDB_SIZE='100'`、`LV_Z_BITS_PER_PIXEL='1'`，以及两个 choice symbol 选择失败。

### 待决策

~~Corne-ish Zen v2 的招牌就是那块 1.54" 墨水屏（电量 / 输出 / 层号）。**是有意为省电关掉，还是抄自某个不带的变体？**~~

**已决策：有意关闭**（省电优先）。因此不需要改回 `CONFIG_ZMK_DISPLAY=y`，而是走「有意的」那一条：清理死配置。

### 待清理清单（因确认关闭屏幕而变成纯 dead weight）

| 位置 | 内容 | 处理建议 |
| --- | --- | --- |
| `left.overlay:63-92` / `right.overlay` 对应段 | `spi0_default`/`spi0_sleep`、`spi3_default`/`spi3_sleep` 四组 pinctrl | 若 underglow 也不要（见第三条），可全删；若保留 underglow 则只留 spi3 |
| `left.overlay:94-115` / `right.overlay` 对应段 | `&spi0` 节点 + `epd: il0323@0`（`gooddisplay,il0323`，dc/busy/reset gpios，pwr/cdi/tcon） | 删 |
| `left.overlay:14` / `right.overlay` | chosen `zephyr,display = &epd` | 删 |
| `dtsi:22`（`config/corneish_zen.dtsi`） | chosen `zmk,display = &epd` | 删（`zmk,display` 在 v0.3 源码中本就无人读取，是无效遗留） |
| `keymap` / `conf` | 无屏幕相关配置需要连带处理 | — |

**保留建议**：`config/corneish_zen.conf:29` 的 `CONFIG_ZMK_DISPLAY=n` **要保留并加注释**——它现在是有意义的设计声明，且必须压过上游 defconfig 的 `=y`（上游 `corneish_zen_v2_left_defconfig:10` 默认开着）。建议写成：

```conf
# 墨水屏有意关闭以延长续航；上游 board defconfig 默认 ZMK_DISPLAY=y，此处显式覆盖
CONFIG_ZMK_DISPLAY=n
```

---

## 🔴 三、underglow 配置全死

| 位置 | 内容 |
| --- | --- |
| `config/corneish_zen.conf:5` | `CONFIG_WS2812_STRIP=y` |
| `config/corneish_zen_v2_left.overlay:116-137` | `led_strip: ws2812@0`（`worldsemi,ws2812-spi`，chain-length 12） |
| overlay chosen | `zmk,underglow = &led_strip` |

但 `CONFIG_ZMK_RGB_UNDERGLOW`（`app/Kconfig:278` 的 menuconfig，`:280` 才 `select LED_STRIP`）从未打开：

```
warning: WS2812_STRIP ... was assigned the value 'y' but got the value 'n'.
         Check these unsatisfied dependencies: ... LED_STRIP (=n)
```

最终 `.config` 无 `CONFIG_LED_STRIP`、无 `CONFIG_WS2812_STRIP`，`app/src/rgb_underglow.c` 整个文件不编译（`#if !DT_HAS_CHOSEN(zmk_underglow)` 分支都没机会走到）。

且**上游 corneish_zen board 里根本没有 underglow**（board 目录 grep `UNDERGLOW` 零命中），这部分是自行添置、从未点亮过的配置。

### 顺带

两侧 overlay 注释自相矛盾：left 写 `/* 6 keys have underglow */`，right 写 `/* 40 keys have underglow */`，`chain-length` 却都是 12。

---

## 🔴 四、DCDC 无处安放

`config/Kconfig.defconfig:47-57` 的写法**语义完全正确**，与官方范式（`app/boards/arm/nice_nano/Kconfig`）一致：

```
config BOARD_ENABLE_DCDC
    bool "Enable DCDC mode"
    select SOC_DCDC_NRF52X
    default y
    depends on (BOARD_NICE_NANO || BOARD_NICE_NANO_V2)

config BOARD_ENABLE_DCDC_HV
    bool "High voltage DCDC converter"
    select SOC_DCDC_NRF52X_HV
    default n
    depends on (BOARD_NICE_NANO_V2)
```

但文件不被读取 → 构建日志里 `DCDC` 出现 **0 次**（`BOARD_ENABLE_DCDC`、`SOC_DCDC_NRF52X`、`SOC_DCDC_NRF52X_HV` 全零）。

### 提交史

```
6037d1d  Upgrade DCDC mode for zmk 0.4 nRF52840 (#28)   ← 从 Kconfig.defconfig 挪到 dtsi
492c431  Revert back to zmk v0.3 (#29)                  ← 又挪回 Kconfig.defconfig
```

**两次都没生效**，因为位置始终不在 BOARD_ROOT 下。

### 两点补充

1. `BOARD_ENABLE_DCDC` 是 **ZMK 自造**的 board-level symbol，不是 Zephyr 的（Zephyr 3.5 `soc/arm/nordic_nrf/nrf52/Kconfig.soc:84-94` 只定义无 prompt 的 `SOC_DCDC_NRF52X` / `SOC_DCDC_NRF52X_HV`）。因此**无法**在 `corneish_zen.conf` 里写 `CONFIG_SOC_DCDC_NRF52X=y` 绕过——无 prompt 的符号，`.conf` 赋值会被丢弃。
2. `_HV` 那版仓库用 `default y`，nice_nano 用 `default n`。nRF52840 的 HV DCDC 一般只在有外部高压需求时开，默认值值得商榷。

**修法**：与第一条一起决策——走 `config/boards/` 布局真接管，或承认 0.3 下无法注入并删除这段死代码。

---

## 🟡 五、`config/corneish_zen.dtsi` 是上游副本，实际只贡献 1 行

这个 120 行文件是上游 `app/boards/arm/corneish_zen/corneish_zen.dtsi` 的手工改写版（证据：`/dts-v1/;` 被注释成 `/* /dts-v1/; */`，只有被 `#include` 时才需要这么做）。

它与上游那份**同时被 include**：overlay 的 `#include "corneish_zen.dtsi"` 按相对路径解析到 `config/` 这份，board dts 的 `#include "corneish_zen.dtsi"` 解析到 board 目录那份。

### 逐项核对

| 内容 | 实际效果 |
| --- | --- |
| `default_transform`（:27-42） | 与上游逐字相同 → 重复定义被 DT 合并，无影响 |
| `five_column_transform`（:44-59） | **定义了但无人使用** |
| flash 分区（:86-119） | 全被 `nrf52840-nosd` snippet `delete-node` 后重建 → 无效 |
| `&usbd { cdc_acm_uart {...} }`（:79-84） | 旧写法，但**能工作**（最终 DTS 里 `cdc_acm_uart` 正确挂在 `usbd@40027000` 下） |
| 删掉上游 `layouts/foostan/corne/*column.dtsi` include | **没有效果**，上游 board dtsi 仍 include 了它们，`foostan_corne_6col_layout` 节点照样存在 |
| chosen `zmk,matrix-transform`（:24） | 唯一真正起作用的贡献 |

### chosen 双写：机制修正

`dtsi:24` 写连字符 `zmk,matrix-transform`，`keymap:14` 写下划线 `zmk,matrix_transform`——**看着像 bug，其实两者等价**：

- Zephyr `scripts/dts/gen_defines.py:994-997` 的 `str2ident()` 把 `[-,.@/+]` 全部替换成 `_`；
- 两个属性名归一化后都是 `DT_CHOSEN_zmk_matrix_transform`（`gen_defines.py:907-908`）；
- 上游 78 个 shield 全用连字符，官方文档 `docs/config/layout.md:41` 也是连字符。

所以连字符写法完全有效，`keymap` 里那条下划线只是**风格偏差**，不是错误。

### 真正的问题是路线过时

chosen 里同时有 `zmk,physical-layout`（上游给的）和 `zmk,matrix-transform`，`app/src/physical_layouts.c:31` 的 `!DT_HAS_CHOSEN(zmk_matrix_transform)` 判定为 false，于是走旧分支并打出：

```
physical_layouts.c:123:2: warning: #warning "Ignoring the physical layouts and using the chosen matrix transform.
Consider setting a chosen physical layout instead."
```

### 干净的现代化改法

删掉 `config/corneish_zen.dtsi` 整个文件、去掉 overlay 里的 `#include`、删掉 `keymap:13-17` 的 chosen。

上游 layout 的 `transform = <&default_transform>` 与本仓库 map 完全对应（42 键、4×12+6），**行为不变**，警告消失，还顺带打开 physical layout 这条路（Studio / 多布局的前置条件）。

---

## 🟡 六、两侧 overlay ~130 行近乎逐字重复

`left`（138 行）/ `right`（152 行）的 diff 只有四处实质差异：

| 差异 | left | right |
| --- | --- | --- |
| kscan `col-gpios` 顺序 | gpio0 31/29/2, gpio1 15/13/11 | 反转：gpio1 11/13/15, gpio0 2/29/31 |
| `col-offset` | 无 | `&default_transform`、`&five_column_transform` 加 `<6>` |
| `&spi3` 块位置 | 文件末尾 | 提前到中间 |
| 注释 | "6 keys have underglow" | "40 keys have underglow" |

`ext-power`、`vbatt`、pinctrl（spi0/spi3）、spi0+epd **全部逐字复制**。

### 建议

抽一个共享 `corneish_zen_common.dtsi`（`#include` 进两个 overlay），左右各自只保留 kscan 引脚 + `col-offset`。改一处面板参数不用改两处——当前状态下改错边是很容易发生的事。

---

## 🟡 七、vbatt 节点属性混搭

`overlay:55-57` 只写了 `compatible = "zmk,battery-nrf-vddh";`（**这是对的**，该驱动读固定 `VDDHDIV5` 通道，绑定 `app/module/dts/bindings/sensor/zmk,battery-nrf-vddh.yaml` 也确实没有任何属性），但生成 DTS 里带着上游 voltage-divider 的残留：

```dts
compatible   = "zmk,battery-nrf-vddh";
io-channels  = <&adc 0x0>;
output-ohms  = <0x1de840>;   /* 1960000 */
full-ohms    = <0x2a4450>;   /* 2770000 */
```

对 vddh 驱动完全无效，但极易让后来读代码的人误以为这里走外部分压。

**建议**：在 overlay 里显式 `/delete-property/ io-channels;`、`/delete-property/ output-ohms;`、`/delete-property/ full-ohms;`，或干脆不覆盖 compatible。

---

## 🟡 八、`label` 已弃用：5 条 DTS 告警

```
'label' is marked as deprecated ... for node /behaviors/homerow_mods_left
                                              /behaviors/homerow_mods_right
                                              /behaviors/mo_left
                                              /behaviors/mo_right
                                              /macros/tap2_macro
                                              /ext-power
```

前五个是 keymap 里 behavior 的 `label = "..."` 属性（`zmk,behavior-hold-tap.yaml`、`zmk,behavior-macro-two-param.yaml` 已标记 deprecated）。**直接删掉即可**，零功能影响。

`ext-power` 那个要小心：`app/src/ext_power_generic.c:36-45` 用 `ext_power->name` 做 settings key：

```c
snprintf(setting_path, sizeof(setting_path), "ext_power/state/%s", ext_power->name);
```

而 `device->name` = `DEVICE_DT_NAME(node_id)` = `DT_PROP_OR(node_id, label, DT_NODE_FULL_NAME(node_id))`（`include/zephyr/device.h:145-146`）。

也就是说**现在这个 `label = "EXT_POWER"` 恰好保住了已保存的开关状态**；直接删 label 会让 key 变成节点名 `ext-power`，用户已存的 ext_power 状态丢失。

**正确做法**：把节点直接命名为 `EXT_POWER`（上游 `nice_nano_v2.dts` 的注释专门写了 *"Node name must match original `EXT_POWER` label to preserve user settings."*），然后再删 label。

---

## 🟢 九、零碎问题

| 位置 | 问题 |
| --- | --- |
| `keymap:56` | `hmr` 的 `require-prior-idle-ms = <59>`，`hml`/`mol`/`mor` 都是 60（`43bec59` bot 提交留下）——语义无害，但没理由不一致 |
| `keymap:86` | `mor` 的 `hold-trigger-key-positions` 末三位是 `38 37 36`（降序），其它三处升序——集合匹配故无影响，纯可读性 |
| `keymap:106` | `combos { compatible = "zmk,combos"; };` 空壳，可删 |
| `dtsi:44-59` | `five_column_transform` 未被使用 |
| `dtsi:86-119` | flash 分区被 snippet 全删重建 |
| `overlay:20-22` | `wakeup-source;`、`init-delay-ms` 缩进错位（两侧都有） |
| `overlay:41-46`、`right:66-75` | 注释掉的 leds / col-gpios 块，属残留 |
| `workflow:30-58` | 整个 `create_png` job 被注释掉，却留在文件里 |
| `workflow:20-21` | `@main` + `install_branch: "main"` 双 main，绘图工具版本不固定（构建部分 `@v0.3` 固定正确，对比鲜明） |
| `README.md` | 仅 13 行，只有"请用 Online Editor" + 图 + 致谢；没写硬件是哪块板、v0.3 pin、屏幕为何关闭 |

### 附：硬件不是官方 board

本仓库 overlay 的引脚：

- row：gpio0 22 / 24、gpio1 0、gpio0 11
- col：gpio0 31 / 29 / 2、gpio1 15 / 13 / 11

**与官方 v1/v2 的 left/right 全都不匹配**，且官方 v0.2 → v0.3 的 `corneish_zen_v2_left.dts` 引脚逐字相同（v0.2: row `gpio0 19/4/31/30`、col `gpio0 21/23/12/gpio1 9/gpio0 7/5`）。

所以这不是「升级导致的错位」，而是**另一块 PCB**。这解释了为什么必须本地覆盖 kscan——也意味着以后若想「回到官方 board 配置」，光删 overlay 是不行的。

---

## ✅ 做对的地方（别改坏）

- **pin 一致性做得好**：`config/west.yml` 的 `revision: v0.3` 与 workflow 的 `build-user-config.yml@v0.3` 对齐，没有出现「代码 pin 0.3、CI 用 main」的经典错配。
- **`nrf52840-nosd` snippet 用得很对**（`940bdaf`），且确实生效：最终分区为 sd@0 4K / code@1000 844K / storage@d4000 128K / boot@f4000 48K。
- **keymap 里每个符号在 v0.3 都真实存在**（逐个核过）：`CAPSLOCK`（`keys.h:328`）、`DELETE`（`:395`）、`LG()`/`LC()`（`modifiers.h:24`）、`zmk,behavior-macro-two-param`、`hold-trigger-key-positions`/`-on-release`、`ZMK_KSCAN_DEBOUNCE_*`。
- **keymap 结构严谨**：11 层 × 42 键无缺无多；所有 `&mol/&mor/&mo/&to` 索引都在 0..10，无越界。
- **`as`/`hml`/`hmr`/`mol`/`mor` 都不是幻觉符号**：v0.3 里**没有** `behavior_auto_shift.c`，这些全是用 `zmk,behavior-hold-tap` 自己搭的，参数合法。
- **`image/` 与 keymap 同步**：在 `6aef6a2` 同一次提交里一起更新，绘图链路（含 `[Draw]` bot 回写）是活的。

---

## 建议的落地顺序

1. ~~**先定两件事**：屏幕要不要（第二条）、underglow 要不要（第三条）。这决定了是「删 ~60 行死配置」还是「打开 `CONFIG_ZMK_DISPLAY=y` + `CONFIG_ZMK_RGB_UNDERGLOW=y`」。~~ **屏幕已确认关闭**；underglow 仍待决定。
2. ~~**删死文件**：`config/corneish_zen_v2_{left,right}_defconfig`、`config/Kconfig`、`config/Kconfig.board`、`config/Kconfig.defconfig`（共 5 个、204 行，零功能损失）。~~ **已完成**（PR #35）；DCDC 需单独决策。
3. **清理屏幕死配置**：按第二条的「待清理清单」删掉 overlay 里的 `spi0`/`epd`/`zephyr,display`、dtsi 里的 `zmk,display`，并在 `corneish_zen.conf:29` 给 `CONFIG_ZMK_DISPLAY=n` 加注释说明这是有意关闭。
4. **删 `config/corneish_zen.dtsi` + overlay 的 `#include` + keymap:13-17 的 chosen**，切到 physical layout（消掉 `#warning`，为升级铺路）。
5. **抽公共 overlay dtsi**，消掉 ~130 行重复。
6. 清 `label`、`combos` 空壳、`five_column_transform`、vbatt 残留属性、缩进、注释掉的代码块；README 补 3 行「这块板是什么 / pin 在 v0.3 / 屏幕有意关闭」。

全部做完，`config/` 大约能从 854 行降到 500 行以内，且**没有一行是「写了但不生效」的**——这本身就是本次评审里最值得改的东西。

---

## 附：仓库文件实况（`wc -l`）

| 文件 | 行数 | 是否被构建读取 |
| --- | --- | --- |
| `config/corneish_zen.keymap` | 210 | ✅ |
| `config/corneish_zen.conf` | 30 | ✅ |
| `config/corneish_zen_v2_left.overlay` | 138 | ✅ |
| `config/corneish_zen_v2_right.overlay` | 152 | ✅ |
| `config/corneish_zen.dtsi` | 120 | ⚠️ 仅经 overlay `#include` 生效 |
| `config/corneish_zen_v2_left_defconfig` | 59 | ❌ 已删除 |
| `config/corneish_zen_v2_right_defconfig` | 59 | ❌ 已删除 |
| `config/Kconfig` | 10 | ❌ 已删除 |
| `config/Kconfig.board` | 19 | ❌ 已删除 |
| `config/Kconfig.defconfig` | 57 | ❌ 已删除 |
| **合计** | **854 → 650** | 实际生效 530 行（+ `corneish_zen.dtsi` 的 1 行） |
