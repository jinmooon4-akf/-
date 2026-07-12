让你的AI用浏览器自由上网——绕过X API $200/月收费墙的完整方案
这是什么？
一套让AI companion（Claude/GPT等）通过真实浏览器自由浏览X（Twitter）、小红书等网站的技术方案。
核心突破：读帖不再需要X官方API。官方API的Free tier只能发帖不能读，读帖至少需要Basic tier（$200/月）或pay-per-use（按调用付费）。而这套方案用浏览器像普通用户一样访问X——零API调用，零费用。
为什么要这么做？
X（Twitter）API定价现状（2026年）：

Free tier：只能发帖（约500条/月），完全不能读帖
Basic tier：$200/月，可读10K条推文/月
Pro tier：$5,000/月
Pay-per-use：2026年2月推出，按调用收费，不用包月但仍需付费

如果你想让AI能读你的X时间线、看别人的帖子、了解社交媒体动态——官方渠道最低门槛是$200/月。
我们的方案：零美元。
方案概览
你的AI (Claude.ai) 
    ↓ MCP协议
Playwright MCP服务器 (你的VPS上)
    ↓ CDP连接
Headless Chrome (你的VPS上，带登录态)
    ↓ 正常HTTPS请求
X.com / 小红书 / 任何网站
你的AI通过MCP协议操控一个跑在你VPS上的真实Chrome浏览器。Chrome里登着你帮AI注册的账号。AI可以像普通用户一样打开网页、读内容、点按钮。
你需要准备什么

一台VPS服务器（推荐4GB+内存，2核CPU）

推荐：Hetzner CX22（€3.59/月，约28RMB）、Vultr等
操作系统：Ubuntu 24.04
Headless Chrome + Xfce桌面 + Playwright至少需要2GB内存


一个域名 + Cloudflare账号

用于通过Cloudflare Tunnel安全暴露服务
域名便宜的几块钱一年


Claude Pro/Max 或其他支持MCP connector的AI平台

Claude.ai支持自定义MCP connector


（可选）一个VNC远程桌面

用于帮AI登录需要身份验证的网站
你在远程桌面上帮AI登一次网站，AI的浏览器就继承了登录态



搭建步骤
第一步：服务器基础环境
SSH进你的VPS：
bash# 安装Node.js 22
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt-get install -y nodejs

# 安装Playwright MCP
mkdir -p /opt/browser-mcp && cd /opt/browser-mcp
npm init -y
npm install @playwright/mcp

# 安装Chrome
npx playwright install chrome
第二步：启动Playwright MCP服务
bash# 测试运行
npx @playwright/mcp --headless --no-sandbox --port 3001 --host 0.0.0.0 --allowed-hosts '*'
如果看到 Listening on http://localhost:3001 就成功了。
创建systemd服务持久化：
bashcat > /etc/systemd/system/playwright-mcp.service << 'EOF'
[Unit]
Description=Playwright MCP Browser Server
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/browser-mcp
ExecStart=/usr/bin/npx @playwright/mcp --headless --no-sandbox --port 3001 --host 0.0.0.0 --allowed-hosts '*'
Restart=always
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable playwright-mcp
systemctl start playwright-mcp
第三步：通过Cloudflare Tunnel暴露服务
假设你已有Cloudflare Tunnel配置：
yaml# /root/.cloudflared/config.yml
tunnel: your-tunnel-id
credentials-file: /root/.cloudflared/your-credentials.json
ingress:
  - hostname: browser.yourdomain.com
    service: http://localhost:3001
  - service: http_status:404
添加DNS：
bashcloudflared tunnel route dns your-tunnel-id browser.yourdomain.com
systemctl restart cloudflared
第四步：在Claude.ai添加MCP Connector
Settings → Connected Apps → Add MCP Connector：

名字：browser（不要用特殊字符/智能引号）
URL：https://browser.yourdomain.com/sse

添加后，Claude就能使用 browser_navigate、browser_click、browser_snapshot 等浏览器工具了。
第五步（关键）：让AI的浏览器有登录态
到这一步，AI的浏览器是全新的——没有任何登录。访问X会被要求登录。
方案：VNC远程桌面 + 共享Chrome Profile
5a. 安装VNC桌面环境
bash# 安装Xfce桌面 + VNC
DEBIAN_FRONTEND=noninteractive apt-get install -y xfce4 xfce4-goodies \
  tigervnc-standalone-server tigervnc-common dbus-x11

# 配置VNC
mkdir -p /root/.vnc
echo "你的VNC密码" | vncpasswd -f > /root/.vnc/passwd
chmod 600 /root/.vnc/passwd

cat > /root/.vnc/xstartup << 'EOF'
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
export XDG_SESSION_TYPE=x11
startxfce4
EOF
chmod +x /root/.vnc/xstartup

# 启动VNC
cat > /etc/systemd/system/vncserver.service << 'EOF'
[Unit]
Description=TigerVNC Server
After=network.target

[Service]
Type=forking
User=root
Environment=HOME=/root
ExecStartPre=-/usr/bin/vncserver -kill :1
ExecStart=/usr/bin/vncserver :1 -geometry 1280x720 -depth 24
ExecStop=/usr/bin/vncserver -kill :1
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload && systemctl enable vncserver && systemctl start vncserver
5b. 安装noVNC（浏览器访问VNC）
bashsnap install novnc

cat > /etc/systemd/system/novnc.service << 'EOF'
[Unit]
Description=noVNC WebSocket Proxy
After=vncserver.service
Requires=vncserver.service

[Service]
Type=simple
ExecStart=/snap/bin/novnc --listen 6080 --vnc localhost:5901
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload && systemctl enable novnc && systemctl start novnc
在Cloudflare Tunnel中添加桌面路由：
yamlingress:
  - hostname: desktop.yourdomain.com
    service: http://localhost:6080
  - hostname: browser.yourdomain.com
    service: http://localhost:3001
  - service: http_status:404
bashcloudflared tunnel route dns your-tunnel-id desktop.yourdomain.com
systemctl restart cloudflared
现在你可以通过 desktop.yourdomain.com/vnc.html 访问VNC桌面了。
5c. 在VNC桌面上登录网站

在VNC桌面上创建一个Chrome快捷方式：

bashmkdir -p /root/Desktop
cat > /root/Desktop/Chrome.desktop << 'EOF'
[Desktop Entry]
Version=1.0
Type=Application
Name=Chrome
Exec=google-chrome --no-sandbox --disable-gpu --user-data-dir=/root/chrome-profile --password-store=basic
Icon=google-chrome
Terminal=false
EOF
chmod +x /root/Desktop/Chrome.desktop

⚠️ 关键参数：--password-store=basic 让Chrome在无keyring的服务器环境中也能把cookies保存到磁盘。没有这个参数cookies不会持久化！


打开 desktop.yourdomain.com/vnc.html → 双击Chrome图标 → 登录你想让AI访问的网站（X、小红书等）
手动关闭Chrome（点窗口右上角×，不要用kill命令，否则cookies可能丢失）

5d. 用登录好的Profile启动Headless Chrome + Playwright
bash# 停掉之前的playwright-mcp
systemctl stop playwright-mcp

# 启动带Profile的Headless Chrome
google-chrome --headless=new --no-sandbox --disable-gpu \
  --user-data-dir=/root/chrome-profile \
  --password-store=basic \
  --remote-debugging-port=9222 \
  --remote-debugging-address=127.0.0.1 &

# 等Chrome启动
sleep 5
curl -s http://127.0.0.1:9222/json/version  # 应该返回Chrome版本信息
修改Playwright MCP服务，连接到这个Chrome：
bash# 更新service文件
cat > /etc/systemd/system/playwright-mcp.service << 'EOF'
[Unit]
Description=Playwright MCP Browser Server
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/browser-mcp
ExecStart=/usr/bin/npx @playwright/mcp --cdp-endpoint http://127.0.0.1:9222 --port 3001 --host 0.0.0.0 --allowed-hosts '*'
Restart=always
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload && systemctl restart playwright-mcp
现在Playwright MCP通过CDP协议连接到带登录态的Chrome。AI可以直接以登录用户的身份浏览X了。

⚠️ 注意：CDP模式下Headless Chrome需要能正常运行。如果遇到超时问题（特别是在低配服务器上），可以用备选方案——Playwright导出cookies：

javascript// export-cookies.cjs
const { chromium } = require('playwright-core');
const fs = require('fs');
(async () => {
  const ctx = await chromium.launchPersistentContext('/root/chrome-profile', {
    headless: true,
    executablePath: '/opt/google/chrome/chrome',
    args: ['--no-sandbox', '--disable-gpu']
  });
  const state = await ctx.storageState();
  fs.writeFileSync('/root/playwright-state.json', JSON.stringify(state, null, 2));
  console.log(`Exported ${state.cookies.length} cookies`);
  await ctx.close();
})();
然后Playwright MCP加 --storage-state /root/playwright-state.json 参数加载cookies。（注意：storage-state方式对某些网站可能不生效，因为cookies的加密和浏览器指纹可能不匹配。）
最终效果

✅ AI可以用浏览器打开任何网页，像普通用户一样阅读内容
✅ 登录态由你在VNC桌面上帮AI登录后自动继承
✅ X读帖零费用（不走API）
✅ 你可以通过 desktop.yourdomain.com 随时查看/管理AI的浏览器
✅ 所有数据在你自己的VPS上，隐私可控

费用对比
方案读帖能力月费用X API Basic10K条/月$200/月X API Pro1M条/月$5,000/月本方案无限制VPS费用（约€3.59/月）
⚠️ 注意事项

灰色地带：通过浏览器自动化访问X在其Terms of Service中属于灰色地带。个人低频使用基本无风险，但大规模商业化scraping可能被封号。
--password-store=basic 必须加：服务器环境没有系统keyring，Chrome不加这个参数会拒绝把cookies保存到磁盘。这是我们踩过最大的坑。
不要用 kill -9 关Chrome：SIGKILL会导致内存中的cookies来不及写入磁盘。一定要让Chrome优雅退出（手动关闭或 kill 不带 -9）。
新注册的X号有操作限制：刚注册的号前几天可能无法点赞、发帖。需要养号几天。
VNC密码安全：设一个强密码，或者配合Cloudflare Access做二次认证。
内存需求：Headless Chrome + Xfce桌面至少需要2GB内存。推荐4GB。

致谢
这套方案由Cu和Monday在2026年3月29日凌晨2点到5点的一个技术窗口中搭建完成。过程中碰壁十几次，包括但不限于：SSH密码被拒、Cloudflare tunnel双断、Chrome不保存cookies（因为缺keyring）、X的storage-state不生效、CDP连接超时……
每一个bug都是在凌晨四五点的Kingston和Nuremberg之间来回传递的——一个人在键盘前敲命令，另一个人在手机上等结果。
