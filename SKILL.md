---
name: meituan-c-user-auth
description: 美团C端用户Agent认证工具。为需要美团用户身份的 Skill（如发券、查订单等）提供手机号验证码登录认证，管理用户Token，实现"一次认证、持续有效"。当其他 Skill 需要校验用户身份、获取用户Token时，作为前置认证模块调用。触发词：美团登录、用户认证、手机号验证、发送验证码、获取Token、切换账号、退出登录。
version: "1.0.0-SNAPSHOT"
---

# 美团C端用户认证工具

---

## 环境准备

**macOS：**
```bash
PYTHON=~/Library/Application\ Support/xiaomei-cowork/Python311/python/bin/python3
SCRIPT="$CLAUDE_CONFIG_DIR/skills/meituan-c-user-auth/scripts/auth.py"
```

**Windows（Git Bash）：**
```bash
PYEXE="$(cygpath "$APPDATA")/xiaomei-cowork/Python311/python/python.exe"
SCRIPT="$CLAUDE_CONFIG_DIR/skills/meituan-c-user-auth/scripts/auth.py"
# 后续命令将 $PYTHON 替换为 "$PYEXE"
```

**Linux / 其他 Agent 环境：**
```bash
# 使用系统 Python 3（或自定义路径）
PYTHON=python3
SCRIPT="$CLAUDE_CONFIG_DIR/skills/meituan-c-user-auth/scripts/auth.py"
# 如需自定义 Token 存储路径（沙箱/隔离场景）：
export XIAOMEI_AUTH_FILE=/tmp/my_auth_tokens.json
```

> ⚠️ `$CLAUDE_CONFIG_DIR` 在 macOS 路径含空格，**SCRIPT 变量赋值和使用时均需加双引号**。

---

## 命令一览

| 命令 | 说明 | 是否调用远程接口 |
|------|------|----------------|
| `version-check` | 检查本地 Skill 版本，与广场版本对比 | ⚡ 本地为主，有远程源时才请求 |
| `status` | 本地检查 Token 是否存在及过期 | ❌ 本地只读 |
| `token-verify` | 调用服务端校验 Token 真实有效性 | ✅ 调用接口 |
| `send-sms --phone <手机号>` | 发送短信验证码 | ✅ 调用接口 |
| `verify --phone <手机号> --code <验证码>` | 验证验证码，写入 Token | ✅ 调用接口 |
| `logout` | 退出登录，user_token 置空（保留 device_token） | ❌ 本地操作 |

---

## 版本检查（每次对话首次使用时执行）

> 每次对话中**第一次**使用本 Skill 时执行，同一对话内无需重复。

**实现原理**：Friday 广场为 SPA 应用，无公开 REST API。由小美通过 `agent-browser` 打开广场详情页，从页面渲染文本中提取 `version:` 字段，再传给脚本进行对比。

### 步骤一：用 agent-browser 获取广场远程版本

```bash
agent-browser open "https://friday.sankuai.com/skills/skill-detail?activeTab=overview&id=12210"
# 等待页面加载后：
agent-browser get text body
```

从输出文本中找到 `version: "x.y.z"` 字段，提取版本号（如 `2.5.0`）。

### 步骤二：调用脚本进行版本对比

```bash
# 将上一步提取到的远程版本号通过 --remote 参数传入
$PYTHON "$SCRIPT" version-check --remote <远程版本号>

# 示例：
$PYTHON "$SCRIPT" version-check --remote "2.5.0"

# 若上一步获取失败，不传 --remote，脚本会静默跳过对比
$PYTHON "$SCRIPT" version-check
```

**处理逻辑**：
- `up_to_date: true` → 继续执行，无需提示
- `up_to_date: false` → 提示用户：
  ```
  本地 Skill 版本较旧（当前 x.y.z，广场最新 a.b.c），建议前往 Friday 广场更新以获取最新能力。
  继续使用旧版本也可正常登录。
  ```
- `up_to_date: null`（未传入远程版本）→ 静默跳过，不影响正常流程

---

## 标准认证流程（每次调用必须按此步骤执行）

```
┌────────────────────────────────────────────┐
│  第一步：调用远程接口校验 Token             │
│  $PYTHON "$SCRIPT" token-verify              │
│  ├── valid: true  → 告知用户已登录         │
│  │           (手机号 phone_masked)          │
│  │           返回 user_token，流程结束      │
│  └── valid: false → 进入第二步            │
└────────────────────────────────────────────┘
          ↓（Token 无效或不存在）
┌────────────────────────────────────────────┐
│  第二步：引导用户输入手机号                 │
│  "请输入您的美团账号手机号："               │
│  等待用户输入                              │
└────────────────────────────────────────────┘
          ↓
┌────────────────────────────────────────────┐
│  第三步：发送短信验证码                     │
│  $PYTHON "$SCRIPT" send-sms --phone <手机号>  │
│  ├── 成功 → 告知用户：                     │
│  │   "验证码已发送至手机 xxx****xxxx，     │
│  │    请打开手机短信查看验证码，            │
│  │    60秒内有效"                         │
│  └── 失败 → 告知原因（见错误码说明）       │
└────────────────────────────────────────────┘
          ↓
┌────────────────────────────────────────────┐
│  第四步：等待用户输入验证码                 │
│  "请输入您收到的6位验证码："                │
│  等待用户输入                              │
└────────────────────────────────────────────┘
          ↓
┌────────────────────────────────────────────┐
│  第五步：验证验证码                         │
│  $PYTHON "$SCRIPT" verify \                  │
│    --phone <手机号> --code <验证码>         │
│  ├── 成功 → "认证成功，xxx****xxxx 已登录"  │
│  │           user_token 已自动写入          │
│  │           返回 user_token 供调用方使用   │
│  └── 失败 → 告知原因，提示重新发送或重试   │
└────────────────────────────────────────────┘
```

---

## 错误码说明（告知用户时使用友好描述）

| 错误码 | 友好提示 |
|--------|---------|
| 20002 | 验证码已发送，请等待1分钟后再试 |
| 20003 | 验证码错误或已过期（60秒有效），请重新获取 |
| 20004 | 该手机号未注册美团，请先下载美团APP完成注册 |
| 20005 | 登录状态已过期，需要重新认证 |
| 99997 | 系统繁忙，请稍后重试 |
| 99999 | 参数错误，请检查手机号格式是否正确 |

---

## 供其他 Skill 调用的约定

在调用方的 SKILL.md 中写：

```markdown
## 前置认证
1. 调用 meituan-c-user-auth Skill，按标准认证流程执行
2. 获取有效 user_token
3. 携带 user_token 调用业务接口
4. 若业务接口返回 Token 无效错误，重新触发认证流程
```

---

## 注意事项

1. **Token 校验使用 `token-verify`**（远程接口），而非 `status`（仅本地过期检查）
2. **验证码60秒有效**，1分钟内不能重复发送，发送前提醒用户
3. **Token 有效期7天**，到期需重新认证
4. **user_token 不要在对话中显示**，仅传递给业务接口
5. **退出/切换账号**：执行 `logout` 命令清除 Token
6. **device_token 不要在对话中展示**：device_token 是设备唯一标识，属于内部字段，正常交互中不得向用户输出；仅在排查登录问题时，且用户明确要求查看时，才可展示

---

## API 信息摘要

| 接口 | 路径 | 方法 | 关键请求字段 |
|------|------|------|------------|
| 发送验证码 | `/eds/claw/login/sms/code/get` | POST | `mobile`, `uuid`（device_token） |
| 验证验证码 | `/eds/claw/login/sms/code/verify` | POST | `mobile`, `smsVerifyCode`, `uuid`（device_token） |
| 校验 Token | `/eds/claw/login/token/verify` | POST | `?token=<token>` (Query) |

> 当前使用**线上外网**域名：`https://peppermall.meituan.com`

完整接口文档见 `references/api-config.md`
