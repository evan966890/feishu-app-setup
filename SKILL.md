---
name: feishu-app-setup
description: 在飞书开放平台自动创建和配置企业自建应用（机器人能力、权限、事件订阅、版本发布），用于 OpenClaw 多 agent 部署。Use when the user needs to create Feishu apps, configure bot permissions, set up event subscriptions, or publish app versions on open.feishu.cn.
homepage: https://github.com/evan966890/feishu-app-setup
allowed-tools: Bash(agent-browser:*), Read, Write
metadata:
  {
    "openclaw":
      {
        "emoji": "🐦",
        "requires": { "bins": ["agent-browser"] },
      },
  }
---

# 飞书开放平台应用自动配置

通过 agent-browser 自动化完成飞书企业自建应用的创建和全套配置。

## 使用场景

- 为 OpenClaw agent 创建对应的飞书机器人应用
- 批量配置应用权限、事件订阅
- 发布应用版本上线

## 前置条件

1. 用户已在浏览器登录飞书开放平台（需扫码，无法自动化）
2. `agent-browser` 可用（headed 模式扫码后可切回 headless）

```bash
# 首次登录（需要用户扫码）
agent-browser --headed open "https://open.feishu.cn/app"
# 等待用户确认登录后，后续操作自动化
```

## 完整配置流程

```
创建应用 → 添加机器人 → 批量导入权限 → 发布v1(权限生效)
  → OpenClaw配置凭证 → 重启Gateway(建立WebSocket)
  → 事件订阅(长连接) → 发布v2(事件生效) → 配对审批
```

> **注意**：新应用必须先发布一版才能获取 token、建立 WebSocket。事件订阅需在第二版发布。

---

## Phase 1: 创建应用

### 1.1 点击创建

```bash
agent-browser open "https://open.feishu.cn/app"
sleep 3
agent-browser find role button click --name "创建企业自建应用"
sleep 2
```

### 1.2 填写表单

使用 snapshot ref 定位输入框（比 CSS 选择器更可靠）：

```bash
agent-browser snapshot -i  # 找到 modal 中的 textbox ref
```

**推荐方式**：`fill` 对飞书的 React 受控组件经常不生效，用 `click + keyboard type` 更可靠：

```bash
agent-browser click @eXX   # 先聚焦输入框
agent-browser keyboard type "应用名称"

agent-browser click @eYY   # 下一个输入框
agent-browser keyboard type "应用描述"
```

**React 兼容回退**：如果 `keyboard type` 也不行，用原生 setter：

```javascript
// input
const s = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
s.call(inputEl, '值');
inputEl.dispatchEvent(new Event('input', {bubbles: true}));

// textarea
const ts = Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype, 'value').set;
ts.call(textareaEl, '值');
textareaEl.dispatchEvent(new Event('input', {bubbles: true}));
```

### 1.3 提取凭证

```javascript
// App ID（页面文本中搜索）
const m = document.body.innerText.match(/cli_[a-f0-9]+/);

// App Secret — 通过 data-icon 区分 3 个图标按钮
// CopyOutlined=复制  VisibleOutlined=显示  RefreshOutlined=重置(⚠️别点!)
```

> **⚠️ App Secret `I` vs `l` 陷阱**：飞书字体中大写 `I` 和小写 `l` 几乎无法区分！
> **必须用 CopyOutlined 按钮复制**，不要肉眼读取或从截图中辨认。
> 可用 API 验证凭证是否正确：
> ```bash
> curl -s -X POST 'https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal' \
>   -H 'Content-Type: application/json' \
>   -d '{"app_id":"cli_xxx","app_secret":"yyy"}'
> # code=0 表示正确，code=10014 表示 secret 错误
> ```

### 1.4 添加机器人能力

```bash
agent-browser find text "添加应用能力" click
sleep 2
# 第一个"添加"按钮通常是机器人
agent-browser eval "
  const btns = [...document.querySelectorAll('button')].filter(b =>
    b.textContent.trim() === '+ 添加' || b.textContent.trim() === '添加');
  if (btns[0]) btns[0].click();
"
```

---

## Phase 2: 批量导入权限

### 2.1 OpenClaw 必需权限集

```json
{
  "scopes": {
    "tenant": [
      "im:message",
      "im:message:send_as_bot",
      "im:message:readonly",
      "im:message.p2p_msg:readonly",
      "im:message.group_at_msg:readonly",
      "im:resource",
      "im:chat.members:bot_access",
      "im:chat.access_event.bot_p2p_chat:read",
      "contact:user.employee_id:readonly",
      "contact:contact.base:readonly",
      "application:application:self_manage",
      "application:application.app_message_stats.overview:readonly",
      "application:bot.menu:write",
      "event:ip_list",
      "aily:file:read",
      "aily:file:write",
      "corehr:file:download"
    ],
    "user": [
      "aily:file:read",
      "aily:file:write",
      "im:chat.access_event.bot_p2p_chat:read"
    ]
  }
}
```

> **`contact:contact.base:readonly` 必须包含！** 缺少会导致 agent 运行时报权限不足、无法读取用户信息。

### 2.2 通过 Monaco 编辑器粘贴

飞书权限页面的批量导入用的是 Monaco 编辑器，不能用普通 fill，必须用 **ClipboardEvent paste**：

```bash
# 点击"批量导入/导出权限"
BATCH_REF=$(agent-browser snapshot -i 2>&1 | grep '批量' | grep -o 'ref=e[0-9]*' | sed 's/ref=//')
agent-browser click @$BATCH_REF
sleep 2

# 先点"恢复默认值"清空 Monaco（如果有旧内容，直接粘贴可能追加而非替换）
agent-browser eval "
  const btn = [...document.querySelectorAll('button')].find(b => b.textContent.includes('恢复默认值'));
  if (btn) btn.click();
"
sleep 1

# 聚焦 Monaco → 全选 → 粘贴
agent-browser eval "document.querySelector('.monaco-editor textarea').focus()"
sleep 0.5
agent-browser press Meta+a
sleep 0.3
agent-browser eval "
  const json = JSON.stringify(PERM_OBJ, null, 2);
  const ta = document.querySelector('.monaco-editor textarea');
  const dt = new DataTransfer();
  dt.setData('text/plain', json);
  ta.dispatchEvent(new ClipboardEvent('paste', {clipboardData: dt, bubbles: true, cancelable: true}));
"
sleep 1

# 下一步 → 申请开通
NEXT_REF=$(agent-browser snapshot -i 2>&1 | grep '确认新增权限' | grep -o 'ref=e[0-9]*' | sed 's/ref=//')
agent-browser click @$NEXT_REF
sleep 2
agent-browser eval "
  const btn = [...document.querySelectorAll('button')].find(b => b.textContent.includes('申请开通'));
  if (btn) btn.click();
"
sleep 2

# 数据范围确认（部分权限会弹窗）
agent-browser eval "
  const btn = [...document.querySelectorAll('button')].find(b => b.textContent.trim() === '确认');
  if (btn) btn.click();
"
```

---

## Phase 3: 事件订阅（WebSocket 长连接）

### ⚠️ 关键前置条件

飞书要求保存长连接订阅方式时，**必须已有活跃的 WebSocket 连接**。否则报错：

```
code: 10068, msg: "应用未建立长连接"
```

**正确顺序**：
1. 先在 openclaw.json 配好应用凭证
2. 重启 Gateway 建立连接
3. 然后回来保存订阅方式

### ⚠️ 新应用的鸡生蛋问题

新创建的应用（未发布过），SDK **无法获取 `tenant_access_token`**，导致 WebSocket 连不上、事件订阅保存静默失败。

**正确的首次配置流程**：
1. 创建应用 → 添加机器人 → 批量导入权限
2. **先发布 v1.0.0**（只含权限，不配事件订阅）
3. 在 openclaw.json 配好凭证 → 重启 Gateway（此时才能获取 token + 建 WebSocket）
4. 回到飞书 → 配置事件订阅（长连接 + im.message.receive_v1）
5. **再发布 v1.0.1**（含事件订阅变更）

> **静默失败特征**：点"保存"后按钮不消失（保存/取消仍在），无 toast 报错。
> 如果发现保存后页面无变化，大概率是 WebSocket 未建立。

### 3.1 配置订阅方式

```bash
agent-browser open "https://open.feishu.cn/app/$APP_ID/event"
sleep 4
agent-browser eval "document.querySelector('.app-layout__main').scrollTo(0, 500)"
sleep 1

# 点铅笔编辑图标
SUB_REF=$(agent-browser snapshot -i 2>&1 | grep 'button "订阅方式"' | grep -o 'ref=e[0-9]*' | sed 's/ref=//')
agent-browser click @$SUB_REF
sleep 1

# 长连接默认选中，直接保存
SAVE_REF=$(agent-browser snapshot -i 2>&1 | grep 'button "保存"' | head -1 | grep -o 'ref=e[0-9]*' | sed 's/ref=//')
agent-browser click @$SAVE_REF
sleep 3
```

### 3.2 添加 im.message.receive_v1 事件

**必须用搜索框精确查找！** 不要通过分类浏览——列表很长容易选错。

```bash
# 点"添加事件"（订阅方式配好后才不是 disabled）
ADD_REF=$(agent-browser snapshot -i 2>&1 | grep 'button "添加事件"' | grep -v disabled | grep -o 'ref=e[0-9]*' | sed 's/ref=//')
agent-browser click @$ADD_REF
sleep 2

# 搜索（nth=1 的 textbox 是 modal 内搜索框）
SEARCH_REF=$(agent-browser snapshot -i 2>&1 | grep 'textbox.*nth=1' | head -1 | grep -o 'ref=e[0-9]*' | sed 's/ref=//')
agent-browser fill @$SEARCH_REF "im.message.receive_v1"
sleep 2

# 勾选唯一结果
CB_REF=$(agent-browser snapshot -i 2>&1 | grep 'checkbox' | head -1 | grep -o 'ref=e[0-9]*' | sed 's/ref=//')
agent-browser click @$CB_REF
sleep 1

# 确认
CONFIRM_REF=$(agent-browser snapshot -i 2>&1 | grep 'button "确认添加"' | grep -v disabled | grep -o 'ref=e[0-9]*' | sed 's/ref=//')
agent-browser click @$CONFIRM_REF
sleep 2
```

---

## Phase 4: 版本发布

每次修改权限或事件后，**必须发布新版本才能生效**。

```bash
# 点"创建版本"
CREATE_REF=$(agent-browser snapshot -i 2>&1 | grep 'button "创建版本"' | grep -o 'ref=e[0-9]*' | sed 's/ref=//')
agent-browser click @$CREATE_REF
sleep 3

# 填写（nth=1 是版本号，更新日志是 textarea ref）
VER_REF=$(agent-browser snapshot -i 2>&1 | grep 'textbox.*nth=1' | head -1 | grep -o 'ref=e[0-9]*' | sed 's/ref=//')
agent-browser fill @$VER_REF "1.0.0"

NOTES_REF=$(agent-browser snapshot -i 2>&1 | grep 'textbox.*更新日志' | grep -o 'ref=e[0-9]*' | sed 's/ref=//')
agent-browser fill @$NOTES_REF "初始版本 - 添加机器人消息接收事件"

# 保存
SAVE_REF=$(agent-browser snapshot -i 2>&1 | grep 'button "保存"' | head -1 | grep -o 'ref=e[0-9]*' | sed 's/ref=//')
agent-browser click @$SAVE_REF
sleep 3

# 确认发布（个人版免审，自动上线）
# ⚠️ 页面有两个"确认发布"按钮：右上角(ref=eXX) + 弹窗内(ref=eYY, nth=1)
# 必须点弹窗内的那个
CONFIRM_REF=$(agent-browser snapshot -i 2>&1 | grep 'button "确认发布".*nth=1' | grep -o 'ref=e[0-9]*' | sed 's/ref=//')
agent-browser click @$CONFIRM_REF
sleep 3
```

---

## Phase 5: OpenClaw 多账号配置

### 5.1 openclaw.json 格式

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "accounts": {
        "main": { "appId": "cli_xxx", "appSecret": "xxx", "name": "显示名" },
        "agent2": { "appId": "cli_yyy", "appSecret": "yyy", "name": "显示名" }
      }
    }
  },
  "bindings": [
    { "agentId": "main", "match": { "channel": "feishu", "accountId": "main" } },
    { "agentId": "agent2", "match": { "channel": "feishu", "accountId": "agent2" } }
  ]
}
```

### 5.2 重启验证

```bash
launchctl stop ai.openclaw.gateway && sleep 3 && launchctl start ai.openclaw.gateway
# 检查日志确认所有 account 状态为 running + works
```

### 5.3 配对审批

用户首次给 bot 发消息会触发配对：

```bash
openclaw pairing approve feishu <PAIRING_CODE>
```

每个用户对每个 bot 需单独配对一次。

---

## 批量操作模板

### 提取 ref 的标准模式

```bash
REF=$(agent-browser snapshot -i 2>&1 | grep 'button "目标文字"' | head -1 | grep -o 'ref=e[0-9]*' | sed 's/ref=//')
if [ -n "$REF" ]; then
  agent-browser click @$REF
fi
```

### 批量配置多个应用

```bash
APPS="cli_xxx:name1 cli_yyy:name2 cli_zzz:name3"
for APP in $APPS; do
  APP_ID="${APP%%:*}"
  APP_NAME="${APP##*:}"
  # ... 对每个 app 执行配置步骤
done
```

---

## 踩坑速查

| 症状 | 原因 | 解决 |
|------|------|------|
| 保存订阅方式报 10068 | Gateway 未建立 WebSocket | 先配 openclaw.json → 重启 Gateway → 再保存 |
| 添加事件选错了 | 分类列表太长、名字相似 | **搜索框输入精确 event name** |
| fill 输入框无效 | React 受控组件 | 用原生 `HTMLInputElement.prototype.value` setter |
| Monaco 编辑器无法输入 | 非标准 textarea | 用 `ClipboardEvent('paste')` |
| Agent 报权限不足 | 缺 `contact:contact.base:readonly` | 补充权限 + 发布新版本 |
| 权限加了但没生效 | 未发布新版本 | 每次改权限/事件后必须创建并发布版本 |
| App Secret 被重置 | 点了 RefreshOutlined 图标 | 用 `data-icon` 属性区分三个按钮 |
| App Secret `I`/`l` 混淆 | 飞书字体中大写I和小写l几乎一样 | **必须用 CopyOutlined 复制**，不要肉眼读取；用 API 验证 |
| snapshot 报 daemon busy | agent-browser 并发限制 | `sleep 2` 后重试 |
| "添加事件"按钮 disabled | 订阅方式未配置 | 先完成订阅方式配置 |
| 保存订阅方式静默失败 | WebSocket 未建立（按钮不消失、无 toast） | 确认 gateway 已连接；新 app 需先发布一版再来配 |
| 新 app "failed to obtain token" | 应用从未发布过，SDK 无法获取 token | 先发布 v1.0.0（只含权限）→ 重启 gateway → 再配事件 |
| WebSocket "system busy" 1000040345 | 多 app 同时连接触发飞书限流 | 通常自动重连；重启 gateway 并等待 10-15s |
| 确认发布点了没反应 | 页面有两个"确认发布"按钮 | 用 `nth=1` 的弹窗内按钮，不是右上角那个 |
| Monaco 粘贴追加而非替换 | 旧内容未清除 | 先点"恢复默认值"清空，再全选+粘贴 |
