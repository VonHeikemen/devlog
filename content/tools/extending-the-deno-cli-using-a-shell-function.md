+++
title = "Extending the deno cli using a shell function" 
description = "Improving the deno cli with a little bit of magic"
date = 2021-04-18
lang = "en"
draft = true
[taxonomies]
tags = ["shell", "deno", "todayilearned"]
+++

I've been using `deno` more and more these days and I must tell you that are still a couple that bug me. But I did manage to solve some of those problems by "adding" sub-commands to cli. No, it's not dark magic, it's just a trick that I learned a while ago and that I'm going to share with you today.

## What do we need for this?

In theory any shell that allows you to declare functions should be enough. In this post I'll be showing you the examples using a POSIX compliant syntax, so they should work on shells like `bash`, `zsh` or `ash`.

## What do we want to achieve?

I would love for `deno` to have some built-in equivalent of `npm scripts`, but it doesn't seem like is going to happen any time soon. I wish I could do something like this.

```sh
deno start hello world
```

It would also be nice if I could initialize those scripts, because they have to live somewhere.

```sh
deno init
```

But as you know those sub-commands don't exists and that's what we are going to fix today (kinda).

## Let us begin

The first step is to locate that one file you know your shell always execute. In `zsh` is `~/.zshrc`. `bash` has `~/.bashrc`. Those two are the only ones I know, if you use any other shell try to find in its documentation the equivalent of that.

### The definition

You ready now? Okay, now in that file you're going to write this.

```sh
deno()
{
  echo "Hello"
}
```

Now if you restart your shell (or "reload" your configuration) and try to call `deno` you should get `hello`. Isn't that nice? Awesome, but now we have created a problem, we can use `deno`. But fear not, we are on a good path.

### Come back to me deno

If we want to call the `deno` **command** we need to use the command known as `command`.

```sh
command deno --version
```

And that you get us all the info `deno` has about itself.

```
deno x.x.x (release, x86_64-unknown-linux-gnu)
v8 x
typescript x.x
```

With this new knowledge we can solve our problem. Let's put it in the function.

```sh
deno()
{
  echo "You're using"
  command deno --version
}
```

Good, and when we call it we should get this.

```
$ deno

You're using
deno x.x.x (release, x86_64-unknown-linux-gnu)
v8 x
typescript x.x
```

### Going back to normal

We know how to call the `deno` command but we still need to make our function behave like the "real `deno`." We are going to do this in a way that can give us the oportunity to extend its behavior later on. For this we are going to use a `case` statement. 

```sh
deno()
{
  local cmd=$1; shift;

  case "$cmd" in
    *)
      command deno $cmd $@
    ;;
  esac
}
```

The first thing this function does is assign the first parameter (`$1`) to the variabl  `cmd` and then remove it from the arguments list (`$@`). After that we compare `cmd` with a pattern. For now the only pattern we have is `*` which is a wildcard that matches everything, is basically our default case. Let's make a test.

```sh
deno --version
```

We good. But notice that if we try to use `deno` without any arguments it'll give us an error. To fix that we are going to check if the first parameter is empty, and if it is we just call `deno`.

```sh
deno()
{
  if [ -z "$1" ];then
    command deno
    return
  fi

  local cmd=$1; shift;

  case "$cmd" in
    *)
      command deno $cmd $@
    ;;
  esac
}
```

### Calling all sub-commands

We are right were we want to be, we can finally start adding our own sub-commands. But first let's make a sanity check.

```diff
  deno()
  {
    if [ -z "$1" ];then
      command deno
      return
    fi

    local cmd=$1; shift;

    case "$cmd" in
+     hello)
+       command deno eval "console.log('deno says hello')"
+     ;;
      *)
        command deno $cmd $@
      ;;
    esac
  }
```

Let's call the `hello` command.

```
$ deno hello

deno says hello
```

So we know for sure this thing works.

### deno scripts

We can work on our ad-hoc replacement for npm scripts. Let's start at the beginning, `start` command of course. Is a convention to use this command to start an application or project.

```diff
  deno()
  {
    if [ -z "$1" ];then
      command deno
      return
    fi

    local cmd=$1; shift;

    case "$cmd" in
+     start)
+       command deno run --allow-run ./Taskfile.js start $@
+     ;;
      *)
        command deno $cmd $@
      ;;
    esac
  }
```

This command will execute a file called `Taskfile.js` using `deno`. The `--allow-run` will let us use `Deno.run` in our code to call external commands. It's test time.

Let's make that `Taskfile.js`.

```js
const cmd = ['echo', 'Taskfile: ', ...Deno.args];
Deno.run({ cmd });
```

Now we use `start`.

```
$ deno start hello

Taskfile start hello
```

Just great. The next step will be to create a "smarter" `Taskfile.js`, one that can call different tasks. I've done something like this in the past (check out the details of it in [here](https://dev.to/vonheikemen/a-simple-way-to-replace-npm-scripts-in-deno-4j0g)), so I'm going to show you the code I would use.

```js
const entrypoint = "./src/main.js";

run(Deno.args, {
  start(...args) {
    exec(["deno", "run", entrypoint, ...args]);
  },
  list() {
    console.log('Available tasks: ');
    Object.keys(this).forEach((k) => console.log(`* ${k}`));
  },
});

function run([name, ...args], tasks) {
  if(tasks[name]) {
    tasks[name](...args);
  } else {
    console.log(`Task "${name}" not found\n`);
    tasks.list();
  }
}

async function exec(args) {
  const proc = await Deno.run({ cmd: args }).status();

  if (proc.success == false) {
    Deno.exit(proc.code);
  }

  return proc;
}
```

With this we can execute the file `./src/main.js` using `deno start`. But now we have a problem, we don't want to write all of that every time we create a new project. What we will do is create another command called `init` to copy this template into our project folder.

```diff
  deno()
  {
    if [ -z "$1" ];then
      command deno
      return
    fi

    local cmd=$1; shift;

    case "$cmd" in
      start)
        command deno run --allow-run ./Taskfile.js start $@
      ;;
+     init)
+       cp /path/to/template/Taskfile.js ./
+       echo "Taskfile.js created"
+     ;;
      *)
        command deno $cmd $@
      ;;
    esac
  }
```

Everything is looking good. The last thing we will deal is calling other tasks besides `start`. With `npm` we can call any script we want using `npm run`, the equivalent for us will be called `x`.

```diff
  deno()
  {
    if [ -z "$1" ];then
      command deno
      return
    fi

    local cmd=$1; shift;

    case "$cmd" in
+     x)
+       command deno run --allow-run ./Taskfile.js $@
+     ;;
      start)
        command deno run --allow-run ./Taskfile.js start $@
      ;;
      init)
        cp /path/to/template/Taskfile.js ./
        echo "Taskfile.js created"
      ;;
      *)
        command deno $cmd $@
      ;;
    esac
  }
```

We are all set. If we had a "task" called `test:api` we can invoke that using this. 

```sh
deno x test:api
```

### A little extra convenience

One more thing, I would like to have a way of calling a specific script and be able to call all my most used libraries without using URLs. We can do this with the help of [import-map](https://deno.land/manual@v1.9.0/linking_to_external_code/import_maps)s, `.json` files that can bind an "alias" with a URL.

I use one like this.

```json
{
  "imports": {
    "@std/": "https://deno.land/std@0.93.0/",
    "@npm/": "https://jspm.dev/",

    "ansi-colors": "https://jspm.dev/ansi-colors@4.1.1",
    "arg": "https://jspm.dev/arg@5.0.0",
    "cheerio": "https://jspm.dev/cheerio@1.0.0-rc.5",
    "exec": "https://deno.land/x/exec@0.0.5/mod.ts",
    "ramda": "https://jspm.dev/ramda@0.27.1",

    "@utils/": "/path/to/deno/utils/"
  }
}
```

`deno` can read this if we provide its path using the `--import-map` flag. Let's add command that uses that.

```diff
  deno()
  {
    if [ -z "$1" ];then
      command deno
      return
    fi

    local cmd=$1; shift;

    case "$cmd" in
      x)
        command deno run --allow-run ./Taskfile.js $@
      ;;
      start)
        command deno run --allow-run ./Taskfile.js start $@
      ;;
      init)
        cp /path/to/template/Taskfile.js ./
        echo "Taskfile.js created"
      ;;
+     s|script)
+       command deno run --import-map="/path/to/deno/import-map.json" $@
+     ;;
      *)
        command deno $cmd $@
      ;;
    esac
  }
```

As with everything else, we need to test this thing. Create a test script with this.

```js
import dayjs from '@npm/dayjs';
import c from 'ansi-colors';

c.enabled = !Deno.noColor;

const date = dayjs().format('{YYYY} MM-DDTHH:mm:ss SSS [Z] A');

console.log(c.green(date));
```

> Might want to specify the library version. Instead of using `@npm/dayjs`, you put `@npm/dayjs@1.10.4`

Now call it.

```
$ deno script ./test.js

Download https://jspm.dev/dayjs@1.10.4
Download https://jspm.dev/npm:dayjs@1.10.4!cjs
{2021} 04-18T11:28:05 929 Z AM
```

Ha! Now I have all I want.

### The final result

After all this process your `deno` function should look like this.

```sh
deno()
{
  if [ -z "$1" ];then
    command deno
    return
  fi

  local cmd=$1; shift

  case "$cmd" in
    x)
      command deno run --allow-run ./Taskfile.js $@
    ;;
    start)
      command deno run --allow-run ./Taskfile.js start $@
    ;;
    init)
      cp /path/to/template/Taskfile.js ./
      echo "Taskfile.js created"
    ;;
    s|script)
      command deno run --import-map="/path/to/deno/import-map.json" $@
    ;;
    *)
      command deno $cmd $@
    ;;
  esac
}
```

## Conclusion

So, what did we do today? We "hid" the `deno` command inside a function in order to be able to add sub-commands that we created. We managed to build an equivalent to `npm run` for `deno` and finally we used import maps to simplify the declaration of our dependencies in a script. Not bad for a day's work.

