+++
title = "ZSH global aliases" 
description = "What are they and how can we use them"
date = 2020-07-12
lang = "en"
[taxonomies]
tags = ["zsh", "shell", "todayilearned"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/zsh-global-aliases-4g15"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/zsh-global-aliases"]
]
+++

If you have spend a significant amount of time writing commands in a shell chances are you have come across these handy little things called **aliases**. They usually come with the promise of boosting your productivity. Sometimes we use them to protect us from long commands with cryptic arguments and sometimes we use them just to save ourselves a few keystrokes. Regardless of your motives I'm going to show you another type of alias you might not know of, global aliases.

## A quick recap

The typical situation where find you yourself in need of an alias is when you have a command you use often, like very often, so much that you are kinda tired of typing that thing over and over again. Let's take this command as an example:

```sh
tmux new-session -A -D -s work
```

> Don't know what it does? [Check this](https://explainshell.com/explain?cmd=tmux+new-session+-A+-D+-s+work).

Now, wouldn't it be nice if I could just type this?

```sh
ts work
```

I already feel productive. Who do you get from the first example to this short beauty? With the `alias` command.

```js
alias ts='tmux new-session -A -D -s'
```

You can write that right in your shell and you will have for as long as your zsh session is alive. If you put it somewhere in your `.zshrc` it'll always be available to you.

Notice that I didn't put `work` in it. That is because that last argument is a name of my choosing. By leaving it out of the alias I can do put anything I want after it.

```sh
ts pomodoro
```

All that sounds good. What's the catch? Well, the alias works well when it's the first thing in our command but it fails if you try to use it in another place. Let's expand our example with another alias.

```sh
alias pomd='gone -e "notify-send -u critical Pomodoro Timeout"'
```

> That [gone](https://github.com/guillaumebreton/gone) command is a pomodoro clock.

If you try to do this `ts pomodoro pomd`, you'll only get this message `[exited]`. That's because `pomd` in that case is just like a normal string, it doesn't have any special meaning in that context. Can we overcome that? Yes. Should we? I don't know but I'll show anyway.

## Make it global

If a "simple" alias is not enough for you then all you have to do is add the flag `-g` to the `alias` command.

```sh
alias -g name="some-command"
```

Let's look at a good use case (our pomodoro example is a terrible one). When you have a command with a long output sometimes you might want to use `less` to get navigate better. So let's do a global alias that pipes the output of a command to `less`.

```sh
alias -g L="| less"
```

Now you can test it with a command.

```sh
du --max-depth=1 L
```

> [This is what it does](https://explainshell.com/explain?cmd=du+--max-depth%3D1+%7C+less). It may take a while if you have bunch of stuff in your current working directory.

See? This is convenient, useful and dangerous at the same time. Now every time a lonely `L` appears without quotes in any part of your command it will get replaced by `| less`. Sometimes that's not what you want. If you try to remove the alias using the `unalias` command like this.

```sh
unalias L
```

> To remove the alias you should quotes, like this: `unalias "L"`

You are actually running this command.

```sh
unalias | less
```

To minimize the risk of confusion it is advised that your global alias is in all caps, that is to make it standout from the rest of the command.

By now you probably think this is stupid and a very bad idea but there is hope.

## Another use case

There is a builtin function called `_expand_alias`, it can help us use these type alias for a greater good.

If you use the `bindkey` with no arguments it gives you a list of every keybinding you have right now in your shell. You should already have the `_expand_alias` bound to a shortcut. 
You can check with this.

```sh
bindkey | grep expand_alias
```

You should get: `"^Xa" _expand_alias`. That means it's bound to `ctrl+x a`.

With this new found knowledge we can create an alias for something that is very difficult to remember (or just too damn long to type) and then expand it before running the command.

Do you remember exactly how is it that you can suppress the stderr messages in a command? No? Well, there you go, there is your alias.

```sh
alias -g @noerr="2> /dev/null"
```

So now you can write something like this.

```sh
mkdir /tmp/this-doesnt-exists/name @noerr
```

But before hitting enter press `ctrl+x` then `a` and you'll get this cryptic beauty.

```sh
mkdir /tmp/this-doesnt-exists/name 2> /dev/null 
```

> Remember kids, don't try random commands you see on the internet. Always [check](https://explainshell.com/explain?cmd=mkdir+%2Ftmp%2Fthis-doesnt-exists%2Fname+2%3E+%2Fdev%2Fnull).

## Conclusion

We have reached the end. Now you know what you can do with an alias, what limitations they have, how you can overcome those, and what you probably shouldn't do with them.

If you want some inspiration to create your own alias, [check this](https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins/common-aliases). That's a plugin for [ohmyzsh](https://github.com/ohmyzsh/ohmyzsh). I would recommend taking a look in the source file and just copy the alias you actually find useful.

## Source

* [5 Types Of ZSH Aliases You Should Know](https://thorsten-hans.com/5-types-of-zsh-aliases)
