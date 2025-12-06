# Git 教程

## 目录
- [Git简介](#git简介)
- [Git安装](#git安装)
- [Git基本配置](#git基本配置)
- [Git基本操作](#git基本操作)
- [Git分支管理](#git分支管理)
- [Git远程仓库](#git远程仓库)
- [Git常见问题及解决方案](#git常见问题及解决方案)

## Git简介

Git是一个开源的分布式版本控制系统，由Linus Torvalds于2005年创建。它可以有效、高速地处理从很小到非常大的项目版本管理。Git是Linus Torvalds为了帮助管理Linux内核开发而开发的一个开放源码的版本控制软件。

### Git的特点

1. **分布式**：每个开发者都拥有完整的代码仓库副本，包括完整的历史记录。
2. **高效性**：Git的操作速度非常快，尤其是在处理大型项目时。
3. **数据完整性**：Git使用SHA-1哈希算法确保代码的完整性。
4. **支持非线性开发**：Git支持成千上万个并行开发的分支。
5. **易于管理**：Git提供了强大的分支和合并工具。

### Git与其他版本控制系统的区别

与传统的集中式版本控制系统（如SVN）不同，Git是分布式的。这意味着每个开发者的计算机上都有一个完整的仓库副本，而不仅仅是最新版本的文件快照。这种设计使得Git在处理大型项目和团队协作时更加高效和灵活。

## Git安装

### Windows系统安装

1. 访问Git官方网站：https://git-scm.com/
2. 下载Windows版本的Git安装程序
3. 运行安装程序，按照向导完成安装
4. 安装完成后，在命令提示符或PowerShell中输入`git --version`验证安装

### macOS系统安装

在macOS上，有多种方式可以安装Git：

#### 使用Homebrew安装（推荐）

```bash
brew install git
```

#### 使用Xcode Command Line Tools安装

```bash
xcode-select --install
```

#### 下载安装包安装

1. 访问Git官方网站：https://git-scm.com/
2. 下载macOS版本的Git安装程序
3. 运行安装程序，按照向导完成安装

### Linux系统安装

#### Ubuntu/Debian系统

```bash
sudo apt update
sudo apt install git
```

#### CentOS/RHEL系统

```bash
sudo yum install git
```

#### Fedora系统

```bash
sudo dnf install git
```

### 验证安装

安装完成后，可以在终端中运行以下命令验证Git是否安装成功：

```bash
git --version
```

如果安装成功，将显示Git的版本号。

## Git基本配置

### 设置用户信息

Git需要知道你是谁，以便在提交代码时记录作者信息。使用以下命令设置你的用户名和邮箱地址：

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱@example.com"
```

### 查看配置信息

可以使用以下命令查看当前的Git配置：

```bash
git config --list
```

或者查看特定的配置项：

```bash
git config user.name
git config user.email
```

### 设置默认编辑器

Git在某些操作中会打开文本编辑器（如提交信息编辑），你可以设置默认的编辑器：

```bash
# 使用VS Code作为默认编辑器
git config --global core.editor "code --wait"

# 使用Vim作为默认编辑器
git config --global core.editor "vim"

# 使用Emacs作为默认编辑器
git config --global core.editor "emacs"
```

### 设置默认分支名称

Git 2.28版本以上可以设置默认的分支名称：

```bash
git config --global init.defaultBranch main
```

### 配置文件位置

Git有三个级别的配置文件：

1. **系统级**：`/etc/gitconfig`（系统范围）
2. **用户级**：`~/.gitconfig`或`~/.config/git/config`（当前用户）
3. **项目级**：`.git/config`（当前项目）

使用`--global`选项会修改用户级配置文件，不使用该选项则修改项目级配置文件。

## Git基本操作

### 初始化仓库

要将一个目录转换为Git仓库，使用以下命令：

```bash
git init
```

这会在当前目录下创建一个`.git`子目录，其中包含Git仓库的所有必要文件。

### 克隆仓库

要从远程仓库克隆一个现有仓库，使用以下命令：

```bash
git clone <仓库URL>
```

例如：

```bash
git clone https://github.com/username/repository.git
```

### 查看仓库状态

使用以下命令查看仓库的当前状态：

```bash
git status
```

这会显示哪些文件已修改、哪些文件已暂存、哪些文件尚未跟踪等信息。

### 添加文件到暂存区

要将文件添加到暂存区，使用以下命令：

```bash
# 添加单个文件
git add filename.txt

# 添加多个文件
git add file1.txt file2.txt

# 添加所有修改和新建的文件
git add .

# 添加所有修改和删除的文件（不包括新建的文件）
git add -u

# 添加所有文件（包括修改、新建和删除的文件）
git add -A
```

### 提交更改

将暂存区的更改提交到本地仓库：

```bash
git commit -m "提交信息"
```

提交信息应该简洁明了，描述本次提交的内容。

### 查看提交历史

使用以下命令查看提交历史：

```bash
# 查看详细提交历史
git log

# 查看简洁的提交历史（单行显示）
git log --oneline

# 查看最近N次提交
git log -n 5

# 查看特定文件的提交历史
git log filename.txt

# 以图形化方式查看分支历史
git log --graph --oneline --all
```

### 查看文件差异

使用以下命令查看文件差异：

```bash
# 查看工作区与暂存区的差异
git diff

# 查看暂存区与最新提交的差异
git diff --cached

# 查看工作区与最新提交的差异
git diff HEAD

# 查看两个提交之间的差异
git diff commit1 commit2
```

### 撤销操作

#### 撤销工作区的修改

```bash
# 撤销单个文件的修改
git checkout -- filename.txt

# 撤销所有文件的修改
git checkout -- .
```

#### 撤销暂存区的修改

```bash
# 将文件从暂存区移回工作区
git reset HEAD filename.txt

# 将所有文件从暂存区移回工作区
git reset HEAD .
```

#### 撤销提交

```bash
# 撤销最近一次提交，但保留更改在工作区
git reset --soft HEAD~1

# 撤销最近一次提交，并丢弃更改
git reset --hard HEAD~1

# 撤销最近一次提交，但保留更改在暂存区
git reset --mixed HEAD~1
```

#### 修改最近一次提交

```bash
# 修改最近一次提交的信息
git commit --amend

# 修改最近一次提交的信息和内容
git add .
git commit --amend
```

### 删除文件

从Git仓库中删除文件：

```bash
# 从工作区和暂存区删除文件
git rm filename.txt

# 仅从暂存区删除文件，保留在工作区
git rm --cached filename.txt
```

### 移动/重命名文件

移动或重命名文件：

```bash
git mv old_filename.txt new_filename.txt
```

### 忽略文件

创建`.gitignore`文件来指定Git应该忽略的文件或目录：

```bash
# 创建.gitignore文件
touch .gitignore
```

`.gitignore`文件示例：

```
# 忽略所有.class文件
*.class

# 忽略所有.log文件
*.log

# 忽略target目录
target/

# 忽略IDE配置文件
.idea/
.vscode/

# 忽略操作系统生成的文件
.DS_Store
Thumbs.db
```

### 查看文件内容

使用以下命令查看特定提交中的文件内容：

```bash
git show commit_id:filename.txt
```

### 比较分支

使用以下命令比较两个分支之间的差异：

```bash
git diff branch1 branch2
```

### 清理工作区

使用以下命令清理工作区中的未跟踪文件：

```bash
# 查看将被删除的文件
git clean -n

# 删除未跟踪的文件
git clean -f

# 删除未跟踪的文件和目录
git clean -fd

# 删除忽略规则中指定的文件
git clean -fX
```

## Git分支管理

### 创建分支

创建新分支：

```bash
# 创建新分支但不切换到该分支
git branch <分支名>

# 创建新分支并切换到该分支
git checkout -b <分支名>
```

### 查看分支

查看所有分支：

```bash
# 查看本地分支
git branch

# 查看所有分支（包括远程分支）
git branch -a

# 查看所有分支及其最后一次提交
git branch -v

# 查看所有分支及其最后一次提交，包括远程分支
git branch -av
```

### 切换分支

切换到指定分支：

```bash
git checkout <分支名>
```

### 删除分支

删除分支：

```bash
# 删除已合并的分支
git branch -d <分支名>

# 强制删除未合并的分支
git branch -D <分支名>
```

### 合并分支

将指定分支合并到当前分支：

```bash
git merge <分支名>
```

### 变基操作

将当前分支的提交变基到指定分支：

```bash
git rebase <分支名>
```

### 解决合并冲突

当合并或变基操作出现冲突时，需要手动解决冲突：

1. 打开冲突文件，编辑解决冲突
2. 使用`git add`标记冲突已解决
3. 继续合并或变基操作：

```bash
# 继续合并操作
git commit

# 继续变基操作
git rebase --continue
```

如果需要取消合并或变基操作：

```bash
# 取消合并操作
git merge --abort

# 取消变基操作
git rebase --abort
```

### 查看分支图

使用以下命令查看分支图：

```bash
git log --graph --oneline --all
```

### 比较分支

比较两个分支之间的差异：

```bash
git diff <分支1> <分支2>
```

### 查看分支包含的提交

查看特定分支包含的提交：

```bash
git log <分支名>
```

### 查看分支合并状态

查看分支是否已合并：

```bash
git branch --merged
git branch --no-merged
```

### 跟踪分支

设置本地分支跟踪远程分支：

```bash
git branch --set-upstream-to=<远程分支名> <本地分支名>
```

或者创建新分支并跟踪远程分支：

```bash
git checkout -b <本地分支名> <远程分支名>
```

### 重命名分支

重命名本地分支：

```bash
git branch -m <旧分支名> <新分支名>
```

## Git远程仓库

### 查看远程仓库

查看当前配置的远程仓库：

```bash
git remote -v
```

### 添加远程仓库

添加新的远程仓库：

```bash
git remote add <远程仓库名> <远程仓库URL>
```

例如：

```bash
git remote add origin https://github.com/username/repository.git
```

### 删除远程仓库

删除已配置的远程仓库：

```bash
git remote rm <远程仓库名>
```

### 重命名远程仓库

重命名已配置的远程仓库：

```bash
git remote rename <旧远程仓库名> <新远程仓库名>
```

### 从远程仓库获取更改

从远程仓库获取更改但不合并：

```bash
git fetch <远程仓库名>
```

### 从远程仓库拉取更改

从远程仓库获取更改并合并到当前分支：

```bash
git pull <远程仓库名> <分支名>
```

### 推送更改到远程仓库

将本地提交推送到远程仓库：

```bash
git push <远程仓库名> <分支名>
```

例如：

```bash
git push origin main
```

### 设置上游分支

设置本地分支的上游分支，以便简化推送和拉取操作：

```bash
git push --set-upstream <远程仓库名> <分支名>
```

或者：

```bash
git push -u <远程仓库名> <分支名>
```

### 克隆特定分支

克隆远程仓库的特定分支：

```bash
git clone -b <分支名> <远程仓库URL>
```

### 查看远程分支信息

查看远程仓库的分支信息：

```bash
git remote show <远程仓库名>
```

### 清理已删除的远程分支

清理本地已删除的远程分支引用：

```bash
git remote prune <远程仓库名>
```

### 标签操作

#### 创建标签

创建轻量级标签：

```bash
git tag <标签名>
```

创建带注释的标签：

```bash
git tag -a <标签名> -m "标签信息"
```

#### 查看标签

查看所有标签：

```bash
git tag
```

查看特定标签的详细信息：

```bash
git show <标签名>
```

#### 推送标签到远程仓库

推送单个标签到远程仓库：

```bash
git push <远程仓库名> <标签名>
```

推送所有标签到远程仓库：

```bash
git push <远程仓库名> --tags
```

#### 删除标签

删除本地标签：

```bash
git tag -d <标签名>
```

删除远程标签：

```bash
git push <远程仓库名> --delete <标签名>
```

### 子模块操作

#### 添加子模块

添加子模块到当前仓库：

```bash
git submodule add <子模块仓库URL> <子模块路径>
```

#### 初始化子模块

初始化子模块：

```bash
git submodule init
```

#### 更新子模块

更新子模块到最新提交：

```bash
git submodule update
```

或者递归更新子模块：

```bash
git submodule update --recursive
```

#### 克隆包含子模块的仓库

克隆包含子模块的仓库并初始化子模块：

```bash
git clone --recursive <仓库URL>
```

或者：

```bash
git clone <仓库URL>
cd <仓库名>
git submodule update --init --recursive
```

## Git常见问题及解决方案

### 问题1：提交后发现错误

**问题描述**：提交后发现代码有错误或提交信息不正确。

**解决方案**：

```bash
# 修改最近一次提交的信息
git commit --amend

# 修改最近一次提交的内容
# 1. 修改代码
# 2. 添加修改的文件
git add .
# 3. 修改提交
git commit --amend
```

### 问题2：误删文件

**问题描述**：误删了工作区中的文件，想要恢复。

**解决方案**：

```bash
# 如果文件已被Git跟踪
git checkout -- <文件名>

# 如果文件未被跟踪但已提交过
git checkout <提交ID> -- <文件名>
```

### 问题3：合并冲突

**问题描述**：合并分支时出现冲突。

**解决方案**：

1. 打开冲突文件，手动解决冲突
2. 使用`git add`标记冲突已解决
3. 完成合并：

```bash
git commit
```

或者取消合并：

```bash
git merge --abort
```

### 问题4：误提交敏感信息

**问题描述**：不小心提交了包含密码、密钥等敏感信息的文件。

**解决方案**：

```bash
# 如果提交尚未推送到远程仓库
# 1. 删除敏感文件
git rm --cached <敏感文件名>
# 2. 修改提交
git commit --amend

# 如果提交已推送到远程仓库
# 1. 使用git filter-branch重写历史
git filter-branch --force --index-filter 'git rm --cached --ignore-unmatch <敏感文件名>' --prune-empty --tag-name-filter cat -- --all
# 2. 强制推送
git push origin --force --all
git push origin --force --tags
```

### 问题5：仓库过大

**问题描述**：Git仓库变得过大，影响性能。

**解决方案**：

```bash
# 清理不必要的文件和历史
git gc --aggressive

# 使用BFG Repo-Cleaner工具清理大文件
# 1. 下载BFG工具：https://rtyley.github.io/bfg-repo-cleaner/
# 2. 运行清理命令
java -jar bfg.jar --strip-blobs-bigger-than 100M
# 3. 清理并推送
git reflog expire --expire=now --all && git gc --prune=now --aggressive
git push origin --force --all
git push origin --force --tags
```

### 问题6：无法推送分支

**问题描述**：推送分支时被拒绝，提示"non-fast-forward"。

**解决方案**：

```bash
# 先拉取远程更改
git pull origin <分支名>

# 如果有冲突，解决冲突后再次推送
git push origin <分支名>

# 或者使用变基方式
git pull --rebase origin <分支名>
git push origin <分支名>
```

### 问题7：误操作导致代码丢失

**问题描述**：误操作（如reset --hard）导致代码丢失。

**解决方案**：

```bash
# 查看操作历史
git reflog

# 找到丢失代码的提交ID
# 恢复到该提交
git reset --hard <提交ID>
```

### 问题8：分支管理混乱

**问题描述**：分支过多或命名不规范，导致管理困难。

**解决方案**：

1. 制定分支命名规范，例如：
   - 功能分支：`feature/功能名称`
   - 修复分支：`fix/问题描述`
   - 发布分支：`release/版本号`
   - 热修复分支：`hotfix/问题描述`

2. 定期清理已合并的分支：

```bash
# 查看已合并的分支
git branch --merged

# 删除已合并的分支
git branch -d <分支名>
```

3. 使用分支策略，如Git Flow或GitHub Flow。

### 问题9：子模块更新问题

**问题描述**：子模块无法正确更新或初始化。

**解决方案**：

```bash
# 初始化子模块
git submodule init

# 更新子模块
git submodule update

# 或者递归更新子模块
git submodule update --recursive

# 克隆包含子模块的仓库
git clone --recursive <仓库URL>
```

### 问题10：提交历史混乱

**问题描述**：提交历史混乱，包含不必要的提交或提交信息不规范。

**解决方案**：

```bash
# 交互式变基，整理最近N次提交
git rebase -i HEAD~N

# 或者整理从指定提交之后的所有提交
git rebase -i <提交ID>
```

在交互式变基界面中，可以：
- `pick`：保留提交
- `reword`：保留提交但修改提交信息
- `edit`：保留提交但暂停以便修改
- `squash`：将提交合并到前一个提交
- `fixup`：将提交合并到前一个提交，并丢弃提交信息
- `exec`：在提交上运行命令
- `drop`：删除提交

### 问题11：无法连接到远程仓库

**问题描述**：无法连接到远程仓库，提示认证失败或连接超时。

**解决方案**：

1. 检查网络连接
2. 检查远程仓库URL是否正确：

```bash
git remote -v
```

3. 如果使用HTTPS，检查凭证管理器是否正确配置
4. 如果使用SSH，检查SSH密钥是否正确配置：

```bash
ssh -T git@github.com
```

5. 如果使用代理，检查代理设置：

```bash
git config --global http.proxy
git config --global https.proxy
```

### 问题12：文件名大小写问题

**问题描述**：在大小写不敏感的系统上修改文件名大小写后，Git无法正确跟踪。

**解决方案**：

```bash
# 配置Git为大小写敏感
git config core.ignorecase false

# 重命名文件
git mv -f oldname.txt NEWNAME.txt
```

### 问题13：提交历史包含大文件

**问题描述**：提交历史中包含大文件，导致仓库过大。

**解决方案**：

```bash
# 使用BFG Repo-Cleaner工具清理大文件
# 1. 下载BFG工具：https://rtyley.github.io/bfg-repo-cleaner/
# 2. 运行清理命令
java -jar bfg.jar --strip-blobs-bigger-than 100M
# 3. 清理并推送
git reflog expire --expire=now --all && git gc --prune=now --aggressive
git push origin --force --all
git push origin --force --tags
```

### 问题14：无法合并特定提交

**问题描述**：只想合并特定提交，而不是整个分支。

**解决方案**：

```bash
# 使用cherry-pick合并特定提交
git cherry-pick <提交ID>

# 如果有冲突，解决冲突后继续
git add .
git cherry-pick --continue

# 或者取消cherry-pick
git cherry-pick --abort
```

### 问题15：无法撤销已推送的提交

**问题描述**：需要撤销已推送到远程仓库的提交。

**解决方案**：

```bash
# 创建一个撤销提交的新提交
git revert <提交ID>

# 或者使用reset强制推送（不推荐，会重写历史）
git reset --hard HEAD~1
git push origin --force
```

## Git高级技巧

### 使用Git Stash暂存工作

```bash
# 暂存当前工作
git stash

# 查看暂存列表
git stash list

# 恢复最近的暂存
git stash pop

# 恢复特定暂存
git stash pop stash@{n}

# 应用暂存但不删除
git stash apply stash@{n}

# 删除暂存
git stash drop stash@{n}

# 清除所有暂存
git stash clear
```

### 使用Git Bisect查找问题提交

```bash
# 开始二分查找
git bisect start

# 标记当前提交为有问题
git bisect bad

# 标记某个提交为无问题
git bisect good <提交ID>

# Git会自动切换到中间提交，测试后标记为good或bad
# 重复此过程直到找到问题提交

# 结束二分查找
git bisect reset
```

### 使用Git Alias创建命令别名

```bash
# 创建全局别名
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit

# 创建复杂别名
git config --global alias.lg "log --graph --oneline --all"
```

### 使用Git Hooks实现自动化

Git Hooks是在特定事件（如提交、推送等）发生时自动执行的脚本。它们位于`.git/hooks/`目录下。

常用Hooks包括：
- `pre-commit`：提交前执行
- `commit-msg`：提交信息编辑后执行
- `pre-push`：推送前执行

例如，创建一个pre-commit hook来检查代码风格：

```bash
#!/bin/sh
# .git/hooks/pre-commit

# 检查代码风格
npm run lint
if [ $? -ne 0 ]; then
    echo "代码风格检查失败，请修复后再提交"
    exit 1
fi
```

### 使用Git Worktree管理多个工作目录

```bash
# 创建新的工作目录
git worktree add <路径> <分支名>

# 列出所有工作目录
git worktree list

# 删除工作目录
git worktree remove <路径>
```

### 使用Git Subtree管理子项目

```bash
# 添加子项目
git subtree add --prefix=<子目录> <子项目仓库URL> <分支>

# 更新子项目
git subtree pull --prefix=<子目录> <子项目仓库URL> <分支>

# 推送子项目更改
git subtree push --prefix=<子目录> <子项目仓库URL> <分支>
```

### 使用Git LFS管理大文件

```bash
# 安装Git LFS
git lfs install

# 跟踪大文件
git lfs track "*.psd"
git lfs track "*.zip"

# 查看跟踪的文件类型
git lfs track

# 提交.gitattributes文件
git add .gitattributes
git commit -m "跟踪大文件"

# 正常使用Git操作
git add file.psd
git commit -m "添加PSD文件"
git push
```

## 总结

Git是一个功能强大的版本控制系统，掌握Git的基本操作和高级技巧可以大大提高开发效率。本教程涵盖了Git的基本概念、安装配置、基本操作、分支管理、远程仓库操作以及常见问题的解决方案。通过不断实践和探索，你将能够更加熟练地使用Git进行版本控制。