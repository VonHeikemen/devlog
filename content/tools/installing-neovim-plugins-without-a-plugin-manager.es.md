+++
title = "Neovim: Instalando plugins manualmente"
description = "Instalando plugins para Neovim usando git clone"
date = 2024-09-17
lang = "es"
[taxonomies]
tags = ["vim", "neovim", "shell"]
+++

Lo único que debemos hacer para instalar un plugin de Neovim es descargarlo en un directorio específico.

## ¿Cómo funciona?

Tanto Vim como Neovim tienen una funcionalidad que llaman [vim packages](https://neovim.io/doc/user/repeat.html#_using-vim-packages). Un paquete simplemente es un directorio que contiene varios plugins.

Esta es la estructura de archivos de un paquete:

```
some-package
├── colors
│   ├── a-colorscheme.vim
│   └── some-colorscheme.lua
├── doc
│   └── my-help-page.txt
├── lua
│   └── some-module
│       └── some-file.lua
└── plugin
    ├── a-file.vim
    └── some-file.lua
```

Según la documentación de Neovim, un [plugin](https://neovim.io/doc/user/usr_05.html#_global-plugins) es un script en el directorio `plugin`. El término oficial es plugin global. En el ejemplo anterior `plugin/a-file.vim` cuenta como un plugin, y `plugin/some-file.lua` es otro plugin.

¿Notan algo interesante? Esa estructura de archivos es la misma que tienen todos los plugins de Neovim que están github. Entonces, lo que nosotros llamamos un plugin la documentación de Neovim lo considera un paquete. 

Bien, ¿pero dónde debemos instalarlos?

## El directorio pack

`packpath` es una opción que tenemos en Neovim, es una lista de directorios. Cada uno de estos directorio puede tener otro directorio llamado `pack` y ahí podemos tener varios grupos de paquetes.

Dentro de `pack` podemos tener esta estructura:

```
pack
└── vendor
    ├── opt
    │   └── package-one
    └── start
        ├── package-two
        └── package-three
```

Aquí el directorio `vendor` es un grupo. `vendor` es sólo un nombre, ustedes pueden elegir cualquier nombre para el grupo.

Dentro de `vendor` tenemos `opt` y `start`. Estos dos directorios determinan en qué momento Neovim va a cargar los plugins.

Los paquetes que ese encuentran en `start` serán cargados de manera automática durante el proceso de inicialización. Esto significa que podremos usarlos inmediatamante.

Los paquetes que se encuentran en `opt` pueden cargarse manualmente usando el comando `:packadd`. Este es el mecanismo que nos da Neovim para hacer lo que se conoce como "lazy loading." En otras palabras, cargar plugins sólo cuando los necesitamos.

## El directorio de descarga

Ahora sí, para conocer el directorio donde podemos descargar un plugin debemos inspeccionar la opción `packpath`. Pueden ejecutar este comando.

```vim
:set packpath?
```

Si les parece que el formato de ese comando no es legible, pueden intentar con este otro comando que usa `lua`.

```lua
:lua vim.tbl_map(print, vim.opt.packpath:get())
```

Esto les mostrará la lista de directorios donde pueden tener un directorio `pack`.

Un ejemplo concreto:

Por defecto el directorio de configuración de Neovim se encuentra en el `packpath`. Entonces el directorio donde tienen su `init.lua` o `init.vim` es un lugar válido para crear el directorio `pack`.

Vamos a fingir que ustedes usan linux y quieren descargar el plugin [mini.nvim](https://github.com/nvim-mini/mini.nvim). Para instalarlo pueden ejecutar este comando en su terminal.

```
git clone https://github.com/nvim-mini/mini.nvim \
  ~/.config/nvim/pack/vendor/start/mini.nvim
```

Luego de descargar un plugin deben generar los "help tags" para que puedan acceder a la documentación del plugin dentro de Neovim. Entonces, luego de descargar el plugin entran a Neovim y ejecutan este comando:

```vim
:helptags ALL
```

Luego de crear los tags podrán usar el comand `:help` para leer la documentación "offline" de un plugin. Por ejemplo, `:help mini.files`.

Ahora que `mini.nvim` está instalado podemos probarlo. Podemos usar el módulo `mini.files` en nuestra configuración.

Vamos a crear el atajo de teclado `<F2>` para hacer que aparezca el explorador de archivos.

```lua
local mini_files = require('mini.files')

mini_files.setup({})

vim.keymap.set('n', '<F2>', function()
  if mini_files.close() then
    return
  end

  mini_files.open()
end)
```

## Un comando de regalo

Si quieren, pueden convertir todo el conocimento que acaban de obtener en código. Pueden crear un comando de Neovim que use `git clone` para descargar un plugin directo desde github.

```lua
vim.api.nvim_create_user_command(
  'GitPlugin',
  function(input)
    local repo = input.fargs
    local url = 'https://github.com/%s/%s.git'
    local plugin_dir = vim.fn.stdpath('config') 
      .. '/pack/vendor/start/%s'

    if repo[1] == nil or repo[2] == nil then
      local msg = 'Debe proveer nombre de usuario y repositorio'
      vim.notify(msg, vim.log.levels.WARN)
      return
    end

    local full_url = url:format(repo[1], repo[2])
    local command = {'git', 'clone', full_url, plugin_dir:format(repo[2])}

    local on_done = function()
      vim.cmd('packloadall! | helptags ALL')
      vim.notify('Listo.')
    end

    vim.notify('Clonando repositorio...')
    vim.fn.jobstart(command, {on_exit = on_done})
  end,
  {nargs = '+'}
)
```

Luego de crear este comando podrán descargar plugins desde Neovim. Sólo necesitan saber el nombre del usuario en github y el nombre repositorio.

Por ejemplo, este comando les permitirá descargar `mini.nvim`.

```
:GitPlugin echasnovski mini.nvim
```

## Conclusión

Ya tienen el conocimento necesario para descargar cualquier plugin de Neovim en github sin necesidad de un manejador de plugins.

Quiero aclarar, esto no significa que los manejadores de plugins son inútiles. Si van a usar Neovim como su editor principal probablemente necesitan uno, ya que Neovim no ofrece un mecanismo para actualizar o eliminar plugins. Los `vim packages` sólo son un mecanismo para usar plugins dentro de Neovim.

