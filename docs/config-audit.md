# HFA 配置审计报告

- 审计日期：2026-09-14（上游版本核对；此前 2026-09-04 升级复核、2026-09-02 初审）
- 审计对象：`singlefile-settings-HFA.json`
- 适用版本：SingleFile 1.24.3（配置基线为 2026-09-04 由用户重新导出的 1.24.0 快照）
- 审计基准：commit `468de78`（1.24.0 导出现状，142 键）
- 当前状态：1.24.0 导出基线 + 1.24.3 新增键，143 键
- 方法：静态审计 + 与 1.24.0 真实导出的键级 diff + 与上游 `v1.24.3` 源码 `src/core/bg/config.js` 的 `DEFAULT_CONFIG` 键级比对 —— JSON 结构、内部一致性、字段语义归类；源码无法确证的语义仍标注为推断

## 结论摘要

- **结构健康**：JSON 语法有效；无影响高保真目标的矛盾配置；无强制修改项。
- **1.24.3 上游核对（2026-09-14）**：本地 142 键与上游 `v1.24.3` 的 `DEFAULT_CONFIG`（143 键）逐键比对 —— **无键被上游删除，共享键默认值无任何变化**；上游仅新增 1 键 `maxAppendedDataLength = 16361`，已按默认值合入，现为 143 键（见「三、发现与处置」）。
- **1.24.0 升级核对（2026-09-04）**：上版 137 键与 1.24.0 真实导出逐键值一致，配置无需功能性调整；唯一差异是 GitHub 5 键随全量导出回潮，已按导出原样纳入。
- 大量 `false` / 空值均为 SingleFile 全量导出的默认状态，无需处理。

## 一、结构总览（当前状态，143 键 = 1.24.0 导出基线 + 1.24.3 新增键）

- profile：仅 `__Default_Settings__`
- 规则：1 条，`url = "*"` → `__Default_Settings__`；`autoSaveProfile = __Disabled_Settings__`（SingleFile 内置隐藏 profile，不出现在导出中，属正常）
- 顶层：`maxParallelWorkers = 12`、`processInForeground = false`
- 键类型分布：布尔 94（true 23 / false 71）、字符串 31（空 21 / 非空 10）、数字 12、数组 4、嵌套对象 1（`acceptHeaders`）、null 1（`customShortcut`）
- 相对 1.24.0 导出现状（commit `468de78`，142 键）：新增 1 键、删除 0、共享键默认值变化 0（`maxAppendedDataLength`，见「三、发现与处置」）

## 二、键分类

### 高保真核心（刻意设置、生效中）

- 压缩关闭：`compressHTML` / `compressCSS` / `compressContent` 均为 `false`
- 屏蔽关闭：`blockScripts` / `blockStylesheets` / `blockImages` / `blockFonts` / `blockVideos` / `blockAudios` / `blockAlternativeImages` / `blockMixedContent` 等均为 `false`
- 清理关闭：`removeFrames` / `removeHiddenElements` / `removeUnusedStyles` / `removeUnusedFonts` / `removeAlternativeFonts` / `removeAlternativeImages` / `removeAlternativeMedias` / `removeNoScriptTags` / `removeSavedDate` 等均为 `false`
- 等待与超时：`loadDeferredImages = true`（`loadDeferredImagesMaxIdleTime = 10000` ms，`loadDeferredImagesDispatchScrollEvent = true`）；`networkTimeout = 30000` ms
- 单资源上限检查关闭：`maxResourceSizeEnabled = false`
- 存档信息：`saveFavicon` / `saveOriginalURLs` / `resolveLinks` / `replaceBookmarkURL` / `insertSingleFileComment` / `insertMetaNoIndex` / `insertMetaCSP` 均为 `true`

### 服务族（全关留空，当前无实际作用）

- S3（`saveToS3 = false`，仅 `S3Domain` 为默认值）、WebDAV（`saveWithWebDAV = false`）、Dropbox、GDrive、REST Form API、MCP、Companion、woleet、raw page、剪贴板、分享、用户脚本、书签联动
- GitHub（`saveToGitHub = false`、`githubToken` / `githubUser` 为空；`githubBranch = "main"`、`githubRepository = "SingleFile-Archives"` 为默认填充）
- 说明：SingleFile 导出为全量格式，这些字段为扩展自带；未设置时扩展回退默认（关闭），不影响当前行为。其中 GitHub 5 键为**每次全量导出的固定成员**：手工删除后，下次导出仍会被扩展自动写回（2026-09-04 由 1.24.0 真实导出证实），故按导出原样保留

### 界面与工作流（默认开启）

- 右键菜单 / 浏览器动作菜单 / 标签页菜单、进度条、系统主题、日志等

### 内部 / 元数据

- `_migratedTemplateFormat = true`：SingleFile 配置模板迁移标记（正常）

## 三、发现与处置

1. **1.24.3 上游核对：新增唯一键 `maxAppendedDataLength`（2026-09-14，已合入）**
   比对方式：本地 142 键 vs 上游 `v1.24.3` 源码 `src/core/bg/config.js` 的 `DEFAULT_CONFIG`（143 键）。
   结果：本地全部键在上游**仍然存在**（删除 0）；共享键默认值 **变化 0**（该文件在 `v1.24.0...v1.24.3` 范围 diff 为 +1 / -0 行）；上游新增唯一键 `maxAppendedDataLength = 16361`。
   语义（源码可证）：自解压归档 ZIP 数据之后允许追加的数据量预算；仅在 `selfExtractingArchive = true` 且 `preventAppendedData = false` 时处于生效路径 —— 本配置二者恰为 `true` / `false`，故该键在 HFA 下确实生效。
   处置：按上游默认值原样写入 `"maxAppendedDataLength": 16361`，与「仓库文件应与扩展导出现状一致」的既定口径相符。该值非刻意调优项，属上游默认。
   版本注记：该键由 **v1.24.3** 引入（v1.24.1 / v1.24.2 均无）。上游 1.24.2 将默认预算由 65535 降至 16361，原因是各 ZIP 读取器容忍尾部数据的回扫窗口差异极大（libarchive 仅 16383 字节）；1.24.3 才修复「导出配置中设置该值仅影响自动保存、手动保存不生效」的透传缺失。**1.24.3 是首个让该键真正按配置生效的版本。**
2. **GitHub 服务族：2026-09-02 删除 → 2026-09-04 随 1.24.0 导出回潮（已按导出原样纳入）**
   现象：`saveToGitHub = false`、`githubToken` / `githubUser` 为空（功能关闭）；`githubBranch = "main"`、`githubRepository = "SingleFile-Archives"` 为 SingleFile 默认填充值。
   处置（2026-09-02，初判）：字段看似矛盾（填了仓库却无凭据）且功能未启用，删除 5 键。
   复核（2026-09-04）：上述 5 键是 SingleFile **每次全量导出的固定默认成员** —— 删除不影响行为（扩展回退关闭），但任何真实导出都会自动写回。为让仓库文件与「扩展导出现状」一致、消除反复 diff 噪音，按 1.24.0 导出原样纳入；功能仍关闭，不影响高保真行为。
3. **1.24.0 升级核对（2026-09-04）**
   将 1.24.0 真实导出与上版（137 键）逐键 diff：值变化 0、删除 0，rules 与顶层（`maxParallelWorkers` / `processInForeground`）一致，唯一差异为上述 GitHub 5 键 → **配置无需功能性调整**。v1.24.0 新增的选项页「Menus」菜单自定义功能，在未自定义菜单的导出中不产生新键；菜单属 UI 入口，与存档行为无关。
4. **大量默认关闭项**（71 个 `false` 布尔、21 个空字符串）
   判定：SingleFile 全量导出自带状态，非刻意配置，无需处理。
5. **配置漂移风险（跟踪项）**
   本文件是全量导出快照：SingleFile 升级会引入新键与迁移标记；文件本身不记录适用版本，长期不更新会落后于扩展能力，直接覆盖导出又会冲掉刻意设置。1.24.0、1.24.3 各已完成一轮核对；后续升级时重复本流程。

## 四、跟进建议

1. 在 README 记录适用的 SingleFile 版本号（2026-09-14 已更新为：SingleFile 1.24.3）。
2. 在 README 固化「刻意设置的键」清单，与导出默认值区分（已并入「高保真策略要点」）。
3. SingleFile 升级后重新导出配置时，先与旧文件 diff，再合入新键 / 迁移项（1.24.0、1.24.3 各已执行一轮，见「三、发现与处置」）。
4. 建议将**扩展本体**升级至 1.24.3：1.24.1–1.24.3 未引入其他配置键，但含多项直接影响归档保真度的修复 —— 复合 `@font-face` 规则全部保留（修复图标字体缺字变豆腐块）、iframe 内 SVG 文档内容保留、sandboxed `srcdoc` iframe 按渲染结果保存、`srcset` 保存失败时不再残留空属性对、BMP / GIF87a 扩展名识别修正。注意 zip.js 升至 2.13.1 后，产出的归档与 1.24.0 逐字节不同。

## 五、语义待确证清单

以下键本轮未逐一到上游源码 / 选项页确证语义，建议以 SingleFile 选项页对应控件说明为准；若与高保真目标冲突再行调整（2026-09-14 已由上游源码确证的 `maxAppendedDataLength` 不在此列）：

- `disableCompression`
- `selfExtractingArchive`
- `groupDuplicateImages` + `maxSizeDuplicateImages`
- `insertEmbeddedImage` / `insertEmbeddedScreenshotImage`
- `saveFilenameTemplateData`
- `moveStylesInHead`
