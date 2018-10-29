### 自定义命令别名

git所有自定义别名都存储在`.gitconfig`文件中，添加方式有两种：

* 编辑`.gitconfig`，添加如下内容

  ```
  [alias]
  	{别名} = {命令}
  ```

* 使用命令`git config --global alias.{别名} {命令}`

---

### 关联远程分支

本地分支名：test；远程分支名：remote-test

基于上面的条件，在推送时，如果直接使用`git push`，会提示使用`git push --set-upstream`来设置上游关联的分支

执行`git push --set-upstream origin remote-test`之后，再次`git push`，报错：`error: src refspec remote-test does not match any.`

这是因为在git全局配置中`push.default=simple`，表示推送到与**当前分支名相同**的远程分支上。如果没有，则拒绝推送。更改配置`git config --global push.default upstream`

*当然，在不设置分支关联的情况下，还是可以使用`git push origin test:remote-test`达到目的*

#### push.default

* nothing - 除非有明确的引用格式存在，否则不做任何推送并伴有错误输出
* current - 推送当前分支到远程或更新远程同名分支
* upstream - 推送当前分支到upstream设置的远程分支
* tracking - 废弃，等同于upstream
* simple - 推送当前分支到远程同名分支，若没有，则拒绝推送
* matching - 推送本地所有分支到远程同名分支

---

### 清理过期分支

`git remote show origin`查看所有分支的情况

`git remote prune origin`清理过期分支