+++
title = "Want to use env variables in that config file? Well, you can... kind of"
description = "Meet the envsubst command"
date = 2021-10-20
lang = "en"
[taxonomies]
tags = ["shell", "todayilearned"]
+++

Imagine there is a tool you want to install in every OS you manage and that tool needs a configuration file. There is a good chance a piece of information in that file needs to change according to the type of OS you're using, or maybe it changes according to the context (whether is a personal computer or a workstation... stuff like that). So you always need to make some tweaks to this config. Wouldn't be great if you could use some variables? It turns out you can... kind of. 

## Meet the envsubst command

With `envsubst` we can embed the value of environment variables inside a string that comes from the **standard input**.

We can try it out. In the terminal we can do something like this.

```sh
echo 'there is no place like $HOME' | envsubst
```

If you are using linux you should get this message.

```
there is no place like /home/user
```

So what I'm trying to say is we can cheat. We can use `envsubst` like a preprocessor to create our config files based on some template.

## Real world example

In my case `envsubst` solved a bit of a problem I had with `npm`. In my home folder I have an `.npmrc` file with the following structure.

```
prefix=<some-path>
ignore-scripts=true

init-author=<some-name> (<some-link>)
init-license=MIT
```

As you might have guessed some of these fields can change according to the OS or context. But now that I know about `envsubst` I can have a template in `~/templates/npmrc` and use that as a base for every other machine.

So how do I do? First thing is the template.

```
prefix=$XDG_CONFIG_HOME/npm/packages
ignore-scripts=true

init-author=$USER_NAME ($CONTACT_LINK)
init-license=MIT
```

This is good enough. 

I always like to test things, so next thing is try showing the result on the terminal.

```sh
envsubst < ~/templates/npmrc
```

When we use `<` we are telling our shell we want to take the content of that file and send it to `envsubst` using the standard input. This is known as a file redirection. So, if all our environment variables are set `envsubst` should show this.

```
prefix=/home/user/.config/npm/packages
ignore-scripts=true

init-author=Heiker (https://github.com/VonHeikemen)
init-license=MIT
```

Now to really make this work we need to redirect this thing to `~/.npmrc`.

```sh
envsubst < ~/templates/npmrc > ~/.npmrc
```

Technically we are done but since we are here let's explore another posibility.

I already showed you that our template can come from any command, the only requirement to keep in mind is we need to get the content from the standard input.

So, if we get creative we could put our template in any server (like say github), download it using `curl` and pass that content to `envsubst`.

```sh
curl -s "https://raw.githubusercontent.com/<user>/<repo>/main/npmrc" \
  | envsubst > ~/.npmrc
```

This works like charm. It means we don't even need the template to live in our local environment.

## A cross-platform alternative

One last thing. If you're using linux you probably have this tool in your system. In case you don't, you could try and get this version which was written using `go`.

* [github.com/a8m/envsubst](https://github.com/a8m/envsubst) 

In their [release page](https://github.com/a8m/envsubst/releases) they have prebuilt binaries for windows, mac and linux. 

## Conclusion

We learned about `envsubst` and how we can use it to create a config file based on a template. We learned `envsubst` needs to take the content of the template from the standard input, this opens the posibility for us to get what we need from other files or commands. Lastly, we saw how we can use file redirections to take final result into the config file we want.

## Source

* [man envsubst](https://www.mankier.com/1/envsubst)

