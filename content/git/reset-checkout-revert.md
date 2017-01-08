+++
date = "2017-01-10T17:28:53+08:00"
draft = false
title = "reset-checkout-revert"
thumbnail = "images/git/hero.svg"

categories = ["git"]

+++

# Reset, Checkout and Revert

`git reset`、`git checkout`和`git revert`是git工具集中最常用的一些命令。它们都允许你对你的仓库undo某类改动，`git reset`和`git checkout`可作用于commits或files，revert只能作用于commits。

由于它们的功能非常的相似，在开发过程中很难区分哪个命令应该用于哪个开发场景。在这篇文章中我们将会比较这三个命令。希望你能在你的仓库中熟练地使用这些命令。

{{% img src="images/git/01.svg" %}}

当你浏览这篇文章时，把每个命令的作用效果与git仓库的三个概念（工作目录，暂存区，提交历史）结合起来会对你很有帮助。

# Commit级操作
`git reset`和`git checkout`根据传入的参数来决定其作用的范围。当参数中不含有文件路径时，它们作用于整个commits。这正是我们这个部分所要讨论的。注意，`git revert`不作用于files。

### Reset
当作用于commit时，reset可以用于改变分支指针的指向。这可以被用来删除当前分支的commits。例如，下面的命令将hotfix分支向后移动两个commits。

    git checkout hotfix
    git reset HEAD~2

hostfix分支尾部的两个commits现在处于悬挂状态，这意味着在下次git执行垃圾回收时这两个commits会被删除。换句话说，你在告诉git正在抛弃这两个commits。请看下面图示：

{{% img src="images/git/02.svg" %}}

`git reset`的这种用法常用于undo还没有被共享的改动。当你正添加一个新的feature且突然想重新开始时，这是你的goto命令。

除了在当前分支来回移动，通过命令行选项`git reset`也可用来改动暂存区和工作目录：
* --soft  -- 不改动暂存区和工作目录
* --mixed -- 更新暂存区到特定的commit，但是不改动工作目录。这是默认行为。
* --hard  -- 更新暂存区和工作目录到特定commit

请看下面图示：

{{% img src="images/git/03.svg" %}}

这些选项常被用于HEAD commit。例如，`git reset --mixed HEAD`将暂存区的改动移回工作目录。另外，如果你想完全抛弃所有未提交的改动，可以使用`git reset --hard HEAD`。这两个场景也是最常用的。

当`git reset`作用于非HEAD commit时一定要格外小心，因为它变更了commit history。

### Checkout
现在你应该很熟悉`git commit`作用于commits了。当作用于分支名时，它允许你在在分支间进行切换。

    git checkout hotfix

上面的命令将HEAD指向另一个分支并更新工作目录到相应分支。由于这个命令有可能覆盖本地的变更，git会要求我们commit或stash工作目录可能会被覆盖的变更。`git reset`与`git checkout`不切换分支。

{{% img src="images/git/04.svg" %}}

如果作用于commits，`git checkout`可以指向任意的commit。这和切换分支类似：将HEAD指向特定的commit。例如：下面的命令将HEAD指向当前commit的祖父commit。

    git checkout HEAD~2

{{% img src="images/git/05.svg" %}}

 当查看工程的历史版本时非常有用。然而，由于没有分支引用当前HEAD，这使你处于detached HEAD state。如果你添加了一些新的commits那就非常危险了，国为当你切换到其它分支后，再没有办法回到刚在的位置了。基于这个原因，你应该在提交commits到detached HEAD这前创建一个新的分支。

### Revert
`git revert`通过添加一个新的commit来undo commits。由于这不改变commit history，所以这是undo变更的安全方法。例如，下面的命令将计算出倒数第三次commit，创建一个新的commit来undo那次的改变。

    git checkout hotfix
    git revert HEAD~2

请看下面图示：

{{% img src="images/git/06.svg" %}}

与`git reset`相比，`git reset`会改写已存在的commit history。由于这个原因，`git revert`应该在公共分支上来进行undo，`git reset`应该在私有分支上进行undo。

你也可以认为`git revert`用来undo已提交的变更，而`git reset`用来undo未提交的变更。

像`git checkout`一样，`git revert`有可能会覆盖工作目录中的文件，因此它将会询问你是否commit或stash在本次操作中可能会丢失的变更。

# File级操作
`git reset`和`git checkout`可以接收可选的文件路径作为参数。这戏剧性的改变了它们的行为，使它们不再作用于整个commit，而是强制它们的作用于单个文件。

### Reset
当使用文件路径作为`git reset`的参数时，`git rest`更新暂存区到特定的commit。例如，下面的命令将获取foo.py的祖父版本并将其更新到暂存区以备下次提交。

    git reset HEAD~2 foo.py

和`git reset`的Commit级操作一样，`git reset`通常和HEAD一起使用。`git reset HEAD foo.py`将unstage foo.py。

{{% img src="images/git/07.svg" %}}

`--soft`、`--mixed`和`--hard`选项对File级的`git reset`不起作用，也就是说，`git reset`只更新暂存区，而从不更新工作目录。

### Checkout
当使用文件路径作为`git checkout`的参数时，`git checkout`只更新工作目录。和该命令Commit级操作不同的是，`git checkout`不会改变HEAD的指向，这意味着它不会切换当前分支。

{{% img src="images/git/08.svg" %}}

例如：下面的命令更新工作目录的foo.py为它的祖父版本。

    git checkout HEAD~2 foo.py

和`git checkout`的Commit级操作一样，`git checkout`可以用来查看工程的历史版本，只不过只能查看特定的文件了。

当你暂存并且提交检出的文件时，`git checkout`的效果和`git revert`非常相似。但是`git checkout`撤销了检出版本之后的所有变更，而`git revert`只撤销了特定版本的变更。

和`git reset`一样，`git checkout`通常和HEAD一起使用。例如，`git checkout HEAD foo.py`丢弃对foo.py所有未提交的改动。这和`git reset HEAD --hard`非常相似，但是它只作用于特定的文件。

# Summary
`git reset`、`git checkout`和`git revert`很容易混淆，但是当你将它们的作用效果与工作目录、暂存区和提交历史相结合时，就会很容易区分它们各自的使用场景了。

下面对这些命令最常用的场景进行了总结：

* `git reset`：**Commit级**：丢弃对私有分支的提交或丢弃未提交的改动
* `git reset`：**File级**：撤销暂存区中的文件
* `git checkout`：**Commit级**：切换分支或查看历史版本
* `git checkout`：**File级**：丢弃对工作目录的改动
* `git revert`：**Commit级**：在公开分支上进行撤消
* `git revert`：**File级**：不适用

# 原英文链接
[resetting-checking-out-and-reverting](https://www.atlassian.com/git/tutorials/resetting-checking-out-and-reverting)

