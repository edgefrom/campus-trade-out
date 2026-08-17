# 校园闲置网页版

微信小程序的移动优先网页版，包含商品浏览、搜索分类、发布编辑、收藏评论、站内聊天、个人中心、安全指南，以及邮箱验证码注册和邮箱密码登录。

## 本地运行

要求 Node.js 22+ 与 MySQL 8.0+。先创建独立数据库和最小权限账号，再复制环境变量：

```bash
cp .env.example .env
npm install
npm run dev
```

前端默认 `http://localhost:5173`，API 默认 `http://localhost:8787`。服务启动时只在 `MYSQL_DATABASE` 指向的库内创建本站数据表，不会创建或删除数据库。

## 外部服务

- 邮箱注册：配置 QQ 邮箱 SMTP 的 `EMAIL_HOST_USER`、`EMAIL_HOST_PASSWORD`（授权码）和 `DEFAULT_FROM_EMAIL`，注册时发送 6 位、10 分钟有效的验证码。
- 商品图片：配置腾讯云 COS 的 `COS_SECRET_ID`、`COS_SECRET_KEY`、`COS_BUCKET`、`COS_REGION`。若桶为公有读，可把自定义域名写入 `COS_PUBLIC_BASE_URL`；否则留空，由站内媒体地址动态生成短期签名。建议使用只允许指定桶路径的最小权限子账号。

## 构建与检查

```bash
npm test
npm run build
NODE_ENV=production npm start
```

生产环境由 Express 托管 `dist/`，nginx 只需反向代理到独立 Node 端口。请启用 HTTPS，并让 `APP_ORIGIN` 与公开站点地址完全一致。

部署到域名子路径时，例如 `/campus-trade/`，构建前设置 `VITE_BASE_PATH=/campus-trade/`，并让 nginx 使用带尾斜杠的 `proxy_pass http://127.0.0.1:8787/` 去除代理前缀。

## 手机端验收

样式从 320px 起采用移动优先布局，重点检查 360×800、390×844、430×932；700px 起切换平板/桌面增强布局。手机端固定底部导航、聊天输入区和所有主要触控目标均至少约 44px，并适配安全区。

## 生产部署参考

- 访问地址：`https://<你的域名>/campus-trade/`
- 应用目录：示例 `/www/wwwroot/campus-market-web`
- 服务：`campus-market-web.service`，仅监听服务器 8787 端口
- nginx：现有 HTTPS 站点通过独立 `/campus-trade/` location 反向代理，配置片段见 `deploy/nginx-campus-market.conf`
- 数据库：独立 MySQL 数据库（如 `campus_market`）
