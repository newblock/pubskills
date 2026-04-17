# PubSkills

> 对外公开的 Kimi / Agent Skill 集合

## 简介

本仓库用于存放可直接使用的 AI Skill 定义文件，兼容 Kimi Code CLI 等支持 `SKILL.md` 规范的 Agent 平台。

## 已收录 Skill

| Skill | 描述 |
|-------|------|
| [zbcom_apply](./zbcom_apply) | 帮助求职者通过 API Key 管理简历基础信息（basics），支持交互式对话和 JSON 批量录入，含管理员审核机制。 |

## 使用方式

将本仓库克隆到本地 Skill 目录即可：

```bash
# 以 Kimi CLI 为例
~/.agents/skills/
```

然后复制需要的 Skill 目录到上述路径中。

## 贡献

欢迎提交 PR 新增或改进 Skill。
