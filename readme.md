# jj 教程：给 Git 用户的 Jujutsu 实战入门

这个仓库用于练习 Jujutsu VCS，也就是 `jj`。它可以和 Git 共存：远端协作仍然可以用 Git/GitHub，本地开发、拆分变更、回滚历史、并行 agent 任务可以交给 `jj`。

本文面向已经懂 Git 的工程师。目标不是逐条翻译命令，而是建立一套能直接工作的心智模型。

本教程按 `jj 0.43.0` 校对命令。

## 目录

- [为什么 Git 用户值得学 jj](#为什么-git-用户值得学-jj)
- [核心心智模型](#核心心智模型)
- [初始化和基础配置](#初始化和基础配置)
- [观察仓库状态](#观察仓库状态)
- [日常单线开发](#日常单线开发)
- [修改旧 change](#修改旧-change)
- [拆分和合并 change](#拆分和合并-change)
- [分叉、合并和 rebase](#分叉合并和-rebase)
- [bookmark 和 Git 互操作](#bookmark-和-git-互操作)
- [撤销和恢复](#撤销和恢复)
- [workspace 和 agent 并行工作流](#workspace-和-agent-并行工作流)
- [完整练习流程](#完整练习流程)
- [常见坑](#常见坑)
- [命令速查](#命令速查)

## 为什么 Git 用户值得学 jj

Git 的默认工作流通常是：

```text
工作区 -> 暂存区 -> commit -> branch -> remote branch
```

`jj` 的默认工作流更像：

```text
正在编辑的 change -> 本地历史图 -> 需要协作时挂 bookmark -> push 到 Git
```

它带来的变化：

| 场景 | Git 常见做法 | jj 常见做法 |
|---|---|---|
| 保存当前工作 | `git add` + `git commit` | 文件变更会进入当前 working-copy commit |
| 继续下一个任务 | `git commit` 后 `git checkout -b` 或继续提交 | `jj new` |
| 修改旧提交 | `git rebase -i` / `git commit --amend` | `jj edit <rev>` 或 `jj squash -i` |
| 拆提交 | `git reset` / `git add -p` / rebase | `jj split` |
| 撤销危险操作 | `git reflog` + reset | `jj undo` / `jj op restore` |
| 并行 agent 任务 | `git worktree` + branch | `jj workspace add` + change |

最重要的区别：`jj` 把“修改历史”设计成日常操作，而不是危险操作。

## 核心心智模型

### change id 和 commit id

`jj log` 里通常能看到两种 id：

```text
@  pppkrtqx user 2026-07-21 9ad5cc49
│  feat: add config
```

这里：

| 名称 | 含义 | 是否稳定 |
|---|---|---|
| change id | 这个变更的身份 | 通常稳定 |
| commit id | 当前内容快照的哈希 | 内容、描述、父节点变化就会变 |

一句话：

```text
change id 是任务身份；commit id 是内容快照。
```

你修改文件、改描述、rebase 之后，commit id 可能变化，但 change id 通常还代表同一件事。

### working-copy commit

在 `jj` 里，工作区本身就是一个 commit。当前正在编辑的 revision 写作 `@`。

Git 用户可以这样对照：

```text
Git:
  工作区 + 暂存区 + HEAD commit

jj:
  @ 这个 working-copy commit
```

这就是为什么 `jj` 不需要 `git add`。你不是把文件放进暂存区，而是在持续编辑当前 change。

### operation

每一次 `jj` 对仓库状态的修改都会产生一个 operation。

```bash
jj op log
```

它比 Git reflog 更像一条完整操作日志。很多误操作可以直接：

```bash
jj undo
```

或者：

```bash
jj op restore <operation-id>
```

### bookmark

`jj` 里的 bookmark 是对外暴露给 Git 的名字，可以理解为 Git branch 的映射，但不要把它当成本地开发的起点。

推荐规则：

```text
本地先用 change 工作；准备 push 或开 PR 时，再给目标 change 挂 bookmark。
```

## 初始化和基础配置

已有 Git 仓库时，在仓库根目录执行：

```bash
jj git init --colocate
```

`--colocate` 表示 `.git` 和 `.jj` 共存在同一个目录：

```text
repo/
  .git/
  .jj/
```

之后建议主要用 `jj` 修改版本状态，少混用这些 Git 命令：

```bash
git commit
git rebase
git merge
git switch
git checkout
git reset
```

可以继续用只读 Git 命令辅助观察：

```bash
git status
git log
git diff
```

注意：colocated 模式下，`git status` 可能显示类似 detached HEAD。这通常是正常的，不要用 `git checkout` 试图修复它。

常用 alias：

```bash
jj config set --user aliases.l '["log", "-r", "all()", "--graph"]'
jj config set --user aliases.ll '["log", "-r", "all()", "--graph", "--patch"]'
```

之后可以用：

```bash
jj l
jj ll
```

## 观察仓库状态

最常用观察命令：

```bash
jj st
jj log
jj log -r 'all()'
jj show @
jj diff
jj evolog
jj op log
```

含义：

| 命令 | 用途 |
|---|---|
| `jj st` | 查看当前工作区状态 |
| `jj log` | 查看默认可见历史图 |
| `jj log -r 'all()'` | 查看全部 revision 图 |
| `jj show @` | 查看当前 change 的描述和 diff |
| `jj diff` | 查看当前 change 相对父节点的 diff |
| `jj evolog` | 查看当前 change 的 commit id 演化 |
| `jj op log` | 查看 jj 操作历史 |

`@` 表示当前 working-copy commit。`@-` 表示当前 change 的父节点。`@--` 表示祖父节点。

## 日常单线开发

### 自动 snapshot

`jj` 不是按时间自动保存，也不是每隔几秒生成一个 commit。

默认行为是：大多数 `jj` 命令执行前，会先 snapshot 当前 working copy。

例如：

```bash
echo '// first app' >> main.go
jj log
```

`jj log` 执行前会 snapshot 工作区。如果文件内容变了，当前 `@` 的 commit id 会变化，但 change id 不变。

手动触发 snapshot：

```bash
jj util snapshot
```

查看状态但不触发 snapshot：

```bash
jj st --ignore-working-copy
jj log --ignore-working-copy
```

### 创建第一个 change

```bash
cat > main.go <<'EOF'
package main

import "fmt"

func main() {
	fmt.Println("hello")
}
EOF

jj st
jj describe -m "feat: add main"
jj log
```

`jj describe` 只修改当前 change 的描述，不会创建新的 change id。

### 修改同一个 change

```bash
echo '// first app' >> main.go
jj log
jj evolog
```

你会看到：

```text
change id 还是同一个
commit id 变了
```

原因是文件内容变了，`jj` snapshot 后生成了新的内容快照。

### 开始下一个 change

```bash
jj new
echo 'debug = true' > conf.toml
jj describe -m "feat: add config"
jj log
```

`jj new` 会创建一个新的空 change，并把 `@` 移过去。

关键点：如果执行 `jj new` 前当前 change 里已经有文件修改，这些修改会留在原 change 里。它不像 `git checkout -b` 那样把未提交修改带到新分支。

### describe、new、commit 的区别

```bash
jj describe -m "feat: add config"
```

只修改当前 change 的描述。

```bash
jj new
```

创建新的空 change，并移动工作区到新 change。

```bash
jj commit -m "feat: add yaml"
```

在没有 path 参数和 `--interactive` 时，等价于：

```text
jj describe -m "feat: add yaml"
jj new
```

也就是：描述当前 change，然后在它上面创建下一个空 change。

如果你更喜欢 Git 的肌肉记忆，可以用 `jj commit -m ...`。如果你想明确控制当前 change 和下一个 change，建议用 `jj describe` + `jj new`。

## 修改旧 change

### 直接编辑旧 change

```bash
jj log -r 'all()'
jj edit <change-id>
```

然后修改文件：

```bash
echo '// fix old change' >> main.go
jj st
jj log
jj evolog
```

`jj edit` 不创建新 change，只是把 `@` 移到已有 change。

强大的地方在于：如果你修改的是一条 stack 底部的 change，它的子孙 change 会自动重新基于新的内容计算。也就是说，`jj` 默认接受“历史会被整理”这件事。

### 更稳的修复方式：fixup 后 squash

直接 `jj edit <old>` 很快，但团队协作时你可能希望先把修复单独放出来看一眼。

```bash
jj new <old-change-id>
echo '// fix old change' >> main.go
jj describe -m "fixup: adjust main"
jj diff
```

确认后合回父 change：

```bash
jj squash
```

交互式选择部分内容合回：

```bash
jj squash -i
```

默认 `jj squash` 是把当前 `@` 的变化移动到父 change。如果 source 被搬空，且没有 `--keep-emptied`，source change 会被 abandoned。

## 拆分和合并 change

### 拆分混合 change

先故意做一个混合 change：

```bash
jj new -m "mixed changes"
echo 'a' > a.txt
echo 'b' > b.txt
jj st
```

拆分：

```bash
jj split
```

如果只想按文件拆：

```bash
jj split a.txt -m "feat: add a"
```

默认不传 fileset 时，`jj split` 会打开交互式 diff editor。常见操作：

```text
Space   选择 / 取消当前 hunk 或行
Enter   展开 / 收起
Tab     切换区域
c       确认拆分
q       退出 / 取消
?       查看帮助
```

拆分后，一个 change 会变成两个 change。被拆出来的新 change 会有新的 change id。

### 把两个 change 合起来

如果当前 `@` 应该合到父 change：

```bash
jj squash
```

如果只合一部分：

```bash
jj squash -i
```

如果要从指定 change 合入目标 change：

```bash
jj squash --from <source> --into <destination>
```

## 分叉、合并和 rebase

### 从当前 change 后面继续

```bash
jj new
```

### 从旧 change 分叉

```bash
jj log -r 'all()'
jj new <base-change-id> -m "feat: branch from old base"
```

历史会类似：

```text
@  feat: branch from old base
│
│ ○ feat: add config
├─╯
○  feat: add main
◆  root
```

在 `jj` 里，分叉不一定需要 Git branch。change 本身就可以形成分叉历史。

### 创建 merge change

如果有两个头：

```text
aaa111  branch A
bbb222  branch B
```

创建一个拥有两个父节点的 merge change：

```bash
jj new aaa111 bbb222 -m "merge branch changes"
```

### rebase 当前 stack

`jj rebase` 有三种常用选择范围：

| 参数 | 含义 |
|---|---|
| `-s`, `--source` | 移动指定 revision 和它的 descendants |
| `-b`, `--branch` | 移动相对 destination 的整条 branch，默认是 `-b @` |
| `-r`, `--revision` | 只移动指定 revision，不带 descendants |

最常用的是移动当前 stack：

```bash
jj rebase -b @ -d <destination>
```

`-d` 是 `--destination` 的别名，在 `jj 0.43.0` 中等价于 `--onto`。

同步主线后，把当前 stack 放到主线后面：

```bash
jj git fetch
jj rebase -b @ -d master
```

如果要明确基于远端 bookmark：

```bash
jj rebase -b @ -d master@origin
```

只移动当前 change：

```bash
jj rebase -r @ -d <destination>
```

把当前 change 和它的后代一起移动：

```bash
jj rebase -s @ -d <destination>
```

## bookmark 和 Git 互操作

### bookmark 是什么

`jj` 的 bookmark 是 Git branch 的对外名字。Git 远端看不到你的本地 change 图，它只能看到被 bookmark 指到的 commit。

推荐原则：

```text
不要一开始就创建 bookmark。
先用 change 本地工作。
准备 push 或开 PR 时，再给目标 change 挂 bookmark。
```

### 创建和移动 bookmark

```bash
jj bookmark create feature/demo -r @
jj bookmark list
```

如果当前 change 后续又改了，bookmark 不一定像 Git 当前 branch 那样自动跟着你移动。推送前显式移动：

```bash
jj bookmark move feature/demo --to @
```

或者用 `set` 创建或更新：

```bash
jj bookmark set feature/demo -r @
```

### 推送到 Git 远端

```bash
jj git fetch
jj git push --bookmark feature/demo
```

指定 remote：

```bash
jj git push --remote origin --bookmark feature/demo
```

先看会推什么：

```bash
jj git push --bookmark feature/demo --dry-run
```

也可以不先创建 bookmark，直接按 change 生成临时 push bookmark：

```bash
jj git push --change @
```

默认生成的名字类似：

```text
push-<change-id>
```

### 删除 bookmark

本地删除，并在下次 push 时传播删除：

```bash
jj bookmark delete feature/demo
jj git push --deleted
```

只忘掉本地 bookmark，不标记远端删除：

```bash
jj bookmark forget feature/demo
```

## 撤销和恢复

### 撤销上一步

```bash
jj undo
```

`jj undo` 撤销上一条 `jj` operation。它不是简单回滚文件，而是恢复 repo 状态。

### 查看操作历史

```bash
jj op log
```

### 恢复到指定 operation

```bash
jj op restore <operation-id>
```

你还可以只查看某次 operation 下的状态：

```bash
jj --at-op <operation-id> st
jj --at-op <operation-id> log
```

这个能力是 `jj` 的安全网。拆错、rebase 错、abandon 错，第一反应应该是：

```bash
jj op log
```

## workspace 和 agent 并行工作流

Git worktree 心智模型：

```text
一个任务 = 一个 worktree 目录 = 一个 Git branch
```

jj workspace 心智模型：

```text
一个任务 = 一个 workspace 目录 = 一个 change 或 stack
需要 push / PR 时，再挂 bookmark
```

### 创建 agent workspace

先同步：

```bash
jj git fetch
```

从主线创建一个给 agent 用的 workspace：

```bash
jj workspace add ../hello-jj-rbac -r master -m "feat: rbac work"
cd ../hello-jj-rbac
```

如果要基于远端主线：

```bash
jj workspace add ../hello-jj-rbac -r master@origin -m "feat: rbac work"
```

### 给 agent 的最小指令

可以这样给子代理：

```text
你在这个 jj workspace 里工作。
只修改当前任务相关文件。
完成后运行 jj st、jj diff，并用 jj describe -m 写清 change 描述。
不要创建 bookmark，不要 push。
```

agent 完成后，你检查：

```bash
jj st
jj diff
jj show @
```

整理变更：

```bash
jj split
jj squash -i
jj rebase -b @ -d master
```

准备推 PR：

```bash
jj bookmark set feature/rbac -r @
jj git push --bookmark feature/rbac
```

清理 workspace：

```bash
cd ../hello-jj-lab
jj workspace forget hello-jj-rbac
rm -rf ../hello-jj-rbac
```

`workspace forget` 只是让当前 repo 忘记那个 workspace；目录删除仍然需要自己处理。

## 完整练习流程

下面是一套从零到常用工作流的练习。建议在临时目录或这个练习仓库里执行。

### 练习 1：基础 change 流程

```bash
jj st
jj log

cat > main.go <<'EOF'
package main

import "fmt"

func main() {
	fmt.Println("hello")
}
EOF

jj describe -m "feat: add main"
jj log

echo '// first app' >> main.go
jj log
jj evolog

jj new -m "feat: add config"
echo 'debug = true' > conf.toml
jj st
jj log
```

观察点：

```text
修改文件不会创建新 change id。
snapshot 后 commit id 会变化。
jj new 会创建新 change id。
```

### 练习 2：分叉

```bash
jj log -r 'all()'
jj new <first-change-id> -m "feat: branch from first change"
echo 'branch A' > branch-a.txt
jj log -r 'all()'
```

观察点：

```text
jj 可以直接从任意旧 change 分叉。
本地分叉不需要 Git branch。
```

### 练习 3：拆分混合 change

```bash
jj new -m "mixed changes"
echo 'a' > a.txt
echo 'b' > b.txt
jj st
jj split
jj log
```

如果想避免交互式 diff editor：

```bash
jj split a.txt -m "feat: add a"
```

观察点：

```text
一个 change 可以拆成两个 change。
拆出来的新 change 会有新的 change id。
```

### 练习 4：修改历史并观察自动 rebase

制造一条线：

```bash
jj new -m "feat: c1"
echo 'c1' > c1.txt

jj new -m "feat: c2"
echo 'c2' > c2.txt

jj new -m "feat: c3"
echo 'c3' > c3.txt

jj log
```

回到底部 change：

```bash
jj edit <c1-change-id>
echo 'c1 modified' > c1.txt
jj log
```

观察点：

```text
c1 的 commit id 会变化。
c2 / c3 会自动重新基于新的 c1。
```

### 练习 5：操作时光机

故意做一个错误操作：

```bash
jj abandon @
```

恢复：

```bash
jj op log
jj undo
```

或者恢复到指定 operation：

```bash
jj op restore <good-operation-id>
```

观察点：

```text
jj 的安全网是 operation log。
出错后先看 jj op log，不要急着乱 reset。
```

## 常见坑

### 1. 把 `jj new` 当成 `git checkout -b`

`jj new` 会把当前已有修改留在原 change，然后移动到新 change。它不是带着未提交修改切到新分支。

正确理解：

```text
当前 change 写完了 -> jj new -> 开始下一个 change
```

### 2. 忘记 bookmark 不一定自动移动

你创建了：

```bash
jj bookmark create feature/demo -r @
```

之后继续修改当前 change，推送前最好显式：

```bash
jj bookmark move feature/demo --to @
```

或者：

```bash
jj bookmark set feature/demo -r @
```

### 3. 在 colocated 仓库里用 Git 命令改状态

可以看：

```bash
git status
git log
git diff
```

谨慎用：

```bash
git checkout
git switch
git reset
git rebase
git commit
```

这些命令可能绕过 `jj` 的操作模型，让工作区状态变复杂。

### 4. 空 change 可能消失

没有描述、没有内容、没有子代的空 change，离开后可能被 abandoned。不要把空 change id 当长期占位符。

### 5. 只记 commit id，不记 change id

commit id 会变。做本地整理时，优先看 change id 和描述。

### 6. 遇到冲突就慌

`jj` 可以把冲突作为仓库状态保存。你可以先查看：

```bash
jj st
jj diff
```

再解决：

```bash
jj resolve
```

如果方向错了：

```bash
jj undo
```

### 7. push 前没有 fetch

`jj git push` 有安全检查，类似 `git push --force-with-lease` 的保护逻辑。推送前先：

```bash
jj git fetch
```

再：

```bash
jj git push --bookmark <name>
```

## 命令速查

### 状态和查看

| 命令 | 用途 |
|---|---|
| `jj st` | 查看状态 |
| `jj log` | 查看默认日志图 |
| `jj log -r 'all()'` | 查看全部 revision 图 |
| `jj show @` | 当前 change 详情 |
| `jj diff` | 当前 change 的 diff |
| `jj evolog` | 当前 change 的 commit id 演化 |
| `jj op log` | 操作历史 |

### 创建和描述 change

| 命令 | 用途 |
|---|---|
| `jj describe -m "msg"` | 修改当前 change 描述 |
| `jj new` | 在当前 change 后创建新 change |
| `jj new <rev>` | 从指定 revision 分叉 |
| `jj new <a> <b>` | 创建 merge change |
| `jj commit -m "msg"` | 描述当前 change 并创建下一个空 change |
| `jj edit <rev>` | 切到已有 change 编辑 |

### 整理历史

| 命令 | 用途 |
|---|---|
| `jj split` | 交互式拆分当前 change |
| `jj split <path> -m "msg"` | 按路径拆出 change |
| `jj squash` | 把当前 change 合到父 change |
| `jj squash -i` | 交互式 squash |
| `jj squash --from <src> --into <dst>` | 指定 source 和 destination |
| `jj rebase -b @ -d <rev>` | 移动当前 stack |
| `jj rebase -r @ -d <rev>` | 只移动当前 revision |
| `jj rebase -s @ -d <rev>` | 移动当前 revision 和 descendants |

### Git 互操作

| 命令 | 用途 |
|---|---|
| `jj git fetch` | 从 Git 远端同步 |
| `jj bookmark list` | 查看 bookmark |
| `jj bookmark create name -r @` | 创建 bookmark |
| `jj bookmark set name -r @` | 创建或更新 bookmark |
| `jj bookmark move name --to @` | 移动 bookmark |
| `jj git push --bookmark name` | 推送指定 bookmark |
| `jj git push --bookmark name --dry-run` | 预览 push |
| `jj git push --change @` | 用当前 change 生成 push bookmark |
| `jj bookmark delete name` | 删除 bookmark 并标记远端删除 |
| `jj bookmark forget name` | 只忘记本地 bookmark |

### 撤销和恢复

| 命令 | 用途 |
|---|---|
| `jj undo` | 撤销上一条 jj operation |
| `jj op log` | 查看 operation 历史 |
| `jj op restore <op-id>` | 恢复到指定 operation |
| `jj --at-op <op-id> st` | 查看某个 operation 下的状态 |

### workspace

| 命令 | 用途 |
|---|---|
| `jj workspace add ../dir -r master -m "msg"` | 创建新 workspace |
| `jj workspace add ../dir -r master@origin -m "msg"` | 基于远端主线创建 workspace |
| `jj workspace list` | 查看 workspace |
| `jj workspace forget <name>` | 忘记 workspace |

## 最重要的判断规则

```text
jj new / jj split / jj commit / jj workspace add:
  通常会创建新的 change id

修改文件 / jj describe / jj log / jj st / jj diff:
  不会创建新的 change id
  但如果触发 snapshot，commit id 可能变化

jj edit:
  不创建新 change
  只是把 @ 移到已有 change

bookmark:
  是对外给 Git/远端看的名字
  本地开发不需要一开始就创建 bookmark

operation log:
  是 jj 的安全网
  出错后先 jj op log / jj undo
```
