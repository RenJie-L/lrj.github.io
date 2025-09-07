---
title: 利用 GitHub Pages + Hexo 快速搭建个人博客
date: 2023-06-25 20:02:55
tags: 杂记
---

拥有一个自己的博客站点是件非常酷炫的事情，利用 **GitHub Pages** 和 **Hexo** 可轻松实现这个目标。

> ps：GitLab 其实也支持 Pages 功能，但公司内网的该功能域名和证书支持均被禁用，实用性较低，故不展开。


## 一、准备工作

### 1. GitHub 创建个人仓库
1. 登录 GitHub 账号，点击页面右上角的 **New repository** 创建新仓库。
2. 仓库名必须为固定格式：`用户名.github.io`（将“用户名”替换为你的 GitHub 账号名称）。
   - 示例：若 GitHub 账号名为 `zuiai-kjl`，则仓库名应为 `zuiai-kjl.github.io`。


### 2. 安装必要工具
以下工具为搭建博客的基础依赖，需提前安装完成：
- **Git**：版本控制工具，用于后续代码提交与部署。（默认大家已安装，此处略过详细步骤）
- **Node.js**：Hexo 基于 Node.js 运行，**最低要求 v18 版本**，推荐安装最新稳定版（可通过 [Node.js 官网](https://nodejs.org/) 下载）。
- **Hexo**：博客框架，通过 npm 命令全局安装，终端执行以下命令：
  ```bash
  npm install -g hexo-cli
  ```


## 二、初始化 Hexo 博客

所有必备工具安装完成后，执行以下命令初始化博客项目：
1. 初始化博客文件夹（将 `<folder>` 替换为你想存放博客的文件夹名称，如 `my-blog`）：
   ```bash
   hexo init <folder>
   ```
2. 进入初始化好的博客目录：
   ```bash
   cd <folder>
   ```
3. 安装项目依赖：
   ```bash
   npm install
   ```

执行完成后，你将得到一个 Hexo 博客的基础雏形，包含默认的目录结构和示例文章。


## 三、写作：发布第一篇文章

### 1. 创建新文章
在博客根目录执行以下命令，创建一篇新的 Markdown 格式文章：
```bash
hexo new "this is title"
```
- 该命令会在博客目录的 `source/_posts` 文件夹下生成一个名为 `this is title.md` 的文件。


### 2. 编辑文章
- 推荐使用 **VS Code** 或其他支持 Markdown 的编辑器（如 Typora）打开 `source/_posts/this is title.md` 文件进行编辑。
- 编辑器通常支持 Markdown 实时预览，可直观查看文章排版效果。


### 3. 本地预览
文章编辑完成后，在博客根目录执行以下两条命令，即可在本地浏览器预览博客效果：
```bash
# 生成静态网页文件
hexo g
# 启动本地服务器
hexo s
```
- 执行 `hexo s` 后，终端会提示访问地址（默认是 `http://localhost:4000`），打开浏览器输入该地址即可查看新增的博文。


## 四、资源管理（以图片为例）

博客中常需插入图片，根据图片数量和大小，推荐两种资源管理方式：


### 1. 单篇文章独立资源文件夹（适合少量图片）
- **配置方式**：在博客根目录的 **站点配置文件**（`_config.yml`）中，将 `post_asset_folder` 设为 `true`：
  ```yaml
  post_asset_folder: true
  ```
- **效果**：之后使用 `hexo new "文章标题"` 创建文章时，会在 `source/_posts` 下同时生成一个与文章同名的文件夹（如 `this is title` 文件夹）。
- **使用**：将该文章所需的图片放入同名文件夹，在 Markdown 中通过相对路径引用（如 `![图片描述](./this is title/图片名.jpg)`）。


### 2. 图床（适合大量图片）
若文章包含大量图片，直接放在 GitHub 会导致网页加载缓慢（尤其国内访问 GitHub 不稳定），此时推荐使用 **图床** 存储图片：
- **原理**：将图片上传到图床平台，获取图片的外部链接，再在 Markdown 中通过外部链接引用图片（语法：`![图片描述](图片外部链接)`）。
- **推荐免费图床**：
  - [Superbed](https://www.superbed.cn/#pricing)
  - [imgtg](https://imgtg.com/)
  - [微博图床（Chrome 插件）](https://chromewebstore.google.com/detail/%E5%BE%AE%E5%8D%9A%E5%9B%BE%E5%BA%8A/pinjkilghdfhnkibhcangnpmcpdpmehk)（国内用户常用，需配合 Chrome 浏览器使用）


## 五、部署：将博客发布到 GitHub Pages

本地预览无误后，需将博客部署到 GitHub Pages，让其他人可通过互联网访问你的博客。以下介绍 **快速部署（推送生成好的网页文件）** 方式（推荐新手使用）。


### 1. 配置部署信息
1. 打开博客根目录的 **站点配置文件**（`_config.yml`），翻到文件末尾的 `deploy` 配置项，修改为以下内容：
   ```yaml
   deploy:
     type: git
     repo: 你的 GitHub 仓库完整路径（需带 .git 后缀）
     branch: master  # 部署到仓库的 master 分支（部分新仓库默认分支为 main，需对应修改）
   ```
   - 示例：若仓库地址为 `https://github.com/zuiai-kjl/zuiai-kjl.github.io`，则 `repo` 应填写 `https://github.com/zuiai-kjl/zuiai-kjl.github.io.git`。

2. 安装 Git 部署插件：在博客根目录执行以下命令，用于支持 Hexo 向 GitHub 推送文件：
   ```bash
   npm install hexo-deployer-git --save
   ```


### 2. 执行部署命令
在博客根目录依次执行以下三条命令，完成部署：
```bash
# 清理本地生成的旧静态文件（避免缓存问题）
hexo clean
# 重新生成最新的静态网页文件
hexo generate
# 部署到 GitHub Pages
hexo deploy
```

- 部署完成后，可进入 GitHub 仓库的 **Actions** 页面查看部署进度（若显示绿色对勾，则部署成功）。
- 访问博客：部署成功后，等待 1-2 分钟，在浏览器中输入 `https://用户名.github.io`（如 `https://zuiai-kjl.github.io`），即可访问你的公开博客。


> ps：通过 `hexo deploy` 部署到 GitHub 的文件是 **Markdown 转化后的静态 HTML 文件**，而非原始的 Markdown 源码。若本地文件丢失或需在其他电脑修改博客，需额外备份源码（可通过 Git 单独管理源码分支）。


## 六、美化配置：更换 Hexo 主题

Hexo 拥有丰富的开源主题，可通过更换主题快速美化博客。以下以 **Fluid 主题** 为例（颜值高、可配置项丰富、文档齐全）。


### 1. 安装 Fluid 主题
在博客根目录执行以下命令，通过 npm 安装 Fluid 主题：
```bash
npm install --save hexo-theme-fluid
```


### 2. 启用主题
1. 打开博客根目录的 **站点配置文件**（`_config.yml`），修改 `theme` 和 `language` 配置项：
   ```yaml
   theme: fluid  # 指定使用 Fluid 主题
   language: zh-CN  # 指定主题显示语言为中文（可按需修改为 en 等）
   ```

2. 复制主题配置文件：
   - 进入 `node_modules/hexo-theme-fluid` 目录，找到主题自带的 `_config.yml` 文件（该文件包含主题的所有配置项）。
   - 在博客根目录新建一个名为 `_config.fluid.yml` 的文件，将上述 `_config.yml` 的内容复制进去（后续修改主题配置只需编辑 `_config.fluid.yml`，避免直接修改主题源码目录的文件，方便主题更新）。


### 3. 预览主题效果
在博客根目录执行以下命令，本地预览美化后的博客：
```bash
hexo clean
hexo g
hexo s
```


### 4. 自定义主题
Fluid 主题支持丰富的自定义配置（如导航栏、侧边栏、评论功能等），可参考官方文档进行配置：
- [Fluid 主题官方文档](https://hexo.fluid-dev.com/docs/guide/#%E5%85%B3%E4%BA%8E%E6%8C%87%E5%8D%97)

更多 Hexo 主题可在 [Hexo 官方主题库](https://hexo.io/themes/) 浏览选择。


## 七、写在最后

- 示例博客：笔者花费小半天搭建的博客地址 → [https://renjie-l.github.io/](https://renjie-l.github.io/)
- 参考文档：
  - [GitHub Pages 快速入门](https://docs.github.com/cn/pages/quickstart)
  - [Hexo 官方文档（中文）](https://hexo.io/zh-cn/docs/)