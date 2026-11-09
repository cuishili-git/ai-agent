---
title: "Git 登录与 Hexo 博文自动发布"
date: 2026-11-09 12:02:13
categories:
  - "技术笔记"
  - "Git"
tags:
  - "hexo"
  - "git"
  - "workbuddy"
---
## 背景

本次会话先排查了「下载 Git 之后如何登录」，随后开发了一个把本地对话内容发布成 Hexo 博文、并可同步到 WorkBuddy 在线空间的 skill。这里把两段结论沉淀成一篇可发布的笔记。

## 结论：Git 本身不需要登录

Git 是装在电脑上的本地版本控制软件，没有账号体系。日常说的「登录 Git」其实混了三件事：

| 事项 | 在哪做 | 说明 |
|---|---|---|
| 配置提交署名 | 本地命令行 | `user.name` / `user.email`，只是给 commit 盖作者章 |
| 注册并登录网站 | 浏览器 | GitHub / Gitee 等托管平台的账号 |
| 建立认证凭证 | 本地 + 网站设置 | SSH 密钥或个人访问令牌，这一步才叫「让 Git 能推拉」 |

本机实测结果：Git `2.55.0.windows.3` 已装好，`~/.ssh/id_ed25519` 已存在，`ssh -T git@github.com` 返回 `Hi cuishili-git! You've successfully authenticated` —— 认证通道早就通了，账号是 `cuishili-git`。唯一缺的是提交署名。

还有一条容易忽略的全局配置：

```
url.https://github.com/.insteadof=git@github.com:
```

它会把 `git@github.com:` 开头的 SSH 地址强行改写成 `https://github.com/`。后果是以为在用 SSH 密钥，实际走 HTTPS，然后莫名弹出要密码或报认证失败。若目标是走 SSH，建议清掉；若本来就用 HTTPS + 令牌，留着也行，但要明白它改写了地址。

## 操作步骤

1. 补齐署名，邮箱用平台账号绑定的那个，否则贡献统计和头像对不上：

   ```bash
   git config --global user.name "你的名字"
   git config --global user.email "绑定的邮箱"
   ```

2. 认证优先 SSH：`ssh-keygen -t ed25519 -C "邮箱"`，把 `.pub` 内容粘到平台 SSH Keys 设置，再用 `ssh -T` 验证。
3. HTTPS 路线必须生成个人访问令牌（PAT），主流平台已不接受账号密码；令牌只显示一次，要立刻存好。

## 坑与权衡

| 坑 | 处理 |
|---|---|
| 输密码/令牌时屏幕不回显 | 正常行为，照常粘贴回车 |
| 改过令牌仍报错 | Windows 凭据管理器里删掉旧的 `git:https://...` 条目 |
| 公司网络封 22 端口 | `~/.ssh/config` 配 `ssh.github.com:443`，别急着换回 HTTPS |

## Hexo 发布 skill 的设计要点

- 直接写 front-matter + 正文到 `source/_posts`，不调用 `hexo new`，因为后者不接受自定义子目录和多级分类；
- 标题含冒号或引号时用 `JSON.stringify` 生成 YAML 双引号字符串，零依赖解决转义；
- 默认只落盘，构建 / 部署 / 推送 / 在线上传都要显式开关，部署与推送还要求 `--confirm-*` 二次确认；
- 同步到 WorkBuddy 在线空间前要剥掉 Hexo front-matter，否则在线文档里会显示成一段原始 YAML 文本；
- Windows 下 `npx.cmd --no-install hexo version` 会挂住不返回，已从命令探测链中移除，并给所有探测加了超时。
