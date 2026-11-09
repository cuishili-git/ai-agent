---
title: "从零开发一个 WorkBuddy Skill：新手一步一步做到上线"
date: 2026-11-09 13:13:26
categories:
  - "技术笔记"
  - "WorkBuddy"
tags:
  - "skill"
  - "workbuddy"
  - "hexo"
  - "教程"
  - "新人向"
---
> 这篇是写给**从没写过 Skill 的人**的。不要求你懂 Hexo、YAML、Git 分支或 GitHub Pages——这些名词第一次出现时都会先解释。
>
> 跟着做完，你会得到：① 一个能被 WorkBuddy 认出来并调用的 Skill；② 一个能打包发给同事的 zip；③ 一篇用这个 Skill 生成、并且**真的能在浏览器打开**的博客文章。
>
> 每一步都写清「敲什么命令」「应该看到什么」「没看到怎么办」。凡是要换成你自己路径的地方，都用 `你的…` 标出来了。

## 第 0 步：先看懂整条路线

别急着敲命令。先把这张图记住，后面每一步都知道自己在哪：

```
  ① 写 Skill             ② 用 Skill          ③ 生成网页           ④ 放到网上
┌──────────────┐      ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ 一个文件夹    │      │ 一段 md 正文 │    │ Hexo         │    │ GitHub Pages │
│ SKILL.md     │ ───> │    变成      │ ─> │ md → html    │ ─> │ 给你一个网址 │
│ + 脚本       │      │ 带标题的文章 │    │ public/      │    │ 别人能打开   │
└──────────────┘      └──────────────┘    └──────────────┘    └──────────────┘
```

四个名词的大白话：

| 名词 | 是什么 | 在这里干嘛 |
|---|---|---|
| **Skill** | 就是**一个文件夹**，里面放一份叫 `SKILL.md` 的说明书，外加可选的脚本 | 我们要做的东西 |
| **SKILL.md** | 说明书。开头几行是「身份证」（叫 front-matter），后面是操作步骤 | 告诉 WorkBuddy 遇到什么话、该按什么步骤做 |
| **Hexo** | 一个把 Markdown 变成 HTML 网页的工具 | 把文章变成网页 |
| **GitHub Pages** | GitHub 提供的免费静态网页托管 | 让网页有个能打开的网址 |

**最容易混的一点**：GitHub 不会帮你把 `.md` 变成网页，Hexo 也不会帮你把网页放到网上。它们之间唯一的交接物是**一堆生成好的 HTML 文件**。整篇教程的后半段，就是在搬这堆文件。

## 第 1 步：确认工具齐全（2 分钟）

打开 **Git Bash**（开始菜单搜 Git Bash，别用 cmd），逐条敲：

```bash
node -v
npm -v
python --version
git --version
ls -1 ~/.workbuddy/skills
```

**应该看到**：四个版本号（Node 18 以上就行）、以及 skills 目录里已有的技能名（目录不存在或空的也正常）。

**没看到怎么办**：缺哪个装哪个；`~/.workbuddy/skills` 不存在就建一个：

```bash
mkdir -p ~/.workbuddy/skills
```

> 为什么先体检：后面每一步都依赖这些命令。做到一半才发现没装 Node，回头重装会打乱节奏。

## 第 2 步：用三行话写清需求

拿我们真实要做的东西举例。原始需求只有一句：

> 「我想对 WorkBuddy 说一句话，它就把一段对话内容写成一篇带标题、带栏目的博客文章。」

动手前，把它拆成三行：

| | 内容 |
|---|---|
| **输入** | 一段文字或 `.md` 文件、标题、栏目、标签 |
| **输出** | `source/_posts/栏目/标题.md`，开头带 front-matter |
| **明确不做** | 不自动发布到网上——发布是不可撤销的外部动作，必须你另外明确说 |

**新手最常犯的错**：一上来就写脚本。先花两分钟写清这三行，后面每个参数、每个开关都有依据，不容易返工。

## 第 3 步：用官方脚手架生成骨架

WorkBuddy 自带 `skill-creator`，里面有初始化脚本，别自己手写目录结构：

```bash
python "你的WorkBuddy安装目录/resources/.../skills/skill-creator/scripts/init_skill.py" \
  hexo-blog-publisher --path ~/.workbuddy/skills
```

找不到这个脚本？用文件管理器在 WorkBuddy 安装目录下搜 `init_skill.py`，把搜到的完整路径粘进来即可。本机实测路径是：

```
D:/software/workbuddy/resources/app.asar.unpacked/resources/plugins/workbuddy-builtin/skills/skill-creator/scripts/init_skill.py
```

**应该看到**：生成出这样一个目录

```
~/.workbuddy/skills/hexo-blog-publisher/
├── SKILL.md                    ← 说明书（核心）
├── scripts/example.py          ← 示例，删
├── references/api_reference.md ← 示例，删
└── assets/example_asset.txt    ← 示例，删
```

生成完**第一件事是把示例文件删掉**：

```bash
D=~/.workbuddy/skills/hexo-blog-publisher
rm -f "$D/scripts/example.py" "$D/references/api_reference.md" "$D/assets/example_asset.txt"
rmdir "$D/assets" 2>/dev/null || true
```

> 为什么放 `~/.workbuddy/skills/`：这是**用户级**目录，所有项目都能用。若放某个项目里的 `.workbuddy/skills/`，就只有那个项目能用。新手一律选用户级。

## 第 4 步：写 SKILL.md 的「身份证」

打开 `SKILL.md`，最上面这几行就是 front-matter（用 `---` 包起来的元数据）：

```yaml
---
name: hexo-blog-publisher
description: 把本地对话内容、聊天导出或 Markdown 文件写成 Hexo 博文，支持指定标题、分类、标签与栏目子目录。当用户要求把对话内容发布/归档到 Hexo 博客、按指定栏目创建文章，或把生成的文章分享给团队时使用。
agent_created: true
---
```

三个字段：

| 字段 | 作用 |
|---|---|
| `name` | 技能名字，跟文件夹名一致 |
| `description` | **最关键**，决定 WorkBuddy 会不会「想起」这个技能 |
| `agent_created` | 标记这是 AI 辅助创建的，保留即可 |

**为什么 `description` 最关键**：WorkBuddy 不会把所有 Skill 一股脑读进上下文，而是先扫所有 `name` + `description`，判断当前这句话要不要用它，命中了才读正文。所以必须写**用户会怎么说**，而不是写功能清单：

| 写法 | 用户说「把刚才的对话发到博客」时能命中吗 |
|---|---|
| `一个用于发布的工具` | ❌ 太抽象，匹配不上 |
| `当用户要求把对话内容发布/归档到 Hexo 博客、按指定栏目创建文章时使用` | ✅ |

> 顺带解释 YAML：`---` 只是「下面几行是元数据」的标记；`键: 值` 就是给这个技能填表。不需要懂更多。

## 第 5 步：写 SKILL.md 正文（给 AI 看的操作手册）

身份证下面开始写正文。照这个最小模板填：

```markdown
# Hexo 博文发布

## 用途
一句话说清能干什么、默认不干什么。

## 触发场景
| 用户说法 | 执行哪些步骤 |
|---|---|
| 「把刚才的对话发到博客」 | 步骤 1-5，只写本地文件 |
| 「发到 XX 栏目，标题叫 XX」 | 步骤 3 带上 --path / --title |
| 「顺便部署上线」 | 步骤 7，必须先确认 |

## 步骤 1：定位博客根目录
（每步给一条能直接复制的命令）

## 步骤 2：整理内容
……

## 参数表
| 参数 | 作用 |
……

## 安全边界
默认只写本地文章；构建、部署、推送、上传都要用户明确要求。

## 常见坑
（把踩过的坑逐条写这里）
```

两条写正文的规矩：

1. **这是写给执行者（AI）看的，不是写给用户看的**。每步都要能直接复制执行，别写「配置好相关参数」这种空话。
2. **把「不该做的事」写明**。比如「默认只写文件，不自动部署」——发布到公开站点不可逆，把它设成默认行为很危险。

## 第 6 步：写脚本（先做一个能跑的最小版）

新建 `scripts/publish_post.js`。先别追求面面俱到，下面这版只用 Node 内置模块，复制就能跑：

```javascript
const fs = require('fs');
const path = require('path');

function arg(flag, fallback) {
  const i = process.argv.indexOf(flag);
  return i >= 0 ? process.argv[i + 1] : fallback;
}

const blogRoot = arg('-b', process.env.HEXO_BLOG_ROOT);
const title    = arg('-t', null);
const srcFile  = arg('-f', null);
const column   = arg('-p', '');

if (!blogRoot || !title || !srcFile) {
  console.error('用法: node publish_post.js -b <博客根目录> -t <标题> -f <正文.md> [-p <栏目>]');
  process.exit(1);
}

// 1) 读正文：去 BOM、统一换行，否则 front-matter 会失效
let body = fs.readFileSync(srcFile, 'utf8')
  .replace(/^\uFEFF/, '').replace(/\r\n/g, '\n');

// 2) 拼 front-matter。JSON.stringify 负责把引号、冒号转义成合法 YAML 字符串
const q = s => JSON.stringify(String(s));
const d = new Date(), pad = n => String(n).padStart(2, '0');
const date = `${d.getFullYear()}-${pad(d.getMonth()+1)}-${pad(d.getDate())} ` +
             `${pad(d.getHours())}:${pad(d.getMinutes())}:${pad(d.getSeconds())}`;
const text = ['---', 'title: ' + q(title), 'date: ' + date, '---', '', body].join('\n');

// 3) 算目标路径，并拦住「../」跳出自录
const posts = path.resolve(blogRoot, 'source', '_posts');
const target = path.resolve(posts, column, title.replace(/[\\/:*?"<>|]/g, '-') + '.md');
if (!target.startsWith(posts + path.sep)) {
  console.error('栏目路径非法（不能跳出 _posts）');
  process.exit(1);
}

// 4) 防覆盖：已存在就拒绝，除非显式 --force
if (fs.existsSync(target) && process.argv.indexOf('--force') < 0) {
  console.error('文件已存在，加 --force 才允许覆盖: ' + target);
  process.exit(1);
}
fs.mkdirSync(path.dirname(target), { recursive: true });
fs.writeFileSync(target, text, 'utf8');
console.log('[ok] ' + target);
```

四个看起来「多余」的地方，都是真出过事的：

| 代码 | 不写会怎样 |
|---|---|
| 去 BOM / 转 LF | Windows 记事本存的 md，开头的 `---` 认不出来 |
| `JSON.stringify` 转义 | 标题里有 `:` 或 `"` 时 YAML 直接解析失败 |
| 检查 `posts + path.sep` | 栏目填 `../../Windows` 会把文件写到博客目录外 |
| 防覆盖 | 一篇写好的文章被静默覆盖，找不回来 |

## 第 7 步：在临时目录里先跑通

**不要拿真实博客做第一次试验**。造一个假的：

```bash
SB=/tmp/hexo-demo
mkdir -p "$SB/source/_posts"
echo "title: demo" > "$SB/_config.yml"
echo "这是我的第一篇文章正文。" > "$SB/正文.md"

node ~/.workbuddy/skills/hexo-blog-publisher/scripts/publish_post.js \
  -b "$SB" -t "我的第一篇" -f "$SB/正文.md" -p 新手村
```

**应该看到**：`[ok] /tmp/hexo-demo/source/_posts/新手村/我的第一篇.md`

再打开确认内容：

```bash
cat "$SB/source/_posts/新手村/我的第一篇.md"
```

开头必须是：

```yaml
---
title: "我的第一篇"
date: 2026-11-09 14:00:00
---
```

**没看到 `---`**：多半是正文文件带了 BOM，回到第 6 步检查去 BOM 那段。

## 第 8 步：补一份自检，把「不该发生的」也测一遍

新手容易只测「正常情况」。至少再测这四件事：

| 测什么 | 怎么测 | 期望 |
|---|---|---|
| 标题带冒号引号 | `-t 'A: "B"'` | 生成的文件里 YAML 仍是合法字符串 |
| 重复运行 | 同一命令跑两次 | 第二次**报错**并提示文件已存在 |
| 路径穿越 | `-p ../../etc` | 直接拒绝 |
| 只预览不写 | 加 `--dry-run`（如果你实现了） | 不产生文件 |

写一个 `selftest.js` 挨个断言即可。本机那版跑了 13 项，包括 BOM/CRLF、防覆盖、`--force`、部署保护等。

> 这一条是分水岭：能跑通正常流程 ≠ 别人乱输参数时不会出事。

## 第 9 步：打包，发给同事

```bash
python "你的WorkBuddy安装目录/.../skill-creator/scripts/package_skill.py" \
  ~/.workbuddy/skills/hexo-blog-publisher "你的输出目录"
```

**应该看到**：校验通过 + 生成一个 `hexo-blog-publisher.zip`。别人把这个包导入自己的 WorkBuddy，就能用同一套流程。

到这里，**Skill 本身已经做完并上线可用了**。下面两步是「用它产出东西，并把东西真正发布到网上」。

## 第 10 步：用一次自己的 Skill

现在可以直接在 WorkBuddy 对话里说：

> 「把这段对话整理成博文，发到 `Skill开发` 栏目，标题叫《从零开发一个 WorkBuddy Skill》」

也可以直接跑命令（效果一样）：

```bash
node ~/.workbuddy/skills/hexo-blog-publisher/scripts/publish_post.js \
  -b "D:/workbuddy/docs/out/blog" \
  -t "从零开发一个 WorkBuddy Skill：新手一步一步做到上线" \
  -f "正文.md" \
  --category 技术笔记,WorkBuddy --tag skill,workbuddy -p Skill开发
```

产出的是一个 `.md` 文件，**它还不是网页**。

## 第 11 步：Hexo 把 md 变成网页

### 11.1 建 `package.json`

在博客根目录新建 `package.json`：

```json
{
  "name": "my-blog",
  "private": true,
  "dependencies": {
    "hexo": "^8.1.2",
    "hexo-generator-archive": "^2.0.0",
    "hexo-generator-category": "^2.0.0",
    "hexo-generator-index": "^4.0.0",
    "hexo-generator-tag": "^2.0.0",
    "hexo-renderer-ejs": "^2.0.0",
    "hexo-renderer-marked": "^7.0.1",
    "hexo-renderer-stylus": "^3.0.1",
    "hexo-server": "^3.0.0",
    "hexo-theme-landscape": "^1.1.0"
  },
  "hexo": { "version": "8.1.2" }
}
```

> ⚠️ **最下面那个 `"hexo"` 字段千万不能删**。删了以后 `hexo generate` 不报错、只打印一屏帮助文本，看起来像成功，其实 `public/` 还是旧的。这是最坑新手的一个静默失败。

### 11.2 装依赖

```bash
cd "你的博客根目录"
npm install --registry=https://registry.npmmirror.com --no-audit --no-fund
```

官方源慢或超时时，用上面这个国内镜像。本机 32 秒装完 243 个包。

### 11.3 最小 `_config.yml`

```yaml
title: 我的技术笔记
author: 你的名字
language: zh-CN
timezone: 'Asia/Shanghai'
url: https://你的用户名.github.io/你的仓库名
root: /你的仓库名/          # 只有 <用户名>.github.io 这种仓库才写 /
permalink: :year/:month/:day/:title/
theme: landscape
```

| 配置项 | 写错的后果 |
|---|---|
| `url` | 站点里所有绝对链接、RSS 指错域 |
| `root` | 项目仓库的 Pages 挂在子路径，少了它 CSS/JS 全部 404、页面没样式 |
| `permalink` | 决定文章网址。**建站后再改，所有旧链接失效** |

### 11.4 生成并验收

```bash
cd "你的博客根目录"
node_modules/.bin/hexo.cmd clean
node_modules/.bin/hexo.cmd generate
```

**应该看到**：`INFO  20 files generated in 300 ms` 这一类输出。判断成功的唯一依据是**有没有 `generated` 这个词**，不是看数字大小（`0 files generated` 也可能正常），也不是只看退出码。

> Windows 必须用 `hexo.cmd`，不能用 `node_modules/.bin/hexo`（那是给 Linux/macOS 的，会报 WinError 193）。

再看产物：

```
public/
├── index.html          ← 首页
├── 2026/11/09/…/index.html  ← 每篇文章一页
├── archives/  categories/  tags/
├── css/  js/  fancybox/
└── .nojekyll           ← 见下
```

**必须补一个空的 `.nojekyll`**：GitHub Pages 默认会跑 Jekyll，它会忽略下划线开头的目录，Hexo 产物容易缺东西。而 `hexo clean` 会把这个文件删掉，所以**每次 generate 后都要重写**：

```bash
: > public/.nojekyll
```

想先在本机看看长什么样：

```bash
node_modules/.bin/hexo.cmd server    # 打开 http://localhost:4000/你的仓库名/
```

## 第 12 步：发布到 GitHub Pages

### 12.1 网页上建仓库

打开 https://github.com/new，填仓库名（例如 `ai-agent`），选 Public，**「Add a README」「Add .gitignore」「Choose a license」三项一个都别勾**。

> 为什么别勾：勾了远端会自带一个初始提交，和你本地的第一次提交没有共同祖先，第一次 push 必然被拒，还得硬合。空仓库最干净。

### 12.2 源码和产物分两个分支

这是新手最容易搞乱的地方，记住一句话：**`main` 放你写的 md，`gh-pages` 放 Hexo 生成的 html**。

```bash
cd "你的博客根目录"
git init -b main
git config user.name "你的账号"          # 只在这个仓库生效，不改全局
git config user.email "账号绑定邮箱"
git add -A
git commit -m "post: 第一篇文章"
git remote add origin ssh://git@ssh.github.com:443/你的用户名/你的仓库名.git
git push -u origin main
```

然后把 `public/` 单独推到 `gh-pages`（用一个临时目录，避免切分支时动到源码）：

```bash
TMP=$(mktemp -d)
cp -r public/. "$TMP/"
git -C "$TMP" init -b gh-pages -q
git -C "$TMP" config user.name "你的账号"
git -C "$TMP" config user.email "账号绑定邮箱"
git -C "$TMP" remote add origin ssh://git@ssh.github.com:443/你的用户名/你的仓库名.git
git -C "$TMP" add -A
git -C "$TMP" commit -m "deploy: 静态站点"
git -C "$TMP" push -f origin gh-pages
```

推完 `gh-pages` 分支，GitHub 通常会**自动启用 Pages**（在仓库 Settings → Pages 里能看到 Source 是 `Deploy from a branch / gh-pages / (root)`；没自动启用就手动选一下）。

> 为什么 `-f` 强制推：每次都是全新的孤立提交，和远端没有共同祖先。这是产物分支，本来就是「`public/` 的完整镜像」，覆盖是预期行为。**只对 `gh-pages` 用，别对 `main` 用**。

### 12.3 打开网址验收

站点地址格式：

```
https://你的用户名.github.io/你的仓库名/
```

```bash
curl -sS -L -o /dev/null -w "%{http_code}\n" https://你的用户名.github.io/你的仓库名/
```

- `404` → 还在构建，**等 30~90 秒，不要重复发布**（这是最常见的误操作）
- `200` → 起来了

**但 200 还不够**：空页面、模板页也可能返回 200。再抓一次首页 HTML，确认里面有你的站名、文章标题、以及 `/你的仓库名/css/style.css`：

```bash
curl -sS -L https://你的用户名.github.io/你的仓库名/ | grep -o "你的文章标题"
```

全中才说明内容真的渲染出来了。

### 12.4 以后每次发文只要一条命令

把 11.4 + 12.2 固化成一个脚本（本机放在 `D:/workbuddy/docs/hexo-deploy.sh`），以后：

```bash
bash hexo-deploy.sh
```

改文章 → 跑脚本 → 等半分钟 → 网页更新。这就是整套流程的终点。

## 第 13 步：日常维护与常见错误

| 现象 | 原因 | 怎么修 |
|---|---|---|
| `hexo generate` 只打印帮助、不报错 | `package.json` 少了 `"hexo"` 字段 | 补回该字段 |
| `Author identity unknown` | 没配 `user.name` / `user.email` | 在仓库里配两条 `git config` |
| `non-fast-forward` / push 被拒 | 建仓库时勾了 README，或忘加 `-f` 推产物分支 | 空仓库重来；只对 `gh-pages` 用 `-f` |
| 页面没样式、CSS 404 | `_config.yml` 少 `root: /仓库名/` | 补上重新生成 |
| 突然要求输密码/令牌 | SSH 地址被 `insteadOf` 规则改写成 HTTPS | 用 `ssh://git@ssh.github.com:443/...` 形式，不匹配该规则 |
| 命令行粘贴令牌没反应 | 输密码时屏幕本就不回显 | 照常粘贴回车即可 |
| 改了文章，网页没变 | 没跑 `clean` + `generate`，或没重新推 `gh-pages` | 跑一遍发布脚本 |
| 提交成功但页面 404 | Pages 还在构建 | 等，别重复发布 |
| 网页上直接改的 HTML 被冲掉 | 产物分支是 `public/` 的镜像，全量覆盖 | 改 `source/_posts/` 里的 md，重新发布 |

## 术语速查

| 词 | 大白话 |
|---|---|
| front-matter | 文章开头 `---` 包住的几行元数据（标题、日期、分类、标签） |
| YAML | 一种「键: 值」的填表格式，front-matter 和 `_config.yml` 都用它 |
| 产物 / `public/` | Hexo 生成的那一堆 HTML、CSS、JS |
| 分支 | 同一仓库里并行的两份内容；这里 `main` 存源码、`gh-pages` 存产物 |
| Pages | GitHub 把某个分支当静态网站发布出来的功能 |
| PAT / 令牌 | 网站密码的替代品，HTTPS 方式推代码时要用；本流程走 SSH 密钥，用不上 |
| `.nojekyll` | 一个空文件，告诉 Pages「别用 Jekyll 加工，原样发」 |

## 完成检查清单

- [ ] `node/npm/python/git` 都能打印版本号
- [ ] `~/.workbuddy/skills/xxx/SKILL.md` 有 `name` / `description` / `agent_created`
- [ ] `description` 写的是「用户会怎么说」
- [ ] 示例文件已删
- [ ] 脚本在临时目录跑通，生成的文件开头是 `---`
- [ ] 重复运行会拒绝覆盖、路径穿越会被拦
- [ ] 打包校验通过，拿到 zip
- [ ] 用 Skill 生成了一篇真实文章
- [ ] `hexo generate` 输出里有 `generated`
- [ ] `public/` 里有 `index.html` 和 `.nojekyll`
- [ ] `main` 和 `gh-pages` 都推上去了
- [ ] 网址返回 200，且页面里能搜到文章标题

---

一句话收尾：**Skill 只是一个带说明书的文件夹；Hexo 只负责把 md 变 HTML；GitHub Pages 只负责把 HTML 发出去。** 把这三件事分开，哪一步出错就查哪一步，不用整条链路重来。
