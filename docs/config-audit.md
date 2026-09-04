# HFA 配置审计报告

- 审计日期：2026-09-04（升级复核；初审 2026-09-02）
- 审计对象：`singlefile-settings-HFA.json`
- 适用版本：SingleFile 1.24.0（2026-09-04 由用户从已升级扩展重新导出并逐键核对）
- 审计基准：commit `bdca9ba`（复核前配置为 137 键）
- 当前状态：1.24.0 全量导出现状，142 键
- 方法：离线静态审计 + 与 1.24.0 真实导出的键级 diff —— JSON 结构、内部一致性、字段语义归类；未比对 SingleFile 官方 schema（离线环境），语义推断处均已标注

## 结论摘要

- **结构健康**：JSON 语法有效；无影响高保真目标的矛盾配置；无强制修改项。
- **1.24.0 升级核对（2026-09-04）**：上版 137 键与 1.24.0 真实导出逐键值一致，**配置无需任何功能性调整**；唯一差异是 GitHub 5 键随全量导出回潮（见「三、发现与处置」），已按导出原样纳入，现为 142 键。
- 大量 `false` / 空值均为 SingleFile 全量导出的默认状态，无需处理。

## 一、结构总览（当前状态，142 键 = 1.24.0 全量导出现状）

- profile：仅 `__Default_Settings__`
- 规则：1 条，`url = "*"` → `__Default_Settings__`；`autoSaveProfile = __Disabled_Settings__`（SingleFile 内置隐藏 profile，不出现在导出中，属正常）
- 顶层：`maxParallelWorkers = 12`、`processInForeground = false`
- 键类型分布：布尔 94（true 23 / false 71）、字符串 31（空 21 / 非空 10）、数字 11、数组 4、嵌套对象 1（`acceptHeaders`）、null 1（`customShortcut`）
- 相对上版（commit `bdca9ba`，137 键）：值变化 0、删除 0，仅新增 GitHub 服务族 5 键（见「三、发现与处置」）

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

1. **GitHub 服务族：2026-09-02 删除 → 2026-09-04 随 1.24.0 导出回潮（已按导出原样纳入）**
   现象：`saveToGitHub = false`、`githubToken` / `githubUser` 为空（功能关闭）；`githubBranch = "main"`、`githubRepository = "SingleFile-Archives"` 为 SingleFile 默认填充值。
   处置（2026-09-02，初判）：字段看似矛盾（填了仓库却无凭据）且功能未启用，删除 5 键。
   复核（2026-09-04）：上述 5 键是 SingleFile **每次全量导出的固定默认成员** —— 删除不影响行为（扩展回退关闭），但任何真实导出都会自动写回。为让仓库文件与「扩展导出现状」一致、消除反复 diff 噪音，按 1.24.0 导出原样纳入；功能仍关闭，不影响高保真行为。
2. **1.24.0 升级核对（2026-09-04，本次）**
   将 1.24.0 真实导出与上版（137 键）逐键 diff：值变化 0、删除 0，rules 与顶层（`maxParallelWorkers` / `processInForeground`）一致，唯一差异为上述 GitHub 5 键 → **配置无需功能性调整**。v1.24.0 新增的选项页「Menus」菜单自定义功能，在未自定义菜单的导出中不产生新键；菜单属 UI 入口，与存档行为无关。
3. **大量默认关闭项**（71 个 `false` 布尔、21 个空字符串）
   判定：SingleFile 全量导出自带状态，非刻意配置，无需处理。
4. **配置漂移风险（跟踪项）**
   本文件是全量导出快照：SingleFile 升级会引入新键与迁移标记；文件本身不记录适用版本，长期不更新会落后于扩展能力，直接覆盖导出又会冲掉刻意设置。本轮已按流程完成 1.24.0 核对；后续升级时重复本流程。

## 四、跟进建议

1. 在 README 记录适用的 SingleFile 版本号（2026-09-04 已更新为：SingleFile 1.24.0）。
2. 在 README 固化「刻意设置的键」清单，与导出默认值区分（已并入「高保真策略要点」）。
3. SingleFile 升级后重新导出配置时，先与旧文件 diff，再合入新键 / 迁移项（1.24.0 已执行一轮，见「三、发现与处置」）。

## 五、语义待确证清单

以下键在离线环境下无法核实准确语义，建议以 SingleFile 选项页对应控件说明为准；若与高保真目标冲突再行调整：

- `disableCompression`
- `selfExtractingArchive`
- `groupDuplicateImages` + `maxSizeDuplicateImages`
- `insertEmbeddedImage` / `insertEmbeddedScreenshotImage`
- `saveFilenameTemplateData`
- `moveStylesInHead`
