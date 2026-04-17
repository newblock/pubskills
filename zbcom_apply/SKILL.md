---
name: zbcom_apply
description: 帮助求职者通过 API Key 管理简历基础信息（basics），支持交互式对话和 JSON 批量录入，含管理员审核机制。
---

# Resume Seeker Skill

## 简介

本 Skill 用于帮助求职者通过自然语言对话，管理自己的简历基础信息（`seeker_basics`）。

支持两种录入方式：
- **交互式对话**：逐字段询问并填写
- **JSON 批量录入**：直接粘贴 JSON 对象，一键导入

**核心流程**：
1. 提交基础信息（姓名 + 微信 + GitHub + 可选字段）
2. 等待管理员审核
3. 审核通过后，方可继续完善技能、经历、教育、项目等高级内容

## 使用前提

用户必须提供 **API Key**（`sk-` 开头）。如果用户没有 API Key：
1. 引导用户先注册账号：`POST /api/auth/register`
2. 注册后系统会发送 API Key 到邮箱
3. 用户将 API Key 提供给你后，保存到对话上下文，后续自动携带

> **重要原则**：注册成功后，Skill **只能**提示用户"请查收邮件获取 API Key"，**不得**直接将 API Key 展示在对话中。必须由用户主动从邮箱复制并粘贴 API Key 到对话中，方可继续后续流程。

## 服务端接口

```
Base URL: http://localhost:5000
```

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/auth/register` | POST | 注册账号 |
| `/api/auth/reset-api-key` | POST | 重置 API Key（验证邮箱密码后重新发送） |
| `/api/seeker/basics` | POST | 提交基础信息 |
| `/api/seeker/basics/:id` | PUT | 修改基础信息 |
| `/api/seeker/me` | GET | 查看我的完整档案（含审核状态） |
| `/api/seeker/my-skills` | POST | 添加技能（需审核通过） |
| `/api/seeker/admin/basics/:id/approve` | PUT | 管理员通过审核 |
| `/api/seeker/admin/basics/:id/reject` | PUT | 管理员拒绝审核 |
| `/api/seeker/admin/basics/pending` | GET | 管理员查看待审核列表 |

## 交互式工作流

### 1. 注册账号 / 重置 API Key

当用户要求**注册账号**时：
1. 收集邮箱、密码、用户类型（`seeker`）
2. 调用 `POST /api/auth/register`
3. **即使后端响应包含 API Key，也严禁直接展示给用户**
4. 统一回复："注册成功，请查收邮件获取 API Key，收到后发给我即可继续。"
5. 等待用户主动提供 API Key 后，再进入"配置 API Key"流程

当用户**忘记/丢失 API Key** 时：
1. 收集邮箱、密码
2. 调用 `POST /api/auth/reset-api-key`
3. 统一回复："API Key 已重置，请查收邮件获取新的 API Key，收到后发给我即可继续。"
4. 等待用户主动提供新的 API Key

### 2. 配置 API Key

当用户提供了 API Key：
- 保存到上下文变量 `apiKey`
- 调用 `GET /api/seeker/me` 验证有效性
- 告知用户"API Key 已保存"

### 2. 添加 / 修改基础信息（交互式）

当用户说"添加基础信息"、"更新简历"、"填写资料"等时：

1. 调用 `GET /api/seeker/me`，检查是否已有 `basics`
2. **如果已有 `basics`**：
   - 展示当前信息和审核状态
   - 询问用户"你想修改哪个字段？"
3. **如果没有 `basics`**：
   - 依次询问用户，**姓名、微信号、GitHub 为必填**，不可跳过
   - 个人网站、自我介绍为选填，可以说"跳过"
4. 收集完毕后，调用 `POST /api/seeker/basics` 或 `PUT /api/seeker/basics/:id`
5. 提交后告知用户：**"基础信息已提交，等待管理员审核。审核通过后才能继续完善技能、经历等内容。"**

### 3. 用户尝试操作高级内容时的拦截

当用户在 `status !== 'approved'` 时尝试：
- 添加技能
- 更新主档案（seekers 表）
- 添加经历 / 教育 / 项目

**必须拦截并提示**：
> "你的基础信息审核状态为 pending/rejected，暂无权操作。请先等待管理员审核通过。"

### 4. 管理员审核（仅当用户是管理员时）

当管理员说"查看待审核"、"通过审核"、"拒绝审核"时：
- 调用 `GET /api/seeker/admin/basics/pending` 查看列表
- 根据提供的 id 调用 `approve` 或 `reject`

## JSON 批量录入

当用户提供了 JSON 对象时：
- 解析 JSON
- 校验 `name`、`wechat`、`github` 必填
- 调用 `POST /api/seeker/basics` 或 `PUT /api/seeker/basics/:id`

### JSON 示例

```json
{
  "name": "张三",
  "wechat": "zhangsan_dev",
  "github": "https://github.com/zhangsan",
  "website": "https://zhangsan.dev",
  "intro": "5年全栈开发经验，擅长 React、Node.js 和云原生技术。"
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | ✅ | 姓名 |
| `wechat` | string | ✅ | 微信号 |
| `github` | string | ✅ | GitHub 地址 |
| `website` | string | ❌ | 个人网站 |
| `intro` | string | ❌ | 自我介绍，1000 汉字以内 |

## 上下文变量

- `apiKey`: 用户的 API Key

## 注意事项

- 所有接口必须携带 `X-API-Key: <apiKey>`
- `name`、`wechat`、`github` 三个字段为必填，Skill 和后端都会校验
- 基础信息提交后状态为 `pending`，审核通过后才解锁后续功能
- 修改基础信息后会重新变为 `pending`，需再次审核
- 如果 API 返回 403，提示"API Key 无效或只能操作自己的数据"
