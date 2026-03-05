# Dify CI/CD 自动部署配置指南

## 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                        GitOps 工作流                            │
└─────────────────────────────────────────────────────────────────┘

   本地开发                        GitHub                         生产服务器
┌─────────────┐            ┌─────────────┐              ┌─────────────┐
│  修改代码    │            │             │              │             │
│  git push   │───────────▶│  GitHub仓库  │─────────────▶│  Webhook    │
│             │            │             │              │  更新脚本    │
└─────────────┘            └──────┬──────┘              └──────┬──────┘
                                 │                            │
                                 ▼                            ▼
                          ┌─────────────┐              ┌─────────────┐
                          │ GitHub Actions            │  拉取镜像    │
                          │ 自动构建     │              │  重启容器    │
                          │ 推送镜像     │              │             │
                          └─────────────┘              └─────────────┘
                                 │
                                 ▼
                          ┌─────────────┐
                          │  GHCR       │
                          │  镜像仓库   │
                          └─────────────┘
```

## 前置条件

1. GitHub 仓库：`https://github.com/GongJiYang/dify-x`
2. GitHub 账号已启用 Packages
3. 服务器上已安装 Docker 和 Docker Compose

## 配置步骤

### 1. 确认 GitHub Actions 工作流

工作流文件已创建：`.github/workflows/custom-build-push.yml`

此工作流会在以下情况自动触发：
- 推送代码到 `main` 分支
- 手动触发（在 GitHub Actions 页面）

### 2. 启用 GitHub Container Registry

1. 访问你的仓库设置：`https://github.com/GongJiYang/dify-x/settings/packages`
2. 确保 Packages 功能已启用
3. 镜像将发布到：`ghcr.io/gongjiyang/dify-x-web:latest`
4. 镜像将发布到：`ghcr.io/gongjiyang/dify-x-api:latest`

### 3. 首次手动触发构建

**方法1：通过 GitHub 网页界面**
1. 访问：`https://github.com/GongJiYang/dify-x/actions`
2. 点击左侧 "Custom Build and Push" 工作流
3. 点击 "Run workflow" 按钮
4. 选择 `main` 分支，点击 "Run workflow"

**方法2：推送代码触发**
```bash
cd /root/dockge/compose/dify-x
git add .github/workflows/custom-build-push.yml
git commit -m "Add CI/CD workflow"
git push origin main
```

### 4. 服务器配置（已完成）

已完成的配置：
- ✅ Docker Compose 使用 GHCR 镜像
- ✅ Webhook 脚本已更新，支持拉取新镜像
- ✅ 静态资源挂载保留（logo、i18n 可直接修改）

### 5. 登录 GitHub Container Registry

在生产服务器上配置镜像拉取权限：

```bash
# 创建 GitHub Personal Access Token
# 1. 访问：https://github.com/settings/tokens
# 2. 生成新 Token，勾选 `read:packages` 权限
# 3. 复制 Token

# 登录 GHCR（替换 YOUR_TOKEN）
echo "YOUR_GITHUB_TOKEN" | docker login ghcr.io -u GongJiYang --password-stdin
```

**注意**：如果仓库是公开的，可能不需要登录。如果遇到权限问题，请执行上述登录命令。

## 使用流程

### 日常开发工作流

```bash
# 1. 本地修改代码
cd /path/to/dify-x

# 2. 提交并推送
git add .
git commit -m "feat: 更新功能"
git push origin main

# 3. GitHub Actions 自动构建（约5-10分钟）

# 4. 生产服务器自动更新（webhook 触发）
# 新镜像会自动拉取并重启容器
```

### 手动更新生产服务器

如果 webhook 失败，可以手动更新：

```bash
cd /root/dockge/compose/dify-x-stack

# 拉取最新镜像
docker compose pull

# 重启容器
docker compose up -d
```

## 镜像命名规则

构建的镜像标签：
- `latest` - 最新的 main 分支构建
- `main-<sha>` - 带提交 SHA 的标签
- `<run_number>` - 运行序号

示例：
```yaml
ghcr.io/gongjiyang/dify-x-web:latest
ghcr.io/gongjiyang/dify-x-web:main-abc123
ghcr.io/gongjiyang/dify-x-web:42
```

## 快速修改（无需重新构建）

以下文件修改后**立即生效**，无需等待 CI/CD：

| 类型 | 路径 | 说明 |
|------|------|------|
| Logo | `web/public/logo.svg` | 直接修改，刷新即生效 |
| 图片 | `web/public/images/*` | 直接修改 |
| 翻译 | `web/i18n/*.json` | 直接修改 |
| 环境变量 | `docker/.env` | 修改后重启容器 |

**快速修改流程**：
```bash
# 1. 修改文件
vi /root/dockge/compose/dify-x/web/public/logo.svg

# 2. 提交到 GitHub（可选，用于备份）
git add web/public/logo.svg
git commit -m "Update logo"
git push origin main

# 3. 生产环境立即生效（无需等待构建）
```

## 需要重新构建的修改

以下修改**必须**通过 CI/CD 重新构建：

| 类型 | 示例 |
|------|------|
| React 组件 | `web/app/**/*.tsx` |
| 样式修改 | `web/app/**/*.css` |
| 新增页面 | `web/app/**/page.tsx` |
| API 逻辑 | `api/**/*.py` |
| 依赖更新 | `package.json`, `pyproject.toml` |

## 故障排查

### 1. 镜像拉取失败

```bash
# 检查登录状态
docker login ghcr.io

# 检查镜像是否存在
curl -X GET https://ghcr.io/v2/gongjiyang/dify-x-web/tags/list
```

### 2. 容器启动失败

```bash
# 查看日志
docker compose logs web
docker compose logs api

# 检查配置
docker compose config
```

### 3. Webhook 未触发

```bash
# 检查 webhook 日志
tail -f /var/log/webhook-updates.log

# 手动触发 webhook 脚本
sh /root/dockge/compose/webhook-updater/update-wrapper.sh
```

## 性能优化建议

1. **镜像缓存**：GitHub Actions 会自动缓存构建层，加快构建速度
2. **构建频率**：避免频繁推送，合并多个修改为一次提交
3. **静态资源**：小改动直接修改源文件，无需触发构建

## 监控和日志

### 查看构建状态
```bash
# 访问 GitHub Actions 页面
https://github.com/GongJiYang/dify-x/actions
```

### 查看容器状态
```bash
cd /root/dockge/compose/dify-x-stack
docker compose ps
docker compose logs -f --tail=100 web
```

### 查看当前镜像版本
```bash
docker images | grep dify-x
```

## 成本说明

- **GitHub Container Registry**：公开仓库免费，私有仓库有存储限制
- **GitHub Actions**：公开仓库免费，私有仓库每月有 2000 分钟免费额度
- **存储成本**：每个镜像约 500MB-1GB

## 安全建议

1. **敏感信息**：不要在代码中硬编码密钥
2. **Token 管理**：定期更新 GitHub Personal Access Token
3. **访问控制**：限制仓库写入权限
4. **镜像扫描**：定期检查镜像安全漏洞

## 联系支持

如有问题，请检查：
1. GitHub Actions 运行日志
2. Webhook 更新日志：`/var/log/webhook-updates.log`
3. Docker 容器日志：`docker compose logs`
