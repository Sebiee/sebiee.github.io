---
title: Pushing all git refs from one remote to another
date: 2025-08-04 12:30:00 +/-0200
categories: [DevOps, Snippets]
tags: [git, remote, useful, code]     # TAG names should always be lowercase
---

Let's say you have a project that you want to migrate from an old gitlab instance to a new one. First, create a new project in your new instance, and then

```bash
git clone git@my-old-gitlab.com:myproject
cd myproject

git remote add new-origin git@my-new-gitlab.com:myproject

# Use git remove -v if you want to see all your configured remotes

# First, we need to delete the symbolic-ref called HEAD, since it's forbidden to push it to a remote repo. We'll add it afterwards
git symbolic-ref --delete refs/remotes/origin/HEAD

# Push all refs (tags also) from origin to new-origin
git push new-origin --tags "refs/remotes/origin/*:refs/heads/*"

# Remove the (now)old origin and rename the new one
git remote remove origin
git remote rename new-origin origin

git symbolic-ref refs/remotes/origin/HEAD refs/remotes/origin/master
```
