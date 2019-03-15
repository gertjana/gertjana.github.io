---
layout: post
title:  "Git statuses in Fish command shell"
date:   2013-09-10
categories: [development]
tags: [tip, fish, git]
---
I've been playing around with <a title="Fish Shell" href="http://www.fishshell.com">fish shell</a> for a while now and like it a lot.

The only thing i missed was the GIT information i used to have in my prompt.

I Solved this by added a file: `~/.config/fish/config.fish` with the following content:
 

``` bash 
# in .config/fish/config.fish:
# Fish git prompt
set __fish_git_prompt_showdirtystate 'yes'
set __fish_git_prompt_showstashstate 'yes'
set __fish_git_prompt_showupstream 'yes'
set __fish_git_prompt_color_branch yellow

# Status Chars
set __fish_git_prompt_char_dirtystate '⚡'
set __fish_git_prompt_char_stagedstate '→'
set __fish_git_prompt_char_stashstate '↩'
set __fish_git_prompt_char_upstream_ahead '↑'
set __fish_git_prompt_char_upstream_behind '↓'

function fish_prompt
  set last_status $status

  set_color $fish_color_cwd
  printf '%s' (prompt_pwd)
  set_color normal
  printf '%s ' (__fish_git_prompt)
  set_color $fish_color_cwd
  printf '&gt; '
  set_color normal
end
```

This should work on both linux and OSX
Also on the mac i'm using TotalTerminal which gives me a nice shell dropping down from the top on pressing ctrl twice
