+++
date = '2025-12-08T8:00:00+08:00'
draft = false
title = 'Git cherry-pick'
tags = ['Git']
+++

### Patch Application

补丁应用类似下面这样（Codex 使用的 OpenAI Patch 是不是借鉴的这个？）的描述文件变更的文本文件，通常包含：

- 哪些文件被修改
- 具体的行级变更
- 上下文信息

```shell
# 创建补丁
git diff > changes.patch          # 未暂存的修改
git diff --cached > changes.patch # 已暂存的修改
git format-patch HEAD~3           # 最近 3 次提交生成补丁

# 应用补丁
git apply changes.patch           # 直接应用，不创建提交
git am changes.patch              # 应用并创建提交（用于format-patch生成的）
```

命令如 `git diff`、`git stach` 和 `git rebase` 都使用 patch。

这种修改方法，如果产生冲突则需要手动处理。

### 3-Way Merge

三方合并则是一种智能的合并算法，使用三个版本来解决合并冲突：

```text
      A - B (feature分支)
     /
Base
     \
      C - D (main分支)
```

1. Base: 共同的祖先提交
2. Current: 当前分支的最新提交
3. Incoming: 要合并进来的分支的最新提交

合并过程：

```bash
git merge feature-branch
git pull
```

当同一行在两个分支都被修改时：

```
<<<<<<< HEAD (Current分支)
当前分支的内容
=======
要合并分支的内容
>>>>>>> feature-branch
```

像的 `git merge`、`git pull` 和 `git cherry-pick` 都使用的该算法

### Cherry-pick Guide

你可能认为 cherry-pick = `git show <commit> --patch` + `git apply`

但实际上，cherry-pick 使用三方合并机制，这解释了为什么它能在冲突时提供合并解决界面，而不仅仅是失败。

Git cherry-pick 用于将已有的提交应用到当前分支，它会提取指定提交引入的更改，然后创建一个包含这些修改的新提交。

- 选择性地应用特定提交（就像摘樱桃一样只挑选想要的）
- 在不同分支间移动提交
- 代码回滚或修复的重新应用
- 跨分支的补丁应用

基本用法

```shell
# 单个提交
git cherry-pick <commit-hash>

# 多个提交
git cherry-pick <commit1> <commit2> <comit3>

# 某个范围内的提交
git cherry-pick start-commit^..end-commit
```

常用选项

- `-e / --edit`: 在提交前编辑提交信息

  ```shell
  git cherry-pick -e <commit>
  ```

- `-n / --no-commit`: 应用更改但不提交

  ```shell
  git cherry-pick -n <commit1> <commit2>
  # 继续修改，然后提交
  git commit -m "Combined changes"
  ```

- `-x`: 在提交中添加来源信息

  ```shell
  git cherry-pick -x <commit>
  ```

- `-s / --signoff`: 在提交信息末尾添加 Signed-off-by 签名

  ```shell
  git cherry-pick -s <commit>
  ```

- `--ff`: 如果可能，执行快速前进合并

  ```
  git cherry-pick --ff <commit>
  ```

- `-m <parent-number>`: 处理合并提交时指定主分支

  ```shell
  # 对于合并提交，需要指定使用哪个父提交
  git cherry-pick -m 1 <merge-commit>
  ```
