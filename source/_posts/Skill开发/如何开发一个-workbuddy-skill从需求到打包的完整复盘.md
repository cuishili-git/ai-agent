---
title: "如何开发一个 WorkBuddy Skill：从需求到打包的完整复盘"
date: 2026-11-09 13:13:26
categories:
  - "技术笔记"
  - "WorkBuddy"
tags:
  - "skill"
  - "workbuddy"
  - "hexo"
  - "教程"
---
> 这是一次真实开发的复盘：从一句「帮我做个 skill」的需求，到 13 项自检全过、打包可分发，最后**用这个 skill 自己把自己写成博客文章并推送到 GitHub**。文末附完整命令。

## 一、先搞清楚 Skill 到底是什么

WorkBuddy 里的 Skill 不是插件，也不是需要注册到某个中心的服务。它本质上就是**一个目录**，目录里必须有 `SKILL.md`，文件开头是 YAML front-matter：

```yaml
---
name: hexo-blog-publisher
description: Turns local conversation records, chat exports or markdown files into Hexo blog posts with a chosen title, categories, tags and column sub-directory. This skill should be used when a user asks to push, publish or archive conversation content to a Hexo blog...
agent_created: true
---
```

关键在于 `description`：它决定了**这个 skill 什么时候会被加载**。WorkBuddy 不会把所有 skill 一股脑塞进上下文，而是先扫描所有 `name` + `description`，判断当前请求是否命中，命中才把 SKILL.md 正文读进来。

由此得出两条设计纪律：

| 纪律 | 为什么 |
|---|---|
| `description` 写「触发场景」，不写「功能清单」 | 模型靠语义匹配决定是否加载。写「一个用于发布的工具」，用户说「把刚才的对话发到博客」时未必命中；要写清「当用户要求把对话内容发布/归档到 Hexo 博客、按指定栏目创建文章时使用」 |
| SKILL.md 正文写给**执行者**看，不是写给用户看 | 它是一份指令集：步骤顺序、参数含义、安全边界、出错怎么办。用户看到的是最终产出，不是这份文档 |

`scripts/`、`references/`、`assets/` 都是可选目录：脚本放可执行代码，参考资料放按需才读的细节（避免 SKILL.md 无限膨胀），静态资源单独放。

## 二、需求落地前，先把边界划清楚

原始需求只有一句：「开发一个 skill，调用 hexo 博客工具，推送本地对话内容到博客，支持自定义栏目和标题」。

这句话里有三个歧义，不动手消解，写出来的东西大概率返工：

| 歧义 | 处理方式 |
|---|---|
| 「栏目」指什么 | Hexo 里至少三层：`categories` 分类链、`tags` 标签、`source/_posts` 下的子目录。**三层全支持**，做成三个独立参数，可叠加 |
| 「本地对话内容」指什么 | 当前会话整理稿、本地 `.md` 文件、直接传入的文本、stdin —— **四种来源都收** |
| 「推送」是写到本地，还是发布到公网 | **默认只写文章文件**。`generate` / `deploy` / `git push` / 云端上传全做成显式开关，部署与推送还要二次确认 |

最后一条是安全边界，不是偷懒。把「发布到公开站点」设成默认行为，等于用户一句模糊的话就能触发不可撤销的外部动作。这条纪律在任何涉及外部系统的工具里都该保持。

## 三、环境体检跑在写第一行代码之前

```bash
node --version; npm --version
hexo --version
ls -d /d/workbuddy/docs
ls -1 ~/.workbuddy/skills
find /d/workbuddy -maxdepth 5 -name "_config.yml" -not -path "*/node_modules/*"
```

实测：Node v22.22.2、npm 10.9.7 都在，文档目录在，但**没装全局 hexo，本机也找不到任何现成的 Hexo 项目**。

这个结果直接改变了架构决策：

- 博客根目录不能写死，必须做成参数（`-b` 或环境变量 `HEXO_BLOG_ROOT`），否则换台机器就废；
- **「写文章」根本不需要 hexo 命令** —— 只是生成一个 `.md` 文件，只有 `generate` / `deploy` 才需要 hexo CLI。把两件事拆开，后面才能在没有博客项目的机器上先把内容跑通。

## 四、骨架用官方脚本生成，别手写

skill-creator 自带 `init_skill.py`：

```bash
python <skill-creator>/scripts/init_skill.py hexo-blog-publisher --path ~/.workbuddy/skills
```

它生成合规目录结构和带 `agent_created: true` 的 SKILL.md 模板。手写容易漏 front-matter 字段，也容易装错层级 —— **用户级**（`~/.workbuddy/skills/`）才跨所有项目可用，项目级只在当前工作区生效。

生成完第一件事：**删掉示例文件**（`scripts/example.py`、`references/api_reference.md`、`assets/example_asset.txt`）。留着它们打包校验也能过，但分发出去是一堆无关文件。

## 五、核心脚本：把 Windows 的坑提前埋进代码

`scripts/publish_post.js` 用 Node 写，只用内置 `fs` + `child_process`，零第三方依赖。

参数设计：

| 参数 | 作用 |
|---|---|
| `-b, --blog-root` | 博客根目录；缺省读 `HEXO_BLOG_ROOT` |
| `-t, --title` | 文章标题，自动做 YAML 转义 |
| `-f, --content-file` | 正文来源文件（另支持 `--content`、`--stdin`） |
| `--category` / `--tag` | 逗号分隔的分类链与标签 |
| `-p, --path` | `_posts` 下的栏目子目录 |
| `--share-copy` | 额外产出「在线阅读版」 |
| `--generate` / `--deploy` / `--push` | 显式开启的动作，后两者需 `--confirm-*` |
| `--json` / `--check` / `--dry-run` | 机器可读输出、环境体检、只看不写 |

真正花时间的不是参数解析，是这几处防御：

**① front-matter 转义。** 标题里出现冒号、引号、`#`、换行，直接拼进 YAML 就会炸或者被解释成注释。统一用双引号 + 转义（等价 `JSON.stringify` 的字符串规则），`categories` / `tags` 逐项处理。

**② 路径穿越。** `--path ../../Windows/System32` 如果不校验，文章会写到博客目录外。做法：`path.resolve` 之后强制检查目标是否仍在 `<blogRoot>/source/_posts` 之下，越界直接拒绝。

**③ 防覆盖。** 目标文件已存在就报错退出，除非显式 `--force`。博客文章一旦被静默覆盖，内容找不回来。

**④ `source` 目录污染。** 「在线阅读版」是给人读的副本，绝不能放进 `source/`，否则 Hexo 会当成第二篇文章生成。脚本里硬拦这个路径。

**⑤ BOM / CRLF。** Windows 产生的 `.md` 常带 UTF-8 BOM 和 CRLF，BOM 会让 front-matter 的 `---` 识别失败。读入统一清洗。

**⑥ 状态日志与数据分流。** `--json` 模式下 stdout 必须是**纯 JSON**，任何 `[ok] ...` 日志都改走 stderr，否则下游 `JSON.parse` 直接崩。

**⑦ hexo CLI 定位。** 顺序为：`--hexo-cmd` 显式指定 → 项目内 `node_modules/.bin/hexo.cmd` → 全局 `hexo`。每个候选执行 `version` 探测并**加超时**，把「命令找不到」和「构建失败」区分开。找不到时 `--check` 给出完整尝试列表，而不是丢一句模糊报错。

## 六、需求中途变了：加一条在线共享通道

开发过程中用户追加：「最好能推送到 WorkBuddy 上面，大家都能用」。

这里有个必须处理的细节：**Hexo 原文不能直接传上去**。那段 `--- / title: / date:` 在在线文档里会原样显示成一段 YAML 文本，阅读体验很糟。于是新增 `--share-copy`，生成「在线阅读版」：剥掉 front-matter、标题转成 H1、分类标签转成一行元信息。

同时查资料库空间权限，得到一个很现实的结论：唯一的团队空间角色是 **reader（只读）**，个人空间才是 owner 可写。所以「放进空间 = 大家都能看」这个假设不成立。跨系统集成时，权限模型要**先查再设计**，不能靠想当然。最终 SKILL.md 里把两种落地方式（争取团队空间写权限 / 改走公开页面）都写清，由用户选。

## 七、自检：三个真实踩过的坑

写了一个独立自检脚本，在临时沙盒目录里跑，不碰任何真实博客。13 项覆盖：中文标题与多级分类、front-matter 转义、BOM/CRLF 清理、防覆盖、路径穿越拦截、部署保护、在线阅读版、`source` 污染拦截等。

| 坑 | 现象 | 根因 | 修法 |
|---|---|---|---|
| `npx.cmd` 挂住 | 自检第一项通过后，整个进程被系统超时终止 | Windows 上 `npx.cmd --no-install hexo version` 不返回，且当时探测没加超时 | 从探测链移除（本来就是冗余候选：项目装了 hexo 走本地 `.bin` 即可），并给所有探测加 8 秒超时 |
| `--json` 的 stdout 混进日志 | 下游 `JSON.parse` 报错 | `[ok] ...` 状态日志走了 stdout | 所有状态日志改走 stderr，stdout 只留 JSON |
| 在线文档顶部出现原始 YAML | 页面显示 `--- / title: / date:` | Hexo front-matter 不是标准 Markdown 内容 | 生成独立的在线阅读版，剥 front-matter + 标题转 H1 |

第一个坑值得单独说：**自检脚本自己也需要超时保护**。被测对象挂住时，如果测试驱动没有超时，表现就是「测试莫名片卡死」，很容易误判成环境问题。这轮是先单独把候选命令逐个 `spawnSync` 加 timeout 复现，才定位到具体是哪一条命令。

## 八、SKILL.md 怎么写才好用

正文按「触发场景表 → 编号步骤 → 参数表 → 安全边界 → 常见坑」组织：

- **触发场景表**：把用户的口语说法映射到具体步骤编号。比如「顺便部署上线」→ 步骤 7 且需逐个确认；「帮我看看博客环境通不通」→ 只跑 `--check`。让执行者不必每次重新推理该做哪几步。
- **编号步骤**：每步给可以直接复制的命令，不写「配置好相关参数」这种空话。
- **安全边界**：单独成节，写死「默认只写本地；任何外部动作需显式要求」。
- **常见坑**：把上面三个坑和 Windows 编码、凭据管理器旧条目、`insteadOf` 地址改写等一并写进去。

## 九、打包与分发

```bash
python <skill-creator>/scripts/package_skill.py ~/.workbuddy/skills/hexo-blog-publisher <输出目录>
```

校验通过后会产出 zip。别人把包导入自己的 WorkBuddy，就能用同一套发布流程 —— 这是「大家都能用」的另一种解释（分发工具，而不是分发内容）。

另外补了一个 `hexo-publish.cmd` 包装器给不想敲长命令的场景，里面先 `chcp 65001` 解决 cmd 默认 GBK 把 UTF-8 中文帮助显示成乱码的问题。

## 十、闭环：用这个 skill 发布这篇博客

最有力的验证是拿真实任务跑一遍。这次的流程是：

1. 把「开发 skill 的全过程」整理成纯正文 Markdown（就是本文，无 front-matter）；
2. 调用刚写好的脚本，指定标题、分类、标签、栏目子目录，生成 `source/_posts/Skill开发/如何开发一个-workbuddy-skill.md`；
3. 本地独立仓库 `git init -b main`，只在这个仓库里设提交身份（**不动全局配置**）；
4. `git add -A && git commit`；
5. `git remote add origin` + `git push -u origin main`。

这里有个很隐蔽的坑：全局配置里有一条

```
url.https://github.com/.insteadof = git@github.com:
```

它会把 `git@github.com:用户名/仓库.git` 这种 scp 风格地址**悄悄改写成 HTTPS**，改完之后 SSH 密钥不参与，转而要求输入令牌，表现就是「明明配好了密钥却突然要密码」。绕开办法是不改全局配置，直接用 `ssh://git@ssh.github.com:443/...` 或 `ssh://git@github.com/...` 形式 —— 前者不匹配 `insteadOf` 规则，且走 443 端口，还能顺带避开网络封 22 的情况。

**结论**：能自己发布自己的工具，才算真的跑通了闭环。

## 十一、可复用的方法论清单

| # | 做法 | 原因 |
|---|---|---|
| 1 | 先跑只读环境体检，再写第一行代码 | 环境结论会直接改变架构（本项目就是「不写死博客根目录」） |
| 2 | `description` 写触发场景 | 决定 skill 会不会被加载，是命中率的关键 |
| 3 | 需求歧义用「全支持 + 安全默认」消解 | 少打断用户，又不牺牲边界 |
| 4 | 外部动作默认关闭，高危动作二次确认 | 发布、部署、推送都不可逆 |
| 5 | 自检要覆盖「不该发生的事」 | 防覆盖、路径穿越、`source` 污染比正常路径更容易出事 |
| 6 | 被测对象和测试驱动都要有超时 | 否则挂住时只会看到「莫名卡死」 |
| 7 | stdout 只放机器可读数据，日志走 stderr | 任何脚本被管道消费时的通用纪律 |
| 8 | 跨系统集成先查权限模型 | 「放进去 = 大家能看」经常不成立 |
| 9 | 最后拿真实任务跑一遍闭环 | 单元测试全过 ≠ 真实场景能用 |

## 附：完整命令

```bash
# 1. 生成骨架
python <skill-creator>/scripts/init_skill.py hexo-blog-publisher --path ~/.workbuddy/skills

# 2. 环境体检
node ~/.workbuddy/skills/hexo-blog-publisher/scripts/publish_post.js \
  -b "博客根目录" --check --json

# 3. 写文章（默认只写本地）
node ~/.workbuddy/skills/hexo-blog-publisher/scripts/publish_post.js \
  -b "博客根目录" -t "文章标题" -f "正文.md" \
  --category 技术笔记,WorkBuddy --tag skill,workbuddy \
  -p Skill开发 --share-copy "在线阅读版.md" --json

# 4. 自检 / 打包
node <docs>/hexo-publish-selftest.js
python <skill-creator>/scripts/package_skill.py \
  ~/.workbuddy/skills/hexo-blog-publisher <输出目录>

# 5. 推送到独立远端仓库
git -C "博客根目录" init -b main
git -C "博客根目录" config user.name "你的账号"
git -C "博客根目录" config user.email "账号绑定邮箱"
git -C "博客根目录" add -A && git -C "博客根目录" commit -m "post: 新增文章"
git -C "博客根目录" remote add origin ssh://git@ssh.github.com:443/用户名/仓库.git
git -C "博客根目录" push -u origin main
```

> 一点提醒：`push` 上去只是**源码存档**，不等于有网页。要让仓库变成能访问的博客，还需在仓库 Settings → Pages 配置构建；且非 `用户名.github.io` 的仓库会发布到子路径，`_config.yml` 里要补 `root: /仓库名/`，否则样式全丢。
