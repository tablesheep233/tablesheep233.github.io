---
layout:     note
title:      "git 小技巧"
subtitle:   " \"git 小技巧\""
author:     "tablesheep"
catalog: true
hide-in-nav: true
header-style: text
tags:
- note
- git
---



### 替换对应邮箱的提交人&邮箱信息😈

> 我真的只是换掉真名而已啦

```shell
git filter-branch --env-filter '
OLD_EMAIL="old email"
CORRECT_NAME="new name"
CORRECT_EMAIL="new email"
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

