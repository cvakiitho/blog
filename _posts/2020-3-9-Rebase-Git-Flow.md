---
layout: post
title: "Rebase Git Flow"
categories: [programming]
tags: [ programming, github, git, gitflow, rebase]
---

Git flow used on cloud projects with fast releases, and only one deployed version.
Variant of git-flow, putting emphasis on clean git history. Two infinite branches, develop+master, and three short-lived branch types: hotfix*, release*, feature*.
This workflow is mapped to our deploy spaces in following way: 

master branch → canary space deployed after promoting - completely identical to production
develop branch → staging space for testing
hotfix* && release* branch → release space


Hotfix and release can be on same space, as we very very rarely have a situation with active hotfix and release branch.


All merges between all 4 named branches should be done by rebase, to evade non-feature merge commits polluting our git history, and messing with further syncs of master - develop.

All branches need to be up to date before merging. Conflicts needs to be solved in non-master branches via `git pull --rebase`

## Overview:

### Examples:

#### New Feature:

Create new branch from latest develop branch

```
  git fetch origin
  git checkout develop
  git checkout -b feature/my_awesome_feature
```

Do some commits, and keep your branch updated with latest changes from develop branch:

```
  git add -u; git commit -m "My great commit"
  git pull origin develop --rebase
  git push -u origin feature/my_awesome_feature
```

Due to possible commit order getting updated, you might need to force push, I recommend to use `--force-with-lease` to verify that upstream branch is in state that you expect, nobody pushed anything, and you fetched latest changes.

```
  git push origin feature/my_awesome_feature --force-with-lease
``` 
After your feature branch is ready, create Pull request, rebase/squash/merge here is up to developer, 

I suggest to merge larger features, so they can be reverted easily via merge commit revert, 

Squash if you didn't clean your feature commit history, and Rebase for small things.



#### Hotfix
Create hotfix branch from master branch:

```
  git fetch origin
  git checkout master
  git checkout -b hotfix-1
```
Do your fix, and push hotfix branch back to origin.(keep your branch up to date)

```
  git add -u; git commit -m "My great commit"
  git pull origin master --rebase
  git push -u origin hotfix-1
```

After push to hotfix*, deployment should be done to release space, after QA you can proceed with PR to master
Merge to master is done with rebase.

![Rebase And Merge](/public/rebase_and_merge.png)



#### Release
Create release branch from develop branch, and push to origin:

```
  git fetch origin
  git checkout develop
  git checkout -b release-1
  git push origin release-1
```


Wait for release space to get updated, do final smoke tests, optionally push extra commits.

After QA verification, you can proceed with PR to master

Merge to master is done with rebase.

![Rebase And Merge](/public/rebase_and_merge.png)


#### Master → Develop sync

Due to translations/security fixes/hotfixes, you occasionally have to sync master back to develop.

This updates commit history, and so it has to be force pushed back to develop.

```
  git fetch origin
  git checkout develop
  git pull origin master --rebase
  git push origin develop --force-with-lease
```

All Devs have to update their local develop branches:

```
  git fetch origin
  git checkout develop
  git reset --hard origin/develop
```


### Final Notes:

- All Merging between named branches is done by rebase, to eliminate merge commits, that are impossible to keep in sync between branches. 
- Feature branches can be merged by any means necessary.
- Master branch is the only branch protected against force-push.
- Force pushing is done with `--force-with-lease`

#### Pros:
- Clean history, nothing gets duplicated

#### Cons:
- Majority of this flow can't be easily required in github, therefore it is dependent on developers discipline 

Linear History can be enforced by branch protection rules in public github:

https://help.github.com/en/github/administering-a-repository/requiring-a-linear-commit-history

Not available in enterprise github.