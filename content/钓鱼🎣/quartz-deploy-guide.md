---
title: Quartz + GitHub Pages 部署全流程（从零开始）
date: 2026-07-14
tags:
  - quartz
  - github-pages
  - tutorial
  - blog
---

# Quartz + GitHub Pages 部署全流程（从零开始）

## 前提条件

| 工具 | 用途 | 安装方式 |
|------|------|---------|
| **Node.js** ≥ 22 | 运行时 | `brew install node` |
| **Git** | 版本控制 | `brew install git` |
| **GitHub CLI** (`gh`) | 仓库操作 + 认证 | `brew install gh` |
| **GitHub 账号** | 托管 + Pages | [github.com](https://github.com) |

> 本文以 `Bloodhound812.github.io` 仓库为例，域名会变成 `<你的用户名>.github.io`。

---

## 第一步：登录 GitHub CLI

```bash
gh auth login
```

按提示选择：
- `GitHub.com` → `HTTPS` → `Login with a web browser`
- 复制屏幕上的一次性验证码，浏览器打开链接粘贴

验证成功后，额外加上 `workflow` 权限（没有这个推送不了 Actions 文件）：

```bash
gh auth refresh -h github.com -s workflow
```

---

## 第二步：创建仓库

如果你还没有 `<用户名>.github.io` 仓库：

```bash
gh repo create <你的用户名>.github.io --public --clone
cd <你的用户名>.github.io
```

如果你已经有这个仓库（比如之前放过静态页面），直接克隆：

```bash
gh repo clone <你的用户名>.github.io
cd <你的用户名>.github.io
```

---

## 第三步：拉取 Quartz v5 源码

Quartz 建议从模板仓库创建，但我们是已有仓库，所以手动拉源码：

```bash
# 克隆 Quartz 最新版到临时目录
git clone --depth 1 https://github.com/jackyzha0/quartz.git /tmp/quartz-temp

# 把源代码复制到你的博客目录
rsync -av --exclude='.git' --exclude='.gitignore' --exclude='.github' /tmp/quartz-temp/ .

# 单独复制 .github（含部署工作流模板）和 .gitignore
cp /tmp/quartz-temp/.gitignore .gitignore
cp -r /tmp/quartz-temp/.github .github
```

> 如果你的仓库原有 `index.html`，删掉它：`rm index.html`

---

## 第四步：安装依赖 + 插件

```bash
npm install                    # 安装 npm 包
npx quartz plugin install      # 安装 Quartz 社区插件（46个）
```

---

## 第五步：配置站点

编辑 `quartz.config.default.yaml`，最少需要改：

```yaml
# 站点标题
pageTitle: 你的博客名

# 语言（中文用 zh-CN）
locale: zh-CN

# 域名（关键！决定生成的链接是否正确）
baseUrl: 你的用户名.github.io

# 页脚链接
footer:
  links:
    GitHub: https://github.com/你的用户名
```

详细配置项见 [[quartz 教程]]。

---

## 第六步：创建 GitHub Actions 部署工作流

删除 Quartz 自带的 workflows（给 quartz 项目本身用的）：

```bash
rm .github/workflows/build-preview.yaml
rm .github/workflows/ci.yaml
rm .github/workflows/deploy-preview.yaml
rm .github/workflows/deploy-v5.yaml
rm .github/workflows/docker-build-push.yaml
```

创建 `.github/workflows/deploy.yml`：

```yaml
name: Deploy Quartz site to GitHub Pages

on:
  push:
    branches:
      - main

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v6
        with:
          node-version: 24
      - name: Cache dependencies
        uses: actions/cache@v5
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
      - name: Cache Quartz plugins
        uses: actions/cache@v5
        with:
          path: .quartz/plugins
          key: ${{ runner.os }}-plugins-${{ hashFiles('quartz.lock.json') }}
          restore-keys: |
            ${{ runner.os }}-plugins-
      - name: Install Dependencies
        run: npm ci
      - name: Install Quartz plugins
        run: npx quartz plugin install
      - name: Build Quartz
        run: npx quartz build
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: public

  deploy:
    needs: build
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

---

## 第七步：添加内容

```bash
mkdir -p content/posts
```

**首页** `content/index.md`：

```markdown
---
title: 欢迎
---

# Hello, World!

欢迎来到我的博客。
```

**第一篇文章** `content/posts/hello.md`：

```markdown
---
title: 你好世界
date: 2026-07-14
tags:
  - blog
---

# 你好世界

这是第一篇。[[欢迎]] - 用双中括号做内部链接。
```

本地预览：

```bash
npx quartz build --serve
# 打开 http://localhost:8080
```

---

## 第八步：提交并推送

```bash
git add -A
git commit -m "feat: initialize Quartz blog"
git push origin main
```

---

## 第九步：启用 GitHub Pages

推送后去 GitHub 仓库页面：
- **Settings** → **Pages** → **Source** → 选 **GitHub Actions**
- 或者用命令行：`gh api repos/<用户名>/<仓库名>/pages -X PUT -F build_type=workflow`

推送即触发部署，去 **Actions** 标签页可以看到进度。完成后访问 `https://<用户名>.github.io/`。

---

## 日常使用

```bash
# 1. 写文章：在 content/posts/ 下新建 .md
# 2. 本地预览确认效果
npx quartz build --serve

# 3. 推送，自动部署
git add -A
git commit -m "feat: 新文章"
git push
```

---

## 关键文件速查

| 文件 | 说明 |
|---|---|
| `content/` | 你的文章目录 |
| `quartz.config.default.yaml` | 站点配置 |
| `.github/workflows/deploy.yml` | 自动部署工作流 |
| `quartz.lock.json` | 插件版本锁 |
| `.gitignore` | 已忽略 node_modules / public / .quartz / CLAUDE.md |
