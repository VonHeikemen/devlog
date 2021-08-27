+++
title = "vim's easy mode: making vim behave like a 'conventional' text editor" 
description = "Exploring vim's builtin easy mode"
date = 2021-04-10
lang = "en"
[taxonomies]
tags = ["vim", "shell", "todayilearned"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/vim-s-easy-mode-making-vim-behave-like-a-conventional-text-editor-4h26"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/vims-easy-mode-making-vim-behave-like-a-conventional-text-editor"]
]
+++

So it turns out vim has an "easy mode." Who knew? That will be the topic today, we will learn what's the deal with this mode, we'll figure out how to enable it, see why almost nobody pays attention to it, and lastly I'll show you some stuff you can do to make it a bit nicer.

## Easy, what does that mean?

According to the documentation it was made to help people that don't use vim often enough to learn its commands. In this mode vim is suppose to behave like a conventional text editor, you know the type, one of those that let us edit things immediately.

### What does it do?

It gives us the gift of convenience. On a more serious note, the first thing you'll notice is that we can insert and edit text in a more conventional way. For the people who have entered `vim` by accident and have no knowledge of how it works this feature alone could be a huge improvement. It also provides some keyboard shortcuts that we all know and love, such as:

* `Ctrl + s`: to save a file.
* `Ctrl + z`: to undo changes.
* `Ctrl + y`: to redo some changes.
* `Ctrl + c`: copy text to the clipboard.
* `Ctrl + x`: cut text to the clipboard.
* `Ctrl + v`: paste text from the clipboard.
* `Ctrl + a`: selects all text in the file.

It even enables some options that will make the experience a bit more intuitive, like mouse support, so we can select text with the mouse.

### How does it work?

Good question. This "easy mode" is actually just a set of options written in a file called [evim.vim](https://github.com/vim/vim/blob/314dd79cac2adc10304212d1980d23ecf6782cfc/runtime/evim.vim). You can check it out with vim if you want, use this command `vim -R -c 'edit $VIMRUNTIME/evim.vim'` (to get out use `ZZ` or just close the terminal). So, when we active this mode vim just reads and executes this file.

### How do we use it?

On the command line we provide the `-y` flag to `vim`, like this `vim -y`. Another way would be using the `evim` command.

## The big flaw on the plan

Cool, now why is it that no one talks about this mode? Well maybe is because it makes it even harder to get out of vim (oh, the irony). They were so close. Anyway, if you want to exit vim while in this mode this is what you do: use `ctrl + o` to go in normal mode, press `:`, type `q` and then press `Enter`.

*Why are you like that, vim?*

It does look like this easy mode was designed to be used in a GUI. That's right, with vim in a graphical interface. So I guess they thought it wasn't necessary, because you would just close the "application" where vim is in.

## Some workarounds

Some command-line experts out there must be thinking "well, just use `nano`." I mean, yeah, that's a good solution. But if you want to give this mode a chance, keep reading.

### The GUI

If what we need is a graphical interface then lets just use one. We could create an alias that uses our favorite terminal to open vim.

```sh
alias evim='xterm -e vim -y'
```

Or maybe you prefer a script.

```sh
#! /usr/bin/env sh

xterm -e vim -y $@
```

> Might want to replace `xterm` with something else.

This way when you call the `evim` command or call the script `vim` will open in another window. And when you finish editing stuff you just close that window.

### The missing piece

On the other hand, if all we want is a shortcut to get out I say we create one. We could do this.

```sh
vim -y -c "inoremap <c-q> <c-o>:q<cr>"
```

This command will open vim in easy mode and create a `Ctrl + q` shortcut that will close vim. Again, you can put this in an alias or script.

### Easy by default

There is another way to enable this mode without using command line arguments. In our home folder, we can create a file called `.vimrc` and in it we can put this.

```vim
" enable 'easy mode'
source $VIMRUNTIME/evim.vim

" Ctrl + q to get out of vim
inoremap <c-q> <c-o>:q<cr>
```

Now when you open `vim` just like that, it will already be in easy mode. And what's even better, is that we can get out of it easily using `Ctrl + q`.

This method is what I would recommend for the people who have no intention of using `vim` frequently (or even use it at all). By doing this, even if you enter vim by accident, you can use it like a normal text editor or at the very least you can get out of it fast.

#### Since we are here

By "here" I mean editing the `.vimrc` file, may I suggest a couple options for you to add?

* Avoid creating temporary files.

```vim
set noswapfile
set nobackup
```

* Highlight the line where your cursor is at.

```vim
set cursorline
```

* Enable syntax highlight and better colors.

```vim
if (has("termguicolors"))
  set termguicolors
endif

syntax enable
```

* Shortcuts for search and replace.

```vim
" Ctrl + f to begin searching
inoremap <c-f> <c-o>/

" Ctl + f (x2) to stop highlighting the search results
inoremap <c-f>f <c-o>:nohlsearch<cr>
inoremap <c-f><c-f> <c-o>:nohlsearch<cr>

" F3 go to the previous match
inoremap <F3> <c-o>:normal Nzz<cr>

" F4 go to the next match
inoremap <F4> <c-o>:normal nzz<cr>

" Ctrl + h begin a search and replace
inoremap <c-h> <c-o>:%s///gc<Left><Left><Left><Left>
```

That last one needs some explanation. After pressing `Ctrl + h` you'll have this in the prompt `%s/|//gc` (`|` is the cursor). This is a substitute command, it works something like this.

```vim
s/{search query}/{the replacement}/gc
```

For example, let's say I want to replace `Left` with `Right`, I would need to write this.

```vim
s/Left/Right/gc
```

This will make `vim` search for every `Left` instance, when it finds one it will ask us if we want to replace it, and it will put `Right` if we say yes.

## Conclusion

Now finally you have all the tools you need to make vim behave like a conventional text editor. From now on your experience with vim doesn't have to be so terrible.

## Sources

* [:help easy](https://vimhelp.org/starting.txt.html#easy)
* [:help evim-keys](https://vimhelp.org/starting.txt.html#evim-keys)

