---
name: zbcom_apply
description: 帮助技术型 OPC 通过 API Key 管理简历基础信息（basics），支持交互式对话和 JSON 批量录入，含管理员审核机制。触发关键字：申请加入、加入社区、申请知北、加入技术社区、加入知北。
---

# Resume Seeker Skill

## 简介

本 Skill 用于帮助技术型 OPC 通过自然语言对话，管理自己的简历基础信息（`seeker_basics`）。

支持两种录入方式：
- **交互式对话**：逐字段询问并填写
- **JSON 批量录入**：直接粘贴 JSON 对象，一键导入

**核心流程**：
1. 提交基础信息（姓名 + 微信 + GitHub + 可选字段）
2. 等待管理员审核
3. 审核通过后，方可继续完善技能、经历、教育、项目等高级内容

## 使用前提

用户必须提供 **API Key**（`sk-` 开头）才能管理简历。但 Skill 的入口**从邮箱开始**，而不是直接问 API Key：

1. **先询问用户的邮箱地址**
2. **判断邮箱是否已注册**：
   - **已注册**：提示用户输入密码，调用登录或重置 API Key 接口，让系统重新发送 API Key 到邮箱
   - **未注册**：收集密码，调用 `POST /api/auth/register` 创建账号，系统自动发送 API Key 到邮箱
3. 用户从邮箱获取 API Key 后，粘贴到对话中，Skill 保存到上下文变量 `apiKey`，后续自动携带

> **重要原则**：注册/重置成功后，Skill **只能**提示用户"请查收邮件获取 API Key"，**不得**直接将 API Key 展示在对话中。必须由用户主动从邮箱复制并粘贴 API Key 到对话中，方可继续后续流程。

## 服务端接口

```
Base URL: https://resume.gpuart.cn
```

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/auth/check-email` | GET | 检查邮箱是否已注册 |
| `/api/auth/register` | POST | 注册账号 |
| `/api/auth/reset-api-key` | POST | 重置 API Key（验证邮箱密码后重新发送） |
| `/api/seeker/basics` | POST | 提交基础信息 |
| `/api/seeker/basics/:id` | PUT | 修改基础信息 |
| `/api/seeker/me` | GET | 查看我的完整档案（含审核状态） |
| `/api/seeker/my-skills` | POST | 添加技能（需审核通过） |
| `/api/seeker/admin/basics/:id/approve` | PUT | 管理员通过审核 |
| `/api/seeker/admin/basics/:id/reject` | PUT | 管理员拒绝审核 |
| `/api/seeker/admin/basics/pending` | GET | 管理员查看待审核列表 |

### 接口参数详情

**所有请求必须携带 `Content-Type: application/json`。**

#### 1. `GET /api/auth/check-email` — 检查邮箱是否已注册

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `email` | string | ✅ | 邮箱地址 |

**请求示例**：
```
GET /api/auth/check-email?email=user@example.com
```

**常见响应**：
- `200` 已注册 — `{ "registered": true, "message": "邮箱已被注册" }`
- `200` 未注册 — `{ "registered": false, "message": "邮箱未注册" }`

#### 2. `POST /api/auth/register` — 注册账号

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `email` | string | ✅ | 邮箱地址 |
| `password` | string | ✅ | 登录密码 |
| `userType` | string | ✅ | 用户类型，后端固定传 `seeker`，对外显示为 **技术型 OPC** / **技术专家**（兼容字段 `type`） |

**请求示例**：
```json
{
  "email": "user@example.com",
  "password": "your-password",
  "userType": "seeker"  /* 后端固定值，对外显示为 技术型 OPC */
}
```

**常见响应**：
- `201` — `{ "message": "注册成功，请查收邮件获取 API Key", "user": { ... } }`
- `409` — `{ "message": "邮箱已被注册" }`
- `400` — `{ "message": "邮箱、密码和用户类型为必填项" }`

#### 3. `POST /api/auth/reset-api-key` — 重置 API Key

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `email` | string | ✅ | 注册邮箱 |
| `password` | string | ✅ | 登录密码 |

**请求示例**：
```json
{
  "email": "user@example.com",
  "password": "your-password"
}
```

**常见响应**：
- `200` — `{ "message": "API Key 已重置，请查收邮件获取新的 API Key" }`
- `401` — `{ "message": "邮箱或密码错误" }`

#### 4. `GET /api/seeker/me` — 查看我的档案

**请求头**：`X-API-Key: <your-api-key>`

**常见响应**：
- `200` — 返回用户完整档案，包含 `basics`（基础信息及 `status`）、`skills`、`experience` 等
- `401/403` — API Key 无效或过期

#### 5. `POST /api/seeker/basics` — 提交基础信息

**请求头**：`X-API-Key: <your-api-key>`

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | ✅ | 姓名 |
| `wechat` | string | ✅ | 微信号 |
| `github` | string | ✅ | GitHub 地址 |
| `website` | string | ❌ | 个人网站 |
| `intro` | string | ❌ | 自我介绍 |

#### 5. `PUT /api/seeker/basics/:id` — 修改基础信息

请求头和字段与 `POST /api/seeker/basics` 相同，URL 中的 `:id` 为 `basics.id`。

## 交互式工作流

### 1. 从邮箱开始（注册 / 登录 / 重置 API Key）

当用户说"申请加入"、"加入社区"或首次进入 Skill 时：

1. **询问邮箱地址**："请告诉我你的邮箱地址"
2. **调用 `GET /api/auth/check-email?email=xxx` 检查邮箱状态**
3. **分流程处理**：

> **重要**：严禁在用户未提供密码前调用 `POST /api/auth/register` 试探注册，这会导致账号被注册但用户不知道密码。

#### A. 邮箱未注册
- 收集**密码**
- 收集**用户类型**（对外显示为 **技术型 OPC** 或 **技术专家**，后端固定传 `seeker`）
- 调用 `POST /api/auth/register`
- 统一回复："注册成功，请查收邮件获取 API Key，收到后发给我即可继续。"

#### B. 邮箱已注册
- 收集**密码**
- 调用 `POST /api/auth/reset-api-key`
- 统一回复："API Key 已重置并发送到你的邮箱，请查收邮件获取新的 API Key，收到后发给我即可继续。"

4. 等待用户主动提供 API Key 后，进入"配置 API Key"流程

### 2. 配置 API Key

当用户提供了 API Key：
- 保存到上下文变量 `apiKey`
- 调用 `GET /api/seeker/me` 验证有效性
- 告知用户"API Key 已保存，接下来可以填写基础信息了"

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
