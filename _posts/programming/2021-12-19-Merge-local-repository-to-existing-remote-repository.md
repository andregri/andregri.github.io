---
layout: single
title: Merging a local repository to an existing remote repository
tags: git
---

Very often happens that I start versioning some software locally and I push it remotely
later :octocat:. When I decide it is the moment to keep the code safe and sound on Github,
the procedure to merge a local-only repository to a fresh remote repository is not straight forward.

First we set the **remote** repository that we want to connect to
```
git remote add origin git@github.com:andregri/go-mux-routers.git
```

We download the main branch locally
```
git pull
```

If we print the commit graph, the local commit and the remote commit are disconnected.
Usually, commits belonging to the same branch or that stem from a branch are visually connected
with bars, dash and slash. This is not the case:
```
git log --all --graph

* commit 7bf303dcffd009abed812cf57277f4abb57c4f1f (HEAD -> master)
  Author: andregri
  Date:   Sun Dec 19 17:30:47 2021 +0100

      git init

* commit 94ee685f2a585a7a981fd9b6b7439e8ea447a415 (main, origin/main)
  Author: andregri
  Date:   Sun Dec 19 17:21:51 2021 +0100

      Initial commit

```

Now we **rebase** the local branch on top of the remote branch:
```
git rebase main
```

and the local branch is connected to the remote one:
```
git log --all --graph

* commit 7bf303dcffd009abed812cf57277f4abb57c4f1f (HEAD -> master)
| Author: andregri
| Date:   Sun Dec 19 17:30:47 2021 +0100
|
|     git init
|
* commit 94ee685f2a585a7a981fd9b6b7439e8ea447a415 (main, origin/main)
  Author: andregri
  Date:   Sun Dec 19 17:21:51 2021 +0100

      Initial commit
```

We **merge** the local branch `master` to the remote branch `main`
to move forward the `main` branch.
```
git checkout main
git merge master
```

Finally, we succeeded in aligning the local repository with the remote repository
and we just need to **push** everything:
```
git checkout main
git push
git branch -d master
```
