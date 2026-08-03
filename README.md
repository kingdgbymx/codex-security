# Codex Security（中文版）

> 基于 [openai/codex-security](https://github.com/openai/codex-security) v0.1.5（commit `a8fc009`）的本地镜像，集成 **DeepSeek 模型支持** 与 **Linux 沙箱兼容补丁**，已实测可用。
>
> English: [README.en.md](README.en.md)（上游原版）

`codex-security` 是一个 CLI 与 TypeScript SDK，用于查找、验证并修复代码中的安全漏洞。扫描由多智能体 Codex 运行时驱动，支持威胁建模、攻击路径分析、漏洞分级、修复建议与结果追踪。

官方文档：<https://learn.chatgpt.com/docs/security/cli>

> 提示：官方版本建议账户开启 [Trusted Access](https://chatgpt.com/cyber) 以获得最佳效果。本镜像已实测可通过 API Key 模式 + DeepSeek 模型完成完整扫描，无需 ChatGPT 登录。

## 特性

- **标准扫描**：单遍全仓库安全审计，输出 `scan-manifest.json` / `findings.json` / `coverage.json` 三件套（可校验、可密封）
- **深度扫描**：`--mode deep` 多轮发现 + 多工作线程（`--workers` / `--subagents` / `--stop-after-no-new` / `--max-discovery-runs`）
- **Diff 扫描**：对 PR、commit、分支 diff、工作区变更做定向审计
- **扫描对比**：`scans compare` 按根因自动匹配新旧扫描（new / persisting / reopened / resolved / unknown）
- **批量扫描**：容器化 CSV 仓库清单扫描（`bulk-scan`），Git 固定 revision，可断点续扫
- **模型可替换**：通过 `model_providers` 接入任意 OpenAI Responses 兼容端点（已实测 DeepSeek V4）

## 环境要求

- Node.js 22.13+（22.x）/ 24.x / 26.x
- Python 3.10+
- 可访问所选模型的 API

## 快速开始

### 构建

```bash
cd sdk/typescript
pnpm install --frozen-lockfile
pnpm run build
```

> 国内网络提示：npmjs 直连常被重置，pnpm 请使用 `https://registry.npmmirror.com`（写入 `~/.npmrc` 或项目 `.npmrc`）。大体积依赖包（如 `@openai/codex-linux-x64`，约 131MB）通过 npmmirror CDN 下载很快。

### 使用官方 OpenAI 模型（需 API Key 或登录）

```bash
# CI 模式：环境变量 API Key（不会落盘）
export OPENAI_API_KEY=sk-xxx
npx codex-security scan . --auth api-key

# ChatGPT 登录模式
npx codex-security login
npx codex-security scan .
```

### 使用 DeepSeek 模型（无需 ChatGPT 登录）✅ 已实测

DeepSeek V4 系列支持 OpenAI Responses API，可直接作为 Codex 的模型提供方：

```bash
export DEEPSEEK_API_KEY=sk-你的DeepSeek密钥
export CODEX_API_KEY=sk-占位            # 仅用于认证流程，任意非空值

node sdk/typescript/bin/codex-security.mjs scan . --auth api-key \
  --model deepseek-v4-flash \
  --effort low \
  --codex 'model_provider="deepseek"' \
  --codex 'model_providers.deepseek.base_url="https://api.deepseek.com"' \
  --codex 'model_providers.deepseek.env_key="DEEPSEEK_API_KEY"'
```

实测结果（示例仓库）：8 分钟完成，正确发现 SQL 注入（CWE-89），覆盖率 complete，报告完整（威胁模型 + 根因 + 修复建议）。

**关键点**（官方文档易踩的坑）：
- 必须设置顶层 `model_provider = "deepseek"`，仅设 `model_providers` 无效（请求会打到 `api.openai.com`）
- 密钥字段名是 `env_key`（不是 `api_key_env_var`）
- DeepSeek 模型名不带 provider 前缀，如 `deepseek-v4-flash`

### TypeScript SDK

```ts
import { CodexSecurity } from "@openai/codex-security";

const security = new CodexSecurity();
const result = await security.run(".");
await security.run(".", { mode: "deep", workers: 2, subagents: 0 });
console.log(result.reportPath);
await security.close();
```

### 容器化批量扫描

```bash
docker compose up --build
# 输入: repositories.csv（仓库清单），输出: results/、state/
```

Dockerfile / compose.yaml 已做加固：`cap_drop: ALL`、seccomp 限制、非 root 用户（10001）。

## 本地补丁说明

本仓库相对官方版本（`a8fc009`）包含一处必要补丁：

**问题**：捆绑的 Codex 运行时 0.144.6 新增保护——当 `CODEX_HOME` 位于系统临时目录下时，拒绝创建 `codex-linux-sandbox` 沙箱别名（arg0 软链）。而官方 codex-security 在 API Key 模式下总是把隔离的 `CODEX_HOME` 建在 `os.tmpdir()`（`/tmp`）下，导致沙箱失效、扫描 agent 无法执行任何 shell 命令、扫描必然失败（Linux + API Key 组合下官方版本即挂）。

**修复**：隔离目录从系统临时目录改为 `~/.codex-security-runtime`（在 `$HOME` 下，非 `/tmp`）：

| 文件 | 改动 |
|------|------|
| `sdk/typescript/src/runtime.ts` | 新增 `codexSecurityTemporaryRoot()`；`prepareOutputDir` / `createIsolatedHome` 默认值改用它 |
| `sdk/typescript/src/api.ts` | `temporaryRoot` 解析改用它（2 处） |

如需恢复官方行为，回退这两个文件的改动即可。

## 仓库结构

```
├── Dockerfile / compose.yaml      # 容器化批量扫描
├── docker/                        # seccomp、AppArmor、entrypoint、git 凭证
├── sdk/typescript/
│   ├── src/                       # CLI + SDK 源码（cli.ts ~3.4k 行）
│   ├── _bundled_plugin/           # 随包插件：15 个技能 + Python workbench + MCP server
│   │   └── skills/                # threat-model / finding-discovery / validation /
│   │                              # attack-path-analysis / triage / fix 等
│   ├── tests-ts/                  # bun 测试
│   └── bin/codex-security.mjs     # CLI 入口
```

## 安全说明

- 扫描进程使用 `codex_security_scan` 文件系统 profile：只读全盘，仅写工作区与扫描状态目录；`approvalPolicy: "never"`
- API Key 通过环境变量传递，不落盘；workbench 子进程会剔除 `OPENAI_API_KEY` / `CODEX_API_KEY`
- 只扫描你信任且有权评估的仓库；本地状态不构成多租户安全边界
- 漏洞报告请走 OpenAI Bugcrowd 项目（见 [SECURITY.md](SECURITY.md)）

## 许可

Apache-2.0（与上游一致，见 [LICENSE](LICENSE)）。
