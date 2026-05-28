---
title: "Hugo + GitHub Actions 自动部署到 GitHub Pages 全程记录"
date: 2026-05-28
tags: ["Hugo", "GitHub Actions", "CI/CD", "部署"]
---

## 仓库结构

```
daydream/                      # 源码仓库（需 git push）
├── daydream/                  # Hugo 站点根目录
│   ├── content/
│   ├── layouts/
│   ├── assets/
│   └── themes/paper/          # git submodule
└── .github/workflows/deploy.yml
```

```
zhongqingg.github.io/          # 部署仓库（由 Actions 自动写入）
```

> 两个独立的 GitHub 仓库。源码仓库负责存储 Hugo 源文件，部署仓库存放构建产物。

---

## 配置过程

### 1. 创建 Personal Access Token

1. 打开 https://github.com/settings/tokens → **Generate new token → Fine-grained token**
2. **Token name**: `PAT`
3. **Repository access**: Only select repositories → 勾选 `daydream` 和 `zhongqingg.github.io`
4. **Permissions → Contents**: Read and write
5. 点 **Generate token**，复制生成的 token

### 2. 添加 Token 到源码仓库 Secrets

打开 https://github.com/zhongqingg/daydream/settings/secrets/actions → **New repository secret**
- **Name**: `PAT`
- **Secret**: 粘贴刚才复制的 token

### 3. Workflow 文件

位置：`.github/workflows/deploy.yml`

```yaml
name: Deploy Hugo to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ./daydream

    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: '0.139.2'

      - run: hugo --minify

      - uses: peaceiris/actions-gh-pages@v4
        with:
          personal_token: ${{ secrets.PAT }}
          external_repository: zhongqingg/zhongqingg.github.io
          publish_branch: main
          publish_dir: ./daydream/public
```

---

## 踩坑记录

### 路径问题

```yaml
defaults:
  run:
    working-directory: ./daydream   # Hugo 在此目录下运行
```

`hugo --minify` 的输出在 `./daydream/public/`（相对工作区根目录），因此：

```yaml
publish_dir: ./daydream/public       # ✅ 正确
publish_dir: ./daydream/daydream/public  # ❌ 错误（多套了一层）
```

### 子模块

GitHub Actions 默认不拉子模块。主题 `themes/paper` 是 git submodule，必须添加：

```yaml
- uses: actions/checkout@v4
  with:
    submodules: recursive
```

否则 `resources.Get "main.css"` 返回 nil，构建报错。

### Hugo 版本

不要用 `hugo-version: 'latest'`，本地与云端版本不一致会导致各种兼容问题。锁定为本地测试通过的版本：

```yaml
hugo-version: '0.139.2'
```

### 兼容性

Hugo v0.139.2 中已知的细节：

| 语法 | 状态 | 替代方案 |
|------|------|----------|
| `site.DisqusShortname` | 已弃用但可用 | `site.Config.Services.Disqus.Shortname`（v0.92+） |
| `site.LanguageCode` | 已弃用但可用 | `site.Language.Locale`（v0.139.2 不可用，需新版） |
| `site.Language.Locale` | v0.139.2 不支持 | 继续用 `site.LanguageCode` |

> 弃用警告不影响运行，可以暂时忽略。

---

## 日常使用

只需在本地编辑内容，然后：

```bash
git add -A
git commit -m "xxx"
git push
```

GitHub Actions 自动完成：拉取源码 → 安装 Hugo → `hugo --minify` 构建 → 推送到 `zhongqingg.github.io`。

大约 **2~3 分钟**后博客更新生效。

---

## 完整流程示意图

```
本地: git push daydream
        │
        ▼
GitHub Actions 启动 Ubuntu 虚拟机
        │
        ▼
actions/checkout（拉取源码 + 子模块）
        │
        ▼
peaceiris/actions-hugo（安装 Hugo v0.139.2）
        │
        ▼
hugo --minify（执行构建，输出 ./daydream/public/）
        │
        ▼
peaceiris/actions-gh-pages（推送 public/ 到 zhongqingg.github.io）
        │
        ▼
zhongqingg.github.io 更新 → CDN 缓存刷新 → 博客生效
```
