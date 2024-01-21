---
title: Remember all your bash history forever
slug: remember-all-your-bash-history-forever
created: 2014-12-29
tags: [bash]
---

Wouldn't it be great to remember every line ever typed at the bash prompt for
life? This article has instructions to configure the bash history just for
that.

# Configuring bash to save every command line

Append the following lines to `/etc/bash.bashrc`:

```sh
HISTTIMEFORMAT='%F %T '
HISTFILESIZE=-1
HISTSIZE=-1
HISTCONTROL=ignoredups
HISTIGNORE=?:??
shopt -s histappend                 # append to history, don't overwrite it
# attempt to save all lines of a multiple-line command in the same history entry
shopt -s cmdhist
# save multi-line commands to the history with embedded newlines
shopt -s lithist
```

This configures bash to save every command line typed at the interactive shell
prompt (`HISTFILESIZE`) to `~/.bash_history` (default), including a timestamp
(`HISTTIMEFORMAT`) and ignoring consecutive duplicate entries (`HISTCONTROL`).
By setting `HISTSIZE` to the same value as `HISTFILESIZE`, all saved commands
are read back to memory when a new interactive shell starts. The default value
for `HISTSIZE` (500) would load only a fraction of the saved history.

When saving the history at shell exit, history lines are appended to existing
ones, instead of replacing them (`shopt -s histappend`).

By setting `HISTIGNORE=?:??`, lines consisting of just one or two characters are
discarded from the history (e.g. `ls` commands).

These settings are a variation of some very useful discussion in this
[debian administration article](http://www.debian-administration.org/articles/543).

# Removing duplicate history entries

We need to get rid of duplicate lines in the history file. For me an
appropiate point to do it is on each reboot of the machine. This can be
accomplished by adding the following crontab entry (add it with `crontab -e`):

    @reboot $HOME/bin/deduplicate-bash-history.sh

where `deduplicate-bash-history.sh` does the actual work:

```sh
# keep only the first occurrence of each duplicate entry in ~/.bash_history
# 
# 1. Break records at points matching a timestap. Special case
# for the first and last lines in the history file (setting RS to a regexp is a
# GNU awk extension). This way the script matches multi-line commands in the file
# 2. Set the timestamp of the current record to the previous record separator,
# trim starting line break
# 3. Skip empty lines (e.g. the one produced while processing the first line)

out=$(mktemp)
gawk -vRS='(^|\n)#[0-9]+\n' \
    '{
        sub(/\n$/, "")
        if ($0 && cmds[$0]++ == 0)
            print time $0
        time = RT
        sub(/^\n/, "", time)
    }' < ~/.bash_history > $out
mv $out ~/.bash_history
```

The script is a bit tricky because, as explained in the comments, special care
must be taken on multi-line commands and the first and last file entries.
Running the script automatically on every reboot before even the user has
logged in ensures that nobody is using the history file, and avoids mangling
its contents.

# Browsing the history

The `Ctrl-R` key is bound by default to `reverse-search-history`. Likewise, the
`Ctrl-S` key is bound to `forward-search-history` (`man 3 readline`), but this
clashes with the standard key assigned to terminal flow control. To fix that,
add the following to `/etc/bash.bashrc`:

    # Instead of using control-s for flow control, pass it to readline to enable
    # forward incremental search
    stty stop ""

Having both keys available makes it easy to go back to the previous match if
you hit one key too many times.

There is another method to get to a specific history entry. As explained in
[this hacker news comment](https://news.ycombinator.com/item?id=7763752), the
up and down keys can be used to jump to the previous/next history line matching
the current line. Add the following lines to `/etc/inputrc` to configure the
key bindings:

    "\e[A": history-search-backward
    "\e[B": history-search-forward

If you just made a typo in the last command you entered, you may want to remove
that entry from the history immediately. Here are some key bindings you can add
to `/etc/inputrc` for that:

    $if Bash
    # delete last command from history
    Control-x: "history -d $((HISTCMD-2))"
    # delete second to last command from history
    Control-y: "history -d $((HISTCMD-3))"
    $endif

# Doesn't this slow down every bash startup? Does it turn bash in a memory hog?

After having this setup running for months, my history file has grown over a
couple of thousand lines. Now, every interactive shell must load and maintain
every history line into memory. I haven't meassured the startup time, but it
still feels immediate anyway. A quick check comparing the RSS field of `ps u -C
bash` indicates that, for each interactive bash shell, maintaining 20k history
lines in memory requires approximately 3 MiB of additional RAM. For me the
convenience of always having every command I ever typed on a shell at my
dispostion outweights this cost.
