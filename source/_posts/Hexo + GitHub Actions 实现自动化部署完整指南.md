---
title: Hexo + GitHub Actions 实现自动化部署完整指南
date: 2025-08-29 10:00:00
tags:
  - Hexo
  - GitHub
  - CI/CD
categories:
  - 博客搭建
---
## 前言
在使用 Hexo 搭建个人博客时，当我们终于把hexo生成的页面挂载到github页面后，每次提交博客都需要使用`hexo clean && hexo g && hexo d`来部署很麻烦，可以借助github actions来实现流水线布置，我们每次提交源代码到github存放源码的分支上，一旦监听到源代码的push事件，流水线就会就会自动拉取环境自动部署，所以我们需要一个存放源代码的分支，以及我们的发布分支，可以使用git命令创建这两个分支，自动CI/CD,github需要密钥用来保证项目拉取和部署的权限，需要在本地生成ssh的密钥，公钥用来拉取项目，私钥用来部署推送更新页面，而且github对于npm安装有频率限制，我们可以通过配置国内镜像源+缓存配置，避免每次全量拉取，提高依赖安装以及部署的速度

## 一.分支规划与准备工作

为了清晰分离源代码和部署文件，我们需要创建两个分支：

- `source` 分支：存放 Hexo 源代码（包括 Markdown 文章、配置文件、主题文件等）
- `main` 分支：存放 Hexo 生成的静态网页文件，用于 GitHub Pages 展示

### 创建分支的具体操作

```bash
# 从当前分支创建并切换到 source 分支（源代码分支）
git checkout -b source

# 推送 source 分支到远程仓库
git push -u origin source

# 切换到 main 分支（部署分支）
git checkout main
git push -u origin main
```

## 二.SSH 密钥配置（核心权限控制）

要实现 GitHub Actions 的自动部署，必须配置 SSH 密钥对来授权 Actions 操作仓库的权限：

### 1. 生成专用 SSH 密钥对

```bash
# 生成专用密钥对，替换为你的邮箱地址
ssh-keygen -t ed25519 -C "your_email@example.com" -f ~/.ssh/hexo-deploy
```

执行后会在 `~/.ssh/` 目录下生成两个文件：
- `hexo-deploy`：私钥文件（用于部署时推送代码）
- `hexo-deploy.pub`：公钥文件（用于拉取仓库代码）

### 2. 配置公钥（Deploy keys）

公钥需要配置到你的 GitHub Pages 仓库（通常是 `username.github.io`）：

1. 打开 `hexo-deploy.pub` 文件，复制其中的全部内容
2. 进入你的 GitHub Pages 仓库 → Settings → Deploy keys
3. 点击 "Add deploy key"，粘贴公钥内容
4. 务必勾选 "Allow write access" 选项（允许推送权限）
5. 点击 "Add key" 完成配置
<img width="902" height="479" alt="屏幕截图 2025-08-29 190446" src="https://github.com/user-attachments/assets/9befac08-b11e-4cdb-9086-5c36af4d5eb0" />

### 3. 配置私钥（Repository secrets）

私钥需要配置到存放 Hexo 源代码的仓库：

1. 打开 `hexo-deploy` 文件，复制其中的全部内容
2. 进入你的源代码仓库 → Settings → Secrets and variables → Actions
3. 点击 "New repository secret"
4. 名称填写 `HEXO_DEPLOY_PRI`（必须与后续配置文件中一致）
5. 粘贴私钥内容，点击 "Add secret" 完成配置
<img width="1829" height="847" alt="image" src="https://github.com/user-attachments/assets/f5798732-4256-4222-9a5d-020afe645f53" />

<img width="1213" height="678" alt="image" src="https://github.com/user-attachments/assets/d75ce0f6-41da-402c-9682-6d9384c9f3bb" />

## 配置 GitHub Actions 工作流

在源代码仓库中创建工作流配置文件，实现自动部署的核心逻辑：

### 创建工作流文件

```bash
# 在 Hexo 项目根目录执行以下命令
mkdir -p .github/workflows
touch .github/workflows/deploy.yml
```

### 完整配置内容

编辑 `.github/workflows/deploy.yml` 文件，添加以下内容(没有这个目录需要自己手动创建)：

```yaml
name: Deploy Hexo Blog

on:
  push:
    branches: [ source ] # 方括号内填入拉取代码分支 当监听到push事件就会开启自动部署deploy

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout source code
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'  # 启用 npm 缓存，减少重复下载

    - name: Install dependencies and build
      run: |
        # 关键：切换为国内淘宝镜像，彻底解决 github 频率限制报错 429 限制
        npm config set registry https://registry.npmmirror.com/
        # 清除缓存，避免镜像切换不生效
        npm cache clean --force
        # 安装依赖（全局和项目依赖都用国内镜像）
        npm install -g hexo-cli --registry=https://registry.npmmirror.com/
        npm install --registry=https://registry.npmmirror.com/
        # 构建静态文件
        hexo clean
        hexo generate

    - name: Deploy to GitHub Pages
      uses: peaceiris/actions-gh-pages@v3
      with:
        deploy_key: ${{ secrets.HEXO_DEPLOY_PRI }}
        publish_dir: ./public
        publish_branch: main  # 与 _config.yml 中的 deploy.branch 一致
```

## 配置优化说明

### 1. 国内镜像源优化

通过配置淘宝 npm 镜像源（`https://registry.npmmirror.com/`），解决了 GitHub 服务器上安装依赖的两大问题：
- 访问速度慢（国内镜像加速）
- 访问频率限制（避免 npm 官方源的限流）

### 2. 缓存机制优化

- 缓存 `node_modules` 目录，避免每次部署重复安装依赖
- 基于 `package-lock.json` 的哈希值生成缓存键，只有依赖变更时才重新安装
- 使用 `npm ci` 替代 `npm install`，利用锁文件实现更快、更一致的依赖安装

### 3. 安全权限控制

- 采用 SSH 密钥对进行身份验证，比传统 token 更安全
- 私钥通过 GitHub Secrets 加密存储，不会暴露在代码中
- 细粒度权限控制：Deploy keys 仅对目标仓库有效，避免全局权限风险

## 使用方法

### 1. 初始配置提交

```bash
# 提交工作流配置文件
git add .github/workflows/deploy.yml
# 提交其他源代码文件
git add .
git commit -m "配置 GitHub Actions 自动部署"
git push origin source
```

### 2. 查看部署状态

1. 进入源代码仓库的 GitHub 页面
2. 点击顶部的 "Actions" 标签
3. 查看名为 "Hexo Auto Deploy" 的工作流运行状态
4. 点击具体运行记录可查看详细日志（便于排查错误）

### 3. 发布新文章流程

```bash
# 创建新文章
hexo new "我的新文章"

# 编辑文章（在 source/_posts 目录下）

# 提交到源代码仓库
git add .
git commit -m "发布新文章：我的新文章"
git push origin source
```

提交后，GitHub Actions 会自动触发部署流程，无需再执行手动部署命令，整个过程约 1-3 分钟。

## 常见问题排查

1. **部署失败提示权限错误**
   - 检查 SSH 密钥对是否匹配
   - 确认公钥是否勾选了 "Allow write access"
   - 检查私钥在 Secrets 中的名称是否为 `HEXO_DEPLOY_KEY`

2. **主题样式丢失**
   - 确保配置文件中 `submodules: true` 已设置（拉取主题子模块）
   - 检查主题是否正确添加到源代码仓库

3. **依赖安装失败**
   - 查看日志确认是否为网络问题
   - 尝试删除缓存后重新部署（可在 GitHub 仓库的 Actions 缓存设置中操作）

4. **部署成功但页面未更新**
   - 检查 GitHub Pages 配置的分支是否为 `main`
   - 浏览器强制刷新（Ctrl+Shift+R 或 Cmd+Shift+R）清除缓存

Hexo 博客的全自动部署流程，就可以只在source/_post目录下只提交md文件（必须是md后缀文件）到github项目的分支上就可以自动部署了
