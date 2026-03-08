# feishu-app-setup — 飞书开放平台应用自动化配置

> 用 agent-browser 自动完成飞书企业自建应用的创建、权限配置、事件订阅、改名和版本发布。

在飞书开放平台上配置一个 OpenClaw bot 应用需要大量重复的点击操作。这个技能把整个流程自动化了：创建应用、添加机器人能力、批量导入权限、配置事件订阅、修改应用名称、发布版本——全部通过 `agent-browser` 在浏览器中自动完成。

## 前置条件

- 已安装 `agent-browser` 技能
- 已登录飞书开放平台（https://open.feishu.cn）
- OpenClaw Gateway 正在运行（事件订阅需要活跃的 WebSocket 连接）
- **`.feishu.cn` 和 `.larksuite.com` 必须在 NO_PROXY 中**（见下方「代理配置」）

## 代理配置（关键！）

飞书是国内服务，**不能走代理**。如果 Gateway plist 配置了 HTTP_PROXY/HTTPS_PROXY，必须把飞书域名加入 NO_PROXY：

```xml
<key>NO_PROXY</key>
<string>...,.feishu.cn,.larksuite.com</string>
<key>no_proxy</key>
<string>...,.feishu.cn,.larksuite.com</string>
```

**不加会怎样**：xray 代理把 HTTPS 请求当 HTTP 转发，飞书返回 `400 The plain HTTP request was sent to HTTPS port`。Gateway 的 WebSocket 连不上，导致：
1. 飞书控制台保存订阅方式失败（检测不到活跃连接）
2. 「添加事件」按钮始终 disabled
3. bot 聊天窗口没有消息输入框

修改 plist 后必须 bootout + bootstrap 重启 Gateway（不能只 stop/start，因为 plist env 不会重新加载）：

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.gateway.plist
sleep 3
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

## 连接浏览器

### 方式一：连接已有浏览器（推荐）

OpenClaw Gateway 运行时通常已启动 Chrome debug 实例。直接连接已登录的浏览器，无需重新登录：

```bash
# 先读取当前 CDP 端口（可能变化）
CDP_PORT=$(cat ~/.chrome-crawl/cdp-port)
agent-browser connect $CDP_PORT
agent-browser open "https://open.feishu.cn/app"
```

**注意**：CDP 端口不固定，Chrome 重启后可能从 9222 变为 9223。始终从 `~/.chrome-crawl/cdp-port` 读取。

**优势**：无需扫码登录，直接可用。适合批量操作。

### 方式二：启动新浏览器

```bash
agent-browser --headed open "https://open.feishu.cn/app"
# 用户扫码登录后，后续操作均可自动化
```

## 完整配置流程

```
代理检查 → 创建应用 → 添加机器人能力 → 权限导入 → OpenClaw 配置 → 重启 Gateway → 事件订阅 → 改名（可选）→ 版本发布 → 配对审批
```

**注意顺序**：OpenClaw 配置和 Gateway 重启必须在事件订阅之前，否则飞书检测不到活跃 WebSocket，保存订阅方式会失败。

---

## Step 1: 创建应用

```bash
agent-browser find role button click --name "创建企业自建应用"
sleep 2
```

### 填写表单（React 受控组件）

飞书开放平台使用 React。`agent-browser fill @ref` 通常可以正常工作，但在 fill 失败时需要回退到原生 setter：

```javascript
// React 受控 input 的可靠填写方式
const s = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
s.call(inputElement, '目标值');
inputElement.dispatchEvent(new Event('input', {bubbles: true}));
```

### 提取凭证

```javascript
// App ID
const m = document.body.innerText.match(/cli_[a-f0-9]+/);
// App Secret — 先点眼睛图标显示
const eyeBtn = document.querySelectorAll('[data-icon="VisibleOutlined"]');
```

**注意**: App Secret 旁边有 3 个图标按钮：复制(CopyOutlined)、显示(VisibleOutlined)、重置(RefreshOutlined)。**千万别点重置**。

## Step 2: 添加机器人能力

```bash
agent-browser find text "添加应用能力" click
sleep 2
agent-browser eval "
  const btns = [...document.querySelectorAll('button')].filter(b => b.textContent.trim() === '+ 添加' || b.textContent.trim() === '添加');
  if (btns[0]) btns[0].click();
"
```

## Step 3: 权限批量导入

### 推荐权限集（OpenClaw 完整权限）

```json
{
  "scopes": {
    "tenant": [
      "im:message",
      "im:message:send_as_bot",
      "im:message:readonly",
      "im:message.p2p_msg:readonly",
      "im:message.group_at_msg:readonly",
      "im:message.group_msg",
      "im:resource",
      "im:chat",
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
      "corehr:file:download",
      "docs:document.content:read",
      "docx:document",
      "drive:drive",
      "cardkit:card:write",
      "sheets:spreadsheet",
      "wiki:wiki:readonly"
    ],
    "user": [
      "aily:file:read",
      "aily:file:write",
      "im:chat.access_event.bot_p2p_chat:read"
    ]
  }
}
```

**必须包含的权限**：
- `contact:contact.base:readonly` — 否则 agent 无法读取用户信息
- `im:message:send_as_bot` — 否则 bot 无法发消息
- `im:chat` — 否则无法获取会话信息
- `docx:document` + `drive:drive` — 文档和云盘操作（pipeline 同步需要）

### 操作步骤

```bash
# 点击"批量导入/导出权限"
agent-browser snapshot -i | grep '批量'
agent-browser click @eXX
sleep 2

# Monaco 编辑器填写：全选 → 粘贴
agent-browser eval "document.querySelector('.monaco-editor textarea').focus()"
agent-browser press Meta+a
sleep 0.3

# 用 ClipboardEvent 粘贴（Monaco 不支持普通 fill）
agent-browser eval "
  const json = JSON.stringify(PERM_JSON, null, 2);
  const ta = document.querySelector('.monaco-editor textarea');
  const dt = new DataTransfer();
  dt.setData('text/plain', json);
  ta.dispatchEvent(new ClipboardEvent('paste', {clipboardData: dt, bubbles: true, cancelable: true}));
"
sleep 1

# 点击"下一步，确认新增权限" → "申请开通"
```

部分权限（如通讯录相关）会弹出"数据范围"确认对话框，需要额外点击"确认"。

## Step 4: 事件订阅（长连接 WebSocket）

### 为什么事件订阅如此重要

**没有 `im.message.receive_v1` 事件订阅 = 飞书聊天窗口没有消息输入框。** 用户看到的症状是能搜到 bot 但无法发消息。

### 先决条件：三件事必须先完成

飞书有个鸡生蛋的问题：保存长连接订阅方式时，飞书会检查是否已有活跃的 WebSocket 连接。**正确顺序**：

1. **NO_PROXY 加 `.feishu.cn`**（见上方「代理配置」）— 否则 WebSocket 400 错误
2. 在 `openclaw.json` 配置应用凭证（账号名必须叫 `default`）
3. `launchctl bootout + bootstrap` 重启 Gateway，确认日志出现 `[ws] ws client ready`

**验证 WebSocket 已连通**：
```bash
tail -20 ~/.openclaw/logs/gateway.log | grep -i 'ws client ready'
# 必须看到: [ws] ws client ready
```

如果看到 `Request failed with status code 400` 或 `The plain HTTP request was sent to HTTPS port`，说明代理没配对，回去检查 NO_PROXY。

否则会报错：`code: 10068, msg: "应用未建立长连接"`

### 配置订阅方式

```bash
agent-browser open "https://open.feishu.cn/app/$APP_ID/event"
sleep 4

# 点「订阅方式」按钮展开选项
agent-browser snapshot -i | grep '订阅方式'
agent-browser click @eXX
sleep 2

# 长连接默认选中（radio checked），直接点保存
agent-browser snapshot -i | grep '保存'
agent-browser click @eXX   # 点「保存」按钮
sleep 3

# 验证：保存成功后「添加事件」按钮变为可点击（不再 disabled）
agent-browser snapshot -i | grep '添加事件'
# 如果仍显示 [disabled]，说明 WebSocket 未连通，检查 Gateway 日志
```

### 添加事件

```bash
# 点「添加事件」（保存订阅方式后才可点击）
agent-browser snapshot -i | grep '添加事件'
agent-browser click @eXX
sleep 2

# 用搜索框精确查找（避免在分类列表中翻找选错）
agent-browser snapshot -i | grep 'textbox.*nth=1'
agent-browser fill @eXX "im.message.receive_v1"
sleep 2

# 勾选搜索结果（checkbox）
agent-browser snapshot -i | grep 'checkbox'
agent-browser check @eXX
sleep 1

# 确认添加（checkbox 勾选后按钮才可点击）
agent-browser snapshot -i | grep '确认添加'
agent-browser click @eXX
sleep 2

# 验证：页面出现「接收消息」链接 + 「删除事件」按钮
agent-browser snapshot -i | grep '接收消息'
```

## Step 5: 应用改名

应用名称在 **基础信息 → 国际化配置** 中修改：

```bash
agent-browser open "https://open.feishu.cn/app/$APP_ID/baseinfo"
sleep 3

# 滚动到"国际化配置"区域
agent-browser eval "document.querySelector('.app-layout__main').scrollTo(0, 1000)"
sleep 1

# 点击编辑铅笔图标
agent-browser snapshot -i | grep 'EditOutlined\|编辑'
agent-browser click @eXX
sleep 2

# 清除旧名称并填写新名称
agent-browser snapshot -i | grep 'textbox'
agent-browser fill @eXX "新名称"
sleep 1

# 保存
agent-browser find role button click --name "保存"
sleep 2
```

**注意**：改名后必须创建新版本并发布，新名字才会在飞书聊天中生效。

### 批量改名

```bash
APP_IDS=("cli_xxx1" "cli_xxx2" "cli_xxx3")
NAMES=("名字1" "名字2" "名字3")

for i in "${!APP_IDS[@]}"; do
  agent-browser open "https://open.feishu.cn/app/${APP_IDS[$i]}/baseinfo"
  sleep 3
  # 编辑国际化配置 → 填写新名称 → 保存
done
```

## Step 6: 版本发布

添加事件后页面顶部会出现「创建版本」按钮。**必须发布新版本，事件订阅才生效。**

```bash
# 点「创建版本」（页面顶部横幅中的按钮）
agent-browser snapshot -i | grep '创建版本'
agent-browser click @eXX
sleep 3

# 填写版本号（必填，placeholder 显示上一版本号）
agent-browser snapshot -i | grep 'textbox.*nth=1'
agent-browser fill @eXX "1.2.0"

# 填写更新说明
agent-browser snapshot -i | grep '此内容将于'
agent-browser fill @eXX "添加事件订阅：接收消息"

# 保存版本
agent-browser snapshot -i | grep '保存'
agent-browser click @eXX
sleep 3

# 保存后出现「确认发布」按钮（可能有两个：页面上和弹窗中）
# 点弹窗中的那个（nth=1）
agent-browser snapshot -i | grep '确认发布'
agent-browser click @eXX   # 选 nth=1 的按钮
sleep 3

# 验证：页面显示「当前修改均已发布」（绿色勾）+ 版本状态「已发布」
```

### 发布注意事项

- 飞书个人版企业自建应用**免审核**，提交即上线
- **版本号必须手动填写**，不会自动递增
- 发布前可能需要确认**数据权限范围**（如"飞书人事"相关权限需设为"全部"）
- 发布成功后状态显示 **"已发布"** + **"当前修改均已发布"**
- 改名 + 发布通常配合进行，流程：**改名 → 保存 → 创建版本 → 发布**

## Step 7: OpenClaw 配置

**重要**：此步骤必须在事件订阅（Step 4）之前完成，因为 Gateway 需要先建立 WebSocket 连接。

### 单账号配置

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "accounts": {
        "default": {
          "appId": "cli_xxx",
          "appSecret": "xxx",
          "name": "Studio一号"
        }
      }
    }
  }
}
```

**账号名必须叫 `default`**。如果用其他名字（如 `main`、`studio1`），Gateway 会报错：
`accounts.default is missing and no valid account-scoped binding exists`

### 多账号配置

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "accounts": {
        "default": { "appId": "cli_xxx", "appSecret": "xxx", "name": "主助手" },
        "agent2": { "appId": "cli_yyy", "appSecret": "yyy", "name": "助手2" }
      }
    }
  },
  "bindings": [
    { "agentId": "main", "match": { "channel": "feishu", "accountId": "default" } },
    { "agentId": "agent2", "match": { "channel": "feishu", "accountId": "agent2" } }
  ]
}
```

### Gateway plist 环境变量

在 `~/Library/LaunchAgents/ai.openclaw.gateway.plist` 的 `EnvironmentVariables` 中添加：

```xml
<key>FEISHU_APP_ID</key>
<string>cli_xxx</string>
<key>FEISHU_APP_SECRET</key>
<string>xxx</string>
```

### 重启 Gateway

修改 plist 后必须 bootout + bootstrap（不能只 stop/start）：

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.gateway.plist
sleep 3
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

**验证**：
```bash
# 检查 WebSocket 是否连通
tail -30 ~/.openclaw/logs/gateway.log | grep -E 'feishu|ws client'
# 期望看到: [feishu] feishu[default]: WebSocket client started
# 期望看到: [ws] ws client ready
```

## Step 8: 配对审批

用户首次给 bot 发消息时，OpenClaw 返回配对请求。每个用户对每个 bot 需要单独配对一次。

```bash
openclaw pairing approve feishu <配对码>
```

## 操作模式：snapshot + ref

所有操作的核心模式：用 `agent-browser snapshot -i` 获取页面元素 ref，用 `grep` 提取，再用 ref 操作。比 JS 选择器更可靠。

```bash
REF=$(agent-browser snapshot -i 2>&1 | grep 'button "保存"' | head -1 | grep -o 'ref=e[0-9]*' | sed 's/ref=//')
if [ -n "$REF" ]; then
  agent-browser click @$REF
fi
```

## 踩坑记录

| 问题 | 原因 | 解决 |
|------|------|------|
| **WebSocket 400 错误** | `.feishu.cn` 不在 NO_PROXY 中，代理把 HTTPS 当 HTTP 转发 | 在 plist NO_PROXY 和 no_proxy 中加 `.feishu.cn,.larksuite.com`，然后 bootout+bootstrap |
| **bot 聊天没有输入框** | 没有配置 `im.message.receive_v1` 事件订阅 | 完成事件订阅 + 发布新版本 |
| **「添加事件」按钮 disabled** | 订阅方式未保存（WebSocket 未连通） | 先修代理 → 重启 Gateway → 确认 ws client ready → 再保存 |
| 保存订阅方式报 10068 | Gateway 未连接 | 先配 openclaw.json → 加 NO_PROXY → bootout+bootstrap → 再保存 |
| **OpenClaw 账号名报错** | 账号名不叫 `default` | `accounts` 中第一个账号必须叫 `default`，不能用 `main`/`studio1` |
| **plist 修改后 stop/start 无效** | launchctl stop/start 不重新加载 plist | 必须 `bootout` + `bootstrap` |
| 添加事件选错 | 分类列表太长 | 用搜索框精确搜索 `im.message.receive_v1` |
| React input fill 无效 | 受控组件问题 | 用原生 setter + dispatchEvent |
| Monaco 编辑器无法 fill | 非标准 input | 用 ClipboardEvent paste |
| 权限不足报错 | 缺 `contact:contact.base:readonly` | 加到权限 JSON 中 |
| 权限变更后仍报错 | 需要新版本 | 权限变更后必须发布新版本 |
| 点错 App Secret 重置 | 3 个图标难区分 | 用 `data-icon` 属性区分 |
| snapshot ref 失效 | daemon busy | `sleep 2` 后重试 |
| **CDP 端口变化** | Chrome 重启后端口可能变 | 始终从 `~/.chrome-crawl/cdp-port` 读取，不要硬编码 |
| 新浏览器打不开飞书 | 需要扫码登录 | 用 `connect $CDP_PORT` 连接已登录浏览器 |
| 改名后不生效 | 未发布版本 | 改名 → 保存 → 创建版本 → 发布 |
| 发布要求确认数据范围 | 部分权限需选范围 | 如"飞书人事"权限选"全部" |
| agent-browser launch 失败 | 需要 raw mode | 用 `connect $CDP_PORT` 连接已有浏览器 |
| **版本号不自动递增** | 飞书要求手动填写版本号 | 读 placeholder 中的上一版本号，手动递增 |

## 依赖

- `agent-browser` 技能（用于浏览器自动化）
- 飞书企业管理员权限（创建自建应用）
- OpenClaw Gateway（事件订阅需要活跃连接）
