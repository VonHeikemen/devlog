+++
title = "Funciones puras y porque son una buena idea" 
description = "Respondiendo a la pregunta ¿qué ganamos al usar funciones puras?"
date = 2020-04-02
lang = "es"
[taxonomies]
tags = ["javascript", "paradigma-funcional", "aprendizaje"]
+++

Cuando se habla del paradigma funcional en la programación uno de los conceptos básicos que resalta por su importancia son la funciones puras. Las personas acostumbradas a este paradigma hacen un gran esfuerzo para mantener su código tanto como sea posible en funciones puras, aquí les explicaré algunas de las razones. Pero primero debemos saber...

## ¿Qué es una función pura?

Una función cuyo resultado es influenciado solamente por sus parámetros de entrada y no tiene ningún efecto observable en el mundo exterior (lo que se conoce como efecto secundario).

### Beneficios

Quiero enfocarme en los beneficios que este tipo de funciones nos aportan a nosotros los humanos que leemos e interpretamos código en nuestra mente.

- Son predecibles

Proporcionarles los mismos datos de entrada siempre produce el mismo resultado. Esta es una de las propiedades más relevantes, y para mí es la más importante. Nos da habilidad de probar con relativa facilidad la efectividad de nuestra solución. 

Digamos que tenemos una función que transforma todas las letras de un texto a mayúscula ¿qué necesitamos para probar que funciona? La función, sus parámetros y el valor esperado.

```js
to_uppercase('hello') == 'HELLO';
```

No necesitamos simular el ambiente externo o herramientas especiales, sólo comparamos con el valor esperado. Esto nos da confianza en lo que hemos creado, porque podemos probar con certeza que funciona de manera adecuada.

- Comprensión

Cuando hablamos de código nos pasamos más tiempo leyendo y analizando que escribiendo. La comunicación es un aspecto que debemos considerar. Una función pura en teoría necesitaría la menor cantidad de contexto para poder entender su comportamiento, ya que todo lo que necesitas saber está (o al menos debería) en el cuerpo de la función y sus argumentos.

Otra propiedad que poseen estas funciones se le conoce como **transparencia referencial**, esto significa que podemos reemplazar la llamada de una función con el valor que retorna.

Por ejemplo, esto.

```js
to_uppercase('hi') + ', user';
```

Puede ser reemplazado por esto.

```js
'HI, user';
```

Quiere decir que una vez que comprendes qué hace una función pura puedes reemplazar mentalmente la llamada de la función con su resultado.

- Composición

Si han creado una función pura hay una alta probabilidad de que hayan creado un componente independiente que pueden aprovechar en diferentes contextos. Al ser completamente independientes y reusables son los candidatos perfectos para ser combinados con otros componentes. Piénsenlo, si "combinan" dos funciones puras en una nueva función, el resultado también será una función pura. Esta es una característica poderosa que les permitirá crear procedimientos complejos con piezas "simples."

## Esto no termina aquí

Las funciones puras pueden ser buenas pero en algún momento debemos abandonar la idea de la pureza y causar un cambio en el mundo (mostrar algo en pantalla, hacer una petición HTTP, etc...) para eso he preparado otros artículos con más detalles sobre el tema.

- [Técnicas de composición](@/web-development/fp-in-js/composition-techniques.es.md)

- [Cómo combinar efectos y funciones puras en javascript](@/web-development/fp-in-js/dealing-with-side-effects-and-pure-functions.es.md)

## Fuentes

- [Functional Programming in JS: What? Why? How? (video)](https://www.youtube.com/watch?v=qtsbZarFzm8&feature=youtu.be)
- [An introduction to functional programming](https://codewords.recurse.com/issues/one/an-introduction-to-functional-programming)
- [Functional-Light JavaScript - Chapter 5: Reducing Side Effects](https://github.com/getify/Functional-Light-JS/blob/master/manuscript/ch5.md/#chapter-5-reducing-side-effects)
