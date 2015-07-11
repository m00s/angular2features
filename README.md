# Angular 2 features

* [Annotations](#annotations)
  * [Directives](#directives)
  * [Components](#components)
* [Templates](#templates)
* [Change detection](#changedetection)
* [Core](#core)
* [DI](#di)
* [Forms](#forms)
* [HTTP](#http)
* [Pipes](#pipes)
* [Router](#router)
* [Test](#test)

## Annotations
### Directives
Directives allow you to attach behavior to elements in the DOM.

Directives are classes which get instantiated as a response to a particular DOM structure. By controlling the DOM structure, what directives are imported, and their selectors, the developer can use the [composition pattern](http://en.wikipedia.org/wiki/Object_composition) to get a desirable application behavior.

Directives are the cornerstone of an Angular application. We use Directives to break complex problems into smaller more reusable components. Directives allow the developer to turn HTML into a DSL and then control the application assembly process.

A directive consists of a single directive annotation and a controller class. When the directive's `selector` matches elements in the DOM, the following steps occur:

* For each directive, the *ElementInjector* attempts to resolve the directive's constructor arguments.
* Angular instantiates directives for each matched element using *ElementInjector* in a depth-first order, as declared in the HTML.

Here is a trivial example of a tooltip decorator. The directive will log a tooltip into the console on every time mouse enters a region:

``` javascript
@Directive({
  selector: '[tooltip]',     | CSS Selector which triggers the decorator
  properties: [              | List which properties need to be bound
    'text: tooltip'          |  - DOM element tooltip property should be
  ],                         |    mapped to the directive text property.
  host: {                    | List which events need to be mapped.
    (mouseover): 'show()'    |  - Invoke the show() method every time
  }                          |    the mouseover event is fired.
})                           |
class Form {                 | Directive controller class, instantiated
                             | when CSS matches.
  text:string;               | text property on the Directive Controller.
                             |
  show(event) {              | Show method which implements the show action.
    console.log(this.text);  |
  }
}
```

Example of usage:

``` html
<span tooltip="Tooltip text goes here.">Some text here.</span>
```

The developer of an application can now freely use the `tooltip` attribute wherever the behavior is needed. The code above has taught the browser a new reusable and declarative behavior.

Notice that data binding will work with this decorator with no further effort as shown below.

``` html
<span tooltip="Greetings {{user}}!">Some text here.</span>
```

**NOTE:** Angular applications do not have a main method. Instead they have a root Component. Dependency Injection then assembles the directives into a working Angular application.

###Components
A component is a directive which uses shadow DOM to create encapsulate visual behavior. Components are typically used to create UI widgets or to break up the application into smaller components.

* Only one component can be present per DOM element.
* Component's CSS selectors usually trigger on element names. (Best practice)
* Component has its own shadow view which is attached to the element as a Shadow DOM.
* Shadow view context is the component instance. (i.e. template expressions are evaluated against the component instance.)

Each Angular component requires a single `@Component` and at least one `@View` annotation. The `@Component` annotation specifies when a component is instantiated, and which properties and hostListeners it binds to.

When a component is instantiated, Angular

* Creates a shadow DOM for the component.
* Loads the selected template into the shadow DOM.
* Creates a child Injector which is configured with the appInjector for the Component.

Example of a component:

``` javascript
@Component({                      | Component annotation
  selector: 'pane',               | CSS selector on <pane> element
  properties: [                   | List which property need to be bound
    'title',                      |  - title mapped to component title
    'open'                        |  - open attribute mapped to component open property
  ],                              |
})                                |
@View({                           | View annotation
  templateUrl: 'pane.html'        |  - URL of template HTML
})                                |
class Pane {                      | Component controller class
  title:string;                   |  - title property
  open:boolean;

  constructor() {
    this.title = '';
    this.open = true;
  }

  // Public API
  toggle() => this.open = !this.open;
  open() => this.open = true;
  close() => this.open = false;
}
```

`pane.html`:
``` html
<div class="outer">
  <h1>{{title}}</h1>
  <div class="inner" [hidden]="!open">
    <content></content>
  </div>
</div>
```

`pane.css`:
``` css
.outer, .inner { border: 1px solid blue;}
.h1 {background-color: blue;}
```

Example of usage:
``` html
<pane #pane title="Example Title" open="true">
  Some text to wrap.
</pane>
<button (click)="pane.toggle()">toggle</button>

```
<!-- -------------------------- -->
### Templates
-----------------------------------
Templates are markup which is added to HTML to declaratively describe how the application model should be
projected to DOM as well as which DOM events should invoke which methods on the controller. Templates contain
syntax which is core to Angular and allows for data-binding, event-binding, template-instantiation.

The design of the template syntax has these properties:


* All data-binding expressions are easily identifiable. (i.e. there is never an ambiguity whether the value should be
  interpreted as string literal or as an expression.)
* All events and their statements are easily identifiable.
* All places of DOM instantiation are easily identifiable.
* All places of variable declaration are easily identifiable.

The above properties guarantee that the templates are easy to parse by tools (such as IDEs) and reason about by people.
At no point is it necessary to understand which directives are active or what their semantics are in order to reason
about the template runtime characteristics.

#### Property bindings
## Property Binding

Binding application model data to the UI is the most common kind of bindings in an Angular application. The bindings
are always in the form of `property-name` which is assigned an `expression`. The generic form is:

**Short form:** `<some-element [some-property]="expression">`

**Canonical form:** `<some-element bind-some-property="expression">`

Where:
* `some-element` can be any existing DOM element.
* `some-property` (escaped with `[]` or `bind-`) is the name of the property on `some-element`. In this case the
  dash-case is converted into camel-case `someProperty`.
* `expression` is a valid expression (as defined in section below).

Example:
```
<div [title]="user.firstName">
```

In the above example the `title` property of the `div` element will be updated whenever the `user.firstName` changes
its value.

Key points:
* The binding is to the element property not the element attribute.
* To prevent custom element from accidentally reading the literal `expression` on the title element, the attribute name
  is escaped. In our case the `title` is escaped to `[title]` through the addition of square brackets `[]`.
* A binding value (in this case `user.firstName` will always be an expression, never a string literal).

**NOTE:** Angular 2 binds to properties of elements rather than attributes of elements. This is
done to better support custom elements, and to allow binding for values other than strings.

**NOTE:** Some editors/server side pre-processors may have trouble generating `[]` around the attribute name. For this
reason Angular also supports a canonical version which is prefixed using `bind-`.

#### String Interpolation

Property bindings are the only data bindings which Angular supports, but for convenience Angular supports an interpolation
syntax which is just a short hand for the data binding syntax.

```
<span>Hello {{name}}!</span>
```

is a short hand for:

```
<span [text|0]=" 'Hello ' + stringify(name) + '!'">_</span>
```

The above says to bind the `'Hello ' + stringify(name) + '!'` expression to the zero-th child of the `span`'s `text`
property. The index is necessary in case there are more than one text nodes, or if the text node we wish to bind to
is not the first one.

Similarly the same rules apply to interpolation inside attributes.

```
<span title="Hello {{name}}!"></span>
```

is a short hand for:

```
<span [title]=" 'Hello ' + stringify(name) + '!'"></span>
```

**NOTE:** `stringify()` is a built in implicit function which converts its argument to a string representation, while
keeping `null` and `undefined` as empty strings.

#### Inline Templates
Data binding allows updating the DOM's properties, but it does not allow for changing of the DOM structure. To change
DOM structure we need the ability to define child templates, and then instantiate these templates into Views. The
Views than can be inserted and removed as needed to change the DOM structure.

**Short form:**
```
Hello {{user}}!
<div template="ng-if: isAdministrator">
  ...administrator menu here...
</div>
```

**Canonical form:**
```
Hello {{user}}!
<template [ng-if]="isAdministrator">
  <div>
    ...administrator menu here...
  </div>
</template>
```

Where:
* `template` defines a child template and designates the anchor where Views (instances of the template) will be
  inserted. The template can be defined implicitly with `template` attribute, which turns the current element into
  a template, or explicitly with `<template>` element. Explicit declaration is longer, but it allows for having
  templates which have more than one root DOM node.
* `viewport` is required for templates. The directive is responsible for deciding when
  and in which order should child views be inserted into this location. Such a directive usually has one or
  more bindings and can be represented as either `viewport-directive-bindings` or
  `viewport-directive-microsyntax` on `template` element or attribute. See template microsyntax for more details.

#### Template Microsyntax

Often times it is necessary to encode a lot of different bindings into a template to control how the instantiation
of the templates occurs. One such example is `ng-for`.

```
<form #foo=form>
</form>
<ul>
  <template [ng-for] #person [ng-for-of]="people" #i="index">
    <li>{{i}}. {{person}}<li>
  </template>
</ul>
```

Where:
* `[ng-for]` triggers the for directive.
* `#person` exports the implicit `ng-for` item.
* `[ng-for-of]="people"` binds an iterable object to the `ng-for` controller.
* `#i=index` exports item index as `i`.

The above example is explicit but quite wordy. For this reason in most situations a short hand version of the
syntax is preferable.

```
<ul>
  <li template="ng-for; #person; of=people; #i=index;">{{i}}. {{person}}<li>
</ul>
```

Notice how each key value pair is translated to a `key=value;` statement in the `template` attribute. This makes the
repeat syntax a much shorter, but we can do better. Turns out that most punctuation is optional in the short version
which allows us to further shorten the text.

```
<ul>
  <li template="ng-for #person of people #i=index">{{i}}. {{person}}<li>
</ul>
```

We can also optionally use `var` instead of `#` and add `:` to `for` which creates the following recommended
microsyntax for `ng-for`.

```
<ul>
  <li template="ng-for: var person of people; var i=index">{{i}}. {{person}}<li>
</ul>
```

Finally, we can move the `ng-for` keyword to the left hand side and prefix it with `*` as so:

```
<ul>
  <li *ng-for="var person of people; var i=index">{{i}}. {{person}}<li>
</ul>
```


The format is intentionally defined freely, so that developers of directives can build an expressive microsyntax for
their directives. The following code describes a more formal definition.

```
expression: ...                     // as defined in Expressions section
local: [a-zA-Z][a-zA-Z0-9]*         // exported variable name available for binding
internal: [a-zA-Z][a-zA-Z0-9]*      // internal variable name which the directive exports.
key: [a-z][-|_|a-z0-9]]*            // key which maps to attribute name
keyExpression: key[:|=]?expression  // binding which maps an expression to a property
varExport: [#|var]local(=internal)? // binding which exports a local variable from a directive
microsyntax: ([[key|keyExpression|varExport][;|,]?)*
```

Where
* `expression` is an Angular expression as defined in section: Expressions
* `local` is a local identifier for local variables.
* `internal` is an internal variable which the directive exports for binding.
* `key` is an attribute name usually only used to trigger a specific directive.
* `keyExpression` is an property name to which the expression will be bound to.
* `varExport` allows exporting of directive internal state as variables for further binding. If no `internal` name
  is specified, the exporting is to an implicit variable.
* `microsyntax` allows you to build a simple microsyntax which can still clearly identify which expressions bind to
  which properties as well as which variables are exported for binding.


**NOTE:** the `template` attribute must be present to make it clear to the user that a sub-template is being created. This
goes along with the philosophy that the developer should be able to reason about the template without understanding the
semantics of the instantiator directive.

<!-- -------------------------- -->
### Change detection
-----------------------------------
<!-- -------------------------- -->
### Core
-----------------------------------
<!-- -------------------------- -->
### DI
-----------------------------------
<!-- -------------------------- -->
### Forms
-----------------------------------
<!-- -------------------------- -->
### HTTP
-----------------------------------
<!-- -------------------------- -->
### Pipes
-----------------------------------
<!-- -------------------------- -->

Pipes can be appended on the end of the expressions to translate the value to a different format. Typically used
to control the stringification of numbers, dates, and other data, but can also be used for ordering, mapping, and
reducing arrays. Pipes can be chained.

**NOTE:** Pipes are known as filters in Angular v1.

Pipes syntax is:

``` javascript
<div class="movie-copy">
  <p>
    {{ model.getValue('copy') | async | uppercase }}
  </p>
  <ul>
    <li>
      <b>Starring</b>: {{ model.getValue('starring') | async }}
    </li>
    <li>
      <b>Genres</b>: {{ model.getValue('genres') | async }}
    </li>
  <ul>
</div>
```

<!-- -------------------------- -->
### Router
-----------------------------------
<!-- -------------------------- -->
### Test
-----------------------------------
