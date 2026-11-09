# AI Agent 与 Skill 开发笔记

Hexo 博客源码仓库，记录 WorkBuddy Skill / AI Agent 工具链的开发复盘、踩坑记录与实操笔记。

文章放在 `source/_posts/` 下按栏目分子目录，每篇带 front-matter（`title` / `date` / `categories` / `tags`）。

## 文章

| 文件 | 标题 | 栏目 | 分类 |
|---|---|---|---|
| `source/_posts/Skill开发/如何开发一个-workbuddy-skill从需求到打包的完整复盘.md` | 如何开发一个 WorkBuddy Skill：从需求到打包的完整复盘 | Skill开发 | 技术笔记 / WorkBuddy |
| `source/_posts/技术随笔/git-登录与-hexo-博文自动发布.md` | Git 登录与 Hexo 博文自动发布 | 技术随笔 | 技术笔记 / Git |

## 目录

- `_config.yml` —— 站点配置，含 Pages 子路径 `root: /ai-agent/`
- `source/_posts/` —— 正文，按栏目分子目录
- `.gitignore` —— 忽略 `node_modules/`、`public/`、部署缓存等生成物

本仓库**只提交源码**（文章 + 配置），不提交任何构建产物。

## 用 hexo-blog-publisher skill 新增文章

```bash
node ~/.workbuddy/skills/hexo-blog-publisher/scripts/publish_post.js \
  -b "$(pwd)" -t "文章标题" -f "正文.md" \
  --category 技术笔记,WorkBuddy --tag skill,workbuddy \
  -p Skill开发 --json
```

标题含冒号、引号会自动做 YAML 转义；`--category` 是分类链、`--tag` 是标签、`-p` 是 `source/_posts` 下的栏目子目录，三者可叠加。
脚本**默认只写文章文件**，不自动 `generate` / `deploy` / `git push`。

## 本地构建

首次构建需先准备 Hexo 依赖（用 `hexo init` 生成 `package.json`，或手动补一份），然后：

```bash
npm install
npx hexo generate
npx hexo server
```

因为配了 `root: /ai-agent/`，本地预览地址是 `http://localhost:4000/ai-agent/`。

## 发布成网页（GitHub Pages）

`git push` 上去只是**源码存档**，不等于有网页。要变成可访问的博客：

1. 仓库 **Settings → Pages**，把 Build and deployment 的 Source 选为 **GitHub Actions**（配好 Hexo 的 workflow），或改用 `public/` 分支发布；
2. 非 `用户名.github.io` 的仓库会发布到 `https://cuishili-git.github.io/ai-agent/`，所以 `_config.yml` 里的 `root` 必须保留；
3. `public/` 由构建产出，不要提交进 `main`。

## 推送通道（本机网络受限时）

本机直连 GitHub 的 22 / 443 端口会被 reset，但环境提供了本地 HTTP 代理。SSH 不认 HTTP 代理，用 `ssh-https-proxy.py` 做 CONNECT 隧道即可复用已有的 ED25519 密钥，无需令牌：

```bash
# 代理端口以 echo $HTTPS_PROXY 为准（形如 http://127.0.0.1:62939）
git config core.sshCommand 'ssh -p 443 -o ProxyCommand="python D:/workbuddy/docs/ssh-https-proxy.py 127.0.0.1 62939 ssh.github.com 443"'
git remote add origin ssh://git@ssh.github.com:443/cuishili-git/ai-agent.git
git push -u origin main
```

**为什么不用 `git@github.com:用户名/仓库.git`**：全局配置里有一条
`url.https://github.com/.insteadof=git@github.com:`，它会把 scp 风格地址悄悄改写成 HTTPS，改写后密钥不参与、转而要求输入令牌。`ssh://git@ssh.github.com:443/...` 不匹配这条规则。

网络恢复正常（22 端口可直连）时，把 `core.sshCommand` 取消、remote 改回普通 SSH 地址即可：

```bash
git config --unset core.sshCommand
git remote set-url origin ssh://git@github.com/cuishili-git/ai-agent.git
```
