# AGENTS.md — compiler_script 仓库说明

给 AI 工具（Claude Code、Codex、Cursor、CodeBuddy 等）阅读的仓库约定。动手前先读本文件。

## 1. 仓库定位

本仓库只放 **GitHub Actions workflow 编排**，不含被编译软件的源码，也不含构建脚本本体。
每个 workflow 在运行时克隆同名的 `github_codespaces_compile` **私有脚本仓库**分支，执行其中的
`get_xxx.sh` / `build_xxx.sh`，再把产物上传为 artifact。读取权限来自 Actions secret `COMPILER_PAT`。

配套的私有脚本仓库：`edidada/github_codespaces_compile`（本地路径 `d:/develops/git/github/github_codespaces_compile`）。
两个仓库**成对使用**：branch 名相同、workflow 与脚本一一对应。

## 2. 分支规划

### 命名规范

```
编程语言/软件名称/操作系统/版本
例：cpp/chromium/linux/latest      rust/rust/linux/1_91_0
    java/openjdk/linux/25          android/aosp_v/linux/android17_release
```

- workflow 文件名 = 分支名把 `/` 换成 `_` 再加 `.yml`，放 `.github/workflows/`。
  例 `cpp/chromium/linux/latest` → `.github/workflows/cpp_chromium_linux_latest.yml`。
- **一分支一软件一 workflow**：分支只承载自己的 workflow（+ README 表格一行），不要堆别的改动。
- 新增 workflow 后，在根目录 `README.md` 的表格里加一行（列：workflow 文件、软件/分支、OS、版本、源脚本）。
- 语言前缀：`c` `cpp` `java` `python` `go` `rust` `ruby` `erlang` `lua` `csharp` `scala` `kotlin`
  `zig` `ts` `javascript` `as` `actionscript` `android` `shell` `dockerfile` 等。
- `latest` 分支 = 滚动跟踪上游最新；`x_y_z` 版本分支 = **固定版本存档**。
  升级 latest 前先 `checkout -b` 出旧版本号分支存档，再改 latest 里的版本号，不要原地直接改。

### main 的角色

`main` 是**聚合分支**：所有"真正干活的"分支最终都 merge 到本地 main，让 main 汇总全部 workflow
（当前 865 个 workflow）。本地合并用 `git merge --no-edit -X ours <branch>`（冲突保留 main 版，
新增文件正常并入）；独立根/orphan 分支用 `--allow-unrelated-histories` 单独并。
`main` 自身不承载构建任务，历史演进中被删除/规范化的旧式 workflow 不要捞回来。

### workflow 标准结构

```yaml
name: build <branch>
on:
  push:            { branches: ['<branch>'] }
  pull_request:    { branches: ['<branch>'] }
  workflow_dispatch:
permissions: { contents: read }
jobs:
  build:
    runs-on: <os>            # 或 ${{ matrix.os }}
    timeout-minutes: 180
    steps:
      - Clone matching private source branch   # GH_TOKEN: ${{ secrets.COMPILER_PAT }}
          gh repo clone "$GITHUB_REPOSITORY_OWNER/github_codespaces_compile" source \
            -- --depth=1 --single-branch --branch '<branch>'   # 或 "$GITHUB_REF_NAME"
      - Execute source script                  # working-directory: source
          chmod +x '<branch>/get_xxx.sh' && './<branch>/get_xxx.sh'
      - Upload built packages                 # actions/upload-artifact@v4
          path: source/_<name>_build/artifacts/* ; if-no-files-found: error
```

脚本仓库的脚本目录路径与分支名一致（如 `cpp/chromium/linux/latest/get_chromium.sh`）。
版本存档分支从 latest `checkout -b` 而来时，脚本路径仍是 `.../latest/`，workflow 按该路径执行即可。

## 3. GitHub Actions OS 列表

### runner 标签 → 实际镜像（实测，wiseism 账户 2026-11）

| 标签 | 实际镜像 | 架构 | 备注 |
|---|---|---|---|
| `ubuntu-latest` | ubuntu-26.04 | x86_64 | GCC 15 / CMake 4.4.5；4 vCPU / 16 GB / 14 GB SSD |
| `ubuntu-22.04` | ubuntu-22.04 | x86_64 | jammy，WebKitGTK 4.0 |
| `ubuntu-24.04` | ubuntu-24.04 | x86_64 | noble，WebKitGTK 4.1 |
| `ubuntu-26.04` | ubuntu-26.04 | x86_64 | 显式 pin，避免 `-latest` 漂移 |
| `ubuntu-24.04-arm` / `ubuntu-22.04-arm` | arm64 变体 | aarch64 | Azure Cobalt 100；**必须显式写 `-arm` 标签** |
| `windows-latest` | windows-2025-vs2026 | x86_64 | VS 18 / MSVC 19.51 |
| `windows-2022` | windows-2022 | x86_64 | VS 2022 / MSVC 19.4x |
| `macos-latest` | macos-26-arm64 | arm64 | Apple Silicon |
| `macos-14` | macOS 14 | arm64 | `macos-14 + arm64` 组合不被支持 |
| `macos-12` | 已退役 | — | 排队 24h 后超时，禁止使用 |
| `[self-hosted, aosp]` | 自托管 | — | AOSP 等超大型构建（源码 ~250 GB，托管 runner 装不下） |

### 本仓库实际使用分布（main 上 865 个 workflow）

`ubuntu-latest` 594 · `ubuntu-22.04` 126 · `ubuntu-24.04` 41 · `macos-latest` 35 ·
`windows-2022` 24 · `macos-14` 14 · 矩阵 `${{ matrix.os }}` 14 · `windows-2025` 12 ·
`windows-latest` 4 · `ubuntu-26.04` 1(+10 矩阵内) · `[self-hosted, aosp]` 1。

### 多 OS / 多 CPU 矩阵约定

- 标准 6 腿：`ubuntu-22.04` / `ubuntu-24.04` / `ubuntu-26.04` / `ubuntu-24.04-arm` /
  `windows-latest` / `macos-latest`，覆盖 x86_64 与 arm64 双架构。
- `strategy: fail-fast: false`，`matrix.include` 显式列出 `os` + `arch` 标注。
- **runner 标签决定真实机器架构**；matrix 里自定义 `arch` 变量只是字符串，不改变架构。
  实测 `(macos-latest, arch: x86_64)` 与 `(macos-latest, arch: arm64)` 两腿 `uname -m` 都是 arm64。
- artifact 名带 `${{ matrix.os }}-${{ matrix.arch }}` 避免冲突。
- x86_64 与 arm64 的构建缓存/预编译产物不可跨架构复用；NASM 汇编仅 x86_64，arm64 需换工具链。

### 平台踩坑（写 workflow 必须考虑）

- macOS 的 BSD `chmod` 不认 `--` → 用 `chmod +x 'file'`，不要 `chmod +x -- 'file'`。
- macOS 自带 bash 3.2 会把 `$VAR中文` 的全角字符吞进变量名 → 变量统一 `${VAR}`。
- BSD `sed -i` 需要备份后缀，GNU 写法会报错 → 用 `node -e` / `python3` 跨平台改写文件。
- Windows runner 默认 `core.autocrlf=true` 会把 `.sh` 转成 CRLF 让 bash 报错 →
  clone 前 `git config --global core.autocrlf false` + `core.longpaths true`；
  脚本仓库分支放 `.gitattributes` 写 `*.sh text eol=lf`；上游 clone 用 `git -c core.autocrlf=false clone`。
- `timeout-minutes` 必写；`concurrency: { cancel-in-progress: true }` 取消旧 run；
  显式 `permissions: contents: read`；保留 `workflow_dispatch`。
- Node.js 20 → 24 迁移、`-latest` 标签漂移、`setup-*` 动作大版本弃用会产生注解（非失败，但要跟进）。
- 超重型构建（Chromium 全量 `chrome`、OceanBase 全平台 AOSP）在 4 核 / 16 GB / 6h 托管 runner
  上跑不完：要么缩小目标（如 Chromium 改编 `//base`），要么用自托管 runner。

## 4. 提交与验证流程

1. 两仓库各 `checkout -b <同名分支>`；compiler_script 侧加 workflow YAML + README 行，
   github_codespaces_compile 侧加 `get_xxx.sh`（+ `.gitattributes`）。
2. `bash -n` 校验脚本、YAML 解析校验 workflow，分别 commit。
3. **先推脚本仓库分支，再推 compiler_script 分支**（push 会触发 Actions run）。
4. `gh run list / watch / view <id> --log-failed` 盯日志，失败则按日志修脚本、重推迭代，
   直到**全部 job 成功**（不能以部分成功收尾），确认 artifact 有真实产物。
5. 成功后把分支 merge 到本地 `main`（见 §2），最后 `git push origin main`（仅限 origin，
   其它 mirror remote 需另行确认）。

数据来源：`ai_agent_scripts/git/github/github_action.txt`（runner OS/CPU 实测笔记）与本仓库 workflow。
