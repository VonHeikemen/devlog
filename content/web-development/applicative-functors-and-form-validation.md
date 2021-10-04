+++
title = "How an applicative functor can help us validate forms"
description = "Validating a form 'functional style'"
date = 2021-10-03
lang = "en"
[taxonomies]
tags = ["javascript", "functional-programming"]
+++

We will 'play' with applicative functors. To be more specific we will use it to validate some user input that comes from a form.

If you don't know what's an applicative maybe you want like summary or something... it ain't going happen. Still haven't found a way to explain them without telling you a bunch of stuff you won't need.

If you really, really want to know more about applicatives I recommend reading at least one of these.

- [Speaking of functors.](@/web-development/learn-fp/the-power-of-map.md)
- [Have you met applicative functors?](@/web-development/learn-fp/applicative-functors.md)
- [Exploring Fantasy Land.](@/web-development/tagged-unions-and-fantasy-land.md)

For now I'll tell you with an example one of the problems we can solve using applicatives.

## Imagine

Imagine a situation like this: you have some value and a regular function, you want to apply the function to the value.

```js
const value = 1;
const fn = (x) => x + 1;
```

The solution is quite simple.

```js
fn(value); // => 2
```

All good. No need for fancy stuff. But now let's say `value` and `fn` are both "trapped" inside a data structure (could be anything).

```js
const Value = [1];
const Fn = [(x) => x + 1];
```

So we have things inside arrays. Okay. But what happens if we want to apply the function? How do we proceed? Like this?

```js
[Fn[0](Value[0])]; // => [2]
```

Oh, that can't be right. In an ideal world we could do something like this.

```js
Value.ap(Fn); // => [2]
```

What we want to treat this apply operation like another method in our structure.

The bad news is that we don't live in a world where arrays can do that. The good news is we can implement `.ap` ourselves.

```js
const List = {
  ap(Fn, Value) {
    return Value.flatMap(x => Fn.map(f => f(x)));
  }
};
```

With this little helper we can solve our problem.

```js
const Value = [1];
const Fn = [(x) => x + 1];

List.ap(Fn, Value); // => [2]
```

## The next step

Let's put our attention in another structure: objects.

Imagine the same situation but this time the things we want to use are inside an object with the same "shape".

```js
const Value = {email: 'this@example.com'};
const Fn = {email: (input) => input.includes('@')};
```

What do we do? We'll take the value from one key and applied to the function with that same key.

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

And now we test.

```js
const Value = {email: 'this@example.com'};
const Fn = {email: (input) => input.includes('@')};

Obj.ap(Fn, Value); // => {email: true}
```

## Let's compose

We are making some good progress. We can apply one validation, but do you think that's enough? Probably not. There is a good chance we need to tell the user what they did wrong. Also, it would be a nice if we could apply more than one validation.

I want a list of pairs. Each pair will have a function and a message. Something like this.

```js
[
  [long_enough, 'Come on, try again.'],
  [is_email, 'Totally not an email.']
]
```

If the function returns `false` then message will be added to an array. Simple, right? Let's just turn that idea into a function.

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

Notice the `input` is the last parameter, that's because I want partially apply the function. Basically, I want to "bind" the `validations` parameter to a value without executing the function. For this I'll just use `Function.bind`.

```js
validate.bind(null, [
  [long_enough, 'Come on, try again.'],
  [is_email, 'Totally not an email.']
]);
```

There are other ways to achieve this effect but I like `.bind`.

Anyway, now let's create the validation that we want to use.

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

Now we can put everything together to make a test.

```js
const input = {
  name: '1',
  email: 'a'
};

const validations = {
  name: validate.bind(null, [
    [long_enough, 'Come on, try again.'],
    [no_numbers, "Don't get smart. No numbers."]
  ]),
  email: validate.bind(null, [
    [long_enough, 'Am I a joke to you?'],
    [is_email, 'Totally not an email.']
  ])
};

Obj.ap(validations, input);
```

`Obj.ap` should return this.

```js
{
  name: [
    "Come on, try again.",
    "Don't get smart. No numbers."
  ],
  email: [
    "Am I a joke to you?",
    "Totally not an email."
  ]
}
```

If we want to check if the form is actually valid we would have to check if any of the keys has an error message.

```js
function is_valid(form_errors) {
  const is_empty = msg => !msg.length;
  return Object.values(form_errors).every(is_empty);
}

is_valid(Obj.ap(validations, input));
```

After this all we need to do is show the error messages (if there are any) to the user. This part of the process will be very different depending on the thing you're building. I can't really show you an example that is generic and good enough for everyone. What I can do is make this imaginary scenerio a little bit more specific.

## A register form

Let's assume each field in our form looks like this in our html.

```html
<div class="field">
  <label class="label">Name of field:</label>
  <div class="control">
    <input name="field-name" class="input" type="text">
  </div>
  <ul data-errors="field-name"></ul>
</div>
```

When the input fails the validation we want to show the list of messages in the `ul` element.

Let's start with something basic, add a listener to the `submit` event in the form.

```js
function submit(event) {
  event.preventDefault();
}


document.forms.namedItem("myform")
  .addEventListener("submit", submit);
```

Now we gather the data from the user. This time around we need more than just the input, we'll also need the name of the field. So our objects are going to be a bit more complex.

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

We add it to the `submit` function.

```js
function submit(event) {
  event.preventDefault();

  const input = collect_data(this);
  console.log(input);
}
```

At this point we need to apply the validations but the current version of `validate` will not be enough. Now we need to handle an object instead of a plain string.

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

So now we pass `field.value` to the validation. And also instead of returning an array, we return an object with this shape.

```js
{
  field: String,
  value: String,
  is_valid: Boolean,
  errors: Array
}
```

We do this because we'll need all this extra data after the validation process.

Just like before, let's pretend we're just validating a name and an email. We'll use the same functions as before with our new `validate`.

```js
function submit(event) {
  event.preventDefault();
  const input = collect_data(this);

  const validations = {
    name: validate.bind(null, [
      [long_enough, 'Come on, try again.'],
      [no_numbers, "Don't get smart. No numbers."]
    ]),
    email: validate.bind(null, [
      [long_enough, 'Am I a joke to you?'],
      [is_email, 'Totally not an email.']
    ])
  };

  const formdata = Obj.ap(validations, input);
  console.log(formdata);
}
```

But you know what? I want to do something funny. I want to take `validations` out of there. I'll be turning that into a function using `Obj.ap.bind`.

```js
const validate_form = Obj.ap.bind(null, {
  name: validate.bind(null, [
    [long_enough, 'Come on, try again.'],
    [no_numbers, "Don't get smart. No numbers."]
  ]),
  email: validate.bind(null, [
    [long_enough, 'Am I a joke to you?'],
    [is_email, 'Totally not an email.']
  ])
});
```

With this our function `submit` can be a little bit more declarative.

```js
function submit(event) {
  event.preventDefault();

  const input = collect_data(this);
  const formdata = validate_form(input);

  console.log(formdata);
}
```

With validations out of the way, we need to check if the form is actually valid. To do this we will check if `.is_valid` is `true` in every field. If the form is valid we want to send the data somewhere, else we would show the error messages.

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
    // show errors
  }
}
```

In this last step we'll show each error message in an `li` element inside the `ul` of each field.

```js
function show_errors(input) {
  const el = document.querySelector(`[data-errors=${input.field}]`);
  el.replaceChildren();

  for (let msg of input.errors) {
    const li = document.createElement('li');
    li.textContent = msg;
    el.appendChild(li);
  }
}
```

But wait... one last thing. We can't have an applicative without a `map` function. Let's fix that.

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

Now I feel better. We'll use this new function to show the messages.


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

Yes, I know, I should be using a regular `for` loop because "side effects". We are done, let's not fight over details here.

To prove this stuff works I have this wonderful codepen example with a semi-functional form.

{{ codepen(id="PojjGpM", title="Applicative functors and form validation", load_js=true) }}

## Conclusion

We took a brief look at the `.ap` method we find in applicative functors. We learned that in javascript there is no such thing so we have to implement it ourselves. Finally we used our new found knowledge to validate a simple input.
