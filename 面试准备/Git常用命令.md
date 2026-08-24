```shell
git log --oneline --graph -n 5
```



Galaxy 前端项目强制提交

```shell
# 添加修改的文件
git add .

# 跳过pre-commit和commit-msg hooks强制提交
git commit -m "提交信息" --no-verify

# 或者使用更安全的强制推送（保留其他人的提交）
git push origin <分支名> --force-with-lease
```

回滚代码

```shell
# 暴力回退两个版本，这条命令只修改本地代码
git reset --hard HEAD~2
# 强制提交到远程仓库
git push --force-with-lease

# 安全回滚
# 回滚两个版本，会逐个回退，每个版本会出现vim，需要确认退出
git revert HEAD~2..HEAD
git revert HEAD~1 HEAD
git push

# 回滚后后悔想撤回回滚
git reflog --date=local
# reset到想要的commit id
git reset --hard 789abcd
git push --force-with-lease
```

合并分支

```bash
git checkout <合并后分支>
git merge <被合并分支>
git commit
git push

git merge --no-ff galaxy/zsc_limit_create_tt_quantity
```

合并多个提交记录

```bash
# 查看近20次的git日志，列举commit id和message
git log --oneline -20

# 这样会把 xxx 之后的所有提交都“收回”为暂存区改动，然后你重新提交一次。
git reset --soft xxx
git commit -m "..."
git push --force-with-lease
```

​    async_task_pending_queue_key = 'deeptest:eval:queue:{full_pipeline}:pending'

​    async_task_running_queue_key = 'deeptest:eval:queue:{full_pipeline}:running'

​    eval_task_pending_queue_key = 'deeptest:eval:task_queue:{eval_task}:pending'

​    eval_task_running_queue_key = 'deeptest:eval:task_queue:{eval_task}:running'
