# 使用Git管理代码仓库-了解

> *git概述*
- 分布式版本控制系统（最先进没有之一）。
- 传说大神Linus（linux作者）花两周时间用C写的来管理Linux源码。
- GitHub为开源项目提供免费Git存储，国内有gitee（码云）、code.aliyun.com 。
- 自建私服，一般选用gitlab。
- 客户端也git bash，也可以安装TortoiseGit（小乌龟），还有更强大的SourceTree（独家一键支持Git Flow）。

> *主要特征*
- 可无中心节点，可以向任意库推送代码。
- git分支操作是核心，而分支只是指针和HEAD（移动指向而已）。
- 只关心文件数据变化，文件改变指针才变。
- 本地库和远程库无本质分别，所以commit只改变本地库，push才改变远程库。
- 工作流程基于分支进行，可无中心分支。
- 操作步骤比较多，命令比较多。

> *与SVN的区别*
- SVN集中式（所以我们常用来管理文档）。
- git关注文件，SVN关注文件内容。
- SVN的版本基于全局的一个有序版本号，记录差异变化。git哈希值做标识。
- SVN拉分支相当于copy，内容多时比较慢，git分支强大得多。
- git复杂需要学习成本，SVN操作要简单很多。
- SVN的权限管理比较丰富，git就那几个角色。
- git工作目录就是项目，只能整体操作，SVN可以任意目录。

# git主要的分支
> 主要分支
1. 主分支master，默认会创建的。
2. 开发分支develop。

# git 常用的几个命令
1. 更新仓库 stash -> fetch/pull
2. 提交内容 add -> commit -> push
3. 合并及解决冲突 merge
4. 切换分支 branch/checkout
5. 暂存 stash