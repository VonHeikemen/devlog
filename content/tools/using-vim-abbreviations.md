+++
title = "Using Vim's abbreviations" 
description = "Exploring some cool use cases for vim abbreviations"
date = 2020-11-03
lang = "en"
[taxonomies]
tags = ["vim", "shell"]
+++

One of the things vim does extremely well is automation. This time we'll explore a feature called "abbreviation" and how can we use it to automate things in insert mode (you can use it in other modes but I'll focus on insert mode). We will go from the most basic use case to some that are more elaborated. But don't worry, we'll take one step at a time. Let's start.

## Step 0

Abbreviations can be assigned with the command `:ab[breviate]`. Since I only want to focus on insert mode I'll be using `:iabbrev`. This is how you would use it.

```
:iabbrev [<expr>] [<buffer>] {abbreviation} {expansion}
```

In here anything that's inside a `[]` is optional. `<expr>` means that you could use a vimscript expression to create the expansion. `<buffer>` means that it only applies to the current buffer. `abbreviation` is the thing you type and that will be replaced by the `expansion`.

You can use this command in [autocommands](https://learnvimscriptthehardway.stevelosh.com/chapters/12.html) to make file specific abbreviations automatically. If you put something like this in your `.vimrc`.

```
:autocmd FileType html,javascript,typescript,vue
  \ :iabbrev <buffer> some-abbreviation some-expansion
```

It would make `some-abbreviation` available to you in `html`, `javascript`, `typescript` and `vue` files. With this knowledge you can now start creating some.

## Hello wordl

Typos. Don't you hate them? Writing `heigth` and then find out something is not working in your code can be very annoying. Well, no more. Now you have the tools (I mean the command) to fix that automatically.

```
:iabbrev heigth height
```

Now when you write `heigth` and press the `space` key it will become `height`.

![fixing a typo](https://res.cloudinary.com/vonheikemen/image/upload/v1604437302/devlog/using-vim-abbreviations/1-typos_03-11-2020_16-51.gif)

But the fun doesn't stop there.

## It's not just for letters

Just like in regular mappings (think `:map` command) we can use special sequences like `<CR>` (the `enter` key), `<Up>`, `<Tab>` and many more. This gives us the power to move around the cursor and create more elaborate snippets.

One of the most common things I do in a javascript file is putting a `console.log()`, and it's not just me, it is so common that some text editors made for developers have a built in shortcut for that. Vim is not such editor, but we can make one.

```
:iabbrev <buffer> con@ console.log();<Left><Left>
```

![a simple snippet](https://res.cloudinary.com/vonheikemen/image/upload/v1604438822/devlog/using-vim-abbreviations/2-simple-snippet_03-11-2020_17-26.gif)

The `@` sign in this case is not important, it's just a silly convention I follow on my vim config, it's just there to make it clear this will be expanded into something else. I use `<buffer>` because I don't want this to be available in every buffer, just the ones where I can use valid javascript syntax. I would use this in an autocommand like the one I showed you before.

You'll notice that there is a space between the parenthesis in the `log` function, that's because we chose to "expand it" with the `space` key. Vim will replace the abbreviation after we press a "non-keyword character" and that character will be inserted after the expansion is done. Basically all the letters and a few special characters are considered "keyword characters" and anything that isn't in that set will begin the expansion. Might want to check `:help 'iskeyword'` for more details.

Anyway, if you don't want that extra space you can begin the expansion with the sequence `<C-]>` that is `control + closing bracket`.

## Wait... did someone said snippets?

Yes, yes I did. With this feature we can create some complex snippets like the ones you find in a fancy IDE. Are you in a javascript file and want to create an immediately invoked function expression (iife)? No problem.

```
:iabbrev <buffer> iife@ (async function() {})();<Left><Left><Left><Left><Left><CR><CR><Up>
```

![iife snippet](https://res.cloudinary.com/vonheikemen/image/upload/v1604442411/devlog/using-vim-abbreviations/3-another-snippet_03-11-2020_18-25.gif)

You know what? Scratch that. There is a problem. All those `<Left>`s are hurting me. Can you just imaging making a snippet for a `switch/case`? Writing it in that style would be awful, it would work but it'll be awful. Don't worry there is an easy solution for this.

### Escaping insert mode

Since we're basically automating keystrokes we can do a very funny thing, we can "press" `<Esc>` to go to normal mode and take advantage of all the vim goodness we can do in that mode. So, we could rewrite it like this.

```
:iabbrev <buffer> iife@ (async function() {})();<Esc>4hi<CR><CR><Up>
```

We got the same snippet but now is less awful. We replaced the five `<Left>`s with `<Esc>4hi`, a bit more cryptical but shorter. Although, if you've been using vim for a while you should know by now what `4hi` would do if you press that yourself. In fact, since we are just entering in insert mode to make one command (`4h`) and then going back we could simplify it a little bit.

```
:iabbrev <buffer> iife@ (async function() {})();<C-o>4h<CR><CR><Up>
```

Now we use `<C-o>` (`control + o`) to go to normal mode, make one command and immediately go back to insert mode.

## Let's get real fancy

How about we go one step further. Let's make a snippet that can put the cursor in a convenient location but without messing around to much with `<Left>`s and `<Right>`s.

We could make yet another version of the `iife@` but I want to give you more ideas. Let's do a "traditional" `for` loop. You know, like this one.

```js
for(let i = 0; i < {PLACEHOLDER}; i++) {

}
```

At the end of the expansion we want the cursor to be where the `{PLACEHOLDER}` is, this time we don't want any `<Left>`s or `<Right>`s, we just want to get there. How do we do it? With a search. Yes, we can also make a search with `/` or `?`.

```
:iabbrev <buffer> forii@ for(let i = 0; i <z; i++) {<CR><CR>}<Esc>?z<CR>xi
```

Notice the `z`? That's my placeholder. After I type everything, I hit the `<Esc>`, press `?` (backward search), look for a `z`, hit enter to submit the search (the `<CR>` part) and that will take me where I want, finally the `xi` part deletes the `z` and lets me go back to insert mode.

![a nice for loop](https://res.cloudinary.com/vonheikemen/image/upload/v1604449379/devlog/using-vim-abbreviations/4-with-placeholder_03-11-2020_20-17.gif)

## More Powerrrr

What else can we do? An argument. Using a word like a function argument. Oooh, that would be cool. Got the perfect use case for this. Let's do it.

Do you know about [vue-vscode-snippets](https://github.com/sdras/vue-vscode-snippets)? No. That's okay, I'll tell you. They are vue snippets for vscode. One of those snippets is called `vimport-export`, it creates this boilerplate.

```js
import Name from '@/components/Name.vue';

export default {
  component: {
    Name,
  }
};
```

The cool thing about it is that it uses the multiple cursor feature of vscode to let you replace `Name` with the name of your component. I'll be honest, I'm jealous, I want multiple cursor in vim. I know there is a plugin, but I want it to be a native feature. Anyway, we don't need no stinking multiple cursors, we literally just need to put the same word in multiple places.

This is how is going to be: I put a word (the name of my component) and then the abbreviation. Like this.

```
MyComponent vcomp@
```

What's going to happen is, I'm going to put `MyComponent` in a register and just paste it anywhere I need to. You ready? Let's start with the import part.

```
:iabbrev <buffer> vcomp@ <Esc>bvediimport <C-o>P from '@/components/<C-o>P.vue';
```

The key component of this is `<Esc>bved`. `<Esc>` will take us right in normal mode, `b` will take the cursor back to the beginning of the previous word (the component name). `ved` will select the entire word and delete it. Thanks to the way `d` works we will have the component available in a register just waiting.

Now that's one part of the problem, and to be fair we could easily make the other part. Let's take it one step further. The common use case for this abbreviation would be to register a component inside another component, in this case we already have an `export default` object, and what we want to do is add it to the `components` property. Basically we want to automate this process:

![register component](https://res.cloudinary.com/vonheikemen/image/upload/v1604454412/devlog/using-vim-abbreviations/5-vexport_03-11-2020_21-40.gif)

The second part starts with `<Esc>` key to get to normal mode, then we use `/` to search for `components: {`, we submit the search and then we enter insert mode in the line below using `o`, we use `<Tab>` to get to the right spot then go paste the component name (which we have in the register) and put a comma. We could stop there but I want to preserve the order of my imports. Next, we take the whole line where the component is, so we hit `<Esc>dd`. Then go up and to the end of the line, that will be `k$`. To go the bottom of the nested object we press `%`, once we're there we need to create a newline so we hit `o` again. This is the tricky part, we hit `<Esc>` to go to normal mode, paste the component in this place using `P` (capital `p`). That's good but now we have a new empty line, we delete that with `jdd`. Finally we go back to the line of our import with `2<C-o>` (we need to jump twice so we need that 2). So... we need this:

```
<Esc>/components: {<CR>o<Tab><Esc>pa,<Esc>ddk$%ko<Esc>Pjdd2<C-o>a
```

Everything together looks like this.

```
:iabbrev <buffer> vcomp@ <Esc>bvediimport <C-o>P from '@/components/<C-o>P.vue';<Esc>/components: {<CR>o<Tab><Esc>pa,<Esc>ddk$%ko<Esc>Pjdd2<C-o>a
```

![register component complete](https://res.cloudinary.com/vonheikemen/image/upload/v1604456390/devlog/using-vim-abbreviations/6-vcomp_03-11-2020_22-15.gif)

If you want to have some fun try to make an abbreviation that takes this.

```js
Example vvcomp@

export default {

};
```

And turn it into this.

```js
import Example from '@/components/Example.vue';

export default {
  components: {
    Example,
  },
};
```

### The cherry on top

I shit you not. Abbreviations work inside macros. Try recording this macro and play it multiple times.

```
0ea vcomp@^]^]j
```

`^]` is the `<Esc>` key.

![macros](https://res.cloudinary.com/vonheikemen/image/upload/v1604457566/devlog/using-vim-abbreviations/7-macro_03-11-2020_22-34.gif)

## Conclusion

That's it. That's all I got for you today. I you hope found something useful or at the very least learned something funny you'll probably never use.

If you need any more ideas check out [my snippets](https://github.com/VonHeikemen/dotfiles/blob/master/my-configs/vim/snippets.vim). If you have any interesting ideas, let me know.

## Sources
- [:help abbreviations](https://vimhelp.org/map.txt.html#Abbreviations)
- [Learn Vimscript the Hard Way: Abbreviations](https://learnvimscriptthehardway.stevelosh.com/chapters/08.html)

