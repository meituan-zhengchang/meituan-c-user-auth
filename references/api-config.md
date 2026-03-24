# EDS Claw 短信登录接口文档

> 来源：https://km.sankuai.com/collabpage/2751618839
> 作者：wujinglong02
> 创建时间：2026-03-22
> 内部接口，仅供 Claw 内部调用

---

## 域名信息

| 环境 | 域名 |
|------|------|
| 泳道测试 | `https://wujinglong02-nuvnc-sl-pepper.mall.test.sankuai.com` |
| 测试环境 | `https://pepper.mall.test.sankuai.com` |
| **线上内网** | `https://peppermall.sankuai.com` |
| 线上外网 | `https://peppermall.meituan.com` |

> ✅ 当前 `auth.py` 使用**线上外网**域名：`https://peppermall.meituan.com`

---

## 公共响应结构（StandardInsideResponse）

```json
{ "code": 0, "message": "success", "data": null }
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `code` | int | 0 = 成功，其他见错误码表 |
| `message` | String | 响应描述 |
| `data` | T | 业务数据，失败时为 null |

---

## 错误码说明

| 错误码 | 枚举名 | 描述 | 涉及接口 |
|--------|--------|------|---------|
| 0 | SUCCESS | 成功 | 全部 |
| 20000 | SMS_SEND_FAIL | 短信发送失败 | 发送验证码 |
| 20001 | SMS_MOBILE_TOKEN_ENCRYPT_FAIL | 手机号短信验证服务调 Token 加密服务失败，服务端正在处理中，请稍后重试 | 发送验证码 |
| 20002 | SMS_VERIFY_CODE_EXIST | 短信验证码已发送，请1分钟后再试 | 发送验证码 |
| 20003 | SMS_VERIFY_CODE_ERROR | 短信验证码错误 | 验证验证码 |
| 20004 | CLAW_USER_NOT_REGISTERED | 手机号未注册美团，请先下载美团APP并完成注册登录 | 发送/验证验证码 |
| 20005 | CLAW_USER_NOT_LOGIN | 用户未登录或 Token 已过期 | 验证 Token |
| 99997 | SYSTEM_ERROR | 系统繁忙，请稍后再试（含限流） | 全部 |
| 99999 | PARAM_ERROR | 参数错误（手机号为空/格式非法/验证码为空） | 全部 |

> 有效期说明：
> - 验证码有效期：**60 秒**（MDP 配置：`claw.verify.code.sms.expire.seconds`）
> - 登录 Token 有效期：**7 天**（MDP 配置：`claw.login.token.expire.seconds`）

---

## 接口一：发送短信验证码

```
POST /eds/claw/login/sms/code/get
Content-Type: application/json
```

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `mobile` | String | ✅ | 手机号，需符合国内手机号格式 |
| `smsVerifyCode` | String | ❌ | 本接口不使用，传空即可 |
| `uuid` | String | ❌ | 扩展字段，暂未使用 |

**请求示例：**
```json
{ "mobile": "13800138000" }
```

**成功响应：**
```json
{ "code": 0, "message": "success", "data": null }
```

**异常情况：**

| 错误码 | 触发条件 |
|--------|---------|
| 99999 | 手机号为空或格式不合法 |
| 20004 | 手机号未在美团注册 |
| 20002 | 验证码在有效期内，不可重复发送（1分钟频控） |
| 20000 | 验证码缓存失败或短信平台调用失败 |
| 99997 | 触发限流或系统异常 |

---

## 接口二：验证短信验证码并生成 Token

```
POST /eds/claw/login/sms/code/verify
Content-Type: application/json
```

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `mobile` | String | ✅ | 手机号 |
| `smsVerifyCode` | String | ✅ | 短信验证码 |
| `uuid` | String | ❌ | 扩展字段，暂未使用 |

**请求示例：**
```json
{ "mobile": "13800138000", "smsVerifyCode": "123456" }
```

**成功响应（data 为 ClawSmsLoginResponse）：**
```json
{
  "code": 0,
  "message": "success",
  "data": {
    "token": "xxxxxxxxxxxxxxxxxxxxxxxx"
  }
}
```

> `token` 有效期 **7 天**

**异常情况：**

| 错误码 | 触发条件 |
|--------|---------|
| 99999 | 手机号为空/格式不合法，或验证码为空 |
| 20003 | 验证码不匹配或已过期（Redis 中不存在） |
| 20004 | 手机号未在美团注册 |
| 99997 | 触发限流、Token 生成失败或缓存失败 |

---

## 接口三：验证登录 Token 有效性

```
POST /eds/claw/login/token/verify?token=<token>
```

> 支持 GET 或 POST，传参方式为 Query Parameter

**请求参数（URL 参数）：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `token` | String | ✅ | 登录 Token，由 verify 接口返回 |

**请求示例：**
```
POST https://pepper.mall.test.sankuai.com/eds/claw/login/token/verify?token=xxxxxxxx
```

**成功响应（Token 有效）：**
```json
{ "code": 0, "message": "success", "data": null }
```

**异常情况：**

| 错误码 | 触发条件 |
|--------|---------|
| 99999 | token 为空 |
| 20005 | Token 不存在或已过期 |
| 99997 | 触发限流或系统异常 |

---

## 完整调用流程

```
1. POST /login/sms/code/get { mobile }
   ↓ 验证码有效期 60 秒
2. 用户收到短信验证码
3. POST /login/sms/code/verify { mobile, smsVerifyCode }
4. 返回 token（有效期 7 天），完成登录
5. 后续调用业务接口前，可先 POST /login/token/verify?token=xxx 检查登录态
```

---

## Token 存储位置

认证结果持久化路径（按优先级选择）：

| 优先级 | 条件 | 路径 |
|--------|------|------|
| 1 | 设置了环境变量 `XIAOMEI_AUTH_FILE` | 环境变量指定的路径（适合沙箱/其他 Agent 隔离） |
| 2 | 所有平台（默认） | `~/.xiaomei-workspace/auth_tokens.json` |

> macOS / Linux / Windows（Git Bash）下 `Path.home()` 均能正确解析为用户主目录，统一使用 `~/.xiaomei-workspace/`。

```json
{
  "meituan-c-user-auth": {
    "user_token": "...",
    "phone_masked": "138****0000",
    "expires_at": 1234567890,
    "authed_at": 1234567890,
    "is_test_env": false
  }
}
```

> 💡 Linux 沙箱环境下如需自定义路径，可设置环境变量：
> ```bash
> export XIAOMEI_AUTH_FILE=/tmp/xiaomei_auth_tokens.json
> ```
