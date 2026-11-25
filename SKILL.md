---
name: ai-era-ci
description: 在用户自己的项目里，把 CI 改成适合 AI 写代码速度的样子：检查拆成四个时候（提交前本地检查、PR 只跑相关测试并限时、合进默认分支后跑全量、上线后用特性开关按比例放量），PR 多到排队时再加「攒批」（几个分支攒成一个 PR，只跑一趟重检查）。边做边用大白话讲，最后给出面试 30 秒版。用户说「帮我把 CI 拆成 4 个时候」「CI 太慢了」「AI 开的 PR 都卡在 CI」「每个 PR 都跑全量测试」「CI 额度不够用」「帮我攒批」「一天几十个 PR 怎么办」「给我的项目配 pre-commit / 只跑相关测试 / 特性开关」时使用。Python、JS/TS、Go 都能用，GitHub Actions 为主，GitLab 也能照做。
---

# AI 时代的 CI：拆成四个时候，PR 多了再攒批

**怎么用这份 skill**
- Claude Code：把这个文件放到项目的 `.claude/skills/ai-era-ci/SKILL.md`，然后说「帮我把 CI 拆成 4 个时候」。
- 其他 agent（Cursor、Codex 等）：把整个文件发给它，再说「照这份 skill 在我的项目里做」。

## 为什么要这么改

- 以前一个人一天大约合一个 PR，每个 PR 都把所有检查跑一遍，也来得及。
- 用 AI 写代码以后，一天几十、上百个 PR。每个都全跑，CI 就排长队、烧额度。加机器只能顶一阵，交得越多，要加的越多。
- 办法有两层，可以同时用：
  1. **跑得少**：每类检查只放在它该在的时候跑（四个时候）。
  2. **跑的趟数少**：几个做完的分支攒成一批，一批只开一个 PR，重检查只跑一趟（攒批）。这一层有代价，PR 真的多到排队、心疼账单了才上。

比喻（给用户讲的时候用）：
- 主分支 = 全班共用的原稿，PR = 交作业，CI = 自动批改机；
- 攒批 = 外卖骑手顺路攒几单，一趟送完。

## 全景

| | 什么时候 | 比喻 | 查什么 | 要多快 | 放在哪 |
|---|---|---|---|---|---|
| ① | 每次 git commit 之前 | 交卷前扫一眼名字 | 格式、lint、能不能编译 | 几秒 | 本地提交前钩子 |
| ② | 开 PR 时 | 交上来只批这道题 | 只跑和改动相关的测试 | 限时 5 分钟（建议值） | PR 上的 CI |
| ③ | 合进默认分支之后 | 全班交齐统一对答案 | 全部测试，包括慢测试 | 可以慢 | push 到默认分支时的 CI |
| ④ | 上线之后 | 发成绩先给几个人看 | 真实用户用着有没有问题 | 先开 5%（建议值），没问题再放大 | 代码里的特性开关 |
| 攒批 | PR 多到排队时（可选） | 外卖顺路攒单 | 分支先过轻检查，几个拼成一个 PR，跑一次重检查 | 一批一趟 | `scripts/batch.py` + 两个 workflow |

**底线：** 只有 ③ 全绿的提交才能拿去发布。默认分支偶尔红了不回滚，直接再提交一个修复（往前修）。

**起步顺序：** 先 ①②；测试慢了加 ③；有真实用户了加 ④；一天几十个 PR、CI 排队或心疼账单了，再考虑攒批。

## 工作规则

- **边做边讲：** 每装好一步，用一两句大白话告诉用户这一步在比喻里是什么、防的是什么问题。不讲长篇道理。
- **先做 ①②，直接开始。** ③、④、攒批先问用户要不要，用户同意了再做。例外：用户已有的 CI 本来就在 PR 上跑全量，拆的时候 ③ 必须和 ② 一起装，测试一个都不能丢。
- 已有的钩子和 CI 文件，不删除、不覆盖；需要改的，先告诉用户改哪几处。
- 不 push，不开 PR，不改仓库设置和分支保护。要提交时先问用户。验证用的临时分支和改动，用完都清理掉。
- 一次只问用户一个问题。
- 「5 分钟」「5%」「10 分钟」都说成建议值，可以调。
- 下面模板里的脚本放进项目后，先用项目自己的格式化工具和 lint 跑一遍，免得它们自己卡在 ① 上。

## 步骤

### 1. 摸清项目（只读）

查清这些，然后用 3 行告诉用户现状和打算，接着直接开始做 ①②：
- 语言、包管理、是不是 monorepo；默认分支叫什么；
- 测试框架、测试有多少；能跑就把全部测试跑一次，记下耗时；
- 哪些是慢测试：要连网络、数据库、浏览器的，端到端的，或者单个就要好几秒的；
- 已有的提交前钩子（pre-commit、husky、lefthook）和 CI（`.github/workflows`、`.gitlab-ci.yml` 等）；
- 有没有特性开关库。

遇到下面几种情况，这样处理：
- **已有 CI，每个 PR 都跑全部测试**（最常见）：不新建重复的，把它拆开。PR 上只留 lint 和相关测试，全部测试挪到 push 默认分支时跑（③）。
- **项目里还没有测试**：② 先只跑 lint、类型检查和构建，并告诉用户「现在没有测试，② 只能查到这一步」。建议先补 3–5 个最关键流程的测试，问用户要不要你来写。
- **不是 GitHub**：GitLab 用 `.gitlab-ci.yml` 的 rules：`merge_request_event` 时跑 ②，默认分支 push 时跑 ③。其他平台同理，命令不变，只换触发条件。
- **某一步已经有了**：保持原样，交付时说一句「这一步你已经有了」。

### 2. 装 ①：提交前检查

- **Python**：pre-commit + ruff，见模板。`rev` 换成当前最新版本，和项目、CI 里的 ruff 版本对齐；新版里 hook id `ruff` 改叫 `ruff-check`。
- **JS/TS**：husky + lint-staged，只查暂存的文件。用项目里已有的 eslint、prettier 配置；本来没有 eslint 的，只加 prettier，不要一下引进一大堆新规则。
  ```
  npm i -D husky lint-staged && npx husky init && echo "npx lint-staged" > .husky/pre-commit
  ```
  package.json 里加：
  ```json
  "lint-staged": {
    "*.{js,jsx,ts,tsx,mjs,cjs,vue,svelte}": ["eslint --fix", "prettier --write"],
    "*.{json,md,css,scss,yml,yaml}": ["prettier --write"]
  }
  ```
- **Go**：pre-commit 的 local hook 跑 `gofmt -l` 和 `go vet`，或者用 lefthook。
- **其他语言**：这门语言最常用的格式化工具 + lint。
- 只放几秒钟就能跑完的检查，测试不要放进来。
- 装好后跑一次全量检查（Python：`pre-commit install && pre-commit run --all-files`）。如果一下改了很多文件，单独提交一次「格式统一」，提交前先问用户。

### 3. 装 ②：PR 只跑相关测试，限时 5 分钟

- PR 的 CI 要满足：
  - `timeout-minutes: 5`；
  - checkout 时 `fetch-depth: 0`，否则算不出改了哪些文件；
  - 先跑一遍 ① 的同款检查，防止有人跳过本地钩子；
  - 然后只跑相关测试。
- 慢测试要标出来，PR 上跳过，留给 ③。Python 用 `@pytest.mark.slow`；JS/TS 约定文件名叫 `*.slow.test.*`。
- 「只跑相关测试」的做法：

| 技术栈 | 做法 |
|---|---|
| Python | 用模板里的 `scripts/related_tests.py`。通用版，顺着 import 关系找，不用改包名 |
| Vitest | `npx vitest related --run --exclude '**/*.slow.test.*' <改动的文件>`，见 JS/TS 模板 |
| Jest | `npx jest --findRelatedTests <改动的文件> --passWithNoTests --testPathIgnorePatterns '\.slow\.test\.'` |
| Nx | `nx affected -t test --base=origin/<目标分支>` |
| Turborepo | `turbo run test --filter=...[origin/<目标分支>]` |
| pnpm 多包 | `pnpm --filter "...[origin/<目标分支>]" test` |
| Go | 用 `git diff` 找出改动的包，再用 `go list -deps` 找出依赖它们的包，只对这些包跑 `go test`；改了 `go.mod` 就全跑 |
| 其他 | 按目录对应：改了哪个模块，就跑这个模块的测试；对应关系说不清时，跑全部快测试 |

- 改了依赖文件、测试配置或 CI 文件，影响面说不清，就跑全部快测试。只改了文档，就跳过。
- 超时了：去优化测试，或者把慢测试挪到 ③。不要调大时间，也不要加机器。

### 4. 验证（必须真跑，把真实输出贴给用户）

1. **①**：建一个临时分支，故意把一个文件的格式写乱再提交。应该被拦下并自动改好；`git add` 之后再提交，通过。
2. **②**：只改一个模块，在本地跑选测试的命令。对比基准用本地的默认分支，不要用 `origin/默认分支`：新加的 CI 文件还没 push，跟远端比会被当成「改了配置」，结果就全跑了。记下三个数：跑了几个测试、一共几个、耗时和全量对比。
3. 验证完，删掉临时分支，撤回为测试做的改动。

### 5. 交付说明（发给用户）

1. **改了哪些文件**：新建的和修改的分开列。
2. **验证结果**：② 跑了几个测试、一共几个，耗时多少，全量要多少。
3. **用户还要自己做的事**：
   - push；
   - 在仓库设置里，把 PR 的 CI 检查设成「合并前必须通过」；
   - 每个协作者在自己电脑上装一次 ① 的钩子。Python 跑 `pre-commit install`；JS/TS 用了 husky 的，`npm install` 时会自动装好。
4. **这次学会了什么**：4 句话，每句对应项目里的一个文件，说清它是哪个「时候」、防的是什么。
5. **面试 30 秒版**：第一人称，用用户项目的真实数字，讲清为什么拆、怎么拆、效果是多少。
6. **简历一句话**：带真实数字。
7. **以后可以直接对你说的话**：
   - 「测试慢了，帮我加第 ③ 步」
   - 「给 <某个功能> 加个开关，先开给 5% 的用户」
   - 「PR 检查超过 5 分钟了，帮我优化测试」
   - 「PR 太多了，帮我攒批」
8. 最后问：要不要现在就加 ③ 或 ④。

### 6. 装 ③：合进默认分支后跑全量（先问）

- push 到默认分支时触发，跑全部测试，包括慢测试。
- 文件顶部注释写清两句：「红了往前修」「只从这里全绿的提交发布」。
- 用户有发布流程的，建议让发布只从 ③ 通过的提交出发；具体怎么接，由用户决定。

### 7. 装 ④：特性开关和按比例放量（先问）

- 项目里已经有开关库或开关服务，就用现成的。
- 没有的话，照模板写：`sha256(开关名:用户ID) % 100`，小于放量比例就打开。同一个用户每次结果都一样。默认 5%。Python 版和 TS 版算出来的桶一致。
- 写好后跑一段验证：模拟 1000 个用户，数一数有多少人打开了，结果应该在 50 上下。

### 8. 攒批（先问；用户的 PR 真多到排队了才建议）

**先跟用户确认三个问题，都能接受再做：**
1. 一批红了，能不能接受多花点时间找是哪个分支的锅？一个分支一个 PR 的话，谁红了一眼就知道。
2. 一批最多攒多久？改动在默认分支外面待得越久，越容易和别人冲突。
3. CI 真的贵吗？不排队、额度够用的话，一个分支一个 PR 就挺好。

**六条规矩：**
1. 每个改动照样一个分支；
2. 分支推上来，先跑轻检查（规范 + 能不能编译，不跑测试）；
3. 过了轻检查的分支排进同一批；
4. 一批开成一个 PR（`batch/N`），在这个 PR 上跑一次重检查；
5. 同一时间最多一个批次，上一批没合进去，新分支等下一批；
6. 批次红了，不改旧分支，从 `batch/N` 拉一个修复分支，用 `add` 加进这一批（往前修）。

**要装的文件：**
- `scripts/batch.py`：通用版，只用 Python 标准库和 git，任何语言的项目都能用。只需要把文件开头的 `LIGHT`、`HEAVY` 两组命令改成这个项目的。
- `.github/workflows/branch-light.yml`：分支推送时跑轻检查。
- `.github/workflows/batch-pr.yml`：`batch/` 开头的 PR 跑重检查。
- 在 ② 的 PR workflow 上加 `if: ${{ !startsWith(github.head_ref, 'batch/') }}`。这样两种做法可以并存：批次 PR 走重检查，普通 PR 还走快检查。

**验证不能碰远程**：在一个临时克隆里删掉 origin，造 2–3 个小分支，跑一遍下面这些，把输出给用户看，然后删掉临时克隆：
- `start` 攒一批；
- 再 `start` 一次，应该被拦下；
- 拉一个修复分支，用 `add` 加进来；
- `status` 看批次里有哪些分支。

**交给用户的用法：**
```
python scripts/batch.py start feat/a feat/b feat/c   # 攒一批（有 gh 会自动开 PR）
git switch -c fix/xxx batch/1   # 批次红了：拉修复分支，改好提交
python scripts/batch.py add fix/xxx
python scripts/batch.py status
```

## 模板

下面的文件都实际跑通过（2026-11-25）：
- 示例项目只改优惠券：13 个测试只跑 5 个；
- 攒批：格式乱的分支被挡在批外，冲突的分支留给下一批，运费分支让批次变红，加修复分支后变绿，批次没合进去时开新批会被拦下；
- 开关：1000 个用户里 37 人打开。

照着改：换成项目的安装命令、版本号和目录名。

### ① .pre-commit-config.yaml（Python）

```yaml
# ① 提交前检查：每次 git commit 之前在你自己电脑上跑，几秒钟就完。
# rev 换成最新版本，并和项目、CI 里的 ruff 版本对齐；新版里 id: ruff 改叫 ruff-check。
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

### ② .github/workflows/pr-fast.yml（Python）

```yaml
# ② 开 PR 时：先查规矩，再只跑和这次改动相关的测试。要快。
name: PR 快检查

on:
  pull_request:

jobs:
  fast:
    # 攒批的 PR（batch/N）走「批次重检查」，这里跳过；两种做法可以并存
    if: ${{ !startsWith(github.head_ref, 'batch/') }}
    runs-on: ubuntu-latest
    # 限时 5 分钟（建议值，可以调）。超时说明测试太慢：去优化测试，或者挪到 ③，不要加机器。
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # 要完整历史，才能算出这次改了哪些文件
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -e ".[dev]" # 换成你项目的安装命令
      - name: 规则检查（和 ① 一样，防止有人跳过本地钩子）
        run: |
          ruff check .
          ruff format --check .
      - name: 只跑和这次改动相关的测试
        run: python scripts/related_tests.py "origin/${{ github.base_ref }}"
```

### ② scripts/related_tests.py（Python 通用版）

```python
"""② 开 PR 时：只跑和这次改动相关的测试（Python 项目通用版）。

用法：python scripts/related_tests.py [对比的分支]
      不写分支时：CI 里用 origin/<PR 的目标分支>，本地用 main。

怎么判断「相关」：
- 改了测试文件 → 跑它
- 改了源码 → 顺着 import 关系，找出直接、间接用到它的模块，再跑 import 了这些模块的测试
- 改了依赖、配置、conftest.py、__init__.py、CI 文件 → 影响面说不清，跑全部快测试
- 只改了文档之类 → 不跑
慢测试（@pytest.mark.slow，或文件里写 pytestmark = pytest.mark.slow）一律不跑，留给 ③。
不依赖任何第三方库，src/ 布局和平铺布局都能用，认得相对导入。
"""

from __future__ import annotations

import ast
import os
import subprocess
import sys
from pathlib import Path

ROOT = Path(__file__).resolve().parent.parent
SKIP_DIRS = {
    ".git",
    ".venv",
    "venv",
    "env",
    "node_modules",
    "build",
    "dist",
    "__pycache__",
    ".tox",
    ".nox",
    ".mypy_cache",
    ".pytest_cache",
    ".ruff_cache",
    "site-packages",
}
# 改到这些，影响面说不清，跑全部快测试
RUN_ALL_FILES = {
    "pyproject.toml",
    "setup.py",
    "setup.cfg",
    "requirements.txt",
    "poetry.lock",
    "uv.lock",
    "Pipfile",
    "Pipfile.lock",
    "tox.ini",
    "pytest.ini",
    "noxfile.py",
    "conftest.py",
    "__init__.py",
    ".pre-commit-config.yaml",
}
RUN_ALL_PREFIXES = (".github/", ".gitlab-ci", "scripts/related_tests.py")
SLOW_MARK = "slow"


def git(*args: str) -> str:
    return subprocess.run(
        ["git", *args], cwd=ROOT, capture_output=True, text=True, check=True
    ).stdout


def default_base() -> str:
    target = os.environ.get("GITHUB_BASE_REF")
    return f"origin/{target}" if target else "main"


def py_files():
    for dirpath, dirnames, filenames in os.walk(ROOT):
        dirnames[:] = [
            d for d in dirnames if d not in SKIP_DIRS and not d.endswith(".egg-info")
        ]
        for f in filenames:
            if f.endswith(".py"):
                yield Path(dirpath, f)


def is_test(p: Path) -> bool:
    return p.name.startswith("test_") or p.name.endswith("_test.py")


def module_name(p: Path) -> str | None:
    """文件 → 模块名。从文件往上找，一直到不再有 __init__.py 的那层目录为止。"""
    parts = [] if p.name == "__init__.py" else [p.stem]
    d = p.parent
    while (d / "__init__.py").exists() and d != ROOT:
        parts.insert(0, d.name)
        d = d.parent
    return ".".join(parts) or None


def imports_of(p: Path, mod: str | None) -> set[str]:
    """这个文件 import 了哪些模块。

    含 from a import b 里可能是子模块的 a.b，也含相对导入。
    """
    try:
        tree = ast.parse(p.read_text(encoding="utf-8"))
    except (SyntaxError, UnicodeDecodeError):
        return set()
    pkg = (mod or "").split(".")
    if p.name != "__init__.py":
        pkg = pkg[:-1]
    out = set()
    for node in ast.walk(tree):
        if isinstance(node, ast.Import):
            out.update(a.name for a in node.names)
        elif isinstance(node, ast.ImportFrom):
            if node.level:
                base = pkg[: len(pkg) - (node.level - 1)] if node.level > 1 else pkg
                base_name = ".".join(base + ([node.module] if node.module else []))
            else:
                base_name = node.module or ""
            if base_name:
                out.add(base_name)
            out.update(
                f"{base_name}.{a.name}" if base_name else a.name for a in node.names
            )
    return out


def only_slow(p: Path) -> bool:
    """整个测试文件都是慢测试吗。"""
    text = p.read_text(encoding="utf-8")
    if (
        f"pytestmark = pytest.mark.{SLOW_MARK}" in text
        or f"pytest.mark.{SLOW_MARK}]" in text
    ):
        return True
    n_tests = text.count("def test_")
    return n_tests > 0 and n_tests == text.count(f"@pytest.mark.{SLOW_MARK}")


def main() -> int:
    sys.stdout.reconfigure(line_buffering=True)
    base = sys.argv[1] if len(sys.argv) > 1 else default_base()
    changed = [
        f for f in git("diff", "--name-only", f"{base}...HEAD").splitlines() if f
    ]
    print(
        f"和 {base} 相比，改了 {len(changed)} 个文件：",
        *changed or ["（没有）"],
        sep="\n  ",
    )

    files = list(py_files())
    tests = [p for p in files if is_test(p)]
    sources = {p: module_name(p) for p in files if not is_test(p)}
    by_path = {str(p.relative_to(ROOT)): p for p in files}

    run_all = [
        f
        for f in changed
        if Path(f).name in RUN_ALL_FILES
        or f.startswith(RUN_ALL_PREFIXES)
        or Path(f).name.startswith("requirements")
    ]
    if run_all:
        print("改到依赖、配置或公共入口，跑全部快测试：", *run_all, sep="\n  ")
        targets = [str(p.relative_to(ROOT)) for p in tests if not only_slow(p)]
    else:
        # 源码改动 → 受影响的模块（顺着 import 往上找，直到不再扩大）
        affected = {
            sources[by_path[f]]
            for f in changed
            if f in by_path and by_path[f] in sources and sources[by_path[f]]
        }
        graph = {m: imports_of(p, m) for p, m in sources.items() if m}
        grew = True
        while grew:
            grew = False
            for m, deps in graph.items():
                if m not in affected and deps & affected:
                    affected.add(m)
                    grew = True
        selected = set()
        for t in tests:
            rel = str(t.relative_to(ROOT))
            if rel in changed or imports_of(t, module_name(t)) & affected:
                if not only_slow(t):
                    selected.add(rel)
        targets = sorted(selected)

    total = sum(1 for p in tests if not only_slow(p))
    if not targets:
        print(f"没有和这次改动相关的快测试（快测试共 {total} 个），跳过。")
        return 0
    print(
        f"要跑的测试文件：{len(targets)} 个（快测试共 {total} 个）",
        *targets,
        sep="\n  ",
    )
    code = subprocess.run(
        [sys.executable, "-m", "pytest", "-m", f"not {SLOW_MARK}", *targets], cwd=ROOT
    ).returncode
    return 0 if code == 5 else code  # 5 = 选中的测试全被过滤掉了，不算失败


if __name__ == "__main__":
    sys.exit(main())
```

### ② .github/workflows/pr-fast.yml（JS/TS，Vitest；Jest 见注释）

```yaml
# ② 开 PR 时：先查规矩，再只跑和这次改动相关的测试。要快。
name: PR 快检查

on:
  pull_request:

jobs:
  fast:
    # 攒批的 PR（batch/N）走「批次重检查」，这里跳过；两种做法可以并存
    if: ${{ !startsWith(github.head_ref, 'batch/') }}
    runs-on: ubuntu-latest
    # 限时 5 分钟（建议值，可以调）。超时说明测试太慢：去优化测试，或者挪到 ③，不要加机器。
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # 要完整历史，才能算出这次改了哪些文件
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - name: 规则检查（和 ① 一样，防止有人跳过本地钩子）
        run: npm run lint --if-present
      - name: 只跑和这次改动相关的测试
        env:
          BASE: origin/${{ github.base_ref }}
        run: |
          # 改了依赖、配置或 CI，影响面说不清：跑全部快测试
          if git diff --name-only "$BASE...HEAD" | grep -qE '(^|/)(package(-lock)?\.json|pnpm-lock\.yaml|yarn\.lock|tsconfig[^/]*\.json|(vite|vitest|jest)\.config\.[^/]+)$|^\.github/'; then
            npx vitest run --exclude '**/*.slow.test.*' --exclude '**/node_modules/**'
            exit $?
          fi
          files=$(git diff --name-only --diff-filter=d "$BASE...HEAD" -- '*.js' '*.jsx' '*.ts' '*.tsx' '*.mjs' '*.cjs' '*.vue' '*.svelte')
          if [ -z "$files" ]; then echo "这次没改代码文件，跳过"; exit 0; fi
          # Jest 换成：npx jest --findRelatedTests $files --passWithNoTests --testPathIgnorePatterns '\.slow\.test\.'
          # shellcheck disable=SC2086 # $files 要按空格拆成多个参数
          npx vitest related --run --passWithNoTests --exclude '**/*.slow.test.*' --exclude '**/node_modules/**' $files
```

### ③ .github/workflows/main-full.yml

```yaml
# ③ 合进默认分支之后：把全部测试（包括慢测试）跑一遍。
# 红了不回滚，直接再提交一个修复（往前修）。
# 发布只从这里全绿的提交里挑。
name: 主分支全量测试

on:
  push:
    branches: [main] # 默认分支不叫 main 的，换成实际的名字

jobs:
  full:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5 # JS/TS 换成 actions/setup-node@v4 + npm ci
        with:
          python-version: "3.12"
      - run: pip install -e ".[dev]" # 换成你项目的安装命令
      - name: 全部测试（包括慢测试）
        run: pytest # JS/TS：npx vitest run；Go：go test ./...
```

### ④ flags.py（Python）

```python
"""④ 上线之后：特性开关 + 按比例放量。

新功能的代码可以先合进去、先上线，但开关只对一部分用户打开。
同一个用户每次算出来的结果都一样，不会一会儿有、一会儿没有。
"""

import hashlib

# 每个开关放给多少比例的用户（0–100）。
# 5 是建议的起步值：先放一小部分人，没问题再往上调。
ROLLOUT_PERCENT: dict[str, int] = {
    "new_checkout": 5,
}


def bucket(flag: str, user_id: str) -> int:
    """把用户稳定地分到 0–99 号桶里。"""
    return int(hashlib.sha256(f"{flag}:{user_id}".encode()).hexdigest(), 16) % 100


def is_enabled(flag: str, user_id: str) -> bool:
    return bucket(flag, user_id) < ROLLOUT_PERCENT.get(flag, 0)
```

### ④ flags.ts（JS/TS）

```ts
// ④ 上线之后：特性开关 + 按比例放量。同一个用户每次算出来的结果都一样。
import { createHash } from "node:crypto";

// 每个开关放给多少比例的用户（0–100）。
// 5 是建议的起步值：先放一小部分人，没问题再往上调。
export const ROLLOUT_PERCENT: Record<string, number> = {
  new_checkout: 5,
};

export function bucket(flag: string, userId: string): number {
  const hex = createHash("sha256").update(`${flag}:${userId}`).digest("hex");
  return Number(BigInt("0x" + hex) % 100n);
}

export function isEnabled(flag: string, userId: string): boolean {
  return bucket(flag, userId) < (ROLLOUT_PERCENT[flag] ?? 0);
}
```

### 攒批 scripts/batch.py（通用版）

```python
"""攒批：把做完的几个分支攒成一批，一批只开一个 PR，只跑一次重检查。

用法（在仓库根目录）：
  python scripts/batch.py start 分支1 分支2  开一个新批次（同一时间只能有一个）
  python scripts/batch.py add 分支          往进行中的批次里加一个分支（如修复分支）
  python scripts/batch.py status            看进行中的批次里有哪些分支

规矩（都是确定的，不靠猜）：
1. 每个改动照样一个分支。
2. 进批之前，每个分支先过一轮轻检查（下面的 LIGHT）。
3. 过了的分支，从最新的默认分支拉出 batch/N，挨个合进去；有冲突的先不进，留给下一批。
4. 有远程仓库就推上去，装了 gh 就自动开 PR，在 PR 上跑重检查；
   没有远程，就在本地跑重检查（下面的 HEAVY）。
5. 同一时间最多一个批次：上一批还没合进默认分支，新分支就等下一批。
6. 批次红了，不改旧分支：从 batch/N 拉一个修复分支，改好后用 add 加进这一批（往前修）。

只用 Python 标准库和 git。不管项目是什么语言，改下面两行命令就能用。
"""

from __future__ import annotations

import json
import os
import shutil
import subprocess
import sys
import tempfile
from pathlib import Path

# ===== 按你的项目改这两组命令（在仓库根目录执行的 shell 命令）=====
# 轻检查：几秒钟，只看规范和能不能编译，不跑测试
LIGHT = ["ruff check .", "ruff format --check ."]
# 重检查（本地没有远程仓库时才用；有远程时由 CI 在批次 PR 上跑）：全部快测试
HEAVY = ["pytest -q -m 'not slow'"]
# =====================================================================

ROOT = Path(__file__).resolve().parent.parent
# 轻检查时借用主目录里已装好的依赖
SHARED_DIRS = ["node_modules", ".venv", "venv", "vendor"]


def git(*args: str, check: bool = True) -> str:
    r = subprocess.run(["git", *args], cwd=ROOT, capture_output=True, text=True)
    if check and r.returncode != 0:
        sys.exit(f"git {' '.join(args)} 失败：\n{r.stderr.strip()}")
    return r.stdout.strip()


def has_remote() -> bool:
    return "origin" in git("remote").split()


def default_branch() -> str:
    ref = git("symbolic-ref", "--short", "refs/remotes/origin/HEAD", check=False)
    if ref:
        return ref.split("/", 1)[1]
    for name in ("main", "master", "trunk", "develop"):
        if git("rev-parse", "--verify", "--quiet", name, check=False):
            return name
    sys.exit("找不到默认分支，请在脚本里写死")


def base() -> str:
    if has_remote():
        git("fetch", "-q", "origin")
        return f"origin/{default_branch()}"
    return default_branch()


def has_gh() -> bool:
    return shutil.which("gh") is not None and has_remote()


def open_batch() -> str | None:
    """进行中的批次。

    有 gh：看有没有还开着的 batch/ PR（squash 合并也认得）；
    没有 gh：看有没有还没合进默认分支的 batch/ 分支。
    """
    if has_gh():
        r = subprocess.run(
            ["gh", "pr", "list", "--state", "open", "--json", "headRefName"],
            cwd=ROOT,
            capture_output=True,
            text=True,
        )
        if r.returncode == 0:
            heads = [p["headRefName"] for p in json.loads(r.stdout or "[]")]
            return next((h for h in heads if h.startswith("batch/")), None)
    b = base()
    for name in git("branch", "--format=%(refname:short)", "--list", "batch/*").split():
        if git("rev-list", "--count", f"{b}..{name}") != "0":
            return name
    return None


def next_batch_name() -> str:
    names = git("branch", "-a", "--format=%(refname:short)").split()
    nums = [
        int(n.rsplit("/", 1)[1])
        for n in names
        if "batch/" in n and n.rsplit("/", 1)[1].isdigit()
    ]
    return f"batch/{max(nums, default=0) + 1}"


def run_all(cmds: list[str], cwd: Path) -> bool:
    for c in cmds:
        print("  $", c)
        if subprocess.run(c, shell=True, cwd=cwd).returncode != 0:
            return False
    return True


def light_check(branch: str) -> bool:
    """在一个临时目录里检出这个分支跑轻检查，不碰你当前的工作区。"""
    print(f"轻检查 {branch}")
    with tempfile.TemporaryDirectory() as tmp:
        wt = Path(tmp) / "wt"
        git("worktree", "add", "-q", "--detach", str(wt), branch)
        try:
            for d in SHARED_DIRS:
                if (ROOT / d).exists() and not (wt / d).exists():
                    os.symlink(ROOT / d, wt / d)
            return run_all(LIGHT, wt)
        finally:
            git("worktree", "remove", "--force", str(wt))


def merge_in(batch: str, branch: str) -> bool:
    git("switch", "-q", batch)
    r = subprocess.run(
        ["git", "merge", "--no-ff", "-q", "-m", f"{batch}：合入 {branch}", branch],
        cwd=ROOT,
        capture_output=True,
        text=True,
    )
    if r.returncode != 0:
        git("merge", "--abort", check=False)
        print(f"  {branch} 和这一批有冲突，先不进，留给下一批")
        return False
    print(f"  已合入 {branch}")
    return True


def branches_in(batch: str) -> list[str]:
    subjects = git("log", "--format=%s", f"{base()}..{batch}").splitlines()
    return [s.split("：合入 ", 1)[1] for s in reversed(subjects) if "：合入 " in s]


def publish(batch: str, created: bool) -> None:
    names = branches_in(batch)
    if not has_remote():
        print("没有远程仓库，在本地跑重检查")
        ok = run_all(HEAVY, ROOT)
        print("重检查：", "通过" if ok else "没通过：拉修复分支，改好后用 add 加进来")
        return
    git("push", "-q", "-u", "origin", batch)
    if not created:
        print("已推送，PR 上的重检查会自动重跑")
        return
    body = "这一批合入的分支：\n" + "\n".join(f"- {n}" for n in names)
    title = f"{batch}：{len(names)} 个改动"
    if has_gh():
        out = subprocess.run(
            [
                "gh",
                "pr",
                "create",
                "--base",
                default_branch(),
                "--head",
                batch,
                "--title",
                title,
                "--body",
                body,
            ],
            cwd=ROOT,
            capture_output=True,
            text=True,
        )
        print(out.stdout.strip() or out.stderr.strip())
    else:
        print(f"已推送 {batch}。没装 gh，请在网页上用它开一个 PR，标题：{title}")


def ensure_clean() -> str:
    if git("status", "--porcelain", "--untracked-files=no"):
        sys.exit("工作区有没提交的改动，先提交或 stash 再攒批")
    return git("rev-parse", "--abbrev-ref", "HEAD")


def cmd_start(branches: list[str]) -> None:
    home = ensure_clean()
    busy = open_batch()
    if busy:
        sys.exit(f"{busy} 还没合进去。同一时间只能有一个批次，这些分支先等下一批。")
    ok = [b for b in branches if light_check(b)]
    for b in [b for b in branches if b not in ok]:
        print(f"{b} 没过轻检查，先不进批")
    if not ok:
        sys.exit("没有能进批的分支")
    batch = next_batch_name()
    git("switch", "-q", "-c", batch, base())
    merged = [b for b in ok if merge_in(batch, b)]
    if not merged:
        git("switch", "-q", home)
        git("branch", "-D", batch)
        sys.exit("一个分支都没合进来（都有冲突），这次不开批次")
    print(f"{batch}：{len(merged)} 个分支攒成一批")
    try:
        publish(batch, created=True)
    finally:
        git("switch", "-q", home)


def cmd_add(branch: str) -> None:
    home = ensure_clean()
    batch = open_batch()
    if not batch:
        sys.exit("现在没有进行中的批次，用 start 开一个")
    if not light_check(branch):
        sys.exit(f"{branch} 没过轻检查")
    try:
        if merge_in(batch, branch):
            publish(batch, created=False)
    finally:
        git("switch", "-q", home)


def cmd_status() -> None:
    batch = open_batch()
    if not batch:
        print("现在没有进行中的批次")
        return
    print(batch)
    for n in branches_in(batch):
        print("  -", n)


if __name__ == "__main__":
    args = sys.argv[1:]
    if args[:1] == ["start"] and len(args) > 1:
        cmd_start(args[1:])
    elif args[:1] == ["add"] and len(args) == 2:
        cmd_add(args[1])
    elif args[:1] == ["status"]:
        cmd_status()
    else:
        sys.exit(__doc__)
```

### 攒批 .github/workflows/branch-light.yml

```yaml
# 攒批第 2 步：每个分支推上来，先跑一轮轻检查。只看规范和能不能编译，不跑测试。
name: 分支轻检查

on:
  push:
    branches-ignore: [main, "batch/**"] # 默认分支不叫 main 的，换成实际的名字

jobs:
  light:
    runs-on: ubuntu-latest
    timeout-minutes: 3
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5 # JS/TS：actions/setup-node@v4 + npm ci + npm run lint
        with:
          python-version: "3.12"
      - run: pip install ruff
      - name: 规则检查
        run: |
          ruff check .
          ruff format --check .
      - name: 能不能编译
        run: python -m compileall -q . # JS/TS：npx tsc --noEmit；Go：go build ./... && go vet ./...
```

### 攒批 .github/workflows/batch-pr.yml

```yaml
# 攒批第 4 步：一批开一个 PR，在这个 PR 上只跑一次重检查（全部快测试）。
# 批次分支一律叫 batch/N，由 scripts/batch.py 创建。
# 红了不撤，往这一批里加一个修复分支（往前修）。
name: 批次重检查

on:
  pull_request:

jobs:
  heavy:
    if: startsWith(github.head_ref, 'batch/')
    runs-on: ubuntu-latest
    timeout-minutes: 10 # 建议值，可以调
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -e ".[dev]" # 换成你项目的安装命令
      - name: 规则检查
        run: |
          ruff check .
          ruff format --check .
      - name: 全部快测试
        run: pytest -q -m "not slow" # JS/TS：npx vitest run --exclude '**/*.slow.test.*'
```
