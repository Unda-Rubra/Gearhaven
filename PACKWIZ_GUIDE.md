# Gearhaven（齿轮星都）整合包协作指南

本仓库是整合包的**源文件仓库**，不是可以直接双击启动的 Minecraft 实例。当前目标环境由 `pack.toml` 定义：

- Minecraft：`1.21.1`
- 模组加载器：NeoForge `21.1.248`
- Packwiz 格式：`packwiz:1.1.0`

本文面向第一次接触 Packwiz 和 GitHub Actions 的协作者。命令默认先给出 **Windows PowerShell** 写法；macOS、Linux 只需按[命令前缀表](#各系统如何运行-packwiz)替换前缀。

## 先记住这五条

1. **所有 Packwiz 命令都在包含 `pack.toml` 的仓库根目录运行。**
2. 从 Modrinth 或 CurseForge 添加模组时，优先使用 `packwiz mr add` / `packwiz cf add`，不要先手动下载 JAR。
3. 手动添加、删除或修改配置、脚本、资源包等整合包文件后，必须运行 `packwiz refresh`。
4. `index.toml` 及 `pack.toml` 中的 `[index]` 哈希由 Packwiz 生成，不要手改。
5. GitHub Actions 只负责导出安装包，**不能证明游戏可以正常启动**；推送前仍要本地启动并测试改动。

## Packwiz 是什么

[Packwiz](https://packwiz.infra.link/) 是一个用命令行维护 Minecraft 整合包的工具。它把整合包描述成适合 Git 管理的文本清单：

- `pack.toml`：整合包名称、版本、Minecraft 与加载器版本等总配置。
- `mods/*.pw.toml`：一个外部模组对应一份下载元数据，记录来源、目标文件名、哈希和更新信息。
- `index.toml`：所有应进入整合包的文件及其哈希索引。
- `config/`、`kubejs/`、`resourcepacks/` 等：最终复制进游戏实例的内部文件。

执行 `mr add` 或 `cf add` 后，`mods/` 中通常只出现一个 `.pw.toml`，而不会出现对应 JAR。这是正常现象：启动器导入导出包时，再按元数据下载真正的模组文件。这样可以减少 Git 仓库体积，也能保留来源和版本信息。

Packwiz **不是** Minecraft 启动器，也不会替代实际进游戏测试。

> 本仓库已经初始化完成。协作者不要再次运行 `packwiz init`。只有负责人创建全新的空仓库时，才应在空目录运行 `packwiz init`，按提示填写整合包信息并选择 Minecraft 与加载器版本。

### 新建一个完全空的整合包（仅负责人）

如果以后另开一个空仓库，先把匹配系统的 Packwiz 放进空目录或加入 `PATH`，再运行一次初始化：

```powershell
# Windows PowerShell
.\packwiz.exe init
```

```bash
# macOS / Linux：使用各自的文件名
./packwiz-macos init
# 或 ./packwiz-linux init
```

按提示填写整合包名称、作者、整合包版本、Minecraft 版本和模组加载器。团队项目应选择明确版本，不要未经测试直接选择“latest”。完成后应看到 `pack.toml` 和 `index.toml`，再建立 `mods/`、`config/` 等目录及忽略规则。`--reinit` 会重建已有包配置，本仓库禁止使用。

## 准备仓库和终端

1. 从 GitHub 仓库页面的 **Code** 菜单复制地址并克隆；也可以使用 GitHub Desktop。
2. 每次工作前先同步目标分支，避免基于旧清单添加模组。
3. 在仓库根目录打开终端。这里应该同时能看到 `pack.toml` 和对应系统的 Packwiz 可执行文件。

Windows 11 可在资源管理器打开仓库目录后，点击地址栏输入 `powershell` 并回车。先验证位置和程序：

```powershell
Get-Item .\pack.toml
.\packwiz.exe --help
.\packwiz.exe list
```

如果 `pack.toml` 不存在，说明当前目录不对；不要继续执行添加、删除或刷新命令。

## 仓库与游戏实例必须分离

> [!CAUTION]
> **绝对不要把本 Git 仓库直接克隆、移动或整体链接到某个启动器实例的 `.minecraft` 目录，也不要把仓库本身设置为游戏目录。**

二者职责不同：

- Git 仓库是整合包的**源代码和构建输入**，包含 `pack.toml`、`index.toml`、`.pw.toml` 与 Packwiz 可执行文件。
- 启动器实例是**运行和测试环境**，包含启动器下载的真实 JAR，以及游戏生成的日志、缓存、存档、崩溃报告和个人设置。

把仓库当作 `.minecraft` 会产生以下问题：

- 游戏每次启动都会改写或生成大量文件，污染 Git 工作区；
- 一次 `packwiz refresh` 可能把未被忽略的日志、缓存、存档甚至个人数据写进索引；
- `.pw.toml` 不是可加载的模组 JAR，所以仓库目录本身并不是完整实例；
- 很难区分团队基线配置与本机运行时改动，也容易误提交整个 `mods/` 或 `saves/`。

推荐把三类目录并列分开：

```text
工作目录/
├── Gearhaven/                    # Git 仓库；只保存源文件
├── Gearhaven-builds/             # 本地导出的 .mrpack / CurseForge ZIP
└── 启动器实例/Gearhaven-Dev/
    └── .minecraft/               # 实际运行、调试和生成配置的目录
```

文件只在确认测试结果后，按明确的相对路径从测试实例**选择性复制回仓库**；不要同步整个 `.minecraft`。

## 各系统如何运行 Packwiz

仓库已经附带三个 Packwiz x86-64 可执行文件：

| 系统 | 首次准备 | 后续命令前缀 |
| --- | --- | --- |
| Windows 10/11 x64（PowerShell） | 无需安装 | `.\packwiz.exe` |
| macOS Intel | `chmod +x ./packwiz-macos` | `./packwiz-macos` |
| macOS Apple Silicon | 同上；本仓库文件为 x86-64，需要 Rosetta 2 | `./packwiz-macos` |
| Linux x86-64 | `chmod +x ./packwiz-linux` | `./packwiz-linux` |

macOS：

```bash
cd /path/to/Gearhaven
chmod +x ./packwiz-macos
./packwiz-macos --help
./packwiz-macos list
```

Linux：

```bash
cd /path/to/Gearhaven
chmod +x ./packwiz-linux
./packwiz-linux --help
./packwiz-linux list
```

Windows 使用 CMD 时可写 `packwiz.exe list`；本文统一使用 PowerShell 的 `.\packwiz.exe list`。PowerShell 中省略 `.\` 通常会得到“无法识别命令”，因为 PowerShell 默认不从当前目录查找程序。

本仓库附带的 macOS、Linux 和 Windows 文件均为 x86-64。若使用 ARM Linux 或其他不兼容平台，请从 [Packwiz 安装说明](https://packwiz.infra.link/installation/)取得匹配架构的版本，或使用 Go 构建；不要把个人下载的替代二进制提交进整合包。

macOS 若提示无法验证开发者，只在确认文件确实来自本仓库后，前往“系统设置 → 隐私与安全性”允许打开。Apple Silicon 若提示需要 Rosetta，按系统提示安装。

下文的 Windows 命令可按以下规则换成其他系统：

```text
Windows: .\packwiz.exe COMMAND
macOS:   ./packwiz-macos COMMAND
Linux:   ./packwiz-linux COMMAND
```

## 从 Modrinth 和 CurseForge 搜索、安装模组

Packwiz 对两个平台使用简称：

- `mr` = Modrinth
- `cf` = CurseForge
- `add` = 添加；`install` 也是同一命令的别名。本项目文档统一使用 `add`。

### 通过名称搜索

```powershell
# 在 Modrinth 搜索
.\packwiz.exe mr add "MOD_NAME"

# 在 CurseForge 搜索
.\packwiz.exe cf add "MOD_NAME"
```

把 `MOD_NAME` 换成模组英文名。Packwiz 会显示搜索结果和可用版本；确认以下内容后再选择：

1. 作者和项目是否正确，避免选到同名模组或非官方搬运。
2. 游戏版本是否为 Minecraft 1.21.1。
3. 加载器是否明确支持 NeoForge。仓库虽然包含 Connector，但这不代表任意 Fabric 模组都兼容。
4. 文件是正式版、测试版还是开发版。
5. Packwiz 询问依赖时，检查依赖列表后再同意安装。

新手不要在模糊搜索时加 `-y`。该参数会自动接受默认项，Packwiz 自己也警告它可能选中并非预期的搜索结果。

### 通过 URL、slug 或项目 ID 添加

已知项目页面时，**使用完整 URL 最不容易选错**：

```powershell
# Modrinth 项目页、版本页、slug 或项目 ID 都可以
.\packwiz.exe mr add "https://modrinth.com/mod/PROJECT_SLUG"

# CurseForge 项目页、文件页、slug 或项目 ID 都可以
.\packwiz.exe cf add "https://www.curseforge.com/minecraft/mc-mods/PROJECT_SLUG"
```

需要锁定某个特定文件时，直接传版本页/文件页 URL。高级用法还可以查看：

```powershell
.\packwiz.exe mr add --help
.\packwiz.exe cf add --help
```

其中 CurseForge 支持 `--addon-id`、`--file-id`，Modrinth 支持 `--project-id`、`--version-id`。

### 添加完成后检查什么

正常结果通常是：

1. `mods/` 新增一个 `项目名.pw.toml`。
2. `index.toml` 被更新。
3. `pack.toml` 的 `[index].hash` 被更新。
4. `mods/` 中没有对应 JAR——由 `.pw.toml` 管理的模组本来就不需要提交 JAR。

然后执行：

```powershell
.\packwiz.exe refresh
.\packwiz.exe list
```

`list` 中应只出现一次该模组。若同一个模组已经由另一个平台或本地 JAR 提供，先停止并清理重复项，不能同时保留两份。

### 资源包也可以用 Packwiz 管理

Modrinth 的资源包页面 URL 可以直接交给 `mr add`。CurseForge 资源包可指定类别：

```powershell
.\packwiz.exe mr add "MODRINTH_RESOURCEPACK_URL"
.\packwiz.exe cf add --category texture-packs "CURSEFORGE_RESOURCEPACK_URL_OR_SLUG"
```

Packwiz 会依据项目类别把元数据放到相应目录。直接把 ZIP 放进 `resourcepacks/` 也能打包，但必须确认作者允许再分发，并在之后运行 `refresh`。

## 删除模组

### 删除由 `.pw.toml` 管理的模组

运行交互式删除命令：

```powershell
.\packwiz.exe remove
```

从列表中选中目标并确认。`remove` 也可写成 `rm`、`delete` 或 `uninstall`。当前附带版本的删除命令是交互式的，不要照搬网上的 `packwiz remove 模组名` 写法。

也可以手动删除对应的 `mods/项目名.pw.toml`，然后执行：

```powershell
.\packwiz.exe refresh
```

### 删除仓库中直接保存的 JAR

本仓库目前也有少数直接放在 `mods/` 的 JAR。删除这类模组时，删除实际 `.jar` 文件，再运行 `refresh`。不要误删其他模组仍需要的前置；先用 `packwiz list` 查看 Packwiz 管理的项目，并在本地启动验证。

删除后至少确认：

- `.pw.toml` 或本地 JAR 已消失；
- `index.toml` 不再记录它；
- `packwiz list` 不再显示它（本地原始 JAR 不会出现在外部模组列表中）；
- 游戏能启动，现有世界或脚本没有缺失依赖。

## `packwiz refresh` 到底做什么

```powershell
.\packwiz.exe refresh
```

`refresh` 会扫描仓库中未被 `.packwizignore` 排除的文件，重新生成 `index.toml` 中的路径和哈希，并更新 `pack.toml` 对整个索引的哈希。

下列改动后必须运行它：

- 新增、修改、移动或删除 `config/` 中的文件；
- 修改 KubeJS 脚本、纹理、数据包或 KubeJS 配置；
- 手动增删 JAR、资源包、光影包、菜单素材等；
- 手动删除 `.pw.toml`；
- 解决合并冲突后重建索引。

`mr add`、`cf add`、`remove` 和 `update` 通常会顺带更新索引，但团队 SOP 仍是在一次整合包改动结束时再运行一次 `refresh`。

`refresh` **不会**：

- 下载所有模组到本地；
- 把模组升级到最新版；
- 判断整合包能否启动；
- 自动提交或推送 Git；
- 修复配置、配方或脚本错误。

因此“运行过 refresh”只表示清单和文件一致，不表示改动已经通过测试。

### `.packwizignore` 与 `.gitignore` 的区别

- `.packwizignore`：决定哪些文件不写入 `index.toml`、不进入导出包。
- `.gitignore`：决定哪些本地文件不提交到 Git。

两者互不替代。本仓库的 `.packwizignore` 已排除 `README.md`、`PACKWIZ_GUIDE.md`、`.github/`、项目级 `/assets/`、开发工具目录、Packwiz 可执行文件以及导出的 `.mrpack`/ZIP；所以这些项目文档和构建工具不会被塞进玩家实例。本仓库的 `.gitignore` 暂时未排除任何文件，若后期有需要可视情况追加。

若某个文件“已经提交到 Git，却没有进入安装包”，先检查 `.packwizignore`；若某个文件“在本地存在，却无法提交”，先检查 `.gitignore`。

## 常用命令速查

以下仍以 Windows PowerShell 为例：

```powershell
# 查看帮助和当前外部模组清单
.\packwiz.exe --help
.\packwiz.exe list

# 添加
.\packwiz.exe mr add "名称、slug、ID 或 URL"
.\packwiz.exe cf add "名称、slug、ID 或 URL"

# 交互式删除
.\packwiz.exe remove

# 刷新文件索引
.\packwiz.exe refresh

# 更新一个项目；名称通常使用 .pw.toml 的文件名（不含扩展名）
.\packwiz.exe update PROJECT_NAME

# 更新所有外部项目——改动面很大，必须单独测试，不要随手执行
.\packwiz.exe update --all

# 本地导出，与 CI 的核心命令相同
.\packwiz.exe mr export
.\packwiz.exe cf export
```

若某个版本因兼容性原因不能升级，可用交互式 `packwiz pin` 固定；解除固定使用 `packwiz unpin`。固定原因应写在提交或 PR 描述中，避免后来的人误以为它可以安全更新。

## 本地导出整合包

本地导出使用与 GitHub Actions 相同的核心命令。导出前先在仓库根目录刷新索引：

```powershell
.\packwiz.exe refresh
```

推荐把产物写到仓库旁边的独立目录，避免把大文件留在 Git 工作区。Windows PowerShell：

```powershell
New-Item -ItemType Directory -Force ..\Gearhaven-builds | Out-Null

.\packwiz.exe mr export `
  -o ..\Gearhaven-builds\Gearhaven-local.mrpack

.\packwiz.exe cf export `
  -o ..\Gearhaven-builds\Gearhaven-local-CurseForge.zip
```

macOS；Linux 将可执行文件名替换为 `packwiz-linux`：

```bash
mkdir -p ../Gearhaven-builds

./packwiz-macos refresh
./packwiz-macos mr export \
  -o ../Gearhaven-builds/Gearhaven-local.mrpack
./packwiz-macos cf export \
  -o ../Gearhaven-builds/Gearhaven-local-CurseForge.zip
```

- `mr export` 生成可导入 Modrinth App、Prism Launcher 等启动器的 `.mrpack`。
- `cf export` 生成 CurseForge 格式 ZIP；默认导出客户端内容。需要筛选服务端模组时可使用 `cf export --side server`，但仍要单独审核配置和客户端专属内部文件。
- `-o` 明确指定输出路径。省略后产物会写到仓库根目录，不推荐这样做，也不要提交本地导出包。
- 导出可能需要下载另一平台的模组文件。必须检查完整终端输出；出现 `failed to download` 时，即使命令最后生成了文件，也不能直接分发。
- 导出包是测试和分发产物，不是新的编辑源。始终以 Git 仓库中的 Packwiz 源文件为准。

导出后，在启动器中创建一个全新的测试实例并导入产物。不要覆盖唯一的开发实例或个人存档；最终验证应覆盖首次安装、完整启动以及本次改动对应的游戏内场景。

## 本仓库目录结构

```text
Gearhaven/
├── pack.toml                  # 整合包元数据、MC/NeoForge 版本、索引哈希
├── index.toml                 # Packwiz 自动生成的文件清单；不要手改
├── .packwizignore            # 不进入整合包的文件规则
├── .gitignore                # 不进入 Git 的本地/运行时文件规则
├── mods/
│   ├── *.pw.toml             # CurseForge/Modrinth 等外部文件元数据
│   └── *.jar                 # 直接作为 override 随包分发的内部 JAR
├── config/                   # 常规模组配置
├── defaultconfigs/           # 新世界的服务端配置默认值
├── kubejs/
│   ├── startup_scripts/      # 注册物品/方块等启动阶段脚本
│   ├── server_scripts/       # 配方、标签、战利品和服务端事件
│   ├── client_scripts/       # 提示、JEI/客户端事件等
│   ├── assets/               # KubeJS 内置资源包内容
│   ├── data/                 # 可选：KubeJS 内置数据包内容
│   └── config/               # KubeJS 自身与脚本使用的配置
├── resourcepacks/            # 直接随包分发的资源包
├── shaderpacks/              # 直接随包分发的光影包
├── options.txt               # 整合包提供的客户端默认选项
├── fancymenu_data/           # FancyMenu 数据
├── .github/workflows/pack.yaml # GitHub Actions 导出流程
├── packwiz.exe               # Windows x86-64 CLI
├── packwiz-macos             # macOS x86-64 CLI
└── packwiz-linux             # Linux x86-64 CLI
```

`kubejs/data/` 是标准位置，但当前仓库尚未创建；确实需要数据包内容时再创建。

### `mods/`

优先保存 `.pw.toml`，不要同时提交由它下载的 JAR。`.pw.toml` 的常见字段包括：

- `name`、`filename`、`side`：名称、下载后的文件名、客户端/服务端适用范围；
- `[download]`：下载方式和哈希；
- `[update.modrinth]` 或 `[update.curseforge]`：项目与版本标识。

只有模组无法由受支持的平台或稳定直链管理时，才考虑提交原始 JAR。提交前确认许可证允许整合包再分发；二进制文件也会永久增大 Git 历史。

#### 直接随包分发的 override JAR

这里的 **override JAR** 指直接放在仓库 `mods/` 下的真实 `.jar`，而不是由 `.pw.toml` 描述并在安装时下载的模组。Packwiz 把它当作普通内部文件，与 `config/*.toml` 或 `kubejs/*.js` 的处理方式相同：

```toml
[[files]]
file = "mods/example.jar"
hash = "..."
```

它与 `.pw.toml` 在索引中的关键区别是：后者的条目包含 `metafile = true`，原始 JAR 没有该标记。

这会带来几个重要结果：

- `packwiz refresh` 只记录该 JAR 的路径和内容哈希；不会识别其中的模组 ID、版本或依赖。
- `packwiz list` 只列出外部元数据，因此**不会显示原始 override JAR**。
- `packwiz update`、`pin`、`unpin` 和交互式 `remove` 不会管理它；升级与删除都必须人工替换/删除 JAR，再运行 `refresh`。
- 原始 JAR 没有 `side = "client"` / `"server"` 元数据，Packwiz 无法按目标端自动筛选，通常会把它作为 override 放入导出包。
- Packwiz 不会按模组 ID 或文件哈希去重。一个原始 JAR、一个 `.pw.toml`，或者来自 CF/MR 的两份 `.pw.toml` 可以同时指向同一模组，最终造成重复加载。

在导出的 `.mrpack` 或 CurseForge ZIP 中，这类内部文件通常位于 `overrides/mods/`。另外，Packwiz 无法自动把 Modrinth 项目映射成 CurseForge 项目，反之亦然；跨平台导出时，某些由 `.pw.toml` 管理的模组也可能被下载并写入目标包的 overrides。这不等于源仓库中存在原始 JAR，应回到仓库检查实际文件类型。

团队规则：

1. 首选 `mr add` 或 `cf add`，保留来源、版本、更新和依赖元数据。
2. 平台不受支持但存在稳定直链时，优先考虑 `packwiz url add`，不要立即提交二进制。
3. 只有私有模组、自研构建、无稳定下载来源或确有再分发需求时，才直接保存 JAR。
4. 提交前确认许可证允许再分发，并在提交或 PR 中记录来源、版本、用途和无法使用 `.pw.toml` 的原因。
5. **绝不把测试实例的整个 `mods/` 复制回仓库。**只复制经过确认的单个 override JAR，并先检查是否已有同模组 `.pw.toml`。
6. 添加、替换或删除 JAR 后运行 `packwiz refresh`，再通过导出包验证它确实存在且只存在一份。

### `config/` 与 `defaultconfigs/`

- `config/` 对应游戏实例根目录中的同名目录，适合整合包的通用客户端、通用端和服务端配置。
- `defaultconfigs/` 是 NeoForge 为**新建世界**准备的 `serverconfig` 默认值。现有世界通常不会因为这里改变而自动同步；维护正式服务器时还要检查实际世界的 `serverconfig/`。
- 不要提交只属于个人的按键、账号、服务器地址、窗口大小或调试配置。`options.txt` 已由项目跟踪，只保留团队有意统一的默认值。

游戏生成的配置可能包含大量无意义重排。提交前只保留与本次改动有关的文件，避免把一次启动产生的全部个人设置一起提交。

### KubeJS

| 目录 | 用途 | 常见重载方式 |
| --- | --- | --- |
| `kubejs/startup_scripts/` | 注册物品、方块以及启动阶段定义 | 通常完整重启游戏；启动脚本热重载并不可靠 |
| `kubejs/server_scripts/` | 配方、标签、战利品、服务端事件 | 游戏内 `/reload` |
| `kubejs/client_scripts/` | 工具提示、JEI 和其他客户端事件 | `F3+T` |
| `kubejs/assets/<namespace>/` | 纹理、模型、语言文件等资源包内容 | `F3+T` |
| `kubejs/data/<namespace>/` | 配方、标签、战利品表、函数等数据包内容 | `/reload` |
| `kubejs/config/` | KubeJS 配置及脚本可访问的配置存储 | 视文件而定 |

建议：

- 命名空间、路径和文件名统一使用小写 ASCII；Linux 服务器区分大小写。
- 游戏内重载只用于快速验证。脚本或资源文件落盘后，仍要运行 `packwiz refresh`。
- 运行时日志位于实例的 `logs/kubejs/`；该目录不应提交。遇到脚本报错时记录完整错误、脚本路径和行号。
- `kubejs/assets/` 会进入游戏；仓库根目录的 `/assets/` 当前被 `.packwizignore` 排除，只适合项目文档或宣传素材。不要把二者混淆。

### 资源包、光影和菜单

`resourcepacks/`、`shaderpacks/`、`fancymenu_data/`都是随整合包复制到相同相对路径的内部文件。修改后运行 `refresh`。对于 ZIP、图片、音频、模型等二进制资源，提交前检查：

- 是否有明确的整合包再分发许可；
- 是否确实需要直接打包，而不是改用 `.pw.toml` 下载；
- 是否包含源工程、缓存、预览图或其他玩家不需要的文件；
- 文件名大小写是否与配置引用完全一致。

## 推荐协作 SOP

> [!IMPORTANT]
> **配置、KubeJS 脚本、纹理和其他内部文件的推荐编辑流程：**
>
> 1. **把当前整合包安装到启动器中的独立开发实例。**
> 2. **直接在该实例的 `.minecraft` 中修改并反复测试。**
> 3. **测试通过后，只把有意保留的改动复制回 Git 仓库，并整理成逻辑清晰的提交。**
>
> 这样无需每改一行配置或脚本就重新构建、重新安装整合包，同时能让 Git 工作区远离游戏运行时生成的噪声。

完整流程：

1. 同步目标分支，确认仓库工作区没有与本次任务无关的改动。
2. 使用本地导出包或 GitHub Actions Artifact，在启动器中创建/更新一个独立的 `Gearhaven-Dev` 测试实例。
3. 首次启动一次，让模组在实例的 `.minecraft` 中生成所需配置。
4. 在测试实例中调整 `config/`、`defaultconfigs/`、`kubejs/`、纹理、资源包、菜单素材等，并用 `/reload`、`F3+T` 或完整重启验证。
5. 测试通过后，按相同相对路径把**有意修改的文件**复制回仓库，例如：

   ```text
   实例/.minecraft/config/example.toml
     → 仓库/config/example.toml

   实例/.minecraft/kubejs/server_scripts/recipes.js
     → 仓库/kubejs/server_scripts/recipes.js
   ```

6. 不要复制 `logs/`、`cache/`、`crash-reports/`、`saves/`、下载缓存、个人账号/服务器信息，也不要整体复制实例的 `mods/`。
7. 用 `git status` 和 `git diff` 逐项检查回写内容，移除游戏自动重排但与任务无关的配置变化。
8. 按逻辑拆分提交，例如“新增配方”“调整客户端默认配置”“更新菜单纹理”，不要把所有实例变化塞进一个提交。
9. 运行 `packwiz refresh` 和 `packwiz list`；若改动会进入整合包，提交中通常应同时包含实际文件、`index.toml` 与更新后的 `pack.toml`。
10. 最后重新导出并导入一个干净实例，验证从零安装仍然成立；推送后再确认对应 GitHub Actions 运行成功。

模组集合本身例外：最终添加或删除模组必须在仓库中通过 `mr add`、`cf add` 或 `remove` 完成。可以先在测试实例临时试装模组，但验证通过后应转成正确的 `.pw.toml`；除非满足 override JAR 规则，否则不要把试装 JAR 复制回仓库。

CI 只执行导出，不启动 Minecraft。即使 Actions 显示成功，模组冲突、Mixin 崩溃、配方错误和世界损坏仍只能通过实际测试发现。

### 多人修改时如何处理冲突

`index.toml` 和 `pack.toml` 的索引哈希是集中生成文件，多人同时修改整合包时很容易冲突。安全做法：

1. 先合并并确认双方真正需要保留的 `.pw.toml`、配置、脚本和资源文件。
2. 删除所有冲突标记。
3. 不要凭手工猜测哈希；在最终文件集合上重新运行 `packwiz refresh`。
4. 再检查 `packwiz list` 并启动测试。

## 使用 GitHub Actions 下载打包结果

### 先理解三个名词

- **Workflow（工作流）**：自动化流程定义。本仓库是 `.github/workflows/pack.yaml`，页面名称为 **Packwiz Export**。
- **Run（运行记录）**：某次提交触发的一次执行，绑定具体分支和 commit。
- **Artifact（构建产物）**：该次运行上传、可供下载的文件，不等同于 GitHub Release。

本仓库的工作流在**每次 push、任意分支**上运行，依次：

1. 检出对应提交；
2. 赋予 `packwiz-macos` 执行权限；
3. 执行 `packwiz-macos mr export` 和 `packwiz-macos cf export`；
4. 上传两个 Artifact：
   - `Artifacts-Modrinth`
   - `Artifacts-CurseForge`

当前工作流没有 `workflow_dispatch`，所以页面上不一定有 **Run workflow** 手动运行按钮。正常触发方式是推送提交。

### 在网页中下载

1. 登录 GitHub 并打开本仓库；你需要对仓库有读取权限。
2. 点击仓库顶部的 **Actions**。
3. 在左侧选择 **Packwiz Export**。
4. 在运行列表中，选择目标**分支和 commit** 对应的记录，不要只凭“最新”二字判断。
5. 等待绿色对勾。黄色圆点表示仍在运行，红色叉号表示构建失败。
6. 打开该次运行，在页面底部找到 **Artifacts** 区域。
7. 按启动器选择：
   - 使用 Modrinth App、Prism Launcher 或支持 `.mrpack` 的启动器：下载 **Artifacts-Modrinth**。
   - 使用 CurseForge 格式导入：下载 **Artifacts-CurseForge**。
8. GitHub 下载下来的是一个“Artifact 外层 ZIP”。先解压这一层：
   - Modrinth Artifact 内部才是要导入的 `.mrpack`；
   - CurseForge Artifact 内部还有真正要导入的 CurseForge `.zip`。
9. 在启动器中选择“从文件导入/Import from file”，选中内层 `.mrpack` 或内层 CurseForge ZIP。不要把包内容手动散放进一个旧实例。

Artifact 默认保留 90 天，仓库设置可以修改期限；需要的测试包应及时下载。具体规则见 [GitHub 官方文档：下载工作流 Artifact](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/download-workflow-artifacts)。

### 使用 GitHub CLI 下载（可选）

安装并登录 [GitHub CLI](https://cli.github.com/) 后：

```powershell
# 列出该工作流最近的运行，记下目标 RUN_ID
gh run list --workflow pack.yaml

# 下载指定运行的 Modrinth 产物
gh run download RUN_ID -n Artifacts-Modrinth

# 下载指定运行的 CurseForge 产物
gh run download RUN_ID -n Artifacts-CurseForge
```

`gh run download` 会把指定 Artifact 解压到当前目录；仍应检查其中真正的 `.mrpack` 或 CurseForge ZIP，并核对运行对应的 commit。

### 构建失败时

1. 打开红色的运行记录。
2. 展开 `build` job，再展开失败的 step；本仓库最关键的是 **Export modpack artifacts**。
3. 记录失败日志、分支和 commit SHA，交给本次改动的作者处理。
4. 修复并推送新提交，等待新的运行成功。
5. 不要因为本次失败就把旧运行的包当作新版本分发。

## 常见问题

### 为什么添加模组后没有 JAR？

由 `.pw.toml` 管理时这是正确结果。导入 `.mrpack`/CurseForge ZIP 的启动器会下载 JAR。

### 为什么双击 `packwiz.exe` 一闪而过？

它是命令行程序。请在仓库目录打开 PowerShell，再运行 `.\packwiz.exe --help`。

### 为什么提示找不到 `pack.toml`？

终端不在仓库根目录。先 `cd` 到同时包含 `pack.toml` 和 Packwiz 可执行文件的目录。

### 为什么修改的文件没有出现在安装包里？

先运行 `packwiz refresh`，再检查该路径是否被 `.packwizignore` 排除。README、协作指南、`.github/`、根目录 `/assets/` 和 Packwiz 二进制被排除是本仓库的预期行为。

### 为什么 Actions 成功但客户端仍崩溃？

该工作流只导出文件，没有启动 Minecraft。必须查看客户端日志/崩溃报告并复现真实启动流程。

### 为什么同一模组出现两次？

通常是同时存在两份 `.pw.toml`，或 `.pw.toml` 与手动 JAR 重复。保留一个明确来源，删除另一个，然后运行 `refresh` 并重新测试。

### 可以直接编辑 `index.toml` 吗？

不可以。它是生成文件。修改真正的源文件，再运行 `packwiz refresh`。

## 参考资料

- [Packwiz 文档](https://packwiz.infra.link/)
- [Packwiz：安装](https://packwiz.infra.link/installation/)
- [Packwiz：添加模组和资源包](https://packwiz.infra.link/tutorials/creating/adding-mods/)
- [KubeJS 文档](https://kubejs.com/)
- [GitHub：下载工作流 Artifact](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/download-workflow-artifacts)
- [本仓库的导出工作流](.github/workflows/pack.yaml)
