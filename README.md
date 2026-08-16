# 弹幕 JSON → ASS 转换器（NX-Jikkyo 渲染模型）

把 Niconico 弹幕导出 JSON（`packet/chat` 结构，如 `jk1.json`）转换为 ASS 字幕。**时间轴、颜色、位置、字号、轨道规则均对照 NX-Jikkyo v1.14.7 官方源码实现，非估算**；由于 ASS 是预渲染字幕、DPlayer 是浏览器实时渲染（字体测量与窗口尺寸不同），视觉上尽量复刻、无法逐像素一致。网页版（纯前端）与 Python 版逻辑相同。

> **作者与来源声明**
> - 本项目（网页工具、Python 脚本、文档）由 **DeepSeek AI 编写生成**，作者按需求提供数据与反馈。
> - 本项目的渲染模型、弹幕解析规则、颜色表、时间轴公式**改编自 [NX-Jikkyo](https://github.com/tsukumijima/NX-Jikkyo)（v1.14.7，MIT License）及其使用的 [tsukumijima/DPlayer fork](https://github.com/tsukumijima/DPlayer)（v1.32.8）**，并非凭空估算。
> - 本项目是**独立工具**，与 NX-Jikkyo 官方项目无关联、非其衍生项目的一部分；仅复刻其渲染规则。
> - 上游仓库均为 **MIT License**，本项目遵循同样许可；使用或分发时请保留上游版权声明（见下文「九、作者与许可」）。

---

## 一、项目文件

| 文件 | 说明 |
|---|---|
| `danmaku-ass-converter.html` | 网页版：上传/URL/粘贴 JSON → 预览 → 下载 .ass，纯前端零依赖，可部署到 GitHub Pages |
| `danmaku_nxjikkyo.py` | Python 版：命令行转换，用 PIL/matplotlib 精确测量文字宽度（需已安装 Noto Sans JP Bold） |
| `convert_nico.py` | 辅助脚本：把 `packet/chat` 格式整理成 `{"comments":[...]}`（含 size/color 字段），并输出统计 |
| `preview_nxjikkyo_v4.ass` | 用 jk1.json 生成的示例（date 模式：视频开始=第一条弹幕发布时间，弹幕落在 0~30 分钟） |

---

## 二、数据来源与源码依据（来源）

本项目的行为全部取自以下两个开源仓库：

1. **NX-Jikkyo**（弹幕观看工具本体）：<https://github.com/tsukumijima/NX-Jikkyo>
2. **tsukumijima/DPlayer fork**（NX-Jikkyo 使用的播放器/弹幕引擎，v1.32.8）：
   - 仓库：<https://github.com/tsukumijima/DPlayer>
   - NX-Jikkyo 的 `client/package.json` 中声明依赖 `"dplayer": "github:tsukumijima/DPlayer#v1.32.8"`

### 关键源码位置

| 规则 | 文件 |
|---|---|
| 弹幕解析（颜色/位置/大小/184） | `NX-Jikkyo/client/src/utils/CommentUtils.ts`（`parseCommentCommand`、`color_table`、`getCommentPosition`、`getCommentSize`、`isSpecialCommandComment`） |
| 播放器实例化参数 | `NX-Jikkyo/client/src/services/player/PlayerController.ts`（`danmaku: { fontSize, speedRate, opacity... }`） |
| 直播弹幕实时显示 | `NX-Jikkyo/client/src/services/player/managers/LiveCommentManager.ts`（`playback_position = video.currentTime`，延迟 = 缓冲延迟 + 用户设置延迟） |
| **录像回放时间轴** | `NX-Jikkyo/client/src/services/player/PlayerController.ts` 的 `fetchVideoJikkyoComments`（`comment_time = chat_date − start_time`） |
| 设置默认值 | `NX-Jikkyo/client/src/stores/SettingsStore.ts`、`Jikkyo.vue`（`comment_font_size: 34`、`comment_speed_rate: 1`、`comment_delay_seconds: 0`） |
| 弹幕渲染引擎（字号/行高/速度/轨道） | `DPlayer fork/src/ts/danmaku.ts`（`ratio`、`itemHeight`、`danSpeed`、`getTunnel`、`_danAnimation`） |
| 阴影/字体样式 | `DPlayer fork/src/ts/danmaku.scss`（`text-shadow: 1.2px 1.2px 4px rgba(0,0,0,.9)`、粗体） |

> 说明：DPlayer 官方 master 分支的 `danmaku.js` **不支持 size（big/small）**；NX-Jikkyo 使用的是 tsukumijima 的 fork，fork 在 `src/ts/danmaku.ts` 中才加入了对 size 的处理。因此必须对照 fork 源码。

---

## 三、弹幕解析规则（对应 `CommentUtils.ts`）

### 1. mail 命令解析（`parseCommentCommand`）

- 先去掉命令串中的 `184`（心形命令，见 FAQ）
- 按空格分词，逐个识别：
  - **颜色词** → 查下表；未识别/无颜色 → 默认 `#FFEAEA`
  - `ue` → 顶部（top）、`shita` → 底部（bottom）、`naka`/无 → 滚动（right）
  - `big` → 大字、`medium` → 中字（默认）、`small` → 小字

### 2. 官方色表（`color_table`，27 个别名）

| 命令 | 颜色 | 命令 | 颜色 |
|---|---|---|---|
| white | `#FFEAEA`（默认粉白） | white2 / niconicowhite | `#CCCC99` |
| red | `#F02840` | red2 / truered | `#CC0033` |
| pink | `#FD7E80` | pink2 | `#FF33CC` |
| orange | `#FDA708` | orange2 / passionorange | `#FF6600` |
| yellow | `#FFE133` | yellow2 / madyellow | `#999900` |
| green | `#64DD17` | green2 / elementalgreen | `#00CC66` |
| cyan | `#00D4F5` | cyan2 | `#00CCCC` |
| blue | `#4763FF` | blue2 / marineblue | `#3399FF` |
| purple | `#D500F9` | purple2 / nobleviolet | `#6633CC` |
| black | `#1E1310` | black2 | `#666666` |
| niconico | `#FFCC00` | | |

注意：这些是 **NX-Jikkyo 官方色值**，与一般印象中的"纯色"不同（如白色是粉白 #FFEAEA、红色是 #F02840）。

### 3. 输入 JSON 格式（两种都支持）

```jsonc
// 格式A：Nico 导出（jk1.json）
{ "packet": [ { "chat": { "vpos": "4487395", "date": "1785223740", "date_usec": "357378",
                            "content": "弹幕内容", "mail": "184 red" } } ] }

// 格式B：已整理（convert_nico.py 输出）
{ "comments": [ { "time": 0.36, "text": "弹幕内容", "type": "right", "size": "medium", "color": "#FFEAEA" } ] }
```

---

## 四、渲染模型（对应 DPlayer fork `danmaku.ts`）

| 参数 | 值 | 源码依据 |
|---|---|---|
| 字号 | `34 × min(播放器宽 ÷ 1024 × 1.25, 1)`；宽 ≥ 819px 时**固定 34px 不再放大** | `ratio = containerWidth / 1024 * ratioRate; if (ratio >= 1) ratio = 1` |
| 行高 | `字号 + 6` | `itemHeight = itemFontSize + 6 * ratio` |
| 行数 | `ceil(播放器高 ÷ 行高)` —— **随播放器大小变化** | `itemY = danHeight / itemHeight`，循环 `i < itemY` |
| 大小倍率 | big = ×1.25、medium = ×1.0、small = ×0.8（**不影响行高**） | `switch (dan.size)` |
| 滚动弹幕 | **固定 5 秒横跨全屏**（全屏模式 5.5s），长弹幕移动更快 | `_danAnimation('right') = (isFullScreen ? 5.5 : 5) / rate` |
| 顶部/底部 | 固定 4 秒居中（全屏 4.5s） | `_danAnimation = (isFullScreen ? 4.5 : 4) / rate` |
| 轨道分配 | 同轨"跟车"（间距约 10px，`danItemRight - 10` 判定）；顶/底同轨同时只有 1 条 | `getTunnel` 精确复刻 |
| 阴影 | `1.2px 1.2px 4px rgba(0,0,0,0.9)`，正文粗体 | `danmaku.scss` `text-shadow` + `font-weight: bold` |
| 透明度 | 默认 1.0（`opacity` 设置未开启时不透明） | DPlayer `opacity` 选项 |

**为什么同屏弹幕数量会变？**
行数 = 播放器高 ÷ 40（字号 34 时）。例如：1920×1080 全屏 → 27 行；1280×720 → 18 行；960×540 → 14 行。
想看和 NX-Jikkyo 一致的同屏数量，就把 W/H 填成你**实际播放器窗口**的像素大小。

---

## 五、时间轴（重要，对应 `fetchVideoJikkyoComments`）

NX-Jikkyo 录像回放的核心代码：

```js
const comment_time = parseFloat(`${chat_date - start_time}.${chat_date_usec}`);
// chat_date  = 每条弹幕的发布时间（unix 秒）
// start_time = 你视频的开始时间（unix 秒）
// 弹幕显示位置 = 弹幕发布时间 − 视频开始时间（相对你的视频）
```

**vpos 不是正确的时间基准**：`vpos` 是相对"直播开播时刻"的 1/100 秒，只有你的视频恰好从直播开播时刻开始时才等价于上式。正确做法是填**你视频的开始时间**：

- **网页版**：用「视频开始时间」**日期时间选择器**填你视频文件开头的时间（**电脑本地时间**即可，浏览器自动转 unix，已验证往返无损）；也可以点「⏱ 自动填入第一条弹幕发布时间」。改完立即生效。
- **Python 版**：`--start-time <unix秒>`，可选 `--end-time <unix秒>` 丢弃范围外弹幕。

```bash
# 例：视频开始于 2026-07-28T07:29:00Z（unix 1785223740），结束于 07:59:00Z（1785225540）
python danmaku_nxjikkyo.py jk1.json out.ass 1920 1080 --start-time 1785223740 --end-time 1785225540
```

> 提示：加载数据后，统计区会显示"发布时间范围（本地时区）"，直接和你的视频时间对照填写即可。

---

## 六、网页版用法与部署（GitHub Pages）

1. 上传 `danmaku-ass-converter.html` 到仓库根目录（改名 `index.html` 可直接访问根地址）；
2. **Settings → Pages** → Deploy from a branch → `main` / root → Save；
3. 访问 `https://用户名.github.io/仓库名/`。

功能：
- 三种输入：上传文件 / 填 URL（需支持 CORS，如 `raw.githubusercontent.com`）/ 粘贴 JSON
- 统计：条数、时间范围、发布时间范围（本地时区）、类型/尺寸/颜色分布、轨道行数
- 选项：W/H（播放器尺寸）、字号、速度倍率、ASS 字体名、无限制轨道、视频开始/结束时间
- 实时画布预览（参数与 ASS 完全一致）+ 下载 .ass

---

## 七、常见问题（FAQ）

**Q1：「184」弹幕为什么显示成文字而不是心形？**
`184` 是 Nico 的心形命令，原站由播放器渲染成 ❤️。ASS 无法直接渲染，本工具按弹幕正文原样显示。如需替换，可把正文里的内容替换成 `♥`。

**Q2：网页预览和 ASS 宽度有差别吗？**
网页用浏览器 `measureText` 测宽（字体近似），Python 版用 PIL + Noto Sans JP Bold 精确测宽。宽度只影响滚动速度的精确度与轨道碰撞，不影响颜色/字号/位置等样式。

**Q3：为什么有的弹幕被丢弃了？**
`getTunnel` 找不到可用轨道时丢弃（同屏超过行数上限）；另外填了视频开始/结束时间后，范围外发布的弹幕也会被丢弃。

**Q4：弹幕间隔太密/太疏？**
行数 = 播放器高 ÷ (字号+6)，把 W/H 填成你实际看 NX-Jikkyo 的窗口大小即可；速度由 `comment_speed_rate`（默认 1）控制。

**Q5：Python 版报 `Font not found`？**
需要系统装有 **Noto Sans JP Bold**（matplotlib 的字体缓存里能找到）。脚本用 matplotlib 的 `findfont("Noto Sans JP Bold")` 定位。

**Q6：视频时长比弹幕范围短/长怎么办？**
用「视频结束时间」丢弃超出视频长度的弹幕；弹幕只出现在视频开始时间之后，之前不会有弹幕（与 NX-Jikkyo 行为一致）。

---

## 九、作者与许可

### 作者

本项目由 **DeepSeek（AI 助手）** 编写生成：

- `danmaku-ass-converter.html`（网页版主工具）
- `danmaku_nxjikkyo.py`（Python 命令行版）
- `convert_nico.py`（格式转换辅助脚本）
- `README.md`（本文档）

作者（用户）负责提供需求、真实数据（如 `jk1.json`）与迭代反馈。

### 来源与改编说明

本工具**并非原创渲染算法**，而是对以下开源项目源码的分析与复刻：

| 上游项目 | 版本 | 许可 | 本工具复刻的内容 |
|---|---|---|---|
| [tsukumijima/NX-Jikkyo](https://github.com/tsukumijima/NX-Jikkyo) | v1.14.7 | MIT © 2024 tsukumi | 弹幕解析规则（`CommentUtils.ts`）、官方色表、录像回放时间轴公式（`fetchVideoJikkyoComments`） |
| [tsukumijima/DPlayer fork](https://github.com/tsukumijima/DPlayer) | v1.32.8 | MIT | 渲染模型（`danmaku.ts`）：字号缩放、行高/行数、滚动/顶底时长、`getTunnel` 轨道分配、阴影样式（`danmaku.scss`） |

> 简单说：**算法和参数来自 NX-Jikkyo / DPlayer fork，代码实现由 DeepSeek 编写。**

### 许可

- 本项目代码以 **MIT License** 发布。
- 上游 NX-Jikkyo 与 DPlayer fork 均为 MIT License，本项目使用、修改其规则时保留了上述出处声明。
- 使用或再分发本项目时，请保留本文件的「作者与来源声明」与上游版权信息。
