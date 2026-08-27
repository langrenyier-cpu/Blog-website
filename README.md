# 个人博客 · Cloudflare Workers 免费版

一个开箱即用的个人博客，零服务器、零数据库、零费用，部署在 Cloudflare 免费套餐上。后端用 [Hono](https://hono.dev/) 跑在 Workers，数据存 KV，图片存 R2，前端静态页走 Workers Static Assets。

---

## ✨ 功能一览

**前台（访客可见）**
- 实时时钟、天气与所在地（浏览器定位 / IP 自动定位）
- 文章列表 + 详情，支持**分页加载更多**与**关键词搜索**
- 文章内可插入图片、可点击链接、富文本排版（字号 12–36px 像素级、粗体 / 斜体 / 下划线 / 删除线 / 荧光笔高亮 / 彩色文字 / 对齐 / 列表 / 引用 / 代码块）
- 联系方式卡片：内置 **22 个平台**标识（微信、邮箱、GitHub、Telegram、YouTube、Discord、Instagram、小红书……），点击复制或跳转
- 友情链接（新窗口打开）
- 文章**分享**：一键复制短链 + 系统分享（移动端/支持的浏览器）
- **邮件订阅**：访客在侧栏填邮箱订阅；订阅人数仅博主后台可见，前台不展示
- **赞助支持**板块：可配置文字 / 图片赞助方式

**后台（`/admin.html`，需登录）**
- 文章增删改查、草稿 / 发布、**文章置顶**
- 图片上传（本地上传 / 外链 / 直接粘贴）
- 站点信息、联系方式、友情链接、赞助配置
- 修改密码
- **订阅管理**：实时查看订阅人数、列表、导出 CSV、删除订阅者

**安全**
- 富文本 XSS 过滤
- Bearer Token 认证（7 天有效），改密码后所有旧登录自动失效
- 所有密钥（Cloudflare API Token、后台密码、私钥）均**不写入仓库**

---

## 🧱 技术架构

| 组件 | 用途 | 免费额度 |
|---|---|---|
| Workers（Hono） | API 后端 + 路由 | 每天 10 万次请求 |
| Workers Static Assets | 前端静态页面（`public/`） | 无限流量 |
| KV | 文章、站点配置、登录凭证、订阅者 | 每天 10 万读 / 1000 写 |
| R2 | 上传的图片 | 10GB 存储 |

> KV 写入次数说明：保存文章、登录、每次文章浏览计数各消耗 1 次写。个人博客日常使用远低于 1000 次/天。

---

## 📁 目录结构

```
personal-blog-cloudflare/
├── wrangler.jsonc        # Cloudflare 配置（KV/R2 绑定、路由在这里填写）
├── package.json
├── gen-assets.mjs        # 把 public/ 各页面内联进 src/html-assets.js 的脚本
├── src/
│   ├── worker.js         # 后端：Hono 路由 + KV + R2
│   └── html-assets.js    # 由 gen-assets.mjs 生成的内联静态资源（勿手改）
└── public/               # 前端静态文件（自动部署为 Workers Assets）
    ├── index.html        # 博客主页
    ├── post.html         # 文章详情页
    ├── admin.html        # 后台管理
    ├── diag.html         # 自检测试页
    ├── css / js
    └── uploads/          # 内置示例图片（avatar.jpg、demo-*.jpg）
```

> 上传的新图片存在 R2 中；示例图片走静态资产，互不冲突。

---

## 🚀 一、本地预览（可选，无需 Cloudflare 账号）

```bash
npm install
npm run dev
```

打开 http://localhost:8787 即可预览（本地由 miniflare 模拟 KV / R2）。

默认后台账号：**admin / admin123**

---

## 🌩️ 二、部署到 Cloudflare（约 5 分钟）

### 前置条件
- 一个免费 Cloudflare 账号：<https://dash.cloudflare.com/sign-up>
- 本机 Node.js 18+

### 步骤 1：登录 Cloudflare

```bash
npx wrangler login
```

会打开浏览器授权，完成后终端显示成功。

### 步骤 2：创建 KV 命名空间

```bash
npx wrangler kv namespace create BLOG_KV
```

复制输出里的 `id`，填入 `wrangler.jsonc`：

```jsonc
"kv_namespaces": [
  { "binding": "BLOG_KV", "id": "粘贴到这里" }
]
```

### 步骤 3：创建 R2 存储桶

```bash
npx wrangler r2 bucket create blog-images
```

桶名任意（`blog-images` 或自定义），把桶名填入 `wrangler.jsonc`：

```jsonc
"r2_buckets": [
  { "binding": "BLOG_R2", "bucket_name": "blog-images" }
]
```

### 步骤 4：部署

```bash
npm run deploy
```

完成后会输出访问地址，形如：

```
https://personal-blog.<你的子域>.workers.dev
```

博客与后台（`https://xxx.workers.dev/admin.html`）即刻可用，示例文章和图片会在第一次访问时自动初始化。

> 若你改了前端（`public/` 下的文件），部署前运行一次 `node gen-assets.mjs` 重新生成内联资源，否则线上可能仍是旧页面。

### 步骤 5：立即修改默认密码 ⚠️

登录后台 → 右上角「修改密码」，把 **admin / admin123** 改成你自己的密码（至少 6 位）。

---

## 🔧 三、进阶

### 绑定自己的域名（可选）

前提：域名已托管在同一 Cloudflare 账号。在 `wrangler.jsonc` 顶层加：

```jsonc
"routes": [
  { "pattern": "blog.example.com", "custom_domain": true }
]
```

再执行 `npm run deploy`，Cloudflare 会自动签发 HTTPS 证书。

### （可选）部署时自定义初始密码

```bash
npx wrangler secret put ADMIN_PASSWORD
```

输入你想要的初始密码。注意：secret 在 Worker 首次初始化时生效；若已用默认密码初始化过，需先清除旧数据（见下方重置方法，删除 `seeded` 和 `auth` 两个键）。

### 忘记密码了怎么办

用一条命令生成新的凭证并写回 KV（把 `<KV_ID>` 换成你的命名空间 id，`新密码` 换成你的新密码）：

```bash
node -e "const c=require('crypto');const salt=c.randomBytes(16).toString('hex');const pw='新密码';require('fs').writeFileSync('new-auth.json',JSON.stringify({username:'admin',salt,hash:c.createHash('sha256').update(salt+pw).digest('hex'),version:Date.now()}))"
npx wrangler kv key put auth --namespace-id=<KV_ID> --path=new-auth.json
rm new-auth.json
```

旧登录会全部失效，用新密码重新登录即可（文章数据不受影响）。

### 常用运维命令

```bash
npx wrangler kv key list --namespace-id=<KV_ID>   # 查看 KV 中的数据键
npx wrangler r2 object list blog-images           # 查看已上传的图片
npx wrangler tail                                 # 实时查看线上日志
npx wrangler secret list                          # 查看已配置的 secret
```

---

## 📦 四、推送到 GitHub（版本管理）

本仓库**不含任何密钥**：`.gitignore` 已排除 `.env / .dev.vars / 私钥 / 证书 / 认证缓存` 等。Cloudflare API Token 仅在 `wrangler login` / `wrangler deploy` 时经浏览器或环境变量传入，从不在代码里。

### 在本机初始化并推送

```bash
# 进入项目目录
cd personal-blog-cloudflare

# 初始化（若还没 init）
git init
git add .
git commit -m "Initial commit: 个人博客 Cloudflare Workers 项目"

# 关联远程仓库（换成你自己的地址）
git remote add origin https://github.com/你的用户名/仓库名.git

# 推送。空仓库直接推；若远程已勾选 README 初始化，先 pull 再推：
#   git pull origin main --allow-unrelated-histories
git push -u origin main        # 或 master，取决于你的默认分支
```

### 用 Personal Access Token 推送（CI / 无交互场景）

1. GitHub → Settings → Developer settings → Personal access tokens → 生成 **classic token**，勾选 `repo` 权限，设短过期时间。
2. 推送时用令牌代替密码：

```bash
git remote set-url origin https://你的用户名:ghp_你的令牌@github.com/你的用户名/仓库名.git
git push -u origin main
# 推完建议把令牌从 remote URL 中移除：
git remote set-url origin https://github.com/你的用户名/仓库名.git
```

> ⚠️ 令牌等同账号密码，用完立刻在 GitHub 里 **Revoke**，不要提交、不要泄露。

### 用 GitHub CLI 创建仓库并推送

```bash
gh auth login
gh repo create 仓库名 --public --description "个人博客 · Cloudflare Workers 免费版"
git push -u origin main
```

---

## 🔒 五、安全须知

- **绝不**把 Cloudflare API Token、后台密码、`.env`、私钥提交到 Git。本仓库 `.gitignore` 已覆盖常见敏感文件。
- 部署后**第一时间修改后台默认密码** `admin123`。
- `ADMIN_PASSWORD` 等敏感值用 `wrangler secret put` 注入，不要写进 `wrangler.jsonc`。
- 公开仓库也请确认没有把 KV 命名空间 id 之外的任何凭证泄露（命名空间 id 本身是非机密的标识符）。

---

## ❓ 常见问题 / 故障排查

| 现象 | 原因 / 解决 |
|---|---|
| 部署后页面是旧的 | 改了 `public/` 没重生成内联资源 → 跑 `node gen-assets.mjs` 再 `npm run deploy` |
| 后台登录失败 | 密码已被改过 → 用「忘记密码」命令重置；或确认 `ADMIN_PASSWORD` secret 已设置 |
| 图片上传 404 | R2 桶名与 `wrangler.jsonc` 不一致 → 核对 `bucket_name` |
| 推送被拒（non-fast-forward） | 远程已有初始提交 → 先 `git pull --allow-unrelated-histories` 再推 |
| `wrangler` 命令找不到 | 先 `npm install`；或用 `npx wrangler ...` |

---

## 许可证

MIT（可自由使用、修改、再分发）。
