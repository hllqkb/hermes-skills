---
name: xiaohongshu-login
description: 通过 chrome-cdp 直连本地 Chrome 打开小红书，利用已有 cookie 免登录。绕过 Browserbase/CDP headless 的 IP 风控和 headless 检测。
tags: [xiaohongshu, cdp, chrome, login, social-media]
---

# 小红书登录（chrome-cdp 方式）

## 场景
需要在浏览器中打开小红书并保持登录状态。

## 为什么不用其他方式
- **Browserbase**：代理 IP 被小红书风控（error_code=300012）
- **Hermes CDP**：启动的是 headless Chrome，User-Agent 暴露为 HeadlessChrome，被检测拦截
- **Midscene**：需要管理员权限，非 admin 有 UIPI 限制无法操控键盘鼠标

## 方案
使用 [chrome-cdp](https://github.com/pasky/chrome-cdp-skill) skill，直连用户本地 Chrome 现有会话，利用已有 cookie 免登录。

## 前提
- 用户需在本地 Chrome 中至少登录过一次小红书（cookie 已保存）
- Chrome 远程调试已开启（chrome://inspect/#remote-debugging 开关打开）
- Node.js 22+ 已安装

## 步骤

### 1. 安装 chrome-cdp skill
```bash
git clone https://github.com/pasky/chrome-cdp-skill.git /tmp/chrome-cdp-skill
cp -r /tmp/chrome-cdp-skill/skills/chrome-cdp "$LOCALAPPDATA/hermes/skills/devops/chrome-cdp"
```

### 2. 确认 Chrome 远程调试已开启
```bash
cat "$LOCALAPPDATA/Google/Chrome/User Data/DevToolsActivePort"
```
如果有输出（端口号 + WebSocket 路径），说明已开启。

### 3. 打开小红书
```bash
# 列出当前标签页
node "$LOCALAPPDATA/hermes/skills/devops/chrome-cdp/scripts/cdp.mjs" list

# 打开新标签并导航到小红书
node "$LOCALAPPDATA/hermes/skills/devops/chrome-cdp/scripts/cdp.mjs" open https://www.xiaohongshu.com
```

### 4. 首次访问需点击 Allow
Chrome 会弹出 "Allow debugging?" 提示，需手动点击 Allow。之后 daemon 保持连接，20 分钟无操作自动退出。

### 5. 常用操作
```bash
TARGET="A81CD583"  # 从 list 输出获取

# 截图
node cdp.mjs shot $TARGET

# 获取页面结构（推荐，比 html 轻量）
node cdp.mjs snap $TARGET

# 点击元素
node cdp.mjs eval $TARGET "document.querySelectorAll('a').forEach(a => { if(a.textContent.trim() === '我') a.click() })"

# 导航到其他页面
node cdp.mjs nav $TARGET https://www.xiaohongshu.com/explore
```

## 注意事项
- chrome-cdp 使用 CSS 选择器，不能用文本选择器
- 要用 eval 执行 JS 来按文本查找并点击元素
- 截图时 DPR=1.5，CSS 像素 = 截图像素 / 1.5
- Daemon 20 分钟无操作自动退出，下次访问重新弹 Allow 提示
- 脚本实际路径：`$LOCALAPPDATA/hermes/skills/devops/chrome-cdp/scripts/cdp.mjs`
