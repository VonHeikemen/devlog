+++
title = "Navigate your command history with ease"
description = "Let's take advantage of all that data in our command history"
date = 2021-10-26
lang = "en"
[taxonomies]
tags = ["shell", "zsh"]
+++

When you spend too much time writting commands in the terminal it comes a time when you can't remember every program you have used during the day, not to mention previous days. Lucky for us we don't have to, the "shell" can do it for you. And today we are going to learn how can take advantage of this feature to improve the user experience in the terminal.

## Commands and the arrow keys

I'm sure everyone here knows about the arrow keys, right? We can navigate the history up and down with them. Some of you might know about `ctrl+r`, which enables an interactive "reverse search" of old commands. But what if I told you we can get the best of two worlds?

We can write the first few letters of a command and press the up arrow key to start navigating throught the commands that begin with those letters.

Let's pretend we have these commands saved in our history.

```
node ./test.js
vi /tmp/text.txt
nvim /tmp/text.txt
vi /tmp/test.js
echo "a string with vi in it"
vi /tmp/other-text.txt
```

We want to look for the files we modified using `vi`. So we start by typing `vi` and right after we press the up key you'll notice your shell shows only these commands.

```
vi /tmp/text.txt
vi /tmp/test.js
vi /tmp/other-text.txt
```

We can stop pressing the up key 20 times just because we don't want to write `npm start`. Type `np` + up arrow and we're there.

Sounds good, how can we enable this feature? Depends on your shell.

* In bash:

```sh
# Up arrow
bind '"\e[A": history-search-backward'

# Down arrow
bind '"\e[B": history-search-forward'
```

* In zsh:

```sh
autoload -U up-line-or-beginning-search
autoload -U down-line-or-beginning-search

# Up arrow
bindkey "${terminfo[kcuu1]}" up-line-or-beginning-search

# Down arrow
bindkey "${terminfo[kcud1]}" down-line-or-beginning-search
```

> Note: if you use `oh-my-zsh` it is already enabled for you.

If for some reason `terminfo` is not defined try this in your terminal: press `ctrl+v` then one of the arrows, weird stuff will appear on the screen. In my case I get:

* Up: `^[OA`
* Down: `^[OB`

So use that instead of `terminfo`. This works on my machine.

```sh
bindkey '^[OA' up-line-or-beginning-search
bindkey '^[OB' down-line-or-beginning-search
```

## Interactive search

If you didn't like the previous "trick" that's fine. Maybe you prefer the good old `ctrl+r`. Me? If I'm going to do an interactive search, I rather use [fzf](https://github.com/junegunn/fzf). With `fzf` we get [these scripts](https://github.com/junegunn/fzf/tree/master/shell), `key-bindings.*sh` in particular enables a history search with a better user experience than the old `ctrl+r`. Sadly there are too many ways to enable this (some depend on your OS) so if you want it, better go look for the method that's appropiate for your case.

## Magic space

Both `bash` and `zsh` have this thing called "history-expansion", it allow us to use pieces of data from the command history in our current command. For example:

* `!!` is the last command in our history. So something like `sudo !!` is totally valid, in here we are saying "run the last command as sudo".

* `!$` is the last argument of the last command. If our last command was `nano /tmp/test.txt`, we could use `cat !$` to check the content of `/tmp/test.txt`.

* `!*` expands to all the arguments of the last command.

There are more variants and combination but those are the ones I use frequently.

Anyway, if I'm using any of these symbols I'm going to "expand" them before running the command. I wouldn't run `sudo !!` without knowing for sure what is `!!`. To do this there is a command called `magic-space`. Just like our first "trick" we need to bind it to a key, which is usually space.

* bash

```sh
# Space, but magical
bind Space:magic-space
```

* zsh

```sh
# Space, but magical
bindkey ' ' magic-space
```

Now we could write `sudo !!` then press space and watch how `!!` transforms into the command we want to run.

## Conclusion

We learned a few tricks that can improve the user experience in our shell. We learned how to speed up our search with the arrow keys. Found out we can use `fzf` to search in the command history, basically like a better version of `ctrl+r`. And finally, we got a tiny glimpse of this thing called history-expansion, we can use to query the history and use some pieces in our commands.

