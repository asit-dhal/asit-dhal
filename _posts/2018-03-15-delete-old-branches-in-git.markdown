---
layout: post
title:  "Delete old branches in Git"
date:   2018-08-01 22:00:47 +0200
categories: git
---

Sometimes there are many branches, accumulated over the years in a git repository. And you need to delete it to save some space or make management of your repository easy.

To get the list of all local branches

{% highlight bash %}

╭─asit@gandua ~/stringify ‹master*›
╰─$ git branch              
  ci-build-script
  dev
* master
  remotes/origin/add-license-1
  remotes/origin/benchmark

{% endhighlight %}


After that, you can iterate over all local branches and get some technical details like branch creation date(age of the branch), last commit date, last commit message etc.

I will try to delete ci-build-script.

{% highlight bash %}

╭─asit@gandua ~/stringify ‹master*›
╰─$ git merge-base ci-build-script master 
e3b47d97284b911baa266853b208f5ea25115e67

{% endhighlight %}

git merge-base tries to find out a good common ancestor. This can be assumed to return the earliest commit id where branch deviates from master branch.

You can use the commit id of the common ancestor to get creation date and the age of the branch.

{% highlight bash %}

╭─asit@gandua ~/stringify ‹master*› 
╰─$ git log — pretty=format:”%ad” — date=short -n 1 e3b47d97284b911baa266853b208f5ea25115e67
2017–09–25
╭─asit@gandua ~/stringify ‹master*› 
╰─$ git log — pretty=format:”%cr” — date=short -n 1 e3b47d97284b911baa266853b208f5ea25115e67
6 months ago

{% endhighlight %}

Similarly, you can use the branch name to get the latest commit date, age of last commit and the subject line of last commit message.

{% highlight bash %}

╭─asit@gandua ~/stringify ‹master*› 
╰─$ git log — pretty=format:”%ad” — date=short -n 1 ci-build-script 
2017–09–26
╭─asit@gandua ~/stringify ‹master*› 
╰─$ git log — pretty=format:”%cr” — date=short -n 1 ci-build-script
5 months ago
╭─asit@gandua ~/stringify  ‹master*› 
╰─$ git log --pretty=format:"%s" -n 1 ci-build-script              
gcc tnd clang config for travis

{% endhighlight %}

From that you can make an intelligent guess, should you delete the branch or not. The branch ci-build-script was created 5 months ago and last committed 5 months ago. The commit message was “gcc and clang config for travis”.

You can delete it

{% highlight bash %}

git branch -d ci-build-script

{% endhighlight %}

-d option won’t allow you delete unless it is merged. In some cases branch were never merged. So, you can use -D option

You can wrap all the above and make a script.

{% highlight bash %}

#!/bin/bash
#
# Copyright(c) 2018 Asit Dhal.
# Distributed under the MIT License (http://opensource.org/licenses/MIT)
#

LIGHT_BLUE='\033[1;34m'
RED='\033[0;31m'
NC='\033[0m' # No Color
INTERACTIVE=""
FORCE_DELETE=""
MASTER="master"
while [[ $# -gt 0 ]]
do
  key="$1"
  case $key in
    -f)
    FORCE_DELETE="y"
    shift
    ;;
    -i)
    INTERACTIVE="i"
    shift
    ;;
    -fi|-if)
    INTERACTIVE="i"
    FORCE_DELETE="y"
    shift
    ;;
    -h)
    echo "$(basename $0) -i Intecative deletion of local branches"
    echo "$(basename $0) -f Force delete the branch(es)"
    echo "$(basename $0)    List local branches with creation and last update date"
    exit
    ;;
    *)
    shift
  esac
done

current_branch=$(git branch | grep "*")
current_branch=${current_branch/* /}

if [ "${current_branch}" != "${MASTER}" ]; then
  printf "Please change the current branch to ${LIGHT_BLUE}${MASTER}${NC}\n"
  exit
fi

if ! git diff-files --quiet; then 
  printf "you have unstaged changes. $(basename $0) needs a clean working index\n"
  git diff-files --name-status
  printf "Please commit or stash them.\n"
  exit
fi

if ! git diff-index --cached --quiet HEAD; then
  printf "your index contains uncommitted changes. $(basename $0) needs a clean working index\n"
  git diff-index --cached --name-status HEAD
  printf "Please commit or stash them.\n"
  exit  
fi

for brnch in $(git branch | sed /\*/d); do
  created_commit_id=$(git merge-base ${brnch} ${MASTER})
  created_date=$(git log --pretty=format:"%ad" --date=short -n 1 ${created_commit_id})
  created_ago=$(git log --pretty=format:"%cr" --date=short -n 1 ${created_commit_id})
  last_updated_date=$(git log --pretty=format:"%ad" --date=short -n 1 ${brnch})
  updated_before=$(git log --pretty=format:"%cr" --date=short -n 1 ${brnch})
  commit_message_subject=$(git log --pretty=format:'%s' -n 1 ${brnch})
  printf "Branch name         : ${brnch} \n"
  printf "Created on          : ${created_date}${RED}(${created_ago})${NC}\n"
  printf "Last updated on     : ${last_updated_date}${RED}(${updated_before}${NC})\n"
  printf "Last commit message : ${commit_message_subject}\n"
  if [ "${INTERACTIVE}" == "i" ]; then
    printf "${LIGHT_BLUE}Delete the branch, followed by [y/n]?${NC} "
    read ip
    if [ "$ip" == "y" ]; then
      if [ "${FORCE_DELETE}" == "y" ]; then
        git branch -D ${brnch}
      else
        git branch -d ${brnch}
        if [ $? -ne 0 ]; then
          printf "${LIGHT_BLUE}Run with -fi command to delete ${brnch}${NC}\n"
        fi
      fi
    fi
  fi
done

{% endhighlight %}

The script has 2 options: -i and -f-.

If the run the script without any options, it will give you list of all local branches and the above information.

{% highlight bash %}

╰─$ git-clean-local-branches 
Branch name : ci-build-script 
Created on : 2017–09–25(6 months ago)
Last updated on : 2017–09–26(5 months ago)
Last commit message : gcc tnd clang config for travis
Branch name : dev 
Created on : 2017–10–14(5 months ago)
Last updated on : 2017–11–25(4 months ago)
Last commit message : stringify improvement, size and type name removal
Branch name : remotes/origin/add-license-1 
Created on : 2017–10–14(5 months ago)
Last updated on : 2017–10–14(5 months ago)
Last commit message : Create LICENSE
Branch name : remotes/origin/benchmark 
Created on : 2017–10–06(5 months ago)
Last updated on : 2017–10–16(5 months ago)
Last commit message : cxx-pretty print integration

{% endhighlight %}

-i option allows you interactive deletion. -f option uses git delete -D(force delete).

{% highlight bash %}

╭─asit@gandua ~/stringify ‹master*› 
╰─$ git-clean-local-branches -fi
Branch name : ci-build-script 
Created on : 2017–09–25(6 months ago)
Last updated on : 2017–09–26(5 months ago)
Last commit message : gcc tnd clang config for travis
Delete the branch, followed by [y/n]? y
Deleted branch ci-build-script (was 2cb728c).
Branch name : dev 
Created on : 2017–10–14(5 months ago)
Last updated on : 2017–11–25(4 months ago)
Last commit message : stringify improvement, size and type name removal
Delete the branch, followed by [y/n]? y
Deleted branch dev (was f8698b4).

{% endhighlight %}

Thanks for reading.