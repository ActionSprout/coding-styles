# Ember.js Style Guide

## Table Of Contents

* [General](#general)
* [Organizing your modules](#organizing-your-modules)
* [Models](#models)
* [Controllers](#controllers)
* [Templates](#templates)
* [Routing](#routing)
* [Ember data](#ember-data)

## General

### Import what you use, do not use globals

For Ember Data, we should import `DS` from `ember-data`, and then
destructure our desired modules.
For Ember, use destructuring [as Ember's internal organization is
not intuitive and difficult to grok, and we should wait until Ember has been
correctly modularized.](https://github.com/ember-cli/ember-cli-shims/issues/53)

Once Ember has officially adopted shims, we will prefer shims over
destructuring.

```js
// Good

import Ember from 'ember';
import DS from 'ember-data';

const { computed } = Ember;

const { attr } = DS;

export default DS.Model.extend({
  firstName: attr('string'),
  lastName: attr('string'),

  fullName: computed('firstName', 'lastName', function _fullName() {
    // Code
  }),
});

// Bad

export default DS.Model.extend({
  firstName: DS.attr('string'),
  lastName: DS.attr('string'),

  fullName: Ember.computed('firstName', 'lastName', function _fullName() {
    // Code
  }),
});
```

### Destructured constants line length

* Prefer splitting a destructure in to multiple lines when there are 3 or more attributes
* Always split a destructure in to mulitple lines when it has more than one level of nesting
* In all cases, optimize for readability, even in violation of the first 2 points; when it doubt, ask a friend

### Don't use Ember's prototype extensions

Avoid Ember's `Date`, `Function` and `String` prototype extensions. Prefer the
corresponding functions from the `Ember` object.

```js
// Good

const {
  String: { w },
  computed,
  on,
} = Ember;

export default DS.Model.extend({
  hobbies: w('surfing skateboarding skydiving'),
  fullName: computed('firstName', 'lastName', function _fullName() { ... }),
  didError: on('error', function _didError() { ... })
});

// Bad

export default DS.Model.extend({
  hobbies: 'surfing skateboarding skydiving'.w(),
  fullName: function() { ... }.property('firstName', 'lastName'),
  didError: function() { ... }.on('error')
});
```

### Use `get` and `set`

Calling `someObj.get('prop')` couples your code to the fact that
`someObj` is an Ember Object. It prevents you from passing in a
POJO, which is sometimes preferable in testing. It also yields a more
informative error when called with `null` or `undefined`.

Although when defining a method in a controller, component, etc. you
can be fairly sure `this` is an Ember Object, for consistency with the
above, we still use `get`/`set`.

```js
// Good
import Ember from 'ember';

const { get, set } = Ember;

set(object, 'isSelected', true);
get(object, 'isSelected');
get(this, 'isPending');

// Ok
this.get('isPending');

// Bad
object.set('isSelected', true);
object.get('isSelected');
```

### Use brace expansion

This allows much less redundancy and is easier to read.

Note that **the dependent keys must be together (without space)** for the brace expansion to work.

```js
// Good
fullName: computed('user.{firstName,lastName}', function _fullName() {
  // Code
}),

// Bad
fullName: computed('user.firstName', 'user.lastName', function _fullName() {
  // Code
}),
```

### Prefer `readOnly` for aliases

Use the following preference order when using a computed property macro to alias an attribute in an Ember object:

1. `readOnly`

   This is the strictest of the three and should therefore be the default. It allows aliasing the property and issues an exception when a `set` is attempted.

2. `oneWay`

   This allows the property to be `set` but does not propogate that new value to the aliased property. Use `oneWay` when the alias is just a default value or when injecting a value is needed for testing.

3. `alias`

   Use `alias` if and only if you need `set`s to propogate. However, before doing so, consider if updating the value upstream could use [Data Down Actions Up](http://www.samselikoff.com/blog/data-down-actions-up/) instead of a two-way binding.

### Trailing Commas

Trailing commas must be used for multiline literals, and must not be used for single-line literals.

```js
// Good
const { get, computed } = Ember;
const {
  get,
  computed,
} = Ember;

export default Ember.Object.extend({
  foo() {
  },

  actions: {
    bar() {
    },
  },
});
```

*Trailing commas are easier to maintain; one can add or remove properties without having to maintain the commas. Also produces cleaner git diffs.*

### All functions must be named

Any function declared with the `function` keyword must have a name. Functions declared with the es6 short "fat arrow" syntax do not need a name.

This helps with debugging, as the name of the function shows up in the backtrace instead of `(anonymous function)`.

When the function is used as part of a macro to define a property, we have a convention of using the property name prefixed with an underscore.

```js
// Good

things.map((thing) => get(thing, 'name'));

fullName: computed('firstName', 'lastName', function _fullName() {
  // Code
}),

// Bad

fullName: computed('firstName', 'lastName', function() {
  // Code
}),
```

## Organizing your modules

### ES6 module files

1. __Framework imports__

   First, import Ember and ember addons.

2. __Internal imports__

   Then, separated by a blank line, import internals.

3. __Destructure constants__

   For example:

   ```js
   const { computed } = Ember;
   ```

   There are [more details about destructuring constants above](#import-what-you-use-do-not-use-globals).

4. __Object definition__

   This is the main body of the file. For details on how to order properties within the object definition, see the next section ([Ember Object Properties](#ember-object-properties)).

5. __Utility functions__

   After the object definition, at the bottom of the file, place any private utility functions.

```js
import Ember from 'ember';
import { task } from 'ember-concurrency';

import openDrawer from 'fern/utils/open-drawer';

const { computed } = Ember;

export default Ember.Object.extend({
  // all the properties, this is where the list we're talking about goes
});

function _someUtilityFunction() {
}
```

### Ember Object Properties

Ordering a module's properties in a predictable manner will make it easier to
scan.

1. __Service Injections__ (`service`)

2. __Plain properties__ (`property`)

   Start with properties that configure the module's behavior. Examples are
   `tagName` and `classNames` on components and `queryParams` on controllers and
   routes. Followed by any other simple properties, like default values for properties.

3. __Single line computed property macros__ (`single-line-function`)

   E.g. `readOnly`, `sort` and other macros. Start with service injections. If the
   module is a model, then `attr` properties should be first, followed by
   `belongsTo` and `hasMany`.

4. __Multi line computed property functions__ (`multi-line-function`)

5. __Observers__ (`observer`)

   We try not to use observers, but if we do, they should go here.

6. __Lifecycle hooks__ (`lifecycle-hooks`),

   The hooks should be chronologically ordered by the order they are invoked in.

7. __Actions__ (`actions`)

8. __Functions__ (`method`)

   Public functions first, internal functions after.


```js
export default Ember.Component.extend({
  // Service Injections
  session: inject.service(),

  // Plain properties
  tagName: 'span',
  classNames: w('as-button'),

  // Single line CP
  post: readOnly('myPost'),

  // Multi line CP
  authorName: computed('author.{firstName,lastName}', function _authorName() {
    // Code
  }),

  // Lifecycle hooks
  didReceiveAttrs() {
    this._super(...arguments);
    // Code
  },

  actions: {
    someAction() {
      // Code
    },
  },

  // Functions
  someFunction() {
    // Code
  },

  _privateInternalFunction() {
    // Code
  },
});

// Utility functions should go after the class

function _someUtilityFunction() {
  // Code
}
```

### Override init

Rather than using the object's `init` hook via `on`, override init and
call `_super` with `...arguments`. This allows you to control execution
order. [Don't Don't Override Init](https://dockyard.com/blog/2015/10/19/2015-dont-dont-override-init)

## Models

### Organization

Models should be grouped as follows:

* Services
* Attributes
* Relationships
* Properties
* Computed Properties ('single-line-function' should be above 'multi-line-function')

```js
// Good

import Ember from 'ember';
import DS from 'ember-data';

const { computed } = Ember;
const { attr, hasMany } = DS;

export default DS.Model.extend({
  // Attributes
  firstName: attr('string'),
  lastName: attr('string'),

  // Relationships
  person: belongsTo('person'),
  children: hasMany('child'),
  
  // Properties
  peopleAgeBrackets: [
    '18-24',
    '25-34',
    '35-44',
    '45-54',
    '55-64',
    '65+',
  ],

  // Computed Properties (single-line functions above multi-line functions)
  label: computed.readOnly('peopleDetails.label'),
  
  fullName: computed('firstName', 'lastName', function _fullName() {
    // Code
  }),
});

// Bad

import Ember from 'ember';
import DS from 'ember-data';

const { computed } = Ember;
const { attr, hasMany } = DS;

export default DS.Model.extend({
  children: hasMany('child'),
  firstName: attr('string'),
  lastName: attr('string'),

  fullName: computed('firstName', 'lastName', function _fullName() {
    // Code
  }),
});

```

## Routes

### Organization

Routes should be grouped as follows:

* Services
* Default route's properties
* Custom properties
* beforeModel() hook
* model() hook
* afterModel() hook
* Other lifecycle hooks in execution order (serialize, redirect, etc)
* Actions
* Custom / private methods

```
const { Route, inject: { service }, get } = Ember;

export default Route.extend({
  // 1. Services
  currentUser: service(),

  // 2. Default route's properties
  queryParams: {
    sortBy: { refreshModel: true },
  },

  // 3. Custom properties
  customProp: 'test',

  // 4. beforeModel hook
  beforeModel() {
    if (!get(this, 'currentUser.isAdmin')) {
      this.transitionTo('index');
    }
  },
  
  // 5. model hook
  model() {
    return this.store.findAll('article');
  },
  
  // 6. afterModel hook
  afterModel(articles) {
    articles.forEach((article) => {
      article.set('foo', 'bar');
    });
  },

  // 7. Other route's methods
  setupController(controller) {
    controller.set('foo', 'bar');
  },

  // 8. All actions
  actions: {
    sneakyAction() {
      return this._secretMethod();
    },
  },

  // 9. Custom / private methods
  _secretMethod() {
    // custom secret method logic
  },
});
```

## Controllers

### Define query params first

For consistency and ease of discover, list your query params first in
your controller. These should be listed above default values.

### Alias your model

It provides a cleaner code to name your model `user` if it is a user. It
is more maintainable, and will fall in line with future routable
components

```js
export default Ember.Controller.extend({
  user: computed.readOnly('model'),
});
```

## Templates

### Avoid using partials

Use components. Partials share scope with the parent view, which makes it hard
to follow the context. Components will provide a consistent scope. Only use
partials if you really, really need the shared scope.

### Don't yield `this`

Use the hash helper to yield what you need instead.

```hbs
{{! Good }}
{{yield (hash thing=thing action=(action "action"))}}

{{! Bad }}
{{yield this}}
```

### Prefer components in `{{#each}}` blocks

Contents of your each blocks should be a single line, use components
when more than one line is needed. This will allow you to test the
contents in isolation via unit tests, as your loop will likely contain
more complex logic in this case.

```hbs
{{! Good }}
{{#each posts as |post|}}
  {{post-summary post=post}}
{{/each}}

{{! Bad }}
{{#each posts as |post|}}
  <article>
    <img src={{post.image}} />
    <h1>{{post.title}}</h2>
    <p>{{post.summar}}</p>
  </article>
{{/each}}
```

### Always use the `action` keyword to pass actions.

Although it's not strictly needed to use the `action` keyword to pass on
actions that have already been passed with the `action` keyword once,
it's recommended to always use the `action` keyword when passing an action
to another component. This will prevent some potential bugs that can happen
and also make it more clear that you are passing an action.

```hbs
{{! Good }}
{{edit-post post=post deletePost=(action deletePost)}}

{{! Bad }}
{{edit-post post=post deletePost=deletePost}}
```

### Line length

* Prefer multi-line expression for HTML elements and components with more than 2 attributes
* Always ensure that HTML elements/components fit within an 96 character line
  and prefer 80 characters; if not, expand to multiline
* In all cases, optimize for readability, even in violation of the first 2 points; when it doubt, ask a friend

### Component line wrapping style

Like this:

```hbs
{{! Good }}

{{#power-select
  selected=adAccount
  options=accounts
  searchField="name"
  placeholder="-"
  onchange=(action 'selectAccount')
  as |account|
}}
  {{account.name}} ({{account.account_id}})
{{/power-select}}
```

### Ordering static attributes, dynamic attributes, and action helpers for HTML elements

Ultimately, we should make it easier for other developers to read templates.
Ordering attributes and then action helpers will provide clarity.

```hbs
{{! Good }}

<button class="wonderful-button"
  data-auto-id="click-me"
  name="wonderful-button"
  disabled={{isDisabled}}
  onclick={{action click}}
>
    Click me
</button>
```

```hbs
{{! Bad }}

<button disabled={{isDisabled}} data-auto-id="click-me" {{action (action click)}} name="wonderful-button" class="wonderful-button">Click me</button>
```

## Routing

### Route naming
Dynamic segments should be underscored. This will allow Ember to resolve
promises without extra serialization
work.

```js
// Good
this.route('foo', { path: ':foo_id' });

// Bad
this.route('foo', { path: ':fooId' });
```

[Example with broken
links](https://ember-twiddle.com/0fea52795863b88214cb?numColumns=3).

## Ember Data

### Be explicit with Ember Data attribute types

Even though Ember Data can be used without using explicit types in
`attr`, always supply an attribute type to ensure the right data
transform is used.

```js
// Good

export default DS.Model.extend({
  firstName: attr('string'),
  jerseyNumber: attr('number'),
});

// Bad

export default DS.Model.extend({
  firstName: attr(),
  jerseyNumber: attr(),
});
```

An exception is in cases where a transform is not desired.
