# 使用Git管理代码仓库-那些概念

> *git概念*

1. 分布式仓库：本地仓库，远程仓库。他们相互独立。
2. 本地三个工作区域：工作目录，暂存区域，本地仓库。
3. 文件状态：已提交（committed），已修改（modified）和已暂存（staged）。
4. 暂存（add）：1.对每个文件计算“校验和”（SHA-1哈希串）；2.把每个文件快照保存到本地仓库（blob类型的对象）存储，并写库；3.将“校验和”加入暂存区列表信息（文件项）。
5. 提交（commit）：1.生成当前目录及列表信息的一个（tree）对象，并写库；2.保存一个提交（commit）对象到库，包含一个指向 tree 对象（暂存内容快照）、本次提交的作者等相关附属信息、指向该提交对象的父对象指针（上个提交对象，或空）。

> *分支管理那些事*

1. 分支（branch）：就是指针，指向了一个 commit 对象。并不断向前移动。
2. 合并（merge）：合并 commit 对象，也即只是列表信息（tree）的合并。此次的 commit 对象则有多个祖先（多个上个提交对象）。
3. 切换分支（checkout）：1.把HEAD 指针移回到指定分支；2.把工作目录中的文件换成了指定分支所指向的 commit（快照内容）。
4. 远程分支（remote branch）：本地存储的是对远程仓库中的分支的无法更改的 remote 索引。用 (远程仓库名)/(分支名) 来表示。
5. 远程分支操作（fetch/pull/push）：1. fetch 命令会更新 remote 索引，并创建一个 FETCH_HEAD 分支；2. checkout 远程分支，可以取到本地分支（跟踪分支 tracking branch）；3. merge 把远程分支的内容合并到当前分支；4. pull 会获取所有远程索引，并把它们的数据都合并到本地分支中来；5. push 将跟踪分支全部数据推送到服务器。