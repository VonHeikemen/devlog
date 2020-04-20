+++
title = "Aprendiendo sobre el paradigma funcional: un camino a seguir" 
description = "Guía del contenido creado para la serie: Un poco del paradigma funcional en tu javascript"
date = 2020-04-03
lang = "es"
[taxonomies]
tags = ["javascript", "paradigma-funcional", "aprendizaje"]
+++

Aprender sobre el paradigma funcional en la programación no es una tarea fácil, sobre todo si buscan material que contenga ejemplos prácticos de cómo usar los conceptos que enseñan. Lo que presento en esta ocasión es una recopilación de las notas que he tomado, y que he transformado en "artículos", así como también enlaces a las fuentes de donde he sacado toda esta información.

Aunque todo este material esté relacionado, no planifiqué escribir todas esas notas. Entonces aquí intentaré darles una sugerencia en el orden de lectura.

## Conceptos básicos

Para empezar quisiera que vieran el video que me convenció de darle una oportunidad a este paradigma. La charla se llama "Programación funcional en JS: ¿Qué? ¿Por qué? ¿Cómo?" En el video se explica lo que es y lo que no es programación funcional, también muestra ejemplos de los conceptos básicos del paradigma en javascript.

{{ youtube(id="qtsbZarFzm8") }}

Si no pudieron entender la charla porque no hablan inglés, no se preocupen, una búsqueda rápida sobre **funciones puras** y sus beneficios técnicos debería ponerlos al tanto.

Ahora bien, yo también hice mi propia investigación y escribí un material que complementa lo que se dice en el video.  

- [Funciones puras y porque son una buena idea](@/web-development/learn-fp/pure-functions.es.md)

- [Cómo combinar efectos y funciones puras en javascript](@/web-development/learn-fp/dealing-with-side-effects-and-pure-functions.es.md)

### Lectura extra

- [Una introducción a la programación funcional (artículo en inglés)](https://codewords.recurse.com/issues/one/an-introduction-to-functional-programming)

## Una herramienta especial

Si ya revisaron todo el material anterior ya cuentan con suficiente conocimiento para empezar a incorporar un poco del estilo funcional en su rutina habitual. No tienen que conocer todos los trucos del libro para beneficiarse de este paradigma.

Quiero que presten una atención especial a algo llamado **aplicación parcial**, al igual que las funciones puras este es un que concepto que puede ayudarlos mucho, incluso si deciden no adoptar el paradigma funcional completamente.

Estas son mis notas (con ejemplos prácticos): 
- [Aplicación parcial](@/web-development/learn-fp/partial-application.es.md).

Si están convencidos de que la aplicación parcial es útil vean este video para que tengan una idea del tipo de cosas que pueden lograr.

{{ youtube(id="m3svKOdZijA") }}

## Cómo armar las piezas

Una cosa es conocer los conceptos y otra es saber utilizarlos de la manera más efectiva posible. Ya tienen las bases y algunas herramientas, pero aún deben estar preguntándose ¿Cómo encaja todo esto? Ese nuestro siguiente paso. 

En este artículo veremos cómo podemos usar lo que hemos aprendido:

- [Técnicas de composición](@/web-development/learn-fp/composition-techniques.es.md)

Y por si acaso se pasaron por alto este video, aquí se los dejo otra vez. Aquí se explica con un poco más de detalle lo que está en el material que yo escribí (porque lo que yo escribí son notas que tomé de aquí).

{{ youtube(id="vDe-4o8Uwl8") }}

## Un paso más allá

Ya tienen una idea de cómo manipular funciones y adaptarlas a sus necesidades. Pero todavía hay un par de conceptos que no están claros, dos en particular: Functors y Monads. Aquí hago mi mejor esfuerzo para decirles cómo pueden usarlos en su beneficio. 

- [Hablando de Functors](@/web-development/learn-fp/the-power-of-map.es.md)

- [Usando un Maybe](@/web-development/learn-fp/using-a-maybe.es.md)

## Contenido extra

- [Reduce: cómo y cuando](@/web-development/learn-fp/reduce-how-and-when.es.md)
- [Lenses: Una alternativa a los getters y setters](@/web-development/learn-fp/lenses-a-k-a-composable-getters-and-setters.es.md)

### Más charlas interesantes

Si se siguen preguntando qué se puede lograr sólo combinando funciones.

- [Mary had a little lambda](https://www.youtube.com/watch?v=7BsfMMYvGaU)
- [Oh Composable World!](https://www.youtube.com/watch?v=SfWR3dKnFIo)

## Hasta la próxima

Si llegaron hasta aquí y ya revisaron todo, entonces saben tanto como yo. No tengo nada más que enseñarles. Ya sea que hayan decidido adoptar el paradigma funcional en su código o no espero que hayan aprendido algo que puedan aplicar en su desarrollo diario.
