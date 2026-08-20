# 奖励机制部署与使用说明

本文件说明「至纯源石 / 龙门币」双货币手动奖励机制的部署步骤、管理员发放方法、用户侧入口与常见问题。

## 一、功能说明

- 两种虚拟货币：
  - **至纯源石**（内部代码 `originium`）：稀缺货币，奖励开发贡献
  - **龙门币**（内部代码 `lungmen`）：通用货币，用于各类奖励
- 发放方式：仅管理员在服务器上通过命令行脚本手动发放，无网页后台、无自助获取途径
- 用户侧：登录后访问「我的 → 奖励与资产」（路径 `/wallet`），查看余额与入账明细（仅本人可见）
- 数据审计：每次发放写入 `currency_ledger` 流水表（含发放原因、操作者、发放后余额快照），不可篡改

## 二、本次改动文件

| 文件 | 说明 |
|---|---|
| `server/currency.ts` | 新增：货币定义与校验（名称修改只需改这里） |
| `server/grant.ts` | 新增：管理员发放命令行脚本 |
| `server/db.ts` | 新增 `wallets`、`currency_ledger` 两张表（启动时自动创建） |
| `server/index.ts` | 新增接口 `GET /api/me/wallet` |
| `src/App.tsx` | 新增钱包页面 `/wallet` 与「我的」入口 |
| `src/types.ts`、`src/styles.css` | 新增类型与样式 |
| `package.json` | 新增 `npm run grant` 脚本 |
| `test/currency.test.ts` | 新增货币校验单元测试 |

## 三、部署步骤（服务器）

要求与主项目一致：Node.js 22+、MySQL 8.0+，已按 `README.md` 部署过旧版本。

1. 拉取最新代码到应用目录（示例 `/www/wwwroot/campus-market-web`）并安装依赖：

   ```bash
   cd /www/wwwroot/campus-market-web
   git pull
   npm install
   ```

   本次**没有新增 npm 依赖**，`npm install` 只是确保环境一致。

2. 执行测试与构建：

   ```bash
   npm test
   NODE_ENV=production npm run build
   ```

3. 重启服务。数据库建表**无需手工操作**：服务启动时 `initDatabase()` 会执行 `CREATE TABLE IF NOT EXISTS`，自动创建 `wallets` 与 `currency_ledger` 两张新表（仅在本站库内，不创建/删除数据库）。

   ```bash
   sudo systemctl restart campus-market-web
   sudo systemctl status campus-market-web
   ```

4. `.env` 无需新增必填项。可选配置 `ADMIN_OPERATOR_NAME`（如 `管理员小王`），用于在发放流水中署名；不配置时依次回退到 `ADMIN_NOTIFICATION_EMAIL`、`cli`。

## 四、管理员发放方法

在服务器应用目录下执行（`--` 用于把参数传给脚本）：

```bash
# 发放 10 个至纯源石给 student@ruc.edu.cn
npm run grant -- student@ruc.edu.cn originium 10 "修复登录页样式问题"

# 发放 200 龙门币（币种也支持中文名）
npm run grant -- student@ruc.edu.cn 龙门币 200 "参与校园闲置线下推广"

# 查询某用户余额与最近 20 条流水
npm run grant -- --list student@ruc.edu.cn

# 跳过交互确认（用于自动化脚本，注意核对参数）
npm run grant -- student@ruc.edu.cn originium 10 "自动化发放" --yes
```

规则与校验（不满足会报错且**不会落库**）：

- 邮箱：必须为已注册账号的邮箱（不区分大小写）
- 币种：`originium` / `至纯源石`、`lungmen` / `龙门币`
- 数量：`1 – 1,000,000` 的**正整数**
- 原因：至少 2 个字符（最多 200），会展示在用户的入账明细中
- 交互确认：默认要求输入 `yes` 才执行；确认前会显示目标用户、当前/新余额与原因

## 五、用户侧查看

用户登录后：

- 手机/桌面：底部或顶部导航进入「我的」→ 点击「奖励与资产」
- 或直接访问：`https://<你的域名>/campus-trade/wallet`
- 页面显示两种货币余额（后端下发名称与说明）与最近 50 条入账明细（`+数量 币种名`、原因、时间）

## 六、验证清单

1. `npm test` 全部通过（含 `currency.test.ts` 的 3 个用例）
2. `npm run build` 类型检查与构建通过
3. 发放一条测试奖励后：
   - 数据库：`SELECT * FROM wallets; SELECT * FROM currency_ledger ORDER BY id DESC LIMIT 5;`
   - 页面：用对应账号登录访问 `/wallet`，余额与明细正确显示
4. 负例：不存在的邮箱、非法币种、0 或小数金额、缺少原因——均应报错且 `currency_ledger` 无新记录

## 七、修改货币名称

货币名称暂定，若要改名：编辑 `server/currency.ts` 中 `CURRENCIES` 的 `name`（与 `description`）即可。前端展示名称由 `GET /api/me/wallet` 实时下发，改后重启服务即全站生效，无需改前端代码。

## 八、常见问题

- **旧库升级后表没建？** 只要服务正常启动过一次，`initDatabase` 会自动建表；确认 MySQL 账号对该库有 CREATE 权限（与现有 `campus_market_app` 建表逻辑一致）。
- **发放后发现发错了怎么办？** 流水不可删除。请在 `currency_ledger` 补一条负数对冲前先记录：推荐做法是通知用户并由管理员执行修正 SQL（示例，务必先备份）：
  ```sql
  UPDATE wallets SET balance = balance - 100 WHERE user_id = <id> AND currency = 'originium';
  INSERT INTO currency_ledger (user_id,currency,amount,balance_after,reason,operator) VALUES (<id>,'originium',-100,<修正后余额>,'冲正：误发','<操作者>');
  ```
- **`ADMIN_OPERATOR_NAME` 怎么生效？** 写入 `.env` 后重启终端会话即可；脚本每次执行时读取。
- **未来扩展**：如需消费/兑换场景，可在 `currency_ledger` 增加类型列（如 `grant`/`redeem`），并复用钱包的扣减事务逻辑。

## 九、本地开发环境搭建与测试（个人电脑，无需服务器）

适用于开发者在 Windows 个人电脑上跑测试、构建与完整功能自测。单元测试与构建**不依赖 MySQL**，只有启动 API 与发放 CLI 需要本地 MySQL。

### 1. 安装 Node.js（手动）

- 浏览器打开 <https://nodejs.org/zh-cn/download>（如直连受限，走你的代理 `127.0.0.1:7897`），下载 **LTS 版 Windows Installer（.msi）**（要求版本 ≥ 22）
- 运行安装包，一路下一步（会弹 UAC，允许即可）；安装器默认自带 npm
- 打开**新的** PowerShell 验证：

  ```powershell
  node -v
  npm -v
  ```

### 2. 配置 npm 网络（二选一）

- 走代理（Clash/代理软件已监听 7897 时）：

  ```powershell
  npm config set proxy http://127.0.0.1:7897
  npm config set https-proxy http://127.0.0.1:7897
  ```

- 或改用国内镜像（不依赖代理）：

  ```powershell
  npm config set registry https://registry.npmmirror.com
  ```

- 取消代理/恢复默认：`npm config delete proxy`、`npm config delete https-proxy`

### 3. 安装依赖并跑测试与构建

```powershell
cd <项目目录>
npm install
npm test        # 单元测试，含 test/currency.test.ts，不需要 MySQL
npm run build   # tsc 类型检查 + vite 打包，不需要 MySQL
```

### 4. 安装 MySQL 8（手动）

- 打开 <https://dev.mysql.com/downloads/installer/> 下载 **mysql-installer-community**（约几百 MB，走代理或镜像）
- 安装时类型选 **Server only**，配置环节设置 root 密码、端口默认 3306
- 完成后用 root 登录并创建独立库与最小权限账号（与 `.env.example` 对应）：

  ```sql
  CREATE DATABASE campus_market CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
  CREATE USER 'campus_market_app'@'localhost' IDENTIFIED BY '换成你的强密码';
  GRANT ALL PRIVILEGES ON campus_market.* TO 'campus_market_app'@'localhost';
  FLUSH PRIVILEGES;
  ```

  （本地也可直接用 root 账号，把 `MYSQL_USER/MYSQL_PASSWORD` 填 root 即可；但建议沿用最小权限账号。）

### 5. 配置 .env

```powershell
copy .env.example .env
```

编辑 `.env`：填写 `MYSQL_PASSWORD`（与第 4 步一致）。本地测试**无需配置** COS 与邮箱（不影响钱包、浏览、会话；只是商品图片上传会返回未配置提示）。可把 `SEED_DEMO_DATA=true` 打开以生成 3 个演示商品。

### 6. 创建测试账号（本地无邮箱服务时的替代方案）

注册需要 QQ 邮箱 SMTP，本地不配邮箱时手动插一个账号：

```powershell
# 在项目目录生成密码哈希（bcryptjs 已在 node_modules 中）
node -e "require('bcryptjs').hash('password123',12).then(h=>console.log(h))"
```

把输出（`$2b$12$...`）填进下面的 SQL 执行：

```sql
INSERT INTO users (email,nickname,password_hash,school_id,verified,email_verified,email_message_notifications)
VALUES ('test@ruc.edu.cn','测试同学','<上面的哈希>','ruc_suzhou',1,1,1);
```

`@ruc.edu.cn` 邮箱 + `email_verified=1` 即为校园认证用户，拥有完整权限。

### 7. 启动并验证钱包功能

```powershell
npm run dev
```

- 浏览器打开 <http://localhost:5173>，用 `test@ruc.edu.cn / password123` 登录
- 另开一个终端执行发放：

  ```powershell
  npm run grant -- test@ruc.edu.cn originium 10 "本地测试发放"
  npm run grant -- --list test@ruc.edu.cn
  ```

- 访问 <http://localhost:5173/wallet>（或「我的 → 奖励与资产」），应看到余额 10 与一条入账明细
- 负例验证：`npm run grant -- nobody@ruc.edu.cn originium 10 x`、`npm run grant -- test@ruc.edu.cn 黄金 10 x` 等均应报错且不落库

### 8. 收尾

- 全部验证通过后，如不想保留本地环境：卸载 MySQL（控制面板）并删除 Node 安装即可；项目内 `node_modules`、`.env` 可随时删除重建
- 部署到服务器仍按本文档第三、四节执行

### 9. 补充：用已有数据库备份测试

若手头已有 `campus_market` 库的 `mysqldump` 备份（.sql 文件），可跳过第 6 步「创建测试账号」，直接导入备份复用真实数据：

```powershell
mysql -u root -p campus_market < "备份文件.sql"
```

注意事项：

- 备份通常**不含**本次新增的 `wallets`、`currency_ledger` 表，属正常现象；服务启动时 `initDatabase()` 会自动补建
- 备份中的用户密码哈希对应的是未知密码，无法直接登录。可重新生成一个已知密码的哈希并更新该用户：
  ```powershell
  node -e "require('bcryptjs').hash('password123',12).then(h=>console.log(h))"
  ```
  ```sql
  UPDATE users SET password_hash='<新哈希>' WHERE email='<某用户邮箱>';
  ```
- 备份里的商品图多为公开 CDN 地址，本地浏览无需 COS 配置即可显示
