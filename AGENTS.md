# AGENTS.md

本仓库是 openai/codex-security v0.1.5（commit `a8fc009`）的本地镜像，含 DeepSeek 模型集成配置与 Linux 沙箱补丁。维护时遵守以下约定。

## 项目定位

- CLI + TypeScript SDK：多智能体 AI 安全扫描（威胁建模 → 发现 → 验证 → 攻击路径 → 修复）
- 扫描由捆绑的 Codex 运行时驱动（`@openai/codex` v0.144.6，二进制随 npm 包分发）
- 插件系统：`sdk/typescript/_bundled_plugin/skills/` 下 15 个技能 + Python workbench

## 构建与测试

```bash
cd sdk/typescript
pnpm install --frozen-lockfile
pnpm run build      # clean + tsc 编译到 dist/
pnpm run types      # 模型 schema 校验 + tsc --noEmit
pnpm run lint       # tsc --noEmit（同 types 的类型检查部分）
pnpm test           # bun test（需要 bun，可选）
```

- Node 要求 ^22.13 || ^24 || ^26；Python 3.10+（扫描时调用插件 Python 脚本）
- pnpm 版本锁定 11.9.0（package.json `packageManager`）

## 网络与镜像（重要）

- npmjs 直连/走代理常被 TLS 重置：pnpm 使用 `https://registry.npmmirror.com`
- 大包（`@openai/codex-linux-x64` ≈131MB）走 npmmirror CDN 很快
- 配置写入用户级 `~/.npmrc`，**不要**提交项目 `.npmrc`
- GitHub 直连超时用 HTTP 代理 `http://121.41.47.146:10809` + `-c http.version=HTTP/1.1`

## 本地补丁（勿随意回退）

**补丁 1：隔离运行时目录移出 /tmp**
- `sdk/typescript/src/runtime.ts`：新增 `codexSecurityTemporaryRoot()`（返回 `~/.codex-security-runtime`）
- `sdk/typescript/src/api.ts`：2 处 `temporaryRoot` 解析改用它

原因：Codex 0.144.6 拒绝在系统临时目录（/tmp）下的 `CODEX_HOME` 创建 `codex-linux-sandbox` 别名；官方版本在 Linux + API Key 模式下扫描必失败。补丁把隔离 home 移到 `$HOME` 下。

**补丁 2：commit/range diff 扫描保存修复**
- `sdk/typescript/_bundled_plugin/scripts/workbench_target.py`：新增 `range_diff_content_digest()`（哈希 `git diff base..head`，codex-security-snapshot/v1 格式）
- `sdk/typescript/_bundled_plugin/scripts/workbench_db.py`：`register_cli_scan` 对 `refs` 目标计算并存储 `contentDigest`；`workbench_completion_binding` 的 `snapshotDigest` 填充条件从「仅 working_tree」放宽为「有 digest 即填」

原因：官方版本对 range diff 不存 digest → SDK 不传 `CODEX_SECURITY_TARGET_SNAPSHOT_DIGEST` → `finalize_scan_contract.py` 强制要求 git_diff 必有 snapshotDigest → 契约校验失败，扫描结果无法保存（扫描本身正常）。

**验证**：`scan` 必须能跑完并产出 findings.json；`cs-scan diff <repo> <base>` 必须能保存为 complete 且 manifest 含 sealedAt + snapshotDigest。若回归，先检查隔离 home 是否又落在 /tmp，或 diff 扫描是否缺 digest。

## 运行扫描（DeepSeek，无需登录）

```bash
export DEEPSEEK_API_KEY=<你的密钥>
export CODEX_API_KEY=<占位非空值>
node sdk/typescript/bin/codex-security.mjs scan <repo> --auth api-key \
  --model deepseek-v4-flash --effort high \
  --codex 'model_provider="deepseek"' \
  --codex 'model_providers.deepseek.name="deepseek"' \
  --codex 'model_providers.deepseek.base_url="https://api.deepseek.com"' \
  --codex 'model_providers.deepseek.env_key="DEEPSEEK_API_KEY"'
```

### 便捷脚本 cs-scan（本机已安装）

`scripts/cs-scan` 封装了上述参数（密钥读 `~/.config/codex-security/env`，600 权限；安装：`ln -sf $(pwd)/scripts/cs-scan /usr/local/bin/cs-scan`），支持三种模式：

```bash
cs-scan <repo>                 # 全仓库扫描（默认 --effort high）
cs-scan diff <repo> <base> [head]   # 提交 diff 扫描（head 默认 HEAD）
cs-scan worktree <repo>        # 未提交工作区改动
```

- 用户显式传 `--model`/`--effort` 时以用户为准
- diff 模式的额外参数（如 `--effort medium`）传透给 CLI

配置要点（踩坑记录）：
- 必须设顶层 `model_provider`，否则请求打到 `api.openai.com`
- 密钥字段为 `env_key`（非 `api_key_env_var`）
- 模型名不带 provider 前缀
- `DEEPSEEK_API_KEY` 必须为真实密钥；`CODEX_API_KEY` 仅作认证门槛（任意非空值）
- effort 默认 high（官方默认 xhigh；low 更省 token，实测可完成扫描）

## 安全约定

- **禁止**提交任何密钥/token（含环境变量值）；README/文档用占位符
- 扫描输出（state 目录、results/）已被 .gitignore 排除，勿强加
- 漏洞报告走上游 Bugcrowd，不在本仓库开 issue 贴 PoC

## 相关文件

| 文件 | 说明 |
|------|------|
| `README.md` | 中文使用文档（含 DeepSeek 集成、补丁说明） |
| `SECURITY.md` | 上游安全策略原文 |
| `scripts/cs-scan` | 一键扫描脚本（本机安装源，改扫描参数默认值主要动这里） |
| `sdk/typescript/_bundled_plugin/skills/` | 扫描技能定义（改扫描行为主要动这里） |
| `docker/` | 容器加固（seccomp/AppArmor/entrypoint） |
