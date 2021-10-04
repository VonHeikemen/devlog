+++
title = "Cómo un funtor aplicativo puede ayudarnos a validar formularios"
description = "Validando formularios al 'estilo funcional'"
date = 2021-10-03
lang = "es"
[taxonomies]
tags = ["javascript", "paradigma-funcional"]
+++

Lo que haremos en esta ocasión será 'jugar' con este concepto de funtor aplicativo, específicamente lo vamos a usar para validar datos que un usuario ha colocado en un formulario.

Si no saben qué es un funtor aplicativo tal vez quieran un breve resumen... pero no puedo hacer eso hoy. Aún no tengo un dominio del tema lo suficientemente bueno para explicarles sin tener que bombardearlos con información que no necesitarán.

Si quieren saber con más detalle les recomiendo leer por lo menos uno de estos artículos.

- [Hablando de Funtors](@/web-development/learn-fp/the-power-of-map.es.md)
- [Un poco de Funtor Aplicativos](@/web-development/learn-fp/applicative-functors.es.md)
- [Explorando Fantasy Land](@/web-development/tagged-unions-and-fantasy-land.es.md)

Por ahora les contaré con un ejemplo uno de los problemas que podemos resolver con un funtor aplicativo.

## Imaginen

Imaginen esta situación. Tienen un valor y una función, y quieren aplicar la función a ese valor.

```js
const valor = 1;
const fn = (x) => x + 1;
```

La solución es simple.

```js
fn(valor); // => 2
```

Todo bien. No necesitamos ningún nada más. Pero ahora imaginemos que estos valores están "atrapados" en una estructura.

```js
const Valor = [1];
const Fn = [(x) => x + 1];
```

En este ejemplo la estructura que usamos es el arreglo. Queremos aplicar la función al valor y queremos que el resultado también sea un arreglo. ¿Cómo harían eso?

```js
[Fn[0](Valor[0])]; // => [2]
```

¿Así? No parece apropiado. En un mundo ideal podríamos hacer algo mejor.

```js
Valor.ap(Fn); // => [2]
```

Lo que queremos es tratar la aplicación de funciones cómo si fuera otra propiedad (o método) de nuestra estructura.

La mala noticia es que no vivimos en ese mundo. La buena noticia es que implementar esta operación nosotros mismos.

```js
const List = {
  ap(Fn, Valor) {
    return Valor.flatMap(x => Fn.map(f => f(x)));
  }
};
```

Con esta pequeña función podríamos resolver nuestro problema.

```js
const Valor = [1];
const Fn = [(x) => x + 1];

List.ap(Fn, Valor); // => [2]
```

## El siguiente paso

Ahora pongamos nuestra atención en otra estructura: objetos.

Imaginemos la misma situación pero esta vez los elementos que queremos usar están atrapados en un objeto de la misma "forma".

```js
const Valor = {email: 'this@example.com'};
const Fn = {email: (input) => input.includes('@')};
```

¿Cómo haríamos en este caso? Bueno, tomamos la función de una propiedad y la aplicamos al valor correspondiente en el otro objeto. Vamos a implementar esos pasos en una función.

```js
const Obj = {
  ap(Fn, Data) {
    const result = {};
    for(let key in Data) {
      result[key] = Fn[key](Data[key]);
    }

    return result;
  }
}
```

Ahora hacemos lo mismo que en el ejemplo anterior.

```js
const Valor = {email: 'this@example.com'};
const Fn = {email: (input) => input.includes('@')};

Obj.ap(Fn, Valor); // => {email: true}
```

## Hagamos una cadena

Bien, podríamos aplicar **una** validación a un campo, ¿pero es suficiente? Probablemente no. Lo mejor sería devolver un mensaje de error para el usuario. Aparte de eso, también sería una buena idea poder aplicar varias funciones a la vez.

Lo que quiero hacer es tomar una función, un mensaje y poner ambos en un arreglo. Y quiero una lista de esos pares. Algo así.

```js
[
  [long_enough, 'Intenta otra vez'],
  [is_email, 'No es válido']
]
```

Si la función retorna `false` entonces el mensaje de error es agregado a un arreglo. Sencillo, ¿Cierto? Vamos a crear una función para manejar esta cadena de validaciones.

```js
function validate(validations, input) {
  const error = [];
  for(let [validation, msg] of validations) {
    const is_valid = validation(input);

    if(!is_valid) {
      error.push(msg);
    }
  }

  return error;
}
```

Noten que he dejado el parámetro `input` de último, esto es porque quiero "aplicar" el parámetro `validations` sin tener que ejecutar la función. Para lograr este efecto usaré `Function.bind`.

```js
validate.bind(null, [
  [long_enough, 'Intenta otra vez'],
  [is_email, 'No es un correo válido']
]);
```

Hay otras formas para lograr la aplicación parcial, pero a mi gusta esta.

Lo siguiente será implementar las validaciones que queremos ejecutar.

```js
function long_enough(input) {
  return input.length >= 2;
}

function is_email(input) {
  return input.includes("@");
}

function no_numbers(input) {
  return !(/\d/.test(input));
}
```

Ahora podemos poner todo junto en un caso de prueba.

```js
const input = {
  name: '1',
  email: 'a'
};

const validations = {
  name: validate.bind(null, [
    [long_enough, 'Nop. Haz un esfuerzo.'],
    [no_numbers, '¿Números? No. Quítalos.']
  ]),
  email: validate.bind(null, [
    [long_enough, 'Intenta otra vez.'],
    [is_email, '¿A quién intentas engañar?']
  ])
};

Obj.ap(validations, input);
```

`Obj.ap` debería devolver esto.

```js
{
  name: [
    "Nop. Haz un esfuerzo.",
    "¿Números? No. Quítalos."
  ],
  email: [
    "Intenta otra vez.",
    "¿A quién intentas engañar?"
  ]
}
```

Si quieren saber si el formulario es válido sólo tendrían que revisar si alguna propiedad contiene errores.

```js
function is_valid(form_errors) {
  const is_empty = msg => !msg.length;
  return Object.values(form_errors).every(is_empty);
}

is_valid(Obj.ap(validations, input));
```

Luego de evaluar si los datos son válidos lo que queda es mostrar los errores al usuario. Esta parte depende mucho del contexto de su programa, no puedo mostrarles un ejemplo lo suficientemente "genérico" pero sí podemos imaginar otra situación más específica.

## Formulario de registro

Vamos a suponer que tenemos un formulario html cualquiera. Cada campo tiene esta estructura.

```html
<div class="field">
  <label class="label">Nombre Campo:</label>
  <div class="control">
    <input name="nombre-campo" class="input" type="text">
  </div>
  <ul data-errors="nombre-campo"></ul>
</div>
```

Cuando el campo es inválido queremos mostrar la lista de errores en el elemento `ul` que tiene el atributo `data-errors`.

¿Cómo empezamos? Primero necesitamos agregar una función al evento `submit` de nuestro formulario.

```js
function submit(event) {
  event.preventDefault();
}


document.forms.namedItem("miformulario")
  .addEventListener("submit", submit);
```

Nuestro siguiente paso será reunir los datos del usuario. Pero en esta escenario no sólo necesitamos el valor de los campos también necesitamos el nombre del campo. Entonces nuestro objeto va a ser un poco más complejo que en el ejemplo anterior.

Vamos a hacer una función que de la información que necesitamos del formulario.

```js
function collect_data(form) {
  const result = {};
  const formdata = new FormData(form);

  for (let entry of formdata.entries()) {
    result[entry[0]] = {
      field: entry[0],
      value: entry[1],
    };
  }

  return result;
}
```

Vamos a probarla en la función submit.

```js
function submit(event) {
  event.preventDefault();

  const input = collect_data(this);
  console.log(input);
}
```

En este punto debemos aplicar las validaciones pero la función `validate` que teníamos no será suficiente, necesitamos manejar un objeto y no una cadena de texto.

```diff
- function validate(validations, input) {
-   const error = [];
+ function validate(validations, field) {
+   const result = {...field};
+   result.errors = [];

    for(let [validation, msg] of validations) {
-     const is_valid = validation(input);
+     result.is_valid = validation(field.value);

-     if(!is_valid) {
-       error.push(msg);
+     if(!result.is_valid) {
+       result.errors.push(msg);
      }
    }

-   return error;
+   return result;
  }
```

Hemos hecho dos cosas. Primero, sacamos el valor del input de `field.value`. Segundo, en lugar de un arreglo ahora devolvemos un objeto con la siguiente "forma".

```js
{
  field: String,
  value: String,
  is_valid: Boolean,
  errors: Array
}
```

Hacemos esto porque es muy probable que necesitemos toda la información extra luego de completar el proceso de validación.

Justo como antes, vamos a fingir que tenemos que validar nombre y correo de un usuario. Vamos a usar las mismas funciones de antes y nuestro nuevo `validate`.

```js
function submit(event) {
  event.preventDefault();
  const input = collect_data(this);

  const validations = {
    name: validate.bind(null, [
      [long_enough, 'Nop. Haz un esfuerzo.'],
      [no_numbers, '¿Números? No. Quítalos.']
    ]),
    email: validate.bind(null, [
      [long_enough, 'Intenta otra vez.'],
      [is_email, '¿A quién intentas engañar?']
    ])
  };

  const formdata = Obj.ap(validations, input);
  console.log(formdata);
}
```

¿Pero saben qué? Quiero hacer algo gracioso. Quiero sacar `validations` de ahí y convertirlo en una función usando `Obj.ap.bind`.

```js
const validate_form = Obj.ap.bind(null, {
  name: validate.bind(null, [
    [long_enough, 'Nop. Haz un esfuerzo.'],
    [no_numbers, '¿Números? No. Quítalos.']
  ]),
  email: validate.bind(null, [
    [long_enough, 'Intenta otra vez.'],
    [is_email, '¿A quién intentas engañar?']
  ])
});
```

Ahora `submit` puede ser un poco más "declarativo".

```js
function submit(event) {
  event.preventDefault();

  const input = collect_data(this);
  const formdata = validate_form(input);

  console.log(formdata);
}
```

Ahora tenemos que evaluar si el formulario es válido. Para esto sólo debemos revisar si todos los campos tienen `.is_valid` en `true`. Entonces si el formulario es válido lo que queremos hacer es enviar los datos, caso contrario debemos mostrar los errores.

```js
function is_valid(formdata) {
  return Object.values(formdata).every((field) => field.is_valid);
}

function submit(event) {
  event.preventDefault();

  const input = collect_data(this);
  const formdata = validate_form(input);

  if(is_valid(formdata)) {
    send_data(input);
  } else {
    // mostrar errores
  }
}
```

Para el último paso lo que haremos será colocar un `li` por cada mensaje de error que tenga nuestro input.

```js
function show_errors(input) {
  const el = document.querySelector(`[data-errors=${input.field}]`);
  el.replaceChildren();

  for (let msg of input.errors) {
    const li = document.createElement("li");
    li.textContent = msg;
    el.appendChild(li);
  }
}
```

Esperen... aún tenemos que encargarnos de un pequeño detalle que olvidé. Un funtor aplicativo también debe tener "método" `map`, no tenemos uno pero vamos a arreglar eso.

```diff
  const Obj = {
+   map(fn, data) {
+     const result = {};
+     for (let key in data) {
+       result[key] = fn(data[key]);
+     }
+
+     return result;
+   },
    ap(Fn, Data) {
      const result = {};
      for (let key in Data) {
        result[key] = Fn[key](Data[key]);
      }

      return result;
    }
  };
```

Ya me siento mejor. Ahora vamos a usar `map` para mostrar los errores.

```js
function submit(event) {
  event.preventDefault();

  const input = collect_data(this);
  const formdata = validate_form(input);

  if(is_valid(formdata)) {
    send_data(input);
  } else {
    Obj.map(show_errors, formdata);
  }
}
```

Okey ya sé, `map` debería usarse para transformar valores. No nos centremos en detalles. Vamos a alegrarnos porque ya todo el trabajo está hecho. Aquí les dejo un formulario semi-funcional en codepen para que vean el código en acción.

{{ codepen(id="PojjGpM", title="Applicative functors and form validation", load_js=true) }}

## Conclusión

Dimos un pequeño vistazo a lo que pueden hacer los aplicativos con el método `.ap`. Sabemos que en javascript no tenemos una implementación "nativa" para esto, pero aún podemos hacer la nuestra. Y finalmente aplicamos este conocimiento para validar un formulario.

