# Hexo 博客项目恢复指南

## 1. 环境准备

首先需要安装以下软件：

1. 安装 [Node.js](https://nodejs.org/)
2. 安装 [Git](https://git-scm.com/)
3. 安装 Hexo CLI：
```bash
npm install -g hexo-cli
```

## 2. 克隆项目

1. 创建一个新文件夹（例如：Blog）
2. 打开命令行，进入该文件夹
3. 克隆项目：
```bash
git clone https://github.com/GoofySatoshi/GoofySatoshi.github.io.git .
git checkout source  # 如果源码在 source 分支
```

## 3. 安装依赖

在项目文件夹中运行：
```bash
npm install
```

这将安装所有必要的依赖，包括：
- hexo
- hexo-deployer-git
- hexo-theme-butterfly
- 其他插件

## 4. 恢复主题

确保 Butterfly 主题配置正确：
1. 检查 _config.yml 中的主题设置
2. 检查 _config.butterfly.yml 的配置

## 5. 部署设置

1. 配置 Git：
```bash
git config --global user.name "GoofySatoshi"
git config --global user.email "3033562734@qq.com"
```

2. 设置 SSH key（如果使用 SSH 部署）：
```bash
ssh-keygen -t rsa -b 4096 -C "3033562734@qq.com"
```
然后将生成的公钥添加到 GitHub

## 6. 测试运行

运行以下命令测试是否正常：
```bash
hexo clean
hexo generate
hexo server
```

访问 http://localhost:4000 查看效果

## 7. 部署

确认一切正常后，可以部署到 GitHub：
```bash
hexo deploy
```

## 注意事项

1. 确保 _config.yml 中的部署配置正确
2. 保持 node_modules 文件夹在 .gitignore 中
3. 主题配置文件 _config.butterfly.yml 需要重新配置
4. 图片等资源文件需要确保都已经同步 