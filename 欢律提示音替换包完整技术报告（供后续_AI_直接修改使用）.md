# 欢律（HeyTap）TWS 耳机提示音个性化替换包 · 完整技术报告

> 本文档为"OPPO 欢律 App 提示音主题本地定制"项目的收尾知识交付，供后续 AI 直接使用/修改。
> 性质：基于 ProxyPin 类 HTTPS 本地代理做响应重写（responseReplace）的**本地配置层玩法**，不涉及任何系统/软件破解。
> 全部结论区分标注：**【已查证】**（实测可复现）／**【推测】**（需真机进一步验证）／**【踩坑结论】**（实战踩坑后归纳）。
> 版本：终版（含 VariantA 全人声版应用成功的完整证据链）

---

## 目录
1. 项目背景与全部需求清单
2. 原包逐文件逆向分析
3. 替换成功的技术原理
4. airpods.bin 二进制格式完整规格
5. 复刻与组装方法（操作手册）
6. 失败排查全历程
7. 最终成功方案与证据链
8. 供后续 AI 修改的变更指南
9. 附录（数据表 / URL / 素材 / 交付物）

---

## 1. 项目背景与全部需求清单

### 1.1 背景
OPPO 欢律（HeyTap）App 的"个性化提示音"功能，本应通过官方服务器下发主题。本项目证明：**用 HTTPS 本地代理（ProxyPin）重写两处接口响应，即可在不破解 App/耳机固件的前提下，把自定义语音素材打包成 App 认可的提示音主题并写入耳机**。

### 1.2 需求清单与达成情况（逐条）

| # | 需求 | 达成方式 | 状态 |
|---|---|---|---|
| 1 | 解压压缩包后详细分析包内每个文件 | 输出首份 `airpods_技术分析.md`（4 文件逐一逆向） | ✅ 已达成 |
| 2 | 详细解析为何该包能替换提示音成功 | 第 3 章：双接口 responseReplace 欺骗链路 + SHA 自洽论证 | ✅ 已达成 |
| 3 | 复刻一个同样能替换提示音的压缩包 | 复刻出"可莉提示音.zip"，随后迭代为小团团系列 | ✅ 已达成 |
| 4 | 对"文件二"（Respond.config）完美复刻 | 严格复现外层键序 / 怪癖缩进（第二主题条目开头为 `   {`（3 空格）+ 换行后 `"activity"` 行首无缩进）/ 仅替换设计字段 / 实测回填 | ✅ 已达成 |
| 5 | 做一个装上即可用的提示音替换包（含真实音频 bin） | 小团团路径成功；**可莉 bin 因私服宕机 + 公网零镜像不可得，缺口如实披露**（见 §6/§9） | ⚠️ 部分达成（小团团成功，可莉素材缺口） |
| 6 | 应用失败排查（主题出现在列表但应用失败） | 字节级复核 + H1/H2/H3 假设排除 + Variant A/B 诊断包 | ✅ 已达成 |
| 7 | 应用成功 | **VariantA 全人声版应用成功**（§7 证据链） | ✅ 已达成 |
| 8 | 出具完整报告文档 | 本文档 | ✅ 本次交付 |

---

## 2. 原包逐文件逆向分析

原压缩包解压后共 4 个文件：

| 文件名 | 大小(字节) | 角色 |
|---|---:|---|
| `LocalBin.config` | 480 | ProxyPin 规则：拦截 **bin 下载 URL**，把响应体换成本地 `airpods.bin` |
| `Respond.config` | 3789 | ProxyPin 规则：拦截 **list_theme 列表接口**，把响应体换成自定义 JSON（UI 把某主题改名"Airpods Pro"） |
| `airpods.bin` | 134548 | 真正下发给耳机的提示音固件包（头部索引表 + 9 段 AAC + 尾部 CRC） |
| `使用方法.txt` | 681 | 5 步操作说明 |

### 2.1 LocalBin.config（480B）
```json
[{"name":"Local","enabled":true,
  "url":"https://iot-earbuds-cn.heytapimage.com/earbuds/test-report-20241217200911378-1007_BDD_0x3EF_CH_IP_AAC_PACKAGE_1202.bin",
  "type":"responseReplace",
  "items":[
    {"enabled":false,"type":"replaceResponseStatus","values":{}},
    {"enabled":false,"type":"replaceResponseHeader","values":{}},
    {"enabled":true,"type":"replaceResponseBody",
     "values":{"bodyType":"file",
               "bodyFile":"/data/user/0/com.network.proxy/cache/file_picker/1746469090771/airpods.bin"}}
  ]}]
```
- `type:"responseReplace"` = 响应重写；三条 items 分别对应 状态码/响应头/响应体，本包**仅启用响应体替换**。【已查证】
- `bodyType:"file"` + `bodyFile` = 命中后 ProxyPin 直接把该本地文件的字节流当作 HTTP 响应体回吐给 App。【已查证】
- 拦截目标 = heytapimage 上那条 `1007_BDD_0x3EF_CH_IP_AAC_PACKAGE_1202.bin`。【已查证】

### 2.2 Respond.config（3789B）
同构规则，拦截 `https://iot-earbuds-cn.heytapmobi.com/v1/earphone/personalize/list_theme`，`replaceResponseBody`(text) 回吐一段伪造 JSON。内嵌 JSON 的 `data` 含 2 个主题：
- 条目1 `title:"默认"`：伪装官方默认主题，`toneUrl:""`（无独立音频包）。
- 条目2 `title:"Airpods Pro"`：本包目标主题，`themeId:"7"`、`themeIdentifier:"1007"`、`materialType:1`，关键字段：

| 字段 | 值 |
|---|---|
| `toneUrl` | `https://iot-earbuds-cn.heytapimage.com/earbuds/test-report-20241217200911378-1007_BDD_0x3EF_CH_IP_AAC_PACKAGE_1202.bin` |
| `toneSHA256` | `f820c86a5f89c857dee4521e8b3627106b72d292692dec4c6b6feceeb4092219` |
| `toneSize` | `134548` |

### 2.3 双规则同一 URL 联动【已查证】
> **Respond.config 条目2 的 `toneUrl` 与 LocalBin.config 的 `url` 逐字符相等。**

即：Respond 骗"列表 UI"，LocalBin 骗"实际下载"，两条规则靠同一个 bin 下载 URL 闭环衔接。

### 2.4 使用方法.txt（681B）——5 步技术意图
1. 装 ProxyPin + 修改版欢律，开 HTTPS 代理，导入两条规则。
2. 进入 LocalBin 规则，响应体类型选"文件"，重选本地 `airpods.bin`（配置里是作者旧路径，必须重选）。
3. **先不开代理**，任选一个非默认主题应用、确认提示音变了，再彻底杀欢律进程（清缓存+排除耳机异常）。【踩坑结论】
4. 开代理总开关 → 开欢律 → 选"妙趣童声"（此时标题应显示"Airpods Pro"）→ 等加载。
5. Enjoy。

---

## 3. 替换成功的技术原理

### 3.1 正常链路
```
欢律App ──GET list_theme──────────► HeyTap(iot-earbuds-cn.heytapmobi.com)
        ◄── [{title, toneUrl, toneSHA256, toneSize, …}]
        用户点主题
        ──GET toneUrl──────────────► HeyTap(iot-earbuds-cn.heytapimage.com)
        ◄── bin 字节流
        App 按 toneSHA256/toneSize 校验 → 蓝牙写入耳机
```

### 3.2 ProxyPin 的双欺骗
| 篡改点 | 规则 | 命中 URL | 效果 |
|---|---|---|---|
| 列表 | Respond.config | `…/personalize/list_theme` | UI 把"妙趣童声"改名"Airpods Pro"，并声明自定义 toneUrl/SHA/Size |
| 下载 | LocalBin.config | heytapimage 上的 `…1007_BDD….bin` | App 下载到的 bin 实际是本地文件 |

### 3.3 为什么 App 会接受
- **HTTPS 信任链【已查证】**：用户安装并信任 ProxyPin 根证书；欢律（修改版/未 Pinning 版本）经代理走 HTTPS，TLS 被 ProxyPin 终结重建，从而能改写明文响应。【推测】官方版欢律可能有 SSL Pinning，故配套"修改版"。
- **App 只按响应内容行事【踩坑结论】**：App 不校验"响应是不是服务器真发的"，只解析 JSON 字段；下载 bin 时只校验**内容 SHA/大小**，不校验来源。

### 3.4 替换成功的必备条件（实战归纳）
1. `toneUrl` 必须是 **App 真实会请求的活域名 URL**（本轮成功的关键修正，见 §6/§7）。
2. `LocalBin.config` 的 `url` 必须与该 `toneUrl` **逐字一致**才能命中。
3. `toneSHA256` / `toneSize` 必须等于**实际下发 bin 的实测 SHA256 / 字节数**（§3.5 强证据）。
4. bin 本身格式合法（头部/条目表/CRC 自洽，§4）。

### 3.5 SHA 自洽的强证据【已查证】
```
实测: sha256(airpods.bin) = f820c86a5f89c857dee4521e8b3627106b72d292692dec4c6b6feceeb4092219
配置: toneSHA256          = f820c86a5f89c857dee4521e8b3627106b72d292692dec4c6b6feceeb4092219   ✓
实测: 大小 = 134548 ; 配置 toneSize = 134548                                              ✓
```
作者事先把真实 bin 的哈希/大小填进伪造 JSON，LocalBin 再回吐同一 bin，App 校验自然通过。**这反证 App 端存在基于 toneSHA256/toneSize 的下载完整性校验。**【已查证事实 + 校验存在性推论】

---

## 4. airpods.bin 二进制格式完整规格

### 4.1 头部前 48 字节（十六进制实测）
```text
00000000  ef 03 00 00 00 07 00 00  42 55 44 4f 55 44 4f 55   ........BUDOUDOU
00000010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00   ................
00000020  00 00 00 00 00 00 00 00  09 00 00 00 07 00 00 00   ................
```

| 偏移 | 字节 | 值 | 含义 |
|---|---|---|---|
| 0x00–0x03 | `ef 03 00 00` | u32 LE = 0x03EF = **1007** | 包魔数/耳机型号代码，与文件名 `1007_BDD_0x3EF`、JSON `themeIdentifier:"1007"` 互证【已查证】 |
| 0x04–0x07 | `00 07 00 00` | u32 LE = 0x00000700 | 版本/标志（byte5=0x07），确切含义未明【推测】 |
| 0x08–0x0F | `42…55` | ASCII = **"BUDOUDOU"** | 布兜兜拼音魔数【已查证】 |
| 0x10–0x27 | 全 0 | 24 字节零块 | 保留/对齐【已查证】 |
| 0x28–0x2B | `09 00 00 00` | u32 LE = **9** | 音频条目数=9【已查证】 |
| 0x2C–0x2F | `07 00 00 00` | u32 LE = 7 | 含义未明【推测】 |
| 0x30 起 | — | 条目表 | 见 §4.2【已查证】 |

### 4.2 条目表（索引表）
- 结构：**9 条 × 12 字节**，每行 3 个 u32 小端 = `(数据区绝对偏移, 声明大小, 第三列)`。【已查证】
- 偏移为**相对文件头的绝对偏移**；第一条偏移 156 = 表头长度，故**表头共 156 字节**。【已查证】
- 条目偏移表：`[156, 14457, 29168, 39660, 59700, 73848, 87456, 102043, 119251]`，末段结束于 134528。【已查证】
- **第三列语义结论【踩坑结论】**：实测逐段计算 crc32 低/高 8 位、累加和、crc16、整行 CRC、ADTS 帧数，**全部不匹配** → 判定第三列是**事件 ID/参数类字段**（值 `[8,12,14,29,28,5,6,51]`），不是每段校验；第 9 条 `0x495FF407` 为打包工具残留。

### 4.3 数据段：9 条 AAC（ffprobe 实测）
全部为 **AAC LC / 16000 Hz / 单声道 / 裸 ADTS**，段内含编码器串 `Lavc61.2.100`（FFmpeg 61.2.100 产出）：

| # | 偏移(dec) | 实际长度(B) | 第三列 | 时长(s) |
|---|---:|---:|---|---:|
| 1 | 156 | 14301 | 8 | 2.49 |
| 2 | 14457 | 14711 | 12 | 2.33 |
| 3 | 29168 | 10492 | 14 | 2.04 |
| 4 | 39660 | 20040 | 29 | 3.74 |
| 5 | 59700 | 14148 | 28 | 2.34 |
| 6 | 73848 | 13608 | 5 | 2.23 |
| 7 | 87456 | 14587 | 6 | 1.95 |
| 8 | 102043 | 17208 | 51 | 2.30 |
| 9 | 119251 | 15277 | 0x495FF407 | 2.74 |

> 说明：第 4 条表内"声明大小"为 59699（=下一条偏移−1），与实际连续长度 20040 不符；按连续偏移切分才能与其余 8 条严丝合缝拼接。属打包工具 quirk，组装新包时以"逐段真实长度 + 顺序拼接"为准。【踩坑结论】

### 4.4 尾部与 CRC【已查证】
```text
00020D80  0A 50 52 4F 4D 50 54 5F 43 52 43 3D 38 41 44 31   .PROMPT_CRC=8AD1
00020D90  42 44 44 43                                       BDDC
```
- 末段音频结束于 134528；其后 1 字节 `0x0A` + ASCII `PROMPT_CRC=8AD1BDDC`。
- 实测 `zlib.crc32( file[0:134528] ) = 0x8AD1BDDC`，与尾部字符串**完全相等** → 这是对"文件头+全部音频"（不含尾部自身）的 32 位 CRC 自校验。

---

## 5. 复刻与组装方法（给后续 AI 的操作手册）

### 5.1 任意语音素材 → 可用 bin 全流程
1. **裁剪素材**：每段提示音裁到所需时长（小团团全人声版每段约 3.5s）。
2. **统一转码**：全部转成 AAC LC / 16 kHz / 单声道 / 裸 ADTS（与原包一致）：
   ```bash
   ffmpeg -i src.mp3 -ac 1 -ar 16000 -c:a aac -profile:a aac_low -b:a 48k segN.aac
   ```
   （编码器版本不必强求 61.2.100，App 仅按 ADTS 解码，实测可接受。）
3. **写头部**：48 字节固定头（魔数 1007 + 版本 + `BUDOUDOU` + 零块 + count=段数 + 0x07）。
4. **写条目表**：N 条 ×12B，`(u32绝对偏移, u32本段长度, u32第三列)`；第三列沿用原包 `[8,12,14,29,28,5,6,51,…]` 顺序即可（事件 ID，无需自定义）。偏移从 156 起、逐段累加。
5. **拼接**：头 + 表（凑足 156 字节）+ 各 AAC 段顺序相接。
6. **写尾部**：`0x0A` + `PROMPT_CRC=%08X`（大写），其中 CRC = `crc32(前 134528/全部音频区之前的字节)`。
7. **回填**：对最终 bin 算 sha256 与 size，填进 Respond.config 的 `toneSHA256`/`toneSize`。

### 5.2 Respond.config 完美复刻规则【踩坑结论】
- 外层是**单元素数组** `[{…}]`。
- 保留原 JSON 的**键顺序**与**怪癖缩进**（第二主题条目开头为 `   {`（3 空格）+ 换行后 `"activity": false` 行首无缩进，即 `'   {\n"activity"'`，而非正常对齐的 6 空格缩进）——App 不强求格式，但 ProxyPin 文本匹配/排错时按原样最稳。
- **只替换"设计字段"**：`title`、`summary`、`toneUrl`、`toneSHA256`、`toneSize`、`previewListImgUrl` 等；其余字段（动画/默认主题条目）原样保留。
- `values.body` 是**内嵌的字符串化 JSON**，注意它本身又是一层 JSON（双层 JSON 嵌套，见 §8 验证清单）。

### 5.3 LocalBin.config 写法与坑
- `url` 必须与 Respond.config 里目标条目的 `toneUrl` **逐字相等**。
- `bodyType:"file"`，`bodyFile` 指向本地 bin。
- **坑【踩坑结论】**：`bodyFile` 里写的是作者手机旧路径，导入后**必须在 ProxyPin 里重新"选择文件"指向你的 bin**，否则命中后回吐的是空/错文件。

---

## 6. 失败排查全历程（踩坑记录）

### 6.1 现象
首版小团团包：`toneUrl` 指向**已宕机**的第三方私服 `tone.giycfo.icu/tone/xiaotuantuan.bin`。用户在列表里能看到主题，但**应用失败**；ProxyPin 悬浮日志只见 `obus-cn.dc.heytapmobi.com` 请求，不见 heytapimage 下载请求。

### 6.2 三个假设
- **H1**：LocalBin 未命中（bodyFile 导入后失效 / URL 不匹配 / 规则未启用）。
- **H2**：App 不走 toneUrl，而是走 obus 材料接口去拿真·布兜兜 bin。
- **H3**：bin 内容被拒（格式/CRC 不对）。

### 6.3 逐项排除实测
- **头部比对【已查证】**：自组 bin 的前 48 字节与原包逐字段一致（`ef030000 / 00070000 / BUDOUDOU / 16 字节零块 / 09000000 / 07000000`）。
- **第三列语义【踩坑结论】**：对原包 9 段分别算 crc32 低 8 位/高 8 位/累加和/crc16/整行 crc/ADTS 帧数，**无一匹配** → 第三列是事件 ID，不是每段校验；第 9 条 `0x495FF407` 为打包工具残留。
- **CRC 尾部自洽**：自组 bin 的尾部 CRC 正确。
- → **H3 排除**（bin 内容格式没问题）。
- 真正根因：**toneUrl 指向了宕机私服**，App 根本没能成功下载到 bin（或根本不该指那），LocalBin 也无从命中 → 归于 **H1 的 URL 目标错误**。

### 6.4 决策
把 `toneUrl` **改回原包作者验证过的活域名 heytapimage 那条 bin 地址**（App 真实会请求它），SHA/Size 回填自组 bin 的实测值；并出：
- **Variant A（主推）**：全人声版，SHA/Size 用实测自组值。
- **Variant B（诊断探针）**：SHA/Size 故意填"真布兜兜"值，用于分辨是"拦截命中但校验失败"还是"根本没命中"。
- 增强说明书：重选文件 / 看命中次数 / A-B 判定表 / 用户反馈清单。

### 6.5 方法论【踩坑结论】
> **以 ProxyPin 悬浮日志中真实出现的 URL 为准来修正拦截目标，而不是相信配置里假设的 URL。** 日志里同时出现 `heytapmobi`(obus 详情) 与 `heytapimage`(bin 下载) = 链路打通的标志。

---

## 7. 最终成功方案与证据链（VariantA 全人声版）

### 7.1 关键改动
- `toneUrl` = 原包活域名 `https://iot-earbuds-cn.heytapimage.com/earbuds/test-report-…1007_BDD_….bin`。
- Respond.config 主题改名"小团团提示音"，summary 写"小团团提示音（社区语音素材全人声组装版…）"。
- `toneSHA256`/`toneSize` = VariantA 全人声 bin 的实测值（§9 数据表）。
- LocalBin 拦截同一 heytapimage URL，`bodyFile` 指向 172985 字节的全人声 bin。

### 7.2 成功证据（用户截图）
| 证据点 | 截图表现 | 解读 |
|---|---|---|
| 详情页标题 | "小团团提示音" | Respond.config 的 UI 欺骗生效 |
| 大小 | 168.9 KB | ≈ 全人声 bin 172985 字节（168.9×1024=172956，吻合）✓ |
| summary | "小团团提示音（社区语音素材全人声组装版…）" | 自定义文案生效 |
| 底部按钮 | 绿色"已应用" | 应用流程走完 |
| 悬浮日志 | **同时出现** `obus-cn.dc.heytapmobi.com/v3_1/…` **与** `iot-earbuds-cn.heytapimage.com/e…` | App 确实请求了活域名 toneUrl → LocalBin 命中 → 本地 bin 下载成功 → SHA 校验通过 → 写入耳机 |

### 7.3 为什么这次能成功【踩坑结论】
1. toneUrl 换成了 App **真实请求的活域名**（不再是宕机私服）。
2. LocalBin 拦截 URL 与 toneUrl 逐字一致，悬浮日志证实命中。
3. toneSHA256/toneSize 回填了**实际下发 bin 的实测值**，校验通过。
4. bin 格式严格复用原包规格（头部/条目表/CRC）。

---

## 8. 供后续 AI 修改的变更指南

| 想改什么 | 改哪里 | 注意 |
|---|---|---|
| 换主题名 | Respond.config 内嵌 JSON 里的 `title` / `summary` | 保留双层 JSON 转义 |
| 换语音素材 | 重新按 §5.1 转码、切段、组 bin | 段数变了要改头部 count 与条目表 |
| 增删提示音段 | 头部 0x28 的 count + 条目表行数 + 偏移表重算 | 第三列事件 ID 沿用原顺序 |
| 换耳机型号 | 头部 0x00 魔数 + 文件名/JSON `themeIdentifier` | 不同型号 bin 布局可能不同，需真机验证【推测】 |
| 换下载域名 | toneUrl + LocalBin.url 同步改 | **必须是 App 真实请求的活域名** |

**每次改动后必须重算**：bin 的 `sha256`、`size`、尾部 `PROMPT_CRC`，并回填 Respond.config。

**验证清单（缺一不可）【踩坑结论】**：
1. zip 解压回环无损坏。
2. Respond.config 是**双层合法 JSON**（外层数组可 parse，内嵌 body 字符串也可 parse）。
3. 每段 AAC 用 ffprobe 都能解出（AAC LC/16k/单声道）。
4. 自组 bin 头部前 48 字节与原包逐字段比对一致。
5. Respond.config 填的 `toneSHA256`/`toneSize` 与最终 bin 实测值**逐字一致**。

**常见坑**：
- 导入 LocalBin 后要在 ProxyPin 里**重选 bin 文件**（bodyFile 旧路径失效）。
- 旧规则冲突：残留的宕机私服规则要禁用/删除。
- 静音段会被固件视为空提示音，建议每段都放真人声。
- **toneUrl 指向的域名必须当下可用**，否则复现首版失败。

---

## 9. 附录

### 9.1 三个 bin 实测数据表

| bin | 大小(B) | SHA256 | 尾部 CRC | 状态 |
|---|---:|---|---|---|
| airpods.bin（原包） | 134548 | `f820c86a5f89c857dee4521e8b3627106b72d292692dec4c6b6feceeb4092219` | `8AD1BDDC`(=crc32[0:134528]) | 原包 |
| 小团团.bin（Variant A/B 共用，4人声+5静音） | 66615 | `e808271647dd4376621120ecb358be1f502d1420bd4f5d5af3c3ec8439d4c6f7` | `95291139` | 诊断/中间版 |
| 小团团.bin（VariantA 全人声版，**成功版**） | 172985 | `f8770baad6b8b436c8030c9eb8d21814252dd0d10c1f340b964ebaef93d345ea` | `778C0234` | ✅ 已应用成功 |

**可莉 bin（仅配置声明，未实际获取到）**：
- `kelitsy.bin`：125563B / `935630eaf7a53949d45d06b7ff4258884b346fdfa085180353bcfcaec8760a70`
- `keliv8.bin`：125563B / `f334bf2c91e554b3f4a1bed00447666008cc49f523244c57c7ee581d1068d916`

### 9.2 VariantA 全人声版 9 段内容与时长（合计 28.37s）
| 段 | 内容 | 时长(s) |
|---|---|---:|
| 1 | 来啦老弟 | 1.86 |
| 2 | 拜拜 | 3.20 |
| 3 | 蹦沙卡拉卡 | 2.56 |
| 4 | 准备好了吗出发 | 2.69 |
| 5 | 充电 zhiting | 3.56 |
| 6 | 充电合集截句 | 3.56 |
| 7 | 叫宝贝 | 3.88 |
| 8 | 马戏团串烧截句 | 3.57 |
| 9 | 来电开场 | 3.49 |

### 9.3 9 条原始素材清单
| 文件 | 内容 | 原始时长(s) | 来源 |
|---|---|---:|---|
| 01 | 来啦老弟 | 1.78 | sound.jichun29.cn |
| 02 | 拜拜 | 3.16 | sound.jichun29.cn |
| 03 | 蹦沙卡拉卡 | 2.53 | sound.jichun29.cn |
| 04 | 准备好了吗出发 | 2.64 | sound.jichun29.cn |
| 05 | 充电提示音 zhiting | 10.06 | B站 |
| 06 | 充电合集 | 17.16 | B站 |
| 07 | 叫宝贝充电 | 17.74 | B站 |
| 08 | 马戏团串烧合集 | 65.85 | B站 |
| 09 | 来电开场 | 21.94 | B站 |

> B站来源：BV1Nv411k7dv / BV1Nk4y117h4 / BV1Ka4y177zZ / av821444807 / BV1Xz4y1J7Ej（全人声版已裁剪为每段约 3.5s）。

### 9.4 关键 URL 清单
| 用途 | URL | 状态 |
|---|---|---|
| 主题列表接口（Respond 拦截） | `https://iot-earbuds-cn.heytapmobi.com/v1/earphone/personalize/list_theme` | 活 |
| bin 下载活域名（LocalBin 拦截，**本轮成功关键**） | `https://iot-earbuds-cn.heytapimage.com/earbuds/test-report-20241217200911378-1007_BDD_0x3EF_CH_IP_AAC_PACKAGE_1202.bin` | 活 |
| 材料/详情接口（应用阶段 App 会请求） | `https://obus-cn.dc.heytapmobi.com/v3_1/…` | 活 |
| 已宕机域名（可莉 bin 唯一分发点，无公网镜像） | `https://tone.giycfo.icu/tone/keliv8.bin`、`…/kelitsy.bin` | ❌ 502/连接失败 |

### 9.5 原包头部十六进制（前 48 字节，供字节比对）
```text
ef 03 00 00 00 07 00 00  42 55 44 4f 55 44 4f 55
00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00  09 00 00 00 07 00 00 00
```

### 9.6 交付物清单
- `airpods_技术分析.md`：首轮逐文件逆向分析（本文档的素材基础）。
- `欢律提示音替换包完整技术报告.md`：本文档。
- 可莉提示音.zip / 小团团系列包：复刻产物（小团团 VariantA 全人声版为最终成功版）。
- 9 段 AAC 抽取样本：`airpods_work/segments/`（seg1…seg9）。

### 9.7 ProxyPin 使用要点
1. 装 App → 开 HTTPS 代理 → 安装并信任根证书。
2. 导入 LocalBin.config + Respond.config，均 `enabled:true`。
3. 进 LocalBin 规则 → 响应体类型选"文件" → **重选**你的 bin。
4. 先关代理、随便应用一个主题热身并杀进程；再开代理、点目标主题。
5. 看悬浮日志是否同时出现 obus + heytapimage 两域名，判定链路是否打通。

---

*报告完。所有证据性数字均来自实测或组织者核实，未作编造。源素材全程只读。*
