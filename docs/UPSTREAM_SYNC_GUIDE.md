# 上游更新与合并指南

本文档用于统一处理本仓库与上游 `cmliu/edgetunnel` 的同步、冲突分析和定制重放。

后续人工或 AI/Codex 执行上游更新时，必须优先阅读并遵循本文档，避免覆盖本地部署配置和业务定制。

## 1. 仓库关系

- 上游仓库：`cmliu/edgetunnel`
- 上游分支：`main`
- 本地开发分支：`main`
- 当前本地 `origin`：执行 `git remote -v` 确认，不得根据历史记录猜测
- 自动同步工作流：`.github/workflows/sync.yml`

开始更新前必须执行：

```bash
git status --short --branch
git remote -v
```

如果工作区存在未提交修改，先提交或暂存，不得直接开始合并。

## 2. 必须保留的本地定制

### 2.1 Wrangler、KV 与 Assets 配置

必须保留 `wrangler.toml` 中的部署配置，重点检查：

```toml
[assets]
directory = "./public"
binding = "ASSETS"

[[kv_namespaces]]
binding = "KV"
```

KV 的实际 `id` 属于当前部署环境配置，更新时不得使用上游值覆盖。

### 2.2 本地后台资源

必须保留以下目录及其全部资源：

```text
public/edt-pages/
```

`_worker.js` 必须继续通过 `env.ASSETS.fetch()` 读取本地后台资源，而不是依赖远程 `edt-pages.github.io`。

需要保留的关键行为：

- `/login` 返回 `public/edt-pages/login.html`
- `/admin` 返回 `public/edt-pages/admin.html`
- 未配置管理员密码时返回 `noADMIN.html`
- 未配置 KV/UUID 时返回 `noKV.html`
- `/edt-pages/vendor/*` 从本地 Assets 返回
- HTML 页面禁止缓存，Vendor 静态资源可长期缓存

当前关键标记：

```text
本地后台资源前缀
返回本地静态资源
返回本地后台页面
```

### 2.3 自定义节点转换服务

必须保留：

```text
https://subapi.trueheart.cc
```

需要同时检查：

- `_worker.js` 中默认 `SUBAPI`
- `public/edt-pages/admin.html` 中相关默认值或提示

## 3. 标准更新原则

核心原则：

> 以上游最新 `_worker.js` 为基线，只重新植入必要的本地定制。

原因：上游会频繁大幅修改 `_worker.js`。如果在旧文件上逐块接受冲突，很容易遗漏上游的新功能、安全修复或变量调整。

推荐顺序：

1. 获取最新上游代码。
2. 查看上游更新文件和提交记录。
3. 尝试普通合并。
4. 如果 `_worker.js` 冲突，以最新版上游文件为基线。
5. 重新植入本地后台 Assets 路由。
6. 重新植入自定义 `SUBAPI`。
7. 保留本地 `wrangler.toml` 和 `public/edt-pages/**`。
8. 完成检查后提交。

## 4. 获取上游更新

确认或添加上游远端：

```bash
git remote get-url upstream
```

如果不存在：

```bash
git remote add upstream https://github.com/cmliu/edgetunnel.git
```

获取最新代码：

```bash
git fetch upstream main
```

查看更新：

```bash
git log --oneline --decorate main..upstream/main
git diff --stat main...upstream/main
git diff --name-status main...upstream/main
```

## 5. 执行合并

推荐先不自动提交：

```bash
git merge --no-commit --no-ff upstream/main
```

### 5.1 无冲突

如果 Git 自动合并成功，仍须执行第 7 节的完整检查。自动合并成功不代表本地定制仍然有效。

### 5.2 `_worker.js` 发生冲突

先确认冲突：

```bash
git status --short
git diff --name-only --diff-filter=U
```

处理原则：

- 上游网络、协议、配置读取和安全逻辑采用最新版本。
- 本地仅恢复第 2 节列出的必要定制。
- 不恢复已经被上游替代的旧实现。
- 不为了减少 diff 而保留过时的上游代码。

可先使用上游版本作为基线：

```bash
git checkout --theirs _worker.js
git add _worker.js
```

然后编辑 `_worker.js`，重新植入：

1. `const 本地后台资源前缀 = '/edt-pages';`
2. Vendor 静态资源路由。
3. `/login`、`/admin`、`noADMIN`、`noKV` 的本地页面响应。
4. `返回本地静态资源()`。
5. `返回本地后台页面()`。
6. `SUBAPI: "https://subapi.trueheart.cc"`。

注意：`git checkout --theirs` 只能在已经确认以“上游为基线”的目标文件上使用，不得对整个仓库执行。

## 6. 禁止操作

除非明确决定放弃所有本地定制，否则禁止：

```bash
git reset --hard upstream/main
git push --force
git checkout --theirs .
git checkout --ours .
```

禁止事项：

- 不得直接覆盖 `wrangler.toml` 的 KV ID。
- 不得删除 `public/edt-pages/**`。
- 不得恢复远程 `edt-pages.github.io` 后台依赖。
- 不得将 `SUBAPI` 恢复为上游默认地址。
- 不得只删除冲突标记而不理解双方逻辑。
- 不得顺手重构与本次同步无关的代码。

## 7. 合并后检查

### 7.1 Git 与语法检查

```bash
git diff --check
node --check _worker.js
rg -n '^(<<<<<<<|=======|>>>>>>>)' .
```

预期：

- `git diff --check` 无输出。
- `node --check` 正常退出。
- 不存在冲突标记。

### 7.2 定制标记检查

```bash
rg -n '本地后台资源前缀|返回本地后台页面|返回本地静态资源|subapi\.trueheart\.cc|\[assets\]|binding = "ASSETS"|\[\[kv_namespaces\]\]' \
  _worker.js public/edt-pages/admin.html wrangler.toml
```

必须确认：

- `_worker.js` 使用上游最新版本号。
- 本地后台三个关键函数/常量仍存在。
- `_worker.js` 和后台页面仍指向 `subapi.trueheart.cc`。
- `ASSETS` 与 `KV` 绑定仍存在。

### 7.3 相对上游差异检查

```bash
git diff --stat upstream/main --
git diff upstream/main -- _worker.js
```

预期差异应主要集中于：

- `_worker.js` 的本地后台适配与 `SUBAPI`
- `public/edt-pages/**`
- `wrangler.toml`

如果出现大量与本地定制无关的 `_worker.js` 差异，应暂停提交并重新分析。

## 8. 提交规范

确认检查通过后：

```bash
git add _worker.js wrangler.toml public/edt-pages .gitignore CHANGELOG README.md
git status --short
git commit -m "合并上游最新更新并保留本地后台、KV及节点转换定制 或 以上游最新版本为基线重放最小定制补丁"
```

提交信息需使用中文，并至少说明：

- 问题或需求描述
- 修复或实现思路
- 复现路径（可选）

推送前再次确认目标仓库：

```bash
git remote -v
git branch -vv
```

不得将用于其他组织或账号的提交推送到错误的 `origin`。

## 9. GitHub Actions 同步失败处理

自动同步出现：

```text
CONFLICT (content): Merge conflict in _worker.js
Automatic merge failed
```

说明本地与上游修改了相同区域。此时自动任务无法代替人工判断，应按本文档第 4～8 节人工合并。

工作流中的 `Sync check` 只是兜底提示。定位错误时必须查看前一个 `Sync upstream changes` 步骤的原始日志，不能仅依据最后的 `exit 1` 判断原因。

## 10. AI/Codex 执行清单

后续要求 AI/Codex 更新上游时，应明确要求：

```text
请先阅读 docs/UPSTREAM_SYNC_GUIDE.md，严格按文档执行上游更新。
必须保留本地后台 Assets、KV 配置和 subapi.trueheart.cc。
以上游最新 _worker.js 为基线，只重放必要的最小定制补丁。
完成后执行文档中的语法、冲突标记和定制标记检查。
```

AI/Codex 必须：

- 先确认工作区、远端和分支状态。
- 先分析上游变化，再修改文件。
- 只保留文档明确列出的定制。
- 不进行无关重构。
- 不执行强制推送。
- 未验证前不得声称合并完成。

