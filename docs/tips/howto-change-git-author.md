---
title: How to change git author name and email
description: how to change the author name and email for committed
tags: ["git"]
---

# How to change git author name and email

[Source](https://stackoverflow.com/questions/750172/how-to-change-the-author-and-committer-name-and-e-mail-of-multiple-commits-in-gi)

This answer uses git-filter-branch, for which the docs now give this warning:

>git filter-branch has a plethora of pitfalls that can produce non-obvious manglings of the intended history rewrite (and can leave you with little time to investigate such problems since it has such abysmal performance). These safety and performance issues cannot be backward compatibly fixed and as such, its use is not recommended. Please use an alternative history filtering tool such as git filter-repo. If you still need to use git filter-branch, please carefully read SAFETY (and PERFORMANCE) to learn about the land mines of filter-branch, and then vigilantly avoid as many of the hazards listed there as reasonably possible.

Specifically, you can fix all the wrong author names and emails for all branches and tags with this command (source: [GitHub help](https://help.github.com/articles/changing-author-info/)):

```shell
#!/bin/sh

git filter-branch --env-filter '
OLD_EMAIL="your-old-email@example.com"
CORRECT_NAME="Your Correct Name"
CORRECT_EMAIL="your-correct-email@example.com"
if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ]
then
    export GIT_COMMITTER_NAME="$CORRECT_NAME"
    export GIT_COMMITTER_EMAIL="$CORRECT_EMAIL"
fi
if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ]
then
    export GIT_AUTHOR_NAME="$CORRECT_NAME"
    export GIT_AUTHOR_EMAIL="$CORRECT_EMAIL"
fi
' --tag-name-filter cat -- --branches --tags
```

For using alternative history filtering tool [git filter-repo](https://github.com/newren/git-filter-repo/), you can first install it and construct a `git-mailmap` according to the format of [gitmailmap](https://htmlpreview.github.io/?https://raw.githubusercontent.com/newren/git-filter-repo/docs/html/gitmailmap.html).

```shell
Proper Name <proper@email.xx> Commit Name <commit@email.xx>
```

And then run filter-repo with the created mailmap:

```shell
git filter-repo --mailmap git-mailmap
```