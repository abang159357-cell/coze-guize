# 推送 GitHub 与触发 CI 交接指南

> 面向「Next.js + TypeScript + pnpm」Web 项目（或其他用 GitHub Actions 触发 CI 的 Node 项目）的**真实可操作**交接流程。基于大其力生活平台项目线上实测沉淀；其他项目照此执行即可，无需再从零踩坑。
>
> 本文件只讲「推送 + CI」、到 **CI / 本地验收变绿为止**。扣子平台的**重新部署与线上验收**归 `general-deploy` skill 与「上线前规定」清单承接，此处不展开；二者衔接点见文末「全链路节奏」。

---

## 一、推送操作流程

### 1. 提交规范

- 提交信息统一前缀：`feat` / `fix` / `docs` / `chore` / `refactor` / `test`，冒号后写中文摘要，例：`fix: 首页搜索框接入可点击搜索`。
- **一次提交只做一个目标**（防目标漂移），别把无关改动混进同一 commit。

### 2. 推送原则（核心，务必遵守）

- **逐一推送**：多个待推送 commit 时，一个一个推，每推一个看一次输出确认成功（如 `c3f922d..6811e8c  6811e8c -> main`），**不要 `git push` 一把梭所有**。理由：commit 多时出错难定位、难回滚；逐一推可让 CI 逐个提交验证，出错即定位到具体那次改动。
- **快进式递进，绝不 force push、绝不 amend 已推送 commit**。要推某个具体 commit 到某个分支用：`git push origin <commit-hash>:main`。
- 推送前先 `git pull`（或确认本地 `main` 与 `origin/main` 关系），避免分叉。

### 3. 推送后必须确认「完全同步」

```bash
git status -sb                       # 应显示 ## main...origin/main（无 ahead）
git log --oneline @{u}..HEAD | wc -l # 应输出 0，表示无未推送提交
```

若 not 0，说明还有没推上去的，回到逐一 push 流程补齐。

### 4. 提交前自查（防止推到远端才发现）

用 `git log && git status` 看本次要推的内容，逐个 commit 核对文件清单是否只含目标改动；发现夹带了不该推的文件（截图、env、临时文件）立即用 `reset --soft` + 拆出，**不要推到远端再清理**。

---

## 二、CI 触发与检查节奏

### 1. 触发条件（本项目 `.github/workflows/ci.yml`）

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

即：**push 到 main 或开 PR 到 main 即自动触发**。推送成功 ≠ CI 通过，需进 GitHub Actions 面板看 `test` job 结果。

### 2. 检查步骤（job 顺序，绿色才算过）

typecheck（`pnpm ts-check`）→ lint（`pnpm lint`）→ 单元测试（`pnpm test`）→ build（`pnpm build`）。四步全绿才通过。

### 3. 关键认知：CI 绿 ≠ 页面可用

- CI 跑的是**静态质量 + 构建正确性**，证明"代码写得对、能编译"，**不代表功能正确、不代表页面能跑起来**。
- 涉及可见界面/接口的项目，**必须另用 `test_run`（web 项目官方验收工具），在本地做页面探活 + 接口真实冒烟**。CI 和 test_run 是两道不同防线，不能互相替代。
- CI 红 ≠ 代码必有错：可能是环境/版本配置问题（见下节常见 CI 失败），先看失败步骤日志再定位，别急着改代码。

### 4. 失败处理

CI 某个 job 失败时：点进 workflow 看失败步骤日志，通常错误信息自带定位；修复后重新 push（或关掉 PR 重开）触发。**不要反复换方案硬试**——同一根因基于证据改一次。

---

## 三、pnpm / CI 版本对齐规则（最常踩的 CI 失败源）

### 问题现象

GitHub Actions 报 `ERR_PNPM_BAD_PM_VERSION`，workflow 直接失败。

### 根因

`package.json` 的 `packageManager` 字段值和 `.github/workflows/ci.yml` 里 `pnpm/action-setup` 的 `version` **不一致**。CI 用 action-setup 指定的版本装 pnpm，npm 侧的 corepack 校验 packageManager 声明，两者对不上就崩。

### 强制要求（两处必须完全一致）

| 位置 | 值必须 |
|---|---|
| `package.json` 的 `packageManager` | `pnpm@<x.y.z>` |
| `.github/workflows/ci.yml` 中 `pnpm/action-setup@v4` 的 `with.version` | `<x.y.z>`（不含 `pnpm@` 前缀） |

本项目实测对齐值：`packageManager: "pnpm@9.15.9"` ↔ ci.yml `version: 9.15.9`。

### 改法

改两处保持一致后重新 push / 重开 workflow。注意：**改 ci.yml 属于 `.github/workflows/`**，若当前 GitHub 凭据缺 `workflow` scope 会 push 失败——此时代码改好提交后，需在 GitHub 网页手动编辑该文件提交，再用 pull 拉回本地对齐。

### 连带注意

- Node 版本 `actions/setup-node` 的 `node-version` 要与项目实际需求一致（本项目为 24）。
- `pnpm install --frozen-lockfile` 要求 `pnpm-lock.yaml` 与 `package.json` 一致；改了依赖必须 `pnpm install` 更新锁文件，否则 CI 失败。

---

## 四、构建期环境变量坑（`NEXT_PUBLIC_` 前缀）

### 问题现象

部署后 sitemap / robots / JSON-LD / metadataBase 输出的域名变成 `localhost:3000`，污染搜索引擎收录。

### 根因（实测关键事实）

- `NEXT_PUBLIC_` 前缀变量在 Next.js **构建时内联**，不是运行时读取。
- **扣子平台只把 `NEXT_PUBLIC_SITE_URL` 注入运行时，不注入 `next build` 构建进程**。所以凡在构建期就被求值快照的代码（如静态 sitemap/robots 顶层 `const SITE_URL = siteUrl()` + `force-static`），拿到的 env 为空 → 回退 localhost。
- 顶层 `export const metadata = { metadataBase: new URL(siteUrl()) }` 是**运行时求值**（metadata 每次请求生成），所以首页能读到正式域名——不要被这一点误导去"照抄"到静态文件。

### 修法（三步，缺一不可）

① 用规范化函数统一生成 URL（缺协议头补 `https`、去尾部斜杠），**不要直接 `new URL(process.env.NEXT_PUBLIC_SITE_URL)`**（配置成 `g9fyprzfc8.coze.site` 这类无协议头值会抛 `Invalid URL` 导致页面 500）；
② URL 拼进**函数体内**（而非模块顶层常量）；
③ **加 `export const dynamic = 'force-dynamic'`**——强制运行时渲染、每次请求读 env，与 metadata 同机制。仅把变量移进函数体不够，静态 sitemap 仍会在构建时求值快照。

### 验证方式

- 部署后 `curl <域名>/sitemap.xml` 看 `<loc>` 是否出现 `localhost`。
- 构建输出里 `sitemap.xml` 应为 `ƒ (Dynamic)` 而非 `○ (Static)`。
- 首页 JSON-LD / 各详情页 metadata 是否用上正式域名。

---

## 五、全链路节奏（推送与部署衔接）

本项目确认的安全节奏（自改后一直遵守）：

```
修改 → 复查(仅读/确认) → 本地验证(test_run: lint+ts-check+探活+接口冒烟)
     → 逐一 push(GitHub, 触发 CI) → 等 CI 绿 → 重新部署 → 线上验收
```

- **本地验证未过不发 push**；用户未确认不停留部署。
- CI 是第二道防线，不等同 test_run；部署执行走 `general-deploy` skill 的 `[deploy]` 配置；线上验收按「上线前规定」条款逐项给可验证结论。

---

## 六、PR 提交流程（无人值守开发与人类验收的 git 层闸门）

> 目标：把「计划定案 → 无人值守开发 → **人类 review + 验收** → 部署」落到 git 上，避免 AI 直接 push main 绕过人工把关。适用于已开启分支保护（main 仅能经 PR 合并）的仓库；未开启保护的仓库也**推荐**走此流程，让每次合并留下 review 记录。

### 1. 功能分支规范

- 从最新 `main` 起分支：`git checkout -b feat/<短名> origin/main`（每功能一分支，命名 `feat/`、`fix/`、`chore/`、`docs/`）。
- 功能分支只在授权范围内开发；无人值守开发在功能分支进行，**不直接改 main**。

### 2. 开发后在功能分支提交、推远端

- 提交规范同「一、提交规范」；逐 commit 推送。
- 推远端：`git push -u origin feat/<短名>`。

### 3. 开 PR（人类可先在 GitHub 网页 review，或命令触发）

- 网页：push 后 GitHub 会提示 `Create pull request`，或访问 `https://github.com/<owner>/<repo>/pull/new/<分支>`。
- PR 描述按仓库的 `PULL_REQUEST_TEMPLATE` 填：变更摘要 / 关联 Issue / 测试证据 / 影响范围 / 验收清单。

### 4. 等 CI 通过

- PR 自动触发 `pull_request: main` 的 CI；等 `test` + `security`(CodeQL) job 全绿。
- 若开了分支保护"Require status checks"，CI 未绿会被 GitHub 强制拦截 merge。

### 5. 人类 review 与 merge

- **AI 不自行合入 main**。由人类在 GitHub 网页 review 后点 **`Merge pull request`**（有分支保护时满足条件才可点）。
- 合并后：删除已合并的分支 `git push origin --delete feat/<短名>`，并拉回同步 `git checkout main && git pull`。

### 6. 边界与例外

- 仅当人类**明确要求"直接 push main 不走 PR"**时才绕过本流程（如临时小修、纯文档），且仍需保证本地验证 + CI 通过。
- 分支保护需在 GitHub 仓库 Settings → Branches 单独开启（见仓库层配置，不在代码文件内）。本流程假定保护已开；未开时本流程仍是推荐的最佳实践。
- 流程对「无人值守」的意义：AI 开发全程在功能分支完成，把"No / 验收决定权"显式保留给人类在 merge 一刻行使。