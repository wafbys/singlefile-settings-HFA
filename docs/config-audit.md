# HFA 配置审计报告

- 审计日期：2026-09-22（1.26.2 上游核对；此前 2026-09-21 1.26.1 上游核对、2026-09-17 1.26.0 上游核对、2026-09-14 1.24.3 上游核对、2026-09-04 升级复核、2026-09-02 初审）
- 审计对象：`singlefile-settings-HFA.json`
- 适用版本：SingleFile 1.26.2（配置基线为 2026-09-04 由用户重新导出的 1.24.0 快照）
- 审计基准：commit `6b916a0`（1.26.1 核对后状态，148 键）
- 当前状态：1.24.0 导出基线 + 1.24.3 / 1.25.0 / 1.26.0 的新增键与改名，148 键（1.26.1 / 1.26.2 均未改动配置面，键集与取值均无变化）
- 方法：静态审计 + 与 1.24.0 真实导出的键级 diff + 与上游 `v1.26.2` 源码 `src/core/bg/config.js` 的 `DEFAULT_CONFIG`（148 键）键级比对，并以脚本复现扩展 `upgrade()` 的迁移逻辑做等价性验证；另对 core `v1.6.7...v1.6.9` 逐文件核对改动是否可达 —— JSON 结构、内部一致性、字段语义归类；源码无法确证的语义仍标注为推断

## 结论摘要

- **结构健康**：JSON 语法有效；148 键与上游 `DEFAULT_CONFIG` 键集完全一致；无影响高保真目标的矛盾配置。发现并修复 1 处**保存格式被静默切换**的历史问题（`compressContent` 被误当作「压缩内容」关掉，见下条），无其他强制修改项。
- **1.26.2 上游核对（2026-09-22）**：上游本版只把内置的 single-file-core 由 1.6.7 升到 1.6.9（扩展自身只改了版本号，`src/` 下无源码改动）。`src/core/bg/config.js` 与 `v1.26.1` **逐字节相同**（SHA-256 一致），所以 `DEFAULT_CONFIG`（148 键）、改名表、`upgrade()` 迁移逻辑与 `LEGACY_FILENAME_REPLACED_CHARACTERS` 重置规则全部不变；本地 148 键与 1.26.2 的 `DEFAULT_CONFIG` 键集**完全一致**，**版本同步本身不需要改动任何键**。core 1.6.8 / 1.6.9 的三处改动都挂在 HFA 已关闭的开关之后（见「三、发现与处置」1），`DEFAULT_MAX_APPENDED_DATA_LENGTH` 仍为 16361。
- **1.26.1 上游核对（2026-09-21）**：上游本版只把内置的 single-file-core 由 1.6.5 升到 1.6.7（扩展自身只有俄语翻译与 CI 镜像两处改动）。`src/core/bg/config.js` 在 1.26.0 之后**一字未改**（该文件最后一次提交 `eb69c68` 早于 `v1.26.0` 标签），所以 `DEFAULT_CONFIG`（148 键）、`DEPRECATED_OPTION_NAMES` 改名表、`upgrade()` 迁移逻辑与 `LEGACY_FILENAME_REPLACED_CHARACTERS` 重置规则都与 1.26.0 相同。本地 148 键与 1.26.1 的 `DEFAULT_CONFIG` 键集**完全一致**（无缺失、无多余），**版本同步本身不要求改动任何键**；需要留意的是 core 1.6.6 / 1.6.7 带来的保存行为变化（见「三、发现与处置」1），以及同批核对查出的、与版本无关的保存格式问题（见下条与「三、发现与处置」2）。
- **保存格式曾被静默切换，已恢复（2026-09-21）**：`compressContent` 在 1.26.x 不是「是否压缩内容」，而是**保存格式总开关**（选项页的「格式」下拉同时驱动 `compressContent` / `selfExtractingArchive` / `extractDataFromPage`）。`35b299b`（2026-09-02）把它当作「压缩内容」连同 `compressHTML` / `compressCSS` 一起关掉，格式因此从「自解压 ZIP (universal)」静默变成「纯 HTML」，`selfExtractingArchive` / `extractDataFromPage` / `preventAppendedData` / `maxAppendedDataLength` / `disableCompression` 全部失效（而此后 `783b206` 等提交仍在按「归档生效」推断 `maxAppendedDataLength`）。2026-09-21 核对 1.26.1 时查明成因，已恢复 `compressContent = true`、`selfExtractingArchive = true`、`extractDataFromPage = true`（回到 `d47c1ee` 的可用状态）；键集不变，仍为 148 键。详见「三、发现与处置」2。
- **1.26.0 上游核对（2026-09-17）**：上游 `DEFAULT_CONFIG` 由 143 键增至 148 键 —— 7 个 `loadDeferredImages*` 键退出，12 个键名加入（6 个为改名对应项、1 个承接被删除的 `…DispatchScrollEvent`、5 个为全新选项）。已按上游默认值合入并采用新键名，现为 148 键，与 1.26.0 的 `DEFAULT_CONFIG` 键集**完全一致**；142 个既有键值全部原样保留，唯一未迁移取值的 `loadDeferredImagesDispatchScrollEvent` 回落的新默认 `true` 与其旧值相同，**行为无变化**（见「三、发现与处置」）。
- **1.24.3 上游核对（2026-09-14）**：本地 142 键与上游 `v1.24.3` 的 `DEFAULT_CONFIG`（143 键）逐键比对 —— 无键被上游删除，共享键默认值无任何变化；上游仅新增 1 键 `maxAppendedDataLength = 16361`，已按默认值合入。
- **1.24.0 升级核对（2026-09-04）**：上版 137 键与 1.24.0 真实导出逐键值一致，配置无需功能性调整；唯一差异是 GitHub 5 键随全量导出回潮，已按导出原样纳入。
- 大量 `false` / 空值均为 SingleFile 全量导出的默认状态，无需处理。

## 一、结构总览（当前状态，148 键 = 1.24.0 导出基线 + 上游 1.24.3 / 1.25.0 / 1.26.0 演进；1.26.1 / 1.26.2 均未引入配置变更，2026-09-21 恢复了被静默切换的保存格式）

- profile：仅 `__Default_Settings__`
- 规则：1 条，`url = "*"` → `__Default_Settings__`；`autoSaveProfile = __Disabled_Settings__`（SingleFile 内置隐藏 profile，不出现在导出中，属正常）
- 顶层：`maxParallelWorkers = 12`、`processInForeground = false`
- 键类型分布：布尔 98（true 27 / false 71）、字符串 31（空 21 / 非空 10）、数字 13、数组 4、嵌套对象 1（`acceptHeaders`）、null 1（`customShortcut`）
- 相对 1.24.3 核对状态（commit `783b206`，143 键）：7 个 `loadDeferredImages*` 键退出、12 个键名加入（8 个 `loadDeferredContent*` + 4 个其他），共 148 键；键值丢失 0（见「三、发现与处置」）
- 2026-09-21 值调整：`compressContent`：`false` → `true`（修正 `35b299b` 的静默格式切换，恢复自解压归档）；`selfExtractingArchive` / `extractDataFromPage`：恢复为 `true`（`df71ed2` 曾误按 HTML 格式改为 `false`）。键集不变，仍为 148 键；详见「三、发现与处置」2
- 键序：与导出格式一致，按码位升序排列（已校验）

## 二、键分类

### 高保真核心（刻意设置、生效中）

- 压缩关闭（页面自身）：`compressHTML` / `compressCSS` 均为 `false`，保存页里的 HTML / CSS 保持可读格式。注意 `compressContent` **不属于**这一类：它是保存格式总开关（见「归档 / 保存格式」）
- 屏蔽关闭：`blockScripts` / `blockStylesheets` / `blockImages` / `blockFonts` / `blockVideos` / `blockAudios` / `blockAlternativeImages` / `blockMixedContent` 等均为 `false`
- 清理关闭：`removeFrames` / `removeHiddenElements` / `removeUnusedStyles` / `removeUnusedFonts` / `removeAlternativeFonts` / `removeAlternativeImages` / `removeAlternativeMedias` / `removeNoScriptTags` / `removeSavedDate` 等均为 `false`
- 等待与超时：`loadDeferredContent = true`（`loadDeferredContentMaxIdleTime = 10000` ms，`loadDeferredContentDispatchScrollEvent = true`）；`networkTimeout = 30000` ms；`loadDeferredContentMinZoomFactor = 0`（不设缩放下限，与 1.24.0 行为一致）
- 单资源上限检查关闭：`maxResourceSizeEnabled = false`
- 存档信息：`saveFavicon` / `saveOriginalURLs` / `resolveLinks` / `replaceBookmarkURL` / `insertSingleFileComment` / `insertMetaNoIndex` / `insertMetaCSP` / `insertCanonicalLink` 均为 `true`（`insertCanonicalLink` 在 1.24.x 中不可配置，由抓取入口 `src/core/content/content.js` 硬编码为 `true`；1.25.0 起提升为一等选项、默认 `true`，1.26.0 起选项页有复选框 —— 本配置行为前后一致）

### 服务族（全关留空，当前无实际作用）

- S3（`saveToS3 = false`，仅 `S3Domain` 为默认值）、WebDAV（`saveWithWebDAV = false`）、Dropbox、GDrive、REST Form API、MCP、Companion、woleet、raw page、剪贴板、分享、用户脚本、书签联动
- GitHub（`saveToGitHub = false`、`githubToken` / `githubUser` 为空；`githubBranch = "main"`、`githubRepository = "SingleFile-Archives"` 为默认填充）
- 说明：SingleFile 导出为全量格式，这些字段为扩展自带；未设置时扩展回退默认（关闭），不影响当前行为。其中 GitHub 5 键为**每次全量导出的固定成员**：手工删除后，下次导出仍会被扩展自动写回（2026-09-04 由 1.24.0 真实导出证实），故按导出原样保留

### 归档 / 保存格式（生效中）

- 保存格式：**自解压 ZIP（universal）** —— `compressContent = true`（总开关）、`selfExtractingArchive = true`、`extractDataFromPage = true`
- `preventAppendedData = false`、`maxAppendedDataLength = 16361`（ZIP 数据之后允许追加的数据预算）、`disableCompression = false`（ZIP 内用 deflate，无损）、`createRootDirectory = false`
- 生效性：core `single-file.js` 的压缩 / 归档阶段整段挂在 `compressContent` 之后（`if (options.compressContent) { … processors.compression.process(pageData, compressionOptions) … }`），资源辅助类也由它选择（`core/processor-helper.js` 的 `return options.compressContent ? getHelperClass(utilInstance) : getHelperInlineClass(utilInstance)`）。本配置为 `true` → 走归档路径，上述键全部参与行为；这也是 07-17 初始配置与 `d47c1ee`（08-20）的状态，`35b299b`（09-02）曾误关，2026-09-21 恢复（见「三、发现与处置」2）
- 归档路径下的去重：core `core/lib/processor-helper.js` 的 `groupDuplicateFonts` / `groupDuplicateImages` 不接受开关、按字节比较，字节相同的字体与图片在归档里各存一份并重写引用；这属于上游既定行为，不影响渲染结果。`groupDuplicateImages`（`true`，上限 `maxSizeDuplicateImages = 1048576`）与 `groupDuplicateStylesheets`（`false`）的门控只作用于**内联（HTML）路径**（`core/lib/processor-helper-inline.js`），本配置走归档路径时不参与
- 归档写入顺序确定化、同一页面两次保存产出相同归档（core 1.6.6 起），便于按哈希判断页面是否变化

### 界面与工作流（默认开启）

- 右键菜单 / 浏览器动作菜单 / 标签页菜单、进度条、系统主题、日志等
- `animateInfobar = true`：1.26.0 新增，控制保存页 infobar 的闪烁与扩散圈动画；仅影响阅读观感，与存档内容无关

### 内部 / 元数据

- `_migratedTemplateFormat = true`：SingleFile 配置模板迁移标记（正常）
- `_migratedDeferredContentOptions = true`：`loadDeferredContent*` 改名的迁移标记；置位后扩展不再重复执行该次改名（正常）

## 三、发现与处置

1. **1.26.1 / 1.26.2 上游核对：配置面零变化，core 升级未绕过 HFA 已关闭的裁剪开关（2026-09-21 / 2026-09-22，无需改动）**
   比对方式：本地 148 键 vs 上游 `v1.26.1` 源码 `src/core/bg/config.js` 的 `DEFAULT_CONFIG`（148 键），脚本键级比对（结果：无缺失、无多余、键序仍为码位升序）；再用各配置相关文件的**最后提交时间**确认本版根本没碰配置面。
   证据链（配置面）：
   - `src/core/bg/config.js` 最后一次提交是 `eb69c68`（2026-09-16 21:35 UTC，"add an option to animate the infobar"），**早于** `v1.26.0` 标签（2026-09-16 23:35 UTC）→ 该文件在 1.26.0 与 1.26.1 中逐字节相同：`DEFAULT_CONFIG`（148 键）、`DEPRECATED_OPTION_NAMES`（7 项改名表）、`upgrade()` 的全部迁移与 `LEGACY_FILENAME_REPLACED_CHARACTERS` 重置规则都没有变化，1.26.0 那轮的结论对 1.26.1 直接成立。
   - `src/ui/common/filename-replacement.js`（`DEFAULT_REPLACED_CHARACTERS` / `DEFAULT_REPLACEMENT_CHARACTERS` 的来源）最后提交 `efd31be`（2026-09-16 15:45 UTC），同样早于 v1.26.0；1.26.1 的取值仍是 11 个形近字符 + `\x00-\x1f` + `\x7F`，`\\` 位于第 11 位，与本文件一致 → **重置迁移依旧不会触发**，两个数组无需改动。
   - `src/core/bg/external-messages.js`（`CAPTURE_OPTION_NAMES` 选项白名单）最后提交同为 `eb69c68` → 外部 API 可用名未变。
   - `v1.26.0...v1.26.1` 的 5 个提交只涉及俄语翻译（#1996）、CI 镜像（Ubuntu 26.04）、升级 `single-file-core` 依赖、版本号：`src/` 下无配置相关改动。
   - core 常量复核：`single-file-core` v1.6.7 的 `DEFAULT_MAX_APPENDED_DATA_LENGTH`（`processors/compression/compression-constants.js`）仍为 **16361**，`maxAppendedDataLength` 取值无需调整。
   证据链（core 行为，开关归属）：上游把一批「保存页更小」的变化列在 core 1.6.6 / 1.6.7 名下。逐一核对 v1.6.7 源码后确认：**凡属「裁剪 / 删除」的，都仍挂在 HFA 已关闭的选项后面，不会被无条件执行。**
   - `core/index.js` 的任务调度只在任务声明的 `option` 为真时执行（`if (!task.option || (…) || this.options[task.option]) return this.processor[task.action]()`）；`REPLACE_DATA_STAGE` 的顺序任务表为 `[{ option: "removeUnusedStyles", … }, { option: "removeAlternativeMedias", … }, { option: "removeUnusedFonts", … }]`。
   - 「浏览器永远不会选用的字体面不再保留」（字重不匹配、被后续 `unicode-range` 子集完全覆盖、仅在制表符 / 换行上相交、字符在页面出现但该字族从不绘制）：实现于 `core/lib/css-fonts-minifier.js`，唯一入口是 `removeUnusedFonts` 选项任务 → 本配置该键为 `false`，**该模块根本不执行**。判定「已用字体」的数据采集受同一开关约束：`core/helper.js` 中 `if (options.removeUnusedFonts && doc.defaultView) { getRootElementUsedFonts(…) }`，故 `usedFonts` / `usedFontsCharacters` 保持为空。
   - 「同字族两个字重的 `@font-face` 合并为带字重区间的单条规则」：`core/lib/processor-helper-inline.js` 的 `groupDuplicateFonts` 虽被无条件列入阶段表，但自身以 `if (options.usedFonts && options.usedFonts.length)` 为前提；上一条的采集被关闭时该数组为空 → 不会发生合并。
   - 「选择器匹配不到任何元素的规则、以及只被这些规则引用的图片一并移除」：实现于 `core/lib/css-rules-minifier.js`，入口是 `removeUnusedStyles` 选项任务 → 本配置该键为 `false`，不删规则；由于 `removeUnusedStyles` 是该阶段第一个顺序任务，不删规则也就意味着那些 URL 仍会被抓取。
   - 结论：1.26.1 的 core 升级对本配置的**存档内容无影响**，无需为它调整任何键。
   1.26.2 复核（2026-09-22）：`v1.26.1...v1.26.2` 在 `src/` 下无任何改动（仅版本号与 `package-lock.json`），`src/core/bg/config.js` 与 1.26.1 **逐字节相同**（两版 SHA-256 均为 `D45E954703C812C4D7C960F5D3ED409E8EBAB99D2203DCB99088C2A4753D94E1`），故上述配置面结论对 1.26.2 直接成立；本地 148 键与 1.26.2 的 `DEFAULT_CONFIG` 再次脚本比对，仍为无缺失、无多余。
   本版实质内容都在 core 1.6.8 / 1.6.9，逐文件确认可达性后均不影响本配置：
   - `modules/css-rules-minifier.js`（`@scope` 内以组合符开头的相对选择器改用 `:scope` 前缀匹配；并用 `CSS.supports("selector(…)")` 把浏览器解析不了的选择器整条排除出匹配与级联）：唯一入口是 `removeUnusedStyles` 选项任务（`core/index.js` 的 REPLACE_DATA_STAGE `{ option: "removeUnusedStyles", action: "removeUnusedStyles" }`）→ 本配置该键为 `false`，**该模块根本不执行**。
   - `core/index.js` 的 `getOnEventAttributeNames()` 增补对 SVG `animate` 元素的扫描（补上 `onbegin` / `onend` / `onrepeat` 三个处理器名）：该函数服务于 `removeEmbedScripts()`，而该动作挂在 `{ option: "blockScripts", action: "removeEmbedScripts" }` 之后（`core/index.js` 第 117 行）→ 本配置 `blockScripts = false`，**不执行剥离**。
   - `core/infobar.js` 的 `::after` 涟漪动画改为两种状态都常驻并加过渡延迟：纯观感，仅受 `animateInfobar`（本配置 `true`）影响，与存档内容无关。
   - core 常量复核：`single-file-core` v1.6.9 的 `DEFAULT_MAX_APPENDED_DATA_LENGTH`（`processors/compression/compression-constants.js`）仍为 **16361**，`maxAppendedDataLength` 无需调整。
   - 结论：1.26.2 对本配置的**存档内容无影响**，无需为它调整任何键。
   精确性说明：core 侧结论以 v1.6.7（1.26.1）与 v1.6.9（1.26.2）标签下的**端状态**源码为据，未与更早版本逐行 diff，故「某项行为是否自某版起才如此」不作断言；`v1.26.1` / `v1.26.2` 标签的 `package.json` 写的是 `"single-file-core": "^1.6.5"`（caret 区间，构建时按 lockfile 解析），「实际打包 1.6.7 / 1.6.9」取自各自的 `package-lock.json` 与上游发布说明，而非构建产物核对。

2. **保存格式被静默切成 HTML：已恢复自解压 ZIP (universal)（2026-09-21 发现并修复，改 3 键）**
   现象：1.26.1 的选项页用**一个「格式」下拉**同时驱动三个键 —— `compressContent = fileFormatSelectInput.value.includes("zip")`、`selfExtractingArchive = …includes("self-extracting")`、`extractDataFromPage = value == "self-extracting-zip-universal"`（`src/ui/bg/ui-options.js` 的 `update()`；下拉项见 `src/ui/pages/options.html`，文案见 `_locales/en/messages.json`：HTML / self-extracting ZIP / self-extracting ZIP (universal) / ZIP）。**没有任何控件直接绑定 `compressContent` 或 `selfExtractingArchive`**，所以「`compressContent = false` + `selfExtractingArchive = true` + `extractDataFromPage = true`」这个组合**不可能由 UI 产生**。
   成因（git 历史）：`compressContent` 是**保存格式总开关**，而 `35b299b`（2026-09-02「优化 HFA 高保真配置：关闭压缩以保留原始资源质量」）把它当作「是否压缩内容」，连同 `compressHTML` / `compressCSS` 一起改成 `false` —— 格式自此从「自解压 ZIP (universal)」静默变成「纯 HTML」，`selfExtractingArchive` / `extractDataFromPage` / `preventAppendedData` / `maxAppendedDataLength` / `disableCompression` 全部失效。此后四轮上游核对（`468de78` / `783b206` / `f46767b`，以及 `df71ed2`）都只比对了键集与键值，没有顺着 core 的门检查「这些键还生不生效」：`783b206` 的提交说明甚至仍写着「仅在 selfExtractingArchive=true 且 preventAppendedData=false 时生效（本配置正是如此）」；`df71ed2` 更是顺着「HTML 格式」把 `selfExtractingArchive` / `extractDataFromPage` 改成了 `false`，方向相反，本提交一并纠正。
   源码依据（core v1.6.7 / 扩展 v1.26.1）：压缩与归档阶段整段挂在 `compressContent` 之后 —— `single-file.js` 的 `if (options.compressContent) { … processors.compression.process(pageData, compressionOptions) … }`；资源辅助类也由它决定：`core/processor-helper.js` 的 `return options.compressContent ? getHelperClass(utilInstance) : getHelperInlineClass(utilInstance)`；MIME 判定 `core/util.js` 的 `return !options.compressContent || options.selfExtractingArchive ? "text/html" : "application/zip"`；扩展下载分支同样以它为准（`src/core/bg/downloads.js` 的 `if (message.compressContent) … else …`）。
   处置（2026-09-21，按仓库既定意图「自解压高保真」，恢复到 `d47c1ee`（2026-08-20）的可用状态）：
   - `compressContent`：`false` → `true` —— 总开关，恢复归档路径；`selfExtractingArchive`：恢复 `true`；`extractDataFromPage`：恢复 `true`（自解压 ZIP universal）。
   - `disableCompression` 保持 `false`：ZIP 内用 deflate，**无损**，解出的资源与原字节一致，仅影响体积。
   - `compressHTML` / `compressCSS` 保持 `false`：保存页里的 HTML / CSS 保持可读格式（09-02 起的既有决策，予以保留）。
   - `preventAppendedData` / `maxAppendedDataLength` / `createRootDirectory` 保持原值，键集仍与上游 `DEFAULT_CONFIG` 完全一致（148 键不变）；`maxAppendedDataLength = 16361` 随格式恢复重新处于生效路径。
   结果：保存格式 = 自解压 ZIP (universal)，产物仍是单个 `.html`（浏览器打开即自动解出页面，ZIP 工具也可直接取出原始资源）。
   附带发现（不影响本配置）：core `index.js` 的 `loadOptionsFromPage()` 在 FINALIZE 阶段**无开关**地（`Object.keys(options).forEach(option => this.options[option] = options[option])`）把页面内嵌的 `data-single-file-options` JSON 覆盖回 `options` —— 再保存一个「曾被 SingleFile 保存过」的页面时，页面里记录的同名字段会临时覆盖用户配置。本配置 `saveFilenameTemplateData = false`、文件名模板不含 `{digest-sha-N}`、`openEditor = false`，故该 JSON 不会被写入，这条路径目前不会触发。
   教训（并入第 8 条漂移风险）：全量导出快照里，「看起来像压缩开关」的键可能实际是**路径总开关**；每轮核对除了键集 / 键值，还要对关键键确认「在 core 的门之后是否仍可达」。
   精确性说明：core `index.js` 与 `helper.js` 已整文件核对（`extractDataFromPageTags` 的生产者未定位，可能在未读的 `core/util.js` / 扩展 `src/core/content/content.js`），但它唯一已知的读取点在归档处理器内，不影响上述结论。

3. **1.25.0 / 1.26.0 上游核对：`loadDeferredContent*` 改名 + 5 个新键（2026-09-17，已合入）**
   比对方式：本地 143 键 vs 上游 `v1.26.0` 源码 `src/core/bg/config.js` 的 `DEFAULT_CONFIG`（148 键）；并以脚本复现扩展 `upgrade()` 的迁移逻辑，验证「旧文件经扩展升级后」与「本文件」在键集与取值上等价。
   改名（7 项，由上游 `DEPRECATED_OPTION_NAMES` 定义）：`loadDeferredImages` / `MaxIdleTime` / `BlockCookies` / `BlockStorage` / `KeepZoomLevel` / `BeforeFrames` → 同名 `loadDeferredContent*`，取值随迁移保留；`loadDeferredImagesDispatchScrollEvent` 映射为 `null`，即**旧键直接删除且不迁移取值**，由新默认 `loadDeferredContentDispatchScrollEvent = true` 接管。本配置旧值即 `true`，与新默认一致，**行为无变化**（这是本次唯一需要人工确认的迁移点）。
   新键（5 个，均按上游默认值合入）：
   - `loadDeferredContentMinZoomFactor = 0`（1.25.0）：加载延迟内容时的页面缩放下限。源码确证有效区间为 (0, 1]，超出该区间一律按 0 处理，即**不设下限**。
   - `insertCanonicalLink = true`（1.25.0）：是否在保存页中写入页面的 canonical 链接。1.24.x 中该行为在抓取入口被硬编码开启、不可配置；提升为选项后默认仍为 `true`，故本配置行为不变。
   - `readMaffMetadata = false`（1.25.0）：源码确证为 core 动作 `readMAFFMetaData` 的开关，开启时会按页面 URL 抓取其所在目录的 `index.rdf`（MAFF 元数据）。默认关闭，保持不额外发起该请求。
   - `animateInfobar = true`（1.26.0）：保存页 infobar 的闪烁 / 扩散圈动画开关；仅影响阅读观感，与存档内容无关。
   - `_migratedDeferredContentOptions = true`：`loadDeferredContent*` 改名的迁移标记。
   精确性说明：上游 `src/core/bg/external-messages.js` 的 `CAPTURE_OPTION_NAMES`（外部扩展可设置的选项白名单）同时保留了新旧两套 `loadDeferred*` 名称，即**外部 API 仍可用旧名传参**；但 profile 导出中只会出现新名，旧名会被迁移逻辑删除。
   结果：148 键与 1.26.0 的 `DEFAULT_CONFIG` 键集完全一致（无缺失、无多余），143 个既有取值全部保留。
   附带校验一：上游把 `maxAppendedDataLength` 改为引用 core 常量 `DEFAULT_MAX_APPENDED_DATA_LENGTH`；经查 single-file-core 1.6.5 该常量仍为 **16361**，本文件取值无需调整。
   附带校验二：1.26.0 新增「旧文件名替换表自动重置为新默认」的迁移（比对 `LEGACY_FILENAME_REPLACED_CHARACTERS`），仅在表内容与旧顺序完全相同时触发。本文件的表已是新默认顺序（`\\` 位于第 11 位而非旧表第 3 位），**不会触发重置**，数组无需改动。
4. **1.24.3 上游核对：新增唯一键 `maxAppendedDataLength`（2026-09-14，已合入）**
   比对方式：当时的 142 键 vs 上游 `v1.24.3` 源码 `src/core/bg/config.js` 的 `DEFAULT_CONFIG`（143 键）。
   结果：本地全部键在上游仍然存在（删除 0）；共享键默认值变化 0（该文件在 `v1.24.0...v1.24.3` 范围 diff 为 +1 / -0 行）；新增唯一键 `maxAppendedDataLength = 16361`。
   语义：自解压归档 ZIP 数据之后允许追加的数据量预算；仅在 `compressContent = true`（归档路径）且 `selfExtractingArchive = true`、`preventAppendedData = false` 时处于生效路径。**注记（2026-09-21）**：该键在 2026-09-02 ~ 2026-09-21 之间因 `compressContent` 被误关而实际不可达；同轮核对恢复 `compressContent = true` 后重新生效（见第 2 条）。键值 `16361` 与上游默认一致。
   版本注记：上游 1.24.2 将默认预算由 65535 降至 16361，原因是各 ZIP 读取器容忍尾部数据的回扫窗口差异极大（libarchive 仅 16383 字节）；1.24.3 才修复「导出配置中设置该值仅影响自动保存、手动保存不生效」的透传缺失。**1.24.3 是首个让该键真正按配置生效的版本。**
5. **GitHub 服务族：2026-09-02 删除 → 2026-09-04 随 1.24.0 导出回潮（已按导出原样纳入）**
   现象：`saveToGitHub = false`、`githubToken` / `githubUser` 为空（功能关闭）；`githubBranch = "main"`、`githubRepository = "SingleFile-Archives"` 为 SingleFile 默认填充值。
   处置（2026-09-02，初判）：字段看似矛盾（填了仓库却无凭据）且功能未启用，删除 5 键。
   复核（2026-09-04）：上述 5 键是 SingleFile **每次全量导出的固定默认成员** —— 删除不影响行为（扩展回退关闭），但任何真实导出都会自动写回。为让仓库文件与「扩展导出现状」一致、消除反复 diff 噪音，按 1.24.0 导出原样纳入；功能仍关闭，不影响高保真行为。
6. **1.24.0 升级核对（2026-09-04）**
   将 1.24.0 真实导出与上版（137 键）逐键 diff：值变化 0、删除 0，rules 与顶层（`maxParallelWorkers` / `processInForeground`）一致，唯一差异为上述 GitHub 5 键 → **配置无需功能性调整**。v1.24.0 新增的选项页「Menus」菜单自定义功能，在未自定义菜单的导出中不产生新键；菜单属 UI 入口，与存档行为无关。
7. **大量默认关闭项**（71 个 `false` 布尔、21 个空字符串）
   判定：SingleFile 全量导出自带状态，非刻意配置，无需处理。
8. **配置漂移风险（跟踪项）**
   本文件是全量导出快照：SingleFile 升级会引入新键、改名与迁移标记；文件本身不记录适用版本，长期不更新会落后于扩展能力，直接覆盖导出又会冲掉刻意设置。1.24.0、1.24.3、1.26.0、1.26.1 各已完成一轮核对；**改名型变更是目前最大的漂移风险**（旧键被静默删除，取值不迁移），故每轮核对都必须同时检查键集与键值。1.26.1 说明并非每次升级都动配置面：该版只换了 core，配置键集零变化。
   **新增风险（2026-09-21）**：键集 / 键值全对，不等于行为对。`compressContent` 这类「路径总开关」被误改后，键集仍与上游完全一致、四轮核对都没报警，但保存格式已经从自解压归档变成纯 HTML（见第 2 条）。故每轮核对还需对少数关键键（`compressContent`、`removeUnused*`、`loadDeferredContent*`、`maxResourceSizeEnabled`、`selfExtractingArchive` 等）确认「在 core 的门之后是否仍可达」，并留意 README 里把它当作别的东西描述的痕迹。后续升级时重复本流程。

## 四、跟进建议

1. 在 README 记录适用的 SingleFile 版本号（2026-09-22 已更新为：SingleFile 1.26.2）。
2. 在 README 固化「刻意设置的键」清单，与导出默认值区分（已并入「高保真策略要点」）。
3. SingleFile 升级后重新导出配置时，先与旧文件 diff，再合入新键 / 迁移项（1.24.0、1.24.3、1.26.0、1.26.1、1.26.2 各已执行一轮，见「三、发现与处置」）。**改名型变更需额外注意**：取值迁移与否由扩展的迁移表决定，映射为 `null` 的键会被静默删除并回落到新默认。
4. 建议将**扩展本体**升级至 1.26.2（2026-09-21 发布；本文件已按 1.26.2 核对）。1.26.2 相对 1.26.1 只把内置 core 由 1.6.7 升到 1.6.9（扩展自身只有版本号），无论升级与否都不影响本配置的键集；core 侧值得知道的变化：`removeUnusedStyles` 清理被修得更准 —— `@scope` 内以组合符开头的相对选择器不再被误删、浏览器解析不了的选择器整条不再参与级联（两者都被本配置的 `removeUnusedStyles = false` 关掉，见「三、发现与处置」1），剥离脚本时会一并移除 SVG 动画元素的 `onbegin` / `onend` / `onrepeat`（本配置 `blockScripts = false`，不触发），以及 infobar 涟漪动画不再重复播放。1.26.1 相对 1.26.0 只换了内置 core（1.6.5 → 1.6.7）与俄语翻译；core 侧值得知道的变化：字体面裁剪与空规则移除被修正得更彻底（两者都被本配置的 `removeUnusedFonts` / `removeUnusedStyles = false` 关掉，见「三、发现与处置」1）、归档写入顺序确定化（同一页面两次保存产出相同归档，便于按哈希判断页面是否变化）、自解压页在发现多于一个归档候选时拒绝解压。1.26.0 相对 1.24.0 的保真度变化（保存页不再能提交表单或设置 base URI，`javascript:` URI 在所有属性与命名空间中被净化；资源字节优先于声明的 content type；响应头完整转发且按大小写不敏感查找；页面文档不再受单资源大小上限约束）依旧包含在内；1.24.1–1.24.3 的修复（复合 `@font-face` 规则全部保留、iframe 内 SVG 文档内容保留、sandboxed `srcdoc` iframe 按渲染结果保存、`srcset` 保存失败时不留空属性对、BMP / GIF87a 扩展名识别修正）同理。zip.js 升至 2.15.0 后产出的归档与 1.24.0 逐字节不同 —— 本配置走自解压归档路径，这些归档侧变化都会实际生效。
5. `loadDeferredContentMinZoomFactor` 是 1.25.0 新增、且**选项页暂无控件**的可调项：它为「加载延迟内容时的页面缩放」设下限（有效区间 (0, 1]，`0` = 不设下限）。若要避免长页面在抓取时被极度缩小、导致依赖布局的懒加载失效，可考虑设为 `0.5` 一类值；当前按上游默认 `0` 保留。
6. **保存格式 = 自解压 ZIP (universal)**（2026-09-21 恢复）：`compressContent = true` + `selfExtractingArchive = true` + `extractDataFromPage = true`。`compressContent` 是**格式总开关**，修改它等于换格式：`false` = 纯自包含 HTML（资源内联为 `data:` URI）。要临时换格式，直接用选项页顶部「格式」下拉（HTML / ZIP / 自解压 ZIP / 自解压 ZIP universal），它会按下拉重写这三个键；手工只改 `selfExtractingArchive` 不会生效（2026-09-02 的教训，见「三、发现与处置」2）。

## 五、语义待确证清单

以下键未逐一到上游源码 / 选项页确证语义，建议以 SingleFile 选项页对应控件说明为准；若与高保真目标冲突再行调整（已由上游源码确证的不在此列：`maxAppendedDataLength`、`loadDeferredContentMinZoomFactor`、`readMaffMetadata`，以及 `compressContent` / `selfExtractingArchive` / `extractDataFromPage` / `preventAppendedData` / `disableCompression`（保存格式与归档路径语义）、`groupDuplicateImages` + `maxSizeDuplicateImages`（内联路径的去重门控与体积上限））：

- `insertEmbeddedImage` / `insertEmbeddedScreenshotImage`
- `saveFilenameTemplateData`
- `moveStylesInHead`
