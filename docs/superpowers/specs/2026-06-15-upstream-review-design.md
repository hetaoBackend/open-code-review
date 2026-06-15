# 设计：`ocr review --upstream`（fork 场景下对比本地分支 vs 远端原始仓库）

- 日期：2026-06-15
- 状态：已确认设计，待实现
- 方案：B（新增专用 `--upstream` 标志）

## 1. 背景与问题

在 fork 工作流里，用户克隆的是自己的 fork，想 review「我当前分支相对**原始仓库**（upstream）改了什么」。当前工具的 range 模式（`--from/--to`）在这件事上有三个障碍：

1. **`--to` 强制必填**（`cmd/opencodereview/flags.go:155-157`）——`ocr review --from upstream/main` 直接报错，但用户的自然意图是「upstream vs 我当前分支（HEAD）」。
2. **ref 必须已存在本地**（`validateReviewRefs` 用 `git rev-parse --verify`，`review_cmd.go:223-248`）——upstream 没 fetch 就报 `not a valid git ref`，工具不会帮忙 fetch，也不提示怎么办。
3. **不能直接用远端 URL**——`--from` 只能传本地可解析的 ref，不能传 `https://github.com/orig/repo`。

用户实测撞到的是第 2 类错误（`not a valid git ref`），且要求**有没有配 `upstream` remote 都要能用**。

## 2. 目标 / 非目标

**目标**
- 一条命令完成「当前分支 vs 远端原始仓库」的 review，无需手动 `git remote add` + `git fetch` + 拼 `--from/--to`。
- 同时支持两种状态：(a) 已配 `upstream` remote；(b) 只有 `origin`（fork），通过传 URL 对比。
- 向后兼容：不改变现有 `--from/--to/--commit` 任何语义。

**非目标**
- 不做 GitHub/GitLab PR API 集成（工具保持纯 git）。
- 不改 range 模式自身（不把 `--to` 默认 HEAD 推广到普通 `--from` 模式）。
- 不自动把 URL 加成永久 remote。

## 3. CLI / UX

```bash
# 对比当前分支(HEAD) vs 原始仓库默认分支（自动探测 main/master）
ocr review --upstream upstream
ocr review --upstream https://github.com/orig/repo

# 指定上游分支
ocr review --upstream upstream --upstream-branch develop

# 覆盖目标（默认 HEAD）
ocr review --upstream upstream --to my-feature

# 离线：跳过 fetch，仅用本地已有的 <remote>/<branch>
ocr review --upstream upstream --no-fetch
```

**新增标志**
- `--upstream <remote|url>`：远端原始仓库，可以是已配置的 remote 名，或一个 git URL。
- `--upstream-branch <branch>`：可选；缺省时自动探测上游默认分支。
- `--no-fetch`：可选；跳过网络 fetch，仅用本地已有的 remote-tracking ref（仅对 remote 名形式有意义）。

**约束**
- `--upstream` 与 `--from`、`--commit` 互斥。
- `--upstream` 可与 `--to` 组合；`--to` 缺省为 `HEAD`。

## 4. 解析算法（统一处理 remote 名与 URL）

`resolveUpstream` 产出一个可用于 range 模式的 `from` ref，再回填 `opts.from`/`opts.to`：

1. **判定形态**：`--upstream` 是 URL（含 `://`、scp 形式 `user@host:path`、或本地路径 `./`、`/`、`file://`）还是 remote 名（在 `git remote` 列表里）。两者都不是 → 报错引导。
2. **定上游分支**：有 `--upstream-branch` 用它；否则 `git ls-remote --symref <target> HEAD`，解析 `ref: refs/heads/<branch>` 得到默认分支。
3. **fetch**：`git fetch <target> <branch>`（除非 `--no-fetch`）。
4. **产出 `from`**：
   - remote 名 → `from = "<remote>/<branch>"`（fetch 后既可读又可解析）。
   - URL → `from = git rev-parse FETCH_HEAD`（SHA；URL 不会写 remote-tracking ref）。
5. **设 `to`**：`opts.to` 缺省 `HEAD`。
6. 交给现有 ModeRange：`merge-base(from, to)..to`。

> 说明：remote 名保留可读 ref（`upstream/main`），URL 用 SHA，二者都能进 `validateReviewRefs` 和 `agent` 的 range 流程，下游零改动。

## 5. 代码改动

- `cmd/opencodereview/flags.go`
  - `reviewOptions` 增加 `upstream`、`upstreamBranch`、`noFetch` 字段。
  - 注册三个 flag；help 文本与 examples 更新。
  - 校验：`--upstream` 与 `--from`/`--commit` 互斥；`--upstream-branch`/`--no-fetch` 仅在 `--upstream` 存在时有意义（否则忽略或报错，取忽略+可选 warning）。
- 新文件 `cmd/opencodereview/upstream.go`
  - `resolveUpstream(repoDir string, runner *gitcmd.Runner, opts reviewOptions) (from, to string, err error)`。
  - 子函数：`isURL(spec) bool`、`remoteExists(repoDir, name) bool`、`discoverDefaultBranch(repoDir, target) (string, error)`（封装 `ls-remote --symref`）、`fetchUpstream(...)`。
- `cmd/opencodereview/review_cmd.go`
  - 在 `resolveRepoDir`（:48-51）之后、`validateReviewRefs`（:52）之前：若 `opts.upstream != ""`，调 `resolveUpstream` 回填 `opts.from`/`opts.to`。
  - 其余流程（validate、`runPreview`、`agent.New`）全部复用，不改。
- `README.md`（及其他语言版可后续跟进）：新增「fork 工作流」一节，给出上面 CLI 示例。

## 6. 错误处理 / 边界

- **remote 不存在且不像 URL**：`no remote named %q; pass a git URL or run "git remote add upstream <url>"`。
- **fetch 失败但本地已有 `<remote>/<branch>`**（remote 名形式）：打印 warning 并回退用本地 ref；URL 形式无回退，直接报错。
- **merge-base 为空**（shallow clone / 历史不相连）：现有报错 `cannot find merge-base between X and Y` 基础上补充提示 `try: git fetch --unshallow`。
- **URL 安全**：匿名 fetch（`git fetch <url> <branch>`），只写 `FETCH_HEAD` 和 objects，不创建永久 remote，不污染用户仓库配置。
- **网络步骤可见性**：fetch 前打印 `Fetching <branch> from <target>…`，让用户知道在走网络。

## 7. 测试

- **单元**
  - `isURL`：`https://`、`git@host:path`、`file:///path`、`./local`、纯 remote 名（应为 false）。
  - `ls-remote --symref` 输出解析 → 默认分支提取。
- **集成**（temp 仓库 + `file://` 路径当 URL，免网络）
  - 建上游 bare 仓库（含 `main` + 一个提交）→ fork clone → 本地分支加提交。
  - 路径 1（remote 名）：`git remote add upstream <bare>` 后跑 `resolveUpstream`，断言 fetch 成功、`from = upstream/main`、merge-base/diff 正确。
  - 路径 2（URL）：直接传 `file://<bare>` 跑 `resolveUpstream`，断言 `from` 为 FETCH_HEAD 的 SHA、merge-base/diff 正确。
  - `--no-fetch`：本地无 ref 时报错；本地有 ref 时不走网络仍成功。
- 与现有 git 测试约定（temp repo helper）保持一致。

## 8. 已知取舍

- merge-base 失败信息在 URL 形式下会显示 SHA 而非可读名（remote 名形式显示 `upstream/main`）。可接受。
- `--no-fetch` 对 URL 形式无意义（URL 必须 fetch 才有 FETCH_HEAD）；此时若加 `--no-fetch` 报错说明。
