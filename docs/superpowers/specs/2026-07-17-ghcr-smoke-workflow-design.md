# GHCR 权限快速验证工作流设计

## 目标

新增一个独立、仅手动触发的 GitHub Actions 工作流，用最短时间验证当前仓库的 `GITHUB_TOKEN` 是否能够在 `ghcr.io/ccdave/llmnex-new-api` 创建并推送容器 Package。该工作流不构建 new-api 项目，不替代正式镜像发布工作流。

## 工作流边界

- 文件：`.github/workflows/llmnex-ghcr-smoke.yml`
- 触发方式：仅 `workflow_dispatch`
- Runner：`ubuntu-latest`
- 权限：`contents: read`、`packages: write`
- 认证：`${{ github.actor }}` 和 `${{ secrets.GITHUB_TOKEN }}`
- 目标镜像：`ghcr.io/ccdave/llmnex-new-api:permission-smoke`
- 不检出仓库代码，不安装项目依赖，不初始化 Buildx，不使用 Actions 缓存。

## 验证流程

1. 使用固定 SHA 的 `docker/login-action` 登录 `ghcr.io`。
2. 在 Runner 临时目录生成一个仅包含 `FROM scratch` 和 OCI source 标签的最小 Dockerfile。
3. 使用 Runner 自带的 Docker CLI 构建空镜像。
4. 推送固定的 `permission-smoke` 标签。
5. 输出明确的成功信息和目标镜像名称。

OCI source 标签固定关联当前仓库：

```text
org.opencontainers.image.source=https://github.com/${GITHUB_REPOSITORY}
```

## 结果解释

- 登录失败：`GITHUB_TOKEN` 未正确提供或 GHCR 认证失败。
- 登录成功但推送失败：镜像构建不是原因，故障位于 GHCR Package 创建权限或仓库 installation 关联。
- 推送成功：`llmnex-new-api` Package 已创建，可以进入 Package settings 配置 `Manage Actions access`，随后重新运行正式发布工作流。

## 副作用与清理

验证成功会留下 `permission-smoke` 标签。重复执行只覆盖同一个标签，不创建递增测试标签。测试标签暂时保留，待正式镜像成功发布后再由仓库所有者从 Package 页面删除。

## 验收标准

- 工作流只能手动触发。
- 一次执行无需构建项目即可完成 GHCR 登录和 Push 验证。
- 使用与正式工作流相同的个人命名空间、Package 名称和 `GITHUB_TOKEN` 权限模型。
- 失败日志能够区分登录失败与 Package Push 失败。
- 不修改现有 `.github/workflows/llmnex-ghcr.yml`。
