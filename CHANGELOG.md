# Changelog

## 0.1.0 - 2026-10-31

首个 DSH 适配版本，自 `codex-vibecoding` 派生。

- 整包语义改为 DSH：`$DSH_HOME` / `~/.dsh`、五级技能根、`schedule_create` 参数模型，示例统一为 bash（`$HOME/...`、`export PYTHONUTF8=1`）。
- **新增 `references/dsh-plugin-development.md`**：DSH 平台扩展判据与落地路径——五类扩展点与成本顺位、profile 现场取证（`dsh.profile.bundles` / `cordis.patch.yml` / `cordis.yml` / `pluginInventory/list`）、最小可运行插件、`defineTool` 工具插件、组合包 manifest（`dsh.manifestVersion` / `bundle.patch` / `client`）、安装启停与 profile 合并语义、调试观测表、8 项验收清单、边界与坑。
- `SKILL.md` 增加 `## DSH 平台扩展` 一节，T2 档由「全流程」扩到「平台扩展（插件/组合包/工具）」。
- `references/skill-ecosystem.md` 的宿主文档锚点改指本机 DSH 安装包 README，并标注官方 `docs/` 与社区文章的可用边界。
- `eval/tasks.md` 增加插件形态的对拍样例。
- 补 MIT LICENSE 与 `.gitattributes`（`eol=lf`）。

平台事实来源为本机 `@deepseek-ai/*` 安装包 README（对 0.2.1-rc.3 权威）；社区文章仅作线索。

## 上游 codex-vibecoding 历史

上游 `codex-vibecoding` 未维护 CHANGELOG，其 v0.1.0 为 Codex 侧首个公开版本（2026-10-26）。DSH 侧版本号独立从 0.1.0 起算。
