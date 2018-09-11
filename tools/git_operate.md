# 使用Git管理代码仓库-最佳实践

> *最佳实践*

1. 能fetch不pull。pull会更新及合并一步完成，fetch仅更新，手动再merge，会更安全。
2. fetch/pull之前暂存 stash。暂存之后，本地会恢复到修改前，可以直接拉代码，再将stash pop出来，再解决冲突时就简单些了，失败了还可以试多次。
3. 及时同步远程的代码。
4. 尽量一次提交全本次任务。利于跟踪提交。
5. 使用私有分支提交，完成开发后，“merge request”到目标分支，删除私有分支。
6. 完善的提交备注。
7. 项目一定要加的文件：.gitignore（需要忽略的文件列表） 和 README.md（项目概要说明）。
8. 少出现 Merge commit xxx into xxx 之类的干扰信息。

> *环境常用配置*
1. SSH：git支持https和git两种传输协议，使用ssh密钥可不用每次输入密码。在~/.ssh里面生成的公钥添加到远程仓库。生成过程都是通用的过程，在此不解释。“git remote set-url origin xxx_branch”可以修改URL，可变更传输协议。
2. config文件：在~/.ssh只可以配置远程仓库参数(名为config)，如端口，用户名，公钥之类。


> config文件的内容：
```
Host github.com
HostName  github.com
User XXXXX
Port 22
IdentityFile ~/.ssh/id_rsa
```
> 用户名和邮箱：git提交代码时显示的。
```
$ git config --global user.name "youname"
$ git config --global user.email "you@mail"
--global全局生效，不加则只此项目生效（需要在项目根目录下）。
```
