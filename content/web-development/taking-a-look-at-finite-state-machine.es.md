+++
title = "Un vistazo a las máquinas de estados finitos"
description = "Veamos si las máquinas de estados son útiles"
date = 2019-11-14
lang = "es"
[taxonomies]
tags = ["javascript", "aprendizaje"]
[extra]
canonical_url = "https://dev.to/vonheikemen/un-vistazo-a-las-maquinas-de-estados-finitos-1181"
+++

## ¿Máquinas de qué-- quién?

Las máquinas de estados finitos son una manera de modelar el comportamiento de un sistema. La idea es que tu "sistema" sólo puede encontrarse en un estado a la vez, y una entrada (evento) puede activar la transición a otro estado.

## ¿Qué tipo de problemas resuelven?

Estados inválidos. ¿Cuántas veces han tenido que usar una variable con un booleano o un atributo como "disabled" para evitar que un usuario haga algo indebido? Al marcar las reglas de comportamiento por adelantado podemos evitar este tipo de cosas.

## ¿Cómo se hace eso en javascript?

Me alegra que preguntaran. La verdadera razón por la escribo esto es para mostrar una librería que vi el otro día. Vamos a usar [robot3](https://thisrobot.life/) para crear un máquina de frases semi-famosas.

Lo que haremos será mostrar una "carta" con una frase y debajo de tendremos un botón que podremos usar para mostrar otra frase.

Haremos esto un paso a la vez. Primero preparemos los posibles estados de la aplicación.

Nuestra carta estará en estado `idle` (algo así como 'esperando') o `loading` (cargando) Crearemos nuestra máquina a partir de eso.

```js
import {
  createMachine,
  state,
  interpret
} from 'https://unpkg.com/robot3@0.2.9/machine.js';

const mr_robot = createMachine({
  idle: state(),
  loading: state()
});
```

Aquí cada `estado` es un índice del "objeto de configuración" que le pasamos a `createMachine`, vean que cada uno de estos índices deben ser el resultado de llamar la función `state`.

Ahora necesitamos transiciones. El estado `idle` cambiará a estado `loading` si ocurre un evento `fetch` (buscar), `loading` volverá a `idle` cuando el evento `done` (terminado) sea despachado.

```diff
 import {
  createMachine,
  state,
+ transition,
  interpret
 } from 'https://unpkg.com/robot3@0.2.9/machine.js';

const mr_robot = createMachine({
-  idle: state(),
-  loading: state()
+  idle: state(transition('fetch', 'loading')),
+  loading: state(transition('done', 'idle'))
 });
```

`transition` es lo que conecta los estados. El primer parámetro que recibe es el nombre del evento que lo activará, el segundo parámetro es el "evento destino" al cual cambiará. El resto de los parámetros consiste en una de funciones que serán ejecutadas cuando ocurra la transición.

Luce bien y todo pero... uhm... ¿cómo hacemos pruebas? Por sí sola la máquina no hace nada. Necesitamos que nuestra máquina sea interpretada y para ello se la pasamos a la función `interpret`, esta función nos devuelve un "servicio" con el cual podemos despachar eventos. Para asegurarnos que de verdad estamos haciendo algo vamos a usar el segundo parámetro de `interpret` el cual será una función que "escuchará" los cambios de estado.

```js
const handler = ({ machine }) => {
  console.log(machine.current);
}

const { send } = interpret(mr_robot, handler);
```

Ahora veamos si está viva.

```js
send('fetch');
send('fetch');
send('fetch');
send('done');

// Deberían ver en la cónsola
// loading (3)
// idle
```

Despachar `fetch` hace que el estado actual se convierta en `loading` y despachar`done` lo regresa a `idle`. Veo que no están impresionados. Bien. Intentemos algo más. Agregemos otro estado `end` y hagamos que `loading` cambie a ese, luego despachamos `done` y vemos qué pasa.

```diff
 const mr_robot = createMachine({
   idle: state(transition('fetch', 'loading')),
-   loading: state(transition('done', 'idle'))
+   loading: state(transition('done', 'end')),
+   end: state()
 });
```
```js
send('done');

// Deberían ver en la cónsola
// idle
```

Enviar `done` mientras el estado es `idle` no activa el estado `loading`, se queda en `idle` porque ese estado no tiene un evento `done`. Y ahora...

```js
// El curso normal de eventos.

send('fetch');
send('done');

// Deberían ver en la cónsola
// loading
// end

// Intenten con `fetch`
send('fetch');

// Ahora...
// end
```

Si enviamos `fetch` (o cualquier otro evento) mientras el estado es `end` resultará en `end` siempre. ¿Por qué? Porque no hay a dónde ir, `end` no tiene transiciones.

Espero que les haya sido útil, si no fue así me disculpo por tanto `console.log`. 

Volvamos a nuestra máquina. Esto es lo que tenemos hasta ahora.

```js
 import {
  createMachine,
  state,
  transition,
  interpret
} from 'https://unpkg.com/robot3@0.2.9/machine.js';

const mr_robot = createMachine({
  idle: state(transition('fetch', 'loading')),
  loading: state(transition('done', 'idle'))
});

const handler = ({ machine }) => {
  console.log(machine.current);
}

const { send } = interpret(mr_robot, handler);
```

Pero aún no es suficiente, ahora debemos extraer datos de alguna parte cuando el estado sea `loading`. Vamos a fingir que buscamos los datos en nuestra función.

```js
function get_quote() {
  // crea un retraso de 3 a 5 segundos.
  const delay = random_number(3, 5) * 1000;

  const promise = new Promise(res => {
    setTimeout(() => res('<quote>'), delay);
  });
  
  // nomás pa' ver
  promise.then(res => (console.log(res), res));

  return promise;
}
```

Para integrar esta función a nuestra máquina vamos a usar la función `invoke`, esta nos ayuda a manejar "funciones asíncronas" (una función que devuelve una promesa) cuando se active el estado, luego cuando la promesa se resuelve envía el evento `done` (si algo falla envía el evento `error`).

```diff
  import {
   createMachine,
   state,
+  invoke,
   transition,
   interpret
 } from 'https://unpkg.com/robot3@0.2.9/machine.js';

 const mr_robot = createMachine({
   idle: state(transition('fetch', 'loading')),
-  loading: state(transition('done', 'idle')),
+  loading: invoke(get_quote, transition('done', 'idle')),
 });
```

Si prueban `send('fetch')` deberían ver en la cónsola.

```
loading

// Esperen unos segundos...

<quote>
idle
```

Espero que estas alturas se estén preguntando ¿Y dónde guardamos los datos? `createMachine` nos deja definir un "contexto" que estará disponible para nosotros en las función que apliquemos en las transiciones.

```js
const context = ev => ({
  data: {},
});
```

```diff
  const mr_robot = createMachine({
    idle: state(transition('fetch', 'loading')),
    loading: invoke(get_quote, transition('done', 'idle')),
- });
+ }, context);
```

Ahora agregaremos una función a nuestra transición `loading`. Será el lugar donde modificaremos el context. Esta función es llamada `reduce` y luce así.

```js
reduce((ctx, ev) => ({ ...ctx, data: ev.data }))
```

Recibe el context actual, una carga (aquí la llamamos `ev`) y lo que sea que devuelva se convertirá en tu nuevo contexto. 

```diff
  import {
   createMachine,
   state,
   invoke,
   transition,
+  reduce,
   interpret
 } from 'https://unpkg.com/robot3@0.2.9/machine.js';

 const mr_robot = createMachine({
   idle: state(transition('fetch', 'loading')),
-  loading: invoke(get_quote, transition('done', 'idle')), 
+  loading: invoke(
+    get_quote, 
+    transition(
+      'done',
+      'idle',
+      reduce((ctx, ev) => ({ ...ctx, data: ev.data }))
+    )
+  ),
 }, context);
```

Hora de probar. ¿Cómo lo hacemos? Modificamos el callback de `interpret`.

```js
const handler = ({ machine, context }) => {
  console.log(JSON.stringify({ 
    state: machine.current,
    context
  }));
}
```

Deberían ver esto.

```
{'state':'loading','context':{'data':{}}}

// esperen unos segundos...

{'state':'idle','context':{'data':'<quote>'}}
```

Estamos listos. Mostremos algo en el navegador.

```html
<main id="app" class="card">
  <section id="card" class="card__content">
     <div class="card__body">
        <div class="card__quote">
          quote
        </div>

        <div class="card__author">
          -- author
        </div>
      </div>
      <div class="card__footer">
        <button id="load_btn" class="btn btn--new">
          More
        </button>
        <a href="#" target="_blank" class="btn btn--tweet">
          Tweet
        </a>
      </div> 
  </section> 
</main>
```

```css
body {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 95vh;
  background: #ddd;
  font-size: 1em;
  color: #212121;
}

.card {
  width: 600px;
  background: white;
  box-shadow: 0 2px 5px 0 rgba(0, 0, 0, 0.16), 0 2px 10px 0 rgba(0, 0, 0, 0.12);
}

.card__content {
  color: #212121;
  padding: 20px;
}

.card__content--loader {
  height: 95px;
  display: flex;
  align-items: center;
  justify-content: center
}

.card__body {
 padding-bottom: 15px;
}

.card__author {
  padding-top: 10px;
  font-style: italic;
}

.card__footer {
  width: 100%;
  display: flex;
  justify-content: space-between;
}

.btn {
  color: #fff;
  cursor: pointer;
  margin-top: 10px;
  margin-left: 10px;
  border-radius: 0.4rem;
  text-decoration: none;
  display: inline-block;
  padding: .3rem .9rem;
}

.btn--new {
  background-color: #2093be;
  border: 0.1rem solid #2093be;
  
}

.btn--tweet {
  background-color: #0074d9;
  border: 0.1rem solid #0074d9;
}

.btn:hover {
  background: #3cb0fd;
  border: 0.1rem solid #3cb0fd;
  text-decoration: none;
}

.hide {
  display: none;
}
```

La última pieza del rompecabezas, los efectos secundarios. Necesitamos agregar otra función a la transición `loading` para poder actualizar el DOM. Podríamos usar `reduce` nuevamente pero es de mala educación hacer eso en algo que se llame `reduce`. Utilizaremos otra función, una llamada `action`.

Pero primero debemos preprarnos. Modificaremos el contexto con las dependencias necesarias. (Este paso es innecesario, esto es sólo por mi alergia a las variables globales)

```diff
 const context = ev => ({
   data: {},
+  dom: {
+    quote: document.querySelector('.card__quote'),
+    author: document.querySelector('.card__author'),
+    load_btn: window.load_btn,
+    tweet_btn: document.querySelector('.btn--tweet'),
+    card: window.card
+  }
 });
```

Ahora sí, efectos secundarios. En este punto deberían asegurarse que `get_quote` devuelva un objeto con las propiedades `quote` y `author`.

```js
function update_card({ dom, data }) {
  dom.load_btn.textContent = 'More';
  dom.quote.textContent = data.quote;
  dom.author.textContent = data.author;

  const web_intent = 'https://twitter.com/intent/tweet?text=';
  const tweet = `${data.quote} -- ${data.author}`;
  dom.tweet_btn.setAttribute(
    'href', web_intent + encodeURIComponent(tweet)
  );
}

function show_loading({ dom }) {
  dom.load_btn.textContent = 'Loading...';
}
```

Juntamos todo.

```diff
  import {
   createMachine,
   state,
   invoke,
   transition,
   reduce,
+  action,
   interpret
 } from 'https://unpkg.com/robot3@0.2.9/machine.js';

 const mr_robot = createMachine({
-  idle: state(transition('fetch', 'loading')),
+  idle: state(transition('fetch', 'loading', action(show_loading))),
   loading: invoke(
     get_quote, 
     transition(
       'done',
       'idle',
       reduce((ctx, ev) => ({ ...ctx, data: ev.data })),
+      action(update_card)
     )
   ),
 }, context);
```

Funciona. Pero se ve mal cuando carga por primera vez. Hagamos otra transición de carga, una que esconda la carta mientras se carga la primera frase.

Empecemos por el HTML.

```diff
 <main id="app" class="card">
+  <section class="card__content card__content--loader"> 
+    <p>Loading</p> 
+  </section>
-  <section id="card" class="card__content">
+  <section id="card" class="hide card__content">
     <div class="card__body">
       <div class="card__quote">
         quote
       </div>

       <div class="card__author">
          -- author
       </div>
     </div>
     <div class="card__footer">
       <button id="load_btn" class="btn btn--new">
         More
       </button>
       <a href="#" target="_blank" class="btn btn--tweet">
         Tweet
       </a>
     </div> 
   </section> 
 </main>
```

Creamos otro estado, `empty`. Podemos reusar la lógica del estado `loading` para esto. Creamos una función que crea transiciones.

```js
const load_quote = (...args) =>
  invoke(
    get_quote,
    transition(
      'done',
      'idle',
      reduce((ctx, ev) => ({ ...ctx, data: ev.data })),
      ...args
    ),
    transition('error', 'idle')
  );
```
```diff
 const mr_robot = createMachine({
   idle: state(transition('fetch', 'loading', action(show_loading))),
-  loading: invoke(
-    get_quote, 
-    transition(
-      'done',
-      'idle',
-      reduce((ctx, ev) => ({ ...ctx, data: ev.data })),
-      action(update_card)
-    )
-  ),
+  loading: load_quote(action(update_card))
 }, context);
```

Ahora la usamos para esconder el esqueleto de la carta en la primera carga y muestre la frase cuando esté lista.

```diff
 const context = ev => ({
   data: {},
   dom: {
     quote: document.querySelector('.card__quote'),
     author: document.querySelector('.card__author'),
+    loader: document.querySelector('.card__content--loader'),
     load_btn: window.load_btn,
     tweet_btn: document.querySelector('.btn--tweet'),
     card: window.card
   }
 });
```

```js
function hide_loader({ dom }) {
  dom.loader.classList.add('hide');
  dom.card.classList.remove('hide');
}
```

```diff
 const mr_robot = createMachine({
+  empty: load_quote(action(update_card), action(hide_loader)),
   idle: state(transition('fetch', 'loading', action(show_loading))),
   loading: load_quote(action(update_card))
 }, context);
-
- const handler = ({ machine, context }) => {
-  console.log(JSON.stringify({ 
-    state: machine.current,
-    context
-  }));
- }
+ const handler = () => {};

 const { send } = interpret(mr_robot, handler);
+
+ const fetch_quote = () => send('fetch');
+
+ window.load_btn.addEventListener('click', fetch_quote);
```

Veamos cómo quedó.

{{ codepen(id="OJJvQzR", title="Finite Random Quote Machine", load_js=true) }}

## ¿Entonces esto de la máquina de estados finitos fue útil?

Eso espero. ¿Notaron que nos permitió hacer un montón de pruebas y planear el comportamiento incluso antes de crear el HTML? Me parece que eso es genial.

¿Intentaron darle click al botón 'loading' mientras cargaba? ¿Causó llamadas repetidas a `get_quote`? Eso es porque hicimos que fuera (casi) imposible que el evento `fetch` ocurriera durante `loading`.

No sólo eso, el comportamiento de la máquina y sus efectos en el mundo exterior están separados. Esto puede ser bueno o malo para ustedes pero eso depende de su tendencia filosófica.

## ¿Quieren saber más?
(me perdonan que todo estos sean en inglés.)

* [XState (concepts)](https://xstate.js.org/docs/about/concepts.html)
* [robot3 - docs](https://thisrobot.life/)
* [Understanding State Machines](https://www.freecodecamp.org/news/state-machines-basics-of-computer-science-d42855debc66/)
