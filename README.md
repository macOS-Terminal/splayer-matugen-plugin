# Matugen配色 · SPlayer-Next 插件

让 [SPlayer-Next](https://splayer-next.imsyy.top/) 的配色实时跟随 [Matugen](https://github.com/InioX/matugen)，
并在「设置 → 外观设置 → 主题色来源」下拉框中新增 **Matugen** 选项。

换壁纸 → Matugen 重新配色 → 播放器秒级跟随，全程无需重启。

```
Matugen 换色  →  写出种子色文件  →  插件读取  →  注入渲染进程  →  播放器整套 M3 调色板即时更新
```

- **单文件插件**，本地导入即用
- **不改启动参数、不重启播放器**，实时生效
- **多平台**：Linux 已实机验证；macOS / FreeBSD / Windows 已完成代码级适配
- **零网络请求**：只读本地 Matugen 输出与播放器状态

## 效果

「主题色来源」下拉框中新增 **Matugen**，与原生选项并列：

```
默认主题色 / 自定义主色 / 跟随封面 / 背景 / Matugen / 无主题色
```

选中后，Matugen 每次生成新配色，播放器会在一秒内跟着变；切回其它来源则立即停止跟随。

## 安装

### 1. 导入插件

SPlayer-Next → **设置 → 插件管理 → 本地导入** → 选择 `matugen-next.js`。

安装后自动启用，名称显示为 **Matugen配色**，作者 **macOS终端**。

### 2. 配置 Matugen

**通常无需手动操作。** 插件首次启动会自动完成两步：

1. 写入模板 `~/.config/matugen/templates/splayer-next.json`：

   ```json
   {
     "seed": "{{colors.source_color.default.hex}}",
     "primary": "{{colors.primary.default.hex}}",
     "mode": "{{mode}}",
     "generatedBy": "splayer-next/matugen-next"
   }
   ```

2. 向 `~/.config/matugen/config.toml` 追加（原文件备份为 `config.toml.bak-matugen-next`）：

   ```toml
   [templates.splayer-next]
   input_path = '/home/<user>/.config/matugen/templates/splayer-next.json'
   output_path = '/home/<user>/.cache/matugen/splayer-next.json'
   ```

此后任何一次 Matugen 调用（换壁纸、`matugen image ...`、`matugen color ...`）都会顺带写出种子色文件，插件据此换色。

> 不想让插件改配置文件？在插件配置里关闭「自动配置 Matugen 模板」，再手动把上面的模板与段落加入自己的配置即可。
> 插件只关心种子色文件的位置，不依赖 Matugen 是否由它配置。

### 3. 选择「Matugen」

**设置 → 外观设置 → 主题色来源 → Matugen**。完成。

## 插件设置

| 设置项 | 默认值 | 说明 |
| --- | --- | --- |
| 启用 Matugen 配色同步 | 开 | 关闭后完全停止读取配色，也不改动播放器配色 |
| 自动配置 Matugen 模板 | 开 | 自动向 `config.toml` 追加模板，写出种子色文件 |
| Matugen 配置文件路径 | 留空（自动） | 自定义 `config.toml` 位置 |
| 种子色文件路径 | 留空（自动） | 自定义模板输出路径，文件中任意 `#rrggbb` 都会作为种子色 |
| 配色检测间隔（毫秒） | 1000 | 轮询种子色文件的间隔 |
| 换色时发送系统通知 | 关 | 仅用于确认插件是否生效 |

## 跨平台路径

| 平台 | Matugen 配置 | Matugen 缓存 / 种子色文件 |
| --- | --- | --- |
| Linux / FreeBSD | `$XDG_CONFIG_HOME/matugen/config.toml` | `$XDG_CACHE_HOME/matugen/splayer-next.json` |
| macOS | `~/Library/Application Support/matugen/config.toml` | `~/Library/Caches/matugen/splayer-next.json` |
| Windows | `%APPDATA%\matugen\config.toml` | `%LOCALAPPDATA%\matugen\splayer-next.json` |

路径中的 `~` 会自动展开；自定义位置请填绝对路径。以上规则对齐 Rust `dirs` crate（Matugen 自身用它决定配置与缓存位置）。

**状态说明**：Linux 已实机验证跑通全流程；macOS / FreeBSD / Windows 已完成代码级适配
（路径解析、资源定位、原子写入、进程信息读取均按各平台差异分别实现），属**理论支持**，未在真机运行。

### 各平台实际处理的差异

不是简单套用同一套路径，而是分别处理：

| 差异点 | Windows | macOS | Linux / FreeBSD |
| --- | --- | --- | --- |
| 配置目录 | `%APPDATA%` | `~/Library/Application Support` | `$XDG_CONFIG_HOME` |
| 缓存目录 | `%LOCALAPPDATA%` | `~/Library/Caches` | `$XDG_CACHE_HOME` |
| 应用资源目录 | `<安装目录>\resources` | `<App>.app/Contents/Resources` | `<安装目录>/resources` |
| 绝对路径判断 | 盘符路径（不能用 `/` 开头判断） | POSIX | POSIX |
| 状态文件写入 | `rename` 可能 `EPERM`，需退化重试 | POSIX 原子替换 | POSIX 原子替换 |
| 父进程命令行 | 无轻量方案，跳过 | `ps -o command=` | `/proc/<pid>/cmdline` |

## 工作原理

### 只传种子色，不传整套调色板

SPlayer-Next 内部用 `@material/material-color-utilities`，从**一个种子色**展开整套 M3 调色板，
再写成 `--s-primary`、`--s-surface` 等 CSS 变量挂到 `<html>` 上。
因此插件只需把 `#rrggbb` 交给主题 store 的 `customColor`，其余全部由播放器自行计算——
这也是能保持与原生配色完全一致的原因。

### 如何把颜色送进渲染进程

插件运行在 SPlayer-Next 的共享插件宿主中（`host.worker.js` 的 `node:vm` 上下文），
该进程**没有 DOM**，而应用默认启动时并未开启 `--remote-debugging-port`。插件分三步解决：

1. **取得宿主能力**：沙箱中的 `Buffer` 是宿主真实对象，
   `Buffer.constructor('return globalThis')()` 可拿到 utility 进程的全局对象，
   进而通过 `process.getBuiltinModule('fs' | 'path' | 'os' | 'http')` 读写文件。
2. **借用 Node inspector**：`process._debugProcess(主进程 pid)` 让 Electron 主进程在
   `127.0.0.1:9229` 开启 inspector，用 WebSocket 发送一次 `Runtime.evaluate`，
   在主进程中 `require('electron')` 取得 `BrowserWindow`，
   再用 `webContents.executeJavaScript()` 把代码送入渲染进程。
   完成后延时 `process._debugEnd()` 关闭通道（同步关闭会掐断本次回复）。
3. **保留常驻 agent**：首次注入时在主进程装一个定时器，轮询本插件的 `state.json`；
   rev 变化即重新注入。同时监听 `browser-window-created` / `did-finish-load`，
   渲染进程重载后自动补回。**后续换色不再需要开关 inspector。**

注入前会先用 `process.pid` 核对 inspector 目标确实是本应用主进程——
`9229~9250` 区间可能被其它 Node/Electron 程序占用，核对不上就改走其它通道，绝不向陌生进程注入代码。

### 主题来源如何扩展

不新增真实的 `source` 取值（那会破坏播放器内部的枚举判断），而是在 store 上定义虚拟属性
`matugenSource`，再把设置项 `themeSource` 的绑定指向它：

- 选中 Matugen → 内部实际为 `source = 'custom'` + `customColor = 种子色` + `imageBackground.matugenActive = true`
- 选中其它值 → 清除标志，透传真实 `source`

这样播放器自身的 `watch → applyThemeToDOM` 链路完全照常工作，不依赖任何未导出的内部函数；
`matugenActive` 是 `imageBackground` 上的响应式布尔量，会随主题 store 一起持久化，
因此**重启后仍记得你选的是 Matugen**。

下拉框选项来自 `out/renderer/assets/searchSettings-*.js` 导出的 `settingsSchema` 数组。
插件在 asar 中定位该 chunk 文件名，在渲染进程里动态 `import` 后向 `options` 插入一项。
设置面板为懒挂载，插件在首次打开设置面板之前即完成注入，所以打开即可见。

## 兜底种子来源

若种子色文件不存在（说明 Matugen 模板尚未配置好），插件会退到 DMS（DankMaterialShell）
记录的 `~/.cache/dms/matugen-seeds.json` 中最后一次种子色，保证开箱即用。

只要主种子色文件**存在**，就以它为准——哪怕此刻内容暂时为空或正在被原子替换，
也不会退到兜底文件（否则两处颜色不一致会让播放器配色来回跳）。

## 已验证行为

在 SPlayer-Next `v1.2.1-nightly.1079`（Electron 44 / Chromium 152，**Linux**）上实测：

- 下拉框出现「Matugen」，选中后配色立即切换（`--s-primary`：`255 218 214` → `225 224 249`）
- `matugen color hex "#795548"` 后 3 秒内自动换色，**无需任何交互**
- 连续换色继续跟随（`#607d8b` → `--s-primary`：`208 230 242`）
- 关闭「启用 Matugen 配色同步」后不再跟随，重新开启立即补上最新颜色
- 重启应用后仍记得「Matugen」，自动应用最新种子色
- **不传 `--remote-debugging-port` 也能工作**（插件自行开启 inspector 通道）
- 渲染进程重载后自动重新注入
- 应用关闭期间 Matugen 换色，下次启动自动跟上
- 种子色文件损坏 / 为空 / 被原子替换时保持上一次颜色，不乱跳
- 纯文本种子文件（含 `#rrggbb`）也能识别
- Matugen 配置注入幂等，重复启动不会写出重复模板段
- 浅色 / 深色模式，以及默认 / 自定义 / 跟随封面 / 无主题色各选项互不干扰
- 9229 被无关进程占用时能识别（pid 核对），不会误注入他人进程

macOS / FreeBSD / Windows：仅完成代码级适配与路径模拟验证，未经真机运行。

## 已知限制

- 依赖插件沙箱可访问宿主 `Buffer`。SPlayer-Next 官方文档明确说明插件沙箱不是安全边界，
  当前版本（`apiLevel 5`）可用；若未来宿主收紧沙箱，配色同步会失效，但不影响播放器本身。
- 「Matugen」选择依赖 `imageBackground.matugenActive` 标志持久化。清除播放器数据或更换
  userData 目录后，需要重新选择一次。
- 注入用到的 `searchSettings-*.js` 文件名做了前缀匹配与内容特征双重兜底；
  若未来前端构建产物结构大改，可能找不到该模块（此时配色同步仍正常，仅下拉框无 Matugen 选项）。
- inspector 通道固定占用 `127.0.0.1:9229`（Node inspector 的默认端口，被占用时不会自动顺延）。
  因此若 9229 已被无关进程占用，插件会退回「应用带 `--remote-debugging-port` 启动」的 CDP 模式；
  若 9229 被应用自身的 `--remote-debugging-port=9229` 占用，建议改用其它调试端口（如 `9223`）。
- 只读 Matugen 输出文件与播放器状态，除本机回环地址上的调试端口外**不发起任何网络请求**。

## 卸载

设置 → 插件管理 → 卸载。

插件写入 `matugen/config.toml` 的模板段不会自动移除（避免误删用户配置），
可手动删除 `[templates.splayer-next]` 段与 `~/.config/matugen/templates/splayer-next.json`；
备份位于 `config.toml.bak-matugen-next`。

运行时文件在 `~/.cache/matugen/splayer-next/`（`state.json`、`ack.json`、`debug.log`），可直接删除。

## 调试

- 插件日志：`{userData}/app-data/logs/<date>.log`，检索 `[Matugen]`
- 插件自身调试日志：`~/.cache/matugen/splayer-next/debug.log`
- 手动触发一次 Matugen 验证：

  ```bash
  matugen color hex "#5a61b1"
  cat ~/.cache/matugen/splayer-next.json   # 应看到 {"seed":"#5a61b1", ...}
  ```

- 手动验证种子色读取逻辑（无需 Matugen）：

  ```bash
  echo '{"seed":"#22cc88"}' > ~/.cache/matugen/splayer-next.json
  ```

## 更新日志

### 1.1.0
- 修复 inspector 注入异常时未关闭通道、导致 `9229` 被永久占用、后续注入全部失败的问题
- 修复 `installAgent` 中的未定义变量导致插件在部分机器上直接失效
- 修正对 Node inspector 端口的错误假设（被占用时不会自动顺延）
- 插件头部新增 `@homepage`，指向本仓库

### 1.0.1
- 完善 macOS / Windows 路径与资源定位（`.app/Contents/Resources`、盘符绝对路径）
- Windows 状态文件写入增加 `rename` 失败的退化重试
- 插件更名为「Matugen配色」，下拉框选项改为「Matugen」

### 1.0.0
- 首个版本：实时跟随 Matugen 配色，主题色来源下拉框新增 Matugen 选项

## 许可

MIT
