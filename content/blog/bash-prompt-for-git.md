+++
date = "2015-05-31T10:42:52+02:00"
draft = false
title = "Bash Prompt for Git"
type = "post"
tags = ["bash", "git", "hacks"]

+++

***Hacking a bash prompt that adapts when entering a git repository.***

<div style="background-color: #f2dede; color: #a94442; border-color: #ebccd1; padding: 15px; border: 1px solid transparent; border-radius: 4px; margin-bottom: 1rem; font-size: 0.8rem">
<img src="/images/glyphicons-197-circle-exclamation-mark.png" style="display: inline-block; top: 5px; position: relative; margin: 0px 5px 0 0px" alt="Disclaimer"/>
Disclaimer: This is a 15min hack which might include bugs and can definitely be made more performant.
</div>

This bash hack will keep the normal prompt, and only change it when entering a directory that is tracked by git. In a git repository it will display the basename of the directory and branch. The branch name is colored accordingly:

<div style="background-color: grey; border-radius: 4px; padding: 5px 0 5px 0; margin-bottom: 1rem">
<ul style="margin-bottom: 0">
  <li><span style="color: red">Red</span> - uncommitted working
  <li><span style="color: yellow">Yellow</span> - your brancgh is ahead
  <li><span style="color: green">Green</span> - nothing to commit
  <li><span style="color: #ccaa2b">Ochre</span> - all other statuses
</ul>
</div>


### Examples

<div style="position:relative">
<pre>
nils@ratchet:~/project$
</pre>
<div style="position: absolute; right: 0; top: 0; background-color: lightgray; padding: 0 5px 0 5px; border-bottom-left-radius: 5px">
Normal Directory
</div>
</div>

<div style="position:relative">
<pre>
[go]<span style="color: red">(master)</span>$
</pre>
<div style="position: absolute; right: 0; top: 0; background-color: lightgray; padding: 0 5px 0 5px; border-bottom-left-radius: 5px">
Non Commited
</div>
</div>

<div style="position:relative">
<pre>
[go]<span style="color: green">(master)</span>$
</pre>
<div style="position: absolute; right: 0; top: 0; background-color: lightgray; padding: 0 5px 0 5px; border-bottom-left-radius: 5px">
Clean
</div>
</div>

### Code

The following code should be included in the `.bashrc` file

~~~ bash
COLOR_RED="\033[0;31m"
COLOR_YELLOW="\033[0;33m"
COLOR_GREEN="\033[0;32m"
COLOR_OCHRE="\033[38;5;95m"
COLOR_BLUE="\033[0;34m"
COLOR_WHITE="\033[0;37m"
COLOR_RESET="\033[0m"
COLOR_RESET2="\e[0m"

function git_color {
  local git_status="$(git status 2> /dev/null)"

  if [[ ! $git_status =~ "working directory clean" ]]; then
    echo -e $COLOR_RED
  elif [[ $git_status =~ "Your branch is ahead of" ]]; then
    echo -e $COLOR_YELLOW
  elif [[ $git_status =~ "nothing to commit" ]]; then
    echo -e $COLOR_GREEN
  else
    echo -e $COLOR_OCHRE
  fi
}

function git_branch {
  local git_status="$(git status 2> /dev/null)"
  local on_branch="On branch ([^${IFS}]*)"
  local on_commit="HEAD detached at ([^${IFS}]*)"

  if [[ $git_status =~ $on_branch ]]; then
    local branch=${BASH_REMATCH[1]}
    echo "($branch)"
  elif [[ $git_status =~ $on_commit ]]; then
    local commit=${BASH_REMATCH[1]}
    echo "($commit)"
  fi
}

function git_dir {
  echo "$(basename `git rev-parse --show-toplevel`)/$(git rev-parse --show-prefix)"
}

PS1='$(git status &> /dev/null || printf "\u@\h:\w\$ ")' # other dir
PS1+='$(git status &> /dev/null && printf "' # if we are in git dir
PS1+="\[$COLOR_RESET\]"         # reset color
PS1+="[\$(git_dir)]"            # get git dir + path
PS1+="\[\$(git_color)\]"        # colors git status
PS1+="\$(git_branch)"           # prints current branch
PS1+="\[$BLUE\]\[$COLOR_RESET2\]\\$ "   # '#' for root, else '$'
PS1+='")'
export PS1


~~~
