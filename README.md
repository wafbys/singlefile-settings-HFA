# SingleFile Settings — HFA

[SingleFile](https://github.com/gildas-lormeau/SingleFile)（把网页保存为单个自包含 HTML 的浏览器扩展）的配置导出，面向**高保真网页存档**：尽可能原样保留页面的资源与表现，而非追求文件体积最小化。

- **HFA**：本仓库对这份配置的代号，与提交历史中反复出现的「高保真」语义一致。
- 仓库只维护一个文件：`singlefile-settings-HFA.json`，所有调整通过 git 提交留痕。

> 配置体检与键分类口径见 [docs/config-audit.md](docs/config-audit.md)。

## 内容概览

- 单个 profile：`__Default_Settings__`（SingleFile 内置的默认 profile）
- 单条规则：`url = "*"` → 对所有页面应用该 profile
- 全局处理：`maxParallelWorkers = 12`、`processInForeground = false`（后台并行处理，不阻塞浏览器）

## 使用方式

1. 安装 SingleFile（Chrome / Edge / Firefox）。
2. 打开 SingleFile 的 Options（选项）页。
3. 通过选项页的备份/恢复（Backup/Restore）功能**导入**本文件；导入前建议先导出一份当前配置留作备份。
4. 导入后规则即对所有页面生效。点击扩展图标保存页面，产出文件名形如 `[HFA] 页面标题 (2026-09-02 15-30-00).html`。

> 选项页中的具体按钮名称随 SingleFile 版本略有差异，以实际界面为准。

## 高保真策略要点

**不裁不压 —— 保留原始资源质量**

- 压缩全部关闭：`compressHTML` / `compressCSS` / `compressContent` 均为 `false`，产出 HTML 保持原始可读格式
- 不屏蔽任何资源：`blockScripts` / `blockStylesheets` / `blockImages` 等均为 `false`
- 默认不做清理裁剪：`removeFrames` / `removeHiddenElements` / `removeUnusedStyles` / `removeUnusedFonts` 均为 `false`
- 单资源大小上限检查处于关闭状态（`maxResourceSizeEnabled = false`）

**把动态加载的内容也抓全**

- 懒加载资源开启（`loadDeferredImages = true`），滚动事件驱动，最多等待 10 秒页面空闲（`loadDeferredImagesMaxIdleTime = 10000` ms）
- 网络请求超时放宽到 30 秒（`networkTimeout = 30000` ms）

**存档元信息**

- 保存 favicon（`saveFavicon`）、保留原始 URL（`saveOriginalURLs`）、解析页面内链接（`resolveLinks`）
- 在保存的页面中插入 SingleFile 注释与 `noindex`、CSP 等 meta（`insertSingleFileComment` / `insertMetaNoIndex` / `insertMetaCSP`）

**命名约定**

- 文件名模板：
  `[HFA] %if-empty<{page-title}|No title> ({date-iso} {time-iso}).{filename-extension}`
- 以 `[HFA]` 前缀区分归档来源；页面无标题时回退为 `No title`；追加存档日期时间，避免同名覆盖。

## 维护与同步

本文件是 SingleFile 的**全量导出**格式：多数键（70 个关闭的布尔项、19 个空字符串等）是扩展导出自带的默认 / 关闭状态，并非刻意配置。刻意设置的键即上文「高保真策略要点」与「命名约定」所列；完整审计口径见 [docs/config-audit.md](docs/config-audit.md)。

- **适用 SingleFile 版本**：1.23.3（导出本配置时所用版本）
- 每次改动前先在 SingleFile 中导出现状留底，避免调坏配置后无法回退。
- 升级 SingleFile 后重新导出配置时，先与旧文件 diff，再决定合入哪些新键 / 迁移项。
- 每个改动配一条说明动机的 git 提交（参见提交历史），便于回溯高保真策略的演进。
