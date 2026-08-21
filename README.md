# Gearheaven（齿轮星都）

Gearheaven 是 Unda Rubra 维护的 Minecraft 整合包。本仓库保存整合包源文件，并使用 Packwiz 管理模组元数据、配置、脚本与导出清单；它不是可以直接启动的 Minecraft 实例。

## 技术基线

- Minecraft `1.21.1`
- NeoForge `21.1.248`
- Packwiz `packwiz:1.1.0`

具体版本以 [`pack.toml`](pack.toml) 为准。

## 协作开发

首次参与项目前，请阅读完整的 **[Packwiz 整合包协作指南](PACKWIZ_GUIDE.md)**。指南包含：

- Windows、macOS 与 Linux 的 Packwiz CLI 使用方法；
- 从 Modrinth / CurseForge 搜索、添加、删除和更新模组；
- `packwiz refresh`、目录结构、配置文件和 KubeJS 约定；
- Git 协作流程；
- 从 GitHub Actions 下载 Modrinth 构建产物，并在需要时本地导出 CurseForge 兼容包。

每次推送都会触发 [`Packwiz Export`](.github/workflows/pack.yaml)，生成供测试和导入启动器的构建产物。Actions 成功只表示导出完成；模组兼容性仍需通过实际启动和游戏内测试验证。
