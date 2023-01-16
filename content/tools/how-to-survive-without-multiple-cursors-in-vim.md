+++
title = "How to survive without multiple cursors in vim" 
description = "Alternatives we can use in vim instead of multiple cursors"
date = 2023-01-15
lang = "en"
[taxonomies]
tags = ["vim", "neovim", "shell"]
[extra]
shared = []
+++

Yes, multiple cursors are magical. They are convenient, easy to use and every modern editor has them. Now vim on the other hand doesn't have this feature. It's fine. We can be happy without them. Well... I can and I'm going to tell you how.

We'll go throught a few scenarios where multiple cursors can be useful and I'll tell you what alternatives we have in vim.

## Replace word under the cursor

In vim we begin this process by searching the word under the cursor, for this we press the `*` key. Then we press the sequence `cgn` to replace the next match. If we want to repeat this action we press the `.` key. If we want to ignore a match we move to the next with `n`.

We can make this process a lot more convenient by making a keybinding.

```vim
nnoremap <leader>j *``cgn
```

With this we can use the leader key + j to replace the word under the cursor. We can navigate to other matches with `n` or `N`, then use the `.` key when we want to replace the text.

{{ asciinema(id="qxb6feyI4ieLUlFUwO2kXzV4I") }}

> See in [asciinema](https://asciinema.org/a/540516). 

## Rename a variable

Maybe the thing we want to change is a variable in our code, in this case we only need to change the valid references. Things get complicated here. Since vim isn't an IDE this kind of features are not available out of the box. But it doesn't mean is impossible, we can still do it, there are plugins that allow us to use LSP servers. It just so happens that rename variables is one the things an LSP server can do.

I use neovim btw, not vim. I just need something like this in my config.

```vim
lua require('lspconfig').tsserver.setup({})

nnoremap <F2> <cmd>lua vim.lsp.buf.rename()<cr>
```

Here I'm using [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig) to configure [tsserver](https://github.com/theia-ide/typescript-language-server). Then I create the keybinding `<F2>` to rename the variable under the cursor.

{{ asciinema(id="217Yk9e9HtmuPi0t8HsNPHOAN") }}

> See in [asciinema](https://asciinema.org/a/540539).

If you use vim you can try out one these plugins:

* [vim-lsp](https://github.com/prabirshrestha/vim-lsp)
* [coc.nvim](https://github.com/neoclide/coc.nvim)

## Replace a selection

So maybe the thing we want to change is not a word, maybe is a sentence or an html attribute. For this we don't have a built-in tool, we need to implement something ourselves. So let's do that.

First thing we should do is add the current selection to the "search register".

```vim
let @/=escape(@", '/')
```

Here we take the text inside the `"` register, where our selection is stored, and put it in the `/` register.

The next step would be to delete the selection and enter insert mode. We use this sequence.

```
"_cgn
```

With `"_` we tell vim that our next operation should store text in the `_` register. With `cgn` we replace the closest match to our search.

If we put the pieces together in a keybinding, we get this.

```vim
xnoremap <leader>j y<cmd>let @/=escape(@", '/')<cr>"_cgn
```

But this command doesn't handle newlines. It won't work with selection with multiple lines. But we can handle that.

```vim
xnoremap <leader>j y<cmd>substitute(escape(@", '/'), '\n', '\\n', 'g')<cr>"_cgn
```

Now here we use the `substitute` function to replace the newline character with `\n`, this way our search term will always be one line.

How do we use this? Same way we did with the previous keybinding `<leader>j` in the section "Replace word under the cursor". But here we must first enter visual mode and select something. Everything else works the same, if we want to replace we use the `.` key, then we move to the next match with `n`.

## Add text to the beginning of a list

Let's say we have a list of words and we want to convert them to an ordered list in markdown.

Want to turn this.

```
volar
html
cssls
```

Into this.

```
1. volar
1. html
1. cssls
```

In vim we have a mode called Visual Block, when in this mode we can add text to each line selected if we go to insert mode using `I` or `A`. After you added the text and exit insert mode vim will repeat the action on every line.

Let's go step by step how to use this feature.

{{ asciinema(id="tjCikrHM5d1QsoSk00u9iUQMa") }}

> See in [asciinema](https://asciinema.org/a/539909).

1. We go to the first character in the line.
1. Press `Ctrl + v` to enter visual block mode.
1. Select the lines we want to change.
1. Press `I` to place the cursor at the beginning of the selection.
1. Add the text.
1. Press `Esc`.

## Append to the end of a list

We can do that too. The steps are almost identical to the previous section, the only difference is we need to extend the selection until the end of the line.

Let's add something to the previous example.

Okay, we have our ordered list but now we want to append `(is supported)` to the end of each item.

{{ asciinema(id="eF1Pdf34IY4Sj5PixzjCqPIxz") }}

> See in [asciinema](https://asciinema.org/a/539912).

1. We go to the first character in the line.
1. Press `Ctrl + v` to enter visual block mode.
1. Select the lines we want to change.
1. Expand the selection to the end of the line using `$`.
1. Press `A` to place the cursor at the end of the selection.
1. Add the text.
1. Press `Esc`.

## Repeat movements

Visual block mode can be useful but is very limited. We can only add text in one place. What do we do in more complex scenarios? We use macros. A macro is a piece of text that describes a sequence of keypresses. We can "record" a macro and repeat the sequence as many times as we want.

How do we use macros? We need to pick a register so the first step is to press `q` followed by a letter. Then we go and do whatever actions we want. We stop recording the macro by pressing `q` again. To repeat these actions we press `@` followed by the register we chose in the first step.

Example time.

We have this list.

```
volar
html
cssls
eslint
```

And we want to turn it into an ordered list of links.

```
1. [volar](http://localhost/how-to-configure-volar-lsp)
1. [html](http://localhost/how-to-configure-html-lsp)
1. [cssls](http://localhost/how-to-configure-cssls-lsp)
1. [eslint](http://localhost/how-to-configure-eslint-lsp)
```

Notice here we need to add text to the beginning and the end of the list. Additionally, we need to copy the item in the middle of the link.

What do we do? We record a macro, modify the first item then repeat the macro to convert the rest of the list. These are the steps.

1. Record the macro in the register `i`. Press `qi`.
1. Modify the first item.
1. We stop recording the macro by pressing `q` again.
1. We repeat the macro three times using `3@i`.

{{ asciinema(id="85qQ7TtkOclesgMJNBDcKk5BK") }}

> See in [asciinema](https://asciinema.org/a/540581).

When we apply a macro using a count we need to consider the position of the cursor. In this particular case I begin the macro by pressing `0`, to make sure the cursor is at the beginning of the line. Then at the very end of the macro I press `j`, so the last movement can place the cursor in the next line.

### Apply macro in specific lines

Another interesting way to apply a macro is by using the `g` command. With it we can begin a search and then execute a command in each line there is a match. In our case we want to apply a macro, we can do that with the command `normal @i` (where `i` can be any register).

Say we want to look for every line with the word `vim` then apply a macro. We do this.

```vim
:g/vim/normal @i
```

Now, you might want to inspect the result of the search before doing anything you'll regret. If you omit the last section with the command then `:g` will just print the lines.

```vim
:g/vim/
```

If everything looks okay then add the `normal @i` bit.

### Apply a macro in a selection

We don't have to use the `g` command. The `normal` commands supports ranges, this means we can select any amount of lines then execute this.

```vim
'<,'>normal @i
```

> Note: Don't worry about writing `'<,'>`, vim will add that for you when you go from visual mode to command mode.

That command will execute the macro in each line of the selection. Keep in mind the cursor will be placed at the beginning of the line automatically.

### Search selection and apply macro

Yet another alternative to the `g` command. Because maybe we don't want to make a regular expresion to search. Most of the time I just want to select something, search it, then apply a macro. We already know how to do all those things, let's just put the pieces together.

Remember this guy?

```vim
y<cmd>let @/=substitute(escape(@", '/'), '\n', '\\n', 'g')<cr>
```

Is the thing we use to search the current selection. After this sequence we need to begin the macro. So we will add this.

```
gvqi
```

Since we lose the selection when pressing `y` we need to reselect everything, so we use `gv`. Then `qi` just begins to record the macro in the register `i`.

Now everything together.

```vim
xnoremap <leader>i y<cmd>let @/=substitute(escape(@", '/'), '\n', '\\n', 'g')<cr>gvqi
```

The story is not over yet. We need to apply the macro in each match. We will use `gn` to navigate to the match and select it. Once the cursor is in the match we apply the macro with `@i`. We are not doing that manually, no, we are going to create a keybinding.

```vim
nnoremap <F8> gn@i
```

Story time.

A few months ago I was trying this plugin manager, [packer.nvim](https://github.com/wbthomason/packer.nvim). I had something like this in my configuration.

```lua
require('packer').startup(function(use)
  use({
    'nvim-lualine/lualine.nvim',
    config = function() require('plugins.lualine') end,
  })
  use({
    'akinsho/bufferline.nvim',
    config = function() require('plugins.bufferline') end,
  })
  use({
    'lukas-reineke/indent-blankline.nvim',
    config = function() require('plugins.indent-blankline') end,
  })
end)
```

It bothered me that I had to repeat `function() require...` for each plugin. And yes, it's packer thing. They do weird stuff with functions. Anyway, I looked around in a few places and found a way to reduce the boilerplate. I wrote this function.

```lua
local function load(name)
  return string.format([[pcall(require, 'plugins.%s')]], name)
end
```

And with it I could write the `config` option like this.

```lua
config = load('lualine')
```

Now it's refactor time. I had to change each `config` option and this is how I did it.

{{ asciinema(id="OUlPjhimpPKIDxEqPJgoxHcYt") }}

> Ver en [asciinema](https://asciinema.org/a/OUlPjhimpPKIDxEqPJgoxHcYt).

1. I select the pattern I want to search. Go to visual mode and select `config = `.
1. I start recording the macro using `<leader>i`.
1. I replace the old function with `load`.
1. End the macro by pressing `q`.
1. Press `n` to go to the next match.
1. Press `<F8>` to apply the macro.

## The Good Old Search and Replace

Sometimes a simple tool can do the job.

So vim has the `substitute` command. This is the syntax.

```vim
%s/<pattern>/<replacement>/g
```

In here `%` is a range, it means the current buffer. Basically, search the entire buffer. The `s` is the actual command, 'cause we don't need to type `substitute`. `<pattern>` is the regular expression we want to search. `<replacement>` is the new text. And `g` is a flag, it tells vim to search the entire line. And notice that each item is separated by `/`, we could use other characters (like `#`) if we wanted to.

Say want to change the word `config` with `setup`. We just do this.

```vim
%s/config/setup/g
```

It wasn't that difficult. We don't need to know regular expressions to use `substitute`.

### Fighting Kirby

Okay. But there is something you should learn about regular expressions. Is just one simple trick, I swear.

What's this fighting Kirby deal? Is a way to remember this.

```
\(.*\)
```

> I learned this from [ThePrimeagen](https://github.com/ThePrimeagen).

With this pattern we can create a "group". Groups can capture the text in the search pattern, and we can reuse that text in the replacement pattern.

Consider this pattern.

```
%s/`\(.*\)`/[\1](#how-to-configure-\1-lsp)
```

Here we capture the text that's surrounded by backticks. Then we reference that text using `\1` and the replacement pattern. Here's a demo.

{{ asciinema(id="odouPFLqH5SSHkJuSVhp28yjR") }}

> Ver en [asciinema](https://asciinema.org/a/542501).

This demo shows how neovim makes a live preview, showing me the effects of the command in realtime. And yes, because of this feature I think search and replace is a decent alternative to multiple cursors.

If you liked the fighting Kirby consider making a keybinding for it.

```vim
cnoremap <F2> \(.*\)
```

This way you can press `<F2>` in command mode and it'll type it for you.

## Conclusion

You are ready. You can go out to the world and be productive in vim. I can't guarantee your happiness but you will survive. And its okay if you think multiple cursors are superior to all of this. Doesn't matter, now you can live without them when using vim.

