# mender-deviceconfig-liubiao

自维护的 Mender `deviceconfig` 服务（设备配置），基于上游 [mendersoftware/mender-server](https://github.com/mendersoftware/mender-server) v4.1.4，只为让开源版能把审计事件交给自建审计服务 [mender-auditlogs-liubiao](https://git.skyrisai.com/yuzus/mender-auditlogs-liubiao)。

## 仓库约定

| 项目 | 内容 |
|---|---|
| 基线 | tag `upstream-v4.1.4` = 上游 `v4.1.4`（`89f60e8f1d9f2923e66f45d3bc8d21deb5fed475`），取自快照 fork [Lenoud/mender-server](https://github.com/Lenoud/mender-server)，保留上游完整历史 |
| 改动方式 | 除 `liubiao/` 目录外，仓库内容与上游 tag 完全一致。**不直接修改上游源码**，修改写成 `liubiao/patches/*.patch`，构建时应用 |
| 构建 | Jenkins `mender-deviceconfig-liubiao-build`（[jenkins-job](https://github.com/Lenoud/jenkins-job) `recipes/mender`），版本锁固定本仓库 commit 与补丁 SHA256 |
| 镜像 | `harbor.skyrisai.com/develop/mender-deviceconfig-liubiao:jenkins-release-v4-1-4-liubiao1-<commit前12位>-<时间>` |
| 部署 | `k8s_services` 的 `mender/values-prod.yaml` 中 `deviceconfig.image` |

`git diff upstream-v4.1.4` 只应看到 `liubiao/` 目录；Jenkins 构建时会核对这一点。

## 补丁

| 补丁 | SHA256 | 原因 |
|---|---|---|
| `0001-deviceconfig-allow-empty-tenant-auditlog.patch` | `3a51a29754099093a1df8d4a24d4202557014cdc0e3c2403729b5c3d21e83314` | 上游 `client/workflows/client.go` 提交审计事件前要求身份带租户（`id.Tenant == ""` 即报错 `Context lacking tenant identity`）。开源版单租户没有租户 ID，打开 `DEVICECONFIG_ENABLE_AUDIT` 后事件提交必定失败，设备配置操作随之失败（2026-11-19 生产实测）。补丁只保留“必须有身份”的检查，空租户原样提交；补充单测覆盖无租户身份 |

覆盖的审计事件：修改配置、下发配置。

## 本地验证补丁

```bash
git apply --check liubiao/patches/*.patch
git apply liubiao/patches/*.patch
(cd backend && go test ./services/deviceconfig/client/workflows/)
git checkout -- backend
```

## 升级上游

1. 先同步 fork `Lenoud/mender-server` 拿到新 tag，再把新 tag 合入本仓库 `main`（保留上游历史，`liubiao/` 不会冲突），新建 `upstream-<tag>` 基线 tag。
2. `git apply --check` 确认补丁仍能应用；不能应用时按新代码重写补丁并重新跑单测。上游 main 已删除 deviceconnect 的审计代码（`f0fa5736`），升级到基于 main 的版本时要重新评估。
3. 更新 jenkins-job `recipes/mender/versions.json` 中的 commit、基线 commit 与补丁 SHA256，重新生成 Job XML。
