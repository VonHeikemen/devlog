+++
title = "Alias global en ZSH" 
description = "Qué son y cómo podemos usarlos"
date = 2020-07-12
lang = "es"
[taxonomies]
tags = ["zsh", "shell", "hoy-aprendi"]
+++

Si han pasado mucho tiempo escribiendo comandos en la terminal lo más probable es que estén al tanto de la existencia los **alias**. Generalmente cuando se habla de ellos también se habla de una promesa de incrementar la productividad. Lo cierto es que algunas veces los usamos para protegernos de comandos largos con argumentos crípticos y algunas veces sólo los usamos para ahorranos algunos caracteres. Sin importar sus motivos les voy a enseñar a crear otro tipo de alias que tal vez no conocían antes, los alias globales.

## Un breve repaso

El motivo más común por el cual sentimos la necesidad de crear un alias es cuando usamos un comando muy a menudo, es tan frecuente su uso que estamos cansados de escribir la misma cosa una y otra vez. Fíjense en este ejemplo.

```sh
tmux new-session -A -D -s work
```

> ¿No saben lo que hace? No se preocupen, pueden revisar [aquí](https://explainshell.com/explain?cmd=tmux+new-session+-A+-D+-s+work).

¿Pero no sería mejor si pudiera sólo escribir esto?

```sh
ts work
```

Vean toda esa productividad. ¿Pero cómo llegamos del primer comando a este último? Con el comando `alias`.

```js
alias ts='tmux new-session -A -D -s'
```

Pueden escribir eso directamente en su terminal y lo tendrán mientra su sesión en zsh esté activa. Si quieren que sea permanente tendrán que colocarlo en algún lugar de su `.zshrc`.

Noten que omití el argumento `work`. Es porque esa última parte es un nombre que yo puedo elegir. Al dejarlo fuera del alias me permite poner cualquier cosa que yo quiera luego. Así.

```sh
ts pomodoro
```

Todo eso suena bien. ¿Cuál es la trampa? Bueno, este alias sólo funciona cuando es la primera cosa que escribimos en nuestro comando pero falla si lo colocamos en otro lugar. Vamos a seguir con nuestro ejemplo, crearemos otro alias para expandir el comando. 

```sh
alias pomd='gone -e "notify-send -u critical Pomodoro Timeout"'
```

> Este comando [gone](https://github.com/guillaumebreton/gone) es un "reloj pomodoro".

Si intentan ejecutar `ts pomodoro pomd` sólo obtendran este mensaje `[exited]`. Eso es porque `pomd` no tiene ningún significado especial en ese contexto, sólo un argumento normal. ¿Podemos solucionar eso? Sí. ¿Deberíamos? No lo sé pero aún así les mostraré.

## A nivel global

Si un "simple" alias no es suficiente para ustedes entonces lo que deben hacer es agregar el argumento `-g` al comando `alias`.

```sh
alias -g nombre="un-comando"
```

Veamos un buen ejemplo para usar un alias global (el del pomodoro es terrible para eso). En ocasiones tienen un comando que saben produce mucho contenido, en ese caso tal vez quieran usar `less` para inspeccionar mejor toda esa cantidad de texto. Así que haremos una alias global para pasar el resultado de un comando a `less`.

```sh
alias -g L="| less"
```

Ahora pueden probarlo.

```sh
du --max-depth=1 L
```
> [Esto es lo que hace](https://explainshell.com/explain?cmd=du+--max-depth%3D1+%7C+less). Puede tomar unos segundos si tiene muchos archivos en su directorio actual.

¿Vieron? Es conveniente, útil y peligroso al mismo tiempo. Ahora cada vez que una `L` solitaria (sin comillas) aparezca en un comando será reemplazada por `| less`. Eso no siempre es conveniente. Por ejemplo, si intentan eliminar el alias con el comando `unalias` así.

```sh
unalias L
```
> La manera correcta sería esta: `unalias "L"`

En realidad están ejecutando esto.

```sh
unalias | less
```

Para minimizar el riesgo de confusión se aconseja que un alias global se escriba todo en mayúscula, para hacer que se destaque del resto del comando.

Tal vez creas que es una mala idea pero no temas, aún hay esperanza.

## Otro caso de uso

En zsh hay una función llamada `_expand_alias`, esta nos puede ayudar a usar un alias global para un bien mayor.

Okey, si usan el comando `bindkey` sin argumentos les devolverá una lista de cada atajo que tienen disponible en zsh. Deberían poder ver `_expand_alias` en esa lista. Pueden revisar rápidamente de esta manera.

```sh
bindkey | grep expand_alias
```

El resultado debería ser: `"^Xa" _expand_alias`. Eso significa que el atajo es `ctrl+x a`. 

Con este nuevo conocimiento podemos crear un alias para algo que sea muy difícil de recordar (o que simplemente tenga demasiadas letras). La idea es "expandir" el alias antes de ejecutar el comando.

¿Qué sería un buen ejemplo? ¿Recuerdan cómo es que se hace para suprimir los mensajes de error? ¿No? Bueno, ahí lo tienen, ese es su alias.

```sh
alias -g @noerr="2> /dev/null"
```

Ahora pueden escribir algo así.

```sh
mkdir /tmp/this-doesnt-exists/name @noerr
```

Pero antes de ejecutarlo presionan `ctrl+x` y luego `a` y deberían ver cómo se transforma en esto.

```sh
mkdir /tmp/this-doesnt-exists/name 2> /dev/null 
```
> Niños, recuerden que nunca deben ejecutar comandos que vean en internet sin saber qué hacen. [Revisen](https://explainshell.com/explain?cmd=mkdir+%2Ftmp%2Fthis-doesnt-exists%2Fname+2%3E+%2Fdev%2Fnull) siempre, antes de hacer cualquier cosa.

Ahora sí pueden presionar `enter` para ejecutarlo.

## Conclusión

Ya llegamos al final. Ahora saben qué pueden hacer con un alias, qué limitaciones tienen y cómo pueden superarlas, también espero que sepan qué cosas no deberían hacer con ellos.

Si quieren algo de inspiración para crear sus propios alias revisen este [plugin](https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins/common-aliases) de [ohmyzsh](https://github.com/ohmyzsh/ohmyzsh). Les recomiendo que sólo revisen el script y copien sólo lo que encuentren útil.

## Fuente

* [5 Types Of ZSH Aliases You Should Know](https://thorsten-hans.com/5-types-of-zsh-aliases)

