---
title: create git hooks in mac
date: 2018-12-15 11:49:19
tags: 
    - git
category:
    - git
---
This article explains how to create git hooks in mac, and how to use the customized git hook chains.

## Preparation for creating hook chain
(This part refers to the [hook-chain](https://stackoverflow.com/questions/8730514/chaining-git-hooks/8734391#8734391), and make some modification for running in mac.)</br>

Motivation is that when you run git commit, the pre-commit script will be invoked before commit operation actually happen. If you want to do multiple checks in the pre-commit phrase, instead of putting all logic in one file of pre-commit, a better idea is separating the check logic in separated files, and trigger each check one by one as a chain.

- Create git templates hooks folder
```bash
mkdir -p ~/.git-templates/hooks
cd ~/.git-templates/hooks
```
  All files in this folder will be copied to each git project's hook folder when running git init. 

- Create hook-chain file under the hooks folder

  file name : hook-chain
```bash
#!/bin/bash

hookname=`basename $0`
git_folder=`dirname $0`
for hook in $(ls $git_folder/$hookname.*)
do
    if [ -f $hook ]
    then
        echo "executing hook : $hook"
        sh "$hook"
        status=$?

        if test $status -ne 0; then
            echo "Hook $hook failed with error code $status"
            exit $status
        fi
    fi
done
```
  What this do is executing all hook files whose name begin with the current git hook. For example, when invoking git hook of pre-commit, this will invoke all scripts starting with pre-commit, such as pre-commit.json, pre-commit.yaml, etc.

- Create hook files

  Refer to [git hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) for all client side git hooks and their meaning. For example, pre-rebase, pre-commit, post-checkout, post-merge, pre-push, etc. Create the ones that you need under ~/.git-templates/hooks.
  
  In my case, I want to create pre-commit hook, so I run:
```
ln -s hook-chain pre-commit
```
  Then I will also create pre-commit.json, pre-commit.yaml, pre-commit.awskey in the same folder as explained in the following sessions. When pre-commit is invoked, it will trigger all these three files of pre-commit.json, pre-commit.yaml and pre-commit.awskey one by one.

  Note that each hook file should be executable.
```
chmod +x ~/.git-templates/hooks/.
```
## Write customized hooks
### Prevent commit the aws access key and secret key
(script is [from](https://gist.github.com/saliceti/7eb0ba0bb5ed875df515))

- file name : pre-commit.awskey
```bash
#!/usr/bin/env bash

# Install globally using https://coderwall.com/p/jp7d5q/create-a-global-git-commit-hook		
# The checks are simple and can give false positives. Amend the hook in the specific repository.

if git rev-parse --verify HEAD >/dev/null 2>&1
then
    against=HEAD
else
    # Initial commit: diff against an empty tree object
    EMPTY_TREE=$(git hash-object -t tree /dev/null)
    against=$EMPTY_TREE
fi

# Redirect output to stderr.
exec 1>&2
 
# Check changed files for an AWS keys
FILES=$(git diff --cached --name-only $against)

if [ -n "$FILES" ]; then
    KEY_ID=$(grep -rE --line-number '(^|[^A-Za-z0-9/+=])AKIA[A-Z0-9]{16}($|[^A-Za-z0-9/+=])' $FILES)
    KEY=$(grep -rE --line-number '(^|[^A-Za-z0-9/+=])[A-Za-z0-9/+=]{40}($|[^A-Za-z0-9/+=])' $FILES)

    if [ -n "$KEY_ID" ] || [ -n "$KEY" ]; then
        exec < /dev/tty # Capture input
        echo "=========== Possible AWS Access Key IDs ==========="
        echo "${KEY_ID}"
        echo ""

        echo "=========== Possible AWS Secret Access Keys ==========="
        echo "${KEY}"
        echo ""

        while true; do
            read -p "[AWS Key Check] Possible AWS keys found. Commit files anyway? (y/N) " yn
            if [ "$yn" = "" ]; then
                yn='N'
            fi
            case $yn in
                [Yy] ) exit 0;;
                [Nn] ) exit 1;;
                * ) echo "Please answer y or n for yes or no.";;
            esac
        done
        exec <&- # Release input
    fi
fi

# Normal exit
exit 0
```
### Prevent commiting invalid json files
- file name : pre-commit.json
```bash
#!/bin/bash
git_dir=$(git rev-parse --show-toplevel)

for file in $(git diff-index --name-only --diff-filter=ACM --cached HEAD -- \
    | grep -E '\.((js)|(json))$'); do
    python -mjson.tool $file 2> /dev/null
    if [ $? -ne 0 ] ; then
        read -p "Find unbroken json in $git_dir/$file, is that what you intended? [y|n] " -n 1 -r < /dev/tty
        echo
        if echo $REPLY | grep -E '^[Yy]$' > /dev/null
        then
            exit 0
        fi
        exit 1
    fi
done
```
### Prevent commiting yaml files with quote in key
Note that standard yaml allows containing quote in keys, but that is probably not something we intented to do.
- file name : pre-commit.yaml
```bash
#!/bin/bash
git_dir=$(git rev-parse --show-toplevel)

for file in $(git diff-index --name-only --diff-filter=ACM --cached HEAD -- \
    | grep -E '\.((yaml)|(yml))$'); do
    python3 "${git_dir}/.git/hooks/is_yaml_key_contains_quote.py" $file
    if [ $? -ne 0 ] ; then
        read -p "Find quote in key in $git_dir/$file, is that what you intended? [y|n] " -n 1 -r < /dev/tty
        echo
        if echo $REPLY | grep -E '^[Yy]$' > /dev/null
        then
            exit 0
        fi
        exit 1
    fi
done
```
- file name : is_yaml_key_contains_quote.py
```python
#!/usr/bin/env python3
import sys
import yaml
import json


def find_quote_in_key(json_obj):
    if not json_obj:
        return
    if type(json_obj) is list:
        for item in json_obj:
            find_quote_in_key(item)
    if type(json_obj) is dict:
        for key, value in json_obj.items():
            if "\"" in key or "'" in key:
                print("quote is in : {}".format(key))
                global is_quote_found
                is_quote_found = True
            find_quote_in_key(value)


file_path = sys.argv[1]
is_quote_found = False
yaml_content = ""
with open(file_path, 'r') as stream:
    yaml_content = yaml.load(stream)
json_obj = json.dumps(yaml_content)
find_quote_in_key(json.loads(json_obj))
if is_quote_found:
    exit(1)
exit(0)
```
### Preventing push to the master branch
(Script is [from](https://github.com/smiley/git-hooks/blob/master/pre-push/prevent-push-to-protected-branch.sh))

- file name : pre-push.protect-master
```bash
#!/bin/sh
protected_branch='push-test'
current_branch=$(git symbolic-ref HEAD | sed -e 's,.*/\(.*\),\1,')
if [ $protected_branch = $current_branch ]
then
    read -p "You're about to push $protected_branch, is that what you intended? [y|n] " -n 1 -r < /dev/tty
    echo
    if echo $REPLY | grep -E '^[Yy]$' > /dev/null
    then
        exit 0
    fi
    exit 1
else
    exit 0
fi
```
Don't forget to execute : ln -s hook-chain pre-push.

## Install hooks

In any git project, run git init will install the hooks under .git/hooks/ folder, however, the existing files won't be overriden, meaning it is needed to delete the existing hooks in the git repository after updating the hook scripts.

```bash
rm .git/hooks/*
git init
```

I suggest creating an alias command for doing that.

Copy the following in the ~/.profile file

```bash
alias my_git_init='rm .git/hooks/*; git init'
```

Then run my_git_init under the git project will always delete the existing hooks and install the latest ones.
