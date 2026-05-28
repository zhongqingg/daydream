# daydream

Donchy 的个人博客，基于 Hugo + paper 主题。

## 目录结构

```
daydream/
├── content/          # 博客文章
│   ├── post/         # 生活
│   ├── work/         # 工作
│   └── paper/        # 学术
├── layouts/          # 模板覆盖
├── assets/           # CSS 源文件
├── static/img/       # 图片
└── themes/paper/     # Hugo 主题 (git submodule)
```

## 本地开发

```bash
# 本地启动预览
hugo server
```

## 自动部署

推送 `main` 分支后，GitHub Actions 自动构建并部署到 [zhongqingg.github.io](https://github.com/zhongqingg/zhongqingg.github.io)。
