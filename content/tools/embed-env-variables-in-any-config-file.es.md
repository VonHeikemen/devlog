+++
title = "¿Quieres usar variables de entorno en ese archivo de configuración? Puedes... casi"
description = "Presentando al comando envsubst"
date = 2021-10-19
lang = "es"
[taxonomies]
tags = ["shell", "hoy-aprendi"]
+++

Imaginen que hay una herramienta que utilizan en todas sus máquinas y esta requiere de un archivo de configuración. Probablemente este archivo necesita de un pedazo de información que cambia de acuerdo al sistema operativo, o tal vez según el contexto (computadora personal o de trabajo, un servidor... cosas así). Entonces siempre tenemos que cambiar algún detalle de este archivo. ¿No sería genial si pudíeramos usar variables? Resulta que podemos... casi.

## Les presento el comando envsubst

`envsubst` puede incrustar el valor de una variable de entorno en una cadena de texto que recibe desde la *entrada estándar*.

Esto quiere decir que en nuestra terminal podemos hacer algo así.

```sh
echo 'there is no place like $HOME' | envsubst
```

Si estamos usando linux deberíamos obtener el mensaje.

```
there is no place like /home/user
```

Básicamente podemos hacer trampa. Tenemos la posibilidad de usar `envsubst` como un pre-procesador y crear nuestros archivos de configuración a partir de una plantilla.

## Un caso de la vida real

Para mí `envsubst` resuelve un problema que tengo con `npm`. En mi carpeta de usuario tengo el archivo `.npmrc` con la siguiente estructura.

```
prefix=<una-ruta>
ignore-scripts=true

init-author=<un-nombre> (<un-enlace>)
init-license=MIT
```

Como podrán imaginar algunos de esos campos pueden variar según la máquina o el contexto, entonces siempre me encuentro cambiando algunas cosas. Pero ahora que sé que existe `envsubst` puedo tener una sola plantilla en `~/templates/npmrc` y usarla como una base para todas mis máquinas.

¿Cómo haríamos? Lo primero será poner algo en nuestra plantilla.

```
prefix=$XDG_CONFIG_HOME/npm/packages
ignore-scripts=true

init-author=$USER_NAME ($CONTACT_LINK)
init-license=MIT
```

Okey, si sólo queremos probar este es el comando que necesitamos.

```sh
envsubst < ~/templates/npmrc
```

Cuando usamos `<` lo que estamos haciendo es tomar el contenido del archivo y enviarlo a `envsubst` usando la entrada estándar. Esto es lo que se conoce como redirección. Entonces, si tenemos todas las variables definidas `envsubst` nos mostrará el resultado en pantalla.

```
prefix=/home/user/.config/npm/packages
ignore-scripts=true

init-author=Heiker (https://github.com/VonHeikemen)
init-license=MIT
```

Para finalizar queremos tomar todo eso y redirigirlo a `~/.npmrc`.

```sh
envsubst < ~/templates/npmrc > ~/.npmrc
```

Técnicamente ya terminamos pero ya que estamos aquí vamos a explorar otra posibilidad.

Como demostramos al principio, la cadena de texto puede venir de cualquier comando, el único requisito que tenemos es que debemos enviar el contenido usando la entrada estándar.

Si nos ponemos creativos podemos alojar el contenido de la plantilla en cualquier servidor (como github) y descargarlo usando `curl`.

```sh
curl -s "https://raw.githubusercontent.com/<user>/<repo>/main/npmrc" \
  | envsubst > ~/.npmrc
```

De esta manera ni siquiera necesitamos que la plantilla esté en nuestra máquina.

## Una alternativa multi-plataforma

Si usan linux es muy probable que ya tengan instalada esta herramienta. En caso de que no sea así pueden intentar descargar esta versión que fue creada usando el lenguaje `go`.

* [github.com/a8m/envsubst](https://github.com/a8m/envsubst) 

En su [página de descarga](https://github.com/a8m/envsubst/releases) están disponibles ejecutables para windows, mac y linux. 

## Conclusión

Aprendimos sobre el comando `envsubst` y cómo podemos usarlo para crear un archivo de configuración a partir de una plantilla. Vimos que `envsubst` recibe una cadena de texto usando la entrada estándar, lo que nos permite recibir nuestra "plantilla" desde un archivo o desde otro comando. Por último vimos cómo usar redirecciones para crear un archivo nuevo a partir del resultado de un comando.

## Fuente

* [man envsubst](https://www.mankier.com/1/envsubst)

