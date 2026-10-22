# OPPO 欢律（HeyMelody）APK 逆向修复 · 完整交接文档

> 本文档目标：**让另一个 AI / 开发者不看本次对话上下文，仅凭本文 + 最终版 APK（v3），即可完整复现、验证、并在官方后续版本上移植这三处 UI 修复。**
> 所有路径 / 类名 / 资源名 / 色值均来自实际反编译工程（apktool 2.9.3 解码产物），已逐条核实。

---

## 1. 目标 App 基础信息

| 项 | 值 |
|---|---|
| 应用名 | 欢律 / HeyMelody（OPPO、一加耳机官方配套 App） |
| 正确包名 | `com.heytap.headset`（注意：曾有人笔误写成 `com.heytap.hostset`，以此为准） |
| versionName | `17.6.5` |
| versionCode | `117006005` |
| minSdkVersion | `24`（Android 7.0，见 apktool.yml sdkInfo） |
| targetSdkVersion | `36` |
| 原始 APK 文件名 | `62061d15ae354e9fa13bbd7e858f56bb.apk~tplv-a9rns2rl98-image_src.apk`（约 23 MB） |
| 原始 APK 备份 | 工程内 `work/original_backup.apk`（**全程勿动原文件，所有改动都在解包副本上做**） |
| 反编译工具版本 | apktool 2.9.3 |

**功能概述**：OPPO/一加蓝牙耳机配套管理。本次修复针对其设备主页（示例耳机：OPPO Enco Free4 丹拿版），涉及：固件更新与"导入固件"底部菜单、降噪/自适应/通透/关闭四种模式圆切换、设备双连、音质音效、启动"用户须知"弹窗等。

---

## 2. 工具链与环境

| 项 | 值 / 说明 |
|---|---|
| 沙箱 OS | Linux x86_64（Ubuntu 22.04 内核系，系统目录只读，**禁止 sudo/apt**） |
| JDK | OpenJDK 11（`/usr/bin/java`，`openjdk 11.0.32`） |
| apktool | 2.9.3，`work/tools/apktool.jar`，直链 `https://github.com/iBotPeaches/Apktool/releases/download/v2.9.3/apktool_2.9.3.jar` |
| jadx | 1.5.0，解压在 `work/tools/jadx_dir/bin/jadx`，直链 `https://github.com/skylot/jadx/releases/download/v1.5.0/jadx-1.5.0.zip` |
| uber-apk-signer | 1.3.0，`work/tools/uber-apk-signer.jar`（自带 zipalign + debug 密钥，一步对齐+签名），直链 `https://github.com/patrickfav/uber-apk-signer/releases/download/v1.3.0/uber-apk-signer-1.3.0.jar` |
| zipalign | 由 uber-apk-signer 内置提供（`--zipAlignPath` 缺省即内置版），无需单独装 Android SDK build-tools |
| 签名 | debug 密钥：`CN=Android Debug, OU=Android, O=US, L=US, ST=US, C=US`，v2+v3 签名，SHA25 过期至 2044。**与官方原版签名不同，无法覆盖安装，须先卸载旧版** |

> 安装方式：全部下载到 `~/.local` 或工作区 `work/tools/`，`chmod +x` jadx 脚本即可，无需 root。

---

## 3. 完整复现操作手册（命令可照抄）

设 `W=/home/user/.doubao/agent_mode/workspace/.sessions/38446908371659522/agents/s_000EVup1adG/work`（实际执行时换成你的工程根）。

### 3.1 备份与反编译
```bash
cd "$W"
cp 原始.apk work/original_backup.apk                 # 先备份
java -jar tools/apktool.jar d -f -o decoded original_backup.apk   # 解出资源+smali
# 另起 jadx 导出可读 Java 源码（读逻辑用，不参与回编译）：
nohup tools/jadx_dir/bin/jadx -d jadx_out --no-src original_backup.apk > jadx.log 2>&1 &
```

### 3.2 回编译 → 对齐 → 签名
```bash
cd "$W"
# 回编译（用 aapt2）
java -jar tools/apktool.jar b decoded -o built_v3_unsigned.apk --use-aapt2
# 对齐 + debug 签名（一步；-o 指定输出目录）
rm -rf signed_v3 && mkdir -p signed_v3
java -jar tools/uber-apk-signer.jar -a built_v3_unsigned.apk -o signed_v3
# 产物：signed_v3/built_v3_unsigned-aligned-debugSigned.apk
cp signed_v3/built_v3_unsigned-aligned-debugSigned.apk OPPO_Enco_Free4_fixed_v3.apk
```

### 3.3 静态回读验证
```bash
java -jar tools/apktool.jar d -f --no-src -o verify_v3 OPPO_Enco_Free4_fixed_v3.apk
# 然后 grep / cat 关键资源确认改动打入
```

### 3.4 工程目录结构（关键部分）
```
work/
├── original_backup.apk            # 原始备份（勿动）
├── decoded/                       # apktool 解包（所有改动在此）
│   ├── apktool.yml                # versionCode=117006005, minSdk=24, targetSdk=36
│   ├── AndroidManifest.xml
│   ├── res/
│   │   ├── values/colors.xml      # 浅色色值
│   │   ├── values-night/colors.xml# 夜间色值
│   │   ├── drawable/              # 各 glass_* / mode_button drawable
│   │   ├── drawable-night/        # 本项目新增 3 个 glass 夜间变体在此
│   │   └── layout/                # 页面/弹窗布局
│   ├── smali/  smali_classes2/ smali_classes3/ smali_classes4/
└── jadx_out/sources/              # jadx 可读 Java（含 com/oplus/melody/...）
```

### 3.5 产物命名规范
- v1：`OPPO_Enco_Free4_darkmode_fixed.apk`（暗黑玻璃背景）
- v2：`OPPO_Enco_Free4_fixed_v2.apk`（+ 弹窗不透明白底、+ 选中蓝初改）
- v3：`OPPO_Enco_Free4_fixed_v3.apk`（+ 真正生效的选中蓝 drawable）← **最终交付，已含全部修复**

### 3.6 常见坑（必读）
1. **废弃/未引用资源会误导**：`res/drawable/glass_mode_circle_selected.xml` 在整个工程里**没有任何布局或代码引用它**（只出现在 `public.xml`）。早期 v2 误改了它，实际不生效。真正的选中圆背景是代码动态设置的 `melody_ui_iot_mode_button_bg_selected.xml`。**改 UI 颜色前，先确认该资源真的被引用（grep 布局 + jadx 搜 R.drawable）。**
2. **代码动态 `setBackgroundResource` 会覆盖布局里写死的 background**：布局 `melody_ui_normal_mode_button.xml` 里写死 `@drawable/glass_mode_circle_normal`，但运行时被 `c.java:94/107` 替换成 `melody_ui_iot_mode_button_bg_selected` / `melody_ui_mode_button_bg`。静态看布局会找错地方，必须结合 jadx 源码。
3. **apktool 回编译报错**：本工程用 `--use-aapt2` 一次通过；若报错，先看是否手改 XML 引入非法字符（色值必须 `#AARRGGBB` 8 位或 `#RRGGBB` 6 位）。
4. **新增 drawable-night 文件不需要动 public.xml**：复用已有资源名时，同名多配置变体由 aapt2 自动打入 resources.arsc。
5. **GitHub API 限流**：批量查 release 用直链版本号，别依赖 api.github.com。

---

## 4. 关键代码地图

### 4.1 设备主页布局
- `res/layout/heymelody_app_activity_detail_main.xml`：根 `RelativeLayout`，`android:background="@drawable/glass_bg_gradient"`；内含底部 `MelodyCompatNavigationView`（背景 `@drawable/glass_bottom_nav_bg`）。
- `res/layout/melody_ui_fragment_detail_main.xml`：内容 `NestedScrollView`，同样 `android:background="@drawable/glass_bg_gradient"`；内含设备名 TextView（`android:textColor="?couiColorLabelPrimary"`）、电量 `MelodyStatusInfoViews`、模式圆容器 `melody_ui_detail_container`。

### 4.2 模式圆控件（选中态背景来源）
- 类 `com.oplus.melody.ui.widget.devicecontrol.c`（即 `UDeviceNormalModeButton`）：
  - 方法 `d(Boolean, boolean)`（jadx 源码第 86–114 行）：选中时
    `imageView.setBackgroundResource(R.drawable.melody_ui_iot_mode_button_bg_selected)`（第94行），图标 `setImageTintList(白色)`；未选中时 `setBackgroundResource(R.drawable.melody_ui_mode_button_bg)`（第107行）。
  - 方法 `setSelectedColor(int)`（第187行）：若 `ModeItem.getColor()` 非空，再 `setBackgroundTintList` 叠一层色。
- `com.oplus.iotui.model.ModeItem`：数据模型，含 `color` 字段（`getColor/setColor`）。
- `com.oplus.melody.ui.component.detail.noisereduction.NoiseReductionItem`：降噪四个模式的数据项，**经查未对任何模式调 `setColor`**，故 `getColor()=null`、不叠 tint——选中圆颜色完全由 drawable 本身决定。

### 4.3 暗黑模式切换机制
- 右上角太阳图标触发的是标准 COUI/AppCompat DayNight 夜间模式切换（切 uiMode → recreate → 资源重新限定符加载）。
- 主文字色链路：布局用 `?couiColorLabelPrimary` → `couiColorLabelPrimary=@color/coui_color_label_primary`：
  - `res/values/colors.xml:932` 浅色 `#e6000000`（近黑）
  - `res/values-night/colors.xml:256` 夜间 `#e6ffffff`（近白）
- 推论：夜间限定符对 colors 生效（文字已变白），则对 drawable-night 同样生效——这是 v1 用 drawable-night 修背景的依据。

### 4.4 弹窗（BottomSheet）机制链路
- 样式 `DefaultBottomSheetDialog`（`res/values/styles.xml:4451`）：
  - `panelBackground=@drawable/coui_default_panel_bg_without_shadow`（一个**无填充、仅顶部圆角**的 shape，本身不带颜色）
  - `panelBackgroundTintColor=?couiColorSurface`
- 布局 `res/layout/design_bottom_sheet_dialog.xml:7`：`design_bottom_sheet` 面板 `android:background="@color/coui_color_surface"`。
- 面板真实颜色 = `coui_color_surface`：
  - `res/values/colors.xml:977` 浅色 `#33ffffff`（20% 白，**半透明 → 透底**）
  - `res/values-night/colors.xml:296` 夜间 `#ff1e1e1e`（不透明，正常）

### 4.5 两个问题弹窗的归属
- **"导入固件"底部菜单**：选项（选择固件版本(内置源)/从URL下载/从本地文件导入/取消）**字符串硬编码在 dex 里**（不在 strings.xml），布局是 COUI 通用 BottomSheet，无独立 XML。
- **启动"用户须知"弹窗**：内容布局 `res/layout/melody_ui_full_page_statement.xml`（标题 + 滚动文本 + `同意并使用`/`不同意` 按钮），容器仍是上述 COUI BottomSheet。
- 两者面板背景都走 4.4 的 `coui_color_surface`，故改这一个色即同时修复两个弹窗。

---

## 5. 三次修复完整清单（核心）

### 总览表

| 版本 | 文件 | 改动点 | 解决问题 |
|---|---|---|---|
| v1 | 新增 `res/drawable-night/glass_bg_gradient.xml` | 浅色页底 → 深色渐变 | 暗黑下白字压浅底不可读 |
| v1 | 新增 `res/drawable-night/glass_bottom_nav_bg.xml` | 白半透明胶囊 → 深色磨砂 | 暗黑底部导航发白 |
| v1 | 新增 `res/drawable-night/glass_mode_circle_normal.xml` | `#1affffff` → `#24ffffff` | 暗黑未选中圆太淡 |
| v2 | `res/values/colors.xml:977` | `coui_color_surface` `#33ffffff`→`#ffffffff` | 浅色弹窗透底重叠 |
| v2 | `res/drawable/glass_mode_circle_selected.xml` | 浅蓝 → 饱和蓝 | ⚠️该资源**未被引用**，实际不生效（见 v3） |
| v3 | `res/drawable/melody_ui_iot_mode_button_bg_selected.xml` | 浅蓝 → 饱和品牌蓝 | 真正让选中圆变饱和蓝 |

### 5.1 v1 —— 暗黑模式玻璃背景（新增 3 个 drawable-night）

**根因**：根布局与内容容器写死 `android:background="@drawable/glass_bg_gradient"`（浅色渐变 `#fff5f8fc → #ffeff4f9 → #ffeaf0f7`），但 `drawable-night/` 下没有它的夜间变体；夜间文字已随主题变白（`#e6ffffff`），背景却仍是浅色 → 白字白底。开发者其实已备好深色版 `drawable/glass_bg_gradient_dark.xml`，只是没接到夜间资源。

**改动1：新增 `res/drawable-night/glass_bg_gradient.xml`**
```xml
<shape android:shape="rectangle" xmlns:android="http://schemas.android.com/apk/res/android">
    <gradient android:startColor="#ff1a1d24" android:centerColor="#ff14171d"
              android:endColor="#ff0f1115" android:angle="270.0" android:type="linear"/>
</shape>
```
（直接采用开发者自备 `glass_bg_gradient_dark.xml` 的色值）

**改动2：新增 `res/drawable-night/glass_bottom_nav_bg.xml`**（深色磨砂胶囊）
```xml
<layer-list xmlns:android="http://schemas.android.com/apk/res/android">
  <item android:bottom="5.0dip"><shape android:shape="rectangle">
    <corners android:radius="28.0dip"/><solid android:color="#66000000"/></shape></item>
  <item android:bottom="7.0dip"><shape android:shape="rectangle">
    <corners android:radius="28.0dip"/>
    <gradient android:startColor="#4d33363d" android:centerColor="#402a2d34"
              android:endColor="#331f2228" android:angle="270.0" android:type="linear"/>
    <stroke android:width="1.0dip" android:color="#22ffffff"/></shape></item>
</layer-list>
```

**改动3：新增 `res/drawable-night/glass_mode_circle_normal.xml`**（暗黑下未选中圆略提亮）
```xml
<shape android:shape="oval" xmlns:android="http://schemas.android.com/apk/res/android">
    <solid android:color="#24ffffff"/>
    <stroke android:width="1.0dip" android:color="#33ffffff"/>
</shape>
```
> `#24ffffff`（白@14%）与开源库 QWEA0 深色玻璃 tint 推荐值一致。
> 影响范围：仅暗黑模式；浅色 `drawable/` 原文件一字未动。

### 5.2 v2 —— 浅色弹窗背景不透明白底（仅改浅色 1 行）

**根因**：COUI BottomSheet 面板着色取 `coui_color_surface`，浅色值 `#33ffffff`（20% 白半透明）→ "导入固件"/"用户须知"弹窗底层内容透上来与文字重叠；夜间值 `#ff1e1e1e` 本就不透明、表现正常。

**改动：`res/values/colors.xml:977`**
```xml
<!-- 改前 --><color name="coui_color_surface">#33ffffff</color>
<!-- 改后 --><color name="coui_color_surface">#ffffffff</color>
```
- **只动 `values/`（浅色）**；`res/values-night/colors.xml:296` 的 `#ff1e1e1e` **保持不变**。
- 影响面已核实：该色仅用于底部弹窗面板背景与其导航栏色，与主页面、偏好卡片无关。

### 5.3 v3 —— 选中态模式圆变饱和品牌蓝（真正生效的那一个）

**根因**：选中圆背景由 `c.java:94` 动态设为 `res/drawable/melody_ui_iot_mode_button_bg_selected.xml`，原渐变 `#66ffffff(40%白) → #330088ff → #4d0088ff` 白多蓝少 → 渲出发白淡蓝（≈#bfe0ff）。

**改动：`res/drawable/melody_ui_iot_mode_button_bg_selected.xml`**
```xml
<!-- 改前 -->
<gradient android:startColor="#66ffffff" android:centerColor="#330088ff"
          android:endColor="#4d0088ff" .../> <stroke ... android:color="#4dffffff"/>
<!-- 改后 -->
<gradient android:startColor="#1a9bff" android:centerColor="#0d84ff"
          android:endColor="#0077e6" .../> <stroke ... android:color="#33ffffff"/>
```
- 对齐 App 主色 `#0088ff`（见 `melody_common_heymelody_theme_blue_normal=#ff0088ff`）。
- 该资源**无 night 变体**，按"有 night 变体才一并改"原则不另建（饱和蓝深浅底通用）。

> ⚠️ **重要警示**：`res/drawable/glass_mode_circle_selected.xml` 是**未被任何布局/代码引用的废弃资源**（仅存在于 public.xml）。v2 曾改它，实际不生效；v3 改对了 `melody_ui_iot_mode_button_bg_selected.xml` 才真正生效。后续维护者**不要再去改 glass_mode_circle_selected.xml**。
> 未选中态 `melody_ui_mode_button_bg`（`solid ?couiColorLabelQuaternary`）不动。

---

## 6. 运用的开源项目（液态玻璃暗黑调研）

三份报告全文在：
- `agents/s_000EVuEnf5G/artifacts/liquid_glass_dark_research.md`
- `agents/s_000EVuEnf5G/artifacts/liquid_glass_dark_research_supplement.md`

| 仓库 | 技术栈 | 实现原理 | 暗黑适配 |
|---|---|---|---|
| QmDeve/AndroidLiquidGlassView | Java/View | `RenderEffect.createBlurEffect` + AGSL runtime shader，`mix(color,tintColor,tintAlpha)`，默认 tint 白 alpha=0 | **库本体 0 处夜间适配** |
| Kyant0/AndroidLiquidGlass | Compose | `BlurEffect` + AGSL lens/色散；库不提供 tint，由调用方 `onDrawSurface` 画 | **库本体 0 处**；catalog 示例示范了完整深色适配 |
| QWEA0/Liquid-Glass-Android | View | **采样玻璃背后像素亮度 + 滞回判定，自动翻转 tint/文字/高光/阴影** | **三者中最完善**，不依赖系统深色开关 |

**三方深色取值表（QWEA0 实测）**：玻璃 tint 深色 `#24FFFFFF`（白@14%）、模糊深色砍半（14px→6px）、文字深色白+黑投影、深色不加黑压暗层、降级实底色 `#2A2A2E`。

**本次实际参考**：v1 未选中圆 `#24ffffff`（直接对齐 QWEA0 深色磨砂 tint）；深色页面底用近黑 `#1a1d24` 系；选中蓝对齐 App 品牌蓝 `#0088ff`。核心结论——**这类玻璃库都不自带夜间适配，必须由 App 侧补 night 资源/色值**，本工程正是这么做的。

---

## 7. 后续官方版本更新 · 移植指南（给其他 AI 的标准工作流）

### 7.1 流程
新版官方 APK → `apktool d` → 对照下面"检查锚点"逐项确认问题是否仍在/资源是否被官方改 → 按第 5 节清单重打改动 → `apktool b` → uber-apk-signer 签名 → 静态+动态验证。

### 7.2 各修复的"检查锚点"（搜索关键词）
| 修复 | 锚点（grep / jadx 搜） |
|---|---|
| v1 暗黑页底 | `glass_bg_gradient`；检查 `res/drawable-night/glass_bg_gradient.xml` 是否存在；`heymelody_app_activity_detail_main.xml`、`melody_ui_fragment_detail_main.xml` 的 background |
| v1 底部导航/模式圆 | `glass_bottom_nav_bg`、`glass_mode_circle_normal` 的 drawable-night |
| v2 弹窗 | `coui_color_surface`（values 浅色值）、`DefaultBottomSheetDialog`、`design_bottom_sheet` |
| v3 选中蓝 | `melody_ui_iot_mode_button_bg_selected.xml`；jadx 搜 `devicecontrol/c.java` 的 `setBackgroundResource` |
| ⚠️防误改 | `glass_mode_circle_selected.xml`（废弃，勿改） |

### 7.3 验证清单
- **静态**：回读签名包确认目标 drawable/XML 已改、浅色/夜间色值符合；uber-apk-signer `zipalign verified` + `signature verified`；aapt/apktool 解码无报错。
- **动态（真机）**：①切暗黑模式，确认页面深底、白字可读、底部导航深色；②打开"导入固件""用户须知"两个弹窗，确认不透明白底、不与底层重叠；③依次点降噪/自适应/通透/关闭，确认选中圆是饱和蓝、白图标清晰。

### 7.4 安装注意
- debug 签名 ≠ 官方签名，**必须先卸载旧版再装**；卸载会清除 App 内耳机配对/设置数据，建议先记录。
- 想保留数据需用与官方一致的签名重签，但**没有官方私钥，技术上不可行**，故只能接受清数据。

---

## 8. 已知限制与声明

1. **未做模拟器/真机动态验证**：本沙箱无 Android SDK / emulator / adb，以上全部为**静态验证**（回编译通过 + 签名校验通过 + 资源回读确认）。真机视觉效果需人工复核。
2. **用途**：个人学习 / 逆向研究，勿用于二次分发或绕过官方权益。
3. **可能随官方更新失效的点**：
   - 资源名 `glass_bg_gradient`、`melody_ui_iot_mode_button_bg_selected`、色名 `coui_color_surface` 若官方改版改名，锚点需重新定位；
   - 模式圆若官方改用 Compose 或换控件类，`devicecontrol/c.java` 逻辑会变；
   - 弹窗若官方换用 CenterDialog 而非 BottomSheet，`coui_color_surface` 的影响范围可能改变。
