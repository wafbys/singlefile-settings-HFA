# HFA 配置审计报告

- 审计日期：2026-09-02
- 审计对象：`singlefile-settings-HFA.json`
- 适用版本：SingleFile 1.23.3（用户提供）
- 审计基准：commit `35b299b`（审计时配置为 142 键）
- 当前状态：GitHub 服务族移除后为 137 键
- 方法：离线静态审计 —— JSON 结构、内部一致性、字段语义归类；未比对 SingleFile 官方 schema（离线环境），语义推断处均已标注

## 结论摘要

- **结构健康**：JSON 语法有效；无影响高保真目标的矛盾配置；无强制修改项。
- **发现 1 项配置残留**（GitHub 服务族），经确认非本意、无用，已从配置中删除。
- 其余大量 `false` / 空值均为 SingleFile 全量导出的默认状态，无需处理。

## 一、结构总览（处置后当前状态，137 键）

- profile：仅 `__Default_Settings__`
- 规则：1 条，`url = "*"` → `__Default_Settings__`；`autoSaveProfile = __Disabled_Settings__`（SingleFile 内置隐藏 profile，不出现在导出中，属正常）
- 顶层：`maxParallelWorkers = 12`、`processInForeground = false`
- 键类型分布：布尔 93（true 23 / false 70）、字符串 27（空 19 / 非空 8）、数字 11、数组 4、嵌套对象 1（`acceptHeaders`）、null 1（`customShortcut`）
- 审计基准（commit `35b299b`）为 142 键；差额 5 = 已删除的 GitHub 服务族（见「三、发现与处置」）

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
- 说明：SingleFile 导出为全量格式，这些字段为扩展自带；未设置时扩展回退默认（关闭），不影响当前行为

### 界面与工作流（默认开启）

- 右键菜单 / 浏览器动作菜单 / 标签页菜单、进度条、系统主题、日志等

### 内部 / 元数据

- `_migratedTemplateFormat = true`：SingleFile 配置模板迁移标记（正常）

## 三、发现与处置

1. **GitHub 服务族残留（已处置）**
   现象：`githubRepository = "SingleFile-Archives"`、`githubBranch = "main"` 已填，但 `githubUser` / `githubToken` 为空、`saveToGitHub = false` —— 字段自相矛盾。
   判定：功能未启用；经确认非配置本意，无用。
   处置（2026-09-02）：删除 `githubBranch` / `githubRepository` / `githubToken` / `githubUser` / `saveToGitHub` 共 5 键。删除后扩展回退默认（关闭），行为不变；JSON 语法已复验。
2. **大量默认关闭项**（70 个 `false` 布尔、19 个空字符串）
   判定：SingleFile 全量导出自带状态，非刻意配置，无需处理。
3. **配置漂移风险（跟踪项）**
   本文件是全量导出快照：SingleFile 升级会引入新键与迁移标记；文件本身不记录适用版本，长期不更新会落后于扩展能力，直接覆盖导出又会冲掉刻意设置。跟进建议见下节。

## 四、跟进建议

1. 在 README 记录适用的 SingleFile 版本号（2026-09-02 已填入：SingleFile 1.23.3）。
2. 在 README 固化「刻意设置的键」清单，与导出默认值区分（已并入「高保真策略要点」）。
3. SingleFile 升级后重新导出配置时，先与旧文件 diff，再合入新键 / 迁移项。

## 五、语义待确证清单

以下键在离线环境下无法核实准确语义，建议以 SingleFile 选项页对应控件说明为准；若与高保真目标冲突再行调整：

- `disableCompression`
- `selfExtractingArchive`
- `groupDuplicateImages` + `maxSizeDuplicateImages`
- `insertEmbeddedImage` / `insertEmbeddedScreenshotImage`
- `saveFilenameTemplateData`
- `moveStylesInHead`
