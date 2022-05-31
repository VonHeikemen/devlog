+++
title = "Navega a través del historial de comandos de una manera eficiente"
description = "Vamos a sacarle provecho a toda esa información en el historial"
date = 2021-10-25
lang = "es"
[taxonomies]
tags = ["shell", "zsh"]
[extra]
shared = [
  ["dev.to", "https://dev.to/vonheikemen/navega-a-traves-del-historial-de-comandos-de-una-manera-eficiente-191d"],
  ["Hashnode", "https://vonheikemen.hashnode.dev/navigate-your-command-history-with-ease-es"]
]
+++

Si pasan mucho tiempo en la terminal va a llegar un momento en el que no podrán recordar todos los comandos que usaron durante el día, y ni hablar de los días anteriores. Por suerte para nosotros nuestro "shell" puede almacenar esta información por nosotros. Ahora les voy a enseñar cómo podemos sacar provecho de esta información para mejorar nuestra experiencia en la terminal.

## Comandos viejos y las flechas

Estoy seguro que todos aquí saben que pueden navegar el historial de comandos usando las flechas. Algunos de ustedes saben que pueden usar `ctrl+r` para activar una especie de buscador para revisar comandos viejos. ¿Pero y si les digo que podemos obtener lo mejor de estos dos métodos?

Podemos escribir las primeras letras de un comando y presionar la flecha hacia arriba para empezar a buscar los comandos que comienzen con ese "prefijo".

Imaginen. Tienen los siguientes comandos en su historial.

```
node ./test.js
vi /tmp/text.txt
nvim /tmp/text.txt
vi /tmp/test.js
echo "a string with vi in it"
vi /tmp/other-text.txt
```

Quieren buscar los archivos que han editado con `vi`, escriben `vi` e inmediatamente presionan la flecha hacia arriba. En este caso en pantalla se muestran sólo los comandos que comienzan con `vi`:

```
vi /tmp/text.txt
vi /tmp/test.js
vi /tmp/other-text.txt
```

Sería genial. Basta de presionar 20 veces la flecha hacia arriba para evitar escribir `npm start`. Sólo usen `np` + flecha hacia arriba.

¿Cómo podemos habilitar esta maravilla?

* En bash:

```sh
# Flecha hacia arriba
bind '"\e[A": history-search-backward'

# Flecha hacia abajo
bind '"\e[B": history-search-forward'
```

* En zsh:

```sh
autoload -U up-line-or-beginning-search
autoload -U down-line-or-beginning-search

# Flecha hacia arriba
bindkey "${terminfo[kcuu1]}" up-line-or-beginning-search

# Flecha hacia abajo
bindkey "${terminfo[kcud1]}" down-line-or-beginning-search
```

> Nota: Si usan `oh-my-zsh` esta funcionalidad ya está activada.

Si por alguna razón `terminfo` no está definido intenten esto en la terminal: presionan `ctrl+v` luego una de las flechas. En mi caso obtengo:

* Arriba: `^[OA`
* Abajo: `^[OB`

Entonces para mí esto también funciona.

```sh
bindkey '^[OA' up-line-or-beginning-search
bindkey '^[OB' down-line-or-beginning-search
```

## Búsqueda interactiva

Si el "truco" anterior no los impresiona está bien. Todos tenemos nuestras preferencias. Para realizar una búsqueda interactiva prefiero usar [fzf](https://github.com/junegunn/fzf). `fzf` tiene [unos scripts](https://github.com/junegunn/fzf/tree/master/shell) (key-bindings.*sh) que nos brinda un buscador con una mejor experiencia de usuario que el tradicional `ctrl+r`. Para habilitarlo hay muchas formas (depende de su sistema operativo), así que será mejor que investiguen por su cuenta cómo hacerlo.

## Espacio mágico

En `bash` y `zsh` hay algo llamado "history-expansion". Basícamente es una forma de consultar la data que tenemos en nuestro historial y usarlo en un comando.

Por ejemplo, tenemos que:

* `!!` se usa para indicar que queremos usar el último comando en nuestro historial. Entonces algo como esto es válido `sudo !!`, aquí estamos diciendo "ejecuta el último comando con `sudo`".

* `!$` es el último argumento del último comando del historial. Si el último comando que usamos fue `nano /tmp/test.txt` y luego usamos `cat !$`, `cat` va a mostrar `/tmp/test.txt`.

* `!*` todos los argumentos del último comando del historial.

Hay más variantes y combinaciones pero esos son los que uso con más frecuencia.

En fin, yo prefiero "expandir" estos símbolos antes de ejecutar el comando. No me gustaría ejecutar un comando como `sudo !!` sin saber qué es `!!`. Para esto uso comando que se llama `magic-space`. Al igual que en nuestro primer "truco" este es un comando que debemos vincular a una tecla, por lo general a la tecla de espacio.

* bash

```sh
# Espacio, pero mágico
bind Space:magic-space
```

* zsh

```sh
# Espacio, pero mágico
bindkey ' ' magic-space
```

Luego de esto podremos expandir todos estos símbolos antes de ejecutar el comando. Podríamos escribir `sudo !!` y luego si presionamos espacio veremos cómo `!!` se transforma en el comando que vamos a ejecutar.

## Conclusión

Descubrimos varias maneras mejorar la experiencia de usuario de nuestro "shell" con una serie de trucos. Aprendimos cómo optimizar la búsqueda de comandos con las flechas. Descubrimos `fzf` como una alternativa al `ctrl+r`. Aprendimos sobre la "expansión de historial" (history-expansion) para consultar el historial y usar el resultado en el comando que estamos creando. Utilizar estos pequeños trucos en su rutina verán que con el tiempo la experiencia usando la terminal se vuelve un poco más placentera.

