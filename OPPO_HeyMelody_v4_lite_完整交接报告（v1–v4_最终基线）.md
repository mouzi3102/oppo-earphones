# OPPO 欢律 HeyMelody · v4 lite 完整交接报告（v1–v4 全部内容）

> **版本/日期：2026-10-15。本报告以 `OPPO_Enco_Free4_fixed_v4_lite.apk` 为最终基线，覆盖 v1（暗黑玻璃）+ v2（弹窗不透明白底）+ v3（选中蓝）+ v4（审计与精简）全部内容。**
> 目标：另一个 AI / 开发者不看本次对话上下文，仅凭本文 + v4 APK 即可复现、验证、并在官方新版上移植。
> 所有路径/类名/色值/字节数均取自反编译工程实测，未编造。沙箱无 emulator/adb，**全部动态验证标注"未做"**。

---

## 1. 文档说明
- 基线：v4_lite（含全部 UI 修复 + 安全精简）。
- 范围：逆向修复过程、关键代码地图、三次 UI 修复、v4 审计分级、精简清单、开源库依据、移植指南、产物清单、限制声明。
- 性质：个人学习/研究用途。

---

## 2. 目标 App 基础信息

| 项 | 值 |
|---|---|
| 应用名 | 欢律 / HeyMelody（OPPO、一加耳机官方配套） |
| 正确包名 | `com.heytap.headset`（用户曾笔误 `com.heytap.hostset`，以此为准） |
| versionName | `17.6.5` |
| versionCode | `117006005` |
| minSdkVersion / targetSdkVersion | `24` / `36` |
| ABI | **仅 arm64-v8a 单 ABI**（约 7MB，无 fat ABI 可瘦） |
| 语言 | **近纯中文**（zh-rCN / zh-rTW / zh-rHK 三变体） |
| 原始 APK | 解包工程内 `work/original_backup.apk`（约 23MB，全程勿动） |

**功能**：OPPO/一加耳机配套（示例机型 Enco Free4 丹拿版）——固件更新与导入、降噪/自适应/通透/关闭四模式、设备双连、Hi-Res/大师调音、用户须知、多设备切换、云登录。

---

## 3. 工具链与可照抄命令

设 `W=.../agents/s_000EVup1adG/work`。

| 工具 | 版本 | 安装 |
|---|---|---|
| JDK | OpenJDK 11 | 系统自带 |
| apktool | 2.9.3 | `work/tools/apktool.jar`（GitHub release 直链） |
| jadx | 1.5.0 | `work/tools/jadx_dir/bin/jadx`（解压 zip） |
| uber-apk-signer | 1.3.0 | `work/tools/uber-apk-signer.jar`（内置 zipalign + debug 密钥） |

```bash
cd "$W"
# 反编译
java -jar tools/apktool.jar d -f -o decoded original_backup.apk
nohup tools/jadx_dir/bin/jadx -d jadx_out original_backup.apk > jadx.log 2>&1 &
# 改资源后回编译
java -jar tools/apktool.jar b decoded -o built_unsigned.apk --use-aapt2
# 对齐 + debug 签名（一步）
rm -rf signed && mkdir signed
java -jar tools/uber-apk-signer.jar -a built_unsigned.apk -o signed
# 产物：signed/built_unsigned-aligned-debugSigned.apk
# 静态回读
java -jar tools/apktool.jar d -f --no-src -o verify built_unsigned-aligned-debugSigned.apk
```

- **签名**：debug，`CN=Android Debug`，v2+v3。与官方签名不同 → **必须先卸载旧版再装**（会清配对数据；无官方私钥无法保留数据）。

---

## 4. 关键代码地图

- **主页布局**：`res/layout/heymelody_app_activity_detail_main.xml`（根 RelativeLayout，背景 `@drawable/glass_bg_gradient`，含底部导航 `@drawable/glass_bottom_nav_bg`）；`res/layout/melody_ui_fragment_detail_main.xml`（NestedScrollView 内容容器，同样 `glass_bg_gradient`，文字 `?couiColorLabelPrimary`）。
- **模式圆控件**：`com.oplus.melody.ui.widget.devicecontrol.c`（UDeviceNormalModeButton）选中时 `setBackgroundResource(R.drawable.melody_ui_iot_mode_button_bg_selected)`，未选中 `melody_ui_mode_button_bg`。
- **数据模型**：`com.oplus.iotui.model.ModeItem`（含 color 字段）；`com.oplus.melody.ui.component.detail.noisereduction.NoiseReductionItem` **未调 setColor**，`getColor()=null` → 选中圆无额外 tint，颜色完全由 drawable 决定。
- **暗黑切换**：标准 DayNight；`?couiColorLabelPrimary → coui_color_label_primary` 浅色 `#e6000000` / 夜间 `#e6ffffff`。
- **COUI 弹窗着色链路**：`DefaultBottomSheetDialog` → `panelBackgroundTintColor=?couiColorSurface`；面板背景 `coui_color_surface`：**浅色 `#ffffffff`（v2 改后）/ 夜间 `#ff1e1e1e`（未动）**。4 类引用位置：① `design_bottom_sheet_dialog.xml`（BottomSheet 面板）② `coui_alert_dialog_background.xml` ③ `coui_alert_dialog_tiny_background.xml` ④ `styles.xml:24` `layoutBackgroundTint`（浮动 AlertDialog）。**smali 按资源 ID 0x7f060795 反查 0 处代码引用**。
- **固件/弹窗**：导入固件三选项字符串硬编码在 dex（无独立布局）；用户须知内容 `melody_ui_full_page_statement.xml`；本地导入真实实现 `com.hl.firmwareimport.LocalFirmwareImporter`（Intent 取 Uri→FileInputStream/OutputStream）。
- **【重要】自升级广播兼容垫片** `smali/com/heytap/upgrade/install/e.smali`：
  - 三处 raw `registerReceiver`（:331/:487/:1550）均被 `if-lt SDK_INT,0x21(=33),:cond` 门控在 **SDK<33 分支**；
  - SDK≥33 走垫片 `LE2/a;->v(...)` 与 `LJa/g;->q(...)`，二者调用带 flag 重载并传 **flags=0x2（RECEIVER_EXPORTED）**；
  - **结论：已正确满足 Android13+ 要求，切勿改成 NOT_EXPORTED(0x4)**（会收不到安装器跨 uid 广播、破坏静默安装）。

---

## 5. 全部 UI 修复清单（v1–v3）

| 版本 | 文件 | 改前 | 改后 | 原因 |
|---|---|---|---|---|
| v1 | `res/drawable-night/glass_bg_gradient.xml`（新增） | 无 night 变体，夜间沿用浅色渐变 | `#ff1a1d24→#ff14171d→#ff0f1115` | 夜间白字压浅玻璃底 |
| v1 | `res/drawable-night/glass_bottom_nav_bg.xml`（新增） | 白半透明胶囊 | 深色磨砂胶囊（半透明 `#66000000`+深色渐变+`#22ffffff` 描边） | 底部导航夜间发白 |
| v1 | `res/drawable-night/glass_mode_circle_normal.xml`（新增） | `#1affffff` | `solid #24ffffff` + `#33ffffff` 描边 | 未选中圆夜间太淡 |
| v2 | `res/values/colors.xml:977` `coui_color_surface` | 浅色 `#33ffffff`（20% 透明白） | 浅色 `#ffffffff`；夜间 `#ff1e1e1e` **不动** | 弹窗透底重叠 |
| v3 | `res/drawable/melody_ui_iot_mode_button_bg_selected.xml` | `#66ffffff→#330088ff→#4d0088ff` | `#1a9bff→#0d84ff→#0077e6`（描边 `#33ffffff`） | 选中蓝发白 |

**v1 关系说明**：开发者自备 `glass_bg_gradient_dark.xml` 的深色渐变色值被直接迁入新增的 `drawable-night/glass_bg_gradient.xml`。

**v2 口径修正**：不是"只修两个底部弹窗"，而是**统一修复所有 COUI 弹窗/对话框面板（BottomSheet + AlertDialog + Tiny Dialog + 浮动 AlertDialog）的浅色透底**；白底深色字对比正常，夜间零改动。

**v3 关键纠错**：v2 曾误改 `glass_mode_circle_selected.xml`，经查该资源**零引用**；真正生效的是 `melody_ui_iot_mode_button_bg_selected.xml`。

**废弃资源最终状态（v4 已删）**：`glass_mode_circle_selected.xml`、`glass_dialog_bg.xml`、`glass_card_bg.xml`、`glass_top_bar_bg.xml`、`glass_bg_gradient_dark.xml` 五个已确认 smali=0/layout=0，**v4 已物理删除并清理 public.xml 登记**。在用的 `glass_button_primary.xml`、`glass_splash_bg.xml` 保留。

---

## 6. v4 审计与精简

### 6.1 问题分级（9 条）

| 级别 | 编号 | 位置 | 描述 | 本版处理 |
|---|---|---|---|---|
| 严重 | S1 | `upgrade/install/e.smali:331/487/1550` | 自升级 registerReceiver 未传 flag | **【误报】不改**（见下） |
| 中等 | M1 | `UrlUtil.smali:36` + `OkHttpFinal.smali:1203` | 调试明文域名 `http://af-test.wanyol.com` | **已中性化** |
| 中等 | M2 | `assets/hl_firmware_db.json` | 固件指向第三方镜像 `16239.kstore.space` | 保留（无官方源） |
| 中等 | M3 | Manifest:54 | `READ_PHONE_STATE` 过度申请嫌疑 | 保留（拿不准） |
| 中等 | M4 | Manifest:10/11 | FOREGROUND_SERVICE 重复声明两行 | **已删一行** |
| 轻微 | — | 5 个废弃 glass drawable | 死资源 | **v4 已删** |
| 轻微 | — | DebugProbesKt.bin / kotlin-tooling-metadata | 调试残留 | **v4 已删** |
| 建议 | — | zh-rTW/zh-rHK 语言 | 繁中 ~550KB | 默认不删（回退风险） |

**S1 误报代码依据**：三处 raw 调用被 `if-lt SDK_INT,33,:cond` 门控在 SDK<33；SDK≥33 走垫片 `E2.a.v`/`Ja.g.q`，传 flags=0x2（RECEIVER_EXPORTED）。Android13+ 只要求"必须显式传 EXPORTED 或 NOT_EXPORTED"——传 0x2 已满足，不会 SecurityException；强改 NOT_EXPORTED 反而收不到安装器广播、破坏静默安装。

### 6.2 功能走查结论
- 正常（推测，待真机）：设备连接、四模式切换、内置源 OTA、用户须知、双连、Hi-Res/调音预设、云登录。
- **本地固件导入** `LocalFirmwareImporter`：真实文件流实现，正常。
- **固件"从 URL 下载"**：**存疑（待真机）**——仅见 EditText 暗示 URL 输入框，未见独立下载类。

### 6.3 v4 实际精简改动
- 删除 5 个零引用 glass drawable（`glass_mode_circle_selected`/`glass_dialog_bg`/`glass_card_bg`/`glass_top_bar_bg`/`glass_bg_gradient_dark`），同步删 public.xml `<public>` 登记。
- 删除 `unknown/DebugProbesKt.bin`(1728B)、`unknown/kotlin-tooling-metadata.json`(625B)，同步删 apktool.yml unknownFiles 登记。
- Manifest 去重 FOREGROUND_SERVICE 两行→一行。
- M1 两处明文调试域名中性化（仅改字符串字面量，不动控制流/寄存器）：
  - `UrlUtil.smali:36`：`http://af-test.wanyol.com` → `https://af-service-cn.allawntech.com`
  - `OkHttpFinal.smali:1203`：同上。

### 6.4 保留未删项及原因
- `assets/hl_firmware_db.json`（第三方固件镜像，无官方源）；
- `READ_PHONE_STATE` 权限（移除需评估运行时）；
- 繁中 zh-rTW/zh-rHK（语言回退风险）；
- 所有动态加载 assets（`assets/html`、`images*`、`*.json`，按名拼路径读取，不可删）。

### 6.5 体积
- v3：**23,037,269 B** → v4：**23,032,629 B**，省 **4,640 B（0.02%）**。
- 瘦身空间天然不足：单 ABI、近纯中文、高置信废弃资源仅数 KB。

---

## 7. 三个开源液态玻璃项目

| 仓库 | 技术栈 | 原理 | 暗黑适配 |
|---|---|---|---|
| QmDeve/AndroidLiquidGlassView | Java/View | RenderEffect + AGSL shader，mix(color,tint) | 库本体 0 处夜间 |
| Kyant0/AndroidLiquidGlass | Compose | BlurEffect + AGSL lens | 库本体 0 处，示例示范深色 |
| QWEA0/Liquid-Glass-Android | View | **采样玻璃背后像素亮度+滞回，自动翻转 tint/文字/高光/阴影** | **最完善** |

- 三方结论：库本体均不做夜间适配，需 App 侧补 night 资源。
- QWEA0 深色取值：玻璃 tint `#24FFFFFF`(白@14%)、blur 砍半、文字白+黑投影、降级底色 `#2A2A2E`。
- 本次实际参考：未选中圆 `#24ffffff`、深色页近黑 `#1a1d24` 系、选中品牌蓝对齐 App `#0088ff`。

---

## 8. 后续版本更新移植指南

工作流：新版官方 APK → `apktool d` → 按下表锚点逐项核查 → 移植修复+精简 → 回编译签名 → 验证。

| 项 | 锚点（搜索关键词） |
|---|---|
| v1 暗黑页底 | `glass_bg_gradient`、`drawable-night/glass_bg_gradient.xml` |
| v1 导航/模式圆 | `glass_bottom_nav_bg`、`glass_mode_circle_normal` 的 night 变体 |
| v2 弹窗 | `coui_color_surface`（values 浅色值）、`DefaultBottomSheetDialog` |
| v3 选中蓝 | `melody_ui_iot_mode_button_bg_selected.xml`、jadx `devicecontrol/c` |
| v4 M1 | `wanyol`（应为 0 命中） |
| v4 M4 | Manifest `FOREGROUND_SERVICE`（应 1 行） |
| v4 废弃资源 | 那 5 个 glass drawable 删除前需**重新核查引用**（官方可能重新启用） |

**验证清单**：
- 静态：回读 XML/色值、uber-apk-signer `zipalign verified`+`signature verified`、删除项缺失/保留项完整。
- 动态（真机）：切暗黑看页底、开导入固件+用户须知两弹窗、点选四模式圆、固件三路径、自升级流程。
- 提醒：废弃资源删除前必须重新 grep 引用；夜间资源可能随官方新增而变化。

---

## 9. 已交付产物清单（以 chats 目录实际为准）

| 文件 | 大小 | 说明 |
|---|---|---|
| `OPPO_Enco_Free4_darkmode_fixed.apk` | 23,037,269 B | v1 暗黑玻璃 |
| `OPPO_Enco_Free4_fixed_v2.apk` | 23,037,269 B | v2 弹窗+选中蓝初改 |
| `OPPO_Enco_Free4_fixed_v3.apk` | 23,037,269 B | v3 真正生效的选中蓝 |
| `OPPO_Enco_Free4_fixed_v4_lite.apk` | 23,032,629 B | **最终基线（v1+v2+v3+精简）** |
| `修复说明.md` | 15,682 B | v1/v2/v3 追加节 |
| `OPPO_HeyMelody_APK_逆向修复交接文档.md` | 18,677 B | v3 版交接 |
| `OPPO_HeyMelody_v3_全面审计报告.md` | 11,885 B | A+B 合并审计 |
| `v4_精简修复说明.md` | 3,950 B | v4 精简说明 |
| `OPPO_HeyMelody_v4_完整交接报告.md` | — | 本文 |

> **FlowEq 调音文件**：本工作区（chats/artifacts）经 `find` 全盘查找**未包含任何 FlowEq/调音文件（N/A）**，不存在则如实标注，不编造。

---

## 10. 已知限制与声明
- 沙箱无 emulator/adb，**仅静态验证**：回编译通过 + 签名/zipalign 校验 + 资源回读。**未做任何动态验证**。
- 真机必验：切暗黑、两个弹窗、四模式圆、固件三路径（尤其"从 URL 下载"存疑）、自升级流程。
- 个人研究/学习用途。
- 随官方更新可能失效：资源名（`glass_bg_gradient`、`melody_ui_iot_mode_button_bg_selected`、`coui_color_surface`）、控件类（`devicecontrol/c`）、弹窗基类若官方改版，锚点需重新定位。
