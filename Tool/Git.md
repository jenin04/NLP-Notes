## Git配置

```
git config --global user.name "Your Name"
git config --global user.email "email@example.com"

ssh-keygen -t rsa -C "youremail@example.com"
```



## **初始化并创建本地版本库**

**git init** 将当前路径下的文件夹初始化本地仓库

 

## **本地版本库常规操作**

![img](..\img\Tool\git\001.jpg)

**git** **add** **<** **file>** 将文件添加到本地版本库的暂存区(stage)

**git commit -m     "<message>"** 会将暂存区的目录树会写到版本库（对象库）中，当前分支会做相应的更新，即当前分支最新指向的目录树就是提交时原暂存区的目录树。

**git reset HEAD** 会把暂存区的目录树会被重写，会被分支指向的目录树所替换，但是工作区不受影响。

**git reset --hard HEAD^** 暂存区回退到分支的上一个版本

**git reset --hard HEAD^^** 暂存区回退到分支的上上个版本

**git reset --hard HEAD~100** 暂存区回退到分支的前100个版本

**git reset --hard <commit_id的前几位>** 暂存区回退到分支的某个版本commit_id

**git rm -r --cached** **<****文件/文件夹名字(.     忽略全部文件)****>**  命令会直接从暂存区删除文件，工作区则不做改变。

**git checkout .**  或 **git     checkout -- <file>** 用暂存区全部或指定文件替换工作区的文件。这个操作很危险，会清除工作区中未添加到暂存区的改动。

**git checkout HEAD .**  或 **git  checkout HEAD <file>** 用当前分支HEAD指向的全部或部分文件替换暂存区和工作区中的文件，比较危险。

**git status** 查看当前仓库状态

**git diff** <source_branch>     <target_branch> 查看两个分支之间的差异

**git rm <file>** 删除工作区、暂存区或分支上的同一个文件,

**git log** 查看提交到日志

**git reflog**  查看命令使用日志

**git check-ignore -v** 文件名 查看忽略规则

 

## **分支操作**

**git branch**  查看分支

**git branch <name>**  创建分支

**git branch -a** 查看本地和远程仓库的所有分支

**git branch -b <name>**  创建并切换到新建的分支上

**git branch -d <name>** 删除本地分支

**git branch -D <name>** 强行删除分支（主要是针对没有合并过的分支）

**git branch -v** 查看所有分支的最后一次操作

**git branch -v** 查看当前分支

**git brabch -b** 分支名 origin/分支名 创建远程分支到本地

**git branch --merged** 查看别的分支和当前分支合并过的分支

**git branch --no-merged** 查看未与当前分支合并的分支

**git branch origin** 分支名 删除远处仓库分支



**git checkout <name>** 切换分支



**git merge <name>**  合并分支到当前分支上

**git merge --no-ff -m** '合并描述' 分支名 不使用Fast forward方式合并，采用这种方式合并可以看到合并记录

 

## **远程仓库常规操作**

### git clone

**git clone <SSH URL|HTTP URL]>** 将远程仓库克隆到本地

例：

$ git clone http[s]://example.com/path/to/repo.git/

$ git clone ssh://example.com/path/to/repo.git/

$ git clone [user@]example.com:path/to/repo.git/（SSH的另一种写法）

$ git clone git://example.com/path/to/repo.git/

（Git支持多种协议，默认的git://使用ssh，但也可以使用https等其他协议。使用https除了速度慢以外，还有个最大的麻烦是每次推送都必须输入口令，但是在某些只开放http端口的公司内部就无法使用ssh协议而只能用https。）

### git remote 

**git remote add <remote_name> <SSH URL or HTTP URL]>** 关联远程仓库

例：$ git remote add origin git@github.com:autoliuweijie/DeepLearning.git

（注意，如果用HTTP URL，以后每次push的时候都要输入github的帐号密码，如果用SSH URL，必须在远程库上添加本机的SSH公钥;）

**git remote -v** 查看远程库

**git remote remove [remote_name]** 删除远程仓库

### git push

**git push <remote_name> <branch_name>**  推送本地分支到远程仓库

例：`git push -u origin master`（第一次推送加上-u）

### git pull

**git pull** 用于从另一个存储库或本地分支获取并集成(整合)

### git fetch

**git fetch** 获取远程仓库中所有的分支到本地(不合并)

 

## **暂存操作**

**git stash** 暂存当前修改

**git stash apply** 恢复最近的一次暂存

**git stash pop** 恢复暂存并删除暂存记录

**git stash list** 查看暂存列表

**git stash drop** 暂存名(例：stash@{0}) 移除某次暂存

**git stash clear** 清除暂存

 

## **标签操作**

**git tag** 标签名 添加标签(默认对当前版本)

**git tag** 标签名 commit_id 对某一提交记录打标签

**git tag -a 标签名 -m** '描述' 创建新标签并增加备注

**git tag** 列出所有标签列表

**git show 标签名** 查看标签信息

**git tag -d 标签名** 删除本地标签

**git push origin 标签名** 推送标签到远程仓库

**git push origin --tags** 推送所有标签到远程仓库

**git push origin :refs/tags/标签名** 从远程仓库中删除标签

 

## **.gitignore语法**

[Git忽略提交规则 - .gitignore配置运维总结](https://www.cnblogs.com/kevingrace/p/5690241.html)

