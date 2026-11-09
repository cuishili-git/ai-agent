# 博客源码仓库

Hexo 博客源码。文章放在 `source/_posts/` 下，按栏目分子目录，每篇带 front-matter（title / date / categories / tags）。

## 目录

- `_config.yml` —— 站点基本配置
- `source/_posts/技术随笔/` —— 对话整理稿转成的博文
- `.gitignore` —— 忽略依赖、生成目录与部署缓存

## 本地构建

在装好依赖后执行：

```bash
npm install
hexo generate
hexo server
```

## 说明

本仓库只提交源码（文章 + 配置），不提交 `public/`、`node_modules/` 等生成物。
