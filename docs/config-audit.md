# HFA 配置审计报告

- 审计对象：`singlefile-settings-HFA.json`
- 适用版本：SingleFile 1.28.0（配置基线为 2026-09-04 由用户重新导出的 1.24.0 快照）
- 审计基准：commit `bbea736`（148 键）
- 最近审计：2026-10-04（1.28.0 上游核对）；此前各版核对见「结论摘要 › 版本核对表」与「附录 A」
- 当前状态：148 键，与上游 `v1.28.0` 的 `DEFAULT_CONFIG` **键集完全一致**（无缺失、无多余，键序按码位升序）；关键值调整：`imageQuality = 1`、`networkTimeout = 0`、`passReferrerOnError = true`、`loadDeferredContentMinZoomFactor = 0.5`、`loadDeferredContentMaxIdleTime = 20000`，均见「三、发现与处置」
- 方法：静态审计 + 与 1.24.0 真实导出的键级 diff + 与上游 `v1.28.0` 源码 `src/core/bg/config.js` 的 `DEFAULT_CONFIG`（148 键）键级比对，并以脚本复现扩展 `upgrade()` 的迁移逻辑做等价性验证；另对 core 改动逐条核对可达性 —— JSON 结构、内部一致性、字段语义归类。源码无法确证的语义仍标注为推断

## 结论摘要

- **结构健康**：JSON 语法有效；148 键与上游 `DEFAULT_CONFIG` 键集完全一致；无影响高保真目标的矛盾配置；无强制修改项。
- **保存格式 = 自解压 ZIP（universal）**：`compressContent = true`（格式总开关）+ `selfExtractingArchive = true` + `extractDataFromPage = true`。2026-09-02 曾被误当「压缩内容」关掉、静默切成纯 HTML，2026-09-21 发现并恢复（见「三、发现与处置」1）。
- **148 键全面复核（2026-09-28）**：28 个偏离上游默认值的键逐一确认与「保真优先」一致或无害，未发现相互矛盾 / 被旁路的配置；同轮修正 2 个削弱抓取完整度的键（见「三、发现与处置」3）。
- **1.28.0 配置面零变化**：`src/core/bg/config.js` 与 1.27.0 逐字节相同（SHA-256 均为 `7C02D720…`），键集与取值均不变，本次版本同步**不改任何键**。
- **版本核对表**（「配置面」指 `DEFAULT_CONFIG` 键集 / 默认值 / 改名表 / `upgrade()` 迁移是否变化；「无」表示该版只升级内置 core，**版本同步本身不需要改键**）：

| 版本 | 发布 | 内置 core | 配置面 | 结论 |
| --- | --- | --- | --- | --- |
| 1.24.0 | 2026-09-03 | — | 与上版逐键一致 | 仅 GitHub 5 键随全量导出回潮 |
| 1.24.3 | 2026-09-09 | 1.6.5 | +1 键 | 新增 `maxAppendedDataLength = 16361` |
| 1.25.0 / 1.26.0 | 2026-09-15 / 09-16 | — | 改名 + 新增 | 7 个 `loadDeferredImages*` → `loadDeferredContent*`，另增 5 键；143 → 148 键 |
| 1.26.1 | 2026-09-20 | 1.6.5 → 1.6.7 | 无 | `config.js` 未改 |
| 1.26.2 | 2026-09-21 | 1.6.7 → 1.6.9 | 无 | `config.js` 与 1.26.1 逐字节相同 |
| 1.26.3 | 2026-09-22 | 1.6.9 → 1.6.10 | 无 | 同上 |
| 1.26.4 | 2026-09-23 | 1.6.10 → 1.6.11 | 无 | 同上 |
| 1.26.5 | 2026-09-24 | 1.6.11 → 1.6.14 | 无 | 同上 |
| 1.27.0 | 2026-09-27 | 1.6.14 → 1.6.19 | 换键 2 处 | 移除失效 `compressCSS`、新增 `imageQuality`（HFA 取 `1`），仍 148 键 |
| 1.28.0 | 2026-10-02 | 1.6.19 → 1.6.22 | 无 | `config.js` 与 1.27.0 逐字节相同 |

> 逐版 core 改动中在 HFA 已启用路径上生效的部分汇总于「附录 B」；完整提交清单见对应 git 提交信息与上游发布说明。

## 一、结构总览（当前状态，148 键）

- profile：仅 `__Default_Settings__`
- 规则：1 条，`url = "*"` → `__Default_Settings__`；`autoSaveProfile = __Disabled_Settings__`（SingleFile 内置隐藏 profile，不出现在导出中，属正常）
- 顶层：`maxParallelWorkers = 12`、`processInForeground = false`
- 键类型分布：布尔 97（true 28 / false 69）、字符串 31（空 21 / 非空 10）、数字 14、数组 4、嵌套对象 1（`acceptHeaders`）、null 1（`customShortcut`）
- 键序：与导出格式一致，按码位升序排列（已校验）
- 键集演进：143 键（1.24.3）→ 1.26.0 合入改名与新增到 148 键；1.27.0 换键 2 处、数量不变；1.26.1 ~ 1.26.5 与 1.28.0 配置面零变化。键值丢失 0。
- 当前刻意调整的关键值（详见「三」）：`compressContent` / `selfExtractingArchive` / `extractDataFromPage = true`（自解压归档）；`networkTimeout = 0`、`passReferrerOnError = true`（抓取完整度）；`loadDeferredContentMinZoomFactor = 0.5`、`loadDeferredContentMaxIdleTime = 20000`（懒加载抓全）；`imageQuality = 1`（将来启用缩放时的保真上限）

## 二、键分类

### 高保真核心（刻意设置、生效中）

- 压缩关闭（页面自身）：`compressHTML = false`，保存页里的 HTML 保持可读格式。`compressCSS` 已于 1.27.0 从配置中移除（自 core 1.6.15 起即无效果）。注意 `compressContent` **不属于**这一类：它是保存格式总开关（见「归档 / 保存格式」）
- 屏蔽关闭：`blockScripts` / `blockStylesheets` / `blockImages` / `blockFonts` / `blockVideos` / `blockAudios` / `blockAlternativeImages` / `blockMixedContent` 等均为 `false`
- 清理关闭：`removeFrames` / `removeHiddenElements` / `removeUnusedStyles` / `removeUnusedFonts` / `removeAlternativeFonts` / `removeAlternativeImages` / `removeAlternativeMedias` / `removeNoScriptTags` / `removeSavedDate` 等均为 `false`
- 等待与超时：`loadDeferredContent = true`（`loadDeferredContentMaxIdleTime = 20000` ms，`loadDeferredContentDispatchScrollEvent = true`）；`networkTimeout = 0`（不设资源抓取超时，2026-09-28 由 `30000` 改回上游默认，见「三、发现与处置」3）；`loadDeferredContentMinZoomFactor = 0.5`（懒加载阶段缩放下限，2026-09-23 由上游默认 `0` 上调，见「三、发现与处置」2）
- 单资源上限检查关闭：`maxResourceSizeEnabled = false`
- 图片不缩放：`imageReductionFactor = 1`（上游默认，不做缩放）；1.27.0 新增的 `imageQuality = 1` 仅在 `imageReductionFactor > 1` 时才有意义，本配置下为惰性键；即便如此仍刻意取上限 `1`（而非上游默认 `0.8`），以免将来启用缩放时悄悄降质（见「五」）
- 抓取完整性：`passReferrerOnError = true`（跨源抓取补发页面 `Referer`，提高防盗链 / 需要 Referer 的资源成功率，2026-09-28 调整）；`networkTimeout = 0`（不设资源抓取超时，见上）
- 失效 / 惰性键（导出携带但当前无实际作用）：`blockAlternativeImages`（core 源码无引用）、`maxResourceSize = 15`（`maxResourceSizeEnabled = false`）、`maxSizeDuplicateImages = 1048576`（归档路径去重不受该键控制）、`autoSaveDelay = 3`（自动保存全关）；均为无害项，见「三、发现与处置」3
- 存档信息：`saveFavicon` / `saveOriginalURLs` / `resolveLinks` / `replaceBookmarkURL` / `insertSingleFileComment` / `insertMetaNoIndex` / `insertMetaCSP` / `insertCanonicalLink` 均为 `true`（`insertCanonicalLink` 在 1.24.x 中不可配置，由抓取入口 `src/core/content/content.js` 硬编码为 `true`；1.25.0 起提升为一等选项、默认 `true`，1.26.0 起选项页有复选框 —— 本配置行为前后一致）

### 服务族（全关留空，当前无实际作用）

- S3（`saveToS3 = false`，仅 `S3Domain` 为默认值）、WebDAV（`saveWithWebDAV = false`）、Dropbox、GDrive、REST Form API、MCP、Companion、woleet、raw page、剪贴板、分享、用户脚本、书签联动
- GitHub（`saveToGitHub = false`、`githubToken` / `githubUser` 为空；`githubBranch = "main"`、`githubRepository = "SingleFile-Archives"` 为默认填充）
- 说明：SingleFile 导出为全量格式，这些字段为扩展自带；未设置时扩展回退默认（关闭），不影响当前行为。其中 GitHub 5 键为**每次全量导出的固定成员**：手工删除后，下次导出仍会被扩展自动写回（2026-09-04 由 1.24.0 真实导出证实），故按导出原样保留

### 归档 / 保存格式（生效中）

- 保存格式：**自解压 ZIP（universal）** —— `compressContent = true`（总开关）、`selfExtractingArchive = true`、`extractDataFromPage = true`
- `preventAppendedData = false`、`maxAppendedDataLength = 16361`（ZIP 数据之后允许追加的数据预算）、`disableCompression = false`（ZIP 内用 deflate，无损）、`createRootDirectory = false`
- 生效性：core `single-file.js` 的压缩 / 归档阶段整段挂在 `compressContent` 之后（`if (options.compressContent) { … processors.compression.process(pageData, compressionOptions) … }`），资源辅助类也由它选择（`core/processor-helper.js` 的 `return options.compressContent ? getHelperClass(utilInstance) : getHelperInlineClass(utilInstance)`）。本配置为 `true` → 走归档路径，上述键全部参与行为；这也是 07-17 初始配置与 `d47c1ee`（08-20）的状态，`35b299b`（09-02）曾误关，2026-09-21 恢复（见「三、发现与处置」1）
- 归档路径下的去重：core `core/lib/processor-helper.js` 的 `groupDuplicateFonts` / `groupDuplicateImages` 不接受开关、按字节比较，字节相同的字体与图片在归档里各存一份并重写引用；这属于上游既定行为，不影响渲染结果。`groupDuplicateImages`（`true`，上限 `maxSizeDuplicateImages = 1048576`）与 `groupDuplicateStylesheets`（`false`）的门控只作用于**内联（HTML）路径**（`core/lib/processor-helper-inline.js`），本配置走归档路径时不参与
- 归档写入顺序确定化、同一页面两次保存产出相同归档（core 1.6.6 起），便于按哈希判断页面是否变化

### 界面与工作流（默认开启）

- 右键菜单 / 浏览器动作菜单 / 标签页菜单、进度条、系统主题、日志等
- `animateInfobar = true`：1.26.0 新增，控制保存页 infobar 的闪烁与扩散圈动画；仅影响阅读观感，与存档内容无关

### 内部 / 元数据

- `_migratedTemplateFormat = true`：SingleFile 配置模板迁移标记（正常）
- `_migratedDeferredContentOptions = true`：`loadDeferredContent*` 改名的迁移标记；置位后扩展不再重复执行该次改名（正常）

## 三、发现与处置

1. **保存格式被静默切成 HTML：已恢复自解压 ZIP (universal)（2026-09-21 发现并修复，改 3 键）**
   现象：选项页用**一个「格式」下拉**同时驱动 `compressContent = value.includes("zip")`、`selfExtractingArchive = …includes("self-extracting")`、`extractDataFromPage = value == "self-extracting-zip-universal"`（`src/ui/bg/ui-options.js` 的 `update()`）。**没有任何控件直接绑定 `compressContent` 或 `selfExtractingArchive`**，所以「`compressContent = false` + `selfExtractingArchive = true` + `extractDataFromPage = true`」这个组合**不可能由 UI 产生**。
   成因：`compressContent` 是**格式总开关**，而 `35b299b`（2026-09-02）把它当作「是否压缩内容」，连同 `compressHTML` / `compressCSS` 一起改成 `false` —— 格式自此从「自解压 ZIP (universal)」静默变成「纯 HTML」，`selfExtractingArchive` / `extractDataFromPage` / `preventAppendedData` / `maxAppendedDataLength` / `disableCompression` 全部失效。此后四轮上游核对（`468de78` / `783b206` / `f46767b` / `df71ed2`）只比对了键集与键值，没顺着 core 的门检查「这些键还生不生效」；`df71ed2` 甚至顺着「HTML 格式」把 `selfExtractingArchive` / `extractDataFromPage` 改成了 `false`。
   源码依据（core v1.6.7 / 扩展 v1.26.1）：压缩 / 归档阶段整段挂在 `compressContent` 之后（`single-file.js` 的 `if (options.compressContent) { … processors.compression.process(…) }`）；资源辅助类由它选择（`core/processor-helper.js` 的 `return options.compressContent ? getHelperClass(…) : getHelperInlineClass(…)`）；MIME 判定 `core/util.js`（`return !options.compressContent || options.selfExtractingArchive ? "text/html" : "application/zip"`）；扩展下载分支同样以它为准（`src/core/bg/downloads.js`）。
   处置（2026-09-21，恢复到 `d47c1ee`（2026-08-20）的可用状态）：`compressContent` `false` → `true`；`selfExtractingArchive` / `extractDataFromPage` 恢复 `true`；`disableCompression` 保持 `false`（ZIP 内 deflate，**无损**，解出资源与原字节一致）；`compressHTML` 保持 `false`（保存页 HTML 可读）；`preventAppendedData` / `maxAppendedDataLength` / `createRootDirectory` 保持原值。键集不变（148 键）。结果：格式 = 自解压 ZIP (universal)，产物仍是单个 `.html`（浏览器打开自动解出页面，ZIP 工具也可取出原始资源）。
   附带发现（不影响本配置）：`core/index.js` 的 `loadOptionsFromPage()` 在 FINALIZE 阶段**无开关**地把页面内嵌的 `data-single-file-options` JSON 覆盖回 `options` —— 再保存「曾被 SingleFile 保存过」的页面时，页面记录的同名字段会临时覆盖用户配置。本配置 `saveFilenameTemplateData = false`、模板不含 `{digest-sha-N}`、`openEditor = false`，该 JSON 不会被写入，此路径目前不会触发。

2. **高保真调优：懒加载缩放下限与空闲等待上调（2026-09-23，改 2 键）**
   - `loadDeferredContentMinZoomFactor`：`0` → `0.5`。懒加载开始时按 `zoomFactor = Math.max(Math.min(verticalZoomFactor, horizontalZoomFactor), minZoomFactor || 0)` 缩放页面（`core/processors/hooks/content/content-hooks-frames-web.js:296-298`），`0` 表示不设下限，长页面会被缩到极小（如 0.05），使依赖布局 / IntersectionObserver 的懒加载不触发；`0.5` 落在有效区间 (0, 1] 内。
   - `loadDeferredContentMaxIdleTime`：`10000` → `20000` ms，给迟到的动态内容更长等待。
   - 键集不变（148 键）。

3. **全面复核：148 键无相互矛盾，两处抓取完整度调整（2026-09-28，改 2 键）**
   逐键与上游 `v1.27.0` 的 `DEFAULT_CONFIG` 比对，挑出全部 28 个偏离默认值的键，再对每个键到 `v1.27.0` / core `v1.6.19` 源码确认实际作用；同时复查有没有「键值对却被别的开关旁路」的情况（`compressContent` 那类教训）。
   结论一（无矛盾）：28 个偏离项全部与「保真优先」一致或无害 —— 关闭屏蔽 / 清理 / 页面压缩，或打开归档格式与存档元信息，或在已关闭开关之后惰性（`maxResourceSize` / `maxSizeDuplicateImages` / `autoSaveDelay`），或与保真无关（`backgroundSave` 只决定产物下载上下文、`processInForeground` 只决定并发度），或纯观感 / 主题 / 菜单项。归档路径的 CSP 经 `core/lib/processor-helper.js` 的 `setMetaCSP` 确认 `script-src 'self' 'unsafe-inline' data: blob:`，与自解压脚本兼容（`insertMetaCSP = true` 不破坏归档）。另注：`blockAlternativeImages` 在 core 源码中无任何引用，是扩展导出携带的失效键，无副作用。
   结论二（2 处调整）：
   - `networkTimeout`：`30000` → `0`。源码依据：`core/util.js` 的 `getContent()` 只在「取响应」阶段与 `setTimeout(reject, networkTimeout)` 竞赛（`response.arrayBuffer()` 在该竞赛之外、本就不受超时约束）；超时即返回 `getFailedFetchResponse()`，除样式表有「回退到页面已加载规则」外，其余资源会保留为原始外链、不再内嵌 —— 30 秒截止会把慢响应资源留成存档外链。上游默认即 `0`，HFA 取 `0`，不做提前放弃。
   - `passReferrerOnError`：`false` → `true`。源码依据：`core/index.js` 中该键为真时把 `options.resourceReferrer` 设为页面目录 URL，`core/util.js` 的 `getContent()` 将其作为 `referrer` 传给抓取；扩展侧 `src/core/bg/requests.js` 的 `injectRefererHeader` 只对携带 SingleFile 请求 ID、且自身没有 `Referer` 的请求补发 —— 对要求 Referer / 防盗链的资源提高成功率，仅在原本无 Referer 时生效。
   - 键集不变（148 键）；`imageQuality` 一并由上游默认 `0.8` 定为 `1`（见「五」）。文档修正：README 原写「网络请求超时放宽到 30 秒」与上游默认 `0` 的事实相反（30 秒是收紧、不是放宽），已改为「不设超时」。

4. **配置漂移风险与「可达性」检查（跟踪项）**
   本文件是全量导出快照，本身不记录适用版本，长期不更新会落后于扩展能力，直接覆盖导出又会冲掉刻意设置。核对时三类变更都要查：**改名型**（旧键被静默删除，取值是否迁移由 `DEPRECATED_OPTION_NAMES` 决定，映射为 `null` 的会回落到新默认）、**删除型**（上游判定失效后从 `DEFAULT_CONFIG` 摘除，如 1.27.0 的 `compressCSS`）、**新增型**（按上游默认值合入）。每轮必须同时比对键集与键值。
   **可达性（2026-09-21 教训）**：键集 / 键值全对，不等于行为对 —— `compressContent` 被误改后键集仍与上游完全一致、四轮核对都没报警，但保存格式已从自解压归档变成纯 HTML（见「三、发现与处置」1）。故每轮还要对关键键（`compressContent`、`removeUnused*`、`loadDeferredContent*`、`maxResourceSizeEnabled`、`selfExtractingArchive` 等）确认「在 core 的门之后是否仍可达」，并留意 README 里把它描述成别的东西的痕迹。后续升级重复本流程。

5. **GitHub 服务族：按全量导出原样保留**
   `saveToGitHub = false`、`githubToken` / `githubUser` 为空（功能关闭）；`githubBranch = "main"`、`githubRepository = "SingleFile-Archives"` 为默认填充。2026-09-02 曾删除 5 键，2026-09-04 复核证实它们是 **SingleFile 每次全量导出的固定成员**（删除后下次导出会自动写回），为消除 diff 噪音按导出原样保留；功能仍关闭，不影响高保真行为。

6. **`maxAppendedDataLength` 语义（1.24.3 起）**：自解压归档 ZIP 数据之后允许追加的数据量预算；仅在 `compressContent = true` 且 `selfExtractingArchive = true`、`preventAppendedData = false` 时处于生效路径（2026-09-02 ~ 09-21 因 `compressContent` 被误关而不可达，恢复后重新生效）。上游 1.24.2 将默认预算由 65535 降至 16361（各 ZIP 读取器容忍尾部数据的回扫窗口差异极大，libarchive 仅 16383 字节）；1.24.3 才修好「导出配置里的该值仅影响自动保存、手动保存不生效」的透传缺失 —— **1.24.3 是首个让该键真正按配置生效的版本**。键值 `16361` 与上游默认一致；core 1.6.5 ~ 1.6.22 该常量一直为 16361。

7. **大量默认关闭项**：69 个 `false` 布尔、21 个空字符串等均为 SingleFile 全量导出自带状态，非刻意配置，无需处理。

## 四、跟进建议

1. README 已记录适用版本（2026-10-04 更新为 SingleFile 1.28.0）。
2. README 已固化「刻意设置的键」清单，与导出默认值区分（见「高保真策略要点」）。
3. SingleFile 升级后重新导出配置时，先与旧文件 diff，再合入新键 / 迁移项；注意**改名 / 删除 / 新增**三类变更，并对关键键确认可达性（见「三、发现与处置」4）。历次核对已覆盖 1.24.0 ~ 1.28.0。
4. 建议将**扩展本体**升级至 1.28.0（2026-10-02 发布）：配置面零变化，无需改键；同版内置 core 由 1.6.19 升到 1.6.22，core 侧在 HFA 生效路径上的收益见「附录 B」。
5. `loadDeferredContentMinZoomFactor`（1.25.0 新增，选项页暂无控件）：加载延迟内容时的页面缩放下限，有效区间 (0, 1]；已于 2026-09-23 设为 `0.5`（见「三、发现与处置」2）。
6. **保存格式 = 自解压 ZIP (universal)**：`compressContent` / `selfExtractingArchive` / `extractDataFromPage` 均为 `true`。`compressContent` 是格式总开关，改它等于换格式（`false` = 纯自包含 HTML，资源内联为 `data:` URI）；要临时换格式用选项页顶部「格式」下拉（HTML / ZIP / 自解压 ZIP / 自解压 ZIP universal），它会按下拉重写这三个键，手工只改 `selfExtractingArchive` 不生效（见「三、发现与处置」1）。

## 五、键语义补充（2026-09-23：待确证清单已清空；2026-09-28：补录 1.27.0 新增键）

上一版挂起的 4 个键已逐一到 `v1.26.4` 源码确证语义（1.26.5 的 `src/core/bg/config.js` 与 1.26.4 逐字节相同，语义与结论不变），清单清空。这 4 键在本配置**均为 `false`**，均未启用，不影响现有行为：

- `insertEmbeddedImage = false`：开启且 `compressContent = true` 时，保存前弹出文件选择框，把用户选定的图片作为「嵌入图片」写进保存页（`src/core/content/content.js:319`）。仅归档路径可用。
- `insertEmbeddedScreenshotImage = false`：开启且 `compressContent = true` 时，在资源初始化前抓取整页截图并作为「嵌入图片」写进保存页（`src/core/content/content.js:267`）。选项页中勾选 `insertEmbeddedImage` 会联动勾上它；两者在 `compressContent = false` 时均禁用。
- `moveStylesInHead = false`：开启时把 `<head>` 之外的 `<style>`（`body style` / `body ~ style`，且计算样式判为隐藏者）在抓取阶段标记（`core/helper.js:293`），收尾阶段移入 `<head>`（`core/index.js:1050`）；关闭时保持样式元素原位置。仅调整保存页内部结构，与资源保留无关。
- `saveFilenameTemplateData = false`：开启时把一个含 `saveUrl` / `saveDate` / 文件名模板等字段的 JSON `<script data-single-file-options>` 写进保存页，供再次保存 / 编辑器复用（`core/index.js:698`）；若 `openEditor = true` 或文件名模板含 `{digest-sha-N}`，会被强制置为 `true`（`core/index.js:188`）。本配置 `openEditor = false` 且模板不含 digest，故保持 `false`，该 JSON 不会写入。

1.27.0 新增 / 移除的键（2026-09-28 补录，均在 `v1.27.0` 源码确证）：

- `imageQuality = 1`（1.27.0 新增，HFA 取上限）：用 `imageReductionFactor` 缩放图片后重新编码 JPEG / WebP 时的质量（0~1，上游默认 `0.8`；取 1 对 WEBP 为无损、对 JPEG 为最高质量）。对 PNG 无效；`imageReductionFactor = 1` 时无效（`src/ui/bg/ui-options.js` 会禁用该输入框）。本配置 `imageReductionFactor = 1`，故当前为惰性键，但故意取 `1` 而非上游默认 `0.8`：HFA 以保真优先于体积，这样将来一旦启用缩放，也不会在重编码环节引入有损降质。选项页控件为数字输入框，取值经 `Math.min(Math.max(value, 0), 1)` 夹取，空值回落 `0.8`。
- `compressCSS`（1.27.0 移除）：自 core 1.6.15 起已无效果（其作用的 UglifyCSS 已从 vendored 依赖移除），被 `imageQuality` 取代。选项名仍被扩展接受、旧设置不报错，但新导出不再包含该键。本配置原值 `false`（关闭），删除后行为不变。

已由上游源码确证、不在此列的键：`maxAppendedDataLength`、`loadDeferredContentMinZoomFactor`、`readMaffMetadata`，以及 `compressContent` / `selfExtractingArchive` / `extractDataFromPage` / `preventAppendedData` / `disableCompression`（保存格式与归档路径语义）、`groupDuplicateImages` + `maxSizeDuplicateImages`（内联路径的去重门控与体积上限）。后续新增键再按同一流程补录。

## 附录 A：历史核对要点

**1.25.0 / 1.26.0 键改名映射（`DEPRECATED_OPTION_NAMES`）**

- 7 项改名：`loadDeferredImages` / `MaxIdleTime` / `BlockCookies` / `BlockStorage` / `KeepZoomLevel` / `BeforeFrames` → 对应 `loadDeferredContent*`，取值随迁移保留；`loadDeferredImagesDispatchScrollEvent` 映射为 `null`（旧键删除且**不迁移取值**），由新默认 `loadDeferredContentDispatchScrollEvent = true` 接管 —— 本配置旧值即 `true`，行为无变化。
- 同轮新增 5 键：`loadDeferredContentMinZoomFactor`、`insertCanonicalLink`、`readMaffMetadata`（1.25.0），`animateInfobar`（1.26.0），`_migratedDeferredContentOptions`（迁移标记）。
- 外部 API `CAPTURE_OPTION_NAMES` 同时保留新旧两套 `loadDeferred*` 名称；profile 导出只含新名，旧名会被迁移逻辑删除。
- 附带：1.26.0 新增「旧文件名替换表自动重置为新默认」迁移，仅在表内容与旧顺序完全相同时触发；本文件的表已是新默认顺序（`\\` 位于第 11 位而非旧表第 3 位），不会触发。

**1.24.0 升级核对（2026-09-04）**：与上版（137 键）逐键 diff，值变化 0、删除 0，rules 与顶层（`maxParallelWorkers` / `processInForeground`）一致，唯一差异为 GitHub 5 键随全量导出回潮 → 配置无需功能性调整。

## 附录 B：core 侧在 HFA 已启用路径上生效的行为变化（均不涉及配置键）

以下为各版内置 core 中、在 HFA 当前开关组合下**实际生效**的保真度 / 健壮性改动（裁剪类均挂在已关闭的 `removeUnused*` 等开关之后，不列）：

- **1.6.6**：归档写入顺序确定化，同一页面两次保存产出相同归档（便于按哈希判断页面是否变化）。
- **1.6.10**：自解压归档内同一内容的样式表按字节去重（`@import` 链自叶向上合并）；SingleFile 自生成图片（视频 poster、屏蔽视频图标、`<canvas>` 位图）改存文件而非内联 `data:` URI —— 只改变归档组织与体积，不影响渲染。归档解压到本地后不再因 `<link crossorigin>` 触发 `file://` CORS 而丢样式表。
- **1.6.11**：修「链接套链接」的非法嵌套页面（如 Substack 首页）保存失败（`HierarchyRequestError`）—— 嵌套复位改为正序（祖先在前）。
- **1.6.12**：非法嵌套修复扩展到 HTML 解析器会丢弃的元素（form 套 form、表格单元在表格外）与影子根，元素以注释对随存档保留、加载时重建；需修复的闭合影子根改存为 open。
- **1.6.13**：`<style>` / `<script>` 文本里 `/>` 只转义 `</`（修再次保存时反斜杠累积、Gemini 列表标记丢失，**不受 `compressHTML` 门控、对本配置生效**）；再次保存时 `<meta name=referrer>` 就地替换而非追加。
- **1.6.14**：infobar 闪烁改为独立覆盖层（纯观感，受 `animateInfobar = true` 影响）。
- **1.6.15 ~ 1.6.19**：影子根 `delegatesFocus` / 手动槽位分配修复、`<model>` 资源嵌入、无预期类型资源不再发 `Accept: undefined`、`<link>` 样式表抓取失败时回退页面已加载规则（Firefox 跨源可读）、`referrerpolicy` 与跨源 referrer 传递、`data-single-file-stylesheet` 残留属性清理；另有 `imageReductionFactor` 缩放链路修复（本配置 `= 1`，不生效）。
- **1.6.20 ~ 1.6.22**：归档读取补齐偏移 —— `processors/compression/compression.js` 的 `getContent()` 新增 `getPrependedDataLength()`，把从页面读到的 ZIP 数据按实际偏移补零对齐后再交给 zip.js，修自解压归档带前置数据时解压错位 / 校验失败；zip.js 升到 2.22.0。字体裁剪（`removeUnused*`）修复挂在已关闭开关之后，不可达。
