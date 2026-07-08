# Git 工作流详解 - 从零到精通

> 本文档涵盖 Git 的核心原理、常用命令、分支策略、团队协作规范及 CI/CD 集成，适合从入门到进阶的开发者阅读。

---

## 目录

- [Part 1: Git 基础原理](#part-1-git-基础原理)
- [Part 2: Git 核心命令大全](#part-2-git-核心命令大全)
- [Part 3: 分支策略](#part-3-分支策略)
- [Part 4: Commit 规范](#part-4-commit-规范)
- [Part 5: 代码评审](#part-5-代码评审)
- [Part 6: Merge vs Rebase](#part-6-merge-vs-rebase)
- [Part 7: 冲突解决](#part-7-冲突解决)
- [Part 8: Git Hooks](#part-8-git-hooks)
- [Part 9: CI/CD 集成](#part-9-cicd-集成)
- [Part 10: 常用操作实战](#part-10-常用操作实战)
- [Part 11: 常见面试题 FAQ](#part-11-常见面试题-faq)

---

## Part 1: Git 基础原理

### 1.1 Git 与 SVN 的对比

| 特性 | Git | SVN |
|------|-----|-----|
| 架构模式 | 分布式 | 集中式 |
| 本地仓库 | 完整历史副本 | 仅工作副本 |
| 离线工作 | 支持 commit/log/diff | 不支持 |
| 分支开销 | 极低（指针操作） | 较高（目录拷贝） |
| 合并能力 | 强大（三路合并） | 较弱 |
| 数据完整性 | SHA-1 校验 | 版本号 |
| 学习曲线 | 较陡 | 平缓 |

**核心差异**：Git 每个开发者本地都有完整的版本库，SVN 必须依赖中央服务器。Git 的分支是轻量级指针，创建/切换几乎零成本；SVN 分支是目录拷贝，代价较高。

### 1.2 Git 三大区域

`
工作区 (Working Directory)
    |  git add
    v
暂存区 (Staging Area / Index)
    |  git commit
    v
本地仓库 (Local Repository)
    |  git push
    v
远程仓库 (Remote Repository)
`

- **工作区**：磁盘上实际的文件目录，开发者直接编辑的地方。
- **暂存区（Index）**：.git/index 文件，保存了下次提交要包含的文件快照。
- **本地仓库**：.git/objects 目录，存储所有对象（blob/tree/commit/tag）。
- **远程仓库**：托管在 GitHub/GitLab/Gitee 等平台上的共享仓库。

### 1.3 Git 对象模型

Git 的底层是一个内容寻址文件系统，所有数据都以对象形式存储，通过 SHA-1 哈希（40位十六进制）寻址。

#### 四种对象类型

| 对象类型 | 说明 | 存储内容 |
|---------|------|----------|
| **blob** | 文件内容 | 文件的二进制内容（不含文件名） |
| **tree** | 目录结构 | blob 和子 tree 的引用列表 |
| **commit** | 提交信息 | tree 引用、父 commit、作者、时间、消息 |
| **tag** | 标签 | commit 引用、标签名、标注信息 |

#### 对象关系图

`
commit
  |-- tree (根目录)
  |     |-- blob (README.md)
  |     |-- blob (pom.xml)
  |     |-- tree (src/)
  |           |-- tree (main/)
  |                 |-- blob (App.java)
  |-- parent commit (上一个提交)
  |-- author
  |-- committer
  |-- message
`

#### 查看对象内容

`ash
# 查看对象类型
git cat-file -t <hash>

# 查看对象内容
git cat-file -p <hash>

# 查看最新 commit 对象
git cat-file -p HEAD

# 查看 tree 对象
git cat-file -p HEAD^{tree}

# 统计对象数量和大小
git count-objects -v
`

### 1.4 .git 目录结构详解

```
.git/
├── HEAD              # 当前检出的分支指针（如 ref: refs/heads/main）
├── config            # 仓库级别配置
├── description       # 仓库描述
├── index             # 暂存区（二进制文件）
├── COMMIT_EDITMSG    # 最近一次 commit 消息
├── MERGE_HEAD        # merge 时目标 commit 的 hash
├── ORIG_HEAD         # 危险操作前保存的 HEAD
├── hooks/            # 钩子脚本目录
│   ├── pre-commit.sample
│   ├── commit-msg.sample
│   └── pre-push.sample
├── info/
│   └── exclude       # 本地忽略规则
├── logs/
│   ├── HEAD          # HEAD 的变更日志（reflog）
│   └── refs/         # 各分支的变更日志
├── objects/          # 所有 Git 对象存储
│   ├── pack/         # 打包后的对象
│   └── xx/           # 松散对象（hash 前两位命名）
└── refs/
    ├── heads/        # 本地分支（文件内容为 commit hash）
    ├── remotes/      # 远程追踪分支
    └── tags/         # 标签
```

**关键文件说明**：

- `HEAD`：当前检出的分支或提交。内容通常是 `ref: refs/heads/main`，detached HEAD 状态时直接是 commit hash。
- `index`：暂存区索引，存储文件名、权限、blob hash 的映射。执行 `git add` 后更新。
- `objects/`：所有对象以 zlib 压缩存储，hash 前两位作子目录名，后38位作文件名。
- `refs/heads/`：每个本地分支对应一个文件，内容是该分支最新 commit 的 hash。

---

## Part 2: Git 核心命令大全

### 2.1 初始化与配置

```bash
# 初始化新仓库
git init
git init <directory>        # 在指定目录初始化
git init --bare             # 初始化裸仓库（用于服务端）

# 克隆远程仓库
git clone <url>
git clone <url> <directory> # 克隆到指定目录
git clone --depth 1 <url>   # 浅克隆（只取最新一次提交）
git clone --branch <branch> <url>  # 克隆指定分支
git clone --recurse-submodules <url>  # 递归克隆子模块

# 全局配置
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global core.editor vim
git config --global init.defaultBranch main
git config --global pull.rebase false     # merge 策略
git config --global alias.lg "log --oneline --graph --all"

# 查看配置
git config --list
git config --list --show-origin  # 显示配置来源文件
git config user.name             # 查看某项配置
```

### 2.2 文件状态管理

```bash
# 查看工作区状态
git status
git status -s               # 短格式输出
git status --ignored        # 显示忽略的文件

# 添加到暂存区
git add <file>              # 添加指定文件
git add .                   # 添加所有变更
git add -A                  # 添加所有（包括删除）
git add -p                  # 交互式分块添加
git add -u                  # 只添加已追踪文件的修改

# 取消暂存
git restore --staged <file> # 从暂存区撤回（推荐，Git 2.23+）
git reset HEAD <file>       # 旧版写法

# 丢弃工作区修改
git restore <file>          # 丢弃工作区修改（Git 2.23+）
git checkout -- <file>      # 旧版写法

# 删除文件
git rm <file>               # 删除文件并暂存
git rm --cached <file>      # 只从暂存区删除，保留工作区
git mv <old> <new>          # 重命名/移动文件
```

### 2.3 提交管理

```bash
# 提交
git commit -m "message"
git commit -am "message"    # add + commit（只对已追踪文件）
git commit --amend          # 修改最近一次提交（慎用已推送的提交）
git commit --amend --no-edit  # 修改提交但不改消息

# 查看提交历史
git log
git log --oneline
git log --oneline --graph --all   # 图形化显示所有分支
git log -p                         # 显示每次提交的 diff
git log -n 5                       # 只显示最近5条
git log --author="name"            # 按作者过滤
git log --since="2024-01-01"       # 按日期过滤
git log --grep="keyword"           # 按提交消息搜索
git log -S "code_string"           # 搜索代码变更（pickaxe）
git log --follow <file>            # 追踪文件重命名历史
git log main..feature              # 查看 feature 比 main 多的提交

# 查看差异
git diff                            # 工作区 vs 暂存区
git diff --staged                   # 暂存区 vs 最新提交
git diff HEAD                       # 工作区 vs 最新提交
git diff <branch1> <branch2>        # 两个分支的差异
git diff <commit1> <commit2>        # 两个提交的差异
git diff --stat                     # 只显示文件统计
```

### 2.4 分支管理

```bash
# 查看分支
git branch                  # 列出本地分支
git branch -r               # 列出远程分支
git branch -a               # 列出所有分支
git branch -v               # 显示每个分支最新提交
git branch --merged         # 已合并到当前分支的分支
git branch --no-merged      # 未合并的分支

# 创建分支
git branch <name>           # 创建分支（不切换）
git checkout -b <name>      # 创建并切换（旧版）
git switch -c <name>        # 创建并切换（Git 2.23+，推荐）
git branch <name> <commit>  # 基于某个提交创建分支

# 切换分支
git checkout <branch>       # 切换分支（旧版）
git switch <branch>         # 切换分支（推荐）
git switch -               # 切换到上一个分支

# 删除分支
git branch -d <name>        # 删除已合并分支
git branch -D <name>        # 强制删除
git push origin --delete <name>  # 删除远程分支

# 重命名分支
git branch -m <old> <new>   # 重命名本地分支
git push origin --delete <old>   # 删除旧远程分支
git push origin <new>            # 推送新分支名
```

### 2.5 合并与变基

```bash
# 合并
git merge <branch>          # 将指定分支合并到当前分支
git merge --no-ff <branch>  # 强制创建合并提交（保留分支历史）
git merge --squash <branch> # 压缩为一次提交
git merge --abort           # 中止合并

# 变基
git rebase <branch>         # 将当前分支变基到目标分支
git rebase --onto <newbase> <upstream> <branch>  # 高级变基
git rebase -i HEAD~3        # 交互式变基最近3个提交
git rebase --continue       # 解决冲突后继续
git rebase --skip           # 跳过当前提交
git rebase --abort          # 中止变基

# Cherry-pick
git cherry-pick <commit>    # 应用某个提交
git cherry-pick A..B        # 应用一段提交范围（不含A）
git cherry-pick A^..B       # 应用一段提交范围（含A）
git cherry-pick --no-commit <commit>  # 只应用变更不提交
```

### 2.6 远程操作

```bash
# 远程仓库管理
git remote -v               # 查看远程仓库
git remote add origin <url> # 添加远程仓库
git remote rename origin upstream  # 重命名
git remote remove origin    # 删除远程仓库
git remote set-url origin <url>    # 修改 URL

# 获取与拉取
git fetch origin            # 获取远程所有分支（不合并）
git fetch origin main       # 获取指定分支
git fetch --all             # 获取所有远程
git fetch --prune           # 同时删除本地已不存在的远程追踪分支

git pull                    # fetch + merge
git pull --rebase           # fetch + rebase（推荐，保持线性历史）
git pull origin main        # 拉取指定远程分支

# 推送
git push origin main        # 推送到远程
git push -u origin main     # 推送并设置上游追踪
git push --force-with-lease # 安全强推（推荐替代 --force）
git push origin --tags      # 推送所有标签
git push origin <tag>       # 推送指定标签
```

### 2.7 撤销与重置

```bash
# reset - 移动 HEAD 指针
git reset --soft HEAD~1     # 撤销提交，保留暂存区和工作区
git reset --mixed HEAD~1    # 撤销提交和暂存，保留工作区（默认）
git reset --hard HEAD~1     # 彻底回退，丢弃所有变更（危险！）
git reset <commit>          # 回退到指定提交

# revert - 创建反向提交（安全，可用于已推送分支）
git revert <commit>         # 撤销某次提交
git revert HEAD~3..HEAD     # 撤销最近3次提交
git revert --no-commit <commit>  # 只应用撤销变更不提交

# clean - 清理未追踪文件
git clean -n                # 预览要删除的文件（dry-run）
git clean -f                # 删除未追踪文件
git clean -fd               # 删除未追踪文件和目录
git clean -fX               # 删除被忽略的文件
git clean -fdx              # 删除所有未追踪文件（含忽略）
```

### 2.8 Stash 暂存

```bash
# 保存工作区
git stash                   # 暂存当前所有变更
git stash push -m "message" # 带描述的暂存
git stash push <file>       # 只暂存指定文件
git stash -u                # 包含未追踪文件
git stash --all             # 包含所有文件（含忽略）

# 查看 stash
git stash list
git stash show stash@{0}    # 查看指定 stash 内容
git stash show -p           # 显示 diff

# 恢复 stash
git stash pop               # 恢复最新 stash 并删除
git stash apply stash@{1}   # 恢复指定 stash（不删除）
git stash branch <branch>   # 从 stash 创建新分支

# 删除 stash
git stash drop stash@{0}    # 删除指定 stash
git stash clear             # 清空所有 stash
```

### 2.9 标签管理

```bash
# 创建标签
git tag v1.0.0              # 轻量标签（只是指针）
git tag -a v1.0.0 -m "Release 1.0.0"  # 附注标签（推荐）
git tag -a v1.0.0 <commit>  # 为历史提交打标签

# 查看标签
git tag
git tag -l "v1.*"           # 按模式过滤
git show v1.0.0             # 查看标签详情

# 推送标签
git push origin v1.0.0      # 推送单个标签
git push origin --tags      # 推送所有标签

# 删除标签
git tag -d v1.0.0           # 删除本地标签
git push origin --delete v1.0.0  # 删除远程标签
```

### 2.10 调试工具

```bash
# blame - 逐行追责
git blame <file>
git blame -L 10,20 <file>   # 只看第10-20行
git blame -w <file>         # 忽略空格变更
git blame -C <file>         # 检测跨文件的代码移动

# bisect - 二分法定位 bug
git bisect start
git bisect bad              # 标记当前版本有问题
git bisect good v1.0.0      # 标记已知正常版本
# Git 自动切换版本，测试后标记
git bisect good             # 测试正常
git bisect bad              # 测试有问题
git bisect reset            # 结束 bisect

# reflog - 引用日志（找回丢失的提交）
git reflog                  # 查看 HEAD 的所有变更记录
git reflog show <branch>    # 查看某分支的 reflog
git checkout HEAD@{3}       # 切换到 reflog 中的某个状态
```

---

## Part 3: 分支策略

### 3.1 Git Flow

Git Flow 是 Vincent Driessen 提出的经典分支模型，适合有明确版本发布周期的项目。

#### 分支结构

```
main          ──●──────────────────────●── (只接受 release/hotfix 合并)
                │                      │
develop       ──●──●──●──●──●──────────●── (集成分支)
                   │     │
feature/login ─────●──●──┘           (功能分支)
feature/pay   ─────────●──●──●──┘
                                |
release/1.0   ─────────────────●──●──┘  (预发布分支)
hotfix/1.0.1  ──────────────────────●──┘ (紧急修复)
```

#### 各分支说明

| 分支 | 生命周期 | 来源 | 合并到 | 说明 |
|------|---------|------|--------|------|
| `main` | 永久 | — | — | 生产代码，每次合并打 tag |
| `develop` | 永久 | main | — | 集成分支，最新开发版 |
| `feature/*` | 临时 | develop | develop | 功能开发 |
| `release/*` | 临时 | develop | main+develop | 预发布修复 |
| `hotfix/*` | 临时 | main | main+develop | 生产紧急修复 |

#### 操作流程示例

```bash
# 开始功能开发
git switch develop
git switch -c feature/user-auth
# ... 开发提交 ...
git switch develop
git merge --no-ff feature/user-auth
git branch -d feature/user-auth

# 创建发布分支
git switch -c release/1.2.0 develop
# 修复发布相关 bug，更新版本号
git switch main
git merge --no-ff release/1.2.0
git tag -a v1.2.0 -m "Release 1.2.0"
git switch develop
git merge --no-ff release/1.2.0
git branch -d release/1.2.0

# 紧急修复
git switch -c hotfix/1.2.1 main
# 修复 bug
git switch main
git merge --no-ff hotfix/1.2.1
git tag -a v1.2.1 -m "Hotfix 1.2.1"
git switch develop
git merge --no-ff hotfix/1.2.1
git branch -d hotfix/1.2.1
```

**适用场景**：版本号明确、需要长期维护旧版本的项目（如桌面软件、移动 App）。

**缺点**：分支较多，流程复杂，不适合持续部署场景。

### 3.2 GitHub Flow

GitHub Flow 是一种轻量级工作流，只有一个长期分支 `main`，通过 Pull Request 进行代码合并。

#### 流程步骤

```
1. 从 main 创建功能分支
2. 在功能分支上开发并频繁提交
3. 推送分支并开 Pull Request
4. Code Review 通过后合并到 main
5. main 立即部署到生产
```

```bash
git switch main
git pull origin main
git switch -c feature/add-login

# 开发...
git add .
git commit -m "feat: add login page"
git push -u origin feature/add-login

# 在 GitHub 上创建 PR，通过 Review 后合并
# 合并后删除远程分支
git push origin --delete feature/add-login
git branch -d feature/add-login
```

**适用场景**：Web 应用、SaaS 产品、持续部署环境。

**优点**：简单直接，适合小团队和高频发布。

### 3.3 GitLab Flow

GitLab Flow 在 GitHub Flow 基础上增加了环境分支，解决了多环境部署问题。

#### 环境分支模型

```
main ──→ pre-production ──→ production
```

```bash
# 功能开发同 GitHub Flow
# 合并到 main 后，通过 merge 推进到各环境分支
git switch pre-production
git merge main
git push origin pre-production

# 测试通过后推进到 production
git switch production
git merge pre-production
git push origin production
```

#### 版本分支模型（适合开源软件）

```
main
 ├── 2-3-stable
 ├── 2-4-stable
 └── 3-0-stable
```

**适用场景**：需要区分多套环境（开发/测试/预发/生产）的企业项目。

### 3.4 Trunk Based Development（主干开发）

所有开发者直接向主干（main/trunk）提交，或通过极短生命周期（1-2天）的特性分支集成。

```bash
# 小团队：直接提交到 main
git switch main
git pull --rebase
# 快速开发
git commit -m "feat: small feature"
git push origin main

# 大团队：短生命周期特性分支
git switch -c feature/tiny-change
# 1-2天内完成并合并
```

**关键实践**：
- 功能开关（Feature Flags）：未完成的功能用开关隐藏
- 持续集成：每次提交触发完整测试套件
- 小批量提交：每次提交尽量小而完整

**适用场景**：DevOps 成熟、测试覆盖率高、持续部署的团队（Google、Facebook 等采用）。

### 3.5 分支命名规范

```
功能分支:  feature/<ticket-id>-<short-desc>
           feature/JIRA-123-user-authentication

修复分支:  fix/<ticket-id>-<short-desc>
           fix/BUG-456-login-timeout

发布分支:  release/<version>
           release/2.1.0

热修复:    hotfix/<version>-<short-desc>
           hotfix/2.0.1-null-pointer

实验分支:  experiment/<desc>
           experiment/new-payment-flow

个人分支:  users/<username>/<desc>
           users/zhangsan/refactor-service
```

---

## Part 4: Commit 规范

### 4.1 Conventional Commits 规范

Conventional Commits 是一种结构化的提交消息规范，格式如下：

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

#### Type 类型

| Type | 说明 | 触发版本号 |
|------|------|----------|
| `feat` | 新功能 | MINOR |
| `fix` | Bug 修复 | PATCH |
| `docs` | 文档变更 | 无 |
| `style` | 代码格式（不影响逻辑） | 无 |
| `refactor` | 重构（无新功能/修复） | 无 |
| `perf` | 性能优化 | PATCH |
| `test` | 添加/修改测试 | 无 |
| `build` | 构建系统或依赖变更 | 无 |
| `ci` | CI 配置变更 | 无 |
| `chore` | 其他杂项（不影响源码） | 无 |
| `revert` | 回滚提交 | — |

#### 示例

```
# 简单功能
feat(auth): add JWT token refresh mechanism

# 带 body 的提交
fix(payment): handle null pointer in checkout flow

Previously, when user cart was empty, the checkout service
would throw NPE. Now returns 400 with proper error message.

Closes #234

# 破坏性变更（触发 MAJOR 版本号）
feat(api)!: remove deprecated v1 endpoints

BREAKING CHANGE: The /api/v1/* endpoints have been removed.
Please migrate to /api/v2/* as documented in MIGRATION.md.

# scope 可选，指明影响范围
feat(user-profile): add avatar upload functionality
fix(database): correct connection pool timeout setting
docs(readme): update installation instructions
```

### 4.2 commitlint + husky 配置

commitlint 用于检查提交消息是否符合规范，husky 用于在 Git hooks 中自动执行检查。

#### 安装

```bash
# 安装依赖
npm install --save-dev @commitlint/cli @commitlint/config-conventional
npm install --save-dev husky

# 初始化 husky
npx husky init
```

#### commitlint.config.js

```javascript
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2, 'always',
      ['feat', 'fix', 'docs', 'style', 'refactor', 'perf', 'test', 'build', 'ci', 'chore', 'revert']
    ],
    'subject-max-length': [2, 'always', 72],
    'subject-case': [2, 'always', 'lower-case'],
    'body-max-line-length': [2, 'always', 100],
  },
};
```

#### .husky/commit-msg

```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"
npx --no -- commitlint --edit $1
```

### 4.3 CHANGELOG 自动生成

使用 `standard-version` 或 `conventional-changelog` 根据提交历史自动生成 CHANGELOG。

```bash
# 安装
npm install --save-dev standard-version

# package.json 配置
```
```json
{
  "scripts": {
    "release": "standard-version",
    "release:minor": "standard-version --release-as minor",
    "release:major": "standard-version --release-as major",
    "release:patch": "standard-version --release-as patch"
  }
}
```

```bash
# 执行发布（自动更新版本号、生成 CHANGELOG、打 tag）
npm run release
git push --follow-tags origin main
```

#### 生成的 CHANGELOG.md 示例

```markdown
# Changelog

## [2.1.0] - 2024-03-15

### Features

* **auth:** add JWT token refresh mechanism (#123)
* **user-profile:** add avatar upload functionality (#145)

### Bug Fixes

* **payment:** handle null pointer in checkout flow (#234)
* **database:** correct connection pool timeout setting (#198)

## [2.0.0] - 2024-02-01

### BREAKING CHANGES

* **api:** remove deprecated v1 endpoints
```

---

## Part 5: 代码评审（Code Review）

### 5.1 Pull Request 模板

在仓库根目录创建 `.github/PULL_REQUEST_TEMPLATE.md`：

```markdown
## 变更描述

<!-- 简要说明本次 PR 的目的和背景 -->

## 变更类型

- [ ] feat: 新功能
- [ ] fix: Bug 修复
- [ ] refactor: 重构
- [ ] docs: 文档更新
- [ ] style: 代码格式
- [ ] test: 测试相关
- [ ] chore: 构建/工具链

## 关联 Issue

Closes #

## 自测清单

- [ ] 本地测试通过
- [ ] 单元测试已添加/更新
- [ ] 接口文档已更新（如有变更）
- [ ] 无敏感信息泄露
- [ ] 无明显性能问题

## 截图/录屏（如适用）

<!-- 如果涉及 UI 变更，请附上截图 -->

## 其他说明

<!-- 给 Reviewer 的补充信息 -->
```

### 5.2 Code Review 检查清单

#### 代码正确性
- [ ] 逻辑正确，边界条件处理完整
- [ ] 异常处理适当，错误消息清晰
- [ ] 并发场景考虑（线程安全、锁、原子操作）
- [ ] 输入验证完整（防 SQL 注入、XSS 等）

#### 代码质量
- [ ] 命名清晰，符合项目规范
- [ ] 函数职责单一，长度适当（< 50行建议）
- [ ] 无重复代码（DRY 原则）
- [ ] 复杂逻辑有注释说明
- [ ] 无调试代码、TODO 未处理

#### 测试
- [ ] 关键路径有单元测试覆盖
- [ ] 测试用例覆盖正常流和异常流
- [ ] 测试代码本身质量良好

#### 性能与安全
- [ ] 无 N+1 查询问题
- [ ] 大数据量场景有分页/流式处理
- [ ] 敏感数据不记录日志
- [ ] 密钥/密码不硬编码

### 5.3 CODEOWNERS 配置

在 `.github/CODEOWNERS` 或根目录 `CODEOWNERS` 中定义代码所有者，PR 会自动请求 Review：

```
# 全局所有者（所有文件变更都需要他们 review）
* @team-lead

# 后端代码
/src/main/java/    @backend-team

# 前端代码
/src/frontend/     @frontend-team

# 安全相关
/src/security/     @security-team @cto

# CI/CD 配置
/.github/          @devops-team
/Dockerfile        @devops-team

# 数据库迁移
/src/main/resources/db/migration/  @dba-team @backend-lead
```

### 5.4 分支保护规则

在 GitHub Settings → Branches 中配置保护规则：

```yaml
# 建议的 main 分支保护配置
branch_protection:
  branch: main
  rules:
    - require_pull_request_reviews:
        required_approving_review_count: 2
        dismiss_stale_reviews: true
        require_code_owner_reviews: true
    - require_status_checks:
        strict: true  # 要求 PR 分支是最新的
        contexts:
          - "CI / build"
          - "CI / test"
          - "CI / lint"
    - restrict_pushes: true        # 禁止直接 push
    - require_signed_commits: true  # 要求签名提交
    - enforce_admins: true          # 管理员也遵守规则
```

### 5.5 高效 Review 技巧

**作为 Author**：
- PR 大小控制在 400 行以内（大 PR Review 质量显著下降）
- 在 PR 描述中说明设计决策和取舍
- 自己先 Review 一遍再请他人
- 主动在复杂代码处添加注释

**作为 Reviewer**：
- 区分 blocking（必须修改）和 nit（建议改进）
- 给出具体的改进建议，而非仅说「这不好」
- 认可好的实践（正向反馈）
- 24小时内完成 Review（避免阻塞）

---

## Part 6: Git Merge vs Rebase

### 6.1 Merge 详解

Merge 将两个分支的历史合并，创建一个新的合并提交（merge commit）。

```
before merge:
main:    A ── B ── C
                    \
feature:             D ── E

after: git merge feature
main:    A ── B ── C ── F  (F 是 merge commit，有两个父节点)
                    \   /
feature:             D─E
```

```bash
git switch main
git merge feature

# 强制创建 merge commit（即使可以 fast-forward）
git merge --no-ff feature

# Fast-forward merge（只在可以 ff 时执行，否则失败）
git merge --ff-only feature
```

**优点**：保留完整历史，能看到每个功能分支的完整轨迹。
**缺点**：历史图复杂，merge commit 较多时日志不易阅读。

### 6.2 Rebase 详解

Rebase 将当前分支的提交「移植」到目标分支的顶端，创建全新的提交（新 hash）。

```
before rebase:
main:    A ── B ── C
              \
feature:       D ── E

after: git rebase main (在 feature 分支上执行)
main:    A ── B ── C
                    \
feature:             D' ── E'  (D', E' 是全新提交)

after: git merge feature (此时为 fast-forward)
main:    A ── B ── C ── D' ── E'  (线性历史)
```

```bash
git switch feature
git rebase main
# 解决冲突后
git rebase --continue

# 变基完成后，切回 main 合并（fast-forward）
git switch main
git merge feature
```

**优点**：产生线性历史，`git log --oneline` 清晰易读。
**缺点**：修改了提交历史，不能用于已推送到共享仓库的分支（黄金法则）。

### 6.3 Interactive Rebase（交互式变基）

```bash
# 对最近4个提交进行整理
git rebase -i HEAD~4
```

编辑器中显示：

```
pick abc1234 feat: add login page
pick def5678 fix typo in login
pick ghi9012 WIP: more login work
pick jkl3456 feat: complete login with validation

# 命令说明：
# p, pick   = 保留该提交
# r, reword = 保留但修改提交消息
# e, edit   = 保留但在此处暂停以修改
# s, squash = 与上一个提交合并（保留消息）
# f, fixup  = 与上一个提交合并（丢弃消息）
# d, drop   = 删除该提交
# x, exec   = 执行 shell 命令
```

修改为：

```
pick abc1234 feat: add login page
fixup def5678 fix typo in login
fixup ghi9012 WIP: more login work
reword jkl3456 feat: complete login with validation
```

常见操作：
- **squash/fixup**：合并多个小提交为一个清晰的提交
- **reword**：修改历史提交的消息
- **drop**：删除错误提交
- **edit**：拆分一个大提交为多个小提交

### 6.4 黄金法则

> **永远不要在公共/共享分支上执行 rebase！**

rebase 会重写提交历史（改变 hash），如果其他人已经基于旧提交工作，会造成严重混乱。

```
安全 rebase：
  本地功能分支 rebase 到最新的 main/develop 上

危险 rebase（禁止）：
  对已 push 到远程的共享分支执行 rebase
  对 main/develop 执行 rebase
```

### 6.5 Squash Merge

将功能分支的所有提交压缩为一个提交后合并，保持 main 历史简洁。

```bash
git switch main
git merge --squash feature/login
git commit -m "feat(auth): complete login feature (#123)"
```

**比较**：

| 方式 | 历史形状 | 保留分支细节 | 适用场景 |
|------|---------|------------|----------|
| merge | 非线性（dag） | 是 | 需要完整历史审计 |
| rebase + merge | 线性 | 是（功能分支内） | 主流开源项目 |
| squash merge | 线性 | 否 | 功能分支 commit 质量不高时 |

---

## Part 7: 冲突解决

### 7.1 冲突标记说明

当 Git 无法自动合并时，会在文件中插入冲突标记：

```
<<<<<<< HEAD
// 当前分支（HEAD）的内容
public String getName() {
    return this.name.trim();
}
=======
// 传入分支的内容
public String getName() {
    return this.name.toLowerCase();
}
>>>>>>> feature/user-update
```

- `<<<<<<< HEAD`：当前分支变更的开始
- `=======`：分隔线
- `>>>>>>> branch-name`：传入分支变更的结束

rebase 时标记略有不同：
```
<<<<<<< HEAD         (目标分支，即 rebase 的基础)
...
=======
...
>>>>>>> abc1234      (正在应用的提交)
```

### 7.2 解决冲突的步骤

```bash
# 1. 查看冲突文件
git status
git diff --diff-filter=U  # 只显示冲突文件

# 2. 手动编辑冲突文件，选择保留的内容

# 3. 标记冲突已解决
git add <resolved-file>

# 4. 继续操作
git merge --continue   # 或
git rebase --continue  # 或
git cherry-pick --continue

# 5. 如果想放弃，回到操作前状态
git merge --abort
git rebase --abort
```

### 7.3 ours / theirs 策略

```bash
# 合并时完全使用本分支版本（放弃 theirs）
git checkout --ours <file>
git add <file>

# 合并时完全使用对方分支版本（放弃 ours）
git checkout --theirs <file>
git add <file>

# 批量解决：使用某一侧的策略
git merge -X ours feature-branch    # 冲突时优先选当前分支
git merge -X theirs feature-branch  # 冲突时优先选对方分支
```

注意 rebase 时 ours/theirs 含义与 merge 相反：
- merge 中 `ours` = 当前分支，`theirs` = 传入分支
- rebase 中 `ours` = 目标分支（rebase onto），`theirs` = 原分支的提交

### 7.4 冲突解决工具

```bash
# 配置 merge tool
git config --global merge.tool vimdiff
git config --global merge.tool vscode

# 使用配置的 merge tool
git mergetool
git mergetool <file>  # 只解决指定文件
```

#### VSCode 配置

```json
// .git/config 或 ~/.gitconfig
[merge]
    tool = vscode
[mergetool "vscode"]
    cmd = code --wait $MERGED
[diff]
    tool = vscode
[difftool "vscode"]
    cmd = code --wait --diff $LOCAL $REMOTE
```

### 7.5 预防冲突的最佳实践

1. **频繁同步**：定期将主分支同步到功能分支（`git pull --rebase` 或 `git rebase main`）
2. **小粒度提交**：每个提交只做一件事，减少冲突范围
3. **模块化设计**：不同功能修改不同文件/模块，避免多人修改同一区域
4. **沟通协调**：在修改共享核心模块前通知团队
5. **使用 .editorconfig**：统一代码格式，避免空白符/换行符冲突

---

## Part 8: Git Hooks

### 8.1 Hooks 概述

Git Hooks 是在特定 Git 事件（提交、推送等）前后自动执行的脚本，存放在 `.git/hooks/` 目录。

#### 客户端 Hooks

| Hook | 触发时机 | 常见用途 |
|------|---------|----------|
| `pre-commit` | `git commit` 前 | lint、格式检查、单元测试 |
| `prepare-commit-msg` | 打开编辑器前 | 自动填充提交模板 |
| `commit-msg` | 提交消息写入后 | 验证 commit message 格式 |
| `post-commit` | 提交完成后 | 通知、触发脚本 |
| `pre-rebase` | rebase 前 | 防止 rebase 共享分支 |
| `pre-push` | `git push` 前 | 运行测试、检查分支 |
| `post-checkout` | checkout 后 | 安装依赖、切换环境 |
| `post-merge` | merge 后 | 安装新依赖 |

#### 服务端 Hooks

| Hook | 触发时机 | 常见用途 |
|------|---------|----------|
| `pre-receive` | 接收 push 前 | 权限检查、格式验证 |
| `update` | 更新每个引用前 | 分支级别权限控制 |
| `post-receive` | 接收 push 后 | 触发 CI/CD、发通知 |

### 8.2 常用 Hook 脚本

#### pre-commit：代码检查

```bash
#!/bin/sh
# .git/hooks/pre-commit

echo "Running pre-commit checks..."

# 1. 运行代码格式检查
if command -v eslint &> /dev/null; then
    npx eslint --fix-dry-run $(git diff --cached --name-only --diff-filter=ACM | grep '.js$')
    if [ $? -ne 0 ]; then
        echo "ESLint check failed. Please fix errors before committing."
        exit 1
    fi
fi

# 2. 防止提交调试代码
if git diff --cached | grep -E '^\+.*(console\.log|debugger|TODO:)' > /dev/null 2>&1; then
    echo "Warning: Found console.log/debugger/TODO in staged changes"
    # exit 1  # 取消注释则阻止提交
fi

# 3. 防止直接提交到 main/master
BRANCH=$(git symbolic-ref --short HEAD)
if [ "$BRANCH" = "main" ] || [ "$BRANCH" = "master" ]; then
    echo "Error: Direct commit to $BRANCH is not allowed!"
    exit 1
fi

exit 0
```

#### commit-msg：消息格式验证

```bash
#!/bin/sh
# .git/hooks/commit-msg

COMMIT_MSG_FILE=$1
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

# Conventional Commits 格式验证
PATTERN='^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\(.+\))?!?: .{1,72}'

if ! echo "$COMMIT_MSG" | grep -qE "$PATTERN"; then
    echo "Error: Commit message format is invalid!"
    echo "Expected: <type>[scope]: <description>"
    echo "Example:  feat(auth): add JWT refresh token"
    exit 1
fi

exit 0
```

#### pre-push：推送前测试

```bash
#!/bin/sh
# .git/hooks/pre-push

PROTECTED_BRANCHES="main master release"
CURRENT_BRANCH=$(git symbolic-ref --short HEAD)

for BRANCH in $PROTECTED_BRANCHES; do
    if [ "$CURRENT_BRANCH" = "$BRANCH" ]; then
        echo "Running tests before pushing to $BRANCH..."
        npm test
        if [ $? -ne 0 ]; then
            echo "Tests failed! Push aborted."
            exit 1
        fi
    fi
done

exit 0
```

### 8.3 Husky 管理 Hooks

Husky 让 Git Hooks 可以版本控制并团队共享：

```bash
# 安装
npm install --save-dev husky
npx husky init

# 添加 pre-commit hook
echo "npm run lint" > .husky/pre-commit

# 添加 commit-msg hook（配合 commitlint）
echo "npx --no -- commitlint --edit \$1" > .husky/commit-msg

# 添加 pre-push hook
echo "npm test" > .husky/pre-push
```

#### 配合 lint-staged（只检查 staged 文件）

```bash
npm install --save-dev lint-staged
```

```json
// package.json
{
  "lint-staged": {
    "*.{js,ts}": ["eslint --fix", "git add"],
    "*.{css,scss}": ["stylelint --fix"],
    "*.{js,ts,json,md}": ["prettier --write"]
  }
}
```

```bash
# .husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"
npx lint-staged
```

---

## Part 9: CI/CD 集成

### 9.1 GitHub Actions

#### 基础 Java CI 工作流

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        java: ['11', '17', '21']

    steps:
    - uses: actions/checkout@v4

    - name: Set up JDK ${{ matrix.java }}
      uses: actions/setup-java@v4
      with:
        java-version: ${{ matrix.java }}
        distribution: 'temurin'
        cache: maven

    - name: Build and Test
      run: mvn -B verify --no-transfer-progress

    - name: Upload coverage
      uses: codecov/codecov-action@v3
      if: matrix.java == '17'

    - name: Upload test results
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: test-results-${{ matrix.java }}
        path: target/surefire-reports/
```

#### 发布工作流

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
        server-id: ossrh
        server-username: MAVEN_USERNAME
        server-password: MAVEN_PASSWORD
        gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}

    - name: Publish to Maven Central
      run: mvn --no-transfer-progress --batch-mode deploy -P release
      env:
        MAVEN_USERNAME: ${{ secrets.OSSRH_USERNAME }}
        MAVEN_PASSWORD: ${{ secrets.OSSRH_TOKEN }}

    - name: Create GitHub Release
      uses: actions/create-release@v1
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        tag_name: ${{ github.ref }}
        release_name: Release ${{ github.ref }}
        draft: false
        prerelease: false
```

### 9.2 GitLab CI/CD

```yaml
# .gitlab-ci.yml
image: maven:3.9.6-eclipse-temurin-17

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"

cache:
  paths:
    - .m2/repository

stages:
  - validate
  - build
  - test
  - security
  - deploy

validate:
  stage: validate
  script:
    - mvn validate

build:
  stage: build
  script:
    - mvn compile -DskipTests
  artifacts:
    paths:
      - target/

test:
  stage: test
  script:
    - mvn test
  coverage: '/Total.*?([0-9]{1,3})%/'
  artifacts:
    reports:
      junit: target/surefire-reports/*.xml
      coverage_report:
        coverage_format: jacoco
        path: target/site/jacoco/jacoco.xml

security_scan:
  stage: security
  script:
    - mvn dependency-check:check
  allow_failure: true

deploy_staging:
  stage: deploy
  script:
    - echo "Deploying to staging..."
    - mvn package -DskipTests
    - scp target/*.jar user@staging:/apps/
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop

deploy_prod:
  stage: deploy
  script:
    - echo "Deploying to production..."
  environment:
    name: production
  when: manual  # 需要手动触发
  only:
    - main
```

### 9.3 Jenkins Pipeline

```groovy
// Jenkinsfile
pipeline {
    agent any

    tools {
        maven 'Maven 3.9'
        jdk 'JDK 17'
    }

    environment {
        NEXUS_URL = 'http://nexus.company.com:8081'
        DOCKER_REGISTRY = 'registry.company.com'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                    jacoco execPattern: 'target/*.exec'
                }
            }
        }

        stage('Deploy to Nexus') {
            when {
                branch 'main'
            }
            steps {
                sh 'mvn deploy -DskipTests'
            }
        }
    }

    post {
        failure {
            emailext to: 'team@company.com',
                     subject: "Build FAILED: ${env.JOB_NAME}",
                     body: "Build ${env.BUILD_NUMBER} failed. Check: ${env.BUILD_URL}"
        }
    }
}
```

---

## Part 10: 常用操作实战

### 10.1 git reflog 找回丢失的提交

```bash
# 场景：不小心执行了 git reset --hard，丢失了提交

# 查看 reflog 找到丢失的 commit hash
git reflog
# 输出示例：
# abc1234 HEAD@{0}: reset: moving to HEAD~2
# def5678 HEAD@{1}: commit: feat: important feature  <-- 这是丢失的提交
# ghi9012 HEAD@{2}: commit: fix: bug fix

# 方法1：恢复到丢失的状态
git reset --hard HEAD@{1}

# 方法2：创建新分支保存丢失的提交
git branch recover-branch HEAD@{1}

# 方法3：cherry-pick 应用丢失的提交
git cherry-pick def5678
```

### 10.2 Git LFS（大文件存储）

```bash
# 安装 Git LFS
git lfs install

# 追踪大文件类型
git lfs track "*.psd"
git lfs track "*.mp4"
git lfs track "*.zip"
git lfs track "assets/**"

# 查看追踪规则（保存在 .gitattributes）
git lfs track

# 提交 .gitattributes
git add .gitattributes
git commit -m "chore: configure Git LFS tracking"

# 查看 LFS 文件状态
git lfs status
git lfs ls-files

# 迁移已有文件到 LFS
git lfs migrate import --include="*.psd"
```

### 10.3 Git Submodule

```bash
# 添加子模块
git submodule add https://github.com/org/lib.git libs/mylib
git commit -m "chore: add mylib submodule"

# 克隆含子模块的仓库
git clone --recurse-submodules <url>
# 或克隆后初始化
git submodule update --init --recursive

# 更新子模块到最新
git submodule update --remote
git submodule update --remote --merge

# 查看子模块状态
git submodule status

# 删除子模块
git submodule deinit libs/mylib
git rm libs/mylib
rm -rf .git/modules/libs/mylib
git commit -m "chore: remove mylib submodule"
```

### 10.4 Monorepo 管理

```bash
# 使用 git sparse-checkout 只检出部分目录
git clone --filter=blob:none --no-checkout <url>
cd repo
git sparse-checkout init --cone
git sparse-checkout set packages/frontend packages/shared
git checkout main

# 修改 sparse-checkout 范围
git sparse-checkout add packages/backend

# 查看当前 sparse-checkout 配置
git sparse-checkout list
```

### 10.5 清理历史中的敏感信息

```bash
# 使用 git filter-repo（推荐，需单独安装）
pip install git-filter-repo

# 删除某个文件的所有历史
git filter-repo --path secrets.txt --invert-paths

# 替换文件中的敏感字符串
git filter-repo --replace-text replacements.txt
# replacements.txt 格式：
# literal:my-secret-key==>REDACTED

# 清理完成后强推
git push origin --force --all
git push origin --force --tags

# 注意：团队成员需要重新 clone 仓库
```

### 10.6 优化大仓库

```bash
# 垃圾回收和压缩
git gc                    # 普通垃圾回收
git gc --aggressive       # 深度优化（耗时但效果好）
git prune                 # 删除不可达对象

# 查看仓库大小
git count-objects -vH

# 找出最大的文件
git rev-list --objects --all | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | sed -n 's/^blob //p' | sort --numeric-sort --key=2 | tail -10

# 浅克隆减少数据量
git clone --depth 50 <url>   # 只克隆最近50个提交
git fetch --deepen 100       # 加深历史
git fetch --unshallow         # 完整历史
```

### 10.7 实用技巧合集

```bash
# 快速切换到上一个分支
git switch -

# 查看某文件在某版本的内容
git show HEAD~3:src/main/App.java

# 从另一个分支获取单个文件
git checkout feature-branch -- src/utils/helper.js

# 创建空提交（触发 CI）
git commit --allow-empty -m "ci: trigger rebuild"

# 查找引入 bug 的提交（配合 git bisect）
git bisect start
git bisect bad HEAD
git bisect good v1.0.0
git bisect run mvn test  # 自动化二分查找

# 临时保存未完成工作并切换分支
git stash push -m "WIP: half-done feature"
git switch hotfix/urgent
# 修复完后
git switch feature/my-work
git stash pop

# 查看两个分支的共同祖先
git merge-base main feature

# 导出 patch 文件（用于邮件传输）
git format-patch -3                    # 导出最近3个提交为 patch
git am < fix-bug.patch                 # 应用 patch
```

---

## Part 11: 常见面试题 FAQ

### Q1. Git 和 SVN 最核心的区别是什么？

**答**：核心区别在于架构模式。Git 是**分布式**版本控制系统，每个开发者本地都有完整的版本仓库（包含全部历史），可以离线提交、查看历史、创建分支；SVN 是**集中式**系统，所有历史都在中央服务器，本地只有工作副本。

其次，Git 的分支是轻量级指针（创建成本接近零），而 SVN 的分支是目录拷贝。Git 使用 SHA-1 内容寻址确保数据完整性，SVN 使用递增版本号。

### Q2. git merge 和 git rebase 的区别？什么时候用哪个？

**答**：
- `git merge`：创建合并提交，**保留分支历史**，历史是非线性的 DAG 图。
- `git rebase`：将提交「移植」到目标分支顶端，产生**线性历史**，但会改变提交 hash。

**使用建议**：
- 本地功能分支同步主分支：用 `rebase`（`git pull --rebase` 或 `git rebase main`），保持线性历史。
- 功能分支合并到主分支：`--no-ff merge`（保留分支轨迹）或 `squash merge`（整洁历史）。
- **绝不对已 push 的共享分支执行 rebase**（黄金法则）。

### Q3. git reset 的三种模式有什么区别？

**答**：
- `--soft`：只移动 HEAD 指针，暂存区和工作区不变。适合「撤销提交但保留所有变更到暂存区，准备重新提交」。
- `--mixed`（默认）：移动 HEAD，重置暂存区，工作区不变。适合「撤销提交和暂存，变更仍在工作区」。
- `--hard`：移动 HEAD，重置暂存区和工作区。**完全丢弃变更**，不可通过常规方式恢复（可用 reflog）。

### Q4. git revert 和 git reset 的区别？

**答**：
- `git reset`：通过**移动 HEAD 指针**来撤销提交，改变了历史，**不安全**（不能用于已推送的共享分支）。
- `git revert`：通过创建一个**新的反向提交**来撤销变更，不修改历史，是**安全的撤销方式**，可用于已推送的分支。

### Q5. 什么是 detached HEAD 状态？如何处理？

**答**：当 HEAD 直接指向某个 commit（而非分支）时，就处于 detached HEAD 状态。通常发生在 `git checkout <commit-hash>` 或 `git checkout <tag>` 时。

在此状态下可以查看和实验，但提交的变更可能丢失（没有分支追踪这些提交）。

处理方式：
```bash
# 如果想保留提交，创建新分支
git switch -c new-branch

# 如果不需要保留，直接切回分支
git switch main
```

### Q6. 如何撤销一次已推送到远程的 commit？

**答**：使用 `git revert`（安全方式）：
```bash
git revert <commit-hash>
git push origin main
```

这会创建一个新提交来撤销变更，不修改历史，是团队协作中的正确做法。

如果是个人分支且确认没有其他人基于该提交工作，也可以强推（谨慎使用）：
```bash
git reset --hard HEAD~1
git push --force-with-lease origin feature-branch
```

### Q7. git fetch 和 git pull 的区别？

**答**：
- `git fetch`：从远程下载最新数据到本地远程追踪分支（`origin/main` 等），**不自动合并到工作分支**。
- `git pull`：等同于 `git fetch` + `git merge`（默认）或 `git fetch` + `git rebase`（配置 `pull.rebase true`）。

推荐使用 `git fetch` + 手动合并，或 `git pull --rebase`，以便更好地控制合并过程。

### Q8. 如何找回被 git reset --hard 删除的提交？

**答**：使用 `git reflog`，它记录了所有 HEAD 的变更：
```bash
git reflog
# 找到丢失的 commit hash
git reset --hard <lost-commit-hash>
# 或创建新分支
git branch recovery HEAD@{1}
```

reflog 默认保留90天（可通过 `gc.reflogExpire` 配置），所以只要不超过保留期，提交就能找回。

### Q9. 什么是 Git 的「三路合并」？

**答**：三路合并是 Git 的默认合并算法，涉及三个版本：
1. **Base**：两个分支的共同祖先
2. **Ours**：当前分支的最新版本
3. **Theirs**：传入分支的最新版本

Git 对每段代码进行比较：如果只有一方相对 base 有变化，自动选择有变化的那方；如果两方都相对 base 有变化，则产生冲突，需要手动解决。

### Q10. cherry-pick 和 merge 的区别？

**答**：
- `git merge`：将整个分支的所有提交合并。
- `git cherry-pick`：只应用**指定的某一个或几个**提交，不合并整个分支。

cherry-pick 常用场景：将 hotfix 分支上的修复提交同步到其他版本分支、只需要某分支上的个别功能而非全部。

### Q11. .gitignore 不生效怎么办？

**答**：`.gitignore` 只对**未被追踪**的文件有效，如果文件已经被 Git 追踪（即已有提交），需要先取消追踪：
```bash
# 取消追踪单个文件
git rm --cached <file>

# 取消追踪整个目录
git rm -r --cached <directory>

# 重新提交
git commit -m "chore: remove tracked files that should be ignored"
```

### Q12. 如何合并多个小提交为一个？

**答**：有两种方式：

方式一：`git rebase -i`（推荐）：
```bash
git rebase -i HEAD~4  # 整合最近4个提交
# 将需要合并的提交改为 squash 或 fixup
```

方式二：`git reset` + 重新提交：
```bash
git reset --soft HEAD~4  # 保留变更，撤销4个提交
git commit -m "feat: complete feature X"
```

### Q13. Git 中如何处理二进制文件？

**答**：Git 对二进制文件（图片、音视频、Office 文档等）的处理不如文本文件好：
- 无法做 diff 和 merge
- 每次修改都存储完整副本，仓库会快速膨胀

解决方案：
- 使用 **Git LFS**（Large File Storage）将大文件存储在外部，仓库只保存指针
- 小型二进制资源可以直接提交，但要注意仓库大小
- 频繁变更的大文件考虑使用对象存储（S3、OSS）+ 在代码中记录版本引用

### Q14. 什么是 Git Worktree？有什么用途？

**答**：`git worktree` 允许从同一个仓库检出多个工作目录，每个工作目录可以在不同的分支上工作：
```bash
# 为 hotfix 创建新工作区
git worktree add ../repo-hotfix hotfix/urgent-fix

# 查看所有工作区
git worktree list

# 删除工作区
git worktree remove ../repo-hotfix
```

用途：在不影响当前开发工作的情况下，同时在多个分支工作（比如处理紧急 hotfix），无需 stash 或切换分支。

### Q15. 如何统计每个人的代码贡献量？

**答**：
```bash
# 统计各作者的提交数
git shortlog -sn --all

# 统计代码行数变更（insertions/deletions）
git log --author="Zhang San" --pretty=tformat: --numstat | awk '{ add += $1; subs += $2; loc += $1 - $2 } END { printf "added lines: %s, removed lines: %s, total lines: %s\n", add, subs, loc }'

# 查看某时间段内的贡献
git log --after="2024-01-01" --before="2024-12-31" --shortlog
```

---

## 附录：常用 Git 别名配置

```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.cm "commit -m"
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --global alias.last "log -1 HEAD"
git config --global alias.unstage "restore --staged"
git config --global alias.undo "reset --soft HEAD~1"
git config --global alias.aliases "config --get-regexp alias"
git config --global alias.contributors "shortlog -sn --all"
```

## 附录：.gitconfig 完整示例

```ini
[user]
    name = Zhang San
    email = zhangsan@company.com

[core]
    editor = vim
    autocrlf = input      # Linux/Mac: input; Windows: true
    safecrlf = warn
    ignorecase = false

[init]
    defaultBranch = main

[pull]
    rebase = true         # git pull 默认使用 rebase

[push]
    default = current

[merge]
    conflictstyle = diff3  # 显示三路合并（含 base 版本）

[diff]
    algorithm = histogram  # 更好的 diff 算法

[alias]
    lg = log --oneline --graph --all --decorate
    st = status
    co = checkout
    undo = reset --soft HEAD~1

[color]
    ui = auto
```

---

*文档版本：v1.0 | 最后更新：2024年*

## Part 12: Git 进阶技巧与最佳实践

### 12.1 高级 Log 查询技巧

```bash
# 自定义 log 格式
git log --pretty=format:"%h %ad | %s%d [%an]" --date=short
# %h: 短 hash  %ad: 作者日期  %s: 主题  %d: 引用  %an: 作者名

# 查找某字符串第一次出现/消失的提交
git log -S 'password' --all  # 添加或删除了该字符串的提交
git log -G 'regex_pattern'   # 差异中匹配正则的提交

# 查看合并到 main 的分支列表
git branch --merged main
git branch -r --merged origin/main  # 远程已合并分支

# 统计各文件的修改频率（找到最脆弱的文件）
git log --name-only --format="" | sort | uniq -c | sort -rn | head -20

# 查看某目录的提交历史
git log --oneline -- src/main/java/com/example/

# 图形化显示含时间信息
git log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit
```

### 12.2 Git 配置进阶

```bash
# 条件配置（按目录使用不同配置，如公司项目用公司邮箱）
# ~/.gitconfig
```
```ini
[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work
[includeIf "gitdir:~/personal/"]
    path = ~/.gitconfig-personal

# ~/.gitconfig-work
[user]
    email = zhangsan@company.com

# ~/.gitconfig-personal
[user]
    email = zhangsan@gmail.com
```

```bash
# 配置 credential helper（缓存密码）
git config --global credential.helper cache          # Linux，默认15分钟
git config --global credential.helper 'cache --timeout=3600'
git config --global credential.helper store          # 永久保存（明文，不安全）
git config --global credential.helper manager-core  # Windows Credential Manager

# 配置代理
git config --global http.proxy http://proxy.company.com:8080
git config --global https.proxy https://proxy.company.com:8080
git config --global --unset http.proxy  # 取消代理

# 设置特定域名走代理
git config --global http.https://github.com.proxy socks5://127.0.0.1:1080
```

### 12.3 Git 签名提交

使用 GPG 签名提交，证明提交确实来自你：

```bash
# 生成 GPG 密钥
gpg --gen-key

# 列出 GPG 密钥
gpg --list-secret-keys --keyid-format=long

# 配置 Git 使用 GPG 签名
git config --global user.signingkey <GPG-KEY-ID>
git config --global commit.gpgsign true    # 所有提交自动签名
git config --global tag.gpgsign true       # 所有 tag 自动签名

# 手动签名提交
git commit -S -m "feat: signed commit"

# 验证签名
git log --show-signature
git verify-commit HEAD
```

### 12.4 Git Notes

Git Notes 允许为提交附加额外信息，而不修改提交本身：

```bash
# 为最新提交添加备注
git notes add -m "Code reviewed by Li Si on 2024-03-15"

# 为指定提交添加备注
git notes add -m "Reviewed" abc1234

# 查看备注
git log --show-notes
git notes show abc1234

# 推送/拉取备注（默认不推送）
git push origin refs/notes/*
git fetch origin refs/notes/*:refs/notes/*
```

### 12.5 GitHub CLI 常用操作

```bash
# 安装 GitHub CLI
# https://cli.github.com/

# 认证
gh auth login

# 仓库操作
gh repo create my-repo --public
gh repo clone owner/repo
gh repo fork owner/repo

# PR 操作
gh pr create --title "feat: new feature" --body "description"
gh pr list
gh pr view 123
gh pr checkout 123
gh pr merge 123 --squash
gh pr review 123 --approve

# Issue 操作
gh issue create --title "Bug: crash on login" --label bug
gh issue list --label bug
gh issue close 456

# 工作流操作
gh workflow run ci.yml
gh run list
gh run view 789
```

---

## Part 13: Git 工作流最佳实践总结

### 13.1 提交最佳实践

**1. 原子提交原则**
每个提交只做一件事，包含完整可运行的变更。不要把「重构+功能+修复」混在一个提交里。

```bash
# 错误示例
git commit -m "fix login bug, add register page, refactor user service"

# 正确示例
git commit -m "fix(auth): resolve NPE when user session expires"
git commit -m "feat(auth): add user registration page"
git commit -m "refactor(user): extract UserValidator to separate class"
```

**2. 提交消息写作规范**
- 第一行（主题行）：50字符以内，使用祈使语气，首字母大写，不加句号
- 空行分隔主题和正文
- 正文（body）：72字符换行，解释「为什么」而非「做了什么」
- Footer：关联 Issue（`Closes #123`）、破坏性变更（`BREAKING CHANGE:`）

**3. 何时提交**
- 完成一个逻辑单元（一个函数、一个接口、一个测试）
- 每天工作结束前
- 切换任务前
- 不要积累太多未提交的变更

### 13.2 分支管理最佳实践

**1. 分支生命周期管理**
```bash
# 定期清理已合并的远程分支
git fetch --prune

# 删除本地已合并的分支
git branch --merged main | grep -v 'main\|master\|develop' | xargs git branch -d

# 查看远程已删除但本地还存在的追踪分支
git remote prune origin --dry-run
```

**2. 保护重要分支**
- `main`/`master`：禁止直接 push，要求 PR + Review + CI 通过
- `develop`：可以直接 push 但建议走 PR
- `release/*`：只允许 hotfix 合并

**3. 功能分支粒度**
- 一个功能分支对应一个 Jira/Linear/GitHub Issue
- 生命周期不超过2-3天（超长分支是合并噩梦）
- 超过3天的功能，用 Feature Flag 拆分多个短期分支

### 13.3 团队协作规范

**代码评审**
- PR 大小：建议 < 400 行（不含测试和自动生成代码）
- 评审时间：提交后24小时内
- 最少2人 approve 才能合并（主干/发布分支）
- PR 合并后立即删除功能分支

**冲突预防**
- 每天早晨先拉取最新代码
- 功能分支每天 rebase/merge 最新主分支
- 大型重构提前通知团队，避免并行修改

**版本管理**
- 使用语义化版本（Semantic Versioning）：`MAJOR.MINOR.PATCH`
- 生产发布打 annotated tag
- CHANGELOG 自动生成

### 13.4 .gitignore 最佳实践

```
# Java 项目标准 .gitignore

# 编译产物
target/
*.class
*.jar
*.war
*.ear

# IDE 文件
.idea/
*.iml
.eclipse
.classpath
.project
.settings/
.vscode/
*.suo
*.ntvs*

# 操作系统文件
.DS_Store
Thumbs.db
desktop.ini

# 环境配置（包含敏感信息）
.env
.env.local
.env.production
application-local.yml
application-prod.yml

# 日志
*.log
logs/

# 依赖（Maven 本地仓库不需要提交）
.m2/

# 测试覆盖率报告
coverage/
htmlcov/
```

---

## Part 14: 常见问题排查

### 14.1 推送被拒绝怎么办？

```bash
# 错误：remote rejected - non-fast-forward
# 原因：远程有新提交，本地历史落后

# 解决方案1：先拉取再推送
git pull --rebase origin main
git push origin main

# 解决方案2（个人分支）：强推
git push --force-with-lease origin feature/my-branch

# 错误：remote rejected - hook declined
# 原因：服务端 hook 拒绝（如分支保护、代码不合规）
# 解决方案：查看错误信息，修复问题后重试
```

### 14.2 克隆速度很慢

```bash
# 方案1：浅克隆（只取最近提交）
git clone --depth 1 <url>

# 方案2：只克隆单一分支
git clone --single-branch --branch main <url>

# 方案3：配置代理
git config --global http.proxy socks5://127.0.0.1:1080

# 方案4：增大缓冲区
git config --global http.postBuffer 524288000
```

### 14.3 换行符问题（CRLF vs LF）

```bash
# Windows 开发者配置（自动转换）
git config --global core.autocrlf true
# 检出时 LF -> CRLF，提交时 CRLF -> LF

# Linux/Mac 开发者配置
git config --global core.autocrlf input
# 提交时 CRLF -> LF，检出时不转换

# 团队统一方案：.gitattributes
```
```
# .gitattributes
* text=auto eol=lf
*.bat text eol=crlf
*.sh text eol=lf
*.java text eol=lf
*.xml text eol=lf
*.properties text eol=lf
*.png binary
*.jpg binary
*.pdf binary
```

### 14.4 大文件导致仓库臃肿

```bash
# 查找大文件
git rev-list --objects --all |
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' |
  sed -n 's/^blob //p' |
  sort -k2 -rn |
  head -10

# 使用 git-filter-repo 删除大文件历史
git filter-repo --strip-blobs-bigger-than 10M

# 使用 BFG Repo Cleaner（更简单）
java -jar bfg.jar --strip-blobs-bigger-than 50M my-repo.git
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

### 14.5 误操作快速修复

```bash
# 误删分支
git reflog              # 找到分支最后的 commit
git branch <branch-name> <commit-hash>

# 误执行 git clean -fd（删除了未追踪文件）
# 无法通过 Git 恢复（Git 从未追踪这些文件）
# 只能从备份或 IDE 本地历史（如 JetBrains 的 Local History）恢复

# 提交消息写错了（未推送）
git commit --amend -m "correct message"

# 提交消息写错了（已推送，个人分支）
git commit --amend -m "correct message"
git push --force-with-lease

# 不小心提交了密码/密钥
# 1. 立即撤销密钥（API Key、密码等）
# 2. 使用 git filter-repo 清除历史
# 3. 强推所有分支
# 4. 通知所有团队成员重新 clone
```

---

## Part 15: Git 内部原理深入

### 15.1 Pack 文件

Git 的松散对象过多时会自动打包（pack），打包后存储在 `.git/objects/pack/` 下的 `.pack` 和 `.idx` 文件中。

```bash
# 手动触发打包
git gc

# 查看 pack 文件
git verify-pack -v .git/objects/pack/*.idx

# pack 文件使用 delta 压缩，相似对象之间只存储差异
# 这是 Git 仓库通常比想象中小的原因
```

### 15.2 引用规格（Refspec）

```bash
# 完整的 push/fetch 格式
git push origin <src>:<dst>
git fetch origin <src>:<dst>

# 示例：推送本地 feature 到远程 test-feature
git push origin feature/login:refs/heads/test-feature

# 删除远程分支（推送空引用）
git push origin :old-branch

# .git/config 中的 fetch refspec
# [remote "origin"]
#     fetch = +refs/heads/*:refs/remotes/origin/*
# 含义：将远程所有 heads 映射到本地 remotes/origin/
```

### 15.3 Git 数据传输协议

Git 支持多种传输协议：

| 协议 | URL 格式 | 说明 |
|------|---------|------|
| Local | `/path/to/repo` 或 `file:///path` | 本地文件系统 |
| SSH | `git@github.com:user/repo.git` | 安全，需要 SSH 密钥 |
| HTTP/HTTPS | `https://github.com/user/repo.git` | 防火墙友好，支持 token |
| Git | `git://github.com/user/repo.git` | 只读，无加密，速度快 |

**推荐**：使用 HTTPS + Personal Access Token 或 SSH Key，避免使用用户名密码。

### 15.4 Git 的快照模型

与 SVN 存储「差异」不同，Git 存储的是每次提交时所有文件的**快照**（通过 tree 和 blob 对象实现）。

- 相同内容的文件只存储一次（内容寻址），节省空间
- 获取任意版本的文件无需重建差异链，速度快
- 分支切换本质是更新 HEAD 指针，不是拷贝文件

```
提交快照示意：

commit A          commit B          commit C
  tree ──→ v1.txt   tree ──→ v1.txt   tree ──→ v1.txt (same blob)
           v2.txt            v2.txt'           v2.txt' (same blob)
           v3.txt            v3.txt            v3.txt'' (new blob)

# v1.txt 和 v2.txt(未修改)在多个提交中指向同一个 blob 对象，不重复存储
```

---

*文档完结 | Git 工作流详解 v2.0*

## Part 16: Git 与 IDE 集成

### 16.1 IntelliJ IDEA / JetBrains 系列

**常用操作**：
- `Alt+9` (Win) / `Cmd+9` (Mac)：打开 Git 工具窗口
- `Ctrl+K`：Commit 对话框
- `Ctrl+Shift+K`：Push 对话框
- `Ctrl+T`：Update Project（pull）
- 右键文件 → Git → Annotate：查看 blame

**Git Staging 模式**（推荐开启）：
`Settings → Version Control → Git → Enable staging area`

开启后可以精细控制哪些变更进入暂存区，类似命令行 `git add -p`。

**Shelf（搁置）**：
IDEA 提供的类 stash 功能，支持命名管理，界面更友好：
右键变更 → Shelve Changes

### 16.2 VS Code

**推荐插件**：
- **GitLens**：超强 Git 历史查看、blame 等
- **Git Graph**：图形化分支历史
- **GitHub Pull Requests**：在 VSCode 内管理 PR

**settings.json 推荐配置**：
```json
{
  "git.autofetch": true,
  "git.confirmSync": false,
  "git.enableSmartCommit": true,
  "gitlens.hovers.currentLine.over": "line",
  "gitlens.blame.format": "${author|10} ${agoOrDate|14-}"
}
```

### 16.3 命令行增强工具

```bash
# tig - 终端 Git 浏览器
# brew install tig (Mac) / apt install tig (Linux)
tig
tig blame <file>
tig log

# lazygit - 终端 Git TUI
# 支持所有常用操作，键盘驱动
lazygit

# delta - 更好看的 diff
# cargo install git-delta
# 配置：
```
```ini
[core]
    pager = delta
[delta]
    navigate = true
    light = false
    side-by-side = true
[interactive]
    diffFilter = delta --color-only
```

---

## Part 17: 企业级 Git 实践

### 17.1 大型团队的 Git 策略

**单一仓库（Monorepo）策略**：
- 适合：紧密耦合的微服务、共享代码库
- 工具：Nx、Turborepo、Bazel
- CI 优化：只构建变更影响的模块

```bash
# 使用 git sparse-checkout 减少检出范围
git sparse-checkout init --cone
git sparse-checkout set packages/my-service packages/shared-lib

# 使用 git worktree 并行工作
git worktree add ../service-a-fix origin/hotfix/service-a
```

**多仓库（Polyrepo）策略**：
- 适合：独立部署、不同技术栈的服务
- 挑战：跨仓库变更协调、依赖版本管理
- 工具：git submodule、git subtree、renovate（依赖更新）

### 17.2 Git 权限管理

**分支保护矩阵**：

| 角色 | feature/* | develop | release/* | main |
|------|-----------|---------|-----------|------|
| 开发者 | push/PR | PR only | 只读 | 只读 |
| Tech Lead | push/PR | push/PR | PR only | 只读 |
| 架构师 | push/PR | push/PR | push/PR | PR only |
| 发布负责人 | push/PR | push/PR | push/PR | PR + merge |

**GitHub Enterprise / GitLab 权限配置**：
```yaml
# GitLab 保护分支配置示例
protected_branches:
  - name: main
    push_access_level: 0        # No one
    merge_access_level: 40      # Maintainer
    code_owner_approval: true
  - name: develop
    push_access_level: 30       # Developer
    merge_access_level: 30
```

### 17.3 自动化发布流程

```yaml
# .github/workflows/auto-release.yml
name: Auto Release on Tag

on:
  push:
    tags: ['v*']

jobs:
  validate-tag:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Validate semantic version
      run: |
        TAG=${GITHUB_REF#refs/tags/}
        if ! [[ $TAG =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
          echo "Invalid tag format: $TAG"
          exit 1
        fi

  build-and-release:
    needs: validate-tag
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0   # 获取完整历史，用于 changelog

    - name: Generate Changelog
      id: changelog
      uses: TriPSs/conventional-changelog-action@v3
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        skip-commit: true

    - name: Create Release
      uses: actions/create-release@v1
      with:
        tag_name: ${{ github.ref }}
        release_name: ${{ github.ref }}
        body: ${{ steps.changelog.outputs.changelog }}
```

### 17.4 Git 审计与合规

**合规要求**：
- 所有变更可追溯（提交关联 Issue/Ticket）
- 关键文件变更必须经过特定人员 Review（CODEOWNERS）
- 签名提交（GPG签名）
- 禁止 force push（保护历史完整性）
- 定期备份仓库

```bash
# 备份裸仓库
git clone --bare https://github.com/org/repo.git repo-backup.git

# 更新备份
cd repo-backup.git
git fetch origin '+refs/heads/*:refs/heads/*' '+refs/tags/*:refs/tags/*'

# 验证仓库完整性
git fsck --full
```

---

## 速查表

### 紧急操作速查

```bash
# 撤销最近一次提交（保留代码）
git reset --soft HEAD~1

# 撤销最近一次提交（丢弃代码）
git reset --hard HEAD~1

# 找回刚丢失的 commit
git reflog; git reset --hard HEAD@{1}

# 保存当前工作并切换分支
git stash; git switch other-branch

# 解决推送被拒绝
git pull --rebase; git push

# 撤销公共分支上的错误提交
git revert <commit>; git push

# 把某提交移动到其他分支
git cherry-pick <commit>

# 查看谁写了这行代码
git blame -L 42,42 <file>

# 定位是哪次提交引入了 bug
git bisect start; git bisect bad; git bisect good <tag>

# 丢弃工作区所有修改
git restore .

# 清除所有未追踪文件
git clean -fd
```

### 常用命令速查表

| 场景 | 命令 |
|------|------|
| 初始化 | `git init` / `git clone <url>` |
| 暂存 | `git add .` / `git add -p` |
| 提交 | `git commit -m "msg"` |
| 状态 | `git status -s` |
| 日志 | `git log --oneline --graph` |
| 差异 | `git diff` / `git diff --staged` |
| 分支 | `git switch -c <branch>` |
| 合并 | `git merge --no-ff <branch>` |
| 变基 | `git rebase main` |
| 暂存工作 | `git stash push -m "msg"` |
| 标签 | `git tag -a v1.0 -m "msg"` |
| 远程 | `git push -u origin <branch>` |
| 回滚 | `git revert <commit>` |
| 找回 | `git reflog` |
| 清理 | `git clean -fd` / `git gc` |

---

*Git 工作流详解 - 完整版 | 涵盖基础原理到企业级实践*

## Part 18: Git 性能与存储优化

### 18.1 仓库性能诊断

```bash
# 查看仓库统计信息
git count-objects -vH
# size: 松散对象大小
# size-pack: 打包文件大小
# prune-packable: 可被垃圾回收的松散对象

# 查看 git 操作耗时
GIT_TRACE_PERFORMANCE=1 git status

# 查看网络传输详情
GIT_TRACE_PACKET=1 GIT_TRACE=1 git fetch

# 查看仓库中最大的对象
git verify-pack -v .git/objects/pack/*.idx | sort -k 3 -n | tail -10
```

### 18.2 Git 配置性能调优

```ini
[core]
    # 文件系统监听（减少 git status 耗时）
    fsmonitor = true        # Git 2.36+ 内置
    untrackedCache = true   # 缓存未追踪文件列表
    preloadindex = true     # 并行加载索引

[feature]
    manyFiles = true        # 大量文件时的优化模式（Git 2.27+）

[pack]
    threads = 0             # 自动检测 CPU 数量
    windowMemory = 256m     # pack 窗口内存限制

[index]
    version = 4             # 更高效的索引格式
```

### 18.3 Partial Clone（部分克隆）

Git 2.22+ 支持只克隆部分内容，显著减少克隆时间和磁盘用量：

```bash
# blobless clone：下载所有 commit 和 tree，但延迟下载 blob
git clone --filter=blob:none <url>

# treeless clone：只下载 commit，延迟下载 tree 和 blob
git clone --filter=tree:0 <url>

# sparse + partial（最小化）
git clone --filter=blob:none --sparse <url>
cd repo
git sparse-checkout set my-module/
```

---

## Part 19: 多平台 Git 配置

### 19.1 GitHub 特有功能

```bash
# 通过 PR 号获取 PR 分支（fetch PR branch）
# 在 .git/config 添加：
# fetch = +refs/pull/*/head:refs/remotes/origin/pr/*

git fetch origin
git checkout origin/pr/123    # 检出第123号 PR

# 或直接：
git fetch origin pull/123/head:pr-123
git switch pr-123

# GitHub Actions 中的 Git 配置
# actions/checkout@v4 默认浅克隆，需要完整历史时：
```
```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0        # 0 表示完整历史
    token: ${{ secrets.GITHUB_TOKEN }}
```

### 19.2 GitLab 特有配置

```bash
# GitLab CI 中提交回仓库
# 需要配置 CI/CD token

# .gitlab-ci.yml 中
```
```yaml
update-changelog:
  script:
    - git config user.email "ci@company.com"
    - git config user.name "CI Bot"
    - git remote set-url origin "https://oauth2:${CI_JOB_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git"
    - git add CHANGELOG.md
    - git commit -m "docs: update changelog [skip ci]"
    - git push origin HEAD:${CI_COMMIT_BRANCH}
```

### 19.3 Gitee（码云）配置

```bash
# 国内访问 GitHub 慢时，使用 Gitee 镜像
# 添加 Gitee 为额外远程仓库
git remote add gitee https://gitee.com/username/repo.git

# 同时推送到 GitHub 和 Gitee
git remote set-url --add origin https://gitee.com/username/repo.git
git push origin  # 同时推送两个远程

# 查看远程配置
git remote -v
```

---

## Part 20: Git 安全最佳实践

### 20.1 SSH 密钥管理

```bash
# 生成 Ed25519 密钥（推荐，比 RSA 更安全更短）
ssh-keygen -t ed25519 -C "your_email@example.com"

# 为不同平台使用不同密钥
# ~/.ssh/config
```
```
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github

Host gitlab.com
    HostName gitlab.com
    User git
    IdentityFile ~/.ssh/id_ed25519_gitlab

Host gitee.com
    HostName gitee.com
    User git
    IdentityFile ~/.ssh/id_ed25519_gitee
```

```bash
# 将公钥添加到 ssh-agent
eval $(ssh-agent -s)
ssh-add ~/.ssh/id_ed25519_github

# 测试连接
ssh -T git@github.com
```

### 20.2 Personal Access Token 管理

```bash
# 使用 git credential store（推荐使用系统钥匙串）
# macOS
git config --global credential.helper osxkeychain
# Windows
git config --global credential.helper manager-core
# Linux
git config --global credential.helper libsecret

# 针对特定 host 配置 token（CI 场景）
git config --global url."https://oauth2:<TOKEN>@github.com".insteadOf "https://github.com"
```

### 20.3 Secret 扫描

```bash
# 使用 git-secrets 防止提交密钥
# brew install git-secrets
git secrets --install  # 安装 hooks 到当前仓库
git secrets --register-aws  # 添加 AWS 密钥检测规则
git secrets --add 'PRIVATE_KEY'  # 自定义规则

# 使用 trufflehog 扫描历史
trufflehog git https://github.com/org/repo --only-verified

# pre-commit 框架集成 detect-secrets
# .pre-commit-config.yaml
```
```yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

---

## 附录 B：Git 命令分类速查

### 高频日常命令
```bash
git status              # 查看状态
git add .               # 暂存全部
git commit -m "..."     # 提交
git push                # 推送
git pull --rebase       # 拉取（变基方式）
git switch -c <branch>  # 创建并切换分支
git merge --no-ff <br>  # 合并分支
git log --oneline -10   # 查看最近10条日志
git diff                # 查看未暂存差异
git stash               # 暂时保存工作区
```

### 中频操作命令
```bash
git rebase -i HEAD~3    # 整理最近提交
git cherry-pick <hash>  # 应用单个提交
git tag -a v1.0 -m ""   # 打标签
git remote -v           # 查看远程
git fetch --prune       # 获取并清理
git reset --soft HEAD~1 # 撤销上次提交
git revert <hash>       # 安全撤销
git blame <file>        # 逐行追责
git show <hash>         # 查看提交详情
```

### 低频但重要的命令
```bash
git bisect start        # 二分法查 bug
git reflog              # 查看操作历史
git worktree add        # 多工作区
git filter-repo         # 重写历史
git submodule update    # 更新子模块
git lfs track "*.mp4"   # 大文件追踪
git gc --aggressive     # 深度清理
git fsck                # 仓库完整性检查
git bundle create       # 打包仓库
git format-patch        # 导出 patch
```

---

## 附录 C：延伸学习资源

| 资源 | 说明 |
|------|------|
| Pro Git Book | https://git-scm.com/book/zh/v2 — 官方免费中文书籍 |
| Git 原理详解 | `git help -a` — 所有命令列表 |
| Oh My Git! | 交互式游戏学 Git |
| Learn Git Branching | 可视化学习分支操作 |
| Conventional Commits | https://www.conventionalcommits.org |
| Git Flow 原文 | Vincent Driessen 的经典博文 |
| GitHub Docs | https://docs.github.com |
| GitLab Docs | https://docs.gitlab.com |

---

*Git 工作流详解 - 企业版完整文档 | 版本 3.0 | 涵盖 Git 2.40+*

## Part 21: Git Flow 实战案例

### 21.1 电商平台版本迭代案例

以下是一个电商团队（10人）使用 Git Flow 进行版本管理的完整案例：

**项目背景**：电商平台，每月发布一个大版本，需要同时维护 2.x 和 3.x 两条线。

```bash
# Sprint 开始
git switch develop
git pull --rebase origin develop

# 开发者A：开发商品搜索优化
git switch -c feature/SHOP-1234-search-optimization
git commit -m "feat(search): add elasticsearch fuzzy match"
git commit -m "feat(search): add search result caching"
git commit -m "test(search): add unit tests for search service"
git push -u origin feature/SHOP-1234-search-optimization
# 创建 PR → Code Review → 合并

# 开发者B：开发购物车并发控制
git switch -c feature/SHOP-1235-cart-concurrency
git commit -m "feat(cart): add optimistic locking for cart updates"
git push -u origin feature/SHOP-1235-cart-concurrency

# Sprint 结束，创建 release 分支
git switch develop
git switch -c release/3.2.0

# release 阶段：只修 bug，不加新功能
git commit -m "fix(search): handle empty keyword edge case"
git commit -m "chore: bump version to 3.2.0"
git commit -m "docs: update API changelog"

# 发布
git switch main
git merge --no-ff release/3.2.0 -m "chore: release version 3.2.0"
git tag -a v3.2.0 -m "Release v3.2.0\n\n- Search optimization\n- Cart concurrency fix"
git push origin main --follow-tags

# 反向合并到 develop
git switch develop
git merge --no-ff release/3.2.0
git push origin develop
git branch -d release/3.2.0

# 生产出现紧急 bug
git switch -c hotfix/3.2.1-payment-timeout main
git commit -m "fix(payment): increase gateway timeout to 30s"
git commit -m "chore: bump version to 3.2.1"

git switch main
git merge --no-ff hotfix/3.2.1-payment-timeout
git tag -a v3.2.1 -m "Hotfix v3.2.1: payment timeout"
git push origin main --follow-tags

git switch develop
git merge --no-ff hotfix/3.2.1-payment-timeout
git push origin develop
git branch -d hotfix/3.2.1-payment-timeout
```

### 21.2 开源项目维护案例

**项目背景**：一个流行的 Java 库，同时维护 1.x（LTS）和 2.x（当前）版本。

```bash
# 仓库分支结构
# main        → 2.x 最新开发
# 1.x-support → 1.x LTS 维护分支

# 修复同时影响 1.x 和 2.x 的 bug
git switch -c fix/issue-789-null-safety main
git commit -m "fix: handle null value in config parser"
# 提 PR，合并到 main

# 将修复 back-port 到 1.x
git switch 1.x-support
git cherry-pick <fix-commit-hash>
git push origin 1.x-support
# 或：git cherry-pick --mainline 1 <merge-commit-hash>  (cherry-pick merge commit)

# 发布 1.x patch 版本
git tag -a v1.8.3 -m "Bugfix release v1.8.3"
git push origin --tags
```

### 21.3 GitHub Flow 持续部署案例

**项目背景**：SaaS Web 应用，每个 PR 合并后自动部署到生产。

```yaml
# .github/workflows/deploy.yml
name: Deploy on merge to main

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: npm ci && npm test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to production
        run: |
          docker build -t app:${{ github.sha }} .
          docker push registry/app:${{ github.sha }}
          kubectl set image deployment/app app=registry/app:${{ github.sha }}
      - name: Verify deployment
        run: kubectl rollout status deployment/app
```

---

## Part 22: 代码审查实战

### 22.1 高效 Code Review 流程

```
提交 PR
  │
  ├── 自动检查（CI/lint/test）通过？
  │       ├── 否 → 作者修复
  │       └── 是 → 继续
  │
  ├── CODEOWNERS 自动指派 Reviewer
  │
  ├── Reviewer 阅读代码（24小时内）
  │       ├── 有问题 → 评论（blocking / non-blocking 标注）
  │       └── 无问题 → Approve
  │
  ├── 作者回应评论，修改代码
  │
  ├── 获得所需 Approve 数量（如2个）
  │
  └── 合并（Squash/Merge/Rebase 视团队规范）
```

### 22.2 常见 Review 评论模板

```markdown
# Blocking 问题（必须修改）
[blocking] 这里存在并发安全问题，多线程下 `count` 可能被竞争修改，
建议使用 AtomicInteger 或加锁。

# Non-blocking 建议（可以改，可以不改）
[nit] 变量名 `data` 语义不清，建议改为 `userProfileList` 更具描述性。

# 提问（寻求解释）
[question] 这里为什么选择 HashMap 而不是 ConcurrentHashMap？
是因为这个方法只在单线程中调用吗？

# 正向反馈
[praise] 这个提前返回的处理很优雅，减少了嵌套层级！

# 建议讨论
[discuss] 这种方式可以工作，但也可以考虑用 Optional 包装来
避免 NPE 风险，不确定哪种更符合项目风格，可以讨论一下。
```

### 22.3 PR 大小控制策略

**按功能分解大 PR**：

```
原计划：实现完整的用户认证模块（500行）

拆分为：
PR1: feat(auth): add User entity and repository（80行）
PR2: feat(auth): add password hashing service（50行）
PR3: feat(auth): add login endpoint with JWT（120行）
PR4: feat(auth): add token refresh mechanism（80行）
PR5: feat(auth): add logout and token blacklist（70行）
PR6: test(auth): add integration tests for auth flow（100行）
```

好处：每个 PR 独立可审查，更容易发现问题，合并后可以快速验证。

---

*Git 工作流详解 - 最终完整版 | 适用于个人开发者到大型企业团队*
