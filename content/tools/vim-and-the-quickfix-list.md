+++
title = "vim and the quickfix list: jump to a location, search and replace in multiple files, and other shenanigans"
description = "The quickfix list and its use cases"
date = 2021-06-13
lang = "en"
[taxonomies]
tags = ["vim", "shell"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/vim-and-the-quickfix-list-jump-to-a-location-search-and-replace-in-multiple-files-and-other-shenanigans-3ki8"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/vim-and-the-quickfix-list"]
]
+++

We are going to learn about an avanced feature of vim, the quickfix list. We're going to figure out how to use it to search (and replace) a pattern in multiple files, and also how can we jump to the location of an error thrown by an external command.

## Quickfix list

Is a special mode where vim show us a list of positions, meaning line and column numbers in a file. This list was made to save the position of the errors we see in the error message of a compiler. This way we could jump quickly to this location, fix the bug and then go to the next or try to compile the code again.

My description can make it sound like a feature with a very minimal scope, but fear not, there is more to it than that. For starters the quickfix list can be created in different ways. We can create it with commands like `:make`, `:vimgrep` and `:grep`. And can also be created programatically with the help of the `setqflist` function, so we do have a great deal of flexibility.

I feel the need to tell you that when I say quickfix list I literally mean the list of positions. There are two ways we can see this list, we have the `quickfix window` and the `location list`. Both of these are "windows" in which the quickfix list is shown. The quickfix window is global, we can only have one in the current active vim session. Meanwhile, we can have many location lists in the current active vim session

## Jump to a location

So, our journey throught the wonderful quickfix list begins with the `:make` command, this is one of vim's native way of calling a compiler. But does the name `make` sound familiar to you? If so, I can assure you it is not a coincidence. vim does assume we have [make](https://www.gnu.org/software/make/) installed on our system. How can you be sure? Well, let's make a test. Create a file called `Makefile` with the following content:

```make
.PHONY: test another

test:
  @echo 'hello'

another:
  @echo 'another'
```

> In this post I will not tell you how to use `make` but if you want to know how you can use it like a regular task runner you can [read this one](https://vinta.ws/code/use-makefile-as-the-task-runner-for-arbitrary-projects.html).

When you call the `:make` command you should get something like this.

```
hello

(1 de 1): hello
```

`make`'s default behaviour is to execute the first "task" it sees in our `Makefile`. Cool, but then how do we get it to execute `another`? We just provide more arguments to our command, like this: `:make [argument]`. If you try to execute `:make another` you should get this.

```
another

(1 de 1): another
```

That's fine, but those commands don't output any error. After showing the messages nothing happens.

This is the perfect time for our first contrived example. Since vim knows how to "read" the errors `gcc` gives let me show you an example using C.

### An example in C

So let's create a file `hello.c` with this.

```c
#include <stdio.h>

int main() {
   printf("Hello, World!\n");
   return 0;
}
```

Notice that there is nothing wrong here. We can compile this thing easily using `make`. So our next step will be to create a `Makefile`. 

```make
.PHONY: run-hello

run-hello:
  gcc -Wall -o hello hello.c
  ./hello
  rm ./hello 
```

With everything in place, we can run `:make --silent run-hello`. If we did everything right we should have our `hello world`.

```
Hello, World!

(1 de 1): Hello, World!
```

If we introduce an error, like say delete a semicolon, this is what we should get.

```
hello.c: In function ‘main’:
hello.c:4:28: error: expected ‘;’ before ‘return’
    printf("Hello, World!\n")
                            ^
                            ;
    return 0;
    ~~~~~~
make: *** [Makefile:7: run-hello] Error 1

(2 de 8): error: expected ‘;’ before ‘return’
```

After showing this message you'll notice that vim took you to the location of the error (how cool is that?). If you want to check the content of the quickfix list you'll need to open the quickfix window using the `:copen` command. You should have something like this. 

```
|| hello.c: In function ‘main’:
hello.c|4 col 28| error: expected ‘;’ before ‘return’                  
||     printf("Hello, World!\n")
||                             ^
||                             ;
||     return 0;
||     ~~~~~~
make: *** [Makefile|7| run-hello] Error 1
```

> To close the quickfix window we use the `:cclose` command.

Now pay attention to this line.

```
hello.c|4 col 28| error: expected ‘;’ before ‘return’
```

In here vim is telling us where is the error, it's showing the file name, the line and the column. Right now what we should do is fix the bug and try to compile again. For the most part this is the workflow that we want. But if you do have more than one item in the quickfix list you could navigate between them using the commands `:cnext` and `:cprev`, to go forward and backwards in the quickfix list.

This is nice and all but what happens if we don't use `gcc`? What if we use `nodejs`? Could vim handle it? Yes, with some help.

### errorformat

If you start using other compilers or interpreters you'll notice that vim can't read properly all the error messages they give you. To overcome this vim offers an option called `errorformat`, a variable that can store the "shape" of an error message, this way vim can recognize it when they appear in the quickfix list.

To test this thing let's try make vim read the errors `node` shows us. Start by creating a file called `greeting.js` and make a simple "hello world" that has an error.

```js
console.log(greeting);
const greeting = 'Hello, World!';
```

Now in our `Makefile` let's add another task.

```make
run-greeting:
  node ./greeting.js
```

If we try to run `:make --silent run-greeting` we'll get this.

```
console.log(greeting);
            ^

ReferenceError: Cannot access 'greeting' before initialization
    at Object.<anonymous> (/tmp/test/greeting.js:1:13)
    at Module._compile (internal/modules/cjs/loader.js:1063:30)
    at Object.Module._extensions..js (internal/modules/cjs/loader.js:1092:10)
    at Module.load (internal/modules/cjs/loader.js:928:32)
    at Function.Module._load (internal/modules/cjs/loader.js:769:14)
    at Function.executeUserEntryPoint [as runMain] (internal/modules/run_main.js:72:12)
    at internal/main/run_main_module.js:17:47
make: *** [Makefile:4: run-greeting] Error 1
```

vim will try to take us to the location of the error but it is not likely to succeed. In my case, it tried to take me to a file it doesn't exists.

To fix this we need to tell vim how to read these messages. I'm going to show one way of doing it.

```vim
set errorformat=%E%.%#ReferenceError:\ %m,%Z%.%#%at\ Object.<anonymous>\ (%f:%l:%c)
```

In here we specify the "shape" of each line in the error message or at least the ones we care about. Each line has its own format and must be separated by a coma. That means...

```
%E%.%#ReferenceError:\ %m
```

And

```
%Z%.%#%at\ Object.<anonymous>\ (%f:%l:%c)
```

Are two expressions that tell vim how to read the error message. We are telling vim where it can find the type of the error (`ReferenceError`) and where is the location data of the error. In our example those two things are in separate lines so we must have these expressions separated by a coma.

If we want to improve readability we could also try to write it this way.

```vim
set errorformat=%E%.%#ReferenceError:\ %m
set errorformat+=%Z%.%#%at\ Object.<anonymous>\ (%f:%l:%c)
```

When we do it this way we don't need to put the coma at the end. But we still need a `\` before each special character (like a blank space) so there is no conflict between vim's syntax and the error format. If you find that annoying you could try another way.

```vim
let &errorformat = 
  \ '%E%.%#ReferenceError: %m,' .
  \ '%Z%.%#at Object.<anonymous> (%f:%l:%c)'
```

When we use `let` we have the oportunity to use strings to write our formats, 
avoiding any sort of conflict between vim's syntax and the `errorformat`. To further improve readability I have every expression in its own line, taking advantage of the `.` operator to concat these strings.

Now if we try to run `:make --silent run-greeting` vim will take us to the right place, which is where `node` says the error is. The quickfix should show us this.

```
|| /tmp/test/greeting.js:1
|| console.log(greeting);
||             ^
|| 
greeting.js|1 col 13 error| Cannot access 'greeting' before initialization
||     at Module._compile (internal/modules/cjs/loader.js:1063:30)
||     at Object.Module._extensions..js (internal/modules/cjs/loader.js:1092:10)
||     at Module.load (internal/modules/cjs/loader.js:928:32)
||     at Function.Module._load (internal/modules/cjs/loader.js:769:14)
||     at Function.executeUserEntryPoint [as runMain] (internal/modules/run_main.js:72:12)
||     at internal/main/run_main_module.js:17:47
|| make: *** [Makefile:4: run-greeting] Error 1
```

Now the `ReferenceError` is no longer on the quickfix list, neither the line that was below. Those has been replaced with this.

```
greeting.js|1 col 13 error| Cannot access 'greeting' before initialization
```

If vim shows that, it means the `errorformat` worked.

We have a bit of an issue, those two expressions will only work with a `ReferenceError` type. You probably need more than that in your day to day workflow. We should really have something like this.

```vim
let &errorformat = 
  \ '%E%.%#AssertionError %m,' .
  \ '%E%.%#TypeError: %m,' .
  \ '%E%.%#ReferenceError: %m,' .
  \ '%E%.%#SyntaxError: %m,' .
  \ '%E%.%#RangeError: %m,' .
  \ '%Z%.%#at Object.<anonymous> (%f:%l:%c),' .
  \ '%-G%.%#'
```

#### Special tokens

That's a lot weird stuff, stuff you might want to know about. Let's dive a little bit into the syntax I use in that last example.

- `%f`: Is the filepath where the error was found.

- `%l`: Is the line number where the error was found.

- `%c`: Is the column number where the error was found.

- `%m`: Is the error message. In our example we use it to capture the text that's right after the type of the error.

- `%E`: Tells vim this is the beginning of a multi-line error message. The letter `E` means this is an error. There other types of messages like warnings (`%W`), informative (`%I`) or general purpose (`%G`).

- `%Z`: Means is the end of the message. Specifically, that is the last line of the message.

- `%.%#`: This one is a wildcard, one that matches everything. So, you could read `%Z%.%#at Object.<anonymous> (%f:%l:%c)` like this: The last line of this message (`%Z`) can start with anything (`%.%#`) followed by `at Object.<anonymous>` and in parenthesis you'll find the filepath (`%f`), the line number (`%l`) and the column (`%c`) where the error was found.

- `%-`: Tells vim it should exclude this piece from the quickfix window. `%-G` could be read like "don't include this message." In our example we use `%-G%.%#` which means "ignore everything else." Now, since `%.%#` matches everything, we put this expression last.

If you want to know more about the special tokens the `errorformat` can have run the command `:help errorformat`.

### makeprg

By now you know you can make vim read any type of error but the current configuration is still tied to `make`. It doesn't have to be like that. We can change the command vim calls when we run `:make`.

Let's say we want to use `node`, just `node`, to achieve this we need to change the option `makeprg`.

```vim
set makeprg=node
```

Or we can do.

```vim
let &makeprg = 'node'
```

Now instead of using `:make --silet run-greeting` we can run `:make ./greeting.js` or `:make %` if we are already editing `greeting.js`.

## Search

Jumping to an error is not the only feature of the quickfix list, we can also use it to explore the code we are working on. For this vim has commands like `:grep` and `:vimgrep`, they create a quickfix list with the results of a search.

### vimgrep

With this command we can take advantage of the built-in search engine that 
comes with vim. It is very much like the good old [grep](https://linux.die.net/man/1/grep), but the `:vimgrep` command uses vim's regex engine. Basically `:vimgrep` is what we'll use when we want to search a pattern (a regular expression) in multiple files. This is how you use it.

```vim
:vimgrep /<pattern>/ <files>
```

The `/` in the pattern are not mandatory, but they are useful when your pattern has characters that cause a conflict with vim's syntax. Like in this example.

```vim
:vimgrep /create table/ db/**/*.sql
```

Here we are searching the pattern `create table` in a folder called `db`. And we are searching only in the files that end with `.sql` extension

These delimeters we use in the search pattern don't have to be `/`. They could be anything that vim doesn't consider to be an "identifier". Find out more about identifiers in the documentation, using the command `:help isident`. This is specially useful when our search pattern already has a `/`. Imagine we are searching for a path in our code, we could write our search like this.

```
:vimgrep #/home/user/code# scripts/*.sh
```

But what happens when we want to ignore a whole directory in our search? There are several solutions. For starters we could set the option `wildignore`. Say we want to ignore a `cache` and `tmp` directories, we could set `wildignore` to something like this.

```vim
:set wildignore=*/cache/*,*/tmp/*
```

As far as I can tell `wildignore` tells vim the paths that should be excluded when doing a path expansion. For example, if we use this pattern `**/*.js` vim will exclude any directory that has `/cache/` or `/tmp/` anywhere in its path. So `:vimgrep` will not search these directories because it will never receive them as arguments. This means `wildignore` can affect other commands, and not just `:vimgrep`. 

Some *stackoverflow* questions suggest this method may not work all the time. In that case we can try to create the argument list with a backtick expression, these will let you call an external command with your shell. We could for example search only on files tracked by git, like this.

```vim
:vimgrep /function/ `git ls-files`
```

In this case `git ls-files` will give us a list file path which will then be processed by `:vimgrep`. Isn't that cool? The best part is that as long as `:vimgrep` gets a valid a file list everything will work as expected.

### grep

And then there is the `:grep` command. This is vim's way of integrating with the search utility `grep`. This command works almost like `:vimgrep` but this time we need to use a "syntax" that is compatible with `grep`. Take this example.

```vim
:grep -r "create table" db/**/*.sql
```

Notice I'm not using `/` as delimeters, also I'm adding the `-r` flag (to enable a recursive search), this is because vim calls `grep` in a non-interactive shell and gives all the arguments to `grep` as is.

But now the question is "when should we use `:grep` instead of `:vimgrep`?" Turns out `:grep` is much faster and efficient than `:vimgrep`. So, `:grep` would be the better choice if your search involves lots of files.

Okay, that's fine. What about the opposite? What advantage `:vimgrep` has over `:grep`? Not much I'd say. If you're more familiar with vim's regular expressions maybe that would be a reason to choose `:vimgrep`. `:vimgrep` also works fine on all platforms, since it's all done inside vim.

Cross-platform support. That's a problem with `:grep`, how does vim solve it? Well, in the same way `:make` does. We can configure the external command vim calls with the variable `grepprg`. Say that instead of using `grep` we want to use [ripgrep](https://github.com/BurntSushi/ripgrep), in order to do that we add this to our config.

```vim
set grepprg=rg\ --vimgrep\ --smart-case\ --follow
```

With that in place vim will use `rg` with all those arguments included when we invoke `:grep`.

### Special guess: FZF

The last external search tool I'll mention is [fzf.vim](https://github.com/junegunn/fzf.vim). This is a plugin that provides an interface where we can execute an interactive search. I won't get into any details here. Just going to tell you something I found out long after I started using it.

Turns out we can populate the quickfix list using FZF, specifically with the results of commands like `:Rg` or `:Ag`. After doing a search you'll have all the matches inside a list ready to be filtered, it's here when you can select an item using `tab` or select all using `Alt + a`, then press `enter`. After this the quickfix list will have every item you selected. This little feature is very useful when you want to execute a certain command only on specific parts of your code using `:cdo`.

## Replace

Speaking of `:cdo`, let me show you one cool thing we can do with it: search and replace in multiple files. If you ever wondered how to do this in vim, the answer is the quickfix list and the `:cdo` command.

In a simple case where all we want to do is replace a known pattern this is what we do:

* Step 1:

Use our favorite search command `:grep`, `:vimgrep` or anything that can populate the quickfix list.

* Step 2 (optional):

Open the quickfix window using `:copen`

* Step 3:

Use `:cdo` to execute a command on every item in the quickfix list. In our case what we want to do is replace the pattern, which we can do using this syntax `s/{pattern}/{replacement}`.

Say that our quickfix list is filled with the results of this search.

```vim
:vimgrep node **/*.js
```

Once that is done we can replace the word `node` with `deno` using this command.

```vim
:cdo s/node/deno/ | update
```

With it vim will run the command `s/node/deno | update` on every item in the quickfix list. We take advantage of the fact `:cdo` can execute any valid vim command and actually do two things, we replace the pattern `node` and save the changes to the file.

### A more advanced use case

Now let's take it one step further, suppose we want to replace some pattern but before doing anything we want filter the results so we only change some parts of our code. Basically we don't want to replace all the matches of a search. How do we do it? One way would be changing the quickfix list so it only has the items we want to replace.

This process will take a bit of effort. First, we need to tell vim how to read its own quickfix list, so we can create modified versions of other quickfix lists. To achieve our goal we need to add a pattern to the `errorformat` option. So in your `.vimrc` you should have something like this.

```vim
set errorformat+=%f\|%l\ col\ %c\|%m
```

We are one step closer but it's still not enough, vim will not let us change the quickfix list. For this we need to be able to write to the buffer where the quickfix list is. We need to run this command.

```vim
:set modifiable
```

Once we do that we can delete the items we want, but we can't run `:cdo` just yet. We need to save the changes we've made with this command. 

```vim
:cgetbuffer
```

Next, just to be on the safe side, make sure you're going to perform the actions on the correct version of the quickfix list. Run, `:cclose` and `:copen`.

Finally you can execute the substitute command.

Here is a demo of the whole process.

{{ asciinema(id="385145") }}

> See in [asciinema](https://asciinema.org/a/385145).

## Improving the experience

As you might have noticed the quickfix list is not the most intuitive thing in the world. But we can make it better. I can give you a few suggestions you can add to your `.vimrc`.

* A better grep.

First thing you might want to do is make vim open the quickfix window after doing a search. Lucky for us the official documentation offers something we can use:

```vim
command! -nargs=+ Grep execute 'silent grep! <args>' | copen
```

I think in this case you could change `grep` with `vimgrep` if you wanted to. The important thing is, with this now you could use `:Grep` (with capital `G`) to make your search.

* Navigating throught the results.

We could "navigate" throught every item in the quickfix list without even opening the quickfix window with this shortcuts.

```vim
" Go to the previous location
nnoremap [q :cprev<CR>

" Go to the next location
nnoremap ]q :cnext<CR>
```

* Manage your window.

If your going to use the quickfix window you better have some keybindings to show and hide it easily.

```vim
" Show the quickfix window
nnoremap <Leader>co :copen<CR>

" Hide the quickfix window
nnoremap <Leader>cc :cclose<CR>
```

* The errorformat.

Let's make sure vim can always read the format on the quickfix list when we want to update it.

```vim
augroup quickfix_group
  autocmd!
  autocmd filetype qf setlocal errorformat+=%f\|%l\ col\ %c\|%m
augroup END
```

* Keybindings

We'll also need some keybindings that only work on the quickfix window. You know, so the "advance use case" won't be so tedious.

```vim
function! QuickfixMapping()
  " Go to the previous location and stay in the quickfix window
  nnoremap <buffer> K :cprev<CR>zz<C-w>w

  " Go to the next location and stay in the quickfix window
  nnoremap <buffer> J :cnext<CR>zz<C-w>w

  " Make the quickfix list modifiable
  nnoremap <buffer> <leader>u :set modifiable<CR>

  " Save the changes in the quickfix window
  nnoremap <buffer> <leader>w :cgetbuffer<CR>:cclose<CR>:copen<CR>

  " Begin the search and replace
  nnoremap <buffer> <leader>r :cdo s/// \| update<C-Left><C-Left><Left><Left><Left>
endfunction

augroup quickfix_group
    autocmd!
    autocmd filetype qf call QuickfixMapping()
augroup END
```

* Now everything put together.

```vim
" :Grep - search and then open the window
command! -nargs=+ Grep execute 'silent grep! <args>' | copen

" Go to the previous location
nnoremap [q :cprev<CR>

" Go to the next location
nnoremap ]q :cnext<CR>

" Show the quickfix window
nnoremap <Leader>co :copen<CR>

" Hide the quickfix window
nnoremap <Leader>cc :cclose<CR>

function! QuickfixMapping()
  " Go to the previous location and stay in the quickfix window
  nnoremap <buffer> K :cprev<CR>zz<C-w>w

  " Go to the next location and stay in the quickfix window
  nnoremap <buffer> J :cnext<CR>zz<C-w>w

  " Make the quickfix list modifiable
  nnoremap <buffer> <leader>u :set modifiable<CR>

  " Save the changes in the quickfix window
  nnoremap <buffer> <leader>w :cgetbuffer<CR>:cclose<CR>:copen<CR>

  " Begin the search and replace
  nnoremap <buffer> <leader>r :cdo s/// \| update<C-Left><C-Left><Left><Left><Left>
endfunction

augroup quickfix_group
  autocmd!
  
  " Use custom keybindings
  autocmd filetype qf call QuickfixMapping()
  
  " Add the errorformat to be able to update the quickfix list
  autocmd filetype qf setlocal errorformat+=%f\|%l\ col\ %c\|%m
augroup END
```

### Plugins

If you know how and you're willing to install some plugins I'd recommend these:

* [vim-qf](https://github.com/romainl/vim-qf)

This one sets some sane defaults to the behaviour of the quickfix window. For example, it can open the quickfix window after calling `:grep`, `:vimgrep` and even `:vimgrep` without having to create new commands. Things don't end there, it also offers some functions we can bind.

* Toggle the quickfix window.

```vim
nmap <Leader>cc <Plug>(qf_qf_toggle)
```

* Navigating throught results

```vim
" Go to previous location
nmap [q <Plug>(qf_qf_previous)zz

" Go to next location
nmap ]q <Plug>(qf_qf_next)zz

function! QuickfixMapping()
  " Go to the previous location and stay in the quickfix window
  nmap <buffer> K <Plug>(qf_qf_previous)zz<C-w>w

  " Go to the next location and stay in the quickfix window
  nmap <buffer> J <Plug>(qf_qf_next)zz<C-w>w
endfunction

augroup quickfix_group
    autocmd!
    autocmd filetype qf call QuickfixMapping()
augroup END
```

The difference between these command and the built-in vim commands is, the plugin commands will not throw an error when we reach the end of the list. Meaning that if we are on last location of the list pressing `]q` will take us to the first item in the quickfix list.

* [quickfix-reflector](https://github.com/stefandtw/quickfix-reflector.vim)

This plugin makes everything I told you in the "advanced use case" section be useless. Once is installed the buffer in the quickfix window acts like a normal buffer. On top of that, every change we make is "reflected" on the actual file.

Remember the example I showed in the demo. Say we have this on the quickfix list.

```
./test dir/a-file.txt|1 col 11| nnoremap <leader>f :FZF
./test dir/a-file.txt|2 col 11| nnoremap <leader>ff :FZF<CR>
./test dir/a-file.txt|3 col 11| nnoremap <leader>fh :History<CR>
./test dir 2/another-file.txt|1 col 11| nnoremap <leader>? :Maps<CR>
./test dir 2/another-file.txt|2 col 11| nnoremap <leader>bb :Buffers<CR>
```

Now say I modify the list, delete the first and fourth item using `dd`.

```diff
- ./test dir/a-file.txt|1 col 11| nnoremap <leader>f :FZF
+ ./test dir/a-file.txt|1 col 11| nnoremap <AAA>f :FZF
  ./test dir/a-file.txt|2 col 11| nnoremap <leader>ff :FZF<CR>
  ./test dir/a-file.txt|3 col 11| nnoremap <leader>fh :History<CR>
- ./test dir 2/another-file.txt|1 col 11| nnoremap <leader>? :Maps<CR>
+ ./test dir 2/another-file.txt|1 col 11| nnoremap <BBB>? :Maps<CR>
  ./test dir 2/another-file.txt|2 col 11| nnoremap <leader>bb :Buffers<CR>
```

If I save these changes with the `:write` command (or the short version `:w`) they will take effect on their respective files. This gives us great power and flexibility because now the changes we can make are only limited by our knowledge of vim.

Any trick you can think that can modify chunks of code should work flawlessly with this plugin. For example if we want to make the same thing I did in the demo we would do it like this:

* Delete the lines we don't want to change

```
./test dir/a-file.txt|1 col 11| nnoremap <leader>f :FZF
./test dir/a-file.txt|3 col 11| nnoremap <leader>fh :History<CR>
./test dir 2/another-file.txt|1 col 11| nnoremap <leader>? :Maps<CR>
```

* Use the "regular" substitution command.

```vim
:%s/leader/localleader/g
```

After that quickfix list should be on this state.

```
./test dir/a-file.txt|1 col 11| nnoremap <localleader>f :FZF
./test dir/a-file.txt|3 col 11| nnoremap <localleader>fh :History<CR>
./test dir 2/another-file.txt|1 col 11| nnoremap <localleader>? :Maps<CR>
```

* Save the changes.

```vim
:write
```

And that's it.

## Conclusion

We got to know what is the quickfix list and its most common use cases.

Now we know we can use `:make` to call any compiler or external command that can run our code and give us error messages. We also have the tools to "teach" vim how to read an error message and put all the information we need in the quickfix list.

We learned that we can use `:vimgrep` and `:grep` to search patterns in multiple files in our project. We had the chance to explore a few methods to search and replace text, with some simple cases and another one slightly more complex. With these examples we learned how to replace text with and without plugins.

Lastly we learned about some options and commands we can use in our `.vimrc` to improve the user experience when we use the quickfix list.

## Sources

- [:help quickfix](https://vimhelp.org/quickfix.txt.html)
- [How to edit the vim quickfix list](https://www.reddit.com/r/vim/comments/7dv9as/how_to_edit_the_vim_quickfix_list/)

