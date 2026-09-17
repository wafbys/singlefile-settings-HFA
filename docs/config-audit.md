# HFA 配置审计报告

- 审计日期：2026-09-17（上游版本核对；此前 2026-09-14 上游核对、2026-09-04 升级复核、2026-09-02 初审）
- 审计对象：`singlefile-settings-HFA.json`
- 适用版本：SingleFile 1.26.0（配置基线为 2026-09-04 由用户重新导出的 1.24.0 快照）
- 审计基准：commit `783b206`（1.24.3 核对后状态，143 键）
- 当前状态：1.24.0 导出基线 + 1.24.3 / 1.25.0 / 1.26.0 的新增键与改名，148 键
- 方法：静态审计 + 与 1.24.0 真实导出的键级 diff + 与上游 `v1.26.0` 源码 `src/core/bg/config.js` 的 `DEFAULT_CONFIG`（148 键）键级比对，并以脚本复现扩展 `upgrade()` 的迁移逻辑做等价性验证 —— JSON 结构、内部一致性、字段语义归类；源码无法确证的语义仍标注为推断

## 结论摘要

- **结构健康**：JSON 语法有效；无影响高保真目标的矛盾配置；无强制修改项。
- **1.26.0 上游核对（2026-09-17）**：上游 `DEFAULT_CONFIG` 由 143 键增至 148 键 —— 7 个 `loadDeferredImages*` 键退出，12 个键名加入（6 个为改名对应项、1 个承接被删除的 `…DispatchScrollEvent`、5 个为全新选项）。已按上游默认值合入并采用新键名，现为 148 键，与 1.26.0 的 `DEFAULT_CONFIG` 键集**完全一致**；142 个既有键值全部原样保留，唯一未迁移取值的 `loadDeferredImagesDispatchScrollEvent` 回落的新默认 `true` 与其旧值相同，**行为无变化**（见「三、发现与处置」）。
- **1.24.3 上游核对（2026-09-14）**：本地 142 键与上游 `v1.24.3` 的 `DEFAULT_CONFIG`（143 键）逐键比对 —— 无键被上游删除，共享键默认值无任何变化；上游仅新增 1 键 `maxAppendedDataLength = 16361`，已按默认值合入。
- **1.24.0 升级核对（2026-09-04）**：上版 137 键与 1.24.0 真实导出逐键值一致，配置无需功能性调整；唯一差异是 GitHub 5 键随全量导出回潮，已按导出原样纳入。
- 大量 `false` / 空值均为 SingleFile 全量导出的默认状态，无需处理。

## 一、结构总览（当前状态，148 键 = 1.24.0 导出基线 + 上游 1.24.3 / 1.25.0 / 1.26.0 演进）

- profile：仅 `__Default_Settings__`
- 规则：1 条，`url = "*"` → `__Default_Settings__`；`autoSaveProfile = __Disabled_Settings__`（SingleFile 内置隐藏 profile，不出现在导出中，属正常）
- 顶层：`maxParallelWorkers = 12`、`processInForeground = false`
- 键类型分布：布尔 98（true 26 / false 72）、字符串 31（空 21 / 非空 10）、数字 13、数组 4、嵌套对象 1（`acceptHeaders`）、null 1（`customShortcut`）
- 相对 1.24.3 核对状态（commit `783b206`，143 键）：7 个 `loadDeferredImages*` 键退出、12 个键名加入（8 个 `loadDeferredContent*` + 4 个其他），共 148 键；键值丢失 0（见「三、发现与处置」）
- 键序：与导出格式一致，按码位升序排列（已校验）

## 二、键分类

### 高保真核心（刻意设置、生效中）

- 压缩关闭：`compressHTML` / `compressCSS` / `compressContent` 均为 `false`
- 屏蔽关闭：`blockScripts` / `blockStylesheets` / `blockImages` / `blockFonts` / `blockVideos` / `blockAudios` / `blockAlternativeImages` / `blockMixedContent` 等均为 `false`
- 清理关闭：`removeFrames` / `removeHiddenElements` / `removeUnusedStyles` / `removeUnusedFonts` / `removeAlternativeFonts` / `removeAlternativeImages` / `removeAlternativeMedias` / `removeNoScriptTags` / `removeSavedDate` 等均为 `false`
- 等待与超时：`loadDeferredContent = true`（`loadDeferredContentMaxIdleTime = 10000` ms，`loadDeferredContentDispatchScrollEvent = true`）；`networkTimeout = 30000` ms；`loadDeferredContentMinZoomFactor = 0`（不设缩放下限，与 1.24.0 行为一致）
- 单资源上限检查关闭：`maxResourceSizeEnabled = false`
- 存档信息：`saveFavicon` / `saveOriginalURLs` / `resolveLinks` / `replaceBookmarkURL` / `insertSingleFileComment` / `insertMetaNoIndex` / `insertMetaCSP` / `insertCanonicalLink` 均为 `true`（`insertCanonicalLink` 在 1.24.x 中不可配置，由抓取入口 `src/core/content/content.js` 硬编码为 `true`；1.25.0 起提升为一等选项、默认 `true`，1.26.0 起选项页有复选框 —— 本配置行为前后一致）

### 服务族（全关留空，当前无实际作用）

- S3（`saveToS3 = false`，仅 `S3Domain` 为默认值）、WebDAV（`saveWithWebDAV = false`）、Dropbox、GDrive、REST Form API、MCP、Companion、woleet、raw page、剪贴板、分享、用户脚本、书签联动
- GitHub（`saveToGitHub = false`、`githubToken` / `githubUser` 为空；`githubBranch = "main"`、`githubRepository = "SingleFile-Archives"` 为默认填充）
- 说明：SingleFile 导出为全量格式，这些字段为扩展自带；未设置时扩展回退默认（关闭），不影响当前行为。其中 GitHub 5 键为**每次全量导出的固定成员**：手工删除后，下次导出仍会被扩展自动写回（2026-09-04 由 1.24.0 真实导出证实），故按导出原样保留

### 界面与工作流（默认开启）

- 右键菜单 / 浏览器动作菜单 / 标签页菜单、进度条、系统主题、日志等
- `animateInfobar = true`：1.26.0 新增，控制保存页 infobar 的闪烁与扩散圈动画；仅影响阅读观感，与存档内容无关

### 内部 / 元数据

- `_migratedTemplateFormat = true`：SingleFile 配置模板迁移标记（正常）
- `_migratedDeferredContentOptions = true`：`loadDeferredContent*` 改名的迁移标记；置位后扩展不再重复执行该次改名（正常）

## 三、发现与处置

1. **1.25.0 / 1.26.0 上游核对：`loadDeferredContent*` 改名 + 5 个新键（2026-09-17，已合入）**
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
2. **1.24.3 上游核对：新增唯一键 `maxAppendedDataLength`（2026-09-14，已合入）**
   比对方式：当时的 142 键 vs 上游 `v1.24.3` 源码 `src/core/bg/config.js` 的 `DEFAULT_CONFIG`（143 键）。
   结果：本地全部键在上游仍然存在（删除 0）；共享键默认值变化 0（该文件在 `v1.24.0...v1.24.3` 范围 diff 为 +1 / -0 行）；新增唯一键 `maxAppendedDataLength = 16361`。
   语义：自解压归档 ZIP 数据之后允许追加的数据量预算；仅在 `selfExtractingArchive = true` 且 `preventAppendedData = false` 时处于生效路径 —— 本配置二者恰为 `true` / `false`，故该键在 HFA 下确实生效。
   版本注记：上游 1.24.2 将默认预算由 65535 降至 16361，原因是各 ZIP 读取器容忍尾部数据的回扫窗口差异极大（libarchive 仅 16383 字节）；1.24.3 才修复「导出配置中设置该值仅影响自动保存、手动保存不生效」的透传缺失。**1.24.3 是首个让该键真正按配置生效的版本。**
3. **GitHub 服务族：2026-09-02 删除 → 2026-09-04 随 1.24.0 导出回潮（已按导出原样纳入）**
   现象：`saveToGitHub = false`、`githubToken` / `githubUser` 为空（功能关闭）；`githubBranch = "main"`、`githubRepository = "SingleFile-Archives"` 为 SingleFile 默认填充值。
   处置（2026-09-02，初判）：字段看似矛盾（填了仓库却无凭据）且功能未启用，删除 5 键。
   复核（2026-09-04）：上述 5 键是 SingleFile **每次全量导出的固定默认成员** —— 删除不影响行为（扩展回退关闭），但任何真实导出都会自动写回。为让仓库文件与「扩展导出现状」一致、消除反复 diff 噪音，按 1.24.0 导出原样纳入；功能仍关闭，不影响高保真行为。
4. **1.24.0 升级核对（2026-09-04）**
   将 1.24.0 真实导出与上版（137 键）逐键 diff：值变化 0、删除 0，rules 与顶层（`maxParallelWorkers` / `processInForeground`）一致，唯一差异为上述 GitHub 5 键 → **配置无需功能性调整**。v1.24.0 新增的选项页「Menus」菜单自定义功能，在未自定义菜单的导出中不产生新键；菜单属 UI 入口，与存档行为无关。
5. **大量默认关闭项**（72 个 `false` 布尔、21 个空字符串）
   判定：SingleFile 全量导出自带状态，非刻意配置，无需处理。
6. **配置漂移风险（跟踪项）**
   本文件是全量导出快照：SingleFile 升级会引入新键、改名与迁移标记；文件本身不记录适用版本，长期不更新会落后于扩展能力，直接覆盖导出又会冲掉刻意设置。1.24.0、1.24.3、1.26.0 各已完成一轮核对；**改名型变更是目前最大的漂移风险**（旧键被静默删除，取值不迁移），故每轮核对都必须同时检查键集与键值。后续升级时重复本流程。

## 四、跟进建议

1. 在 README 记录适用的 SingleFile 版本号（2026-09-17 已更新为：SingleFile 1.26.0）。
2. 在 README 固化「刻意设置的键」清单，与导出默认值区分（已并入「高保真策略要点」）。
3. SingleFile 升级后重新导出配置时，先与旧文件 diff，再合入新键 / 迁移项（1.24.0、1.24.3、1.26.0 各已执行一轮，见「三、发现与处置」）。**改名型变更需额外注意**：取值迁移与否由扩展的迁移表决定，映射为 `null` 的键会被静默删除并回落到新默认。
4. 建议将**扩展本体**升级至 1.26.0。对归档保真度有实质影响的变化：保存的页面不再能提交表单或设置 base URI，`javascript:` URI 在所有属性与命名空间中被净化；资源字节优先于其声明的 content type；响应头完整转发且按大小写不敏感查找；页面文档不再受单资源大小上限约束。1.24.1–1.24.3 的修复（复合 `@font-face` 规则全部保留、iframe 内 SVG 文档内容保留、sandboxed `srcdoc` iframe 按渲染结果保存、`srcset` 保存失败时不留空属性对、BMP / GIF87a 扩展名识别修正）同样包含在内。zip.js 已升至 2.15.0，产出的归档与 1.24.0 逐字节不同。
5. `loadDeferredContentMinZoomFactor` 是 1.25.0 新增、且**选项页暂无控件**的可调项：它为「加载延迟内容时的页面缩放」设下限（有效区间 (0, 1]，`0` = 不设下限）。若要避免长页面在抓取时被极度缩小、导致依赖布局的懒加载失效，可考虑设为 `0.5` 一类值；当前按上游默认 `0` 保留。

## 五、语义待确证清单

以下键未逐一到上游源码 / 选项页确证语义，建议以 SingleFile 选项页对应控件说明为准；若与高保真目标冲突再行调整（已由上游源码确证的不在此列：`maxAppendedDataLength`、`loadDeferredContentMinZoomFactor`、`readMaffMetadata`）：

- `disableCompression`
- `selfExtractingArchive`
- `groupDuplicateImages` + `maxSizeDuplicateImages`
- `insertEmbeddedImage` / `insertEmbeddedScreenshotImage`
- `saveFilenameTemplateData`
- `moveStylesInHead`
