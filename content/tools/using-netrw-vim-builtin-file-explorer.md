+++
title = "Using Netrw, vim's builtin file explorer" 
description = "Where we learn how to use Netrw"
date = 2021-01-10
lang = "en"
[taxonomies]
tags = ["vim", "shell"]
+++

Did you know that vim has a file explorer? It's a plugin that comes bundled with vim. It's called Netrw... and it's not very popular, at least not if you compare it with something like NERDtree. The reasons for this could include 1) is not very intuitive. 2) has a few annoying limitations. 3) doesn't look cool. Today we are going to learn how to use it, how to get around those limitations and in the process we're going to turn it into something more intuitive and easier to use.

## Meeting Netrw

Let's begin our exploration by taking a quick look at it. We can do this if we try to open a directory using vim (something like this: `vim .`). Assuming you don't have a heavily customized `vimrc`, Netrw should look like this.

![netrw in full screen](https://res.cloudinary.com/vonheikemen/image/upload/v1609938838/devlog/using-vim-netrw/netrw-full-screen_2021-01-06_09-13-11.png)

The first thing we see is a banner with information about the current directory. This is what is shows us.

- Information about Netrw, the name and version (v156).
- The path of the current directory.
- The criteria it's using to sort the files, in this case is doing it by name.
- Another sort criteria. This time is a sequence that describes the priority it gives to a file according to its suffix.
- "Quick help". In here you can see a few hints about actions that Netrw can perform.

Fun fact, you can actually interact with some of the "options" in the banner. So, if you put the cursor on the line that says "sorted" and press `Enter` you'll change the order of the files. You can order them by name, last update, size or the extension of the file. And the quick help can show you keymaps for some common tasks.

After the banner we have our directories and files. `../` is the parent directory and `./` is the current directory. Lastly, we have our files sorted perfectly.

## Usage

Now that we now how Netrw looks like let's cover some of its basic features.

### How to call it

Our first stop is the `:Explore` command. Using it with no arguments will show the directory of the file we are editing. If we don't want that we could give it the path to the directory we want. Depending on your vim config, specifically the `hidden` option, it will do things differently. If `hidden` is disabled (this is the default) and there are no unsaved changes in the current file, `:Explore` will make Netrw occupy the entire window. If we do have unsaved changes in a file it will create a horizontal split and have Netrw in the upper window.

![netrw taking half the screen](https://res.cloudinary.com/vonheikemen/image/upload/v1609951259/devlog/using-vim-netrw/netrw-half-screen_2021-01-06_12-39-50.png)

> If we wanted a vertical split we would use the `:Explore!` command.

If `hidden` is enabled Netrw will always occupy the whole window.

Now let's talk about some `:Explore` variants we have available.

- Hexplore: Will create a horizontal split and show the directory in the lower window. The variant with an `!` will show the directory in the opposite side.

- Vexplore: Will create a vertical split and show the directory on the left side. The variant with an `!` will show the directory in the opposite side.

- Sexplore: Will create a horizontal split and show the directory in the upper window. The variant with an `!` will create a vertical split and show the directory on the left side. 

- Texplore: Will create a new tabpage to show the directory.

- Lexplore: It works almost like `Vexplore`, but `Lexplore` will open a file on the window where we called the command. It will also work as way to toggle a Netrw window. You can watch it in action in this demo.

{{ asciinema(id="Fa9y0AieDUImMHZjUbjKzjlwn") }}

> See in [asciinema](https://asciinema.org/a/Fa9y0AieDUImMHZjUbjKzjlwn).

### Navigation

If we want to move between directories and files these are the keymaps we need to know:

- `Enter`: Opens a directory or a file.
- `-`: Go up to the parent directory.
- `u`: Go back to the previous directory in the history.
- `gb`: Jump to the most recent directory saved on the "Bookmarks". To create a bookmark we use `mb`.

Let's recap. If we want to "go down a directory" we use `Enter`. To "go up" we use `-`. To go back, `u`. And if we want to "jump" quickly to a directory of our choosing we should first add it to the bookmarks (using `mb`) and then we can use `gb` to go there.

### File operations

We know how to move, now let's see how can we perform some of the most common task on our files.

- `p`: Opens a preview window.

- `<C-w>z`: `Ctrl + w` and then `z`. Closes the preview window.

- `gh`: Toggles the hidden files.

- `%`: Creates a file. Well... it actually doesn't, it just gives you the opportunity to create one. When you press `%` vim will ask the name you want to give the file and then it lets you edit it. After entering the name you have to save the file (using `:write`) to create it.

- `R`: Renames a file

- `mt`: Assign the "target directory" used by the move and copy commands.

- `mf`: Marks a file or directory. Any action that can be performed on multiple files depend on these marks. So if you want to copy, move or delete files, you need to mark them.

- `mc`: Copy the marked files in the target directory.

- `mm`: Move the marked files to the target directory.

- `mx`: Runs an external command on the marked files.

- `D`: Deletes a file or an empty directory. vim will not let us delete a non-empty directory. I'll show how to bypass this later on.

- `d`: Creates a directory.

### Perform an action on multiple files

After reading about those keymaps I bet you're wondering how does one copy or move a file. I'll do an example moving some files around.

This will be a three step process:

- Assign the target directory.

- Mark the files we want to move.

- Run the appropriate command, in our case `mm`.

{{ asciinema(id="YkvegGilPQpSbABrOcFZYhN7W") }}

> See in [asciinema](https://asciinema.org/a/YkvegGilPQpSbABrOcFZYhN7W).

This is what happens in the demo:

- *00:00-00:17* I use `:Explore` to open Netrw. Then check the content of `test dir`.

- *00:18* I assign `test dir` as the target directory. Notice how the banner updates to show us the target directory. This line gets added.

```
"   Copy/Move Tgt: /tmp/vim/test dir/ (local)
```

- *00:20-00:25* I mark the files `a-file.txt` and `another-file.txt`. To indicate they are marked vim shows us the name of the files in bold.

- *00:25-00:27*  I press `mm` to move the files, and these disappear from the current window.

- *00:29* Now I check if the files are inside `test dir` (they are).

And this is it, to copy and delete this is the process. To run external commands and delete files is the same thing, except we don't need a target directory.

## Netrw's limitations

- When moving files.

This happens on linux and maybe macOS is the same. In our previous example we moved `a-file.txt` to `test dir`, and that worked great, but if you try to move back `a-file.txt` to the parent directory you'll get this error. 

```
**error** (netrw) tried using g:netrw_localmovecmd<mv>; it doesn't work!
```

> This doesn't happen when you try to copy.

As far as I know this happens when the current directory (in the buffer) and the directory we are browsing don't match. To fix this you can set the global variable `g:netrw_keepdir` to zero.

```vim
let g:netrw_keepdir = 0
```

- When performing an action on marked files.

When you try to do something on marked files, the action only applies to the files that are listed in the current buffer.

Let's say we have this file structure.

```
vim
├── mini-plugins
│   ├── better-netrw.vim
│   ├── guess-indentation.vim
│   └── project-buffers.vim
├── test dir
│   ├── a-file.txt *
│   ├── another-file.txt *
│   └── text.txt
├── custom-commands.vim
└── init.vim *
```

The files with an `*` are the ones we have marked. If we are in the `vim` directory and we try to move the files to `mini plugins` only `init.vim` will be in the target directory. In theory this is a good thing, because we will always have in sight the files we are operating on.

- Netrw can't delete non-empty directories using `D`.

And of course the answer to this is: use an external command. If you paid attention on previous sections you'll know that the `mx` keymap can help us do just that. Here it is in action.

{{ asciinema(id="YbscBomZSa752kXnEASUnaxlx") }}

> See in [asciinema](https://asciinema.org/a/YbscBomZSa752kXnEASUnaxlx).

So just in case. The solution: mark the directories with `mf`, use `mx` and type the command you need (`rm -r` in the demo). That's it. But can we make this more convenient? Yes we can, and we are going to that in the next section.

## Customization

If you decided to give Netrw a chance you might want to make some tweaks to make it nicer.

### Recommended config

Keep the current directory and the browsing directory synced. This helps you avoid the move files error.

```vim
let g:netrw_keepdir = 0
```

Change the size of the Netrw window when it creates a split. I think 30% is fine.

```vim
let g:netrw_winsize = 30
```

Hide the banner (if you want). To show it temporarily you can use `I` inside Netrw.

```vim
let g:netrw_banner = 0
```

Hide dotfiles on load.

```vim
let g:netrw_list_hide = '\(^\|\s\s\)\zs\.\S\+'
```

Change the copy command. Mostly to enable recursive copy of directories.

```vim
let g:netrw_localcopydircmd = 'cp -r'
```

Highlight marked files in the same way search matches are.

```vim
hi! link netrwMarkFile Search
```

> This is the easiest way I could think of to highlight marks. This may cause some confusion if you begin a search in Netrw and have marked files. If you wish to apply other colors search information about the `highlight` command.

### Keymaps

Now that Netrw looks better-ish, let's make it easier to use.

#### Better call Netrw

We begin by changing the way we call Netrw. We bind `:Lexplore` to a shortcut so we can toggle it whenever we want.

```vim
nnoremap <leader>dd :Lexplore %:p:h<CR>
nnoremap <Leader>da :Lexplore<CR>
```

- `Leader dd`: Will open Netrw in the directory of the current file.
- `Leader da`: Will open Netrw in the current working directory.

#### Navigation

Unfortunately we don't have a direct way to assign a keymap in Netrw. We can still have them but it does requires a few steps.

Netrw is a plugin that defines its own filetype, so we are going to use that to our advantage. What we are going to do is place our keymaps inside a function and create an `autocommand` that calls it everytime vim opens a filetype `netrw`.

```vim
function! NetrwMapping()
endfunction

augroup netrw_mapping
  autocmd!
  autocmd filetype netrw call NetrwMapping()
augroup END
```

With this in our config all we have to do now is place the keymaps inside `NetrwMapping`. Like this.

```vim
function! NetrwMapping()
  nmap <buffer> H u
  nmap <buffer> h -^
  nmap <buffer> l <CR>

  nmap <buffer> . gh
  nmap <buffer> P <C-w>z

  nmap <buffer> L <CR>:Lexplore<CR>
  nmap <buffer> <Leader>dd :Lexplore<CR>
endfunction
```

Since we don't have access to the functions Netrw uses internally (at least not all of them), we use `nmap` to make our keymaps. For example, `H` will be the same thing as pressing `u`, and `u` will trigger the command we want to execute. So this is what we have:

- `H`: Will "go back" in history.
- `h`: Will "go up" a directory.
- `l`: Will open a directory or a file.
- `.`: Will toggle the dotfiles.
- `P`: Will close the preview window.
- `L`: Will open a file and close Netrw.
- `Leader dd`: Will just close Netrw.

With these (plus the recommended config) Netrw can become a decent file explorer. But wait, we can still do more.

#### Marks

Let's find a better way to manage the marks on files. I suggest using `<Tab>`.

```vim
nmap <buffer> <TAB> mf
nmap <buffer> <S-TAB> mF
nmap <buffer> <Leader><TAB> mu
```

- `Tab`: Toggles the mark on a file or directory.
- `Shift Tab`: Will unmark all the files in the current buffer.
- `Leader Tab`: Will remove all the marks on all files.

#### File managing

Since there are quite a few commands related to files we are going to use the `f` key as a prefix to group these together.

```vim
nmap <buffer> ff %:w<CR>:buffer #<CR>
nmap <buffer> fe R
nmap <buffer> fc mc
nmap <buffer> fC mtmc
nmap <buffer> fx mm
nmap <buffer> fX mtmm
nmap <buffer> f; mx
```

- `ff`: Will create a file. But like create it for real. This time, after doing `%` we use `:w<CR>` to save the empty file and `:buffer #<CR>` to go back to Netrw.
- `fe`: Will rename a file.
- `fc`: Will copy the marked files.
- `fC`: We will use this to "skip" a step. After you mark your files you can put the cursor in a directory and this will assign the target directory and copy in one step.
- `fx`: Will move marked files.
- `fX`: Same thing as `fC` but for moving files.
- `f;`: Will be for running external commands on the marked files.

We can still do a couple of things, if you don't mind using some of Netrw's internal variables.

Show a list of marked files.

```vim
nmap <buffer> fl :echo join(netrw#Expose("netrwmarkfilelist"), "\n")<CR>
```

Show the target directory, just in case we want to avoid the banner.

```vim
nmap <buffer> fq :echo 'Target:' . netrw#Expose("netrwmftgt")<CR>
```

Now we can use that along side `mt`.

```vim
nmap <buffer> fd mtfq
```

Again, this is only useful if you really, really want to avoid showing the banner.

#### Bookmarks

In the same way we grouped the file related actions, we do it for bookmarks.

```vim
nmap <buffer> bb mb
nmap <buffer> bd mB
nmap <buffer> bl gb
```

- `bb`: To create a bookmark.
- `bd`: To remove the most recent bookmark.
- `bl`: To jump to the most recent bookmark.

#### Remove files recursively

Last thing we will do is "automate" that process we did to remove non empty directories. For this we will need a function.

```vim
function! NetrwRemoveRecursive()
  if &filetype ==# 'netrw'
    cnoremap <buffer> <CR> rm -r<CR>
    normal mu
    normal mf
    
    try
      normal mx
    catch
      echo "Canceled"
    endtry

    cunmap <buffer> <CR>
  endif
endfunction
```

First thing we do in this function is check if we are in a buffer controlled by Netrw. Then we prepare the remove command. We take advantage of the fact that vim makes us drop to command mode and create a keymap (`<CR>`) that will write the command for us. Next, we use `normal mu` to clear all the marks, 'cause we don't want to remove anything by accident. We then mark the directory under the cursor with `normal mf`. Here comes the funny part, `normal mx` will ask us what command we want to execute, and is at this point when we can abort the process using `ctrl + c` or press `Enter` which will trigger the command `rm -r`. Lastly, we undo the keymap we created in the beginning of the function, because it will be terrible to have it permanently.

And how do we use it?

Creating a keymap inside `NetrwMapping`, of course.

```vim
nmap <buffer> FF :call NetrwRemoveRecursive()<CR>
```

You can find every option and function in this article [here](https://gist.github.com/VonHeikemen/fa6f7c7f114bc36326cda2c964cb52c7).

## Conclusion

Netrw might not be the best file manager in the vim ecosystem but with a little effort we can turn it into an intuitive file explorer. Even if you don't adopt in Netrw in your workflow, knowing how to use it can be handy in some situations. You never know when you're going to be stuck in a remote server without your favorite vim plugins at hand.

## Source

[:help netrw](https://vimhelp.org/pi_netrw.txt.html)

