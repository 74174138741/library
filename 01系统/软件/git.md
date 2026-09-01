# Git

个人速查：先建立「四层存储」的心智模型，再按任务找命令。  
`main` 是发行版；日常改动走分支；commit 尽量小；已推送的历史用 `revert`，未推送的用 `reset`。

---

## 1. 心智模型

Git 不是「保存文件」，而是在四层之间搬运快照：

```
工作区          暂存区           本地仓库              远程
(working tree)  (index / staging)  (.git / commits)     (origin)
     │               │                  │                  │
     │  git add      │   git commit     │   git push       │
     ├──────────────►├─────────────────►├─────────────────►│
     │               │                  │                  │
     │               │   git checkout / switch             │
     │◄──────────────┴──────────────────┤                  │
     │                                  │   git fetch      │
     │                                  │◄─────────────────┤
     │               git pull = fetch + merge/rebase       │
     │◄─────────────────────────────────┴──────────────────┤
```

| 层 | 是什么 | 典型命令 |
|---|---|---|
| 工作区 | 磁盘上正在改的文件 | 编辑器、`git status` |
| 暂存区 | 下一次 commit 会带走的变更 | `git add`、`.gitignore` |
| 本地仓库 | 已提交的历史（分支、commit） | `git commit`、`git log`、`git merge` |
| 远程 | GitHub / GitLab 上的副本 | `git push` / `git pull` / `git fetch` |

切换分支后看到的，是**该分支最新 commit** 对应的工作区，不是「刚才没存盘的草稿」。没 commit 的改动，换分支时要么带着走，要么冲突被拦住。

---

## 2. 原则

1. **`main` / `master` 是发行版。**  
   只放可运行、可回滚的结果。不要把 debug 草稿、实验痕迹直接合进去。

2. **规范大于个人效率。**  
   单人无所谓；多人时个人捷径会拖垮团队。分支命名、commit 信息、谁能推 `main`，开工前先约定。

3. **两个最小化。**
   - **commit 粒度最小**：一次只做一件事，方便理解和回滚。
   - **merge 复杂度最小**：避免一次性大改；越早合、越小合，冲突越少。

4. **协作时尽早把「最小可用」合进主线。**  
   A 先交出能跑、无报错的一小块，B 基于最新主线继续。两人始终知道对方进度，错了也能回溯。前提是双方代码水平接近，否则主线会被半成品污染。

---

## 3. 身份与认证

账户以**远程平台**为单位：先有 GitHub / GitLab 账号，再在本机写身份，再用 SSH 或令牌让两边能通信。

### 本机身份

```bash
git config --global user.name "xxx"
git config --global user.email "xxx@163.com"
```

查看：

```bash
git config user.name
git config user.email
git config --list
```

`--global` 写用户级配置（`~/.gitconfig`）；不加则只对当前仓库生效。

### SSH 密钥

SSH 是加密通道。本机一对密钥，通常放在当前用户目录，按**机器 + 用户**存在，不是「账号自带」：

```
~/.ssh/
├── id_ed25519        # 私钥，不能泄露
└── id_ed25519.pub    # 公钥，可以公开、可上传到多个平台
```

生成并查看公钥：

```bash
ssh-keygen -t ed25519 -C "你的邮箱"
cat ~/.ssh/id_ed25519.pub
```

把公钥贴到 GitHub：Settings → SSH and GPG keys。验证：

```bash
ssh -T git@github.com
```

认证大意：服务器用你登记过的公钥，验证你能用对应私钥完成一次签名。私钥不出门，公钥可以随便贴。

### GitLab 个人令牌

HTTPS 场景常用 **Personal Access Token** 代替密码。令牌只显示一次，生成后立刻存好。

---

## 4. 仓库

### 本地新建

```bash
git init -b main
```

删掉 Git 元数据（工作区文件还在，历史没了）：

```bash
rm -rf .git
```

### 绑定远程

SSH（推荐，和上面的密钥配套）：

```bash
git remote add origin git@github.com:用户名/仓库名.git
```

HTTPS：

```bash
git remote add origin https://github.com/用户名/仓库名.git
```

查看已绑定的远程：

```bash
git remote -v
```

一个仓库可以有多个 remote；日常默认名叫 `origin`。

### 克隆

整库：

```bash
git clone git@github.com:用户名/仓库名.git
```

只要某个子目录（sparse checkout，省流量）：

```bash
git clone --filter=blob:none --no-checkout https://github.com/org/repo.git
cd repo
git sparse-checkout init --cone
git sparse-checkout set 只要的目录名
git checkout
```

---

## 5. 日常：暂存、提交、忽略

这三步都还在**本机**，没有上云。

```bash
git status                 # 工作区 / 暂存区现在怎样
git add .                  # 全部可跟踪改动进暂存区
git add a.cpp              # 只暂存某个文件
git commit -m "说明这次做了什么"
```

`.gitignore` 阻止匹配到的文件进入暂存区（`git add` 会跳过它们）。已被跟踪的文件，后加 ignore **不会**自动取消跟踪，需要另做 `git rm --cached`。

`git commit --no-verify` 会跳过 **pre-commit 等钩子**，不是「确认没问题就可以随便推」。钩子是在拦格式/测试时，只有你清楚自己在跳过什么再用。

---

## 6. 分支

分支是指向某次 commit 的指针。合并、切换，默认都作用在**本地仓库**。

```bash
git branch                 # 列出本地分支，当前分支前有 *
git branch <name>          # 基于当前 HEAD 建分支（不切换）
git switch <name>          # 切换（新写法；旧写法 git checkout <name>）
git switch -c <name>       # 创建并切换
```

删除（先切到别的分支）：

```bash
git branch -d <name>       # 已合并才删
git branch -D <name>       # 强制删，未合并的也会掉
```

合并：把「另一条分支的历史」合进**当前**分支，工作区随之更新。

```bash
git switch main
git merge <要合进来的分支>
```

想跟远程对齐，一般是 `fetch` + `merge`，或直接 `pull`，而不是「在本地 merge 远程文件夹」。

---

## 7. 和远程同步

### 推送

第一次把本地分支推上去，并记下上游（之后可直接 `git push` / `git pull`）：

```bash
git push -u origin dev
```

`-u` 只需要设一次。日常更常见的是指定远程和分支，不依赖上游：

```bash
git push origin <分支名>
git push                   # 已设置上游时
```

### 拉取

```bash
git fetch origin           # 只更新远程快照，不改工作区
git pull origin <分支名>   # fetch + 合并到当前分支
```

`fetch` 适合先看远程有什么再决定怎么合；`pull` 适合当前分支已跟踪上游、直接跟上。

本地相对远程多了 / 少了什么：

```bash
git fetch origin
git status
git log --oneline HEAD..origin/main    # 远程有、本地没有
git log --oneline origin/main..HEAD    # 本地有、远程没有
git diff origin/main                   # 内容差
```

---

## 8. 看状态与历史

```bash
git status
git rev-parse HEAD                     # 当前提交完整 hash
git log -1 --oneline HEAD              # 最新一条：短 hash + 说明
git log                                # 完整历史
git log --graph --oneline --decorate --all
git reflog                             # 本机 HEAD 动过哪些位置（含 reset、切分支）
```

图形界面（VS Code / Cursor 的 Git 面板、行内 diff）适合看**工作区相对某次 commit 改了什么**，比纯命令快。hash、reflog、远程领先/落后，仍用命令更准。

---

## 9. 回溯

先拿到目标 commit 的 hash：`git log`、`git reflog`，或编辑器的 Git 历史。

| 场景 | 用什么 | 效果 |
|---|---|---|
| 改错了，**还没 push** | `git reset` | 移动当前分支指针 |
| 已经 push，别人可能基于它开发 | `git revert` | 新增一次「反做」提交，历史仍在 |

```bash
git reset --soft <hash>    # 回退提交，改动留在暂存区
git reset --mixed <hash>   # 默认：回退提交，改动留在工作区
git reset --hard <hash>    # 提交、暂存、工作区全部回到该 hash（未提交的会丢）
```

`--hard` 在未推送的私人分支上好用；对已共享的 `main` 不要用，会改写别人已经拉走的历史。

```bash
git revert <hash>          # 生成新 commit，抵消那一次的改动
```

`revert` 保持历史线性，适合团队主线。

---

## 10. 命令对照

| 我想… | 命令 |
|---|---|
| 标明我是谁 | `git config --global user.name / user.email` |
| 测 SSH | `ssh -T git@github.com` |
| 新仓库 | `git init -b main` |
| 抄一份远程 | `git clone …` |
| 绑远程 | `git remote add origin …` |
| 看改了什么 | `git status` |
| 选进下一次提交 | `git add` |
| 留下快照 | `git commit -m "…"` |
| 建 / 切 / 删分支 | `git branch` / `git switch` / `git branch -d` |
| 合进当前分支 | `git merge` |
| 上传 | `git push -u origin <branch>` 或 `git push origin <branch>` |
| 只看远程、先不合 | `git fetch` |
| 跟上远程 | `git pull` |
| 未推送：回到某次提交 | `git reset` |
| 已推送：撤销某次提交 | `git revert` |
| 我刚才把 HEAD 弄哪去了 | `git reflog` |
