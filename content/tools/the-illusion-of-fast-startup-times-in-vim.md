+++
title = "The illusion of fast startup times in vim"
description = "Speed up vim's startup times with this one weird 'trick'"
date = 2021-08-22
lang = "en"
[taxonomies]
tags = ["vim", "neovim", "shell"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/the-illusion-of-fast-startup-times-in-vim-bfo"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/the-illusion-of-fast-startup-times-in-vim"]
]
+++

With an emphasis on the word "Illusion". What I'm going to show you is not a "magic trick" that will make vim go faster. Don't get your hopes up. The only thing we'll do here is delay the inevitable.

## What do we want to achieve?

I want to load my optional plugins after vim interface is fully loaded and ready to go.

## Why?

Because when I open vim there is a little timeframe of maybe 2 or 3 seconds where my mind goes from "what was I going to do?" to "Yeah, that thing". That's when I want to load the plugins. This way we only the deal with necessary stuff at startup, and then rest of the "nice to have" can happen in this timeframe where **I** am not doing anything in the editor.

## Let's get to it

In vim we have an [event](https://vimhelp.org/autocmd.txt.html#autocmd-events) called `VimEnter`, it gets emitted when is done executing all the initial scripts and plugins. In theory this should be enough, but not for me, I really want to be sure. So what I'd like to do is wait 20 miliseconds after this event is dispatched, it should be enough time for vim to load the interface, and then load the plugins.

For this we will need an autocommand and the `timer_start` function. So let's make a test.We'll print a message 5 seconds after vim has started. We'll put this in our `.vimrc`.

```vim
function! s:load_plugins(t) abort
  echom "vim is ready"
endfunction

augroup user_cmds
  autocmd!
  autocmd VimEnter * call timer_start(5000, function('s:load_plugins'))
augroup END
```

Next time we open vim we should see the message `vim is ready`, after a few seconds. If you missed it you can check with command `:messages`.

Now we know it works. Change the delay time from `5000` to `20`. And in `s:load_plugins` put all the necessary commands to load your plugins.

This next part will depend a lot on how you manage your plugins. I'm going to use a built-in feature known as [packages](https://vimhelp.org/repeat.txt.html#packages). Some plugins managers don't use it, so make sure it does by reading the docs.

The last piece of the puzzle is the `packadd` command, is what we'll use to load all our "optional plugins". So our function should be something like this. 

```vim
function! s:load_plugins(t) abort
  " list of plugins
  packadd vim-surround
  packadd vim-obsession

  "(optional) run all the scripts in the `after` folder
  runtime! OPT after/plugin/*.vim

  "(optional) emit a user 'event'
  " so we can call other autocommands in this moment
  doautocmd User PluginsLoaded
endfunction

augroup user_cmds
  autocmd!
  autocmd VimEnter * call timer_start(20, function('s:load_plugins'))
augroup END
```

That's it. But I'm not done.

## What about lua?

Neovim (version 0.5) makes it possible to write our config file entirely in lua, and a lot of people are doing just that. It's possible to write this in your lua config, but the current limitations of the API will force you to write a lot of vimscript.

Here is what you'll need to do in lua.

```lua
function load_plugins()
  vim.cmd [[
    packadd vim-surround
    packadd vim-obsession

    runtime! OPT after/plugin/*.vim
    doautocmd User PluginsLoaded
  ]]
end

vim.cmd [[
  augroup user_cmds
    autocmd!
    autocmd VimEnter * lua vim.defer_fn(load_plugins, 20)
  augroup END
]]
```

I'm using a global function just because is the easiest way to write it. In general I advice against it. If you know how to create and use a lua module, use that instead.

## Fair warning

You should consider what plugins you're going to load using this method.

Let's take `NERDTree` (a file manager) as an example. If you open up a folder in the terminal, something like `vim .`, `NERDTree` can show you the files in the folder automatically. But that specific feature will not work if you load it after `VimEnter`.

Syntax plugins can also be affected by this. If you load them late in the process you might actually see when the default colors change to the ones declared by the plugin. Is not a nice experience.

## Conclusion

We learned about `VimEnter`, an event that vim emits when the startup process is done. We learned about `timer_start` and `vim.defer_fn`, how we can use them to execute a function at a later time. Finally, we pieced together all of this to load all optional plugins in a specific moment when we as users are not interacting with the editor.

But we should also be careful with this. Loading a plugin outside the startup process can have unexpected side effects. So we should first decide what plugins can be optional.

## Sources

* [:help timers](https://vimhelp.org/eval.txt.html#timers) 
* [:help :doautocmd](https://vimhelp.org/autocmd.txt.html#%3Adoautocmd) 
* [:help User](https://vimhelp.org/autocmd.txt.html#User)
* [:help :packadd](https://vimhelp.org/repeat.txt.html#%3Apackadd)
* [:help :runtime](https://vimhelp.org/repeat.txt.html#%3Aruntime) 
* [:help vim.defer_fn](https://neovim.io/doc/user/lua.html#vim.defer_fn())

