+++
title = "La ilusión de un inicio rápido en vim"
description = "Mejorando el tiempo de inicio de vim con un pequeño 'truco'"
date = 2021-08-22
lang = "es"
[taxonomies]
tags = ["vim", "neovim", "shell"]
+++

"Ilusión" es la palabra clave. Lo que voy a enseñarles no es ningún "truco mágico" que hará vim más rápido. Alejen ese pensamiento de sus mentes. Lo único que haremos hoy será retrasar lo inevitable.

## ¿Qué queremos lograr?

Quiero todos los plugins opcionales se inicialicen luego de que la interfaz esté totalmente cargada.

## ¿Por qué?

En mi caso, cuando veo que el editor ya está listo hay una pequeña ventana de tiempo de 2 o 3 segundos donde mi mente pasa de "¿Qué es lo que voy a hacer?" a "Ah, sí, esa cosa". Durante ese tiempo no estoy haciendo nada y es exactamente ahí donde quiero carguen todos los plugins. De esta manera ejecutamos sólo necesario para que vim pueda funcionar al inicio, y dejamos todo lo demás que es "opcional" en esa ventana de tiempo donde **yo** no estoy haciendo nada con el editor.

## Manos a la obra

En vim tenemos un [evento](https://vimhelp.org/autocmd.txt.html#autocmd-events) que se llama `VimEnter`, este nos indica el momento cuando vim termina todo su ritual de inicialización. Pero no es suficiente para mí, yo quiero estar seguro de que todo está listo. Entonces lo que quiero hacer es esperar unos 20 milisegundos después de este evento, el cual debería ser tiempo suficiente para que vim presente la interfaz, y luego cargar los plugins.

Necesitamos un "autocomando" y la función `timer_start`. Hagamos una pequeña prueba. Vamos a imprimir un mensaje 5 segundos luego de que vim inicie. Colocaremos esto en nuestro `.vimrc`.

```vim
function! s:load_plugins(t) abort
  echom "vim está listo"
endfunction

augroup user_cmds
  autocmd!
  autocmd VimEnter * call timer_start(5000, function('s:load_plugins'))
augroup END
```

La próxima vez que inicien vim, deberían ver el mensaje `vim está listo` después de unos segundos. Si no vieron el mensaje o desapareció rápidamente, pueden revisar con el commando `:messages`.

Una vez que estamos seguros que nuestra función es ejecutada cambiamos el tiempo de `5000` a `20`. Y en `s:load_plugins` ya podemos colocar los comandos necesarios para cargar los plugins.

Esta siguiente parte va a depender de su manejador de plugins. Vamos a usar la funcionalidad nativa conocida como [packages](https://vimhelp.org/repeat.txt.html#packages). Si no están seguros si la usan o no, revisen la documentación de su manejador plugins.

La última pieza del rompecabezas es el comando `packadd`, es el mecanismo que utiliza vim para cargar un plugin "opcional". Entonces nuestro código debería quedar así.

```vim
function! s:load_plugins(t) abort
  " lista de plugins
  packadd vim-surround
  packadd vim-obsession

  "(opcional) ejecutar scripts en el directorio `after`
  runtime! OPT after/plugin/*.vim

  "(opcional) emitimos un 'evento' propio
  " para indicar que ya todo está listo
  doautocmd User PluginsLoaded
endfunction

augroup user_cmds
  autocmd!
  autocmd VimEnter * call timer_start(20, function('s:load_plugins'))
augroup END
```

Eso es todo. Pero aún no termino.

## ¿Qué hay de lua?

En `neovim` (a partir de la versión 0.5) se ha incluido soporte para crear la configuración enteramente en lua, y mucha gente ha migrado su configuración a este lenguaje. En efecto también podemos hacer lo mismo en lua, pero las limitaciones actuales los obligará a escribir mucho vimscript.

He aquí lo que necesitan.

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

Aquí estoy creando una función global porque es lo más fácil. Por lo general deberían evitar ese tipo de funciones. Si saben cómo crear y usar un módulo en lua, les sugiero que hagan eso.

## Advertencia

Consideren qué plugins van a cargar de esta manera. 

Vamos a tomar `NERDTree`(un gestor de archivos) como ejemplo. Si abren un directorio desde la terminal, algo como `vim .`, `NERDTree` puede activarse automáticamente para enseñarles los archivos del directorio. Pero si retrasan la carga inicial de `NERDTree` este no podrá cumplir su función en este caso específico.

Ocurre algo similar con los plugins de sintaxis, si los cargan tarde puede que noten cuando vim cambia los colores por defecto a los colores que declara el plugin. No es una experiencia agradable.

## Conclusión

Aprendimos sobre el evento `VimEnter`, que es emitido por vim al finalizar su proceso inicial. Llegamos a conocer las funciones `timer_start` y `vim.defer_fn`, que podemos usarlas para ejecutar una función en un momento que nos parezca más conveniente. Finalmente, unimos todas estas piezas para cargar nuestros plugins opcionales en un momento específico donde nosotros como usuarios no estamos interactuando con el editor.

Pero también deberíamos tener cuidado porque cargar un plugin fuera del proceso inicial de vim puede tener efectos no deseados. Entonces deberíamos evaluar la situación por plugin, decidir qué debería ser un plugin opcional y qué no.

## Fuentes

* [:help timers](https://vimhelp.org/eval.txt.html#timers) 
* [:help :doautocmd](https://vimhelp.org/autocmd.txt.html#%3Adoautocmd) 
* [:help User](https://vimhelp.org/autocmd.txt.html#User)
* [:help :packadd](https://vimhelp.org/repeat.txt.html#%3Apackadd)
* [:help :runtime](https://vimhelp.org/repeat.txt.html#%3Aruntime) 
* [:help vim.defer_fn](https://neovim.io/doc/user/lua.html#vim.defer_fn())

