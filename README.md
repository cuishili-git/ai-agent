# AI Agent 与 Skill 开发笔记

Hexo 博客 · **站点地址：https://cuishili-git.github.io/ai-agent/**

记录 WorkBuddy Skill / AI Agent 工具链的开发复盘、踩坑记录与实操笔记。

## 两个分支，互不覆盖

| 分支 | 内容 |
|---|---|
| `main` | **源码**：文章 Markdown、`_config.yml`、`package.json`、`package-lock.json` |
| `gh-pages` | **产物**：`public/` 的完整镜像（`index.html` / `css/` / `js/` / `.nojekyll`） |

GitHub Pages 用 `Deploy from a branch` → `gh-pages` / `(root)`。实测推送 `gh-pages` 分支后 Pages 自动启用，首页与文章页均返回 200。线上仓库里没有 `.md` 源文件，改内容要改 `main` 里的 md 再重新发布。

## 文章

| 标题 | 栏目 | 分类 |
|---|---|---|
| 从零开发一个 WorkBuddy Skill：新手一步一步做到上线 | `Skill开发` | 技术笔记 / WorkBuddy |
| Git 登录与 Hexo 博文自动发布 | `技术随笔` | 技术笔记 / Git |

## 一条命令发布

```bash
bash D:/workbuddy/docs/hexo-deploy.sh
```

脚本依次执行：`hexo clean` → `hexo generate`（**校验输出里必须出现 `generated`**）→ 写 `.nojekyll` → 用独立临时仓库 force push 到 `gh-pages`。全程不碰 `main`。

只构建不推送（本地验收）：

```bash
SKIP_PUSH=1 bash D:/workbuddy/docs/hexo-deploy.sh
```

## 手动构建

```bash
cd D:/workbuddy/docs/out/blog
npm install --registry=https://registry.npmmirror.com
node_modules/.bin/hexo.cmd clean
node_modules/.bin/hexo.cmd generate
node_modules/.bin/hexo.cmd server    # http://localhost:4000/ai-agent/
```

**Windows 必须用 `hexo.cmd`**，不能用 `node_modules/.bin/hexo`（那是 Unix 垫片，会报 WinError 193）。

## 配置要点（都是踩过的坑）

| 项 | 说明 |
|---|---|
| `package.json` 里的 `"hexo": {"version": ...}` | 缺了它 `hexo generate` 只打印 help、**不报错**，看似成功实则没生成 |
| `root: /ai-agent/` | 项目仓库的 Pages 挂子路径，缺了 CSS / JS 全 404 |
| `url` | 影响所有绝对链接、RSS、sitemap |
| `permalink` | 建站后再改，所有已发布链接全部失效 |
| `.nojekyll` | 关掉 Pages 的 Jekyll（否则下划线开头目录被忽略）；`clean` 会删它，脚本每次重写 |
| `clean` + `generate` 连跑 | 避免缓存导致「改了但没生效」；判断成功看输出有没有 `generated`，不是看数字大小 |
| 提交后立刻 404 | Pages 构建约 30~90 秒，等待即可，**不要重复发布** |
| `_posts` 子目录 | 会成为文章 URL 的一部分（如 `/2026/11/09/Skill开发/标题/`） |

## 新增文章

用 `hexo-blog-publisher` skill 生成带 front-matter 的文章，再跑发布脚本：

```bash
node ~/.workbuddy/skills/hexo-blog-publisher/scripts/publish_post.js \
  -b "D:/workbuddy/docs/out/blog" -t "文章标题" -f "正文.md" \
  --category 技术笔记,WorkBuddy --tag skill,workbuddy \
  -p Skill开发 --json

bash D:/workbuddy/docs/hexo-deploy.sh
```

脚本默认只写文章文件，不自动构建、部署或推送。

## 推送通道（本机网络受限）

本机直连 GitHub 的 22 / 443 端口会被 reset。用 `D:\workbuddy\docs\ssh-https-proxy.py` 做 HTTP CONNECT 隧道，复用已验证的 ED25519 密钥，**无需 token**（本机 `git credential fill` 取不到凭据，所以没走 API 部署那条路）。

```bash
# 隧道脚本自动从 HTTPS_PROXY 读取代理地址，端口变化无需改配置
git config core.sshCommand 'ssh -p 443 -o ProxyCommand="python D:/workbuddy/docs/ssh-https-proxy.py ssh.github.com 443"'
git remote add origin ssh://git@ssh.github.com:443/cuishili-git/ai-agent.git
```

**不要用 `git@github.com:用户名/仓库.git`**：全局 `url.https://github.com/.insteadof=git@github.com:` 会把它悄悄改写成 HTTPS，密钥不参与、转而要令牌。

完整步骤见 `D:\workbuddy\docs\Hexo博客推送Skill使用说明.md`。
